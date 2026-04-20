---
date: '2026-04-19T20:59:18-07:00'
draft: true
title: 'Wayland'
---
## What Wayland is and why it exists

For thirty years, Linux GUIs ran on X11 — specifically, on the X.Org server, a giant process that owned the display hardware, received rendering commands from clients over a socket, and drew pixels. Apps were thin clients; the server did everything.

This made sense in 1984. Your "client" might be a remote mainframe; the "server" was the graphics terminal in front of you. Network-transparent, thin clients, shared display resources — all virtues for the era.

By the 2000s, every assumption had flipped:

- **Clients render their own pixels on GPUs.** The X server became a middleman shuffling buffers around.
- **Compositing window managers** (Compiz et al.) bolted onto X as extensions, doing what the server should've been doing natively. The protocol stack grew Composite, Damage, XRender, XFixes, DRI2, DRI3, Present — each a hack layered on the 1984 core.
- **Security model was dead.** Any X client could read any window's contents, inject keystrokes into any other window, read the keyboard globally, screenshot the whole display. Acceptable when only your own processes connected; unacceptable in a world of sandboxed apps.
- **The X server's code** — particularly the input handling and the "no-one really owns rendering" mess — became nearly impossible to evolve.

**Wayland** (started 2008, Kristian Høgsberg) proposed a redesign from the other direction. The observations driving it:

1. Modern apps already render their own buffers using OpenGL/Vulkan. The display server doesn't need to draw anything _for_ them; it just needs to composite their completed buffers.
2. The "compositor" and the "display server" are the same thing. Stop pretending they're separate.
3. Design the protocol to pass GPU buffer handles (file descriptors), not drawing commands.
4. Make the security model opt-in: a client gets no access to other clients' buffers, no global input, no screen capture, unless an explicit protocol extension (and usually user consent) grants it.
5. Keep the core protocol minimal; make functionality modular via versioned extensions.

The result is a protocol that's conceptually much smaller than X11, architecturally simpler, and secure by default — at the cost of network transparency, which Wayland doesn't provide (use waypipe, VNC, or RDP if you need remote display).

## The architectural shift in one picture

```
  X11 model:
  ┌──────────┐   draw    ┌──────────┐  pixels   ┌──────────┐
  │   App    │ ────────▶ │ X server │ ────────▶ │  screen  │
  └──────────┘           └──────────┘           └──────────┘
                              ▲
                              │
                         ┌─────────────┐
                         │ WM + Comp   │
                         │ (separate!) │
                         └─────────────┘

  Wayland model:
  ┌──────────┐                                   ┌──────────┐
  │   App    │ renders → buffer (GPU fd)         │  screen  │
  │ (client) │ ────────▶ commit ──┐              └──────────┘
  └──────────┘                    │                    ▲
                                  ▼                    │
                            ┌───────────────────────────┐
                            │   Compositor              │
                            │   (is the Wayland server) │
                            └───────────────────────────┘
```

In X11, three separate actors (app, server, WM/compositor) cooperate. In Wayland, there are two: the app and the compositor. The compositor owns the display, reads input, and composites client buffers. The "Wayland server" isn't a separate process; it's a library (`libwayland-server`) linked into the compositor.

This collapses a huge amount of complexity. No more race conditions between WM and server. No more "is the compositor alive?" checks. No more Composite-extension plumbing. The compositor is the authority, period.

## The wire: a local-only, object-oriented IPC protocol

The Wayland wire protocol is:

- **Unix-socket only.** No TCP. Socket lives at `$XDG_RUNTIME_DIR/wayland-0` (or `wayland-1`, etc.). Typically `/run/user/1000/wayland-0`.
- **Bidirectional, async, message-framed.** Client and compositor both send messages. Messages are small and framed with headers.
- **Object-oriented.** Every message targets an **object ID** and invokes a **method** on that object's interface.
- **Supports fd passing.** Messages can carry file descriptors alongside data — essential for handing GPU buffers, keymaps, clipboard contents without copying.
- **Strictly ordered per-object.** Requests arrive in the order sent; events arrive in the order generated. Cross-object ordering is not guaranteed.

Think of it as **local, typed RPC over a Unix socket with fd passing**. If that sounds like D-Bus, you're not wrong — the philosophical overlap is real. The differences:

- Wayland is lower-level, single-purpose (just display), and far faster (no broker in the middle).
- Wayland has no service discovery or naming; there's one socket, one compositor.
- Wayland's type system is simpler — no variants, no recursive types.
- Wayland's object model is more tightly coupled to the protocol itself: new objects are created by invoking methods, lifetime is rigidly managed.

Wayland and D-Bus coexist in a session. The compositor speaks Wayland to apps for display, and D-Bus to system services (via session bus — e.g., to show notifications, to talk to xdg-desktop-portal for screen capture permissions).

### Message format, roughly

Each message on the wire is:

```
 4 bytes: object ID (sender: client gives IDs to its objects; compositor gives IDs to its)
 2 bytes: message size (including header)
 2 bytes: opcode (which method on this object's interface)
 N bytes: arguments, typed
```

Argument types are a small set:

- `int`, `uint`, `fixed` (24.8 fixed-point)
- `string` (length-prefixed UTF-8)
- `object` (object ID, referring to another object)
- `new_id` (an ID the sender is allocating for a newly-created object)
- `array` (length-prefixed bytes)
- `fd` (passed out-of-band via SCM_RIGHTS; the message just notes "an fd is here")

The "new_id" type is how you create objects: you call a factory method on an existing object, passing an ID you've chosen, and now a new object exists at that ID.

## Interfaces: the vocabulary

The protocol is defined as a set of **interfaces**, each with a name, a version, and a set of requests (client→compositor messages) and events (compositor→client messages). An **object** is an instance of an interface, identified by an ID local to the connection.

Each interface is defined in XML; `wayland-scanner` generates C code from the XML at build time. You almost never write this XML; you consume it.

The core protocol defines about a dozen interfaces. The ones that matter most:

|Interface|Role|
|---|---|
|`wl_display`|Object 1, always. The root of everything. Get the registry, sync, handle errors.|
|`wl_registry`|Enumerates globals (interfaces the compositor exposes). Clients bind to what they need.|
|`wl_compositor`|Factory for `wl_surface` and `wl_region`.|
|`wl_surface`|A rectangular region that can display a buffer. The fundamental drawing target.|
|`wl_buffer`|A pixel buffer, usually backed by a GPU dma-buf fd.|
|`wl_shm`, `wl_shm_pool`|Shared-memory buffer allocation (CPU rendering path).|
|`wl_seat`|A collection of input devices (keyboard, pointer, touch) associated with a user.|
|`wl_keyboard`, `wl_pointer`, `wl_touch`|Input devices.|
|`wl_output`|A monitor. Position, size, refresh, scale factor.|
|`wl_callback`|A one-shot promise-like object (e.g., "tell me when the frame is presented").|
|`wl_data_device`, `wl_data_source`, `wl_data_offer`|Clipboard and drag-and-drop.|

That's most of the core vocabulary. A striking thing: **there's no "window" interface in core Wayland**. A `wl_surface` is just a rectangle; what makes it a window is an extension.

### The registry pattern

Every client starts the same way: connect to the socket, which hands you a `wl_display` object at ID 1. Call `wl_display.get_registry` to obtain a `wl_registry`. The registry fires a `global` event for every interface the compositor supports, with a name, version, and an opaque "name" integer. The client inspects what's available and calls `wl_registry.bind` to get instances of the globals it needs.

```
client                                          compositor
  │                                                   │
  │  connect(/run/user/1000/wayland-0)                │
  │ ─────────────────────────────────────────────▶    │
  │                                                   │
  │  wl_display.get_registry(new_id=2)                │
  │ ─────────────────────────────────────────────▶    │
  │                                                   │
  │  wl_registry.global(name=1, iface="wl_compositor", version=6)
  │ ◀─────────────────────────────────────────────    │
  │  wl_registry.global(name=2, iface="wl_shm", version=2)
  │ ◀─────────────────────────────────────────────    │
  │  wl_registry.global(name=3, iface="xdg_wm_base", version=5)
  │ ◀─────────────────────────────────────────────    │
  │  ... many more ...                                │
  │                                                   │
  │  wl_registry.bind(name=1, iface="wl_compositor",  │
  │                    version=6, new_id=3)           │
  │ ─────────────────────────────────────────────▶    │
  │                                                   │
  │  (now object 3 is a wl_compositor, usable)        │
```

The `wl_registry` is both a directory and a negotiation. The compositor advertises an interface version; the client binds to a version it understands (capped at both sides' maximum). Globals can appear and disappear at runtime (e.g., a monitor is plugged in → new `wl_output` global → `global` event fires). Clients handle `global_remove` for cleanup.

## Core protocol walkthrough: drawing a window

Here's the full choreography for the simplest app: "show a solid-colored window until the user closes it."

**1. Connect and bind.** As above: connect, get registry, bind `wl_compositor`, `wl_shm` (or a GPU buffer interface), `xdg_wm_base` (the modern windowing extension), `wl_seat`.

**2. Create a surface.**

```
client:  wl_compositor.create_surface(new_id=10)
```

Object 10 is now a `wl_surface`. Just a rectangle. No position, no size yet, not visible.

**3. Give it a role — make it a window.** This is where `xdg-shell` comes in (more below). You call `xdg_wm_base.get_xdg_surface(surface=10, new_id=11)` to get an `xdg_surface`, then `xdg_surface.get_toplevel(new_id=12)` to declare "this is a top-level window."

```
client:  xdg_wm_base.get_xdg_surface(new_id=11, surface=10)
client:  xdg_surface.get_toplevel(new_id=12)
client:  xdg_toplevel.set_title("Hello")
client:  xdg_toplevel.set_app_id("com.example.hello")
client:  wl_surface.commit()     ← "I've set up my role; tell me my size"
```

The first `commit` with no buffer says to the compositor: "I'm ready; please configure me."

**4. Compositor sends a configure event.**

```
compositor:  xdg_toplevel.configure(width=800, height=600, states=[activated])
compositor:  xdg_surface.configure(serial=42)
```

The compositor decides the size — might be whatever the client suggested, might be different (tiling WMs often force sizes). The client must ack this serial before its next commit, which is its contract: "I see the configure; I'll render to these dimensions."

**5. Render a frame.**

Client allocates a buffer (shm or dma-buf), renders pixels into it.

For shm:

```
client:  create an fd backing a shared memory region (memfd_create)
client:  mmap it, draw pixels
client:  wl_shm.create_pool(new_id=13, fd=<fd>, size=N)     ← fd passed here!
client:  wl_shm_pool.create_buffer(new_id=14, offset=0, width=800, height=600,
                                     stride=3200, format=ARGB8888)
```

Object 14 is a `wl_buffer`.

**6. Attach buffer, damage, commit.**

```
client:  wl_surface.attach(buffer=14, x=0, y=0)
client:  wl_surface.damage_buffer(x=0, y=0, width=800, height=600)   ← which pixels changed
client:  xdg_surface.ack_configure(serial=42)
client:  wl_surface.commit()
```

The **commit** is the atomic unit of Wayland frame submission. Between commits, the client stages changes (attach, damage, region setting, subsurface updates). The commit makes them visible together. No half-rendered frames, no tearing.

**7. Compositor composes and releases.**

Compositor adds this surface's buffer to its scene, composes with everything else, submits to KMS. When the GPU's done reading the buffer for display, compositor sends:

```
compositor:  wl_buffer.release(buffer=14)
```

Client can now recycle buffer 14 for the next frame.

**8. The frame callback dance.**

Clients don't want to render frames blindly. The idiomatic pattern:

```
client:  wl_surface.frame(new_id=15)    ← "tell me when this frame is shown"
client:  wl_surface.commit()

... later, after the frame is presented ...

compositor:  wl_callback.done(callback=15, time=12345)   ← "frame shown at time X"
```

The client renders its next frame only after receiving `done`. This naturally paces rendering to the display's refresh rate — no vsync code, no polling, just "await the callback." Lovely.

**9. Input.**

If the pointer enters the surface:

```
compositor:  wl_pointer.enter(serial=..., surface=10, x=100.5, y=200.2)
compositor:  wl_pointer.motion(time=..., x=101.0, y=200.5)
compositor:  wl_pointer.button(serial=..., time=..., button=BTN_LEFT, state=pressed)
compositor:  wl_pointer.frame()     ← "that was one atomic input event"
```

Keyboard focus similarly: `wl_keyboard.enter(serial, surface, keys_currently_pressed)` tells the client it has focus, followed by `wl_keyboard.key` events.

Touch events come via `wl_touch`, with `down`, `motion`, `up`, `frame` groupings.

Critical: **input events are only delivered to surfaces the compositor decides should receive them.** No global input, no snooping. A client that's not focused sees no keyboard events.

**10. Close.**

User clicks X in the decoration, or presses Alt+F4:

```
compositor:  xdg_toplevel.close()
```

Just a polite request. The client decides whether to actually close (save prompts etc.). To really close, client destroys its objects and disconnects.

That's the entire core loop. Every Wayland app, from a Hello World to Firefox, is doing variations on this.

## Extensions: where everything actually lives

Core Wayland is deliberately tiny. Almost everything interesting — window management, decorations, clipboard, screen capture, virtual keyboards, layer shell for panels, xdg foreign references, fractional scaling, color management, tearing control — lives in **protocol extensions**.

Extensions are grouped into a few families:

**`xdg-shell`** — the desktop windowing protocol. Defines `xdg_wm_base`, `xdg_surface`, `xdg_toplevel`, `xdg_popup`. Standard across all desktop compositors. This is what gives a surface its identity as a window, with a title, app ID, states (maximized, fullscreen, activated), move/resize interaction, minimize/maximize requests. Stable and universal.

**`xdg-decoration`** — negotiates whether the client or the server draws the window title bar. GNOME's philosophy is client-side decorations (apps draw their own title bars, see GTK's headerbars); KDE prefers server-side. This extension lets them agree per-window.

**`wlr-*`** (wlroots protocols) — a family developed alongside the **wlroots** compositor library (sway, hyprland, many others). Includes:

- `wlr-layer-shell` — for panels, bars, wallpapers, lockscreens. Surfaces anchored to edges, above or below normal windows.
- `wlr-screencopy`, `wlr-export-dmabuf` — screen capture.
- `wlr-output-management` — monitor configuration.
- `wlr-foreign-toplevel-management` — task bars and window switchers.

These aren't standardized across all compositors. GNOME doesn't implement wlr-layer-shell; it has its own equivalents. KDE does implement many wlr protocols. This fragmentation is real — a bar written against wlr-layer-shell won't run on GNOME.

**`wp-*`** (wayland-protocols staging and stable) — protocols being standardized across compositors. Includes:

- `wp-viewporter` — scaling and cropping for video playback.
- `wp-linux-dmabuf` — the modern GPU buffer sharing path (replaces legacy shm for GPU-accelerated apps).
- `wp-fractional-scale` — HiDPI at non-integer ratios.
- `wp-presentation-time` — precise timing info for media/games.
- `wp-tearing-control` — opt-in tearing for games that want lower latency than vsync allows.
- `wp-color-management` — HDR and color-managed rendering. Still evolving.

**`ext-*`** — newer stable cross-compositor protocols, often the "real" version of a wlr experiment. `ext-idle-notify`, `ext-session-lock`, etc.

**Compositor-specific** — KDE's `kde-*` protocols, GNOME's internal ones, Hyprland's `hyprland-*`. Generally avoided by portable apps.

**Security-sensitive operations go through xdg-desktop-portal**, not Wayland extensions. Screenshots, screen recording, remote desktop, file choosers, global hotkeys — these cross the app-compositor trust boundary, and the portal mediates them (showing a consent dialog, enforcing policy). The portal implementation uses compositor-specific Wayland extensions internally, but the app only sees the D-Bus portal API.

This matters for understanding the security model: **Wayland protocol extensions are generally for functionality that's safe to expose to any client.** Things requiring user consent live in portals. The division is the core of Wayland's security story.

## The protocol definition: XML, and why you'll occasionally read it

Protocol interfaces are defined in XML files. Look at `/usr/share/wayland/wayland.xml` on your system for the core protocol; `/usr/share/wayland-protocols/` has the xdg and wp extensions.

A snippet — the `wl_surface.attach` request:

```xml
<request name="attach">
  <description summary="set the surface contents">
    Set a buffer as the content of this surface.
    ...
  </description>
  <arg name="buffer" type="object" interface="wl_buffer" allow-null="true"/>
  <arg name="x" type="int"/>
  <arg name="y" type="int"/>
</request>
```

Reading these is honestly the fastest way to understand what a protocol actually does. Summaries are thorough; edge cases are documented; versioning is explicit (`since="3"` annotations on new arguments). When debugging, when wondering "can I do X?", when writing a client, go read the XML.

## `WAYLAND_DEBUG=1`: the strace of Wayland

Every client (if built with `libwayland`, which is almost all of them) honors the environment variable `WAYLAND_DEBUG=1`. Set it and run any Wayland app:

```bash
WAYLAND_DEBUG=1 foot 2>&1 | head -60
```

You'll see every protocol message in both directions:

```
[1234567.890]  -> wl_display@1.get_registry(new id wl_registry@2)
[1234567.891] wl_display@1.delete_id(2)            (ignore, this is sync)
[1234567.892] wl_registry@2.global(1, "wl_compositor", 6)
[1234567.893] wl_registry@2.global(2, "wl_shm", 2)
[1234567.894] wl_registry@2.global(3, "xdg_wm_base", 5)
...
[1234567.901]  -> wl_registry@2.bind(1, "wl_compositor", 6, new id [unknown]@4)
[1234567.902]  -> wl_compositor@4.create_surface(new id wl_surface@5)
...
```

`->` means client sending; no prefix means compositor → client. You see the full handshake, binding, surface creation, buffer attach, commit, input events, everything.

This is pedagogically invaluable. Pick an app, run with `WAYLAND_DEBUG=1`, watch the handshake. You'll understand in five minutes what documentation takes an hour to explain.

## The compositor side

From the compositor's perspective, being a Wayland server means:

1. **Listen on the Unix socket.**
2. **Accept connections; track per-client object tables.** Object IDs are per-connection; IDs allocated by the client are in one range, by the server another.
3. **Dispatch requests to handlers.** Usually done by `libwayland-server`, which parses XML-generated code to know the wire format.
4. **Composite client surfaces into a scene.** This is the work: layout, damage tracking, GPU compositing.
5. **Manage output devices.** Enumerate monitors, configure modes via KMS, handle hotplug.
6. **Handle input.** Read evdev via libinput, decide which surface gets focus, dispatch input events.
7. **Talk to KMS.** Submit the composited frame via atomic commits for tear-free, vsync-paced presentation.

Nobody writes a compositor from scratch anymore. You use a library:

**`libwayland-server`** — the base protocol library. Raw; you implement everything above.

**`wlroots`** — a library of reusable compositor primitives: scene graph, output management, input routing, atomic KMS commits, XDG shell implementation. sway, hyprland, Way Cooler, and many others are wlroots-based. If you want to write a compositor in 2026, this is where you start. Written in C.

**`smithay`** — the Rust equivalent. COSMIC (System76's desktop) uses it.

**`mutter`** — GNOME's compositor-as-library, but tightly coupled to GNOME Shell. Not really reusable outside GNOME.

**`KWin`** — KDE's; same story, KDE-coupled.

Writing a tiny wlroots-based compositor is roughly 1000 lines of C for something functional. The wlroots tutorial compositor (tinywl) is 500 lines and handles toplevel windows, input, multiple outputs. This is the right scale for "Wayland is simpler than X11" to be concretely true.

## Client libraries

App developers almost never use `libwayland-client` directly. They use a toolkit:

- **GTK, Qt** — handle Wayland entirely. You write `QWindow` or `GtkWindow`; the toolkit does Wayland.
- **SDL2/SDL3** — for games; has a Wayland backend.
- **glfw** — similar, for OpenGL/Vulkan apps.
- **winit** (Rust) — cross-platform windowing, Wayland backend.

Direct `libwayland-client` usage is for:

- Toolkit authors.
- Wayland utilities (layer-shell bars, lockscreens, screenshot tools).
- Embedded/kiosk apps where you want zero extra dependencies.

The XML is scanned by `wayland-scanner` to generate C proxies — one function per request, one callback slot per event, type-safe marshaling.

## Subsurfaces and the compositing model

One architectural piece worth understanding: **subsurfaces**. A surface can have child surfaces positioned relative to it. Commits propagate atomically up the tree — you can update a video surface's frame and a UI overlay's text in sync.

Why this matters: video players, web browsers, games with HUDs. The video decoder can push YUV frames directly to a subsurface; the UI toolkit renders an overlay on the parent surface; the compositor combines them in hardware if it can (overlay planes). This is why Wayland video playback can be zero-copy all the way from decoder to display, while X11 playback typically went through an extra copy.

## The lifecycle and the protocol's ceremoniousness

Wayland is very precise about object lifetimes. You don't just stop using an object; you explicitly destroy it, and the protocol specifies the order of destruction when objects nest.

This feels ceremonious at first — "I have to destroy the `xdg_toplevel` before the `xdg_surface` before the `wl_surface`?" Yes. But the reward is that the protocol is unambiguous about state. The compositor always knows what exists. No zombie windows, no "surface gone but still referenced" bugs. It's a protocol for a world where every message matters.

The same care applies to **versioning**. Every interface has a version. When a client binds, it specifies which version it wants, up to the compositor's maximum. New requests and events are tagged with `since="N"` in the XML. Clients check the bound version before calling newer methods. There's no ambiguity about what an object can do; you know exactly at bind time.

## Errors

Wayland is synchronous-looking but actually asynchronous. Requests don't return; they either succeed silently or trigger an error event. The compositor sends a `wl_display.error` event with the offending object, error code, and message, and then **disconnects**. Any protocol error is fatal.

This is brutal, deliberately. Protocol errors are bugs; crashing is the right response. "Silently try to recover from malformed requests" is how X11 got into its mess. Wayland's model: get it right, or you don't get to play.

For clients, this means: test with `WAYLAND_DEBUG=1`, handle all events, don't reference destroyed objects, and respect version limits.

## Security model in practice

Concretely, what a Wayland client **cannot** do:

- Read another client's buffers.
- Read global keyboard state or inject synthetic events.
- Screenshot the display.
- Record the screen.
- Move/resize windows it doesn't own.
- Register global hotkeys outside its focused surfaces.
- See the positions of other windows.
- Capture the pointer when unfocused.

What it **can** do (unchanged):

- Render into its own surfaces.
- Receive input for surfaces it owns, when focused.
- Request clipboard contents (compositor-mediated; only on focus and with copy-event context).
- Use standard protocol features: titles, icons, minimize requests.

For the "can't" list, there are extensions that grant specific capabilities _in specific compositors_ (e.g., `wlr-virtual-keyboard` for on-screen keyboards in wlroots compositors). And for desktop-level operations (screen capture, global shortcuts), there's xdg-desktop-portal with user consent.

The practical effect: Linux desktop is suddenly _harder_ for utilities that took X11 freedoms for granted. AutoKey, xdotool, clipboard managers — all had to adapt, use portals, or rely on compositor-specific extensions. Most have. Some haven't; those apps work only under XWayland.

## XWayland: the compatibility layer

XWayland is an X11 server running as a Wayland client. Old X11 apps connect to it as if it were X.Org; it translates to Wayland for the real compositor.

Rooted mode (default): XWayland draws each X11 window as a Wayland surface, with the compositor treating each as a normal window. It's invisible; X apps just work.

Limitations:

- **Fractional scaling is problematic.** X11 apps don't know about scale factors, so HiDPI support is either "render at 1x, get blurry" or "render at 2x, get too big."
- **Global shortcuts still work within XWayland** (among X apps), so legacy tools that inject keys into other X apps still function — but only affecting other X apps, not Wayland natives.
- **Clipboard integration needs a bridge** (usually the compositor does this). Works, sometimes flakily.
- **Performance for DRI3/Present is good;** XWayland does the buffer-passing zero-copy path in most cases.

You can tell which of an app's windows are XWayland: `xlsclients` lists X11 clients; anything there is running through XWayland.

## Input methods, accessibility, and the hard parts

Two things Wayland has struggled with longer than the rest:

**Input methods.** Chinese/Japanese/Korean input requires an IME (IBus, Fcitx) composing text before it's delivered to the app. X11 had XIM (rough but functional); Wayland has `zwp_input_method_v2` (experimental, compositor-specific for a long time). It's mostly working now in 2026; the long road here is a reminder that redesigns take time to cover all the edges.

**Accessibility.** AT-SPI (the accessibility bus) runs over D-Bus, independent of X11/Wayland, so screen readers mostly work. But things like "screen reader draws a focus rectangle on the focused element" require cooperation that's compositor-dependent.

**Global hotkeys.** An app wanting "Ctrl+Alt+X always opens my thing, even when I'm not focused" — impossible in pure Wayland by design. The xdg-desktop-portal `GlobalShortcuts` portal (recent) is the sanctioned path, with user consent.

These aren't protocol flaws exactly; they're the cost of the stricter model. X11 let any client do anything; Wayland requires explicit pathways for everything, and building those pathways has taken years.

## Debugging flow for GUI issues

When something's off:

1. **`echo $XDG_SESSION_TYPE`** — am I actually on Wayland?
2. **`echo $WAYLAND_DISPLAY`** — which socket?
3. **`xlsclients`** — is this app XWayland?
4. **`WAYLAND_DEBUG=1 <app> 2>wayland.log`** — protocol trace.
5. **`journalctl --user -b`** — compositor and session logs.
6. **`busctl --user list | grep portal`** — portals alive?
7. **`wayland-info`** — which globals does the compositor advertise? What versions?

For compositor development: wlroots has built-in debug logging; mutter has `MUTTER_DEBUG=1` and friends; KWin has `QT_LOGGING_RULES` + `KWIN_*` vars.

## Things that trip people up

**Committing without attaching a buffer.** The initial commit to get a configure is correct and intentional. Subsequent commits need a buffer or they're no-ops.

**Forgetting to ack configure.** After an `xdg_surface.configure`, the client must `ack_configure` with the serial before the next commit changes rendered content. Violate this and the compositor may treat your commit as stale.

**Releasing buffers too eagerly.** After `wl_surface.attach`, the compositor owns the buffer until it sends `release`. If the client reuses the buffer before release, you get visual glitches or data races.

**Version mismatches.** Client calls a method that was added in version 3, but bound at version 2. Protocol error, disconnect. Always check your bound version against `since=` annotations.

**Assuming synchronicity.** Requests don't return values. A "create surface" request returns nothing; you give it a new_id, and the object exists from your perspective immediately. The server processes in order, and errors come back async.

**Scaling confusion.** HiDPI on Wayland is well-defined: the compositor tells the surface its scale factor via `wl_surface.preferred_buffer_scale` (or legacy output-based calculation). The client renders at that scale into the buffer and declares `wl_surface.set_buffer_scale`. Mishandling this is the source of most "why is my app blurry on Wayland" bugs.

## The honest state in 2026

Wayland has won for new development. GNOME, KDE, Sway, Hyprland are the dominant desktops and all default to Wayland. New compositors aren't targeting X11. Toolkits (GTK4, Qt6) assume Wayland.

That said, some rough edges remain:

- **NVIDIA proprietary drivers** are the eternal pain point; they gained real Wayland support only in 2023-2024 with the explicit sync protocol and are still rougher than AMD/Intel.
- **Color management / HDR** is stabilizing but not fully uniform across compositors.
- **Remote desktop** works via PipeWire + portals, but the UX varies by compositor.
- **Some legacy apps** still need XWayland and may never be ported (old Java Swing, proprietary CAD, etc.).

For most users most of the time, Wayland is transparent. You're probably on it. Your terminal is a Wayland client. Your browser might be native Wayland or XWayland depending on the build. Your notifications come from a Wayland-aware daemon. It all mostly works.

## Quick reference card

```bash
# Environment
echo $XDG_SESSION_TYPE           # wayland | x11
echo $WAYLAND_DISPLAY            # socket name, e.g., wayland-0
echo $XDG_RUNTIME_DIR            # typically /run/user/$UID, where the socket lives
echo $DISPLAY                    # X or XWayland display

# Capabilities
wayland-info                     # globals exposed by compositor, with versions
xlsclients                       # X11 clients (all via XWayland if in Wayland session)
loginctl show-session $XDG_SESSION_ID -p Type

# Debug
WAYLAND_DEBUG=1 <app>            # protocol trace for this client
WAYLAND_DEBUG=client <app>       # same, only client→server
WAYLAND_DEBUG=server <app>       # only server→client
MUTTER_DEBUG=1                   # GNOME/mutter specifics
QT_LOGGING_RULES="qt.qpa.wayland=true"  # Qt's Wayland backend

# XML definitions
ls /usr/share/wayland/            # core protocol
ls /usr/share/wayland-protocols/  # xdg-shell, wp-*, staging/, stable/, unstable/
less /usr/share/wayland/wayland.xml

# Who's the compositor?
ps -o cmd= -p $(loginctl show-session $XDG_SESSION_ID -p Leader --value)
```

## Where to learn more

- **[The Wayland Book](https://wayland-book.com)** by Drew DeVault — by far the best single resource. Walks you through writing a client and a compositor, protocol by protocol. Read top to bottom; it clicks into place.
- **`/usr/share/wayland/wayland.xml`** and **`/usr/share/wayland-protocols/`** — the XML is readable and thorough.
- **wlroots source** (`https://gitlab.freedesktop.org/wlroots/wlroots`) — reading a real compositor library makes abstractions concrete. Start with `tinywl`.
- **`WAYLAND_DEBUG=1`** on apps you use every day. The fastest path to intuition.
- **`libwayland` docs** — the man pages aren't deep but are accurate.
- **freedesktop.org wayland-protocols repo** — the canonical home for extensions, with design discussion in merge requests.

The mental model to carry: **Wayland is a local, object-oriented, fd-passing RPC protocol between apps and a compositor, with a minimal core and versioned extensions for specific capabilities, designed so that the compositor owns all display and input state and clients are isolated by default.** The protocol's "feel" — ceremonious, strict, explicit — is the price of a system where what you see on screen is exactly what the protocol says should be there, with no hidden state, no races, and no back doors. Once that model clicks, Wayland becomes legible and the entire Linux GUI stack gets simpler, not harder, than it was with X11.