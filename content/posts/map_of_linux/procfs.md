---
date: '2026-04-19T22:20:20-07:00'
draft: true
title: 'Chapter 3: procfs'
weight: 30
---
## What procfs is and why it exists

Early Unix had a problem: how does userspace inspect what the kernel is doing? The kernel tracks processes, memory, open files, network connections, load averages, interrupt counters, thousands of things. Userspace tools — `ps`, `top`, `free`, `netstat`, `uptime` — need to see this. Options available in the 1970s/80s:

- **Read `/dev/kmem`** directly, walking kernel data structures by interpreting raw memory. This is how classical `ps` worked. Every kernel change broke `ps` because the struct layouts shifted. Required setuid root or setgid kmem. Horrifying by modern standards.
- **Specialized syscalls.** Add a syscall per question. Doesn't scale to thousands of queries; each is a kernel ABI commitment.
- **Structured files.** Expose kernel state as files readable with normal `read()` syscalls. Text, so layout changes are visible and versionable. This became procfs.

**procfs** is a virtual filesystem, mounted at `/proc`, introduced in 8th Edition Unix (~1984), adopted into Linux very early (0.97, 1992). It exposes two broad categories of state:

1. **Per-process info** under `/proc/<pid>/` — one directory per running process, containing that process's details.
2. **System-wide info** directly under `/proc/` — kernel parameters, hardware info, subsystem state.

The key move: **these aren't real files.** Every read triggers the kernel to serialize its in-memory state as text, on the fly. When you `cat /proc/meminfo`, no file is read; the kernel's memory management code runs, formats current counters, and hands them back. Same for every file under `/proc`.

This design choice is why procfs became the foundation for nearly every Linux observability tool. `ps`, `top`, `htop`, `free`, `uptime`, `lsof`, `pmap`, `ss`, `vmstat`, `iostat`, `pidstat`, `systemd-cgls`, dozens of others are thin wrappers reading `/proc`.

## procfs vs. sysfs vs. the rest

You've seen sysfs. The big confusion is what goes where, and why there are so many virtual filesystems under `/`.

| Filesystem | Mount | Primary purpose |
|---|---|---|
| **procfs** | `/proc` | Process info + legacy kernel state. Multi-line text dumps common. Written 1990s, not super structured. |
| **sysfs** | `/sys` | Kernel object model: devices, drivers, buses. One value per file. Structured. Written 2000s. |
| **devtmpfs** | `/dev` | Device nodes. You open these and use read/write/ioctl. |
| **cgroup2** | `/sys/fs/cgroup` | cgroup hierarchy and controllers. |
| **debugfs** | `/sys/kernel/debug` | Driver debugging. No ABI stability. |
| **tracefs** | `/sys/kernel/tracing` | ftrace interface. |
| **configfs** | `/sys/kernel/config` | User-creatable kernel objects (USB gadgets, iSCSI, etc.). |
| **bpffs** | `/sys/fs/bpf` | Pinned BPF programs/maps. |

The split isn't perfectly clean — procfs grew accretively, and some things that "should" have gone to sysfs are still in procfs for backward compatibility. Rough rules:

- **Process-specific → procfs.** Always. Every process is in `/proc/<pid>/`.
- **Kernel global tunables → `/proc/sys/` (procfs).** `sysctl` is just a wrapper for reading/writing these.
- **Device / driver state → sysfs.** Sensors, block devices, network interfaces' metadata.
- **New subsystems → sysfs or a dedicated filesystem.** Since ~2003, new kernel-userspace interfaces rarely add to procfs.

Pragmatically: if you want per-process info, you're in `/proc`. If you want hardware info, you're probably in `/sys`. If you want kernel tunables, you're in `/proc/sys/`.

## The per-process tree: `/proc/<pid>/`

Every running process has a directory named by its PID. Inside is a rich set of files and subdirectories describing that process. The most important ones:

| Path | Content |
|---|---|
| `cmdline` | Command line, null-separated. |
| `status` | Human-readable status: state, UIDs, memory, signals, threads. |
| `stat` | Space-separated machine-parseable status (the file `ps` parses). |
| `comm` | Short command name (the 15-char thread name, writable). |
| `exe` | Symlink to the executable. |
| `cwd` | Symlink to current working directory. |
| `root` | Symlink to the root directory (differs inside chroots/containers). |
| `environ` | Environment variables, null-separated. |
| `fd/` | Directory of open file descriptors, each a symlink. |
| `fdinfo/` | Matching metadata for each fd (flags, position, type). |
| `maps` | Memory mappings: libraries, heap, stack, anon regions, with addresses and perms. |
| `smaps` | Like maps but with detailed per-mapping memory accounting. |
| `smaps_rollup` | Aggregated smaps totals. Cheaper. |
| `pagemap` | Per-virtual-page to physical-page map. Binary. |
| `limits` | Resource limits (rlimits). |
| `io` | Total I/O counters (bytes read/written). |
| `stack` | Kernel-side stack trace of the process. |
| `wchan` | What kernel function the process is blocked in. |
| `syscall` | Current syscall being executed, if any. |
| `ns/` | Namespace links (mnt, pid, net, user, uts, ipc, cgroup, time). |
| `task/` | Subdirectory of threads (each thread has its own `/proc/<pid>/task/<tid>/`). |
| `oom_score`, `oom_score_adj` | OOM killer heuristics. |
| `sched` | Scheduler statistics. |
| `cgroup` | Which cgroups this process is in. |
| `net/` | Per-net-namespace network state (same structure as `/proc/net/`). |
| `mountinfo`, `mounts` | Mount namespace contents. |
| `attr/` | LSM (SELinux/AppArmor) attributes. |

The directory exists as long as the process does. When a process exits, its `/proc/<pid>/` directory vanishes.

### Tour: what's in a process

Pick any process you own and explore. Here's `bash`:

```bash
cd /proc/self        # self is a magic symlink to the current process's PID
ls -la
cat status
cat cmdline | tr '\0' ' '; echo
readlink exe
ls -la fd/
cat maps | head
cat limits
```

Walking through useful outputs:

**`status`** — the human-readable dump. Every value you'd reach `ps` for is here:

```bash
cat /proc/$$/status | head -30
# Name:   bash
# Umask:  0022
# State:  S (sleeping)
# Tgid:   12345
# Ngid:   0
# Pid:    12345
# PPid:   12340
# TracerPid:      0
# Uid:    1000    1000    1000    1000
# Gid:    1000    1000    1000    1000
# FDSize: 256
# Groups: 10 1000 ...
# VmPeak:    12384 kB
# VmSize:    12340 kB
# VmRSS:      4892 kB
# Threads:   1
# SigQ:   0/31172
# ...
```

Every tool that shows process memory, state, and ownership reads `status` (or its faster cousin `stat`).

**`cmdline`** — null-separated args:

```bash
tr '\0' ' ' < /proc/self/cmdline; echo
```

The nulls are there so args containing spaces aren't ambiguous. `ps -f` prints this with spaces for display.

**`maps`** — the entire memory layout:

```bash
cat /proc/self/maps | head
# 55f9e4c00000-55f9e4c04000 r--p 00000000 fd:00 1234  /usr/bin/bash
# 55f9e4c04000-55f9e4c81000 r-xp 00004000 fd:00 1234  /usr/bin/bash
# 55f9e4c81000-55f9e4cb9000 r--p 00081000 fd:00 1234  /usr/bin/bash
# 55f9e4cb9000-55f9e4cbc000 r--p 000b8000 fd:00 1234  /usr/bin/bash
# 55f9e4cbc000-55f9e4cc5000 rw-p 000bb000 fd:00 1234  /usr/bin/bash
# 55f9e4cc5000-55f9e4ccf000 rw-p 00000000 00:00 0
# 7f2b3c000000-7f2b3c021000 rw-p 00000000 00:00 0
# ...
# 7ffd8c000000-7ffd8c021000 rw-p 00000000 00:00 0     [stack]
# ffffffffff600000-ffffffffff601000 --xp 00000000 00:00 0 [vsyscall]
```

Columns: address range, permissions (r/w/x/p=private or s=shared), offset, device major:minor, inode, pathname. Anonymous regions have `00:00 0` for device/inode. Named regions (executables, libraries) have the file info. Special mappings like `[heap]`, `[stack]`, `[vdso]`, `[vvar]` are labeled.

**`smaps`** — same regions but with detailed memory accounting:

```bash
head -20 /proc/self/smaps
# 55f9e4c00000-55f9e4c04000 r--p 00000000 fd:00 1234  /usr/bin/bash
# Size:                 16 kB
# KernelPageSize:        4 kB
# MMUPageSize:           4 kB
# Rss:                  16 kB
# Pss:                  16 kB        ← proportional set size (shared memory / sharers)
# Shared_Clean:          0 kB
# Shared_Dirty:          0 kB
# Private_Clean:        16 kB
# Private_Dirty:         0 kB
# Referenced:           16 kB
# Anonymous:             0 kB
# Swap:                  0 kB
# ...
```

`Pss` (proportional set size) is how fair memory accounting works — shared memory divided across all sharers. This is what `ps_mem` and similar tools use for accurate per-process memory.

**`fd/`** — every open file descriptor as a symlink:

```bash
ls -la /proc/self/fd/
# lrwx------ 1 zl zl 64 Apr 19 15:00 0 -> /dev/pts/3
# lrwx------ 1 zl zl 64 Apr 19 15:00 1 -> /dev/pts/3
# lrwx------ 1 zl zl 64 Apr 19 15:00 2 -> /dev/pts/3
# lrwx------ 1 zl zl 64 Apr 19 15:00 3 -> socket:[1234567]
# lrwx------ 1 zl zl 64 Apr 19 15:00 4 -> anon_inode:[eventfd]
```

Sockets show as `socket:[inode]` — cross-reference with `/proc/net/tcp` by inode to find endpoints. Anon inodes (eventfd, signalfd, timerfd, epoll, pidfd, bpf) show their type. Pipes show `pipe:[inode]`.

This is exactly how `lsof` works: walks `/proc/<pid>/fd/` for every PID, resolves the symlinks, correlates with `/proc/net/*` and mount info.

**`fdinfo/`** — matching metadata for each fd:

```bash
cat /proc/self/fdinfo/3
# pos:    0
# flags:  02004002      ← O_NONBLOCK | ...
# mnt_id: 15
```

For special fd types, there's often more: `fdinfo/N` for an epoll fd lists its watched fds; for bpf, it describes the program.

**`io`** — lifetime I/O counters:

```bash
cat /proc/self/io
# rchar: 5432109876        ← total bytes requested via read (including cached)
# wchar: 12345             ← same for write
# syscr: 1234              ← count of read-like syscalls
# syscw: 5                 ← count of write-like syscalls
# read_bytes: 1048576      ← bytes actually caused block-layer reads
# write_bytes: 0           ← bytes actually caused block-layer writes
# cancelled_write_bytes: 0
```

The distinction between `rchar`/`wchar` (logical) and `read_bytes`/`write_bytes` (physical) is important: a program that reads heavily but from the page cache has high `rchar` but low `read_bytes`.

**`ns/`** — namespace membership:

```bash
ls -la /proc/self/ns/
# lrwxrwxrwx 1 zl zl 0 Apr 19 15:00 cgroup -> 'cgroup:[4026531835]'
# lrwxrwxrwx 1 zl zl 0 Apr 19 15:00 ipc -> 'ipc:[4026531839]'
# lrwxrwxrwx 1 zl zl 0 Apr 19 15:00 mnt -> 'mnt:[4026531841]'
# lrwxrwxrwx 1 zl zl 0 Apr 19 15:00 net -> 'net:[4026531840]'
# lrwxrwxrwx 1 zl zl 0 Apr 19 15:00 pid -> 'pid:[4026531836]'
# lrwxrwxrwx 1 zl zl 0 Apr 19 15:00 user -> 'user:[4026531837]'
# lrwxrwxrwx 1 zl zl 0 Apr 19 15:00 uts -> 'uts:[4026531838]'
# ...
```

The inode numbers identify namespaces. Two processes with the same `net` namespace inode share a network namespace (same interfaces, same routing, same netns). Container runtimes compare these to know containment. `nsenter` uses these symlinks as targets.

**`task/`** — threads:

```bash
ls /proc/1234/task/
# 1234  1235  1236  1237   ← one directory per thread (thread IDs)

cat /proc/1234/task/1235/status       # per-thread status
```

Each thread directory has the same files as the process directory, scoped to that thread. `top -H` shows threads by reading this.

### `/proc/self` and `/proc/thread-self`

Magic symlinks:

- `/proc/self` → the current process's `/proc/<pid>/`.
- `/proc/thread-self` → the current thread's `/proc/<pid>/task/<tid>/`.

Invaluable in scripts: `readlink /proc/self/exe` reliably gives you the path to the running binary regardless of PID.

## System-wide files: the most useful ones

A walking tour of the non-process files that matter. Almost every observability tool reads at least one.

### CPU and scheduling

**`/proc/cpuinfo`** — per-CPU (per hardware thread) details:

```bash
head -30 /proc/cpuinfo
# processor       : 0
# vendor_id       : AuthenticAMD
# cpu family      : 25
# model           : 116
# model name      : AMD Ryzen AI Max+ 395 w/ Radeon 890M Graphics
# stepping        : 1
# cpu MHz         : 4300.000
# cache size      : 1024 KB
# physical id     : 0
# siblings        : 32
# core id         : 0
# cpu cores       : 16
# flags           : fpu vme de pse tsc msr pae mce cx8 ... sse4_2 avx avx2 ...
```

One section per logical CPU. `flags` tells you feature availability (AVX, AES-NI, SGX, etc.). `lscpu` parses this.

**`/proc/stat`** — system-wide CPU counters:

```bash
head -5 /proc/stat
# cpu  1234 56 7890 12345678 100 0 200 0 0 0
# cpu0 50 2 300 500000 5 0 10 0 0 0
# cpu1 ...
# intr 1000000 ...
# ctxt 5000000
```

Fields per CPU line: user, nice, system, idle, iowait, irq, softirq, steal, guest, guest_nice (in clock ticks, typically 1/100 second). Every `top`, `vmstat`, `mpstat` reading computes deltas of this file.

**`/proc/loadavg`** — the famous three numbers:

```bash
cat /proc/loadavg
# 0.38 0.42 0.35 2/1823 29145
# 1min 5min 15min running/total latest-pid
```

What `uptime` reads.

**`/proc/uptime`** — seconds since boot, idle time:

```bash
cat /proc/uptime
# 45692.38 687890.22
```

**`/proc/schedstat`**, **`/proc/<pid>/schedstat`**, **`/proc/<pid>/sched`** — scheduler statistics. Useful for latency analysis.

### Memory

**`/proc/meminfo`** — the overview:

```bash
head -20 /proc/meminfo
# MemTotal:       131072000 kB
# MemFree:         45678900 kB
# MemAvailable:    78912345 kB
# Buffers:          1234567 kB
# Cached:          34567890 kB
# SwapCached:             0 kB
# Active:          12345678 kB
# Inactive:        23456789 kB
# SwapTotal:       16777216 kB
# SwapFree:        16777216 kB
# Dirty:              23456 kB
# Writeback:              0 kB
# AnonPages:       11234567 kB
# Mapped:           2345678 kB
# ...
```

`MemAvailable` is the best "how much can I allocate without pain" number, accounting for reclaimable cache. `free`, `vmstat` both read this.

**`/proc/vmstat`** — virtual memory counters:

```bash
grep -E 'pgfault|pswpin|pswpout|pgpgin|pgpgout' /proc/vmstat
```

Page faults, swap in/out, scan activity. `vmstat 1` is deltas of this.

**`/proc/buddyinfo`** — free memory fragmented by order:

```bash
cat /proc/buddyinfo
# Node 0, zone   DMA      1   1   1   0   2   1   1   0   1   1   3
# Node 0, zone   DMA32    9   5   3   4   3   1   2   2   1   1   100
# Node 0, zone   Normal   500 300 200 100 80 60 40 30 20 10 1500
```

One column per order (order N = 2^N pages). Used for understanding memory fragmentation — if only low-order slots have free pages, large allocations will trigger compaction or fail.

**`/proc/zoneinfo`** — per-zone detailed stats.

**`/proc/slabinfo`** (root-only) — kernel slab allocator usage.

**`/proc/swaps`** — active swap areas.

### Filesystems and mounts

**`/proc/mounts`** — symlink to `/proc/self/mounts`, the mount namespace of the reader. Classic format:

```bash
cat /proc/mounts | head
# /dev/nvme0n1p2 / btrfs rw,seclabel,relatime,compress=zstd:1,subvol=/root 0 0
# /dev/nvme0n1p1 /boot ext4 rw,seclabel,relatime 0 0
# ...
```

**`/proc/self/mountinfo`** — richer, structured:

```bash
head -3 /proc/self/mountinfo
# 26 1 259:2 / / rw,relatime shared:1 - btrfs /dev/nvme0n1p2 rw,seclabel,...
```

Includes mount IDs, parent IDs, propagation flags. What modern tools parse.

**`/proc/filesystems`** — filesystem types the kernel supports. `nodev` prefix means it's not backed by a block device (like `tmpfs`, `proc` itself).

**`/proc/partitions`** — block devices the kernel sees.

**`/proc/diskstats`** — per-device I/O counters, what `iostat` reads.

### Networking

Networking info lives under `/proc/net/`, which is actually a per-netns view. Every network namespace has its own; the current namespace's netns files appear here.

```bash
ls /proc/net/ | head -20
# anycast6  dev          netfilter     stat
# arp       dev_mcast    netlink       tcp
# bonding   dev_snmp6    netstat       tcp6
# ...
```

**`/proc/net/tcp`**, **`/proc/net/tcp6`**, **`/proc/net/udp`** — active sockets:

```bash
head /proc/net/tcp
#   sl  local_address rem_address   st tx_queue rx_queue tr tm->when retrnsmt   uid  timeout inode
#    0: 00000000:0016 00000000:0000 0A 00000000:00000000 00:00000000 00000000     0        0 12345
```

Addresses are hex, reversed-endian (`0100007F:1F40` = 127.0.0.1:8000). The `inode` connects to fds via the `socket:[N]` symlinks. `ss` and `netstat` parse this, though modern `ss` prefers netlink for performance.

**`/proc/net/route`**, **`/proc/net/fib_trie`** — routing table.

**`/proc/net/dev`** — per-interface packet counters (what `ifconfig`/`ip -s` reads).

**`/proc/net/netstat`**, **`/proc/net/snmp`** — TCP/UDP/IP protocol statistics.

**`/proc/net/netlink`** — open netlink sockets.

### Kernel tunables: `/proc/sys/`

This is the big one. Every knob the kernel exposes for tuning lives under `/proc/sys/`, organized by subsystem:

```
/proc/sys/kernel/       # general kernel
/proc/sys/vm/           # virtual memory
/proc/sys/net/          # networking (every protocol)
/proc/sys/fs/           # filesystem
/proc/sys/dev/          # devices
/proc/sys/debug/        # debug
/proc/sys/user/         # per-user-namespace limits
```

Reading gives current value; writing (as root) changes it.

```bash
# Some useful ones
cat /proc/sys/kernel/hostname
cat /proc/sys/vm/swappiness
cat /proc/sys/net/ipv4/ip_forward
cat /proc/sys/fs/file-max

# Change (this boot only)
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward

# Persist across reboots: use /etc/sysctl.d/
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-forward.conf
sudo sysctl --system
```

`sysctl` is just the wrapper: `sysctl net.ipv4.ip_forward` is equivalent to `cat /proc/sys/net/ipv4/ip_forward` (with dots translated to slashes). `sysctl -w net.ipv4.ip_forward=1` is `echo 1 > ...`.

Thousands of tunables live here. `sysctl -a` lists them all.

### Kernel and hardware info

**`/proc/version`** — kernel version string.

**`/proc/cmdline`** — the kernel's command line (what was passed by the bootloader).

**`/proc/kallsyms`** (root-readable) — all kernel symbols with addresses. Used by tracing tools to resolve addresses.

**`/proc/modules`** — loaded kernel modules. What `lsmod` reads.

**`/proc/interrupts`** — interrupt counts per CPU per IRQ. Good for "which CPU is handling all the network interrupts?"

**`/proc/softirqs`** — same for softirqs.

**`/proc/iomem`**, **`/proc/ioports`** — I/O memory and port address space.

**`/proc/devices`** — registered character and block device major numbers.

**`/proc/crypto`** — kernel crypto API: available ciphers/hashes with properties.

**`/proc/consoles`** — registered console devices.

**`/proc/sysrq-trigger`** — writing characters here triggers SysRq functions. `echo b > /proc/sysrq-trigger` reboots *immediately*. Very dangerous; also very useful when the system is half-dead.

**`/proc/kmsg`** — kernel ring buffer (what `dmesg` reads, though `dmesg` uses the `klogctl` syscall now).

## Hands-on: what real tools are doing

The instructive exercise: run common tools with `strace` and watch them read `/proc`.

```bash
strace -e openat,read ps aux 2>&1 | grep /proc | head -20
# openat(AT_FDCWD, "/proc", ...) = 5
# openat(AT_FDCWD, "/proc/1/stat", ...) = 6
# openat(AT_FDCWD, "/proc/1/status", ...) = 6
# openat(AT_FDCWD, "/proc/1/cmdline", ...) = 6
# ... for every PID ...
```

`ps` opens `/proc`, lists directories that look like PIDs, and for each reads a handful of files. That's *all* it does.

```bash
strace -e openat top -n1 -b 2>&1 | grep /proc | head -30
# top reads the same per-PID files, plus /proc/stat, /proc/meminfo, /proc/loadavg
```

Same story for `free`, `vmstat`, `uptime`, `lsof`. Every "system monitoring tool" is a procfs parser.

Conversely, when you wonder "how would I get X in a shell", the answer is usually "read from /proc". A few examples:

```bash
# Memory usage summary
awk '/MemTotal|MemAvailable|SwapFree/' /proc/meminfo

# What's my PID's peak RSS?
grep VmPeak /proc/$$/status

# List my shell's open fds with their targets
ls -la /proc/$$/fd/

# What sockets does a process own?
ls -la /proc/<pid>/fd/ | grep socket

# Reverse lookup: which process owns this socket?
INODE=1234567  # from /proc/net/tcp or from /proc/<pid>/fd/
for p in /proc/[0-9]*/fd/*; do
    target=$(readlink "$p" 2>/dev/null)
    [ "$target" = "socket:[$INODE]" ] && echo "$p"
done

# Which process has the most memory?
awk 'FNR==1{pid=gensub(".*/([0-9]+)/.*", "\\1", 1, FILENAME)} /VmRSS/{print $2, pid}' /proc/*/status | sort -n | tail

# System load history as-is
cat /proc/loadavg
```

Every one of these is just read syscalls against `/proc`. The content is free; the parsing is the only effort.

## Reading procfs efficiently

procfs reads are cheap but not free. Each read triggers the kernel to regenerate the content. For tight loops (monitoring every 100ms), consider:

- **Use `seq_file`-aware I/O.** Many procfs files use the seq_file interface, which lets you read them efficiently with ordinary read(2). Just `open` + `read` in a loop.
- **Don't pread() offsets.** Most procfs files don't support `pread()` meaningfully — content is regenerated on each read, so seeking is meaningless. Open, read to EOF, close.
- **Use `smaps_rollup` instead of `smaps`.** smaps walks all VMAs and is expensive on processes with many mappings (JVMs, web browsers). smaps_rollup gives you the aggregate totals cheaply.
- **Prefer netlink for frequently-updated network state.** `ss` (uses netlink) outperforms parsing `/proc/net/tcp` for large socket counts.

For high-frequency monitoring, BPF-based tools (via bcc/bpftrace) bypass procfs entirely and hook kernel tracepoints directly. procfs is still the answer for one-off queries and for portability.

## Writable procfs: the knobs

Some procfs files accept writes. The main categories:

**`/proc/sys/*`** — kernel tunables (via `sysctl` or direct echo). Hundreds of knobs. Usually root-only; some readable-only-by-root (like `kernel.kptr_restrict`).

**`/proc/<pid>/oom_score_adj`** — adjust OOM killer bias for a process. Write -1000 to 1000. Lower = less likely to be killed.

**`/proc/<pid>/comm`** — writable within the process (for setting thread names).

**`/proc/<pid>/coredump_filter`** — control what gets dumped on SIGSEGV.

**`/proc/<pid>/clear_refs`** — write 1/2/3/4 to reset the "referenced" bits in page tables. Useful for tracking memory access patterns.

**`/proc/<pid>/mem`** — random-access to a process's memory. Used by debuggers (`gdb` reads/writes this with proper permissions and ptrace attachment).

**`/proc/sysrq-trigger`** — trigger SysRq actions: `s` sync, `u` remount-RO, `b` reboot, `o` poweroff, `f` invoke OOM-killer, `l` show stack on all CPUs. Emergency tool.

**`/proc/kcore`** (root-read-only) — a live view of kernel memory as an ELF core file. You can `objdump` or `gdb` against it. Requires `kernel.kptr_restrict=0` to be useful.

## Tutorial: what to reach for

A cheat sheet organized by "what I'm trying to find out."

**"What's this process doing?"**
```bash
PID=$(pgrep foo)
cat /proc/$PID/status           # overview
cat /proc/$PID/cmdline | tr '\0' ' '; echo
readlink /proc/$PID/exe
ls -la /proc/$PID/fd/           # what fds are open?
cat /proc/$PID/stack            # blocked in what kernel function?
cat /proc/$PID/wchan            # short answer to same question
cat /proc/$PID/io               # how much I/O has it done?
```

**"What's eating my memory?"**
```bash
cat /proc/meminfo | head
# Per process, by PSS (fair sharing):
for pid in $(ls /proc/ | grep -E '^[0-9]+$'); do
    pss=$(awk '/^Pss:/{s+=$2}END{print s}' /proc/$pid/smaps_rollup 2>/dev/null)
    [ -n "$pss" ] && echo "$pss $(readlink /proc/$pid/exe 2>/dev/null)"
done | sort -n | tail -10
```

**"Who's got this port open?"**
```bash
# Find the inode via /proc/net/tcp (or use ss which does this)
ss -tlnp
# or manual: port 8080 = hex 1F90
grep ":1F90 " /proc/net/tcp /proc/net/tcp6
# returns the inode. Then:
INODE=...
for fd in /proc/[0-9]*/fd/*; do
    [ "$(readlink "$fd" 2>/dev/null)" = "socket:[$INODE]" ] && echo "$fd"
done
```

**"What's the system doing right now?"**
```bash
cat /proc/loadavg
cat /proc/uptime
head /proc/stat                 # CPU ticks
head /proc/meminfo              # memory
cat /proc/interrupts | head     # IRQs (which CPU? which device?)
cat /proc/pressure/cpu          # PSI: CPU pressure
cat /proc/pressure/memory       # PSI: memory pressure
cat /proc/pressure/io           # PSI: I/O pressure
```

**"Is my program really in the container?"**
```bash
# Compare namespace inodes between containers / host
readlink /proc/$$/ns/net
readlink /proc/<other-pid>/ns/net
# Same inode = same namespace
```

**"What kernel knobs apply?"**
```bash
sysctl -a | grep <topic>
# Or
ls /proc/sys/<subsystem>/
```

## Things that trip people up

**procfs reads are snapshots, not atomic across files.** If you read `/proc/<pid>/status` and then `/proc/<pid>/maps`, they're two separate snapshots — the process can have changed between. Stat it once, read what you need, parse carefully.

**Permissions vary.** Reading another user's `/proc/<pid>/environ` or `/proc/<pid>/maps` requires ptrace capability or same UID. You'll see 0 bytes or `EACCES`. `hidepid=2` mount option hides other users' PIDs entirely.

**`/proc/net/*` is per-netns.** Inside a container, `/proc/net/tcp` shows *that container's* sockets, not the host's. To see another namespace, read `/proc/<pid>/net/tcp` where `<pid>` is in that namespace.

**Things can disappear mid-read.** Process exits while you're reading `/proc/<pid>/`: reads return `ESRCH` or 0 bytes. Scripts that iterate over `/proc/*` need to handle this gracefully.

**Text format changes are rare but do happen.** The kernel tries hard, but new fields get appended to `stat`, `status`, etc. Parse by name when possible, not by positional offset.

**`/proc/pid/status` vs `/proc/pid/stat`.** `status` is pretty, fielded, human-readable. `stat` is one line, space-separated, positional (the format `ps` has parsed forever). `stat` is faster to parse but has gotchas (the `comm` field can contain spaces and is parenthesized — parse backwards from the end to avoid confusion).

**`MemFree` isn't "memory available."** Reclaimable cache isn't in `MemFree` but is available if needed. Use `MemAvailable`.

**`/proc/cpuinfo` on virtualized systems lies about cores.** Shows vCPUs. Use `lscpu` or `/sys/devices/system/cpu/` for the richer view.

**Binary files in procfs.** A few files are binary: `/proc/<pid>/pagemap`, `/proc/kcore`, `/proc/<pid>/mem`. Don't `cat` them.

**`/proc/self/exe` can point to deleted files.** If a binary is replaced while running, the symlink target shows `(deleted)`. The kernel still has the original inode; you can copy the binary back via `cp /proc/<pid>/exe /tmp/recovered`.

**Container-in-container sees "host" /proc.** Unless you remount procfs with proper namespacing, `/proc/1` is the outer init, not the container's. Modern container runtimes mount their own procfs correctly; custom setups may not.

**`/proc/loadavg` is a 1/5/15 minute decay average.** A spike lasting 30 seconds barely moves the 15-minute number. For instantaneous load, use PSI or compute from `/proc/stat` deltas.

**Writing to procfs isn't always transactional.** Some knobs take effect immediately, some require a specific write pattern (certain sysctl bits). Read the doc for anything unusual.

## Documentation

The kernel ships comprehensive documentation for procfs:

```bash
# In the kernel source tree or installed docs:
less /usr/share/doc/kernel-doc*/Documentation/filesystems/proc.rst
# or online:
# https://www.kernel.org/doc/html/latest/filesystems/proc.html
```

This file is the authoritative reference — every field in `/proc/<pid>/status`, every file under `/proc/`, documented. It's long (~3000 lines) but well-organized. The single best resource.

`man 5 proc` is the abridged version; covers most practical needs.

## How procfs fits with everything else

- **vs. sysfs.** procfs = processes + legacy kernel state. sysfs = structured hardware/driver model. New subsystems go to sysfs.
- **vs. syscalls.** Many procfs reads duplicate syscall info (e.g., `/proc/self/status` has stuff from `getrlimit`, `getrusage`, `prctl`). procfs is for inspection; syscalls are for action.
- **vs. netlink.** Structured, binary, bidirectional. Faster for frequent updates and programmatic use. procfs is simpler to script against and easier to inspect by hand.
- **vs. BPF.** BPF is for high-frequency, low-overhead observation with kernel hooks. procfs is for point-in-time snapshots. Different tools, different time scales.
- **vs. tracefs / ftrace.** Tracing is event-driven; procfs is state-driven. To ask "is this happening?", use ftrace/BPF. To ask "what's happening right now?", use procfs.
- **vs. cgroupfs.** cgroup v2's `/sys/fs/cgroup/` mirrors procfs's style (text-based, simple reads/writes) but with explicit hierarchy. Some procfs fields (`/proc/<pid>/cgroup`) reference cgroups.
- **vs. `ps`/`top`/etc.** Those *are* procfs clients. Nothing in them is privileged or magic; they just parse.

## Quick reference card

```bash
# Per-process
/proc/<pid>/cmdline        # NULL-separated args
/proc/<pid>/comm           # command name (short)
/proc/<pid>/status         # human-readable status
/proc/<pid>/stat           # machine-readable status
/proc/<pid>/exe            # symlink to binary
/proc/<pid>/cwd            # symlink to cwd
/proc/<pid>/fd/            # open fds
/proc/<pid>/fdinfo/        # fd metadata
/proc/<pid>/maps           # memory mappings
/proc/<pid>/smaps          # detailed memory per region
/proc/<pid>/smaps_rollup   # aggregated memory (faster)
/proc/<pid>/io             # I/O counters
/proc/<pid>/limits         # rlimits
/proc/<pid>/stack          # kernel stack trace
/proc/<pid>/wchan          # what blocked in
/proc/<pid>/ns/            # namespace links
/proc/<pid>/task/          # threads
/proc/<pid>/cgroup         # cgroups
/proc/self                 # magic symlink to current PID

# System overview
/proc/cpuinfo              # CPUs
/proc/meminfo              # memory
/proc/stat                 # CPU/interrupt counters
/proc/loadavg              # load
/proc/uptime               # up time
/proc/version              # kernel version
/proc/cmdline              # kernel boot args
/proc/modules              # loaded modules
/proc/interrupts           # IRQ counters per CPU
/proc/diskstats            # per-block-device I/O
/proc/mounts               # mount table (=/proc/self/mounts)
/proc/filesystems          # supported FS types
/proc/partitions           # block devices
/proc/swaps                # swap areas

# Memory details
/proc/vmstat               # VM counters
/proc/buddyinfo            # free memory by order
/proc/zoneinfo             # per-zone details
/proc/pressure/{cpu,memory,io}   # PSI

# Networking (per netns)
/proc/net/dev              # per-interface packet counters
/proc/net/tcp, tcp6        # TCP sockets
/proc/net/udp, udp6        # UDP sockets
/proc/net/route            # routing table
/proc/net/netstat          # protocol stats
/proc/net/snmp             # IP/ICMP/TCP/UDP SNMP-style counters
/proc/net/netlink          # netlink sockets

# Kernel tunables
/proc/sys/                 # entry point for sysctl(8)
/proc/sys/kernel/          # general
/proc/sys/vm/              # virtual memory
/proc/sys/net/             # networking
/proc/sys/fs/              # filesystem

# Tools that are just procfs clients
ps, top, htop, free, uptime, vmstat, lsof, pmap, mpstat,
pidstat, netstat, ifconfig (partly), fuser, sysctl, lscpu, lsmod

# Docs
man 5 proc                       # short reference
Documentation/filesystems/proc.rst   # exhaustive
```

## Where to learn more

- **`man 5 proc`** — the best single man page in Linux. Covers everything, compact.
- **`Documentation/filesystems/proc.rst`** — the kernel's own docs. More verbose, more current, includes fields added recently. Read both.
- **`Documentation/admin-guide/sysctl/`** — per-subsystem sysctl knob documentation (kernel.rst, vm.rst, net/, fs.rst, etc.).
- **`strace` on your favorite tools.** Seeing which procfs files `ps`, `top`, `lsof` read teaches you the canonical mappings.
- **`Documentation/accounting/`** — the I/O counter semantics in `/proc/<pid>/io`.
- **The kernel source under `fs/proc/`** — if you want to know *exactly* what a file reports, `grep` in the code. Short, readable C.

The mental model to carry: **procfs is a virtual filesystem exposing kernel state as text files — one directory per process under `/proc/<pid>/`, one subtree of kernel tunables under `/proc/sys/`, and a collection of system-wide files for memory, CPU, I/O, network, and subsystem state at the top level — all generated on demand when read, all parseable with ordinary tools, and the foundation for every Linux observability utility you've ever used.** Once you internalize that `ps`/`top`/`lsof`/`free` are just awk scripts over `/proc`, the kernel becomes dramatically more legible — you can answer any system-state question with a shell and a little patience, and you know exactly where to reach for custom telemetry.