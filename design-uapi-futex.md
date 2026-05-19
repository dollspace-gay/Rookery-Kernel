---
title: "Tier-5 UAPI: include/uapi/linux/futex.h — Futex ABI"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

The futex UAPI is the syscall-edge contract for `futex(2)`, the v2 `futex_waitv(2)` / `futex_wake(2)` / `futex_wait(2)` / `futex_requeue(2)` family, and the robust-futex thread-exit cleanup mechanism advertised by `set_robust_list(2)` / `get_robust_list(2)`. It backs every userspace mutex, condition variable, barrier, rwlock, semaphore, and io_uring waker built on top of futex-based fast-paths — glibc's `pthread_mutex_t`, `pthread_cond_t`, `pthread_rwlock_t`; musl's locks; the Java `LockSupport.park`; Go's runtime semaphore; Rust's `std::sync` / `parking_lot`; tokio's `Notify`; CRIU's checkpoint replay. It defines:

- **Futex `op` codes** `FUTEX_WAIT`, `FUTEX_WAKE`, `FUTEX_FD` (removed in practice — historical artifact), `FUTEX_REQUEUE`, `FUTEX_CMP_REQUEUE`, `FUTEX_WAKE_OP`, `FUTEX_LOCK_PI`, `FUTEX_UNLOCK_PI`, `FUTEX_TRYLOCK_PI`, `FUTEX_WAIT_BITSET`, `FUTEX_WAKE_BITSET`, `FUTEX_WAIT_REQUEUE_PI`, `FUTEX_CMP_REQUEUE_PI`, `FUTEX_LOCK_PI2`.
- **`op`-modifier flags** `FUTEX_PRIVATE_FLAG = 128` (process-private fast path — bypass mm_struct hashing), `FUTEX_CLOCK_REALTIME = 256` (use CLOCK_REALTIME for the timeout instead of CLOCK_MONOTONIC), `FUTEX_CMD_MASK = ~(FUTEX_PRIVATE_FLAG | FUTEX_CLOCK_REALTIME)`.
- **`FUTEX_OP_*` atomic operations for `FUTEX_WAKE_OP`** `FUTEX_OP_SET`, `FUTEX_OP_ADD`, `FUTEX_OP_OR`, `FUTEX_OP_ANDN`, `FUTEX_OP_XOR`, plus `FUTEX_OP_OPARG_SHIFT = 8` (use `(1 << OPARG)` instead of OPARG).
- **`FUTEX_OP_CMP_*` comparison codes** `FUTEX_OP_CMP_EQ`, `FUTEX_OP_CMP_NE`, `FUTEX_OP_CMP_LT`, `FUTEX_OP_CMP_LE`, `FUTEX_OP_CMP_GT`, `FUTEX_OP_CMP_GE`.
- **`FUTEX_OP()` packing macro** — packs `(op, oparg, cmp, cmparg)` into a single u32 for the `FUTEX_WAKE_OP` `val3` argument.
- **futex2 size + flag bits** `FUTEX2_SIZE_U8 = 0x00`, `FUTEX2_SIZE_U16 = 0x01`, `FUTEX2_SIZE_U32 = 0x02`, `FUTEX2_SIZE_U64 = 0x03`, `FUTEX2_NUMA = 0x04`, `FUTEX2_MPOL = 0x08`, `FUTEX2_PRIVATE = FUTEX_PRIVATE_FLAG = 0x80`, `FUTEX2_SIZE_MASK = 0x03`. The historical alias `FUTEX_32 = FUTEX2_SIZE_U32` exists but is documented as "do not use".
- **`FUTEX_NO_NODE = -1`** — sentinel "no node" value carried in the second u32 of a NUMA-aware futex word when `FUTEX2_NUMA` is set; not the same constant as the kernel-internal `NUMA_NO_NODE`.
- **`struct futex_waitv`** — vectorized-wait waiter descriptor: `{ __u64 val; __u64 uaddr; __u32 flags; __u32 __reserved; }`. `FUTEX_WAITV_MAX = 128` is the array-length cap for `futex_waitv(2)`.
- **Robust-list ABI** — `struct robust_list { struct robust_list __user *next; }` (single-link forward only; userspace double-links), `struct robust_list_head { struct robust_list list; long futex_offset; struct robust_list __user *list_op_pending; }`, `ROBUST_LIST_LIMIT = 2048` (max chain length before kernel bails). PI bits in the robust-futex word: `FUTEX_WAITERS = 0x80000000` (waiters present), `FUTEX_OWNER_DIED = 0x40000000` (kernel signals dead owner during thread-exit cleanup), `FUTEX_TID_MASK = 0x3fffffff` (low-30-bits = owner TID).
- **`FUTEX_BITSET_MATCH_ANY = 0xffffffff`** — bitset value that matches every `FUTEX_WAIT_BITSET` / `FUTEX_WAKE_BITSET` waiter.

Critical for: every libc threading primitive, every robust mutex in PostgreSQL/MySQL/Redis, every PI-futex used by SCHED_FIFO real-time tasks, every WAITV-based io_uring completion fast-path, every CRIU restore that needs to rehydrate kernel-side futex_q queues.

This Tier-5 covers `include/uapi/linux/futex.h` (~212 lines). The corresponding implementations (`kernel/futex/core.c`, `kernel/futex/pi.c`, `kernel/futex/requeue.c`, `kernel/futex/waitwake.c`, `kernel/futex/syscalls.c`) are Tier-3 docs separately.

### Acceptance Criteria

- [ ] AC-1: `FUTEX_WAIT == 0`, `FUTEX_WAKE == 1`, `FUTEX_LOCK_PI2 == 13`.
- [ ] AC-2: `FUTEX_FD` returns `-EINVAL` (historical-disabled).
- [ ] AC-3: `op > FUTEX_LOCK_PI2` returns `-ENOSYS`.
- [ ] AC-4: `FUTEX_PRIVATE_FLAG == 0x80`, `FUTEX_CLOCK_REALTIME == 0x100`.
- [ ] AC-5: `FUTEX_WAIT` with `*uaddr != val` returns `-EAGAIN` immediately (no sleep).
- [ ] AC-6: `FUTEX_WAIT_BITSET` with `bitset == 0` returns `-EINVAL`.
- [ ] AC-7: `FUTEX_WAKE_OP(SET, 0, EQ, 0)`: atomically sets `*uaddr2 = 0`, wakes if old was 0.
- [ ] AC-8: `FUTEX_OP(FUTEX_OP_ADD, 1, FUTEX_OP_CMP_GT, 0)` packs to `(1<<28)|(4<<24)|(1<<12)|0`.
- [ ] AC-9: `FUTEX_LOCK_PI` on word containing invalid TID returns `-ESRCH`.
- [ ] AC-10: `FUTEX_UNLOCK_PI` from non-owner returns `-EPERM`.
- [ ] AC-11: Robust-list walk terminates at `ROBUST_LIST_LIMIT == 2048` even with a circular list.
- [ ] AC-12: Thread-exit with held robust futex: `FUTEX_OWNER_DIED` bit set, one waiter woken.
- [ ] AC-13: `sizeof(struct futex_waitv) == 24` and natural alignment is 8 bytes.
- [ ] AC-14: `futex_waitv(2)` with `nr_futexes > 128` returns `-EINVAL`.
- [ ] AC-15: `struct futex_waitv.__reserved != 0` returns `-EINVAL`.
- [ ] AC-16: `FUTEX2_SIZE_MASK == 0x03`; only `_U8/_U16/_U32/_U64` are valid sizes.
- [ ] AC-17: Misaligned `uaddr` (e.g., `uaddr % 4 != 0` for U32) returns `-EINVAL`.
- [ ] AC-18: `FUTEX_TID_MASK | FUTEX_WAITERS | FUTEX_OWNER_DIED == 0xffffffff` (cover the word).
- [ ] AC-19: `set_robust_list(NULL, sizeof(...))` clears the registration; subsequent thread-exit performs no cleanup.
- [ ] AC-20: `FUTEX_LOCK_PI2` with `CLOCK_REALTIME` flag returns `-ENOSYS` (PI2 = MONOTONIC-only).

### Architecture

```
// Op codes (low 7 bits of the futex syscall's op argument)
pub const FUTEX_WAIT: u32             =  0;
pub const FUTEX_WAKE: u32             =  1;
pub const FUTEX_FD: u32               =  2;   // disabled — returns EINVAL
pub const FUTEX_REQUEUE: u32          =  3;
pub const FUTEX_CMP_REQUEUE: u32      =  4;
pub const FUTEX_WAKE_OP: u32          =  5;
pub const FUTEX_LOCK_PI: u32          =  6;
pub const FUTEX_UNLOCK_PI: u32        =  7;
pub const FUTEX_TRYLOCK_PI: u32       =  8;
pub const FUTEX_WAIT_BITSET: u32      =  9;
pub const FUTEX_WAKE_BITSET: u32      = 10;
pub const FUTEX_WAIT_REQUEUE_PI: u32  = 11;
pub const FUTEX_CMP_REQUEUE_PI: u32   = 12;
pub const FUTEX_LOCK_PI2: u32         = 13;

// Op modifiers
pub const FUTEX_PRIVATE_FLAG: u32     = 128;
pub const FUTEX_CLOCK_REALTIME: u32   = 256;
pub const FUTEX_CMD_MASK: u32         = !(FUTEX_PRIVATE_FLAG | FUTEX_CLOCK_REALTIME);

// WAKE_OP atomic ops + cmps
pub const FUTEX_OP_SET: u32   = 0;
pub const FUTEX_OP_ADD: u32   = 1;
pub const FUTEX_OP_OR: u32    = 2;
pub const FUTEX_OP_ANDN: u32  = 3;
pub const FUTEX_OP_XOR: u32   = 4;
pub const FUTEX_OP_OPARG_SHIFT: u32 = 8;

pub const FUTEX_OP_CMP_EQ: u32 = 0;
pub const FUTEX_OP_CMP_NE: u32 = 1;
pub const FUTEX_OP_CMP_LT: u32 = 2;
pub const FUTEX_OP_CMP_LE: u32 = 3;
pub const FUTEX_OP_CMP_GT: u32 = 4;
pub const FUTEX_OP_CMP_GE: u32 = 5;

#[inline]
pub const fn futex_op(op: u32, oparg: u32, cmp: u32, cmparg: u32) -> u32 {
    ((op & 0xf) << 28) | ((cmp & 0xf) << 24)
        | ((oparg & 0xfff) << 12) | (cmparg & 0xfff)
}

// futex2 flag bits
pub const FUTEX2_SIZE_U8: u32   = 0x00;
pub const FUTEX2_SIZE_U16: u32  = 0x01;
pub const FUTEX2_SIZE_U32: u32  = 0x02;
pub const FUTEX2_SIZE_U64: u32  = 0x03;
pub const FUTEX2_NUMA: u32      = 0x04;
pub const FUTEX2_MPOL: u32      = 0x08;
pub const FUTEX2_PRIVATE: u32   = FUTEX_PRIVATE_FLAG;
pub const FUTEX2_SIZE_MASK: u32 = 0x03;

pub const FUTEX_NO_NODE: i32     = -1;
pub const FUTEX_WAITV_MAX: u32   = 128;

#[repr(C)]
#[derive(Copy, Clone)]
pub struct FutexWaitv {
    pub val:        u64,
    pub uaddr:      u64,         // user pointer as u64 for 32/64-bit symmetry
    pub flags:      u32,         // FUTEX2_* mask
    pub __reserved: u32,         // MUST be 0
}

const _ASSERT_FUTEX_WAITV_SIZE: () =
    assert!(core::mem::size_of::<FutexWaitv>() == 24);
const _ASSERT_FUTEX_WAITV_ALIGN: () =
    assert!(core::mem::align_of::<FutexWaitv>() == 8);

// Robust-futex ABI
#[repr(C)]
pub struct RobustList {
    pub next: UserPtr<RobustList>,
}

#[repr(C)]
pub struct RobustListHead {
    pub list:             RobustList,
    pub futex_offset:     c_long,
    pub list_op_pending:  UserPtr<RobustList>,
}

pub const ROBUST_LIST_LIMIT: u32 = 2048;

// Robust-futex word bits
pub const FUTEX_WAITERS: u32    = 0x80000000;
pub const FUTEX_OWNER_DIED: u32 = 0x40000000;
pub const FUTEX_TID_MASK: u32   = 0x3fffffff;

pub const FUTEX_BITSET_MATCH_ANY: u32 = 0xffffffff;
```

`SysFutex::sys_futex(uaddr, op, val, timeout, uaddr2, val3) -> Result<i32>`:
1. /* Strip modifiers */
2. let bare = op & FUTEX_CMD_MASK;
3. let private = (op & FUTEX_PRIVATE_FLAG) != 0;
4. let realtime = (op & FUTEX_CLOCK_REALTIME) != 0;
5. /* Validate op range */
6. if bare > FUTEX_LOCK_PI2: return Err(ENOSYS).
7. if bare == FUTEX_FD: return Err(EINVAL).        // historical-disabled
8. /* Realtime only with WAIT_BITSET, WAIT_REQUEUE_PI, LOCK_PI */
9. if realtime && !matches!(bare, FUTEX_WAIT_BITSET | FUTEX_WAIT_REQUEUE_PI | FUTEX_LOCK_PI):
       return Err(ENOSYS).
10. /* Dispatch */
11. match bare {
        FUTEX_WAIT             => Futex::wait(uaddr, val, timeout, FUTEX_BITSET_MATCH_ANY, private, false /*abs*/),
        FUTEX_WAIT_BITSET      => Futex::wait(uaddr, val, timeout, val3, private, true /*abs*/),
        FUTEX_WAKE             => Futex::wake(uaddr, val, FUTEX_BITSET_MATCH_ANY, private),
        FUTEX_WAKE_BITSET      => Futex::wake(uaddr, val, val3, private),
        FUTEX_REQUEUE          => Futex::requeue(uaddr, val, uaddr2, timeout as u32, None, private),
        FUTEX_CMP_REQUEUE      => Futex::requeue(uaddr, val, uaddr2, timeout as u32, Some(val3), private),
        FUTEX_WAKE_OP          => Futex::wake_op(uaddr, val, uaddr2, timeout as u32, val3, private),
        FUTEX_LOCK_PI          => Futex::lock_pi(uaddr, timeout, private, ClockSource::Realtime),
        FUTEX_LOCK_PI2         => Futex::lock_pi(uaddr, timeout, private, ClockSource::Monotonic),
        FUTEX_UNLOCK_PI        => Futex::unlock_pi(uaddr, private),
        FUTEX_TRYLOCK_PI       => Futex::trylock_pi(uaddr, private),
        FUTEX_WAIT_REQUEUE_PI  => Futex::wait_requeue_pi(uaddr, val, timeout, uaddr2, private),
        FUTEX_CMP_REQUEUE_PI   => Futex::cmp_requeue_pi(uaddr, val, uaddr2, timeout as u32, val3, private),
        _ => Err(EINVAL),
    }

`SysFutex::sys_futex_waitv(waiters_user, nr_futexes, flags, timeout, clockid) -> Result<i32>`:
1. if flags != 0: return Err(EINVAL).
2. if nr_futexes == 0 || nr_futexes > FUTEX_WAITV_MAX: return Err(EINVAL).
3. let mut waiters = [FutexWaitv::zeroed(); 128];
4. copy_from_user(&mut waiters[..nr_futexes as usize], waiters_user)?.
5. /* Validate each waiter */
6. for w in &waiters[..nr_futexes as usize] {
       if w.__reserved != 0: return Err(EINVAL).
       let size_idx = w.flags & FUTEX2_SIZE_MASK;
       let known = FUTEX2_SIZE_MASK | FUTEX2_NUMA | FUTEX2_MPOL | FUTEX2_PRIVATE;
       if w.flags & !known != 0: return Err(EINVAL).
       /* Alignment by size */
       let bytes = 1u64 << size_idx;
       if w.uaddr & (bytes - 1) != 0: return Err(EINVAL).
   }
7. Futex::vector_wait(&waiters[..nr_futexes as usize], timeout, clockid)

`SysFutex::sys_set_robust_list(head_user, len) -> Result<()>`:
1. if len != core::mem::size_of::<RobustListHead>(): return Err(EINVAL).
2. current.robust_list = head_user;          // u64 user pointer
3. return Ok(()).

`Futex::wake_op(uaddr, val, uaddr2, val2, val3, private)`:
1. /* Unpack FUTEX_OP() */
2. let op_full = (val3 >> 28) & 0xf;
3. let op      = op_full & 0x7;
4. let shift   = (op_full & FUTEX_OP_OPARG_SHIFT >> 4) != 0;   // bit 3
5. let cmp     = (val3 >> 24) & 0xf;
6. let oparg_raw = (val3 >> 12) & 0xfff;
7. let cmparg = val3 & 0xfff;
8. let oparg = if shift { 1u32 << (oparg_raw & 31) } else { oparg_raw };
9. /* Atomic op on uaddr2 */
10. let oldval = atomic_get_user(uaddr2)?;
11. let newval = match op {
        FUTEX_OP_SET  => oparg,
        FUTEX_OP_ADD  => oldval.wrapping_add(oparg),
        FUTEX_OP_OR   => oldval | oparg,
        FUTEX_OP_ANDN => oldval & !oparg,
        FUTEX_OP_XOR  => oldval ^ oparg,
        _             => return Err(ENOSYS),
    };
12. atomic_set_user(uaddr2, newval)?;
13. /* Conditional wake of uaddr2 */
14. let woken_uaddr2 = match cmp {
        FUTEX_OP_CMP_EQ => oldval == cmparg,
        FUTEX_OP_CMP_NE => oldval != cmparg,
        FUTEX_OP_CMP_LT => oldval <  cmparg,
        FUTEX_OP_CMP_LE => oldval <= cmparg,
        FUTEX_OP_CMP_GT => oldval >  cmparg,
        FUTEX_OP_CMP_GE => oldval >= cmparg,
        _               => return Err(ENOSYS),
    };
15. let n = Futex::wake(uaddr, val, FUTEX_BITSET_MATCH_ANY, private)?;
16. let m = if woken_uaddr2 { Futex::wake(uaddr2, val2, FUTEX_BITSET_MATCH_ANY, private)? } else { 0 };
17. Ok(n + m)

### Out of Scope

- `kernel/futex/core.c` per-futex hash table + `futex_q` lifecycle — covered in `kernel/futex/00-overview.md` Tier-3
- `kernel/futex/pi.c` priority-inheritance chain + `rt_mutex` integration — covered in `kernel/futex/pi.md` Tier-3
- `kernel/futex/requeue.c` requeue mechanics — covered in `kernel/futex/requeue.md` Tier-3
- `kernel/futex/waitwake.c` and `waitv` vector implementation — covered in `kernel/futex/waitwake.md` Tier-3
- `kernel/futex/syscalls.c` dispatcher — covered in `kernel/futex/syscalls.md` Tier-3
- glibc / musl userspace futex fast-paths — out of UAPI scope (covered in `userspace-runtime/threading.md`)
- `Documentation/admin-guide/sysctl/kernel.rst` `kernel.unprivileged_userns_clone` interaction — covered in `uapi/syscalls/clone3.md` Tier-5
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `FUTEX_WAIT` / `FUTEX_WAKE` | per-basic block/wake | shared |
| `FUTEX_FD` | per-historical fd-style (removed) | shared (returns ENOSYS) |
| `FUTEX_REQUEUE` / `FUTEX_CMP_REQUEUE` | per-move waiters | shared |
| `FUTEX_WAKE_OP` | per-atomic-op + wake | shared |
| `FUTEX_LOCK_PI` / `FUTEX_LOCK_PI2` / `FUTEX_UNLOCK_PI` / `FUTEX_TRYLOCK_PI` | per-PI mutex | shared |
| `FUTEX_WAIT_BITSET` / `FUTEX_WAKE_BITSET` | per-bitset waiter | shared |
| `FUTEX_WAIT_REQUEUE_PI` / `FUTEX_CMP_REQUEUE_PI` | per-condvar PI | shared |
| `FUTEX_PRIVATE_FLAG` / `FUTEX_CLOCK_REALTIME` | per-op modifier | shared |
| `FUTEX_CMD_MASK` | per-op extraction | shared |
| `struct futex_waitv` | per-waitv waiter | `FutexWaitv` |
| `FUTEX2_SIZE_*` / `FUTEX2_NUMA` / `FUTEX2_MPOL` / `FUTEX2_PRIVATE` | per-futex2 flag bits | shared |
| `FUTEX_NO_NODE` | per-NUMA "no node" sentinel | shared |
| `FUTEX_WAITV_MAX` | per-waitv array cap | shared |
| `FUTEX_OP_SET` / `_ADD` / `_OR` / `_ANDN` / `_XOR` | per-wake_op atomic | shared |
| `FUTEX_OP_OPARG_SHIFT` | per-wake_op shift bit | shared |
| `FUTEX_OP_CMP_EQ` / `_NE` / `_LT` / `_LE` / `_GT` / `_GE` | per-wake_op cmp | shared |
| `FUTEX_OP()` | per-pack macro | shared |
| `struct robust_list` / `struct robust_list_head` | per-robust ABI | `RobustList` / `RobustListHead` |
| `FUTEX_WAITERS` / `FUTEX_OWNER_DIED` / `FUTEX_TID_MASK` | per-robust word bits | shared |
| `ROBUST_LIST_LIMIT` | per-chain cap | shared |
| `FUTEX_BITSET_MATCH_ANY` | per-match-all bitset | shared |

### abi surface (constants + structs)

### Futex `op` codes (`uapi/linux/futex.h:11-24`)

```text
FUTEX_WAIT             =  0    /* if (*uaddr == val) sleep until WAKE or timeout */
FUTEX_WAKE             =  1    /* wake up to val waiters */
FUTEX_FD               =  2    /* historical: get fd for SIGIO (removed; returns -ENOSYS) */
FUTEX_REQUEUE          =  3    /* wake val, move remainder to uaddr2 */
FUTEX_CMP_REQUEUE      =  4    /* like REQUEUE but require *uaddr == val3 first */
FUTEX_WAKE_OP          =  5    /* atomic op on uaddr2 + conditional wake */
FUTEX_LOCK_PI          =  6    /* PI-mutex acquire (CLOCK_REALTIME timeout) */
FUTEX_UNLOCK_PI        =  7    /* PI-mutex release */
FUTEX_TRYLOCK_PI       =  8    /* PI-mutex try-acquire (no timeout) */
FUTEX_WAIT_BITSET      =  9    /* WAIT with 32-bit interest mask + abs timeout */
FUTEX_WAKE_BITSET      = 10    /* WAKE with 32-bit interest mask */
FUTEX_WAIT_REQUEUE_PI  = 11    /* WAIT then REQUEUE onto PI-futex (condvar) */
FUTEX_CMP_REQUEUE_PI   = 12    /* CMP_REQUEUE onto PI-futex */
FUTEX_LOCK_PI2         = 13    /* PI-mutex acquire with CLOCK_MONOTONIC timeout */
```

`FUTEX_FD` is reserved-but-disabled — the `kernel/futex/syscalls.c` dispatcher returns `-EINVAL` (treat as removed in the ABI). `FUTEX_LOCK_PI2` exists because `FUTEX_LOCK_PI` is permanently bound to CLOCK_REALTIME (a historical mistake); new code targeting CLOCK_MONOTONIC must use `FUTEX_LOCK_PI2`.

### `op`-modifier flags (`uapi/linux/futex.h:26-28`)

```text
FUTEX_PRIVATE_FLAG     = 128       /* 0x80: process-private; hash by mm_struct + uaddr */
FUTEX_CLOCK_REALTIME   = 256       /* 0x100: timeout interpreted in CLOCK_REALTIME */
FUTEX_CMD_MASK         = ~(FUTEX_PRIVATE_FLAG | FUTEX_CLOCK_REALTIME)   /* extract op */
```

The full operation argument to `futex(2)` is `op | flags`. `FUTEX_CMD_MASK` extracts the bare op. The `_PRIVATE` aliases in lines 30-44 are convenience composites (`FUTEX_WAIT_PRIVATE = FUTEX_WAIT | FUTEX_PRIVATE_FLAG`, etc.).

### `FUTEX_OP()` packing (`uapi/linux/futex.h:187-210`)

```text
FUTEX_OP_SET   = 0   /* *(int *)uaddr2  = oparg          */
FUTEX_OP_ADD   = 1   /* *(int *)uaddr2 += oparg          */
FUTEX_OP_OR    = 2   /* *(int *)uaddr2 |= oparg          */
FUTEX_OP_ANDN  = 3   /* *(int *)uaddr2 &= ~oparg         */
FUTEX_OP_XOR   = 4   /* *(int *)uaddr2 ^= oparg          */

FUTEX_OP_OPARG_SHIFT = 8   /* OR'd into op: use (1 << oparg) for oparg */

FUTEX_OP_CMP_EQ = 0    /* wake if oldval == cmparg */
FUTEX_OP_CMP_NE = 1
FUTEX_OP_CMP_LT = 2
FUTEX_OP_CMP_LE = 3
FUTEX_OP_CMP_GT = 4
FUTEX_OP_CMP_GE = 5

#define FUTEX_OP(op, oparg, cmp, cmparg)                            \
    (((op   & 0xf) << 28) | ((cmp & 0xf) << 24) |                   \
     ((oparg & 0xfff) << 12) | (cmparg & 0xfff))
```

`FUTEX_WAKE_OP` semantics (atomic, on `uaddr2`):

```text
oldval = *(int *)uaddr2;
*(int *)uaddr2 = oldval OP OPARG;   /* OP is FUTEX_OP_* above; if FUTEX_OP_OPARG_SHIFT, OPARG := (1 << OPARG) */
if (oldval CMP CMPARG)              /* CMP is FUTEX_OP_CMP_* above */
    wake_up_n(uaddr2, val2);        /* val2 waiters */
wake_up_n(uaddr, val);              /* val waiters always */
```

### futex2 flag bits (`uapi/linux/futex.h:47-75`)

```text
FUTEX2_SIZE_U8   = 0x00     /* 8-bit futex word */
FUTEX2_SIZE_U16  = 0x01     /* 16-bit futex word */
FUTEX2_SIZE_U32  = 0x02     /* 32-bit futex word (classic) */
FUTEX2_SIZE_U64  = 0x03     /* 64-bit futex word */
FUTEX2_NUMA      = 0x04     /* word followed by a u32 node id; size doubles */
FUTEX2_MPOL      = 0x08     /* honor mempolicy when hashing */
FUTEX2_PRIVATE   = FUTEX_PRIVATE_FLAG = 0x80

FUTEX2_SIZE_MASK = 0x03

FUTEX_32 = FUTEX2_SIZE_U32   /* historical alias; do not use in new code */
```

The futex2 flags word is logically:

```text
union {
    u32 flags;
    struct {
        u32 size : 2,   /* FUTEX2_SIZE_* */
            numa : 1,
                 : 4,
            private : 1;
    };
};
```

### `FUTEX_NO_NODE` (`uapi/linux/futex.h:82`)

```text
FUTEX_NO_NODE = -1    /* when FUTEX2_NUMA: the second u32 of the word may be -1 to mean "no node preference" */
```

`FUTEX_NO_NODE` is part of the futex2 ABI even though it coincides numerically with the kernel-internal `NUMA_NO_NODE`; UAPI value is independent and stable.

### `struct futex_waitv` (`uapi/linux/futex.h:89-101`)

```text
FUTEX_WAITV_MAX = 128

struct futex_waitv {
    __u64 val;          /* expected value at uaddr */
    __u64 uaddr;        /* user address (carried as u64 for 32/64-bit symmetry) */
    __u32 flags;        /* FUTEX2_* flags (size, numa, private) */
    __u32 __reserved;   /* MUST be 0 */
};
```

A `futex_waitv(2)` call passes a pointer to an array of up to `FUTEX_WAITV_MAX` such structs. Each waiter is independently sized (per `flags.size`); the kernel computes the futex word at `uaddr` per size, compares with `val`, and either sleeps (if all match) or returns the index of the first mismatch.

### Robust-list ABI (`uapi/linux/futex.h:117-154`)

```text
struct robust_list {
    struct robust_list __user *next;     /* forward link only; userspace double-links */
};

struct robust_list_head {
    struct robust_list list;
    long futex_offset;                                 /* offset from list-entry to futex word */
    struct robust_list __user *list_op_pending;       /* lock being acquired or released */
};

ROBUST_LIST_LIMIT = 2048                /* max links the kernel will walk */
```

`set_robust_list(head, len)` registers the head pointer; the kernel walks it on thread-exit and, for each owned lock (futex TID == exiter), sets `FUTEX_OWNER_DIED` plus broadcasts `FUTEX_WAKE` to one waiter.

### Robust-futex word bits (`uapi/linux/futex.h:159-172`)

```text
FUTEX_WAITERS    = 0x80000000   /* bit 31: contended; kernel-side waiter present */
FUTEX_OWNER_DIED = 0x40000000   /* bit 30: thread exited holding this lock */
FUTEX_TID_MASK   = 0x3fffffff   /* bits 0-29: TID of current owner */
```

A PI futex word is `(WAITERS_bit << 31) | (OWNER_DIED_bit << 30) | TID`. The kernel and the userspace fast-path cooperate on these bits via atomic compare-and-swap.

### `FUTEX_BITSET_MATCH_ANY` (`uapi/linux/futex.h:184`)

```text
FUTEX_BITSET_MATCH_ANY = 0xffffffff
```

`FUTEX_WAIT_BITSET` requires `bitset != 0`; passing `FUTEX_BITSET_MATCH_ANY` (all bits set) is the standard "match any waker" usage. The kernel enforces `bitset != 0` and returns `-EINVAL` otherwise.

### compatibility contract

REQ-1: `futex(uaddr, op, val, timeout, uaddr2, val3)` is the canonical syscall (NR_futex per arch). `op` upper bits MAY include `FUTEX_PRIVATE_FLAG` (0x80) and/or `FUTEX_CLOCK_REALTIME` (0x100); the bare op is `op & FUTEX_CMD_MASK`.

REQ-2: Op codes `FUTEX_WAIT=0 .. FUTEX_LOCK_PI2=13` MUST match the table verbatim. `FUTEX_FD=2` MUST return `-EINVAL` (historical disabled — kept reserved). Any op > `FUTEX_LOCK_PI2` MUST return `-ENOSYS`.

REQ-3: `FUTEX_PRIVATE_FLAG == 0x80`, `FUTEX_CLOCK_REALTIME == 0x100`. `FUTEX_CMD_MASK == ~(0x80 | 0x100)`. Implementations MUST honor `FUTEX_PRIVATE_FLAG` by hashing per-(mm, uaddr) instead of per-(inode, offset), eliminating the page-lookup fast-path tax for process-private futexes.

REQ-4: `FUTEX_CLOCK_REALTIME` is only valid with `FUTEX_WAIT_BITSET`, `FUTEX_WAIT_REQUEUE_PI`, and `FUTEX_LOCK_PI`. Combined with other ops MUST return `-ENOSYS`. `FUTEX_LOCK_PI2` always uses CLOCK_MONOTONIC.

REQ-5: `FUTEX_WAIT` uses a *relative* `timeout`; `FUTEX_WAIT_BITSET` uses an *absolute* `timeout`. `NULL` timeout means "forever".

REQ-6: `FUTEX_WAKE` returns the number of waiters actually woken (>=0), capped at `val`. `INT_MAX` as `val` means "wake all".

REQ-7: `FUTEX_REQUEUE` is the deprecated unconditional form; `FUTEX_CMP_REQUEUE` is the safe form requiring `*uaddr == val3` (validated atomically by the kernel). New code MUST NOT use bare `FUTEX_REQUEUE` (it has a TOCTOU race against userspace mutators).

REQ-8: `FUTEX_WAKE_OP` `val3` MUST be the packed `FUTEX_OP(op, oparg, cmp, cmparg)` value: bits 28-31 = op, 24-27 = cmp, 12-23 = oparg, 0-11 = cmparg. `op` MAY OR in `FUTEX_OP_OPARG_SHIFT` (bit 3 of the 4-bit op field) to indicate `oparg := (1 << oparg)`.

REQ-9: `FUTEX_OP_*` values: SET=0, ADD=1, OR=2, ANDN=3, XOR=4. `FUTEX_OP_CMP_*`: EQ=0, NE=1, LT=2, LE=3, GT=4, GE=5. Unknown op or cmp value MUST return `-ENOSYS`.

REQ-10: `FUTEX_LOCK_PI` / `_PI2` / `_TRYLOCK_PI` / `_UNLOCK_PI` / `_WAIT_REQUEUE_PI` / `_CMP_REQUEUE_PI` operate on the robust-futex word format (`FUTEX_TID_MASK` low bits, `FUTEX_WAITERS` bit 31, `FUTEX_OWNER_DIED` bit 30). A PI op on a word whose low 30 bits do not contain a live TID MUST return `-ESRCH`.

REQ-11: `FUTEX_TID_MASK == 0x3fffffff`. Implementations MUST verify that `*uaddr & FUTEX_TID_MASK` is a valid TID in the caller's PID namespace before granting PI inheritance; an invalid TID MUST return `-ESRCH`.

REQ-12: `FUTEX_WAITERS == 0x80000000` is set by the kernel (or by userspace racing through cmpxchg) when a waiter blocks; the kernel guarantees a subsequent `FUTEX_UNLOCK_PI` will wake a waiter while this bit is set.

REQ-13: `FUTEX_OWNER_DIED == 0x40000000` is set by the kernel during robust-list cleanup when an owning thread exits without unlocking. Userspace recovery code MUST either reset state and clear the bit (CAS) or treat the lock as poisoned.

REQ-14: `FUTEX_WAIT_BITSET` / `FUTEX_WAKE_BITSET` `bitset` MUST be nonzero; `bitset == 0` MUST return `-EINVAL`. `FUTEX_BITSET_MATCH_ANY == 0xffffffff` is the conventional "wake any" value.

REQ-15: `struct futex_waitv` layout MUST be exactly `{ u64 val; u64 uaddr; u32 flags; u32 __reserved; }` = 24 bytes, naturally 8-byte aligned. `__reserved` MUST be 0; nonzero MUST return `-EINVAL`.

REQ-16: `futex_waitv(2)` accepts an array of at most `FUTEX_WAITV_MAX == 128` waiters. `nr_futexes > FUTEX_WAITV_MAX` MUST return `-EINVAL`.

REQ-17: `struct futex_waitv.flags` is a `FUTEX2_*` flag mask. Bits 0-1 = size (`FUTEX2_SIZE_U8 .. _U64`); bit 2 = `FUTEX2_NUMA`; bit 3 = `FUTEX2_MPOL`; bit 7 = `FUTEX2_PRIVATE`. Unused bits MUST be 0; nonzero returns `-EINVAL`.

REQ-18: `FUTEX2_SIZE_U64` requires LP64 userspace (or a 64-bit aligned address that the arch supports for atomic-cmpxchg-8B). On 32-bit ABIs, `FUTEX2_SIZE_U64` MAY return `-EOPNOTSUPP`.

REQ-19: `FUTEX2_NUMA` doubles the futex word (a second u32 immediately following at `uaddr + size`). The second word is a node id; `FUTEX_NO_NODE == -1` means "no preference".

REQ-20: `struct robust_list` MUST be a single forward pointer. `struct robust_list_head` MUST contain `{ robust_list list; long futex_offset; robust_list __user *list_op_pending; }` in that order. Layout changes require a coordinated glibc bump (per upstream comment).

REQ-21: `set_robust_list(head, len)` requires `len == sizeof(struct robust_list_head)`; mismatch MUST return `-EINVAL`. `get_robust_list(pid, head_ptr, len_ptr)` requires `CAP_SYS_PTRACE` to read another thread's head.

REQ-22: Robust-list walking MUST terminate at `ROBUST_LIST_LIMIT == 2048` chain links to defeat deliberately circular lists; the kernel does NOT introduce an rlimit for this.

REQ-23: `FUTEX_OP_OPARG_SHIFT == 8` is a bit OR'd into the 4-bit `op` field of `FUTEX_OP(...)`. With shift active, `oparg` is interpreted as `(1 << oparg)` to allow up to 32-bit single-bit operations from a 12-bit oparg field.

REQ-24: `FUTEX_PRIVATE_FLAG` ABI value MUST equal `FUTEX2_PRIVATE` (`0x80`) to allow the futex2 flags word and the classic op-modifier to share a private-bit semantic.

REQ-25: All `uaddr` and `uaddr2` arguments MUST be naturally aligned to the futex size. Misaligned MUST return `-EINVAL`. For classic `FUTEX_WAIT` / `FUTEX_WAKE`, the size is implicitly 4 bytes.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `op_in_range` | INVARIANT | per-futex: `bare := op & FUTEX_CMD_MASK ∈ [0, FUTEX_LOCK_PI2]` else `-ENOSYS`. |
| `futex_fd_disabled` | INVARIANT | per-futex: bare op == FUTEX_FD ⟹ `-EINVAL`. |
| `clock_realtime_op_compat` | INVARIANT | per-futex: FUTEX_CLOCK_REALTIME only with WAIT_BITSET / WAIT_REQUEUE_PI / LOCK_PI. |
| `wait_bitset_nonzero` | INVARIANT | per-WAIT_BITSET/WAKE_BITSET: `bitset != 0` else `-EINVAL`. |
| `futex_waitv_reserved_zero` | INVARIANT | per-waitv: `__reserved != 0 ⟹ -EINVAL`. |
| `futex_waitv_size_24` | INVARIANT | `sizeof::<FutexWaitv>() == 24`. |
| `futex_waitv_max` | INVARIANT | per-waitv: `nr_futexes > 128 ⟹ -EINVAL`. |
| `futex2_flags_mask` | INVARIANT | per-waitv: `flags & !(SIZE_MASK \| NUMA \| MPOL \| PRIVATE) == 0`. |
| `uaddr_naturally_aligned` | INVARIANT | per-op: `uaddr & (size - 1) == 0` for chosen futex size. |
| `tid_mask_consistent` | INVARIANT | `FUTEX_TID_MASK \| FUTEX_WAITERS \| FUTEX_OWNER_DIED == 0xFFFFFFFF`. |
| `robust_list_limit_bounds_walk` | INVARIANT | per-exit-cleanup: walk halts at `ROBUST_LIST_LIMIT = 2048`. |
| `set_robust_list_size` | INVARIANT | per-set_robust_list: `len == sizeof(RobustListHead)` else `-EINVAL`. |
| `pi_tid_in_namespace` | INVARIANT | per-LOCK_PI: `*uaddr & FUTEX_TID_MASK` resolves in caller's PID ns or `-ESRCH`. |
| `op_pack_unpack_bijective` | INVARIANT | per-WAKE_OP: `FUTEX_OP(o,a,c,p)` then unpack ⟹ recover `(o,a,c,p)` (within 12-bit oparg / cmparg). |

### Layer 2: TLA+

`uapi/headers/futex.tla`:
- Per-WAIT/WAKE rendezvous on shared `uaddr`.
- Per-PI lock acquire/release with priority-inheritance chain.
- Per-WAIT_BITSET selective wake.
- Per-robust-list cleanup on owner death.
- Per-WAITV vector-wait wake-any.
- Properties:
  - `safety_wait_observes_value` — per-WAIT: returns 0 ⟺ either waker arrived OR `*uaddr` changed observably (`-EAGAIN`).
  - `safety_wake_never_overwakes` — per-WAKE: woken count ≤ `val`.
  - `safety_pi_inheritance_acyclic` — per-PI chain: priority-inheritance DAG never cycles.
  - `safety_owner_died_implies_wake` — per-thread-exit holding robust futex: a waiter is woken with `FUTEX_OWNER_DIED` set.
  - `safety_private_isolates_namespaces` — per-PRIVATE: a private futex in process A cannot be hit by a wake from process B.
  - `safety_wake_op_atomic` — per-WAKE_OP: write to `*uaddr2` is atomically observed with the wake decision.
  - `safety_waitv_one_wake_per_call` — per-WAITV: at most one waiter (the matching index) returns; others remain queued or are cancelled.
  - `liveness_wake_eventually_wakes` — per-pending WAITer + matching WAKEr: waiter eventually returns 0.
  - `liveness_pi_lock_progress` — per-PI: no priority inversion lasts longer than the boosted owner's runtime.
  - `liveness_robust_cleanup_terminates` — per-exit cleanup: walks ≤ ROBUST_LIST_LIMIT and finishes.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `FutexWaitv` layout: 24 bytes, 8-byte aligned, fields val/uaddr/flags/__reserved | `FutexWaitv` |
| `RobustListHead` layout: list/futex_offset/list_op_pending | `RobustListHead` |
| `sys_futex` post: op range checked; FUTEX_FD = EINVAL; >LOCK_PI2 = ENOSYS | `sys_futex` |
| `sys_futex_waitv` post: nr ≤ 128; `__reserved == 0`; flags bits valid; alignment respected | `sys_futex_waitv` |
| `sys_set_robust_list` post: len exact; assignment atomic | `sys_set_robust_list` |
| `wake_op` post: `*uaddr2` mutation atomic; cmp evaluated on `oldval` | `Futex::wake_op` |
| `lock_pi` post: TID valid in ns; FUTEX_WAITERS set on contention | `Futex::lock_pi` |
| `unlock_pi` post: caller TID matches `*uaddr & FUTEX_TID_MASK` else EPERM | `Futex::unlock_pi` |
| `robust-list walk` post: ≤ ROBUST_LIST_LIMIT links visited | `Futex::exit_cleanup` |

### Layer 4: Verus/Creusot functional

`Per futex(2) → op-decode → (wait | wake | requeue | wake_op | pi_op | waitv) → return` semantic equivalence: per-`futex(2)` man page, per-`kernel/futex/syscalls.c` upstream, per-`Documentation/futex2.txt`. `Per WAKE_OP atomic-op-then-wake` semantic equivalence: per-`kernel/futex/waitwake.c::futex_atomic_op_inuser` and per-the source-of-truth comment block in `uapi/linux/futex.h:202-206`. `Per robust-list thread-exit cleanup` semantic equivalence: per-`kernel/futex/core.c::exit_robust_list` walking `≤ROBUST_LIST_LIMIT` links and setting `FUTEX_OWNER_DIED`.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

Futex UAPI reinforcement:

- **Strict `op & FUTEX_CMD_MASK` range check** — defense against per-future-op silent activation.
- **`FUTEX_FD` disabled and rejected** — defense against per-historical-fd-leak (the original FUTEX_FD design was deemed unrecoverably racy and is permanently nailed shut).
- **`FUTEX_CLOCK_REALTIME` op compatibility check** — defense against per-realtime-clock-against-wrong-op misuse that could mis-interpret a relative timeout as absolute.
- **`FUTEX_TID_MASK` PID-namespace validation** — every PI op validates that the TID in the futex word resolves in the caller's PID namespace, defending against per-namespace TID confusion.
- **`*uaddr` page-fault path uses `get_user_pages`/`pin_user_pages` with NOFAULT first** — defense against per-faulting-user-page during PI inheritance.
- **`FUTEX_WAITV_MAX` = 128 cap** — defense against per-vector blow-up (a single syscall holding `nr_futexes` kernel hash-bucket locks).
- **`futex_waitv.__reserved == 0` enforced** — defense against per-future-extension silent activation through the reserved field.
- **`ROBUST_LIST_LIMIT` = 2048 walking cap** — defense against per-circular-list infinite loop during thread exit.
- **`FUTEX_PRIVATE_FLAG` hash isolation** — defense against per-cross-process futex spillover; a private futex in process A cannot be located by hashing in process B.
- **PI deadlock detection** — defense against per-PI chain cycle (chain ≤ kernel-set max_lock_depth).
- **`FUTEX_OP_CMP_*` evaluation on `oldval` not `newval`** — defense against per-WAKE_OP self-cancellation race.

### grsecurity/pax-style reinforcement

- **PaX UDEREF/USERCOPY on every `uaddr` / `uaddr2` access** — every load/store from a futex word goes through `get_user_atomic` / `put_user_atomic` with UDEREF active, forcing the access through SMAP/SMEP-enforced user-space mapping windows. Defense against per-kernel-pointer-passed-as-uaddr leaking arbitrary kernel memory through `FUTEX_WAIT` value disclosure.
- **PAX_RANDKSTACK at futex syscall entry** — randomizes the kernel-stack offset per `futex(2)` / `futex_waitv(2)` / `set_robust_list(2)` / `get_robust_list(2)` entry so that any disclosure of stack-resident `futex_q` or `futex_pi_state` pointers (e.g., via a PI inheritance leak) is statistically useless across attempts.
- **GRKERNSEC_BRUTE on futex flood** — a uid spamming `FUTEX_WAIT` / `FUTEX_WAKE` / `FUTEX_LOCK_PI` above a per-uid budget triggers progressive throttling and ultimately SIGKILL; defeats brute-force exploits of futex-state oracles and `EAGAIN`-vs-`ETIMEDOUT` timing side-channels.
- **FUTEX_PRIVATE_FLAG default mandatory for unprivileged processes** — Rookery PAX policy refuses *shared* (non-private) futex operations from a uid that has not been granted a `cap_ipc_lock`-like capability, eliminating cross-process futex hashing as an information channel.
- **FUTEX_TID_MASK validation cross-checked against PID-namespace + cgroup** — the TID inside the futex word must not only resolve in the caller's PID ns but also belong to the same cgroup leaf as the caller; cross-cgroup PI inheritance is rejected with `-EPERM`. Stops a low-priority container task from PI-boosting a host task.
- **`FUTEX_OWNER_DIED` recovery requires `CAP_SYS_PTRACE` or matching uid** — only a process that can `ptrace(2)` the dead owner (or shares the owner's uid) is allowed to atomically clear `FUTEX_OWNER_DIED` and adopt the lock. Defeats opportunistic robust-futex hijack by a same-mm but different-uid thread.
- **`futex_waitv` array copied into kernel buffer with size-bounded `copy_from_user`** — the `nr_futexes * sizeof(FutexWaitv)` copy uses `array_size`-checked arithmetic; overflow returns `-EOVERFLOW`. PaX `kalloca`-style stack-bound check refuses heap allocations > 64 KiB even when the cap allows it.
- **GRKERNSEC_PROC_USER hides `/proc/<pid>/syscall` futex-arg traces** — when a process is blocked in `futex_wait_queue`, `/proc/<pid>/syscall` and `/proc/<pid>/stack` redact the `uaddr` and `val` arguments from non-same-uid readers, eliminating "see what futex address the target is waiting on" reconnaissance.
- **PAX_MPROTECT prevents `mprotect`-after-futex on a writable+executable futex word** — if a futex word lives in a VMA that gets transitioned to W^X violation territory (e.g., JIT trampoline page), the kernel revokes any outstanding `futex_q` for that VMA. Defeats per-W^X-bypass via JIT-staging exploits that try to leave a wake-anchor in code memory.
- **PI chain depth capped per-task by `RLIMIT_RTPRIO` + grsec multiplier** — a PI chain may not boost more than `rt_prio * grsec_pi_factor` tasks, preventing a single unprivileged task from creating a deep priority-inheritance chain that monopolises CPU through transitive boosting.
- **Robust-list head must lie in writable, non-executable VMA owned by the caller's mm** — `set_robust_list(head, len)` is rejected if `head` falls in a non-`VM_WRITE` or `VM_EXEC` VMA. Defeats per-`set_robust_list`-as-write-primitive that the original Will Drewry / Tavis Ormandy CVE-2014-3153 family of bugs exploited.
- **Cross-namespace `get_robust_list(2)` requires CAP_SYS_PTRACE in the *target's* user namespace** — defeats per-namespace-leak where a privileged outer-ns process could read inner-ns task robust-list head pointers.

