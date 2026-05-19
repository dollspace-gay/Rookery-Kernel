---
title: "Tier-5 syscall: mbind(2) — syscall 237"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`mbind(2)` binds a NUMA memory-allocation policy to a VMA-range `[addr, addr+len)`. After binding, every subsequent page-fault in the range obeys the policy: which set of NUMA nodes the kernel may allocate from, and the allocation algorithm (round-robin interleave, weighted interleave, strict bind, preferred-fallback, local-node, preferred-many). Optional `MPOL_MF_MOVE` / `MPOL_MF_MOVE_ALL` flags cause the kernel to migrate already-resident pages in the range to satisfy the new policy.

Critical for: NUMA-aware databases, HPC workloads, JVM/CRuntime NUMA tuning, container memory locality, hugepage-on-node placement, CXL-tier-aware placement.

### Acceptance Criteria

- [ ] AC-1: `mbind(addr, len, MPOL_BIND, mask{0}, 64, 0)` then fault-in: page resident on node 0.
- [ ] AC-2: `mbind(..., MPOL_INTERLEAVE, mask{0,1}, ..., 0)` faults alternate nodes for sequential page indices.
- [ ] AC-3: `mbind(..., MPOL_DEFAULT, NULL, 0, 0)` reverts to task policy.
- [ ] AC-4: `MPOL_BIND` with `nodemask=0`: `-EINVAL`.
- [ ] AC-5: `addr` unaligned: `-EINVAL`.
- [ ] AC-6: `mode = 9999`: `-EINVAL`.
- [ ] AC-7: `MPOL_MF_MOVE_ALL` without CAP_SYS_NICE: `-EPERM`.
- [ ] AC-8: Offline-node in mask: `-ENODEV` or `-EINVAL` per kernel.
- [ ] AC-9: `MPOL_MF_STRICT|MPOL_MF_MOVE`: pinned page un-moveable; returns positive count.
- [ ] AC-10: `MPOL_MF_LAZY`: page PTE becomes PROT_NONE; next access re-faults.
- [ ] AC-11: VMA-split observed in `/proc/<pid>/numa_maps`.
- [ ] AC-12: `MPOL_WEIGHTED_INTERLEAVE`: histogram of pages matches configured weights ±5%.

### Architecture

```rust
#[syscall(nr = 237, abi = "sysv")]
pub fn sys_mbind(addr: usize, len: usize, mode: i32,
                 nodemask: UserPtr<u64>, maxnode: usize, flags: u32) -> isize {
    Mempolicy::do_mbind(addr, len, mode, nodemask, maxnode, flags)
}
```

`Mempolicy::do_mbind(addr, len, mode, nodemask_uptr, maxnode, flags) -> isize`:
1. /* Pre-validate */
2. if addr & (PAGE_SIZE - 1) != 0: return -EINVAL;
3. let mode_flags = mode & MPOL_MODE_FLAGS;
4. let policy = (mode & !MPOL_MODE_FLAGS) as u16;
5. if policy > MPOL_MAX: return -EINVAL;
6. if flags & !MPOL_MF_VALID != 0: return -EINVAL;
7. if (flags & MPOL_MF_MOVE_ALL) && !cap(CAP_SYS_NICE): return -EPERM;
8. /* Copy nodemask */
9. let nodes = Mempolicy::get_nodes(nodemask_uptr, maxnode)?;
10. /* Build new policy */
11. let new = Mempolicy::mpol_new(policy, mode_flags, &nodes)?;
12. /* Apply to VMAs */
13. let mm = current().mm;
14. mmap_write_lock(mm);
15. Mempolicy::mbind_range(mm, addr, addr + len, &new);
16. /* Migration */
17. let mut moved = 0;
18. if flags & (MPOL_MF_MOVE | MPOL_MF_MOVE_ALL) != 0 {
19.   moved = Mempolicy::queue_pages_range(mm, addr, addr + len, &new, flags)?;
20. }
21. mmap_write_unlock(mm);
22. /* Strict accounting */
23. if flags & MPOL_MF_STRICT != 0 && moved > 0: return moved as isize;
24. Ok(0)

`Mempolicy::mpol_new(policy, flags, nodes) -> Result<Policy>`:
1. match policy {
2.   MPOL_DEFAULT  => { if !nodes.is_empty() { return Err(EINVAL); } }
3.   MPOL_BIND | MPOL_INTERLEAVE | MPOL_PREFERRED_MANY | MPOL_WEIGHTED_INTERLEAVE
4.       => if nodes.is_empty() { return Err(EINVAL); }
5.   MPOL_LOCAL => if !nodes.is_empty() { return Err(EINVAL); }
6.   MPOL_PREFERRED => { /* empty = LOCAL */ }
7.   _ => return Err(EINVAL),
8. };
9. let p = Policy { mode: policy, flags, nodes: nodes.clone(), refcnt: 1 };
10. Mempolicy::mpol_set_nodemask(&mut p)?;
11. Ok(p)

### Out of Scope

- NUMA balancing scheduler logic (covered in Tier-3 `kernel/sched/numa.md`).
- Page-migration mechanics (covered in Tier-3 `mm/migrate.md`).
- Hugepage NUMA placement (covered in Tier-3 `mm/hugetlb.md`).
- Implementation code.

### signature

```c
long mbind(void *addr, unsigned long len, int mode,
           const unsigned long *nodemask, unsigned long maxnode,
           unsigned int flags);
```

```c
enum {
    MPOL_DEFAULT             = 0,
    MPOL_PREFERRED           = 1,
    MPOL_BIND                = 2,
    MPOL_INTERLEAVE          = 3,
    MPOL_LOCAL               = 4,
    MPOL_PREFERRED_MANY      = 5,
    MPOL_WEIGHTED_INTERLEAVE = 6,
};

/* mode flags ORed into mode */
#define MPOL_F_STATIC_NODES     (1u << 15)
#define MPOL_F_RELATIVE_NODES   (1u << 14)
#define MPOL_F_NUMA_BALANCING   (1u << 13)

/* mbind flags */
#define MPOL_MF_STRICT          (1u << 0)
#define MPOL_MF_MOVE            (1u << 1)
#define MPOL_MF_MOVE_ALL        (1u << 2)
#define MPOL_MF_LAZY            (1u << 3)
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `addr` | `void *` | in | Range start. Must be page-aligned (`PAGE_SIZE`). |
| `len` | `unsigned long` | in | Range length in bytes. Rounded up to page granularity for VMA splitting. |
| `mode` | `int` | in | Lower bits: `MPOL_*` policy. Upper bits: `MPOL_F_*` modifiers. |
| `nodemask` | `const unsigned long *` | in | Bitmap of allowed NUMA nodes. May be `NULL` only for `MPOL_DEFAULT` / `MPOL_LOCAL`. |
| `maxnode` | `unsigned long` | in | One more than the highest node id; bounds the bitmap. |
| `flags` | `unsigned int` | in | `MPOL_MF_*` migration / strictness flags. |

### return value

| Value | Meaning |
|---|---|
| `0` | Success; policy installed and (if requested) migration completed. |
| `> 0` | `MPOL_MF_STRICT` with `MPOL_MF_MOVE`: number of pages that could not be moved. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EFAULT` | `nodemask` user pointer faults; `addr`+`len` straddles unmapped region. |
| `EINVAL` | `mode` not a known policy; `maxnode > MAX_NUMNODES + 1`; `nodemask` empty for BIND/INTERLEAVE; `addr` not page-aligned; reserved bits set in `flags`. |
| `ENOMEM` | VMA split allocation failed; migration target out of memory. |
| `EPERM` | `MPOL_MF_MOVE_ALL` without `CAP_SYS_NICE`. |
| `EIO` | `MPOL_MF_STRICT`: a page could not be migrated and strict was requested. |
| `ENODEV` | A node in `nodemask` is offline or has no memory. |
| `E2BIG` | `maxnode` exceeds kernel-supported bitmap. |

### abi surface

```text
__NR_mbind  (x86_64)  = 237
__NR_mbind  (arm64)   = 235
__NR_mbind  (riscv)   = 235
__NR_mbind  (i386)    = 274

/* nodemask layout: little-endian array of unsigned long; bit i = node i. */
/* maxnode is inclusive-of-bitmap-width, so bits [0, maxnode-1] are valid. */
```

### compatibility contract

REQ-1: Syscall number is **237** on x86_64. ABI-stable.

REQ-2: `mode & ~MPOL_MODE_FLAGS` selects the policy enum; `mode & MPOL_MODE_FLAGS` selects modifiers. Combinations validated against `mpol_new()` table.

REQ-3: `maxnode` validation: `(maxnode + BITS_PER_LONG - 1) / BITS_PER_LONG` longs copied from user via `get_nodes`. `maxnode > MAX_NUMNODES + 1` ⟹ `-EINVAL`.

REQ-4: `MPOL_DEFAULT`: clear policy on range; nodemask must be empty (`NULL` ok).

REQ-5: `MPOL_BIND`: strict — allocations may only come from `nodemask`; if none has memory ⟹ OOM. Nodemask must be non-empty.

REQ-6: `MPOL_INTERLEAVE`: round-robin across `nodemask` indexed by VA (deterministic). Nodemask must be non-empty.

REQ-7: `MPOL_WEIGHTED_INTERLEAVE`: weighted round-robin via sysfs weight table `/sys/kernel/mm/mempolicy/weighted_interleave/node<N>`. Sum must be > 0.

REQ-8: `MPOL_PREFERRED`: single preferred node, soft fallback to any. Empty nodemask = `MPOL_LOCAL`.

REQ-9: `MPOL_LOCAL`: preferred = task's CPU's node at fault time. Equivalent to `MPOL_PREFERRED` with empty mask.

REQ-10: `MPOL_PREFERRED_MANY`: set of preferred nodes, soft fallback. New in 6.x; nodemask non-empty.

REQ-11: `MPOL_MF_MOVE`: migrate pages owned solely by the caller's process. Skip shared mappings unless `MPOL_MF_MOVE_ALL` (`CAP_SYS_NICE`).

REQ-12: `MPOL_MF_STRICT`: combined with no `MOVE`, returns `EIO` if any page violates. With `MOVE`, returns count of un-moveable pages.

REQ-13: `MPOL_MF_LAZY`: mark pages PROT_NONE so the next fault re-places per policy (NUMA-balancing piggyback).

REQ-14: VMAs are split at the boundaries `[addr, addr+len)` so the policy precisely tags the range. Mergeable adjacent VMAs with equal policy re-merge.

REQ-15: Policy is per-VMA, separate from task-policy (`set_mempolicy`). VMA policy wins for that range.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `policy_enum_total` | INVARIANT | every MPOL_* dispatched. |
| `nodemask_bounded` | INVARIANT | nodemask read ≤ maxnode bits. |
| `cap_check_move_all` | INVARIANT | MOVE_ALL ⟹ CAP_SYS_NICE held. |
| `vma_split_atomic_under_mmap_write` | INVARIANT | mbind_range under mmap_write_lock. |

### Layer 2: TLA+

`mm/mempolicy-mbind.tla`:
- States: per-validate, per-mpol_new, per-vma-split, per-migration.
- Properties:
  - `safety_no_policy_install_on_bad_args` — invalid args ⟹ no VMA mutation.
  - `safety_move_all_requires_cap` — CAP_SYS_NICE enforced.
  - `safety_strict_returns_unmoved_count` — strict path returns count.
  - `liveness_mbind_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_mbind` post: success ⟹ VMA policy = new | `Mempolicy::do_mbind` |
| `mpol_new` post: nodemask matches policy table | `Mempolicy::mpol_new` |
| `queue_pages_range` post: pages obey new policy or counted as un-moveable | `Mempolicy::queue_pages_range` |

### Layer 4: Verus / Creusot functional

Per-`mbind(2)` man-page semantic equivalence. numactl + libnuma selftests pass.

### hardening

(Inherits row-1 from `uapi/00-overview.md` § Hardening.)

`mbind(2)` reinforcement:

- **Per-`MPOL_MF_MOVE_ALL` CAP_SYS_NICE gate** — defense against per-cross-process migration abuse.
- **Per-nodemask copy bounded by `maxnode`** — defense against per-user-ptr OOB.
- **Per-VMA-split under mmap_write_lock** — defense against per-concurrent-mutate UAF.
- **Per-offline-node rejection** — defense against per-stranded allocation.
- **Per-mode-flags validation** — defense against per-reserved-bit smuggling.

### grsecurity / pax surface

- **PaX UDEREF on `nodemask` copy_from_user** — defense against per-user-pointer kernel-deref bug; SMAP forced on the get_nodes path.
- **CAP_SYS_NICE for cross-process mempolicy operations** — grsec narrows `MPOL_MF_MOVE_ALL` and any mbind affecting another task's VMA to require `CAP_SYS_NICE` in the task's userns; root-in-non-init-userns insufficient.
- **GRKERNSEC_PROC_GETPID parity on numa_maps** — `/proc/<pid>/numa_maps` (which reveals mbind regions) honors process-hiding rules.
- **PAX_USERCOPY_HARDEN on nodemask bitmap** — bitmap copy uses whitelisted slab; size strictly bounded by `MAX_NUMNODES` constant compiled-in.
- **GRKERNSEC_MEM block on offline/forbidden nodes** — nodes blacklisted via boot param refuse mbind requests.
- **Per-VMA policy ref ≤ rlim** — defense against per-policy-object DoS via tiny VMA splits; grsec caps the number of distinct VMA policies per process.
- **PAX_REFCOUNT on mempolicy refcnt** — defense against per-refcount-overflow UAF.
- **PaX KERNSEAL on policy table** — `mpol_ops[]` dispatch table mapped read-only post-init.

