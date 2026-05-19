# Tier-5 syscall: futex_wake(2) — syscall 454

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/futex/syscalls.c (SYSCALL_DEFINE4(futex_wake), futex2_to_flags)
  - kernel/futex/waitwake.c (futex_wake, futex_wake_op)
  - kernel/futex/core.c (futex_hash, get_futex_key)
  - include/uapi/linux/futex.h (FUTEX2_SIZE_U8/U16/U32/U64, FUTEX2_NUMA, FUTEX2_PRIVATE)
  - arch/x86/entry/syscalls/syscall_64.tbl (454  common  futex_wake)
-->

## Summary

`futex_wake(2)` is one of three new "futex2" syscalls (alongside `futex_wait(2)` and `futex_requeue(2)`) introduced in Linux 6.7 to replace the multiplexed `futex(2)` op codes. It wakes up to `nr` waiters that are blocked on the userspace word at `uaddr` whose stored wake-bitset intersects `mask`. The `flags` argument carries a `FUTEX2_*` packed encoding that selects element size (u8/u16/u32/u64), numa awareness, and the private/shared mapping discipline.

Critical for: pthread mutex unlock fast paths, glibc nptl, Rust `std::sync` primitives, wine/proton ESYNC/FSYNC emulation, sub-int futex words for fine-grained lock packing.

## Signature

```c
long futex_wake(void *uaddr, unsigned long mask, unsigned int nr, unsigned int flags);
```

```c
/* flags */
#define FUTEX2_SIZE_U8     0x00
#define FUTEX2_SIZE_U16    0x01
#define FUTEX2_SIZE_U32    0x02
#define FUTEX2_SIZE_U64    0x03
#define FUTEX2_SIZE_MASK   0x03
#define FUTEX2_NUMA        0x04   /* uaddr followed by int node-hint */
#define FUTEX2_PRIVATE     0x80   /* process-private mapping; cheaper key */
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `uaddr` | `void *` | in | Userspace address of the futex word. Must be naturally aligned for the chosen element size. |
| `mask` | `unsigned long` | in | Bitset; only waiters whose stored bitset intersects this are eligible to wake. `mask == 0` returns `-EINVAL`. |
| `nr` | `unsigned int` | in | Maximum number of waiters to wake. `0` is legal (no-op success). |
| `flags` | `unsigned int` | in | `FUTEX2_*` bit-packed: size (low 2 bits), NUMA, PRIVATE. Unknown bits ⟹ `-EINVAL`. |

## Return value

| Value | Meaning |
|---|---|
| `>= 0` | Number of waiters actually woken (may be less than `nr`). |
| `-1` + `errno` | Failure; nothing woken. |

## Errors

| errno | Trigger |
|---|---|
| `EFAULT` | `uaddr` user pointer invalid or unmapped. |
| `EINVAL` | `mask == 0`, bad `flags` (unknown bits, reserved bits set), bad element size, misaligned `uaddr`. |
| `ENOSYS` | Kernel built without futex2 support (`CONFIG_FUTEX`). |
| `EAGAIN` | Transient: futex hash bucket migration in flight; caller retries. |

## ABI surface

```text
__NR_futex_wake  (x86_64)  = 454
__NR_futex_wake  (arm64)   = 454
__NR_futex_wake  (riscv)   = 454
__NR_futex_wake  (i386)    = 454

/* futex2 vs legacy futex(2): legacy uses 6 args with FUTEX_WAKE opcode and
   only u32 words; futex2 splits the syscall + supports u8/u16/u64 + a
   bitset large enough for unsigned long. */
```

## Compatibility contract

REQ-1: Syscall number is **454** on x86_64. ABI-stable.

REQ-2: `flags & FUTEX2_SIZE_MASK` MUST decode to one of `U8`, `U16`, `U32`, `U64`. The chosen size dictates required alignment of `uaddr`.

REQ-3: `uaddr` MUST be aligned to the chosen element size; misalignment ⟹ `-EINVAL`.

REQ-4: `mask == 0` ⟹ `-EINVAL`. A wake call with no eligible bitset is always a programmer error.

REQ-5: `nr == 0` succeeds with return value `0`. No waiters consulted; no fault on `uaddr`.

REQ-6: `flags & FUTEX2_PRIVATE` selects the per-mm key path; cheaper (no inode pinning) and isolates the hash from other processes.

REQ-7: Cross-process shared futex requires the mapping at `uaddr` to be `MAP_SHARED` and backed by a vma whose key derivation is stable across forks. Without `FUTEX2_PRIVATE` the kernel pins the inode + page offset.

REQ-8: `FUTEX2_NUMA` extends the futex word: the kernel reads `int node;` immediately following `uaddr`, recording the node hint with each waiter so wakes prefer local CPUs.

REQ-9: Per-`FUTEX2_NUMA` rules:
- The `int node;` slot uses host-native int width (4 bytes on all current ABIs).
- `node == -1`: kernel auto-detects (current CPU at wait time).
- Hint mismatch is not an error; it only changes wake priority.

REQ-10: Wake target order: the kernel walks the hash bucket waiter list FIFO; ties broken by NUMA hint (prefer same node).

REQ-11: The wake count returned reflects only waiters whose `bitset & mask != 0`.

REQ-12: Reserved fields in `flags` (currently bits 3, 5, 6 and 8-31) MUST be zero on entry.

REQ-13: The legacy `futex(uaddr, FUTEX_WAKE, ...)` remains available for backwards compatibility but is being deprecated in favor of futex2 for all new userspace.

REQ-14: Wake is observably ordered with respect to userspace `STORE uaddr; futex_wake(uaddr)` such that any concurrent `futex_wait(uaddr, expected)` that races and reads the new value will NOT block.

REQ-15: Bitset width is `unsigned long` — 64 bits on LP64, 32 bits on ILP32. Userspace MUST use the same width as the running kernel ABI.

## Acceptance Criteria

- [ ] AC-1: Single-thread wake of N waiters with `mask = ~0UL` returns `min(N, nr)`.
- [ ] AC-2: Wake with disjoint mask returns `0`.
- [ ] AC-3: `mask == 0` returns `-EINVAL`.
- [ ] AC-4: Misaligned `uaddr` for chosen size returns `-EINVAL`.
- [ ] AC-5: `nr == 0` returns `0`, does not fault even if `uaddr` is unmapped.
- [ ] AC-6: `flags` reserved bits non-zero returns `-EINVAL`.
- [ ] AC-7: `FUTEX2_PRIVATE` wake reaches only same-mm waiters.
- [ ] AC-8: `FUTEX2_NUMA` with node hint prefers local-CPU waiters.
- [ ] AC-9: Legacy `futex(uaddr, FUTEX_WAKE, ...)` and futex2 on the same address interoperate.
- [ ] AC-10: u8 / u16 / u64 element-size variants all wake correctly with type-appropriate alignment.

## Architecture

```rust
#[syscall(nr = 454, abi = "sysv")]
pub fn sys_futex_wake(
    uaddr: UserPtr<u8>,
    mask: usize,
    nr: u32,
    flags: u32,
) -> isize {
    Futex2::wake(uaddr, mask, nr, flags)
}
```

`Futex2::wake(uaddr, mask, nr, flags) -> isize`:
1. let f = Futex2Flags::parse(flags)?;             // EINVAL
2. if mask == 0 { return Err(EINVAL); }
3. if !uaddr.is_aligned_to(f.element_size()) { return Err(EINVAL); }
4. if nr == 0 { return Ok(0); }
5. let key = Futex2::get_key(uaddr, f)?;           // EFAULT
6. let bucket = Futex2::hash_bucket(&key);
7. let woken = bucket.with_lock(|b| Futex2::wake_waiters(b, &key, mask, nr));
8. Ok(woken as isize)

`Futex2::wake_waiters(bucket, key, mask, nr) -> u32`:
1. let mut count = 0u32;
2. for waiter in bucket.iter_matching(key) {
3.   if waiter.bitset & mask == 0 { continue; }
4.   waiter.wake_state = WakeState::Woken;
5.   wake_q_add(&mut wake_q, waiter.task);
6.   count += 1;
7.   if count == nr { break; }
8. }
9. wake_up_q(&wake_q);
10. count

`Futex2::get_key(uaddr, flags) -> Result<FutexKey>`:
1. let mm = current.mm();
2. let access = if flags.is_private() { KeyAccess::Private(mm.id()) } else { KeyAccess::Shared };
3. let key = match access {
4.   KeyAccess::Private(id) => FutexKey::private(id, uaddr.as_usize()),
5.   KeyAccess::Shared      => {
6.     let (inode, offset) = mm.lookup_file_offset(uaddr)?;
7.     FutexKey::shared(inode, offset)
8.   }
9. };
10. Ok(key)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `mask_zero_rejected` | INVARIANT | `mask == 0` ⟹ `-EINVAL`. |
| `align_enforced` | INVARIANT | misaligned `uaddr` ⟹ `-EINVAL`. |
| `nr_zero_no_fault` | INVARIANT | `nr == 0` returns 0 without dereferencing uaddr. |
| `bitset_intersect_only` | INVARIANT | only waiters with `bitset & mask != 0` are woken. |
| `wake_q_drained` | INVARIANT | every queued task is woken before return. |
| `private_isolates_mm` | INVARIANT | `FUTEX2_PRIVATE` key never collides across mm. |

### Layer 2: TLA+

`kernel/futex2-wake.tla`:
- Per-wake transitions: parse flags, key, hash, walk waiters, wake.
- Properties:
  - `safety_no_spurious_wake` — wake count ≤ actual matching waiters.
  - `safety_no_cross_mm_priv` — private wake never reaches other mm.
  - `safety_mask_zero_rejected` — `mask == 0` ⟹ no waiters touched.
  - `liveness_wake_terminates` — every futex_wake returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Futex2Flags::parse` post: returns Err for unknown bits | `Futex2Flags::parse` |
| `Futex2::get_key` post: private key disjoint from shared | `Futex2::get_key` |
| `Futex2::wake` post: ret ≤ nr | `Futex2::wake` |
| `Futex2::wake_waiters` post: every counted waiter is in WakeState::Woken | `Futex2::wake_waiters` |

### Layer 4: Verus / Creusot functional

Per-`futex(2)` and `futex_wake(2)` man pages; glibc nptl pthread_mutex_unlock fast path semantic equivalence; rust `std::thread::park`/`unpark` wake guarantee.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`futex_wake(2)` reinforcement:

- **Per-element-size alignment check** — defense against per-misaligned-atomic UB.
- **Per-`mask == 0` reject** — defense against per-no-op-wake bug masking real wakes.
- **Per-`FUTEX2_PRIVATE` mm-keyed hash** — defense against per-cross-process info-leak via shared buckets.
- **Per-reserved-flag-bit zero** — defense against per-future-ABI-collision.
- **Per-`nr` upper-bound** — defense against per-DoS via huge wake batches.

## Grsecurity / PaX surface

- **PaX UDEREF on uaddr** — defense against per-uaddr kernel-pointer smuggling; SMAP forced for the inevitable copy_from_user (NUMA-hint variant).
- **FUTEX2_PRIVATE mandatory in hardened mode** — `GRKERNSEC_FUTEX_PRIVATE_ONLY` rejects `!FUTEX2_PRIVATE` from non-init userns to deny cross-namespace covert channels via shared futex hash buckets.
- **Per-`futex_hash` salt randomized per-boot** — defense against per-DoS via hash-collision flooding (PaX RANDKSTACK partner).
- **PAX_USERCOPY on NUMA-hint read** — bounded copy of the `int node` field uses whitelisted slab; rejects oversize copies.
- **GRKERNSEC_HIDESYM on futex bucket pointers** — internal `struct futex_q` addresses never exposed via `/proc` or oops.
- **Per-`FUTEX2_NUMA` capability gate (CAP_SYS_NICE)** — defense against per-unprivileged NUMA-side-channel sniff.
- **PAX_REFCOUNT on `mm_users` during private-key derivation** — defense against per-refcount-overflow UAF.
- **Per-wake-count rate-limited audit** — defense against per-futex-storm DoS, logs spam thresholds.
- **Per-reserved-flag-bit strict zero (no silent-mask)** — defense against per-flag-smuggle for future bits.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `futex_wait(2)` (sibling syscall, see `futex_wait.md`).
- `futex_requeue(2)` (sibling syscall, see `futex_requeue.md`).
- Robust-list / PI-futex semantics (covered in Tier-3 `kernel/futex/pi.md`).
- Implementation code.
