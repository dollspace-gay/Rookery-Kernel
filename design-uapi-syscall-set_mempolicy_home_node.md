---
title: "Tier-5 syscall: set_mempolicy_home_node(2) — syscall 450"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`set_mempolicy_home_node(2)` (added in Linux 6.1) sets a **home node hint** on the VMA-level mempolicy covering `[addr, addr+len)`. The home node biases allocation when the policy is `MPOL_BIND`, `MPOL_INTERLEAVE`, `MPOL_PREFERRED_MANY`, or `MPOL_WEIGHTED_INTERLEAVE` — among the policy's allowed nodes, the home node is preferred (subject to the policy's actual selection algorithm: bind/interleave still respect the rule, but home tiebreaks). This decouples *who is allowed* from *who is preferred*.

Critical for: CXL-tier-aware allocation, multi-tier memory placement (DDR + CXL + HBM), large-page interleave with locality bias, heterogeneous-NUMA workloads.

### Acceptance Criteria

- [ ] AC-1: VMA with `MPOL_INTERLEAVE` on {0,1}: `set_mempolicy_home_node(addr, 4K, 1, 0)` ⟹ subsequent allocations prefer node 1 when tying.
- [ ] AC-2: VMA with `MPOL_DEFAULT`: `-EINVAL`.
- [ ] AC-3: `home_node = MAX_NUMNODES`: `-EINVAL`.
- [ ] AC-4: `home_node` not in mask: `-EINVAL`.
- [ ] AC-5: `flags = 1`: `-EINVAL`.
- [ ] AC-6: `addr` unaligned: `-EINVAL`.
- [ ] AC-7: Range straddles unmapped region: `-EINVAL`.
- [ ] AC-8: Mixed VMAs (one with policy, one without): `-EINVAL` and no mutation.
- [ ] AC-9: `numa_maps` reflects `home=<node>` after success.
- [ ] AC-10: After `mremap`, home node hint preserved.
- [ ] AC-11: After `execve`, hint gone.

### Architecture

```rust
#[syscall(nr = 450, abi = "sysv")]
pub fn sys_set_mempolicy_home_node(addr: usize, len: usize,
                                   home_node: usize, flags: usize) -> isize {
    Mempolicy::do_set_home_node(addr, len, home_node, flags)
}
```

`Mempolicy::do_set_home_node(addr, len, home, flags) -> isize`:
1. if flags != 0: return -EINVAL;
2. if addr & (PAGE_SIZE - 1) != 0: return -EINVAL;
3. if len == 0: return -EINVAL;
4. if home >= MAX_NUMNODES: return -EINVAL;
5. if !node_state(home, N_MEMORY): return -EINVAL;
6. /* Pre-walk: validate every VMA */
7. let mm = current().mm;
8. mmap_write_lock(mm);
9. let mut vma_it = find_vma(mm, addr);
10. let mut cursor = addr;
11. while cursor < addr + len {
12.    let vma = vma_it.ok_or_else(|| { mmap_write_unlock(mm); -EFAULT })?;
13.    if vma.vm_start > cursor { mmap_write_unlock(mm); return -EFAULT; }
14.    let pol = vma.policy.as_ref().ok_or_else(|| { mmap_write_unlock(mm); -EINVAL })?;
15.    match pol.mode {
16.       MPOL_BIND | MPOL_INTERLEAVE | MPOL_PREFERRED_MANY | MPOL_WEIGHTED_INTERLEAVE => {
17.         if !pol.nodes.has(home) { mmap_write_unlock(mm); return -EINVAL; }
18.       }
19.       _ => { mmap_write_unlock(mm); return -EINVAL; }
20.    }
21.    cursor = min(vma.vm_end, addr + len);
22.    vma_it = next_vma(vma_it);
23. }
24. /* Second walk: apply */
25. let mut vma_it2 = find_vma(mm, addr);
26. let mut cur2 = addr;
27. while cur2 < addr + len {
28.    let vma = vma_it2.unwrap();
29.    let lo = max(vma.vm_start, addr);
30.    let hi = min(vma.vm_end, addr + len);
31.    Mempolicy::vma_split_and_set_home(mm, vma, lo, hi, home);
32.    cur2 = hi;
33.    vma_it2 = next_vma(vma_it2);
34. }
35. mmap_write_unlock(mm);
36. Ok(0)

### Out of Scope

- VMA policy mechanics (covered in `mbind.md` / Tier-3 `mm/mempolicy.md`).
- Weighted-interleave weight table (Tier-3 `mm/mempolicy.md`).
- Implementation code.

### signature

```c
long set_mempolicy_home_node(unsigned long addr, unsigned long len,
                             unsigned long home_node, unsigned long flags);
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `addr` | `unsigned long` | in | Range start (page-aligned). |
| `len` | `unsigned long` | in | Range length in bytes. |
| `home_node` | `unsigned long` | in | Target home node id; must be in policy's nodemask. |
| `flags` | `unsigned long` | in | Reserved; must be 0. |

### return value

| Value | Meaning |
|---|---|
| `0` | Home node set on every VMA in the range. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EINVAL` | `addr` unaligned; `len == 0`; `flags != 0`; `home_node >= MAX_NUMNODES`; range straddles unmapped region; VMA has no policy (MPOL_DEFAULT) or policy is `MPOL_PREFERRED`/`MPOL_LOCAL`; `home_node` not in policy nodemask. |
| `EFAULT` | Range outside the caller's mm. |
| `ENOMEM` | VMA-split allocation failed. |
| `EPERM` | Home node not in caller's mems_allowed (grsec-hardened). |

### abi surface

```text
__NR_set_mempolicy_home_node  (x86_64)  = 450
__NR_set_mempolicy_home_node  (arm64)   = 450
__NR_set_mempolicy_home_node  (riscv)   = 450
__NR_set_mempolicy_home_node  (i386)    = 450

/* Per-VMA hint; survives mremap/madvise consistent with VMA policy. */
```

### compatibility contract

REQ-1: Syscall number is **450** on x86_64. ABI-stable since Linux 6.1.

REQ-2: Only valid when the VMA's policy is `MPOL_BIND`, `MPOL_INTERLEAVE`, `MPOL_PREFERRED_MANY`, or `MPOL_WEIGHTED_INTERLEAVE`. Other policies ⟹ `-EINVAL`.

REQ-3: `home_node` must be a member of the VMA's nodemask; else `-EINVAL`.

REQ-4: `home_node` must be `< MAX_NUMNODES`; offline / no-memory node ⟹ `-EINVAL`.

REQ-5: `flags` must be 0 (reserved for future); else `-EINVAL`.

REQ-6: VMAs in the range are walked; each must individually have a compatible policy. If any does not ⟹ `-EINVAL`, none of them are modified (all-or-nothing).

REQ-7: VMA-split at boundaries; merge-on-equal after.

REQ-8: Home node affects allocation tiebreaking:
- `MPOL_INTERLEAVE`: still round-robins, but the home node is used when a tie occurs (e.g., first allocation in a range).
- `MPOL_WEIGHTED_INTERLEAVE`: weight from home node honored; home gets preference if weights equal.
- `MPOL_PREFERRED_MANY`: home is the first preference, then the rest of the set.
- `MPOL_BIND`: home is preferred for actual allocation; falls through to other bind-set nodes on OOM.

REQ-9: Home node is per-VMA; does not propagate to task policy.

REQ-10: `mremap` of the range preserves the home node hint.

REQ-11: `mmap_write_lock` held while modifying VMAs.

REQ-12: Home node is reflected in `/proc/<pid>/numa_maps` as `home=<node>`.

REQ-13: `fork()` copies VMAs including home node. `execve()` resets (no policy).

REQ-14: Setting `home_node` is idempotent (re-setting to same value is success).

REQ-15: Requires `CAP_SYS_NICE` if the home node falls outside the task's effective cpuset mems_allowed (grsec-hardened).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `all_or_nothing` | INVARIANT | invalid VMA in range ⟹ no VMA modified. |
| `flags_reserved_zero` | INVARIANT | flags != 0 ⟹ EINVAL. |
| `home_in_policy_nodemask` | INVARIANT | success ⟹ home ∈ vma.policy.nodes for every VMA. |
| `mmap_write_lock_balanced` | INVARIANT | balanced across all early returns. |

### Layer 2: TLA+

`mm/mempolicy-home-node.tla`:
- States: per-validate, per-prewalk, per-apply.
- Properties:
  - `safety_no_partial_apply_on_validation_failure`.
  - `safety_home_node_in_mask_postcondition`.
  - `liveness_set_home_node_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_set_home_node` post: every VMA in range has policy.home == home | `Mempolicy::do_set_home_node` |
| `vma_split_and_set_home` post: VMA boundaries preserved | `Mempolicy::vma_split_and_set_home` |

### Layer 4: Verus / Creusot functional

Per-`set_mempolicy_home_node(2)` man-page semantic equivalence. selftests/mm/mempolicy_home_node.c parity.

### hardening

(Inherits row-1 from `uapi/00-overview.md` § Hardening.)

`set_mempolicy_home_node(2)` reinforcement:

- **Per-`flags == 0` reserved** — defense against per-extension-field smuggling.
- **Per-all-or-nothing apply** — defense against per-partial-state on bad input.
- **Per-mmap_write_lock balanced** — defense against per-stuck-lock DoS.
- **Per-home in policy nodemask** — defense against per-policy-bypass tiebreak.
- **Per-offline-node rejection** — defense against per-stranded preference.

### grsecurity / pax surface

- **PaX UDEREF parity** — though no user-pointer copy here, the validation path still runs under SMAP-strict mode; defense-in-depth against per-VMA-policy pointer bug via vm_area_struct.
- **CAP_SYS_NICE for out-of-cpuset home node** — grsec narrows: if `home_node` is not in the task's effective `cpuset.mems_allowed`, requires `CAP_SYS_NICE` in the task's userns. Upstream allows it as long as it's in the VMA policy's mask; grsec adds the cpuset gate.
- **GRKERNSEC_PROC_GETPID parity on numa_maps `home=`** — the home= field is hidden from `/proc/<pid>/numa_maps` when process-hiding applies.
- **GRKERNSEC_MEM block on forbidden nodes** — nodes blacklisted at boot refuse home-node setting.
- **PAX_REFCOUNT on mempolicy refcnt during walk** — defense against per-policy UAF during VMA-split phase.
- **PaX KERNSEAL on `mpol_ops` table** — read-only post-init.
- **GRKERNSEC_RBAC home_node ACL** — subjects can be denied this syscall entirely; useful for sandboxed allocators.
- **Per-VMA count cap** — VMA-split from this syscall is bounded by the same per-process VMA limit as `mbind`, preventing per-tiny-split DoS.

