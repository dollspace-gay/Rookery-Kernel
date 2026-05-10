---
title: "Tier-3: kernel/cgroup/cpuset.c — cpuset cgroup controller (per-cgroup CPU + NUMA-node mask + load-balance partitioning + isolcpus)"
tags: ["tier-3", "kernel-cgroup", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

cpuset is the cgroup controller that constrains per-cgroup task affinity to a CPU subset + NUMA-node subset. Foundation of: kubernetes pod cpu pinning (kubelet writes `cpuset.cpus` per pod), systemd `CPUAffinity=` directive, container-runtime cpuset isolation (containerd, podman, docker), Workload-Manager cpuset partitioning (slurm, lsf, openpbs), real-time kernel isolation (per-cgroup-RT exclusive CPUs), NUMA-aware database deployments (per-node memory affinity).

Two scheduler interactions: (1) per-task `cpus_allowed` mask intersected with cpuset membership at every wake/migrate; (2) sched-domain partitioning — exclusive cpusets with `cpuset.sched_load_balance=0` create per-cpuset sched-domains, eliminating cross-cpuset load-balance traffic.

This Tier-3 covers `kernel/cgroup/cpuset.c` (~4400 lines).

### Acceptance Criteria

- [ ] AC-1: kubernetes pod with `requests.cpu=2,limits.cpu=2` + cpu-manager-policy=static → kubelet creates cgroup with `cpuset.cpus=4,5` (or whichever CPUs); pod tasks pinned.
- [ ] AC-2: systemd `CPUAffinity=0,2,4,6` per-service → service tasks restricted to 4 CPUs.
- [ ] AC-3: Partition root test: create cgroup A with `cpuset.cpus.exclusive=0-3 cpuset.cpus.partition=root` → CPUs 0-3 removed from parent's effective + sched-domain isolated.
- [ ] AC-4: Isolated partition test: same as AC-3 with partition=isolated → no in-cgroup load-balance; manual placement via `taskset` required.
- [ ] AC-5: NUMA migration test: cpuset.mems=0 → tasks alloc on node 0; change to cpuset.mems=1 with memory_migrate=1 → existing pages migrated to node 1.
- [ ] AC-6: CPU hotplug stress: offline a CPU in active cpuset → tasks migrate cleanly; online → cpuset.cpus.effective restored.
- [ ] AC-7: sched_setaffinity validation: task in cpuset cpus=0-3 calling sched_setaffinity(mask=4-7) returns -EINVAL (cross-cpuset).
- [ ] AC-8: kselftest `tools/testing/selftests/cgroup/test_cpuset_*.sh` passes.

### Architecture

`Cpuset` lives in `kernel::cgroup::cpuset::Cpuset`:

```
struct Cpuset {
  css: CgroupSubsysState,
  cpus_allowed: CpuMask,
  cpus_effective: CpuMask,
  exclusive_cpus: CpuMask,
  exclusive_cpus_effective: CpuMask,
  mems_allowed: NodeMask,
  mems_effective: NodeMask,
  flags: u64,
  partition_root_state: PartitionRootState,  // MEMBER / ROOT / ISOLATED
  parent_cs: AtomicPtr<Cpuset>,
  prs_lock: SpinLock<()>,
  attach_in_progress: AtomicI32,
  old_mems_allowed: NodeMask,
  pn: KBox<PressureNotify>,
  fmeter: KBox<FreqMeter>,
  remote_sibling: ListEntry,
  use_parent_ecpus: AtomicBool,
  child_ecpus_count: AtomicI32,
}
```

Subsystem init `Subsystem::init`:
1. Allocate `top_cpuset` (root cgroup's cpuset).
2. `top_cpuset.cpus_allowed = cpu_possible_mask`.
3. `top_cpuset.mems_allowed = node_possible_map`.
4. Register cpuset_cgrp_subsys with cgroup core.

Per-cgroup css_alloc `Cpuset::css_alloc(parent)`:
1. Allocate Cpuset struct.
2. Inherit `cpus_allowed` from parent.
3. Inherit `mems_allowed` from parent.
4. partition_root_state = MEMBER.
5. Return css.

Write `cpuset.cpus`:
1. Parse list-format string into CpuMask.
2. Validate: must be subset of parent's cpus.effective AND non-empty.
3. Take cgroup_mutex.
4. Update cs.cpus_allowed.
5. Recompute cs.cpus_effective = cs.cpus_allowed AND parent.cpus_effective AND cpu_active_mask.
6. Recursively recompute descendants.
7. For each task in cgroup: per-task `cpus_allowed` updated via `set_cpus_allowed_ptr(task, &new_mask)` (cross-ref `kernel/sched/core.md`).
8. If partition root: trigger `rebuild_sched_domains_locked()`.
9. Drop cgroup_mutex.

Per-task attach `Cpuset::attach`:
1. For each task in `threadgroup`:
   - Compute new cpu mask = task.original_cpus_allowed AND new_cs.cpus_effective.
   - `set_cpus_allowed_ptr(task, &new_mask)`.
   - Apply mems_allowed: task.mems_allowed = new_cs.mems_effective.
   - If memory_migrate: schedule task page migration.

Partition root state transition `Cpuset::update_prstate(cs, new_prs)`:
1. Validate transition allowed (e.g., MEMBER→ROOT requires cpuset.cpus.exclusive non-empty).
2. Take cpus_write_lock.
3. Update cs.partition_root_state = new_prs.
4. Recompute parent's effective_xcpus (subtract cs's exclusive_cpus).
5. `rebuild_sched_domains_locked()` to update sched-domain partitioning.
6. Drop cpus_write_lock.

CPU hotplug callback `cpuset_update_active_cpus()`:
1. Walk all cpusets in tree.
2. For each: cpus_effective = cpus_allowed AND parent.cpus_effective AND cpu_active_mask.
3. For each task: if current CPU not in new cpus_effective, migrate via `migrate_one_irq` + `set_cpus_allowed_ptr`.
4. `rebuild_sched_domains()` if any partition root affected.

Sched-domain partitioning: `rebuild_sched_domains_locked()`:
1. Walk cpusets in BFS order.
2. For each partition root: collect cpus.effective into a sched-domain mask.
3. Build per-partition sched-domain hierarchy (cross-ref `kernel/sched/topology.md`).
4. Cross-partition load-balance: for "isolated" partitions, exclude from any sched-domain (no balance).

PSI integration: per-cpuset `cpuset.memory_pressure` reads from per-cgroup PSI accumulators (cross-ref `kernel/sched/psi.md`).

### Out of Scope

- cgroup-v1 / v2 framework (covered in `kernel/cgroup/cgroup-core.md` Tier-3)
- sched-domain construction (covered in `kernel/sched/topology.md` Tier-3)
- Per-task sched_setaffinity (covered in `kernel/sched/core.md` Tier-3)
- cgroup-rstat (covered in `kernel/cgroup/rstat.md` Tier-3)
- Per-cgroup mem controller (covered in `kernel/cgroup/memcg.md` future Tier-3)
- 32-bit-only paths
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct cpuset` | per-cgroup cpuset state | `kernel::cgroup::cpuset::Cpuset` |
| `cpuset_init()` | subsystem init | `Subsystem::init` |
| `cpuset_init_smp()` | post-SMP init (top_cpuset = all-cpus) | `Subsystem::init_smp` |
| `cpuset_css_alloc(parent)` / `_css_free(css)` | per-cgroup cpuset css alloc | `Cpuset::css_alloc` / `_css_free` |
| `cpuset_css_online(css)` / `_css_offline(css)` | per-cgroup css online/offline | `Cpuset::css_online` / `_css_offline` |
| `cpuset_can_attach(of, &threadgroup)` / `_attach(...)` / `_cancel_attach(...)` | per-task attach validation + commit | `Cpuset::can_attach` / `_attach` / `_cancel_attach` |
| `cpuset_attach_task(p, cpus, mems)` | per-task attach (apply cpus_allowed + nodemask) | `Cpuset::attach_task` |
| `cpuset_fork(task, ...)` | per-task fork hook (inherit parent cpuset membership) | `Cpuset::fork_hook` |
| `cpuset_cpus_allowed(task)` / `_mems_allowed(task)` | per-task cpu/mem mask resolution | `Task::cpuset_cpus_allowed` / `_mems_allowed` |
| `cpuset_node_allowed(node, gfp)` | check if node is in current task's mems_allowed (for slab/page-alloc) | `Task::cpuset_node_allowed` |
| `cpuset_zone_allowed(z, gfp)` | per-zone variant | `Task::cpuset_zone_allowed` |
| `cpuset_mem_spread_node()` / `_slab_spread_node()` | per-task mem-spread node (round-robin across mems_allowed) | `Task::cpuset_mem_spread_node` |
| `current_cpuset_is_being_rebound()` | rebind-in-progress check | `Cpuset::is_rebinding` |
| `cpuset_update_active_cpus()` | per-CPU hotplug callback (recompute cpuset cpu masks) | `Cpuset::update_active_cpus` |
| `cpuset_cpus_allowed_fallback(task)` | fallback when no allowed CPUs (CPU hotplug) | `Task::cpuset_cpus_allowed_fallback` |
| `update_prstate(cs, new_prs)` | partition root state transition | `Cpuset::update_prstate` |
| `cpuset_can_fork(task, css_set)` / `_post_fork(...)` / `_cancel_fork(...)` | per-fork validation | `Cpuset::can_fork` / `_post_fork` / `_cancel_fork` |
| `rebuild_sched_domains()` | recompute sched-domains based on partition cpusets (cross-ref `kernel/sched/topology.md`) | `Subsystem::rebuild_sched_domains` |
| `update_partition_exclusive_flag(cs, new_xcpus)` | per-cpuset exclusive-cpu mask update | `Cpuset::update_partition_exclusive_flag` |
| `cpuset_cgrp_subsys` | per-cgroup-controller subsystem descriptor | `cpuset::SUBSYS_OPS` |

### compatibility contract

REQ-1: Per-cpuset attributes (cgroup-v2 + legacy v1):
- `cpuset.cpus` (writable, list-format string e.g. "0-3,8-11"): allowed CPUs.
- `cpuset.cpus.effective` (read-only, after intersect with parent + online CPUs): actual CPUs available.
- `cpuset.cpus.exclusive` (writable, v2): subset of cpuset.cpus reserved for exclusive use by this cgroup + descendants.
- `cpuset.cpus.exclusive.effective` (read-only).
- `cpuset.cpus.partition` (writable: "member" / "root" / "isolated"): partition root state.
- `cpuset.mems` (writable, list-format): allowed NUMA nodes.
- `cpuset.mems.effective` (read-only).
- `cpuset.mem_exclusive` (legacy v1).
- `cpuset.cpu_exclusive` (legacy v1).
- `cpuset.sched_load_balance` (legacy v1).
- `cpuset.memory_migrate` (writable: 0/1): migrate task pages on cpuset.mems change.
- `cpuset.memory_pressure` (read-only).
- `cpuset.memory_spread_page` / `_spread_slab` (writable v1; v2 default-on).

REQ-2: Per-cgroup membership change semantics:
- Write to cpuset.cpus → recompute effective masks for this cgroup + descendants → propagate to per-task `cpus_allowed` (intersect with task's pre-cgroup affinity) → trigger migration of any task currently on now-disallowed CPU.

REQ-3: Per-cgroup partition state:
- "member" (default): cpus shared with parent + siblings; load-balanced together.
- "root": cpus exclusively reserved from parent's pool; per-partition sched-domain created.
- "isolated": "root" + no in-cgroup load-balance; tasks pinned to specific CPUs (used for RT workloads, dpdk-style isolated polling).

REQ-4: CPU hotplug integration: when CPU goes offline, all cpusets with that CPU in `cpus.effective` recomputed; tasks pinned to offlined CPU migrated via per-task `cpuset_cpus_allowed_fallback` (cross-ref `kernel/sched/core.md`).

REQ-5: NUMA membership change: write to cpuset.mems with `memory_migrate=1` triggers per-task page migration to new node mask via `migrate_pages(task, ...)` (cross-ref `mm/00-overview.md`).

REQ-6: Per-task fork hook: child task inherits parent's cpuset membership; immediately apply cpus_allowed + mems_allowed.

REQ-7: Per-task `cpuset_cpus_allowed(task)` returns intersection of (task's per-task cpus_allowed) AND (cpuset.cpus.effective); used by sched_setaffinity validation + per-task CPU selection.

REQ-8: Sched-domain rebuild: cpuset partition changes trigger `rebuild_sched_domains_locked()` to recompute sched-domains; per-partition isolated rb-trees created; cross-partition load-balance traffic eliminated.

REQ-9: `cpuset.memory_pressure` read-only file: per-cpuset pressure-stall info from PSI (cross-ref `kernel/sched/psi.md` future Tier-3); shows cumulative reclaim time-stall in this cpuset.

REQ-10: `cpuset.memory_spread_page` / `_spread_slab` (when set): page allocator uses round-robin across mems_allowed instead of preferring local node; useful for tail-latency-sensitive workloads.

REQ-11: Cgroup-v1 vs v2 surface differences: v1 has separate `cpu_exclusive` + `mem_exclusive` + `sched_load_balance` files; v2 unified into `cpuset.cpus.exclusive` + `cpuset.cpus.partition`. v0 supports both (cross-ref `kernel/cgroup/00-overview.md`).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cpuset_no_uaf` | UAF | `Arc<Cpuset>` outlives all attached tasks; css_offline waits for refcount==0. |
| `cpus_effective_consistent` | INVARIANT | cs.cpus_effective ⊆ cs.cpus_allowed ⊆ parent.cpus_effective ∪ {topcpus}. |
| `partition_state_valid` | TRANSITION | partition_root_state transitions only follow allowed sequences (MEMBER↔ROOT, ROOT↔ISOLATED, ROOT→MEMBER). |
| `task_cpus_allowed_subset` | INVARIANT | task.cpus_allowed ⊆ cs.cpus_effective for cgroup containing task. |

### Layer 2: TLA+

`models/cgroup/cpuset_partition.tla` (parent-declared in `kernel/cgroup/00-overview.md`): proves partition root state machine + concurrent cpu hotplug + concurrent partition transition never produce inconsistent sched-domain.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Cpuset::attach_task` post: task.cpus_allowed ⊆ cs.cpus_effective; task.mems_allowed ⊆ cs.mems_effective | `Cpuset::attach_task` |
| `Cpuset::update_prstate(MEMBER → ROOT)` post: parent's effective_xcpus = parent's cpus_allowed − all-children's exclusive_cpus | `Cpuset::update_prstate` |
| CPU hotplug invariant: post-hotplug, every task is on a CPU in its cgroup's cpus_effective | `cpuset_update_active_cpus` |

### Layer 4: Verus/Creusot functional

`Cpuset::attach + Cpuset::update_cpus + sched_setaffinity` round-trip equivalence: per-task cpus_allowed at any instant equals intersection of (task's pre-cgroup affinity) AND (per-cgroup cpus_effective).

### hardening

(Inherits row-1 features from `kernel/cgroup/00-overview.md` § Hardening.)

cpuset-specific reinforcement:

- **Per-cgroup cpus.effective recompute under cgroup_mutex** — defense against concurrent cpu_hotplug + partition transition causing inconsistent state.
- **Partition root state transition validated** — invalid transitions (e.g., ISOLATED→MEMBER without going through ROOT) rejected.
- **CPU hotplug fallback** — if cgroup's cpus.effective becomes empty post-hotplug, tasks migrated to parent cgroup's cpus.effective; if parent also empty, fallback to top_cpuset (with WARN).
- **Per-task cpus_allowed validation** — `sched_setaffinity` syscall (cross-ref `kernel/sched/core.md`) intersects requested mask with current cgroup's cpus.effective; user-supplied mask outside cpuset rejected with -EINVAL.
- **NUMA-page migration triggered only with explicit `memory_migrate=1`** — defense against accidental page-thrashing from cpuset.mems change.
- **Partition exclusive-cpu enforcement** — child cgroup attempting to use exclusive-cpu owned by parent's other partition rejected.
- **Per-cgroup mems.effective restricts page allocator** — `cpuset_node_allowed` check in slab/page allocator paths; defense against per-cgroup memory leakage to disallowed nodes.

