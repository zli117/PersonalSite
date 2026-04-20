---
date: '2026-04-19T23:36:06-07:00'
draft: true
title: 'Chapter 13: GUI overview'
weight: 130
---

## Why this piece exists

"GUI" is one word that hides six separable problems. When people say "the Linux graphical stack is complicated," they usually mean they can't draw a clear line from the kernel to the pixel, don't know where X11 ends and the compositor begins, can't remember whether a toolkit is part of GNOME or GNOME is part of a toolkit, and assume it's all probably NVIDIA's fault.

The complexity is real, but it's layered. Each layer is learnable. This piece is the map — an orientation to what the graphical stack *is*, which pieces live at which layer, and how they fit. It doesn't replace the deep dives on DRM/KMS, libinput, or Wayland; it sits between them, naming the whole landscape so the deep dives have context.

By the end, you should be able to answer: when I click a button in Firefox, what software is involved between my finger and the pixel that confirms the click? And inversely: why does that same question have a different answer on a different Linux desktop?

## The six things a "GUI" actually does

A graphical session has to handle six distinct concerns. Almost every piece of software in the stack exists to handle one or two of them:

1. **Talk to the GPU.** Allocate buffers, run shaders, submit command streams, wait for completion.
2. **Set display modes.** Probe what monitors are connected, at what resolutions, and configure the display hardware to drive them.
3. **Read input devices.** Keyboards, mice, touchpads, touchscreens, tablets — from raw hardware events to meaningful gestures.
4. **Compose windows.** Take rendered pixel buffers from each app, arrange them spatially, scale them, and produce a single frame for the display.
5. **Provide a windowing protocol.** A standard way for apps and the compositor to talk: here's my window, here's your input event, please resize me.
6. **Render widgets.** The buttons, text fields, menus, scroll bars that apps are actually built from.

Each concern is a layer. Different projects handle different subsets. Understanding *which* concern a given piece of software handles is half the battle.

## The stack, bottom to top

```
   ┌─────────────────────────────────────────────────────────────┐
   │            APPLICATION                                       │
   │   (Firefox, gedit, your code editor, a game)                 │
   └─────────────────────────────────────────────────────────────┘
                              │
                              ▼
   ┌─────────────────────────────────────────────────────────────┐
   │            TOOLKIT                                           │
   │   GTK, Qt, Electron, SDL, winit, egui, ...                   │
   │   (widgets, layouts, theming, event loop, accessibility)     │
   └─────────────────────────────────────────────────────────────┘
                              │
                 ┌────────────┴────────────┐
                 ▼                         ▼
   ┌──────────────────────┐     ┌──────────────────────┐
   │   WAYLAND PROTOCOL   │     │   X11 PROTOCOL       │
   │   (native path)      │     │   (legacy path)      │
   └──────────┬───────────┘     └──────────┬───────────┘
              │                            │
              ▼                            ▼
   ┌──────────────────────┐     ┌──────────────────────┐
   │   WAYLAND COMPOSITOR │     │   X.Org SERVER       │
   │   mutter, kwin,      │     │   (or XWayland,      │
   │   sway, hyprland     │     │    running under a   │
   │                      │     │    Wayland compositor)
   └──────────┬───────────┘     └──────────┬───────────┘
              │                            │
              └────────────┬───────────────┘
                           ▼
   ┌─────────────────────────────────────────────────────────────┐
   │            libinput                                          │
   │   (touchpad gestures, palm rejection, accel curves)          │
   │                                                              │
   │            Mesa + libdrm                                     │
   │   (OpenGL/Vulkan → hardware commands, buffer sharing)        │
   └─────────────────────────────────────────────────────────────┘
                              │
                              ▼
   ┌─────────────────────────────────────────────────────────────┐
   │            KERNEL                                            │
   │   DRM/KMS (GPU driver + display)                             │
   │   evdev   (/dev/input/event*)                                │
   │   drivers: amdgpu, i915, xe, nouveau, nvidia, msm, ...       │
   └─────────────────────────────────────────────────────────────┘
                              │
                              ▼
                        HARDWARE (GPU, monitor, keyboard, mouse)
```

Each layer has its own concerns, its own vocabulary, and its own failure modes. The rest of this piece walks them bottom-up.

## Layer 1: The kernel — DRM/KMS and evdev

The kernel owns two graphical subsystems.

**DRM/KMS** is the GPU and display subsystem. DRM handles rendering — memory allocation, command submission, GPU scheduling, sync objects. KMS handles display — enumerating connectors, reading EDID, setting modes, programming the hardware to scan out buffers at the right timing. Together they live behind `/dev/dri/card*` (privileged, for mode setting) and `/dev/dri/renderD128+` (unprivileged, for rendering).

**evdev** is the kernel's input subsystem. Every input device — keyboard, mouse, touchpad, touchscreen, joystick — surfaces as `/dev/input/event*`, streaming structured `input_event` records (type, code, value, timestamp). The kernel doesn't interpret: it doesn't know "this is a touchpad" vs "this is a touchscreen"; it just reports axes and button codes. Classification happens in userspace.

Access to both is gated by **logind**: when you log in at the physical seat, logind grants your session ACLs on `/dev/dri/card*` and `/dev/input/event*`. When you switch VTs, you lose access; when you come back, you regain it.

Each subsystem has its own guide; this piece doesn't reprise them. The point here is to establish: **everything graphical starts here**. Every pixel you see traveled through KMS; every keypress you made came through evdev.

## Layer 2: Userspace GPU and input libraries

The kernel exposes raw capabilities; userspace turns them into usable APIs.

### Mesa + libdrm

GPU drivers on Linux split into two halves. The kernel half (`amdgpu`, `i915`, `xe`, `nouveau`, `nvidia-drm`) handles memory and command submission. The userspace half takes app-level APIs — OpenGL, Vulkan, OpenCL, OpenCL, VA-API for video decode — and compiles them into hardware-specific command streams the kernel driver will submit.

**Mesa** is the project that implements the userspace half for every open-source driver: AMD (via `radv` for Vulkan, `radeonsi` for GL), Intel (`anv`, `iris`, `hasvk`), NVIDIA via `nouveau` and the newer `NVK`, Qualcomm via `turnip`, ARM via `panvk`. It's one of the largest projects in the Linux userspace graphics world.

**libdrm** is the thin syscall-wrapping library that everyone uses to talk to the kernel DRM interface. Mesa depends on it; so do compositors.

NVIDIA's proprietary driver ships its own closed-source userspace libraries that parallel Mesa's functionality, talking to NVIDIA's kernel module. This is the historical reason NVIDIA has been rough on Linux: the open-source ecosystem assumes Mesa conventions; NVIDIA does things its own way.

### libinput

The input-side parallel. Reads raw evdev events, classifies the device, applies the processing that turns noise into meaning: tap-to-click, palm rejection, acceleration curves, two-finger scroll, pinch-to-zoom, kinetic scroll continuation. Emits high-level events that compositors forward to apps.

libinput replaced a constellation of X11 input drivers (`xf86-input-synaptics`, `xf86-input-evdev`, etc.) and now serves every Wayland compositor *and* X.Org (via the thin `xf86-input-libinput` driver). One library, one behavior, across every desktop.

Both Mesa and libinput are consumed by the next layer up.

## Layer 3: The display server / compositor

Here's the central fork in Linux graphical history.

### The X11 model (1984–present)

The X Window System was designed for 1984's world: network-transparent remote GUIs, clients that might be mainframes, a server process owning the hardware and drawing on behalf of clients. One X server process (`Xorg`) owned the display, arbitrated between clients, and implemented a network-transparent protocol.

The key 1984 decision: the server did the drawing. Clients sent drawing commands; the server rendered them. This made sense when a client was 300 miles away; it made less sense when clients wanted to use GPUs for their own rendering.

X accumulated extensions to adapt — Composite (1998+), Damage, XRender, DRI2, DRI3, Present — letting clients render to offscreen buffers and hand pixmaps to a compositor. But the core model was a liability. The X server grew to ~400k lines of C with decades of accumulated legacy, an input system that predated modern multitouch, a security model from the pre-networking era (any client could read any other client's buffers, inject events, screenshot the whole screen), and a codebase nobody wanted to evolve further.

X.Org ran most Linux desktops from ~1990 to the mid-2020s. It still runs many, and it still works — but no new development is happening on its core.

### The Wayland model (2008–present)

Wayland is a redesign that started in 2008 (Kristian Høgsberg) and reached practical dominance in the 2020s. Key architectural differences from X:

- **There's no separate display server.** The *compositor* is the Wayland server. One process does compositing and protocol handling; no more cooperation between three separate programs (app, server, WM+compositor).
- **Clients render their own buffers** using GPU APIs (EGL/Vulkan) or CPU rendering, then hand file descriptors to the compositor. The compositor composites completed buffers; it doesn't draw for clients.
- **Strict security model.** A Wayland client can't see other clients' buffers, inject events into other clients, read global keyboard state, or screenshot the screen without explicit compositor cooperation. What X allowed freely, Wayland requires mediation for.
- **No network transparency.** Built into the protocol's assumptions; remote display needs separate tools (VNC, RDP, waypipe).

The protocol is a local, object-oriented RPC over a Unix socket, with fd passing for buffers. Core is minimal; everything interesting lives in extensions.

### Who the compositors are

| Compositor | Desktop | Base library |
|---|---|---|
| `mutter` | GNOME | own (internal to GNOME) |
| `KWin` | KDE Plasma | own (Qt-based) |
| `sway` | sway (i3-like tiling) | wlroots |
| `hyprland` | Hyprland (dynamic tiling) | wlroots (forked) |
| `river` | river | wlroots |
| `cosmic-comp` | COSMIC (System76) | smithay (Rust) |
| `weston` | reference | own |
| `gamescope` | Steam Deck / gaming | own |

Most modern compositors are either part of a desktop environment (mutter, KWin, cosmic-comp) or wlroots-based (sway, hyprland, river, and dozens more). wlroots is a library of reusable compositor primitives — scene graph, atomic KMS commits, output management, input routing — that made "write your own compositor" a weekend project instead of a year's work.

### Wayland vs X11: where we are now

As of 2026, Wayland has won for new development. GNOME, KDE, Sway, Hyprland all default to Wayland on modern distros. NVIDIA's proprietary driver, historically the biggest Wayland holdout, gained proper explicit-sync support in 2023-2024 and is now workable.

But most Linux desktops run a **mixed stack**: the compositor is Wayland, but a bunch of apps are still X11 clients. The bridge is **XWayland** — an X11 server that runs *as a Wayland client*, providing X11 compatibility without a separate X.Org installation.

When you run an X11-era Electron app under GNOME on Wayland:

1. The app talks X11 protocol.
2. XWayland, running as a Wayland client, receives the X11 calls.
3. XWayland translates: for each X window, it creates a Wayland surface; X drawing operations become GPU rendering into buffers; Wayland protocol delivers the buffers to mutter.
4. mutter composites the XWayland surfaces alongside native Wayland surfaces.

This works remarkably well. You can tell which windows are X clients with `xlsclients`. Fractional scaling is the main remaining rough edge — X11 apps don't understand per-surface scale factors, so they either render at 1x (too small) or 2x (right-size but heavy) on HiDPI displays.

## Layer 4: The windowing protocol

The protocol between apps and the compositor. Either Wayland or X11.

For X11, the protocol is largely "legacy but stable." Depth lives in the extension documentation; most developers don't touch it directly and rely on toolkits.

For Wayland, the protocol is the one covered in depth in its own guide. Core is tiny (`wl_compositor`, `wl_surface`, `wl_seat`, and a handful more); the interesting functionality lives in extensions:

- **`xdg-shell`** — the windowing protocol for desktops. Almost universal: `xdg_toplevel` is a top-level window, `xdg_popup` is a popup menu.
- **`wlr-*`** — the wlroots-originated protocols: `wlr-layer-shell` for panels/bars/wallpapers, `wlr-screencopy` for screen capture, `wlr-output-management` for monitor config. Implemented by wlroots compositors and KDE; not by GNOME.
- **`wp-*`** — staging-to-stable cross-compositor protocols: fractional scale, color management, tearing control, presentation time.
- **`ext-*`** — newer stable cross-compositor protocols: idle notification, session lock.

Protocol extension fragmentation is real. A bar written for `wlr-layer-shell` doesn't run on GNOME. GNOME and KDE have been slowly standardizing around `ext-*` and `wp-*` protocols, but the gap exists.

## Layer 5: xdg-desktop-portal and the security bridge

Wayland's strict isolation — clients can't screenshot, can't record the screen, can't do global shortcuts, can't pick files outside their own — would make a lot of legitimate apps impossible. The bridge is **xdg-desktop-portal**, a D-Bus service running on the session bus that mediates these operations.

An app calls a portal method like `org.freedesktop.portal.Screenshot` or `.FileChooser` or `.ScreenCast`. The portal shows a consent UI ("this app wants to take a screenshot — allow?"), performs the operation if approved, returns the result. Under the hood, the portal uses compositor-specific Wayland extensions to actually capture or inject — but the app only sees the uniform D-Bus API.

This is also the path for **Flatpak and Snap apps**: sandboxed apps have only D-Bus session bus access, and portals are how they reach anything outside their sandbox. Native Wayland apps also use portals for screenshots and screen capture, even without sandboxing.

```bash
busctl --user list | grep portal
# org.freedesktop.portal.Desktop
busctl --user tree org.freedesktop.portal.Desktop
```

The portal is a *different* GNOME/KDE/wlroots implementation per desktop, but the API is uniform. If portals are broken, lots of modern apps break — and because many users never notice their screenshots come through portals, "screenshot doesn't work" becomes a diagnostic puzzle.

## Layer 6: Toolkits

The toolkit is what application developers actually program against. It provides widgets, layouts, theming, event loops, accessibility hooks. The toolkit handles the Wayland/X11 protocol details so apps don't have to.

The major toolkits:

| Toolkit | Language | Used by |
|---|---|---|
| **GTK** (3, 4) | C (bindings for most languages) | GNOME apps, Firefox, GIMP, Inkscape, much of XFCE, Cinnamon |
| **Qt** (5, 6) | C++ (QML for declarative UI) | KDE apps, Telegram, VLC, OBS, Krita, many independents |
| **Electron** | Chromium + Node.js | VS Code, Slack, Discord, Obsidian, Signal, countless others |
| **Tauri** | Rust + system webview | Lighter-weight Electron alternative |
| **SDL2/3** | C | Games, emulators |
| **GLFW** | C | OpenGL/Vulkan apps, games, tools |
| **winit** | Rust | Cross-platform Rust GUI apps |
| **egui, iced, slint, xilem** | Rust | Newer Rust native toolkits |
| **Flutter** | Dart | Cross-platform, growing Linux support |
| **Tk, FLTK, wxWidgets** | various | Older/niche |

Each toolkit carries its own visual style. GTK apps follow Adwaita (GNOME's human interface guidelines). Qt apps follow Breeze (KDE's) or their own. Electron apps do their own thing, usually matching the app's web design. This is why "Linux doesn't have a consistent look" — it's the natural consequence of toolkit plurality.

Toolkits also embed:

- **Fonts** — fontconfig discovers, FreeType rasterizes, HarfBuzz shapes. Every toolkit uses all three underneath.
- **Rendering** — Cairo for 2D vector, or direct GPU via GL/Vulkan, or sometimes Skia (Chromium's engine).
- **Accessibility** — AT-SPI over D-Bus, letting screen readers (Orca) work.
- **Input methods** — IBus or Fcitx over D-Bus, for CJK/Unicode composition.

## Layer 7: Desktop environments

A **desktop environment** is a curated bundle of compositor + session manager + settings daemon + file manager + terminal + text editor + image viewer + stock apps, with a consistent theme and human-interface guidelines.

The big ones:

- **GNOME** — opinionated, minimalist, uses mutter as the compositor, GTK as the toolkit. Dominant default on Fedora, Debian, Ubuntu.
- **KDE Plasma** — highly configurable, KWin compositor, Qt toolkit. More traditional desktop with many knobs.
- **XFCE** — lightweight, GTK, traditional (desktop, panel, menu). Largely still X11.
- **MATE** — GNOME 2 fork, GTK.
- **Cinnamon** — GNOME 3 fork (Linux Mint default), GTK.
- **LXQt** — lightweight Qt desktop.
- **Budgie** — smaller polished DE.
- **Pantheon** — elementary OS's desktop, Vala-based, polished.
- **COSMIC** — System76's new Rust-based DE.

Or you can skip a DE entirely and assemble your own: a compositor (sway, hyprland, river) + a bar (waybar, eww) + a launcher (wofi, rofi, fuzzel) + a notification daemon (mako, dunst, swaync) + a terminal + whatever else. This is the tiling-window-manager path — more assembly required, tighter to your preferences.

A DE is not a toolkit — GNOME uses GTK, but GTK isn't GNOME. A DE is not a compositor — GNOME *bundles* mutter, but you can run mutter standalone. A DE is not a set of apps — though most ship their own. What a DE *is* is a curated stack with consistent defaults.

## A complete operation, layer by layer

Trace: you press Ctrl+Alt+T in GNOME on Wayland to open a terminal.

1. **Hardware.** Keyboard generates a USB HID report.
2. **Kernel.** `usbhid` or `hid-multitouch` driver parses the HID report. evdev emits `EV_KEY` events on `/dev/input/event5`.
3. **libinput** (inside mutter). Reads evdev, classifies the device as a keyboard, emits a key event with the scancode.
4. **mutter.** Consults its keybinding table. Ctrl+Alt+T is bound (in `org.gnome.settings-daemon.plugins.media-keys.terminal` — read via gsettings at startup) to "launch default terminal."
5. **mutter** reads `org.gnome.desktop.default-applications.terminal` via gsettings: `gnome-terminal`.
6. **mutter** spawns `gnome-terminal` via `g_spawn_async` (ultimately `fork` + `execve`).
7. **`gnome-terminal` starts.** Links against GTK.
8. **GTK** reads `$WAYLAND_DISPLAY`, connects to the Wayland socket at `$XDG_RUNTIME_DIR/wayland-0`.
9. **GTK** obtains the Wayland registry, binds `wl_compositor`, `xdg_wm_base`, `wl_seat`, `zwp_linux_dmabuf_v1`, and a dozen other globals.
10. **GTK** creates a surface: `wl_compositor.create_surface`, then `xdg_surface.get_toplevel`. Sets title, app_id.
11. **GTK** renders the terminal widget into a GPU buffer using Mesa's OpenGL (or increasingly Vulkan) backend. Allocates the buffer via DRM GEM, exports as dma-buf fd.
12. **GTK** sends the dma-buf fd to mutter via `wl_surface.attach` (which uses SCM_RIGHTS to pass the fd over the Unix socket).
13. **GTK** sends `wl_surface.commit` — the atomic "apply all pending state" signal.
14. **mutter** imports the dma-buf as a GEM handle in its own context. Adds the surface to its scene graph.
15. **mutter** composites its scene into a framebuffer via GPU commands, or (if the surface can go on a hardware overlay plane) assigns the buffer to a plane directly.
16. **mutter** submits an atomic KMS commit: new framebuffer on the primary plane, with `PAGE_FLIP_EVENT` requested.
17. **Kernel DRM/KMS** programs the AMD (or Intel, or NVIDIA) display hardware: at the next vblank, scan out the new framebuffer.
18. **Display hardware** reads the buffer over the panel's link (DisplayPort / eDP / HDMI).
19. **Panel pixels change.** Photons travel to your eye.
20. **Kernel DRM/KMS** fires the page-flip completion event on mutter's `/dev/dri/card0` fd.
21. **mutter** reads the event, emits `wl_surface.frame` callback to gnome-terminal, sends `wl_buffer.release` when the buffer is no longer needed.
22. **gnome-terminal** knows the frame was shown, can render the next one.

Every step observable: evdev with `evtest`, libinput with `libinput debug-events`, Wayland with `WAYLAND_DEBUG=1 gnome-terminal`, GPU with `amdgpu_top`, KMS state with `drm_info`. No step is magic; each is a normal syscall or protocol message.

## How each piece matches to each layer

A cheat sheet for "when someone mentions X, what layer is that?"

| Name | Layer |
|---|---|
| `amdgpu`, `i915`, `xe`, `nouveau` | Kernel — DRM driver |
| `/dev/dri/card0` | Kernel — DRM device node |
| `/dev/input/event5` | Kernel — evdev device node |
| Mesa | Userspace GPU library |
| `radv`, `radeonsi`, `anv`, `iris` | Mesa drivers |
| libdrm | Userspace — syscall wrapper |
| libinput | Userspace — input library |
| xkbcommon | Userspace — keymap library |
| X.Org / Xorg | Display server (X11) |
| mutter, KWin, sway, hyprland | Wayland compositors |
| wlroots | Compositor library |
| XWayland | X11 server as Wayland client |
| Wayland protocol | App↔compositor protocol |
| X11 protocol | App↔X server protocol |
| xdg-shell | Wayland extension — desktop windows |
| wlr-layer-shell | Wayland extension — panels/bars |
| xdg-desktop-portal | D-Bus service — sandboxed-app bridge |
| GTK, Qt, Electron | Toolkits |
| Cairo, Skia | 2D rendering libraries (inside toolkits) |
| FreeType, HarfBuzz, fontconfig | Font stack (inside toolkits) |
| AT-SPI | Accessibility bus (over D-Bus) |
| IBus, Fcitx | Input methods (over D-Bus) |
| gnome-shell | GNOME's compositor + shell + top bar |
| Plasma | KDE's shell + panels |
| GNOME, KDE, XFCE, sway | Desktop environments |

## Common troubleshooting, by layer

When a graphical problem happens, the layer it lives in determines what tool to reach for.

**"The display is black / doesn't show."** Kernel or Mesa layer. `dmesg | grep drm`, `ls /dev/dri/`, try `simpledrm` fallback, check whether the GPU driver loaded.

**"My Wayland session won't start / falls back to X11."** Compositor-to-Mesa/driver layer. `journalctl -u gdm -b` or `journalctl --user -b` for session failures. Usually an NVIDIA/driver incompatibility or a missing Mesa backend.

**"Input device doesn't work."** libinput / evdev layer. `sudo libinput debug-events` to see libinput; if nothing shows, it's below libinput. `sudo evtest` to check kernel evdev.

**"App runs but blurry on HiDPI."** Toolkit or XWayland layer. Almost always an X11 app running under XWayland without scale support.

**"Screenshot / screen record doesn't work."** Portal layer. `systemctl --user status xdg-desktop-portal` and backend-specific portals.

**"Copy/paste between windows is flaky."** Compositor + XWayland bridging. Some compositors do this well, some poorly, especially crossing the X11/Wayland boundary.

**"Fonts render wrong / are tiny / fuzzy."** Font stack (fontconfig, FreeType, HarfBuzz). Usually fontconfig misconfiguration.

**"App won't theme correctly."** Toolkit layer. GTK apps read GTK theme settings; Qt apps read Qt theme settings; trying to theme one with the other's config is a common mistake.

**"Tearing in games."** Compositor + DRM layer. Enable VRR if supported, or enable `tearing-control` protocol, or fullscreen unredirect (old X11 trick).

**"Keyboard shortcut doesn't work when window isn't focused."** Wayland security by design. Use `xdg-desktop-portal`'s `GlobalShortcuts` API (newer), or a compositor-specific hotkey config (sway, hyprland).

**"Two monitors at different scales look wrong."** Fractional scaling support in compositor + toolkit. Modern GNOME/KDE handle this; some toolkits (especially XWayland apps) don't.

The pattern: **identify the layer, then reach for that layer's tool.** Shotgunning diagnostics rarely helps.

## How this fits with the rest of the series

The graphical stack sits on top of everything else covered in the series:

- **Syscalls, sysfs, procfs** — the foundation every graphical process uses to talk to the kernel, read device state, observe behavior.
- **udev** — how the GPU and input devices get names, permissions, and symlinks.
- **DRM/KMS** — the kernel graphics interface in detail (Part 5).
- **libinput** — the input library in detail (Part 6).
- **systemd** — the service manager that starts the display manager, the compositor, and every desktop-environment background service.
- **D-Bus** — the bus that compositors, portals, and desktop services use to coordinate.
- **logind** — grants device ACLs to the active session, without which no compositor could open the GPU or input.
- **polkit** — mediates privileged operations the desktop invokes on your behalf.
- **NetworkManager** — talks to GNOME/KDE network applets via D-Bus; status flows up into the top bar.
- **Wayland protocol** — the specific app-compositor protocol in detail (Part 14).
- **Capstone** — ties the whole boot-to-desktop sequence together (Part 15).

Every layer here depends on the systemd/D-Bus/logind triangle you've already seen. The graphical stack is enormous but not standalone; it's a consumer of the rest of the system, not a separate world.

## Where to go next

If this overview raised specific questions:

- **"How do pixels actually get to the screen?"** → the DRM/KMS guide.
- **"How does my touchpad gesture get from hardware to app?"** → the libinput guide.
- **"How does my app talk to the compositor?"** → the Wayland protocol guide.
- **"How does this all come up from a cold boot?"** → the capstone.

And some things this series *doesn't* cover but which sit in this territory:

- **Specific toolkit development** (writing GTK / Qt apps). Those have their own documentation trees.
- **GPU programming** (Vulkan, OpenGL, CUDA). Broad topic, largely orthogonal to desktop Linux.
- **Color management and HDR internals**. Current, evolving, would be its own piece.
- **Accessibility architecture.** AT-SPI, screen readers, the accessibility API surface. Important, but its own topic.
- **Input methods in depth.** IBus and Fcitx — the CJK composition ecosystem. Its own topic.

## Quick reference card

```bash
# What session am I in?
echo $XDG_SESSION_TYPE              # wayland or x11
echo $WAYLAND_DISPLAY               # wayland-0 etc
echo $DISPLAY                       # :0 or XWayland display

# Which GPU driver?
lspci -k | grep -A 3 -i vga
ls /dev/dri/

# What compositor?
ps -o comm= -p $(loginctl show-session $XDG_SESSION_ID -p Leader --value)
# or: ps auxf | grep -E 'mutter|kwin|sway|hyprland'

# X11 client list (including XWayland clients)
xlsclients

# What toolkits are my running apps using?
# GTK apps: loaded `libgtk-*.so`; Qt apps: `libQt*.so`
for pid in $(pgrep -f firefox); do
    grep -h 'gtk\|Qt' /proc/$pid/maps 2>/dev/null | awk '{print $NF}' | sort -u
done

# Portal status
systemctl --user status xdg-desktop-portal*

# Live protocol trace for one app
WAYLAND_DEBUG=1 gnome-terminal 2>&1 | head -50

# GPU activity
amdgpu_top                          # AMD
intel_gpu_top                       # Intel
nvtop                               # cross-vendor
sudo radeontop                      # AMD alternative

# Input activity
sudo libinput debug-events
sudo evtest                         # raw evdev

# Display configuration
drm_info                            # comprehensive KMS dump
wlr-randr                           # wlroots output management
gnome-randr query                   # GNOME-specific
kscreen-doctor -o                   # KDE-specific
```

## Where to learn more

- **The deep-dive guides in this series** — DRM/KMS (Part 5), libinput (Part 6), Wayland protocol (Part 14). Together they cover the technical substrate of what this piece overviews.
- **The Wayland Book** — `https://wayland-book.com`. The single best external resource on Wayland.
- **The X.Org documentation** — `https://www.x.org/wiki/`. Mostly historical, but useful for understanding legacy quirks.
- **freedesktop.org project pages** — Mesa, wayland-protocols, libinput, xdg-desktop-portal all have decent docs linked from `https://www.freedesktop.org/`.
- **Per-compositor docs** — `mutter`, `KWin`, `sway`, `hyprland` each have their own documentation and quirks; the README in each project repo is the starting point.

The mental model to carry: **the Linux graphical stack is six concerns handled by software at seven layers, from kernel driver through userspace libraries, compositor, protocol, portal, toolkit, and desktop environment.** Each layer has a specific job and a specific interface; understanding *which* layer owns *which* problem lets you find the right tool for anything that goes wrong — and lets the deep-dive guides on DRM/KMS, libinput, and Wayland fit into a coherent whole rather than feeling like three unrelated pieces.