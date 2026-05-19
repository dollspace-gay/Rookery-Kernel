---
title: "Tier-5 syscall: futex_requeue(2) — syscall 456"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`futex_requeue(2)` is the futex2 counterpart to legacy `FUTEX_CMP_REQUEUE`. It wakes up to `nr_wake` waiters on the source futex `waiters[0]` and then atomically migrates up to `nr_requeue` of the remaining waiters from the source bucket to the target futex bucket described by `waiters[1]`. A guard value comparison (the futex2 form of `CMP_REQUEUE`) is performed under the source bucket lock to detect concurrent modification.

The two-element `struct futex_waitv` argument bundles the source and target futexes — including independent address, expected value, mask, and flags for each — so requeue across element-size or PRIVATE/SHARED boundaries is meaningfully validated.

Critical for: pthread_cond_broadcast (wake one, requeue rest to the mutex), POSIX condvar fairness, Wine/Proton multi-event waits.

### Acceptance Criteria

- [ ] AC-1: 4 source waiters, `nr_wake=1, nr_requeue=2`: 1 wakes, 2 move to target, 1 stays on source. Return 3.
- [ ] AC-2: Compare mismatch returns `-EAGAIN`, source untouched.
- [ ] AC-3: `flags != 0` returns `-EINVAL`.
- [ ] AC-4: Source key == target key returns `-EINVAL`.
- [ ] AC-5: Size mismatch between entries returns `-EINVAL`.
- [ ] AC-6: `nr_wake = 0, nr_requeue = 0` with compare hit returns 0; with miss returns `-EAGAIN`.
- [ ] AC-7: Misaligned `uaddr` on either entry returns `-EINVAL`.
- [ ] AC-8: `__reserved` non-zero on either entry returns `-EINVAL`.
- [ ] AC-9: pthread_cond_broadcast pattern (wake 1, requeue all to mutex) survives stress.
- [ ] AC-10: Requeue preserves each migrated waiter's bitset.

### Architecture

```rust
#[syscall(nr = 456, abi = "sysv")]
pub fn sys_futex_requeue(
    waiters: UserPtr<[FutexWaitV; 2]>,
    flags: u32,
    nr_wake: i32,
    nr_requeue: i32,
) -> isize {
    Futex2::requeue(waiters, flags, nr_wake, nr_requeue)
}
```

`Futex2::requeue(waiters_ptr, flags, nr_wake, nr_requeue) -> isize`:
1. if flags != 0 { return Err(EINVAL); }
2. if nr_wake < 0 || nr_requeue < 0 { return Err(EINVAL); }
3. let waiters: [FutexWaitV; 2] = waiters_ptr.read()?;          // EFAULT
4. Futex2::validate_waitv_entry(&waiters[0])?;
5. Futex2::validate_waitv_entry(&waiters[1])?;
6. let fs = Futex2Flags::parse(waiters[0].flags)?;
7. let ft = Futex2Flags::parse(waiters[1].flags)?;
8. if fs.element_size() != ft.element_size() { return Err(EINVAL); }
9. let key_s = Futex2::get_key(UserPtr::from_u64(waiters[0].uaddr), fs)?;
10. let key_t = Futex2::get_key(UserPtr::from_u64(waiters[1].uaddr), ft)?;
11. if key_s == key_t { return Err(EINVAL); }
12. let bs = Futex2::hash_bucket(&key_s);
13. let bt = Futex2::hash_bucket(&key_t);
14. let (first, second) = order_locks(bs, bt);
15. first.with_lock(|_| second.with_lock(|_| {
16.   let actual = Futex2::read_user_atomic(UserPtr::from_u64(waiters[0].uaddr), fs.element_size())?;
17.   if actual != (waiters[0].val as u64).truncate_to(fs.element_size()) {
18.     return Err(EAGAIN);
19.   }
20.   let woken = Futex2::wake_n(bs, &key_s, nr_wake as u32);
21.   let moved = Futex2::move_n(bs, bt, &key_s, &key_t, nr_requeue as u32);
22.   Ok((woken + moved) as isize)
23. }))

`Futex2::move_n(src, dst, key_s, key_t, n) -> u32`:
1. let mut count = 0u32;
2. while count < n {
3.   let Some(w) = src.pop_first_matching(key_s) else { break; };
4.   w.key = key_t.clone();
5.   dst.push(w);
6.   count += 1;
7. }
8. count

### Out of Scope

- `futex_wake(2)` (see `futex_wake.md`).
- `futex_wait(2)` (see `futex_wait.md`).
- `futex_waitv(2)` (variable-length vector wait; Tier-5 separately).
- PI-requeue (Tier-3 `kernel/futex/pi.md`).
- Implementation code.

### signature

```c
long futex_requeue(
    struct futex_waitv *waiters,   /* exactly two entries: [0]=source, [1]=target */
    unsigned int flags,            /* requeue-level flags (currently must be 0) */
    int nr_wake,                   /* max waiters to wake on source */
    int nr_requeue                 /* max waiters to migrate to target */
);
```

```c
struct futex_waitv {
    __u64 val;
    __u64 uaddr;
    __u32 flags;     /* per-futex FUTEX2_* */
    __u32 __reserved;
};
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `waiters` | `struct futex_waitv *` | in | Pointer to **exactly two** `struct futex_waitv` entries. `[0]` describes the source; `[1]` the target. |
| `flags` | `unsigned int` | in | Top-level requeue flags. Currently MUST be `0`; unknown bits ⟹ `-EINVAL`. |
| `nr_wake` | `int` | in | Maximum number of source waiters to wake. `>= 0`. |
| `nr_requeue` | `int` | in | Maximum number of source waiters to move to target. `>= 0`. |

### return value

| Value | Meaning |
|---|---|
| `>= 0` | Total (`woken + requeued`). |
| `-1` + `errno` | Failure; nothing moved or woken. |

### errors

| errno | Trigger |
|---|---|
| `EAGAIN` | `*waiters[0].uaddr != waiters[0].val` at compare-and-act time. |
| `EFAULT` | `waiters` array or either `uaddr` invalid. |
| `EINVAL` | `flags != 0`, bad per-futex `flags`, `nr_wake < 0`, `nr_requeue < 0`, source and target keys identical, `__reserved != 0`, misaligned `uaddr`, mismatched element sizes between source and target. |
| `ENOSYS` | Kernel lacks futex2 support. |

### abi surface

```text
__NR_futex_requeue  (x86_64)  = 456
__NR_futex_requeue  (arm64)   = 456
__NR_futex_requeue  (riscv)   = 456
__NR_futex_requeue  (i386)    = 456

/* The shape is "exactly two futex_waitv entries"; future protocol may
   extend to N entries (already supported by sys_futex_waitv(2)) but
   futex_requeue stays a fixed two-entry call. */
```

### compatibility contract

REQ-1: Syscall number is **456** on x86_64. ABI-stable.

REQ-2: `flags` MUST be zero. The per-entry `flags` field on each `futex_waitv` carries the size / NUMA / PRIVATE bits.

REQ-3: `waiters[0]` and `waiters[1]` MUST both have the same element-size selection in their per-entry `flags` — requeueing across size classes is rejected with `-EINVAL`.

REQ-4: Source and target keys MUST be distinct after derivation; requeue-to-self ⟹ `-EINVAL` (use `futex_wake(2)` instead).

REQ-5: The source bucket lock is acquired, then under it: `*waiters[0].uaddr` is compared against `waiters[0].val`. Mismatch ⟹ release locks, `-EAGAIN`.

REQ-6: Lock acquisition follows the canonical bucket-pointer-order rule (lower bucket pointer first) to avoid AB-BA deadlock between source and target buckets.

REQ-7: After successful compare, up to `nr_wake` waiters whose `bitset & waiters[0].val_mask != 0` are woken from source; up to `nr_requeue` of the remainder are moved (key updated, bucket pointer updated) to target.

REQ-8: Requeue does NOT compare `*waiters[1].uaddr` against `waiters[1].val`. The target word's value is irrelevant — only the bucket assignment matters.

REQ-9: PRIVATE/SHARED encoding on each entry derives an independent key. A PRIVATE source paired with a SHARED target requeue is legal **if** both can be derived; the resulting waiters move to the target's hash bucket.

REQ-10: `__reserved` on each `futex_waitv` MUST be zero.

REQ-11: `nr_wake == 0 && nr_requeue == 0` is a no-op aside from the value compare; returns 0 if compare succeeds, `-EAGAIN` otherwise.

REQ-12: PI-requeue (FUTEX_CMP_REQUEUE_PI semantics) is NOT available via futex2 in this release.

REQ-13: Returned count is the sum `woken + requeued`, both clamped to the requested caps and to the number of eligible waiters.

REQ-14: Misaligned `uaddr` for the selected size ⟹ `-EINVAL`.

REQ-15: NUMA hint on either entry is honored; requeue preserves the migrated waiter's existing NUMA hint (which originated at its `futex_wait` call).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_zero_only` | INVARIANT | top-level `flags != 0` ⟹ `-EINVAL`. |
| `keys_distinct` | INVARIANT | `key_s == key_t` ⟹ `-EINVAL`. |
| `size_match` | INVARIANT | element sizes differ ⟹ `-EINVAL`. |
| `bucket_lock_order` | INVARIANT | source/target buckets always locked in canonical pointer order. |
| `compare_under_lock` | INVARIANT | value compare and waiter manipulation under same locked section. |
| `migrate_updates_key` | INVARIANT | every requeued waiter has its key replaced with `key_t`. |

### Layer 2: TLA+

`kernel/futex2-requeue.tla`:
- Per-requeue transitions: validate, lock-order, compare, wake-n, move-n.
- Properties:
  - `safety_no_AB_BA` — two-bucket lock acquisition order canonical, no deadlock.
  - `safety_no_self_requeue` — source key ≠ target key invariant.
  - `safety_atomic_wake_and_move` — wake set and move set both computed under same lock section.
  - `liveness_terminates` — every futex_requeue returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Futex2::requeue` post: ret == woken + moved | `Futex2::requeue` |
| `Futex2::move_n` post: every moved waiter is on `dst` with `key == key_t` | `Futex2::move_n` |
| `Futex2::validate_waitv_entry` post: rejects `__reserved != 0` | `Futex2::validate_waitv_entry` |
| `order_locks` post: deterministic pointer order | `order_locks` |

### Layer 4: Verus / Creusot functional

Per-`futex_requeue(2)` man page; pthread_cond_broadcast wake-one-requeue-rest pattern; glibc nptl semantics.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`futex_requeue(2)` reinforcement:

- **Per-canonical-lock-order** — defense against per-AB-BA deadlock between source / target buckets.
- **Per-key-distinct enforcement** — defense against per-self-requeue infinite loop.
- **Per-size-match enforcement** — defense against per-element-size confusion across requeue.
- **Per-top-level `flags == 0`** — defense against per-future-bit smuggle.
- **Per-`__reserved` zero check** — defense against per-extension-field smuggling.

### grsecurity / pax surface

- **PaX UDEREF on `waiters` array and both `uaddr`s** — defense against per-pointer smuggling; SMAP forced.
- **FUTEX2_PRIVATE mandatory in hardened mode** — `GRKERNSEC_FUTEX_PRIVATE_ONLY` rejects shared-mapping requeue between non-init-userns processes.
- **Per-canonical bucket-lock-order enforcement** — defense against per-attacker-induced AB-BA stall.
- **PAX_USERCOPY on `futex_waitv` read** — bounded copy of the two-entry array uses whitelisted slab; rejects oversize.
- **Per-`__reserved` strict zero** — defense against per-future-ABI smuggle.
- **Per-`flags` top-level strict zero** — defense against per-future-flag smuggle until kernel explicitly allocates.
- **GRKERNSEC_HIDESYM on bucket / queue pointers** — internal addresses not exposed via oops.
- **PAX_REFCOUNT on per-waiter task refcount during migration** — defense against per-refcount UAF when waiter is concurrently woken by signal.
- **Per-bucket lock hold bounded** — defense against per-DoS via large `nr_requeue` (kernel caps internal batch).
- **Per-cross-userns requeue capability gate** — `GRKERNSEC_FUTEX_NS_GUARD` requires `CAP_SYS_ADMIN` to requeue between mm in different user namespaces (covert-channel attenuation).
- **Per-NUMA hint preserved sanitized** — hint is copied verbatim from waiter, never re-read from userspace during requeue.

