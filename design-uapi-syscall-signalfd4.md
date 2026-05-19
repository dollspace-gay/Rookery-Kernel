---
title: "Tier-5 syscall: signalfd4(2) — syscall 289"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`signalfd4(2)` creates a file descriptor that can be `read(2)` to
receive signals as data. It transforms signal delivery from
asynchronous handler-based to synchronous fd-readable, allowing
signals to be integrated into `epoll(7)` / `select(2)` / `poll(2)`
event loops without losing them and without the async-signal-safety
constraints of handler code.

Caller passes a `sigset_t` mask defining WHICH signals this fd
consumes. Those signals MUST be BLOCKED in the caller's process
mask, otherwise they get handler-delivered / default-actioned
before signalfd can consume them. `read(fd)` dequeues one or more
`signalfd_siginfo` (fixed 128 bytes) records from the calling
thread's pending queues intersected with the fd's mask.

`signalfd4` (vs. legacy `signalfd`) accepts `flags`: `SFD_CLOEXEC`
and `SFD_NONBLOCK`. Existing signalfd can be re-armed by passing
`fd >= 0` — mask is REPLACED, not merged. `SIGKILL`/`SIGSTOP`
cannot be consumed (silently stripped from mask).

Critical for: every modern event loop integrating signals — sd-bus,
libevent, libuv, Go runtime, Rust tokio, Node.js. CLOEXEC default
is critical for container security (fd must not leak across
`execve`).

This Tier-5 covers `fs/signalfd.c::SYSCALL_DEFINE4(signalfd4, ...)` plus `do_signalfd4` and the read-path `signalfd_dequeue`.

### Acceptance Criteria

- [ ] AC-1: Syscall 289 on x86_64; 74 on generic.
- [ ] AC-2: Create signalfd for {SIGUSR1}; block SIGUSR1; send SIGUSR1; read 128 bytes; `ssi_signo == SIGUSR1`.
- [ ] AC-3: SFD_NONBLOCK and no pending: `-EAGAIN`.
- [ ] AC-4: SFD_CLOEXEC; fork+exec: child does not inherit fd.
- [ ] AC-5: Without SFD_CLOEXEC; fork+exec: child DOES inherit fd.
- [ ] AC-6: read with count = 127: `-EINVAL`.
- [ ] AC-7: sizemask = 4: `-EINVAL`.
- [ ] AC-8: flags unknown bit 0x1: `-EINVAL`.
- [ ] AC-9: mask includes SIGKILL+SIGSTOP: silently stripped.
- [ ] AC-10: Update existing signalfd (fd >= 0): mask replaced; previous mask discarded.
- [ ] AC-11: fd >= 0 not signalfd: `-EINVAL`.
- [ ] AC-12: Multiple readers on same blocked signal: first to consume wins.
- [ ] AC-13: Realtime signal queued 5 times: read returns 5 records (640 total).
- [ ] AC-14: ssi_pid/ssi_uid reflect kernel-authoritative sender; cannot be spoofed via `rt_sigqueueinfo`.
- [ ] AC-15: poll returns POLLIN when masked signal pending; nothing when not.
- [ ] AC-16: close fd: ctx freed; pending signals remain in process queue.
- [ ] AC-17: ssi_code reflects source (SI_USER / SI_QUEUE / SI_TIMER / SI_TKILL).

### Architecture

```rust
#[syscall(nr = 289, abi = "sysv")]
pub fn sys_signalfd4(
    fd: c_int, mask: UserPtr<Sigset>, sizemask: usize, flags: c_int,
) -> isize {
    if sizemask != size_of::<Sigset>() { return -EINVAL; }
    if flags & !(SFD_CLOEXEC | SFD_NONBLOCK) != 0 { return -EINVAL; }
    let mut sigmask = Sigset::from_user(mask)?;
    sigmask.del(SIGKILL); sigmask.del(SIGSTOP);
    Signalfd::do_signalfd4(fd, sigmask, flags)
}
```

`do_signalfd4`: if `fd == -1` allocate `SignalfdCtx { sigmask, lock, wqh }`, install via `AnonInode::get_file("[signalfd]", &SIGNALFD_FOPS, ...)`, return new fd. Else `Fd::get(fd)`, check `f_op == &SIGNALFD_FOPS` (`-EINVAL`), replace `ctx.sigmask` under `ctx.lock`, `wake_all` on `ctx.wqh`, return `fd`.

`SIGNALFD_FOPS.read`: validate `count >= 128`; loop: under siglock dequeue up to `count/128` records (signals intersect with `ctx.sigmask`), transcode to `SignalfdSiginfo` (zero-initialized for padding), `copy_to_user` each 128-byte record. If at least one read return bytes; else if `O_NONBLOCK` return `-EAGAIN`, else `wait_interruptible(&ctx.wqh)`.

### Out of Scope

- Legacy `signalfd(2)` (3-arg form; expands to `signalfd4` with `flags = 0`).
- `rt_sigaction(2)` / `rt_sigprocmask(2)` (separate Tier-5 docs).
- `epoll_ctl(2)` / `epoll_wait(2)` (separate Tier-5 docs).
- Signal-generation paths (`kill(2)`, `rt_sigqueueinfo(2)`, POSIX timers) — separate Tier-5 docs.
- anon_inode internals (`fs/anon_inodes.c`, Tier-3).
- Compat 32-bit `signalfd4` (Tier-3).
- Implementation code.

### signature

```c
int signalfd4(int fd, const sigset_t *mask, size_t sizemask, int flags);
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `fd` | `int` | in | `-1` = create new fd. Otherwise existing signalfd whose mask is REPLACED. |
| `mask` | `const sigset_t *` | in | Signals to consume. SIGKILL/SIGSTOP silently stripped. |
| `sizemask` | `size_t` | in | Must equal `sizeof(sigset_t)` (8 on LP64). |
| `flags` | `int` | in | `SFD_CLOEXEC | SFD_NONBLOCK`. |

### return value

| Value | Meaning |
|---|---|
| `>= 0` | New (or updated) file descriptor. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EBADF`  | `fd >= 0` but not a valid open fd. |
| `EINVAL` | `fd >= 0` but not a signalfd; `sizemask != sizeof(sigset_t)`; unknown flag bit. |
| `EMFILE` | Per-process fd limit reached. |
| `ENFILE` | System-wide fd limit reached. |
| `ENODEV` | anon_inodefs unavailable. |
| `ENOMEM` | Allocation failure for `signalfd_ctx`. |
| `EFAULT` | `mask` not accessible. |

### abi surface

```text
__NR_signalfd4 (x86_64)    = 289
__NR_signalfd4 (i386)      = 327
__NR_signalfd4 (generic)   = 74    /* arm64/riscv/loongarch */
__NR_signalfd4 (powerpc)   = 313
__NR_signalfd4 (s390x)     = 322
__NR_signalfd4 (sparc)     = 317
```

### Flags

```text
SFD_CLOEXEC    O_CLOEXEC   (0x80000)
SFD_NONBLOCK   O_NONBLOCK  (0x00800)
```

### `signalfd_siginfo` (128 bytes, stable layout)

```c
struct signalfd_siginfo {
    __u32 ssi_signo; __s32 ssi_errno; __s32 ssi_code;
    __u32 ssi_pid;   __u32 ssi_uid;   __s32 ssi_fd;
    __u32 ssi_tid;   __u32 ssi_band;  __u32 ssi_overrun;
    __u32 ssi_trapno; __s32 ssi_status; __s32 ssi_int;
    __u64 ssi_ptr;   __u64 ssi_utime; __u64 ssi_stime;
    __u64 ssi_addr;  __u16 ssi_addr_lsb;
    __u8  __pad[46];          /* padding to 128 bytes */
};
```

Fixed 128-byte ABI; padding MUST be zero on every write.

### compatibility contract

REQ-1: Syscall **289** on x86_64; **74** on generic. ABI-stable.

REQ-2: `sizemask` MUST equal `sizeof(sigset_t)` (8 on LP64). Mismatch → `-EINVAL`.

REQ-3: `flags`: only `SFD_CLOEXEC | SFD_NONBLOCK` allowed. Else `-EINVAL`.

REQ-4: `mask` copied via UDEREF-protected `copy_from_user`; SIGKILL/SIGSTOP silently cleared.

REQ-5: If `fd == -1` (create):
- Allocate `signalfd_ctx { sigmask: mask }`.
- `anon_inode_getfd("[signalfd]", &signalfd_fops, ctx, O_RDWR | flags)`.
- Return new fd.

REQ-6: If `fd >= 0` (update):
- `fdget(fd)` and validate `file->f_op == &signalfd_fops`; else `-EINVAL`.
- Replace `ctx.sigmask` under `ctx.lock`.
- Wake blocked readers.
- Return the same `fd`.

REQ-7: `read(fd, buf, count)`:
- `count >= sizeof(signalfd_siginfo)` (128) else `-EINVAL`.
- Reads up to `count / 128` records.
- Source: `current.signal.shared_pending ∪ current.pending` intersected with `ctx.sigmask`.
- Populate `signalfd_siginfo` with kernel-recorded metadata (no spoofing — kernel set fields at generation time).
- No pending, non-blocking → `-EAGAIN`; blocking → sleep on `ctx.wqh`.
- Returns total bytes (multiple of 128).

REQ-8: `poll/epoll`: `POLLIN | POLLRDNORM` when any signal in `ctx.sigmask` is pending for the reader; `0` otherwise.

REQ-9: `close(fd)` frees `signalfd_ctx`; pending signals remain in process queue.

REQ-10: Opening signalfd is non-destructive; consumption is destructive at read time.

REQ-11: Multiple signalfds with overlapping masks: first read wins; kernel does NOT duplicate delivery.

REQ-12: `signalfd_siginfo.ssi_*` reflect kernel-recorded sender identity (same as `rt_sigtimedwait`); cannot be spoofed.

REQ-13: `fork()`: child inherits fd (unless `SFD_CLOEXEC`+next `execve`); each process reads from its own pending queue.

REQ-14: `execve()` with `SFD_CLOEXEC`: fd closed. Otherwise fd survives across exec (security-sensitive — see Hardening).

REQ-15: `dup`/`dup2`/`F_DUPFD` standard; signalfd is position-less (no `lseek`).

REQ-16: Audit: `AUDIT_SYSCALL` captures mask and resulting fd.

REQ-17: Realtime signals: each enqueued instance produces ONE record on read; FIFO preserved.

REQ-18: Legacy `signalfd(2)` is the same path with `flags == 0`.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `sizemask_strict` | INVARIANT | sizemask != 8 ⟹ -EINVAL. |
| `flags_strict` | INVARIANT | (flags & !(SFD_CLOEXEC|SFD_NONBLOCK)) != 0 ⟹ -EINVAL. |
| `kill_stop_sanitized` | INVARIANT | sigmask excludes SIGKILL/SIGSTOP. |
| `count_minimum` | INVARIANT | read with count < 128 ⟹ -EINVAL. |
| `fixed_record_size` | INVARIANT | each read produces exactly 128-byte records; padding zeroed. |
| `mask_replace_atomic` | INVARIANT | update under ctx.lock; concurrent readers see consistent mask. |
| `cloexec_honored` | INVARIANT | SFD_CLOEXEC ⟹ fd closed by exec. |

### Layer 2: TLA+

`fs/signalfd4.tla`:
- States: VALIDATE → (CREATE | UPDATE) → READ_LOOP → DEQUEUE → COPY_OUT → (LOOP | SLEEP).
- Concurrent: producer enqueuing while reader reads; multiple readers; fork inheritance; close-during-read.
- Properties:
  - `safety_no_double_consumption` — a single pending signal consumed by exactly one read across all signalfds + sigwaits.
  - `safety_fixed_record_size` — each record copy-out exactly 128 bytes.
  - `safety_mask_atomic_update` — concurrent readers observe old or new mask, never torn.
  - `liveness_reader_progress` — reader woken when matching signal becomes pending.
  - `safety_cloexec_inheritance` — SFD_CLOEXEC: fd closed across execve.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_signalfd4` post: >= 0 ⟹ fd is signalfd with given mask | `sys_signalfd4` |
| `do_signalfd4 create` post: ctx.sigmask = sanitized input | `do_signalfd4` |
| `signalfd_read` post: result is N*128 bytes; each record padding-zero | `SIGNALFD_FOPS.read` |
| `SignalfdSiginfo::from_kernel_siginfo` post: ssi_* reflect kernel fields | transcode |

### Layer 4: Verus / Creusot functional

POSIX-extension `signalfd(2)` semantic equivalence. LTP `signalfd01..04`, `signalfd4_01..02`. systemd-journal signalfd integration tests.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

- **Per-sizemask strict** — defense against per-mask-truncation.
- **Per-flags strict** — defense against per-future-bit leakage.
- **Per-SIGKILL/SIGSTOP sanitize** — defense against per-trick-into-consuming uncatchable signals.
- **Per-fixed-128-byte records** — defense against per-padding-leak; every record zeroed first.
- **Per-fd-type check on update path** — defense against per-fd-type-confusion.
- **Per-CLOEXEC default-recommended** — defense against per-fd-leak across exec.
- **Per-ctx.lock for mask replace** — defense against per-torn-mask race between reader and updater.

### grsecurity / pax-style reinforcement

- **PAX_RANDKSTACK at syscall entry** — randomizes kernel-stack layout per call; combined with per-call zero-init of `SignalfdSiginfo`, defeats per-stack-leak via 46-byte padding.
- **PaX UDEREF on `sigset_t`** — `copy_from_user(mask)` runs with SMAP/PAN + UDEREF; pointer mistakes cannot dereference user space.
- **PaX MEMORY_SANITIZE** — `SignalfdSiginfo::default()` zeroes all 128 bytes including 46-byte `__pad`. Even when only `ssi_signo/ssi_pid/ssi_uid` are meaningful, remaining bytes are zero — no per-stack/heap leak.
- **GRKERNSEC_SIGNALS authentic ssi_pid/ssi_uid** — kernel set sender identity at generation time per the allow-list at `rt_sigqueueinfo`, `kill`, etc.; signalfd surfaces those authentic values. Caller cannot spoof `ssi_pid`/`ssi_uid`.
- **GRKERNSEC_BRUTE on signalfd churn** — repeated create/update/close cycles by single uid observed by brute-force counter.
- **PaX KERNEXEC** — `do_signalfd4`, `signalfd_read`, transcode `from_kernel_siginfo` in read-only-after-init kernel text; cannot be patched to skip size check, padding-zero, or mask sanitize.
- **GRKERNSEC_PROC restrictions** — `/proc/<pid>/fdinfo/<fd>` reveals the signalfd mask; per-uid hide-filter prevents unrelated uids from observing which signals a process consumes.
- **GRKERNSEC_CHROOT** — chroot'd processes may be restricted from creating signalfds that consume signals from host pid_ns; combined with `ssi_pid` pid_ns translation, contains cross-jail signal observability.
- **GRKERNSEC_SIGNALS CLOEXEC mandatory for protected subjects** — RBAC may MANDATE `SFD_CLOEXEC` for SUID/protected subjects; attempts to create non-CLOEXEC signalfd by such subjects rejected with audit. Prevents fd leak to `execve`'d child that could read signals destined for the original process.
- **PaX TASK_HARDENING + RANDSTRUCT** — `signalfd_ctx` slab in hardened region with redzone; OOB writes blocked. Field order randomized; OOB-write primitives cannot deterministically clobber `ctx.sigmask` to escalate signal consumption privileges.

