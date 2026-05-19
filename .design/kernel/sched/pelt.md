# Tier-3: kernel/sched/pelt.c — Per-Entity Load Tracking (PELT)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/sched/00-overview.md
upstream-paths:
  - kernel/sched/pelt.c (~490 lines)
  - kernel/sched/pelt.h
  - kernel/sched/sched-pelt.h (generated y^n lookup table)
  - include/linux/sched.h (struct sched_avg)
  - kernel/sched/fair.c (call sites: update_load_avg, attach_entity_load_avg, ...)
-->

## Summary

**PELT** is the kernel's per-entity load-and-utilisation estimator. It models each `sched_entity` (task or cgroup group_se) and each runqueue (`cfs_rq`, plus `rq->avg_rt`, `rq->avg_dl`, `rq->avg_irq`, `rq->avg_hw`) as a **geometric series of 1024us (~1ms) periods**, with decay factor `y` chosen such that `y^32 = 0.5` — i.e. a contribution from ~32ms ago counts half as much as a contribution from the last ms. Per-period samples are: `load` (entity weight while runnable), `runnable` (entity's runnable_weight while runnable on a cfs_rq), and `running` (boolean "was this entity the chosen `cfs_rq->curr`"). The accumulated `*_sum` is divided by a position-dependent divider `LOAD_AVG_MAX - 1024 + period_contrib` to yield the bounded `*_avg`. Frequency- and CPU-invariance scale `running` contributions by `arch_scale_freq_capacity(cpu)` and `arch_scale_cpu_capacity(cpu)` so that a task's `util_avg` is comparable across heterogeneous CPUs (asymmetric capacity / big.LITTLE) and across DVFS levels.

PELT's outputs drive: (1) CFS load-balancing decisions (`task_h_load`, `cpu_load`, `migration_cost`), (2) `schedutil` CPU-frequency selection (`util_avg → required OPP`), (3) EAS / energy-aware placement (`find_energy_efficient_cpu`), (4) cgroup share recomputation (`calc_group_shares` ∝ `grq->load_avg / tg->load_avg`), and (5) overutilized-domain detection.

This Tier-3 covers `kernel/sched/pelt.c` (~490 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct sched_avg` | per-entity / per-cfs_rq PELT state | `SchedAvg` |
| `runnable_avg_yN_inv[]` (in `sched-pelt.h`) | per-y^n lookup table (n ∈ [0..31]) | `Pelt::Y_N_INV` |
| `LOAD_AVG_PERIOD` (32) | per-half-life | `Pelt::LOAD_AVG_PERIOD` |
| `LOAD_AVG_MAX` (47742) | per-sum upper bound `1024 / (1 - y)` | `Pelt::LOAD_AVG_MAX` |
| `PELT_MIN_DIVIDER` (LOAD_AVG_MAX - 1024) | per-divider floor | `Pelt::PELT_MIN_DIVIDER` |
| `get_pelt_divider(sa)` | per-period-dependent divider | `Pelt::pelt_divider` |
| `decay_load(val, n)` | per-y^n multiplication | `Pelt::decay_load` |
| `__accumulate_pelt_segments(p, d1, d3)` | per-three-segment contribution | `Pelt::accumulate_segments` |
| `accumulate_sum(delta, sa, load, runnable, running)` | per-update step 1/2 | `Pelt::accumulate_sum` |
| `___update_load_sum(now, sa, load, runnable, running)` | per-time-delta + accumulate | `Pelt::update_load_sum` |
| `___update_load_avg(sa, load)` | per-`*_sum → *_avg` divide | `Pelt::update_load_avg` |
| `__update_load_avg_blocked_se(now, se)` | per-sleeping-entity decay | `Pelt::update_load_avg_blocked_se` |
| `__update_load_avg_se(now, cfs_rq, se)` | per-task / group_se update | `Pelt::update_load_avg_se` |
| `__update_load_avg_cfs_rq(now, cfs_rq)` | per-cfs_rq aggregate update | `Pelt::update_load_avg_cfs_rq` |
| `update_rt_rq_load_avg(now, rq, running)` | per-rq RT util track | `Pelt::update_rt_rq_load_avg` |
| `update_dl_rq_load_avg(now, rq, running)` | per-rq DL util track | `Pelt::update_dl_rq_load_avg` |
| `update_hw_load_avg(now, rq, capacity)` | per-rq HW pressure track | `Pelt::update_hw_load_avg` |
| `update_irq_load_avg(rq, running)` | per-rq IRQ util track | `Pelt::update_irq_load_avg` |
| `update_other_load_avgs(rq)` | per-RT/DL/HW/IRQ batch | `Pelt::update_other_load_avgs` |
| `cfs_se_util_change(sa)` | per-est-util sentinel reset | `Pelt::se_util_change` |
| `arch_scale_freq_capacity(cpu)` | per-arch freq invariance | `Arch::scale_freq_capacity` |
| `arch_scale_cpu_capacity(cpu)` | per-arch CPU capacity invariance | `Arch::scale_cpu_capacity` |
| `rq_clock_pelt(rq)` | per-PELT monotonic clock (excludes throttled / IRQ-stolen) | `Rq::clock_pelt` |

## Compatibility contract

REQ-1: struct sched_avg (per entity, per cfs_rq, per rq_avg_*):
- `last_update_time`: ns timestamp of last `___update_load_sum`; quantised to 1024ns granularity.
- `load_sum`: `Σ load * y^p_i` over period samples; bounded by `LOAD_AVG_MAX * 1024 = ~48.9M * scale_load_down`.
- `runnable_sum`: `Σ runnable_weight * (running_period) * y^p_i << SCHED_CAPACITY_SHIFT`.
- `util_sum`: `Σ (running ∈ {0, freq*cpu_capacity_scale}) * y^p_i << SCHED_CAPACITY_SHIFT`; bounded by `LOAD_AVG_MAX << SCHED_CAPACITY_SHIFT`.
- `period_contrib`: `0..1023` — microseconds elapsed in the current (partial) period.
- `load_avg`: `load_sum / pelt_divider * (se_weight / load)` (bounded by se's load).
- `runnable_avg`: `runnable_sum / pelt_divider`.
- `util_avg`: `util_sum / pelt_divider` (bounded by `SCHED_CAPACITY_SCALE = 1024`).
- `util_est`: `EWMA` of recent `util_avg` peaks (per-`util_est_*` helpers in `fair.c`).

REQ-2: Y constants:
- `LOAD_AVG_PERIOD = 32`.
- `y = 2^(-1/32)`, so `y^32 = 0.5`, `y^1024 ≈ 0`.
- `LOAD_AVG_MAX = 47742` ≈ `1024 * Σ_{n=0..∞} y^n = 1024 / (1 - y) ≈ 1024 / (1 - 0.978572) ≈ 47742`.
- `runnable_avg_yN_inv[n]` (n ∈ [0..31]) = `floor(y^n * 2^32)` — pre-computed lookup so `decay_load` is constant-time for n < 32.

REQ-3: decay_load(val, n) → u64:
- If `n > LOAD_AVG_PERIOD * 63 = 2016`: return 0 (under-flow to zero past ~63 half-lives).
- `local_n = n` (cast to u32).
- If `local_n ≥ LOAD_AVG_PERIOD`:
  - `val >>= local_n / LOAD_AVG_PERIOD` (one halving per 32 periods).
  - `local_n %= LOAD_AVG_PERIOD`.
- `val = mul_u64_u32_shr(val, runnable_avg_yN_inv[local_n], 32)` (multiply by `y^local_n` in fixed-point Q32).
- Return val.

REQ-4: __accumulate_pelt_segments(periods, d1, d3) → u32 (sum of the three segments c1 + c2 + c3):
- /* c1 = d1 * y^periods : remainder of last incomplete period, decayed by `periods` half-lives. */
- `c1 = decay_load((u64)d1, periods)`.
- /* c2 = 1024 * Σ_{n=1..periods-1} y^n = LOAD_AVG_MAX - decay_load(LOAD_AVG_MAX, periods) - 1024. */
- `c2 = LOAD_AVG_MAX - decay_load(LOAD_AVG_MAX, periods) - 1024`.
- /* c3 = d3 * y^0 = d3 : current partial period. */
- `c3 = d3`.
- Return `c1 + c2 + c3`.

REQ-5: accumulate_sum(delta, sa, load, runnable, running) → periods:
- `contrib = (u32)delta` (used only if periods == 0).
- `delta += sa.period_contrib`.
- `periods = delta / 1024`.
- /* Step 1: decay existing `*_sum` over `periods` half-life-units. */
- If periods:
  - `sa.load_sum = decay_load(sa.load_sum, periods)`.
  - `sa.runnable_sum = decay_load(sa.runnable_sum, periods)`.
  - `sa.util_sum = decay_load((u64)sa.util_sum, periods)`.
  - /* Step 2: compute three-segment contribution for new periods. */
  - `delta %= 1024`.
  - If `load`: `contrib = __accumulate_pelt_segments(periods, 1024 - sa.period_contrib, delta)`.
- `sa.period_contrib = delta`.
- If `load`: `sa.load_sum += load * contrib`.
- If `runnable`: `sa.runnable_sum += runnable * contrib << SCHED_CAPACITY_SHIFT`.
- If `running`: `sa.util_sum += contrib << SCHED_CAPACITY_SHIFT`.
- Return periods.

REQ-6: ___update_load_sum(now, sa, load, runnable, running) → int (1 = period(s) crossed):
- `delta = now - sa.last_update_time`.
- If `(s64)delta < 0`: `sa.last_update_time = now; return 0` (clock-rewind protection during early-boot TSC switch).
- `delta >>= 10` (convert ns to ~us via 1024-ns quantum).
- If `delta == 0`: return 0 (no full microsecond elapsed).
- `sa.last_update_time += delta << 10`.
- If `!load`: `runnable = running = 0` (running ⟹ load).
- If `!accumulate_sum(delta, sa, load, runnable, running)`: return 0 (no period crossed; `*_avg` unchanged).
- Return 1.

REQ-7: ___update_load_avg(sa, load):
- `divider = get_pelt_divider(sa) = PELT_MIN_DIVIDER + sa.period_contrib`.
- `sa.load_avg = div_u64(load * sa.load_sum, divider)`.
- `sa.runnable_avg = div_u64(sa.runnable_sum, divider)`.
- `WRITE_ONCE(sa.util_avg, sa.util_sum / divider)`.

REQ-8: __update_load_avg_blocked_se(now, se) → int:
- For a sleeping (blocked) sched_entity: `load = 0, runnable = 0, running = 0`.
- `if (___update_load_sum(now, &se.avg, 0, 0, 0)) { ___update_load_avg(&se.avg, se_weight(se)); trace_pelt_se_tp(se); return 1; }`.
- Return 0.

REQ-9: __update_load_avg_se(now, cfs_rq, se) → int:
- `load = !!se.on_rq` (entity-is-on-runqueue boolean).
- `runnable = se_runnable(se)` — for task: `!!on_rq`; for group_se: `grq->h_nr_runnable`.
- `running = (cfs_rq.curr == se)`.
- `if (___update_load_sum(now, &se.avg, load, runnable, running)) { ___update_load_avg(&se.avg, se_weight(se)); cfs_se_util_change(&se.avg); trace_pelt_se_tp(se); return 1; }`.
- Return 0.
- /* `se_weight(se)` for task = `se.load.weight`; for group_se = `tg.weight * grq.load_avg / tg.load_avg` (computed elsewhere). */

REQ-10: __update_load_avg_cfs_rq(now, cfs_rq) → int:
- `load = scale_load_down(cfs_rq.load.weight)`.
- `runnable = cfs_rq.h_nr_runnable`.
- `running = (cfs_rq.curr != NULL)`.
- `if (___update_load_sum(now, &cfs_rq.avg, load, runnable, running)) { ___update_load_avg(&cfs_rq.avg, 1); trace_pelt_cfs_tp(cfs_rq); return 1; }`.
- Return 0.
- /* Note: divisor `load` passed as 1 — cfs_rq's `load_avg` is summed externally, not weighted at divide time. */

REQ-11: update_rt_rq_load_avg(now, rq, running) → int (CONFIG-gated under RT class):
- `load = running, runnable = running, running = running` (all same).
- `if (___update_load_sum(now, &rq.avg_rt, running, running, running)) { ___update_load_avg(&rq.avg_rt, 1); trace_pelt_rt_tp(rq); return 1; }`.
- /* RT tracks only `util_avg`; `load_avg`/`runnable_avg` are meaningless for RT here. */

REQ-12: update_dl_rq_load_avg(now, rq, running) → int (same shape as RT for `rq.avg_dl`).

REQ-13: update_hw_load_avg(now, rq, capacity) → int (CONFIG_SCHED_HW_PRESSURE):
- Tracks **HW pressure** (thermal / hardware throttling): "delta capacity" = `arch_scale_cpu_capacity(cpu) - capacity_now`.
- `load = runnable = running = capacity` (treated symmetrically).
- `if (___update_load_sum(now, &rq.avg_hw, capacity, capacity, capacity))`: update_load_avg + trace.

REQ-14: update_irq_load_avg(rq, running) → int (CONFIG_HAVE_SCHED_AVG_IRQ):
- IRQ time is **not** accounted in `rq->clock_task`, so PELT cannot use `rq_clock_pelt` directly.
- Scale `running` by `arch_scale_freq_capacity(cpu_of(rq))` then by `arch_scale_cpu_capacity(cpu_of(rq))` (full invariance).
- Two-step update:
  - `___update_load_sum(rq.clock - running, &rq.avg_irq, 0, 0, 0)` — back-fill decay to "just-before-IRQ" time.
  - `___update_load_sum(rq.clock, &rq.avg_irq, 1, 1, 1)` — add the IRQ-busy period.
- If either updated: `___update_load_avg(&rq.avg_irq, 1); trace_pelt_irq_tp(rq)`.

REQ-15: update_other_load_avgs(rq) → bool (caller: `__sched_balance_update_blocked_averages`, etc.):
- `now = rq_clock_pelt(rq)`; `curr_class = rq.donor.sched_class`; `hw_pressure = arch_scale_hw_pressure(cpu_of(rq))`.
- `lockdep_assert_rq_held(rq)`.
- Aggregate-OR:
  - `update_rt_rq_load_avg(now, rq, curr_class == &rt_sched_class)`.
  - `update_dl_rq_load_avg(now, rq, curr_class == &dl_sched_class)`.
  - `update_hw_load_avg(rq_clock_task(rq), rq, hw_pressure)` (uses `clock_task` not `clock_pelt` — HW doesn't care about invariance).
  - `update_irq_load_avg(rq, 0)`.

REQ-16: Frequency + CPU invariance:
- `util_sum` contribution for a `running` quantum is `contrib << SCHED_CAPACITY_SHIFT` — but the **time** that gets accumulated is `rq_clock_pelt`, which is defined as `rq->clock_task * arch_scale_freq_capacity(cpu) * arch_scale_cpu_capacity(cpu) / (SCHED_CAPACITY_SCALE^2)`.
- Net: `util_avg ∈ [0, 1024]` represents the fraction of **maximum-CPU-capacity at maximum-frequency** that the entity uses. A task that uses 50% of a 1.0GHz CPU running at 50% of its max-2GHz frequency yields `util_avg ≈ 256` (25%).

REQ-17: Geometric-series math:
- Definition: `u_avg = Σ_{n=0..∞} u_n * y^n / (Σ_{n=0..∞} y^n) = u_sum / (1024 / (1-y)) ≈ u_sum / LOAD_AVG_MAX`.
- Per-period bookkeeping: when a new period rolls over, multiply `u_sum` by `y` (decay_load with n=1); add new sample `u_0`. This is equivalent to relabeling `u_i → u_{i+1}` and prepending `u_0`.
- Tail bound: terms beyond `LOAD_AVG_PERIOD * 63 = 2016` periods (~2 seconds) round to 0 in the lookup math.

REQ-18: PELT-clock vs task-clock vs IRQ:
- `rq->clock`: monotonic ns since boot (sched_clock).
- `rq->clock_task = rq->clock - rq->prev_irq_time - rq->prev_steal_time` — excludes IRQ + virt-steal.
- `rq->clock_pelt = scaled rq->clock_task` by `arch_scale_freq_capacity` and `arch_scale_cpu_capacity` (PELT-internal, only used for fair / RT / DL utilisation).
- IRQ load uses `rq->clock` directly because its invariance is applied per-call (REQ-14).
- HW pressure uses `rq->clock_task` (unscaled, since "delta capacity" is intrinsic).

REQ-19: Per-cfs_se_util_change (estimated-util sentinel):
- On `__update_load_avg_se` (running task update), reset `sa.util_est & UTIL_AVG_UNCHANGED` so `util_est_update` will refresh.

REQ-20: Group-entity propagation summary (cross-ref `fair.c::propagate_entity_load_avg`):
- `sched_entity` (task): `load_sum := runnable; load_avg = se_weight(se) * load_sum`.
- `sched_entity` (group): `se_weight = tg->weight * grq->load_avg / tg->load_avg`; `se_runnable = grq->h_nr_runnable`.
- `cfs_rq` aggregates: `load_sum = Σ se_weight(se) * se.avg.load_sum`; `load_avg = Σ se.avg.load_avg`; `runnable_sum = Σ se.avg.runnable_sum`; `runnable_avg = Σ se.avg.runnable_avg`.

REQ-21: Clock-rewind protection:
- `___update_load_sum` early-returns 0 on `(s64)(now - last_update_time) < 0` to handle sched_clock TSC handover at boot.
- `decay_load` clamps to 0 past `LOAD_AVG_PERIOD * 63` periods to bound runaway-blocked decay.

REQ-22: Caller contract:
- All `__update_load_avg_*` and `update_*_load_avg` callers must hold `rq->lock` (asserted by `lockdep_assert_rq_held` in `update_other_load_avgs`).
- `__update_load_avg_se` requires `se` to be either:
  - on `cfs_rq` (`on_rq` may be true), OR
  - the blocked-decay path uses `__update_load_avg_blocked_se` instead.

## Acceptance Criteria

- [ ] AC-1: `decay_load(LOAD_AVG_MAX, 32) == LOAD_AVG_MAX / 2` (half-life identity).
- [ ] AC-2: `decay_load(val, n)` for n > 2016 returns 0.
- [ ] AC-3: `accumulate_sum` with `periods == 0`: only `period_contrib += delta` and direct `_sum` contributions; no decay.
- [ ] AC-4: `accumulate_sum` with `periods > 0`: `load_sum, runnable_sum, util_sum` each multiplied by `y^periods`.
- [ ] AC-5: `___update_load_sum` returns 0 when `delta < 1024ns` (no quantum elapsed).
- [ ] AC-6: `___update_load_sum` returns 0 on backward clock without updating sums.
- [ ] AC-7: `___update_load_avg` with `period_contrib == 0` uses `divider = PELT_MIN_DIVIDER` (= 46718).
- [ ] AC-8: `__update_load_avg_blocked_se(now, se)` with `se.on_rq == 0` decays `load_sum/util_sum/runnable_sum` only (no new contribution).
- [ ] AC-9: `__update_load_avg_se(now, cfs_rq, se)` with `cfs_rq.curr == se`: bumps `util_sum`.
- [ ] AC-10: `__update_load_avg_cfs_rq` `running` boolean = `(cfs_rq.curr != NULL)`.
- [ ] AC-11: `update_irq_load_avg` performs a two-step `___update_load_sum` (decay-to-pre-IRQ, then add IRQ window).
- [ ] AC-12: `update_irq_load_avg`'s `running` argument is pre-scaled by `arch_scale_freq_capacity` ∘ `arch_scale_cpu_capacity`.
- [ ] AC-13: `update_other_load_avgs` requires `rq->lock` held (lockdep assert).
- [ ] AC-14: `util_avg ≤ SCHED_CAPACITY_SCALE` (= 1024) under normal operation (bounded by util_sum / divider).
- [ ] AC-15: `load_avg` for a task ≤ `se.load.weight` (bounded by load_sum / divider * weight / load).

## Architecture

```
struct SchedAvg {
  last_update_time: u64,      // ns, quantised to 1024-ns
  load_sum:         u64,      // Σ load * y^p, bounded ≤ LOAD_AVG_MAX * load
  runnable_sum:     u64,      // Σ runnable * y^p << SCHED_CAPACITY_SHIFT
  util_sum:         u32,      // Σ running * y^p << SCHED_CAPACITY_SHIFT
  period_contrib:   u32,      // 0..1023 us into current period
  load_avg:         u64,      // load_sum / divider * weight / load
  runnable_avg:     u64,      // runnable_sum / divider
  util_avg:         u32,      // util_sum / divider, ≤ 1024
  util_est:         u32,      // EWMA of recent util peaks (managed in fair.c)
}

const LOAD_AVG_PERIOD: u32 = 32;          // y^32 = 0.5
const LOAD_AVG_MAX:    u32 = 47742;       // 1024 / (1 - y)
const PELT_MIN_DIVIDER: u32 = LOAD_AVG_MAX - 1024; // = 46718

// y^n in fixed-point Q32, n ∈ [0..31]
const Y_N_INV: [u32; 32] = sched_pelt::RUNNABLE_AVG_YN_INV;
```

`Pelt::decay_load(val: u64, n: u64) -> u64`:
1. If `n > LOAD_AVG_PERIOD * 63`: return 0.
2. local_n = n as u32.
3. If `local_n ≥ LOAD_AVG_PERIOD`:
   - val >>= local_n / LOAD_AVG_PERIOD.
   - local_n %= LOAD_AVG_PERIOD.
4. Return `mul_u64_u32_shr(val, Y_N_INV[local_n], 32)`.

`Pelt::accumulate_segments(periods: u64, d1: u32, d3: u32) -> u32`:
1. c1 = decay_load(d1 as u64, periods) as u32.
2. c2 = LOAD_AVG_MAX - decay_load(LOAD_AVG_MAX as u64, periods) as u32 - 1024.
3. c3 = d3.
4. Return c1 + c2 + c3.

`Pelt::accumulate_sum(delta: u64, sa: &mut SchedAvg, load: u64, runnable: u64, running: bool) -> u64`:
1. contrib = delta as u32.
2. delta += sa.period_contrib as u64.
3. periods = delta / 1024.
4. If periods != 0:
   - sa.load_sum = decay_load(sa.load_sum, periods).
   - sa.runnable_sum = decay_load(sa.runnable_sum, periods).
   - sa.util_sum = decay_load(sa.util_sum as u64, periods) as u32.
   - delta %= 1024.
   - If load != 0: contrib = accumulate_segments(periods, 1024 - sa.period_contrib, delta as u32).
5. sa.period_contrib = delta as u32.
6. If load != 0: sa.load_sum += load * contrib as u64.
7. If runnable != 0: sa.runnable_sum += (runnable * contrib as u64) << SCHED_CAPACITY_SHIFT.
8. If running: sa.util_sum += (contrib as u64) << SCHED_CAPACITY_SHIFT) as u32.
9. Return periods.

`Pelt::update_load_sum(now: u64, sa: &mut SchedAvg, load: u64, runnable: u64, running: bool) -> bool`:
1. delta = now - sa.last_update_time.
2. If (delta as i64) < 0: sa.last_update_time = now; return false.
3. delta >>= 10.
4. If delta == 0: return false.
5. sa.last_update_time += delta << 10.
6. If load == 0: runnable = 0; running = false.
7. If accumulate_sum(delta, sa, load, runnable, running) == 0: return false.
8. Return true.

`Pelt::update_load_avg(sa: &mut SchedAvg, load: u64)`:
1. divider = PELT_MIN_DIVIDER + sa.period_contrib.
2. sa.load_avg = div_u64(load * sa.load_sum, divider as u64).
3. sa.runnable_avg = div_u64(sa.runnable_sum, divider as u64).
4. WRITE_ONCE(sa.util_avg, sa.util_sum / divider).

`Pelt::update_load_avg_blocked_se(now: u64, se: &mut SchedEntity) -> bool`:
1. If update_load_sum(now, &mut se.avg, 0, 0, false): update_load_avg(&mut se.avg, se_weight(se)); trace_pelt_se(se); return true.
2. Return false.

`Pelt::update_load_avg_se(now: u64, cfs_rq: &CfsRq, se: &mut SchedEntity) -> bool`:
1. load = if se.on_rq != 0 { 1 } else { 0 }.
2. runnable = se_runnable(se).
3. running = cfs_rq.curr == Some(se as *const _).
4. If update_load_sum(now, &mut se.avg, load, runnable, running):
   - update_load_avg(&mut se.avg, se_weight(se)).
   - cfs_se_util_change(&mut se.avg).
   - trace_pelt_se(se).
   - Return true.
5. Return false.

`Pelt::update_load_avg_cfs_rq(now: u64, cfs_rq: &mut CfsRq) -> bool`:
1. load = scale_load_down(cfs_rq.load.weight).
2. runnable = cfs_rq.h_nr_runnable as u64.
3. running = cfs_rq.curr.is_some().
4. If update_load_sum(now, &mut cfs_rq.avg, load, runnable, running):
   - update_load_avg(&mut cfs_rq.avg, 1).
   - trace_pelt_cfs(cfs_rq).
   - Return true.
5. Return false.

`Pelt::update_irq_load_avg(rq: &mut Rq, running: u64) -> bool`:
1. cpu = cpu_of(rq).
2. running = cap_scale(running, arch_scale_freq_capacity(cpu)).
3. running = cap_scale(running, arch_scale_cpu_capacity(cpu)).
4. ret  = update_load_sum(rq.clock - running, &mut rq.avg_irq, 0, 0, false) as u8.
5. ret |= update_load_sum(rq.clock,           &mut rq.avg_irq, 1, 1, true)  as u8.
6. If ret != 0: update_load_avg(&mut rq.avg_irq, 1); trace_pelt_irq(rq).
7. Return ret != 0.

`Pelt::update_other_load_avgs(rq: &mut Rq) -> bool`:
1. lockdep_assert_rq_held(rq).
2. now = rq.clock_pelt().
3. curr_class = rq.donor.sched_class.
4. hw_pressure = arch_scale_hw_pressure(cpu_of(rq)).
5. Return any-of:
   - update_rt_rq_load_avg(now, rq, curr_class == &rt_sched_class).
   - update_dl_rq_load_avg(now, rq, curr_class == &dl_sched_class).
   - update_hw_load_avg(rq.clock_task(), rq, hw_pressure).
   - update_irq_load_avg(rq, 0).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `decay_load_half_life` | INVARIANT | per-decay_load: `decay_load(LOAD_AVG_MAX, 32) == LOAD_AVG_MAX / 2`. |
| `decay_load_zero_beyond_63h` | INVARIANT | per-decay_load: `n > LOAD_AVG_PERIOD * 63 ⟹ ret == 0`. |
| `decay_load_monotone` | INVARIANT | per-decay_load: `n1 ≤ n2 ⟹ decay_load(v, n1) ≥ decay_load(v, n2)`. |
| `accumulate_sum_no_overflow` | INVARIANT | per-accumulate_sum: `load_sum + load * contrib ≤ LOAD_AVG_MAX * max_load` (saturating). |
| `period_contrib_bounded` | INVARIANT | per-accumulate_sum: post-condition `0 ≤ period_contrib < 1024`. |
| `util_avg_bounded` | INVARIANT | per-update_load_avg: `util_avg ≤ SCHED_CAPACITY_SCALE` (= 1024). |
| `load_avg_bounded_by_weight` | INVARIANT | per-update_load_avg_se: `se.avg.load_avg ≤ se_weight(se)`. |
| `last_update_time_monotone` | INVARIANT | per-update_load_sum: `last_update_time'` is non-decreasing across calls (clock-rewind clamps to `now`). |
| `update_other_load_avgs_holds_rq_lock` | INVARIANT | per-update_other_load_avgs: `rq->lock` held (lockdep). |

### Layer 2: TLA+

`kernel/sched/pelt.tla`:
- Periodic update model with abstract time advancing in 1024-ns quanta.
- Operations: ENQUEUE_SE, DEQUEUE_SE, CURR_SWITCH, TICK, IRQ_WINDOW.
- Properties:
  - `safety_geometric_decay` — after `k` half-life-periods of no contribution, `*_sum ≤ initial / 2^k`.
  - `safety_util_avg_bounded` — `util_avg ∈ [0, 1024]` always.
  - `safety_no_double_account` — between two updates with `delta < 1024ns`, no change to `*_sum`.
  - `liveness_blocked_decays_to_zero` — a blocked entity's `load_avg` eventually decays below any positive ε.
  - `safety_invariance_compose` — `util_avg` of a task that runs at fraction `f` of capacity `C` at frequency `F` converges to `f * C * F / (C_max * F_max) * 1024`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `decay_load` post: `ret == val * Y_N_INV[n%32] / 2^(32 + n/32)` (modulo rounding) | `Pelt::decay_load` |
| `accumulate_segments` post: `ret == c1 + c2 + c3` where c1, c2, c3 per REQ-4 | `Pelt::accumulate_segments` |
| `accumulate_sum` post: `period_contrib ∈ [0, 1024)`; if `periods == 0`: `_sum` unchanged except direct add | `Pelt::accumulate_sum` |
| `update_load_sum` post: `last_update_time' = old + (delta/1024)*1024` (quantised) | `Pelt::update_load_sum` |
| `update_load_avg` post: `load_avg = load * load_sum / divider`; `util_avg = util_sum / divider` | `Pelt::update_load_avg` |
| `update_load_avg_blocked_se` post: `_sum` only decays; `_avg` reflects the decayed `_sum` | `Pelt::update_load_avg_blocked_se` |
| `update_load_avg_se` post: `running ⟹ util_sum` increases (modulo decay); `runnable ⟹ runnable_sum` increases | `Pelt::update_load_avg_se` |

### Layer 4: Verus/Creusot functional

Per-PELT semantic equivalence to `kernel/sched/pelt.c`:
- Definition: `u_avg = Σ_{n=0..} u_n * y^n / Σ_{n=0..} y^n` where `y = 2^(-1/32)` and `u_n ∈ [0, weight]`.
- Per-`Documentation/scheduler/sched-pelt.c` (the actually-shipped program used to generate `sched-pelt.h`) — the lookup table is regenerated identically.
- Per-`Documentation/scheduler/schedutil.rst` for the `util_avg → DVFS OPP` mapping consumed downstream by `cpufreq_schedutil`.
- Per-paper "The Linux Scheduler: a Decade of Wasted Cores" and Vincent Guittot's PELT-rewrite series.

## Hardening

(Inherits row-1 features from `kernel/sched/00-overview.md` § Hardening.)

PELT reinforcement:

- **Per-clock-rewind early-return** — defense against per-TSC-handover at sched_clock init causing negative delta.
- **Per-decay_load max-n cap (`LOAD_AVG_PERIOD * 63`)** — defense against per-long-sleep-then-update overflow in `decay_load`.
- **Per-`period_contrib < 1024` invariant** — defense against per-divider underflow in `get_pelt_divider`.
- **Per-`util_avg ≤ SCHED_CAPACITY_SCALE` clamp via divider math** — defense against per-schedutil-picks-impossible-OPP.
- **Per-`WRITE_ONCE(util_avg)` for cross-CPU readers** — defense against per-torn-read in `cpu_util_*`.
- **Per-`load == 0 ⟹ runnable = running = 0` normalisation** — defense against per-stale-curr-after-dequeue inflation.
- **Per-`update_other_load_avgs` `lockdep_assert_rq_held`** — defense against per-cross-CPU race updating rq_avg.
- **Per-IRQ two-step decay-then-add** — defense against per-IRQ-time leaking into fair util.
- **Per-invariance scale via `arch_scale_freq_capacity` / `arch_scale_cpu_capacity`** — defense against per-DVFS-skew misleading energy decisions.
- **Per-`mul_u64_u32_shr` (fixed-point Q32)** — defense against per-`decay_load` precision loss for tiny `_sum`.
- **Per-`scale_load_down` of cfs_rq weight** — defense against per-overflow in `load_sum * scale_load_down(weight)`.
- **Per-`PELT_MIN_DIVIDER` floor** — defense against per-divide-by-tiny-divider near a freshly-initialised `period_contrib == 0`.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- High-level scheduler-class overview (covered in `cfs.md` Tier-3)
- CFS / EEVDF runqueue manipulation (covered in `fair-internals.md` Tier-3)
- Load-balance across CPUs (covered in `load-balance.md` Tier-3)
- Energy-aware placement (`find_energy_efficient_cpu`) (covered in `cfs.md` Tier-3)
- `schedutil` cpufreq driver (covered in `cfs.md` Tier-3 / cross-ref `drivers/cpufreq/*`)
- Util-est EWMA logic (lives in `fair.c::util_est_*`, covered in `fair-internals.md`)
- Implementation code
