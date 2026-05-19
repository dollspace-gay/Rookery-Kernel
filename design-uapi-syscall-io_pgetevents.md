---
title: "Tier-5 syscall: io_pgetevents(2) — syscall 333"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`io_pgetevents(2)` is the signal-mask-aware cousin of `io_getevents(2)`. It performs the same per-context completion reap, but also atomically swaps the calling thread's signal mask for the duration of the wait — the same `pselect`/`ppoll` race-free pattern. The signal-mask is delivered via the wrapping struct `__aio_sigset { sigset_t *ss; size_t ss_len; }`, mirroring how `pselect6`/`ppoll` carry their `sigset_t*` past the limited 6-argument syscall ABI.

The primary value over `io_getevents(2)`: a thread that wants to block in AIO completion waits while leaving certain signals deliverable (or blocked) can do so without the classic `sigprocmask`/`io_getevents` race window. Critical for: AIO callers integrated with `signalfd`-style frameworks, async libraries that want SIGRTMIN-N forwarding, and selftest coverage of the pselect-style signal-safe wait pattern.

### Acceptance Criteria

- [ ] AC-1: `sig == NULL`: behaves exactly like `io_getevents`.
- [ ] AC-2: `sig` with `sigmask` blocking SIGUSR1: SIGUSR1 sent during wait does NOT interrupt; remains pending until mask restored.
- [ ] AC-3: `sig` with mask, SIGINT (not blocked) delivered: `-EINTR` returned.
- [ ] AC-4: `sig->sigsetsize != sizeof(sigset_t)` → `-EINVAL`.
- [ ] AC-5: `sig` user pointer faults → `-EFAULT`.
- [ ] AC-6: `min_nr < 0` or `nr < min_nr` → `-EINVAL`.
- [ ] AC-7: `timeout = {0,0}`, no completion ready: returns `0`.
- [ ] AC-8: Two concurrent `io_pgetevents` threads with different masks: each thread sees the mask it provided.
- [ ] AC-9: After syscall returns, `task->blocked` is the original mask, not the supplied one.
- [ ] AC-10: `io_destroy(ctx)` during wait → `-EAGAIN`.
- [ ] AC-11: Already-pending signal at entry: returns `-EINTR` even before sleep starts.

### Architecture

```rust
#[syscall(nr = 333, abi = "sysv")]
pub fn sys_io_pgetevents(
    ctx_id: AioContextT,
    min_nr: i64,
    nr: i64,
    events: UserPtrMut<IoEvent>,
    timeout: UserPtr<KernelTimespec>,
    sig: UserPtr<AioSigset>,
) -> isize {
    Aio::do_io_pgetevents(ctx_id, min_nr, nr, events, timeout, sig)
}
```

`Aio::do_io_pgetevents(...) -> isize`:
1. /* Validate bounds */
2. if min_nr < 0 || nr < min_nr { return -EINVAL; }
3. /* Pull sig struct */
4. let user_sigmask = match sig.is_null() {
   - true  => None,
   - false => {
     - let s = sig.copy_in::<AioSigset>()?;            // EFAULT
     - if s.sigsetsize != size_of::<SigSet>() { return -EINVAL; }
     - Some((s.sigmask, s.sigsetsize))
     },
   };
5. /* Install user mask */
6. let saved = match user_sigmask {
   - Some((p, len)) => Aio::set_user_sigmask(p, len)?,  // EFAULT / EINVAL
   - None => None,
   };
7. /* Reap */
8. let ret = Aio::do_io_getevents_inner(ctx_id, min_nr, nr, events, timeout);
9. /* Race-free restore */
10. Aio::restore_user_sigmask(&ret);                     // sets TIF_RESTORE_SIGMASK
11. ret

`Aio::set_user_sigmask(uptr, len) -> Result<Option<SigSet>>`:
1. let mut new_mask = SigSet::empty();
2. uptr.copy_in(&mut new_mask)?;
3. /* Mask SIGKILL/SIGSTOP off */
4. new_mask.remove(SIGKILL).remove(SIGSTOP);
5. let saved = current().blocked;
6. current().saved_sigmask = saved;
7. set_thread_flag(TIF_RESTORE_SIGMASK);
8. /* Apply */
9. current().blocked = new_mask;
10. recalc_sigpending();
11. /* If pending: signal_pending check post */
12. Ok(Some(saved))

`Aio::restore_user_sigmask(ret) -> ()`:
1. /* Race-free: only restore if no signal is being delivered */
2. if test_and_clear_thread_flag(TIF_RESTORE_SIGMASK) {
3.   if ret < 0 || ret as i64 == EINTR as i64 {
4.     /* Will be done by signal-return path */
5.     return;
6.   }
7.   current().blocked = current().saved_sigmask;
8.   recalc_sigpending();
9. }

### Out of Scope

- `io_setup(2)`, `io_submit(2)`, `io_destroy(2)` (covered in their own Tier-5 docs).
- `io_uring` (covered separately).
- `pselect6(2)`, `ppoll(2)` (covered separately under their own Tier-5 docs).
- y2038 `io_pgetevents_time64` variant on 32-bit ABIs (covered in y2038 Tier-5 cluster).
- Implementation code.

### signature

```c
int io_pgetevents(aio_context_t ctx_id,
                  long min_nr,
                  long nr,
                  struct io_event *events,
                  struct timespec *timeout,
                  const struct __aio_sigset *sig);
```

```c
struct __aio_sigset {
    const sigset_t __user *sigmask;
    size_t                 sigsetsize;
};
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `ctx_id` | `aio_context_t` | in | Per-`io_setup` handle; resolves to `kioctx`. |
| `min_nr` | `long` | in | Minimum events to wait for; `>= 0` and `<= nr`. |
| `nr` | `long` | in | Maximum events to copy out. |
| `events` | `struct io_event *` | out | Per-caller event buffer; `nr * sizeof(struct io_event)`. |
| `timeout` | `struct timespec *` | in (optional) | Per-call wait limit; `NULL` means infinite. |
| `sig` | `const struct __aio_sigset *` | in (optional) | Wrapping struct carrying user `sigmask`+`sigsetsize`. `NULL` makes the call signal-mask-transparent (identical to `io_getevents`). |

### return value

| Value | Meaning |
|---|---|
| `>= 0` | Number of events copied to `events[]`; may be less than `min_nr` on timeout. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EINVAL` | Bad `ctx_id`, `min_nr < 0`, `nr < min_nr`, or `sig->sigsetsize != sizeof(sigset_t)`. |
| `EFAULT` | `events`, `timeout`, `sig`, or `sig->sigmask` user pointer faults. |
| `EINTR` | Wait interrupted by signal not blocked by `sig->sigmask`. |
| `EAGAIN` | Context being destroyed concurrently. |
| `ENOSYS` | Built without `CONFIG_AIO`. |

### abi surface

```text
__NR_io_pgetevents (x86_64) = 333
__NR_io_pgetevents (arm64)  = 292
__NR_io_pgetevents (riscv)  = 292
__NR_io_pgetevents (i386)   = 385

/* __aio_sigset is two pointers/size_t; ABI-stable since 4.18. */
/* On 32-bit ABI, io_pgetevents_time64 (syscall 416 on i386) is the
   y2038-safe alias and accepts a __kernel_timespec. */
```

### compatibility contract

REQ-1: Syscall number is **333** on x86_64. ABI-stable since 4.18.

REQ-2: When `sig == NULL`, behavior is exactly `io_getevents`.

REQ-3: When `sig != NULL`, the kernel copies the `__aio_sigset` from user memory, then copies the pointed-to `sigset_t` (size must equal `sizeof(sigset_t)`, else `-EINVAL`).

REQ-4: The kernel installs the user-provided sigmask via `set_user_sigmask(sigmask, sigsetsize)` before sleeping, and restores the saved mask before returning (race-free vs signal delivery). If a signal is pending at the moment of `set_user_sigmask`, the wait returns `-EINTR` immediately.

REQ-5: `restore_saved_sigmask()` is called on every exit path, including error and timeout.

REQ-6: `min_nr`, `nr`, `events`, and `timeout` semantics are identical to `io_getevents(2)`.

REQ-7: Per-`exit_aio` race: getevents wakes with `-EAGAIN`.

REQ-8: Per-compat ABI: `compat_io_pgetevents` translates `compat___aio_sigset`, `compat_timespec`, and a compat `sigset_t` (no extension fields).

REQ-9: Per-y2038: 32-bit ABIs gained `io_pgetevents_time64` (syscall 416 on i386, 416 on arm32) with `__kernel_timespec`.

REQ-10: Per-`sigsetsize`: equal to `_NSIG / 8` (== 8 on Linux). Other values → `-EINVAL`.

REQ-11: Reaping copy and waitqueue parking are identical to `io_getevents` (`read_events_ring`, `aio_complete` wake).

REQ-12: Per-`task->saved_sigmask`: the original mask is saved in `current->saved_sigmask`, and `TIF_RESTORE_SIGMASK` is set so the syscall return path restores it on the way out (idiomatic pselect-style).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `sigsetsize_strict` | INVARIANT | sigsetsize != sizeof(sigset_t) ⟹ EINVAL. |
| `mask_restore_on_all_paths` | INVARIANT | every exit path restores saved_sigmask. |
| `pending_signal_no_oversleep` | INVARIANT | signal pending at set_user_sigmask ⟹ EINTR. |
| `sigkill_sigstop_unblockable` | INVARIANT | user mask cannot block SIGKILL/SIGSTOP. |
| `ctx_refcount_balanced` | INVARIANT | lookup_ioctx paired with put_ioctx. |
| `bounds_check_total` | INVARIANT | min_nr<0 / nr<min_nr ⟹ EINVAL. |

### Layer 2: TLA+

`fs/aio-pgetevents.tla`:
- States: per-call sigmask-install, copy-loop, wait-park, completion-wake, sigmask-restore.
- Properties:
  - `safety_sigmask_atomic` — install-then-sleep is race-free (no signal lost between mask-install and sleep).
  - `safety_sigmask_restore` — original mask restored on every exit.
  - `safety_no_blocked_sigkill` — SIGKILL never blocked by user mask.
  - `liveness_progress_under_completion_or_signal` — call terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_io_pgetevents` post: terminates with EINTR / count / EAGAIN / errno | `Aio::do_io_pgetevents` |
| `set_user_sigmask` post: TIF_RESTORE_SIGMASK set; SIGKILL/SIGSTOP cleared | `Aio::set_user_sigmask` |
| `restore_user_sigmask` post: blocked == saved_sigmask | `Aio::restore_user_sigmask` |

### Layer 4: Verus / Creusot functional

Per-`io_pgetevents(2)` man-page + glibc semantic equivalence. Selftests: `tools/testing/selftests/aio/` pselect-pattern subset pass.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`io_pgetevents(2)` reinforcement:

- **Per-sigset_t copy bounded** — defense against per-overlong copy via crafted `sigsetsize`.
- **Per-SIGKILL/SIGSTOP unblockable** — defense against per-user-mask uninterruptible-task.
- **Per-saved-sigmask restored unconditionally** — defense against per-mask-leak.
- **Per-context lookup refcounted** — defense against per-ctx UAF.
- **Per-`aio_max_nr` cap** — defense against per-system AIO exhaustion DoS.

### grsecurity / pax-style reinforcement

- **PaX UDEREF on `sig`, `sigmask`, `events`, `timeout` copy_from_user / copy_to_user** — defense against per-user-pointer kernel-deref bug; SMAP forced.
- **GRKERNSEC_HIDESYM on sched_clock-based wait timings** — defense against per-wait-jitter side-channel that could reveal scheduler/timer entropy via mask-install timing.
- **AIO RLIMIT_MEMLOCK + sysctl_aio_max_nr enforcement** — per-userns AIO context cap; defense against per-userns aio_ring pin-DoS.
- **PaX PAX_REFCOUNT on kioctx, sigmask state** — defense against per-refcount-overflow UAF in the pselect-style race window.
- **Per-`saved_sigmask` clobber audited** — grsec asserts no path can leave the syscall with `task->blocked` != `task->saved_sigmask` if the syscall installed a user mask; defense against per-priv-escalation by leaking SIGSTOP-blocking across the syscall boundary.
- **GRKERNSEC_PROC_GETPID** — per-task `/proc/<pid>/status` SigBlk/SigCgt fields filtered for non-owner; defense against per-mask-fingerprinting info-leak.
- **PAX_USERCOPY_HARDEN on `__aio_sigset` and `sigset_t` copy** — bounded copy uses whitelisted slab region.
- **Per-pending-signal early-EINTR** — grsec enforces that if a signal is pending at mask-install, the call returns EINTR without ever entering the waitqueue; defense against per-uninterruptible-sleep DoS by stacking pending signals.
- **PaX KERNEXEC on signal-return trampoline** — defense against per-W^X violation in mask-restore path.
- **GRKERNSEC_NO_GETPID-style leak guard** — `/proc/<pid>/task/*/status` does not expose AIO mask transitions to non-owners.

