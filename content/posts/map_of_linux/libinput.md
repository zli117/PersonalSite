---
date: '2026-04-19T23:01:21-07:00'
draft: true
title: 'Chapter 6: libinput'
weight: 60
---
## What libinput is and why it exists

Getting input from a keyboard or mouse on Linux sounds like it should be easy. The kernel provides `/dev/input/event*` character devices; you open one, read structured events (`struct input_event` — a type, code, value, timestamp), and you're done. For a dumb keyboard, it basically is that easy.

Nothing about modern input is dumb.

A touchpad generates hundreds of events per second describing finger positions, pressures, contact areas, and movements. From that raw stream you have to infer: was that a tap (click) or just a rest? Is the user scrolling (two fingers moving together) or zooming (two fingers moving apart)? Are they palm-resting while typing, and if so, which contacts should be ignored? Did they just click the physical button under the pad, and if so, with which fingers on the surface? What about kinetic scrolling — should momentum continue after they lift?

A mouse needs pointer acceleration — a curve mapping physical movement to cursor movement, tuned so slow movements give precision and fast ones give reach. Different curves for different device types, different sensitivities.

A touchscreen needs tap detection, multi-finger gestures, edge swipes. A tablet with a stylus needs pressure curves, tilt, button mapping, proximity detection, double-click handling that doesn't trigger on accidental hover. A trackpoint (those little nubs on ThinkPads) needs its own acceleration.

In the X11 era, this logic lived scattered across X input drivers: `xf86-input-synaptics`, `xf86-input-evdev`, `xf86-input-wacom`, `xf86-input-mtrack`. Each reimplemented some subset of gesture/tap/accel logic, with different quality and quirks. Configurations were `xorg.conf` snippets that most users could never correctly write.

**libinput** (started 2013, Peter Hutterer at Red Hat) consolidated all of this. One library, one set of algorithms, one configuration model for every input device class. It reads raw events from `/dev/input/event*` (the evdev interface), recognizes the device type, applies appropriate logic, and emits high-level events: "key A pressed," "pointer moved by (dx, dy)," "two-finger scroll started," "tap-to-click at (x, y)."

On X11, `xf86-input-libinput` is a thin X driver that embeds libinput. On Wayland, the compositor links libinput directly. Either way, the same library makes the decisions — so your touchpad behaves identically in GNOME on Wayland, KDE on X11, or sway.

Nearly every modern Linux desktop uses libinput. The exceptions: pre-Wayland KDE used `libevdev` + custom touchpad code for a while; some embedded systems roll their own. For a Fedora / Ubuntu / Arch desktop in 2026, libinput is the answer.

## The stack underneath: evdev

Before libinput there's the kernel's **evdev** subsystem. Every input device — keyboard, mouse, touchpad, touchscreen, tablet, even the power button — shows up as a character device under `/dev/input/`:

```bash
ls /dev/input/
# by-id  by-path  event0  event1  event2  ...  mice  mouse0  mouse1
```

`event0` through `eventN` are the evdev nodes. Each represents one logical device. The kernel assigns numbers as devices appear, so they're not stable across reboots — use the symlinks under `/dev/input/by-id/` or `/dev/input/by-path/` for stable references.

An evdev event is a fixed-size struct:

```c
struct input_event {
    struct timeval time;   // timestamp
    __u16 type;            // EV_KEY, EV_REL, EV_ABS, EV_SYN, ...
    __u16 code;            // KEY_A, REL_X, ABS_MT_POSITION_X, ...
    __s32 value;           // pressed/released, displacement, coordinate
};
```

Types of event:
- `EV_KEY` — buttons and keys. `KEY_A` pressed or released.
- `EV_REL` — relative motion. `REL_X` +5 means "cursor moved right 5 units." Used by mice.
- `EV_ABS` — absolute position. `ABS_X`, `ABS_Y` on a touchscreen or tablet. `ABS_MT_*` for multi-touch.
- `EV_SYN` — synchronization marker. Groups multiple events into one atomic change ("touch moved AND pressure changed, both in the same frame").
- Plus a long tail: `EV_MSC`, `EV_LED`, `EV_SW`, `EV_SND`, `EV_FF` (force feedback) — all narrower.

Every keyboard press generates at least three events: an `EV_MSC MSC_SCAN` with the hardware scancode, an `EV_KEY KEY_A` with value 1 (pressed), and an `EV_SYN SYN_REPORT` to mark the end of the frame. Release is the mirror.

You can watch this directly:

```bash
sudo dnf install evtest            # or apt/pacman equivalent
sudo evtest
# Select a device; events stream as they happen.
```

Type a key. You'll see raw kernel-level events in real time. This is what libinput reads.

The kernel does minimal interpretation. It doesn't know "this is a touchpad" vs "this is a touchscreen"; it just reports axes and button codes. Device capabilities (which axes it supports, which keys, its name, bus type, IDs) are readable via ioctls on the evdev fd. libinput reads capabilities at startup, classifies the device, and decides what behavior to apply.

## The libinput model: contexts, devices, events

libinput organizes itself around a few objects:

**Context.** The top-level handle. A compositor or session creates one, tells it how to find devices (either by giving it a udev context, or by manually adding devices), and gets an fd to poll for events.

**Device.** One input device, wrapping an evdev fd. Has a type classification: keyboard, pointer, touchpad, touchscreen, tablet, switch. Has capabilities (supports scroll? has tap-to-click available? can do left-handed?) and configuration options.

**Event.** High-level output. Unlike raw evdev events, libinput events describe *meaning*, not just signals:

- `LIBINPUT_EVENT_KEYBOARD_KEY` — key press/release, with the keycode.
- `LIBINPUT_EVENT_POINTER_MOTION` — accelerated motion delta.
- `LIBINPUT_EVENT_POINTER_BUTTON` — button press/release.
- `LIBINPUT_EVENT_POINTER_SCROLL_WHEEL` / `_FINGER` / `_CONTINUOUS` — scrolling, with source.
- `LIBINPUT_EVENT_TOUCH_DOWN` / `_MOTION` / `_UP` / `_FRAME` — multitouch contacts.
- `LIBINPUT_EVENT_GESTURE_SWIPE_BEGIN` / `_UPDATE` / `_END` — three- or four-finger swipes.
- `LIBINPUT_EVENT_GESTURE_PINCH_BEGIN` / `_UPDATE` / `_END` — pinch-to-zoom.
- `LIBINPUT_EVENT_GESTURE_HOLD_BEGIN` / `_END` — hold gestures (newer).
- `LIBINPUT_EVENT_TABLET_TOOL_AXIS` / `_BUTTON` / `_TIP` / `_PROXIMITY` — stylus stuff.
- `LIBINPUT_EVENT_TABLET_PAD_BUTTON` / `_RING` / `_STRIP` — tablet pad buttons and rings.
- `LIBINPUT_EVENT_SWITCH_TOGGLE` — lid closed, tablet mode, etc.

The consumer (compositor) gets these events ready to act on. "Tap detected, treat as button 1 click." "Two-finger scroll, emit scroll events." No gesture logic in the compositor. No palm rejection. That's all inside libinput.

## The event pipeline

When you move your finger on the touchpad, here's what happens:

1. **Hardware sends a report** over the internal bus (I²C, USB, SMBus, whatever).
2. **Kernel driver** (i2c-hid, hid-multitouch, synaptics_i2c, etc.) parses the report and generates evdev events.
3. **evdev fd** delivers those events to any reader (libinput) via `read()`.
4. **libinput's per-device state machine** processes the events. For a touchpad: tracks each slot (one per finger in contact), computes velocity, detects taps, applies palm rejection, applies acceleration.
5. **libinput emits high-level events** via its own callback/event queue.
6. **Compositor reads libinput events** from the libinput fd and dispatches them into its input handling. For Wayland, this means deciding which surface has pointer focus and sending `wl_pointer.motion` to that client.
7. **Client (app)** receives the Wayland event, updates internal state, potentially renders a response.

Steps 4-5 are libinput's contribution. Without it, you'd have to do all of that in the compositor.

## Device detection and udev

libinput uses **udev** to discover input devices and read metadata. When a new device appears (USB dongle plugged in, Bluetooth mouse connected), udev fires a uevent; libinput subscribes and reacts by opening the new `/dev/input/event*` node.

udev also tags devices. The file `/usr/lib/udev/hwdb.d/60-input-id.hwdb` and similar contain rules that classify devices — "this vendor ID is a Wacom tablet" or "this product is a Magic Trackpad." libinput uses these tags to decide device capabilities beyond what evdev caps report.

Crucially, logind's seat ACLs gate the `/dev/input/event*` nodes. You can't open input devices without being on the active session for the seat they're attached to — exactly as with `/dev/dri/card0`. When you switch VTs, you lose input access; when you come back, you regain it.

## Tap-to-click, palm rejection, acceleration: the heuristics

The hard parts of touchpad handling all live in libinput. A brief tour of the important ones:

### Tap-to-click

Touch a touchpad briefly, lift quickly, and libinput emits a synthetic button click. Rules:
- Contact duration must be under ~180ms.
- Movement during the contact must be under a small threshold.
- Contact must be at least some minimum distance from the edge (palm heuristic).

Multi-finger taps map to different buttons: one finger = left click, two fingers = right click, three = middle. The exact mapping is configurable.

Default-off on many desktops (because false taps annoy users); GNOME enables it by default; KDE asks.

### Click-to-click (software click)

On modern clickpads (touchpads with no separate buttons — the whole surface clicks), a physical click with one finger on the pad is left-click, two fingers is right-click, three is middle. Same logic as tap-to-click but triggered by the actual physical button, not a tap.

### Palm rejection

While typing, your palms rest on the touchpad. libinput ignores contacts that:
- Appear within a few hundred ms of a keyboard event.
- Are large (high contact area — palms are bigger than fingertips).
- Are near the edges of the pad.

The key-to-palm window is tunable, but the defaults usually work.

### Acceleration

Pointer movement isn't linear. libinput applies an acceleration curve: slow finger movement = low gain (precision), fast = high gain (big sweeps). There are a few curves:

- **Flat** — linear mapping, what gamers often want.
- **Adaptive** — the default, a velocity-dependent curve tuned separately for mouse, touchpad, trackpoint.
- **Custom** — you can define your own curve as piecewise-linear points. Advanced.

The curve constants differ by device type because the underlying physical dynamics differ: a touchpad contact moves far slower than a mouse at its peak, so the curve has to compress differently.

### Middle-button emulation

Pressing both left and right mouse buttons simultaneously fires a middle-click. Off by default on multi-button mice; on by default on two-button-only devices.

### Scroll

Scroll sources differ:

- **Wheel** — a physical wheel with detents; emits discrete `wheel` scroll events.
- **Finger** — two-finger scroll on a touchpad; emits continuous `finger` scroll events, with pixel-level precision.
- **Continuous** — Apple Magic Mouse style; treats pointer surface as scroll if a gesture matches.

Compositors use the source to decide behavior: wheel scrolling scrolls in lines; finger scrolling scrolls pixel-smooth and supports kinetic flick continuation.

### Natural scrolling

"Natural scrolling" inverts the scroll direction (finger moves up → content moves up, like a touchscreen). macOS-style, increasingly the default. libinput has a per-device option.

### Kinetic scroll continuation

After you flick-scroll and lift your fingers, the scroll continues with decay. Mostly handled by the compositor/toolkit, but libinput emits the source info and end-of-scroll markers that let the app know when to decide.

## Configuration: how compositors set libinput options

libinput itself has no config file. It's a library; configuration is done by the compositor, per-device, at runtime via API calls.

On **GNOME**, settings are in gsettings under `org.gnome.desktop.peripherals.*`:

```bash
gsettings list-recursively org.gnome.desktop.peripherals.touchpad
# org.gnome.desktop.peripherals.touchpad tap-to-click false
# org.gnome.desktop.peripherals.touchpad natural-scroll true
# org.gnome.desktop.peripherals.touchpad speed 0.0
# org.gnome.desktop.peripherals.touchpad accel-profile 'default'
# ...
```

The Settings GUI writes these; `gsd-wacom`, `gsd-mouse`, `gsd-keyboard` daemons read them and push to the compositor, which calls libinput.

On **KDE**, configured via System Settings → Input Devices; stored in `~/.config/kcminputrc` or similar.

On **sway / hyprland / river**, configured in the compositor's config file directly using libinput option names:

```
# sway config example
input "1267:12357:ELAN0504:01_04F3:3091_Touchpad" {
    tap enabled
    natural_scroll enabled
    middle_emulation enabled
    pointer_accel 0.3
    accel_profile adaptive
    scroll_method two_finger
}
```

The device identifier is `vendor:product:name` or a glob (`input "type:touchpad"`).

## Hands-on: inspect input on your system

### List devices

```bash
libinput list-devices
# Long list: each device with its path, capabilities, current settings.
```

You'll see per-device info like:

```
Device:           ELAN Touchpad
Kernel:           /dev/input/event6
Group:            1
Seat:             seat0, default
Capabilities:     pointer gesture
Tap-to-click:     disabled
Tap-and-drag:     enabled
Tap drag lock:    disabled
Left-handed:      disabled
Nat.scrolling:    disabled
Middle emulation: disabled
Calibration:      n/a
Scroll methods:   *two-finger edge
Click methods:    *button-areas clickfinger
Disable-w-typing: enabled
Accel profiles:   flat *adaptive
Rotation:         n/a
```

The stars mark current values. Everything configurable shows here.

### Watch events live

```bash
sudo libinput debug-events
# Every libinput event as it happens. Move pointer, type keys, tap, scroll.
```

Output like:

```
 event4   KEYBOARD_KEY            +2.1s   KEY_A (30) pressed
 event4   KEYBOARD_KEY            +2.2s   KEY_A (30) released
 event6   POINTER_MOTION          +3.5s   0.24/-0.12 (-0.50/-0.25 unaccelerated)
 event6   POINTER_BUTTON          +4.1s   BTN_LEFT (272) pressed, seat count: 1
 event6   GESTURE_SWIPE_BEGIN     +5.0s   3
 event6   GESTURE_SWIPE_UPDATE    +5.0s   3  15.00/-3.00 (15.00/-3.00 unaccelerated)
 event6   GESTURE_SWIPE_END       +5.3s   3
```

This is *libinput's output*, post-processing. Two-finger scroll shows as `POINTER_SCROLL_FINGER`; tap shows as `POINTER_BUTTON` (synthesized). Invaluable when a compositor isn't behaving as expected — you can see whether the bad events are coming from libinput or being mangled higher up.

### Compare to raw evdev

For the same device, `sudo evtest /dev/input/event6` shows the raw events — every single `ABS_MT_POSITION_X` update, every `BTN_TOOL_FINGER`. You see the volume difference: libinput compresses a torrent of low-level events into a few high-level ones.

### Diagnose a specific device

```bash
sudo libinput record /dev/input/event6 > touchpad-session.yml
# now use the touchpad normally for 10 seconds
# Ctrl-C
```

`libinput record` captures the full evdev stream (compressed YAML). You can replay it with `libinput replay` or analyze it with `libinput analyze`. This is what libinput developers ask for in bug reports — it's reproducible, complete, and anonymized.

```bash
libinput analyze per-slot-delta touchpad-session.yml
# Plots finger motion; useful for pressure / stuck-contact issues.
```

### Measure a device's capabilities

```bash
sudo libinput measure fuzz /dev/input/event6      # check if the device reports jitter
sudo libinput measure touchpad-tap                # run the tap-to-click latency test
sudo libinput measure touch-size                  # calibrate palm-rejection thresholds
```

Diagnostic subcommands. Usually you don't run these; they're for bug reporting or device-quirk development.

### Test what a compositor will see

```bash
sudo libinput debug-gui
# Opens a window showing all input events in a visual form.
# Great for "why isn't my stylus working?" debugging.
```

A GUI that visualizes libinput output — pointer movements, touches, tablet pen position. Helpful for hardware testing.

## Tablets and styluses: a special case

libinput has a rich tablet model because digitizers (Wacom, Huion, Apple Pencil on iPad Linux hacks) have complex inputs:

- **Tool proximity** — the pen is near but not touching (`PROXIMITY_IN` / `OUT`).
- **Tool tip** — pen touched down / lifted (`TIP_DOWN` / `UP`).
- **Pressure** — how hard the pen is pressed.
- **Tilt** — pen angle from vertical.
- **Rotation** — some pens detect twist.
- **Buttons** on the pen barrel.
- **Pad buttons**, rings, strips on the tablet itself.

libinput exposes all of these via `LIBINPUT_EVENT_TABLET_TOOL_*` events. Apps like Krita, GIMP, Blender read them to implement pressure-sensitive brushes. The Wayland protocol for tablets (`tablet_v2`) is essentially a pass-through for libinput's tablet events.

On X11, `xf86-input-libinput` exposes tablet events through the X Input eXtension (XI2) protocol. Same data, different transport.

## Quirks: the weird-hardware database

Some input devices are weird. A trackpoint that reports wrong button codes. A touchscreen whose axes are inverted. A keyboard with an unusual scancode for the Fn key. libinput handles these with a **quirks database**: `/usr/share/libinput/*.quirks`.

A quirks file is INI-format; each section matches a device and applies overrides:

```ini
[Some Weird Touchpad]
MatchUdevType=touchpad
MatchName=*Weird Touchpad*
AttrTouchSizeRange=25:20
ModelPressurePad=1
```

When libinput starts on a matching device, it applies the quirks before computing defaults.

You can list applied quirks:

```bash
libinput list-quirks /dev/input/event6
# Shows every quirk matching this device.
```

For new or uncommon hardware, quirks sometimes need to be added upstream. If your touchpad is misbehaving and `libinput debug-events` shows obviously wrong values, a quirk file fix is often the answer.

## How libinput fits with Wayland and X11

### Wayland

The compositor embeds libinput. It polls libinput's fd, processes events, and translates them into Wayland protocol events:

- Pointer motion → `wl_pointer.motion` to the focused surface's client.
- Keyboard keys → `wl_keyboard.key` (plus keymap negotiation for scancode-to-keysym translation — compositor ships a keymap to clients, clients do the mapping).
- Touch → `wl_touch.down/up/motion`.
- Gestures → `zwp_pointer_gestures_v1.swipe/pinch_*` extension events.
- Tablets → `zwp_tablet_v2` extension events.

The compositor decides focus. A Wayland client only sees input events when its surface is focused. libinput generates the events; the compositor routes them.

Configuration flows GUI → gsettings/kconfig → compositor → libinput API.

### X11

On X.Org, `xf86-input-libinput` is a driver module that embeds libinput. The X server loads it, it registers as an input driver for matching devices, and translates libinput events into XI2 events that X clients receive.

Configuration is `xorg.conf.d` snippets with `Option` lines:

```
Section "InputClass"
    Identifier "libinput touchpad catchall"
    MatchIsTouchpad "on"
    Driver "libinput"
    Option "Tapping" "on"
    Option "NaturalScrolling" "true"
EndSection
```

Desktop Settings GUIs abstract this, usually via `xinput set-prop` calls that set X properties the driver reads.

### The same algorithms everywhere

This is the point: whether you're on Wayland or X, GNOME or KDE, Fedora or Ubuntu, the input processing is libinput. Same tap detection, same palm rejection, same acceleration curves. Consistency is the main win.

## Things that trip people up

**"My touchpad stopped working after the display sleeps."** Usually a seat/ACL issue; the session lost input access when it went inactive. `loginctl show-session $XDG_SESSION_ID` should show `Active=yes` when you're using it. If not, a logind or compositor bug.

**"Tap-to-click doesn't work."** Default off on many compositors/distros. Check `libinput list-devices`; the `Tap-to-click` line will say disabled. Enable via your desktop's settings or compositor config.

**"Pointer acceleration feels wrong."** Different accel profiles; try `flat` if you want raw. In GNOME: `gsettings set org.gnome.desktop.peripherals.mouse accel-profile 'flat'`.

**"Scroll direction is wrong."** Natural scrolling toggle; per-device setting.

**"My gaming mouse's extra buttons don't work."** libinput forwards them as `BTN_SIDE`, `BTN_EXTRA`, etc. The issue is usually the *app* or *compositor* not binding those. `libinput debug-events` will confirm they're arriving.

**"My tablet pen draws jittery lines."** Pressure curve or fuzz filter. Check quirks; run `libinput measure fuzz`.

**"I want a feature that isn't configurable."** libinput has a firm philosophy of "sensible defaults, minimal knobs." Peter Hutterer (lead dev) is explicit that adding options is avoided; the defaults are where the work goes. Controversial, but the reasoning is that exposing every knob pushes complexity onto users. If a behavior seems wrong *for everyone*, the fix is upstream in libinput itself, not a config option.

**"My old keymap doesn't work."** Keymap logic lives outside libinput — it's in `xkbcommon` (a separate library). libinput emits scancodes; the compositor uses xkbcommon to translate them via the selected layout.

**"My touchscreen treats palms as taps."** Palm rejection on touchscreens is harder than on touchpads because there's no keyboard correlation. libinput uses edge rejection and size thresholds; some devices have quirks.

**"After kernel upgrade, my touchpad is wrong."** The kernel driver fingerprints the device differently; libinput falls back to defaults. Check `dmesg` for device probing, compare `libinput list-devices` output to known-good.

**"libinput debug-events shows nothing."** You need to be on the active seat. Run as root, or temporarily switch to your user's active session.

**"xorg.conf Option doesn't do anything on Wayland."** Right — Wayland compositors don't read `xorg.conf`. Use the compositor's own config.

## How libinput fits with everything else

- **vs. evdev.** evdev is the kernel interface libinput reads from. You could bypass libinput and read evdev directly (some programs do for games, macro recorders). But you lose all gesture/palm/accel logic.
- **vs. xkbcommon.** xkbcommon handles keyboard *layouts* — scancode to keysym mapping (`KEY_A` in QWERTY US → 'a'; same key in Dvorak → something else). libinput doesn't interpret scancodes; it passes them up. xkbcommon and libinput are used together by every Wayland compositor.
- **vs. udev.** udev discovers input devices and tags them; libinput reads the tags.
- **vs. logind.** logind grants ACL access to `/dev/input/event*` for the active session.
- **vs. the compositor.** Compositor is libinput's consumer. Decides focus, dispatches events, handles keybindings.
- **vs. Wayland / X11.** The protocols delivering input to apps. libinput feeds them; it's not aware of them.
- **vs. toolkits.** GTK, Qt, etc. receive Wayland or X11 events, translate to their own event abstractions, deliver to app code. libinput is invisible to them.
- **vs. HID.** HID is the *USB/Bluetooth* layer — the protocol a USB mouse uses on the wire. `hid-generic` and `hid-multitouch` in the kernel parse HID reports and emit evdev events. libinput is one step up from HID.

## Quick reference card

```bash
# Inspect
libinput list-devices                        # every device and its current config
libinput list-quirks /dev/input/eventN       # what device quirks apply

# Live debug
sudo libinput debug-events                   # libinput output (post-processed)
sudo libinput debug-events --show-keycodes   # include key values
sudo evtest                                  # raw evdev (pre-libinput)
sudo libinput debug-gui                      # visual event display

# Record / analyze
sudo libinput record > session.yml           # capture evdev stream
libinput replay session.yml                  # replay
libinput analyze per-slot-delta session.yml  # per-finger motion analysis

# Measure (calibration / bug-report aids)
sudo libinput measure fuzz
sudo libinput measure touchpad-tap
sudo libinput measure touch-size

# Configuration (examples by desktop)
# GNOME:
gsettings list-recursively org.gnome.desktop.peripherals.touchpad
gsettings set org.gnome.desktop.peripherals.touchpad tap-to-click true
gsettings set org.gnome.desktop.peripherals.mouse accel-profile 'flat'

# KDE:
# System Settings → Input Devices (GUI), or edit ~/.config/kcminputrc

# sway / wlroots compositors:
# In your config file:
#   input <identifier> {
#       tap enabled
#       natural_scroll enabled
#       accel_profile adaptive
#   }

# X11 legacy:
xinput list
xinput list-props <device-id>
xinput set-prop <device-id> "libinput Tapping Enabled" 1

# Device identification
ls -la /dev/input/by-id/
ls -la /dev/input/by-path/
cat /proc/bus/input/devices                 # all known input devices, capabilities

# Kernel-level inspection
sudo dmesg | grep -i input
cat /sys/class/input/eventN/device/name
cat /sys/class/input/eventN/device/uevent
```

## Where to learn more

- **libinput documentation** — `https://wayland.freedesktop.org/libinput/doc/latest/`. Comprehensive, per-feature pages, excellent diagrams of the event pipeline. The single best resource.
- **`man libinput`, `man libinput-debug-events`, `man libinput-record`** — CLI references.
- **Peter Hutterer's blog** (`who-t.blogspot.com`) — the lead developer writes in-depth posts on design decisions, pointer acceleration curves, touchpad gestures. If you want the *why*, this is where it lives.
- **`Documentation/input/` in the kernel tree** — for the evdev substrate.
- **Quirks files** in `/usr/share/libinput/*.quirks` — reading these is educational; every weird device family is represented.
- **The `libinput-tools` package** (may be called `libinput-utils` on some distros) contains the `debug-events`, `record`, `replay`, `measure`, and `analyze` subcommands.

The mental model to carry: **libinput is a userspace library that reads raw kernel input events from evdev, classifies devices (keyboard, pointer, touchpad, touchscreen, tablet, switch), applies consistent processing — pointer acceleration, tap detection, palm rejection, multi-finger gesture recognition, kinetic scrolling heuristics — and emits high-level events that compositors turn into Wayland/X11 protocol messages and ultimately deliver to apps.** Once you grasp that input processing is *one library* used by every modern desktop, and that its defaults are the defaults by design, the input ecosystem stops looking like a bag of weird X config files and becomes a single consistent layer you can inspect, debug, and configure per-compositor.