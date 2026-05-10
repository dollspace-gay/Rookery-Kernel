# Tier-3: mm/damon/ — DAMON (Data Access MONitor) + DAMOS (DAMON-based Operations Schemes)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: mm/00-overview.md
upstream-paths:
  - mm/damon/core.c (~3269 lines)
  - mm/damon/vaddr.c (per-VMA monitoring)
  - mm/damon/paddr.c (per-phys-page monitoring)
  - mm/damon/dbgfs.c
  - mm/damon/sysfs.c
  - include/linux/damon.h
  - Documentation/admin-guide/mm/damon/{start, usage}.rst
-->

## Summary

DAMON (Data Access MONitor) is a lightweight per-region working-set + access-frequency profiler. Per-DAMON-context `damon_ctx` defines per-target (task/system) + per-attrs (sample interval, aggr interval, regions_update interval, min/max regions). Per-iter samples per-region access-frequency via sample-based pte-A-bit-check or PG_idle; aggregates into nr_accesses per-region; auto-splits high-frequency regions + merges low-frequency. **DAMOS** layers per-region apply-action triggers on top: per-condition (size + freq + age) action {STAT/WILLNEED/COLD/PAGEOUT/HUGEPAGE/NOHUGEPAGE/LRU_PRIO/LRU_DEPRIO/MIGRATE_HOT/MIGRATE_COLD/...}. Critical for: working-set-size profiling, tiered memory (CXL.mem), proactive reclaim, DAMOS_PAGEOUT policy.

This Tier-3 covers `damon/core.c` (~3269 lines) + brief overview of ops backends.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct damon_ctx` | per-monitoring context | `DamonCtx` |
| `struct damon_target` | per-target (task/syswide) | `DamonTarget` |
| `struct damon_region` | per-(addr-range, nr_accesses) | `DamonRegion` |
| `struct damon_attrs` | per-intervals | `DamonAttrs` |
| `struct damon_operations` | per-arch backend (vaddr / paddr / fvaddr) | `DamonOperations` |
| `struct damos` | per-action-scheme | `Damos` |
| `struct damos_filter` | per-DAMOS filter | `DamosFilter` |
| `struct damos_quota` | per-DAMOS budget | `DamosQuota` |
| `damon_new_ctx()` | per-ctx alloc | `Damon::new_ctx` |
| `damon_select_ops()` | per-ctx ops backend | `Damon::select_ops` |
| `damon_call()` | per-ctx start/stop | `Damon::call` |
| `damon_start()` / `damon_stop()` | per-ctx lifecycle | `Damon::start` / `stop` |
| `damon_register_ops()` | per-ops backend register | `Damon::register_ops` |
| `damos_new_filter()` | per-DAMOS filter | `Damos::new_filter` |
| `damos_add_filter()` | per-DAMOS attach filter | `Damos::add_filter` |
| `DAMON_OPS_VADDR` / `_FVADDR` / `_PADDR` | per-ops ID | UAPI |
| `DAMOS_*` action enum | STAT/WILLNEED/COLD/PAGEOUT/HUGEPAGE/NOHUGEPAGE/LRU_PRIO/LRU_DEPRIO/MIGRATE_HOT/MIGRATE_COLD/... | UAPI |

## Compatibility contract

REQ-1: Per-damon_ctx:
- attrs.sample_interval: us between samples (~5ms).
- attrs.aggr_interval: us between aggregations (~100ms).
- attrs.ops_update_interval: us between region-pruning (~1s).
- attrs.min_nr_regions: per-target lower bound.
- attrs.max_nr_regions: per-target upper bound.
- adaptive_targets: per-target list.
- schemes: per-DAMOS list.

REQ-2: Per-ops backend:
- DAMON_OPS_VADDR: per-task; samples user-space virt-addrs via mm-walk + PTE-A-bit.
- DAMON_OPS_FVADDR: per-fixed-virt-addr (no auto-target-discovery).
- DAMON_OPS_PADDR: per-phys-addr; uses PG_idle/PG_young pageflags.

REQ-3: Per-iter algorithm:
- Sample: per-region pick a random page; check A-bit + clear.
- Aggregate (every aggr_interval): nr_accesses tallied; per-region.
- Split: high-nr_accesses region split into smaller.
- Merge: low-nr_accesses adjacent regions merged.
- Apply DAMOS: per-region matching action-condition: invoke action.

REQ-4: DAMOS scheme:
- pattern.min_sz_region / max_sz_region.
- pattern.min_nr_accesses / max_nr_accesses (in this aggr-window).
- pattern.min_age / max_age (per-aggr-windows).
- action: damos_action enum.
- quota: per-action total bytes budget.
- watermarks: per-DAMON_HEALTH thresholds.
- filters: per-DAMOS filter (e.g. anon-only / specific-memcg).

REQ-5: DAMOS actions:
- DAMOS_STAT: count + log; no-op.
- DAMOS_WILLNEED: madvise(MADV_WILLNEED).
- DAMOS_COLD: madvise(MADV_COLD).
- DAMOS_PAGEOUT: proactive reclaim via madvise(MADV_PAGEOUT).
- DAMOS_HUGEPAGE / NOHUGEPAGE: madvise THP.
- DAMOS_LRU_PRIO / LRU_DEPRIO: per-folio reactivate / deactivate.
- DAMOS_MIGRATE_HOT / MIGRATE_COLD: per-folio NUMA migrate (tiered mem).

REQ-6: damon_call:
- Per-call queues callback for damon-thread.
- Per-callback runs at next aggr-interval; sees current ctx state.

REQ-7: Per-userspace ABI:
- /sys/kernel/mm/damon/admin/{kdamonds, contexts, schemes, ...} (modern sysfs).
- (Deprecated /sys/kernel/debug/damon/...).

REQ-8: Per-NUMA-aware DAMON:
- DAMON_OPS_PADDR aware of node.
- DAMOS_MIGRATE_{HOT,COLD} per-tier.

REQ-9: Per-watermark:
- DAMOS active iff system-metric (e.g. free-memory %) in watermark range.

REQ-10: Per-quota:
- DAMOS per-N-aggr-window budget cap (bytes).
- Goal-driven auto-tuning: adjust budget per-feedback metric.

REQ-11: Per-thread (kdamond):
- Per-damon_ctx has a kdamond kernel-thread.
- kdamond runs the sample → aggregate → apply-DAMOS loop.

## Acceptance Criteria

- [ ] AC-1: damon_new_ctx + damon_select_ops(VADDR): ctx ready.
- [ ] AC-2: damon_start(ctx): kdamond runs; samples per-region per-attrs.
- [ ] AC-3: After aggr_interval: per-region.nr_accesses populated.
- [ ] AC-4: High-freq region: auto-split into smaller.
- [ ] AC-5: Low-freq adjacent: auto-merged.
- [ ] AC-6: DAMOS_PAGEOUT scheme: cold regions proactively paged-out.
- [ ] AC-7: DAMOS_MIGRATE_HOT/COLD (CXL tiered): hot pages → fast tier; cold → slow.
- [ ] AC-8: DAMOS_STAT: per-aggr stats accumulated.
- [ ] AC-9: damon_stop: kdamond exits cleanly.
- [ ] AC-10: /sys/kernel/mm/damon/admin/...: per-ctx + per-scheme exposed.
- [ ] AC-11: DAMOS quota cap enforced.

## Architecture

Per-damon_ctx:

```
struct DamonCtx {
  attrs: DamonAttrs,
  passed_sample_intervals: u64,
  next_aggregation_sis: u64,
  next_ops_update_sis: u64,
  adaptive_targets: ListHead<DamonTarget>,
  schemes: ListHead<Damos>,
  ops: DamonOperations,
  kdamond: *TaskStruct,
  kdamond_lock: Mutex<()>,
  callback: DamonCallback,
}

struct DamonTarget {
  list: ListLink,
  pid: PidT,                                     // for VADDR
  regions_list: ListHead<DamonRegion>,
  nr_regions: u32,
  rss_low: u64,                                  // pgsteal_pageout etc.
  ...
}

struct DamonRegion {
  list: ListLink,
  ar: Addr Range { start, end },
  sampling_addr: u64,
  nr_accesses: u32,
  age: u32,
}

struct DamonAttrs {
  sample_interval: u64,                          // us
  aggr_interval: u64,
  ops_update_interval: u64,
  min_nr_regions: u32,
  max_nr_regions: u32,
}

struct Damos {
  list: ListLink,
  pattern: DamosPattern,                          // size + freq + age range
  action: DamosAction,
  apply_interval_us: u64,
  quota: DamosQuota,
  wmarks: DamosWmarks,
  filters: ListHead<DamosFilter>,
  stat: DamosStat,
}
```

`Damon::new_ctx() -> *DamonCtx`:
1. ctx = kzalloc(sizeof(DamonCtx)).
2. /* Default attrs */
3. ctx.attrs.sample_interval = 5000us.
4. ctx.attrs.aggr_interval = 100ms.
5. ctx.attrs.ops_update_interval = 1s.
6. mutex_init(&ctx.kdamond_lock).
7. Return ctx.

`Damon::select_ops(ctx, ops_id) -> Result<()>`:
1. ops = registered_ops[ops_id].
2. if !ops.init: return -EINVAL.
3. ctx.ops = ops.

`Damon::start(ctx) -> Result<()>`:
1. ctx.kdamond = kthread_run(kdamond_fn, ctx, "kdamond.%d", ...).
2. /* kdamond_fn runs sample / aggregate / apply loop */.

`Damon::kdamond_fn(data)`:
1. ctx = data.
2. ctx.ops.init(ctx).
3. while !kthread_should_stop:
   - sample
   - if (passed_sample_intervals * attrs.sample_interval) >= attrs.aggr_interval:
     - aggregate.
     - damon_split_regions / damon_merge_regions.
     - per-scheme: damon_apply_scheme.
   - if (passed * sample) >= attrs.ops_update_interval:
     - ctx.ops.update.
4. ctx.ops.cleanup(ctx).

`Damon::kdamond_sample(ctx)`:
1. ctx.ops.prepare_access_checks(ctx).
2. ndelay(ctx.attrs.sample_interval).
3. nr_accesses = ctx.ops.check_accesses(ctx).
4. /* Per-region.nr_accesses++ if accessed */

`Damon::damon_apply_scheme(ctx, t, r, scheme)`:
1. /* Per-region match pattern? */
2. if r.ar.end-r.ar.start ∉ [pattern.min_sz_region, max_sz_region]: skip.
3. if r.nr_accesses ∉ [pattern.min_nr_accesses, max_nr_accesses]: skip.
4. if r.age ∉ [pattern.min_age, pattern.max_age]: skip.
5. /* Per-quota check */
6. if scheme.quota.cur_charged > scheme.quota.bytes: skip; record skipped.
7. /* Apply action */
8. ctx.ops.apply_scheme(ctx, t, r, scheme).
9. scheme.stat.{sz_tried, sz_applied} += region-size.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `region_age_monotonic` | INVARIANT | per-aggr: r.age += 1 (or reset on split/merge). |
| `nr_regions_in_range` | INVARIANT | min_nr_regions ≤ #regions ≤ max_nr_regions. |
| `sample_interval_lt_aggr` | INVARIANT | attrs.sample_interval < attrs.aggr_interval. |
| `scheme_quota_not_exceeded` | INVARIANT | per-scheme.quota.cur_charged ≤ quota.bytes. |
| `kdamond_only_when_running` | INVARIANT | ctx.kdamond != NULL ⟹ ctx running. |

### Layer 2: TLA+

`mm/damon.tla`:
- Per-ctx sample → aggregate → split/merge → apply-DAMOS loop.
- Properties:
  - `safety_no_drift_between_intervals` — per-iter: next_aggregation_sis monotonic.
  - `safety_per_scheme_action_per_region_eligible` — per-apply: pattern matched.
  - `liveness_kdamond_eventually_aggregates` — per-aggr_interval elapsed ⟹ aggregate runs.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Damon::start` post: kdamond running; ctx.ops.init invoked | `Damon::start` |
| `Damon::stop` post: kdamond stopped; ctx.ops.cleanup invoked | `Damon::stop` |
| `Damon::kdamond_sample` post: per-region nr_accesses updated | `Damon::kdamond_sample` |
| `Damon::damon_apply_scheme` post: per-eligible region action applied; quota deducted | `Damon::damon_apply_scheme` |

### Layer 4: Verus/Creusot functional

`Per-target periodic per-region access-frequency profiling + per-DAMOS pattern-matched action` semantic equivalence: per-Documentation/admin-guide/mm/damon/usage.rst.

## Hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening.)

DAMON-specific reinforcement:

- **Per-ctx kdamond_lock for state mutation** — defense against per-ctx torn-update.
- **Per-region bounded by min/max_nr_regions** — defense against per-target region-bomb.
- **Per-DAMOS quota** — defense against per-action runaway-budget.
- **Per-DAMOS watermarks gate** — defense against per-system-pressure inappropriate action.
- **Per-DAMOS_PAGEOUT throttled** — defense against per-thrash via excessive reclaim.
- **Per-CAP_SYS_ADMIN for sysfs write** — defense against per-unprivileged DAMON-config.
- **Per-VADDR target task refcount** — defense against per-target UAF.
- **Per-ops_update_interval bounded** — defense against per-tick CPU storm.
- **Per-filter per-scheme** — defense against per-VMA cross-target leak.
- **Per-MIGRATE_HOT/COLD requires NUMA + tiered config** — defense against per-non-tiered system mis-migrate.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- mm/00-overview (Tier-2)
- mm/damon/{vaddr, paddr, fvaddr}.c (per-ops backend; covered separately)
- mm/damon/sysfs.c (per-userspace ABI; covered separately)
- DAMOS_LRU_PRIO interaction with reclaim (covered in `reclaim.md` Tier-3)
- Implementation code
