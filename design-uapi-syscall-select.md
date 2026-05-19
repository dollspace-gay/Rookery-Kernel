---
title: "Tier-5 syscall: select(2) — syscall 23"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`select(2)` waits for one of three sets of file descriptors to become ready for reading, writing, or exception-signalling, with a `struct timeval` (microsecond-granularity) timeout. It is the oldest of the kernel multiplexers (4.2BSD, 1983), surviving in the Linux UAPI for portability with code that pre-dates `poll(2)` and `epoll(7)`.

Unlike `pselect6(2)`, `select(2)` has no atomic signal-mask install (subject to the classic self-pipe race) and uses microsecond `timeval` instead of nanosecond `timespec`. Critical for: portable libraries (libcurl pre-multi, openssl s_client demo), legacy BSD ports, embedded utilities, and any code that uses the FD_SET / FD_CLR / FD_ISSET macros.

Linux historically updates the user `timeout` argument with the unconsumed time — a behaviour different from POSIX and from every other Unix; modern code must treat `timeout` as written-back-or-clobbered and not rely on its post-call value.

### Acceptance Criteria

- [ ] AC-1: select with all NULL bitmaps and timeout {0,500000} sleeps 500ms, returns 0.
- [ ] AC-2: select with nfds=10 and bit-2 set in readfds and pipe ready: returns 1, FD_ISSET(2, readfds) true.
- [ ] AC-3: nfds = 1024 returns -EINVAL.
- [ ] AC-4: nfds > RLIMIT_NOFILE.rlim_cur returns -EINVAL.
- [ ] AC-5: timeout.tv_usec = 1_000_000 returns -EINVAL.
- [ ] AC-6: timeout.tv_sec = -1 returns -EINVAL.
- [ ] AC-7: select interrupted by SIGCHLD: returns -1, EINTR; SA_RESTART ignored.
- [ ] AC-8: timeout updated post-call with remaining time (Linux quirk).
- [ ] AC-9: Same fd ready for read+write counted twice in return value.
- [ ] AC-10: NULL timeout select interrupted by SIGINT: EINTR returned.

### Architecture

```rust
#[syscall(nr = 23, abi = "sysv")]
pub fn sys_select(
    nfds: i32,
    rfds: UserPtr<FdSet>, wfds: UserPtr<FdSet>, efds: UserPtr<FdSet>,
    tmo: UserPtr<TimeVal>,
) -> isize {
    Select::kern_select(nfds, rfds, wfds, efds, tmo)
}
```

`Select::kern_select(nfds, rfds, wfds, efds, tmo) -> isize`:
1. if nfds < 0 || nfds > FD_SETSIZE || nfds > current.rlim_nofile() { return Err(EINVAL); }
2. let (deadline, mut tv_in) = if tmo.is_null() {
3.     (KTime::MAX, None)
4. } else {
5.     let tv = unsafe { tmo.copy_in::<TimeVal>()? };
6.     TimeVal::validate(&tv)?;
7.     (KTime::monotonic_plus(tv.to_ktime()), Some(tv))
8. };
9. let start = KTime::monotonic();
10. let count = Pselect::core_sys_select(nfds, rfds, wfds, efds, deadline);
11. /* Linux quirk: write back unconsumed time. */
12. if let Some(_) = tv_in {
13.     let elapsed = KTime::monotonic() - start;
14.     let remain = if elapsed >= deadline_to_ktime(deadline) { TimeVal::zero() }
15.                  else { TimeVal::from_ktime(deadline_to_ktime(deadline) - elapsed) };
16.     unsafe { tmo.copy_out(&remain)?; }
17. }
18. count

### Out of Scope

- pselect6(2) sigmask + nanosec (covered in Tier-5 `pselect6.md`).
- poll(2) (covered in Tier-5 `poll.md`).
- epoll(7) (covered in Tier-5 epoll docs).
- Per-driver f_op->poll implementations.
- Implementation code.

### signature

```c
int select(int nfds,
           fd_set *readfds, fd_set *writefds, fd_set *exceptfds,
           struct timeval *timeout);

struct timeval {
    time_t      tv_sec;    /* seconds */
    suseconds_t tv_usec;   /* microseconds (0..999_999) */
};
```

```c
#define FD_SETSIZE  1024
typedef struct {
    unsigned long fds_bits[FD_SETSIZE / (8 * sizeof(unsigned long))];
} fd_set;
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `nfds` | `int` | in | One past highest fd of interest; bounded by `FD_SETSIZE` and `RLIMIT_NOFILE`. |
| `readfds` | `fd_set *` | in/out | Read-ready bitmap; `NULL` to ignore. |
| `writefds` | `fd_set *` | in/out | Write-ready bitmap; `NULL` to ignore. |
| `exceptfds` | `fd_set *` | in/out | Exception bitmap (OOB on sockets, errors); `NULL` to ignore. |
| `timeout` | `struct timeval *` | in/out | Relative timeout; `NULL` = infinite; on return updated with unconsumed time (Linux-specific). |

### return value

| Value | Meaning |
|---|---|
| `>0` | Sum of bits set across all three output bitmaps (POSIX). |
| `0` | Timeout expired. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EFAULT` | Any fd_set / timeout pointer faults. |
| `EINTR` | Signal delivered before any fd ready. |
| `EINVAL` | `nfds < 0`, `nfds > FD_SETSIZE`, `nfds > RLIMIT_NOFILE`, `timeout->tv_usec` out of range, `timeout->tv_sec` negative. |
| `ENOMEM` | Cannot allocate poll table for large nfds. |

### abi surface

```text
__NR_select   (x86_64)  = 23
__NR_select   (arm64)   = absent — arm64 exposes only pselect6(2).
__NR_select   (riscv)   = absent
__NR_select   (i386)    = 142     (old-select also at 82; new-select is 142)

/* Linux quirk: timeout is OUT-decremented with remaining time.   */
/* POSIX says timeout is read-only; portable code must restore it. */
```

### compatibility contract

REQ-1: Syscall number is **23** on x86_64. ABI-stable since v1.

REQ-2: `nfds` clamped to `min(FD_SETSIZE, RLIMIT_NOFILE)`; values out of range → `-EINVAL`.

REQ-3: For each non-NULL bitmap pointer, copy `(nfds + 7) / 8` bytes from user; output overwritten on return.

REQ-4: `timeout` semantics:
- `NULL` → infinite wait.
- `{0, 0}` → non-blocking probe (single pass).
- `{S, U}` with `U ∈ [0, 1_000_000)` → wait up to S*1e6 + U microseconds.
- Negative `tv_sec` or `tv_usec ∉ [0, 1e6)` → `-EINVAL`.

REQ-5: **Linux-specific** post-call timeout update: on any non-EFAULT exit, `*timeout` is written back with the unconsumed portion of the original timeout. This is for compatibility with old loops that pass the same timeout into successive select() calls. POSIX leaves `*timeout` unspecified.

REQ-6: No signal mask install — interrupts propagate immediately. This is the central race vs `pselect6(2)`: a signal handler that arms a self-pipe between the mask-check and the select-call has its signal silently dropped on the wakeup if the kernel raced through.

REQ-7: Return value is the **sum** of bits set across all three output bitmaps; an fd ready for both read AND write counts twice.

REQ-8: For each set fd in any input bitmap, the kernel calls `f_op->poll(file, table)`; the table aggregates wait-queue entries across all polled fds.

REQ-9: `do_select` loops: scan bitmaps, accumulate ready fds, sleep till deadline-or-wake, rescan. The inner sleep uses `schedule_hrtimeout_range()` with a busy-loop pessimization for very-short timeouts.

REQ-10: select(2) is **not restarted** by SA_RESTART; ERESTART* converted to EINTR in syscall exit (BSD compat).

REQ-11: Compat ABI: 32-bit callers use `__NR_select` with `struct old_timeval32`. The pre-2.6.10 i386 wrapper `__NR_oldselect` (= 82) takes all 5 args in a single argument-block struct.

REQ-12: Per-FD_SETSIZE = 1024 is a hard ABI cap; fd indices ≥ 1024 cannot be polled by select(2). Use poll(2) or epoll(7) instead.

REQ-13: Per-namespace: no userns-specific behavior.

REQ-14: select(2) does not consume entropy, does not affect signal state, and does not require any capability.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `nfds_clamped` | INVARIANT | nfds ∈ [0, min(FD_SETSIZE, RLIMIT_NOFILE)]. |
| `tv_validated` | INVARIANT | tv_usec ∈ [0, 1e6), tv_sec >= 0. |
| `bytes_copy_bounded` | INVARIANT | copy size = (nfds+7)/8 per bitmap. |
| `timeout_writeback` | INVARIANT | on non-EFAULT exit, *tmo updated with remain. |
| `no_signal_mask_change` | INVARIANT | current.blocked unchanged across call. |

### Layer 2: TLA+

`fs/select-syscall.tla`:
- States: per-validate, per-copy-in, per-poll-pass, per-wait, per-wake, per-copy-out, per-tmo-writeback.
- Properties:
  - `safety_bitmap_copy_bounded` — copy_in / copy_out exactly (nfds+7)/8 bytes.
  - `safety_eintr_signal_only` — EINTR ⟹ signal pending.
  - `safety_count_sum` — return == sum-of-popcount across three bitmaps.
  - `liveness_deadline_exit` — deadline expires ⟹ return 0 + tmo zeroed.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `kern_select` post: ret >= 0 ⟹ count matches popcount | `Select::kern_select` |
| `TimeVal::validate` post: tv_usec in range | `TimeVal::validate` |
| `core_sys_select` post: out bitmaps zero for cleared bits | `Pselect::core_sys_select` |

### Layer 4: Verus / Creusot functional

Per-select(2) BSD-1983 + POSIX-2017 semantic equivalence; LTP `select01..select04` and glibc `tst-select` pass.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`select(2)` reinforcement:

- **Per-FD_SETSIZE hard cap** — defense against per-bitmap-overflow.
- **Per-nfds RLIMIT_NOFILE strict** — defense against per-fd-amplification.
- **Per-tv_usec range validated** — defense against per-overflow in ktime arithmetic.
- **Per-EINTR no SA_RESTART** — defense against per-signal-loss across rearm (POSIX-correct).
- **Per-relative timeout (POSIX)** — defense against per-CLOCK_REALTIME skew.
- **Per-timeout-writeback bounded** — defense against per-out-pointer write to closed memory.
- **No-sigmask-change** — orthogonal to caller's signal state; callers needing race-free signal coalescing must use pselect6.

### grsecurity / pax surface

- **PaX UDEREF on rfds/wfds/efds/timeout copy_from_user / copy_to_user** — four independently validated user pointers; SMAP forced; bounded per-pointer.
- **GRKERNSEC_RLIMIT_NOFILE** — strict cap on nfds; defense against per-fd-table-blow DoS.
- **PAX_USERCOPY_HARDEN** — bitmap copy uses whitelisted slab; bounded by `(nfds+7)/8`.
- **Self-pipe race documented (NOT fixed)** — select(2) leaves the classic signal race; grsec recommends pselect6(2) and warns in dmesg-ratelimited if a non-CAP_SYS_ADMIN binary calls select(2) inside a sigprocmask critical section.
- **GRKERNSEC_TIME monotonic** — defense against per-CLOCK_REALTIME jump; deadline arithmetic uses CLOCK_MONOTONIC.
- **Linux-quirk timeout writeback explicit** — grsec audits the writeback path (it is a SMAP-protected copy_to_user; UDEREF guards apply).
- **GRKERNSEC_HIDESYM on do_select vtable** — symbol not in kallsyms.
- **Per-FD_SETSIZE rejection logged** — over-FD_SETSIZE attempts ratelimited to dmesg as potential exploit-probe.
- **No-cap-required honored** — select(2) callable by any uid given resource budget; grsec does not gate by capability.
- **PAX_REFCOUNT on per-fd fget_light during do_select** — defense against per-fput-race UAF.

