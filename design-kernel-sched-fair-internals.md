---
title: "Tier-3: kernel/sched/fair.c — CFS / EEVDF C-file internals"
tags: ["tier-3", "kernel", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`kernel/sched/fair.c` implements the SCHED_NORMAL / SCHED_BATCH / SCHED_IDLE policy class via the **EEVDF** (Earliest Eligible Virtual Deadline First) replacement of classic CFS, while preserving the existing `sched_class fair_sched_class` ABI and `/proc/sched_debug` formats. Per-CPU runnable tasks live on `cfs_rq->tasks_timeline`, an `rb_root_cached` of `sched_entity` keyed on `(vruntime, deadline, min_vruntime)` with a min/max-vruntime augmenting walker (`min_vruntime_cb`). Each `sched_entity` carries its own `vruntime` (weighted virtual time), `deadline` (`vruntime + r_i/w_i`), `vlag` (preserved virtual lag across sleep), `slice` (request size `r_i`), and `avg` (PELT state). Picking the next task = `pick_eevdf()` heap-search for an entity whose `vruntime ≤ avg_vruntime(cfs_rq)` (eligibility) with the earliest `deadline`. Sleeper-fairness is encoded in `place_entity()` re-anchoring `se->vruntime` to `avg_vruntime - vl_i` on wakeup. Hierarchical (cgroup) nesting works by walking `se->parent` via `for_each_sched_entity()` so per-entity load propagation reaches the root rq. `CONFIG_CFS_BANDWIDTH` throttles a `cfs_rq` when its `runtime_remaining` falls below zero and refills via the period hrtimer.

This Tier-3 covers `kernel/sched/fair.c` (~14282 lines) at the **C-function level** — the per-entity data structures, the enqueue/dequeue/pick paths, the EEVDF placement math, throttle/unthrottle, and NUMA placement. It is the C-file companion to the higher-level `kernel/sched/cfs.md`.

### Acceptance Criteria

- [ ] AC-1: `place_entity` on initial fork with `ENQUEUE_INITIAL`: `se->vruntime ≈ avg_vruntime`, `se->deadline = vruntime + vslice/2` (when `PLACE_DEADLINE_INITIAL` set).
- [ ] AC-2: `place_entity` on wakeup with `vlag ≠ 0`: vlag inflated by `(W + w_i) / W` then subtracted from `avg_vruntime`.
- [ ] AC-3: `pick_eevdf` returns an entity whose `vruntime ≤ avg_vruntime(cfs_rq)` (eligible) and has the earliest deadline among eligibles.
- [ ] AC-4: `update_curr` advances `curr->vruntime` by `delta_exec * NICE_0_LOAD / curr->load.weight`.
- [ ] AC-5: `update_deadline`: when `curr->vruntime ≥ curr->deadline`: re-arms `deadline = vruntime + r_i/w_i` and returns true (triggers resched).
- [ ] AC-6: `enqueue_task_fair` walks all ancestors via `for_each_sched_entity` until on_rq.
- [ ] AC-7: `dequeue_entity` on DELAY_DEQUEUE-eligible sleep: leaves entity on rbtree, sets `sched_delayed=1`, returns false.
- [ ] AC-8: `update_entity_lag` clamps `se->vlag` to `±2 * se->slice` (bounded sleeper credit).
- [ ] AC-9: nice 0 = 1024 weight; nice +1 = 820; nice -1 = 1277 (per `prio_to_weight`).
- [ ] AC-10: `throttle_cfs_rq`: cfs_rq added to `cfs_b->throttled_cfs_rq`, all descendants frozen, `throttled=1`.
- [ ] AC-11: `do_sched_cfs_period_timer`: at every period, refill `cfs_b->runtime = quota + burst (≤ max)`; distribute to throttled cfs_rqs.
- [ ] AC-12: cgroup nesting: a task_se's enqueue propagates `h_nr_runnable` increments up to `rq->cfs`.
- [ ] AC-13: `task_numa_fault` increments `p->numa_faults[task_faults_idx(...)]`; `task_numa_placement` updates `numa_preferred_nid`.
- [ ] AC-14: `migrate_task_rq_fair`: `se->avg.last_update_time = 0` post-migration (forces re-sync at destination).
- [ ] AC-15: `reweight_entity` preserves rbtree ordering by `__dequeue_entity` before mutation, `__enqueue_entity` after.

### Architecture

```
struct SchedEntity {
  load:            LoadWeight,           // weight, inv_weight
  run_node:        RbNode,               // augmented (min_vruntime, min_slice, max_slice)
  group_node:      ListHead,
  on_rq:           u8,
  sched_delayed:   u8,
  rel_deadline:    u8,
  custom_slice:    u8,
  exec_start:      u64,
  sum_exec_runtime: u64,
  prev_sum_exec_runtime: u64,
  vruntime:        u64,
  deadline:        u64,
  min_vruntime:    u64,                  // augment: subtree min
  min_slice:       u64,                  // augment
  max_slice:       u64,                  // augment
  vlag:            s64,                  // captured (V - v_i) at last dequeue
  slice:           u64,                  // request size r_i (ns)
  depth:           i32,
  parent:          Option<*SchedEntity>,
  cfs_rq:          *CfsRq,
  my_q:            Option<*CfsRq>,       // Some(_) iff group_se
  avg:             SchedAvg,             // see pelt.md
}

struct CfsRq {
  load:            LoadWeight,
  sum_weight:      u64,
  h_nr_runnable:   u32,
  h_nr_queued:     u32,
  h_nr_idle:       u32,
  nr_queued:       u32,
  tasks_timeline:  RbRootCached,         // augmented; key = (vruntime, deadline)
  curr:            Option<*SchedEntity>,
  next:            Option<*SchedEntity>, // pick buddy
  zero_vruntime:   u64,
  avg_vruntime:    i64,                  // Σ (v_i - zero) * w_i
  avg_load:        i64,                  // Σ w_i
  min_vruntime:    u64,
  avg:             SchedAvg,
  tg:              *TaskGroup,
  on_list:         u8,
  // CONFIG_CFS_BANDWIDTH:
  runtime_enabled: u8,
  runtime_remaining: i64,
  throttled:       u8,
  throttle_count:  u32,
  throttled_clock: u64,
  throttled_clock_pelt: u64,
  throttled_clock_pelt_time: u64,
  pelt_clock_throttled: u8,
  throttled_list:  ListHead,
  throttled_csd_list: ListHead,
  throttled_limbo_list: ListHead,
}
```

`Fair::pick_eevdf(cfs_rq, protect) -> Option<*SchedEntity>`:
1. If `cfs_rq.nr_queued == 1`: return curr-if-on-rq or leftmost.
2. If `sched_feat(PICK_BUDDY) && protect && cfs_rq.next.eligible()`: return next.
3. If curr && (!curr.on_rq || !entity_eligible(curr)): curr = None.
4. If curr && protect && protect_slice(curr): return curr.
5. Heap-walk rbtree:
   a. While node:
      - left = node.rb_left.
      - if left && vruntime_eligible(cfs_rq, __node_2_se(left).min_vruntime): node = left; continue.
      - se = __node_2_se(node); if entity_eligible(cfs_rq, se): best = se; break.
      - node = node.rb_right.
6. Tie-break: if !best || (curr && entity_before(curr, best)): best = curr.
7. Return best.

`Fair::place_entity(cfs_rq, se, flags)`:
1. vruntime = avg_vruntime(cfs_rq).
2. If !se.custom_slice: se.slice = SYSCTL_SCHED_BASE_SLICE.
3. vslice = calc_delta_fair(se.slice, se).
4. If sched_feat(PLACE_LAG) && cfs_rq.nr_queued && se.vlag:
   a. W = cfs_rq.sum_weight + (curr.on_rq ? avg_vruntime_weight(cfs_rq, curr.load.weight) : 0).
   b. w_i = avg_vruntime_weight(cfs_rq, se.load.weight).
   c. lag = se.vlag * (W + w_i) / W.
   d. If w_i > W: update_zero = true.
5. se.vruntime = vruntime - lag.
6. If update_zero: update_zero_vruntime(cfs_rq, -lag).
7. If sched_feat(PLACE_REL_DEADLINE) && se.rel_deadline: se.deadline += se.vruntime; se.rel_deadline = 0; return.
8. If sched_feat(PLACE_DEADLINE_INITIAL) && (flags & ENQUEUE_INITIAL): vslice /= 2.
9. se.deadline = se.vruntime + vslice.

`Fair::update_curr(cfs_rq)`:
1. curr = cfs_rq.curr.
2. delta_exec = update_se(rq_of(cfs_rq), curr).
3. If delta_exec ≤ 0: return.
4. curr.vruntime += calc_delta_fair(delta_exec, curr).
5. resched = update_deadline(cfs_rq, curr).
6. If entity_is_task(curr): dl_server_update(&rq.fair_server, delta_exec).
7. account_cfs_rq_runtime(cfs_rq, delta_exec).
8. If cfs_rq.nr_queued == 1: return.
9. If resched || !protect_slice(curr): resched_curr_lazy(rq); clear_buddies(cfs_rq, curr).

`Fair::enqueue_entity(cfs_rq, se, flags)`:
1. curr = (cfs_rq.curr == se).
2. If curr: place_entity(cfs_rq, se, flags).
3. update_curr(cfs_rq).
4. update_load_avg(cfs_rq, se, UPDATE_TG | DO_ATTACH).
5. se_update_runnable(se); update_cfs_group(se).
6. If !curr: place_entity(cfs_rq, se, flags).
7. account_entity_enqueue(cfs_rq, se).
8. update_stats_enqueue_fair(cfs_rq, se, flags).
9. If !curr: __enqueue_entity(cfs_rq, se).
10. se.on_rq = 1.
11. If cfs_rq.nr_queued == 1: check_enqueue_throttle(cfs_rq); list_add_leaf_cfs_rq(cfs_rq); thaw PELT clock if was throttled.

`Fair::dequeue_entity(cfs_rq, se, flags) -> bool`:
1. update_curr(cfs_rq); clear_buddies(cfs_rq, se).
2. If !(flags & DEQUEUE_DELAYED) && sched_feat(DELAY_DEQUEUE) && (flags & DEQUEUE_SLEEP) && !(flags & (DEQUEUE_SPECIAL|DEQUEUE_THROTTLE)) && !entity_eligible(cfs_rq, se):
   - update_load_avg(cfs_rq, se, 0); update_entity_lag(cfs_rq, se); set_delayed(se); return false.
3. action = UPDATE_TG | (task_on_rq_migrating(task_of(se)) ? DO_DETACH : 0).
4. update_load_avg(cfs_rq, se, action); se_update_runnable(se).
5. update_entity_lag(cfs_rq, se).
6. If sched_feat(PLACE_REL_DEADLINE) && !sleep: se.deadline -= se.vruntime; se.rel_deadline = 1.
7. If se != cfs_rq.curr: __dequeue_entity(cfs_rq, se).
8. se.on_rq = 0.
9. account_entity_dequeue(cfs_rq, se).
10. return_cfs_rq_runtime(cfs_rq); update_cfs_group(se).
11. If flags & DEQUEUE_DELAYED: clear_delayed(se).
12. If cfs_rq.nr_queued == 0: update_idle_cfs_rq_clock_pelt(cfs_rq); freeze PELT clock if throttled.
13. Return true.

`Fair::throttle_cfs_rq(cfs_rq) -> bool`:
1. __assign_cfs_rq_runtime(cfs_b, cfs_rq, 1) — last-chance grab; if granted: return false.
2. list_add_tail_rcu(&cfs_rq.throttled_list, &cfs_b.throttled_cfs_rq).
3. walk_tg_tree_from(cfs_rq.tg, tg_throttle_down, tg_nop, rq).
4. cfs_rq.throttled = 1.
5. Return true.

`Fair::distribute_cfs_runtime(cfs_b) -> bool`:
1. For each cfs_rq on cfs_b.throttled_cfs_rq (RCU):
   - if !remaining: throttled = true; break.
   - runtime = min(-cfs_rq.runtime_remaining + 1, cfs_b.runtime).
   - cfs_b.runtime -= runtime; cfs_rq.runtime_remaining += runtime.
   - if positive: unthrottle (sync local, async via CSD remote).
2. Return throttled.

### Out of Scope

- High-level scheduler-class overview (covered in `cfs.md` Tier-3)
- Load-balance across CPUs / `sched_balance_*` (covered in `load-balance.md` Tier-3)
- Per-Entity Load Tracking math (covered in `pelt.md` Tier-3)
- Sched-domain / NUMA topology builder (covered in `topology.md` Tier-3)
- Real-time scheduler (covered in `rt.md` Tier-3)
- Deadline scheduler (covered in `deadline.md` Tier-3)
- Idle class (covered in `idle.md` Tier-3)
- Core scheduler infrastructure (`__schedule`, `context_switch`) (covered in `core.md` Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct sched_entity` | per-runnable-unit | `SchedEntity` |
| `struct cfs_rq` | per-CPU CFS runqueue | `CfsRq` |
| `struct sched_avg` | per-entity PELT load state | `SchedAvg` (cross-ref `pelt.md`) |
| `struct cfs_bandwidth` | per-tg bandwidth pool | `CfsBandwidth` |
| `struct task_group` | per-cgroup tg | `TaskGroup` |
| `fair_sched_class` | per-class vtable | `Fair::SCHED_CLASS` |
| `enqueue_task_fair()` | per-wakeup enqueue | `Fair::enqueue_task` |
| `dequeue_task_fair()` | per-sleep dequeue | `Fair::dequeue_task` |
| `dequeue_entities()` | per-walk dequeue inner | `Fair::dequeue_entities` |
| `enqueue_entity()` | per-cfs_rq enqueue | `Fair::enqueue_entity` |
| `dequeue_entity()` | per-cfs_rq dequeue | `Fair::dequeue_entity` |
| `__enqueue_entity()` / `__dequeue_entity()` | rbtree insert/erase | `Fair::rbtree_insert` / `_erase` |
| `place_entity()` | per-enqueue vruntime anchor | `Fair::place_entity` |
| `update_curr()` | per-tick exec_runtime + vruntime | `Fair::update_curr` |
| `update_se()` | per-entity delta_exec accumulator | `Fair::update_se` |
| `update_deadline()` | per-quanta `se->deadline` rearming | `Fair::update_deadline` |
| `update_entity_lag()` | per-dequeue capture lag | `Fair::update_entity_lag` |
| `entity_eligible()` / `vruntime_eligible()` | per-EEVDF eligibility predicate | `Fair::entity_eligible` |
| `avg_vruntime()` / `update_zero_vruntime()` | per-cfs_rq weighted average virtual time | `Fair::avg_vruntime` |
| `pick_eevdf()` | per-EEVDF select | `Fair::pick_eevdf` |
| `pick_next_entity()` / `pick_task_fair()` / `pick_next_task_fair()` | per-class entry | `Fair::pick_task` / `pick_next_task` |
| `wakeup_preempt_fair()` | per-wake preempt check | `Fair::wakeup_preempt` |
| `set_next_entity()` / `put_prev_entity()` | per-pick set/restore | `Fair::set_next_entity` / `put_prev_entity` |
| `calc_delta_fair()` / `__calc_delta()` | per-weight virtual-time scaler | `Fair::calc_delta_fair` |
| `reweight_entity()` / `reweight_task_fair()` | per-nice / cgroup re-share | `Fair::reweight_entity` |
| `update_cfs_group()` / `calc_group_shares()` | per-cgroup group_se weight | `Fair::update_cfs_group` |
| `update_load_avg()` / `attach_entity_load_avg()` / `detach_entity_load_avg()` | per-entity PELT integration | cross-ref `pelt.md` |
| `for_each_sched_entity()` (macro) | per-parent walk | `SchedEntity::ancestors_mut` |
| `parent_entity()` / `find_matching_se()` | per-cgroup nesting | `SchedEntity::parent` |
| `throttle_cfs_rq()` / `unthrottle_cfs_rq()` | per-bandwidth gate | `Fair::throttle_cfs_rq` / `unthrottle_cfs_rq` |
| `tg_throttle_down()` / `tg_unthrottle_up()` | per-walk_tg_tree freeze/unfreeze | `Fair::tg_throttle_down` / `tg_unthrottle_up` |
| `__account_cfs_rq_runtime()` / `account_cfs_rq_runtime()` | per-tick runtime drain | `Fair::account_cfs_rq_runtime` |
| `assign_cfs_rq_runtime()` / `__assign_cfs_rq_runtime()` | per-refill from global pool | `Fair::assign_cfs_rq_runtime` |
| `__refill_cfs_bandwidth_runtime()` | per-period refill | `Fair::refill_cfs_bandwidth_runtime` |
| `distribute_cfs_runtime()` | per-period unthrottle waiters | `Fair::distribute_cfs_runtime` |
| `do_sched_cfs_period_timer()` / `sched_cfs_period_timer()` | per-period hrtimer | `Fair::cfs_period_timer` |
| `do_sched_cfs_slack_timer()` / `sched_cfs_slack_timer()` | per-slack-return hrtimer | `Fair::cfs_slack_timer` |
| `task_numa_fault()` / `task_numa_work()` | per-NUMA-hint capture | `Fair::task_numa_fault` / `numa_work` |
| `task_numa_placement()` / `task_numa_migrate()` | per-task NUMA placement | `Fair::numa_placement` / `numa_migrate` |
| `should_numa_migrate_memory()` / `numa_migrate_preferred()` | per-folio migrate-or-not | `Fair::should_numa_migrate_memory` |
| `task_fork_fair()` / `task_dead_fair()` | per-life-cycle | `Fair::task_fork` / `task_dead` |
| `task_tick_fair()` | per-HZ tick | `Fair::task_tick` |
| `init_cfs_rq()` / `init_tg_cfs_entry()` | per-init | `CfsRq::init` |
| `migrate_task_rq_fair()` | per-migrate sync | `Fair::migrate_task_rq` |
| `sync_entity_load_avg()` / `remove_entity_load_avg()` | per-migrate PELT detach | `Fair::sync_entity_load_avg` |
| `select_task_rq_fair()` | per-wake CPU select (cross-ref `cfs.md`) | `Fair::select_task_rq` |

### compatibility contract

REQ-1: struct sched_entity (per-runnable-unit, embedded in `task_struct.se` or `task_group.se[cpu]`):
- `load`: `struct load_weight { unsigned long weight; u32 inv_weight; }` — `weight = prio_to_weight[nice+20] << SCHED_FIXEDPOINT_SHIFT (~ 10 on 64-bit)`.
- `run_node`: `struct rb_node` linked into `cfs_rq->tasks_timeline.rb_root`.
- `group_node`: list head for `cfs_rq->leaf_cfs_rq_list` (group-level only).
- `on_rq`: 1 if currently in the cfs_rq rbtree.
- `exec_start`: rq_clock_task at last `update_curr()`.
- `sum_exec_runtime`: per-entity total exec ns.
- `prev_sum_exec_runtime`: snapshot at last `set_next_entity()` (for `sched_rr_get_interval`-class helpers).
- `vruntime`: weighted virtual time (ns).
- `deadline`: EEVDF deadline = `vruntime + calc_delta_fair(slice, se)`.
- `min_vruntime`: per-rbtree augment = `min(se->vruntime, se->{left,right}->min_vruntime)` (subtree min).
- `min_slice`, `max_slice`: per-rbtree augment for `cfs_rq_min_slice()` / `cfs_rq_max_slice()`.
- `vlag`: `(V - v_i)` captured at last dequeue; restored at next enqueue.
- `slice`: request size `r_i` (ns), defaulted to `sysctl_sched_base_slice` or set via `sched_setattr(SCHED_ATTR_RUNTIME)`.
- `custom_slice`: 1 if `slice` was overridden by user.
- `rel_deadline`: deadline stored relative-to-vruntime during sleep.
- `sched_delayed`: DELAY_DEQUEUE deferred-dequeue flag.
- `depth`: hierarchy depth (root cfs_rq se = 0).
- `parent`: parent group_se (for `for_each_sched_entity()` walk).
- `cfs_rq`: cfs_rq this se belongs to.
- `my_q`: child group's cfs_rq if this se IS a group_se; NULL for task_se.
- `avg`: `struct sched_avg` (PELT state — see `pelt.md`).

REQ-2: struct cfs_rq (per-CPU, per-task_group):
- `load`: `struct load_weight` — sum of `se->load.weight` for runnable entities.
- `sum_weight`: per-EEVDF: sum of `se->load.weight` over enqueued entities.
- `h_nr_runnable`: hierarchical (including child group_se's children) count of SCHED_NORMAL runnable.
- `h_nr_queued`: hierarchical count of enqueued (runnable + delayed).
- `h_nr_idle`: hierarchical count of SCHED_IDLE.
- `nr_queued`: this-cfs_rq enqueued count.
- `tasks_timeline`: `struct rb_root_cached` keyed on `(vruntime, deadline)`.
- `curr`: `struct sched_entity *` currently set_next_entity'd (may be NULL on empty cfs_rq).
- `next`: per-EEVDF "next buddy" hint for cache-friendly pick.
- `zero_vruntime`: per-EEVDF zero-vruntime anchor for `avg_vruntime` overflow avoidance.
- `avg_vruntime`, `avg_load`: running weighted sums for `avg_vruntime()`.
- `min_vruntime`: monotonic-non-decreasing per-cfs_rq lower bound (used historically; EEVDF prefers `avg_vruntime`).
- `avg`: cfs_rq aggregate PELT state.
- `tg`: pointer to enclosing task_group.
- `on_list`: 1 if linked into `rq->leaf_cfs_rq_list`.
- `runtime_enabled`, `runtime_remaining`, `runtime`: per-CONFIG_CFS_BANDWIDTH local quota cache.
- `throttled`, `throttle_count`, `throttled_clock`, `throttled_list`, `throttled_csd_list`, `throttled_limbo_list`, `throttled_clock_pelt`, `throttled_clock_pelt_time`, `throttled_clock_self`, `pelt_clock_throttled`: per-throttle bookkeeping.
- `nr_spread_over`: per-debug spread metric.

REQ-3: place_entity(cfs_rq, se, flags):
- Compute `vruntime = avg_vruntime(cfs_rq)` (weighted-average virtual time of runnable set).
- If `!se->custom_slice`: `se->slice = sysctl_sched_base_slice`.
- `vslice = calc_delta_fair(se->slice, se)` = `se->slice * NICE_0_LOAD / se->load.weight`.
- If `sched_feat(PLACE_LAG) ∧ cfs_rq->nr_queued ∧ se->vlag`:
  - Inflate vlag for the imminent re-weighting of `V` after enqueue: `vl_i = (W + w_i) * vl'_i / W`, where `W = cfs_rq->sum_weight` (+ curr if on_rq), `w_i = avg_vruntime_weight(cfs_rq, se->load.weight)`.
  - `lag = vl_i (clamped to safe range)`.
  - If `weight > load`: defer `update_zero_vruntime` to keep `zero_vruntime` near the heavy entity.
- `se->vruntime = vruntime - lag`.
- If `sched_feat(PLACE_REL_DEADLINE) ∧ se->rel_deadline`: restore `se->deadline += se->vruntime; se->rel_deadline = 0; return`.
- If `sched_feat(PLACE_DEADLINE_INITIAL) ∧ (flags & ENQUEUE_INITIAL)`: `vslice /= 2`.
- `se->deadline = se->vruntime + vslice`.

REQ-4: entity_eligible(cfs_rq, se):
- An entity is eligible at `V = avg_vruntime(cfs_rq)` iff `se->vruntime ≤ V`.
- Computed by `vruntime_eligible(cfs_rq, se->vruntime)` which uses the `(avg_vruntime, avg_load)` aggregates to avoid recomputing the weighted average.

REQ-5: pick_eevdf(cfs_rq, protect) → struct sched_entity *:
- If `cfs_rq->nr_queued == 1`: short-circuit return `curr or leftmost`.
- If `sched_feat(PICK_BUDDY) ∧ protect ∧ cfs_rq->next ∧ entity_eligible(cfs_rq, cfs_rq->next)`: return `cfs_rq->next`.
- If `curr ∧ (!curr->on_rq ∨ !entity_eligible(curr))`: curr = NULL.
- If `curr ∧ protect ∧ protect_slice(curr)`: return `curr` (don't preempt mid-slice).
- Walk rbtree starting at root:
  - Prefer left subtree if its `min_vruntime` is eligible — that subtree contains entities with earlier deadlines among eligible.
  - Else evaluate node: if eligible, candidate.
  - Else descend right.
- `best = best ?: leftmost` (eligibility-checked).
- Tie-break by `entity_before(curr, best)` if `curr` is still eligible.
- Return best.

REQ-6: update_curr(cfs_rq):
- `curr = cfs_rq->curr` (may be NULL if no entity currently set_next'd).
- `delta_exec = update_se(rq_of(cfs_rq), curr)` (per-REQ-7).
- If `delta_exec ≤ 0`: return.
- `curr->vruntime += calc_delta_fair(delta_exec, curr)` — advance weighted virtual time.
- `resched = update_deadline(cfs_rq, curr)`:
  - If `vruntime_cmp(se->vruntime, "<", se->deadline)`: return false (still inside slice).
  - Refresh `slice = sysctl_sched_base_slice` (unless custom).
  - `se->deadline = se->vruntime + calc_delta_fair(se->slice, se)` — rearm.
  - Call `avg_vruntime(cfs_rq)` to refresh aggregates.
  - Return true.
- If `entity_is_task(curr)`: `dl_server_update(&rq->fair_server, delta_exec)`.
- `account_cfs_rq_runtime(cfs_rq, delta_exec)` (per-REQ-13).
- If `nr_queued == 1`: return (no preempt of a singleton).
- If `resched ∨ !protect_slice(curr)`: `resched_curr_lazy(rq); clear_buddies(cfs_rq, curr)`.

REQ-7: update_se(rq, se) → delta_exec:
- `now = rq_clock_task(rq); delta_exec = now - se->exec_start`.
- If `delta_exec ≤ 0`: return.
- `se->exec_start = now`.
- If `entity_is_task(se)`:
  - Account against `rq->curr` (proxy-exec aware): `rq->curr->se.exec_start = now; rq->curr->se.sum_exec_runtime += delta_exec`.
  - `cgroup_account_cputime(donor=task_of(se), delta_exec)` — cgroup attribution to the donor.
- Else (group_se): `se->sum_exec_runtime += delta_exec`.
- `schedstat_set(stats->exec_max, max(delta_exec, stats->exec_max))`.

REQ-8: enqueue_task_fair(rq, p, flags):
- If `task_is_throttled(p) ∧ enqueue_throttled_task(p)`: park on throttled limbo and return.
- If `!p->se.sched_delayed ∨ (flags & ENQUEUE_DELAYED)`: `util_est_enqueue(&rq->cfs, p)`.
- If `flags & ENQUEUE_DELAYED`: `requeue_delayed_entity(se); return`.
- If `p->in_iowait`: `cpufreq_update_util(rq, SCHED_CPUFREQ_IOWAIT)`.
- /* Walk upward through groups */
- for_each_sched_entity(se):
  - If `se->on_rq`: requeue_delayed_entity if delayed; else break (ancestor already on rq).
  - `cfs_rq = cfs_rq_of(se)`.
  - If we propagated a `slice` from a child: `se->slice = slice; se->custom_slice = 1`.
  - `enqueue_entity(cfs_rq, se, flags)` — per-REQ-9.
  - `slice = cfs_rq_min_slice(cfs_rq)`.
  - Bump `h_nr_runnable, h_nr_queued, h_nr_idle`.
  - `flags = ENQUEUE_WAKEUP` (subsequent ancestors look like wakeups, not initial).
- Second walk: propagate load_avg up; rebalance group shares; bump h_* on un-walked ancestors.
- If first-fair-task-on-rq: `dl_server_start(&rq->fair_server)`.
- `add_nr_running(rq, 1); check_update_overutilized_status(rq); hrtick_update(rq)`.

REQ-9: enqueue_entity(cfs_rq, se, flags):
- `curr = (cfs_rq->curr == se)`.
- If curr: `place_entity(cfs_rq, se, flags)` (renormalise before update_curr).
- `update_curr(cfs_rq)`.
- `update_load_avg(cfs_rq, se, UPDATE_TG | DO_ATTACH)` (cross-ref `pelt.md`).
- `se_update_runnable(se)`.
- `update_cfs_group(se)` (for group_se: recompute weight from `calc_group_shares`).
- If !curr: `place_entity(cfs_rq, se, flags)` (per-REQ-3).
- `account_entity_enqueue(cfs_rq, se)`: `cfs_rq->load.weight += se->load.weight; cfs_rq->sum_weight += ...; ++cfs_rq->nr_queued`.
- If `flags & ENQUEUE_MIGRATED`: `se->exec_start = 0` (no-longer-hot).
- `update_stats_enqueue_fair(...)`.
- If !curr: `__enqueue_entity(cfs_rq, se)` — `rb_add_augmented_cached(&se->run_node, &cfs_rq->tasks_timeline, __entity_less, &min_vruntime_cb)`.
- `se->on_rq = 1`.
- If first entity (`nr_queued == 1`): `check_enqueue_throttle(cfs_rq); list_add_leaf_cfs_rq(cfs_rq)`; un-pause PELT clock if was throttled.

REQ-10: dequeue_task_fair(rq, p, flags):
- Returns `bool` — false ⟹ delayed-dequeue (the entity stays on rbtree until eligible).
- Delegates to `dequeue_entities(rq, &p->se, flags)`:
  - Returns -1 (delayed), 0 (throttled mid-walk), 1 (complete).
- If complete: `util_est_dequeue(&rq->cfs, p)`; finalize.

REQ-11: dequeue_entities(rq, se, flags) → int:
- For each `se` upward:
  - `dequeue_entity(cfs_rq, se, flags)` — per-REQ-12.
  - If returns false (DELAY_DEQUEUE): stop, return -1 (delayed).
  - Decrement `h_nr_runnable, h_nr_queued, h_nr_idle`.
  - If `cfs_rq->load.weight ≠ 0`: parent still has siblings; stop walk; set_next_buddy(se->parent) if task_sleep.
  - Else: `flags |= DEQUEUE_SLEEP; flags &= ~(DEQUEUE_DELAYED | DEQUEUE_SPECIAL)`; continue.
- Second walk: propagate load_avg up.
- `sub_nr_running(rq, h_nr_queued)`.
- If `task_delayed`: `__block_task(rq, p)` (final transition to TASK_INTERRUPTIBLE/UNINTERRUPTIBLE).

REQ-12: dequeue_entity(cfs_rq, se, flags) → bool:
- `update_curr(cfs_rq); clear_buddies(cfs_rq, se)`.
- If `!(flags & DEQUEUE_DELAYED)` ∧ `sched_feat(DELAY_DEQUEUE)` ∧ sleep-not-special ∧ `!entity_eligible(cfs_rq, se)`:
  - Set `se->sched_delayed = 1` (and decrement `h_nr_runnable` for tasks); leave on rbtree; return false.
- `update_load_avg(cfs_rq, se, UPDATE_TG | maybe DO_DETACH)`.
- `update_entity_lag(cfs_rq, se)`:
  - `se->vlag = clamp(V - se->vruntime, -limit, limit)` to preserve sleeper-fairness debt.
- If `sched_feat(PLACE_REL_DEADLINE) ∧ !sleep`: `se->deadline -= se->vruntime; se->rel_deadline = 1`.
- If `se ≠ cfs_rq->curr`: `__dequeue_entity(cfs_rq, se)` — `rb_erase_augmented_cached(...)`.
- `se->on_rq = 0; account_entity_dequeue(cfs_rq, se); return_cfs_rq_runtime(cfs_rq); update_cfs_group(se)`.
- If `flags & DEQUEUE_DELAYED`: `clear_delayed(se)`.
- If `nr_queued == 0`: `update_idle_cfs_rq_clock_pelt(cfs_rq)`; if throttled: list_del_leaf_cfs_rq and stop PELT clock.
- Return true.

REQ-13: account_cfs_rq_runtime(cfs_rq, delta_exec) (CONFIG_CFS_BANDWIDTH):
- If `cfs_b->quota == RUNTIME_INF`: noop.
- `cfs_rq->runtime_remaining -= delta_exec`.
- If `runtime_remaining > 0`: return.
- Else: `assign_cfs_rq_runtime(cfs_rq)` — borrow from `cfs_b->runtime`; if still negative ⟹ `resched_curr(rq)` so `pick_next_task` enters `check_cfs_rq_runtime` ⟹ `throttle_cfs_rq`.

REQ-14: assign_cfs_rq_runtime(cfs_rq) → bool:
- `raw_spin_lock(&cfs_b->lock)`.
- `__assign_cfs_rq_runtime(cfs_b, cfs_rq, sched_cfs_bandwidth_slice())`:
  - `min_amount = target_runtime - cfs_rq->runtime_remaining`.
  - If `cfs_b->quota == RUNTIME_INF`: take min_amount.
  - Else: take `min(min_amount, cfs_b->runtime)`; if pool empty: start_cfs_bandwidth(cfs_b) to wake the period timer.
  - `cfs_b->runtime -= amount; cfs_rq->runtime_remaining += amount`.
- Return `cfs_rq->runtime_remaining > 0`.

REQ-15: throttle_cfs_rq(cfs_rq) → bool:
- `__assign_cfs_rq_runtime(cfs_b, cfs_rq, 1)` — last-chance 1ns grab.
- If got runtime: return false (don't throttle).
- Else: `list_add_tail_rcu(&cfs_rq->throttled_list, &cfs_b->throttled_cfs_rq)`.
- `walk_tg_tree_from(cfs_rq->tg, tg_throttle_down, tg_nop, rq)`:
  - For each descendant: increment `throttle_count`; if `!nr_queued`: stop PELT clock and `list_del_leaf_cfs_rq`.
- `cfs_rq->throttled = 1; cfs_rq->throttled_clock = rq_clock(rq)` (set later for first-throttle accounting).
- Return true.

REQ-16: unthrottle_cfs_rq(cfs_rq):
- Early-out if `runtime_enabled ∧ runtime_remaining ≤ 0` (can't safely unthrottle).
- `cfs_rq->throttled = 0; update_rq_clock(rq)`.
- Acc throttled_time into `cfs_b->throttled_time`; `list_del_rcu(&cfs_rq->throttled_list)`.
- `walk_tg_tree_from(cfs_rq->tg, tg_nop, tg_unthrottle_up, rq)`:
  - Decrement `throttle_count`; if reaches 0 ∧ `!nr_queued`: re-arm PELT clock.
- Re-link cfs_rq into leaf list if not already.
- If `rq->curr == rq->idle ∧ rq->cfs.nr_queued`: `resched_curr(rq)`.

REQ-17: do_sched_cfs_period_timer(cfs_b, overrun, flags) (hrtimer cb at period boundary):
- If `cfs_b->quota == RUNTIME_INF`: out_deactivate.
- `__refill_cfs_bandwidth_runtime(cfs_b)`: `cfs_b->runtime = cfs_b->quota + cfs_b->burst (saturating, ≤ max_cfs_runtime)`.
- If no throttled cfs_rqs: mark `cfs_b->idle = 1`; possibly deactivate timer.
- Else: while throttled ∧ runtime > 0: drop lock, `distribute_cfs_runtime(cfs_b)`, reacquire.

REQ-18: distribute_cfs_runtime(cfs_b) → throttled-remaining:
- For each `cfs_rq` on `cfs_b->throttled_cfs_rq` (RCU):
  - `runtime = -cfs_rq->runtime_remaining + 1; clamp to cfs_b->runtime`.
  - `cfs_b->runtime -= runtime; cfs_rq->runtime_remaining += runtime`.
  - If now positive: `unthrottle_cfs_rq_async(cfs_rq)` (CSD to target CPU) for cross-CPU; local list for same-CPU.
- Return whether any remain throttled.

REQ-19: Hierarchical (cgroup) nesting via `for_each_sched_entity()`:
- Each cgroup with `cpu` controller has a `task_group`; per-CPU it has `tg->cfs_rq[cpu]` and `tg->se[cpu]`.
- A task_se belongs to a leaf cfs_rq; that cfs_rq's group_se belongs to a parent cfs_rq; ... up to `rq->cfs`.
- `parent_entity(se) = se->parent` (NULL if at root).
- enqueue_task_fair walks upward enqueueing each ancestor that is not already on_rq; on first sleep dequeue walks downward (well — upward in tree, but unwinds bottom-up) dequeueing siblings-empty ancestors.
- `find_matching_se(&se, &pse)` — used by `wakeup_preempt_fair` to compare two task_se's at common-ancestor depth.

REQ-20: NUMA placement (CONFIG_NUMA_BALANCING):
- `task_numa_work()` (mmuwork): periodically scan a slice of the task's VMAs, marking PROT_NONE so subsequent accesses fault into `do_numa_page` → `task_numa_fault()`.
- `task_numa_fault(last_cpupid, mem_node, pages, flags)` — accumulates per-(nid, priv/shared) fault counters in `p->numa_faults`.
- `task_numa_placement(p)` — periodically: compute `p->numa_preferred_nid = preferred_group_nid(p, max_nid)`.
- `numa_migrate_preferred(p)` — periodically attempt `task_numa_migrate(p)`.
- `task_numa_migrate(p)` → `task_numa_env env; sd = sd_numa; env.imbalance_pct = 100 + (sd->imbalance_pct - 100)/2; task_numa_find_cpu(); migrate_task_to or migrate_swap`.
- `should_numa_migrate_memory(p, folio, src_nid, dst_cpu)` — gate for `migrate_misplaced_folio` to prevent ping-pong.

REQ-21: avg_vruntime() / update_zero_vruntime() (EEVDF arithmetic core):
- `avg_vruntime(cfs_rq) = cfs_rq->zero_vruntime + cfs_rq->avg_vruntime / cfs_rq->avg_load`, where:
  - `avg_vruntime = Σ (v_i - zero_vruntime) * w_i` (over enqueued + curr).
  - `avg_load = Σ w_i`.
- `update_zero_vruntime(cfs_rq, delta)`: shift `zero_vruntime += delta; avg_vruntime -= delta * avg_load` (re-anchor to avoid overflow when entities accumulate).
- All enqueue/dequeue paths update `avg_vruntime, avg_load` incrementally.

REQ-22: Sleeper-fairness via vlag preservation:
- On dequeue: `update_entity_lag` writes `se->vlag = V - se->vruntime`, clamped to a window proportional to `2 * se->slice`.
- On wake re-enqueue: `place_entity` reads `se->vlag`, inflates per the (W + w_i) / W formula to compensate for the imminent V re-weighting, and computes `se->vruntime = avg_vruntime - lag`. A long sleeper retains its "owed" credit but cannot accumulate unbounded.

REQ-23: Reweighting (nice / cgroup shares):
- `reweight_task_fair(rq, p, lw)` → `reweight_entity(cfs_rq_of(se), se, lw->weight)`:
  - If `se->on_rq ∧ se != cfs_rq->curr`: `__dequeue_entity` so rbtree key changes safely.
  - `cfs_rq->load.weight += new - old; cfs_rq->sum_weight += new - old`.
  - `se->load.weight = new`; refresh `inv_weight`.
  - Re-`__enqueue_entity` if was removed.
- `update_cfs_group(se)` for group_se: `weight = calc_group_shares(cfs_rq_of_my_q(se))` — load-proportional share within `tg->shares`.

REQ-24: Buddies — `set_next_buddy(se)`:
- Hint to `pick_eevdf`/`pick_next_entity` to prefer `cfs_rq->next` if eligible.
- Set on `task_sleep` so the soon-to-wake sibling tends to follow.
- Cleared by `clear_buddies` on dequeue or when chosen.

REQ-25: task_tick_fair(rq, curr, queued):
- For each ancestor `se` of `curr->se`: `cfs_rq = cfs_rq_of(se); entity_tick(cfs_rq, se, queued)`.
- `entity_tick`: `update_curr(cfs_rq); update_load_avg(cfs_rq, curr, UPDATE_TG); update_cfs_group(curr); hrtick_update(rq); task_tick_numa(rq, curr)`.

REQ-26: task_fork_fair(p):
- `update_rq_clock(rq); cfs_rq = task_cfs_rq(current); update_curr(cfs_rq)`.
- New `se->vruntime = avg_vruntime(cfs_rq)`; deadline reset via `place_entity` on first enqueue with `ENQUEUE_INITIAL`.
- If `sysctl_sched_child_runs_first` ∧ curr's deadline ≤ se's: swap so child runs first.

REQ-27: migrate_task_rq_fair(p, new_cpu):
- `sync_entity_load_avg(&p->se)` against old cfs_rq's PELT timestamp.
- `remove_entity_load_avg(&p->se)` from old cfs_rq totals.
- `migrate_se_pelt_lag(&p->se)` to age load_sum by missed PELT periods.
- Mark `p->se.exec_start = 0` so destination doesn't see it as "hot".
- `p->se.vruntime` is preserved (relative); next `place_entity` on enqueue at the destination re-anchors via `vlag`.

REQ-28: Per-DELAY_DEQUEUE state machine:
- Tasks sleeping with `vruntime < V` (have not yet earned a slice of receipt) are marked `se->sched_delayed=1` and **left on the rbtree** until they become eligible — they continue to age and contribute to fairness accounting, just don't get picked.
- Spurious wakeups during the delayed window → `requeue_delayed_entity`: refresh lag, re-place.
- Real blocking → `__block_task` finalizes the sleep when dequeue finally completes.

REQ-29: Per-PELT-clock-throttle:
- When all entities in a throttled cfs_rq dequeue: `pelt_clock_throttled = 1; throttled_clock_pelt = rq_clock_pelt(rq)`.
- During unthrottle / re-enqueue: account `throttled_clock_pelt_time += rq_clock_pelt(rq) - throttled_clock_pelt` so PELT does not double-decay during throttle.

REQ-30: Per-sched_feat bitset:
- Feature flags PLACE_LAG, PLACE_DEADLINE_INITIAL, PLACE_REL_DEADLINE, RUN_TO_PARITY, PREEMPT_SHORT, NEXT_BUDDY, PICK_BUDDY, DELAY_DEQUEUE, DELAY_ZERO, NO_RT_PUSH_IPI_AT_HIGH_PRI, etc. — gate behavior via `sched_feat(NAME)`. Each is a column in `/sys/kernel/debug/sched/features`. Default set is preserved across Rookery for ABI parity.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `rbtree_key_invariant` | INVARIANT | per-rbtree: in-order traversal yields non-decreasing `(vruntime, deadline)`. |
| `min_vruntime_augment_holds` | INVARIANT | per-rbtree: `se.min_vruntime = min(se.vruntime, left.min_vruntime, right.min_vruntime)`. |
| `pick_eevdf_returns_eligible` | POSTCOND | per-pick_eevdf: result satisfies `entity_eligible(cfs_rq, result)`. |
| `place_entity_vlag_bounded` | INVARIANT | per-place_entity: post-condition `|se.vlag| ≤ 2 * se.slice` (after clamp). |
| `enqueue_dequeue_h_nr_balance` | INVARIANT | per-enqueue/dequeue: `h_nr_runnable` increments/decrements balance across `for_each_sched_entity` walk. |
| `delay_dequeue_keeps_on_rq` | INVARIANT | per-dequeue_entity: `sched_delayed=1` ⟹ `se.on_rq == 1` AND still in rbtree. |
| `throttle_throttle_count_monotone` | INVARIANT | per-throttle: `tg_throttle_down` increments `throttle_count`; never overflow. |
| `cfs_bandwidth_runtime_nonneg` | INVARIANT | per-distribute_cfs_runtime: `cfs_b.runtime ≥ 0` at every assignment point. |
| `vruntime_monotone_per_quantum` | INVARIANT | per-update_curr: `curr.vruntime` is strictly non-decreasing within a quantum (delta_exec > 0). |
| `avg_vruntime_no_overflow` | INVARIANT | per-update_zero_vruntime: `|avg_vruntime| ≤ U64_MAX / 2` (re-anchoring prevents overflow). |

### Layer 2: TLA+

`kernel/sched/fair-internals.tla`:
- Per-CPU `cfs_rq` rbtree + per-task `se` model.
- Operations: ENQUEUE, DEQUEUE, DELAY_DEQUEUE_DEFER, PICK_EEVDF, UPDATE_CURR, PLACE_ENTITY, THROTTLE, UNTHROTTLE, REWEIGHT.
- Properties:
  - `safety_eligible_pick` — `pick_eevdf` result is eligible (`v_i ≤ V`).
  - `safety_no_stuck_throttle` — every `throttle_cfs_rq` is eventually followed by `unthrottle_cfs_rq` while `cfs_b.runtime` is refilled.
  - `safety_delayed_eventually_eligible` — `sched_delayed` entity eventually crosses `v_i ≤ V` and is pickable.
  - `safety_vlag_bounded` — `|vlag| ≤ 2 * slice` always.
  - `liveness_eventually_picked` — any eligible runnable entity is eventually picked (under fair pick policy).
  - `liveness_bandwidth_period_progress` — `cfs_b.runtime` is refilled at every period boundary modulo deactivation.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `place_entity` post: `se.vruntime = avg_vruntime - lag (after clamp)` | `Fair::place_entity` |
| `pick_eevdf` post: `result.deadline ≤ any other eligible entity's deadline` | `Fair::pick_eevdf` |
| `update_curr` post: `curr.vruntime' = curr.vruntime + delta_exec * 1024 / curr.load.weight` (ignoring `inv_weight` rounding) | `Fair::update_curr` |
| `update_deadline` post: ret ⟺ `vruntime ≥ deadline`; on true `deadline' = vruntime + r_i/w_i` | `Fair::update_deadline` |
| `enqueue_entity` post: `se.on_rq == 1 ∧ cfs_rq.nr_queued == nr_queued + 1` | `Fair::enqueue_entity` |
| `dequeue_entity` (non-delayed) post: `se.on_rq == 0 ∧ cfs_rq.nr_queued == nr_queued - 1` | `Fair::dequeue_entity` |
| `throttle_cfs_rq` post: ret ⟹ `cfs_rq.throttled == 1 ∧ on cfs_b.throttled_cfs_rq list` | `Fair::throttle_cfs_rq` |
| `unthrottle_cfs_rq` post: `cfs_rq.throttled == 0 ∧ NOT on throttled_cfs_rq list` | `Fair::unthrottle_cfs_rq` |
| `for_each_sched_entity` walk terminates at root (parent == NULL) | `SchedEntity::ancestors_mut` |

### Layer 4: Verus/Creusot functional

Per-EEVDF semantic equivalence:
- Definition: `lag_i = w_i * (V - v_i)`; entity is eligible iff `lag_i ≥ 0` iff `v_i ≤ V`.
- Definition: `vd_i = ve_i + r_i / w_i` (virtual deadline = virtual eligible time + virtual request).
- `pick_eevdf` chooses argmin over eligibles of `vd_i`.

Per-`Documentation/scheduler/sched-eevdf.rst` + `sched-design-CFS.rst` + paper "Earliest Eligible Virtual Deadline First" (Stoica & Abdel-Wahab, 1996).

CFS-bandwidth semantic equivalence: per-`Documentation/scheduler/sched-bwc.rst` (period/quota/burst).

NUMA semantic equivalence: per-`Documentation/admin-guide/mm/numa_memory_policy.rst` + AutoNUMA fault-based placement.

### hardening

(Inherits row-1 features from `kernel/sched/00-overview.md` § Hardening.)

CFS / EEVDF C-internals reinforcement:

- **Per-rbtree augmented walker invariants checked** — defense against per-broken-augment leading to non-eligible pick.
- **Per-`vlag` clamp `±2 * slice`** — defense against per-unbounded sleeper-credit accumulation.
- **Per-`avg_vruntime` periodic re-anchor via `update_zero_vruntime`** — defense against per-i64 overflow of weighted sum.
- **Per-`for_each_sched_entity` walk bounded by cgroup nesting depth** — defense against per-runaway recursion in pathological cgroup trees.
- **Per-`cfs_b->lock` separate from per-cfs_rq rq->lock; period-timer drops both during distribution** — defense against per-AB-BA deadlock between bandwidth pool and per-CPU rq.
- **Per-async unthrottle via CSD with `WARN_ON_ONCE(!list_empty(&throttled_csd_list))`** — defense against per-double-enqueue.
- **Per-`cfs_b->runtime ≤ max_cfs_runtime` saturating refill** — defense against per-quota+burst overflow.
- **Per-`runtime_enabled ∧ runtime_remaining ≤ 0` guard at unthrottle** — defense against per-immediate-re-throttle.
- **Per-`update_se` `delta_exec ≤ 0` early-return** — defense against per-clock-non-monotonic-during-init.
- **Per-`migrate_task_rq_fair` `sync_entity_load_avg` + `remove_entity_load_avg`** — defense against per-cross-CPU PELT double-count.
- **Per-DELAY_DEQUEUE entity left on rbtree** — defense against per-loss-of-fairness during contended sleep.
- **Per-`set_protect_slice`/`protect_slice`** — defense against per-pathological wake-preempt-thrash.
- **Per-`assert_list_leaf_cfs_rq`** — defense against per-broken cfs_rq leaf list during cgroup churn.

