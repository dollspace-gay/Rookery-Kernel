---
title: "Tier-5 syscall: rt_tgsigqueueinfo(2) — syscall 297"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`rt_tgsigqueueinfo(2)` queues a realtime signal with a caller-supplied
`siginfo_t` to a SPECIFIC thread of a specific thread group. It is
the thread-directed counterpart of `rt_sigqueueinfo(2)` — same
allow-list, same sender-identity overwrite, same `RLIMIT_SIGPENDING`
accounting, but the delivery target is `(tgid, tid)` and enqueue
goes to the thread's PRIVATE pending queue (`task.pending`).

The `tgid` argument is the PID-recycling guard: if `tid` has been
recycled into a different process between the caller's lookup and
the syscall, the kernel detects the mismatch and returns `-ESRCH`.

Critical for: glibc `pthread_sigqueue(3)`, debugger-controlled
per-thread signal injection, GC pause primitives, per-thread cancel
(`pthread_cancel` via `SIGCANCEL`), libraries needing
"deliver-to-this-thread-only" without racing pid recycling.

This Tier-5 covers `kernel/signal.c::SYSCALL_DEFINE4(rt_tgsigqueueinfo, ...)`.

### Acceptance Criteria

- [ ] AC-1: Syscall number 297 on x86_64; 240 on generic.
- [ ] AC-2: Send SIGRTMIN to (peer.tgid, peer.tid_2) with `si_value.sival_int = 0xCAFEBABE`. ONLY tid_2 dequeues.
- [ ] AC-3: `si_signo` differs from `sig`: `-EINVAL`.
- [ ] AC-4: `tgid=0` or `tid=0`: `-EINVAL`.
- [ ] AC-5: tid exists but its tgid != argument: `-ESRCH` (recycling guard).
- [ ] AC-6: Non-existent tid: `-ESRCH`.
- [ ] AC-7: Foreign target, `si_code = SI_TKILL`: `-EPERM`.
- [ ] AC-8: Self target, `si_code = SI_TKILL`: success.
- [ ] AC-9: Receiver observes `info.si_pid == caller.tgid` regardless of supplied value.
- [ ] AC-10: 1024 SIGRTMIN to thread (RLIMIT_SIGPENDING=1024): 1025th → `-EAGAIN`.
- [ ] AC-11: Cross-uid no CAP_KILL: `-EPERM`. Same uid: success.
- [ ] AC-12: `sig == 0` with valid `(tgid, tid)`: returns 0; no signal queued.
- [ ] AC-13: Realtime signal queued 3 times to same thread: 3 distinct siginfos dequeued FIFO.
- [ ] AC-14: `uinfo` faulting page: `-EFAULT`; no partial enqueue.
- [ ] AC-15: Thread blocked on `sig`: queued on private queue; unblock → delivered before next enqueue (FIFO for rt).

### Architecture

```rust
#[syscall(nr = 297, abi = "sysv")]
pub fn sys_rt_tgsigqueueinfo(
    tgid: pid_t, tid: pid_t, sig: c_int,
    uinfo: UserPtr<UapiSiginfo>,
) -> isize {
    if tgid <= 0 || tid <= 0 { return -EINVAL; }
    if sig < 0 || sig as usize > NSIG { return -EINVAL; }
    let mut info = UapiSiginfo::default();
    Siginfo::copy_from_user_strict(uinfo, &mut info)?;
    if info.si_signo != sig { return -EINVAL; }

    let me = Task::current();
    let target_is_self = (tgid == me.tgid) && (tid == me.tid);
    Signal::check_user_si_code(&info, target_is_self)?;

    info.si_pid = me.tgid_in(Task::pid_ns_of_tid(tid));
    info.si_uid = Cred::current().uid.in_ns(Task::userns_of_tid(tid));

    Signal::do_rt_tgsigqueueinfo(tgid, tid, sig, &info)?;
    0
}
```

`Signal::do_rt_tgsigqueueinfo`: `find_task_by_pid_ns(tid)` → if `task.tgid != tgid` return `-ESRCH` (recycling guard) → `check_kill_permission` → if `sig == 0` return Ok → `do_send_specific(sig, &info, task, PIDTYPE_PID)` (RLIMIT-checked enqueue onto private queue under siglock) → `signal_wake_up_state`.

### Out of Scope

- `rt_sigqueueinfo(2)` (separate Tier-5 — process-directed variant).
- `rt_sigaction(2)` / `rt_sigprocmask(2)` / `rt_sigtimedwait(2)` / `rt_sigpending(2)` / `rt_sigsuspend(2)` (separate Tier-5 docs).
- `kill(2)` / `tkill(2)` / `tgkill(2)` (separate Tier-5 docs).
- Signal-delivery internals (`do_send_sig_info`, `send_signal`, `complete_signal` — Tier-3).
- Compat 32-bit variants (Tier-3).
- POSIX timer + mqueue siginfo paths (Tier-3).
- Implementation code.

### signature

```c
long rt_tgsigqueueinfo(pid_t tgid, pid_t tid, int sig, siginfo_t *uinfo);
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `tgid` | `pid_t` | in | Target thread group id (positive). PID-recycling guard. |
| `tid` | `pid_t` | in | Target thread id (positive). Must belong to thread group `tgid`. |
| `sig` | `int` | in | Signal `0..SIGRTMAX (64)`. `0` = permission probe only. |
| `uinfo` | `siginfo_t *` | in | Caller-prepared siginfo. `si_signo == sig`; `si_code` in user allow-list. |

### return value

| Value | Meaning |
|---|---|
| `0`  | Signal queued (or `sig == 0` and permission OK). |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EINVAL` | `tgid <= 0`, `tid <= 0`, `sig < 0` or `sig > _NSIG`; `uinfo.si_signo != sig`. |
| `EPERM`  | `uinfo.si_code` reserved (`SI_TKILL`, `SI_TIMER`, `SI_MESGQ`, etc.) for non-self target without `CAP_SYS_ADMIN`. |
| `EPERM`  | Caller lacks signal permission. |
| `ESRCH`  | No thread `tid`, OR `task.tgid != tgid` argument (recycling guard). |
| `EAGAIN` | `RLIMIT_SIGPENDING` exceeded. |
| `EFAULT` | `uinfo` not accessible. |

### abi surface

```text
__NR_rt_tgsigqueueinfo (x86_64)    = 297
__NR_rt_tgsigqueueinfo (i386)      = 335
__NR_rt_tgsigqueueinfo (generic)   = 240   /* arm64/riscv/loongarch */
__NR_rt_tgsigqueueinfo (powerpc)   = 322
__NR_rt_tgsigqueueinfo (s390x)     = 330
__NR_rt_tgsigqueueinfo (sparc)     = 326
```

### Allow-list for user-origin (same as `rt_sigqueueinfo`)

```text
SI_USER        only when (tgid == caller.tgid AND tid == caller.tid)
SI_QUEUE       unrestricted
other si_code <= 0 — passed through unless in {SI_TKILL, SI_TIMER, SI_MESGQ,
                    SI_ASYNCIO, SI_SIGIO, SI_DETHREAD} (require CAP_SYS_ADMIN
                    for non-self)
positive si_code — kernel-reserved; rejected for non-self
```

### compatibility contract

REQ-1: Syscall **297** on x86_64; **240** on generic. ABI-stable.

REQ-2: `sig` validated against `[0, _NSIG]`. Out-of-range → `-EINVAL`.

REQ-3: `tgid` and `tid` must be positive; else `-EINVAL`. No broadcast form.

REQ-4: `uinfo` copied via strict 128-byte `__copy_siginfo_from_user`. UDEREF-protected; compat zero-extends.

REQ-5: `uinfo.si_signo == sig`; mismatch → `-EINVAL`.

REQ-6: User-origin `si_code` allow-list (same as `rt_sigqueueinfo`):
- Self target (`tgid == caller.tgid AND tid == caller.tid`): permissive.
- Foreign: `si_code <= 0` AND not in `{SI_TKILL, SI_TIMER, SI_MESGQ, SI_ASYNCIO, SI_SIGIO, SI_DETHREAD}` unless `CAP_SYS_ADMIN`. Else `-EPERM`.

REQ-7: Sender-identity overwrite:
- `si_pid = current.tgid` (in target's pid_ns).
- `si_uid = current.cred.uid` (mapped through target's user_ns).

REQ-8: Target lookup via `find_task_by_pid_ns(tid, current.pid_ns)`. Miss or reaped → `-ESRCH`.

REQ-9: **PID-recycling guard**: verify `task.tgid == tgid` argument. Mismatch → `-ESRCH`. Critical difference from `tkill(2)`.

REQ-10: Permission: same as `kill(2)` / `rt_sigqueueinfo`. Real or effective uid equals target's real or saved-set uid, OR `CAP_KILL` in target's user_ns.

REQ-11: Enqueue goes to THREAD's private queue (`task.pending`), NOT shared tgroup queue. Only this thread will dequeue.

REQ-12: Realtime queueing: `struct sigqueue` allocation per call; FIFO link onto `task.pending.list`. `RLIMIT_SIGPENDING` per real-uid; over-limit → `-EAGAIN`.

REQ-13: Non-realtime signal: classical single-bit on the thread's private queue; second queue with bit set drops siginfo.

REQ-14: Wakeup: target woken if `TASK_INTERRUPTIBLE`/`TASK_KILLABLE` and `sig` not blocked. Otherwise remains pending.

REQ-15: `sig == 0`: validate target + tgid-guard + permission; no enqueue; no wakeup.

REQ-16: Audit: `AUDIT_SYSCALL` captures sender tgid+uid, target tgid+tid, signum, sanitized si_code.

REQ-17: Cross-pid-ns: tgid/tid in caller's pid_ns. Recycling guard runs in caller's view. Receiver observes `si_pid` in receiver's pid_ns.

REQ-18: Compat 32-bit: `compat_rt_tgsigqueueinfo` narrows padding; preserves allow-list + recycling guard.

REQ-19: `RLIMIT_SIGPENDING` under `current.cred.user.lock`; no cross-uid overflow.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `tgid_tid_positive` | INVARIANT | tgid > 0 ∧ tid > 0 OR -EINVAL. |
| `sig_range_strict` | INVARIANT | sig ∈ [0, NSIG] OR -EINVAL. |
| `si_signo_matches_sig` | INVARIANT | info.si_signo != sig ⟹ -EINVAL. |
| `recycling_guard` | INVARIANT | task.tgid != tgid_arg ⟹ -ESRCH. |
| `user_si_code_allow_list` | INVARIANT | non-self ⟹ si_code in allow-list OR -EPERM. |
| `sender_identity_overwritten` | INVARIANT | si_pid/si_uid set kernel-authoritatively. |
| `siginfo_copy_bounds` | INVARIANT | 128-byte strict copy on LP64. |
| `rlimit_sigpending_enforced` | INVARIANT | queue full ⟹ -EAGAIN. |
| `enqueue_on_private_queue` | INVARIANT | enqueued on task.pending, not shared. |

### Layer 2: TLA+

`kernel/rt_tgsigqueueinfo.tla`:
- States: VALIDATE → COPY → CHECK_SI_SIGNO → CHECK_SI_CODE → OVERWRITE_SENDER_ID → LOOKUP_TID → RECYCLING_GUARD → CHECK_PERMISSION → ENQUEUE_PRIVATE → WAKE.
- Concurrent: PID recycling between lookup and tgid-check; sibling thread sending; cross-pid-ns.
- Properties:
  - `safety_recycling_guard_holds` — if tgid changes between caller's check and enqueue, syscall returns -ESRCH; signal NEVER delivered to wrong process.
  - `safety_private_queue_only` — observable only by target thread, never siblings.
  - `safety_si_pid_authentic` — receiver sees kernel-authoritative si_pid.
  - `safety_rlimit_bounded` — slab allocation ≤ RLIMIT_SIGPENDING per uid.
  - `liveness_rt_signal_delivered` — every queued rt signal eventually delivered.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_rt_tgsigqueueinfo` post: 0 ⟹ exactly one sigqueue entry on task.pending | `sys_rt_tgsigqueueinfo` |
| `do_rt_tgsigqueueinfo` post: task.tgid == arg.tgid verified | `do_rt_tgsigqueueinfo` |
| `do_send_specific` post: enqueue on private queue under siglock | `do_send_specific` |
| `check_user_si_code` post: Ok ⟹ allow-listed OR self | `check_user_si_code` |

### Layer 4: Verus / Creusot functional

POSIX `pthread_sigqueue(3)` semantic equivalence. LTP `rt_tgsigqueueinfo01..02`, glibc `nptl/tst-pthread-sigqueue*`.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

- **Per-PID-recycling guard via tgid argument** — defense against per-tid-reuse race (the reason this syscall exists vs. the older `tkill(2)`).
- **Per-strict `si_signo == sig`** — defense against per-mismatch signal smuggling.
- **Per-user-origin si_code allow-list** — defense against per-spoof of `SI_TIMER`/`SI_TKILL`/`SI_MESGQ`.
- **Per-sender identity overwrite** — defense against per-spoof of `si_pid`/`si_uid`.
- **Per-RLIMIT_SIGPENDING strict** — defense against per-slab-exhaustion via per-thread queue flooding.
- **Per-permission via real/saved uid OR CAP_KILL** — defense against per-cross-uid injection.
- **Per-private-queue enqueue** — defense against per-sibling stealing the signal.
- **Per-pid_ns translation for tid lookup** — defense against per-namespace pid confusion.
- **Per-user_ns mapping for si_uid** — defense against per-container uid leakage.

### grsecurity / pax-style reinforcement

- **PAX_RANDKSTACK at syscall entry** — randomizes kernel-stack layout per call; combined with per-call `UapiSiginfo` zero-init, defeats per-stack-leak via siginfo padding side-channel.
- **PaX UDEREF on `siginfo_t`** — `copy_siginfo_from_user` runs with SMAP/PAN + UDEREF; pointer mistakes cannot dereference user space.
- **PaX MEMORY_SANITIZE** — local `UapiSiginfo` zeroed before strict copy; enqueued siginfo never leaks per-stack/heap bytes into receiver.
- **GRKERNSEC_SIGNALS SI_USER strict-clone** — `si_code = SI_USER` to foreign `(tgid, tid)` REJECTED; SI_USER reserved for kernel-synthesized `kill(2)`.
- **GRKERNSEC_SIGNALS SI_TKILL / SI_QUEUE allow-list** — only `SI_QUEUE` freely allowed to non-self targets; `SI_TKILL`/`SI_TIMER`/`SI_MESGQ`/`SI_ASYNCIO`/`SI_SIGIO`/`SI_DETHREAD` gated by `CAP_SYS_ADMIN`. Allow-list at REQ-6 enforced and re-audited by grsec on suspicious patterns.
- **GRKERNSEC_BRUTE on signal-flood** — repeated cross-uid `rt_tgsigqueueinfo` (probing for PID-recycle races) observed by per-uid brute-force counter.
- **GRKERNSEC bounded-tid for tgkill-class lookups** — `find_task_by_tid_pid_ns` rate-limited per-uid against blind tid enumeration; combined with PID-recycling guard, prevents blind-injection attempts.
- **PaX KERNEXEC** — `do_rt_tgsigqueueinfo` and `do_send_specific` in read-only-after-init kernel text; cannot be patched to skip recycling guard or allow-list.
- **GRKERNSEC_HARDEN_PTRACE** — tracer cannot use `rt_tgsigqueueinfo` to inject across credential boundaries; cross-cred ptrace check fails before enqueue.
- **GRKERNSEC_PROC restrictions** — `/proc/<tid>/status` (used to discover tgid) per-uid hide-filtered; unprivileged callers cannot enumerate foreign tids.
- **PaX TASK_HARDENING + RANDSTRUCT** — sigqueue slab allocation audited under `RLIMIT_SIGPENDING`; `task_struct.pending.list` head in randomized layout, blocking deterministic OOB-write redirects of the private pending-queue head.

