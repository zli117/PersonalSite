---
date: '2026-04-19T20:57:31-07:00'
draft: true
title: 'Ioctl'
---
## What ioctl is and why it exists

Unix's core abstraction is the file descriptor: `open`, `read`, `write`, `close`. Beautifully simple, and wonderfully general when what you're doing is moving bytes around. But devices aren't just byte streams. A serial port has a baud rate. A disk has a geometry. A network interface has an MTU and a MAC address. A tape drive rewinds. A terminal has a window size. None of these fit `read()`/`write()`.

Early Unix needed an escape hatch. In 1979, V7 introduced **`ioctl(fd, request, argp)`** — "I/O control" — a syscall that says: "take this open file descriptor, perform this driver-specific operation identified by `request`, using this argument." A device driver exposes a bundle of operations through a single syscall, keyed by an opcode. Each driver defines its own opcodes and argument structures.

This works. It also means `ioctl` is the most chaotic syscall in Unix. There are thousands of distinct ioctls. Each one is effectively a bespoke function call into a specific driver. The type signature is a lie — the "third argument" is different for every request, and you're trusted to pass the right thing.

**Modern Linux's position on ioctl:** necessary evil, generally avoided for new designs. New kernel-userspace interfaces prefer netlink (structured, versioned, extensible), sysfs (declarative attributes), BPF (programmable), or filesystems like configfs (state as directories). But mountains of legacy and many niche interfaces remain ioctl-based, so you still need to understand it.

## The [[Syscalls|syscall]] signature

```c
int ioctl(int fd, unsigned long request, ...);
```

- **`fd`** — an open file descriptor. The driver behind the fd is what handles the ioctl.
- **`request`** — an opcode. Traditionally an integer, usually encoded to include direction (read/write/none), size of argument, a "type" byte identifying the subsystem, and a command number.
- **`argp`** — optional, typed-differently-per-request pointer (or sometimes an integer cast to a pointer, or nothing at all).

Return value: usually 0 on success, -1 on error with `errno` set. Some ioctls return meaningful positive values (e.g., `FIONREAD` returns bytes available). You have to know which.

The critical insight: **the file descriptor determines which driver receives the ioctl.** `ioctl(fd_for_a_socket, ...)` goes to the socket layer. `ioctl(fd_for_/dev/video0, ...)` goes to the V4L2 driver. `ioctl(fd_for_/dev/sda, ...)` goes to the block layer. Same syscall, completely different universe based on what `fd` points to.

## The request encoding (_IO macros)

Requests aren't arbitrary integers. They're constructed with a set of macros that pack metadata into the number:

```c
#define _IO(type, nr)         /* no data */
#define _IOR(type, nr, size)  /* driver writes to userspace: read from driver's POV */
#define _IOW(type, nr, size)  /* driver reads from userspace: write to driver's POV */
#define _IOWR(type, nr, size) /* both directions */
```

A typical definition:

```c
#define VIDIOC_QUERYCAP  _IOR('V',  0, struct v4l2_capability)
#define TUNSETIFF        _IOW('T', 202, int)
#define BLKGETSIZE64     _IOR(0x12, 114, size_t)
```

- **Direction bits** — `_IOR`, `_IOW`, `_IOWR`, `_IO`. Critical: the "R/W" is from **userspace's** perspective. `_IOR` means "userspace reads data back" (driver fills a buffer). `_IOW` means "userspace writes data" (driver consumes it).
- **Type (magic number)** — a byte that identifies the subsystem. `'V'` = V4L2, `'T'` = TUN/TAP, `0x12` = block devices. There's a list in `Documentation/userspace-api/ioctl/ioctl-number.rst` in the kernel tree.
- **Command number** — sequential within the type.
- **Size** — derived from the type of the argument struct; the kernel can sanity-check.

You rarely build these yourself. You `#include <linux/something.h>` and use the provided names. The packing helps the kernel catch bugs: if userspace passes a struct that's the wrong size for the request, the kernel can notice.

## A worked example: getting the terminal window size

Every terminal you've used in a shell knows its size (`$COLUMNS`, `$LINES`). How? `ioctl(TIOCGWINSZ)`.

```c
#include <stdio.h>
#include <sys/ioctl.h>
#include <unistd.h>

int main(void) {
    struct winsize ws;
    if (ioctl(STDOUT_FILENO, TIOCGWINSZ, &ws) == -1) {
        perror("ioctl TIOCGWINSZ");
        return 1;
    }
    printf("rows=%u cols=%u\n", ws.ws_row, ws.ws_col);
    return 0;
}
```

Compile and run:

```bash
cc -o winsz winsz.c
./winsz
# rows=48 cols=180
```

Breaking it down:

- `STDOUT_FILENO` is fd 1 — points to the terminal (when not redirected).
- `TIOCGWINSZ` is "Terminal I/O Control, Get Window Size" — defined in `<sys/ioctl.h>`.
- `&ws` is a pointer to a `struct winsize` the kernel will fill in.

`stty size`, `tput cols`, `resize`, every full-screen TUI program — all calling `ioctl(TIOCGWINSZ)`. Your shell registers a signal handler for `SIGWINCH`, receives the signal on resize, re-runs the ioctl, updates the environment.

## The major families

Most ioctls you'll encounter fall into these subsystems:

**Terminal control (`<sys/ioctl.h>`, `<termios.h>`)** — `TIOCGWINSZ`, `TIOCSWINSZ`, `TIOCGPGRP`, `TIOCSCTTY`. Though for terminal attribute changes (baud rate, cooked mode, echo), the `tcgetattr`/`tcsetattr` wrappers are preferred over direct ioctls.

**Socket / networking (`<sys/socket.h>`, `<net/if.h>`)** — `SIOCGIFADDR`, `SIOCGIFMTU`, `SIOCGIFFLAGS` and a long tail of others. Historically the only way to configure network interfaces. Now largely superseded by netlink; `ip` uses netlink, `ifconfig` uses ioctls, which is one reason `ifconfig` is deprecated.

**Block devices (`<linux/fs.h>`, `<sys/mount.h>`)** — `BLKGETSIZE64` (size of a block device), `BLKDISCARD` (TRIM), `BLKZEROOUT`, `FITRIM`, `FIFREEZE`/`FITHAW`. How `lsblk`, `fdisk`, and `blkdiscard` operate.

**Filesystem-specific (`<linux/fs.h>`)** — `FICLONE` (reflink copy on CoW filesystems), `FIDEDUPERANGE`, `FS_IOC_GETFLAGS` (chattr attributes), `FS_IOC_FIEMAP` (extent mapping). How `cp --reflink`, `chattr`, `filefrag` work.

**V4L2 — video capture (`<linux/videodev2.h>`)** — `VIDIOC_QUERYCAP`, `VIDIOC_ENUM_FMT`, `VIDIOC_S_FMT`, `VIDIOC_QBUF`, `VIDIOC_STREAMON`. The entire webcam/capture-card API is ioctls. Same with V4L2 radio, some broadcast/encoder hardware.

**DRM / graphics (`<drm/drm.h>`, `<xf86drm.h>`)** — every GPU operation: mode setting, buffer allocation, command submission. Userspace libraries like libdrm and Mesa wrap these; you virtually never call them directly. Wayland compositors and X servers are big ioctl clients.

**TUN/TAP (`<linux/if_tun.h>`)** — `TUNSETIFF`, `TUNSETPERSIST`. How VPNs get their virtual network interfaces.

**Input (`<linux/input.h>`)** — `EVIOCGNAME`, `EVIOCGBIT`, `EVIOCGRAB`. How `evtest`, libinput, and X/Wayland read keyboards and mice.

**KVM (`<linux/kvm.h>`)** — hypervisors create VMs, vCPUs, memory regions, and run guests via ioctls on `/dev/kvm`. QEMU, crosvm, Firecracker all use these.

**Generic SCSI / NVMe passthrough (`<scsi/sg.h>`, `<linux/nvme_ioctl.h>`)** — send raw commands to storage devices. How `smartctl`, `nvme-cli`, `sg3_utils` work.

Each of these is essentially its own kernel-userspace protocol with ioctls as the transport.

## Inspecting ioctls a program uses

When you're curious how a tool talks to the kernel, or debugging something that's not working, `strace -e ioctl` is your best friend:

```bash
strace -e ioctl -o /tmp/trace.log stty size
tail /tmp/trace.log
```

You'll see lines like:

```
ioctl(0, TIOCGWINSZ, {ws_row=48, ws_col=180, ws_xpixel=0, ws_ypixel=0}) = 0
```

`strace` decodes well-known ioctls and prints the arguments structurally. For unknown ones, it prints the raw number:

```
ioctl(3, _IOC(_IOC_READ, 0x12, 0x72, 0x8), [0x1fffffe00]) = 0
```

That's `BLKGETSIZE64` on an fd for `/dev/sda`, returning 549755813376 bytes (the `0x1fffffe00` when multiplied by 512, roughly — actually the raw return).

Decoding an unknown ioctl number by hand: take the number, extract direction/type/command/size with the reverse of the `_IO*` macros. Or grep the kernel headers for the specific constant:

```bash
grep -r "0x1272" /usr/include/linux/    # poke around
```

For many devices, the number encodes enough that `_IOC_TYPE(req)` gives you the magic number (`'V'` → V4L2, etc.), and that narrows the search.

## Why the interface is so awkward

A few rough edges you'll bump into:

**1. The prototype lies.** `ioctl(fd, request, ...)` in `<sys/ioctl.h>` is a variadic stub. The "real" signature depends on the request. Compilers can't type-check. You have to read docs and get the arg type right.

**2. Endianness and alignment are yours.** ioctls take raw structs by address. The kernel and userspace must agree on layout exactly — including padding, bitfield ordering, and integer widths. This is why the kernel defines structs with explicit widths (`__u32`, not `unsigned int`) in headers under `<linux/...>`.

**3. 32-bit vs 64-bit is a minefield.** A 32-bit binary on a 64-bit kernel sends ioctls that look different at the ABI level (struct sizes change). The kernel has "compat" wrappers for many common ioctls; obscure ones sometimes don't, and you get mysterious failures when running 32-bit tools.

**4. Privilege is the driver's call.** No uniform policy. Some ioctls require `CAP_NET_ADMIN`, some require `CAP_SYS_ADMIN`, some require ownership of the fd, some require nothing. You read man pages or source code.

**5. ABI versioning is haphazard.** Older ioctls had no versioning. Newer ones often include a "size" or "version" field in the struct. If a driver adds fields, old userspace sending old-size structs still works (hopefully), but new userspace on an old kernel gets `-ENOTTY` or `-EINVAL`.

**6. No introspection.** Unlike netlink (self-describing messages) or D-Bus (introspection API), ioctls are opaque. You cannot ask "what ioctls does this fd support?" — you have to know.

These are exactly the reasons new subsystems don't use ioctl anymore.

## Writing a driver ioctl (the other side)

Briefly, for perspective — this is what a driver author writes:

```c
static long mydev_ioctl(struct file *filp, unsigned int cmd, unsigned long arg) {
    switch (cmd) {
    case MYDEV_GET_COUNT: {
        int count = get_count();
        if (copy_to_user((void __user *)arg, &count, sizeof(count)))
            return -EFAULT;
        return 0;
    }
    case MYDEV_SET_MODE: {
        struct mydev_mode mode;
        if (copy_from_user(&mode, (void __user *)arg, sizeof(mode)))
            return -EFAULT;
        return set_mode(&mode);
    }
    default:
        return -ENOTTY;   // "not a typewriter" — classic Unix for "unknown ioctl"
    }
}
```

Key points:

- `copy_to_user` / `copy_from_user` — never dereference userspace pointers directly; these helpers handle faults and the user/kernel boundary.
- `-ENOTTY` is the canonical "I don't know this ioctl" error.
- The driver registers this function as its `.unlocked_ioctl` file operation, and the kernel dispatches based on fd.

## A more involved example: creating a TUN device

Here's a classic "userspace grabs a virtual network interface" — what WireGuard userspace tools, OpenVPN, etc. do.

```c
#include <fcntl.h>
#include <linux/if.h>
#include <linux/if_tun.h>
#include <stdio.h>
#include <string.h>
#include <sys/ioctl.h>
#include <unistd.h>

int main(void) {
    int fd = open("/dev/net/tun", O_RDWR);
    if (fd < 0) { perror("open"); return 1; }

    struct ifreq ifr = {0};
    ifr.ifr_flags = IFF_TUN | IFF_NO_PI;
    strncpy(ifr.ifr_name, "mytun0", IFNAMSIZ);

    if (ioctl(fd, TUNSETIFF, &ifr) < 0) {
        perror("ioctl TUNSETIFF");
        return 1;
    }

    printf("Created %s on fd %d. Sleeping. Configure with:\n", ifr.ifr_name, fd);
    printf("  sudo ip addr add 10.9.0.1/24 dev %s\n", ifr.ifr_name);
    printf("  sudo ip link set %s up\n", ifr.ifr_name);
    sleep(600);
    return 0;
}
```

Run this as root:

```bash
sudo ./tuntest &
ip link show mytun0
# 42: mytun0: <POINTOPOINT,MULTICAST,NOARP> mtu 1500 qdisc noop state DOWN ...
```

The single `ioctl(fd, TUNSETIFF, &ifr)` call asked the TUN driver to create a new virtual interface named `mytun0` and associate it with `fd`. Reading from `fd` now gives you IP packets destined for that interface; writing pushes packets out. That's the entire API.

## The direct-mode alternative: block devices

Sometimes ioctl is the _only_ way to do something. Getting the size of a block device:

```c
#include <fcntl.h>
#include <linux/fs.h>
#include <stdint.h>
#include <stdio.h>
#include <sys/ioctl.h>
#include <unistd.h>

int main(int argc, char **argv) {
    int fd = open(argv[1], O_RDONLY);
    uint64_t sz;
    ioctl(fd, BLKGETSIZE64, &sz);
    printf("%s: %lu bytes\n", argv[1], sz);
    close(fd);
}
```

```bash
sudo ./blksize /dev/nvme0n1
# /dev/nvme0n1: 2048408248320 bytes
```

`stat()` on a block device only gives you a minor number and metadata — it doesn't know the device size. The size lives in the block layer, accessed via ioctl.

## NVMe passthrough — a modern, structured example

Newer kernels use ioctl less for device control and more for **passthrough** — letting userspace send raw protocol commands:

```c
struct nvme_passthru_cmd cmd = {
    .opcode       = 0x06,  // Identify
    .nsid         = 0,
    .addr         = (uint64_t)(uintptr_t)buffer,
    .data_len     = 4096,
    .cdw10        = 1,     // Identify Controller
};
ioctl(nvme_fd, NVME_IOCTL_ADMIN_CMD, &cmd);
```

`nvme-cli` does exactly this to query SMART data, firmware versions, etc. The ioctl is thin — it's really a shim to let userspace construct NVMe commands without reimplementing the driver. Similar patterns for SCSI (`SG_IO`), for ATA (`HDIO_DRIVE_CMD`), for DVD drives (MMC commands).

## When new kernel interfaces _do_ use ioctl

Even in 2026, ioctl isn't dead — just avoided for new "broad" APIs. Where it still shows up:

- **High-bandwidth paths** where the overhead of message marshaling matters (KVM hypercalls, io_uring's setup calls, DRM command submission).
- **Hardware-specific passthrough** where the argument is literally a hardware command (NVMe, SCSI).
- **Existing subsystems** extending their ioctl vocabulary because adding one more opcode is easier than migrating to netlink.

For new general-purpose config interfaces ("describe/set the state of this thing"), developers reach for netlink, sysfs, configfs, or BPF. For low-level per-request operations on an fd, ioctl is still the natural fit.

## Debugging ioctl problems

When an ioctl returns `-1`:

**`strace`** first:

```bash
strace -e ioctl -f <command>
```

Common errnos:

- **`ENOTTY` (25)** — driver doesn't recognize this request. Either wrong fd type (you ioctl'd a socket with a tty ioctl), or kernel too old, or driver doesn't implement it.
- **`EINVAL` (22)** — request is recognized but argument is wrong (bad flags, wrong size, invalid values).
- **`EFAULT` (14)** — argument pointer is bad (NULL, unmapped, wrong length). Classic "I passed the struct by value instead of by pointer."
- **`EPERM` / `EACCES`** — permission denied. Often need root or a specific capability.
- **`ENODEV`** — device disappeared or is in wrong state.

**Check the right header:**

```bash
grep -rn "VIDIOC_QUERYCAP" /usr/include/linux/
# /usr/include/linux/videodev2.h:2483:#define VIDIOC_QUERYCAP _IOR('V',  0, struct v4l2_capability)
```

The header is the source of truth for the struct and encoding.

**Read the kernel source.** `Documentation/userspace-api/ioctl/` in the kernel tree is the most authoritative list. Per-subsystem documentation (e.g., `Documentation/admin-guide/media/`) often has detailed ioctl writeups.

## Things that trip people up

**Wrong fd type.** You opened `/dev/video0` and sent it a block-device ioctl. `ENOTTY`. The ioctl opcode encodes its expected "type" byte, and drivers filter on it, but not rigorously — usually it just doesn't match any case in the driver's switch.

**Struct mismatch between libc and kernel.** If you use headers from an older system or mismatch `<linux/if.h>` vs `<net/if.h>`, struct sizes differ and you get `EINVAL` or corruption. Stick with one consistent set (kernel headers `<linux/*>` for kernel-defined structs; glibc headers for POSIX stuff).

**Mixing up direction.** `_IOR` vs `_IOW`. Driver doc says "this ioctl returns capabilities" so you think `_IOW`, but it's `_IOR` (userspace reads back). The macro convention is confusing.

**Threading and signals.** Some ioctls block. They're interruptible by signals (returning `EINTR`). Long-running ioctls (like V4L2's `VIDIOC_DQBUF` waiting for a frame) need careful signal handling.

**Nonblocking isn't universal.** `O_NONBLOCK` on the fd affects `read`/`write`, but many ioctls are always blocking or always non-blocking regardless of the fd flag. Read per-ioctl docs.

**Compat layer surprises.** Running 32-bit tools on 64-bit kernel: usually fine for common ioctls, occasionally broken for obscure ones with pointer-containing structs that need manual compat handling. "It works on x86_64 native but fails on i386 chroot" is a classic signature.

## How it fits with everything else

Positioning ioctl against its neighbors:

- **vs read/write/mmap** — read/write are for streaming bytes. ioctl is for out-of-band control. Most drivers implement both: read/write for the data path, ioctl for the control path. mmap is for exposing device memory or kernel buffers directly to userspace, often negotiated via ioctl first.
    
- **vs sysfs** — sysfs is declarative (one value per file, steady-state config). ioctl is imperative (perform this operation now, possibly with complex side effects). You'd use sysfs to set a persistent attribute, ioctl for "eject the disc" or "start DMA."
    
- **vs netlink** — netlink is structured, extensible, async, and has broadcast. It's the modern pick for configuration APIs. ioctl is synchronous, point-to-point, opaque. New networking features go to netlink; ioctl networking (SIOC*) is legacy.
    
- **vs D-Bus** — D-Bus is _userspace-to-userspace_ IPC at the daemon/app level. ioctl is userspace-to-kernel. Different planes. Nothing prevents a userspace daemon from wrapping an ioctl interface in a D-Bus service (and many do — NetworkManager wraps a pile of kernel ioctls+netlink behind a D-Bus API).
    
- **vs BPF** — BPF lets you run custom code inside the kernel via a well-defined VM. ioctl is just a function call with an opcode. BPF replaces some ioctl-based filtering interfaces (seccomp, xdp, etc.) because "load a program" is far more expressive than "pick one of these 80 preset operations."
    

## Quick reference card

```bash
# Observe ioctls a program uses
strace -e ioctl -f <command>
strace -e ioctl -o trace.log -s 4096 <command>    # more detail, save to file

# Search for an ioctl definition
grep -rn "IOCTL_NAME" /usr/include/linux/
grep -rn "IOCTL_NAME" /usr/include/

# Decode a raw number (in a program)
#   _IOC_DIR(n)   -> 0=NONE, 1=WRITE, 2=READ, 3=RW
#   _IOC_TYPE(n)  -> magic byte (subsystem)
#   _IOC_NR(n)    -> command number
#   _IOC_SIZE(n)  -> argument size
```

Common headers to know:

|Header|Contents|
|---|---|
|`<sys/ioctl.h>`|The syscall and tty ioctls|
|`<termios.h>`|Terminal attributes (preferred over raw TC* ioctls)|
|`<linux/fs.h>`|Filesystem/block ioctls (`BLK*`, `FIBMAP`, `FITRIM`, `FICLONE`)|
|`<net/if.h>`, `<sys/socket.h>`|Socket/interface ioctls (`SIOC*`)|
|`<linux/if_tun.h>`|TUN/TAP|
|`<linux/videodev2.h>`|V4L2|
|`<linux/input.h>`|Input devices (EVIOC*)|
|`<linux/kvm.h>`|KVM hypervisor|
|`<scsi/sg.h>`|SCSI generic passthrough|
|`<linux/nvme_ioctl.h>`|NVMe passthrough|

## Where to learn more

- **`man 2 ioctl`** — short, and points at subsystem-specific man pages.
- **`man 2 ioctl_list`** — a listing of common ioctls (incomplete but useful as a start).
- **`Documentation/userspace-api/ioctl/`** in the Linux source — the most complete listing, organized by magic byte.
- **Subsystem docs** — `Documentation/admin-guide/media/` for V4L2, `Documentation/block/` for block layer, etc. The per-subsystem docs are where real ioctl usage is explained.
- **`<linux/*.h>` headers in `/usr/include/linux/`** — the actual definitions, usually with brief comments. Surprisingly readable.
- **strace source** — has the decoders for hundreds of ioctls. Great reference for "what does this opaque number mean".

The mental model to take away: **ioctl is a generic escape hatch that lets a file descriptor carry driver-specific operations, each identified by an opaque encoded integer and an argument whose type the caller must know out-of-band.** It's the oldest and least-structured kernel-userspace interface, kept alive by legacy and by use cases where its directness pays off. Modern design prefers structured alternatives, but understanding ioctl is still essential because so much of Linux's hardware and filesystem surface area still lives behind it.
