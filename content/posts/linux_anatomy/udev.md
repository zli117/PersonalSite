---
date: '2026-04-19T20:59:03-07:00'
draft: true
title: 'Udev'
---
## What udev is and why it exists

Before udev, Linux had two bad options for device files under `/dev`: ship a giant static directory containing every conceivable device node (`MAKEDEV`-style, thousands of entries, most unused), or use `devfs` — an in-kernel filesystem that auto-created nodes but imposed its own naming scheme and was impossible to customize flexibly from userspace.

**udev** replaced both. The deal: the kernel knows when devices appear and disappear, but all _policy_ about those devices — their names, permissions, symlinks, which daemons get notified — lives in userspace, driven by rules you can read and modify. `/dev` becomes a `devtmpfs` or `tmpfs` populated dynamically as devices come and go.

On modern Linux (anything with [[systemd]]), udev is **systemd-udevd**, a component of systemd. Same concepts, same rule syntax as the original standalone udev, just integrated.

**Its three jobs:**

1. **Populate `/dev`** — create device nodes when hardware appears, remove them when it leaves.
2. **Name devices predictably** — map `/sys/devices/pci0000:00/.../net/wlan0` to a stable, human-friendly name like `wlp194s0`.
3. **Dispatch events** — notify other daemons (NetworkManager, systemd, desktop environments) that something changed, so they can react.

## The event flow, end to end

When hardware appears (or at boot, when the kernel enumerates existing hardware):

```
┌─────────────┐     uevent       ┌─────────────────┐     processed event    ┌──────────────┐
│   kernel    │ ───────────────▶ │ systemd-udevd   │ ──────────────────────▶ │  listeners   │
│  (driver)   │  netlink mcast   │  (applies rules)│   netlink mcast         │ (NM, etc.)   │
└─────────────┘                  └────────┬────────┘                         └──────────────┘
                                          │
                                          ▼
                                   creates /dev node,
                                   sets permissions,
                                   adds symlinks,
                                   triggers helpers
```

The kernel emits a **uevent** — a small structured message with `ACTION=add`, a `DEVPATH=/devices/...`, the subsystem, and a bundle of environment-variable-style properties. udevd receives it, matches it against rules, applies whatever the rules say, then re-emits a processed event that other daemons subscribe to.

**Key idea:** udev is stateless and reactive. It doesn't "know" what devices exist; it reacts to kernel events. At boot, systemd-udevd triggers synthetic `add` events for everything already present in sysfs (`udevadm trigger`), which looks to the rules exactly like hotplug.

## Inspecting what udev sees

This is the most important skill. Before you write a rule, you need to know what attributes your device has. `udevadm info` is the tool.

```bash
# All attributes of a device, walking up the sysfs tree to its parents
udevadm info --attribute-walk /sys/class/net/wlp194s0

# Just the properties udev sees for it
udevadm info --query=property /sys/class/net/wlp194s0

# Given a /dev node, show its sysfs path
udevadm info --query=path --name=/dev/sda
```

The `--attribute-walk` output is worth understanding deeply. You'll see something like:

```
  looking at device '/devices/pci0000:00/.../net/wlp194s0':
    KERNEL=="wlp194s0"
    SUBSYSTEM=="net"
    DRIVER==""
    ATTR{address}=="a4:b1:c1:xx:xx:xx"
    ATTR{type}=="1"

  looking at parent device '/devices/pci0000:00/0000:00:1c.0/0000:01:00.0':
    KERNELS=="0000:c2:00.0"
    SUBSYSTEMS=="pci"
    DRIVERS=="mt7925e"
    ATTRS{vendor}=="0x14c3"
    ATTRS{device}=="0x7925"
```

Note the plurals: `KERNEL`/`SUBSYSTEM`/`ATTR` refer to the device itself; `KERNELS`/`SUBSYSTEMS`/`ATTRS`/`DRIVERS` match any ancestor in the tree. You use the plurals when you want to match, say, "any device whose PCI parent is vendor 0x14c3." This is the main technique for writing rules that are specific without being fragile.

**Watching events live:**

```bash
udevadm monitor                # kernel uevents + processed udev events
udevadm monitor --property     # same, but show all properties
udevadm monitor --udev --subsystem-match=net    # filter
```

Plug and unplug something while this runs — you'll see the whole event stream. This is invaluable for debugging.

## Where rules live

```
/usr/lib/udev/rules.d/    # distro-shipped rules. Don't edit.
/etc/udev/rules.d/        # your rules. Overrides distro rules with same filename.
/run/udev/rules.d/        # ephemeral, rarely used directly.
```

Files are named `NN-description.rules` where NN is a two-digit priority (lower numbers run first). Look through `/usr/lib/udev/rules.d/` once to see real-world examples — that's where predictable interface naming, input device permissions, and so on are defined.

The file `73-seat-late.rules`, `99-systemd.rules`, `80-net-setup-link.rules` are good ones to read.

## Rule syntax

A rule is one line. Comments start with `#`. Each rule is a sequence of comma-separated **key-op-value** terms:

```
KEY=="match_value", KEY=="match_value", ..., KEY="assign_value", KEY+="append_value"
```

**Operators split into two camps:**

|Operator|Meaning|
|---|---|
|`==`|match (equal)|
|`!=`|match (not equal)|
|`=`|assign|
|`+=`|append to list|
|`:=`|assign and lock (subsequent rules can't override)|

A rule executes its assignment side only if **all** the match-side terms succeed. Matching uses shell-glob wildcards (`*`, `?`, `[abc]`), not regex.

**Common match keys:**

|Key|What it matches|
|---|---|
|`ACTION`|`add`, `remove`, `change`, `bind`, `unbind`|
|`KERNEL`|kernel-assigned name (e.g., `sda`, `wlan0`)|
|`SUBSYSTEM`|`block`, `net`, `tty`, `input`, `usb`, etc.|
|`ATTR{name}`|sysfs attribute on the device itself|
|`ENV{name}`|an environment property set by kernel or earlier rules|
|`KERNELS`, `SUBSYSTEMS`, `ATTRS{name}`, `DRIVERS`|same, but match any ancestor|
|`TAG`|tags set by earlier rules or the kernel|

**Common assignment keys:**

|Key|Effect|
|---|---|
|`NAME`|set the device name (rare now — mostly for net ifaces)|
|`SYMLINK`|create a symlink in `/dev` (use `+=`)|
|`OWNER`, `GROUP`, `MODE`|set ownership and permissions on the `/dev` node|
|`ENV{name}`|set a property for later rules or exported to listeners|
|`TAG`|tag the device (e.g., `TAG+="systemd"` makes it a `.device` unit)|
|`RUN`|run a program (use sparingly; see below)|
|`LABEL`, `GOTO`|flow control within rules|
|`IMPORT{type}`|import properties from a program, file, parent, or builtin|

## Tutorial: rule examples you can actually use

### Example 1: stable symlink for a USB serial device

Problem: you have a USB-serial adapter (e.g., a Flipper Zero, an ESP32 dev board) that shows up as `/dev/ttyUSB0` or `/dev/ttyACM0`, but the number changes depending on plug order. You want a stable symlink.

First, find the device's attributes:

```bash
udevadm info --attribute-walk --name=/dev/ttyACM0 | less
```

Look for something unique — `ATTRS{idVendor}`, `ATTRS{idProduct}`, and ideally `ATTRS{serial}` if the device has a serial number.

Create `/etc/udev/rules.d/99-flipper.rules`:

```
SUBSYSTEM=="tty", ATTRS{idVendor}=="0483", ATTRS{idProduct}=="5740", ATTRS{serial}=="flip_MyFlipper", SYMLINK+="flipper", MODE="0660", GROUP="dialout"
```

Reload and test:

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger                    # re-run rules on existing devices
ls -l /dev/flipper                      # should be a symlink
```

Now scripts can reference `/dev/flipper` regardless of enumeration order.

### Example 2: permissions for a hardware device

Problem: you want your user to be able to access a Yubikey or an FTDI adapter without sudo.

```
SUBSYSTEM=="usb", ATTRS{idVendor}=="1050", MODE="0660", GROUP="plugdev", TAG+="uaccess"
```

The `TAG+="uaccess"` is the modern way: systemd-logind will grant ACL access to whichever user is on the active seat. That's how your desktop session can suddenly use a USB key without group membership. For headless/server use, fall back to `GROUP=` + user in that group.

### Example 3: rename a network interface

Problem: you have two USB WiFi dongles and want stable names `wifi-ap` and `wifi-mon` based on MAC.

```
SUBSYSTEM=="net", ACTION=="add", ATTR{address}=="aa:bb:cc:dd:ee:01", NAME="wifi-ap"
SUBSYSTEM=="net", ACTION=="add", ATTR{address}=="aa:bb:cc:dd:ee:02", NAME="wifi-mon"
```

Two caveats for net renaming specifically:

- Must happen at `add` time before the iface is used. Predictable naming (`80-net-setup-link.rules`) runs early; make sure your file sorts before anything that uses the iface, or use a higher priority (lower number).
- Better modern approach: use a **.link** file under `/etc/systemd/network/` with `[Match] MACAddress=` and `[Link] Name=`. Cleaner integration with systemd-networkd/NM, no rules debugging.

### Example 4: auto-mount a specific USB drive

Problem: whenever you plug your backup drive in, you want a script to run.

```
SUBSYSTEM=="block", ACTION=="add", ENV{ID_FS_UUID}=="1234-abcd", RUN+="/usr/local/bin/backup-on-connect"
```

**Big warning on `RUN`:** udev rules run synchronously in udevd's workers, with a short timeout. The rule **is not meant to run long programs**. If the backup script takes more than a few seconds, udev kills it. The correct pattern for nontrivial work:

```
SUBSYSTEM=="block", ACTION=="add", ENV{ID_FS_UUID}=="1234-abcd", TAG+="systemd", ENV{SYSTEMD_WANTS}="backup-on-connect.service"
```

This tags the device as a systemd-managed device and starts `backup-on-connect.service` when it appears. The service runs outside udev's context, with normal systemd lifecycle (logging, restart policy, timeouts you control, cgroup). **Always prefer this pattern over `RUN` for anything nontrivial.**

### Example 5: conditional properties

Problem: you want to mark certain input devices as "gaming" so other rules can match.

```
SUBSYSTEM=="input", ATTRS{idVendor}=="046d", ATTRS{idProduct}=="c24f", ENV{MY_GAMING}="1"
SUBSYSTEM=="input", ENV{MY_GAMING}=="1", TAG+="uaccess"
```

Rules are evaluated in file-sort order, so the first rule sets the property and the second matches on it.

## The testing loop

Iterate like this:

```bash
# 1. Edit your rule
sudoedit /etc/udev/rules.d/99-myrule.rules

# 2. Check syntax and see what it would do for a specific device
sudo udevadm test /sys/class/net/wlp194s0 2>&1 | less

# 3. Reload rules into the running udevd
sudo udevadm control --reload-rules

# 4. Re-trigger events on matching devices (no replug needed)
sudo udevadm trigger --subsystem-match=net --action=add

# 5. Verify
udevadm info --query=property /sys/class/net/wlp194s0
```

`udevadm test` is the killer tool — it simulates rule evaluation for a device you specify and prints every rule that matched, every assignment made, and the final property set. When a rule isn't doing what you expect, this tells you why.

## The [[sysfs]] mental model

udev rules feel magical until you grasp sysfs. It helps to remember:

- Every device the kernel knows about has a directory under `/sys/devices/...`.
- `/sys/class/<type>/` and `/sys/subsystem/<name>/` are **symlinks** organized by category, pointing into the real `/sys/devices/` tree.
- Each directory has files representing attributes (`vendor`, `model`, `address`, `power/control`). Reading them reads live kernel state; writing them (sometimes) changes it.
- The parent-child chain in sysfs reflects the physical/logical chain (PCI root port → PCI device → USB controller → USB hub → USB device → interface → endpoint).

`udevadm info --attribute-walk` simply walks up this chain and dumps attributes at each level. Once you're comfortable reading sysfs directly, udev rules make a lot more sense.

```bash
# Poke around sysfs
ls /sys/class/net/
cat /sys/class/net/wlp194s0/address
ls -l /sys/class/net/wlp194s0/device           # symlink to the PCI device
cat /sys/class/net/wlp194s0/device/vendor
```

## Hardware database (`hwdb`)

One more piece: udev maintains a **hardware database** that maps vendor/product IDs to human-readable names and extra properties. Files in `/usr/lib/udev/hwdb.d/` and `/etc/udev/hwdb.d/`. Compiled into a binary database by `systemd-hwdb update`.

This is how `lsusb`-style tools get names, and how, say, a specific keyboard can get `KEYBOARD_KEY_*` remappings shipped system-wide. You rarely need to add to it, but when you do — for custom keyboard scancode remapping, for example — it's a hwdb entry, not a rule:

```
# /etc/udev/hwdb.d/90-mykeyboard.hwdb
evdev:input:b0003v1234p5678*
  KEYBOARD_KEY_70029=esc
```

Then `sudo systemd-hwdb update && sudo udevadm trigger --subsystem-match=input`.

## Interactions with other subsystems

- **systemd**: rules with `TAG+="systemd"` create `.device` units. Services can declare `BindsTo=sys-subsystem-net-devices-wlp194s0.device` to start/stop with hardware presence. `ENV{SYSTEMD_WANTS}="..."` starts a service when the device appears (the clean way to "run something on hotplug").
- **NetworkManager**: subscribes to udev net events; its decision to manage/ignore an interface can be steered by rule-set properties like `NM_UNMANAGED`.
- **logind / uaccess**: `TAG+="uaccess"` grants the logged-in user ACL access — the mechanism behind "my webcam just works when I log in."
- **Polkit**: some polkit rules match on udev tags to allow session users to mount/unmount specific devices.

## Debugging flowchart

When a rule isn't behaving:

1. **Does the device look like you think?** `udevadm info --attribute-walk --name=<path>`. Verify the attributes you're matching actually exist at the level you're matching (device vs. parent — singular vs. plural keys).
2. **Is the rule even being considered?** `sudo udevadm test <sysfs-path>` and look for your rule in the trace.
3. **Is another rule overriding it?** Check file sort order in `/usr/lib/udev/rules.d/` + `/etc/udev/rules.d/`. Rename your file to a higher number to run later.
4. **Did you reload?** `sudo udevadm control --reload-rules` after editing.
5. **Did you re-trigger?** Existing devices don't see new rules until you `udevadm trigger` or replug.
6. **Watch it live:** `udevadm monitor --property` while you plug or trigger.
7. **Check journal:** `journalctl -u systemd-udevd` for errors.

## What udev is _not_ for

A few anti-patterns worth knowing:

- **Long-running processes in `RUN`.** Use `SYSTEMD_WANTS`.
- **Complex decision logic in rules.** Rules are declarative pattern-matching; past a certain complexity, call out to a helper program via `IMPORT{program}="/path/to/helper"` that returns `KEY=VALUE` lines.
- **Primary persistence for service state.** udev rules run on every boot for every matching device; if you need one-time setup, do it in a systemd service triggered by the device.
- **Network configuration.** udev names interfaces and tags them; it doesn't configure IPs or routes. That's NetworkManager/systemd-networkd.

## Quick reference card

```bash
# Inspect
udevadm info --query=property --name=/dev/sda
udevadm info --attribute-walk --name=/dev/sda
udevadm monitor --property
udevadm monitor --subsystem-match=usb

# Test a rule
sudo udevadm test /sys/class/net/wlp194s0

# Apply changes
sudo udevadm control --reload-rules
sudo udevadm trigger                        # re-run add events on everything
sudo udevadm trigger --subsystem-match=net  # narrower
sudo udevadm trigger --action=change /sys/class/net/wlp194s0

# Rule file locations
/usr/lib/udev/rules.d/      # distro (don't edit)
/etc/udev/rules.d/          # yours
```

## Where to learn more

- `man udev` — the canonical syntax reference. Short and readable.
- `man udevadm` — the tool.
- `man systemd.link` — the alternative `.link` mechanism for net/block device settings (cleaner than rules for that use case).
- `/usr/lib/udev/rules.d/` — read 5-10 of these. Best way to learn realistic patterns.
- `man hwdb` — for the hardware database side.

Once you have the mental model — _kernel emits events, udevd runs declarative rules keyed on sysfs attributes, result is dispatched to everyone who cares_ — udev stops feeling mysterious. It's just a rule engine on top of a well-designed event stream.
