---
title: "Tier-5 syscall: rt_sigtimedwait(2) — syscall 128"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`rt_sigtimedwait(2)` SYNCHRONOUSLY waits for one of a specified set of
signals to become pending for the calling thread, with an optional
timeout. Unlike asynchronous signal delivery (handler-based), the
signal is **consumed** from the pending queue and its `siginfo_t` is
returned directly to the caller — no handler is invoked, no
sigaltstack is entered, no `SA_RESTART` restart-loop applies.

Semantics:

- Caller passes `set` of signals to wait for. These signals should
  typically be BLOCKED in the caller's mask (otherwise they may be
  delivered asynchronously before this syscall checks them).
- If any signal in `set` is already pending (per-thread or shared),
  the kernel dequeues one, fills `*info` (if non-NULL), and returns
  the signal number IMMEDIATELY.
- If none is pending and `timeout != NULL && *timeout == 0`: returns
  `-EAGAIN` immediately (poll mode).
- If none is pending and `timeout == NULL`: blocks indefinitely.
- If none is pending and `timeout != NULL && *timeout > 0`: blocks up
  to `*timeout` seconds + nanoseconds; on expiry returns `-EAGAIN`.
- Wakeup on signal NOT in `set` (asynchronous unblocked signal, fatal
  signal, ptrace-stop) returns `-EINTR`.

`SIGKILL (9)` and `SIGSTOP (19)` CANNOT be waited for via this syscall
(they are silently removed from `set`). Realtime signals (32..64)
participate fully and queued multiple instances are dequeued one at a
time per call.

This Tier-5 covers `kernel/signal.c::SYSCALL_DEFINE4(rt_sigtimedwait, ...)`
plus its `do_sigtimedwait` workhorse (~120 lines of entry surface).

### Acceptance Criteria

- [ ] AC-1: Syscall number 128 on x86_64; 137 on generic.
- [ ] AC-2: Block SIGUSR1, queue SIGUSR1, call `rt_sigtimedwait({SIGUSR1}, &info, NULL, 8)`: returns SIGUSR1; info.si_signo = SIGUSR1.
- [ ] AC-3: No pending, `timeout={0,0}`: returns `-EAGAIN`.
- [ ] AC-4: No pending, `timeout={1,0}` then no signal arrives: blocks 1s, returns `-EAGAIN`.
- [ ] AC-5: Wait for SIGUSR1, other thread sends SIGUSR2 (not in set): wakeup with `-EINTR`.
- [ ] AC-6: Wait for SIGUSR1, timeout={5,0}, signal arrives after 1s: returns SIGUSR1.
- [ ] AC-7: `info=NULL`: signal still dequeued.
- [ ] AC-8: `sigsetsize=4`: `-EINVAL`.
- [ ] AC-9: `timeout->tv_nsec = 1_000_000_001`: `-EINVAL`.
- [ ] AC-10: `set` includes SIGKILL+SIGSTOP: silently ignored; if only those, behaves as if set is empty (waits indefinitely / times out).
- [ ] AC-11: Realtime SIGRTMIN queued 3 times: 3 successive calls dequeue 3 times.
- [ ] AC-12: `info.si_code` reflects source (SI_USER from `kill`, SI_QUEUE from `rt_sigqueueinfo`, SI_TKILL from `tkill`).
- [ ] AC-13: Multithreaded: signal in shared queue is claimed by exactly one waiter.
- [ ] AC-14: `task.blocked` restored to original on return.

### Architecture

```rust
#[syscall(nr = 128, abi = "sysv")]
pub fn sys_rt_sigtimedwait(
    set:    UserPtr<Sigset>,
    info:   UserPtr<UapiSiginfo>,
    timeout: UserPtr<Timespec>,
    sigsetsize: usize,
) -> isize {
    if sigsetsize != size_of::<Sigset>() { return -EINVAL; }

    let mut waitset = Sigset::from_user(set)?;
    waitset.del(SIGKILL); waitset.del(SIGSTOP);

    let ts: Option<Timespec> = if !timeout.is_null() {
        let t = Timespec::from_user(timeout)?;
        if t.tv_sec < 0 || t.tv_nsec < 0 || t.tv_nsec >= 1_000_000_000 {
            return -EINVAL;
        }
        Some(t)
    } else { None };

    let mut sinfo = UapiSiginfo::default();
    let signo = Signal::do_sigtimedwait(&waitset, &mut sinfo, ts.as_ref())?;

    if !info.is_null() { sinfo.to_user(info)?; }
    signo as isize
}
```

`Signal::do_sigtimedwait(set, info_out, timeout)`:
1. let task = Task::current();
2. let saved_blocked = task.blocked.clone();
3. /* REQ-7: while waiting, block everything except `set` */
4. task.real_blocked = !set.clone() | saved_blocked;
5. loop {
6.   let _g = SpinLockGuard::lock_irq(&task.sighand.siglock);
7.   /* REQ-5: try dequeue */
8.   if let Some(s) = Signal::dequeue_pending_intersect(task, set, info_out) {
9.     task.real_blocked = saved_blocked;
10.    return Ok(s);
11.  }
12.  drop(_g);
13.  match timeout {
14.    Some(ts) if ts.is_zero() => { task.real_blocked = saved_blocked; return Err(-EAGAIN); }
15.    Some(ts) => match Sched::schedule_hrtimeout(ts) {
16.      WokenBy::Timer => { task.real_blocked = saved_blocked; return Err(-EAGAIN); }
17.      WokenBy::Signal => { /* loop and re-check */ }
18.    },
19.    None => Sched::schedule_interruptible(),
20.  }
21.  /* REQ-6: detect EINTR vs in-set */
22.  if Signal::has_pending_outside(set) {
23.    task.real_blocked = saved_blocked;
24.    return Err(-EINTR);
25.  }
26. }

### Out of Scope

- `rt_sigaction(2)` / `rt_sigprocmask(2)` / `rt_sigpending(2)` (separate Tier-5 docs).
- `rt_sigsuspend(2)` (separate Tier-5 doc).
- `rt_sigqueueinfo(2)` / `rt_tgsigqueueinfo(2)` (separate Tier-5 docs).
- `signalfd4(2)` (separate Tier-5 doc).
- Signal-delivery internals (`kernel/signal.c::dequeue_signal`, `collect_signal` — Tier-3).
- Compat 32-bit variants (Tier-3).
- hrtimer machinery (Tier-3 `kernel/time/hrtimer.c`).
- Implementation code.

### signature

```c
long rt_sigtimedwait(const sigset_t *set,
                     siginfo_t *info,
                     const struct timespec *timeout,
                     size_t sigsetsize);
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `set` | `const sigset_t *` | in | Set of signals to wait for. MUST be non-NULL. |
| `info` | `siginfo_t *` | out | Pointer to receive `siginfo_t` of the dequeued signal. `NULL` means "do not return info". |
| `timeout` | `const struct timespec *` | in | Maximum wait time. `NULL` = block forever. `{0,0}` = poll. Otherwise relative interval. |
| `sigsetsize` | `size_t` | in | Size of `sigset_t` in bytes — MUST equal `sizeof(sigset_t)` (8 on LP64). |

### return value

| Value | Meaning |
|---|---|
| `> 0` | Signal number that was dequeued. |
| `-1` + `errno` | Failure or interruption / timeout. |

### errors

| errno | Trigger |
|---|---|
| `EAGAIN` | Timeout expired or `*timeout == {0,0}` and no matching pending signal. |
| `EINTR`  | Wakeup by a signal NOT in `set` (asynchronous delivery, fatal). |
| `EINVAL` | `sigsetsize != sizeof(sigset_t)`. |
| `EINVAL` | `timeout != NULL` and `tv_nsec < 0` or `tv_nsec >= 1_000_000_000` or `tv_sec < 0`. |
| `EFAULT` | `set`, `info`, or `timeout` non-NULL and not accessible. |

### abi surface

```text
__NR_rt_sigtimedwait (x86_64)    = 128
__NR_rt_sigtimedwait (i386)      = 177
__NR_rt_sigtimedwait (generic)   = 137   /* arm64/riscv/loongarch */
__NR_rt_sigtimedwait (powerpc)   = 176
__NR_rt_sigtimedwait (s390x)     = 177
__NR_rt_sigtimedwait (sparc)     = 105
__NR_rt_sigtimedwait (mips O32)  = 4197
__NR_rt_sigtimedwait (mips N64)  = 5126
```

### `siginfo_t` (extract — `uapi/asm-generic/siginfo.h`)

```c
typedef struct siginfo {
    int si_signo;
    int si_errno;
    int si_code;
    union sigval { /* ... 128 bytes total ... */ } _sifields;
} siginfo_t;
```

Standard `siginfo_t` is 128 bytes on LP64. The kernel populates
`si_signo`, `si_code`, and the relevant union arm based on the signal
source (`SI_USER`, `SI_KERNEL`, `SI_QUEUE`, `SI_TIMER`, `SI_TKILL`,
`SI_ASYNCIO`, `SI_MESGQ`, `SI_SIGIO`).

### `struct timespec`

```c
struct timespec {
    time_t tv_sec;   /* signed 64-bit on LP64 */
    long   tv_nsec;  /* 0 .. 999_999_999     */
};
```

### compatibility contract

REQ-1: Syscall number is **128** on x86_64; **137** on generic-syscall
arches. ABI-stable forever.

REQ-2: `sigsetsize` MUST equal kernel's native `sizeof(sigset_t)` (8 on
LP64). Mismatch → `-EINVAL`.

REQ-3: `set` is copied from user; `SIGKILL` and `SIGSTOP` bits are
silently cleared (sanitize). The effective wait-set excludes them.

REQ-4: If `timeout` is non-NULL: copy from user, validate
`tv_sec >= 0`, `0 <= tv_nsec < 1_000_000_000`. Otherwise `-EINVAL`.

REQ-5: Fast-path: under `task.sighand.siglock`:
- Check per-thread pending queue intersected with `set`.
- Check shared pending queue intersected with `set`.
- If any: dequeue ONE signal (lowest-numbered signal first; realtime
  signals dequeued in arrival order), populate `siginfo_t`, return
  `si_signo`.

REQ-6: Slow-path (no signal pending):
- If `timeout == NULL`: sleep on `TASK_INTERRUPTIBLE` until awakened.
- If `*timeout == {0,0}`: return `-EAGAIN` immediately.
- Otherwise: arm hrtimer for the timeout; sleep on
  `TASK_INTERRUPTIBLE`; awakened by signal-delivery OR timer.
- On wakeup, retry fast-path. If matched: return signal number. If
  timer expired: `-EAGAIN`. If wakeup by signal not in `set`:
  `-EINTR`.

REQ-7: `current.real_blocked` is temporarily set to
`~set | original.blocked` for the duration of the wait — i.e. while
sleeping, the task is "blocking everything except the signals in
`set`". This ensures the wait wakes ONLY for in-set signals (other
signals queue but do not wake the wait).

REQ-8: On wakeup-and-dequeue, `task.blocked` is restored to its
original pre-call value before returning to userspace.

REQ-9: If `info != NULL`: populate `*info` via `copy_to_user`. The
`siginfo_t` reflects the dequeued signal's metadata:
- `si_signo` = dequeued signal number.
- `si_code` = source code (`SI_USER`, `SI_QUEUE`, `SI_TIMER`,
  `SI_TKILL`, `SI_KERNEL`, etc.).
- `si_pid` / `si_uid` = sender (for SI_USER/SI_QUEUE/SI_TKILL).
- `si_value` = sival_t for SI_QUEUE.
- `si_addr` = faulting address for synchronous traps.

REQ-10: `info == NULL`: signal is still dequeued; no metadata
returned.

REQ-11: Realtime signals: only ONE instance per call is dequeued.
Multiple queued instances remain in queue.

REQ-12: Non-realtime signals: standard "occurrence" bit; second
generation while pending is coalesced (per POSIX). The bit is cleared
on dequeue.

REQ-13: Fatal signal arrival (not in `set`): wakeup with `-EINTR`;
fatal handling proceeds normally on return-to-user.

REQ-14: Audit: `AUDIT_SYSCALL` record on every `rt_sigtimedwait`
captures `set` (effective) and the returned signum (if successful).

REQ-15: Atomicity: dequeue + siginfo copy is done under siglock; the
signal is consumed exactly once even under concurrent
`rt_sigtimedwait` calls from sibling threads sharing the process.

REQ-16: Multithreaded: signals in the shared pending queue are
race-claimed by the first thread to wake and dequeue. Threads
waiting on overlapping sets must use external synchronization if they
require ordered receipt.

REQ-17: Compat (32-bit on 64-bit kernel): handled by
`compat_rt_sigtimedwait_time32` and `_time64` variants for
year-2038-safe `timespec`.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `sigsetsize_strict` | INVARIANT | sigsetsize ≠ 8 ⟹ -EINVAL. |
| `timespec_validated` | INVARIANT | tv_nsec ∈ [0, 1e9) ∧ tv_sec ≥ 0 OR -EINVAL. |
| `kill_stop_excluded_from_set` | INVARIANT | Effective waitset excludes SIGKILL/SIGSTOP. |
| `single_dequeue` | INVARIANT | At most one signal consumed per successful call. |
| `blocked_restored` | INVARIANT | task.blocked restored on every return path. |
| `siglock_held_during_dequeue` | INVARIANT | Dequeue under siglock + IRQs off. |

### Layer 2: TLA+

`kernel/rt_sigtimedwait.tla`:
- States: VALIDATE → ARM_REAL_BLOCKED → CHECK_PENDING → (DEQUEUE | SLEEP → WAKEUP → CHECK_PENDING) → RESTORE_BLOCKED → COPY_OUT.
- Concurrent: send_signal on multiple CPUs; sibling thread also waiting.
- Properties:
  - `safety_exactly_one_consumer` — each generated signal consumed by exactly one waiter.
  - `safety_blocked_restored_on_all_exits` — task.blocked is `saved_blocked` post-call.
  - `liveness_progress` — if a matching signal eventually arrives within timeout, call returns it.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_rt_sigtimedwait` post: returns >0 ⟹ info has matching si_signo | `sys_rt_sigtimedwait` |
| `do_sigtimedwait` post: real_blocked restored on all return paths | `do_sigtimedwait` |
| `Timespec::from_user` byte-layout 16 bytes (LP64) | `Timespec::from_user` |
| `UapiSiginfo::to_user` byte-layout 128 bytes | `UapiSiginfo::to_user` |

### Layer 4: Verus / Creusot functional

POSIX `sigtimedwait(2)` / `sigwait(3)` / `sigwaitinfo(2)` semantic
equivalence. LTP `rt_sigtimedwait01..04`, glibc test suite signal/*.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`rt_sigtimedwait(2)` reinforcement:

- **Per-sigsetsize strict** — defense against per-mask-truncation.
- **Per-timespec range-check** — defense against
  per-overflow/underflow in hrtimer arithmetic.
- **Per-real_blocked save/restore on every path** — defense against
  per-leaked blocked-mask after EINTR.
- **Per-siglock-held dequeue** — defense against per-double-consume.
- **Per-SIGKILL/SIGSTOP sanitize** — defense against per-trick-into-waiting on uncatchable.
- **Per-siginfo zeroed before populate** — defense against
  per-stack-leak via uninitialized padding.
- **Per-EINTR semantics for out-of-set signals** — defense against
  per-deadlock when fatal signal arrives.

### grsecurity / pax-style reinforcement

- **PAX_RANDKSTACK at syscall entry** — randomizes kernel stack on
  each call; combined with the bounded siglock window, defeats
  per-syscall stack-layout reuse during the LONG sleep-and-wakeup
  cycle.
- **PaX UDEREF on `sigset_t` / `siginfo_t` / `timespec`** — all three
  user pointers traversed via UDEREF / SMAP+PAN-toggled
  `get_user`/`put_user`; a kernel pointer error cannot silently
  dereference into user space.
- **PaX MEMORY_SANITIZE** — local `UapiSiginfo` is zero-initialized
  before populate; the union-arms not relevant to the dequeued
  signal-source remain zero, preventing per-stack/heap leak of
  arbitrary kernel bytes into the siginfo copy-out.
- **GRKERNSEC_SIGNALS si_code authenticity** — `info.si_code` reflects
  the kernel-recorded source (SI_USER / SI_QUEUE / SI_TIMER / SI_TKILL
  / SI_KERNEL). The signal-generation paths set this server-side; user
  cannot spoof a fake si_code via `rt_sigqueueinfo` because the
  SI_USER/SI_QUEUE/SI_TKILL allow-list is enforced at signal-GEN
  time. By the time `rt_sigtimedwait` dequeues, the recorded si_code
  is trustworthy.
- **GRKERNSEC_BRUTE on signal-flood** — repeated
  `rt_sigtimedwait({nonempty}, NULL, {0,0}, ...)` polling at high rate
  combined with crash-triggers (probing for race) is observable by
  grsec rate-policy and may trip the per-uid brute-force counter.
- **PaX KERNEXEC** — `do_sigtimedwait` lives in
  read-only-after-init kernel text; cannot be patched to skip
  real_blocked save/restore.
- **GRKERNSEC_HARDEN_PTRACE** — tracer cannot use ptrace to inject a
  signal into a tracee that the tracee is waiting for, across
  credential boundaries; cross-cred ptrace fails BEFORE
  rt_sigqueueinfo executes.
- **GRKERNSEC_PROC restrictions** — `/proc/<pid>/wchan` and `/proc/<pid>/stack`
  reveal that a thread is sleeping in `do_sigtimedwait`; per-uid
  hide-filter conceals this from non-owners.
- **Per-grsec `gradm` learning** — RBAC may restrict the set of
  signals a subject is allowed to sigwait for (e.g. forbid
  sigwait-on-SIGSEGV for non-debugger subjects to prevent crash
  intelligence).
- **PaX TASK_HARDENING** — extreme timeout values (`tv_sec` very
  large) bounded against `KTIME_MAX`; per-syscall timespec arithmetic
  is overflow-checked, defending against per-hrtimer-overflow DoS.

