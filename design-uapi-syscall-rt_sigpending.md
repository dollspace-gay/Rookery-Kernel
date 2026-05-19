---
title: "Tier-5 syscall: rt_sigpending(2) — syscall 127"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`rt_sigpending(2)` returns the set of signals that are **pending
delivery to the calling thread but currently blocked** by the thread's
signal mask. A pending signal is one that has been generated (via
`kill(2)`, `tkill(2)`, `tgkill(2)`, `pidfd_send_signal(2)`,
`raise(3)`, hardware fault, etc.) but cannot be delivered yet because
the thread's `blocked` mask filters it out.

The returned set is the UNION of:

- The **per-thread** pending queue (`task.pending.signal`) for signals
  directed at this specific thread (e.g. by `tkill`, `tgkill` targeting
  this tid, or synchronous traps).
- The **per-process (shared)** pending queue (`task.signal.shared_pending.signal`)
  for process-directed signals (e.g. by `kill(pid, ...)`,
  `pidfd_send_signal` on a tgid, broadcast).

INTERSECTED with the caller's current `blocked` mask. Signals that are
pending but NOT blocked do not appear (they would have been delivered
at the next syscall boundary or already drained).

Historical `sigpending(2)` used the smaller `sigset_t` (single-word);
on modern arches only `rt_sigpending` is wired. Userspace
`sigpending(3)` maps to `rt_sigpending(2)`.

This Tier-5 covers `kernel/signal.c::SYSCALL_DEFINE2(rt_sigpending, ...)`
plus its `do_sigpending` workhorse (~40 lines of entry surface).

### Acceptance Criteria

- [ ] AC-1: Syscall number is 127 on x86_64; 136 on generic.
- [ ] AC-2: No signals pending: `rt_sigpending(&s, 8)` returns 0, `s == 0`.
- [ ] AC-3: Block SIGUSR1, then `kill(self, SIGUSR1)`, then `rt_sigpending(&s, 8)`: bit for SIGUSR1 set in `s`.
- [ ] AC-4: Same as AC-3 but with SIGUSR1 NOT blocked: bit NOT set (delivered already).
- [ ] AC-5: Block SIGRTMIN, queue 3 instances via `rt_sigqueueinfo`, then `rt_sigpending`: SIGRTMIN bit set (count not reported).
- [ ] AC-6: `rt_sigpending(&s, 4)`: wrong size → `-EINVAL`.
- [ ] AC-7: `rt_sigpending(NULL, 8)`: `-EFAULT`.
- [ ] AC-8: `rt_sigpending(bad_ptr, 8)`: `-EFAULT`.
- [ ] AC-9: Block SIGKILL attempted (no-op per REQ-7); kill via SIGUSR1 instead and confirm SIGKILL never appears in result.
- [ ] AC-10: Per-process signal `kill(getpid(), SIGUSR2)` while SIGUSR2 blocked: appears in result of ANY thread in process that has SIGUSR2 blocked.
- [ ] AC-11: Calling `rt_sigpending` does NOT consume signals (call twice: same result, signals still pending).
- [ ] AC-12: Concurrent `kill(self, SIGUSR1)` and `rt_sigpending`: result is either before-or-after, not torn.

### Architecture

```rust
#[syscall(nr = 127, abi = "sysv")]
pub fn sys_rt_sigpending(
    set: UserPtr<Sigset>,
    sigsetsize: usize,
) -> isize {
    // REQ-2: sigsetsize
    if sigsetsize != size_of::<Sigset>() { return -EINVAL; }

    let mut snapshot = Sigset::default();
    Signal::do_sigpending(&mut snapshot);

    // REQ-8: copy out
    snapshot.to_user(set)?;
    0
}
```

`Signal::do_sigpending(out)`:
1. let task = Task::current();
2. let _g = SpinLockGuard::lock_irq(&task.sighand.siglock);
3. /* REQ-4: union per-thread + shared */
4. let mut pending = task.pending.signal.clone();
5. pending.union_with(&task.signal.shared_pending.signal);
6. /* REQ-5: intersect with blocked */
7. pending.intersect_with(&task.blocked);
8. /* REQ-7 is implicit: SIGKILL/SIGSTOP cleared in blocked, so 0 in result */
9. *out = pending;
10. drop(_g);

### Out of Scope

- `rt_sigaction(2)` / `rt_sigprocmask(2)` / `rt_sigsuspend(2)` (separate Tier-5 docs).
- `rt_sigtimedwait(2)` / `rt_sigqueueinfo(2)` / `rt_tgsigqueueinfo(2)` (separate Tier-5 docs).
- `signalfd4(2)` (separate Tier-5 doc).
- Signal-delivery path (`kernel/signal.c::get_signal` — Tier-3).
- Compat (32-bit on 64-bit) `compat_rt_sigpending` (Tier-3).
- `/proc/<pid>/status` `SigPnd:` field (Tier-3 procfs).
- Implementation code.

### signature

```c
long rt_sigpending(sigset_t *set, size_t sigsetsize);
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `set` | `sigset_t *` | out | Pointer to receive the pending-and-blocked set. MUST be non-NULL. |
| `sigsetsize` | `size_t` | in | Size of `sigset_t` in bytes — MUST equal `sizeof(sigset_t)` for the kernel's native word width. On LP64, this is `8`. |

### return value

| Value | Meaning |
|---|---|
| `0`  | Success. `*set` populated with pending-and-blocked signals. |
| `-1` + `errno` | Failure. `*set` unmodified. |

### errors

| errno | Trigger |
|---|---|
| `EINVAL` | `sigsetsize != sizeof(sigset_t)` for the kernel's native pointer width. |
| `EFAULT` | `set` is NULL or not writable. |

### abi surface

```text
__NR_rt_sigpending (x86_64)    = 127
__NR_rt_sigpending (i386)      = 176
__NR_rt_sigpending (generic)   = 136   /* arm64/riscv/loongarch */
__NR_rt_sigpending (powerpc)   = 175
__NR_rt_sigpending (s390x)     = 176
__NR_rt_sigpending (sparc)     = 109
__NR_rt_sigpending (mips O32)  = 4196
__NR_rt_sigpending (mips N64)  = 5125
```

### `sigset_t` layout

```c
#define _NSIG       64
#define _NSIG_BPW   (__BITS_PER_LONG)
#define _NSIG_WORDS (_NSIG / _NSIG_BPW)

typedef struct {
    unsigned long sig[_NSIG_WORDS];
} sigset_t;
```

### Conceptual computation

```text
result = (current.pending.signal UNION current.signal.shared_pending.signal)
         INTERSECT current.blocked
```

### compatibility contract

REQ-1: Syscall number is **127** on x86_64; **136** on generic-syscall
arches. ABI-stable forever.

REQ-2: `sigsetsize` MUST equal the kernel's native `sizeof(sigset_t)`
(8 bytes on LP64). Mismatch MUST return `-EINVAL` BEFORE any user-memory
access.

REQ-3: The result set MUST be computed under
`task.sighand.siglock` with IRQs disabled to ensure a consistent
snapshot across concurrent signal-delivery on another CPU.

REQ-4: The set MUST include signals from BOTH the per-thread pending
queue AND the per-process shared pending queue.

REQ-5: The set MUST be INTERSECTED with the thread's current `blocked`
mask — un-blocked pending signals (which would be delivered at the next
return-to-user) are NOT reported.

REQ-6: Realtime signals (`SIGRTMIN..SIGRTMAX`, 32..64) appear in the
set if at least one instance is queued and the signal is blocked. The
COUNT of queued realtime instances is NOT exposed via this syscall (use
`/proc/<pid>/status` `SigQ:` for cumulative count).

REQ-7: `SIGKILL (9)` and `SIGSTOP (19)` can never appear in the result
because they can never be blocked (`task.blocked` is sanitized
unconditionally; intersection yields 0 for those bits).

REQ-8: The `set` pointer is written exactly once with the full snapshot.
If `copy_to_user` fails, the syscall returns `-EFAULT` and may leave
`*set` partially written (POSIX permits this; the kernel does not roll
back).

REQ-9: Multithreaded process: the per-thread pending queue is private to
the calling thread; the shared pending queue is visible to ALL threads
(only one thread will ultimately dequeue and deliver a shared signal).
Thus `rt_sigpending` results may differ across threads of the same
process — each sees its own per-thread queue UNION the shared queue,
intersected with its own mask.

REQ-10: A signal that has been generated AND is unblocked but not yet
delivered (still in the queue, awaiting return-to-user) is NOT in the
result — it has already been removed from the "pending-and-blocked"
set by virtue of being unblocked.

REQ-11: Audit: `AUDIT_SYSCALL` record on `rt_sigpending` does NOT
include the returned set by default (it is read-only and considered
non-mutating).

REQ-12: Compat (32-bit userspace on 64-bit kernel): handled by
`compat_rt_sigpending` with `sizeof(compat_sigset_t) == 8`.

REQ-13: The syscall does NOT consume or alter any pending signal — it
is read-only. Calling it repeatedly with the same mask returns the
same set provided no signal-generation or delivery has occurred in
between.

REQ-14: The user buffer MAY alias with internal kernel buffers — copy
is done from a local `sigset_t` snapshot, never directly from the
pending queue field, so concurrent delivery cannot tear the user copy.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `sigsetsize_strict` | INVARIANT | sigsetsize ≠ 8 ⟹ -EINVAL. |
| `result_is_intersection` | INVARIANT | Post: out == (per_thread UNION shared) AND blocked. |
| `kill_stop_excluded` | INVARIANT | Post: out AND {SIGKILL, SIGSTOP} == 0. |
| `siglock_held_during_snapshot` | INVARIANT | Snapshot under siglock + IRQs off. |
| `read_only_no_mutation` | INVARIANT | task.pending unchanged across call. |
| `no_torn_user_copy` | INVARIANT | User receives complete snapshot or `-EFAULT`. |

### Layer 2: TLA+

`kernel/rt_sigpending.tla`:
- States: VALIDATE → LOCK → UNION → INTERSECT → UNLOCK → COPY_OUT.
- Concurrent: send_signal on another CPU AND get_signal dequeue.
- Properties:
  - `safety_consistent_snapshot` — result reflects a single point-in-time state of pending+blocked.
  - `safety_no_mutation` — pending queue length unchanged.
  - `liveness_terminates` — every call completes in bounded time (no queue traversal at this layer; bit-OR is O(1) per word).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_rt_sigpending` post: out reflects pending-and-blocked set | `sys_rt_sigpending` |
| Union-then-intersect commutative with concurrent delivery only at lock boundary | `Signal::do_sigpending` |
| `Sigset::to_user` byte-layout 8 bytes | `Sigset::to_user` |

### Layer 4: Verus / Creusot functional

POSIX `sigpending(2)` semantic equivalence. LTP `rt_sigpending01..02`
pass. Manual stress: block all signals, queue many, read pending —
expected = blocked-intersect-generated.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`rt_sigpending(2)` reinforcement:

- **Per-sigsetsize strict** — defense against per-mask-truncation.
- **Per-siglock IRQ-off snapshot** — defense against torn read.
- **Per-snapshot-not-direct-field copy** — defense against
  concurrent-delivery race producing torn user copy.
- **Per-SIGKILL/SIGSTOP never reported** — defense against
  false-positive signaling that kernel allows blocking.
- **Per-read-only** — no state mutation; idempotent.
- **Per-user-pointer validated via put_user** — defense against
  arbitrary write via bogus pointer.

### grsecurity / pax-style reinforcement

- **PAX_RANDKSTACK at syscall entry** — each `rt_sigpending` call
  randomizes kernel stack offset; defeats per-syscall ROP probing for
  the signal subsystem.
- **PaX UDEREF on `sigset_t` writeback** — the `put_user` to `set` is
  performed via UDEREF / SMAP+PAN-toggled fast path; a kernel pointer
  that mistakenly aliases user space cannot silently leak kernel
  memory contents into the destination buffer.
- **PaX MEMORY_SANITIZE** — local `Sigset snapshot` is initialized to
  all-zero before population; no uninitialized stack bytes leak into
  the user buffer via padding (sigset_t has no padding on LP64, but
  the discipline is enforced for ILP32 + future _NSIG widening).
- **GRKERNSEC_PROC restrictions** — `/proc/<pid>/status` `SigPnd` and
  `ShdPnd` fields are subject to per-uid hide-filter; userspace
  enumeration of another user's pending sets via /proc is gated even
  though `rt_sigpending` on SELF is always permitted.
- **GRKERNSEC_SIGNALS strict-clone** — pending-queue access via ptrace
  (`PTRACE_PEEKSIGINFO`) is cross-cred-denied; an attacker cannot use
  ptrace to read pending info that would otherwise be hidden by RBAC.
- **GRKERNSEC_BRUTE on signal-flood polling** — sustained high-rate
  `rt_sigpending` calls coupled with self-kill loops (used historically
  to leak timing side-channels through the queue) are observable by
  the grsec rate-policy and may trigger per-uid brute-force back-off.
- **PaX KERNEXEC** — the `do_sigpending` code path lives in
  read-only-after-init text; cannot be patched to leak the unmasked
  pending set.
- **GRKERNSEC_HARDEN_PTRACE** — tracer cannot use `PTRACE_PEEKSIGINFO`
  across credential boundaries; the per-thread queue is accessible only
  to the tracee itself via this syscall.
- **Per-grsec `gradm` learning** — RBAC may restrict whether a subject
  is allowed to observe pending realtime queue presence (used in
  high-assurance configs to suppress side-channel via signal arrival
  timing).
- **Per-grsec audit-on-read** — if `audit_signal_event_enabled`,
  `rt_sigpending` may be recorded in audit log (off by default;
  enabled for forensic workloads).

