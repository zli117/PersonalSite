---
date: '2026-04-19T22:53:27-07:00'
draft: true
title: 'Capstone'
---
# Capstone: from power button to desktop

We've walked through the layers of desktop Linux one at a time — kernel interfaces, udev, systemd, D-Bus, NetworkManager, PAM, logind, polkit, DRM/KMS, Wayland. Each in isolation makes sense. The harder question is what happens when they all run at once: how do they *compose*?

This capstone traces a single real journey: **you press the power button on your laptop. Moments later, you're looking at the GNOME desktop, and your WiFi is connected.** Every layer of the series appears, in sequence, doing its actual job.

The trace is specific to a modern Fedora Workstation on a laptop with an AMD GPU and a MediaTek MT7925 WiFi card. The exact drivers differ on other hardware, but the shape is universal.

## Before we start: what's already configured

A few things are assumed in place from prior use:

- A Fedora Workstation install with systemd, NetworkManager, GDM, GNOME on Wayland.
- UEFI firmware, Secure Boot enabled, systemd-boot or GRUB as bootloader.
- A user `user` with a password and a graphical session history.
- A saved WiFi connection profile `HomeNet` in `/etc/NetworkManager/system-connections/`, with `connection.autoconnect=yes` and the PSK stored with `psk-flags=0` (system-wide, not agent-owned — so it can activate before anyone logs in).

These determine what happens. If linger were enabled, or the WiFi password were agent-owned, or the user were an SSSD-backed domain account, the trace would branch.

## The scene

You open the lid. The laptop is off — S5 or fully powered down. You press power.

## 0:00 — Firmware does its thing

The details here are out of scope for the series, so we'll move fast. The firmware (UEFI) runs its early boot: initializes RAM, probes the CPU, enumerates enough PCI to find the boot device, checks Secure Boot signatures on the bootloader binary, and hands control to it. On Fedora, that's typically `shimx64.efi` (Microsoft-signed) → `grubx64.efi` or `systemd-bootx64.efi` (Red Hat-signed).

The bootloader reads its config, finds the default kernel and initramfs, verifies their signatures against the kernel key, loads them into memory, and jumps to the kernel.

Everything up to now runs in firmware context with physical memory, no virtual memory, no processes. The kernel is about to change all that.

## 0:00.5 — Kernel starts

The Linux kernel takes over. In roughly this order:

1. **Set up the CPU.** Enable long mode, turn on virtual memory with an initial page table, enable the MMU.
2. **Parse the kernel command line.** `/proc/cmdline` will later reflect this. Contains things like `root=UUID=...`, `rd.luks.uuid=...`, `rhgb quiet`, `resume=...`, plus any knobs like `mitigations=auto`.
3. **Initialize core subsystems.** Memory allocator, scheduler, timer, SMP. Each CPU core comes online.
4. **Unpack the initramfs.** A cpio archive loaded by the bootloader, unpacked into a tmpfs mounted at `/`. Contains just enough userspace to find and mount the real root filesystem: device drivers, a minimal `/dev`, `udevadm`, `systemd`, LUKS tooling if encrypted, `fsck`.
5. **Load essential drivers.** The kernel auto-detects hardware via PCI/ACPI enumeration; modaliases are matched against built-in or initramfs-loaded modules. For our machine: `amdgpu` for the GPU (though it may wait), `nvme` for the NVMe controller, `mt7925e` for WiFi (may wait), `xhci_hcd` for USB, dozens more.
6. **Start PID 1.** The kernel execs `/init` in the initramfs, which is a symlink to `systemd`. PID 1 is now userspace.

Everything the series has covered at the kernel level — syscalls, sysfs, procfs, device nodes, drivers — is now live. `/proc/` has an entry for PID 1. `/sys/` has the kernel device model populated. `/dev/` has the essential nodes (null, zero, console, the root block device) via devtmpfs.

## 0:01 — systemd takes PID 1, in the initramfs

systemd-in-initramfs is a lean variant. Its goal: mount the real root filesystem at `/sysroot`, then hand off to a full systemd running from that root.

It does this by running a small set of units: `initrd.target` pulls in `systemd-udevd` (to populate `/dev` as drivers fire), `cryptsetup` (for LUKS), mount units for the root partition. Once `/sysroot` is mounted, it runs `systemd-fsck-root.service`, then pivots.

**The pivot is a classic Unix trick:** `switch_root` swaps the root filesystem. The initramfs contents are unmounted, the real root becomes `/`, and a new `systemd` from `/usr/lib/systemd/systemd` re-execs as PID 1. The kernel and all file descriptors persist; only userspace has been replaced.

At this moment, `/sysroot` no longer exists; `/` is the real filesystem. `/proc`, `/sys`, `/dev`, `/run` are remounted appropriately.

## 0:02 — systemd's main boot

Now the real boot graph runs. systemd reads `/etc/systemd/system/default.target` (a symlink to `graphical.target`) and starts resolving dependencies.

The sequence is parallel; many things happen at once. Roughly:

**`sysinit.target`** (~0:02 to 0:04):
- `systemd-udevd.service` starts (the userspace udev daemon). Kernel has been emitting uevents for every discovered device; udev now processes them, applies `/usr/lib/udev/rules.d/*` and `/etc/udev/rules.d/*`, assigns predictable names (the WiFi card becomes `wlp194s0`), sets permissions, creates symlinks under `/dev/disk/by-*`, `/dev/input/by-id/`, etc.
- `systemd-modules-load.service` loads any modules explicitly configured.
- `systemd-tmpfiles-setup.service` creates `/run` directories from `/etc/tmpfiles.d/` and `/usr/lib/tmpfiles.d/`.
- `systemd-journald.service` starts. The kernel ring buffer (`/dev/kmsg`) and earlier logs are captured. From here on, every unit's stdout/stderr lands in the journal.
- Other fs mount units for `/boot`, `/home`, `/var`, swap, etc.

Simultaneously, as drivers bind:
- `amdgpu` binds to the GPU. A `/dev/dri/card0` appears. KMS enumerates connectors (eDP-1 for the built-in panel, any external DP/HDMI). The kernel hands off the firmware framebuffer to `amdgpu`'s display driver.
- `mt7925e` binds to the WiFi chip. A `phy0` appears. `wlp194s0` is created as a network interface.
- `snd_hda_intel` binds to the audio codec. ALSA devices appear under `/dev/snd/`.
- Input devices: keyboard, touchpad, power button all show up under `/dev/input/event*`.

For each device, udev emits processed uevents on its netlink socket. Subscribers will react later.

**`basic.target`** (~0:04):
- `dbus.service` starts `dbus-broker`. The system bus socket at `/run/dbus/system_bus_socket` is now live. Any system service that wants to expose a D-Bus API from here on can.
- `systemd-logind.service` starts. Creates seat0 (all enumerated graphical/input hardware is attached to it by default). No sessions yet.
- `polkit.service` starts. Reads action declarations from `/usr/share/polkit-1/actions/` and rules from `/etc/polkit-1/rules.d/` and `/usr/share/polkit-1/rules.d/`.
- `systemd-resolved.service` starts. Begins listening on `127.0.0.53:53` as a stub resolver. `/etc/resolv.conf` is (via symlink) `/run/systemd/resolve/stub-resolv.conf`, pointing at it.
- `firewalld.service` starts. Loads zone definitions, applies nftables rules for the default zone (`FedoraWorkstation`).

**`multi-user.target`** pulls in (~0:05):
- `NetworkManager.service` starts. Reads `/etc/NetworkManager/NetworkManager.conf` and per-connection profiles under `/etc/NetworkManager/system-connections/`. Claims the system bus name `org.freedesktop.NetworkManager`. Enumerates interfaces via netlink: `lo`, `wlp194s0`, any wired interfaces, any virtual interfaces (docker0 if Docker installed).

NetworkManager doesn't activate anything yet — it's still deciding what to manage. The loopback comes up immediately. `wlp194s0` is marked as managed (nothing in `unmanaged-devices=` excludes it).

## 0:06 — NetworkManager activates the saved profile

For every managed device, NetworkManager looks for a connection profile with `connection.autoconnect=yes` that can bind to that device. Our `HomeNet` profile matches.

NetworkManager begins activation:

1. **Device state: `prepare`.** `wlp194s0` is brought up at the link layer via netlink (`RTM_SETLINK`, `IFF_UP`).
2. **Device state: `config`.** NetworkManager spawns (or talks to an already-running) `wpa_supplicant` over D-Bus. Tells it: "associate to SSID `HomeNet` with this PSK, on this interface." wpa_supplicant drives the 802.11 state machine: active scan for the SSID, authentication, association, 4-way handshake using the PSK to derive the session key.
3. **The kernel's mt7925e driver and cfg80211/mac80211** coordinate with the firmware on the card to complete the handshake. An encrypted WiFi link is now established.
4. **Device state: `ip-config`.** NetworkManager's internal DHCP client sends a DHCPDISCOVER on `wlp194s0`. The home router replies with an OFFER: IP `192.168.1.47`, gateway `192.168.1.1`, DNS servers `192.168.1.1`, lease 24 hours. DHCPREQUEST and DHCPACK complete the handshake.
5. **Route and address installation.** NetworkManager uses netlink (`RTM_NEWADDR`, `RTM_NEWROUTE`) to install `192.168.1.47/24` on `wlp194s0`, add a default route via `192.168.1.1`.
6. **DNS push.** NetworkManager calls `org.freedesktop.resolve1.Manager.SetLinkDNS` on resolved via D-Bus: "for interface wlp194s0, DNS is 192.168.1.1." resolved updates its per-link DNS config.
7. **Firewall zone.** NetworkManager calls `org.fedoraproject.FirewallD1.zone.changeZoneOfInterface` on firewalld: "put wlp194s0 in zone FedoraWorkstation." firewalld recompiles its nftables ruleset.
8. **Device state: `activated`.** NetworkManager emits a `StateChanged` D-Bus signal. `Connectivity` check runs (fetches a known URL to decide `full`/`portal`/`limited`/`none`). Suppose it returns `full`.

WiFi is up before anyone has logged in. Email sync, Syncthing, background services (if enabled at the system level) can now talk to the network.

## 0:08 — GDM starts

`gdm.service` is pulled in by `graphical.target`. GDM is the GNOME display manager. Its job: take ownership of a display, show a login screen, authenticate users, spawn their sessions.

GDM starts its own minimal session — a **greeter session** — as the `gdm` user. This is itself a logind session (Class=greeter). It runs its own small Wayland compositor (a cut-down mutter) on a new VT (tty1 or tty7 depending on config), opens `/dev/dri/card0` via the ACLs logind granted to the gdm user for its greeter session, performs a KMS mode set via atomic commit on the eDP-1 connector at the panel's native resolution, and starts scanning out the login screen.

The login screen appears. The whole path from power button to here typically takes 5-15 seconds, depending on hardware and storage speed.

## 0:12 — You log in

You type your password. GDM's login UI is a Wayland client of its own greeter compositor. When you press Enter, GDM's authentication worker (`gdm-session-worker`) runs the PAM stack from `/etc/pam.d/gdm-password`.

Roughly, the PAM stack does:

**auth stack:**
- `pam_env` sets baseline environment variables.
- `pam_faillock preauth` checks the lockout table; your account isn't locked.
- `pam_unix` reads `/etc/shadow`, hashes your typed password with the stored salt, compares. Match.
- On success, subsequent sufficient modules don't need to run.

**account stack:**
- `pam_nologin` checks if `/etc/nologin` exists. It doesn't.
- `pam_unix` verifies your account isn't expired.
- `pam_access` checks `/etc/security/access.conf` for user/host rules. Default pass.

**session stack (the interesting one):**
- `pam_selinux` prepares the SELinux context transition.
- `pam_loginuid` writes your UID to `/proc/self/loginuid`, which is immutable — audit records will track back to this login.
- `pam_limits` reads `/etc/security/limits.conf` and sets rlimits (file descriptors, process count, core size).
- `pam_systemd` is the critical one. It calls `org.freedesktop.login1.Manager.CreateSession` over D-Bus with your UID, the TTY, the session class (`user`), the session type (will be `wayland`), your remote-ness (`no`), your seat (`seat0`).

**Meanwhile in logind**, `CreateSession` triggers:

1. Allocate a new session ID, say `2`.
2. Create a cgroup scope `session-2.scope` under `user-1000.slice` (which itself sits under `user.slice`).
3. Check if `user@1000.service` is already running. It isn't (you haven't logged in yet). Start it — this spawns `systemd --user` as your user, which will manage your per-user services from `/etc/systemd/user/` and `~/.config/systemd/user/`.
4. Create `/run/user/1000` as a tmpfs with mode 0700 owned by you. Set `XDG_RUNTIME_DIR=/run/user/1000` in the session environment. This is where your Wayland socket, D-Bus session socket, PulseAudio/PipeWire sockets will live.
5. Grant device ACLs. For every hardware node on `seat0`, add an ACL entry for UID 1000: `/dev/dri/card0`, `/dev/dri/renderD128`, `/dev/input/event*`, `/dev/snd/*`. From this moment, your UID can open the GPU and input devices.
6. Mark session-2 as `Active=yes` on `seat0`. The greeter session becomes `Active=no` but `State=online`. GDM's greeter compositor still holds DRM master on eDP-1, but is about to give it up.
7. Emit `SessionNew` D-Bus signal.

## 0:13 — The session starts

GDM, now authenticated, terminates its greeter compositor (releasing DRM master) and execs `gnome-session` as your user within `session-2.scope`. This process becomes your session leader.

`gnome-session` reads `/usr/share/gnome-session/sessions/gnome.session` (or `gnome-wayland.session`) to decide what to run. It sets up the session environment and starts the compositor: `gnome-shell` (which includes mutter as the Wayland compositor, linked in).

**gnome-shell launches as the Wayland compositor:**

1. Opens `/dev/dri/card0`. Because of logind's ACL, this succeeds without root.
2. Becomes DRM master on eDP-1's CRTC (atomic commit to take the lease).
3. Initializes EGL on the GPU via Mesa's `radv` or `radeonsi` driver. Opens `/dev/dri/renderD128` too.
4. Reads input devices via **libinput**, which itself opens `/dev/input/event*` nodes (again, ACL-granted). Configures tap-to-click, acceleration, any user settings from gsettings.
5. Creates the Wayland socket at `$XDG_RUNTIME_DIR/wayland-0`.
6. Sets `WAYLAND_DISPLAY=wayland-0` in the session environment.
7. Performs a KMS atomic commit: primary plane showing an initial framebuffer (black, for now), CRTC active at the panel's mode. First frame scanned out.
8. Starts the session bus — actually, `dbus-broker --scope=user` was already started by `user@1000.service` as `dbus.service`, which is why `DBUS_SESSION_BUS_ADDRESS` is already set by the time gnome-shell reads it.

gnome-shell is now a full Wayland server. It registers its D-Bus interfaces: `org.gnome.Shell`, `org.gnome.Mutter.DisplayConfig`, and several more.

## 0:14 — Session services spawn

`gnome-session` reads the `[Desktop Entry]` files from `/usr/share/gnome-session/components/`, `/etc/xdg/autostart/`, `~/.config/autostart/`. In order, it starts:

- **`gsd-*`** — GNOME Settings Daemon helpers: `gsd-power`, `gsd-media-keys`, `gsd-color`, `gsd-keyboard`, `gsd-wacom`, etc. Each is a long-lived helper that listens for D-Bus signals (logind's `PrepareForSleep`, `PropertiesChanged` on hardware, etc.) and reacts.
- **`evolution-data-server`** — contacts, calendar, mail accounts.
- **`gnome-keyring-daemon`** — holds your secrets. It's already been started earlier by PAM (`pam_gnome_keyring.so` in the session stack), which unlocked the default keyring using your login password.
- **`tracker-miner-fs`** — indexes your files.
- **`xdg-desktop-portal`** and **`xdg-desktop-portal-gnome`** — the portal mediating sandboxed apps' access to host resources (screenshots, file pickers, etc.).
- **Your autostart apps** — anything you've dropped into `~/.config/autostart/`: Syncthing, a clipboard manager, whatever.

The Wayland clients among these (everything with a GUI — the settings daemons are mostly background, but the shell extensions, the top bar, the dock, the notification system) connect to `$WAYLAND_DISPLAY`, get the registry, bind interfaces (`wl_compositor`, `xdg_wm_base`, `wl_seat`, etc.), and start their lives.

## 0:15 — The desktop is alive

GNOME Shell itself is compositor *and* client of its own Wayland server (it draws the top bar, the Activities overview, the dock). The top bar queries NetworkManager over D-Bus: "what's the current connectivity?" NetworkManager replies: connected via wlp194s0 to HomeNet, full connectivity. The WiFi icon appears in the top bar, showing signal strength (gnome-shell periodically asks for RSSI). The battery icon queries UPower (`org.freedesktop.UPower`), which has been tracking `/sys/class/power_supply/BAT0/` since boot. The clock… just reads the system time.

Every piece of information you see on the top bar is a D-Bus property being polled or subscribed from a system service.

Under the hood, the compositor has assembled its scene, rendered it to a framebuffer using the GPU, and submitted an atomic KMS commit with `PAGE_FLIP_EVENT`. The flip happens on the next vblank. Pixels arrive on the screen.

You see the GNOME desktop. The WiFi icon shows a solid connection. Your cursor is visible and tracks your touchpad. You're in.

## Total elapsed time

On a decent SSD: 10-20 seconds from power button to usable desktop. Most of it is kernel init and systemd dependency resolution; the login and session startup are fast once they start.

## The cast, one more time

If we list every major component involved:

- **Firmware + bootloader** (out of scope): hand off to kernel
- **Kernel**: drivers, scheduler, memory, devtmpfs, sysfs, procfs, netlink
- **udev**: device naming, ACL priming via tags, uevent dispatch
- **systemd** (PID 1): dependency graph, service management, cgroup scopes
- **dbus-broker**: message routing for the system bus
- **systemd-logind**: session/seat tracking, device ACLs
- **systemd-resolved**: DNS stub resolver
- **firewalld**: nftables zone management
- **NetworkManager**: WiFi connection lifecycle
- **wpa_supplicant**: 802.11 authentication
- **polkit**: authorization (waiting in the wings, will fire when you do privileged things)
- **GDM**: display manager, greeter session, authentication entry
- **PAM**: authentication pipeline (auth, account, session stacks)
- **user@.service / systemd --user**: per-user service management
- **dbus-broker (session)**: per-user D-Bus bus
- **gnome-session**: session orchestration
- **gnome-shell / mutter**: Wayland compositor
- **Mesa + DRM/KMS**: GPU access, display configuration, atomic modesetting
- **libinput**: input event processing

Every single one of these we've discussed in the series. None of it is magic. The whole desktop is a dependency graph of cooperating processes, each doing one job well, coordinating over well-defined interfaces.

## The observation that makes this legible

Here's the punchline. If you can `systemctl list-dependencies graphical.target` and `busctl list` and `loginctl list-sessions` and read the output — if those three commands mean something to you — you've got a complete, live map of everything above. The boot isn't a black box. It's a graph you can query.

Try it now. On your running Fedora Workstation:

```bash
# The unit dependency tree that brought you here
systemctl list-dependencies graphical.target

# Your session, from logind's perspective
loginctl show-session $XDG_SESSION_ID

# Everyone on the system bus
busctl list

# Your user's systemd instance and its units
systemctl --user list-units

# The networking state as NetworkManager sees it
nmcli device status && nmcli connection show --active

# The compositor's view of the display
drm_info | grep -A 2 "Connector:"

# The full cgroup tree of your session
systemd-cgls /user.slice/user-$(id -u).slice/session-$XDG_SESSION_ID.scope
```

Every question you might ask — *what started this? who owns that? which process holds DRM master? what did PAM authenticate against? which cgroup does this live in?* — has a short command that answers it. The observatory is the point. Desktop Linux is a system you can see clearly, once you know where to look.

Welcome to the machine.