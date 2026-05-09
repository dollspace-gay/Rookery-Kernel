---
title: "Tier-3: fs/proc/proc-kmsg — /proc/kmsg (kernel printk consumer)"
tags: ["design-doc", "tier-3", "fs", "procfs"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for `/proc/kmsg` — the legacy interface for consuming kernel `printk()` output. Used by:
- `klogd` (legacy kernel-log daemon; reads from /proc/kmsg + writes to syslog)
- `syslogd` (when configured to read from /proc/kmsg)
- `dmesg` (with `-c` flag; alternative to `syslog(2)` syscall path)

Modern systems prefer `/dev/kmsg` (cross-ref `kernel/printk/00-overview.md` Tier-3 if added) for non-destructive reads; `/proc/kmsg` is destructive — once read, log entries are removed from the kernel ring buffer (per the `SYSLOG_ACTION_READ` semantic). Most systemd-managed distros disable klogd entirely and consume kernel logs via `kmsg` device (which doesn't drain).

`/proc/kmsg` is essentially a thin procfs wrapper around the `do_syslog(SYSLOG_ACTION_READ)` syscall path; the actual ring-buffer + per-message handling lives in `kernel/printk/printk.c`.

Sub-tier-3 of `fs/proc/00-overview.md`. Pairs with future `kernel/printk/00-overview.md` (where /dev/kmsg + actual ring buffer + syslog(2) syscall live).

### Requirements

- REQ-1: `/proc/kmsg` open requires CAP_SYS_ADMIN (or CAP_SYSLOG, per dmesg_restrict).
- REQ-2: Open exclusivity: second concurrent open returns -EBUSY.
- REQ-3: Read returns next chunk of next-unread kernel printk message; format byte-identical (`<level>kernel-message-text\n`).
- REQ-4: Read is destructive — consumed bytes removed from ring buffer (matches `do_syslog(SYSLOG_ACTION_READ, ...)`).
- REQ-5: Block on empty buffer; O_NONBLOCK returns -EAGAIN.
- REQ-6: poll(POLLIN) when ring buffer has unread bytes.
- REQ-7: Bridge to `do_syslog(SYSLOG_ACTION_READ, ...)`; per-system permission gate consistent with syslog(2) syscall.
- REQ-8: dmesg_restrict sysctl honored: 0 (any user) / 1 (CAP_SYSLOG only).
- REQ-9: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: Open as root succeeds; open as unprivileged user with dmesg_restrict=1 returns -EPERM. (covers REQ-1, REQ-8)
- [ ] AC-2: Concurrent open test: process A opens /proc/kmsg; process B's open returns -EBUSY. (covers REQ-2)
- [ ] AC-3: Read test: trigger printk via test-module; subsequent read returns the message bytes byte-identical to dmesg output. (covers REQ-3)
- [ ] AC-4: Destructive test: read N bytes from /proc/kmsg; subsequent dmesg shows those bytes are gone (consumed). (covers REQ-4)
- [ ] AC-5: Blocking test: open /proc/kmsg; ring buffer empty; read blocks; printk fires; read unblocks + returns message. (covers REQ-5)
- [ ] AC-6: O_NONBLOCK test: open with O_NONBLOCK; empty buffer → read returns -EAGAIN. (covers REQ-5)
- [ ] AC-7: poll test: poll() on /proc/kmsg fd returns POLLIN when ring buffer has unread bytes. (covers REQ-6)
- [ ] AC-8: Bridge test: open /proc/kmsg as klogd-equivalent + read 1MB; identical to syslog(SYSLOG_ACTION_READ) output. (covers REQ-7)
- [ ] AC-9: dmesg_restrict toggle test: write `0` to /proc/sys/kernel/dmesg_restrict; unprivileged user can now open + read. (covers REQ-8)
- [ ] AC-10: Hardening section present and follows template. (covers REQ-9)

### Architecture

### Rust module organization

- `kernel::fs::proc::kmsg::Kmsg` — top-level /proc/kmsg entry
- `kernel::fs::proc::kmsg::Open` — exclusive-open path + permission check
- `kernel::fs::proc::kmsg::Read` — bridge to `do_syslog(ACTION_READ)`
- `kernel::fs::proc::kmsg::Poll` — poll() integration
- `kernel::printk::Syslog` — `do_syslog(...)` framework (cross-ref future `kernel/printk/00-overview.md`)
- `kernel::printk::RingBuffer` — kernel printk ring buffer

### Locking and concurrency

- **`syslog_lock`** (mutex): per-system /proc/kmsg open exclusivity
- **`log_wait`** (waitqueue): blocks read on empty buffer
- **`logbuf_lock`** (per-system spinlock): kernel ring-buffer reader/writer coordination

### Error handling

- `Err(EBUSY)` — second concurrent open
- `Err(EPERM)` — non-CAP_SYS_ADMIN / non-CAP_SYSLOG with dmesg_restrict=1
- `Err(EAGAIN)` — O_NONBLOCK + empty buffer
- `Err(EFAULT)` — userspace buffer fault
- `Err(EINTR)` — blocking read interrupted by signal

### Out of Scope

- /dev/kmsg (cross-ref future `kernel/printk/00-overview.md`)
- syslog(2) syscall (cross-ref `kernel/printk/00-overview.md`)
- 32-bit-only paths
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| /proc/kmsg procfs wrapper | `fs/proc/kmsg.c` |
| Kernel printk + ring buffer + syslog action handlers | `kernel/printk/printk.c` |
| Per-CPU printk-safe path | `kernel/printk/printk_safe.c` |
| Public API + LOG_LEVEL constants | `include/linux/printk.h` |

### compatibility contract

### `/proc/kmsg` open semantics

```c
int fd = open("/proc/kmsg", O_RDONLY);
```

Open requires CAP_SYS_ADMIN (or CAP_SYSLOG; per `dmesg_restrict` sysctl). Per-process exclusive: only one task at a time can hold an open `/proc/kmsg` fd (due to the destructive-read semantic — multiple consumers would race).

### Read semantics

`read(fd, buf, count)`:
- If kernel ring buffer has unread messages → return next chunk of message bytes
- If buffer is empty → block until next printk (or return EAGAIN with O_NONBLOCK)
- After read, the consumed bytes are removed from the kernel ring buffer (DESTRUCTIVE — important difference from /dev/kmsg)

Each message line is human-readable: `<level>kernel-message-text\n`. Format byte-identical to upstream's `do_syslog`.

### Permission gate

Per `dmesg_restrict` sysctl:
- `dmesg_restrict=0` (default-INSECURE; not used in modern systems): any user can read
- `dmesg_restrict=1` (default-SECURE on most distros): CAP_SYSLOG required (or CAP_SYS_ADMIN historically)

Plus `CONFIG_PRINTK_INDEX` and `kptr_restrict` interactions per upstream's printk security model.

### `poll()` support

`poll(fd, ...)` returns POLLIN when ring buffer has unread bytes. Allows non-blocking consumers to wait for new log messages without continuous polling.

### Kernel ring buffer (cross-ref `kernel/printk/`)

`/proc/kmsg` consumes from the same ring buffer as `dmesg` and `/dev/kmsg`. Ring buffer is backed by `struct printk_ringbuffer` (lockless ring, per upstream's lockless-printk overhaul); per-message has metadata (timestamp, log level, facility, pid for `/dev/kmsg` non-destructive variant) plus text payload.

Identical ring buffer semantics.

### `do_syslog(SYSLOG_ACTION_*, ...)` interface

`/proc/kmsg` reads bridge to `do_syslog(SYSLOG_ACTION_READ, ...)`:
- `SYSLOG_ACTION_READ` — destructive read
- `SYSLOG_ACTION_READ_ALL` — non-destructive (used by /dev/kmsg)
- `SYSLOG_ACTION_READ_CLEAR` — destructive read + clear
- `SYSLOG_ACTION_CLEAR` — clear without read
- `SYSLOG_ACTION_CONSOLE_OFF` / `_ON` / `_LEVEL` — console log-level control
- `SYSLOG_ACTION_SIZE_UNREAD` / `_BUFFER` — query buffer state

`/proc/kmsg` only uses ACTION_READ. /dev/kmsg uses ACTION_READ_ALL.

### Open exclusivity

Per-system: at most one task holds `/proc/kmsg` open at a time. Second `open()` returns -EBUSY.

### Userspace consumers

- `klogd -k /proc/kmsg` — legacy
- `syslog-ng` configured for /proc/kmsg
- Custom debug tooling reading printk output for telemetry

Modern systemd disables klogd in favor of /dev/kmsg + journald.

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Open exclusivity (compare-and-set on syslog_lock; race-free) | `kani::proofs::fs::proc::kmsg::open_safety` |
| Bridge to `do_syslog` (per-call buffer-size bounds-check) | `kani::proofs::fs::proc::kmsg::read_safety` |
| poll() integration (no use-after-free of waitqueue) | `kani::proofs::fs::proc::kmsg::poll_safety` |

### Layer 2: TLA+ models

(none new owned at this granularity; relies on printk-ringbuffer model from future `kernel/printk/00-overview.md`)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-system /proc/kmsg open count | ≤ 1 at any time (exclusivity) | `kani::proofs::fs::proc::kmsg::open_invariants` |
| Read consume offset | post-read offset ≥ pre-read offset (monotonic, never re-read) | `kani::proofs::fs::proc::kmsg::offset_invariants` |

### Layer 4: Functional correctness (opt-in)

(deferred — same gate as `fs/proc/00-overview.md`)

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **CONSTIFY** | `proc_kmsg_operations` `static const` | § Mandatory |
| **MEMORY_SANITIZE** | freed-after-read buffer cleared (carries kernel-internal log strings) | § Default-on configurable off |
| **DMESG_RESTRICT** | default `dmesg_restrict=1` (require CAP_SYSLOG to read) | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE, CONSTIFY**: see above
- **USERCOPY**: read uses `copy_to_user` with explicit byte-count
- **SIZE_OVERFLOW**: per-byte-count arithmetic uses checked operators
- **KERNEXEC**: dispatch via `static const fn-ptr`

### Row-2 / GR-RBAC integration

- LSM hook `security_syslog(SYSLOG_ACTION_READ)` for permission check.
- Default useful GR-RBAC policy: deny `/proc/kmsg` open outside gradm-marked `log_admin` role; printk-output may carry sensitive kernel-state strings (addresses with non-`%pK` formats — though that should be rare per kptr_restrict).

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

