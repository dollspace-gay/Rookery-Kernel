---
title: "Tier-5 syscall: epoll_pwait(2) — syscall 281"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`epoll_pwait(2)` extends `epoll_wait(2)` with an atomic temporary signal mask, modeled after `pselect(2)` / `ppoll(2)`. The kernel atomically swaps the calling thread's signal mask to `*sigmask` for the duration of the wait, restoring the original on return. This closes the canonical "signal-arrives-between-sigprocmask-and-epoll_wait" race that plagued `epoll_wait + sigprocmask` userspace patterns. The `sigsetsize` argument is `sizeof(sigset_t)` — a guard against userspace shipping a mis-sized sigset to a kernel built with a different `_NSIG`. Modern userspace SHOULD use `epoll_pwait2(2)` for nanosecond-precision timeouts; `epoll_pwait` keeps the millisecond `timeout` of legacy `epoll_wait`.

Critical for: signal-aware event loops (SIGCHLD-watching supervisors, SIGTERM-graceful-shutdown daemons), libev `EV_SIGNAL` integration, runtime schedulers needing race-free `EINTR` handling, terminal multiplexers (tmux, screen) on SIGWINCH, every reactor that needs "wake on event OR signal, atomically".

### Acceptance Criteria

- [ ] AC-1: `epoll_pwait(epfd, ev, 1, -1, NULL, 8)` behaves identically to `epoll_wait(epfd, ev, 1, -1)`.
- [ ] AC-2: `epoll_pwait(..., sigmask, 8)` with `sigmask` blocking SIGUSR1 does not return EINTR on SIGUSR1 delivery.
- [ ] AC-3: `epoll_pwait(..., sigmask, 8)` with `sigmask` unblocking SIGUSR1 DOES return EINTR on SIGUSR1.
- [ ] AC-4: After return, caller's signal mask is restored to pre-call value.
- [ ] AC-5: `sigsetsize = 7` returns EINVAL.
- [ ] AC-6: `sigsetsize = 0, sigmask = NULL` returns EINVAL.
- [ ] AC-7: `sigmask = bad-pointer, sigsetsize = 8` returns EFAULT, mask NOT changed.
- [ ] AC-8: `events = bad-pointer` returns EFAULT, mask NOT changed.
- [ ] AC-9: SIGKILL pending interrupts wait regardless of `sigmask` blocking SIGKILL.
- [ ] AC-10: Race-free SIGCHLD reception during wait — signal arriving in the `sigmask`'s unblocked window is observed.
- [ ] AC-11: Atomic mask install: no observable window in which both the temporary mask is partially installed and a signal is delivered.
- [ ] AC-12: `maxevents = 0` returns EINVAL.

### Architecture

```rust
#[syscall(nr = 281, abi = "sysv")]
pub fn sys_epoll_pwait(
    epfd: i32,
    events: UserPtr<EpollEvent>,
    maxevents: i32,
    timeout: i32,
    sigmask: UserPtr<SigSet>,
    sigsetsize: usize,
) -> KResult<i32> {
    EpollPwait::do_epoll_pwait(epfd, events, maxevents, timeout, sigmask, sigsetsize)
}
```

`EpollPwait::do_epoll_pwait(epfd, events, maxevents, timeout, sigmask_uptr, sigsetsize) -> KResult<i32>`:
1. /* Validate sigsetsize */
2. if sigsetsize != core::mem::size_of::<SigSet>() { return Err(EINVAL); }
3. /* Validate maxevents */
4. if maxevents <= 0 || maxevents > EP_MAX_EVENTS { return Err(EINVAL); }
5. /* UDEREF on events */
6. let nbytes = (maxevents as usize) * core::mem::size_of::<EpollEvent>();
7. access_ok_uderef(events, nbytes)?;
8. /* If sigmask provided, copy + UDEREF */
9. let mask: Option<SigSet> = if !sigmask_uptr.is_null() {
   - access_ok_uderef(sigmask_uptr, sigsetsize)?;
   - let mut m: SigSet = SigSet::zeroed();
   - copy_from_user(&mut m, sigmask_uptr, sigsetsize)?;
   - m.clear(SIGKILL); m.clear(SIGSTOP);
   - Some(m)
10. } else { None };
11. /* Atomic install + wait + restore */
12. let saved = mask.map(|m| set_user_sigmask(&m));
13. let result = EpollWait::do_wait_internal(epfd, events, maxevents, timeout);
14. /* Restore mask on all paths; restore_saved_sigmask handles ERESTARTSYS */
15. if saved.is_some() {
    - if matches!(result, Err(ERESTARTSYS) | Err(EINTR)) {
      - /* Mark for restart-restore */
      - set_restore_sigmask();
    - } else {
      - restore_user_sigmask(saved.unwrap());
    - }
16. }
17. result

`set_user_sigmask(new_mask: &SigSet) -> SigSet`:
1. let _lock = current().sighand.siglock.lock_irq();
2. let saved = current().blocked.clone();
3. current().blocked = *new_mask;
4. /* Re-evaluate pending */
5. recalc_sigpending();
6. saved

`restore_user_sigmask(saved: SigSet)`:
1. let _lock = current().sighand.siglock.lock_irq();
2. current().blocked = saved;
3. recalc_sigpending();

### Out of Scope

- `epoll_wait(2)` (Tier-5 separate doc — variant without sigmask).
- `epoll_pwait2(2)` (Tier-5 separate doc — nanosecond timespec timeout).
- `epoll_ctl(2)` (Tier-5 separate doc — watch management).
- `epoll_create(2)` / `epoll_create1(2)` (Tier-5 separate docs).
- `pselect(2)` / `ppoll(2)` (Tier-5 separate docs — sibling-pattern syscalls).
- Signal-delivery internals (Tier-3 in `kernel/signal.md`).
- Implementation code.

### signature

```c
int epoll_pwait(int epfd, struct epoll_event *events, int maxevents,
                int timeout, const sigset_t *sigmask, size_t sigsetsize);
```

### parameters

| Param | Type | Direction | Semantics |
|---|---|---|---|
| `epfd` | `int` | in | epoll file descriptor. |
| `events` | `struct epoll_event __user *` | out | Output buffer for ready events. |
| `maxevents` | `int` | in | Max events to return; MUST be > 0. |
| `timeout` | `int` | in | Milliseconds: `-1` = block, `0` = poll, `>0` = bounded. |
| `sigmask` | `const sigset_t __user *` | in | Signal mask to install atomically; `NULL` to skip. |
| `sigsetsize` | `size_t` | in | `sizeof(sigset_t)`; MUST equal `_NSIG / 8`. |

### return value

| Value | Meaning |
|---|---|
| ≥ 0 | Number of events written. `0` ⇒ timeout. |
| `-1` | Error; `errno` set. |

### errors

| `errno` | Cause |
|---|---|
| `EBADF` | `epfd` is not a valid open fd. |
| `EINVAL` | `epfd` is not an epoll fd; `maxevents <= 0` or `> EP_MAX_EVENTS`; `sigsetsize != sizeof(sigset_t)`. |
| `EFAULT` | `events` or `sigmask` (if non-NULL) is not in caller's accessible address space. |
| `EINTR` | Signal interrupted the wait before any event arrived. |
| `ENOMEM` | Kernel could not allocate temporary buffer. |

### abi surface

```text
__NR_epoll_pwait  (x86_64) = 281
__NR_epoll_pwait  (i386)   = 319
__NR_epoll_pwait  (arm64)  = 22
__NR_epoll_pwait  (generic)= 22

/* Sigset size is arch-dependent: 8 bytes on x86_64/arm64, 16 on i386. */
sizeof(sigset_t) == _NSIG / 8

/* epoll_event identical to epoll_wait. */

/* Sigmask semantics: */
/* On entry: saved_mask = current.blocked; current.blocked = *sigmask. */
/* On return (whether success, EINTR, EFAULT): current.blocked = saved_mask. */
/* If returning EINTR with SA_RESTART pending, the kernel sets restore_saved_sigmask */
/* so the restart-syscall path also restores the original mask. */
```

### compatibility contract

REQ-1: Syscall number is **281** on x86_64; **319** on i386; **22** on arm64/generic.

REQ-2: `maxevents` MUST satisfy `0 < maxevents ≤ EP_MAX_EVENTS`; otherwise `EINVAL`.

REQ-3: `events` MUST be a writable user buffer of `maxevents * sizeof(struct epoll_event)` bytes. UDEREF pre-validated. `EFAULT` on bad pointer.

REQ-4: `timeout` semantics identical to `epoll_wait(2)`: `-1` block, `0` poll, `>0` ms-bounded.

REQ-5: `sigmask == NULL` ⇒ behave identically to `epoll_wait`; no sigmask swap occurs. `sigsetsize` still validated.

REQ-6: `sigmask != NULL` ⇒ MUST point to readable user memory of `sigsetsize` bytes. UDEREF check. `EFAULT` on bad pointer.

REQ-7: `sigsetsize != sizeof(sigset_t)` ⇒ `EINVAL`. This catches userspace built against the wrong kernel `_NSIG`.

REQ-8: Atomic mask swap: kernel installs `*sigmask` as current's blocked-set BEFORE checking for pending events, AND restores the original blocked-set BEFORE returning. The swap is atomic with respect to signal delivery: a signal that becomes unblocked by `*sigmask` and is pending WILL fire during the wait (and may interrupt with `EINTR`).

REQ-9: SIGKILL and SIGSTOP are excluded from `*sigmask` (cannot be blocked); they always interrupt.

REQ-10: On `EINTR`: kernel sets `restore_saved_sigmask` so that if the signal handler returns and the syscall is restarted, the mask is correctly restored. With `SA_RESTART`, restart with the ORIGINAL mask (not the temporary).

REQ-11: PaX UDEREF on `events` (writable) AND `sigmask` (readable, if non-NULL).

REQ-12: Pre-validate `sigsetsize` BEFORE copy_from_user(sigmask) to avoid reading beyond user-buffer.

REQ-13: Per-epfd state unchanged by mask install: only the calling thread's signal mask is affected.

REQ-14: Concurrent `epoll_pwait` from multiple threads on the same epfd: each thread's sigmask install is per-thread (thread-local `current.blocked`), independent.

REQ-15: LSM hook: same as `epoll_wait` (no separate hook).

REQ-16: Within a PID namespace, signal delivery still goes through the per-task `signal_struct` queue; pidns does not change `epoll_pwait` semantics.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `sigsetsize_exact` | INVARIANT | per-pwait: sigsetsize == sizeof(sigset_t). |
| `mask_swap_atomic` | INVARIANT | per-set_user_sigmask: under siglock. |
| `mask_restored_all_paths` | INVARIANT | per-return: original mask restored (incl. EFAULT, EINTR). |
| `uderef_pre_copy` | INVARIANT | per-sigmask: UDEREF before copy_from_user. |
| `sigkill_excluded` | INVARIANT | per-mask: SIGKILL/SIGSTOP cleared after copy. |
| `restart_preserves_mask` | INVARIANT | per-ERESTARTSYS: restore_saved_sigmask set. |

### Layer 2: TLA+

`fs/epoll-pwait.tla`:
- States: per-thread blocked-set, per-task pending-set, per-epfd rdllist.
- Properties:
  - `safety_no_signal_lost` — per-pending-in-temp-unblocked: signal delivered or pending preserved.
  - `safety_mask_restored` — per-return: blocked-set == pre-call value.
  - `safety_atomic_install` — no window where partial mask permits unintended delivery.
  - `liveness_eventual_return` — per-pwait: returns event, timeout, or EINTR.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_epoll_pwait` post: mask restored on all paths | `EpollPwait::do_epoll_pwait` |
| `set_user_sigmask` post: returned old-mask matches caller's prior | `set_user_sigmask` |
| `restore_user_sigmask` post: current.blocked == saved | `restore_user_sigmask` |

### Layer 4: Verus / Creusot functional

Per-`epoll_pwait(2)` man-page equivalence. LTP `epoll_pwait01..02` pass. POSIX `pselect`/`ppoll` analogous semantics for the atomic-sigmask property.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`epoll_pwait(2)` reinforcement:

- **Per-sigsetsize exact match** — defense against per-undersized-sigset info-leak.
- **Per-atomic mask install** — defense against per-race signal-between-mask-and-wait.
- **Per-restore on all return paths** — defense against per-mask-leak across syscall.
- **Per-SIGKILL/SIGSTOP exclusion** — defense against per-uninterruptible-signal hijack.
- **Per-UDEREF on both userspace pointers** — defense against per-kernel-pointer info-leak.

### grsecurity / pax-style reinforcement

- **PaX UDEREF on `sigset_t *sigmask`** — every copy_from_user(sigmask) MUST traverse UDEREF SMAP/SMEP gate; rejects kernel-half pointers before mask install. UDEREF is checked BEFORE the mask is copied so that an EFAULT does NOT modify the caller's signal mask.
- **PaX UDEREF on `struct epoll_event *events`** — output buffer pre-validated; rollback on copy fault preserves rdllist atomicity.
- **PaX UDEREF on `siginfo_t`-adjacent** — not directly used here; signal-delivery infrastructure inherits UDEREF gating.
- **GRKERNSEC_HARDEN_PTRACE process-isolation** — peer ptracers cannot observe the temporary signal mask of another thread; mask is per-thread and not exposed via /proc.
- **PIDFD_SELF safe self-target** — watching `pidfd_open(getpid(), 0)` for self-exit is supported; combined with epoll_pwait, allows race-free "wait for self-exit OR signal" patterns.
- **GRKERNSEC_PROC restrictions** — `/proc/<pid>/status:SigBlk` is filtered for non-owners; during an `epoll_pwait` call the temporary mask is visible only to the calling thread.
- **signalfd CLOEXEC mandatory** — when `epoll_pwait` is used with a watched signalfd, the signalfd CLOEXEC default is enforced by grsec.
- **EPOLL fd-lifetime refcount strict** — epfd refcount verified before each wait; concurrent close races caught.
- **GRKERNSEC_AUDIT_GROUP** — repeated `epoll_pwait(sigmask = block-all)` from a marked group is audited (potential signal-denial attack).
- **No_new_privs neutral** — NNP does not change epoll_pwait semantics.
- **Anti-fingerprint hardening** — `sigmask` bits do not leak to `events.data`; user-supplied data is opaque.
- **Per-sigmask CAP_KILL gating** — masking signals from one's own process is unprivileged; grsec adds NO additional capability requirement to prevent legitimate `pselect`-equivalent patterns. However, attempts to mask SIGKILL/SIGSTOP are silently dropped (cannot be blocked).
- **Atomic recalc_sigpending under siglock** — grsec asserts siglock held throughout mask swap to prevent torn-mask UAF / inconsistent pending-set.
- **Seccomp interaction** — seccomp filters may block syscall 281 in favor of 441 (`epoll_pwait2`); default-deny hardening accepts both since both are race-free.

