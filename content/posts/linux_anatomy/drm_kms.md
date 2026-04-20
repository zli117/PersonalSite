---
date: '2026-04-19T20:55:51-07:00'
draft: true
title: 'DRM/KMS'
---
## What DRM/KMS is and why it exists

The journey of "userspace program wants pixels on the screen" on Linux has been a long architectural drama. In the 1990s, X.Org (then XFree86) was a root-running monolith that poked graphics cards directly via port I/O and memory-mapped registers. Every GPU driver was an X module; every mode change (resolution, refresh rate) happened in userspace; crashing the X server usually meant a blank screen until reboot. "Kernel involvement in graphics" was limited to getting the PCI device mapped and out of the way.

This broke as soon as more than one thing wanted the GPU:

- **Multiple users switching VTs** — the X server owned the hardware; Ctrl+Alt+F1 had to tear the whole thing down.
- **Multiple processes rendering simultaneously** — DRI1 (early direct rendering) let clients render in parallel, but arbitration was ad-hoc and unsafe.
- **Compositing** — required shared GPU state between the compositor and clients.
- **Suspend/resume** — userspace drivers couldn't reliably re-init hardware.
- **Security** — userspace with raw MMIO access is a disaster from every angle.

**DRM** (Direct Rendering Manager) is the kernel subsystem that took over. Started in the early 2000s, matured through DRI2 and DRI3 handoffs with X.Org. Its job: own the GPU, safely multiplex it between clients, manage GPU memory, handle command submission, coordinate with the display hardware. Device nodes live at `/dev/dri/card*` (privileged, includes mode setting) and `/dev/dri/renderD128+` (unprivileged rendering only).

**KMS** (Kernel Mode Setting) is the subset of DRM that handles _displays_ specifically: enumerating connectors, reading EDID, picking resolutions and refresh rates, configuring CRTCs (the display controllers), programming the hardware to scan out specific buffers. Before KMS (~2008), mode setting was in userspace X drivers — hence the era of "X crashed, monitor went black, reboot." Now it's kernel-side and survives compositor crashes.

Together, **DRM/KMS is the kernel's graphics subsystem**. Everything above it — Mesa, Wayland compositors, X.Org, games, video players — speaks DRM ioctls (directly or through libdrm) to get pixels to the screen and to make the GPU compute.

## The two halves: render vs. display

A modern GPU really does two distinct jobs:

1. **Render.** Take vertex buffers, textures, shaders; produce pixel data in memory. This is the "compute" part — fundamentally about moving data through shader cores, allocating memory, submitting command buffers, waiting for completion.
    
2. **Display.** Take pixel data sitting in memory; pump it out a display connector at the right timing and format. This is scanout — reading buffers at the exact pixel clock, composing hardware planes, driving HDMI/DisplayPort.
    

These are often physically separate silicon (separate engines in a single GPU, or entirely separate ICs on an SoC). The Linux kernel reflects this split:

```
            userspace (Mesa, compositor, game)
                      │
                      ▼
                  libdrm (thin ioctl wrapper)
                      │
                      ▼
          ┌───────────────────────────┐
          │   DRM core subsystem      │
          │                           │
          │   ┌──────┐    ┌──────┐    │
          │   │ GEM  │    │  KMS │    │
          │   │(mem) │    │(mode)│    │
          │   └──────┘    └──────┘    │
          │                           │
          │   ┌───────────────────┐   │
          │   │ driver-specific   │   │
          │   │ amdgpu / i915 /   │   │
          │   │ xe / nouveau /    │   │
          │   │ msm / etnaviv /   │   │
          │   │ v3d / nvidia (*)  │   │
          │   └───────────────────┘   │
          └───────────────────────────┘
                      │
                      ▼
                     GPU
```

NVIDIA's proprietary driver reimplements its own version of DRM/KMS interfaces; the open-source NVIDIA kernel module `nvidia-drm` speaks the same ioctls through a shim. The `nouveau` driver is the traditional open-source NVIDIA driver; `nova` is the emerging Rust-based replacement.

This split is why your system has two device nodes per GPU:

- **`/dev/dri/card0`** — the full interface: mode setting, buffer allocation, everything. Privileged (historically root-only; now seat-granted to your session by logind).
- **`/dev/dri/renderD128`** — render-only: GPU compute and buffer allocation, but no display control. Safer to hand to unprivileged clients, containers, sandboxed apps.

Wayland compositors open `card0` (they need mode setting). GPU-accelerated apps like Firefox or a Vulkan game open `renderD128` (they just need to render).

```bash
ls -l /dev/dri/
# crw-rw----+ 1 root video  226,   0 Apr 18 10:00 card0
# crw-rw-rw- 1 root render  226, 128 Apr 18 10:00 renderD128
# lrwxrwxrwx 1 root root           8 Apr 18 10:00 by-path/pci-0000:03:00.0-card -> ../card0
# lrwxrwxrwx 1 root root          13 Apr 18 10:00 by-path/pci-0000:03:00.0-render -> ../renderD128

getfacl /dev/dri/card0       # see who's been granted access this session
```

Your user has group `video` (for cards) and `render` (for render nodes) typically, plus ACLs from logind.

## GEM: the memory manager

GPU memory management is subtle. A GPU needs buffers — framebuffers, textures, shader constants, geometry. These can live in VRAM (dedicated GPU memory), system RAM (mapped into the GPU's address space), or migrate between. The kernel needs to track them, multiplex them between clients, enforce isolation, and hand file descriptors around when buffers need to be shared (between processes, between GPU and display, between GPU and video decoder).

**GEM** (Graphics Execution Manager) is DRM's object and memory system. Every GPU buffer is a GEM object with a name (an ID), a size, and backing storage (driver-managed). Clients allocate buffers via ioctls, get back a handle, use it, destroy it.

GEM handles are per-file-descriptor — meaningful only within one open of `/dev/dri/card0`. To share a buffer across processes or export it to something else (the display, a video codec, a Wayland compositor), you convert the handle to a **dma-buf fd**:

```
# Export GEM handle → dma-buf fd
handle → DRM_IOCTL_PRIME_HANDLE_TO_FD → fd

# Import dma-buf fd → GEM handle
fd → DRM_IOCTL_PRIME_FD_TO_HANDLE → handle
```

This dma-buf fd is just a file descriptor. You can pass it through a Unix socket with SCM_RIGHTS, send it over Wayland as an `fd` argument, or hand it to a V4L2 video capture device. The underlying memory is shared zero-copy; importers get their own handles pointing at the same storage.

This is _the_ mechanism for everything: Wayland compositors import client buffers as dma-bufs; video playback pipes decoder output straight to the compositor via dma-buf; screen capture tools export compositor output as dma-buf to encoders. Zero-copy end to end, all the way from decoder to display.

The memory itself might be:

- **VRAM** (discrete GPUs) — fast, but limited.
- **GTT/GART** (integrated/discrete) — system memory the GPU can access directly.
- **Shared system memory** (integrated GPUs like AMD APUs, Apple Silicon, Intel iGPUs) — unified memory architecture; CPU and GPU share the same RAM.

The kernel driver decides placement based on usage hints and memory pressure, sometimes migrating buffers between. Well-written userspace code gives hints; kernel handles the details.

## Command submission: how work gets to the GPU

To render something, userspace:

1. Allocates buffers (GEM).
2. Builds a **command buffer** — a sequence of GPU instructions (driver-specific format).
3. Submits it via an ioctl: "execute these commands, using these buffers, signal this sync object when done."
4. Waits for completion (or moves on; work is async).

The submission ioctl is driver-specific in its exact form, but the pattern is universal. `DRM_IOCTL_I915_GEM_EXECBUFFER2`, `DRM_IOCTL_AMDGPU_CS`, `DRM_IOCTL_MSM_GEM_SUBMIT` — same idea, different flavors.

Modern drivers are moving to **user-mode submission**: the ring buffer is mapped to userspace, and userspace writes commands directly, ringing a doorbell to tell the GPU. Much lower overhead than an ioctl per submit. AMD's new user-mode queues, Intel's xe driver (the successor to i915), and Apple Silicon drivers all embrace this.

Userspace never writes raw command buffers by hand. Mesa does it, translating OpenGL or Vulkan calls into hardware-specific command streams. For Vulkan, the translation is thin; for OpenGL it's more involved because GL's state-machine model doesn't map naturally to modern GPUs. Either way, Mesa is the big userspace consumer of DRM.

## Sync objects and fences

GPU work is asynchronous. You submit commands; they execute later; you need to know when they're done. DRM has **sync objects** (drm_syncobj) for this — GPU-aware primitives that can be waited on by CPU or by other GPU work.

Sync objects came in several generations:

- **Implicit sync** (old): the kernel tracked per-buffer fences automatically. Submit work touching buffer X; kernel knows when it's done; next submitter reading X waits. Simple, but causes false dependencies.
    
- **Explicit sync** (new): clients pass sync objects explicitly, creating only the dependencies they need. Much better for modern GPUs with dozens of queues. The Vulkan semantic model.
    
- **Timeline semaphores**: sync objects that increment through a sequence of values, so you can wait on "value ≥ N". Vulkan 1.2+ maps to these directly.
    

The move from implicit to explicit sync is one of the bigger recent DRM changes. Wayland's `linux-drm-syncobj-v1` protocol extension (2024+) exposes explicit sync to Wayland compositors, fixing NVIDIA-specific stuttering and enabling correct tear-free rendering across all GPUs.

## KMS: the display side, in depth

Now the display half. KMS structures the hardware as three layers of objects:

```
                        wall outlet (monitor)
                               ▲
                               │
                          ┌─────────┐
                          │Connector│   ← HDMI-A-1, DP-2, eDP-1, ...
                          └────┬────┘      "which port on the back?"
                               │
                          ┌─────────┐
                          │ Encoder │   ← TMDS, DP, MIPI, LVDS, ...
                          └────┬────┘      "what signal format?"
                               │
                          ┌─────────┐
                          │  CRTC   │   ← controller 0, 1, ...
                          └────┬────┘      "which display engine drives it?"
                               │
                 ┌─────────────┼─────────────┐
                 │             │             │
            ┌─────────┐   ┌─────────┐   ┌─────────┐
            │ Primary │   │ Overlay │   │ Cursor  │
            │  plane  │   │  plane  │   │  plane  │   ← planes (scanout sources)
            └─────────┘   └─────────┘   └─────────┘
                 ▲             ▲             ▲
                 │             │             │
              buffer        buffer        buffer    ← GEM objects / dma-bufs
```

Four kinds of objects, each programmable:

**Connector** — a physical output on the GPU. HDMI, DisplayPort, eDP (embedded DP for laptop panels), VGA (still around), LVDS, DSI (mobile displays), Virtual (for virtual GPUs). The connector reads EDID from attached monitors, reports supported modes, and tracks hotplug.

**Encoder** — translates digital pixel data to the physical signal format (TMDS for HDMI, packets for DisplayPort, etc.). Usually a 1:1 mapping to connectors — not something userspace picks.

**CRTC** (Cathode Ray Tube Controller, name is historical) — the _display controller_. Reads pixel data from memory at the pixel clock, applies timing, and pushes to encoders. A GPU has a fixed number of CRTCs; that's your maximum simultaneous display count. Each CRTC can drive one or more connectors (mirroring).

**Plane** — a source of pixel data for a CRTC. CRTCs composite planes in hardware:

- **Primary plane** — the "main" framebuffer.
- **Overlay planes** — additional layers for video, cursors, HUDs.
- **Cursor plane** — hardware cursor, moves independently of the rest.

The plane model is important for performance. Hardware overlay planes let you display a video frame without involving the GPU compositor at all — video decoder outputs YUV to a buffer, it's assigned to an overlay plane, the display hardware scans it out directly. Zero GPU work for playback.

### Mode setting: the central act

A **mode** is a complete timing description: horizontal/vertical resolution, pixel clock, sync pulses, front/back porches, refresh rate. An EDID from the monitor advertises supported modes; userspace picks one.

Mode setting is: "configure this CRTC to run at this mode, driving this connector, with this plane showing this buffer." One operation, many parameters.

**Before atomic KMS** (pre-2015 era), mode setting was a sequence of ioctls: set CRTC, set framebuffer, set plane, set property, each separately. Multi-step changes (like "change to DP on this monitor and HDMI on that one, both at new resolutions") couldn't be done atomically — the display was in an intermediate broken state between ioctls. Mode flicker, wrong framebuffer briefly visible, user-visible tearing.

**Atomic KMS** (stable ~2015) solved this. The new ioctl `DRM_IOCTL_MODE_ATOMIC` takes a batch of property changes: "set these N properties across these M objects, and commit them all together — either all apply, or none do." Hardware might implement this as "program the next frame's state registers, swap on vblank," so the transition happens in a single scanout cycle.

All modern compositors use atomic KMS exclusively. The legacy path is still in the kernel for old userspace but deprecated.

### Properties: how you actually program KMS

Every KMS object has **properties**. Connectors have `EDID`, `DPMS` (power state), `content protection`. CRTCs have `ACTIVE`, `MODE_ID`. Planes have `SRC_X/Y/W/H` (what region of the buffer to read), `CRTC_X/Y/W/H` (where on screen to show it), `FB_ID` (which framebuffer to scan out), `rotation`.

An atomic commit is: "for each (object, property) pair in this batch, set the new value; commit." You iterate property by property, building up the state, then fire the ioctl.

```bash
# Inspect KMS state on your system (Fedora: dnf install libdrm-utils)
drm_info                              # long structured dump of everything
drm_info -j                           # JSON variant

# Simpler: show connectors and modes
cat /sys/class/drm/card0-*/status     # connected/disconnected
for f in /sys/class/drm/card0-*/modes; do echo "--- $f ---"; cat "$f"; done
```

`drm_info` is the single best tool for understanding what KMS is exposing on your box. Run it once; read the output; everything about the hardware will be in there.

### Framebuffer objects and formats

A **framebuffer** (drm_framebuffer) is a handle tying together GEM buffer(s) and pixel format metadata (width, height, stride, pixel format like XRGB8888 or NV12). You can't scan out a raw GEM buffer; you wrap it in a framebuffer first.

Pixel formats can be:

- **Packed RGB** — XRGB8888, RGBA8888, RGB565, etc.
- **YUV planar** — NV12 (Y plane + interleaved UV plane), YUV420 (three planes), etc. For video.
- **Compressed** — some hardware supports framebuffer compression for bandwidth savings.

**Modifiers** describe tiling and compression. Traditional linear buffers are straightforward; modern GPUs prefer tiled layouts for cache locality (e.g., AMD's DCC compression, Intel's Y-tiling). A modifier is a 64-bit value encoding "how this buffer's bytes are arranged in memory." Importing a buffer requires the importer to understand its modifier, which is why dma-buf fd passing also carries modifier info.

Linear (modifier 0) always works across all drivers. Tiled is faster but driver-specific unless both sides negotiate a common modifier. This is where Wayland's `linux-dmabuf-v1` protocol comes in: compositor advertises supported formats+modifiers, client picks one both agree on.

### Vblank, page flip, and frame pacing

A display scans out a frame at a fixed rate — say 60 Hz. Between frames there's a brief **vblank** (vertical blanking) period where nothing is being scanned. Changing what's on screen during scanout causes tearing; changing during vblank is invisible.

**Page flip** is the technique: program the CRTC to read from a new framebuffer, but only effective on the next vblank. Double or triple buffering: one buffer is being displayed, one is being rendered; flip them on vblank.

KMS exposes this with:

- `DRM_MODE_PAGE_FLIP_EVENT` — send me an event when the flip completes.
- `DRM_IOCTL_WAIT_VBLANK` — wake me up on the next vblank.
- Atomic commits with `DRM_MODE_ATOMIC_NONBLOCK` flag — submit and continue, receive completion event via the file descriptor.

Event delivery goes through the `/dev/dri/card0` file descriptor: `read()` on it returns `drm_event` structs. A standard epoll-able event source. This is how compositors pace their rendering — they submit a frame, wait for the page-flip completion event, then start rendering the next frame.

**VRR** (Variable Refresh Rate, aka FreeSync/G-SYNC) breaks the fixed-interval assumption. The CRTC can delay vblank within a range; the compositor page-flips whenever it's ready, up to a max rate. Less latency, no tearing, no wasted frames. KMS exposes VRR as a per-connector property (`VRR_ENABLED`).

**Tearing control** (Wayland's `wp-tearing-control`) lets clients opt in to tearing — submitting frames mid-scanout for minimum latency, accepting the visual tear. Games prefer this over vsync when frame rate exceeds refresh.

## The full flow: pixel from app to screen

Putting it all together. Say you're running `foot` (a Wayland terminal) in GNOME on an AMD GPU.

1. **foot** allocates a buffer. In practice: calls into GTK → allocates via Wayland's `linux-dmabuf-v1` protocol → receives a dma-buf fd from the compositor's buffer allocator, or allocates its own via Mesa's Vulkan/GL backend which uses `amdgpu` ioctls.
2. **foot** renders into the buffer. Vulkan commands → Mesa's `radv` driver → command buffer → submitted via `DRM_IOCTL_AMDGPU_CS` → GPU renders → sync object signals on completion.
3. **foot** sends the dma-buf fd to **mutter** (GNOME's compositor) via Wayland: `wl_surface.attach(buffer)` + `wl_surface.commit`. An fd passes over the Unix socket via SCM_RIGHTS.
4. **mutter** imports the dma-buf (`DRM_IOCTL_PRIME_FD_TO_HANDLE`), gets a GEM handle.
5. **mutter** builds its scene, either:
    - **Compositing in GPU**: renders all client surfaces into its own framebuffer via its own command buffers (also amdgpu, also submitted to the GPU).
    - **Direct scanout**: if foot's surface covers a plane exactly, assign the dma-buf directly to a hardware plane — no compositing work needed.
6. **mutter** issues an atomic commit: set the primary plane's `FB_ID` to the new framebuffer, flags include `PAGE_FLIP_EVENT`.
7. **Kernel KMS** programs the AMD display hardware: next vblank, switch the primary plane to the new buffer.
8. **AMD display engine** scans out the buffer over DisplayPort at 60 Hz (or whatever).
9. On vblank, KMS fires the page-flip completion event on mutter's `/dev/dri/card0` fd.
10. **mutter** reads the event, knows the frame is presented, sends `wl_surface.frame` callback `done` to foot, emits the `wl_buffer.release` when the buffer is no longer needed.
11. **foot** knows to start rendering the next frame.

Every step is observable:

- `strace -fe ioctl foot` — see DRM ioctls foot makes.
- `WAYLAND_DEBUG=1 foot` — see Wayland protocol.
- `sudo dmesg -w` — kernel driver messages.
- `amdgpu_top` — real-time GPU activity, memory usage, utilization.
- `sudo perf trace -e 'drm:*'` — DRM tracepoints.

## Hands-on: peek at KMS state

Three angles to look at KMS.

### Via sysfs

```bash
ls /sys/class/drm/
# card0  card0-DP-1  card0-DP-2  card0-HDMI-A-1  card0-eDP-1  renderD128

# Is my laptop panel connected?
cat /sys/class/drm/card0-eDP-1/status
# connected

# What modes does it support?
cat /sys/class/drm/card0-eDP-1/modes
# 2560x1600
# 1920x1200
# ...

# Current mode?
cat /sys/class/drm/card0-eDP-1/enabled
cat /sys/class/drm/card0-eDP-1/dpms
```

### Via drm_info (libdrm-utils)

```bash
drm_info | less
```

Scroll through to see every connector, every CRTC, every plane, every property with current values and ranges. It's verbose but complete. Use `drm_info -j | jq` if you want to query specific parts programmatically.

### Via debugfs

For the really low-level view:

```bash
sudo ls /sys/kernel/debug/dri/0/
# amdgpu_firmware_info  amdgpu_vm_info  framebuffer  state  gem_info ...

sudo cat /sys/kernel/debug/dri/0/state
# Full atomic state dump: every CRTC, plane, connector, their current properties.

sudo cat /sys/kernel/debug/dri/0/framebuffer
# Every allocated framebuffer with size, format, refcount.
```

debugfs for DRM is rich. Driver-specific files expose GPU clock states, ring buffer activity, VRAM usage, thermal info. Not stable ABI, not intended for scripting, but invaluable when debugging.

## Tools that drive KMS directly

You rarely write raw DRM ioctls, but it's worth knowing what exists:

- **`modetest`** (`libdrm-tests` or `libdrm-utils`) — bare-metal KMS testing tool. `modetest -M amdgpu` lists everything. `modetest -M amdgpu -s 42@39:1920x1080` sets a mode without any compositor. Useful for "is the hardware broken or is userspace broken?" debugging.
- **`kmscube`** (`kmscube` package) — renders a spinning cube directly to KMS, no X/Wayland needed. Classic "did my GBM+EGL+KMS stack work?" sanity test.
- **`drm_info`** — structured dump (as above).
- **`kmsprint`** (part of `kms++` library) — more human-readable inspection.
- **`gamescope`** — Valve's session compositor. Runs as a thin compositor directly on KMS for games, wrapping an app (typically Steam) and exposing a single surface. Very low overhead.

For an embedded system running a single graphical app with no desktop, your tool is **GBM + EGL + KMS**: GBM (Generic Buffer Management) allocates buffers via DRM, EGL provides the GL context, KMS does the display. A "Linux graphics stack without X or Wayland." Common in kiosks, car infotainment, Steam Deck's gaming mode.

## Multi-GPU and PRIME

Many laptops have two GPUs: an integrated (iGPU, inside the CPU, low power) and a discrete (dGPU, separate chip, high performance). The display is usually physically wired only to the iGPU. How do you render on the dGPU and display on the iGPU?

**PRIME** is the answer. Both GPUs appear as DRM devices (`card0` for iGPU, `card1` for dGPU, etc.). The app opens the dGPU's render node, renders, exports the result as dma-buf, the iGPU imports it and composites/scans it out.

Selection happens via env vars or config:

```bash
# Force NVIDIA (on Optimus laptops)
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia glxinfo | grep renderer

# Force AMD dGPU (on hybrid AMD systems)
DRI_PRIME=1 glxinfo | grep renderer

# Which GPU do I have?
DRI_PRIME=0 glxinfo | grep renderer   # iGPU
DRI_PRIME=1 glxinfo | grep renderer   # dGPU
```

Wayland compositors handle PRIME transparently now; the user just sees "the faster GPU is available when needed." Hybrid GPU power management (dGPU asleep when not in use) saves significant battery and is handled by the kernel plus a userspace daemon (`switcheroo-control` on GNOME).

## The driver family

DRM drivers, roughly by hardware:

- **`amdgpu`** — AMD GCN through RDNA4. Mature, excellent.
- **`radeon`** — pre-GCN AMD (TeraScale and older).
- **`i915`** — Intel through Gen12 (Tiger Lake etc.). Mature.
- **`xe`** — Intel's new driver, successor to i915. Used for newer Intel GPUs (Meteor Lake, Arc discrete, etc.).
- **`nouveau`** — open-source NVIDIA driver. Historically reverse-engineered; now has NVIDIA-published firmware for Turing+ but still constrained on newer cards.
- **`nvidia-drm`** — proprietary NVIDIA driver's kernel module. Speaks DRM ioctls, backed by NVIDIA's closed userspace.
- **`nova`** — new Rust-based NVIDIA driver, in development, aims to replace `nouveau` for modern GPUs.
- **`msm`** — Qualcomm Adreno (Snapdragon mobile GPUs).
- **`v3d`** — Raspberry Pi's VideoCore.
- **`panfrost`, `panthor`** — ARM Mali.
- **`etnaviv`** — Vivante.
- **`lima`** — older ARM Mali.
- **`vkms`** — Virtual KMS, for testing without hardware.
- **`virtio-gpu`** — virtualized GPUs (QEMU, VMs).
- **`qxl`, `bochs`, `cirrus`** — simple virtual GPUs, mostly legacy.
- **`simpledrm`** — takes over from firmware framebuffer (EFI-GOP), minimal, used early in boot before real driver binds.

When a system boots, `simpledrm` usually holds the screen (inheriting the firmware's framebuffer) until the real driver loads and takes over. The handoff is usually invisible.

```bash
lspci -k | grep -A 3 -i vga           # which driver is bound to your GPU
lsmod | grep -E 'amdgpu|i915|xe|nouveau|nvidia'    # which DRM drivers are loaded
```

## Color management, HDR, wide-gamut

KMS has been growing color-pipeline capabilities. A modern display pipeline can:

- Apply **degamma** (linearize input).
- Apply **color transformation matrix** (CTM, e.g., sRGB → BT.2020).
- Apply **gamma** (output-display non-linearity).
- Drive HDR formats (HDR10, Dolby Vision) via infoframes.
- Handle wide color gamuts (DCI-P3, Rec.2020).

These are KMS properties on CRTCs and planes. Wayland protocols are still stabilizing the user-facing side (`wp-color-management-v1` landed stable in 2024). In 2026, HDR works on Wayland in GNOME and KDE for supported GPUs, but the full color-managed pipeline across apps is still a moving target.

## Explicit sync: the recent NVIDIA saga

Worth a brief note as current history. For years, NVIDIA's proprietary driver had stuttering and flickering issues under Wayland, particularly with Xwayland games. Root cause: NVIDIA's driver used a different synchronization model than the open-source drivers, and the kernel DRM ioctl for buffer sharing assumed implicit sync.

Fix: **explicit sync protocol** for Wayland (`linux-drm-syncobj-v1`), plus kernel support for user-space submitted syncobjs, plus NVIDIA driver support, plus compositor support. Took about two years to land across the stack. As of 2024+, NVIDIA Wayland is dramatically better. Still not as bulletproof as AMD/Intel on some corner cases, but the gap has closed.

## Things that trip people up

**"My second monitor doesn't show up."** Check `cat /sys/class/drm/card0-*/status` — `disconnected` means the kernel doesn't see it, a hardware/cable/port issue. `connected` but `disabled` means the compositor hasn't enabled it.

**"Suspend/resume broke my display."** The driver must re-initialize the display controller on resume. Regressions here happen; `dmesg` after resume is the first place to look. For NVIDIA, `nvidia-suspend.service` used to need to run; less of an issue now.

**"Full-screen games tear."** Without VRR, tearing during vsync-off is inherent. With VRR enabled, it should be smooth within the VRR range. Check compositor settings — most require you to enable VRR per-output.

**"The cursor is stuck at the wrong scale."** Cursor planes don't always respect scaling consistently. This is a long-standing rough edge with HiDPI, especially with mixed-DPI multi-monitor setups. Mostly fixed on GNOME/KDE in current versions.

**"I can't run compositor X; it says DRM master is taken."** Only one process can be DRM master (hold CRTC control) at a time. If another compositor is running (VT 1 has GNOME, you're on VT 2), it's master; switching VTs transfers mastership. `sudo` from within an active session doesn't magic you into mastership; you need to be on the active seat.

**"`NO_HZ` timer issues" / "why is my compositor using so much CPU?"** Compositors wake on vblank. If they're drawing unnecessarily (damage tracking bug, wrong frame pacing), they'll hit 60 wakeups/sec. Tools: `powertop`, `perf top`, compositor-specific debug logs.

**"I can render but can't display."** Classic symptom of granting a user render-node access but not card access. Check `getfacl /dev/dri/card0` — your session should have an ACL. If not, logind isn't granting seat access, usually because you're in a TTY that's not the active seat (e.g., SSH, or `su - otheruser`).

## How DRM/KMS fits with everything else

- **vs Wayland** — Wayland is userspace app↔compositor protocol; DRM is compositor↔kernel. Wayland buffers (via `linux-dmabuf`) _are_ DRM buffers, passed by fd.
- **vs X11** — modern X.Org uses DRM/KMS too; it runs on top of the same kernel infrastructure a Wayland compositor does. X.Org is a compositor-plus in kernel terms; Xwayland is an X.Org implementation that uses a Wayland compositor as its display backend instead of direct KMS.
- **vs OpenGL/Vulkan** — these are userspace APIs. Mesa implements them on top of DRM. Nothing in GL or Vulkan semantics _requires_ DRM; it's the Linux transport.
- **vs V4L2** — video capture/codec subsystem. Shares dma-buf with DRM, so a webcam frame can be imported as a DRM buffer for zero-copy display or compositor input.
- **vs sysfs/procfs** — DRM has sysfs entries for status and debugfs for state dumps, but the real interface is ioctls. sysfs is read-only reflection.
- **vs BPF** — DRM has tracepoints (`trace_drm_*`) that BPF can attach to. Useful for "who's submitting the most GPU work?" investigations.
- **vs systemd/logind** — logind grants DRM access ACLs to active sessions. Without logind's cooperation, you can't open `/dev/dri/card0` as a regular user.

The architecture is: **DRM/KMS is the privileged, kernel-enforced interface to graphics hardware, exposing GPU rendering and display control via ioctls on character devices, with dma-buf as the universal buffer-sharing mechanism.** Everything graphical ultimately bottoms out here.

## Quick reference card

```bash
# Device discovery
ls /dev/dri/                              # card*, renderD*
ls /sys/class/drm/                        # cards + connectors + render nodes
lspci -k | grep -A 3 -i vga               # driver binding
lsmod | grep -E 'drm|amdgpu|i915|xe|nouveau|nvidia'

# KMS inspection
drm_info                                  # comprehensive dump
drm_info -j | jq '.[] | .connectors'      # structured queries
cat /sys/class/drm/card0-eDP-1/status     # per-connector status
cat /sys/class/drm/card0-eDP-1/modes

# Test tools
sudo modetest -M amdgpu                   # enumerate
sudo modetest -M amdgpu -s <conn>@<crtc>:<mode>    # force mode set
kmscube                                   # minimal GBM+EGL+KMS test

# Debug
sudo ls /sys/kernel/debug/dri/0/          # debugfs (driver-specific)
sudo cat /sys/kernel/debug/dri/0/state    # atomic state dump
sudo dmesg -w                             # kernel messages
sudo perf trace -e 'drm:*'                # tracepoints
amdgpu_top / intel_gpu_top / nvtop        # live GPU monitoring

# PRIME (hybrid GPU selection)
DRI_PRIME=1 glxinfo | grep renderer       # force AMD dGPU
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia <cmd>   # NVIDIA

# Permissions
getfacl /dev/dri/card0                    # session ACL
groups                                    # video, render should be here
```

## Where to learn more

- **`Documentation/gpu/` in the kernel source** — authoritative, deep. Per-driver docs plus cross-cutting topics like atomic mode setting, sync, memory management. Highly recommended if you want depth.
- **DRM subsystem page on kernel.org** — entry point.
- **`man 7 drm`** — terse but accurate overview.
- **freedesktop.org Mesa and DRM project pages** — for userspace side.
- **Daniel Vetter's LWN articles** — Vetter is a longtime DRM maintainer; his articles explain design decisions as they happen. Searching LWN for DRM coverage is rewarding.
- **`drm_info` source** — short, readable, pulls together the ioctl/property model nicely.
- **`kmscube` source** — the simplest real example of GBM+EGL+KMS in ~500 lines of C.
- **wlroots' KMS backend** — production-quality atomic KMS driving code in a readable codebase.

The mental model to carry: **DRM is the kernel's GPU and display subsystem, exposing two separable concerns — GEM (memory/rendering) and KMS (display) — through character-device ioctls, with dma-buf fds as the universal cross-subsystem buffer-sharing primitive, and atomic commits as the modern transactional interface to the display pipeline.** Every GPU interaction on Linux, from a Wayland compositor presenting a frame to a video player decoding straight to an overlay plane to a Vulkan game pushing commands, flows through these abstractions. Once the object model (connector → CRTC → plane → buffer) and the separation between render and scanout clicks, the whole Linux graphics story becomes navigable — and the reason Wayland feels "cleaner than X11" makes concrete sense: X11 accumulated thirty years of substitutes for DRM/KMS; Wayland compositors just use DRM/KMS directly.