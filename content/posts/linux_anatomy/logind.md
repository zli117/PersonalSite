---
date: '2026-04-19T21:16:07-07:00'
draft: true
title: 'Logind'
---
## What logind is and why it exists

Multi-user Unix assumed a simple world: users `ssh` or `telnet` in, get a shell, interact over that one session, log out. No graphics, no hot-plugging devices, no competing claims on hardware. The classical `utmp`/`wtmp` files tracked "who's logged in" and that was about it.

The desktop era broke all of this. Consider the problems:

- **Multiple users on one machine simultaneously.** Alice is logged in at the physical console in a graphical session; Bob SSHes in; Charlie switches to VT 3 and starts another graphical session. Who owns the audio device right now? Who can mount the USB stick that just got plugged in? Who's allowed to shut the machine down?
    
- **Fast user switching.** Alice locks her session; Bob logs in graphically; Bob uses the machine; Alice comes back and unlocks. The GPU, keyboard, mouse, audio have to be handed back and forth.
    
- **Session-scoped resources.** When Alice logs out, every process she started should die (no forgotten daemons clinging on). Her X server, her `gnome-shell`, her background syncing daemon, her Firefox — all gone. But her systemd user services, cron jobs, and SSH agent should persist correctly based on policy.
    
- **Power management decisions.** "Suspend the machine" is a privileged operation. Should any logged-in user be allowed to do it? Only the active user? What if there are multiple active users?
    
- **Idle and lock state.** Is anyone actually using the machine? Should the screen turn off? Should we go to sleep?
    
- **Hardware access for GUI.** The graphical session needs access to `/dev/dri/card0`, `/dev/input/event*`, `/dev/snd/*`. The user running it shouldn't be root. How does unprivileged Alice get those ACLs when she logs in at the console, and lose them when she switches away?
    
- **Wayland, which has no network transparency.** The compositor needs to know "is this user on the active seat right now or not" to decide whether to even try to open the display.
    

**`systemd-logind`** is the daemon that owns all of this. Started in 2011 as part of systemd, it tracks users, sessions, and seats, coordinates access to hardware, governs power operations, and exposes everything on D-Bus so other components can cooperate. Before logind, this landscape was a jumble of ConsoleKit, PolicyKit, pam_console, pam_foreground, udev ACLs, and desktop-specific hacks. logind consolidated the lot.

On any modern systemd-based Linux, logind is running whether you notice it or not. It's part of the plumbing under GDM/SDDM, under `su`/`sudo`, under SSH's session setup, under your Wayland compositor, under GNOME Shell's "suspend" button.

## The three core concepts: user, session, seat

logind's model is three nested objects. Getting these distinct in your head is most of understanding logind.

### User

A **user** is a logged-in Linux user, identified by UID. When the first session for a user opens, logind creates a user-scope state: spawns `user@UID.service` (the user's systemd manager, running the user-level units), allocates runtime resources like `/run/user/UID`, applies user-specific policy.

When the _last_ session for that user closes, logind tears this down — unless **lingering** is enabled for the user, in which case `user@UID.service` keeps running and their services persist.

Commands:

```bash
loginctl list-users
#   UID USER  LINGER STATE
#  1000 zl    no     active

loginctl show-user zl
# UID=1000
# GID=1000
# Name=zl
# Timestamp=...
# RuntimePath=/run/user/1000
# Service=user@1000.service
# Slice=user-1000.slice
# Display=2                       ← the "primary" session for this user
# State=active
# Sessions=3 2
# IdleHint=no

loginctl enable-linger zl         # user services persist past logout
loginctl disable-linger zl
```

Lingering matters for: running user-level services (syncthing, a personal media server, a cron-replacement timer) without staying logged in.

### Session

A **session** is one instance of a user being logged in. One user can have many sessions simultaneously (graphical session on seat0, SSH session from a laptop, SSH session from a phone). Each session has a numeric ID, a type, a class, a leader process, and a scope.

```bash
loginctl list-sessions
# SESSION  UID USER  SEAT  TTY
#       2 1000 zl    seat0 tty2        ← the graphical session
#       5 1000 zl          pts/0       ← an SSH session
#       8  501 bob   seat1 tty3        ← another user, another seat

loginctl show-session 2
# Id=2
# User=1000
# Name=zl
# Timestamp=...
# VTNr=2
# Seat=seat0
# TTY=tty2
# Remote=no
# Service=gdm-password
# Leader=1854
# Type=wayland
# Class=user
# Active=yes                    ← currently foregrounded on its seat
# State=active
# IdleHint=no
# LockedHint=no
```

Fields worth knowing:

|Field|Meaning|
|---|---|
|`Type`|`tty`, `x11`, `wayland`, `mir`, `unspecified`|
|`Class`|`user` (normal), `greeter` (GDM/SDDM), `lock-screen`, `background`, `manager`|
|`State`|`active` (foregrounded on its seat), `online` (exists but not foregrounded), `closing`|
|`Active`|boolean for "this session owns its seat's hardware right now"|
|`Remote`|`yes` for SSH, `no` for local|
|`VTNr`|the virtual terminal number, if any|
|`Leader`|PID of the session's root process (your login shell, your display manager's session spawner, etc.)|
|`Scope`|the systemd scope unit that contains all processes in this session|

**Sessions map to cgroup scopes.** Every process spawned in a session lives under `session-N.scope` under `user-UID.slice`. When the session ends, logind sends SIGTERM/SIGKILL to the whole scope — that's how "log out kills everything you started" is reliably implemented. No more hunting for stragglers.

```bash
systemctl status session-2.scope
# session-2.scope - Session 2 of User zl
#      CGroup: /user.slice/user-1000.slice/session-2.scope
#              ├─ 1854 gdm-session-worker [pam/gdm-password]
#              ├─ 1870 /usr/libexec/gnome-session-binary ...
#              ├─ 1892 /usr/bin/gnome-shell
#              └─ ... everything started from this session
```

### Seat

A **seat** is a set of hardware — one keyboard, one mouse, one display, maybe audio devices — representing _one physical place_ where a user can sit down and interact. Almost every laptop and desktop has exactly one seat, named `seat0`. Multi-seat setups (shared lab computer, multi-head kiosk) have `seat1`, `seat2`, etc.

Seats exist because of the multi-user graphical case. Two users on the same machine, each with their own keyboard/mouse/monitor, plugged into the same chassis — each gets their own seat, and the hardware is partitioned so their inputs and outputs don't cross.

```bash
loginctl list-seats
# SEAT
# seat0

loginctl show-seat seat0
# Id=seat0
# ActiveSession=2
# Sessions=2
# CanMultiSession=yes
# CanTTY=yes
# CanGraphical=yes
```

At any moment, **one session per seat is active** (the one currently foregrounded on the hardware). Ctrl+Alt+F3 switches VTs, which changes which session is active on `seat0`. Fast user switching in GNOME similarly switches the active session.

Devices are assigned to seats via udev tags. By default, all hardware is on `seat0`. For multi-seat, you tag devices manually:

```bash
# Pin a specific USB hub's devices to seat1
loginctl attach seat1 /sys/devices/pci0000:00/0000:00:14.0/usb1/1-3

# What's on each seat?
loginctl seat-status seat0
```

Unless you're building a multi-seat system, you'll only ever see `seat0`. But the concept matters because _session.Active_ is defined relative to a seat, and seat-tied hardware access is the mechanism for granting the desktop its device ACLs.

## The lifecycle in motion

Here's what happens for the three common login paths.

### Graphical login at the console

1. Machine boots. `gdm.service` starts on `graphical.target`. GDM spawns a greeter session — a special session with `Class=greeter` running as the `gdm` user, running its own Wayland/X compositor.
2. You type your password. GDM authenticates via PAM. The PAM stack includes `pam_systemd.so`, which talks to logind over D-Bus to register a session.
3. logind creates `session-2.scope`, sets up `/run/user/1000/`, starts `user@1000.service` if this is your first session, applies the session class (`user`), binds it to `seat0`, marks it active.
4. logind updates device ACLs: `/dev/dri/card0`, `/dev/input/event*`, `/dev/snd/*` gain ACL entries granting your UID access. Your compositor can now open them.
5. GDM spawns your actual session: `gnome-session`, which in turn launches `gnome-shell`, a settings daemon, autostart apps. All under `session-2.scope`.
6. Your compositor opens `/dev/dri/card0`, becomes DRM master for its CRTCs, starts compositing.

Check with:

```bash
loginctl show-session $XDG_SESSION_ID
getfacl /dev/dri/card0        # you'll see "user:1000:rw-"
```

### Fast user switching

You click "Switch User" in GNOME. GDM starts a second greeter session on a different VT. Bob logs in, gets `session-8` on seat0, `Active=yes`. Your `session-2` goes to `Active=no`, `State=online`. Device ACLs flip: Bob now has access to `/dev/dri/card0`, you don't. Your compositor, trying to use the GPU, gets `EPERM` or loses DRM master gracefully and sits dormant.

When you switch back (Ctrl+Alt+F2, or via GDM), the reverse happens: your session goes active, Bob's goes online, ACLs flip back, your compositor regains DRM master and resumes.

### SSH login

1. `sshd` accepts the connection, authenticates you, sets up the PTY.
2. `pam_systemd.so` registers a session with logind. This session has `Type=tty`, `Remote=yes`, `Seat=` (empty — no seat for remote sessions), no VTNr.
3. logind still creates a `session-N.scope` and, if this is your first session, starts `user@UID.service` and sets up `/run/user/UID/`.
4. Your shell runs under `session-N.scope`.
5. No device ACLs are granted (no seat).

This means SSHing in and then trying to launch a GUI app doesn't work — you have no seat, no DRI access. It also means your SSH session is still tracked by logind, still has a user slice, still runs user systemd services if any.

```bash
# From inside an SSH session:
echo $XDG_SESSION_TYPE      # tty
loginctl show-session $XDG_SESSION_ID | grep -E 'Remote|Seat'
# Seat=                    (empty)
# Remote=yes
```

## The D-Bus interface: how other components talk to logind

logind's power comes from being a D-Bus service that everything else integrates with. Service name: `org.freedesktop.login1`. Object paths: `/org/freedesktop/login1` (the manager), `/org/freedesktop/login1/session/_2` (sessions), `/org/freedesktop/login1/user/_1000` (users), `/org/freedesktop/login1/seat/seat0` (seats).

```bash
busctl tree org.freedesktop.login1
busctl introspect org.freedesktop.login1 /org/freedesktop/login1
```

Key methods on the Manager interface:

- `PowerOff`, `Reboot`, `Suspend`, `Hibernate`, `HybridSleep`, `SuspendThenHibernate` — power operations, with polkit checks.
- `Lock`, `Unlock`, `LockSession`, `UnlockSession` — lock screen coordination.
- `TerminateSession`, `TerminateUser`, `KillSession`, `KillUser` — force-close.
- `Inhibit` — take an **inhibitor lock** (see below).
- `GetSeat`, `GetUser`, `GetSession`, `ListSeats`, `ListUsers`, `ListSessions` — introspection.
- `ActivateSession`, `ActivateSessionOnSeat` — bring a session to the foreground (fast user switching).
- `SetUserLinger` — enable/disable linger.
- `ScheduleShutdown`, `CancelScheduledShutdown` — delayed poweroff.

Key signals:

- `SessionNew`, `SessionRemoved`
- `UserNew`, `UserRemoved`
- `SeatNew`, `SeatRemoved`
- `PrepareForShutdown(bool start)` — fires right before the system goes down (and right before it comes back).
- `PrepareForSleep(bool start)` — fires before suspend/resume.

The signals are how apps integrate. A media player listens for `PrepareForSleep(true)` and pauses playback. A backup daemon listens for `PrepareForShutdown(true)` and flushes state. A sync client listens for `SessionNew` to start syncing when a user logs in.

## Inhibitor locks: the cooperative model

"I'm burning a DVD, please don't suspend." "I'm playing a movie, please don't dim the screen." "I'm in the middle of a system update, please don't let the user shut down."

Inhibitor locks are logind's mechanism for this. Any app can take a lock against one or more events:

- `sleep` — prevent suspend/hibernate.
- `shutdown` — prevent poweroff/reboot.
- `idle` — prevent the idle hint from going to true (so screen doesn't blank, session isn't considered idle).
- `handle-power-key`, `handle-suspend-key`, `handle-hibernate-key`, `handle-lid-switch` — prevent the corresponding hardware event from being auto-handled by logind.

Two lock modes:

- **`block`** — absolutely prevents the event. An app taking a block lock can veto shutdown or sleep entirely. Requires privilege (usually root or special polkit rule). Used sparingly, typically only by critical services (the package manager during an update).
- **`delay`** — the event happens anyway, but logind waits up to `InhibitDelayMaxSec` (5 seconds by default) for the lock holder to do last-minute work. Most apps use delay locks: pause, save state, release the lock.

```bash
systemd-inhibit --list
# WHO              UID USER    PID  COMM            WHAT           WHY                                 MODE
# NetworkManager   0   root    942  NetworkManager  sleep          NetworkManager needs to turn off... delay
# GNOME Shell     1000 zl      1902 gnome-shell     handle-power-key:handle-suspend-key... handle   block
# ...
```

You can take one yourself:

```bash
systemd-inhibit --what=sleep --who="me" --why="running experiment" sleep 600
# Now the system can't suspend for 10 minutes.
```

This is how compositors and media players cooperate with power management without having to fight logind — they say "don't suspend while I'm busy", release when done, and everything just works.

## Power operations and polkit

All the power-affecting methods (`PowerOff`, `Reboot`, `Suspend`, etc.) go through polkit for authorization. Default polkit rules typically allow:

- Any user with an _active_ local session to suspend/hibernate.
- Any user with an active local session to reboot/shutdown _if they're the only user logged in_.
- Root, or users in specific groups (`wheel`), to always do these operations.

Remote SSH sessions are generally denied (no active seat → no "active local session").

When `systemctl poweroff` runs as an unprivileged user, it calls `org.freedesktop.login1.Manager.PowerOff`, which hits polkit, which consults rules, which may prompt via polkit agent (GNOME's dialog, `pkexec` prompt on TTY), which either allows or denies.

You can see the decision process:

```bash
pkaction --verbose --action-id org.freedesktop.login1.power-off
# description:       Power off the system
# message:           Authentication is required to power off the system
# implicit any:      auth_admin_keep
# implicit inactive: auth_admin_keep
# implicit active:   yes                  ← active local user gets it free
```

When `Suspend` is called, logind fires `PrepareForSleep(true)`, waits for all delay inhibitors (up to the timeout), then executes the syscall (`echo mem > /sys/power/state` or equivalent), and on resume fires `PrepareForSleep(false)`.

## Policy: `/etc/systemd/logind.conf`

logind's behavior is configurable. The main knobs:

```ini
[Login]
HandlePowerKey=poweroff         # what to do when the power button is pressed
HandleSuspendKey=suspend
HandleHibernateKey=hibernate
HandleLidSwitch=suspend         # laptop lid closed
HandleLidSwitchExternalPower=suspend
HandleLidSwitchDocked=ignore    # lid closed while on a dock

PowerKeyIgnoreInhibited=no
LidSwitchIgnoreInhibited=yes

IdleAction=ignore               # what to do when system idle (not per-session!)
IdleActionSec=30min

InhibitDelayMaxSec=5            # how long to wait for delay inhibitors

KillUserProcesses=yes           # kill everything on logout (subject to linger)
KillOnlyUsers=                  # restrict kill to specific users
KillExcludeUsers=root

RuntimeDirectorySize=10%        # /run/user/UID cap
RuntimeDirectoryInodesMax=
UserStopDelaySec=10             # grace period after last session before stopping user@.service

SessionsMax=8192                # max concurrent sessions
InhibitorsMax=8192

RemoveIPC=yes                   # clean up SysV IPC on user logout
```

Changes take effect on `systemctl restart systemd-logind` (which will invalidate existing sessions — don't do this casually on a live desktop).

A few of these have real consequences:

- **`HandleLidSwitch`** — the difference between "laptop sleeps when I close it" and "laptop runs with lid closed." For a server or a laptop docked to external monitors, you often want `ignore` or `lock`.
    
- **`KillUserProcesses`** — historically `no` on many distros (POSIX said logout shouldn't kill background jobs). systemd flipped this to `yes` for predictability. If you run `nohup`-style background tasks, either they need linger, or this needs to be `no`, or you should just write a proper systemd service.
    
- **`RuntimeDirectorySize`** — `/run/user/UID` is a tmpfs. Wayland sockets, PulseAudio sockets, D-Bus sockets, lots of little files. If you hit the cap, weird breakage ensues.
    
- **`IdleAction`** — system-wide idle, not per-user idle. Most people leave this off and let the desktop environment handle user idle (screen blank, lock).
    

## `loginctl`: the CLI

```bash
# Overview
loginctl                             # same as list-sessions
loginctl list-sessions
loginctl list-users
loginctl list-seats

# Detail
loginctl show-session <id>
loginctl show-user <username-or-uid>
loginctl show-seat <seat>

# Session control
loginctl activate <session>          # bring to foreground (fast user switching)
loginctl lock-session <session>
loginctl unlock-session <session>
loginctl terminate-session <session>
loginctl kill-session <session>

# User control
loginctl terminate-user <user>
loginctl enable-linger <user>
loginctl disable-linger <user>

# Seat control
loginctl seat-status <seat>
loginctl attach <seat> <sysfs-path>  # assign a device to a seat
loginctl flush-devices               # clear all seat assignments

# Power (via logind)
loginctl poweroff
loginctl reboot
loginctl suspend
loginctl hibernate

# User's currently-logged-in info
loginctl user-status                 # your own sessions
```

One thing that confuses people: `loginctl` and `systemctl` both have `poweroff`, `reboot`, `suspend`. They do essentially the same thing — both end up in logind for authorization. `systemctl poweroff` is more common, `loginctl poweroff` works identically.

## Tutorial: tracing a session

Let's walk through observing a real session on your machine.

```bash
# 1. Find your session ID (inside a shell, sourced from env)
echo $XDG_SESSION_ID
# 2

# 2. See its metadata
loginctl show-session $XDG_SESSION_ID
# ... look at Type, Class, State, Active, Seat

# 3. See its cgroup scope
systemctl status session-$XDG_SESSION_ID.scope
# Tree of every process belonging to this session

# 4. See your user slice
systemctl status user-$(id -u).slice
# Includes the session scope and user@UID.service

# 5. See your user services
systemctl --user list-units --type=service

# 6. Watch session events live
busctl monitor org.freedesktop.login1 &
# In another terminal: switch VTs, lock screen, etc., watch signals fire
kill %1

# 7. Who's got inhibitors right now?
systemd-inhibit --list

# 8. What device ACLs does your session have?
for d in /dev/dri/card* /dev/input/event* /dev/snd/*; do
  acl=$(getfacl "$d" 2>/dev/null | grep "user:$(id -u)")
  [ -n "$acl" ] && echo "$d -> $acl"
done
```

This walk teaches you more about logind in 30 seconds than any doc.

## Linger in depth

Linger is the escape hatch from "logout kills everything." It's a per-user bit: if set, `user@UID.service` keeps running past the last session close, and user systemd services / timers continue executing.

```bash
loginctl enable-linger zl
cat /var/lib/systemd/linger/zl       # empty file; presence = lingering
```

When does lingering matter:

- **Personal services that should run 24/7.** Syncthing, a personal Git server, a Matrix bridge, a home-automation integration. Put them in `~/.config/systemd/user/`, enable them, enable linger. They survive logout and reboot.
- **Headless servers you SSH into.** Without linger, your tmux session dies when SSH disconnects (if `KillUserProcesses=yes`). With linger, `systemctl --user` services persist.

When it doesn't:

- **Shared graphical machines.** Leaving user services running for every logout is usually wasteful.
- **Interactive programs you want ended.** Linger doesn't save your interactive shell or manually-backgrounded jobs — those still die unless launched under a systemd user scope.

The correct idiom for "I want this daemon to run even after I log out" is:

```bash
loginctl enable-linger $USER
cat > ~/.config/systemd/user/mydaemon.service <<'EOF'
[Unit]
Description=My daemon

[Service]
ExecStart=/home/%u/bin/mydaemon
Restart=on-failure

[Install]
WantedBy=default.target
EOF
systemctl --user daemon-reload
systemctl --user enable --now mydaemon.service
```

Survives logout, survives reboot.

## Integration points worth knowing

**PAM.** `pam_systemd.so` is the bridge. Most PAM service configs (login, sshd, gdm-password, su) include `session optional pam_systemd.so`, which calls `CreateSession` on logind at session start. Without this, logind has no idea your login happened.

**systemd user manager.** Each user's `user@UID.service` runs `systemd --user`, which is a per-user systemd instance managing `.service`/`.timer`/`.target` units out of `~/.config/systemd/user/` and `/etc/systemd/user/`. logind starts and stops this based on session state.

**Wayland/X11 compositors.** They open `/dev/dri/card*` and `/dev/input/event*`, which they only have access to because of logind's ACLs on the active session. When you switch away, they lose access; when you switch back, they regain it. A well-written compositor handles the transition gracefully (pauses rendering, releases DRM master, re-acquires on return).

**polkit.** Many polkit rules check session properties via logind: "is this user in an active local session?" drives many default allow/deny decisions. polkit calls `GetSession` and `GetSeat` on logind to answer these questions.

**Desktop environments.** GNOME Shell, KDE Plasma, and their settings daemons listen for logind D-Bus signals (`PrepareForSleep`, `PrepareForShutdown`, `SessionNew`) and issue methods (`Suspend`, `Lock`) in response to user actions.

**SSH.** `sshd` itself doesn't depend on logind, but session tracking and `$XDG_RUNTIME_DIR` setup for SSHed-in users come from the `pam_systemd.so` line in `/etc/pam.d/sshd`.

## Things that trip people up

**"My user services stop when I log out."** Linger not enabled. `loginctl enable-linger $USER`.

**"I SSHed in but can't run GUI apps."** No seat → no DRI/input ACLs. Use `ssh -X` for X11 forwarding, or waypipe for Wayland, or RDP/VNC. Granting `/dev/dri/card0` access manually works for rendering but not for display.

**"systemctl poweroff says 'Access denied'."** Polkit rule doesn't allow it. Usually because you're remote (SSH), or there are multiple users logged in, or your session isn't active. `pkaction --verbose --action-id org.freedesktop.login1.power-off` shows the policy.

**"My session shows `State=closing` and never goes away."** Some process is blocking cleanup. `systemctl status session-N.scope` to see what's still there. Usually a misbehaving daemon or a frozen PTY; `loginctl terminate-session N` forces it.

**"/run/user/1000 is full."** `RuntimeDirectorySize` hit. Usually a leaking socket or tmpfile from some app. Check with `du -h /run/user/1000`. On modern systems the default cap is reasonable, but video editing or audio-heavy apps can balloon it.

**"Closing the lid suspends when I want it docked."** `HandleLidSwitchDocked=ignore` in `logind.conf`, plus make sure logind detects the dock (an external monitor on the right port usually does it). Or use `HandleLidSwitch=ignore` if you never want lid-close to suspend.

**"My graphical session loses hardware on VT switch."** This is by design — active session owns the hardware. The compositor should handle the transition (most modern ones do). If it doesn't, that's a compositor bug; complain upstream.

**"Why is `/tmp` shared but `/run/user/1000` private?"** `/tmp` is global (classic Unix); `/run/user/UID` is per-user, tmpfs, created/destroyed by logind, with mode 0700. Use it for per-user sockets and runtime files; logind cleans up automatically on last-session-close (or on reboot if lingering).

**"Bus sessions don't carry XDG environment to cron/systemd units."** User systemd manager inherits a minimal environment. If a user service needs `DISPLAY`, `DBUS_SESSION_BUS_ADDRESS`, `WAYLAND_DISPLAY`, etc., use `systemctl --user import-environment VAR VAR VAR` from your session startup, or set explicitly in the unit's `Environment=`.

## How logind fits with everything else

- **vs PAM.** PAM authenticates; logind tracks. PAM's `pam_systemd.so` is the bridge that notifies logind.
- **vs systemd the init system.** logind is a separate daemon that happens to ship with systemd. It uses systemd's cgroup and unit machinery — sessions are scopes, users have slices, user managers are services. But logind itself is not PID 1.
- **vs udev.** udev names devices and tags them with seat assignments. logind applies the session-scoped ACLs on top.
- **vs polkit.** polkit asks logind "who is this caller, are they active, are they local?" to make authorization decisions.
- **vs the compositor.** Compositor relies on logind for device ACLs and session-active state. When inactive, compositor releases DRM master; when active again, reacquires.
- **vs the user's systemd instance.** Each logged-in user has their own `systemd --user` running inside `user@UID.service`. logind starts and stops this based on session presence (or keeps it running if lingering).

## Quick reference card

```bash
# Inspection
loginctl                              # overview (list-sessions)
loginctl list-sessions
loginctl list-users
loginctl list-seats
loginctl show-session <id>
loginctl show-user <user>
loginctl show-seat <seat>
loginctl user-status

# Session identity inside a shell
echo $XDG_SESSION_ID
echo $XDG_SESSION_TYPE                # tty, wayland, x11
echo $XDG_RUNTIME_DIR                 # /run/user/UID
echo $XDG_SEAT                        # seat0 or empty

# Session control
loginctl activate <id>                # fast user switch
loginctl lock-session <id>
loginctl unlock-session <id>
loginctl terminate-session <id>

# User services / linger
loginctl enable-linger <user>
loginctl disable-linger <user>

# Inhibitors
systemd-inhibit --list
systemd-inhibit --what=sleep --who=me --why=... <cmd>

# Power
loginctl poweroff
loginctl reboot
loginctl suspend
loginctl hibernate

# D-Bus
busctl tree org.freedesktop.login1
busctl introspect org.freedesktop.login1 /org/freedesktop/login1
busctl monitor org.freedesktop.login1

# Config
$EDITOR /etc/systemd/logind.conf
# or drop-in: /etc/systemd/logind.conf.d/*.conf
systemctl restart systemd-logind      # disruptive on a live desktop!

# cgroup view
systemctl status user-$(id -u).slice
systemctl status session-$XDG_SESSION_ID.scope
systemctl --user list-units
```

## Where to learn more

- **`man logind.conf`, `man loginctl`, `man systemd-logind.service`, `man sd-login`, `man pam_systemd`** — the references. `man logind.conf` is where all the policy knobs are documented. `man sd-login` documents the C API other daemons use to integrate.
- **freedesktop.org logind D-Bus spec** — the authoritative list of methods, signals, properties. Shorter and cleaner than the man pages for the API surface.
- **The systemd source tree** under `src/login/` — readable C, not huge, and it's clarifying to see how sessions are actually represented.
- **Lennart Poettering's "systemd for administrators" blog series, part 13** — original design essay on multi-seat. Dated but explains the why.
- **`busctl introspect`** on logind's objects — your system's _current_ logind is always the authoritative answer to "what methods exist, what do they do, what's the current state."

The mental model to carry: **logind tracks who's logged in (user), how they're logged in (session), and where they're sitting (seat), arbitrates access to hardware based on active-seat policy, governs power operations through inhibitors and polkit, and exposes all of it as a D-Bus API for everything else in the userspace stack to coordinate around.** It's the answer to "how does a modern Linux desktop handle multi-user, multi-session, hot-plug, suspend/resume, graphical sessions, and device access all at once," and most of the machinery becomes legible once you have the user/session/seat triangle in your head.