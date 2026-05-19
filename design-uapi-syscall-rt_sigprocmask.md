---
title: "Tier-5 syscall: rt_sigprocmask(2) — syscall 14"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`rt_sigprocmask(2)` examines or alters the calling thread's **signal
mask** — the set of signals whose delivery is currently blocked. While a
signal is blocked it remains queued in the per-thread or per-process
pending queue but is not delivered (handler not invoked, default action
not executed) until the mask is cleared. The mask is per-thread: in a
multithreaded process each thread has its own mask, although the
per-process pending queue and disposition table are shared.

The `how` argument selects the update operation:

- `SIG_BLOCK`   — `new_mask = current_mask UNION *set`.
- `SIG_UNBLOCK` — `new_mask = current_mask AND NOT *set`.
- `SIG_SETMASK` — `new_mask = *set` (verbatim replace).

`SIGKILL (9)` and `SIGSTOP (19)` CANNOT be blocked — the kernel silently
masks them out of any input set. The historical `sigprocmask(2)` (using
the smaller `sigset_t`) is no longer wired on modern arches; userspace
`sigprocmask(3)` always maps to `rt_sigprocmask(2)`.

This Tier-5 covers `kernel/signal.c::SYSCALL_DEFINE4(rt_sigprocmask, ...)`
plus its `do_sigprocmask` workhorse (~70 lines of entry surface).

### Acceptance Criteria

- [ ] AC-1: Syscall number is 14 on x86_64; 135 on generic.
- [ ] AC-2: `rt_sigprocmask(SIG_BLOCK, &m_usr1, NULL, 8)`: SIGUSR1 added to mask.
- [ ] AC-3: `rt_sigprocmask(SIG_UNBLOCK, &m_usr1, NULL, 8)`: SIGUSR1 removed.
- [ ] AC-4: `rt_sigprocmask(SIG_SETMASK, &empty, NULL, 8)`: mask cleared.
- [ ] AC-5: `rt_sigprocmask(SIG_BLOCK, &m_kill_stop, &old, 8)`: returns 0; effective mask EXCLUDES SIGKILL/SIGSTOP.
- [ ] AC-6: `rt_sigprocmask(3, &set, NULL, 8)`: invalid `how` → `-EINVAL`.
- [ ] AC-7: `rt_sigprocmask(3, NULL, &oset, 8)`: NULL set ⟹ `how` IGNORED, returns 0 with oset populated.
- [ ] AC-8: `rt_sigprocmask(SIG_BLOCK, &set, NULL, 4)`: wrong size → `-EINVAL`.
- [ ] AC-9: `rt_sigprocmask(SIG_BLOCK, bad_ptr, NULL, 8)`: `-EFAULT`.
- [ ] AC-10: `oset` reflects PRE-call mask.
- [ ] AC-11: Unblocking SIGUSR1 while a SIGUSR1 is pending: TIF_SIGPENDING set; signal delivered at next entry-to-user.
- [ ] AC-12: Multithreaded: rt_sigprocmask in thread A does NOT change thread B's mask.
- [ ] AC-13: Realtime signal (e.g. SIGRTMIN+1) can be blocked; multiple instances queue while blocked.
- [ ] AC-14: SIG_IGN'd signal: unblocking discards pending instances.

### Architecture

```rust
#[syscall(nr = 14, abi = "sysv")]
pub fn sys_rt_sigprocmask(
    how: i32,
    set: UserPtr<Sigset>,
    oset: UserPtr<Sigset>,
    sigsetsize: usize,
) -> isize {
    // REQ-2: sigsetsize
    if sigsetsize != size_of::<Sigset>() { return -EINVAL; }

    // REQ-3: validate `how` only when `set` provided
    let new_mask: Option<Sigset> = if !set.is_null() {
        match how {
            SIG_BLOCK | SIG_UNBLOCK | SIG_SETMASK => {}
            _ => return -EINVAL,
        }
        Some(Sigset::from_user(set)?)
    } else {
        None
    };

    let mut old_mask = Sigset::default();
    Signal::do_sigprocmask(how, new_mask.as_ref(), Some(&mut old_mask));

    if !oset.is_null() {
        old_mask.to_user(oset)?;
    }
    0
}
```

`Signal::do_sigprocmask(how, set, oset)`:
1. let task = Task::current();
2. let _g = SpinLockGuard::lock_irq(&task.sighand.siglock);
3. /* REQ-10: snapshot pre-call mask */
4. if let Some(o) = oset { *o = task.blocked.clone(); }
5. /* REQ-3..7: apply */
6. if let Some(s) = set {
7.   let mut effective = s.clone();
8.   /* REQ-4: sanitize KILL/STOP */
9.   effective.del(SIGKILL); effective.del(SIGSTOP);
10.  match how {
11.    SIG_BLOCK   => task.blocked.union_with(&effective),
12.    SIG_UNBLOCK => task.blocked.minus(&effective),
13.    SIG_SETMASK => task.blocked = effective,
14.    _ => unreachable!(),
15.  }
16.  /* REQ-8: recalc */
17.  Signal::recalc_sigpending(task);
18. }
19. drop(_g);

### Out of Scope

- `rt_sigaction(2)` (separate Tier-5 doc).
- `rt_sigpending(2)` / `rt_sigsuspend(2)` / `rt_sigtimedwait(2)` (separate Tier-5 docs).
- `rt_sigreturn(2)` (separate Tier-5 doc).
- Signal-delivery path (`kernel/signal.c::get_signal`, `do_signal` — Tier-3).
- Compat (32-bit on 64-bit) `compat_rt_sigprocmask` (Tier-3).
- Implementation code.

### signature

```c
long rt_sigprocmask(int how,
                    const sigset_t *set,
                    sigset_t *oset,
                    size_t sigsetsize);
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `how` | `int` | in | `SIG_BLOCK (0)`, `SIG_UNBLOCK (1)`, or `SIG_SETMASK (2)`. Ignored if `set == NULL`. |
| `set` | `const sigset_t *` | in | Pointer to new mask bits. `NULL` means "do not change mask". |
| `oset` | `sigset_t *` | out | Pointer to receive the previous mask. `NULL` means "do not return". |
| `sigsetsize` | `size_t` | in | Size of `sigset_t` in bytes — MUST equal `sizeof(sigset_t)` for the kernel's native word width. On LP64, this is `8`. |

### return value

| Value | Meaning |
|---|---|
| `0`  | Success. Mask updated; `oset` (if non-NULL) populated. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EINVAL` | `set != NULL` and `how` is not `SIG_BLOCK`, `SIG_UNBLOCK`, or `SIG_SETMASK`. |
| `EINVAL` | `sigsetsize != sizeof(sigset_t)` for the kernel's native pointer width. |
| `EFAULT` | `set` non-NULL but not readable, or `oset` non-NULL but not writable. |

### abi surface

```text
__NR_rt_sigprocmask (x86_64)    = 14
__NR_rt_sigprocmask (i386)      = 175
__NR_rt_sigprocmask (generic)   = 135   /* arm64/riscv/loongarch */
__NR_rt_sigprocmask (powerpc)   = 174
__NR_rt_sigprocmask (s390x)     = 175
__NR_rt_sigprocmask (sparc)     = 103
__NR_rt_sigprocmask (mips O32)  = 4195
__NR_rt_sigprocmask (mips N64)  = 5014
```

### `how` constants (`uapi/asm-generic/signal-defs.h`)

```text
SIG_BLOCK   = 0     /* OR *set into current mask          */
SIG_UNBLOCK = 1     /* AND NOT *set from current mask      */
SIG_SETMASK = 2     /* replace current mask with *set      */
```

### `sigset_t` layout (`uapi/asm-generic/signal.h:21`)

```c
#define _NSIG       64
#define _NSIG_BPW   (__BITS_PER_LONG)        /* 64 on LP64 */
#define _NSIG_WORDS (_NSIG / _NSIG_BPW)      /* 1 word on LP64, 2 on ILP32 */

typedef struct {
    unsigned long sig[_NSIG_WORDS];
} sigset_t;
```

Bit `n - 1` of the bitmap represents signal number `n` (1-based). Thus
`sigset_t.sig[0]` bit 0 = `SIGHUP (1)`, bit 8 = `SIGKILL (9)`, bit 18 =
`SIGSTOP (19)`, bit 31 = `SIGRTMIN (32)`, bit 63 = `SIGRTMAX (64)`.

### compatibility contract

REQ-1: Syscall number is **14** on x86_64; **135** on generic-syscall
arches. ABI-stable forever.

REQ-2: `sigsetsize` MUST equal the kernel's native `sizeof(sigset_t)`:
8 bytes on LP64. Mismatch MUST return `-EINVAL` BEFORE any user-memory
access.

REQ-3: `how` is validated only when `set != NULL`. If `set == NULL`, the
kernel only reads `oset` and ignores `how` — it MUST NOT return
`-EINVAL` for an invalid `how` in that case.

REQ-4: Kernel sanitizes input: after copying `*set` from user, it
unconditionally clears `SIGKILL` and `SIGSTOP` bits before applying. The
resulting effective mask is what is installed AND what is returned via
`oset` on the next call.

REQ-5: `SIG_BLOCK`: `task.blocked |= *set` (with KILL/STOP cleared).

REQ-6: `SIG_UNBLOCK`: `task.blocked &= ~(*set)`.

REQ-7: `SIG_SETMASK`: `task.blocked = *set` (with KILL/STOP cleared).

REQ-8: After installing the new mask, the kernel calls
`recalc_sigpending()` for the current task. If any previously-blocked
signal is now unblocked AND is pending (per-thread queue or
per-process queue), the task's `TIF_SIGPENDING` flag is set so that the
next return-to-userspace path delivers it.

REQ-9: Atomicity: the read of the old mask, the install of the new
mask, and the `recalc_sigpending` call are performed under
`task.sighand.siglock` with IRQs disabled. Concurrent signal-delivery
on another CPU sees either the pre-call or post-call mask consistently
— never a torn state.

REQ-10: `oset` is populated with the PRE-call mask, not the post-call
mask. The semantic is "swap" — if userspace later calls
`rt_sigprocmask(SIG_SETMASK, oset_copy, NULL, 8)` it restores the prior
mask exactly.

REQ-11: Multithreaded process: the mask is per-thread. Calling
`rt_sigprocmask` in thread A does NOT affect thread B. (Compare to
`rt_sigaction` which is per-process via shared sighand.)

REQ-12: Realtime signals (`SIGRTMIN..SIGRTMAX`, 32..64) participate
fully — they can be blocked / unblocked. The pending-queue policy
differs (queued instances retained across multi-block / unblock cycle).

REQ-13: Audit: `AUDIT_SYSCALL` record on every `rt_sigprocmask`
includes `how` and a snapshot of the new effective mask (post-sanitize).

REQ-14: Compat (32-bit userspace on 64-bit kernel): handled by
`compat_rt_sigprocmask` with `sizeof(compat_sigset_t) == 8`. Logically
identical; layout-equivalent on LP64 due to two-word ILP32 sigset_t.

REQ-15: `current.real_blocked` (used for `rt_sigsuspend`,
`rt_sigtimedwait`) is NOT touched by this syscall — only `task.blocked`
is updated.

REQ-16: When transitioning a signal from blocked-to-unblocked AND that
signal has `SIG_IGN` disposition, any queued instances are discarded
(consistent with `rt_sigaction` flush-on-ignore semantics).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `sigsetsize_strict` | INVARIANT | sigsetsize ≠ 8 ⟹ -EINVAL. |
| `how_validated_only_if_set` | INVARIANT | set == NULL ⟹ how never read. |
| `kill_stop_unblockable` | INVARIANT | Post-call: blocked AND {SIGKILL, SIGSTOP} == 0. |
| `swap_atomic` | INVARIANT | oset reflects pre-call mask; new mask is post-sanitize. |
| `recalc_sigpending_called` | INVARIANT | TIF_SIGPENDING set iff any pending signal unblocked. |
| `siglock_held_during_update` | INVARIANT | All mask reads/writes under siglock + IRQs off. |

### Layer 2: TLA+

`kernel/rt_sigprocmask.tla`:
- States: VALIDATE → LOCK → READ_OLD → APPLY → RECALC → UNLOCK → COPY_OUT.
- Concurrent: signal-delivery via `send_signal` on another CPU.
- Properties:
  - `safety_no_torn_mask` — delivery side reads either pre- or post-mask, never torn.
  - `safety_kill_stop_never_blocked` — SIGKILL/SIGSTOP bits stay 0 in `blocked`.
  - `liveness_pending_eventually_delivered` — unblocking a pending signal ⟹ delivery at next user-return.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_rt_sigprocmask` post: mask updated atomically; oset has prior value | `sys_rt_sigprocmask` |
| `Sigset::from_user` / `to_user` byte-layout 8 bytes | `Sigset::from_user`, `to_user` |
| Sanitization preserves all non-KILL/STOP bits | `do_sigprocmask` |

### Layer 4: Verus / Creusot functional

POSIX `sigprocmask(2)` / `pthread_sigmask(3)` semantic equivalence.
LTP `rt_sigprocmask01..03`, `kselftests/signal/sigprocmask*` pass.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`rt_sigprocmask(2)` reinforcement:

- **Per-SIGKILL/SIGSTOP sanitize** — defense against per-mask-hide-kill.
- **Per-sigsetsize strict** — defense against per-mask-truncation / overrun.
- **Per-siglock IRQ-off update** — defense against per-torn mask read in delivery.
- **Per-recalc_sigpending post-update** — defense against per-stuck-pending after unblock.
- **Per-thread mask scope** — defense against cross-thread mask coupling.
- **Per-audit record on every change** — defense against per-mask-elision.

### grsecurity / pax-style reinforcement

- **PAX_RANDKSTACK at syscall entry** — each `rt_sigprocmask` call
  randomizes the kernel stack offset; combined with the brief siglock
  window this defeats per-syscall stack-layout reuse for ROP probing
  against the signal subsystem.
- **PaX UDEREF on `sigset_t`** — `set` and `oset` are accessed via
  UDEREF-toggled / SMAP+PAN-toggled `get_user`/`put_user`; a stray
  kernel pointer dereference cannot silently access user space, so a
  kernel bug that reads a wrong `sigset_t` pointer faults instead of
  exfiltrating data.
- **PaX MEMORY_SANITIZE** — the temporary on-stack `Sigset` is zeroed
  before copy-in (no leak of prior stack-resident sigset_t bytes via
  copy-out into `oset`).
- **GRKERNSEC_SIGNALS strict-clone** — per-thread `blocked` is treated
  as a security-relevant field: ptrace-modifying another task's mask
  via `PTRACE_PEEK/POKEUSR` is gated by yama+grsec and respects
  cross-cred denial. RBAC may forbid unblocking specific signals
  (e.g. SIGSEGV) for non-debugger subjects.
- **GRKERNSEC_PROC restrictions** — `/proc/<pid>/status` `SigBlk` /
  `SigCgt` fields honor per-uid hide-filter so an unprivileged user
  cannot enumerate other users' signal masks.
- **GRKERNSEC_BRUTE on signal-flood probing** — repeated
  `rt_sigprocmask(SIG_BLOCK, &full, ...)` immediately followed by
  fault-trigger patterns (classic crash-hardening bypass attempts) are
  counted toward the per-uid brute-force counter; sustained activity
  freezes the uid per the grsec policy.
- **GRKERNSEC_HARDEN_PTRACE** — tracer cannot use
  `PTRACE_GETSIGMASK` / `PTRACE_SETSIGMASK` across credential
  boundaries; cross-cred ptrace fails before this syscall is reached.
- **PaX KERNEXEC** — the `do_sigprocmask` code path resides in
  read-only-after-init kernel text; a transient kernel-write primitive
  cannot patch the SIG_BLOCK / SIG_UNBLOCK / SIG_SETMASK dispatch.
- **Per-grsec `gradm` learning** — subject policy may restrict the
  signals a process is allowed to BLOCK (preventing a hostile subject
  from masking SIGTERM/SIGKILL-equivalents in an exception-handler
  evasion attempt; SIGKILL is already unblockable by REQ-4 but RBAC
  generalizes the protection).

