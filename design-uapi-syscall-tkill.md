---
title: "Tier-5 syscall: tkill(2) — syscall 200"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`tkill(2)` sends a signal to a SPECIFIC thread identified by its
kernel-internal tid (the value returned by `gettid(2)`, NOT a TGID).
It is the older thread-directed signal API, predating `tgkill(2)`.
Its fundamental defect: only `tid` is accepted, with NO PID-recycling
guard — between the caller's discovery of `tid` (e.g. via `/proc`)
and the syscall, the kernel may have recycled the tid into a
different process. `tkill` will then deliver to that wrong process.

`tkill` is therefore **DEPRECATED** in favor of `tgkill(2)`, which
takes both `tgid` and `tid` and verifies them. New code MUST use
`tgkill`. `tkill` remains only for ABI compatibility with legacy
binaries (pre-NPTL glibc, very early threading libraries).

Signal arrives with `si_code = SI_TKILL`, `si_pid = current.tgid`,
`si_uid = current.cred.uid` — kernel-authoritative, set at enqueue.
`sig == 0` performs the permission probe only.

`SIGKILL`/`SIGSTOP` deliverable but their actions remain group-wide
(SIGKILL kills the whole thread group, SIGSTOP stops all threads);
only the wake-up target is thread-directed.

Critical for: legacy pre-NPTL applications; older glibc's `raise(3)`
on very old kernels. Modern code uses `tgkill`.

This Tier-5 covers `kernel/signal.c::SYSCALL_DEFINE2(tkill, ...)`.

### Acceptance Criteria

- [ ] AC-1: Syscall number 200 on x86_64; 130 on generic.
- [ ] AC-2: `tkill(peer.tid_2, SIGUSR1)` with peer.tid_1 also a sibling: ONLY tid_2 dequeues.
- [ ] AC-3: tid = 0 / -1: `-EINVAL`.
- [ ] AC-4: sig = -1 / 65: `-EINVAL`.
- [ ] AC-5: Non-existent tid: `-ESRCH`.
- [ ] AC-6: Cross-uid without CAP_KILL: `-EPERM`. Same uid: success.
- [ ] AC-7: si_code observed by peer == `SI_TKILL`.
- [ ] AC-8: si_pid observed == caller.tgid (in peer's pid_ns); si_uid mapped to peer's user_ns.
- [ ] AC-9: sig = 0, valid tid, permission OK: returns 0; no signal queued.
- [ ] AC-10: `tkill(target, SIGKILL)`: entire thread group terminates.
- [ ] AC-11: `tkill(target, SIGSTOP)`: entire thread group stops.
- [ ] AC-12: PID-recycling race (target tid reused mid-syscall): signal IS delivered to new owner (documented limitation — use tgkill).
- [ ] AC-13: 1024 SIGRTMIN via tkill (RLIMIT_SIGPENDING=1024): 1025th → `-EAGAIN`.

### Architecture

```rust
#[syscall(nr = 200, abi = "sysv")]
pub fn sys_tkill(tid: c_int, sig: c_int) -> isize {
    if tid <= 0 { return -EINVAL; }
    if sig < 0 || sig as usize > NSIG { return -EINVAL; }
    Signal::do_tkill(/* tgid */ 0, tid, sig)
}
```

`Signal::do_tkill(tgid_arg, tid, sig)`: `find_task_by_pid_ns(tid, current.pid_ns)` (`-ESRCH` on miss). If `tgid_arg != 0` (called via `tgkill`) verify `task.tgid == tgid_arg`; else (`tkill` path) NO recycling guard — documented limitation. `check_kill_permission(task, sig)` (`-EPERM`). If `sig == 0` return 0. Build kernel-authoritative `UapiSiginfo` (`si_code = SI_TKILL`; `si_pid = current.tgid` in target's pid_ns; `si_uid` mapped to target's user_ns). `do_send_specific(sig, &info, task, PIDTYPE_PID)` — RLIMIT-checked enqueue onto `task.pending` under siglock. `signal_wake_up_state(task, 0)`.

### Out of Scope

- `tgkill(2)` (separate Tier-5 — modern replacement with recycling guard).
- `kill(2)` (separate Tier-5 — process-directed).
- `rt_sigqueueinfo(2)` / `rt_tgsigqueueinfo(2)` (separate Tier-5 — caller-supplied siginfo).
- `rt_sigaction(2)` / `rt_sigprocmask(2)` / `rt_sigtimedwait(2)` (separate Tier-5 docs).
- Signal-delivery internals (`do_send_sig_info`, `send_signal`, `complete_signal` — Tier-3).
- Compat 32-bit `tkill` (Tier-3).
- Implementation code.

### signature

```c
long tkill(int tid, int sig);
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `tid` | `int` | in | Target thread id (positive). |
| `sig` | `int` | in | Signal `0..SIGRTMAX (64)`. `0` = permission probe only. |

### return value

| Value | Meaning |
|---|---|
| `0`  | Signal queued (or `sig == 0` and permission OK). |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EINVAL` | `tid <= 0`; `sig < 0` or `sig > _NSIG (64)`. |
| `EPERM`  | Caller lacks signal permission. |
| `ESRCH`  | No thread `tid` in caller's pid_ns. |
| `EAGAIN` | `RLIMIT_SIGPENDING` exceeded. |

### abi surface

```text
__NR_tkill (x86_64)    = 200
__NR_tkill (i386)      = 238
__NR_tkill (generic)   = 130   /* arm64/riscv/loongarch */
__NR_tkill (powerpc)   = 208
__NR_tkill (s390x)     = 237
__NR_tkill (sparc)     = 220
```

### compatibility contract

REQ-1: Syscall **200** on x86_64; **130** on generic. ABI-stable (legacy binaries depend).

REQ-2: `tid` validated: MUST be `> 0`. Zero or negative → `-EINVAL`. (No implicit "self" via 0; that is what `kill(2)` provides.)

REQ-3: `sig` validated against `[0, _NSIG]`. Out-of-range → `-EINVAL`.

REQ-4: Target lookup: `find_task_by_pid_ns(tid, current.pid_ns)`. Miss or `EXIT_DEAD` (reaped) → `-ESRCH`.

REQ-5: **NO PID-recycling guard** — the central defect of `tkill`. Cannot detect that `tid` has been recycled between caller discovery and syscall. The signal goes to whoever owns `tid` now, possibly wrong process. Documented mitigation: use `tgkill(2)` instead.

REQ-6: Permission: identical to `kill(2)`. Real or effective uid equals target's real or saved-set uid, OR `CAP_KILL` in target's user_ns. `SIGCONT` exception within same session.

REQ-7: `siginfo_t` is **kernel-authoritative** (NOT caller-supplied):
- `si_signo = sig`.
- `si_errno = 0`.
- `si_code = SI_TKILL`.
- `si_pid = current.tgid` (in target's pid_ns).
- `si_uid = current.cred.uid` (mapped through target's user_ns).

REQ-8: Enqueue: signal added to target's **private** pending queue (`task.pending`) via `do_send_specific` — NOT shared tgroup queue.

REQ-9: Wakeup: target woken if `TASK_INTERRUPTIBLE`/`TASK_KILLABLE` and `sig` not blocked. Otherwise remains pending on private queue.

REQ-10: `sig == 0`: validate target + permission only; no enqueue, no wakeup. Useful for "is this thread alive AND I can signal it?" probes.

REQ-11: `SIGKILL` via tkill: still terminates entire thread group (group-action on delivery). `SIGSTOP` via tkill: stops entire thread group. Thread-direction only affects wakeup target.

REQ-12: `RLIMIT_SIGPENDING` enforced for realtime signals; over-limit → `-EAGAIN`. Standard signals coalesce via single-bit semantics.

REQ-13: Audit: `AUDIT_SYSCALL` captures sender tgid, sender uid, target tid, signal, si_code = SI_TKILL.

REQ-14: Cross-pid-ns: `tid` in caller's pid_ns; `si_pid` mapped to receiver's pid_ns.

REQ-15: Compat: no architecture-specific compat conversion needed (both args int).

REQ-16: Deprecation: `tkill` SHOULD emit `pr_info_ratelimited` warning when used by binary newer than threshold (kernel policy controlled via `sysctl kernel.tkill_warn`).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `tid_positive` | INVARIANT | tid > 0 OR -EINVAL. |
| `sig_range_strict` | INVARIANT | sig ∈ [0, NSIG] OR -EINVAL. |
| `siginfo_kernel_authoritative` | INVARIANT | si_pid/si_uid/si_code set by kernel, not caller. |
| `si_code_eq_SI_TKILL` | INVARIANT | every tkill produces si_code == SI_TKILL. |
| `enqueue_on_private_queue` | INVARIANT | enqueued on task.pending, not shared. |
| `rlimit_sigpending_enforced` | INVARIANT | queue full ⟹ -EAGAIN. |
| `no_recycling_guard_documented` | KNOWN-LIMITATION | tkill does NOT verify tgid; use tgkill. |

### Layer 2: TLA+

`kernel/tkill.tla`:
- States: VALIDATE → LOOKUP_TID → CHECK_PERMISSION → BUILD_SIGINFO → ENQUEUE_PRIVATE → WAKE.
- Concurrent: PID recycling between caller-discovery and syscall (modeled as "wrong process"); sibling thread sending; cross-pid-ns.
- Properties:
  - `safety_siginfo_authentic` — si_pid/si_uid/si_code never caller-controlled.
  - `safety_private_queue_only` — signal enqueued on task.pending exclusively.
  - `liveness_target_woken` — eligible target eventually wakes.
  - `documented_recycling_hazard` — model includes "wrong-process" arrow to make explicit that tkill cannot prevent it.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_tkill` post: 0 ⟹ one sigqueue entry on task.pending with SI_TKILL | `sys_tkill` |
| `do_tkill` post: si_code == SI_TKILL; si_pid == current.tgid (ns-mapped) | `do_tkill` |
| `do_send_specific` post: enqueue under siglock; RLIMIT honored | `do_send_specific` |
| `find_task_by_pid_ns` post: returns task with task.pid == tid, caller's pid_ns | lookup |

### Layer 4: Verus / Creusot functional

POSIX `pthread_kill(3)` semantic equivalence (note: modern glibc `pthread_kill` uses `tgkill`, not `tkill`). LTP `tkill01..02`.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

- **Per-tid > 0 strict** — defense against per-sentinel-value confusion (no implicit "self" via 0).
- **Per-sig range strict** — defense against per-out-of-band signal numbers.
- **Per-permission via real/saved uid OR CAP_KILL** — defense against per-cross-uid injection.
- **Per-siginfo kernel-authoritative** — defense against per-spoof of sender identity (unlike `rt_sigqueueinfo`, caller cannot supply siginfo here).
- **Per-private-queue enqueue** — defense against per-sibling stealing the signal.
- **Per-pid_ns translation for tid lookup** — defense against per-namespace pid confusion.
- **Per-deprecation warning** — defense against per-legacy-code-path reliance; encourages migration to `tgkill(2)`.
- **Documented per-PID-recycling hazard** — explicit known-limitation; mitigation is `tgkill(2)`.

### grsecurity / pax-style reinforcement

- **PAX_RANDKSTACK at syscall entry** — randomizes kernel-stack layout per call; defeats per-stack-layout reuse during the brief enqueue window.
- **PaX UDEREF** — `do_tkill` does not dereference user pointers in its main path (siginfo kernel-built); UDEREF policy applies uniformly across signal entries.
- **PaX MEMORY_SANITIZE** — `UapiSiginfo::default()` zeroes all 128 bytes including union arms not relevant to SI_TKILL; receiver reading via `rt_sigtimedwait` or `signalfd4` never observes per-stack bytes.
- **GRKERNSEC_SIGNALS SI_TKILL authenticity** — `si_code = SI_TKILL` set by kernel here. Combined with `rt_sigqueueinfo`'s user-origin allow-list (which REJECTS user-supplied `SI_TKILL`), receiver can rely on `SI_TKILL` indicating a true `tkill`/`tgkill` invocation, not a forgery.
- **GRKERNSEC_SIGNALS SI_USER strict-clone (cross-reference)** — `kill(2)` produces `SI_USER`, `tkill` produces `SI_TKILL`; grsec distinguishes them and forbids user code from forging either via `rt_sigqueueinfo`.
- **GRKERNSEC_BRUTE on signal-flood** — repeated cross-uid `tkill` attempts (probing for permission races) trip per-uid brute-force counter.
- **GRKERNSEC bounded-tid for tkill/tgkill lookups** — `find_task_by_pid_ns` rate-limited per-uid against blind tid enumeration. Combined with lack of recycling guard on `tkill`, this is critical: rate-limiting prevents spraying `tkill` across tid space looking for races.
- **PaX KERNEXEC** — `do_tkill` in read-only-after-init kernel text; cannot be patched to skip permission check or produce a forged si_code.
- **GRKERNSEC_HARDEN_PTRACE** — tracer cannot use `tkill` to inject across credential boundaries; cross-cred ptrace check fails.
- **GRKERNSEC_PROC restrictions** — `/proc/<pid>/task/<tid>` per-uid hide-filtered; unprivileged callers cannot enumerate foreign tids.
- **Per-grsec gradm — tkill deprecation policy** — RBAC may forbid certain subjects from `tkill`, mandating `tgkill(2)` (with built-in recycling guard) for thread-directed signaling. SUID binaries should be in this policy by default.
- **PaX TASK_HARDENING** — sigqueue slab on private queue audited under `RLIMIT_SIGPENDING`; over-limit hard `-EAGAIN` with no kmalloc retry escalation, defeating per-thread queue-flood DoS via `tkill`.

