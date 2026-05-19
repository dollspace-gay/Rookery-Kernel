# Tier-5 syscall: rt_sigsuspend(2) — syscall 130

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/signal.c (sys_rt_sigsuspend)
  - include/linux/signal.h (set_current_blocked, sigsuspend helper)
  - include/uapi/asm-generic/signal-defs.h (sigset_t)
  - arch/x86/entry/syscalls/syscall_64.tbl (130  common  rt_sigsuspend)
-->

## Summary

`rt_sigsuspend(2)` atomically replaces the caller's signal mask with
a caller-specified `mask` and suspends the caller until a signal NOT
in `mask` is delivered. On wakeup the original mask is restored
BEFORE the signal handler runs. This is the canonical race-free
"wait for a specific signal while atomically unblocking it"
primitive — it solves the test-then-pause window that `pause(2)`
followed by `sigprocmask(2)` cannot close.

Semantics:

- Replace `task.blocked` with `mask` (sanitized — `SIGKILL`/`SIGSTOP` cleared).
- `TASK_INTERRUPTIBLE`, schedule.
- On wakeup with a signal NOT in `mask` pending: run signal-delivery, restore old mask, return `-EINTR`.

`rt_sigsuspend` NEVER returns successfully — it ALWAYS returns
`-EINTR` after delivering an unblocked signal.

Critical for: pthread_sigmask race-free wait, `pselect(2)` /
`ppoll(2)` internals, `signalfd(2)`-vs-handler hybrid patterns,
init-style supervisors that must observe `SIGCHLD` / `SIGTERM`
without missing it.

This Tier-5 covers `kernel/signal.c::SYSCALL_DEFINE2(rt_sigsuspend, ...)`.

## Signature

```c
long rt_sigsuspend(const sigset_t *mask, size_t sigsetsize);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `mask` | `const sigset_t *` | in | Temporary signal mask during the suspend. Non-NULL. |
| `sigsetsize` | `size_t` | in | Must equal `sizeof(sigset_t)` (8 on LP64). |

## Return value

| Value | Meaning |
|---|---|
| `-1` + `errno = EINTR` | Normal exit: signal delivered (handler ran). |

`rt_sigsuspend` cannot return successfully.

## Errors

| errno | Trigger |
|---|---|
| `EINTR`  | Normal completion — a signal not in `mask` was delivered. |
| `EINVAL` | `sigsetsize != sizeof(sigset_t)`. |
| `EFAULT` | `mask` not accessible. |

## ABI surface

```text
__NR_rt_sigsuspend (x86_64)    = 130
__NR_rt_sigsuspend (i386)      = 179
__NR_rt_sigsuspend (generic)   = 133   /* arm64/riscv/loongarch */
__NR_rt_sigsuspend (powerpc)   = 178
__NR_rt_sigsuspend (s390x)     = 179
__NR_rt_sigsuspend (sparc)     = 107
```

### `sigset_t`

```c
typedef struct {
    unsigned long sig[_NSIG_WORDS];   /* 64-bit word on LP64, NSIG=64 */
} sigset_t;
```

`sizeof(sigset_t)` on LP64 = 8 bytes.

## Compatibility contract

REQ-1: Syscall **130** on x86_64; **133** on generic. ABI-stable.

REQ-2: `sigsetsize` MUST equal `sizeof(sigset_t)` (8 on LP64). Mismatch → `-EINVAL`.

REQ-3: `mask` copied via UDEREF-protected `copy_from_user`. `SIGKILL`/`SIGSTOP` bits silently cleared — they always wake the suspend.

REQ-4: Atomicity under `task.sighand.siglock`:
- Save `saved_blocked = task.blocked`.
- Save `task.saved_sigmask = saved_blocked` (per SUSv4: visible to handler via sigframe restore).
- Set `task.blocked = mask` (sanitized).
- Set `TASK_INTERRUPTIBLE`.
- `recalc_sigpending()` while holding siglock.

REQ-5: If a signal pending outside `mask` AT lock-check: skip schedule, proceed to delivery. Closes the race where the signal arrived between userspace's pre-mask check and syscall entry.

REQ-6: Else: drop siglock, `schedule()`. Wakeup by `signal_wake_up` on any signal NOT in `mask` (including fatal).

REQ-7: On wakeup: `set_restore_sigmask()` flags `TIF_RESTORE_SIGMASK` so the return-to-user path restores `saved_blocked` AFTER signal delivery (handler sees the SAVED mask). Return `-EINTR`.

REQ-8: Handler-mask interaction: effective blocked = `saved_blocked | sa_mask | {sig}` (unless `SA_NODEFER`). The suspend-mask is NOT visible to the handler — it was a transient kernel-only "sleep-mask".

REQ-9: After handler return: `TIF_RESTORE_SIGMASK` triggers `task.blocked = saved_blocked` on syscall-exit; flag cleared.

REQ-10: Fatal signal during suspend: process terminates; kernel runs `do_coredump`/`do_exit` directly; no userspace restore.

REQ-11: Nested suspend (from a handler): each `rt_sigsuspend` holds its own `saved_blocked` on its stack frame; outer save preserved.

REQ-12: Audit: `AUDIT_SYSCALL` captures effective mask and delivered signum.

REQ-13: Non-restartable: `restart_syscall(2)` is NEVER returned; `-EINTR` is the success case.

REQ-14: ptrace SIGSTOP wakes the suspend even if `SIGSTOP` is "in mask"; ptrace stop is unblockable.

REQ-15: Compat 32-bit: `compat_rt_sigsuspend` expands the 32-bit sigset to 64-bit kernel sigset.

REQ-16: POSIX cancellation point: pthread_cancel → `SIGCANCEL` (internal glibc) delivered; returns `-EINTR`.

## Acceptance Criteria

- [ ] AC-1: Syscall number 130 on x86_64; 133 on generic.
- [ ] AC-2: Mask = full set minus SIGUSR1; other thread sends SIGUSR1: handler runs, returns -EINTR, post-call mask == original.
- [ ] AC-3: Mask = full set minus SIGUSR1; other thread sends SIGUSR2 (blocked): handler does NOT run; continues sleeping.
- [ ] AC-4: SIGKILL in mask: silently ignored; SIGKILL still terminates.
- [ ] AC-5: sigsetsize != 8: -EINVAL.
- [ ] AC-6: mask = NULL: -EFAULT.
- [ ] AC-7: Pre-existing pending signal not in mask at call time: handler runs immediately; no sleep observed.
- [ ] AC-8: Nested call from within a handler: outer saved_blocked preserved.
- [ ] AC-9: Handler observes mask = (original | sa_mask | {sig}); NOT the suspend-mask.
- [ ] AC-10: Post-call task.blocked == pre-call task.blocked.
- [ ] AC-11: Fatal signal during suspend: process terminates; no userspace-restore observed.
- [ ] AC-12: pselect/ppoll built atop rt_sigsuspend race-free under signal storm.

## Architecture

```rust
#[syscall(nr = 130, abi = "sysv")]
pub fn sys_rt_sigsuspend(
    mask:       UserPtr<Sigset>,
    sigsetsize: usize,
) -> isize {
    if sigsetsize != size_of::<Sigset>() { return -EINVAL; }
    let mut new_blocked = Sigset::from_user(mask)?;
    new_blocked.del(SIGKILL); new_blocked.del(SIGSTOP);
    Signal::do_rt_sigsuspend(new_blocked)
}
```

`Signal::do_rt_sigsuspend`: under siglock save `saved_blocked` and copy into `task.saved_sigmask`; set `task.blocked = new_blocked`; `recalc_sigpending`; if `signal_pending_outside(task, &new_blocked)` skip schedule, else set `TASK_INTERRUPTIBLE` and `Sched::schedule()`. On wakeup, `Signal::set_restore_sigmask(task)`; return `-EINTR`. The return-to-user path observes `TIF_RESTORE_SIGMASK`, runs `signal_delivery`, then `task.blocked = task.saved_sigmask; clear TIF_RESTORE_SIGMASK`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `sigsetsize_strict` | INVARIANT | sigsetsize != 8 ⟹ -EINVAL. |
| `kill_stop_sanitized` | INVARIANT | Effective new_blocked excludes SIGKILL & SIGSTOP. |
| `atomic_mask_swap` | INVARIANT | siglock held during {save; set; pending-check}. |
| `saved_sigmask_set` | INVARIANT | task.saved_sigmask == pre-call task.blocked. |
| `tif_restore_sigmask_set` | INVARIANT | On every wakeup, TIF_RESTORE_SIGMASK set. |
| `return_always_eintr` | INVARIANT | No success return; result is -EINTR (or -EINVAL/-EFAULT pre-suspend). |

### Layer 2: TLA+

`kernel/rt_sigsuspend.tla`:
- States: VALIDATE → COPY → SANITIZE → LOCK → SAVE_OLD → SET_NEW → RECALC → (PENDING ? PROCEED : SLEEP → WAKEUP) → SET_RESTORE → RETURN_EINTR.
- Concurrent: signal arrival between SET_NEW and RECALC; arrival during SLEEP; sibling thread in suspend.
- Properties:
  - `safety_atomic_swap` — no state where mask = new but signal_pending check missed.
  - `safety_saved_mask_invariant` — task.saved_sigmask == pre-call blocked across the whole suspend.
  - `safety_post_handler_restore` — after handler runs, task.blocked == pre-call blocked.
  - `liveness_signal_wakes_suspend` — every signal not in mask eventually wakes the suspend.
  - `safety_no_missed_signal` — signal generated AT ANY TIME after lock-acquire is observed.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_rt_sigsuspend` post: returns -EINTR on every non-validation path | `sys_rt_sigsuspend` |
| `do_rt_sigsuspend` post: TIF_RESTORE_SIGMASK set; saved_sigmask = pre-call blocked | `do_rt_sigsuspend` |
| `signal_delivery_post`: task.blocked = saved_sigmask after handler return | return-to-user |
| `Sigset::from_user` byte-layout 8 bytes (LP64) | `Sigset::from_user` |

### Layer 4: Verus / Creusot functional

POSIX `sigsuspend(2)` / SUSv4 atomic-suspend semantic equivalence. LTP `rt_sigsuspend01..03`, glibc `signal/tst-sigsuspend*`.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

- **Per-sigsetsize strict** — defense against per-mask-truncation reading uninitialized bytes.
- **Per-SIGKILL/SIGSTOP sanitize** — defense against per-trick-into-unkillable suspend.
- **Per-atomic swap under siglock** — defense against per-test-then-pause race (the bug rt_sigsuspend exists to close).
- **Per-saved_sigmask preserved** — defense against per-leaked blocked-mask after EINTR.
- **Per-TIF_RESTORE_SIGMASK strict** — defense against per-handler observing the wrong mask.
- **Per-recalc_sigpending on lock release** — defense against per-missed-wakeup (signal generated mid-sleep).

## Grsecurity / PaX-style Reinforcement

- **PAX_RANDKSTACK at syscall entry** — randomizes kernel-stack layout per call; combined with per-call `saved_blocked` on stack, defeats per-stack-layout reuse during the LONG sleep.
- **PaX UDEREF on `sigset_t`** — `copy_from_user(mask)` runs with SMAP/PAN + UDEREF. 8-byte read bounds-checked.
- **PaX MEMORY_SANITIZE** — local `Sigset` zero-initialized before copy-from-user; high bits beyond NSIG cleared. No per-stack-leak via the mask channel.
- **GRKERNSEC_SIGNALS atomic-swap audit** — grsec audits (caller-uid, target-pid, mask, delivered-signum) on every rt_sigsuspend wakeup. Combined with `GRKERNSEC_BRUTE`, signal-flood + rt_sigsuspend race-probing trips per-uid brute-force counter.
- **GRKERNSEC_BRUTE on signal-flood** — sustained rt_sigsuspend + cross-uid probes rate-limited per-uid; may trigger lockout.
- **PaX KERNEXEC** — `do_rt_sigsuspend` in read-only-after-init kernel text; cannot be patched to skip `saved_sigmask` save or `TIF_RESTORE_SIGMASK` set.
- **GRKERNSEC_HARDEN_PTRACE** — tracer cannot inject signal into a suspended tracee across credential boundaries; cross-cred ptrace check fails before signal-gen.
- **GRKERNSEC_PROC restrictions** — `/proc/<pid>/wchan` reveals the task is sleeping in `do_rt_sigsuspend`; per-uid hide-filter conceals from non-owners.
- **PaX TASK_HARDENING + RANDSTRUCT** — `saved_sigmask` lives in `task_struct` with per-struct slab redzone and `__randomize_layout`; OOB writes from neighboring fields blocked; field offset randomized so memory-corruption primitives cannot deterministically target it.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `rt_sigaction(2)` / `rt_sigprocmask(2)` / `rt_sigtimedwait(2)` / `rt_sigpending(2)` / `rt_sigqueueinfo(2)` / `rt_tgsigqueueinfo(2)` (separate Tier-5 docs).
- `pselect6(2)` / `ppoll(2)` — built atop rt_sigsuspend semantics (separate Tier-5 docs).
- Signal-delivery internals (`get_signal`, `signal_delivery`, sigframe setup — Tier-3).
- Compat 32-bit variants (Tier-3).
- Implementation code.
