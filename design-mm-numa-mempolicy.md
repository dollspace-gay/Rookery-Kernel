---
title: "Tier-3: mm/mempolicy.c — NUMA memory policy (per-task / per-VMA / per-system)"
tags: ["tier-3", "mm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

NUMA memory policy controls per-task / per-VMA / per-system page-allocation node-preference. Modes: `MPOL_DEFAULT` (system policy), `MPOL_PREFERRED` (try one node first), `MPOL_BIND` (must allocate from node-set), `MPOL_INTERLEAVE` (round-robin across set), `MPOL_LOCAL` (current-CPU's node), `MPOL_PREFERRED_MANY` (prefer set, fall back), `MPOL_WEIGHTED_INTERLEAVE` (weighted RR). Per-VMA `set_mempolicy` / `mbind` syscalls. Per-task `set_mempolicy(2)`. Per-system `numa_balancing` migrates pages toward access-affinity. Critical for: large-NUMA databases, HPC, KVM-host with multi-socket.

This Tier-3 covers `mempolicy.c` (~3948 lines).

### Acceptance Criteria

- [ ] AC-1: set_mempolicy(MPOL_PREFERRED, {1}): subsequent allocs prefer node 1.
- [ ] AC-2: set_mempolicy(MPOL_INTERLEAVE, {0,1,2,3}): allocs round-robin across nodes.
- [ ] AC-3: mbind range, MPOL_BIND, {0}: subsequent in-range allocs strictly node 0.
- [ ] AC-4: mbind + MPOL_MF_MOVE: existing pages migrated.
- [ ] AC-5: get_mempolicy MPOL_F_NODE: returns alloc-node of address.
- [ ] AC-6: cpuset shrink: mempolicy.nodemask remapped.
- [ ] AC-7: NUMA balancing: hot pages migrate toward task's CPU node.
- [ ] AC-8: THP allocation respects mempolicy: huge_node returns correct node.
- [ ] AC-9: MPOL_WEIGHTED_INTERLEAVE: per-node-weight respected.
- [ ] AC-10: fork: child mempolicy != parent's (independent counter).
- [ ] AC-11: numactl wrapping: works correctly.

### Architecture

Per-policy state:

```
struct Mempolicy {
  mode: u8,                                       // MPOL_*
  flags: u8,                                      // MPOL_F_*
  refcnt: AtomicI32,
  v: union {
    nodes: Nodemask,                              // MPOL_BIND / INTERLEAVE
    preferred_node: i32,                          // MPOL_PREFERRED
  },
  cpuset_mems_allowed: Nodemask,                  // for static/relative
  user_nodemask: Nodemask,                        // user-supplied (cpuset-relative)
  weight: Option<&[u8]>,                          // for WEIGHTED_INTERLEAVE
}
```

Per-task:

```
struct TaskStruct {
  ...
  mempolicy: Option<&Mempolicy>,
  il_prev: i32,                                  // last node for INTERLEAVE
}
```

Per-VMA:

```
struct VmAreaStruct {
  ...
  vm_policy: Option<&Mempolicy>,                 // mbind
}
```

`Mempolicy::do_set_mempolicy(mode, nmask, maxnode) -> Result<()>`:
1. Validate mode + flags + nmask.
2. new = Mempolicy::new(mode, flags, &nodes, current.mempolicy.user_nodemask).
3. Per-task: old = task.mempolicy; task.mempolicy = new.
4. mpol_put(old).
5. task.il_prev = NUMA_NO_NODE.

`Mempolicy::do_mbind(start, len, mode, nmask, flags) -> Result<()>`:
1. Validate range.
2. new = Mempolicy::new(mode, flags, &nodes, ...).
3. mmap_write_lock(mm).
4. for each VMA in [start, end):
   - old = vma.vm_policy.
   - vma.vm_policy = new.
   - mpol_put(old).
5. mmap_write_unlock.
6. If flags & MPOL_MF_MOVE: migrate_pages_to_nodes(start, len, target_nodes, MIGRATE_SYNC).

`Mempolicy::policy_node(pol, addr) -> i32`:
1. switch pol.mode:
   - MPOL_DEFAULT / MPOL_LOCAL: numa_node_id().
   - MPOL_PREFERRED: pol.v.preferred_node.
   - MPOL_BIND / MPOL_INTERLEAVE / MPOL_PREFERRED_MANY: handled per-zonelist or interleave.
   - MPOL_WEIGHTED_INTERLEAVE: weighted_interleave_nodes(pol).

`Mempolicy::interleave_nodes(pol) -> i32`:
1. nid = next_node_in(pol.v.nodes, current.il_prev).
2. current.il_prev = nid.
3. Return nid.

`Mempolicy::alloc_pages(pol, gfp, order, addr) -> Page`:
1. nid = Mempolicy::policy_node(pol, addr).
2. zonelist = node_zonelist(nid, gfp).
3. if pol.mode == MPOL_BIND ∨ MPOL_PREFERRED_MANY:
   - zonelist = filter_by_nodemask(zonelist, pol.v.nodes).
4. Return __alloc_pages(gfp, order, zonelist).

`Mempolicy::rebind(task, new_cpuset_mems_allowed)`:
1. pol = task.mempolicy.
2. if pol.flags & MPOL_F_RELATIVE_NODES:
   - pol.v.nodes = remap_relative(pol.user_nodemask, new_cpuset_mems_allowed).
3. else if !(pol.flags & MPOL_F_STATIC_NODES):
   - pol.v.nodes = pol.user_nodemask & new_cpuset_mems_allowed.
4. pol.cpuset_mems_allowed = new_cpuset_mems_allowed.

### Out of Scope

- mm/00-overview (Tier-2)
- Page allocator buddy (covered in `page-allocator.md` Tier-3)
- Migration (covered in `migration-compaction.md` Tier-3)
- cpuset (covered in `kernel/cgroup/` separately)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct mempolicy` | per-task or per-VMA policy | `Mempolicy` |
| `do_set_mempolicy()` | per-task setter | `Mempolicy::do_set_mempolicy` |
| `do_get_mempolicy()` | per-task getter | `Mempolicy::do_get_mempolicy` |
| `do_mbind()` | per-VMA-range setter | `Mempolicy::do_mbind` |
| `mpol_new()` | per-policy alloc | `Mempolicy::new` |
| `mpol_put()` | per-policy free | `Mempolicy::put` |
| `mpol_dup()` | per-policy clone | `Mempolicy::dup` |
| `mpol_rebind_*` | per-cpuset-change rebind nodes | `Mempolicy::rebind` |
| `policy_node()` | per-policy node-pick | `Mempolicy::policy_node` |
| `policy_zonelist()` | per-policy zonelist | `Mempolicy::policy_zonelist` |
| `alloc_pages_mpol()` | per-policy alloc | `Mempolicy::alloc_pages` |
| `interleave_nodes()` | per-MPOL_INTERLEAVE step | `Mempolicy::interleave_nodes` |
| `huge_node()` | per-THP node-pick | `Mempolicy::huge_node` |
| `migrate_to_node()` | per-mbind migrate | `Mempolicy::migrate_to_node` |
| `MPOL_*` enum | mode IDs | UAPI |
| `MPOL_F_NODE` / `_MEMS_ALLOWED` / `_RELATIVE_NODES` / `_STATIC_NODES` | mode flags | UAPI |
| `MPOL_MF_MOVE` / `_MOVE_ALL` / `_STRICT` | mbind flags | UAPI |

### compatibility contract

REQ-1: Per-policy modes:
- MPOL_DEFAULT: use system or VMA-default.
- MPOL_PREFERRED: prefer node, fall back.
- MPOL_BIND: must alloc in nodeset.
- MPOL_INTERLEAVE: round-robin in nodeset.
- MPOL_LOCAL: current-CPU's node.
- MPOL_PREFERRED_MANY (newer): prefer set, fall back.
- MPOL_WEIGHTED_INTERLEAVE: weighted-RR in nodeset.

REQ-2: Per-policy flags (top-bits of mode):
- MPOL_F_STATIC_NODES: nodemask interpreted as physical-node IDs.
- MPOL_F_RELATIVE_NODES: nodemask relative to cpuset.
- MPOL_F_NUMA_BALANCING: enable NUMA balancing.

REQ-3: set_mempolicy(2):
- syscall_set_mempolicy(mode, nmask, maxnode).
- per-task: current.mempolicy = new policy.
- per-task default for subsequent allocs.

REQ-4: mbind(2):
- syscall_mbind(start, len, mode, nmask, maxnode, flags).
- per-VMA range: vma.vm_policy = new policy.
- per-MPOL_MF_MOVE: migrate existing pages.
- per-MPOL_MF_STRICT: error if any page outside nodeset.

REQ-5: get_mempolicy(2):
- per-mode + per-nodemask query.
- MPOL_F_NODE: query node of given addr.
- MPOL_F_MEMS_ALLOWED: query effective-allowed-nodes.

REQ-6: Per-page-alloc dispatch (alloc_pages_mpol):
- if !pol ∨ pol.mode == MPOL_DEFAULT: per-default zonelist.
- elif pol.mode == MPOL_INTERLEAVE: nid = interleave_nodes(pol).
- elif pol.mode == MPOL_PREFERRED: nid = pol.preferred_node.
- elif pol.mode == MPOL_BIND: zonelist filtered by pol.nodemask.
- elif pol.mode == MPOL_LOCAL: nid = numa_node_id().

REQ-7: interleave_nodes:
- per-task interleave-counter (current.il_prev).
- next_node_in(pol.nodemask, current.il_prev).

REQ-8: Per-cpuset rebind:
- cpuset_attach moves task between cpusets → mempolicy.nodemask remapped per RELATIVE_NODES flag.
- mpol_rebind_policy walks per-task, per-VMA.

REQ-9: huge_node (THP):
- per-VMA huge-page allocation: pol-aware node-pick.
- Per-MPOL_INTERLEAVE: per-PMD's offset for index.

REQ-10: Per-mbind MOVE/MOVE_ALL:
- migrate_pages with target = preferred-node or interleave-step.
- Per-MOVE_ALL: bypasses migrate-may-fail policy.

REQ-11: Per-NUMA-balancing:
- Per-task scan VMAs: mark PROT_NONE → next access NUMA-fault.
- migrate_misplaced_folio per task's run-CPU node.
- Throttled by /proc/sys/kernel/numa_balancing_*.

REQ-12: Per-task mempolicy lifetime:
- Per-fork: child gets parent's mempolicy (mpol_dup).
- Per-exec: reset to default.
- Per-task-exit: mpol_put.

REQ-13: Per-cpuset interaction:
- mempolicy.nodemask intersected with cpuset's allowed-nodes.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `mode_in_range` | INVARIANT | pol.mode ∈ MPOL_*. |
| `nodemask_within_max` | INVARIANT | pol.v.nodes ⊆ node_possible_map. |
| `preferred_in_nodes_or_no_node` | INVARIANT | MPOL_PREFERRED: preferred_node ∈ nodes ∨ NUMA_NO_NODE. |
| `refcnt_balanced` | INVARIANT | per-Mempolicy alloc balanced via mpol_put. |
| `interleave_advances` | INVARIANT | interleave_nodes returns next node ≠ il_prev. |

### Layer 2: TLA+

`mm/mempolicy.tla`:
- Per-task / per-VMA mempolicy + per-cpuset rebind.
- Properties:
  - `safety_BIND_strict` — per-MPOL_BIND alloc: chosen node ∈ pol.v.nodes.
  - `safety_INTERLEAVE_uniform` — per-INTERLEAVE: long-run uniform across nodes.
  - `liveness_rebind_eventually_propagates` — per-cpuset-move: per-task pol.nodes remapped.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Mempolicy::do_set_mempolicy` post: task.mempolicy=new; old released | `Mempolicy::do_set_mempolicy` |
| `Mempolicy::do_mbind` post: per-VMA-in-range vm_policy=new; old released | `Mempolicy::do_mbind` |
| `Mempolicy::alloc_pages` post: chosen-node satisfies pol-mode | `Mempolicy::alloc_pages` |
| `Mempolicy::interleave_nodes` post: returned-node ∈ pol.v.nodes | `Mempolicy::interleave_nodes` |
| `Mempolicy::rebind` post: pol.v.nodes mapped per-flags | `Mempolicy::rebind` |

### Layer 4: Verus/Creusot functional

`Per-task / per-VMA mempolicy → per-alloc node-pick + per-mbind range-migrate; per-cpuset-aware rebind` semantic equivalence: per-numa(7) + Documentation/admin-guide/mm/numa_memory_policy.rst.

### hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening.)

NUMA-mempolicy reinforcement:

- **Per-mode validated** — defense against per-config invalid mode.
- **Per-nodemask within possible** — defense against per-config invalid node.
- **Per-cpuset-aware rebind** — defense against per-task-move stale nodemask.
- **Per-MPOL_BIND strict** — defense against per-fall-back violating per-app contract.
- **Per-mbind MOVE_ALL CAP_SYS_NICE** — defense against per-unprivileged force-migrate.
- **Per-il_prev per-task** — defense against per-cross-task interleave-state interference.
- **Per-mpol_put on old policy** — defense against per-set-mempolicy memory-leak.
- **Per-MPOL_PREFERRED_MANY backoff** — defense against per-strict-bind alloc-fail.
- **Per-WEIGHTED_INTERLEAVE per-node-weight cap** — defense against per-config insane weight.
- **Per-NUMA-balancing rate-limited** — defense against per-thrash.

