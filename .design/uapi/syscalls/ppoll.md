# Tier-5 syscall: ppoll(2) — syscall 271

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/select.c (SYSCALL_DEFINE5(ppoll), do_sys_poll, poll_select_set_timeout, sigprocmask)
  - kernel/signal.c (restore_saved_sigmask, set_user_sigmask)
  - include/linux/poll.h
  - include/uapi/asm-generic/poll.h (POLL*)
  - include/uapi/linux/time.h (struct timespec)
  - arch/x86/entry/syscalls/syscall_64.tbl (271  64  ppoll)
-->

## Summary

`ppoll(2)` is the signal-mask-aware variant of `poll(2)`. It performs the same fd readiness wait, but additionally:

- Takes `tmo_p` as a `struct timespec` (nanosecond granularity) instead of milliseconds.
- Atomically swaps the calling thread's signal mask to `*sigmask` for the duration of the wait, restoring it on return.

The atomic sigmask install closes the classic "self-pipe / pselect race" — there is no window between unmasking SIGCHLD and entering the wait during which a signal could be delivered and silently lost. Critical for: `dbus`, `systemd`, `chrony`, `pulseaudio`, `qemu-block` IO threads, and any signal-driven event loop that must coalesce signal-fd and fd-readiness in a race-free way.

## Signature

```c
int ppoll(struct pollfd *fds, nfds_t nfds,
          const struct timespec *tmo_p,
          const sigset_t *sigmask,
          size_t sigsetsize);
```

```c
struct timespec {
    time_t tv_sec;   /* seconds */
    long   tv_nsec;  /* nanoseconds (0..999_999_999) */
};
```

Note: glibc's public `ppoll` wrapper has a 4-argument signature (no `sigsetsize`); the kernel ABI exposes the 5th argument so libc can pass `sizeof(sigset_t) == _NSIG/8`.

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `fds` | `struct pollfd *` | in/out | Per-fd events / revents (see poll(2)). |
| `nfds` | `nfds_t` | in | Length of `fds`; bounded by `RLIMIT_NOFILE`. |
| `tmo_p` | `const struct timespec *` | in | Relative timeout; `NULL` → infinite wait. |
| `sigmask` | `const sigset_t *` | in | Atomic mask for the duration of the wait; `NULL` → unchanged. |
| `sigsetsize` | `size_t` | in | `sizeof(*sigmask)`; must equal `_NSIG/8` (= 8 on x86_64). |

## Return value

| Value | Meaning |
|---|---|
| `>0` | Count of fds with non-zero revents. |
| `0` | Timeout expired. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EFAULT` | Any of `fds`, `tmo_p`, `sigmask` faults. |
| `EINTR` | Signal delivered (after mask swap) before any fd readied. |
| `EINVAL` | `nfds > RLIMIT_NOFILE`, `tmo_p->tv_nsec` out of range, `sigsetsize != _NSIG/8`. |
| `ENOMEM` | Per-fd poll table allocation failed. |

## ABI surface

```text
__NR_ppoll  (x86_64)  = 271
__NR_ppoll  (arm64)   = 73
__NR_ppoll  (riscv)   = 73
__NR_ppoll  (i386)    = 309   (32-bit timespec; ppoll_time64 = 414)

/* sigsetsize: x86_64 = 8 (kernel sigset_t is 64-bit). 32-bit ABIs pass 16 for
   compat-sigset_t handling via the COMPAT shim. */
```

## Compatibility contract

REQ-1: Syscall number is **271** on x86_64. ABI-stable since 2.6.16.

REQ-2: `sigsetsize` must equal `_NSIG/8` (8 bytes on 64-bit; 16 on compat 32-bit), else `-EINVAL`. This catches userspace using a future-incompatible sigset layout.

REQ-3: `tmo_p == NULL`: wait indefinitely (interruptible only by signal).

REQ-4: `tmo_p->tv_sec < 0` or `tv_nsec >= 1_000_000_000` or `tv_nsec < 0`: `-EINVAL`.

REQ-5: Mask install sequence (atomic w.r.t. signal delivery):
- `set_user_sigmask(sigmask, sigsetsize)`: copies in, saves old mask in `current->saved_sigmask`, replaces `current->blocked`, sets `TIF_RESTORE_SIGMASK`.
- The thread then enters `do_sys_poll`.
- On any return path: if signal was actually pending, `restore_saved_sigmask` runs in the syscall-exit `do_signal()` slowpath, restoring `current->blocked` before signal dispatch.

REQ-6: If no signal was delivered (return > 0 or 0), the old mask is restored unconditionally; `TIF_RESTORE_SIGMASK` is cleared in `__sys_ppoll` exit.

REQ-7: Per-`tmo_p`: the kernel converts to monotonic absolute deadline at entry; subsequent loop iterations subtract elapsed time from this deadline (no `tmo_p` mutation visible to user — it is `const`).

REQ-8: Per-glibc: `tmo_p` is treated as a relative timeout (matching POSIX `ppoll(2)` semantics). Some BSD-derived libc variants treat it as absolute; Linux ABI is strictly relative.

REQ-9: Per-signal: the mask must not block `SIGKILL`/`SIGSTOP` — the kernel silently removes those bits before applying (`sigdelsetmask(..., sigmask(SIGKILL) | sigmask(SIGSTOP))`).

REQ-10: Compat ABI: 32-bit callers use `__NR_ppoll_time32` (= 309 on i386) with `struct old_timespec32`; the COMPAT shim normalizes to `timespec64` and calls the same `do_sys_poll`.

REQ-11: `pollfd` semantics, POLL* mask handling, POLLNVAL on closed fd, EINTR-not-SA_RESTART, all inherited from poll(2) (see Tier-5 `poll.md`).

REQ-12: Per-`current->saved_sigmask` race: kernel must ensure that, between mask swap and the first reachable `schedule()`, no signal is silently dropped. Linux uses `set_current_blocked` which holds `tasklist_lock` for the publication; PaX-spinlock invariants apply.

REQ-13: Per-restart: ppoll(2) is never restarted by SA_RESTART; `ERESTART*` is converted to `EINTR` in syscall exit.

REQ-14: ppoll(2) does not consume entropy and does not require any capability.

## Acceptance Criteria

- [ ] AC-1: ppoll with NULL tmo and NULL sigmask behaves like poll(-1).
- [ ] AC-2: ppoll with tmo {0,500_000_000} sleeps ~500 ms and returns 0.
- [ ] AC-3: ppoll with sigmask={SIGINT} and SIGINT pending before entry: blocks; SIGINT held until exit.
- [ ] AC-4: ppoll with sigmask={} (empty) and SIGINT pending: returns -1, EINTR; old mask restored.
- [ ] AC-5: sigsetsize=4 (wrong) returns -1, EINVAL.
- [ ] AC-6: tmo.tv_nsec = 1_000_000_000 returns -1, EINVAL.
- [ ] AC-7: nfds > RLIMIT_NOFILE returns -1, EINVAL.
- [ ] AC-8: Signal mask includes SIGKILL bit: bit silently removed; SIGKILL still terminates.
- [ ] AC-9: tmo_p memory is unmodified on return (verified via memcmp).
- [ ] AC-10: After EINTR exit, current->blocked equals pre-call mask.
- [ ] AC-11: Atomic mask install: SIGCHLD blocked across ppoll cannot be lost when ppoll returns 0.

## Architecture

```rust
#[syscall(nr = 271, abi = "sysv")]
pub fn sys_ppoll(
    fds: UserPtr<PollFd>, nfds: u64,
    tmo: UserPtr<TimeSpec>, sigmask: UserPtr<SigSet>,
    sigsetsize: usize,
) -> isize {
    Ppoll::do_ppoll(fds, nfds, tmo, sigmask, sigsetsize)
}
```

`Ppoll::do_ppoll(fds, nfds, tmo, sigmask, sigsetsize) -> isize`:
1. let ts = if tmo.is_null() { None } else { Some(unsafe { tmo.copy_in()? }) }; // EFAULT
2. if let Some(ts) = &ts { TimeSpec::validate(ts)?; }                          // EINVAL
3. Signal::set_user_sigmask(sigmask, sigsetsize)?;                              // EFAULT/EINVAL
4. let deadline = ts.map(|t| KTime::monotonic_plus(t.to_ktime())).unwrap_or(KTime::MAX);
5. let ret = Poll::do_sys_poll(fds, nfds, deadline);
6. Signal::restore_user_sigmask(ret != Err(EINTR));   // false ⟹ defer to do_signal()
7. ret

`Signal::set_user_sigmask(uptr, size) -> Result<()>`:
1. if uptr.is_null() { return Ok(()); }
2. if size != size_of::<SigSet>() { return Err(EINVAL); }
3. let mut new = unsafe { uptr.copy_in::<SigSet>()? };
4. new.delset(SIGKILL); new.delset(SIGSTOP);
5. spin_lock_irq(&current().sighand.siglock);
6. current().saved_sigmask = current().blocked.clone();
7. set_current_blocked(&new);
8. set_thread_flag(TIF_RESTORE_SIGMASK);
9. spin_unlock_irq(&current().sighand.siglock);
10. Ok(())

`Signal::restore_user_sigmask(restore_now)`:
1. if !test_thread_flag(TIF_RESTORE_SIGMASK) { return; }
2. if restore_now {
3.     clear_thread_flag(TIF_RESTORE_SIGMASK);
4.     set_current_blocked(&current().saved_sigmask);
5. }
6. /* Else: signal-exit slowpath restores after running handler. */

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `sigsetsize_strict` | INVARIANT | size != sizeof(SigSet) ⟹ EINVAL pre-alloc. |
| `tv_nsec_validated` | INVARIANT | tv_nsec ∈ [0, 1e9) before kernel uses it. |
| `sigkill_unmaskable` | INVARIANT | new.has(SIGKILL) == false after install. |
| `mask_restored_or_deferred` | INVARIANT | every exit either restores blocked or sets TIF_RESTORE_SIGMASK. |
| `tmo_const_unchanged` | INVARIANT | user `tmo_p` memory not written. |

### Layer 2: TLA+

`fs/ppoll-syscall.tla`:
- States: per-copy-tmo, per-install-mask, per-poll, per-wait, per-wake, per-restore-mask.
- Properties:
  - `safety_atomic_install` — no signal delivered to thread between save_blocked and set_blocked.
  - `safety_no_lost_signal` — pending signal after mask swap ⟹ EINTR or queued for delivery.
  - `safety_kill_stop_unmaskable` — SIGKILL/SIGSTOP stripped from new mask.
  - `liveness_mask_restored` — every return path eventually restores blocked.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_ppoll` post: ret ∈ {-1,0,>0} matches poll outcome | `Ppoll::do_ppoll` |
| `set_user_sigmask` post: TIF_RESTORE_SIGMASK set ⟹ saved_sigmask valid | `Signal::set_user_sigmask` |
| `restore_user_sigmask` post: blocked == saved or TIF flagged | `Signal::restore_user_sigmask` |
| `TimeSpec::validate` post: tv_sec >= 0 and tv_nsec in range | `TimeSpec::validate` |

### Layer 4: Verus / Creusot functional

Per-ppoll(2) POSIX-2017 semantic equivalence; LTP `ppoll01..ppoll04` and glibc `tst-ppoll` pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`ppoll(2)` reinforcement:

- **Per-sigsetsize strict** — defense against per-future-sigset smuggle.
- **Per-atomic mask install (TIF_RESTORE_SIGMASK + siglock)** — defense against per-signal-race (classic pselect race).
- **Per-SIGKILL / SIGSTOP unmaskable** — defense against per-immortal-task escape.
- **Per-tv_nsec range** — defense against per-overflow in ktime arithmetic.
- **Per-deadline monotonic** — defense against per-CLOCK_REALTIME-jump skew.
- **Per-mask restored on EINTR even if signal handler reruns syscall** — defense against per-mask-leak across exits.

## Grsecurity / PaX surface

- **PaX UDEREF on fds, tmo_p, sigmask copy_from_user** — defense against per-userptr kernel-deref; SMAP forced. All three pointers are independently validated; partial-NULL is honored.
- **Sigmask atomic install/restore under siglock** — grsec audits TIF_RESTORE_SIGMASK transitions; any path that exits without restoring is reported under GRKERNSEC_AUDIT_KERN.
- **GRKERNSEC_RLIMIT_NOFILE strict** — same defense as poll(2) against per-nfds-amplification DoS.
- **PAX_USERCOPY_HARDEN** — sigset copy uses whitelisted slab; bounded by sigsetsize and validated against kernel `_NSIG/8`.
- **PAX_REFCOUNT on saved_sigmask publication** — defense against per-saved_sigmask UAF across signal-handler reentry.
- **GRKERNSEC_TIME monotonic only** — defense against per-CLOCK_REALTIME-jump-induced premature-timeout abuse.
- **SIGKILL/SIGSTOP unmaskable enforced before publication** — defense against per-immortal-task chroot escape.
- **GRKERNSEC_PROC sigmask visibility** — `/proc/<pid>/status SigBlk` line not exposed to non-CAP_SYS_PTRACE during ppoll wait.
- **PaX-spinlock invariants on set_current_blocked** — defense against per-preempt-race; spinlock-irq is hard-locked.
- **Per-signal-restart conversion to EINTR** — defense against per-SA_RESTART smuggling that would re-enter with stale mask.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- poll(2) base behavior (covered in Tier-5 `poll.md`).
- pselect6(2) (covered in Tier-5 `pselect6.md`).
- Per-driver f_op->poll implementations.
- Implementation code.
