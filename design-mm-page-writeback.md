---
title: "Tier-3: mm/page-writeback.c — Dirty-page throttle and writeback bandwidth control"
tags: ["tier-3", "mm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-11
updated: 2026-05-11
---


## Design Specification

### Summary

`mm/page-writeback.c` implements the **dirty-page writeback throttle** that bounds how many cache pages may be in the *dirty* state before producer tasks are paused so the *flusher* kthreads (`wb_workfn`) can drain them to disk. Per-`vm.dirty_background_ratio` / `vm.dirty_background_bytes`: soft threshold at which background flushing starts. Per-`vm.dirty_ratio` / `vm.dirty_bytes`: hard threshold at which tasks dirtying pages enter `balance_dirty_pages` and are *throttled* (sleep). Per-`vm.dirty_writeback_centisecs`: periodic flusher wakeup interval. Per-`vm.dirty_expire_centisecs`: maximum age of a dirty page before it is forcibly written back. Per-bdi (backing-device-info) tracking via `struct bdi_writeback` (`wb`): each bdi+memcg pair owns dirty/writeback statistics, a write-bandwidth estimate, and a *task_ratelimit* derived setpoint. The IO-less throttling design replaces the older "task waits on actual IO" approach: `balance_dirty_pages` computes a *pause* duration based on `task_dirty_ratelimit` and `io_schedule_timeout`s the dirtying task without holding any IO descriptor, decoupling dirtier latency from disk completion. Two `wb_domain`s exist: the global one (`global_wb_domain`) and one per memcg (when `CONFIG_CGROUP_WRITEBACK`); per-task throttling honors the lower of the two pos_ratios. Critical for: memory-pressure-without-OOM, fair shares between dirty-heavy and clean-heavy producers, container writeback isolation.

This Tier-3 covers `mm/page-writeback.c` (~3110 lines). The flusher kthread loop body in `fs/fs-writeback.c` is referenced for context but its detailed design lives in a separate Tier-3.

### Acceptance Criteria

- [ ] AC-1: Dirtying pages while below `bg_thresh`: zero throttle pauses; current->nr_dirtied_pause set.
- [ ] AC-2: Crossing `bg_thresh`: `wb_start_background_writeback` called, flusher kthread wakes.
- [ ] AC-3: Crossing hard `thresh`: dirtying task enters `balance_dirty_pages` and is `io_schedule_timeout`-paused.
- [ ] AC-4: `task_ratelimit` decreases as `dirty` approaches `thresh` (pos_ratio polynomial verified).
- [ ] AC-5: `task_ratelimit == 0` ⟹ pause = `max_pause` (saturated throttle).
- [ ] AC-6: `BDP_ASYNC` flag: returns `-EAGAIN` instead of sleeping.
- [ ] AC-7: Fatal signal pending: loop breaks even if still over threshold.
- [ ] AC-8: vm.dirty_writeback_centisecs = 0: periodic flusher disabled (timer not rearmed).
- [ ] AC-9: vm.dirty_expire_centisecs honored: oldest dirty inode flushed by then.
- [ ] AC-10: vm.dirty_ratio = 40 then write dirty_bytes = 1<<30: dirty_ratio reads 0 (mutual exclusion).
- [ ] AC-11: cgroup memcg over its own threshold but global below: still throttled (mdtc.pos_ratio < gdtc.pos_ratio chosen).
- [ ] AC-12: BDI_CAP_STRICTLIMIT: per-bdi proportional share strictly enforces wb_thresh regardless of global slack.
- [ ] AC-13: `wb_calc_thresh` honors `min_ratio` / `max_ratio` clamps.
- [ ] AC-14: `folio_clear_dirty_for_io` keeps PAGECACHE_TAG_DIRTY set in xarray (concurrent sync-walk discovery).
- [ ] AC-15: `folio_start_writeback` increments WB_WRITEBACK, `folio_end_writeback` decrements and increments WB_WRITTEN.
- [ ] AC-16: `folio_wait_writeback_killable` returns -EINTR on SIGKILL while waiting.
- [ ] AC-17: `tag_pages_for_writeback` is preemption-safe (re-locks per XA_CHECK_SCHED batch).

### Architecture

```
struct WbDomain {
  lock: SpinLock,
  completions: FpropGlobal,
  period_timer: Timer,
  period_time: u64,                  // jiffies
  dirty_limit_tstamp: u64,
  dirty_limit: u64,                  // smoothed
}

struct DirtyThrottleControl {
  wb: *BdiWriteback,
  dom: *WbDomain,
  gdtc: Option<*DirtyThrottleControl>, // mdtc has gdtc parent; gdtc has None
  wb_completions: *FpropLocalPercpu,
  avail: u64,
  dirty: u64,
  thresh: u64,
  bg_thresh: u64,
  wb_dirty: u64,
  wb_thresh: u64,
  wb_bg_thresh: u64,
  pos_ratio: i64,                    // scaled by RATELIMIT_CALC_SHIFT
  freerun: bool,
  dirty_exceeded: bool,
}

struct BdiWriteback {
  bdi: *BackingDevInfo,
  state: AtomicU64,                  // WB_registered | WB_writeback_running | ...
  last_old_flush: u64,
  b_dirty: ListHead<Inode>,
  b_io: ListHead<Inode>,
  b_more_io: ListHead<Inode>,
  b_dirty_time: ListHead<Inode>,
  list_lock: SpinLock,
  bw_time_stamp: u64,
  dirtied_stamp: u64,
  written_stamp: u64,
  write_bandwidth: u64,
  avg_write_bandwidth: u64,
  dirty_ratelimit: u64,
  balanced_dirty_ratelimit: u64,
  pos_ratio: i64,
  dirty_exceeded: bool,
  stat: [PercpuCounter; NR_WB_STAT_ITEMS],
  dwork: DelayedWork,
  bw_dwork: DelayedWork,
  start_all_reason: u32,
  memcg_css: Option<*CgroupSubsysState>,
  blkcg_css: Option<*CgroupSubsysState>,
  refcnt: PercpuRef,                 // cgwb only
}

struct BackingDevInfo {
  bdi_list: ListHead,
  wb_list: ListHead,
  min_ratio: u32,                    // numerator over 1<<RATIO_SHIFT
  max_ratio: u32,
  min_bytes: u64,
  max_bytes: u64,
  capabilities: u32,                 // BDI_CAP_*
  last_bdp_sleep: u64,
  ra_pages: u32,
  io_pages: u32,
  wb: BdiWriteback,                  // embedded root wb
  cgwb_tree: RadixTree<u64, BdiWriteback>, // CONFIG_CGROUP_WRITEBACK
  cgwb_release_mutex: Mutex,
  wb_waitq: WaitQueue,
}
```

`Writeback::balance_dirty_pages_ratelimited_flags(mapping, flags) -> Result<()>`:
1. if !mapping_can_writeback(mapping): return Ok(()).
2. let bdp = this_cpu(&bdp_ratelimits).
3. let dirtied = current.nr_dirtied.
4. if dirtied < current.nr_dirtied_pause ∧ bdp.load() < ratelimit_pages: return Ok(()).
5. /* Slow path */
6. let wb = inode_to_wb(mapping.host).
7. Writeback::balance_dirty_pages(wb, dirtied, flags)?.
8. bdp.store(0).
9. Ok(())

`Writeback::balance_dirty_pages(wb, pages_dirtied, flags) -> Result<()>`:
1. let mut gdtc = DirtyThrottleControl::gdtc(wb).
2. let mut mdtc = DirtyThrottleControl::mdtc(wb, &gdtc).
3. let strictlimit = wb.bdi.capabilities & BDI_CAP_STRICTLIMIT != 0.
4. let start_time = jiffies.
5. loop:
   a. let now = jiffies.
   b. let nr_dirty = global_node_page_state(NR_FILE_DIRTY).
   c. balance_domain_limits(&mut gdtc, strictlimit).
   d. if mdtc.valid(): balance_domain_limits(&mut mdtc, strictlimit).
   e. if !writeback_in_progress(wb) ∧ (nr_dirty > gdtc.bg_thresh ∨ (strictlimit ∧ gdtc.wb_dirty > gdtc.wb_bg_thresh)):
      - wb_start_background_writeback(wb).
   f. if gdtc.freerun ∧ (!mdtc.valid() ∨ mdtc.freerun):
      - current.dirty_paused_when = now.
      - current.nr_dirtied = 0.
      - current.nr_dirtied_pause = min(intv, m_intv).
      - return Ok(()).
   g. if !writeback_in_progress(wb): wb_start_background_writeback(wb).
   h. mem_cgroup_flush_foreign(wb).
   i. balance_wb_limits(&mut gdtc, strictlimit).
   j. let mut sdtc = &gdtc.
   k. if mdtc.valid():
      - balance_wb_limits(&mut mdtc, strictlimit).
      - if mdtc.pos_ratio < gdtc.pos_ratio: sdtc = &mdtc.
   l. wb.dirty_exceeded = gdtc.dirty_exceeded ∨ (mdtc.valid() ∧ mdtc.dirty_exceeded).
   m. if now > wb.bw_time_stamp.load() + BANDWIDTH_INTERVAL:
      - __wb_update_bandwidth(&gdtc, &mdtc, true).
   n. let dirty_ratelimit = wb.dirty_ratelimit.load().
   o. let task_ratelimit = ((dirty_ratelimit as u128) * sdtc.pos_ratio) >> RATELIMIT_CALC_SHIFT.
   p. let max_pause = wb_max_pause(wb, sdtc.wb_dirty).
   q. let (min_pause, nr_dirtied_pause) = wb_min_pause(wb, max_pause, task_ratelimit, dirty_ratelimit).
   r. let (period, pause) = if task_ratelimit == 0 { (max_pause, max_pause) } else {
        let p = HZ * pages_dirtied / task_ratelimit;
        (p, p - (now - current.dirty_paused_when))
      }.
   s. if pause < min_pause: /* virtual-time accounting */ break.
   t. if pause > max_pause: pause = max_pause.
   u. if flags & BDP_ASYNC: return Err(EAGAIN).
   v. __set_current_state(TASK_KILLABLE).
   w. wb.bdi.last_bdp_sleep = jiffies.
   x. io_schedule_timeout(pause).
   y. current.dirty_paused_when = now + pause.
   z. current.nr_dirtied = 0; current.nr_dirtied_pause = nr_dirtied_pause.
   aa. if task_ratelimit != 0: break.
   bb. if sdtc.wb_dirty <= wb_stat_error(): break.
   cc. if fatal_signal_pending(current): break.
6. Ok(())

`Writeback::domain_dirty_limits(dtc)`:
1. /* Per-global or per-memcg dtc */
2. let (bytes, bg_bytes, ratio, bg_ratio) = if dtc.is_global() {
     (vm_dirty_bytes, dirty_background_bytes, vm_dirty_ratio, dirty_background_ratio)
   } else {
     /* memcg ratios from memcg config */
     mem_cgroup_dirty_limits(dtc.dom.memcg)
   }.
3. dtc.avail = if dtc.is_global() { global_dirtyable_memory() } else { mem_cgroup_avail(dtc.dom.memcg) }.
4. let thresh = if bytes != 0 { bytes / PAGE_SIZE } else { ratio * dtc.avail / 100 }.
5. let mut bg = if bg_bytes != 0 { bg_bytes / PAGE_SIZE } else { bg_ratio * dtc.avail / 100 }.
6. if bg >= thresh: bg = thresh / 2.
7. dtc.thresh = thresh; dtc.bg_thresh = bg.

`Writeback::wb_calc_thresh(wb, thresh) -> u64`:
1. let dom = &global_wb_domain.
2. let numerator = wb.completions.local_fraction().
3. let denominator = dom.completions.global_fraction().
4. let mut share = (thresh * numerator) / denominator.
5. let (min_share, max_share) = wb_min_max_ratio(wb).
6. if share < min_share: share = min_share.
7. if share > max_share: share = max_share.
8. share

`Writeback::wb_position_ratio(dtc)`:
1. let setpoint = (dtc.thresh + dtc.bg_thresh) / 2.
2. let limit = hard_dirty_limit(dtc.dom, dtc.dirty).
3. if dtc.dirty < dirty_freerun_ceiling(dtc.thresh, dtc.bg_thresh):
   - dtc.pos_ratio = 2 << RATELIMIT_CALC_SHIFT.
   - return.
4. let mut pos = pos_ratio_polynom(setpoint, dtc.dirty, limit).
5. /* Adjust by per-wb share vs global setpoint */
6. let wb_setpoint = setpoint * dtc.wb_thresh / dtc.thresh.
7. let wb_pos = pos_ratio_polynom(wb_setpoint, dtc.wb_dirty, dtc.wb_thresh).
8. pos = min(pos, wb_pos).
9. dtc.pos_ratio = pos.

`Writeback::__wb_update_bandwidth(gdtc, mdtc, update_ratelimit)`:
1. let now = jiffies.
2. let elapsed = now - wb.bw_time_stamp.
3. if elapsed < BANDWIDTH_INTERVAL: return.
4. let dirtied = wb_stat(wb, WB_DIRTIED) - wb.dirtied_stamp.
5. let written = wb_stat(wb, WB_WRITTEN) - wb.written_stamp.
6. wb_update_write_bandwidth(wb, elapsed, written).
7. if update_ratelimit:
   - wb_update_dirty_ratelimit(gdtc, dirtied, elapsed).
   - if mdtc: wb_update_dirty_ratelimit(mdtc, dirtied, elapsed).
8. wb.bw_time_stamp = now.
9. wb.dirtied_stamp = wb_stat(wb, WB_DIRTIED).
10. wb.written_stamp = wb_stat(wb, WB_WRITTEN).

`Writeback::folio_mark_dirty(folio) -> bool`:
1. let mapping = folio_mapping(folio).
2. if mapping.is_some():
   - if folio_test_reclaim(folio): folio_clear_reclaim(folio).
   - return mapping.a_ops.dirty_folio(mapping, folio).
3. noop_dirty_folio(None, folio)

`Writeback::filemap_dirty_folio(mapping, folio) -> bool`:
1. if folio_test_set_dirty(folio): return false.
2. __folio_mark_dirty(folio, mapping, !folio_test_private(folio)).
3. if mapping.host.is_some(): __mark_inode_dirty(mapping.host, I_DIRTY_PAGES).
4. true

`Writeback::folio_mark_dirty_inner(folio, mapping, warn)`:
1. xa_lock_irqsave(&mapping.i_pages).
2. if folio.mapping.is_some() (not truncated):
   - if warn ∧ !folio_test_uptodate(folio): WARN.
   - folio_account_dirtied(folio, mapping).
   - __xa_set_mark(&mapping.i_pages, folio.index, PAGECACHE_TAG_DIRTY).
3. xa_unlock_irqrestore.

`Writeback::folio_clear_dirty_for_io(folio) -> bool`:
1. VM_BUG_ON_FOLIO(!folio_test_locked(folio)).
2. let mapping = folio_mapping(folio).
3. if mapping.is_some() ∧ mapping_can_writeback(mapping):
   - /* fancy dance: ensure (a) accounting (b) PTE dirty-bit sync (c) clean return */
   - if folio_mkclean(folio) is racing: re-set dirty.
   - if folio_test_clear_dirty(folio):
     - folio_account_cleaned(folio, inode_to_wb(mapping.host)).
     - return true.
4. false

`Writeback::folio_start_writeback(folio, keep_write)`:
1. let mapping = folio_mapping(folio).
2. xa_lock_irqsave(&mapping.i_pages).
3. /* Set PG_writeback */
4. folio_test_set_writeback(folio).
5. /* Account */
6. NR_WRITEBACK += nr; NR_ZONE_WRITE_PENDING += nr; WB_WRITEBACK += nr.
7. __xa_set_mark(&mapping.i_pages, folio.index, PAGECACHE_TAG_WRITEBACK).
8. if !keep_write: __xa_clear_mark(&mapping.i_pages, folio.index, PAGECACHE_TAG_DIRTY).
9. /* TOWRITE → cleared */
10. __xa_clear_mark(&mapping.i_pages, folio.index, PAGECACHE_TAG_TOWRITE).
11. xa_unlock_irqrestore.
12. wb_inode_writeback_start(inode_to_wb(mapping.host)).

`Writeback::folio_end_writeback(folio) -> bool`:
1. let mapping = folio_mapping(folio).
2. xa_lock_irqsave(&mapping.i_pages).
3. /* Clear PG_writeback */
4. folio_test_clear_writeback(folio).
5. __xa_clear_mark(&mapping.i_pages, folio.index, PAGECACHE_TAG_WRITEBACK).
6. /* Account */
7. NR_WRITEBACK -= nr; NR_ZONE_WRITE_PENDING -= nr; WB_WRITEBACK -= nr; WB_WRITTEN += nr.
8. xa_unlock_irqrestore.
9. wb_inode_writeback_end(inode_to_wb(mapping.host)).
10. wake_up_page(folio, PG_writeback).
11. true

### Out of Scope

- `fs/fs-writeback.c` — `wb_workfn`, `wb_writeback_work`, periodic kicks, `__mark_inode_dirty`, inode-level dirtytime (separate Tier-3 in `fs/`).
- `mm/backing-dev.c` — `cgwb_create`, `wb_init`/`exit`, `bdi_register` lifecycle (separate Tier-3 `backing-dev.md`).
- `mm/filemap.c` — `do_writepages` callers, `__filemap_fdatawrite_range`, sync writepage iteration (covered in `page-cache.md`).
- `mm/memcontrol.c` — memcg writeback domain integration, `mem_cgroup_track_foreign_dirty`, `mem_cgroup_flush_foreign` (covered in `memcg.md`).
- `block/blk-cgroup.c` — blkcg writeback throttling (separate Tier-3 in `block/`).
- `include/linux/lib/fprop.h` — floating-proportions math (separate Tier-3 in `lib/`).
- Stable-pages integrity (DIF/DIX, BDI_CAP_STABLE_WRITES) — covered in `block/integrity.md`.
- Per-fs `dirty_folio` / `writepage` / `writepages` callbacks — per-filesystem Tier-4.
- Implementation code.

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct dirty_throttle_control` | per-throttle calc context (gdtc/mdtc) | `DirtyThrottleControl` |
| `struct wb_domain` | per-domain writeout completions | `WbDomain` |
| `struct bdi_writeback` | per-bdi+memcg writeback context | `BdiWriteback` |
| `struct backing_dev_info` | per-bdi metadata | `BackingDevInfo` |
| `struct fprop_local_percpu` | per-CPU floating-proportions | `FpropLocalPercpu` |
| `balance_dirty_pages()` | per-throttle dirtier loop | `Writeback::balance_dirty_pages` |
| `balance_dirty_pages_ratelimited()` | per-fast-path entry | `Writeback::balance_dirty_pages_ratelimited` |
| `balance_dirty_pages_ratelimited_flags()` | per-async-aware variant | `Writeback::balance_dirty_pages_ratelimited_flags` |
| `domain_dirty_limits()` | per-domain thresh/bg_thresh calc | `Writeback::domain_dirty_limits` |
| `global_dirty_limits()` | per-global thresh/bg pair | `Writeback::global_dirty_limits` |
| `global_dirtyable_memory()` | per-global universe | `Writeback::global_dirtyable_memory` |
| `node_dirty_ok()` | per-NUMA-node dirty ceiling | `Writeback::node_dirty_ok` |
| `wb_calc_thresh()` | per-bdi dirty share | `Writeback::wb_calc_thresh` |
| `cgwb_calc_thresh()` | per-cgwb dirty share | `Writeback::cgwb_calc_thresh` |
| `wb_position_ratio()` | per-bdi setpoint deviation | `Writeback::wb_position_ratio` |
| `wb_update_write_bandwidth()` | per-bdi bw-estimate update | `Writeback::wb_update_write_bandwidth` |
| `wb_update_dirty_ratelimit()` | per-bdi task_ratelimit update | `Writeback::wb_update_dirty_ratelimit` |
| `__wb_update_bandwidth()` | per-periodic bw refresh | `Writeback::wb_update_bandwidth_inner` |
| `wb_update_bandwidth()` | per-public-API refresh | `Writeback::wb_update_bandwidth` |
| `wb_bandwidth_estimate_start()` | per-fresh-estimate seed | `Writeback::wb_bandwidth_estimate_start` |
| `wb_max_pause()` | per-pause ceiling | `Writeback::wb_max_pause` |
| `wb_min_pause()` | per-pause floor + grace | `Writeback::wb_min_pause` |
| `dirty_poll_interval()` | per-freerun poll interval | `Writeback::dirty_poll_interval` |
| `wb_over_bg_thresh()` | per-flusher continue-condition | `Writeback::wb_over_bg_thresh` |
| `wb_domain_init()` / `_exit()` | per-domain lifecycle | `WbDomain::init` / `exit` |
| `wb_writeout_inc()` | per-completion event | `BdiWriteback::writeout_inc` |
| `wb_domain_writeout_add()` | per-domain completion add | `Writeback::wb_domain_writeout_add` |
| `writeout_period()` | per-period timer | `Writeback::writeout_period` |
| `bdi_set_min_ratio()` / `_max_ratio()` | per-bdi cap setters | `BackingDevInfo::set_min_ratio` / `set_max_ratio` |
| `bdi_set_min_bytes()` / `_max_bytes()` | per-bdi cap byte setters | `BackingDevInfo::set_min_bytes` / `set_max_bytes` |
| `bdi_set_strict_limit()` | per-bdi strict toggle | `BackingDevInfo::set_strict_limit` |
| `tag_pages_for_writeback()` | per-range TOWRITE tag | `Writeback::tag_pages_for_writeback` |
| `writeback_iter()` / `writeback_get_folio()` | per-folio iteration over PAGECACHE_TAG_DIRTY | `Writeback::writeback_iter` |
| `do_writepages()` | per-mapping dispatch | `Writeback::do_writepages` |
| `folio_mark_dirty()` | per-folio dirty-notify | `Writeback::folio_mark_dirty` |
| `__folio_mark_dirty()` | per-folio xarray-tag setter | `Writeback::folio_mark_dirty_inner` |
| `filemap_dirty_folio()` | per-fs-no-buffer-head dirtier | `Writeback::filemap_dirty_folio` |
| `folio_account_dirtied()` | per-stat-credit | `Writeback::folio_account_dirtied` |
| `folio_account_cleaned()` | per-stat-debit | `Writeback::folio_account_cleaned` |
| `folio_redirty_for_writepage()` | per-skip-redirty | `Writeback::folio_redirty_for_writepage` |
| `folio_clear_dirty_for_io()` | per-pre-writeout clear | `Writeback::folio_clear_dirty_for_io` |
| `__folio_cancel_dirty()` | per-truncate-only cancel | `Writeback::folio_cancel_dirty` |
| `__folio_start_writeback()` | per-PG_writeback set | `Writeback::folio_start_writeback` |
| `__folio_end_writeback()` | per-PG_writeback clear | `Writeback::folio_end_writeback` |
| `folio_wait_writeback()` | per-completion wait | `Writeback::folio_wait_writeback` |
| `folio_wait_writeback_killable()` | per-fatal-aware wait | `Writeback::folio_wait_writeback_killable` |
| `folio_wait_stable()` | per-stable-page wait | `Writeback::folio_wait_stable` |
| `noop_dirty_folio()` | per-fs-fallback dirty | `Writeback::noop_dirty_folio` |
| `wb_inode_writeback_start()` / `_end()` | per-bdi wb-inode counter | `BdiWriteback::inode_writeback_start` / `end` |
| `dirty_background_ratio_handler()` etc. | per-sysctl change handlers | `Writeback::sysctl_*_handler` |
| `dirty_writeback_centisecs_handler()` | per-flusher interval handler | `Writeback::sysctl_writeback_centisecs_handler` |
| `writeback_set_ratelimit()` | per-rebalance ratelimit_pages | `Writeback::set_ratelimit` |
| `page_writeback_init()` | per-subsystem init | `Writeback::init` |
| `vm.dirty_background_ratio` / `vm.dirty_ratio` / `vm.dirty_*_bytes` / `vm.dirty_writeback_centisecs` / `vm.dirty_expire_centisecs` / `vm.laptop_mode` / `vm.highmem_is_dirtyable` | per-sysctl knobs | shared |

### compatibility contract

REQ-1: struct dirty_throttle_control (`gdtc`/`mdtc` pair):
- dom: parent wb_domain (global or memcg).
- gdtc: parent global dtc for memcg dtc (NULL for global).
- wb: target bdi_writeback.
- wb_completions: per-wb completion counter pointer.
- avail: per-domain available pages.
- dirty: domain dirty pages.
- thresh: domain hard dirty threshold.
- bg_thresh: domain background threshold.
- wb_dirty: wb's share of dirty.
- wb_thresh: wb's share of thresh.
- wb_bg_thresh: wb's share of bg_thresh.
- pos_ratio: deviation-from-setpoint signal scaled by RATELIMIT_CALC_SHIFT.
- freerun: bool — domain below freerun-ceiling, no throttle.
- dirty_exceeded: bool — over wb hard limit.

REQ-2: struct wb_domain:
- lock: per-domain spinlock.
- completions: fprop_global of writeback completions.
- period_timer: per-period timer driving writeout_period.
- period_time: jiffies at period start.
- dirty_limit_tstamp: last update of dirty_limit.
- dirty_limit: smoothed dirty hard limit.

REQ-3: struct bdi_writeback (per-bdi+memcg slot — sketch in scope):
- bdi: parent backing_dev_info.
- state: WB_registered / WB_writeback_running / WB_has_dirty_io / WB_start_all flag-bits.
- last_old_flush: jiffies of last periodic flush.
- nr_pages: outstanding pages to write.
- b_dirty / b_io / b_more_io / b_dirty_time: per-state inode lists.
- list_lock: serializes the per-state inode lists.
- bw_time_stamp: jiffies of last bw refresh.
- dirtied_stamp / written_stamp: page counters at last bw refresh.
- write_bandwidth: smoothed bytes/s estimate.
- avg_write_bandwidth: longer-EWMA bandwidth.
- dirty_ratelimit: throttle setpoint (pages/s).
- balanced_dirty_ratelimit: target setpoint.
- pos_ratio: last computed pos_ratio.
- dirty_exceeded: per-wb dirty-over-limit flag.
- stat[NR_WB_STAT_ITEMS]: percpu_counter array (WB_RECLAIMABLE, WB_WRITEBACK, WB_DIRTIED, WB_WRITTEN — defined in `include/linux/backing-dev-defs.h`).
- dwork: delayed_work driving wb_workfn.
- bw_dwork: delayed_work driving bandwidth refresh.
- start_all_reason: enum reason starting a wakeup.
- congested: per-cgwb-only congestion linkage.
- memcg_css / blkcg_css: per-cgwb owners.
- refcnt: percpu_ref for safe teardown (cgwb only).

REQ-4: struct backing_dev_info (sketch in scope):
- bdi_list / wb_list: linkage.
- min_ratio / max_ratio: bdi's promised floor / ceiling of dirty share (parts of 1<<RATIO_SHIFT).
- min_bytes / max_bytes: byte equivalent caps.
- capabilities: BDI_CAP_STRICTLIMIT | BDI_CAP_WRITEBACK_ACCT | BDI_CAP_NO_ACCT_AND_WRITEBACK | ....
- last_bdp_sleep: jiffies of last balance_dirty_pages sleep on this bdi.
- io_pages: per-IO max pages submission.
- wb: embedded root bdi_writeback (non-cgwb case).
- cgwb_tree: radix-tree of cgwbs (CONFIG_CGROUP_WRITEBACK).
- cgwb_release_mutex: serializes cgwb teardown.
- wb_waitq: waitqueue for wb teardown.
- ra_pages / io_pages / device.

REQ-5: balance_dirty_pages_ratelimited{,_flags}(mapping, flags):
- /* Per-fast-path: only call slow path every N dirties */
- bdp = this_cpu_ptr(&bdp_ratelimits).
- if mapping_can_writeback(mapping):
  - dirtied = current.nr_dirtied.
  - if dirtied >= current.nr_dirtied_pause ∨ *bdp >= ratelimit_pages:
    - wb = inode_to_wb(mapping.host).
    - balance_dirty_pages(wb, dirtied, flags).
    - *bdp = 0.

REQ-6: balance_dirty_pages(wb, pages_dirtied, flags):
- Loop:
  - balance_domain_limits(gdtc, strictlimit).
  - if mdtc: balance_domain_limits(mdtc, strictlimit).
  - if !writeback_in_progress(wb) ∧ (nr_dirty > gdtc.bg_thresh ∨ (strictlimit ∧ gdtc.wb_dirty > gdtc.wb_bg_thresh)):
    - wb_start_background_writeback(wb).
  - if gdtc.freerun ∧ (!mdtc ∨ mdtc.freerun):
    - current.dirty_paused_when = now; current.nr_dirtied = 0.
    - current.nr_dirtied_pause = min(intv, m_intv).
    - break.
  - if !writeback_in_progress(wb): wb_start_background_writeback(wb).
  - mem_cgroup_flush_foreign(wb).
  - balance_wb_limits(gdtc, strictlimit).
  - sdtc = gdtc.
  - if mdtc:
    - balance_wb_limits(mdtc, strictlimit).
    - if mdtc.pos_ratio < gdtc.pos_ratio: sdtc = mdtc.
  - wb.dirty_exceeded = gdtc.dirty_exceeded ∨ (mdtc ∧ mdtc.dirty_exceeded).
  - if jiffies > wb.bw_time_stamp + BANDWIDTH_INTERVAL: __wb_update_bandwidth(gdtc, mdtc, true).
  - dirty_ratelimit = wb.dirty_ratelimit.
  - task_ratelimit = (dirty_ratelimit * sdtc.pos_ratio) >> RATELIMIT_CALC_SHIFT.
  - max_pause = wb_max_pause(wb, sdtc.wb_dirty).
  - min_pause = wb_min_pause(wb, max_pause, task_ratelimit, dirty_ratelimit, &nr_dirtied_pause).
  - if task_ratelimit == 0: period = pause = max_pause; goto pause.
  - period = HZ * pages_dirtied / task_ratelimit.
  - pause = period - (now - current.dirty_paused_when).
  - if pause < min_pause: update virtual time; break.
  - if pause > max_pause: pause = max_pause; bump now.
  - if flags & BDP_ASYNC: ret = -EAGAIN; break.
  - __set_current_state(TASK_KILLABLE).
  - bdi.last_bdp_sleep = jiffies.
  - io_schedule_timeout(pause).
  - current.dirty_paused_when = now + pause.
  - current.nr_dirtied = 0; current.nr_dirtied_pause = nr_dirtied_pause.
  - if task_ratelimit: break.
  - if sdtc.wb_dirty <= wb_stat_error(): break.
  - if fatal_signal_pending(current): break.

REQ-7: domain_dirty_limits(dtc):
- /* Per-domain — global or memcg */
- if dom == global_wb_domain:
  - bytes = vm_dirty_bytes; bg_bytes = dirty_background_bytes.
  - ratio = vm_dirty_ratio; bg_ratio = dirty_background_ratio.
  - dtc.avail = global_dirtyable_memory().
- else (memcg):
  - dtc.avail = mem_cgroup_wb_stats…
- if bytes: thresh = bytes / PAGE_SIZE; else thresh = ratio * avail / 100.
- if bg_bytes: bg_thresh = bg_bytes / PAGE_SIZE; else bg_thresh = bg_ratio * avail / 100.
- bg_thresh = min(bg_thresh, thresh / 2).
- dtc.thresh = thresh; dtc.bg_thresh = bg_thresh.

REQ-8: global_dirtyable_memory():
- x = global_zone_page_state(NR_FREE_PAGES).
- x -= min(x, totalreserve_pages).
- x += global_node_page_state(NR_INACTIVE_FILE) + global_node_page_state(NR_ACTIVE_FILE).
- if !vm_highmem_is_dirtyable: x -= highmem_dirtyable_memory(x).
- return x + 1 /* prevent 0 */.

REQ-9: wb_calc_thresh(wb, thresh):
- /* Per-bdi share of a domain's thresh, weighted by recent completion fraction */
- numerator = wb_completions_fraction.
- denominator = global_completions_fraction.
- share = thresh * numerator / denominator.
- /* Clamp to per-bdi min_ratio / max_ratio */
- if share < min_bdi_share: share = min_bdi_share.
- if share > max_bdi_share: share = max_bdi_share.
- return share.

REQ-10: wb_position_ratio(dtc):
- /* Pos ratio = signal: > 1<<RATELIMIT_CALC_SHIFT under-dirty (speed up); < 1<<RATELIMIT_CALC_SHIFT over-dirty (slow down) */
- setpoint = (thresh + bg_thresh) / 2.
- limit = hard_dirty_limit(dom, dirty).
- if dirty < freerun_ceiling: pos_ratio = 2 << RATELIMIT_CALC_SHIFT; return.
- /* polynomial centered on setpoint */
- pos_ratio = pos_ratio_polynom(setpoint, dirty, limit).
- /* per-wb adjust */
- pos_ratio *= wb_share_adjust.
- dtc.pos_ratio = pos_ratio.

REQ-11: wb_update_write_bandwidth(wb, elapsed, written):
- bw = (written * HZ) / elapsed.
- /* EWMA */
- wb.write_bandwidth = (wb.write_bandwidth * 7 + bw) / 8.
- /* Longer-EWMA */
- wb.avg_write_bandwidth = (wb.avg_write_bandwidth * 3 + bw) / 4.

REQ-12: wb_update_dirty_ratelimit(dtc, dirtied, elapsed):
- /* per-Documentation/admin-guide/mm/dynamic-writeback.rst */
- ref_balanced = dom.dirty_ratelimit * (write_bw / dirty_bw) * pos_ratio.
- step = balanced - dirty_ratelimit.
- step >>= RATELIMIT_CALC_SHIFT.
- wb.dirty_ratelimit += step.
- /* Clamp */
- if wb.dirty_ratelimit < 1: wb.dirty_ratelimit = 1.

REQ-13: __wb_update_bandwidth(gdtc, mdtc, update_ratelimit):
- now = jiffies.
- elapsed = now - bw_time_stamp.
- if elapsed < BANDWIDTH_INTERVAL: return.
- /* Per-period BW + ratelimit refresh */
- wb_update_write_bandwidth(wb, elapsed, written - written_stamp).
- if update_ratelimit: wb_update_dirty_ratelimit(gdtc, dirtied - dirtied_stamp, elapsed).
- if mdtc ∧ update_ratelimit: wb_update_dirty_ratelimit(mdtc, ...).
- bw_time_stamp = now.
- dirtied_stamp = wb_stat(wb, WB_DIRTIED).
- written_stamp = wb_stat(wb, WB_WRITTEN).

REQ-14: wb_max_pause(wb, wb_dirty):
- bw = max(wb.dirty_ratelimit, wb.avg_write_bandwidth).
- max_pause = HZ * wb_dirty / (bw + 1).
- max_pause = min(max_pause, MAX_PAUSE).
- /* MAX_PAUSE = HZ/10 by default */
- return max_pause.

REQ-15: wb_min_pause(wb, max_pause, task_ratelimit, dirty_ratelimit, &nr_dirtied_pause):
- /* Compute min pause from desired-throttle steepness */
- t = HZ / 10.
- if dirty_ratelimit < t: t = dirty_ratelimit / 8.
- *nr_dirtied_pause = max(1, ratelimit_pages * dirty_ratelimit / wb.write_bandwidth).
- return min(t, max_pause).

REQ-16: tag_pages_for_writeback(mapping, start, end):
- /* Walk i_pages xarray range [start, end]; tag DIRTY → TOWRITE */
- XA_STATE(xas, &mapping.i_pages, start).
- xas_lock_irq(&xas).
- xas_for_each_marked(&xas, page, end, PAGECACHE_TAG_DIRTY):
  - xas_set_mark(&xas, PAGECACHE_TAG_TOWRITE).
  - if ++tagged % XA_CHECK_SCHED == 0: xas_pause; xa_unlock; cond_resched; relock.
- xa_unlock.

REQ-17: writeback_iter / writeback_get_folio:
- /* Per-PAGECACHE_TAG_TOWRITE iteration walking mapping->i_pages for next folio */
- folio = writeback_get_folio(mapping, wbc) [under xas].
- folio_prepare_writeback: confirm dirty + clear-dirty-for-io, set TOWRITE-not-DIRTY.
- Return iter state for the writepage caller.

REQ-18: folio_mark_dirty / __folio_mark_dirty / filemap_dirty_folio:
- mapping->a_ops->dirty_folio is fs-callback.
- For no-buffer-head fs: filemap_dirty_folio → folio_test_set_dirty → __folio_mark_dirty.
- __folio_mark_dirty under xa_lock_irqsave(&mapping->i_pages):
  - if folio.mapping (not truncated): folio_account_dirtied + __xa_set_mark(..., PAGECACHE_TAG_DIRTY).

REQ-19: folio_account_dirtied / _cleaned:
- account: NR_FILE_DIRTY +nr; NR_ZONE_WRITE_PENDING +nr; NR_DIRTIED +nr; WB_RECLAIMABLE +nr; WB_DIRTIED +nr; bdp_ratelimits +nr; current.nr_dirtied += nr.
- clean (truncate path): NR_FILE_DIRTY -nr; NR_ZONE_WRITE_PENDING -nr; WB_RECLAIMABLE -nr.

REQ-20: folio_clear_dirty_for_io(folio):
- WARN if !locked.
- if mapping_can_writeback: account_cleaned, clear-dirty-bit (keep TAG_DIRTY in xarray for sync-walk discovery).
- return previous-dirty.

REQ-21: __folio_start_writeback / __folio_end_writeback:
- start: PG_writeback set, NR_WRITEBACK +nr, NR_ZONE_WRITE_PENDING +nr, WB_WRITEBACK +nr; xa_set_mark PAGECACHE_TAG_WRITEBACK; wb_inode_writeback_start.
- end: clear PG_writeback, NR_WRITEBACK -nr, NR_ZONE_WRITE_PENDING -nr, WB_WRITEBACK -nr, WB_WRITTEN +nr; xa_clear_mark PAGECACHE_TAG_WRITEBACK; wb_inode_writeback_end; wake_up_page(folio, PG_writeback).

REQ-22: folio_wait_writeback / _killable / _stable:
- wait_on_folio_bit(folio, PG_writeback), TASK_UNINTERRUPTIBLE (or TASK_KILLABLE for _killable).
- _stable: also waits if mapping requires stable pages (e.g., integrity / DIF).

REQ-23: do_writepages(mapping, wbc):
- /* Per-fs dispatch */
- if mapping.a_ops.writepages: ret = mapping.a_ops.writepages(mapping, wbc).
- else: ret = generic_writepages(mapping, wbc).
- /* Bandwidth refresh */
- if wbc.nr_to_write < initial: wb_update_bandwidth(wb).

REQ-24: wb_over_bg_thresh(wb):
- /* Flusher continue-condition */
- balance_domain_limits(gdtc, false).
- if gdtc.dirty > gdtc.bg_thresh: return true.
- if mdtc: balance_domain_limits(mdtc, false); if mdtc.dirty > mdtc.bg_thresh: return true.
- if wb_stat(wb, WB_RECLAIMABLE) > wb.bdi.min_ratio_pages: return true (per-bdi bg threshold).
- return false.

REQ-25: writeback_set_ratelimit():
- /* Rebalance: ratelimit_pages = vm_total_pages / (num_online_cpus * 32) */
- ratelimit_pages = vm_total_pages / (num_online_cpus * 32).
- if ratelimit_pages < 16: ratelimit_pages = 16.

REQ-26: bdi_set_min_ratio / _max_ratio / _bytes / _strict_limit:
- All take a backing_dev_info and set the corresponding cap atomically under bdi_lock.
- Strict-limit (BDI_CAP_STRICTLIMIT): wb thresholds use *only* the per-bdi proportional share, not the global average; intended for slow devices.

REQ-27: page_writeback_init():
- wb_domain_init(&global_wb_domain, GFP_KERNEL).
- cpuhp_setup_state(CPUHP_AP_ONLINE_DYN, "mm/writeback:online", page_writeback_cpu_online).
- cpuhp_setup_state(CPUHP_MM_WRITEBACK_DEAD, ...).
- register_sysctl_init("vm", vm_page_writeback_sysctls).

REQ-28: Per-sysctl semantics:
- vm.dirty_ratio / dirty_bytes are mutually exclusive: writing one clears the other.
- vm.dirty_background_ratio / dirty_background_bytes likewise.
- vm.dirty_writeback_centisecs == 0 disables periodic writeback.
- vm.dirty_expire_centisecs default 30s.
- vm.laptop_mode > 0: writeback waits for longer batches, reducing disk spin-ups.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `balance_dirty_pages_loop_bounded` | INVARIANT | per-balance_dirty_pages: loop exits within bounded iterations OR breaks on fatal_signal_pending OR returns Err(EAGAIN). |
| `pause_within_range` | INVARIANT | per-balance_dirty_pages: pause in [0, max_pause]; max_pause ≤ MAX_PAUSE. |
| `pos_ratio_signed_bound` | INVARIANT | per-wb_position_ratio: pos_ratio in [0, 2<<RATELIMIT_CALC_SHIFT]. |
| `dirty_ratelimit_nonzero` | INVARIANT | per-wb_update_dirty_ratelimit: wb.dirty_ratelimit ≥ 1 post-update. |
| `bg_thresh_le_thresh_div_2` | INVARIANT | per-domain_dirty_limits: bg_thresh ≤ thresh/2. |
| `thresh_le_global_avail` | INVARIANT | per-domain_dirty_limits: thresh ≤ avail. |
| `xa_lock_held_for_dirty_xarray_ops` | INVARIANT | per-__folio_mark_dirty / _start_writeback / _end_writeback: mapping.i_pages xa_lock held. |
| `wb_stat_signed_balance` | INVARIANT | per-folio_account_dirtied / _cleaned: WB_RECLAIMABLE accumulates exactly (no double-count). |
| `bdp_ratelimits_per_cpu_no_cross` | INVARIANT | per-bdp_ratelimits: only accessed via this_cpu_*. |
| `task_ratelimit_in_u64` | INVARIANT | per-balance_dirty_pages: (dirty_ratelimit * pos_ratio) >> RATELIMIT_CALC_SHIFT fits in u64 (no overflow). |

### Layer 2: TLA+

`mm/page-writeback.tla`:
- Per-dirty-page-add (dirtier task) → balance_dirty_pages_ratelimited → balance_dirty_pages slow-path → io_schedule_timeout pause → wake.
- Per-flusher kthread (wb_workfn): consumes b_io list, writes folios, folio_end_writeback decrements WB_WRITEBACK.
- Per-periodic bw refresh (bw_dwork).
- Properties:
  - `safety_dirty_never_exceeds_thresh_for_long` — per-balance_dirty_pages: dirtier eventually pauses such that dirty drops below thresh in bounded time.
  - `safety_freerun_no_pause` — per-balance_dirty_pages: dirty < freerun ⟹ zero pause.
  - `safety_strictlimit_per_bdi` — per-BDI_CAP_STRICTLIMIT: wb_dirty bounded by wb_thresh regardless of global slack.
  - `safety_mdtc_min_pos_ratio` — per-cgwb: balance honors min(mdtc.pos_ratio, gdtc.pos_ratio).
  - `safety_fatal_signal_breaks_loop` — per-balance_dirty_pages: fatal_signal_pending exits loop ≤ 1 iteration.
  - `liveness_dirty_eventually_written` — per-flusher-wake: if dirty > 0 ∧ flusher scheduled then eventually WB_WRITTEN increments.
  - `liveness_balance_dirty_pages_returns` — per-balance_dirty_pages: terminates.
  - `safety_bw_stamp_monotone` — per-__wb_update_bandwidth: bw_time_stamp monotone non-decreasing.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `balance_dirty_pages` post: returns Ok(()) ∨ Err(EAGAIN); never panics | `Writeback::balance_dirty_pages` |
| `balance_dirty_pages_ratelimited_flags` post: bdp_ratelimits == 0 on slow-path return | `Writeback::balance_dirty_pages_ratelimited_flags` |
| `domain_dirty_limits` post: dtc.thresh ≥ dtc.bg_thresh * 2 | `Writeback::domain_dirty_limits` |
| `wb_calc_thresh` post: share in [min_share, max_share] | `Writeback::wb_calc_thresh` |
| `wb_position_ratio` post: dtc.pos_ratio set, no overflow | `Writeback::wb_position_ratio` |
| `wb_update_dirty_ratelimit` post: wb.dirty_ratelimit ≥ 1 | `Writeback::wb_update_dirty_ratelimit` |
| `wb_max_pause` post: 0 ≤ ret ≤ MAX_PAUSE | `Writeback::wb_max_pause` |
| `wb_min_pause` post: 1 ≤ ret ≤ max_pause | `Writeback::wb_min_pause` |
| `tag_pages_for_writeback` post: every DIRTY in [start, end] is also TOWRITE | `Writeback::tag_pages_for_writeback` |
| `folio_mark_dirty_inner` post: i_pages xa_lock not held on return | `Writeback::folio_mark_dirty_inner` |
| `folio_clear_dirty_for_io` post: PAGECACHE_TAG_DIRTY still set in xarray (only PG_dirty cleared) | `Writeback::folio_clear_dirty_for_io` |
| `folio_start_writeback` post: PG_writeback set, PAGECACHE_TAG_WRITEBACK set, WB_WRITEBACK +nr | `Writeback::folio_start_writeback` |
| `folio_end_writeback` post: PG_writeback clear, WB_WRITEBACK -nr, WB_WRITTEN +nr, waiters woken | `Writeback::folio_end_writeback` |
| `bdi_set_min_ratio` post: bdi.min_ratio ≤ bdi.max_ratio | `BackingDevInfo::set_min_ratio` |

### Layer 4: Verus/Creusot functional

`Per-dirtier loop → balance_dirty_pages → io_schedule_timeout pause → wake; concurrent flusher → wb_workfn → folio_start_writeback → folio_end_writeback → WB_WRITTEN increments → pos_ratio recomputed lower → dirtier eventually freed from throttle` — semantic equivalence to per-Documentation/admin-guide/sysctl/vm.rst and per-Documentation/admin-guide/mm/dynamic-writeback.rst formulas (`pos_ratio = (limit - dirty) / (limit - setpoint)`).

### hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening.)

Writeback-throttle reinforcement:

- **Per-balance_dirty_pages fatal-signal escape** — defense against per-OOM-task stuck-in-throttle.
- **Per-pause bounded by max_pause (≤ MAX_PAUSE, HZ/10)** — defense against per-runaway-throttle starvation.
- **Per-pos_ratio polynomial overflow-checked (u128 intermediate)** — defense against per-multiply overflow.
- **Per-strictlimit per-bdi enforcement** — defense against per-slow-USB swamping the page cache.
- **Per-cgwb refcount via percpu_ref** — defense against per-cgwb teardown UAF.
- **Per-i_pages xa_lock_irqsave around dirty / writeback tag mutation** — defense against per-tag racing with truncate / sync.
- **Per-folio_clear_dirty_for_io keeps xarray TAG_DIRTY** — defense against per-sync-walk losing in-flight folios.
- **Per-bdi.last_bdp_sleep timestamp** — defense against per-flusher / dirtier livelock detection.
- **Per-BDP_ASYNC returns -EAGAIN** — defense against per-async-IO blocking.
- **Per-folio_test_set_dirty atomic** — defense against per-double-account on concurrent dirtying.
- **Per-tag_pages_for_writeback preemption-safe (XA_CHECK_SCHED rebatch)** — defense against per-livelock on enormous mappings.
- **Per-sysctl handler mutual-exclusion (dirty_ratio vs dirty_bytes)** — defense against per-stale-setting confusion.
- **Per-vm.highmem_is_dirtyable default 0** — defense against per-32-bit highmem starvation.

