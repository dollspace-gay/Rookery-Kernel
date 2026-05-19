# Tier-3: kernel/sched/topology — sched-domain hierarchy + root domains

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - kernel/sched/topology.c
  - include/linux/sched/topology.h
-->

## Summary
Tier-3 design for the scheduler-topology subsystem: builds the per-CPU `struct sched_domain` hierarchy used by the load-balancer (`kernel/sched/load-balance.md`) and the per-root-domain `struct root_domain` shared by RT/DL classes (`kernel/sched/rt.md`, `kernel/sched/deadline.md`).

Topology is rebuilt lazily on CPU hotplug, cpuset reconfiguration, sched-domain attribute change, NUMA topology change. Construction reads cpu-to-numa-node + cpu-to-cluster + cpu-to-llc-cache mappings from arch code (`drivers/base/topology.c` + arch-specific topology setup) and produces sched-domains for every level (SMT, MC, DIE, NUMA-1, NUMA-2…).

Sub-tier-3 of `kernel/sched/00-overview.md`. Pairs with `arch/x86/kernel-platform.md` (CPU topology probing) and `kernel/cpuset/00-overview.md` (cpuset edits trigger rebuild).

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| sched-domain construction + tear-down | `kernel/sched/topology.c` |
| Public API | `include/linux/sched/topology.h` |

## Compatibility contract

### sched-domain levels (NUMA box example)

For a 2-socket, 16-core, 2-thread/core NUMA box:

```
SMT (2 threads / core)
 └─ MC (16 cores in socket; share LLC)
     └─ NUMA-1 (2 sockets; remote-node distance < threshold)
```

Each level exposes flags (e.g., `SD_SHARE_CPUCAPACITY` at SMT, `SD_SHARE_PKG_RESOURCES` at MC).

Identical level construction so `lstopo`, `numactl --hardware`, `chrt`, and `taskset` see identical sched-domain shape.

### `/sys/kernel/debug/sched/domains/cpu*/`

Per-CPU per-domain directory exposing `name`, `flags`, `min_interval`, `max_interval`, `busy_factor`, `imbalance_pct`, `cache_nice_tries`, `flags`, `groups_flags`. r/w sysctls update the live sched-domain set.

Path + file format identical so existing tools (e.g., `tuned`'s `sched-tuning` profile) work unchanged.

### `struct sched_domain` layout

Layout-equivalent. Key fields: `parent`, `child`, `groups`, `min_interval`, `max_interval`, `busy_factor`, `imbalance_pct`, `cache_nice_tries`, `flags`, `level`, `nr_balance_failed`, `last_balance`, `balance_interval`, `name`, `private`.

### `struct root_domain` layout

Per-root-domain shared state for RT/DL classes: `dlo_mask`, `cpupri`, `cpudl`, `online`, `span`, `def_root_domain`. Layout-equivalent.

### Topology-change triggers

| Trigger | Reaction |
|---|---|
| CPU hotplug add/remove | partial rebuild of affected domain branch |
| `cpuset` reconfig | rebuild affected partition |
| `/proc/sys/kernel/sched_schedstats` toggle | per-domain stats reset |
| Sched-domain flag tunable write | rebuild affected domain |
| ACPI SRAT / SLIT update (rare) | NUMA-distance-driven rebuild |

Identical triggers + reaction.

## Requirements

- REQ-1: Per-CPU sched-domain hierarchy: SMT → MC → DIE → NUMA-N levels constructed identically to upstream from arch-supplied cpu-topology.
- REQ-2: Per-level flags: SMT=`SD_SHARE_CPUCAPACITY|SD_SHARE_PKG_RESOURCES`; MC=`SD_SHARE_PKG_RESOURCES`; DIE=`(none of the SHARE_*)`; NUMA-N=`SD_NUMA`. Identical bitmask sets.
- REQ-3: `struct sched_domain` layout-equivalent.
- REQ-4: `struct root_domain` layout-equivalent; one per-cpuset-partition.
- REQ-5: NUMA distance table consumed from arch (`numa_distance` array); used to determine NUMA-level boundaries (cutoff at `LOCAL_DISTANCE`).
- REQ-6: CPU hotplug rebuild: `partition_sched_domains_locked` correctly invalidates + rebuilds only affected domain branches; doesn't disturb unaffected CPUs' rqs.
- REQ-7: cpuset partition rebuild: `partition_sched_domains` accepts a partition descriptor and rebuilds; identical algorithm.
- REQ-8: `/sys/kernel/debug/sched/domains/cpu*/` directory hierarchy + r/w files identical.
- REQ-9: `cpuset` "isolated CPUs" support: CPUs in `isolated` partition get domains with `SD_LOAD_BALANCE=0` so balancer skips them. Identical.
- REQ-10: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: A 2-socket NUMA box exposes `cpu0/domain{0,1,2}/name` reading `SMT`/`MC`/`NUMA` (or upstream-equivalent names depending on topology). (covers REQ-1, REQ-2)
- [ ] AC-2: `pahole struct sched_domain` byte-identical first-cache-line. (covers REQ-3)
- [ ] AC-3: `pahole struct root_domain` byte-identical first-cache-line. (covers REQ-4)
- [ ] AC-4: CPU hotplug: `echo 0 > /sys/devices/system/cpu/cpu5/online` then `echo 1 > ...` → domain hierarchy reconstructs; surrounding CPUs' `last_balance` not perturbed. (covers REQ-6)
- [ ] AC-5: cpuset edit: `mkdir /sys/fs/cgroup/cs.A; echo 0-3 > cs.A/cpuset.cpus; echo 1 > cs.A/cpuset.cpu_partition` → partition rebuilds; CPUs 0-3 get an isolated root_domain. (covers REQ-7)
- [ ] AC-6: A CPU placed in "isolated" partition has `SD_LOAD_BALANCE=0` on all its sched-domains; load_balance is a no-op for it. (covers REQ-9)
- [ ] AC-7: NUMA distance synthetic test: a 4-node configuration with SLIT distances `{10, 21, 31, 31}` produces NUMA-distance levels at 21 and 31 cutoffs. (covers REQ-5)
- [ ] AC-8: Hardening section present and follows template. (covers REQ-10)

## Architecture

### Rust module organization

- `kernel::sched::topology::TopologyBuilder` — top-level constructor
- `kernel::sched::topology::SchedDomain` — per-CPU per-level domain
- `kernel::sched::topology::SchedGroup` — per-domain CPU grouping
- `kernel::sched::topology::RootDomain` — per-partition shared RT/DL state
- `kernel::sched::topology::partition::Partitioner` — `partition_sched_domains_locked`
- `kernel::sched::topology::numa::NumaDistance` — NUMA-distance table consumer
- `kernel::sched::topology::debugfs::SchedDomainsDebugFs` — `/sys/kernel/debug/sched/domains/*`

### Locking and concurrency

- **`sched_domains_mutex`**: per-system mutex; serializes any rebuild
- **`cpu_hotplug_lock`** (read held by partition rebuild path)
- **Per-rq lock**: held briefly during `cpu_attach_domain` rcu-pointer update

The rcu-pointer update for a CPU's `rq->sd` allows balancer hot-paths to read sched-domains without locks (rcu_read_lock).

### Error handling

- `Err(ENOMEM)` — kmalloc fail during build → fallback to `def_root_domain` + null sched-domains for affected CPUs
- `Err(EINVAL)` — bad partition descriptor

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| `partition_sched_domains_locked` rcu-update + free-after-grace | `kani::proofs::sched::topology::partition_safety` |
| `cpu_attach_domain` rcu-pointer swap + old-domain free path | `kani::proofs::sched::topology::attach_safety` |
| sched_group cyclic-list construction (avoid double-link) | `kani::proofs::sched::topology::group_list_safety` |
| NUMA-distance level cutoff arithmetic | `kani::proofs::sched::topology::numa_safety` |

### Layer 2: TLA+ models

(no dedicated model; topology is a one-shot construction whose correctness is captured by Kani Layer-3 invariants below)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-CPU sched-domain hierarchy | for every CPU `c`, walking `rq[c]->sd->parent` chain visits each level exactly once with monotonically-increasing `level`; `span` bitmaps satisfy `child.span ⊆ parent.span` | `kani::proofs::sched::topology::hierarchy_invariants` |
| sched_groups cyclic list | each group's `next` pointer eventually returns to itself; group masks union = domain span | `kani::proofs::sched::topology::group_invariants` |
| root_domain.span | union of CPU masks of all members exactly equals span | `kani::proofs::sched::topology::root_domain_invariants` |

### Layer 4: Functional correctness (opt-in)

- **NUMA-distance level partitioning theorem** via Creusot — proves: given a SLIT distance matrix, the level-cutoff algorithm produces a valid partition (transitive: nodes at level-N share level-N domain).

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | `root_domain` refcount uses `Refcount` (saturating) | § Mandatory |
| **CONSTIFY** | per-level flag-mask templates are `static const` | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: per-domain + per-group + per-root_domain slabs
- **SIZE_OVERFLOW**: NUMA-distance + level arithmetic uses checked operators

### Row-2 / GR-RBAC integration

- LSM hook `security_capable(CAP_SYS_ADMIN)` already required for `/sys/kernel/debug/sched/domains/*` writes; GR-RBAC policy further can deny per-subject.
- LSM hook on cpuset partition mutation already covered in `kernel/cgroup/cpuset.md`.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — `/proc/sys/kernel/sched_domain/*` and isolcpus sysfs read/write bounds-checked; no `sched_domain` slab leakage.
- **PAX_KERNEXEC** — `build_sched_domains`, `partition_sched_domains_locked`, and per-level setup callbacks reside in W^X kernel text.
- **PAX_RANDKSTACK** — per-syscall kstack offset so cpuset-rebuild grooming via tight cgroup attach/detach cannot rely on predictable stack alignment.
- **PAX_REFCOUNT** — `sched_domain` ref, `sched_domain_topology_level` ref, and `root_domain->refcount` saturating-refcounted; underflow on hotplug oopses.
- **PAX_MEMORY_SANITIZE** — freed `sched_domain` / `sched_group` / `sched_group_capacity` slabs scrubbed during topology rebuild.
- **PAX_UDEREF** — topology walks read CPU masks and capacity values via kernel mappings only; no implicit user-page reach.
- **PAX_RAP / kCFI** — `sched_domain_flags_f` and per-level `sd_init` callbacks type-signatured; mismatched signature is a hard CFI fault.
- **GRKERNSEC_HIDESYM** — `sched_domain_topology`, `default_topology`, `root_domain` symbol addresses hidden from `/proc/kallsyms`.
- **GRKERNSEC_DMESG** — topology-build WARN splats (broken domain hierarchy, empty `sg_capacity`) gated to CAP_SYSLOG.
- **sched_domain hierarchy CAP_SYS_ADMIN** — mutating sched-domain flags, isolcpus, or cpuset hierarchies requires CAP_SYS_ADMIN; PAX_USERCOPY guarantees the attribute write traverses capability check first.
- **isolcpus integration** — `housekeeping_cpumask` / `isolcpus=` taints validated at boot and locked under `cpus_read_lock()`; an attacker cannot post-hoc re-attach a SCHED_OTHER task to an isolated CPU without CAP_SYS_NICE + cpuset rebuild.
- **Rationale** — sched-domain topology is the basis of every cross-CPU placement decision (load-balance, NUMA, EAS, isolated/nohz_full partitions). A corrupted topology is corrupted placement plus a wedge into RCU/timer infrastructure that assumes the domain hierarchy is well-formed.

## Open Questions

(none — sched-domain topology is exhaustively specified by upstream)

## Out of Scope

- Load-balancing algorithm itself (cross-ref `kernel/sched/load-balance.md`)
- cpuset implementation (cross-ref `kernel/cgroup/cpuset.md`)
- Arch-specific cpu-topology probe (cross-ref `arch/x86/kernel-platform.md`, `drivers/base/topology.c`)
- 32-bit-only paths
- Implementation code
