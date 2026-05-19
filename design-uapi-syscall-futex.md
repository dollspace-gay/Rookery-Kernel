---
title: "Tier-5 syscall: futex(2) — syscall 202"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`futex(2)` is the kernel building block on which **all userspace synchronization primitives** (glibc / musl mutex/condvar/rwlock/semaphore, C++ std::mutex, Rust std::sync::Mutex, Go runtime, Wine, Java J9 / HotSpot park) stand. A futex is a userspace 32-bit aligned word at virtual address `uaddr`; userspace performs the uncontended fast path with a single atomic CAS, and only enters the kernel on contention. The `futex_op` discriminator selects a wait, wake, requeue, compare-requeue, priority-inheritance, wake-on-other-side, or wait-multiple operation. Per-`FUTEX_WAIT` atomically validates `*uaddr == val` and parks the caller on the kernel wait queue keyed by `(mm, page, offset)` or `(inode, offset)` for shared mappings; per-`FUTEX_WAKE` wakes up to `val` waiters on the same key. Critical for: every contended lock and condvar in the system; signal-correct EINTR semantics; robust-list cleanup on task exit; priority-inheritance to keep latency bounds.

This Tier-5 covers the userspace ABI of syscall 202. Runtime hash-bucket layout, PI state, and robust-list teardown owned by `kernel/futex.md` (Tier-3, planned).

### Acceptance Criteria

- [ ] AC-1: `FUTEX_WAIT` with `*uaddr == val` blocks; signal arrives → `-EINTR`.
- [ ] AC-2: `FUTEX_WAIT` with `*uaddr != val` → `-EAGAIN`.
- [ ] AC-3: `FUTEX_WAIT` with relative 100ms timeout, no wake → `-ETIMEDOUT`.
- [ ] AC-4: `FUTEX_WAKE` wakes correct count; returns count.
- [ ] AC-5: `FUTEX_PRIVATE_FLAG`: same-mm only.
- [ ] AC-6: `FUTEX_CLOCK_REALTIME` + `FUTEX_WAIT_BITSET` honored: absolute REALTIME timeout.
- [ ] AC-7: `FUTEX_WAIT_BITSET` with `val3 == 0` → `-EINVAL`.
- [ ] AC-8: `FUTEX_WAKE_BITSET` filters by mask.
- [ ] AC-9: `FUTEX_CMP_REQUEUE` with `*uaddr != val3` → `-EAGAIN`; else wakes + requeues count.
- [ ] AC-10: `FUTEX_WAKE_OP` performs arithmetic + comparison on `*uaddr2`, conditional wake.
- [ ] AC-11: `FUTEX_LOCK_PI`: uncontended fast-path returns 0; contended: boosts owner priority.
- [ ] AC-12: `FUTEX_LOCK_PI` by current owner → `-EDEADLK`.
- [ ] AC-13: `FUTEX_UNLOCK_PI` by non-owner → `-EPERM`.
- [ ] AC-14: Owner death with robust-list registered: waiters see `EOWNERDEAD` and `FUTEX_OWNER_DIED` bit.
- [ ] AC-15: Unknown `futex_op` → `-EINVAL`.
- [ ] AC-16: `uaddr` unaligned (e.g., +1) → `-EINVAL`.

### Architecture

Rookery surface in `kernel/futex/syscall.rs`:

```rust
#[repr(i32)]
pub enum FutexOp {
    Wait              = 0,
    Wake              = 1,
    CmpRequeue        = 4,
    WakeOp            = 5,
    LockPi            = 6,
    UnlockPi          = 7,
    TrylockPi         = 8,
    WaitBitset        = 9,
    WakeBitset        = 10,
    WaitRequeuePi     = 11,
    CmpRequeuePi      = 12,
    LockPi2           = 13,
}

pub const FUTEX_PRIVATE_FLAG:   i32 = 0x80;
pub const FUTEX_CLOCK_REALTIME: i32 = 0x100;
pub const FUTEX_CMD_MASK:       i32 = !(FUTEX_PRIVATE_FLAG | FUTEX_CLOCK_REALTIME);
pub const FUTEX_TID_MASK:       u32 = 0x3fff_ffff;
pub const FUTEX_WAITERS:        u32 = 0x8000_0000;
pub const FUTEX_OWNER_DIED:     u32 = 0x4000_0000;
pub const FUTEX_BITSET_MATCH_ANY: u32 = 0xffff_ffff;
```

`Futex::syscall(uaddr, op_flags, val, ts_or_val2, uaddr2, val3) -> isize`:
1. let op = op_flags & FUTEX_CMD_MASK;
2. let private = (op_flags & FUTEX_PRIVATE_FLAG) != 0;
3. let realtime = (op_flags & FUTEX_CLOCK_REALTIME) != 0;
4. /* alignment */
5. if !uaddr.is_aligned(4) { return -EINVAL; }
6. match FutexOp::try_from(op).map_err(|_| -EINVAL)? {
   - Wait        => futex_wait(uaddr, val, ts_or_val2, FUTEX_BITSET_MATCH_ANY, private, realtime),
   - Wake        => futex_wake(uaddr, val, FUTEX_BITSET_MATCH_ANY, private),
   - WaitBitset  => {
       - if val3 == 0 { return -EINVAL; }
       - futex_wait(uaddr, val, ts_or_val2, val3, private, realtime)
     },
   - WakeBitset  => {
       - if val3 == 0 { return -EINVAL; }
       - futex_wake(uaddr, val, val3, private)
     },
   - CmpRequeue  => futex_cmp_requeue(uaddr, val, ts_or_val2 as u32, uaddr2, val3, private),
   - WakeOp      => futex_wake_op(uaddr, val, ts_or_val2 as u32, uaddr2, val3, private),
   - LockPi      => futex_lock_pi(uaddr, ts_or_val2, false, private),
   - LockPi2     => futex_lock_pi(uaddr, ts_or_val2, realtime, private),
   - UnlockPi    => futex_unlock_pi(uaddr, private),
   - TrylockPi   => futex_trylock_pi(uaddr, private),
   - WaitRequeuePi => futex_wait_requeue_pi(uaddr, val, ts_or_val2, uaddr2, val3, private),
   - CmpRequeuePi  => futex_cmp_requeue_pi(uaddr, val, ts_or_val2 as u32, uaddr2, val3, private),
 }

`futex_wait(uaddr, val, ts, bitmask, private, realtime)`:
1. let key = get_futex_key(uaddr, private)?;
2. let cur = atomic_load_user(uaddr).map_err(|_| -EFAULT)?;
3. if cur != val { return -EAGAIN; }
4. let q = FutexQ { task: current, key, bitmask, ... };
5. hb = hash_futex(key); hb.lock();
6. let cur2 = atomic_load_user(uaddr)?;
7. if cur2 != val { hb.unlock(); return -EAGAIN; }
8. hb.queue(&q);
9. hb.unlock();
10. /* sleep with timeout */
11. match wait_with_timeout(ts, realtime) {
    - Wake     => return 0,
    - Signal   => { hb.dequeue(&q); return -EINTR; },
    - Timeout  => { hb.dequeue(&q); return -ETIMEDOUT; },
    }

`futex_wake(uaddr, max, bitmask, private)`:
1. let key = get_futex_key(uaddr, private)?;
2. let hb = hash_futex(key);
3. hb.lock();
4. let mut woken = 0;
5. for q in hb.iter_key(key) {
   - if woken >= max { break; }
   - if (q.bitmask & bitmask) == 0 { continue; }
   - wake_task(q.task);
   - hb.remove(q);
   - woken += 1;
 }
6. hb.unlock();
7. return woken as isize;

### Out of Scope

- `kernel/futex.md` Tier-3 hash-bucket, PI, robust-list internals
- `futex_waitv.md` sibling — multi-futex wait
- `set_robust_list.md` / `get_robust_list.md` — robust-list registration
- glibc / musl userspace lock implementations
- Implementation code

### signature

```c
long futex(uint32_t *uaddr,
           int futex_op,
           uint32_t val,
           const struct timespec *timeout,  /* or uint32_t val2 for requeue ops */
           uint32_t *uaddr2,
           uint32_t val3);
```

Rust ABI shim:

```rust
pub fn sys_futex(uaddr: *mut u32,
                 futex_op: i32,
                 val: u32,
                 timeout: *const __kernel_timespec,
                 uaddr2: *mut u32,
                 val3: u32) -> isize;
```

Syscall number: **202**.
On 32-bit architectures, the time-64-safe variant is `sys_futex_time64` (syscall 422).

### parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `uaddr` | `u32 *` | IN | userspace 32-bit aligned word — futex key |
| `futex_op` | `i32` | IN | operation + flags (`FUTEX_PRIVATE_FLAG`, `FUTEX_CLOCK_REALTIME`) |
| `val` | `u32` | IN | per-op expected value, wake count, requeue count, etc. |
| `timeout` | `const __kernel_timespec *` | IN | per-op timeout (FUTEX_WAIT) or `val2` cast (FUTEX_CMP_REQUEUE) |
| `uaddr2` | `u32 *` | IN | per-op second futex address (requeue, wake_op, wait_requeue_pi) |
| `val3` | `u32` | IN | per-op bitmask or expected value (FUTEX_WAIT_BITSET, FUTEX_CMP_REQUEUE expected) |

### return

- **Success**: per-op (count of waiters woken for `WAKE` / `WAKE_OP` / `REQUEUE`; `0` for `WAIT*` / `LOCK_PI` / `UNLOCK_PI`).
- **Failure**: `-1` and `errno`; Rust internal returns negated errno.

### errors

| errno | Trigger |
|---|---|
| `EAGAIN` | per-`FUTEX_WAIT`: `*uaddr != val` at the point of sleep (race-resolution; userspace retries fast path); per-`CMP_REQUEUE`: `*uaddr != val3`; per-`LOCK_PI`: lock taken under no contention |
| `EINTR` | wait interrupted by signal that is not handled with `SA_RESTART`-equivalent; per-`WAIT_REQUEUE_PI` interrupted before requeue |
| `EINVAL` | `uaddr` not 4-byte aligned; unknown `futex_op`; PI op on already-held lock by current; PI op on mismatched owner tid; `val` exceeds per-op cap; `val3 == 0` for `WAIT_BITSET`/`WAKE_BITSET` |
| `EFAULT` | `uaddr` / `uaddr2` not user-readable; `timeout` not readable |
| `ETIMEDOUT` | per-`FUTEX_WAIT` / `WAIT_BITSET` / `LOCK_PI`: timeout elapsed |
| `EDEADLK` | PI op would self-deadlock (current already holds futex) |
| `ENOSYS` | unsupported `futex_op` on this kernel build |
| `ESRCH` | PI op: TID encoded in `*uaddr` does not refer to a living task |
| `EPERM` | per-PI op: caller cannot become owner (capability or namespace mismatch) |
| `ENOMEM` | hash bucket allocation failed (rare) |
| `EOWNERDEAD` | PI op: previous owner died holding lock; lock acquired with `FUTEX_OWNER_DIED` bit set |
| `ENOTRECOVERABLE` | PI op: lock state is not recoverable |
| `EBUSY` | (legacy) PI op against contended state in racy phase |

### abi surface

`futex_op` core operations (low 7 bits after masking flags):

| Op | Value | Purpose |
|---|---|---|
| `FUTEX_WAIT` | 0 | atomically validate `*uaddr == val`, sleep until wake or timeout |
| `FUTEX_WAKE` | 1 | wake up to `val` waiters on `uaddr` |
| `FUTEX_FD` | 2 | (removed in 2.6.26) |
| `FUTEX_REQUEUE` | 3 | (deprecated; use `CMP_REQUEUE`) |
| `FUTEX_CMP_REQUEUE` | 4 | wake `val` then requeue up to `val2` to `uaddr2` if `*uaddr == val3` |
| `FUTEX_WAKE_OP` | 5 | wake on `uaddr` then conditionally on `uaddr2` per op-encoding in `val3` |
| `FUTEX_LOCK_PI` | 6 | acquire priority-inheritance lock at `uaddr` |
| `FUTEX_UNLOCK_PI` | 7 | release priority-inheritance lock at `uaddr` |
| `FUTEX_TRYLOCK_PI` | 8 | try-acquire PI lock; `-EAGAIN` if held |
| `FUTEX_WAIT_BITSET` | 9 | `WAIT` filtered by `val3` bitmask |
| `FUTEX_WAKE_BITSET` | 10 | `WAKE` filtered by `val3` bitmask |
| `FUTEX_WAIT_REQUEUE_PI` | 11 | wait, then PI-requeue on wake |
| `FUTEX_CMP_REQUEUE_PI` | 12 | requeue with PI |
| `FUTEX_LOCK_PI2` | 13 | `LOCK_PI` with `CLOCK_REALTIME` timeout |

Flag bits (OR'd into `futex_op`):

| Flag | Value | Purpose |
|---|---|---|
| `FUTEX_PRIVATE_FLAG` | `128` (`0x80`) | key is per-process (uses `mm` instead of `inode`) — fast path |
| `FUTEX_CLOCK_REALTIME` | `256` (`0x100`) | `timeout` is absolute on `CLOCK_REALTIME`; default is relative on `CLOCK_MONOTONIC` |

`FUTEX_BITSET_MATCH_ANY = 0xffffffff` — match-all bitmask.

Owner-tid encoding in `*uaddr` for PI ops:

| Bit pattern | Meaning |
|---|---|
| `tid` low 30 bits | owner TID |
| `FUTEX_WAITERS = 0x80000000` | one or more waiters |
| `FUTEX_OWNER_DIED = 0x40000000` | owner died holding lock |
| `FUTEX_TID_MASK = 0x3fffffff` | TID extraction mask |

### compatibility contract

REQ-1: `uaddr` alignment:
- MUST be 4-byte aligned else `-EINVAL`.
- MUST be in caller-readable userspace else `-EFAULT`.

REQ-2: `FUTEX_WAIT`:
- Read `*uaddr` atomically.
- If `*uaddr != val`: return `-EAGAIN`.
- Else: park on key derived from `(mm-or-inode, page, offset)` until wake / signal / timeout.
- On signal: `-EINTR`.
- On timeout: `-ETIMEDOUT`.
- On wake: return `0`.

REQ-3: `FUTEX_WAKE`:
- Wake up to `val` waiters parked on `uaddr`.
- Return count actually woken (≤ val).
- If no waiters: return `0`.

REQ-4: `FUTEX_PRIVATE_FLAG`:
- Without flag: key is `(inode, offset, offset_in_page)` — works across `mmap(MAP_SHARED)`-shared mappings.
- With flag: key is `(mm, address)` — fast path; only same-mm waiters match.

REQ-5: `FUTEX_CLOCK_REALTIME`:
- Without: `timeout` interpreted as relative `CLOCK_MONOTONIC`.
- With: `timeout` interpreted as absolute `CLOCK_REALTIME`.
- For `LOCK_PI` only `LOCK_PI2` honors this flag.

REQ-6: `FUTEX_WAIT_BITSET`:
- Like `WAIT` but caller-supplied `val3` bitmask MUST be non-zero (else `-EINVAL`).
- Wake only matches if `wake_mask & wait_mask != 0`.

REQ-7: `FUTEX_WAKE_BITSET`:
- Like `WAKE` but only wakes waiters whose `wait_mask & val3 != 0`.

REQ-8: `FUTEX_CMP_REQUEUE`:
- Reads `timeout` field as `val2` (count of waiters to requeue).
- Atomically: if `*uaddr == val3`, wake `val` from `uaddr` and move up to `val2` to `uaddr2`.
- If `*uaddr != val3`: return `-EAGAIN`.
- Returns total (woken + requeued).

REQ-9: `FUTEX_WAKE_OP`:
- `val3` encodes an arithmetic op + comparison on `*uaddr2`:
  - bits 28-31: opcode (SET/ADD/OR/ANDN/XOR).
  - bit 27: shift-arg flag.
  - bits 16-26: opcode argument.
  - bits 12-15: comparison (EQ/NE/LT/LE/GT/GE).
  - bits 0-11: compare argument.
- Wake `val` on `uaddr`; then perform op on `*uaddr2`; if comparison holds, wake `val2` on `uaddr2`.

REQ-10: `FUTEX_LOCK_PI` / `LOCK_PI2`:
- `val` MUST be zero.
- `val3` MUST be zero.
- TID encoding at `*uaddr` MUST be lock-currently-held or zero.
- On contention: kernel boosts current owner priority to caller's priority (priority inheritance).
- On owner-died: returns `-EOWNERDEAD` with `FUTEX_OWNER_DIED` set in `*uaddr`.
- Idempotent: caller already owns ⟹ `-EDEADLK`.

REQ-11: `FUTEX_UNLOCK_PI`:
- Caller MUST currently own the futex.
- Wake highest-priority waiter and hand off ownership atomically.

REQ-12: `FUTEX_TRYLOCK_PI`:
- Non-blocking lock attempt.
- Held: `-EAGAIN`.
- Acquired: `0`.

REQ-13: `FUTEX_WAIT_REQUEUE_PI` / `CMP_REQUEUE_PI`:
- Used by glibc `pthread_cond_wait` PI variant.
- `WAIT_REQUEUE_PI` waits on `uaddr`; on wake, must be requeued to `uaddr2` (a PI futex).

REQ-14: Signal semantics:
- Pending signal aborts wait with `-EINTR`.
- Non-restartable: `pthread_cond_wait` retries internally; raw `futex(WAIT)` is not auto-restarted.

REQ-15: Robust-list teardown:
- Per-thread robust-list head registered via `set_robust_list(2)`.
- On task exit, kernel walks list, sets `FUTEX_OWNER_DIED` bit on each held lock, and wakes one waiter.

REQ-16: Timeout precision:
- `__kernel_timespec`: 64-bit `tv_sec` on every architecture in `sys_futex_time64`.
- 32-bit legacy `struct timespec` accepted only on 32-bit and pre-time64 builds.

REQ-17: Memory-policy interaction:
- For shared futex, the (inode, offset) key derives from the underlying page; mremap/munmap of waiting region behaves as detailed in `futex_get_key`.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `uaddr_aligned` | INVARIANT | `uaddr & 3 != 0 ⟹ -EINVAL` |
| `op_known` | INVARIANT | unknown op ⟹ -EINVAL |
| `wait_val_recheck` | INVARIANT | `*uaddr != val` under hb-lock ⟹ -EAGAIN |
| `bitset_nonzero` | INVARIANT | `WAIT_BITSET`/`WAKE_BITSET` ∧ `val3 == 0` ⟹ -EINVAL |
| `pi_self_deadlock` | INVARIANT | `LOCK_PI` ∧ owner==current ⟹ -EDEADLK |
| `pi_unlock_owner_only` | INVARIANT | `UNLOCK_PI` ∧ owner != current ⟹ -EPERM |
| `wake_count_bounded` | INVARIANT | post: returned ≤ `val` |
| `key_stable_under_hb_lock` | INVARIANT | key bucket lock held during enqueue/dequeue |

### Layer 2: TLA+

`uapi/futex.tla`:
- Variables: `users(*uaddr)`, `wait_queue(key)`, `pi_owner(key)`.
- Properties:
  - `safety_no_lost_wakeup` — WAIT after WAKE always converges (no wakeup-lost between cmp and queue).
  - `safety_pi_no_inversion` — PI owner priority ≥ max(waiter priority).
  - `safety_robust_owner_died` — task exit ⟹ all robust-list locks gain `OWNER_DIED`.
  - `liveness_wake_eventually_returns` — WAKE terminates.
  - `liveness_wait_terminates` — WAIT terminates on wake / signal / timeout.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `futex_wait` post(0): waiter dequeued and woken | `Futex::wait` |
| `futex_wake` post: returns count, count ≤ `val` | `Futex::wake` |
| `cmp_requeue` post: returns woken+requeued | `Futex::cmp_requeue` |
| `lock_pi` post(0): `*uaddr & TID_MASK == current.tid` | `Futex::lock_pi` |
| `unlock_pi` post: prior owner == current; ownership transferred to highest-pri waiter | `Futex::unlock_pi` |

### Layer 4: Verus/Creusot functional

Per-`futex(2)` man page, `Documentation/userspace-api/futex2.rst`, glibc `nptl/lowlevellock.h`, glibc `nptl/pthread_mutex_lock.c` (PI path), Rust std `sys::unix::futex`, Go runtime `runtime·futex`. Per-LTP `testcases/kernel/syscalls/futex/*` round-trip.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

futex reinforcement:

- **Per-PI owner-died strict** — defense against per-stale-owner UAF.
- **Per-robust-list bounded traversal** — defense against per-malicious-list infinite loop on task exit.
- **Per-hash-bucket lock strictly held during cmp-and-queue** — defense against per-lost-wakeup.
- **Per-key derivation refuses non-userspace pages** — defense against per-kernel-page-futex.
- **Per-PI priority boost bounded** — defense against per-priority-inversion attack via long chains.

### grsecurity/pax-style reinforcement

- **PAX_RANDKSTACK on futex entry** — randomize kernel stack at every futex syscall; futex is the highest-frequency syscall in many workloads (every contended mutex) and a known target for stack layout discovery via timing differences in PI chains.
- **PaX UDEREF on `copy_from_user`/`copy_to_user` of `*uaddr`, `*uaddr2`, `timeout`** — enforces unsigned-only userspace deref; refuses kernel pointers masqueraded as `uaddr` (the historical futex use-after-free chain begins with type-confused mappings).
- **GRKERNSEC_BPF_HARDEN considerations** — while futex is not BPF, it shares the "arbitrary user-controlled state machine touching kernel data structures" property; treat PI chain depth as an unbounded primitive and refuse PI chain depth > 64 (configurable) with audit log.
- **CAP_BPF gating not appropriate** — futex is a baseline primitive; instead apply per-uid quota: cap simultaneous futex waiters per uid and rate-limit `FUTEX_LOCK_PI` per uid.
- **GRKERNSEC_FIFO scaled to futex flood** — per-uid quota on simultaneous waiters and per-uid rate-limit on `FUTEX_WAIT` / `FUTEX_LOCK_PI`; refuses queue with `-EAGAIN` past quota to break futex-flood DoS against kernel hash buckets (the historical "hash collision storm" attack class). The same per-uid bucket-fairness policy that grsec's GRKERNSEC_FIFO applies to pipes applies here.
- **PaX UDEREF on robust-list head fetch** — `set_robust_list(2)` registers a userspace pointer that is walked at task exit; UDEREF the head pointer + every `next` pointer along the chain; refuses kernel-mapped pages.
- **Robust-list traversal bounded** — under hardened policy cap robust-list walks at 1024 entries; longer lists are truncated with audit log; defense against per-malicious-list kernel-stall on task exit (a known DoS pattern).
- **GRKERNSEC_HIDESYM on `FUTEX_OWNER_DIED` reflection** — under `kernel.kptr_restrict ≥ 2` the kernel does not leak prior-owner TID through `*uaddr` to processes outside the owner's ns_pid namespace.
- **PI chain depth + priority-boost cap** — refuse priority boosts that would invert a CAP_SYS_NICE-protected scheduling class; reject deep boost chains; mitigates PI-inversion DoS against realtime tasks.
- **EINTR/EAGAIN audit pattern detection** — repeated `EAGAIN` cycles on the same `uaddr` are characteristic of livelock or torn-write probing; rate-limit + log per uid.
- **`FUTEX_CLOCK_REALTIME` strict honoring** — refuse `FUTEX_CLOCK_REALTIME` for non-`WAIT_BITSET` / non-`LOCK_PI2` ops; defense against clock-source confusion exploits.

