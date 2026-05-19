---
title: "Tier-5 syscall: tgkill(2) — syscall 234"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`tgkill(2)` sends a signal to a SPECIFIC thread identified by its
`(tgid, tid)` pair. It is the modern replacement for `tkill(2)`,
adding a `tgid` argument that allows the kernel to verify the
target thread's thread-group membership and thereby close the
PID-recycling race window that `tkill` cannot.

The kernel resolves `tid` to a `task_struct` in caller's pid_ns,
THEN verifies `task.tgid == tgid`. Mismatch → `-ESRCH`. This is
the entire reason `tgkill` exists. Permission checks identical to
`kill(2)` / `tkill(2)`.

`siginfo_t` is kernel-authoritative (NOT caller-supplied):
`si_code = SI_TKILL`, `si_pid = current.tgid`, `si_uid = current.cred.uid`.
Signal enqueued on the thread's PRIVATE pending queue (`task.pending`).
`sig == 0` performs the permission probe only.

`SIGKILL`/`SIGSTOP` deliver via `tgkill` but their actions remain
group-wide; only the wake-up target is thread-directed.

Critical for: modern threading libraries (`pthread_kill(3)`),
GC-runtime safepoints (Go, Rust, JVM, V8), debugger thread-stop,
per-thread cancellation. `tgkill` is the SECURE thread-directed
kill — new code MUST use it instead of `tkill`.

This Tier-5 covers `kernel/signal.c::SYSCALL_DEFINE3(tgkill, ...)` plus the shared `do_tkill` workhorse (parameterized by tgid).

### Acceptance Criteria

- [ ] AC-1: Syscall number 234 on x86_64; 131 on generic.
- [ ] AC-2: `tgkill(peer.tgid, peer.tid_2, SIGUSR1)` with tid_1 sibling: ONLY tid_2 dequeues.
- [ ] AC-3: tgid = 0 / -1 / tid = 0 / -1: `-EINVAL`.
- [ ] AC-4: sig = -1 / 65: `-EINVAL`.
- [ ] AC-5: Non-existent tid: `-ESRCH`.
- [ ] AC-6: tid exists but `task.tgid != arg.tgid` (recycle simulation): `-ESRCH`; no signal delivered.
- [ ] AC-7: Cross-uid without CAP_KILL: `-EPERM`. Same uid: success.
- [ ] AC-8: si_code observed == `SI_TKILL`; si_pid == caller.tgid (in peer's pid_ns); si_uid mapped.
- [ ] AC-9: sig = 0, valid (tgid, tid), permission OK: returns 0; no signal queued.
- [ ] AC-10: `tgkill(target_tgid, target_tid, SIGKILL)`: entire thread group terminates.
- [ ] AC-11: `tgkill(target_tgid, target_tid, SIGSTOP)`: entire thread group stops.
- [ ] AC-12: PID-recycling race modeled: tgkill returns `-ESRCH`, NOT delivered to wrong process — security property absent from tkill.
- [ ] AC-13: 1024 SIGRTMIN via tgkill (RLIMIT_SIGPENDING=1024): 1025th → `-EAGAIN`.
- [ ] AC-14: glibc `pthread_kill(target_thread, SIGUSR1)` observable via strace as `tgkill(self_tgid, target_tid, SIGUSR1)`.

### Architecture

```rust
#[syscall(nr = 234, abi = "sysv")]
pub fn sys_tgkill(tgid: c_int, tid: c_int, sig: c_int) -> isize {
    if tgid <= 0 || tid <= 0 { return -EINVAL; }
    if sig < 0 || sig as usize > NSIG { return -EINVAL; }
    Signal::do_tkill(tgid, tid, sig)
}
```

`Signal::do_tkill(tgid_arg, tid, sig)`: RCU read-lock; `find_task_by_pid_ns(tid, current.pid_ns)` (`-ESRCH`). If `tgid_arg != 0` (tgkill path) verify `task.tgid == tgid_arg` (else `-ESRCH`). `check_kill_permission(task, sig)` (`-EPERM`). If `sig == 0` return 0. Build kernel-authoritative `UapiSiginfo` (`si_code = SI_TKILL`; `si_pid = current.tgid` in target's pid_ns; `si_uid` mapped to target's user_ns). `do_send_specific(sig, &info, task, PIDTYPE_PID)` — RLIMIT-checked enqueue onto `task.pending` under siglock. `signal_wake_up_state(task, 0)`.

### Out of Scope

- `tkill(2)` (separate Tier-5 — legacy without recycling guard).
- `kill(2)` (separate Tier-5 — process-directed).
- `rt_sigqueueinfo(2)` / `rt_tgsigqueueinfo(2)` (separate Tier-5 — caller-supplied siginfo).
- `rt_sigaction(2)` / `rt_sigprocmask(2)` / `rt_sigtimedwait(2)` (separate Tier-5 docs).
- Signal-delivery internals (`do_send_sig_info`, `send_signal`, `complete_signal` — Tier-3).
- Compat 32-bit `tgkill` (Tier-3).
- Implementation code.

### signature

```c
long tgkill(int tgid, int tid, int sig);
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `tgid` | `int` | in | Target thread group id (positive). PID-recycling guard. |
| `tid` | `int` | in | Target thread id (positive). Must belong to thread group `tgid`. |
| `sig` | `int` | in | Signal `0..SIGRTMAX (64)`. `0` = permission probe only. |

### return value

| Value | Meaning |
|---|---|
| `0`  | Signal queued (or `sig == 0` and permission OK). |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EINVAL` | `tgid <= 0`, `tid <= 0`, `sig < 0`, or `sig > _NSIG`. |
| `EPERM`  | Caller lacks signal permission. |
| `ESRCH`  | No thread `tid` exists, OR `task.tgid != tgid` (recycling guard). |
| `EAGAIN` | `RLIMIT_SIGPENDING` exceeded. |

### abi surface

```text
__NR_tgkill (x86_64)    = 234
__NR_tgkill (i386)      = 270
__NR_tgkill (generic)   = 131   /* arm64/riscv/loongarch */
__NR_tgkill (powerpc)   = 250
__NR_tgkill (s390x)     = 241
__NR_tgkill (sparc)     = 234
```

### compatibility contract

REQ-1: Syscall **234** on x86_64; **131** on generic. ABI-stable.

REQ-2: `tgid` and `tid` MUST both be `> 0`. Else `-EINVAL`. No broadcast forms.

REQ-3: `sig` validated against `[0, _NSIG]`. Out-of-range → `-EINVAL`.

REQ-4: Target lookup: `find_task_by_pid_ns(tid, current.pid_ns)`. Miss or `EXIT_DEAD` → `-ESRCH`.

REQ-5: **PID-recycling guard**: verify `task.tgid == tgid` argument. Mismatch → `-ESRCH`. Runs under RCU read-lock so the task struct cannot be freed between lookup and guard check.

REQ-6: Permission: identical to `kill(2)` / `tkill(2)`. Real or effective uid equals target's real or saved-set uid, OR `CAP_KILL` in target's user_ns. `SIGCONT` exception within session.

REQ-7: `siginfo_t` is **kernel-authoritative** (NOT caller-supplied):
- `si_signo = sig`.
- `si_errno = 0`.
- `si_code = SI_TKILL`.
- `si_pid = current.tgid` (in target's pid_ns).
- `si_uid = current.cred.uid` (mapped through target's user_ns).

REQ-8: Enqueue on target's **private** pending queue (`task.pending`) via `do_send_specific`. NOT shared tgroup queue. Only the targeted thread will dequeue.

REQ-9: Wakeup: target woken if `TASK_INTERRUPTIBLE`/`TASK_KILLABLE` and `sig` not blocked. Otherwise remains pending until thread unblocks `sig`.

REQ-10: `sig == 0`: validate `(tgid, tid)` exists + tgid matches + permission; no enqueue; no wakeup. Race-free "is this thread of this process alive AND I can signal it?" probe.

REQ-11: `SIGKILL` via tgkill: terminates entire thread group. `SIGSTOP`: stops entire group. Thread-direction only affects wakeup target.

REQ-12: `RLIMIT_SIGPENDING` enforced for realtime signals; over-limit → `-EAGAIN`. Standard signals coalesce via single-bit.

REQ-13: Audit: `AUDIT_SYSCALL` captures sender tgid+uid, target tgid+tid, signum, `SI_TKILL`.

REQ-14: Cross-pid-ns: `tgid`/`tid` in caller's pid_ns. Recycling guard runs in caller's view. Receiver observes `si_pid` in receiver's pid_ns.

REQ-15: Compat: all three args int; no architecture-specific compat conversion needed.

REQ-16: glibc modern `pthread_kill(3)` implements as `tgkill(self_tgid, target_tid, sig)`. Recycling guard ensures the thread is still part of the SAME process at the moment of the signal.

REQ-17: Shared implementation: `do_tkill(tgid, tid, sig)` is parameterized: `tkill` calls with `tgid = 0` (skip guard); `tgkill` calls with `tgid = arg.tgid` (apply guard).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `tgid_tid_positive` | INVARIANT | tgid > 0 ∧ tid > 0 OR -EINVAL. |
| `sig_range_strict` | INVARIANT | sig ∈ [0, NSIG] OR -EINVAL. |
| `recycling_guard` | INVARIANT | task.tgid != tgid_arg ⟹ -ESRCH; no enqueue. |
| `siginfo_kernel_authoritative` | INVARIANT | si_pid/si_uid/si_code set by kernel, not caller. |
| `si_code_eq_SI_TKILL` | INVARIANT | every tgkill produces si_code == SI_TKILL. |
| `enqueue_on_private_queue` | INVARIANT | enqueued on task.pending, not shared. |
| `rlimit_sigpending_enforced` | INVARIANT | queue full ⟹ -EAGAIN. |
| `rcu_protects_task` | INVARIANT | task struct alive between lookup and guard check. |

### Layer 2: TLA+

`kernel/tgkill.tla`:
- States: VALIDATE → RCU_LOCK → LOOKUP_TID → RECYCLING_GUARD → CHECK_PERMISSION → BUILD_SIGINFO → ENQUEUE_PRIVATE → WAKE.
- Concurrent: PID recycling between lookup and tgid-check (atomic via RCU); sibling sending; cross-pid-ns; target exits mid-syscall.
- Properties:
  - `safety_recycling_guard_holds` — tid recycled into different tgid ⟹ -ESRCH; signal NEVER delivered to wrong process.
  - `safety_siginfo_authentic` — si_pid/si_uid/si_code never caller-controlled.
  - `safety_private_queue_only` — enqueued on task.pending exclusively.
  - `safety_rcu_no_uaf` — task struct cannot be freed during guard check.
  - `liveness_target_woken` — eligible target eventually wakes.
  - `equivalence_with_tkill_when_no_recycle` — when no recycling, observable effects identical to tkill.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_tgkill` post: 0 ⟹ one sigqueue entry on task.pending with SI_TKILL ∧ task.tgid == arg.tgid | `sys_tgkill` |
| `do_tkill` post: tgid_arg != 0 ⟹ task.tgid == tgid_arg verified | `do_tkill` |
| `do_send_specific` post: enqueue under siglock; RLIMIT honored | `do_send_specific` |
| `RCU read-side lock`: task struct alive during lookup → check → enqueue | RCU section |

### Layer 4: Verus / Creusot functional

POSIX `pthread_kill(3)` semantic equivalence. LTP `tgkill01..03`, glibc `nptl/tst-pthread-kill*`. Go runtime safepoint trace assertions.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

- **Per-PID-recycling guard via tgid argument** — defense against per-tid-reuse race (the central security improvement over `tkill(2)`).
- **Per-RCU-protected lookup** — defense against per-UAF if target task exits between lookup and guard check.
- **Per-(tgid, tid) > 0 strict** — defense against per-sentinel confusion.
- **Per-sig range strict** — defense against per-out-of-band signal numbers.
- **Per-permission via real/saved uid OR CAP_KILL** — defense against per-cross-uid injection.
- **Per-siginfo kernel-authoritative** — defense against per-spoof of sender identity.
- **Per-private-queue enqueue** — defense against per-sibling stealing the signal.
- **Per-pid_ns translation** — defense against per-namespace pid confusion; recycling guard runs in caller's pid_ns.
- **Per-RLIMIT_SIGPENDING strict** — defense against per-slab-exhaustion via per-thread queue flood.

### grsecurity / pax-style reinforcement

- **PAX_RANDKSTACK at syscall entry** — randomizes kernel-stack layout per call; defeats per-stack-layout reuse during the brief RCU + enqueue window.
- **PaX UDEREF** — `do_tkill` does not dereference user pointers in its main path (siginfo kernel-built); UDEREF policy applies uniformly across signal entries.
- **PaX MEMORY_SANITIZE** — `UapiSiginfo::default()` zeroes all 128 bytes including union arms not relevant to SI_TKILL; receiver via `rt_sigtimedwait` or `signalfd4` never observes per-stack bytes.
- **GRKERNSEC_SIGNALS SI_TKILL authenticity** — `si_code = SI_TKILL` set by kernel here. Combined with `rt_sigqueueinfo`'s user-origin allow-list (REJECTS user-supplied `SI_TKILL`), receiver can rely on `SI_TKILL` indicating a true `tkill`/`tgkill`, not a forgery via the queue-info path.
- **GRKERNSEC_SIGNALS SI_USER strict-clone (cross-reference)** — `kill(2)` produces `SI_USER`; `tgkill` produces `SI_TKILL`. Grsec distinguishes and forbids forgery of either via `rt_sigqueueinfo`.
- **GRKERNSEC_BRUTE on signal-flood** — repeated cross-uid `tgkill` (probing for permission races) trips per-uid brute-force counter.
- **GRKERNSEC bounded-tid for tkill/tgkill lookups** — `find_task_by_pid_ns` rate-limited per-uid against blind tid enumeration. Even with recycling guard, sustained enumeration is observable + rate-limited.
- **PaX KERNEXEC** — `do_tkill` in read-only-after-init kernel text; cannot be patched to skip recycling guard or produce forged si_code.
- **GRKERNSEC_HARDEN_PTRACE** — tracer cannot use `tgkill` to inject across credential boundaries; cross-cred ptrace check fails.
- **GRKERNSEC_PROC restrictions** — `/proc/<pid>/task/<tid>` and `/proc/<tid>/status` per-uid hide-filtered; unprivileged callers cannot enumerate foreign `(tgid, tid)` pairs.
- **Per-grsec gradm — tgkill REQUIRED for thread-directed kill** — RBAC may forbid `tkill(2)` and mandate `tgkill(2)` for thread-directed signaling, making PID-recycling guard mandatory at policy level. SUID and protected subjects should be in this policy by default.
- **PaX TASK_HARDENING + RANDSTRUCT** — sigqueue slab allocation audited under `RLIMIT_SIGPENDING`; over-limit hard `-EAGAIN` with no kmalloc retry escalation. `task_struct.pending.list` head and `task.tgid` in randomized layouts; memory-corruption primitives cannot deterministically forge a recycling-guard pass or redirect the private queue head.

