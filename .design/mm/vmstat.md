# Tier-3: mm/vmstat.c — VM statistics

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: mm/00-overview.md
upstream-paths:
  - mm/vmstat.c (~2435 lines)
  - include/linux/vmstat.h
  - include/linux/mmzone.h (per_cpu_zonestat, per_cpu_nodestat, vm_stat_diff)
-->

## Summary

The **VM statistics subsystem** maintains a three-tier counter hierarchy — per-CPU diffs, per-zone / per-node atomics, and global atomics — that lets every hot path (page alloc, free, LRU shuffle, dirty/writeback) update accounting without touching a cache line shared across CPUs. Per-`__mod_zone_page_state(zone, item, delta)` reads the local-CPU `per_cpu_zonestat::vm_stat_diff[item]`, adds `delta`, and only when `abs(x) > stat_threshold` does it commit to `zone->vm_stat[item]` (atomic) and `vm_zone_stat[item]` (global atomic). Per-`__mod_node_page_state` is the symmetric node-counter path with bytes-vs-pages handling for memcg subpage accounting. Per-`stat_threshold` is computed by `calculate_normal_threshold` = `min(125, 2 * fls(num_online_cpus()) * (1 + fls(zone_managed_pages >> 27)))` — a coarse log scale of CPUs and memory size. Per-`refresh_zone_stat_thresholds` also sets `zone->percpu_drift_mark = high_wmark + max_drift` when `num_online_cpus() * threshold > (low_wmark - min_wmark)`, ensuring watermark math sees real values when drift could cause false positives. Per-`refresh_cpu_vm_stats` (called periodically by `vmstat_update` workqueue and by NOHZ via `quiet_vmstat`) `this_cpu_xchg`'es every per-CPU diff into zone+global atomics. Per-`vmstat_shepherd` walks online CPUs every `sysctl_stat_interval` (default HZ), skipping `cpu_is_isolated()` CPUs, queueing `vmstat_update` work where `need_update(cpu)` returns true. Per-`/proc/vmstat`, `/proc/zoneinfo`, `/proc/buddyinfo`, `/proc/pagetypeinfo` serialise system-wide counters and zone/pageset state. Per-`vm_event_states` (per-CPU `unsigned long event[NR_VM_EVENT_ITEMS]`) handles event counters (PGFAULT, PSWPIN, COMPACTSTALL, …). Per-NUMA `vm_numa_event` tracks NUMA_HIT/MISS/FOREIGN/LOCAL/OTHER toggled by `sysctl_vm_numa_stat`. Critical for: page allocator watermark accuracy, /proc telemetry, memcg accounting, NUMA balancing decisions, kswapd wake-up.

This Tier-3 covers `mm/vmstat.c` (~2435 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `vm_zone_stat[]` | global zone counters | `VmStat::ZONE_GLOBAL` |
| `vm_node_stat[]` | global node counters | `VmStat::NODE_GLOBAL` |
| `vm_numa_event[]` | global NUMA event counters | `VmStat::NUMA_GLOBAL` |
| `vm_event_states` | per-CPU event counters | `VmStat::EVENT_STATES` |
| `__mod_zone_page_state()` | per-zone diff update (preempt-disabled) | `VmStat::mod_zone_local` |
| `__mod_node_page_state()` | per-node diff update | `VmStat::mod_node_local` |
| `mod_zone_page_state()` | per-zone (cmpxchg-local or irq-save) | `VmStat::mod_zone` |
| `mod_node_page_state()` | per-node (cmpxchg-local or irq-save) | `VmStat::mod_node` |
| `__inc_zone_state()` / `__dec_zone_state()` | per-zone ±1 | `VmStat::inc_zone_local` / `dec_zone_local` |
| `__inc_node_state()` / `__dec_node_state()` | per-node ±1 | `VmStat::inc_node_local` / `dec_node_local` |
| `inc_zone_page_state()` / `dec_zone_page_state()` | overstep-mode ±1 | `VmStat::inc_zone` / `dec_zone` |
| `mod_zone_state()` / `mod_node_state()` | per-cmpxchg core loop | `VmStat::mod_zone_inner` / `mod_node_inner` |
| `calculate_normal_threshold()` | per-stat_threshold calc | `VmStat::calc_normal_threshold` |
| `calculate_pressure_threshold()` | per-pressure stat_threshold | `VmStat::calc_pressure_threshold` |
| `refresh_zone_stat_thresholds()` | per-recompute thresholds | `VmStat::refresh_thresholds` |
| `set_pgdat_percpu_threshold()` | per-pgdat threshold under pressure | `VmStat::set_pgdat_threshold` |
| `refresh_cpu_vm_stats()` | per-CPU diff → atomic fold | `VmStat::refresh_cpu_stats` |
| `cpu_vm_stats_fold()` | per-offline-cpu fold | `VmStat::fold_offline_cpu` |
| `drain_zonestat()` | per-unpopulated-zone drain | `VmStat::drain_zonestat` |
| `fold_diff()` | per-diff → global atomic | `VmStat::fold_diff` |
| `fold_vm_zone_numa_events()` | per-NUMA event fold | `VmStat::fold_zone_numa_events` |
| `fold_vm_numa_events()` | per-all-zones NUMA fold | `VmStat::fold_numa_events` |
| `sum_vm_events()` / `all_vm_events()` | per-event aggregate across CPUs | `VmStat::sum_events` / `all_events` |
| `vm_events_fold_cpu()` | per-offline-cpu event fold | `VmStat::fold_offline_cpu_events` |
| `sum_zone_node_page_state()` | per-node sum across zones | `VmStat::sum_zone_node_page_state` |
| `node_page_state()` / `node_page_state_pages()` | per-node atomic read | `VmStat::node_page_state` |
| `vmstat_update()` | per-CPU delayed_work | `VmStat::update_work` |
| `vmstat_shepherd()` | per-deferrable-work scheduler | `VmStat::shepherd_work` |
| `quiet_vmstat()` | per-NOHZ idle fold | `VmStat::quiet` |
| `need_update()` | per-CPU has-diffs check | `VmStat::need_update` |
| `vmstat_refresh()` | per-/proc/sys/vm/stat_refresh handler | `VmStat::sysctl_refresh` |
| `zoneinfo_show()` / `zoneinfo_show_print()` | per-/proc/zoneinfo seq_op | `VmStat::zoneinfo_show` |
| `vmstat_show()` / `vmstat_start()` / `vmstat_next()` / `vmstat_stop()` | per-/proc/vmstat seq_op | `VmStat::vmstat_seq` |
| `frag_show()` / `frag_start()` / `frag_next()` / `frag_stop()` | per-/proc/buddyinfo seq_op | `VmStat::buddyinfo_seq` |
| `pagetypeinfo_show()` | per-/proc/pagetypeinfo seq_op | `VmStat::pagetypeinfo_seq` |
| `fragmentation_index()` / `extfrag_for_order()` | per-debugfs/compaction frag | `VmStat::fragmentation_index` / `extfrag_for_order` |
| `vmstat_text[]` | per-name table | `VmStat::TEXT` |
| `sysctl_vm_numa_stat_handler()` | per-NUMA-stat toggle | `VmStat::sysctl_numa_stat` |
| `init_mm_internals()` | per-boot init | `VmStat::init` |

## Compatibility contract

REQ-1: Counter tiers:
- /* Per-CPU diff: s8 vm_stat_diff[NR_VM_ZONE_STAT_ITEMS] in per_cpu_zonestat */
- /* Per-zone atomic: atomic_long_t vm_stat[NR_VM_ZONE_STAT_ITEMS] in zone */
- /* Global atomic: atomic_long_t vm_zone_stat[NR_VM_ZONE_STAT_ITEMS] */
- /* Symmetric for node: per-CPU vm_node_stat_diff in per_cpu_nodestat, per-pgdat vm_stat, global vm_node_stat */
- /* Per-CPU event: unsigned long event[NR_VM_EVENT_ITEMS] in vm_event_states */
- /* Per-CPU NUMA event: vm_numa_event[NR_VM_NUMA_EVENT_ITEMS] in per_cpu_zonestat */
- /* Global NUMA atomic: atomic_long_t vm_numa_event[NR_VM_NUMA_EVENT_ITEMS] */

REQ-2: __mod_zone_page_state(zone, item, delta):
- /* Caller guarantees IRQ-disabled or preempt-disabled non-IRQ-update context */
- pcp = zone.per_cpu_zonestats.
- p = &pcp.vm_stat_diff[item].
- preempt_disable_nested.
- x = delta + __this_cpu_read(*p).
- t = __this_cpu_read(pcp.stat_threshold).
- if abs(x) > t:
  - zone_page_state_add(x, zone, item).   /* commits to zone->vm_stat[item] + global */
  - x = 0.
- __this_cpu_write(*p, x).
- preempt_enable_nested.

REQ-3: __mod_node_page_state(pgdat, item, delta):
- if vmstat_item_in_bytes(item):
  - VM_WARN_ON_ONCE(delta & (PAGE_SIZE - 1)).
  - delta >>= PAGE_SHIFT.
- /* Otherwise identical to zone path against per_cpu_nodestats.vm_node_stat_diff[item] and pgdat.vm_stat[item] */

REQ-4: __inc_zone_state / __dec_zone_state (overstep optimisation):
- v = __this_cpu_inc_return(*p) (or dec).
- t = stat_threshold.
- if v > t (or v < -t):
  - overstep = t >> 1.
  - zone_page_state_add(v + overstep, zone, item) (or v - overstep).
  - __this_cpu_write(*p, -overstep) (or +overstep).

REQ-5: mod_zone_state(zone, item, delta, overstep_mode) (CONFIG_HAVE_CMPXCHG_LOCAL):
- /* cmpxchg-local loop avoids local_irq_save */
- o = this_cpu_read(*p).
- loop:
  - t = this_cpu_read(pcp.stat_threshold).
  - n = delta + (long)o.
  - if abs(n) > t:
    - os = overstep_mode * (t >> 1).
    - z = n + os.   /* overflow */
    - n = -os.
  - else: z = 0.
- until this_cpu_try_cmpxchg(*p, &o, n) succeeds.
- if z != 0: zone_page_state_add(z, zone, item).

REQ-6: mod_zone_page_state(zone, item, delta) (no CMPXCHG_LOCAL):
- local_irq_save(flags); __mod_zone_page_state; local_irq_restore.

REQ-7: calculate_normal_threshold(zone) -> i32:
- mem = zone_managed_pages(zone) >> (27 - PAGE_SHIFT).
- threshold = 2 * fls(num_online_cpus()) * (1 + fls(mem)).
- return min(125, threshold).

REQ-8: calculate_pressure_threshold(zone) -> i32:
- watermark_distance = low_wmark_pages(zone) - min_wmark_pages(zone).
- threshold = max(1, watermark_distance / num_online_cpus()).
- return min(125, threshold).

REQ-9: refresh_zone_stat_thresholds():
- for_each_online_pgdat(pgdat):
  - for_each_online_cpu(cpu): per_cpu_ptr(pgdat.per_cpu_nodestats, cpu).stat_threshold = 0.
- for_each_populated_zone(zone):
  - threshold = calculate_normal_threshold(zone).
  - for_each_online_cpu(cpu):
    - per_cpu_ptr(zone.per_cpu_zonestats, cpu).stat_threshold = threshold.
    - per_cpu_ptr(pgdat.per_cpu_nodestats, cpu).stat_threshold = max(threshold, prev_pgdat_threshold).
  - tolerate_drift = low_wmark_pages(zone) - min_wmark_pages(zone).
  - max_drift = num_online_cpus() * threshold.
  - if max_drift > tolerate_drift: zone.percpu_drift_mark = high_wmark_pages(zone) + max_drift.

REQ-10: refresh_cpu_vm_stats(do_pagesets) -> bool:
- changed = false.
- for_each_populated_zone(zone):
  - for i in 0..NR_VM_ZONE_STAT_ITEMS:
    - v = this_cpu_xchg(pzstats.vm_stat_diff[i], 0).
    - if v: atomic_long_add(v, &zone.vm_stat[i]); global_zone_diff[i] += v.
  - if do_pagesets:
    - cond_resched.
    - if decay_pcp_high(zone, pcp): changed = true.
    - /* NUMA: expire remote-node pcp pageset after 3 idle ticks */
    - if remote node ∧ expire-counter == 0 ∧ pcp.count > 0: drain_zone_pages.
- for_each_online_pgdat(pgdat):
  - for i in 0..NR_VM_NODE_STAT_ITEMS:
    - v = this_cpu_xchg(p.vm_node_stat_diff[i], 0).
    - if v: atomic_long_add(v, &pgdat.vm_stat[i]); global_node_diff[i] += v.
- if fold_diff(global_zone_diff, global_node_diff): changed = true.
- return changed.

REQ-11: fold_diff(zone_diff, node_diff) -> bool:
- for i in 0..NR_VM_ZONE_STAT_ITEMS:
  - if zone_diff[i]: atomic_long_add(zone_diff[i], &vm_zone_stat[i]); changed = true.
- for i in 0..NR_VM_NODE_STAT_ITEMS:
  - if node_diff[i]: atomic_long_add(node_diff[i], &vm_node_stat[i]); changed = true.
- return changed.

REQ-12: cpu_vm_stats_fold(cpu) (offline path):
- /* No race since cpu is offline */
- For each zone: copy vm_stat_diff[i] into atomics, zero diff.
- For each NUMA-event: copy vm_numa_event[i] into atomics, zero diff.
- For each pgdat: copy vm_node_stat_diff[i] into atomics, zero diff.
- fold_diff into globals.

REQ-13: drain_zonestat(zone, pzstats):
- /* Called only when zone is unpopulated and no other users of pzstats exist */
- For i in 0..NR_VM_ZONE_STAT_ITEMS: if pzstats.vm_stat_diff[i]: commit and zero.
- For i in 0..NR_VM_NUMA_EVENT_ITEMS: if pzstats.vm_numa_event[i]: commit and zero.

REQ-14: vmstat_update(work):
- if refresh_cpu_vm_stats(true):
  - queue_delayed_work_on(smp_processor_id(), mm_percpu_wq, &vmstat_work, round_jiffies_relative(sysctl_stat_interval)).

REQ-15: need_update(cpu) -> bool:
- for_each_populated_zone(zone):
  - if memchr_inv(pzstats.vm_stat_diff, 0, sizeof) != NULL: return true.
  - if per_cpu_ptr(pgdat.per_cpu_nodestats, cpu).vm_node_stat_diff != all-zero: return true.
- return false.

REQ-16: vmstat_shepherd(work):
- cpus_read_lock.
- for_each_online_cpu(cpu):
  - rcu guard.
  - if cpu_is_isolated(cpu): continue.   /* nohz_full / cpuset isolation */
  - if !work_busy(dw.work) ∧ need_update(cpu):
    - queue_delayed_work_on(cpu, mm_percpu_wq, dw, 0).
  - cond_resched.
- cpus_read_unlock.
- schedule_delayed_work(&shepherd, round_jiffies_relative(sysctl_stat_interval)).

REQ-17: quiet_vmstat():
- /* NOHZ-idle entry */
- if system_state != SYSTEM_RUNNING: return.
- if !delayed_work_pending(vmstat_work): return.
- if !need_update(smp_processor_id()): return.
- refresh_cpu_vm_stats(false).

REQ-18: vmstat_refresh (sysctl /proc/sys/vm/stat_refresh):
- schedule_on_each_cpu(refresh_vm_stats).
- /* Negative-stat warning */
- for i in zone_stat: if val < 0 ∧ i ∉ {NR_ZONE_WRITE_PENDING, NR_FREE_CMA_PAGES}: pr_warn.
- for i in node_stat: if val < 0 ∧ i ∉ {NR_WRITEBACK}: pr_warn.

## Acceptance Criteria

- [ ] AC-1: __mod_zone_page_state: delta accumulates in per-CPU diff; commits only when |x| > stat_threshold.
- [ ] AC-2: __mod_node_page_state: bytes items emit VM_WARN_ON_ONCE for non-page-aligned delta and divide by PAGE_SIZE.
- [ ] AC-3: __inc_zone_state with overstep: on v > t writes -overstep to diff (anti-thrash).
- [ ] AC-4: mod_zone_state cmpxchg-local: succeeds without local_irq_save.
- [ ] AC-5: calculate_normal_threshold: 1 CPU, 1 GB zone ⟹ threshold ≈ 8; 1024 CPUs, 16 GB ⟹ threshold = 125 (clamped).
- [ ] AC-6: calculate_pressure_threshold: max(1, watermark_distance / num_online_cpus()), capped at 125.
- [ ] AC-7: refresh_zone_stat_thresholds: percpu_drift_mark set iff num_online_cpus * threshold > low_wmark - min_wmark.
- [ ] AC-8: refresh_cpu_vm_stats: this_cpu_xchg moves non-zero diffs to atomics; returns true iff any diff committed.
- [ ] AC-9: vmstat_shepherd: skips cpu_is_isolated CPUs.
- [ ] AC-10: quiet_vmstat: no-op when !need_update.
- [ ] AC-11: cpu_vm_stats_fold: offline CPU's diffs folded into atomics.
- [ ] AC-12: drain_zonestat: unpopulated-zone diffs drained.
- [ ] AC-13: vmstat_refresh: negative stat (except NR_ZONE_WRITE_PENDING / NR_FREE_CMA_PAGES / NR_WRITEBACK) ⟹ pr_warn.
- [ ] AC-14: /proc/zoneinfo: per-zone NR_FREE_PAGES, watermarks (min/low/high/promo), spanned/present/managed, vm_stat[NR_VM_ZONE_STAT_ITEMS], NUMA events.
- [ ] AC-15: /proc/vmstat: NR_VM_ZONE_STAT_ITEMS + NR_VM_NUMA_EVENT_ITEMS + NR_VM_NODE_STAT_ITEMS + NR_VM_STAT_ITEMS + NR_VM_EVENT_ITEMS rows; trailing "nr_unstable 0" for deprecation compat.
- [ ] AC-16: /proc/buddyinfo: free_area[order].nr_free per zone per order via data_race read.
- [ ] AC-17: sysctl_vm_numa_stat_handler: toggling clears all NUMA counters when disabled; static-branch enable/disable.

## Architecture

```
struct PerCpuZoneStat {
  vm_stat_diff: [i8; NR_VM_ZONE_STAT_ITEMS],
  stat_threshold: i32,
  vm_numa_event: [u64; NR_VM_NUMA_EVENT_ITEMS],   // CONFIG_NUMA
}
struct PerCpuNodeStat {
  vm_node_stat_diff: [i8; NR_VM_NODE_STAT_ITEMS],
  stat_threshold: i32,
}
struct VmEventState {
  event: [u64; NR_VM_EVENT_ITEMS],
}
```

`VmStat::mod_zone_local(zone, item, delta)` (preempt-disabled non-IRQ-update context):
1. pcp = zone.per_cpu_zonestats.
2. preempt_disable_nested.
3. x = delta + this_cpu_read(pcp.vm_stat_diff[item]).
4. t = this_cpu_read(pcp.stat_threshold).
5. if x.unsigned_abs() > t:
   - VmStat::zone_page_state_add(x, zone, item).
   - x = 0.
6. this_cpu_write(pcp.vm_stat_diff[item], x as i8).
7. preempt_enable_nested.

`VmStat::mod_node_local(pgdat, item, delta)`:
1. if vmstat_item_in_bytes(item):
   - debug_assert!(delta & (PAGE_SIZE - 1) == 0).
   - delta >>= PAGE_SHIFT.
2. /* Otherwise identical: per_cpu_nodestats.vm_node_stat_diff[item], pgdat.vm_stat[item] */

`VmStat::mod_zone_inner(zone, item, delta, overstep_mode)` (CMPXCHG_LOCAL):
1. o = this_cpu_read(p).
2. loop:
   - z = 0.
   - t = this_cpu_read(pcp.stat_threshold).
   - n = delta + o as i64.
   - if n.unsigned_abs() > t:
     - os = overstep_mode * (t >> 1).
     - z = n + os.
     - n = -os.
   - if this_cpu_try_cmpxchg(p, &mut o, n as i8): break.
3. if z != 0: VmStat::zone_page_state_add(z, zone, item).

`VmStat::calc_normal_threshold(zone) -> i32`:
1. mem = zone_managed_pages(zone) >> (27 - PAGE_SHIFT).
2. threshold = 2 * fls(num_online_cpus()) * (1 + fls(mem)).
3. return min(125, threshold).

`VmStat::calc_pressure_threshold(zone) -> i32`:
1. watermark_distance = low_wmark_pages(zone) - min_wmark_pages(zone).
2. threshold = max(1, watermark_distance / num_online_cpus()).
3. return min(125, threshold).

`VmStat::refresh_thresholds()`:
1. for pgdat in each_online_pgdat:
   - for cpu in each_online_cpu: per_cpu_ptr(pgdat.per_cpu_nodestats, cpu).stat_threshold = 0.
2. for zone in each_populated_zone:
   - threshold = VmStat::calc_normal_threshold(zone).
   - for cpu in each_online_cpu:
     - per_cpu_ptr(zone.per_cpu_zonestats, cpu).stat_threshold = threshold.
     - prev = per_cpu_ptr(pgdat.per_cpu_nodestats, cpu).stat_threshold.
     - per_cpu_ptr(pgdat.per_cpu_nodestats, cpu).stat_threshold = max(threshold, prev).
   - tolerate = low_wmark_pages(zone) - min_wmark_pages(zone).
   - max_drift = num_online_cpus() * threshold.
   - if max_drift > tolerate: zone.percpu_drift_mark = high_wmark_pages(zone) + max_drift.

`VmStat::refresh_cpu_stats(do_pagesets) -> bool`:
1. let mut global_zone_diff = [0; NR_VM_ZONE_STAT_ITEMS].
2. let mut global_node_diff = [0; NR_VM_NODE_STAT_ITEMS].
3. let mut changed = false.
4. for zone in each_populated_zone:
   - for i in 0..NR_VM_ZONE_STAT_ITEMS:
     - v = this_cpu_xchg(pzstats.vm_stat_diff[i], 0).
     - if v != 0:
       - atomic_long_add(v, &zone.vm_stat[i]).
       - global_zone_diff[i] += v.
       - /* NUMA: pcp.expire = 3 (3 ticks until drain consideration) */
   - if do_pagesets:
     - if decay_pcp_high(zone, pcp): changed = true.
     - /* NUMA remote-pageset expire-and-drain */
5. for pgdat in each_online_pgdat:
   - for i in 0..NR_VM_NODE_STAT_ITEMS:
     - v = this_cpu_xchg(p.vm_node_stat_diff[i], 0).
     - if v != 0: atomic_long_add(v, &pgdat.vm_stat[i]); global_node_diff[i] += v.
6. if VmStat::fold_diff(global_zone_diff, global_node_diff): changed = true.
7. return changed.

`VmStat::fold_diff(zone_diff, node_diff) -> bool`:
1. let mut changed = false.
2. for i in 0..NR_VM_ZONE_STAT_ITEMS:
   - if zone_diff[i] != 0: atomic_long_add(zone_diff[i], &vm_zone_stat[i]); changed = true.
3. for i in 0..NR_VM_NODE_STAT_ITEMS:
   - if node_diff[i] != 0: atomic_long_add(node_diff[i], &vm_node_stat[i]); changed = true.
4. return changed.

`VmStat::fold_offline_cpu(cpu)`:
1. for zone in each_populated_zone:
   - pzstats = per_cpu_ptr(zone.per_cpu_zonestats, cpu).
   - for i in 0..NR_VM_ZONE_STAT_ITEMS:
     - if pzstats.vm_stat_diff[i] != 0:
       - v = pzstats.vm_stat_diff[i]; pzstats.vm_stat_diff[i] = 0.
       - atomic_long_add(v, &zone.vm_stat[i]); global_zone_diff[i] += v.
   - /* CONFIG_NUMA: same for vm_numa_event */
2. for pgdat in each_online_pgdat:
   - p = per_cpu_ptr(pgdat.per_cpu_nodestats, cpu).
   - for i in 0..NR_VM_NODE_STAT_ITEMS: similar.
3. VmStat::fold_diff(global_zone_diff, global_node_diff).

`VmStat::update_work(work)` (per-CPU delayed_work):
1. if VmStat::refresh_cpu_stats(true):
   - queue_delayed_work_on(smp_processor_id(), mm_percpu_wq, &vmstat_work, round_jiffies_relative(sysctl_stat_interval)).

`VmStat::need_update(cpu) -> bool`:
1. let mut last_pgdat = None.
2. for zone in each_populated_zone:
   - pzstats = per_cpu_ptr(zone.per_cpu_zonestats, cpu).
   - if memchr_inv(pzstats.vm_stat_diff, 0, size_of::<_>()): return true.
   - if last_pgdat == Some(zone.zone_pgdat): continue.
   - last_pgdat = Some(zone.zone_pgdat).
   - n = per_cpu_ptr(zone.zone_pgdat.per_cpu_nodestats, cpu).
   - if memchr_inv(n.vm_node_stat_diff, 0, size_of::<_>()): return true.
3. return false.

`VmStat::quiet()` (NOHZ idle):
1. if system_state != SYSTEM_RUNNING: return.
2. if !delayed_work_pending(this_cpu_ptr(&vmstat_work)): return.
3. if !VmStat::need_update(smp_processor_id()): return.
4. VmStat::refresh_cpu_stats(false).

`VmStat::shepherd_work(work)`:
1. cpus_read_lock.
2. for cpu in each_online_cpu:
   - rcu guard:
     - if cpu_is_isolated(cpu): continue.
     - if !work_busy(dw.work) ∧ VmStat::need_update(cpu):
       - queue_delayed_work_on(cpu, mm_percpu_wq, dw, 0).
   - cond_resched.
3. cpus_read_unlock.
4. schedule_delayed_work(&shepherd, round_jiffies_relative(sysctl_stat_interval)).

`VmStat::sum_events(*ret)`:
1. memset(ret, 0, NR_VM_EVENT_ITEMS * sizeof(unsigned long)).
2. for cpu in each_online_cpu:
   - this = &per_cpu(vm_event_states, cpu).
   - for i in 0..NR_VM_EVENT_ITEMS: ret[i] += this.event[i].

`VmStat::all_events(*ret)`:
1. cpus_read_lock; VmStat::sum_events(ret); cpus_read_unlock.

`VmStat::vmstat_seq_start(m, *pos) -> *unsigned long`:
1. if *pos >= NR_VMSTAT_ITEMS: return None.
2. BUILD_BUG_ON(ARRAY_SIZE(vmstat_text) != NR_VMSTAT_ITEMS).
3. VmStat::fold_numa_events.
4. v = kmalloc_array(NR_VMSTAT_ITEMS, sizeof(unsigned long), GFP_KERNEL); m.private = v.
5. for i in 0..NR_VM_ZONE_STAT_ITEMS: v[i] = global_zone_page_state(i).
6. v += NR_VM_ZONE_STAT_ITEMS.
7. /* CONFIG_NUMA */ for i in 0..NR_VM_NUMA_EVENT_ITEMS: v[i] = global_numa_event_state(i).
8. for i in 0..NR_VM_NODE_STAT_ITEMS: v[i] = global_node_page_state_pages(i); if THP: v[i] /= HPAGE_PMD_NR.
9. global_dirty_limits(&v[NR_DIRTY_BG_THRESHOLD], &v[NR_DIRTY_THRESHOLD]).
10. v[NR_MEMMAP_PAGES] = nr_memmap_pages; v[NR_MEMMAP_BOOT_PAGES] = nr_memmap_boot_pages.
11. VmStat::all_events(v + NR_VM_STAT_ITEMS_OFFSET).
12. v[PGPGIN] /= 2; v[PGPGOUT] /= 2.  /* sectors → kbytes */
13. return &m.private[*pos].

`VmStat::zoneinfo_show_print(m, pgdat, zone)`:
1. seq_printf "Node %d, zone %8s".
2. if is_zone_first_populated(pgdat, zone):
   - seq_printf "\n  per-node stats".
   - for i in 0..NR_VM_NODE_STAT_ITEMS: print pages (THP-scaled if in_thp).
3. seq_printf pages free, boost, min, low, high, promo, spanned, present, managed, cma.
4. seq_printf protection: (lowmem_reserve[0..MAX_NR_ZONES]).
5. if !populated_zone: return.
6. for i in 0..NR_VM_ZONE_STAT_ITEMS: print zone_page_state.
7. /* CONFIG_NUMA */ fold_vm_zone_numa_events; for i: print zone_numa_event_state.
8. for cpu in each_online_cpu: pageset (count, high, batch, high_min, high_max, stat_threshold).
9. node_unreclaimable, start_pfn, reserved_highatomic, free_highatomic.

`VmStat::sysctl_numa_stat(table, write, buffer, length, ppos) -> i32`:
1. mutex_lock(&vm_numa_stat_lock).
2. if write: oldval = sysctl_vm_numa_stat.
3. ret = proc_dointvec_minmax(table, write, buffer, length, ppos).
4. if oldval == new: goto out.
5. if new == ENABLE_NUMA_STAT: static_branch_enable(&vm_numa_stat_key); pr_info("enable numa statistics").
6. else: static_branch_disable(&vm_numa_stat_key); invalid_numa_statistics(); pr_info("disable numa statistics, and clear numa counters").
7. out: mutex_unlock; return ret.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `mod_zone_local_preempt_disabled` | INVARIANT | per-__mod_zone_page_state: body runs with preempt disabled. |
| `mod_zone_local_threshold_commit` | INVARIANT | per-__mod_zone_page_state: if |x| > t then diff cleared and zone atomic increased by x. |
| `mod_zone_local_diff_bounded` | INVARIANT | per-__mod_zone_page_state: post-call diff in [-threshold, +threshold]. |
| `mod_node_bytes_aligned` | INVARIANT | per-__mod_node_page_state: vmstat_item_in_bytes ⟹ delta is PAGE_SIZE-aligned. |
| `inc_overstep_pre_charge` | INVARIANT | per-__inc_zone_state: v > t ⟹ diff becomes -(t/2). |
| `cmpxchg_loop_terminates` | LIVENESS | per-mod_zone_state: cmpxchg loop terminates in finite steps. |
| `threshold_clamped_125` | INVARIANT | per-calculate_normal_threshold: result ≤ 125. |
| `pressure_threshold_ge1` | INVARIANT | per-calculate_pressure_threshold: result ≥ 1. |
| `refresh_thresholds_drift_correct` | INVARIANT | per-refresh_zone_stat_thresholds: percpu_drift_mark > high_wmark when set. |
| `refresh_cpu_xchg_atomicity` | INVARIANT | per-refresh_cpu_vm_stats: this_cpu_xchg paired with atomic_long_add for committed v. |
| `fold_offline_no_loss` | INVARIANT | per-cpu_vm_stats_fold: sum(diff_before) == sum committed to atomics. |
| `shepherd_skip_isolated` | INVARIANT | per-vmstat_shepherd: cpu_is_isolated CPUs never queued. |
| `vmstat_text_len_matches_items` | INVARIANT | per-init: ARRAY_SIZE(vmstat_text) == NR_VMSTAT_ITEMS. |

### Layer 2: TLA+

`mm/vmstat.tla`:
- Per-CPU diff accumulate + per-threshold commit + per-shepherd flush + per-NOHZ quiet.
- Properties:
  - `safety_diff_threshold_bound` — per-CPU diff in [-T, T] outside of update windows.
  - `safety_sum_invariant` — sum(per_cpu_diff[item]) + zone.vm_stat[item] = monotonic conservation across mod and fold.
  - `safety_global_eq_sum_zone` — for each item: global vm_zone_stat[item] = sum over zones of zone.vm_stat[item] (modulo concurrent updates).
  - `safety_refresh_clears_diff` — per-refresh: post diff is 0 for items that committed.
  - `safety_shepherd_skip_isolated` — per-shepherd: isolated CPUs never queued.
  - `liveness_shepherd_periodic` — per-shepherd: eventually re-armed every sysctl_stat_interval.
  - `liveness_diff_drains` — per-CPU-with-updates: eventually flushed to global.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `VmStat::mod_zone_local` post: \|new_diff\| ≤ threshold | `VmStat::mod_zone_local` |
| `VmStat::mod_zone_inner` post: monotonic conservation (delta == diff_delta + atomic_delta) | `VmStat::mod_zone_inner` |
| `VmStat::calc_normal_threshold` post: 0 ≤ ret ≤ 125 | `VmStat::calc_normal_threshold` |
| `VmStat::refresh_thresholds` post: ∀cpu: stat_threshold ∈ [0, 125] | `VmStat::refresh_thresholds` |
| `VmStat::refresh_cpu_stats` post: per-CPU diffs ≡ 0 modulo concurrent updates after xchg | `VmStat::refresh_cpu_stats` |
| `VmStat::fold_diff` post: globals incremented exactly by zone_diff and node_diff | `VmStat::fold_diff` |
| `VmStat::shepherd_work` post: queued ⟹ !cpu_is_isolated ∧ need_update | `VmStat::shepherd_work` |
| `VmStat::quiet` post: !need_update post-flush | `VmStat::quiet` |
| `VmStat::vmstat_seq_start` post: NR_VMSTAT_ITEMS rows filled | `VmStat::vmstat_seq_start` |

### Layer 4: Verus/Creusot functional

`Per-counter-update → per-CPU diff → threshold-cross commit → zone/node atomic → global atomic, periodically folded by vmstat_update / vmstat_shepherd / quiet_vmstat, reported by /proc/vmstat / /proc/zoneinfo` semantic equivalence: per-Documentation/admin-guide/sysctl/vm.rst (stat_interval, stat_refresh, numa_stat) + Documentation/admin-guide/mm/numa_memory_policy.rst.

## Hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening.)

vmstat reinforcement:

- **Per-CPU diff is s8** — defense against per-counter-cache-line-contention by making 32 items fit in a single 32-byte block; threshold cap of 125 ensures no signed-overflow.
- **Per-stat_threshold clamp at 125** — defense against per-unbounded drift accumulating before fold.
- **Per-percpu_drift_mark = high_wmark + max_drift** — defense against per-watermark-false-positive when max_drift > tolerate_drift (page allocator sees true free pages, not optimistic estimate).
- **Per-preempt_disable_nested in __mod_*** — defense against per-PREEMPT_RT counter corruption (under PREEMPT_RT, local_lock_irq does not disable preemption).
- **Per-cmpxchg-local loop in mod_zone_state** — defense against per-IRQ-disabled-window-too-long on heavy update paths.
- **Per-vmstat_item_in_bytes VM_WARN_ON_ONCE** — defense against per-misaligned subpage memcg accounting reaching global counters.
- **Per-fold_diff atomic_long_add only when non-zero** — defense against per-spurious-cache-line-bounce on no-op flushes.
- **Per-vmstat_shepherd skip cpu_is_isolated** — defense against per-NOHZ-full-disturbance (RT workloads).
- **Per-cpu_vm_stats_fold on hotunplug** — defense against per-counter-loss when CPU disappears.
- **Per-drain_zonestat on memory hotremove** — defense against per-stale-diff in unpopulated zone.
- **Per-quiet_vmstat NOHZ-flush** — defense against per-CPU-tickless drift growing without bound.
- **Per-stat_refresh sysctl pr_warn on negative stat** — defense against per-silent-accounting-corruption (operator sees imbalance).
- **Per-sysctl_vm_numa_stat static_branch** — defense against per-NUMA-hot-path branch cost when disabled.
- **Per-/proc/buddyinfo data_race annotation** — defense against per-spurious-KCSAN noise on diagnostic-only reads.
- **Per-zone.lock around walk_zones_in_node print** — defense against per-pageset-mutation during /proc/zoneinfo emit.
- **Per-pagetypeinfo freecount cap (100000)** — defense against per-long-spinlock-hold triggering hard-lockup-detector while iterating huge free_list.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- mm/page_alloc.c watermark math consuming percpu_drift_mark (covered in `page-allocator.md` Tier-3)
- mm/vmscan.c counters consumed by reclaim decisions (covered in `reclaim.md` Tier-3 if expanded)
- mm/memcontrol.c memcg counter integration (covered in `memcg.md` Tier-3)
- mm/compaction.c fragmentation_index consumer (covered in `migration-compaction.md` Tier-3)
- kernel/sched/isolation.c cpu_is_isolated implementation (covered separately if expanded)
- kernel/workqueue.c delayed_work / mm_percpu_wq (covered separately if expanded)
- Implementation code
