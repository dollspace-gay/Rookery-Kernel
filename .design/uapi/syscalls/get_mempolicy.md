# Tier-5 syscall: get_mempolicy(2) — syscall 239

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - mm/mempolicy.c (SYSCALL_DEFINE5(get_mempolicy), kernel_get_mempolicy, do_get_mempolicy)
  - mm/mempolicy.c (get_policy_nodemask, get_vma_policy, lookup_node)
  - include/uapi/linux/mempolicy.h (MPOL_*, MPOL_F_NODE, MPOL_F_ADDR, MPOL_F_MEMS_ALLOWED)
  - arch/x86/entry/syscalls/syscall_64.tbl (239  common  get_mempolicy)
-->

## Summary

`get_mempolicy(2)` queries either the **current task's** NUMA policy (default), the policy installed on the **VMA containing `addr`** (`MPOL_F_ADDR`), the **node id of `addr`'s page** (`MPOL_F_NODE | MPOL_F_ADDR`), or the **node id that would be picked next** by the task policy (`MPOL_F_NODE`). It can also return the task's `cpuset.mems_allowed` (`MPOL_F_MEMS_ALLOWED`).

Critical for: `numactl --show`, libnuma introspection, NUMA-aware schedulers, leak-diagnostic tools.

## Signature

```c
long get_mempolicy(int *mode, unsigned long *nodemask,
                   unsigned long maxnode, void *addr, unsigned long flags);
```

```c
/* flags: */
#define MPOL_F_NODE          (1u << 0)   /* return node id, not policy */
#define MPOL_F_ADDR          (1u << 1)   /* consult policy of VMA at addr */
#define MPOL_F_MEMS_ALLOWED  (1u << 2)   /* return cpuset.mems_allowed */
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `mode` | `int *` | out | Receives the policy enum (or node id under `MPOL_F_NODE`); may be `NULL` to skip. |
| `nodemask` | `unsigned long *` | out | Receives the policy's node-bitmap. May be `NULL`. |
| `maxnode` | `unsigned long` | in | Caller's bitmap width; kernel truncates if smaller than policy nodemask, returns `-EINVAL` if too small for `MEMS_ALLOWED`. |
| `addr` | `void *` | in | VMA-lookup address; required for `MPOL_F_ADDR`. |
| `flags` | `unsigned long` | in | `MPOL_F_*` combo. Mutually-exclusive rules apply. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Success; `*mode` and/or `*nodemask` filled. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EFAULT` | `mode` / `nodemask` / `addr` user pointer faults. |
| `EINVAL` | Invalid `flags` combo; `maxnode` zero or insufficient; `MPOL_F_ADDR` without `addr` in any VMA; conflicting flags (e.g. `MEMS_ALLOWED | NODE`). |
| `E2BIG` | `maxnode` exceeds kernel-supported width. |
| `EPERM` | `addr` refers to another task's range and inspector lacks `PTRACE_MODE_READ_REALCREDS`. (Grsec-only; upstream returns the info.) |

## ABI surface

```text
__NR_get_mempolicy  (x86_64)  = 239
__NR_get_mempolicy  (arm64)   = 236
__NR_get_mempolicy  (riscv)   = 236
__NR_get_mempolicy  (i386)    = 275

/* Output nodemask layout: identical to mbind/set_mempolicy input. */
```

## Compatibility contract

REQ-1: Syscall number is **239** on x86_64. ABI-stable.

REQ-2: Default mode (`flags == 0`): returns the task's policy and its nodemask. Equivalent to introspecting `current->mempolicy`.

REQ-3: `MPOL_F_ADDR`: looks up VMA at `addr`; returns that VMA's policy (or task fallback if VMA has none).

REQ-4: `MPOL_F_NODE` alone (no `MPOL_F_ADDR`): returns the node that the *next* allocation under the task policy would target — i.e. resolves INTERLEAVE / WEIGHTED_INTERLEAVE / PREFERRED to a concrete node and writes the id to `*mode`.

REQ-5: `MPOL_F_NODE | MPOL_F_ADDR`: returns the node id of the **page currently mapped** at `addr` (faulting it in if necessary) — writes to `*mode`.

REQ-6: `MPOL_F_MEMS_ALLOWED`: ignores policy; writes the task's `cpuset.mems_allowed` to `*nodemask`. Mutually exclusive with `NODE`/`ADDR`.

REQ-7: `nodemask` validation: `maxnode` ≥ width needed for the policy's nodemask. If smaller for `MEMS_ALLOWED`: `-EINVAL`; for policy nodemask: truncate (upstream behavior preserved).

REQ-8: `addr` validation: must lie in a VMA when `MPOL_F_ADDR` set; otherwise `-EFAULT`.

REQ-9: `mode == NULL`: skip mode write. `nodemask == NULL`: skip mask write.

REQ-10: When `MPOL_F_ADDR` set, kernel takes `mmap_read_lock` to look up VMA; releases before copy_to_user.

REQ-11: Copy-to-user uses `put_user` for `*mode` and `copy_to_user` for nodemask, page-faults handled.

REQ-12: `MPOL_LOCAL` returned as `MPOL_PREFERRED` with empty nodemask (legacy behavior).

REQ-13: `MPOL_F_NODE` with task policy `MPOL_DEFAULT`: returns the local node of the calling CPU.

REQ-14: For `MPOL_BIND` and `MPOL_F_NODE`: returns the first set node in mask.

REQ-15: This syscall does not mutate state.

## Acceptance Criteria

- [ ] AC-1: `get_mempolicy(&m, mask, 64, NULL, 0)` after `set_mempolicy(MPOL_INTERLEAVE, {0,1}, 64)`: `m == MPOL_INTERLEAVE`, mask = {0,1}.
- [ ] AC-2: `get_mempolicy(&m, NULL, 0, addr, MPOL_F_ADDR)`: returns VMA policy.
- [ ] AC-3: `get_mempolicy(&m, NULL, 0, addr, MPOL_F_ADDR|MPOL_F_NODE)`: `m == node id of page at addr`.
- [ ] AC-4: `get_mempolicy(NULL, mask, 64, NULL, MPOL_F_MEMS_ALLOWED)`: mask = cpuset.mems_allowed.
- [ ] AC-5: `flags = MPOL_F_NODE | MPOL_F_MEMS_ALLOWED`: `-EINVAL`.
- [ ] AC-6: `MPOL_F_ADDR` with addr outside any VMA: `-EFAULT`.
- [ ] AC-7: `maxnode = 0` with `MPOL_F_MEMS_ALLOWED`: `-EINVAL`.
- [ ] AC-8: After `MPOL_F_NODE` resolution for INTERLEAVE: subsequent calls observe round-robin progress.
- [ ] AC-9: `mode = NULL` + `nodemask` non-NULL: only mask written.
- [ ] AC-10: `nodemask = NULL` + `mode` non-NULL: only mode written.
- [ ] AC-11: User-pointer fault during copy_to_user: `-EFAULT`, no info leaked.

## Architecture

```rust
#[syscall(nr = 239, abi = "sysv")]
pub fn sys_get_mempolicy(mode: UserPtr<i32>, nodemask: UserPtr<u64>,
                         maxnode: usize, addr: usize, flags: u32) -> isize {
    Mempolicy::do_get_mempolicy(mode, nodemask, maxnode, addr, flags)
}
```

`Mempolicy::do_get_mempolicy(mode_p, nm_p, maxnode, addr, flags) -> isize`:
1. /* Flag-combo validation */
2. if flags & !(MPOL_F_NODE | MPOL_F_ADDR | MPOL_F_MEMS_ALLOWED) != 0: return -EINVAL;
3. if (flags & MPOL_F_MEMS_ALLOWED) && (flags & (MPOL_F_NODE | MPOL_F_ADDR)) != 0: return -EINVAL;
4. /* MEMS_ALLOWED shortcut */
5. if flags & MPOL_F_MEMS_ALLOWED != 0 {
6.   if maxnode == 0 { return -EINVAL; }
7.   let mems = current_cpuset_mems();
8.   return Mempolicy::copy_nodes_to_user(nm_p, maxnode, &mems);
9. }
10. /* Resolve policy */
11. let (pol_kind, pol_nodes, node_id) = if flags & MPOL_F_ADDR != 0 {
12.    let mm = current().mm;
13.    mmap_read_lock(mm);
14.    let vma = find_vma(mm, addr).ok_or(-EFAULT)?;
15.    let pol = get_vma_policy(vma, addr);
16.    let kind = pol.mode;
17.    let nodes = pol.nodes.clone();
18.    let n = if flags & MPOL_F_NODE != 0 { lookup_node(mm, addr)? } else { -1 };
19.    mmap_read_unlock(mm);
20.    (kind, nodes, n)
21. } else {
22.    let pol = current_task_policy();
23.    let kind = pol.mode;
24.    let nodes = pol.nodes.clone();
25.    let n = if flags & MPOL_F_NODE != 0 { resolve_next_node(&pol) } else { -1 };
26.    (kind, nodes, n)
27. };
28. /* Write outputs */
29. if !mode_p.is_null() {
30.    let m = if flags & MPOL_F_NODE != 0 { node_id } else { pol_kind as i32 };
31.    put_user(mode_p, m)?;
32. }
33. if !nm_p.is_null() && (flags & MPOL_F_NODE) == 0 {
34.    Mempolicy::copy_nodes_to_user(nm_p, maxnode, &pol_nodes)?;
35. }
36. Ok(0)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flag_combo_validated` | INVARIANT | invalid combos rejected before any state read. |
| `no_state_mutation` | INVARIANT | task / vma policy unchanged after call. |
| `mmap_read_lock_balanced` | INVARIANT | every read_lock paired with read_unlock on all paths. |
| `null_output_skipped` | INVARIANT | NULL `mode` or NULL `nodemask` not written. |

### Layer 2: TLA+

`mm/mempolicy-get.tla`:
- States: per-validate, per-resolve-policy, per-write-output.
- Properties:
  - `safety_no_mutation`.
  - `safety_node_resolution_deterministic_per_step`.
  - `liveness_get_mempolicy_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_get_mempolicy` post: outputs reflect current policy snapshot | `Mempolicy::do_get_mempolicy` |
| `copy_nodes_to_user` post: bytes ≤ maxnode bits, fault-safe | `Mempolicy::copy_nodes_to_user` |

### Layer 4: Verus / Creusot functional

Per-`get_mempolicy(2)` man-page semantic equivalence. numactl --show parity.

## Hardening

(Inherits row-1 from `uapi/00-overview.md` § Hardening.)

`get_mempolicy(2)` reinforcement:

- **Per-mutually-exclusive flag rejection** — defense against per-flag-combo undefined-state.
- **Per-mmap_read_lock balanced on all error paths** — defense against per-stuck-lock DoS.
- **Per-copy_to_user fault rollback** — defense against per-partial-info-leak.
- **Per-NULL output skip** — defense against per-bogus-write to kernel ptr.

## Grsecurity / PaX surface

- **PaX UDEREF on output `mode` / `nodemask` copy_to_user** — defense against per-user-pointer kernel-deref bug; SMAP forced.
- **GRKERNSEC_PROC_GETPID gate for cross-task introspection** — when `MPOL_F_ADDR` is used against an address that belongs to another task's mm (rare but possible via shared-mapping inheritance), grsec requires `PTRACE_MODE_READ_REALCREDS`; without it, returns `-EPERM`.
- **CAP_SYS_NICE for policy info on non-owned VMAs** — grsec narrows information disclosure on shared NUMA layouts.
- **PAX_USERCOPY_HARDEN on output bitmap** — bitmap copy uses whitelisted slab; size strictly `maxnode`-bounded.
- **GRKERNSEC_MEM info-leak suppression** — `MPOL_F_MEMS_ALLOWED` returns mems_allowed *intersected* with the task's RBAC-allowed nodes (may be narrower than cpuset).
- **PaX KERNSEAL on `mpol_ops` table** — read-only post-init.
- **GRKERNSEC_RBAC ACL on numa_maps-equivalent introspection** — get_mempolicy results subject to the same hide-list as `/proc/<pid>/numa_maps`.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- VMA lookup mechanics (covered in Tier-3 `mm/mmap.md`).
- Cpuset semantics (Tier-3 `kernel/cgroup/cpuset.md`).
- Implementation code.
