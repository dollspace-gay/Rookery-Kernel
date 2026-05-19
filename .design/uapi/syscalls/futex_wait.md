# Tier-5 syscall: futex_wait(2) — syscall 455

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/futex/syscalls.c (SYSCALL_DEFINE6(futex_wait), futex2_to_flags)
  - kernel/futex/waitwake.c (futex_wait, futex_wait_setup, futex_wait_queue_me)
  - kernel/futex/core.c (futex_hash, get_futex_key, futex_q)
  - include/uapi/linux/futex.h (FUTEX2_SIZE_*, FUTEX2_NUMA, FUTEX2_PRIVATE)
  - arch/x86/entry/syscalls/syscall_64.tbl (455  common  futex_wait)
-->

## Summary

`futex_wait(2)` is the futex2 blocking counterpart to `futex_wake(2)`. The caller atomically tests that the userspace word at `uaddr` equals `val` and, if so, sleeps until either a matching `futex_wake(2)` arrives (with `wake_mask & mask != 0`), the absolute timeout `timeout` (interpreted in clock `clockid`) expires, or a signal is delivered. Like `futex_wake`, `flags` carries the `FUTEX2_*` size/NUMA/PRIVATE encoding, and the bitset `mask` selects which wake calls are eligible to wake this waiter.

Critical for: pthread mutex contended path, pthread_cond_wait, glibc nptl, Rust `parking_lot`/`std::sync`, Wine/Proton ESYNC/FSYNC.

## Signature

```c
long futex_wait(
    void *uaddr,
    unsigned long val,
    unsigned long mask,
    unsigned int flags,
    struct __kernel_timespec *timeout,    /* absolute, NULL = no timeout */
    clockid_t clockid                     /* CLOCK_MONOTONIC or CLOCK_REALTIME */
);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `uaddr` | `void *` | in | Userspace address of the futex word. Naturally aligned for chosen element size. |
| `val` | `unsigned long` | in | Expected value of `*uaddr`. Compared atomically under bucket lock; mismatch ⟹ `-EAGAIN`. Width truncated to element size. |
| `mask` | `unsigned long` | in | Bitset stored with the waiter. Wake-side `wake_mask & this mask != 0` is required to fire this waiter. `mask == 0` ⟹ `-EINVAL`. |
| `flags` | `unsigned int` | in | `FUTEX2_*` packed (size, NUMA, PRIVATE). |
| `timeout` | `struct timespec *` | in | Absolute deadline in `clockid` time. `NULL` ⟹ wait forever. |
| `clockid` | `clockid_t` | in | `CLOCK_MONOTONIC` or `CLOCK_REALTIME`; other clocks ⟹ `-EINVAL`. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Woken by `futex_wake(2)` (intended path). |
| `-1` + `errno` | Failure or non-wake exit. |

## Errors

| errno | Trigger |
|---|---|
| `EAGAIN` | `*uaddr != val` at the moment of compare-and-queue (caller should re-read userspace state and retry the lock). |
| `ETIMEDOUT` | Absolute `timeout` reached before any matching wake. |
| `EINTR` | Signal delivered to the calling thread; caller retries (man-page contract). |
| `EFAULT` | `uaddr` or `timeout` user pointer invalid. |
| `EINVAL` | `mask == 0`, bad `flags`, bad `clockid`, misaligned `uaddr`, `timeout` with out-of-range nsec. |
| `ENOSYS` | Kernel lacks futex2 support. |

## ABI surface

```text
__NR_futex_wait  (x86_64) = 455
__NR_futex_wait  (arm64)  = 455
__NR_futex_wait  (riscv)  = 455
__NR_futex_wait  (i386)   = 455

/* Timeout is ABSOLUTE, not relative — unlike legacy FUTEX_WAIT op of futex(2). */
```

## Compatibility contract

REQ-1: Syscall number is **455** on x86_64. ABI-stable.

REQ-2: Compare-and-queue is atomic under the hash bucket spinlock; the read of `*uaddr` and the queue insert happen with no intervening modification.

REQ-3: If `*uaddr != val` at queue time, the bucket lock is released and `-EAGAIN` returned. The userspace word is NOT re-read after queue.

REQ-4: `mask == 0` ⟹ `-EINVAL`. The stored bitset MUST have at least one bit so that any wake can match.

REQ-5: `flags & FUTEX2_SIZE_MASK` selects element size; `*uaddr` is read accordingly (one byte through eight bytes).

REQ-6: `clockid` MUST be `CLOCK_MONOTONIC` or `CLOCK_REALTIME`; tai/boot/raw not accepted.

REQ-7: `timeout` is ABSOLUTE, in `clockid` time. `timeout->tv_nsec` must be in [0, 1e9); negative or >= 1e9 ⟹ `-EINVAL`.

REQ-8: `timeout` deadline already in the past ⟹ atomic compare followed by immediate `-ETIMEDOUT` (if compare matches) or `-EAGAIN` (if not).

REQ-9: Spurious wake-ups ARE possible (signals, internal cgroup migration); the caller MUST treat any return that is not the desired userspace state as a request to retry.

REQ-10: Stored bitset width matches `unsigned long` on the running ABI (LP64: 64 bits, ILP32: 32 bits).

REQ-11: A waiter is removed from its bucket on: matched wake, requeue (by `futex_requeue(2)`), timeout, signal, or process exit.

REQ-12: `FUTEX2_PRIVATE` and `FUTEX2_NUMA` encoding matches `futex_wake(2)`; the wait-side and wake-side flags MUST agree on size and PRIVATE for the keys to match.

REQ-13: PI (priority-inherit) futex semantics are NOT available via futex2 in this release; users requiring PI continue to use legacy `futex(FUTEX_LOCK_PI, ...)`.

REQ-14: `CLOCK_REALTIME` waits absorb wall-clock jumps: a backward jump that pushes the deadline into the future extends the wait; a forward jump that crosses the deadline fires `-ETIMEDOUT`.

REQ-15: Signal mask is honored: blocked signals do NOT wake the waiter (`-EINTR` only when delivered).

## Acceptance Criteria

- [ ] AC-1: Wait with matching `val` blocks; `futex_wake` of the same address wakes; return 0.
- [ ] AC-2: Wait with mismatched `val` returns `-EAGAIN` immediately.
- [ ] AC-3: `mask == 0` returns `-EINVAL`.
- [ ] AC-4: Past-absolute `timeout` with matching `val` returns `-ETIMEDOUT`.
- [ ] AC-5: SIGUSR1 delivered during wait returns `-EINTR`.
- [ ] AC-6: Wait then `futex_wake(mask = ~ours)` does NOT wake (returns continues to block).
- [ ] AC-7: u8 / u16 / u64 element-size variants compare the correct width.
- [ ] AC-8: Misaligned `uaddr` returns `-EINVAL`.
- [ ] AC-9: `CLOCK_BOOTTIME` returns `-EINVAL`.
- [ ] AC-10: `timeout->tv_nsec = 1_000_000_000` returns `-EINVAL`.
- [ ] AC-11: `FUTEX2_PRIVATE` wait + `FUTEX2_PRIVATE` wake in same mm interoperate.

## Architecture

```rust
#[syscall(nr = 455, abi = "sysv")]
pub fn sys_futex_wait(
    uaddr: UserPtr<u8>,
    val: usize,
    mask: usize,
    flags: u32,
    timeout: UserPtr<KernelTimespec>,
    clockid: i32,
) -> isize {
    Futex2::wait(uaddr, val, mask, flags, timeout, clockid)
}
```

`Futex2::wait(uaddr, val, mask, flags, timeout, clockid) -> isize`:
1. let f = Futex2Flags::parse(flags)?;
2. if mask == 0 { return Err(EINVAL); }
3. if !uaddr.is_aligned_to(f.element_size()) { return Err(EINVAL); }
4. let clock = Clock::try_from(clockid)?;
5. let deadline = timeout.map(|t| Clock::read_user(clock, t)).transpose()?;
6. let key = Futex2::get_key(uaddr, f)?;
7. let bucket = Futex2::hash_bucket(&key);
8. let q = FutexQ::new(current.task(), &key, mask);
9. bucket.with_lock(|b| {
10.   let actual = Futex2::read_user_atomic(uaddr, f.element_size())?;
11.   if actual != val.truncate_to(f.element_size()) {
12.     return Err(EAGAIN);
13.   }
14.   b.insert(&q);
15.   Ok(())
16. })?;
17. let res = Futex2::sleep_until(&q, deadline);
18. bucket.with_lock(|b| b.remove_if_present(&q));
19. match res {
20.   WaitOutcome::Woken    => Ok(0),
21.   WaitOutcome::Timeout  => Err(ETIMEDOUT),
22.   WaitOutcome::Signal   => Err(EINTR),
23. }

`Futex2::read_user_atomic(uaddr, sz) -> Result<u64>`:
1. match sz {
2.   1 => Ok(uaddr.read::<u8>()? as u64),
3.   2 => Ok(uaddr.read::<u16>()? as u64),
4.   4 => Ok(uaddr.read::<u32>()? as u64),
5.   8 => Ok(uaddr.read::<u64>()?),
6.   _ => Err(EINVAL),
7. }

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `compare_atomic_with_queue` | INVARIANT | the compare and the queue happen under one bucket lock. |
| `mask_zero_rejected` | INVARIANT | `mask == 0` ⟹ `-EINVAL`. |
| `align_enforced` | INVARIANT | misaligned `uaddr` ⟹ `-EINVAL`. |
| `mismatch_no_queue` | INVARIANT | `*uaddr != val` ⟹ no waiter is inserted. |
| `deadline_absolute` | INVARIANT | sleep uses absolute clockid deadline, not relative. |
| `cleanup_on_exit` | INVARIANT | every exit path removes the waiter from the bucket if present. |

### Layer 2: TLA+

`kernel/futex2-wait.tla`:
- Per-wait transitions: parse, key, hash, lock, compare, queue, sleep, exit.
- Properties:
  - `safety_no_lost_wake` — if a matching wake arrives between compare-success and sleep, it fires the waiter.
  - `safety_no_double_remove` — waiter removed exactly once.
  - `safety_mismatch_returns_eagain` — `*uaddr != val` at compare ⟹ `-EAGAIN`, no sleep.
  - `liveness_wake_or_timeout` — every wait eventually returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Futex2::wait` post: ret == 0 ⟹ matched-wake observed | `Futex2::wait` |
| `Futex2::wait` post: -EAGAIN ⟹ no queue insert | `Futex2::wait` |
| `Futex2::read_user_atomic` post: width-correct read | `Futex2::read_user_atomic` |
| `FutexQ::new` post: bitset == mask | `FutexQ::new` |

### Layer 4: Verus / Creusot functional

Per-`futex_wait(2)` man page; glibc nptl pthread_mutex_lock contended path; pthread_cond_wait wake-set invariant; Wine ntdll futex emulation.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`futex_wait(2)` reinforcement:

- **Per-compare-and-queue atomicity** — defense against per-lost-wake race.
- **Per-`mask == 0` reject** — defense against per-no-eligible-wake bug.
- **Per-element-size alignment** — defense against per-misaligned atomic UB.
- **Per-absolute-deadline contract** — defense against per-relative-vs-absolute time confusion.
- **Per-bucket-lock isolation** — defense against per-cross-key state corruption.

## Grsecurity / PaX surface

- **PaX UDEREF on uaddr and timeout** — defense against per-uaddr / per-timeout kernel-pointer smuggling; SMAP forced.
- **FUTEX2_PRIVATE mandatory in hardened mode** — `GRKERNSEC_FUTEX_PRIVATE_ONLY` rejects shared-mapping waits from non-init userns; covert-channel attenuation via shared hash buckets.
- **Per-`mask == 0` strict reject** — defense against per-pathological-bitset DoS that holds bucket lock indefinitely.
- **PAX_USERCOPY on timespec read** — bounded `copy_from_user` of `struct __kernel_timespec` uses whitelisted slab.
- **Per-`clockid` allow-list** — defense against per-future-clock-ABI smuggle (only MONOTONIC and REALTIME accepted).
- **Per-`futex_hash` salt randomized per-boot** — defense against per-collision DoS.
- **GRKERNSEC_HIDESYM on `struct futex_q`** — internal queue addresses not exposed via oops / /proc.
- **PAX_REFCOUNT on `mm_users` during shared-key derivation** — defense against per-refcount UAF.
- **Per-`EINTR` signal-mask honored** — defense against per-signal-storm bypass of pthread cancellation.
- **Per-bucket spinlock with bounded hold-time** — defense against per-DoS through compare path; reads of `*uaddr` are SMAP-guarded `__copy_from_user_inatomic`.
- **Per-reserved-flag-bit zero** — defense against per-flag-smuggle for future futex2 extensions.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `futex_wake(2)` (see `futex_wake.md`).
- `futex_requeue(2)` (see `futex_requeue.md`).
- PI-futex semantics (Tier-3 `kernel/futex/pi.md`).
- Robust-futex list (Tier-3 `kernel/futex/robust.md`).
- Implementation code.
