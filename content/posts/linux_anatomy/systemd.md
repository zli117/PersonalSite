---
date: '2026-04-19T20:58:50-07:00'
draft: true
title: 'Chapter 7: systemd'
weight: 70
---
## What systemd is and why it exists

Before systemd, Linux boot was a parade of shell scripts. SysV init ran `/etc/init.d/*` scripts sequentially based on runlevels; Upstart (Ubuntu's attempt) added event-based triggers. The problems piled up:

- **Slow.** Sequential script execution; no parallelism beyond crude tricks.
- **Unreliable.** Every daemon invented its own forking/PID-file dance; the init system had no reliable way to know if a service was actually running.
- **Fragmented.** Daemon supervision, socket activation, device events, cron, logging, login sessions — each was a separate project with its own config format.
- **Opaque.** A service failing at boot meant reading a pile of shell scripts and guessing at ordering dependencies.

systemd (started 2010 by Lennart Poettering at Red Hat, now the default on most major distros) took the position that **init shouldn't just start daemons — it should be a platform for managing system state**. The design bet was that consolidating service management, socket activation, device events, cgroups, logging, and session tracking into one coherent model would be net-simpler than the alternative, even if the individual project got large.

Whether this trade was worth it is an eternal flamewar. Pragmatically: on any mainstream distro in 2026, systemd owns the shape of the system, and fluency with it pays off enormously.

**The core promise:** a single, consistent, declarative model — _units_ — for describing everything the system needs to manage. One format, one set of commands, one log stream.

## The unit: the central abstraction

Everything systemd manages is a **unit**. A unit is a named configuration object with a type (given by its file extension) and a set of properties. Think of units as typed declarations of intent — "this service should run", "this mount point should exist", "this target is the goal of booting."

The unit types you'll encounter:

| Type         | What it describes                                        |
| ------------ | -------------------------------------------------------- |
| `.service`   | A process or daemon to run                               |
| `.socket`    | A listening socket that can activate a service on demand |
| `.timer`     | A scheduled trigger (replaces cron)                      |
| `.target`    | A grouping/synchronization point (like old runlevels)    |
| `.mount`     | A filesystem to mount                                    |
| `.automount` | A mount point that mounts on first access                |
| `.device`    | A device node (auto-generated from udev)                 |
| `.path`      | A filesystem path to watch                               |
| `.slice`     | A cgroup resource-limit container                        |
| `.scope`     | A cgroup for externally-managed processes                |
| `.swap`      | A swap file or partition                                 |

All units share a common syntax and a common set of management commands. You don't learn ten different daemon supervisors; you learn one system.

**Where units live:**

```
/usr/lib/systemd/system/    ← distro-shipped units. Don't edit.
/etc/systemd/system/        ← your units + overrides. Wins over /usr/lib.
/run/systemd/system/        ← runtime-generated. Vanishes on reboot.
```

Same three-tier layout as udev, firewalld, and most modern Linux config: vendor / admin / runtime, with the admin layer winning.

User-level services (running under your login, not root) live in parallel directories under `~/.config/systemd/user/`, etc.

## Reading a service unit

Here's a real service file — simplified version of what sshd's unit looks like:

```ini
[Unit]
Description=OpenSSH server daemon
Documentation=man:sshd(8) man:sshd_config(5)
After=network.target auditd.service
Wants=sshd-keygen.target

[Service]
Type=notify
ExecStartPre=/usr/sbin/sshd -t
ExecStart=/usr/sbin/sshd -D $OPTIONS
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=on-failure
RestartSec=42s

[Install]
WantedBy=multi-user.target
```

Three sections, each with its job:

**`[Unit]`** — metadata and dependencies. `Description` is for humans. `After=`/`Before=` specify ordering (not dependencies — just order). `Requires=`/`Wants=` specify dependencies — if A `Wants` B, then starting A also starts B; `Requires` is stricter (if B fails, A fails too).

**`[Service]`** — how to run it. `Type=` tells systemd how to tell when the service is "started" (covered below). `ExecStart=` is the main command. `ExecStartPre`, `ExecStartPost`, `ExecStop`, `ExecReload` are lifecycle hooks. `Restart=` controls auto-restart policy.

**`[Install]`** — what happens on `enable`. `WantedBy=multi-user.target` means "when multi-user.target is started, also start this service." This is how units get pulled in at boot.

## Service types — the most important single concept

`Type=` in a service unit tells systemd **when the service is considered "started"**, which matters because other units may be waiting on it. Get this wrong and you'll have services racing.

|Type|Meaning|
|---|---|
|`simple`|The main process is the one in `ExecStart`. Considered started immediately after fork. The default, but often wrong.|
|`exec`|Like `simple`, but considered started only after `execve()` succeeds. Usually what you actually want if not `simple`.|
|`forking`|Classic daemon: `ExecStart` forks and the parent exits. Started when parent exits. Needs `PIDFile=` for reliable tracking.|
|`oneshot`|A command that runs once and exits. Considered started after it completes. Combine with `RemainAfterExit=yes` to make the unit "stay active" afterward.|
|`notify`|The service calls `sd_notify(READY=1)` to signal readiness. Most reliable for daemons that support it.|
|`notify-reload`|Like `notify` but the service also signals during reload.|
|`dbus`|Service is "up" when it acquires a well-known D-Bus name.|
|`idle`|Like `simple` but delayed until other jobs finish. Cosmetic, for tty services so their output doesn't interleave.|

**If you're writing a new service**, prefer `Type=notify` (if you control the code and can call `sd_notify`) or `Type=exec`. `Type=simple` is the default but only correct if the service has no meaningful startup sequence.

## Dependencies and ordering — the distinction that trips everyone

These are **orthogonal**. Keep them straight:

**Dependency** (`Requires=`, `Wants=`, `BindsTo=`, `PartOf=`): "if A is active, so must be B." Changes what starts when you start A. Says nothing about order.

**Ordering** (`After=`, `Before=`): "when both A and B are starting, start B before A." Says nothing about whether either needs the other.

You almost always want both. "Start foo.service only after the network is up" needs _both_ `Requires=network.target` and `After=network.target`.

The flavors of dependency:

|Directive|Behavior|
|---|---|
|`Wants=X`|Pull in X when starting. If X fails, keep going anyway. Soft dependency.|
|`Requires=X`|Pull in X. If X fails to start, also fail. If X stops later, stop too.|
|`Requisite=X`|X must be _already_ active; don't pull it in. Fail if not.|
|`BindsTo=X`|Like `Requires=`, but also stops us if X stops for any reason (even cleanly). Tight coupling.|
|`PartOf=X`|Stop/restart signals to X propagate to us. One-way version of `BindsTo`.|
|`Conflicts=X`|Starting us stops X, and vice versa.|

**Inverse directives** also exist and are surprisingly useful: `WantedBy=`, `RequiredBy=`. These appear in `[Install]` sections to say "when this other unit is enabled, pull me in" — the main mechanism for services attaching to `multi-user.target`.

## Targets: the closest thing to runlevels

Targets are units that don't "do" anything — they're synchronization points. You start a target, and the dependency graph pulls in everything that should be active at that state.

The classic ones:

| Target                                                | What it means                                                                                                     |
| ----------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| `default.target`                                      | What boot aims for (usually an alias for `graphical.target` or `multi-user.target`)                               |
| `multi-user.target`                                   | Multi-user text-mode system (roughly old runlevel 3)                                                              |
| `graphical.target`                                    | GUI-enabled system (roughly old runlevel 5)                                                                       |
| `basic.target`                                        | Mounts, sockets, basic services ready                                                                             |
| `sysinit.target`                                      | Early boot: filesystems mounted, swap on                                                                          |
| `network-online.target`                               | "The network is actually usable" (as opposed to `network.target`, which just means "network stack is configured") |
| `rescue.target`                                       | Single-user mode with a root shell                                                                                |
| `emergency.target`                                    | Absolute minimum: root FS mounted read-only, shell, nothing else                                                  |
| `shutdown.target`, `reboot.target`, `poweroff.target` | Shutdown states                                                                                                   |
| `suspend.target`, `hibernate.target`                  | Sleep states                                                                                                      |

Change the default target:

```bash
sudo systemctl set-default multi-user.target   # boot to text mode
sudo systemctl set-default graphical.target    # boot to GUI
```

Switch immediately (like old `init 3`):

```bash
sudo systemctl isolate multi-user.target
```

`isolate` means "activate this target and stop everything not needed by it" — the runlevel-change analog.

## The three-part life cycle: enable, start, running

There are two orthogonal axes you need to hold in your head:

- **Is this unit _enabled?_** — will it start automatically when its `WantedBy=` target starts? (Lives in `[Install]` section, takes effect via symlinks in `*.wants/` directories.)
- **Is this unit _active?_** — is it running right now?

Four combinations, all valid:

|Enabled|Active|Meaning|
|---|---|---|
|Yes|Yes|Normal running service|
|Yes|No|Enabled but stopped (unusual — failed? explicitly stopped?)|
|No|Yes|Running now, won't start next boot|
|No|No|Dormant|

The commands:

```bash
systemctl start foo.service      # make active now
systemctl stop foo.service       # make inactive now
systemctl enable foo.service     # start automatically at boot
systemctl disable foo.service    # don't start at boot
systemctl enable --now foo.service    # both
systemctl disable --now foo.service   # both
systemctl restart foo.service
systemctl reload foo.service     # if ExecReload= defined; otherwise restart
systemctl status foo.service     # state + recent logs + cgroup contents
```

`enable` works by creating symlinks in `/etc/systemd/system/*.wants/` (or wherever the unit's `WantedBy=` points). `disable` removes them. This is why unit files are declarative: enabling is a graph modification, not a runtime action.

## Inspecting state

```bash
systemctl list-units                       # active units
systemctl list-units --all                 # including inactive but loaded
systemctl list-units --type=service        # filter
systemctl list-units --failed              # just the broken ones
systemctl list-unit-files                  # all units on disk, enabled/disabled status

systemctl status foo.service               # most useful single command
systemctl cat foo.service                  # the effective unit file (with overrides applied)
systemctl show foo.service                 # every single property (verbose)
systemctl show foo.service -p MainPID      # one property

systemctl list-dependencies foo.service    # what foo pulls in
systemctl list-dependencies --reverse foo.service   # what pulls in foo

systemctl is-active foo.service            # scripting-friendly
systemctl is-enabled foo.service
systemctl is-failed foo.service
```

`systemctl status` is the one to memorize. It gives you: load state, active state, substate, enabled status, the ExecStart command, main PID, memory/CPU/task counts, cgroup tree with all processes, and the last ~10 log lines. When a service misbehaves, this is your first stop.

## Tutorial: writing your first service

Say you want to run a Python script at boot, with auto-restart on failure, and logs going to the journal.

**1. Write the script** (`/usr/local/bin/myapp`):

```python
#!/usr/bin/env python3
import time, sys
while True:
    print("working...", flush=True)
    time.sleep(10)
```

```bash
sudo chmod +x /usr/local/bin/myapp
```

**2. Create the unit** at `/etc/systemd/system/myapp.service`:

```ini
[Unit]
Description=My example service
After=network-online.target
Wants=network-online.target

[Service]
Type=exec
ExecStart=/usr/local/bin/myapp
Restart=on-failure
RestartSec=5s

# Run as an unprivileged user
DynamicUser=yes

# Security hardening — start with these, add more as needed
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
NoNewPrivileges=yes

[Install]
WantedBy=multi-user.target
```

**3. Activate:**

```bash
sudo systemctl daemon-reload             # tell systemd a new/changed unit exists
sudo systemctl enable --now myapp.service
systemctl status myapp.service
journalctl -u myapp.service -f           # follow the logs
```

**4. Iterate:** edit the unit, then:

```bash
sudo systemctl daemon-reload
sudo systemctl restart myapp.service
```

Forgetting `daemon-reload` after editing a unit is the #1 gotcha. Changes don't take effect until systemd re-parses the file.

**A few design decisions in that unit worth noting:**

- `Type=exec` (not `simple`) — guarantees the binary is actually runnable before considering us started.
- `Restart=on-failure` — auto-restart only on non-zero exit or signal death, not when we exit cleanly.
- `DynamicUser=yes` — systemd creates a transient user at start, destroys it at stop. Good for services that don't need a persistent UID.
- `ProtectSystem=strict`, `ProtectHome=yes`, `PrivateTmp=yes`, `NoNewPrivileges=yes` — sandboxing flags. Start restrictive; loosen only when something breaks.

## Drop-ins: the right way to modify vendor units

You should almost never edit a file in `/usr/lib/systemd/system/` — updates will overwrite it. Instead, use **drop-ins**: override fragments that layer on top of the original.

```bash
sudo systemctl edit sshd.service
```

This opens an editor for `/etc/systemd/system/sshd.service.d/override.conf`. Put only what you want to change:

```ini
[Service]
Environment="SSHD_EXTRA_OPTS=-p 2222"
```

Save, exit — systemd reloads automatically. `systemctl cat sshd.service` shows you the effective composition (vendor unit + overrides).

For fields that _append_ vs. _replace_, there's a special rule: most list-like fields (like `ExecStart=`) **replace** the previous value. To clear and reset, you write the field empty first:

```ini
[Service]
ExecStart=
ExecStart=/usr/sbin/sshd -D -p 2222
```

Without the empty line, systemd refuses (duplicate `ExecStart=`). This is a common gotcha.

Related:

```bash
sudo systemctl edit --full sshd.service    # edit the whole unit (copies to /etc)
sudo systemctl revert sshd.service         # undo all edits, back to vendor
```

## Timers: cron, but with all of systemd's features

A timer is a unit that triggers another unit on a schedule. Two files: the `.timer` and the `.service` it activates.

**`/etc/systemd/system/backup.service`:**

```ini
[Unit]
Description=Nightly backup

[Service]
Type=oneshot
ExecStart=/usr/local/bin/do-backup
```

**`/etc/systemd/system/backup.timer`:**

```ini
[Unit]
Description=Run nightly backup

[Timer]
OnCalendar=daily
Persistent=true
RandomizedDelaySec=15m

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl enable --now backup.timer
systemctl list-timers                        # see when things will run
```

Why timers over cron:

- **Unified logging.** Output goes to the journal; you can `journalctl -u backup.service` to see every run's logs.
- **Dependency-aware.** The service can `Requires=network-online.target`, and the timer won't fire if dependencies aren't met.
- **`Persistent=true`** — if the machine was off at the scheduled time, run on next boot (cron just skips).
- **Resource limits via the service.** MemoryMax, CPU weight, etc.
- **Runs as a unit**, inheriting all the hardening you put in the service.

Calendar syntax (`OnCalendar=`) is expressive: `daily`, `weekly`, `*-*-* 03:00:00`, `Mon..Fri 09:00`, `2026-*-01`. `systemd-analyze calendar '<expr>'` validates and shows the next N occurrences.

Alternative triggers: `OnBootSec=`, `OnStartupSec=`, `OnActiveSec=`, `OnUnitActiveSec=`, `OnUnitInactiveSec=`. Useful for "run 5 minutes after boot" or "run every hour, measured from when the last run finished."

## Socket activation: the clever part

A `.socket` unit owns a listening port/path. When something connects, systemd starts the corresponding `.service` and hands it the socket.

Why this is a big deal:

- **Services don't need to be running continuously** — they start on demand.
- **Boot ordering becomes simpler.** A client doesn't need the server to be up; it just connects to the socket, and systemd brings the server up.
- **Zero-downtime restarts.** systemd holds the socket across service restarts, so clients don't see a gap.

Example — `/etc/systemd/system/echo.socket`:

```ini
[Unit]
Description=Echo socket

[Socket]
ListenStream=1234
Accept=no

[Install]
WantedBy=sockets.target
```

And `/etc/systemd/system/echo.service`:

```ini
[Unit]
Description=Echo service
Requires=echo.socket

[Service]
ExecStart=/usr/local/bin/echo-server
StandardInput=socket
```

Connect to port 1234, systemd starts `echo.service` with the socket passed as fd 3 (or stdin if `StandardInput=socket`). This is how modern sshd's socket activation works, how `cups.socket` + `cups.service` let printing daemons be lazy-loaded, etc.

## Path units: react to filesystem changes

A `.path` unit watches a path and triggers a service when something changes. Useful for "process files as they appear":

```ini
# /etc/systemd/system/incoming.path
[Path]
PathExistsGlob=/var/spool/incoming/*.txt

[Install]
WantedBy=multi-user.target
```

```ini
# /etc/systemd/system/incoming.service
[Service]
Type=oneshot
ExecStart=/usr/local/bin/process-incoming
```

Drop a `.txt` file in `/var/spool/incoming/`, systemd starts `incoming.service`.

## Mount and automount units

You can write mounts as units instead of `/etc/fstab` entries:

```ini
# /etc/systemd/system/mnt-data.mount
# File name MUST match the mount path: / becomes -, so /mnt/data → mnt-data.mount

[Mount]
What=/dev/disk/by-uuid/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
Where=/mnt/data
Type=ext4
Options=defaults,noatime
```

Or pair it with an `.automount` for lazy mounting:

```ini
# /etc/systemd/system/mnt-data.automount
[Automount]
Where=/mnt/data
TimeoutIdleSec=30min

[Install]
WantedBy=multi-user.target
```

Mount on first access, unmount after 30 min of idle. Handy for rarely-touched network shares.

For most users, `/etc/fstab` is still fine — systemd generates equivalent `.mount` units from it at boot (`systemd-fstab-generator`). But for conditional or complex mounts, unit files are more powerful.

## cgroups and resource control

Every service runs inside a cgroup (cgroup v2 on modern systems). This gives systemd two superpowers:

**Reliable process tracking.** PIDs can be forged and reused; cgroup membership can't. `systemctl status` shows you the cgroup tree — every process the service has spawned, including daemonized grandchildren — and `systemctl kill foo.service` gets them all.

**Resource limits.** You can set these directly in unit files:

```ini
[Service]
MemoryMax=2G
MemoryHigh=1.5G          # soft limit; throttles before hitting MemoryMax
CPUWeight=50             # relative share, default 100
CPUQuota=25%             # hard cap
IOWeight=50
TasksMax=100             # max number of tasks (processes + threads)
```

Browse the live cgroup tree:

```bash
systemd-cgls                      # tree of cgroups with process names
systemd-cgtop                     # top-style view, sortable by CPU/mem/IO
```

Slices let you group services for collective limits:

```bash
systemctl status user.slice       # all user processes
systemctl status system.slice     # all system services
systemctl status machine.slice    # containers/VMs managed by systemd
```

A service's cgroup path reflects its slice: `/sys/fs/cgroup/system.slice/nginx.service/`.

## Logging: journalctl

`systemd-journald` catches everything a service writes to stdout/stderr (and structured `sd_journal_send` calls), stores it in a binary indexed journal, and serves it up via `journalctl`.

```bash
journalctl                          # everything, pager
journalctl -u sshd.service          # one unit
journalctl -u sshd -u NetworkManager     # multiple
journalctl -f                       # follow (tail -f)
journalctl -u foo -f                # follow one unit
journalctl -b                       # this boot
journalctl -b -1                    # previous boot
journalctl --list-boots             # see boot IDs
journalctl --since "2 hours ago"
journalctl --since yesterday --until "1 hour ago"
journalctl -p err                   # priority: emerg, alert, crit, err, warning, notice, info, debug
journalctl -p err..alert            # range
journalctl -k                       # kernel messages only (like dmesg)
journalctl -o json-pretty           # structured output
journalctl _UID=1000                # match a journal field
journalctl /usr/bin/sshd            # binary-based filter
```

**Persistence:** by default, Fedora persists the journal to `/var/log/journal/`. If yours isn't and you want it:

```bash
sudo mkdir -p /var/log/journal
sudo systemd-tmpfiles --create --prefix /var/log/journal
sudo systemctl restart systemd-journald
```

**Disk usage:**

```bash
journalctl --disk-usage
sudo journalctl --vacuum-size=500M
sudo journalctl --vacuum-time=2weeks
```

Size limits go in `/etc/systemd/journald.conf`: `SystemMaxUse=`, `SystemKeepFree=`, etc.

## User services

systemd also runs per-user, for your session:

```bash
systemctl --user list-units
systemctl --user enable --now my-script.service
```

Units live in `~/.config/systemd/user/`. Great for personal automations: syncthing, custom scripts, language servers. No root needed; no fighting with `sudo`.

One caveat: user-level units stop when you log out unless you enable **lingering** for your user:

```bash
sudo loginctl enable-linger $USER
```

Now your user services keep running across logouts.

## Security hardening

Unit files have dozens of directives to sandbox services. Starting points (add these to your service's `[Service]` block progressively):

```ini
NoNewPrivileges=yes              # can't gain privileges via setuid, etc.
ProtectSystem=strict             # / is read-only except /var, /etc, /run
ProtectHome=yes                  # /home, /root, /run/user invisible
PrivateTmp=yes                   # private /tmp and /var/tmp
PrivateDevices=yes               # no /dev except core nodes
ProtectKernelTunables=yes        # /proc/sys, /sys read-only
ProtectKernelModules=yes         # can't load/unload modules
ProtectControlGroups=yes         # /sys/fs/cgroup read-only
RestrictAddressFamilies=AF_UNIX AF_INET AF_INET6    # whitelist socket types
RestrictNamespaces=yes           # can't create namespaces
LockPersonality=yes              # can't change execution domain
MemoryDenyWriteExecute=yes       # no W+X memory (thwarts many exploits)
SystemCallFilter=@system-service # seccomp whitelist
CapabilityBoundingSet=           # drop all capabilities (add only what you need)
```

Audit a unit's hardening:

```bash
systemd-analyze security foo.service
```

It scores 0–10 (0 = most secure, 10 = "exposed"), explaining each setting and suggesting improvements.

## Analyzing boot

```bash
systemd-analyze                           # total boot time, breakdown
systemd-analyze blame                     # services ranked by startup time
systemd-analyze critical-chain            # what was on the critical path
systemd-analyze plot > boot.svg           # visual timeline
systemd-analyze dot | dot -Tsvg > deps.svg  # dependency graph (huge)

systemd-analyze verify /etc/systemd/system/myapp.service   # lint a unit
systemd-analyze calendar 'Mon..Fri *-*-* 09:00:00'         # validate a calendar expr
systemd-analyze timespan '2h 30min'                        # parse a time
```

When boot feels slow, `blame` and `critical-chain` are your first stops. Slow services that aren't on the critical path don't actually delay boot — that's what `critical-chain` filters for.

## Common tasks and their commands

| Task                          | Command                                                                                                   |
| ----------------------------- | --------------------------------------------------------------------------------------------------------- |
| See what's broken             | `systemctl --failed`                                                                                      |
| Restart a misbehaving service | `systemctl restart foo`                                                                                   |
| Watch logs live               | `journalctl -u foo -f`                                                                                    |
| See why it failed             | `systemctl status foo` then `journalctl -xeu foo`                                                         |
| Start fresh at boot           | `systemctl enable --now foo`                                                                              |
| Disable permanently           | `systemctl disable --now foo && systemctl mask foo` (mask prevents accidental activation by dependencies) |
| Edit a unit safely            | `systemctl edit foo`                                                                                      |
| Reload after editing a file   | `systemctl daemon-reload`                                                                                 |
| Change boot target            | `systemctl set-default multi-user.target`                                                                 |
| Boot to rescue                | add `systemd.unit=rescue.target` to kernel cmdline                                                        |
| Reboot / shutdown             | `systemctl reboot` / `systemctl poweroff`                                                                 |
| Suspend / hibernate           | `systemctl suspend` / `systemctl hibernate`                                                               |

`mask` is worth knowing: it symlinks the unit to `/dev/null`, making it un-startable by any mechanism until you `unmask` it. Harsher than `disable`, useful for truly killing something off.

## Interactions with everything else

- **[[udev]]:** tags devices with `TAG+="systemd"` to create `.device` units. `ENV{SYSTEMD_WANTS}=...` in a rule activates a service on device presence.
- **[[D-Bus]]:** systemd is heavily D-Bus-driven. `systemctl` is a D-Bus client; most operations go over `org.freedesktop.systemd1`.
- **logind:** `systemd-logind` manages user sessions and seats. `loginctl` is its CLI. Handles the "who is logged in, which session is active, what hardware access do they get" questions.
- **networkd, resolved, timesyncd, homed:** optional systemd components. On Fedora Workstation, you have `resolved` (DNS stub at 127.0.0.53) but `NetworkManager` instead of `networkd`, and usually `chrony` instead of `timesyncd`. You can mix.
- **Containers:** `systemd-nspawn` is a lightweight container runtime. `machinectl` manages machines. Not commonly used directly, but podman can generate `.service` files for containers via `podman generate systemd`, integrating containers with the host's service model.
- **Portable services:** `portablectl` attaches service images (like a lightweight package format) as units. Niche but interesting for shipping services as immutable trees.

## The philosophical reality

systemd is opinionated. Its model — declarative units, cgroup-based tracking, integrated logging, dependency graphs, pervasive D-Bus — is _different_ from the classical Unix "each tool does one thing" aesthetic. People who value that aesthetic often find systemd overreaching; people who value consistency and reliability find it a massive improvement.

Practically: when you're fighting systemd, it's usually because you're pattern-matching to old init-script habits. The new model rewards leaning in — write proper units with proper dependencies, use socket activation, use timers instead of cron, use the journal, use drop-ins. Systems built this way are remarkably legible; you can inspect any running process's full context in seconds with `systemctl status`.

## Debugging flow

When something's wrong:

1. **`systemctl --failed`** — what's broken?
2. **`systemctl status foo.service`** — state, PID, cgroup, recent log lines.
3. **`journalctl -xeu foo.service`** — full logs for the unit with explanatory messages.
4. **`systemctl cat foo.service`** — what does the effective unit file say?
5. **`systemctl list-dependencies foo.service`** — what does it depend on?
6. **`systemd-analyze verify`** — syntactic/semantic errors in the unit.
7. **`SYSTEMD_LOG_LEVEL=debug systemctl restart foo`** — verbose systemd internals for one command.
8. **`systemctl show foo.service`** — every property, when `cat` doesn't show enough.

For boot-time issues: boot with `systemd.log_level=debug` on the kernel cmdline, or `systemd.journald.forward_to_console=1` to see boot logs on the console when the display is broken.

## Quick reference card

```bash
# Units: inspection
systemctl status <unit>
systemctl cat <unit>
systemctl show <unit> [-p Property]
systemctl list-units [--all] [--failed] [--type=service]
systemctl list-unit-files
systemctl list-dependencies [--reverse] <unit>

# Units: control
systemctl start|stop|restart|reload <unit>
systemctl enable|disable [--now] <unit>
systemctl mask|unmask <unit>
systemctl isolate <target>
systemctl reboot|poweroff|suspend|hibernate

# Authoring
systemctl edit <unit>                 # drop-in override
systemctl edit --full <unit>          # edit copy in /etc
systemctl revert <unit>                # undo edits
systemctl daemon-reload                # after editing files manually

# Logs
journalctl -u <unit> [-f] [-b] [--since ...] [-p err]
journalctl -k                          # kernel
journalctl --disk-usage
sudo journalctl --vacuum-time=2weeks

# Boot analysis
systemd-analyze
systemd-analyze blame
systemd-analyze critical-chain
systemd-analyze security <unit>
systemd-analyze verify <unit-file>

# cgroups / resources
systemd-cgls
systemd-cgtop
systemctl show <unit> -p MemoryCurrent,CPUUsageNSec

# User services
systemctl --user ...
loginctl enable-linger $USER           # let user services run without login

# Timers
systemctl list-timers
systemd-analyze calendar '<expr>'
```

## Where to learn more

- **`man systemd.unit`, `man systemd.service`, `man systemd.timer`, `man systemd.socket`** — the reference. Each section is tight.
- **`man systemd.exec`, `man systemd.resource-control`** — the massive property catalogs for `[Service]`.
- **`man systemd.directives`** — an index of every directive, pointing to its man page.
- **Lennart Poettering's systemd-for-admins blog series** (old but foundational — explains the _why_).
- **`systemctl cat` on a few vendor services** — read how Fedora's maintainers write real units. Great pedagogy.

The mental model to carry away: **systemd is a dependency-resolving state engine over typed units, with cgroup-based process tracking, a unified log stream, and D-Bus as the control plane.** Learn to read and write units, learn the service/timer/socket trio, get fluent with `systemctl status` and `journalctl -u`, and the rest unfolds naturally. Most mysteries on a modern Linux box dissolve into a five-second `systemctl status` away.