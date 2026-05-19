# Tier-3: kernel/sched/cfs — CFS / EEVDF scheduler

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - kernel/sched/fair.c
  - kernel/sched/sched.h
  - kernel/sched/core.c
  - kernel/sched/loadavg.c
  - kernel/sched/pelt.c
  - kernel/sched/pelt.h
  - kernel/sched/topology.c
  - kernel/sched/cpufreq.c
  - kernel/sched/cpufreq_schedutil.c
  - kernel/sched/clock.c
  - kernel/sched/autogroup.c
  - kernel/sched/build_policy.c
  - kernel/sched/build_utility.c
  - kernel/sched/cputime.c
  - kernel/sched/cpuacct.c
  - kernel/sched/idle.c
  - kernel/sched/isolation.c
  - kernel/sched/membarrier.c
  - kernel/sched/wait.c
  - kernel/sched/wait_bit.c
  - kernel/sched/swait.c
  - include/linux/sched.h
  - include/linux/sched/topology.h
  - include/uapi/linux/sched.h
  - include/uapi/linux/sched/types.h
-->

## Summary
Tier-3 design for the kernel's default scheduler class — CFS / EEVDF (the `kernel/sched/fair.c` content). Linux 7.x has migrated from classic CFS to EEVDF (Earliest Eligible Virtual Deadline First) semantics within the same `fair.c` file. This Tier-3 owns the per-CPU runqueue (rbtree of `sched_entity` ordered by virtual deadline), task wake-up, load tracking (PELT — Per-Entity Load Tracking), CPU-frequency scaling (schedutil), the per-task autogroup mechanism, runqueue load-balancing across CPUs/NUMA nodes, and idle handling.

This is **the fifth and final MANDATORY Layer-3-Kani-harness subsystem** per `00-overview.md` D4 (the others being mm: page-allocator + slab + virtual-memory; arch/x86: paging; fs: vfs/dcache). Runqueue ordering invariants — leftmost-rbtree-node has earliest virtual deadline among runnable tasks on this CPU — are mechanically verified.

The scheduler is the most concurrency-sensitive single component of the kernel: every task wake/sleep/migration interleaves with every other CPU's runqueue.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| CFS / EEVDF core | `kernel/sched/fair.c` |
| Common scheduler infrastructure (rq, sched_class, switch_to) | `kernel/sched/sched.h`, `kernel/sched/core.c` |
| Load average computation | `kernel/sched/loadavg.c` |
| Per-Entity Load Tracking | `kernel/sched/pelt.c`, `kernel/sched/pelt.h` |
| NUMA + cache topology | `kernel/sched/topology.c`, `include/linux/sched/topology.h` |
| CPU frequency selection | `kernel/sched/cpufreq.c`, `kernel/sched/cpufreq_schedutil.c` |
| Sched-clock (sched_clock_t monotonic-ish source) | `kernel/sched/clock.c` |
| Autogroup (per-tty automatic grouping) | `kernel/sched/autogroup.c` |
| CPU time accounting | `kernel/sched/cputime.c`, `kernel/sched/cpuacct.c` |
| Idle scheduler class (when no task runnable) | `kernel/sched/idle.c` |
| CPU isolation (`isolcpus=`, nohz_full) | `kernel/sched/isolation.c` |
| Membarrier syscall | `kernel/sched/membarrier.c` |
| Wait queues | `kernel/sched/wait.c`, `kernel/sched/wait_bit.c`, `kernel/sched/swait.c` |
| Build helpers (single-file aggregation) | `kernel/sched/build_policy.c`, `kernel/sched/build_utility.c` |
| Public types | `include/linux/sched.h`, `include/uapi/linux/sched.h`, `include/uapi/linux/sched/types.h` |

## Compatibility contract

### `/proc/<pid>/sched`, `/proc/<pid>/schedstat`

Format-identical. Per-task scheduler stats: vruntime, nr_voluntary_switches, nr_involuntary_switches, etc.

### `/proc/sched_debug`, `/proc/schedstat`

System-wide scheduler debug: per-CPU runqueue dump, per-task vruntime, runnable counts. Format-identical.

### `/proc/loadavg`

`<1m> <5m> <15m> <runnable>/<total> <last_pid>` — format-identical.

### Sysctls (`/proc/sys/kernel/sched_*`)

- `sched_child_runs_first` (boolean)
- `sched_compat_yield`
- `sched_min_granularity_ns`
- `sched_latency_ns`
- `sched_wakeup_granularity_ns`
- `sched_migration_cost_ns`
- `sched_cfs_bandwidth_slice_us`
- `sched_rr_timeslice_ms` (RT)
- `sched_rt_period_us` / `sched_rt_runtime_us` (RT bandwidth)
- `sched_deadline_period_min_us` / `sched_deadline_period_max_us` (DL admission)
- `sched_util_clamp_min` / `sched_util_clamp_max` (util-clamp)
- `sched_energy_aware`
- `sched_uclamp_used`

Numeric values + range identical.

### Syscalls (cross-ref `kernel/00-overview.md` Compatibility contract)

`sched_setaffinity`, `sched_getaffinity`, `sched_setscheduler`, `sched_getscheduler`, `sched_setparam`, `sched_getparam`, `sched_setattr`, `sched_getattr`, `sched_yield`, `sched_get_priority_min`, `sched_get_priority_max`, `sched_rr_get_interval`, `nice`, `setpriority`, `getpriority`, `sched_get_period`, `sched_get_runtime`. Each gets a Tier-5 doc.

`SCHED_*` policy enum: `SCHED_NORMAL=0` (CFS/EEVDF), `SCHED_FIFO=1`, `SCHED_RR=2`, `SCHED_BATCH=3`, `SCHED_IDLE=5`, `SCHED_DEADLINE=6`, `SCHED_EXT=7` (cross-ref `kernel/sched/ext.md`). Numeric values byte-identical.

### Cgroup integration

cgroup v2 `cpu` controller knobs: `cpu.max`, `cpu.weight`, `cpu.weight.nice`, `cpu.uclamp.min`, `cpu.uclamp.max`, `cpu.stat`, `cpu.idle`, `cpu.pressure`. Format-identical.

### Userspace-observable behavior

The scheduler's nice→weight mapping, default time slice, wake-up preemption decisions all match upstream so reference workloads exhibit identical scheduling outcomes within ±5% (per `kernel/00-overview.md` AC-3).

## Requirements

- REQ-1: CFS / EEVDF: per-CPU runqueue is an rbtree of `struct sched_entity` ordered by virtual deadline (`se->deadline`). The leftmost node is the next picked task per `pick_eevdf` algorithm. Identical to upstream `kernel/sched/fair.c::pick_eevdf`.
- REQ-2: Virtual runtime (`se->vruntime`) accounting: weighted by nice value via the upstream `sched_prio_to_weight` table (40 entries, prios -20 to +19). Each tick: `se->vruntime += delta_exec * NICE_0_LOAD / se->load.weight`.
- REQ-3: PELT (Per-Entity Load Tracking): per-entity exponentially-decayed load average; same decay constant + windows as upstream.
- REQ-4: Wakeup preemption: when a sleeping task wakes, decide whether to preempt the running task per `wakeup_preempt` algorithm. Identical to upstream.
- REQ-5: Load balancing across CPUs: every `LB_INTERVAL` ms, each CPU's `run_rebalance_domains` evaluates whether to migrate tasks. Domain hierarchy (SMT → MC → DIE → NUMA) per upstream's topology detection.
- REQ-6: NUMA balancing (when CONFIG_NUMA_BALANCING=y): periodic per-task page-fault triggers based on access patterns; preferred-node selection per upstream's `numa_preferred_nid` algorithm.
- REQ-7: schedutil cpufreq governor: when active, drives frequency from PELT util signal + utilization-clamps; identical decision rules.
- REQ-8: Autogroup: per-session-id automatic task grouping; identical `/proc/<pid>/autogroup` interface.
- REQ-9: idle scheduler class: when no other class has runnable tasks, runs the per-CPU idle task; integrates with cpu_idle_loop for C-state selection.
- REQ-10: CPU isolation (`isolcpus=`, `nohz_full=`, cgroup `cpuset.cpus`): isolated CPUs run only their assigned tasks; identical sysfs surface.
- REQ-11: Membarrier syscall: per upstream's MEMBARRIER_CMD_* set; serializes IPI delivery for cross-CPU memory ordering.
- REQ-12: Wait queues: per upstream's `wake_up_*`, `wait_event_*`, `prepare_to_wait_*` API set; sleepable + spinning variants.
- REQ-13: Layer-2 TLA+ models per `kernel/00-overview.md` Layer 2: runqueue, load_balance, preempt, qspinlock (cross-ref locking).
- REQ-14: Layer-3 invariant harnesses MANDATORY: CFS rbtree leftmost-node-has-smallest-vruntime; PI-graph acyclicity; cgroup hierarchical accounting (cross-ref `kernel/00-overview.md` D4).
- REQ-15: Layer-4 Verus opt-ins declared in `kernel/00-overview.md`: CFS fairness theorems (every runnable task eventually runs; weights produce proportional CPU time on average).
- REQ-16: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: A `schbench`-style microbenchmark on Rookery shows latency + throughput within ±5% of upstream on identical hardware. (covers REQ-1, REQ-3, REQ-4)
- [ ] AC-2: `nice -n 19 cpu_bound &; nice -n -10 cpu_bound &` shows the high-priority task receiving expected proportional CPU per `sched_prio_to_weight`. (covers REQ-2)
- [ ] AC-3: `sched_yield` test exercises the FIFO + RR + DEADLINE + NORMAL policies; behavior matches upstream. (covers per-policy semantics)
- [ ] AC-4: Load-balance test on multi-socket hardware: imbalanced workload migrates per upstream's domain hierarchy. (covers REQ-5)
- [ ] AC-5: NUMA-balancing test: a task accessing remote-node memory eventually migrates to the local CPU. (covers REQ-6)
- [ ] AC-6: schedutil frequency test: a CPU-bound workload pushes frequency to max; idle drops to min. (covers REQ-7)
- [ ] AC-7: autogroup test: two terminal sessions get distinct autogroups; CPU shares fair across sessions per upstream. (covers REQ-8)
- [ ] AC-8: idle test: `cat /proc/cpuinfo` reports the same C-state distribution as upstream after a quiet 10s. (covers REQ-9)
- [ ] AC-9: `isolcpus=2` boot: CPU 2 runs only its pinned task; verifiable via `/proc/<pid>/status Cpus_allowed`. (covers REQ-10)
- [ ] AC-10: `tools/testing/selftests/membarrier/` tests pass. (covers REQ-11)
- [ ] AC-11: A wait_event-driven test (e.g., a kthread sleeping on a condvar) wakes correctly on signal. (covers REQ-12)
- [ ] AC-12: `make tla` passes models/kernel/sched/{runqueue,load_balance,preempt}.tla + cross-ref locking models. (covers REQ-13)
- [ ] AC-13: `make verify` passes mandatory L3 harnesses for CFS rbtree + PI-graph + cgroup accounting. (covers REQ-14)
- [ ] AC-14: Layer-4 Verus proof of `pick_eevdf` + nice→weight mapping correctness compiles + verifies. (covers REQ-15)
- [ ] AC-15: Hardening section present and follows template. (covers REQ-16)

## Architecture

### Rust module organization

- `kernel::sched::core::Rq` — per-CPU runqueue
- `kernel::sched::core::SchedClass` — sched-class trait (CFS, RT, DL, EXT, IDLE all implement)
- `kernel::sched::fair::CfsRq` — CFS runqueue (rbtree of `SchedEntity`)
- `kernel::sched::fair::pick_eevdf` — EEVDF picker
- `kernel::sched::pelt` — PELT load tracking
- `kernel::sched::topology` — NUMA + cache topology + domain hierarchy
- `kernel::sched::loadavg` — load-average computation
- `kernel::sched::cpufreq::schedutil` — schedutil governor (cross-ref `drivers/cpufreq-cpuidle.md`)
- `kernel::sched::clock` — sched_clock substrate
- `kernel::sched::autogroup` — per-tty grouping
- `kernel::sched::cputime` — CPU-time accounting
- `kernel::sched::cpuacct` — cgroup CPU-account controller
- `kernel::sched::idle` — idle scheduler class
- `kernel::sched::isolation` — CPU isolation
- `kernel::sched::membarrier` — membarrier syscall
- `kernel::sched::wait` — wait queues + completion

### Key data structures

- `Rq` — per-CPU runqueue: `rq->lock` (raw_spinlock), runnable counts per class, current task pointer, idle task pointer
- `CfsRq` — CFS-class subqueue: rbtree, leftmost cache, total weight, vruntime min
- `SchedEntity` — represents a schedulable entity (task or task group): `vruntime`, `deadline`, `load.weight`, `runnable`, parent CFS-rq
- `SchedDomain` — topology domain at one level (SMT, MC, DIE, NUMA): cpumask, balance interval, balance flags
- `RtRq`, `DlRq`, `IdleRq` — per-class subqueues (cross-ref sub-docs `rt.md`, `deadline.md`, `idle.md`)

### Locking and concurrency

- **rq->lock** (raw_spinlock per CPU): held during enqueue/dequeue/schedule. CFS rbtree updates serialize via this.
- **task->pi_lock** (per-task spinlock): held during cross-CPU wakeup before acquiring rq->lock.
- **Cross-CPU coordination**: `try_to_wake_up` may need to take target CPU's rq->lock; uses `task_rq_lock` with retry.
- **NUMA-balancing fault path**: per-mm `mm->numa_lock` (rwsem); fault handler acquires read.
- **Topology rebuild**: `sched_domains_mutex` (mutex) — serializes topology changes (hotplug, cpuset reconfig).

TLA+ model `models/kernel/sched/runqueue.tla` (mandatory per `kernel/00-overview.md` Layer 2) proves:
- Mutual exclusion: at most one CPU's `__schedule` chooses a given task at a time
- Progress: every runnable task eventually runs (under fairness)
- Ordering: rbtree leftmost is always the smallest-vruntime runnable task

### Error handling

Most scheduler operations don't return Result (e.g., `schedule()` returns `()` — control flow doesn't fail). Specific syscalls do:
- `Err(EPERM)` — `sched_setscheduler` to RT/DL without CAP_SYS_NICE
- `Err(EINVAL)` — bad policy / priority
- `Err(EBUSY)` — DEADLINE admission test failed
- `Err(ESRCH)` — target task gone (rare; race window)

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| CFS rbtree leftmost-update | `kani::proofs::kernel::sched::fair::leftmost_safety` |
| Wakeup-preemption check | `kani::proofs::kernel::sched::fair::wakeup_preempt_safety` |
| PELT decay arithmetic | `kani::proofs::kernel::sched::pelt::decay_safety` |
| Load-balance task migration | `kani::proofs::kernel::sched::lb::migrate_safety` |
| Topology rebuild under hotplug | `kani::proofs::kernel::sched::topology::rebuild_safety` |
| Membarrier IPI sequence | `kani::proofs::kernel::sched::membarrier::ipi_safety` |

### Layer 2: TLA+ models (mandatory per `kernel/00-overview.md` Layer 2)

- `models/kernel/sched/runqueue.tla` — proves CFS rbtree ordering invariants under concurrent enqueue / dequeue / wakeup / migration. Owned here.
- `models/kernel/sched/load_balance.tla` — proves load-balancer's task migration preserves runqueue invariants. Owned here.
- `models/kernel/sched/preempt.tla` — proves preemption-disable / preemption-enable counter is well-balanced and never goes negative.
- `models/kernel/sched/eevdf_pick.tla` (NEW) — proves: among all runnable entities with deadline ≤ now + max_lag, `pick_eevdf` returns one with the earliest deadline.

### Layer 3: Kani harnesses for data-structure invariants (MANDATORY per `00-overview.md` D4)

| Data structure | Invariant | Harness |
|---|---|---|
| CFS rbtree (per CPU) | "The leftmost node has the smallest virtual deadline among runnable entities on this CPU" + "rbtree red-black invariants hold" | `kani::proofs::kernel::sched::cfs::rbtree_invariants` |
| Cgroup hierarchical CPU accounting | "Sum of children's vruntime weighted by load == parent's vruntime; CPU usage rolls up correctly" | `kani::proofs::kernel::sched::cgroup::accounting_invariants` |
| Domain hierarchy | "Every domain's cpumask is a subset of its parent's cpumask; cpusets respect domain boundaries" | `kani::proofs::kernel::sched::topology::domain_invariants` |
| Wait-queue list | "Every entry on a wait-queue has its `wait_queue_entry` linked exactly once" | `kani::proofs::kernel::sched::wait::queue_invariants` |

### Layer 4: Functional correctness (opt-in; declared in `kernel/00-overview.md` Layer 4)

- **CFS fairness theorem** via Verus — proves: under fair scheduling assumption + no priority changes, every runnable task receives CPU time proportional to its load weight, on average over a sufficiently-long window.
- **Nice→weight mapping correctness** via Creusot — proves: `sched_prio_to_weight[nice + 20]` produces the upstream-defined geometric series (each step factor 1.25).
- **`pick_eevdf` selection correctness** via Verus — proves: returns the entity with earliest virtual deadline among eligible runnables.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **PRIVATE_KSTACKS** | Per-task kernel stack used during `__schedule`'s switch_to() — preserves stack isolation per `00-security-principles.md` mandate | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT**: task refcount uses `Refcount` (saturating); cross-ref `kernel/task-lifecycle.md`
- **AUTOSLAB**: rq, sched_entity, sched_domain allocated via per-type slab caches
- **MEMORY_SANITIZE**: per-task scheduler structs zeroed on free
- **CONSTIFY**: `sched_class` vtables (CFS, RT, DL, EXT, IDLE) are `static const`
- **SIZE_OVERFLOW**: vruntime + load-weight arithmetic uses checked or saturating operators per upstream's algorithm

### Row-2 / GR-RBAC integration

`sched_setscheduler` syscalls invoke `security_task_setscheduler` LSM hook. GR-RBAC (per loaded policy) can deny RT / DL policy escalation as defense-in-depth above existing CAP_SYS_NICE check.

### Userspace-visible behavior changes

None beyond upstream defaults. Scheduler decisions are statistical; reference workloads should observe within ±5% of upstream behavior.

### Verification

(See § Verification above; CFS rbtree invariants are mandatory per `00-overview.md` D4.)

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — bounded copy on task/cred/sched_attr buffers (sched_setattr, sched_setaffinity, util-clamp UAPI).
- **PAX_KERNEXEC** — W^X for BPF JIT'd code (sched_ext programs that hook fair.c-equivalent callbacks), kprobe/uprobe trampolines on the scheduler tick.
- **PAX_RANDKSTACK** — per-syscall kernel-stack randomization; obscures CFS rbtree traversal stack frames between picks.
- **PAX_REFCOUNT** — saturating refcount on task_struct, task_group, cred, mm_struct; prevents wraparound during fast-path autogroup attach/detach.
- **PAX_MEMORY_SANITIZE** — zero-on-free for task_struct, signal_struct, cred, sched_entity; ensures freed `SchedEntity` cannot resurface as a stale rbtree node.
- **PAX_MEMORY_STACKLEAK** — kernel-stack zeroing on syscall exit; wipes leaked PELT-window arithmetic and load-balance migration stack frames.
- **PAX_UDEREF** — strict user-pointer access for cpumask + sched_attr UAPI copies.
- **PAX_RAP / kCFI** — indirect-call signature enforcement for `sched_class` vtable (pick_next_task_fair, enqueue_task_fair, task_tick_fair) and schedutil cpufreq governor callbacks.
- **GRKERNSEC_HIDESYM** — hide kernel addresses in /proc/<pid>/* + kallsyms; suppresses `sched_entity` and `cfs_rq` pointer leaks via /proc/sched_debug.
- **GRKERNSEC_HARDEN_PTRACE** — restrict ptrace cross-uid (Yama scope ≥ 1); blocks attacker from observing victim's vruntime drift.
- **GRKERNSEC_BRUTE** — exponential delay on consecutive brute attempts; throttles fork-bomb wake-storm probes against CFS pick.
- **GRKERNSEC_KSTACKOVERFLOW** — kernel-stack overflow guard against deep load-balance recursion across NUMA hierarchies.
- **GRKERNSEC_DMESG** — restrict syslog so sched-domain rebuild WARN traces leaking topology pointers are unreadable to unprivileged users.
- **GRKERNSEC_SYSCTL_DISABLE** — disable dangerous sysctls (kernel.sched_*, sched_min_granularity_ns tunables) by default behind admin gesture.
- **GRKERNSEC_CONFIG_AUDIT** — boot-time runtime-config integrity check; verifies CONFIG_FAIR_GROUP_SCHED / SCHED_AUTOGROUP / NUMA_BALANCING match signed config.

Per-doc rationale: CFS/EEVDF owns the rbtree and PELT arithmetic that every task on the system touches, plus the sched_ext BPF callback surface; a corrupted leftmost pointer or a stale `sched_entity` directly translates into kernel ROP via the `sched_class` indirect-call path. PaX RAP/kCFI binds the fair_sched_class vtable, MEMORY_SANITIZE prevents stale `SchedEntity` aliasing, and Grsecurity HIDESYM/DMESG closes the /proc/sched_debug enumeration channel that would otherwise expose runqueue layout to side-channel attackers.

## Open Questions

(none — CFS / EEVDF semantics are exhaustively specified by upstream Linux + the EEVDF paper)

## Out of Scope

- RT scheduler class (cross-ref `kernel/sched/rt.md` Tier-3 sub-doc)
- Deadline scheduler class (cross-ref `kernel/sched/deadline.md`)
- sched_ext (BPF-driven) class (cross-ref `kernel/sched/ext.md`)
- Idle subscheduler in-depth (cross-ref `kernel/sched/idle.md`)
- 32-bit-only paths
- Implementation code
