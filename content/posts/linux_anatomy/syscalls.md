---
date: '2026-04-19T20:58:01-07:00'
draft: true
title: 'Chapter 1: Syscalls'
weight: 1
---
## What a syscall is and why it exists

The fundamental division in an operating system: **user mode** can compute, access its own memory, and not much else. **Kernel mode** can touch hardware, access any memory, configure the CPU, and generally do anything. A userspace program running forever in user mode can add numbers together but can't open a file, allocate memory from the OS, send a packet, or even exit cleanly.

A **syscall** is the gate between the two. It's a controlled crossing: the CPU switches privilege level, jumps to a specific kernel entry point, runs kernel code on the process's behalf, then returns to user mode with a result. The kernel defines a fixed set of these gates — on x86_64 Linux, around 450 of them — and anything a program wants to do that requires privilege goes through one.

Every library function you've ever called bottoms out in syscalls, eventually. `printf` calls `fwrite` calls `write`. `malloc` uses `mmap` and/or `brk`. `fopen` calls `open`. `exec*()` family calls `execve`. Network libraries call `socket`, `connect`, `recvfrom`. Even measuring time (`clock_gettime`) can be a syscall, though there's a clever optimization for that (vDSO, later).

**Why this matters for debugging and understanding:** if you can see what syscalls a program is making and what they return, you can understand almost anything it's doing. `strace` is a superpower for exactly this reason.

## The mechanical path of a syscall

What actually happens when your program calls `write(1, "hi\n", 3)`:

```
  userspace                                  kernel
  ─────────                                  ──────
  1. put args in registers                
     (x86_64 SysV: rdi, rsi, rdx, r10, r8, r9)
  2. put syscall number in rax              
  3. execute SYSCALL instruction   ──────▶  4. CPU switches to ring 0
                                            5. jumps to entry_SYSCALL_64
                                            6. saves user registers
                                            7. looks up sys_call_table[rax]
                                            8. calls the handler (sys_write)
                                            9. handler does the work
                                           10. return value placed in rax
                                           11. SYSRET switches back to ring 3
 4. continue with rax as return value
```

The `SYSCALL` instruction on x86_64 is a dedicated CPU primitive introduced with AMD64; earlier x86 used `int 0x80` (software interrupt) or `SYSENTER`. ARM64 uses `SVC`. RISC-V uses `ECALL`. The specific instruction varies; the model is the same — a privileged-transition primitive that the OS has pre-configured to land in a well-defined entry point.

**Syscall numbers are ABI.** Once a syscall number is assigned, it's forever. Linux's `syscalls_64.tbl` in the kernel source is append-only. This is why you can run a binary compiled in 2005 against a 2026 kernel and it still works. Contrast with library functions, which version with the library.

**Each architecture has its own numbering.** `write` is syscall 1 on x86_64, but 64 on arm64. The numbers aren't portable across architectures, but the syscall _names_ and _semantics_ generally are.

## You almost never call syscalls directly

In practice, userspace calls **libc wrappers** that bundle the syscall. glibc's `write()` function:

1. Sets up errno handling.
2. Puts args in registers.
3. Executes the `SYSCALL` instruction.
4. Checks the return value for error range.
5. If error, sets `errno` and returns -1.
6. Otherwise returns the value.

The libc wrapper is why you set `errno` and check for `-1`, even though the raw kernel ABI is different (negative values -4095..-1 are errors; libc translates them to the `-1`/`errno` convention).

You _can_ make syscalls directly with the `syscall(2)` function, which is a libc wrapper that lets you invoke a syscall by number (useful for syscalls glibc hasn't added wrappers for):

```c
#include <sys/syscall.h>
#include <unistd.h>

long tid = syscall(SYS_gettid);    // no glibc wrapper for gettid historically
```

Static Go binaries and languages that bypass libc (Go's runtime, some Rust with `no_std`) emit syscall instructions directly. That's fine; the kernel ABI is the real interface. libc is just a convenient wrapper.

## The major families

~450 syscalls sounds like a lot, but they group into coherent families. If you know what's in each family, you can find what you need by category.

### Process lifecycle

|Syscall|Purpose|
|---|---|
|`fork` / `vfork` / `clone` / `clone3`|Create a new process/thread. `clone` is the primitive; `fork` is a thin wrapper.|
|`execve` / `execveat`|Replace the current process image with a new program.|
|`exit` / `exit_group`|Terminate (single thread / whole process).|
|`wait4` / `waitid`|Wait for child termination, collect status.|
|`getpid` / `getppid` / `gettid`|Identity queries.|
|`setuid` / `setgid` / `setresuid` / `capset`|Change credentials and capabilities.|
|`prctl` / `arch_prctl`|Per-process control knobs (hundreds of sub-operations).|

`fork+exec` is the Unix model for running a program: duplicate yourself, then replace the child's image with the target binary. Everything your shell does to run a command uses this pair.

`clone` is the general-purpose primitive — `fork`, pthreads, containers all call into it with different flags (`CLONE_VM`, `CLONE_NEWNS`, `CLONE_NEWUSER`, etc.). Namespaces, the foundation of containers, are clone flags.

### File I/O

|Syscall|Purpose|
|---|---|
|`open` / `openat` / `openat2`|Open a path, return an fd.|
|`close`|Release an fd.|
|`read` / `pread64` / `readv`|Read from fd.|
|`write` / `pwrite64` / `writev`|Write to fd.|
|`lseek`|Reposition file offset.|
|`fsync` / `fdatasync` / `syncfs`|Flush to disk.|
|`dup` / `dup2` / `dup3`|Duplicate an fd.|
|`pipe` / `pipe2`|Create a pipe pair.|
|`fcntl`|General fd control (flags, locks, FD_CLOEXEC).|
|`ioctl`|Arbitrary driver-specific operations.|

This is the beating heart of Unix. Almost everything — files, sockets, pipes, devices, timers, signals, event queues — is an fd, so this family applies universally.

**`*at` variants** (`openat`, `fstatat`, `unlinkat`, etc.) take a directory fd and a relative path. Safer than path-based versions because the path resolution is anchored to a specific directory, immune to races where the path changes underfoot. Modern code prefers them.

### Filesystem metadata

|Syscall|Purpose|
|---|---|
|`stat` / `fstat` / `lstat` / `statx`|Get file metadata.|
|`chmod` / `fchmod` / `fchmodat`|Change permissions.|
|`chown` / `fchown` / `fchownat`|Change ownership.|
|`rename` / `renameat2`|Move/rename.|
|`link` / `symlink` / `readlink`|Hardlinks and symlinks.|
|`unlink` / `rmdir`|Remove.|
|`mkdir` / `mkdirat`|Create directories.|
|`getdents64`|List directory contents (what `ls` is built on).|
|`mount` / `umount2` / `pivot_root`|Filesystem mounting.|
|`statfs` / `fstatfs`|Filesystem-wide info (free space, etc.).|

`statx` is the modern catch-all; it's extensible and can request specific fields, which matters on filesystems where some attributes are expensive to compute.

### Memory management

|Syscall|Purpose|
|---|---|
|`mmap` / `mremap` / `munmap`|Map memory regions (anonymous or file-backed).|
|`mprotect`|Change permissions (RWX) on a region.|
|`madvise`|Hint the kernel about access patterns.|
|`brk` / `sbrk`|Grow/shrink the heap (rarely used directly; libc uses mmap for big allocs).|
|`mlock` / `munlock` / `mlockall`|Pin pages in RAM (prevent swapping).|
|`userfaultfd`|Userspace page fault handling.|

`malloc` in glibc uses `mmap` for large allocations and `brk` for small ones from an arena. Every significant program you run has mmap'd its executable and libraries — `pmap <pid>` or `cat /proc/<pid>/maps` shows the layout.

### Signals

|Syscall|Purpose|
|---|---|
|`rt_sigaction`|Install a signal handler.|
|`rt_sigprocmask`|Block/unblock signals.|
|`kill` / `tgkill` / `pidfd_send_signal`|Send a signal.|
|`sigaltstack`|Alternate stack for handlers (useful for SIGSEGV).|
|`signalfd`|Receive signals via an fd instead of async interrupts.|

Signals are Unix's asynchronous notification mechanism — `SIGINT` on Ctrl-C, `SIGCHLD` when a child exits, `SIGSEGV` on a bad memory access. Handling them correctly is notoriously hard (restrictions on what's safe inside a handler); `signalfd` turns the asynchronous problem into a synchronous one at some cost.

### Networking (sockets)

|Syscall|Purpose|
|---|---|
|`socket` / `socketpair`|Create sockets.|
|`bind` / `listen` / `accept4`|Server side.|
|`connect`|Client side.|
|`sendto` / `recvfrom` / `sendmsg` / `recvmsg`|Data transfer.|
|`shutdown`|Half-close a connection.|
|`getsockopt` / `setsockopt`|Socket options.|
|`getsockname` / `getpeername`|Address queries.|

Sockets are fds, so they also work with `read`, `write`, `close`, `select`, etc. The socket-specific syscalls handle connection setup and options.

### I/O multiplexing and async

|Syscall|Purpose|
|---|---|
|`select` / `pselect6`|Classic, O(n), awkward. Still used.|
|`poll` / `ppoll`|Like select but with a saner API.|
|`epoll_create1` / `epoll_ctl` / `epoll_wait`|Scalable event notification. The dominant approach on Linux.|
|`io_uring_setup` / `io_uring_enter` / `io_uring_register`|Modern async I/O via shared ring buffers.|
|`timerfd_create`|Timers as fds.|
|`eventfd`|User-defined fd for signaling.|
|`inotify_init` / `fanotify_init`|Filesystem change notification.|

**epoll** is how high-performance servers scale. 10,000 connections with epoll costs about the same as 100.

**io_uring** (2019+) is the newest and most interesting — submission and completion queues shared between userspace and kernel, so you can batch thousands of I/O operations with minimal syscalls. Fundamentally changes the cost model. Used by modern databases, proxies, and increasingly by runtimes.

### Time

|Syscall|Purpose|
|---|---|
|`clock_gettime`|Current time (various clocks: REALTIME, MONOTONIC, BOOTTIME).|
|`clock_nanosleep` / `nanosleep`|Sleep with nanosecond precision.|
|`getitimer` / `setitimer` / `timer_create`|Timers.|
|`adjtimex` / `clock_adjtime`|NTP clock adjustment.|

`clock_gettime` is the only syscall most programs call often, which is why it got special treatment via the vDSO (below).

### IPC

|Syscall|Purpose|
|---|---|
|`pipe2`|Anonymous pipe.|
|`mq_open` / `mq_send` / `mq_receive`|POSIX message queues.|
|`shmget` / `shmat` / `shmdt`|SysV shared memory (legacy).|
|`memfd_create`|Anonymous memory as an fd (modern, passable between processes).|
|`futex`|The primitive under all userspace locks.|

**`futex`** is worth knowing about even if you never call it directly. Every mutex, semaphore, and condition variable in glibc's pthreads is built on `futex`. The fast path of lock/unlock is a userspace atomic CAS; only contended cases call into the kernel via `futex`.

### Security and isolation

|Syscall|Purpose|
|---|---|
|`seccomp`|Install syscall filters (BPF-based).|
|`unshare`|Detach parts of the process's context (create new namespaces).|
|`setns`|Join an existing namespace.|
|`capget` / `capset`|Manipulate capabilities (fine-grained privilege).|
|`prctl(PR_SET_NO_NEW_PRIVS)`|Forbid gaining privileges via setuid etc.|
|`pivot_root`|Swap root filesystem (containers).|
|`landlock_create_ruleset`|Sandboxing a process's filesystem access.|
|`pidfd_open` / `pidfd_send_signal`|Race-free process references.|

Containers are built almost entirely from these: namespaces via `unshare`/`clone`, cgroups via filesystem syscalls against `/sys/fs/cgroup`, filesystem isolation via `pivot_root`, syscall filtering via `seccomp`. "Docker" is a userspace daemon that orchestrates syscalls — there's no "container syscall."

### BPF

|Syscall|Purpose|
|---|---|
|`bpf`|Load programs, create maps, attach to hooks. Dozens of sub-operations.|

A single syscall with a command enum, similar to `ioctl` in style. BPF is the fastest-growing kernel interface — tracing, networking (XDP), security (LSM-BPF), scheduling, all extensible via BPF programs attached via this syscall.

## Error handling

Syscalls return a single value. On the raw kernel ABI, negative values in the range -4095..-1 are errors (the negative of an `errno` code); everything else is a success value. libc wrappers translate this to the familiar `-1` with `errno` set.

The core set of errno values you'll meet constantly:

|errno|Meaning|
|---|---|
|`EPERM`|Operation not permitted (capability-level deny).|
|`EACCES`|Permission denied (ACL-level deny — note the distinction).|
|`ENOENT`|No such file or directory.|
|`EEXIST`|File exists (when you didn't want it to).|
|`EINTR`|Interrupted by signal; retry.|
|`EAGAIN` / `EWOULDBLOCK`|Would block in non-blocking mode; try later.|
|`EINVAL`|Invalid argument.|
|`ENOMEM`|Out of memory.|
|`ENOTTY`|Inappropriate ioctl (wrong fd type).|
|`EBADF`|Bad fd (not open, wrong kind).|
|`ENXIO`|No such device or address.|
|`EPIPE`|Writing to a pipe with no readers.|
|`ETIMEDOUT`|Operation timed out.|
|`ECONNREFUSED` / `ECONNRESET` / `EHOSTUNREACH`|Network errors.|

`man 3 errno` has the whole list; it's worth skimming once to recognize the categories.

**`EINTR` is a gotcha.** Many blocking syscalls return `-1/EINTR` when interrupted by a signal, and your code has to retry. glibc can install signal handlers with `SA_RESTART` to make this automatic for some syscalls, but not all are restartable. Correctly-written code either loops on `EINTR` or uses `signalfd` to sidestep the issue.

## Blocking, non-blocking, and async

Three models for handling I/O that might take time:

**Blocking (default).** `read(fd, buf, 1024)` parks the thread until data is available. Simple, inefficient under concurrency.

**Non-blocking.** `fcntl(fd, F_SETFL, O_NONBLOCK)`, then `read` returns `-1/EAGAIN` if no data. You spin or multiplex. The whole point of `epoll` is to tell you when non-blocking fds are ready without polling them all.

**Asynchronous.** You submit an operation and get notified when done, continuing other work in the meantime. Linux has several attempts at this:

- POSIX AIO (libaio) — works only for direct I/O on files; awkward.
- io_uring — modern, works for everything, fastest. Submits requests to a ring buffer the kernel consumes; completions appear on another ring.

If you're writing a server or performance-sensitive tool today, the choice is **epoll for event-driven**, **io_uring for maximum throughput**.

## Observability: strace and friends

The single most useful thing you can do to understand a program is watch its syscalls. `strace` is the canonical tool.

```bash
strace ls                                  # trace every syscall
strace -f make                             # follow forks and threads
strace -e openat ls                        # only specific syscalls
strace -e 'trace=!write' cmd               # exclude some
strace -e trace=network curl example.com   # a whole category
strace -c cmd                              # summary: count and total time per syscall
strace -T cmd                              # per-call elapsed time
strace -tt cmd                             # wall-clock timestamps
strace -y cmd                              # show fd paths (what 3 is actually pointing to)
strace -yy cmd                             # even more (protocol info for sockets)
strace -p <pid>                            # attach to a running process
strace -o trace.log cmd                    # to a file
strace -s 256 cmd                          # longer string truncation (default 32)
```

Reading strace output teaches you what the system is really doing. `ls` is roughly: `openat(".")`, `getdents64` in a loop, `statx` on each entry, `write(1, ...)` to print, `close`, `exit_group`. No magic.

`strace` works via `ptrace(2)`, stopping the process on every syscall. This imposes real overhead (often 20-100x slowdown). For production or perf-sensitive tracing, reach for:

- **`perf trace`** — similar UI, uses ftrace/perf events instead of ptrace. Lower overhead.
- **`bpftrace`** — one-liners using BPF. `bpftrace -e 'tracepoint:syscalls:sys_enter_openat { printf("%s %s\n", comm, str(args->filename)); }'`. Near-zero overhead, system-wide.
- **`bcc` tools** — `opensnoop`, `execsnoop`, `syscount`, dozens more. BPF-powered, production-safe.

## vDSO: syscalls that aren't

Some operations are so common and so simple that the overhead of a syscall (hundreds of cycles) matters. Linux solves this with the **vDSO** (virtual dynamic shared object) — a small library the kernel maps into every process's address space, exposing a few functions that implement some "syscalls" purely in userspace.

The usual suspects in the vDSO:

- `clock_gettime`
- `gettimeofday`
- `time`
- `getcpu`

These read time from a shared memory region the kernel updates. No ring transition. Benchmarks that claim "my program does a million `clock_gettime` calls per second" are possible because those aren't really syscalls.

`strace` doesn't see vDSO calls (they don't enter the kernel). You can see the vDSO mapped in `/proc/self/maps`:

```bash
cat /proc/self/maps | grep vdso
```

## The kernel-userspace ABI and stability

Linus's famous rule: **"We do not break userspace."** The kernel-userspace ABI — syscall numbers, argument layouts, semantics — is effectively forever. Once a syscall is added, removing or changing it breaks every binary that uses it.

This is why:

- Syscall numbers are never reused.
- New functionality usually gets new syscalls (`openat2` alongside `openat`, `clone3` alongside `clone`) rather than modifying old ones.
- Obsolete syscalls stick around. `fstat` still works even though `statx` supersedes it.
- Glibc version mismatches are a bigger deal than kernel version mismatches. An old kernel with a new glibc might miss syscalls. An old binary on a new kernel almost always works.

Contrast this with the in-kernel API (between modules and the kernel core), which changes constantly. The stable boundary is at the syscall layer, not anywhere internal.

## Adding a syscall (the other side, briefly)

To add a new syscall, a kernel developer:

1. Picks the next number in each `syscalls_*.tbl` (one per arch).
2. Writes the handler using `SYSCALL_DEFINEn(name, ...)` macros.
3. Adds a declaration in `<linux/syscalls.h>`.
4. Writes man pages and selftests.
5. Gets it reviewed on `linux-api@` for ABI soundness.

The bar is high. New syscalls take months of bikeshedding. Often features get added as sub-commands of existing multi-op syscalls (`bpf`, `prctl`, `ioctl`, `fcntl`) precisely to avoid the permanent-ABI commitment. This is why `bpf` has dozens of sub-commands — adding one is cheaper than a new syscall.

## Architecture differences

While syscall _names_ and _semantics_ are largely portable, the _ABI_ varies:

|Arch|Invocation|Args in|
|---|---|---|
|x86_64|`SYSCALL`|rdi, rsi, rdx, r10, r8, r9|
|i386|`int 0x80` or `SYSENTER`|ebx, ecx, edx, esi, edi, ebp|
|arm64|`SVC #0`|x0..x5|
|riscv64|`ECALL`|a0..a5|

Syscall numbers differ, too. `grep write /usr/include/asm-generic/unistd.h` etc., or better, `ausyscall --dump` or the kernel's `Documentation/arch/*/syscalls.rst`.

## 32-bit compat, year-2038, and why modern ABIs changed

A subtle thing: historical syscalls used `long`-sized (or worse) time and size types, which are 32-bit on 32-bit systems. That causes:

- **Y2038 problem** — `time_t` as 32-bit signed seconds overflows in January 2038.
- **Large-file problem** — `off_t` as 32-bit limits files to 2 GB.
- **Stat struct churn** — old `stat` couldn't represent nanosecond timestamps, large inodes, etc.

Modern glibc and kernel have newer syscalls (`statx`, `clock_gettime64`, `futex_time64`) that use 64-bit fields always, and 32-bit distros are transitioning to use them. On x86_64, you already use them transparently.

This is visible in `strace` — you'll see `clock_gettime` on x86_64 but `clock_gettime64` on armhf.

## Security boundary and attack surface

The syscall interface is the main attack surface between unprivileged code and the kernel. An exploitable bug in a syscall handler is a privilege escalation. This drives several defensive measures:

- **seccomp** — userspace can filter its own syscalls. Containers, sandboxes, and browsers aggressively reduce syscall attack surface. Chrome's renderer has ~150 allowed syscalls; the rest return `-EPERM`.
- **SELinux / AppArmor** — per-process MAC policies that can intercept and deny syscalls based on context and target.
- **Landlock** — a newer unprivileged sandboxing syscall for apps to restrict themselves.
- **Pledge/Unveil (OpenBSD concept)** — not in Linux but inspired things like Landlock.

`seccomp` is the one that's most often invoked explicitly. Programs can drop privilege to a narrow syscall whitelist on startup — a huge defense-in-depth win.

## Namespaces and containers

Containers are built from syscalls. A container is a process (or group) running with:

- **Separate namespaces** (mount, PID, network, UTS, IPC, user, cgroup) — created via `unshare` or `clone` flags.
- **A pivoted root filesystem** (`pivot_root`) — the container's view of `/`.
- **cgroup membership** — resource limits via the cgroup filesystem.
- **A seccomp filter** — restricted syscall set.
- **Dropped capabilities** — via `capset`.

There's no "container" primitive; containers are a policy composed from these primitives. This is why you can have Docker, Podman, LXC, systemd-nspawn, Firecracker — they're all different userspace orchestrations of the same kernel syscalls.

## Tutorial: watch what a simple command actually does

```bash
strace -f -e trace=%file,%process -o trace.log -- sh -c 'echo hi'
less trace.log
```

You'll see the full opening of `sh`, mapping of libc, the `execve` of `sh` (or `-c` inline), the echo (which is usually a shell builtin — so no exec), the write to stdout, and exit. Roughly 30-50 syscalls for the simplest command.

Do it with `python -c 'print("hi")'` and you'll see hundreds — Python loads a lot on startup.

Do it with a Rust/Go binary (which doesn't use libc/dynamic loading): dramatically fewer.

This is one of the best intuition-builders for how programs are actually structured.

## The syscall as performance unit

Each syscall crossing costs roughly 100-500 nanoseconds on modern x86_64, depending on the specific call and whether mitigations like Spectre defenses are in effect. Sounds fast, but at millions of operations per second, it dominates.

Strategies to reduce syscall overhead:

- **Batch.** `readv`/`writev` can do one syscall for multiple buffers. `sendmmsg`/`recvmmsg` for multiple UDP packets.
- **Larger buffers.** One `read(fd, buf, 65536)` is cheaper than 65536 `read(fd, buf, 1)`.
- **io_uring.** Submit N operations in one syscall, collect completions in one syscall.
- **vDSO.** For the few things that qualify.
- **mmap.** Read a file as memory instead of repeatedly `read`-ing.
- **Splice/sendfile.** Move data between fds without round-tripping through userspace.

High-performance network code lives and dies by syscall count per request.

## Things that trip people up

**Partial reads and writes.** `read(fd, buf, N)` can return less than N even in normal operation. Short writes to sockets or pipes are especially common. Always loop.

**Thinking fork is cheap.** It's cheaper than you think (copy-on-write), but `execve` on top of it still costs. If you're forking a lot, `posix_spawn` or direct `clone+execve` can be faster.

**Ignoring EINTR.** Any blocking syscall can return EINTR. Long-running loops that don't handle it become unkillable or flaky.

**Confusing errno semantics.** After a successful syscall, `errno` is _not_ reset. Only check it when the syscall indicated failure (return value of -1 for most, NULL or 0 for some).

**Signal handlers and syscalls.** Only async-signal-safe functions (a short list in `man 7 signal-safety`) are safe inside a handler. `malloc`, `printf`, most of stdlib — not safe.

**Thread safety of errno.** It's a thread-local variable in modern libc. On very old systems it wasn't. Assume TLS now.

## Observing and understanding the landscape

A few commands to get comfortable with:

```bash
# How many syscalls does glibc know?
grep -c '^SYSCALL' /usr/include/asm/unistd_64.h

# Which syscalls does this program use, with frequency?
strace -c -f <cmd>

# System-wide syscall rate, no overhead
sudo perf stat -e 'syscalls:*' -a sleep 5

# Top syscalls per process, live
sudo bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm, args->id] = count(); }'

# What files did this thing touch?
strace -f -e trace=file -o files.log <cmd>

# Deep dive: manual page for any syscall
man 2 openat
```

## How syscalls fit with everything else

- **vs ioctl** — ioctl is a syscall (`ioctl(2)`), dispatched to a driver. Syscalls are the generic mechanism; ioctl is one opaque, per-driver dispatch point within that mechanism.
- **vs D-Bus** — syscalls talk to the kernel. D-Bus talks to userspace daemons. A `systemctl` invocation makes syscalls (connect to D-Bus socket, send bytes, read reply) that implement a D-Bus exchange with PID 1.
- **vs shared libraries** — libraries are userspace code that eventually calls syscalls. Same CPU rings the whole time; no privilege transition. You can bypass them (write your own syscall wrappers) if you want.
- **vs sysfs/procfs** — reading `/proc/cpuinfo` is an `openat`+`read`+`close`. The kernel fakes a file to serialize state. So it's syscalls all the way down; sysfs/proc just reuse the file interface for read-only state.
- **vs netlink** — netlink messages travel over a socket, which means `socket`/`sendmsg`/`recvmsg` syscalls. Different content, same mechanism.
- **vs BPF** — BPF programs run in the kernel without a syscall per event. You attach them via the `bpf` syscall, but their execution is purely kernel-side, triggered by hooks.

Essentially: **everything userspace does that touches anything outside the process is mediated by syscalls.** They're the fundamental operations; almost every other mechanism (files, sockets, D-Bus, ioctl, BPF, namespaces, whatever) is a particular use of the syscall interface.

## Quick reference card

```bash
# Observability
strace <cmd>                              # every syscall
strace -f -e openat,execve <cmd>          # filter + follow forks
strace -c <cmd>                           # counts and totals
strace -p <pid>                           # attach to running
perf trace <cmd>                          # lower-overhead strace
sudo bpftrace -l 'tracepoint:syscalls:*'  # enumerable BPF hooks

# Reference
man 2 <syscall>                           # man pages are in section 2
man 7 signal-safety                       # what's safe in handlers
grep -rn 'SYSCALL_DEFINE' /usr/src/linux/ # kernel source, if installed
/usr/include/asm/unistd_64.h              # numbers on x86_64
/usr/include/linux/                       # struct definitions
```

## Where to learn more

- **`man 2 <anything>`** — comprehensive and mostly excellent. `man 2 intro` is a good starting point.
- **Kerrisk's "The Linux Programming Interface"** — the book. Thorough, canonical, 1500 pages. If you're serious about systems programming, read it.
- **`Documentation/userspace-api/` in the kernel source** — the up-to-date authoritative references.
- **`lwn.net`** — especially articles tagged "syscall". LWN is the best long-form reporting on kernel-userspace interface evolution (io_uring, pidfd, landlock, etc., all covered in depth).
- **Strace your tools for a day.** The single fastest way to build intuition. Pick a program, trace it, read the trace, understand every line.

The mental model to carry: **syscalls are the kernel's API — a fixed set of named operations, each identified by a number, invoked via a special instruction that transitions from user to kernel mode, returning a value and possibly an error.** Everything userspace does that requires the kernel — files, memory, processes, networking, timing, devices — is ultimately a syscall or a short chain of them. Fluency with `strace`, a mental model of the major families, and a sense of their cost and failure modes is the baseline for understanding how Linux actually works.