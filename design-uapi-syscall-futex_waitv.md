---
title: "Tier-5 syscall: futex_waitv(2) — syscall 449"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`futex_waitv(2)` is the **multi-futex wait** primitive — wait on any of N futex addresses, returning when any one of them is awakened, or when a timeout elapses, or when a signal arrives. Designed to subsume `select`/`poll`-style "wait on N independent condition variables" patterns currently emulated by glibc / proton / wine using helper threads plus single `futex(WAIT)` per variable. Each waiter element selects 8/16/32-bit operand size and per-key private/shared mode independently via `FUTEX2_*` flags. The syscall is named "v" by analogy with `readv`/`writev`. Critical for: Wine/Proton fsync compatibility, low-latency event-merging in game engines and database message queues, modern `mutex`/`condvar` runtimes that batch multiple wait conditions.

This Tier-5 covers the userspace ABI of syscall 449.

### Acceptance Criteria

- [ ] AC-1: `nr_futexes == 0` → `-EINVAL`.
- [ ] AC-2: `nr_futexes == 129` → `-EINVAL`.
- [ ] AC-3: `flags == 1` → `-EINVAL`.
- [ ] AC-4: `clockid == CLOCK_BOOTTIME` → `-EINVAL`.
- [ ] AC-5: `__reserved` non-zero in any element → `-EINVAL`.
- [ ] AC-6: Element with unknown `flags` bit → `-EINVAL`.
- [ ] AC-7: Element with `SIZE_U16` and `uaddr` odd → `-EINVAL`.
- [ ] AC-8: Three elements; `*uaddr[1] != val[1]` at queue time → `-EAGAIN`.
- [ ] AC-9: Three elements; concurrent `futex_wake(uaddr[2])` → returns `2`.
- [ ] AC-10: Three elements; signal → `-EINTR`; all dequeued.
- [ ] AC-11: Past-due absolute timeout → immediate `-ETIMEDOUT`.
- [ ] AC-12: `timeout == NULL` ∧ no wake ∧ no signal → blocks indefinitely (test bounded by external SIGTERM).
- [ ] AC-13: `SIZE_U8` element: `val > 0xff` → `-EINVAL`.
- [ ] AC-14: `FUTEX2_PRIVATE` element wakes only by same-mm wake.
- [ ] AC-15: `FUTEX2_NUMA` element queued on local-node bucket; wakes correctly.

### Architecture

Rookery surface in `kernel/futex/syscall_waitv.rs`:

```rust
#[repr(C, align(8))]
pub struct FutexWaitv {
    pub val:        u64,
    pub uaddr:      u64,
    pub flags:      u32,
    pub _reserved:  u32,
}

pub const FUTEX_WAITV_MAX: u32 = 128;

bitflags! {
    pub struct Futex2Flags: u32 {
        const SIZE_U8  = 0;
        const SIZE_U16 = 1;
        const SIZE_U32 = 2;
        const SIZE_U64 = 3;
        const NUMA     = 1 << 2;
        const PRIVATE  = 1 << 3;
        const SIZE_MASK = 0x3;
        const ALL = 0x3 | (1 << 2) | (1 << 3);
    }
}
```

`Futex::waitv(waiters_uptr, nr, flags, ts_uptr, clockid) -> isize`:
1. /* top-level flags reserved */
2. if flags != 0 { return -EINVAL; }
3. /* nr bounds */
4. if nr == 0 || nr > FUTEX_WAITV_MAX { return -EINVAL; }
5. /* clockid */
6. if clockid != CLOCK_REALTIME && clockid != CLOCK_MONOTONIC { return -EINVAL; }
7. /* copy waitv array */
8. let mut v: SmallVec<FutexWaitv, 16> = copy_from_user_array(waiters_uptr, nr).map_err(|_| -EFAULT)?;
9. /* per-element validate */
10. for w in &v {
     - if w._reserved != 0 { return -EINVAL; }
     - if w.flags & !Futex2Flags::ALL.bits() != 0 { return -EINVAL; }
     - let sz = w.flags & Futex2Flags::SIZE_MASK.bits();
     - let align_req = 1 << sz; /* 1,2,4,8 */
     - if (w.uaddr % align_req as u64) != 0 { return -EINVAL; }
     - if !validate_val_upper_bits(w.val, sz) { return -EINVAL; }
   }
11. /* copy timeout */
12. let deadline = if ts_uptr.is_null() {
      - None
    } else {
      - let ts = copy_from_user::<KernelTimespec>(ts_uptr).map_err(|_| -EFAULT)?;
      - if !ts.is_valid() { return -EINVAL; }
      - Some((clockid, ts))
    };
13. /* snapshot-and-queue */
14. let mut queued: SmallVec<FutexQ, 16> = SmallVec::new();
15. for (i, w) in v.iter().enumerate() {
      - let key = get_futex_key2(w.uaddr, w.flags)?;
      - let cur = atomic_load_user_sized(w.uaddr, w.flags & SIZE_MASK)?;
      - if cur != truncate(w.val, w.flags & SIZE_MASK) {
        - dequeue_all(&queued);
        - return -EAGAIN;
      - }
      - let q = FutexQ { idx: i as u32, key, task: current, woken: AtomicBool::new(false) };
      - hb = hash_futex(key); hb.lock(); hb.queue(&q); hb.unlock();
      - queued.push(q);
    }
16. /* sleep until any wake / signal / timeout */
17. let res = wait_for_any(&queued, deadline);
18. /* atomic dequeue all */
19. dequeue_all(&queued);
20. match res {
    - WokenIdx(i) => return i as isize,
    - Signal      => return -EINTR,
    - Timeout     => return -ETIMEDOUT,
    }

`wait_for_any(queued, deadline)`:
1. loop {
   - if any queued.q.woken.load(Acquire) { return WokenIdx(q.idx); }
   - if signal_pending() { return Signal; }
   - if let Some(d) = deadline { if now(d.0) ≥ d.1 { return Timeout; } }
   - park(deadline);
 }

`dequeue_all(queued)`:
1. for q in queued: hash_futex(q.key).lock_remove(q).

### Out of Scope

- `kernel/futex.md` Tier-3 — hash-bucket internals, PI, robust list
- `futex.md` sibling — classic single-futex syscall
- `futex_wake.md` / `futex_wait.md` — futex2 single-key variants (syscalls 454, 455)
- `futex_requeue.md` — futex2 requeue (syscall 456)
- Wine / Proton fsync user-side
- Implementation code

### signature

```c
long futex_waitv(struct futex_waitv *waiters,
                 unsigned int nr_futexes,
                 unsigned int flags,
                 struct __kernel_timespec *timeout,
                 clockid_t clockid);
```

Rust ABI shim:

```rust
pub fn sys_futex_waitv(waiters: *mut FutexWaitv,
                       nr_futexes: u32,
                       flags: u32,
                       timeout: *const __kernel_timespec,
                       clockid: i32) -> isize;
```

Syscall number: **449**.

### parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `waiters` | `struct futex_waitv *` | IN | array of per-element wait specs |
| `nr_futexes` | `u32` | IN | element count, `1 ≤ nr ≤ FUTEX_WAITV_MAX = 128` |
| `flags` | `u32` | IN | reserved; must be `0` |
| `timeout` | `const __kernel_timespec *` | IN | absolute timeout on `clockid`; `NULL` ⟹ no timeout |
| `clockid` | `i32` | IN | `CLOCK_REALTIME` or `CLOCK_MONOTONIC` |

`struct futex_waitv` (UAPI, fixed 32-byte layout):

```c
struct futex_waitv {
    __u64 val;        /* expected value */
    __u64 uaddr;      /* user pointer to futex */
    __u32 flags;      /* FUTEX2_SIZE_* | FUTEX2_PRIVATE | FUTEX2_NUMA */
    __u32 __reserved; /* MUST be 0 */
};
```

`FUTEX2_*` flags (per-element):

| Constant | Value | Purpose |
|---|---|---|
| `FUTEX2_SIZE_U8` | `0` | 8-bit futex |
| `FUTEX2_SIZE_U16` | `1` | 16-bit futex |
| `FUTEX2_SIZE_U32` | `2` | 32-bit futex (compat with classic `futex`) |
| `FUTEX2_SIZE_U64` | `3` | 64-bit futex |
| `FUTEX2_SIZE_MASK` | `3` | size discriminant mask |
| `FUTEX2_NUMA` | `1 << 2` | enable per-NUMA-node bucket steering |
| `FUTEX2_PRIVATE` | `1 << 3` | private (per-mm) key (equivalent to classic `FUTEX_PRIVATE_FLAG`) |

### return

- **Success**: non-negative index `i` (`0 ≤ i < nr_futexes`) of the waiter that was woken.
- **Failure**: `-1` and `errno`; Rust internal returns negated errno.

### errors

| errno | Trigger |
|---|---|
| `EAGAIN` | per-element `*uaddr != val` was observed during pre-queue snapshot — userspace retries fast path |
| `EINTR` | wait interrupted by signal before any element woken |
| `EINVAL` | `nr_futexes == 0` or `> FUTEX_WAITV_MAX`; per-element `__reserved != 0`; per-element invalid `FUTEX2_SIZE_*`; non-zero `flags`; bad `clockid` (not REALTIME / MONOTONIC); per-element `uaddr` not aligned to its size |
| `EFAULT` | `waiters` not user-readable; per-element `uaddr` not user-readable; `timeout` not readable |
| `ETIMEDOUT` | absolute timeout elapsed before any element woken |
| `ENOSYS` | unsupported on this build |
| `ENOMEM` | per-waitv array allocation failed |
| `EOVERFLOW` | timeout values overflow internal representation |

### abi surface

Constants:

| Constant | Value | Purpose |
|---|---|---|
| `FUTEX_WAITV_MAX` | `128` | max waiters per call |
| `CLOCK_REALTIME` | `0` | wall-clock timeout |
| `CLOCK_MONOTONIC` | `1` | monotonic timeout |

ABI rules:
- `struct futex_waitv` is **exactly 32 bytes** with explicit alignment; padding zero.
- `__reserved` is forward-compatibility reserved; non-zero ⟹ `-EINVAL`.
- `flags` (syscall-level) is forward-compatibility reserved; non-zero ⟹ `-EINVAL`.
- `timeout` is **always absolute** when present (no per-call relative option, unlike `futex(2)`).
- `clockid` is restricted to `CLOCK_REALTIME` and `CLOCK_MONOTONIC` only.

### compatibility contract

REQ-1: `nr_futexes` bounds:
- `nr_futexes == 0` → `-EINVAL`.
- `nr_futexes > FUTEX_WAITV_MAX` → `-EINVAL`.

REQ-2: Top-level `flags`:
- MUST be zero (reserved).

REQ-3: `clockid` validation:
- MUST be `CLOCK_REALTIME` or `CLOCK_MONOTONIC`; else `-EINVAL`.

REQ-4: `timeout` semantics:
- `timeout == NULL`: no deadline; wait indefinitely.
- `timeout != NULL`: absolute on `clockid`; in the past ⟹ `-ETIMEDOUT` immediately.
- `tv_nsec` validation: must be `[0, 999_999_999]`.

REQ-5: Per-element copy:
- All `nr_futexes` entries copied from user atomically (kernel copies into bounded stack/slab buffer); failure ⟹ `-EFAULT`.

REQ-6: Per-element `flags`:
- Size bits MUST be `0..=3`.
- `FUTEX2_NUMA | FUTEX2_PRIVATE` may be combined with size.
- Bits outside `SIZE_MASK | NUMA | PRIVATE` ⟹ `-EINVAL`.

REQ-7: Per-element `__reserved`:
- MUST be zero else `-EINVAL`.

REQ-8: Per-element `uaddr`:
- Alignment to size (`u8` ≥ 1, `u16` ≥ 2, `u32` ≥ 4, `u64` ≥ 8); misalignment ⟹ `-EINVAL`.
- Readable in user address space; else `-EFAULT`.

REQ-9: Per-element `val`:
- Compared zero-extended against atomic load of `*uaddr` truncated to its size.
- For `SIZE_U8`: low 8 bits of `val` compared; high bits MUST be zero else `-EINVAL`.
- For `SIZE_U16`: low 16 bits; high bits MUST be zero.
- For `SIZE_U32`: low 32 bits; high bits MUST be zero.

REQ-10: Snapshot-then-queue:
- For each element: atomic load `*uaddr`; if `!= val` → return `-EAGAIN`.
- Else queue waiter on per-element bucket.
- After all queued, sleep.
- One wake on any element ⟹ return that element's index.

REQ-11: Concurrent wake from other path:
- A `FUTEX_WAKE` / `FUTEX_WAKE_BITSET` / `futex_wake` (futex2) on any of the element's keys wakes the multi-wait.
- The kernel returns the index of the **first** awakened element if multiple wake concurrently.

REQ-12: Signal:
- Pending signal aborts wait → `-EINTR`.
- All queued elements dequeued atomically.

REQ-13: Timeout:
- Per `clockid` absolute deadline; arrives → `-ETIMEDOUT`.

REQ-14: Cancellation cleanup:
- On any return path (success / signal / timeout / fault), all elements dequeued.

REQ-15: `FUTEX2_NUMA`:
- Routes the waiter to the per-NUMA-node bucket matching the page's home node.
- Optimization, not required for correctness.

REQ-16: Forward-compat:
- Kernel that does not support a future `FUTEX2_*` bit returns `-EINVAL` rather than silently ignoring.

REQ-17: No `WAKE`-equivalent variant:
- `futex_waitv` has no symmetric `futex_wakev`; wake-side is per-element classic `futex(WAKE)` or `futex_wake` (futex2 entry, syscall 454).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `nr_bounds` | INVARIANT | `nr ∈ [1, 128]` else -EINVAL |
| `flags_zero` | INVARIANT | top-level flags != 0 ⟹ -EINVAL |
| `reserved_zero` | INVARIANT | per-element __reserved != 0 ⟹ -EINVAL |
| `clockid_subset` | INVARIANT | clockid ∉ {REALTIME, MONOTONIC} ⟹ -EINVAL |
| `align_per_size` | INVARIANT | uaddr % (1 << size) != 0 ⟹ -EINVAL |
| `val_upper_zero` | INVARIANT | val high-bits > size width ⟹ -EINVAL |
| `dequeue_on_any_return` | INVARIANT | every return path dequeues all queued waiters |
| `snapshot_consistent` | INVARIANT | eagain implies dequeue-all before return |

### Layer 2: TLA+

`uapi/futex_waitv.tla`:
- Variables: `queued_set`, `wake_event`, `deadline`, `signal_pending`.
- Properties:
  - `safety_no_partial_queue` — return with -EAGAIN ⟹ queued_set empty.
  - `safety_dequeue_on_terminate` — every termination path empties queued_set.
  - `safety_index_match` — returned idx corresponds to an element that observed wake.
  - `liveness_terminate` — every call terminates on wake / signal / timeout / EAGAIN.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `waitv` post(ok): `0 ≤ ret < nr` | `Futex::waitv` |
| `waitv` post: queued_set empty | `Futex::waitv` |
| `wait_for_any` post: woken-by index matches return | `wait_for_any` |
| `dequeue_all` post: every queued waiter removed | `dequeue_all` |

### Layer 4: Verus/Creusot functional

Per-`futex_waitv(2)` man page, `Documentation/userspace-api/futex2.rst`, Wine `dlls/ntdll/unix/sync.c`, Proton fsync emulation, glibc `nptl/futex_waitv.c`. Per-LTP `testcases/kernel/syscalls/futex/futex_waitv*.c` round-trip.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

futex_waitv reinforcement:

- **Per-call bounded array copy (≤ 128)** — defense against per-large-allocation DoS.
- **Per-element strict size + alignment** — defense against per-misaligned-load fault.
- **Per-clockid restricted** — defense against per-clock-source confusion.
- **Per-return dequeue invariant** — defense against per-stale-queue UAF.
- **Per-snapshot atomic-load** — defense against per-torn-load race.

### grsecurity/pax-style reinforcement

- **PAX_RANDKSTACK on futex_waitv entry** — randomize kernel stack at every multi-wait entry; futex_waitv is increasingly used in latency-sensitive code (Wine, Proton, modern runtimes) and the higher syscall frequency is matched by higher value as a layout-discovery target.
- **PaX UDEREF on `copy_from_user` of the `struct futex_waitv` array, timeout, and every per-element `uaddr`** — enforces unsigned-only userspace deref across the entire multi-wait surface; the array-copy primitive is a type-confusion attack vector when an attacker shapes the array to span SLAB cache boundaries.
- **GRKERNSEC_BPF_HARDEN considerations** — futex_waitv is structurally a constrained mini-VM over up to 128 user-controlled keys; apply BPF-style hardening: bound `FUTEX_WAITV_MAX` to a per-uid policy lower than 128 (e.g., 32 for unprivileged); audit every call with `nr_futexes` outside a small range.
- **CAP_BPF gating not appropriate** — futex_waitv is a baseline runtime primitive; instead apply per-uid quota: cap simultaneous multi-wait calls per uid and rate-limit.
- **GRKERNSEC_FIFO scaled to futex flood (multi-wait variant)** — per-uid quota on simultaneous active waitv calls and per-uid rate-limit; refuses with `-EAGAIN` past quota; eliminates a particularly nasty DoS variant where a single task ties up 128 hash buckets per call by repeatedly queuing.
- **PaX UDEREF on per-element atomic load** — the per-element pre-queue load of `*uaddr` must respect UDEREF semantics; refuse kernel-mapped pages even when `FUTEX2_PRIVATE` is set.
- **Bounded per-call hash-bucket footprint** — refuse `futex_waitv` calls whose elements would touch more than N (configurable, default 32) distinct hash buckets per call under hardened policy; defense against per-bucket-saturation DoS.
- **GRKERNSEC_HIDESYM on `FUTEX2_NUMA` reflection** — the per-NUMA bucket selection observable through wake-latency timing leaks NUMA topology; under `kernel.kptr_restrict ≥ 2`, the kernel disables NUMA-routing for unprivileged tasks (transparent fallback to non-NUMA buckets).
- **Forward-compat strict rejection** — unknown `FUTEX2_*` bits or non-zero top-level `flags` MUST return `-EINVAL` (never "silent ignore"); defense against per-future-flag confusion exploit.
- **EAGAIN/EINTR audit pattern detection** — repeated `EAGAIN` cycles on the same waitv array signal a torn-write or livelock probe; rate-limit + log per uid.
- **Deadline-in-the-past handling explicit** — past-due absolute timeout returns `-ETIMEDOUT` without ever queuing; closes a race between queue and immediate-timeout dequeue.

