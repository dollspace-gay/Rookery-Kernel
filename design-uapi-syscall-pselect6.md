---
title: "Tier-5 syscall: pselect6(2) — syscall 270"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`pselect6(2)` is the signal-mask-aware variant of `select(2)`. It performs the same fd-bitmap readiness wait, with nanosecond-granularity timeout and atomic signal-mask swap for the duration of the wait. The `6` in the name reflects that the kernel ABI takes 6 arguments — POSIX `pselect(3)` is 5, but the kernel splits `sigmask`/`sigsetsize` into a single packed pointer-to-struct to fit the 6-register sysv calling convention on i386 (where 7 args won't fit).

`pselect6` is the moral twin of `ppoll`. Same atomic-mask-install closes the "self-pipe race"; same nanosecond `timespec`; same nfds-bounded by `RLIMIT_NOFILE`. Critical for: tcsh, BSD-port code, openssh, GNU `tail -f`, and the few remaining select-based event loops that need race-free signal coalescing.

### Acceptance Criteria

- [ ] AC-1: pselect6(0, NULL, NULL, NULL, &{0,500000000}, NULL) sleeps 500ms and returns 0.
- [ ] AC-2: pselect6 on a fd 1024 or higher: nfds > FD_SETSIZE returns -EINVAL.
- [ ] AC-3: pselect6 with sigsetsize=4 returns -EINVAL.
- [ ] AC-4: pselect6 with timeout.tv_nsec = 1e9 returns -EINVAL.
- [ ] AC-5: pselect6 with sigmask={SIGCHLD} and SIGCHLD pending: blocked across wait, no EINTR.
- [ ] AC-6: pselect6 with sigmask={} and SIGCHLD pending: EINTR; old mask restored on return.
- [ ] AC-7: pselect6 with read pipe ready: returns 1, FD_ISSET(fd, readfds) true.
- [ ] AC-8: Same fd ready for read and write: return value counts twice (POSIX-compliant).
- [ ] AC-9: User timeout struct unmodified on return (verified via memcmp).
- [ ] AC-10: sigmask attempting to mask SIGKILL: bit silently stripped; SIGKILL still terminates.

### Architecture

```rust
#[syscall(nr = 270, abi = "sysv")]
pub fn sys_pselect6(
    nfds: i32,
    rfds: UserPtr<FdSet>, wfds: UserPtr<FdSet>, efds: UserPtr<FdSet>,
    tmo: UserPtr<TimeSpec>,
    sigmask_arg: UserPtr<SigSetArgpack>,
) -> isize {
    Pselect::do_pselect(nfds, rfds, wfds, efds, tmo, sigmask_arg)
}
```

`Pselect::do_pselect(nfds, rfds, wfds, efds, tmo, sigarg) -> isize`:
1. if nfds < 0 || nfds > FD_SETSIZE || nfds > current.rlim_nofile() { return Err(EINVAL); }
2. let ts = if tmo.is_null() { None } else { Some(unsafe { tmo.copy_in()? }) };
3. if let Some(ts) = &ts { TimeSpec::validate(ts)?; }
4. let sm = if sigarg.is_null() { None } else { Some(unsafe { sigarg.copy_in::<SigSetArgpack>()? }) };
5. if let Some(sm) = &sm {
6.     if sm.sigsetsize != size_of::<SigSet>() { return Err(EINVAL); }
7.     let mask = unsafe { UserPtr::<SigSet>::from_raw(sm.sigmask).copy_in()? };
8.     Signal::install_blocked(mask);
9. }
10. let deadline = ts.map(|t| KTime::monotonic_plus(t.to_ktime())).unwrap_or(KTime::MAX);
11. let ret = Pselect::core_sys_select(nfds, rfds, wfds, efds, deadline);
12. Signal::restore_user_sigmask(ret != Err(EINTR));
13. ret

`Pselect::core_sys_select(nfds, rfds, wfds, efds, deadline) -> isize`:
1. let bytes = (nfds as usize + 7) / 8;
2. let mut k_in = [FdSetKernel::zero(); 3];     // in / out per bitmap × 3
3. let mut k_out = [FdSetKernel::zero(); 3];
4. for (i, p) in [rfds, wfds, efds].iter().enumerate() {
5.     if !p.is_null() { unsafe { p.copy_in_bytes(&mut k_in[i], bytes)?; } }
6. }
7. let count = Select::do_select(nfds, &k_in, &mut k_out, deadline)?;
8. for (i, p) in [rfds, wfds, efds].iter().enumerate() {
9.     if !p.is_null() { unsafe { p.copy_out_bytes(&k_out[i], bytes)?; } }
10. }
11. Ok(count as isize)

### Out of Scope

- select(2) base behavior (covered in Tier-5 `select.md`).
- ppoll(2) (covered in Tier-5 `ppoll.md`).
- Per-driver f_op->poll implementations.
- Implementation code.

### signature

```c
int pselect6(int nfds,
             fd_set *readfds, fd_set *writefds, fd_set *exceptfds,
             const struct timespec *timeout,
             void *sigmask_arg);

/* sigmask_arg points to a packed pair: */
struct sigset_argpack {
    const sigset_t *sigmask;
    size_t          sigsetsize;
};
```

```c
typedef struct {
    unsigned long fds_bits[FD_SETSIZE / (8 * sizeof(unsigned long))];
} fd_set;
#define FD_SETSIZE  1024
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `nfds` | `int` | in | One past highest fd to monitor; must be ≥ 0, ≤ FD_SETSIZE, ≤ RLIMIT_NOFILE. |
| `readfds` | `fd_set *` | in/out | Read-readiness bitmap; `NULL` if none. |
| `writefds` | `fd_set *` | in/out | Write-readiness bitmap; `NULL` if none. |
| `exceptfds` | `fd_set *` | in/out | Exception/OOB bitmap; `NULL` if none. |
| `timeout` | `const struct timespec *` | in | Relative timeout; `NULL` = infinite. |
| `sigmask_arg` | `void *` | in | Packed `(sigmask, sigsetsize)`; `NULL` = no mask change. |

### return value

| Value | Meaning |
|---|---|
| `>0` | Total fds with readiness across the three bitmaps. |
| `0` | Timeout. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EFAULT` | Any fd_set / timeout / sigmask_arg pointer faults. |
| `EINTR` | Signal delivered (after mask swap) before any fd ready. |
| `EINVAL` | `nfds < 0`, `nfds > FD_SETSIZE`, `nfds > RLIMIT_NOFILE`, `timeout->tv_nsec` out of range, `sigsetsize != _NSIG/8`. |
| `ENOMEM` | Cannot allocate per-fd poll-table slab (large nfds). |

### abi surface

```text
__NR_pselect6  (x86_64)  = 270
__NR_pselect6  (arm64)   = 72
__NR_pselect6  (riscv)   = 72
__NR_pselect6  (i386)    = 308   (32-bit timespec; pselect6_time64 = 413)

/* The 6th argument is *not* (sigmask, sigsetsize) but a pointer to a
   2-field struct in user memory.  This is the i386-7-arg workaround.  */
```

### compatibility contract

REQ-1: Syscall number is **270** on x86_64. ABI-stable since 2.6.16.

REQ-2: `nfds` clamped to `min(FD_SETSIZE, RLIMIT_NOFILE)`; values out of range error with `-EINVAL`.

REQ-3: For each non-NULL bitmap pointer, copy `(nfds + 7) / 8` bytes from user; results overwritten on return.

REQ-4: `timeout->tv_nsec` validated to `[0, 1_000_000_000)`. Negative `tv_sec` → `-EINVAL`.

REQ-5: `sigmask_arg`: if non-NULL, copy `struct sigset_argpack { sigmask*, sigsetsize }` from user. `sigsetsize` must equal `_NSIG/8` (= 8 on x86_64), else `-EINVAL`. Then copy the sigset from `sigmask_arg.sigmask`.

REQ-6: Mask install identical to ppoll(2):
- save current.blocked → current.saved_sigmask.
- new mask = user_mask & ~(SIGKILL | SIGSTOP).
- set_current_blocked(new).
- set TIF_RESTORE_SIGMASK.

REQ-7: On EINTR, mask restoration deferred to do_signal slowpath (so handler runs under restored mask). On other exit paths, mask restored inline.

REQ-8: `do_select` walks fds 0..nfds, calling `f_op->poll` for each fd that is set in any of the three input bitmaps. Output bitmap bit set iff the corresponding POLLIN/POLLOUT/POLLPRI condition holds.

REQ-9: Return value is the **sum** of bits set across all three output bitmaps; a fd ready for both read AND write counts twice. This is POSIX-mandated.

REQ-10: Timeout is **relative** at entry; kernel converts to absolute monotonic deadline. User `timeout` memory is not modified (it is const per the kernel ABI; glibc's wrapper handles the historical Linux quirk of decrementing the user timeout — for pselect, Linux follows POSIX and does NOT decrement).

REQ-11: Compat ABI: 32-bit callers use `__NR_pselect6_time32` with `struct old_timespec32` and `compat_sigset_argpack`. COMPAT shim normalizes and dispatches to `do_pselect`.

REQ-12: Per-namespace: no userns-specific behavior; sigmask and fd ranges are per-process.

REQ-13: Per-FD_SETSIZE = 1024 is a hard ABI cap. Larger fd counts require poll(2) or epoll(7).

REQ-14: pselect6(2) does not consume entropy and does not require any capability.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `nfds_clamped` | INVARIANT | nfds ∈ [0, min(FD_SETSIZE, RLIMIT_NOFILE)]. |
| `sigsetsize_strict` | INVARIANT | sigsetsize != sizeof(SigSet) ⟹ EINVAL. |
| `bytes_copy_bounded` | INVARIANT | copy size = (nfds+7)/8 per bitmap. |
| `mask_restored_or_deferred` | INVARIANT | every exit restores blocked or TIF flagged. |
| `timeout_const_unchanged` | INVARIANT | user timeout not written. |

### Layer 2: TLA+

`fs/pselect6-syscall.tla`:
- States: per-validate, per-copy-tmo, per-install-mask, per-copy-bitmaps, per-select, per-copy-out, per-restore.
- Properties:
  - `safety_atomic_install` — same as ppoll; no signal lost.
  - `safety_kill_stop_unmaskable` — SIGKILL/SIGSTOP stripped.
  - `safety_no_bitmap_leak` — copy_out only on success or EINTR.
  - `safety_count_consistent` — return == popcount(out bitmaps).
  - `liveness_select_terminates` — deadline expires ⟹ return 0.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_pselect` post: ret >= 0 ⟹ count matches bitmap popcount | `Pselect::do_pselect` |
| `core_sys_select` post: out bitmaps disjoint with cleared bits | `Pselect::core_sys_select` |
| `SigSetArgpack` copy: sigsetsize equals kernel size | `Pselect::do_pselect` |

### Layer 4: Verus / Creusot functional

Per-pselect(2) POSIX-2017 semantic equivalence; LTP `pselect01..pselect03` and glibc `tst-pselect` pass.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`pselect6(2)` reinforcement:

- **Per-FD_SETSIZE hard cap** — defense against per-bitmap-amplification.
- **Per-nfds RLIMIT_NOFILE strict** — defense against per-fd allocation DoS.
- **Per-sigsetsize strict** — defense against per-future-sigset smuggle.
- **Per-atomic mask install** — defense against per-signal-race.
- **Per-SIGKILL / SIGSTOP unmaskable** — defense against per-immortal-task escape.
- **Per-bytes copy bounded by nfds** — defense against per-overcopy / underread.
- **Per-relative timeout (POSIX)** — defense against per-CLOCK_REALTIME skew.

### grsecurity / pax surface

- **PaX UDEREF on rfds/wfds/efds/timeout/sigmask_arg copy_from_user** — five independently-validated user pointers; SMAP forced; bounded per-pointer.
- **Sigmask atomic install/restore under siglock** — identical guard to ppoll(2); TIF_RESTORE_SIGMASK transitions audited under GRKERNSEC_AUDIT_KERN.
- **GRKERNSEC_RLIMIT_NOFILE** — strict cap on nfds; defense against per-fd-table-blow DoS.
- **PAX_USERCOPY_HARDEN** — bitmap copy uses whitelisted slab; bounded by `(nfds+7)/8`.
- **PAX_REFCOUNT on saved_sigmask** — defense against per-saved_sigmask UAF.
- **GRKERNSEC_TIME monotonic** — defense against per-CLOCK_REALTIME jump.
- **SIGKILL/SIGSTOP unmaskable** — enforced before publication; defense against per-immortal-task escape.
- **sigset_argpack double-deref bounded** — grsec validates inner sigmask pointer separately to avoid TOCTOU between outer struct copy-in and inner sigset copy-in.
- **GRKERNSEC_HIDESYM on do_select vtable** — symbol not in kallsyms.
- **Per-FD_SETSIZE rejection logged** — over-FD_SETSIZE attempts logged to dmesg ratelimited (potential exploit-probe indicator).

