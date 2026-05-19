# Tier-5 syscall: set_mempolicy(2) — syscall 238

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - mm/mempolicy.c (SYSCALL_DEFINE3(set_mempolicy), kernel_set_mempolicy, do_set_mempolicy)
  - mm/mempolicy.c (mpol_new, mpol_set_nodemask, mpol_put, current->mempolicy)
  - include/uapi/linux/mempolicy.h (MPOL_*, MPOL_F_*)
  - include/linux/mempolicy.h (struct mempolicy, get_task_policy)
  - arch/x86/entry/syscalls/syscall_64.tbl (238  common  set_mempolicy)
-->

## Summary

`set_mempolicy(2)` installs (or clears) the calling task's **default** NUMA memory-allocation policy. Unlike `mbind(2)` which is per-VMA, `set_mempolicy(2)` is per-task: it controls fallback allocations for ranges that have no VMA policy, plus all kernel allocations made on behalf of the task (page-cache, slab-fallback). Children inherit; `execve` resets to MPOL_DEFAULT.

Critical for: numactl front-end, JVM/OpenMP thread placement, page-cache locality, anonymous-mapping placement when no `mbind` is used.

## Signature

```c
long set_mempolicy(int mode, const unsigned long *nodemask, unsigned long maxnode);
```

```c
/* Same modes as mbind(2): */
#define MPOL_DEFAULT              0
#define MPOL_PREFERRED            1
#define MPOL_BIND                 2
#define MPOL_INTERLEAVE           3
#define MPOL_LOCAL                4
#define MPOL_PREFERRED_MANY       5
#define MPOL_WEIGHTED_INTERLEAVE  6

/* Mode flag modifiers ORed into mode: */
#define MPOL_F_STATIC_NODES       (1u << 15)
#define MPOL_F_RELATIVE_NODES     (1u << 14)
#define MPOL_F_NUMA_BALANCING     (1u << 13)
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `mode` | `int` | in | Lower bits: `MPOL_*` policy. Upper bits: `MPOL_F_*` modifier flags. |
| `nodemask` | `const unsigned long *` | in | Node-bitmap for BIND / INTERLEAVE / PREFERRED / PREFERRED_MANY / WEIGHTED_INTERLEAVE. NULL for MPOL_DEFAULT / MPOL_LOCAL. |
| `maxnode` | `unsigned long` | in | One more than highest valid bit position in `nodemask`. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Policy installed on `current` task. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EFAULT` | `nodemask` user pointer faults. |
| `EINVAL` | `mode` unknown; reserved bits set; `nodemask` empty when policy requires nodes; `nodemask` non-empty when policy requires none; `maxnode > MAX_NUMNODES + 1`. |
| `ENOMEM` | Policy allocation failed. |
| `E2BIG` | `maxnode` exceeds compile-time bound. |
| `ENODEV` | A node in `nodemask` is offline. |

## ABI surface

```text
__NR_set_mempolicy  (x86_64)  = 238
__NR_set_mempolicy  (arm64)   = 237
__NR_set_mempolicy  (riscv)   = 237
__NR_set_mempolicy  (i386)    = 276

/* No 32-on-64 wrapper required — unsigned long array marshaled identically
   for the same word width. compat_set_mempolicy() handles 32-bit-on-64. */
```

## Compatibility contract

REQ-1: Syscall number is **238** on x86_64. ABI-stable.

REQ-2: Per-`current->mempolicy` is replaced atomically (old refcounted-put after new installed).

REQ-3: `mode` policy/flag split identical to `mbind(2)`.

REQ-4: `MPOL_DEFAULT` (with `NULL` nodemask, `maxnode = 0`) clears `current->mempolicy` to the system fallback (no per-task policy).

REQ-5: `MPOL_F_STATIC_NODES`: nodemask is absolute node ids — does not remap on cpuset change.

REQ-6: `MPOL_F_RELATIVE_NODES`: nodemask is relative to the task's current cpuset — remaps on cpuset mems_allowed change.

REQ-7: Without either modifier, nodemask is intersected with cpuset.mems_allowed at apply time and re-evaluated on cpuset changes.

REQ-8: `MPOL_F_NUMA_BALANCING` modifier: must be combined with `MPOL_BIND`; opts-in to AutoNUMA migration even under strict bind.

REQ-9: `fork(2)` copies policy (refcount + 1). `execve(2)` resets to `MPOL_DEFAULT`. Thread creation shares (per `clone` flags).

REQ-10: `MPOL_BIND` with an empty `mems_allowed` intersection ⟹ no usable nodes; allocations fall back to system default (per upstream semantics for non-strict mode flags).

REQ-11: Per-`mpol_set_nodemask` validates nodes are online and resident (have `N_MEMORY` flag).

REQ-12: Task policy is consulted by `alloc_pages_current()` when no VMA policy applies.

REQ-13: Per-`get_mempolicy(MPOL_F_NODE | MPOL_F_ADDR, ...)` reflects the installed policy for the task.

REQ-14: Policy install path is non-blocking (uses spin_lock only); no `mmap_lock` needed since it doesn't touch VMAs.

REQ-15: `maxnode == 0` legal iff policy needs no nodes.

## Acceptance Criteria

- [ ] AC-1: `set_mempolicy(MPOL_INTERLEAVE, {1,2}, 64)` then `mmap()` + fault: pages interleave across nodes 1 and 2.
- [ ] AC-2: `set_mempolicy(MPOL_DEFAULT, NULL, 0)` clears policy; falls back to system default.
- [ ] AC-3: `set_mempolicy(MPOL_BIND, {0}, 64)` and node 0 OOM: allocation fails / OOM-kill triggered.
- [ ] AC-4: `mode = MPOL_BIND` with NULL nodemask: `-EINVAL`.
- [ ] AC-5: `mode = 9999`: `-EINVAL`.
- [ ] AC-6: `maxnode = MAX_NUMNODES + 2`: `-EINVAL` / `-E2BIG`.
- [ ] AC-7: Offline node in mask: `-ENODEV`.
- [ ] AC-8: After `fork`: child inherits policy; refcount +1.
- [ ] AC-9: After `execve`: policy back to `MPOL_DEFAULT`.
- [ ] AC-10: `MPOL_F_STATIC_NODES`: cpuset mems_allowed change does not remap mask.
- [ ] AC-11: `MPOL_F_RELATIVE_NODES`: cpuset change relocates mask.
- [ ] AC-12: Concurrent `set_mempolicy` from two threads: last-writer-wins; old policy refcount drops to 0.

## Architecture

```rust
#[syscall(nr = 238, abi = "sysv")]
pub fn sys_set_mempolicy(mode: i32, nodemask: UserPtr<u64>, maxnode: usize) -> isize {
    Mempolicy::do_set_mempolicy(mode, nodemask, maxnode)
}
```

`Mempolicy::do_set_mempolicy(mode, nodemask_uptr, maxnode) -> isize`:
1. let mode_flags = mode & MPOL_MODE_FLAGS;
2. let policy = (mode & !MPOL_MODE_FLAGS) as u16;
3. if policy > MPOL_MAX: return -EINVAL;
4. /* Copy and validate node bitmap */
5. let nodes = Mempolicy::get_nodes(nodemask_uptr, maxnode)?;
6. /* Build policy object */
7. let new = Mempolicy::mpol_new(policy, mode_flags, &nodes)?;
8. /* Intersect with cpuset unless STATIC */
9. if mode_flags & MPOL_F_STATIC_NODES == 0 {
10.    Mempolicy::mpol_set_nodemask(&mut new, &current_cpuset_mems());
11. }
12. /* Install atomically */
13. let task = current();
14. let old = task.swap_mempolicy(new);
15. Mempolicy::mpol_put(old);
16. Ok(0)

`Mempolicy::get_nodes(uptr, maxnode) -> Result<NodeMask>`:
1. if maxnode == 0: return Ok(NodeMask::empty());
2. if maxnode > MAX_NUMNODES + 1: return Err(EINVAL);
3. let nlongs = (maxnode + BITS_PER_LONG - 1) / BITS_PER_LONG;
4. let mut bits = vec![0u64; nlongs];
5. unsafe { uptr.copy_in(&mut bits)?; }
6. /* Clear bits beyond maxnode */
7. let extra = nlongs * BITS_PER_LONG - maxnode;
8. if extra > 0 { bits[nlongs-1] &= (1u64 << (BITS_PER_LONG - extra)) - 1; }
9. let mask = NodeMask::from_bits(&bits);
10. /* Validate node ids online */
11. for node in mask.iter() {
12.   if !node_online(node) || !node_state(node, N_MEMORY) {
13.     return Err(ENODEV);
14.   }
15. }
16. Ok(mask)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `policy_swap_atomic` | INVARIANT | task->mempolicy swap uses RCU/atomic pointer. |
| `old_policy_put_after_swap` | INVARIANT | refcount of replaced policy decremented exactly once. |
| `nodemask_intersect_cpuset_unless_static` | INVARIANT | non-STATIC ⟹ intersection performed. |
| `offline_node_rejected` | INVARIANT | offline / no-memory node ⟹ ENODEV. |

### Layer 2: TLA+

`mm/mempolicy-set.tla`:
- States: per-validate, per-mpol_new, per-swap, per-mpol_put.
- Properties:
  - `safety_atomic_swap` — concurrent set_mempolicy observes serialized order.
  - `safety_old_freed` — every replaced policy reaches refcount 0.
  - `safety_static_isolation` — STATIC mask not rebound by cpuset.
  - `liveness_set_mempolicy_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_set_mempolicy` post: current.mempolicy = new ∨ -errno | `Mempolicy::do_set_mempolicy` |
| `mpol_new` post: nodes consistent with policy table | `Mempolicy::mpol_new` |
| `get_nodes` post: bits ⊆ N_MEMORY | `Mempolicy::get_nodes` |

### Layer 4: Verus / Creusot functional

Per-`set_mempolicy(2)` man-page semantic equivalence. libnuma selftests pass.

## Hardening

(Inherits row-1 from `uapi/00-overview.md` § Hardening.)

`set_mempolicy(2)` reinforcement:

- **Per-atomic-swap of task->mempolicy** — defense against per-half-installed policy UAF.
- **Per-old-policy refcount-put guaranteed** — defense against per-policy leak.
- **Per-offline-node rejection** — defense against per-stranded allocation.
- **Per-reserved-bit validation in `mode`** — defense against per-extension-field smuggling.
- **Per-execve reset to MPOL_DEFAULT** — defense against per-policy-survival across SUID exec.

## Grsecurity / PaX surface

- **PaX UDEREF on `nodemask` copy_from_user** — defense against per-user-pointer kernel-deref bug; SMAP forced.
- **CAP_SYS_NICE for any policy that affects another task** — `set_mempolicy` itself targets `current`, so no extra cap normally needed; grsec adds `CAP_SYS_NICE` if a thread changes a *thread-group-leader*'s policy that other threads rely on.
- **GRKERNSEC_PROC_GETPID parity on numa_maps** — `/proc/<pid>/numa_maps` and `/proc/<pid>/status` policy fields honor process-hiding.
- **PAX_USERCOPY_HARDEN on nodemask bitmap** — whitelisted slab; size capped at compile-time `MAX_NUMNODES`.
- **GRKERNSEC_MEM enforces N_MEMORY check** — nodes without memory or behind a CXL-tier-restricted boundary are rejected.
- **PAX_REFCOUNT on struct mempolicy refcnt** — defense against per-refcount-overflow UAF (especially across fork chain).
- **PaX KERNSEAL on `mpol_ops` dispatch table** — read-only post-init.
- **GRKERNSEC_RBAC mempolicy ACL** — subjects can be denied set_mempolicy entirely (e.g. sandboxed renderers).
- **execve scrub including mempolicy** — grsec verifies MPOL_DEFAULT installed before SUID transition completes.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- VMA-policy mechanics (covered in `mbind.md`).
- Cpuset interaction details (covered in Tier-3 `kernel/cgroup/cpuset.md`).
- NUMA balancing scheduler (Tier-3 `kernel/sched/numa.md`).
- Implementation code.
