---
title: "Tier-3: kernel/sched/load-balance — periodic + idle + nohz load balancing"
tags: ["design-doc", "tier-3", "scheduler"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for the CFS load-balancer: the periodic + newly-idle + nohz-idle balancing paths inside `kernel/sched/fair.c` that move tasks between CPUs to equalize load across CPU groups in the sched-domain hierarchy. Uses Per-Entity Load Tracking (PELT) signals (`load_avg`, `runnable_avg`, `util_avg`) to make migration decisions.

Sub-tier-3 of `kernel/sched/cfs.md` and `kernel/sched/00-overview.md`. Pairs with `kernel/sched/topology.md` (sched-domain construction). Foundational for any multi-CPU performance: a misbehaving load balancer manifests as poor throughput on NUMA boxes and laptop battery drain (idle CPUs wake unnecessarily).

### Requirements

- REQ-1: Periodic balance: `run_rebalance_domains` invoked from `scheduler_tick` at the cadence dictated by per-domain `balance_interval`. Cadence + flag-respect identical.
- REQ-2: `load_balance(cpu, rq, sd, idle, continue_balancing)` — finds busiest group + queue in `sd`; calls `detach_tasks` + `attach_tasks` to migrate. Algorithm identical.
- REQ-3: `find_busiest_group` heuristic: avg_load + group_imbalance + group_util + group_misfit + spare-capacity + asym_packing decision tree identical.
- REQ-4: `find_busiest_queue` selects worst-loaded rq in busiest group; respects `migration_disabled` per-task pinning.
- REQ-5: `newidle_balance(rq, rf)` — on rq becoming idle, attempts to pull from sibling/peer CPUs; bounded by `sched_migration_cost_ns` to avoid pull-thrash.
- REQ-6: nohz idle balance: when tick-stopped CPUs become idle, single-CPU kick via `nohz_balancer_kick` heuristics; selected CPU runs `_nohz_idle_balance` on behalf of others. Identical.
- REQ-7: Per-Entity Load Tracking (PELT): `update_load_avg` decay-and-accumulate via `runnable_avg_yN_inv` table; window 1024us; half-life 32ms. Bit-identical decay table.
- REQ-8: PELT signals: `load_avg`, `runnable_avg`, `util_avg`, `util_est` exposed at task / cfs_rq / rq levels. Identical formulas.
- REQ-9: Migration cost gate: a task whose `last_switch_time` is within `sched_migration_cost_ns` of `now` is excluded from `detach_tasks` (cache-warm preservation).
- REQ-10: Asymmetric CPU capacity (big.LITTLE): `SD_ASYM_CPUCAPACITY` domains use `migrate_misfit` task migration to move heavy tasks to big cores. Identical heuristic.
- REQ-11: NUMA balancing: `SD_NUMA` domains use page-scan-driven balance hints from `mm/numa.c` (cross-ref `mm/numa.md`). `migrate_pages` triggered if NUMA node imbalance crosses threshold.
- REQ-12: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: A 4-CPU box, 8 SCHED_OTHER hogs spawned with `taskset -c 0` → within 50ms balancer distributes ~2 hogs per CPU. (covers REQ-1, REQ-2, REQ-3)
- [ ] AC-2: One CPU goes idle while another is overloaded → newidle_balance pulls a task within `< sched_migration_cost_ns × 2`. (covers REQ-5)
- [ ] AC-3: Tickless build (`CONFIG_NO_HZ_FULL=y`) with 3 CPUs idle + 1 overloaded → exactly one CPU is kicked to do balance; others stay idle (verifiable via `trace-cmd` `sched:sched_nohz_balance_*`). (covers REQ-6)
- [ ] AC-4: PELT signal monotonicity test: a task running 50% duty-cycle for many windows → `load_avg` converges to ~50% of rq capacity ± expected decay-table precision. (covers REQ-7, REQ-8)
- [ ] AC-5: A pinned task (`taskset -c 2`) is not migrated by load_balance even when CPU 2 is overloaded — pinning respected. (covers REQ-4)
- [ ] AC-6: A task that just woke (`last_switch_time = now - 100us`) is excluded from detach_tasks; same task at `now - 10ms` (well past `sched_migration_cost_ns=500us`) is migration-eligible. (covers REQ-9)
- [ ] AC-7: A big.LITTLE topology with one util-heavy task on a LITTLE core → balancer migrates it to a big core within one period. (covers REQ-10)
- [ ] AC-8: PELT decay table byte-equivalent to upstream's; verifiable via shared-test-vectors hashed digest. (covers REQ-7)
- [ ] AC-9: Hardening section present and follows template. (covers REQ-12)

### Architecture

### Rust module organization

- `kernel::sched::balance::periodic::PeriodicBalancer` — `run_rebalance_domains`
- `kernel::sched::balance::newidle::NewidleBalancer` — `newidle_balance`
- `kernel::sched::balance::nohz::NohzBalancer` — nohz balance kick + delegated balance
- `kernel::sched::balance::load_balance::LoadBalance` — core `load_balance`
- `kernel::sched::balance::find_busiest::FindBusiest` — group/queue selection
- `kernel::sched::balance::detach_attach::DetachAttach` — task migration helpers
- `kernel::sched::pelt::Pelt` — PELT signals + decay
- `kernel::sched::pelt::DecayTable` — `runnable_avg_yN_inv` (static const)

### Locking and concurrency

- **Per-CPU rq lock**: held during `load_balance` for source rq; double_lock_balance for source+dest
- **`balance_lock` per sched-domain**: serializes balancing for that domain
- **PELT updates**: lockless per-task; per-cfs_rq updates under rq lock

### Error handling

(load-balancer is a fire-and-forget worker; no errors returned to caller — failures = no migrations + may_skip)

### Out of Scope

- sched-domain construction (cross-ref `kernel/sched/topology.md`)
- Per-class scheduling (cross-ref `kernel/sched/cfs.md`, `rt.md`, `deadline.md`)
- 32-bit-only paths
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Periodic balancing (`run_rebalance_domains`, `load_balance`, `find_busiest_group`/`...queue`) | `kernel/sched/fair.c` |
| Newly-idle balancing (`newidle_balance`) | `kernel/sched/fair.c` |
| nohz idle balance kick (`nohz_csd_func`, `_nohz_idle_balance`, `nohz_balancer_kick`) | `kernel/sched/fair.c` |
| Per-Entity Load Tracking (PELT) | `kernel/sched/pelt.c`, `kernel/sched/pelt.h` |
| Idle path | `kernel/sched/idle.c`, `include/linux/sched/idle.h` |

### compatibility contract

### sysctl tunables

- `/proc/sys/kernel/sched_migration_cost_ns` — minimum task-residency threshold to consider for migration (default `500000` ns)
- `/proc/sys/kernel/sched_min_granularity_ns` (cross-ref `kernel/sched/cfs.md`)

Format-identical so userspace performance-tuning scripts work unchanged.

### sched-domain flags consumed

`SD_BALANCE_NEWIDLE`, `SD_BALANCE_FORK`, `SD_BALANCE_EXEC`, `SD_BALANCE_WAKE`, `SD_LOAD_BALANCE`, `SD_PREFER_SIBLING`, `SD_SHARE_CPUCAPACITY`, `SD_SHARE_PKG_RESOURCES`, `SD_NUMA`, `SD_ASYM_CPUCAPACITY`. Identical bit values + behavior so domain-construction in topology.md sees the same balancer.

### nohz idle balancer kick

When tickless CPUs are idle and remote-CPU load imbalance crosses thresholds, exactly one CPU is kicked via `smp_call_function_single_async` to do nohz balancing on behalf of others. Identical heuristics so power management matches upstream.

### `struct sched_avg` (PELT signals) layout

`include/linux/sched.h` (referenced from sched.h):
- `last_update_time: u64`
- `load_sum: u64`
- `runnable_sum: u64`
- `util_sum: u32`
- `period_contrib: u32`
- `load_avg: unsigned long`
- `runnable_avg: unsigned long`
- `util_avg: unsigned long`
- `util_est: unsigned long`

Layout-equivalent so userspace `bpftrace` scripts targeting these offsets work.

### `update_load_avg`, `attach_entity_load_avg`, `detach_entity_load_avg`

PELT decay-and-accumulate algorithm: contributions decay with half-life ~32ms via the `runnable_avg_yN_inv` decay table. Algorithm + table identical to upstream so PELT signal behavior is bit-for-bit equivalent.

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| `detach_tasks` list manipulation under double rq lock | `kani::proofs::sched::balance::detach_safety` |
| `attach_tasks` list manipulation | `kani::proofs::sched::balance::attach_safety` |
| double_lock_balance acquire-order (always-ordered-by-CPU-id) | `kani::proofs::sched::balance::double_lock_safety` |
| PELT decay arithmetic (no overflow on accumulator) | `kani::proofs::sched::pelt::decay_safety` |
| nohz_csd_func cross-CPU send-side | `kani::proofs::sched::balance::nohz_safety` |

### Layer 2: TLA+ models

- `models/sched/load_balance.tla` — proves: under continuous workload, group-imbalance metric eventually decreases below threshold (progress); migrations preserve task identity (safety: no task is duplicated/lost across rq).

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-task `sched_avg` | `load_sum` ≥ `util_sum` (load is super-set of util) at every observable state | `kani::proofs::sched::pelt::sched_avg_invariants` |
| nohz_balancer state | exactly 0 or 1 CPU is the designated balancer at any given tick | `kani::proofs::sched::balance::nohz_singleton` |

### Layer 4: Functional correctness (opt-in)

- **PELT decay-table correctness theorem** via Verus — proves: the precomputed `runnable_avg_yN_inv` table matches `(0.5)^(N/32) * 2^32` to within rounding bound for all 32 entries.
- **load_balance progress theorem** via Verus — proves: under continuously-imbalanced load, repeated invocations strictly decrease group-imbalance metric.

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **SIZE_OVERFLOW** | PELT accumulator arithmetic (load_sum, util_sum) uses checked operators | § Mandatory |
| **CONSTIFY** | `runnable_avg_yN_inv` decay table is `static const [u32; 32]` | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB**: per-domain + per-group structures
- **SIZE_OVERFLOW**: see above
- **MEMORY_SANITIZE**: per-task sched_avg cleared on task exit

### Row-2 / GR-RBAC integration

- LSM hook `security_task_movememory` (already standard in upstream); GR-RBAC policy can deny NUMA migration of pages for sealed processes.
- LSM hook `security_task_setscheduler` already covered in `kernel/sched/cfs.md` / `rt.md` / `deadline.md`.
- Default GR-RBAC policy: empty so balancer behavior matches upstream.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

