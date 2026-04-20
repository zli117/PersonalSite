---
date: '2026-04-19T20:58:33-07:00'
draft: true
title: 'Chapter 2: sysfs'
weight: 20
---
## What sysfs is and why it exists

In classical Unix, everything was a file — except the kernel's internal state. By the 2.4-era Linux kernel, that state was leaking into userspace through an inconsistent mess: ioctls on random device files, `/proc` entries with ad-hoc text formats, sysctl syscalls, driver-specific character devices. If you wanted to know "what PCI devices are on this bus" or "what's this disk's queue depth", each driver had its own answer.

**sysfs**, introduced in 2.6 (around 2003), was the cleanup. A virtual filesystem, mounted at `/sys`, that exposes the kernel's **device model** — its internal map of buses, devices, drivers, and classes — as a structured directory tree. One consistent interface for userspace to inspect and (sometimes) modify kernel state.

The design rules, roughly:

- **One value per file.** `/sys/class/net/wlp194s0/address` contains exactly the MAC address and nothing else — not a formatted report. You `cat` it, you get one piece of data.
- **Text, not binary.** Numbers in decimal or hex ASCII, not packed structs. (A few exceptions like `config` on PCI.)
- **Directory structure reflects kernel structure.** The parent/child relationships are real — a USB device's directory is literally inside its controller's directory.
- **Symlinks for alternative views.** The same device appears at multiple paths (by type, by subsystem, by bus) via symlinks back to one canonical directory.

This discipline is why modern tools like `udev`, `lsblk`, `lspci`, `ip`, `ethtool`, `powertop`, and most of GNOME's hardware panels all work — they read sysfs instead of inventing a new [[ioctl]].

**sysfs vs procfs vs devtmpfs (the sibling you'll confuse it with):**

| Filesystem | Mount                 | Purpose                                                                                               |
| ---------- | --------------------- | ----------------------------------------------------------------------------------------------------- |
| `sysfs`    | `/sys`                | Structured kernel device/driver model. One value per file.                                            |
| `procfs`   | `/proc`               | Per-process info (`/proc/<pid>/*`) + legacy kernel state dumps. Often multi-line text.                |
| `devtmpfs` | `/dev`                | Device nodes (character/block). You open these and use read/write/ioctl.                              |
| `configfs` | `/sys/kernel/config`  | Like sysfs but you create directories to configure kernel objects (USB gadgets, iSCSI targets, etc.). |
| `debugfs`  | `/sys/kernel/debug`   | Kitchen sink for driver debugging. No stability guarantees.                                           |
| `tracefs`  | `/sys/kernel/tracing` | ftrace interface.                                                                                     |

People conflate "/proc vs /sys" often. Roughly: **`/proc` is "processes and legacy kernel info", `/sys` is "hardware and kernel objects, done right."** New kernel interfaces almost always pick sysfs.

## The top-level layout

```
/sys
├── block/           ← symlinks, one per block device (by name)
├── bus/             ← devices and drivers organized by bus type
├── class/           ← symlinks, by device class (net, tty, input, drm, ...)
├── dev/             ← symlinks by (major:minor) device number
├── devices/         ← THE REAL TREE. Everything else points here.
├── firmware/        ← firmware interfaces (EFI, ACPI, DMI)
├── fs/              ← filesystem-specific state (btrfs, cgroup, bpf, pstore)
├── hypervisor/      ← hypervisor info when virtualized
├── kernel/          ← kernel globals (debug, tracing, security, mm)
├── module/          ← loaded kernel modules and their parameters
└── power/           ← system-wide power management state
```

The single most important thing to internalize: **`/sys/devices/` is the real hierarchy.** Every other top-level directory is a view — a set of symlinks that organizes the same devices by some secondary axis (by bus, by class, by dev number). When you follow `/sys/class/net/wlp194s0`, it resolves to something like `/sys/devices/pci0000:00/0000:00:1c.0/0000:01:00.0/net/wlp194s0`.

This matters because navigating a device's _real_ location tells you its _physical topology_: which PCIe root port, which bridge, which slot. The views are for convenience; the tree is for truth.

## The real tree: `/sys/devices/`

Each device is a directory. The hierarchy mirrors the hardware:

```
/sys/devices/
├── pci0000:00/                         ← PCI host bridge (domain 0, bus 0)
│   ├── 0000:00:1c.0/                   ← PCIe root port (BDF: 00:1c.0)
│   │   ├── 0000:01:00.0/               ← device behind the root port
│   │   │   ├── vendor                  ← file: "0x14c3\n"
│   │   │   ├── device                  ← file: "0x7925\n"
│   │   │   ├── class                   ← file: "0x028000\n" (network controller)
│   │   │   ├── driver -> ../../../.../drivers/mt7925e    ← symlink to its driver
│   │   │   ├── net/
│   │   │   │   └── wlp194s0/           ← the netdev this device exposes
│   │   │   │       ├── address         ← file: "a4:b1:c1:xx:xx:xx\n"
│   │   │   │       ├── mtu             ← file: "1500\n"
│   │   │   │       ├── operstate       ← file: "up\n"
│   │   │   │       └── statistics/
│   │   │   └── ieee80211/
│   │   │       └── phy0/               ← the WiFi PHY this device provides
```

Two structural ideas to note:

**Parent-child reflects physical containment.** A USB device's directory literally lives inside its USB controller's directory, which lives inside the PCI device that is the controller, which lives inside the root bridge. You can walk up the tree with `..` and learn what the device is attached to, all the way to the CPU.

**Devices expose their "products."** A WiFi card doesn't just sit there — it produces a `net/` subdirectory (containing the netdev) and an `ieee80211/` subdirectory (containing the PHY). A SATA disk produces a `block/` subdirectory. The subdirectory names tell you what kinds of kernel objects this device registered.

## The views: `/sys/class`, `/sys/bus`, `/sys/block`, `/sys/dev`

These exist because different tools want different indexes into the same data.

**`/sys/class/<type>/`** — "give me everything of this kind." Each entry is a symlink to the real device in `/sys/devices`.

```bash
ls /sys/class/net/           # all network interfaces
ls /sys/class/block/         # all block devices
ls /sys/class/input/         # all input devices
ls /sys/class/drm/           # GPU-related stuff (cards, connectors)
ls /sys/class/power_supply/  # batteries and AC adapters
ls /sys/class/hwmon/         # temperature/fan/voltage sensors
ls /sys/class/tty/           # serial devices (real and virtual)
ls /sys/class/backlight/     # screen brightness controls
```

**`/sys/bus/<bustype>/`** — "organize by bus." Each bus has `devices/` and `drivers/` subdirectories.

```bash
ls /sys/bus/pci/devices/     # every PCI device, by BDF address
ls /sys/bus/pci/drivers/     # every PCI driver currently loaded
ls /sys/bus/usb/devices/     # USB devices, by bus-port address
ls /sys/bus/i2c/devices/     # I²C devices (lots on laptops)
```

**`/sys/block/`** — shortcut for `/sys/class/block/` but only for whole disks (not partitions).

**`/sys/dev/char/<maj>:<min>`, `/sys/dev/block/<maj>:<min>`** — look up a device by its `(major, minor)` number (the thing in `/dev` as `c 1 3` or `b 8 0`). Useful in scripts when you have a `stat` result.

All of these resolve to real locations under `/sys/devices/`:

```bash
readlink -f /sys/class/net/wlp194s0
# /sys/devices/pci0000:00/0000:00:1c.0/0000:01:00.0/net/wlp194s0

readlink -f /sys/block/nvme0n1
# /sys/devices/pci0000:00/0000:00:06.0/0000:02:00.0/nvme/nvme0/nvme0n1
```

The `device` symlink goes the other direction — from a netdev or partition back to the hardware device that owns it:

```bash
readlink -f /sys/class/net/wlp194s0/device
# /sys/devices/pci0000:00/0000:00:1c.0/0000:01:00.0
```

## Attributes: the files inside device directories

The files inside a device directory are **attributes** — one value per file. There are a few flavors:

- **Read-only status** — `address`, `vendor`, `device`, `class`, `speed`. You `cat` them.
- **Read-write controls** — `mtu`, `power/control`, `queue/scheduler` on a disk, `brightness` on a backlight. Writing changes the kernel's state.
- **Structured binary blobs** — `config` on a PCI device (256+ bytes of PCI config space), `eeprom` on some devices. `hexdump` these, don't `cat`.
- **Triggers** — write a specific value to take an action. Writing `1` to `/sys/bus/pci/rescan` re-enumerates PCI.
- **Attribute groups** — some devices expose sub-directories like `power/`, `queue/`, `statistics/` that group related attributes.

Reading is always safe. Writing requires root and may require knowing the exact accepted format.

**Find attributes for a device:**

```bash
ls /sys/class/net/wlp194s0/
#   address         broadcast       carrier         flags         mtu
#   operstate       phys_port_id    statistics/     ...

# Walk up the chain and list at every level
udevadm info --attribute-walk --name=/dev/sda
```

**Read one:**

```bash
cat /sys/class/net/wlp194s0/address
# a4:b1:c1:xx:xx:xx

cat /sys/class/power_supply/BAT0/capacity
# 87
```

**Write one (examples that actually matter in practice):**

```bash
# Change I/O scheduler for an NVMe drive
echo mq-deadline | sudo tee /sys/block/nvme0n1/queue/scheduler

# Set CPU governor
echo performance | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# Set screen brightness (value depends on max_brightness in same dir)
echo 800 | sudo tee /sys/class/backlight/amdgpu_bl1/brightness

# Suspend to idle
echo freeze | sudo tee /sys/power/state
```

Writes to sysfs are synchronous — the kernel processes them immediately and returns success/error. If you get `EINVAL`, the value was rejected; `EPERM`, you need root; `EACCES`, the attribute isn't writable.

## Tutorial: tasks you can actually do

### Task 1: find everything connected to your laptop's hardware

```bash
# List PCI devices with names (from hwdb)
lspci -tv

# Same, but from sysfs directly
for dev in /sys/bus/pci/devices/*; do
  echo "=== $(basename "$dev") ==="
  echo "  vendor:  $(cat "$dev/vendor")"
  echo "  device:  $(cat "$dev/device")"
  echo "  driver:  $(basename "$(readlink "$dev/driver" 2>/dev/null)")"
done

# USB topology
lsusb -t
# Or from sysfs:
ls /sys/bus/usb/devices/
```

Any tool that shows you hardware is ultimately reading sysfs. Understanding this lets you write small scripts for anything they don't expose.

### Task 2: which driver owns a device?

Every device in `/sys/devices` that has a driver bound has a `driver` symlink:

```bash
readlink /sys/bus/pci/devices/0000:01:00.0/driver
# ../../../../bus/pci/drivers/mt7925e
```

Reverse direction — which devices does a driver have bound?

```bash
ls /sys/bus/pci/drivers/mt7925e/
# 0000:01:00.0  bind  module  new_id  remove_id  uevent  unbind
```

The files there let you manually bind/unbind:

```bash
# Unbind a PCI device from its driver (rare, but sometimes needed for passthrough)
echo 0000:01:00.0 | sudo tee /sys/bus/pci/drivers/mt7925e/unbind

# Bind it back
echo 0000:01:00.0 | sudo tee /sys/bus/pci/drivers/mt7925e/bind
```

### Task 3: find the battery charge, power draw, and health

```bash
ls /sys/class/power_supply/
# AC  BAT0

cat /sys/class/power_supply/BAT0/capacity            # percent
cat /sys/class/power_supply/BAT0/status              # Charging / Discharging / Full
cat /sys/class/power_supply/BAT0/energy_now          # µWh remaining
cat /sys/class/power_supply/BAT0/energy_full         # µWh when full
cat /sys/class/power_supply/BAT0/energy_full_design  # µWh when new
cat /sys/class/power_supply/BAT0/power_now           # µW instantaneous draw
cat /sys/class/power_supply/BAT0/cycle_count         # charge cycles
```

That last ratio (`energy_full / energy_full_design`) is your battery's state of health. Every battery applet is basically doing this division.

### Task 4: read temperatures without installing anything

```bash
for sensor in /sys/class/hwmon/hwmon*; do
  name=$(cat "$sensor/name" 2>/dev/null)
  echo "=== $name ($(basename "$sensor")) ==="
  for t in "$sensor"/temp*_input; do
    [ -f "$t" ] || continue
    label_file="${t%_input}_label"
    label=$(cat "$label_file" 2>/dev/null || basename "$t")
    temp=$(( $(cat "$t") / 1000 ))
    echo "  $label: ${temp}°C"
  done
done
```

This is essentially what `sensors` from `lm_sensors` does. Values in hwmon are typically in **milli-units** (millidegrees, millivolts, microwatts) by convention — divide accordingly.

### Task 5: runtime power management for a USB device

```bash
# Find the device
lsusb
# Bus 001 Device 003: ID 046d:c08b Logitech mouse

# Find it in sysfs
grep -l "046d" /sys/bus/usb/devices/*/idVendor
# /sys/bus/usb/devices/1-2/idVendor

# Enable autosuspend (let it sleep when idle)
echo auto | sudo tee /sys/bus/usb/devices/1-2/power/control
# The other option is "on" (never autosuspend)

# See the current state
cat /sys/bus/usb/devices/1-2/power/runtime_status
# active | suspended | ...
```

`powertop --auto-tune` essentially walks `/sys/bus/*/devices/*/power/control` writing `auto` everywhere.

### Task 6: see what firmware your system has

```bash
# DMI (SMBIOS) information — vendor, product, BIOS version, serial
ls /sys/class/dmi/id/
cat /sys/class/dmi/id/sys_vendor
cat /sys/class/dmi/id/product_name
cat /sys/class/dmi/id/bios_version

# ACPI tables
ls /sys/firmware/acpi/tables/         # DSDT, SSDT, etc. as raw files

# EFI variables (if booted UEFI)
ls /sys/firmware/efi/efivars/
```

This is how `dmidecode`, `fwupd`, and various system-info tools work.

## Modifying sysfs: when and how

Writing to sysfs is the "poke" interface for kernel subsystems. A non-exhaustive tour of useful write targets:

| Path                                                    | Effect                                                |
| ------------------------------------------------------- | ----------------------------------------------------- |
| `/sys/power/state`                                      | write `freeze`, `mem`, `disk` to suspend              |
| `/sys/class/backlight/*/brightness`                     | screen brightness                                     |
| `/sys/class/leds/*/brightness`                          | LED control (caps lock, keyboard backlight)           |
| `/sys/block/*/queue/scheduler`                          | I/O scheduler (`none`, `mq-deadline`, `bfq`, `kyber`) |
| `/sys/block/*/queue/read_ahead_kb`                      | readahead size                                        |
| `/sys/bus/pci/rescan`                                   | rescan PCI bus (useful after hotplug)                 |
| `/sys/bus/usb/devices/*/authorized`                     | enable/disable a USB device (USBGuard-style)          |
| `/sys/devices/system/cpu/cpu*/online`                   | hot-unplug a CPU                                      |
| `/sys/devices/system/cpu/cpu*/cpufreq/scaling_governor` | CPU frequency governor                                |
| `/sys/class/net/*/mtu`                                  | change MTU (also doable via `ip link`)                |
| `/sys/module/<mod>/parameters/<p>`                      | change a module parameter (not all are writable)      |

**Persistence:** sysfs writes are in-memory; they don't survive reboot. To persist, you need:

- **A udev rule** for things tied to a device (permissions, power settings, I/O scheduler per disk).
- **A systemd service** for things you want at boot (`ExecStart=/bin/sh -c 'echo ... > ...'`).
- **A sysctl-style config** in `/etc/tmpfiles.d/` — `systemd-tmpfiles` can write to sysfs paths declaratively:
    
    ```
    # /etc/tmpfiles.d/99-cpufreq.confw /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor - - - - performance
    ```
    

## Poking around safely

A mental model for risk:

- **Reading anything is safe.** You won't break things by `cat`-ing sysfs files. The only exception is certain `debugfs` attributes that trigger action on read (rare).
- **Writing is usually safe if the kernel accepts the value.** The kernel validates. Bad values usually just give `EINVAL`.
- **Some writes reset or detach hardware.** `echo 0 > /sys/class/net/eth0/device/rescan` is harmless. `echo 1 > /sys/bus/pci/devices/XXX/remove` physically unbinds a device and can lock up a running system if you pick a system disk. Think before you write to `remove`, `unbind`, `reset`, or anything in `power/`.
- **debugfs (`/sys/kernel/debug/`) is the wild west.** No stability guarantees, vendor-specific, often undocumented. Great for digging into driver internals; don't script against it.

## How sysfs fits with everything else

- **udev** reads sysfs attributes to write rules. `ATTR{address}` and `ATTRS{vendor}` in a udev rule are literally grepping attribute files in sysfs. `udevadm info` is a structured dump of a device's sysfs tree.
- **NetworkManager** reads `/sys/class/net/*` to discover interfaces, and reads `operstate`, `carrier`, `speed`, etc. to track link state.
- **systemd** creates `.device` units from sysfs entries (tagged via udev). `systemctl list-units --type=device` shows you the units; each corresponds to a sysfs path.
- **Desktop environments** read `/sys/class/power_supply/*` for the battery applet, `/sys/class/backlight/*` for brightness sliders, `/sys/class/thermal/*` for thermal status.
- **Containers** often mount a restricted view of `/sys` (read-only, or only specific subtrees). Privileged containers can see and write it all, which is why "`--privileged`" is such a big deal.

## The read/write etiquette

A few conventions worth internalizing so sysfs stops surprising you:

- **Units are usually in the file name or obvious from context.** `temp1_input` is millidegrees Celsius. `energy_now` in `power_supply` is microwatt-hours. `brightness` is in whatever range `max_brightness` dictates.
- **Booleans are `0` / `1`.**
- **Enum-like attributes list their options** — e.g., `scaling_available_governors` next to `scaling_governor`.
- **Newlines.** Files typically end with `\n`. Use `echo`, not `printf '%s'`, unless the kernel is picky (very rare; `/sys/kernel/config/` is an exception).
- **No locking or atomicity across multiple files.** If you read `energy_now` and then `power_now`, they're two independent reads — the kernel's state may have moved. For coherent snapshots, you sometimes have to use the driver's netlink interface instead.

## Debugging flow

When you want to understand a piece of hardware or track down why something behaves strangely:

1. **Find its sysfs path.** `/sys/class/<type>/<name>` for the friendly name, then `readlink -f` for the real path.
2. **List its attributes.** `ls -la` in the directory. Read anything that looks relevant.
3. **Walk up the parents.** `cd ..` repeatedly, checking each level. You'll find the controller, the bus, the root complex. `udevadm info --attribute-walk` automates this.
4. **Check the driver.** `readlink driver` tells you what's bound. `cat driver/module/version` sometimes gives version info.
5. **Look at the `uevent` file.** Every device has one; it's a dump of the properties the kernel would emit in a uevent. Useful for scripting.
6. **Cross-reference to dmesg.** If the driver logs anything at bind/probe time, `dmesg | grep <driver>` connects kernel log messages to the device.

## Things sysfs is not

- **Not a config language.** You can't write "the desired state" and have sysfs diff it. Every write is an immediate action.
- **Not transactional.** No multi-file atomic updates.
- **Not versioned.** Attribute paths and names are an ABI the kernel _tries_ to keep stable, but drivers sometimes break this. Real-world tools hedge by checking for attribute existence.
- **Not for process data.** Use `/proc/<pid>/`.
- **Not for kernel tunables that aren't device-specific.** Those live in `/proc/sys/` (accessed via `sysctl`), not sysfs. Confusing, but historical.

## Quick reference card

```bash
# Discover
ls /sys/class/                                  # all device classes
ls /sys/class/net/                              # all network interfaces
ls /sys/bus/                                    # all bus types
ls /sys/bus/pci/devices/                        # PCI devices by BDF

# Resolve symlinks to real paths
readlink -f /sys/class/net/wlp194s0

# Follow a device back to its hardware
readlink -f /sys/class/net/wlp194s0/device
readlink -f /sys/block/nvme0n1/device

# Find a driver's devices
ls /sys/bus/pci/drivers/<drivername>/
ls /sys/bus/usb/drivers/<drivername>/

# Inspect attributes
cat /sys/class/net/wlp194s0/address
cat /sys/class/power_supply/BAT0/capacity
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq

# Modify (requires root)
echo performance | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
echo auto        | sudo tee /sys/bus/usb/devices/*/power/control

# Walk attributes like udev does
udevadm info --attribute-walk --name=/dev/sda
udevadm info --query=property --path=/sys/class/net/wlp194s0
```

## Where to learn more

- `man 5 sysfs` — brief overview, mostly about the filesystem mechanics.
- **`Documentation/ABI/`** in the kernel source tree — the closest thing to a canonical attribute reference. Files organized by stability level (`stable/`, `testing/`, `obsolete/`) and subsystem.
- **`Documentation/filesystems/sysfs.rst`** — the design intent. Short, worth reading once.
- **Browse for 10 minutes.** `cd /sys`, `ls`, read some files, walk up parents. The mental model consolidates fast once you see the shape.

Once you internalize that sysfs is **a structured, navigable view of the kernel's device model, one attribute per file, mirroring real topology**, everything else falls into place. Every hardware utility you've used is a thin wrapper reading from here. Knowing how to read it directly means you can debug, script, and understand Linux hardware at a level most GUI users never touch.