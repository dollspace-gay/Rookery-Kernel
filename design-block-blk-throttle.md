---
title: "Tier-3: block/blk-throttle.c — Block IO throttling (cgroup blkio v1 + io.max v2)"
tags: ["tier-3", "block", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

The block IO **throttle** policy enforces per-cgroup BPS / IOPS ceilings on each block device. Per-`throtl_data` lives on `request_queue->td`, embedding a top-level `throtl_service_queue` whose `pending_tree` is a red-black tree of children `throtl_grp` keyed by their next eligible dispatch time (`disptime`). Per-`throtl_grp` is a per-blkcg-per-disk policy-data structure storing per-rw `bps[2]`, `iops[2]` limits plus per-rw `bytes_disp[2]`, `io_disp[2]` token-bucket counters and per-rw slice windows `slice_start[2]`, `slice_end[2]` (`DFL_THROTL_SLICE = HZ/10`). Per-`__blk_throtl_bio(bio)` either dispatches (within limit) or queues onto the group's `qnode` and arms the parent service-queue's `pending_timer`. Per-`throtl_pending_timer_fn` walks the rb-tree leftmost-first, calls `throtl_select_dispatch` to extract bios at their `disptime`, propagates them up to the parent service queue, and on reaching `throtl_data->service_queue` queues `dispatch_work` onto the `kthrotld` workqueue. Per-`blk_throtl_dispatch_work_fn` then re-submits via `submit_bio_noacct_nocheck`. Hierarchical (recursive) limit application is gated by `cgroup_subsys_on_dfl(io_cgrp_subsys)` — on default hierarchy a child sq's `parent_sq` is its parent group's sq, on legacy hierarchy every group is parented directly to `throtl_data->service_queue` (flat). Critical for: container BPS/IOPS QoS, multi-tenant disk isolation, deterministic priority-inversion handling for `bio_issue_as_root_blkg`.

This Tier-3 covers `block/blk-throttle.c` (~1849 lines).

### Acceptance Criteria

- [ ] AC-1: `__blk_throtl_bio` within bps + iops limit: returns false (not throttled), bio dispatched immediately.
- [ ] AC-2: `__blk_throtl_bio` over iops limit: returns true (throttled), bio queued, `disptime` set, pending_timer armed.
- [ ] AC-3: `__blk_throtl_bio` over bps limit: returns true, bio queued in bps tier, `BIO_TG_BPS_THROTTLED` set when charged.
- [ ] AC-4: `bio_issue_as_root_blkg` over limit: bypasses queueing, charges debt to `bytes_disp` / `io_disp` (priority-inversion).
- [ ] AC-5: Slice expires → `throtl_trim_slice` decays `bytes_disp` / `io_disp` by `time_elapsed - DFL_THROTL_SLICE` of budget.
- [ ] AC-6: Hierarchy v2 (`io_cgrp_subsys` default): child sq's `parent_sq` = parent group's sq; bios climb tier-by-tier.
- [ ] AC-7: Hierarchy v1 (legacy): every group's `parent_sq` = `td->service_queue` (flat).
- [ ] AC-8: Root blkg on v2: `tg_bps_limit` / `tg_iops_limit` return `U64_MAX` / `UINT_MAX` regardless of `tg->bps` / `tg->iops`.
- [ ] AC-9: `tg_set_conf` (legacy) / `tg_set_limit` (v2): triggers `tg_update_carryover` → `tg_conf_updated` → `tg_update_has_rules` subtree.
- [ ] AC-10: `THROTL_QUANTUM = 32` global per-round cap honored by `throtl_select_dispatch`.
- [ ] AC-11: `THROTL_GRP_QUANTUM = 8` R/W split (6/2) honored by `throtl_dispatch_tg`.
- [ ] AC-12: `throtl_schedule_pending_timer`: expires clamped to `jiffies + 8 * DFL_THROTL_SLICE`.
- [ ] AC-13: `blk_throtl_cancel_bios`: every tg gets `THROTL_TG_CANCELING`; queued bios drained via timer in finite jiffies.
- [ ] AC-14: `blk_throtl_dispatch_work_fn`: runs on `kthrotld_workqueue`; submits drained bios via `submit_bio_noacct_nocheck` under `blk_plug`.
- [ ] AC-15: Limit decreased while bios queued: `__tg_update_carryover` rebases `bytes_disp` / `io_disp` so new rate is honored without "free" burst.

### Architecture

```
struct ThrotlData {
  service_queue: ThrotlServiceQueue,   // parent_sq = None
  queue:         *RequestQueue,
  nr_queued:     [u32; 2],             // [READ, WRITE] at top sq
  dispatch_work: WorkStruct,           // -> kthrotld_workqueue
}

struct ThrotlGrp {
  pd:                 BlkgPolicyData,
  service_queue:      ThrotlServiceQueue,    // parent_sq = parent tg's sq (v2) or td's sq (v1)
  qnode_on_self:      [ThrotlQnode; 2],
  qnode_on_parent:    [ThrotlQnode; 2],
  rb_node:            RbNode,                // in parent_sq.pending_tree
  disptime:           u64,                   // jiffies
  flags:              u32,                   // THROTL_TG_*
  bps:                [u64; 2],              // U64_MAX = no limit
  iops:               [u32; 2],              // UINT_MAX = no limit
  has_rules_bps:      [bool; 2],
  has_rules_iops:     [bool; 2],
  bytes_disp:         [i64; 2],              // signed for carryover
  io_disp:            [i32; 2],
  slice_start:        [u64; 2],
  slice_end:          [u64; 2],
  stat_bytes:         BlkgRwstat,
  stat_ios:           BlkgRwstat,
  td:                 *ThrotlData,
}

struct ThrotlServiceQueue {
  parent_sq:                Option<*ThrotlServiceQueue>,
  queued:                   [ListHead; 2],     // qnode chains, per rw
  nr_queued_bps:            [u32; 2],
  nr_queued_iops:           [u32; 2],
  pending_tree:             RbRootCached,      // children by disptime
  nr_pending:               u32,
  first_pending_disptime:   u64,
  pending_timer:            TimerList,         // -> throtl_pending_timer_fn
}

struct ThrotlQnode {
  bios:    BioList,
  tg:      *ThrotlGrp,
  node:    ListHead,                            // -> ThrotlServiceQueue.queued[rw]
}
```

`Throtl::throtl_bio(bio) -> bool`:
1. rcu_read_lock; spin_lock_irq(queue_lock).
2. sq = &tg.service_queue; rw = bio_data_dir(bio).
3. loop:
   - if ThrotlGrp::within_limit(tg, bio, rw):
     - ThrotlGrp::charge_iops(tg, bio); ThrotlGrp::trim_slice(tg, rw).
   - elif bio_issue_as_root_blkg(bio):
     - ThrotlGrp::charge_bps(tg, bio); ThrotlGrp::charge_iops(tg, bio).
   - else: break (queue).
   - qn = &tg.qnode_on_parent[rw]; sq = sq.parent_sq; tg = sq_to_tg(sq).
   - if tg.is_none(): bio_set_flag(BIO_BPS_THROTTLED); throttled = false; goto unlock.
4. /* queued */ td.nr_queued[rw] += 1; ThrotlGrp::add_bio(bio, qn, tg); throttled = true.
5. if tg.flags & (WAS_EMPTY | IOPS_WAS_EMPTY):
   - ThrotlGrp::update_disptime(tg).
   - ThrotlServiceQueue::schedule_next_dispatch(parent_sq, true).
6. unlock_irq, rcu_unlock; return throttled.

`ThrotlGrp::within_limit(tg, bio, rw) -> bool`:
1. if bio_flagged(BIO_BPS_THROTTLED):
   - return sq.nr_queued_iops[rw] == 0 ∧ ThrotlGrp::dispatch_iops_time(tg, bio) == 0.
2. if sq_queued(sq, rw) > 0:
   - if sq.nr_queued_bps[rw] == 0 ∧ ThrotlGrp::dispatch_bps_time(tg, bio) == 0:
     - ThrotlGrp::charge_bps(tg, bio).
   - return false.
3. return ThrotlGrp::dispatch_time(tg, bio) == 0.

`ThrotlGrp::dispatch_time(tg, bio) -> u64`:
1. BUG_ON sq_queued(sq, rw) > 0 ∧ bio != peek.
2. wait = ThrotlGrp::dispatch_bps_time(tg, bio).
3. if wait != 0: return wait.
4. ThrotlGrp::charge_bps(tg, bio).
5. return ThrotlGrp::dispatch_iops_time(tg, bio).

`Throtl::pending_timer_fn(timer)`:
1. sq = container_of(timer); tg = sq_to_tg(sq); td = sq_to_td(sq).
2. q = tg ? tg.pd.blkg.q : td.queue.
3. spin_lock_irq(q.queue_lock).
4. if !q.root_blkg: goto out.
5. label again:
   - parent_sq = sq.parent_sq; dispatched = false.
   - loop:
     - ret = ThrotlServiceQueue::select_dispatch(sq).
     - if ret: dispatched = true.
     - if ThrotlServiceQueue::schedule_next_dispatch(sq, false): break.
     - unlock; cpu_relax; lock.
   - if !dispatched: goto out.
   - if parent_sq:
     - if tg.flags & (WAS_EMPTY | IOPS_WAS_EMPTY):
       - ThrotlGrp::update_disptime(tg).
       - if !ThrotlServiceQueue::schedule_next_dispatch(parent_sq, false):
         - sq = parent_sq; tg = sq_to_tg(sq); goto again.
   - else: queue_work(kthrotld_workqueue, &td.dispatch_work).
6. out: unlock.

`ThrotlServiceQueue::select_dispatch(parent_sq) -> u32`:
1. nr_disp = 0.
2. loop:
   - if !parent_sq.nr_pending: break.
   - tg = ThrotlServiceQueue::first(parent_sq).
   - if time_before(jiffies, tg.disptime): break.
   - nr_disp += ThrotlGrp::dispatch_one_round(tg).
   - sq = &tg.service_queue.
   - if sq_queued(sq, READ) > 0 ∨ sq_queued(sq, WRITE) > 0:
     - ThrotlGrp::update_disptime(tg).
   - else: ThrotlGrp::dequeue(tg).
   - if nr_disp ≥ THROTL_QUANTUM: break.
3. return nr_disp.

`Throtl::dispatch_work_fn(work)`:
1. td = container_of(work); td_sq = &td.service_queue; q = td.queue.
2. bio_list = BioList::new().
3. spin_lock_irq(q.queue_lock).
4. for rw in [READ, WRITE]:
   - while bio = throtl_pop_queued(td_sq, None, rw): bio_list.push(bio).
5. spin_unlock_irq.
6. if !bio_list.is_empty():
   - blk_start_plug; for bio in bio_list: submit_bio_noacct_nocheck(bio, false); blk_finish_plug.

`Throtl::init_disk(disk) -> Result<()>`:
1. q = disk.queue.
2. td = kzalloc_node(ThrotlData, GFP_KERNEL, q.node)?
3. INIT_WORK(&td.dispatch_work, Throtl::dispatch_work_fn).
4. ThrotlServiceQueue::init(&td.service_queue).
5. memflags = blk_mq_freeze_queue(q); blk_mq_quiesce_queue(q).
6. q.td = td; td.queue = q.
7. ret = blkcg_activate_policy(disk, &THROTL_POLICY).
8. if ret < 0: q.td = None; kfree(td).
9. blk_mq_unquiesce_queue(q); blk_mq_unfreeze_queue(q, memflags).
10. return ret.

`Throtl::cancel_bios(disk)`:
1. q = disk.queue.
2. if !blk_throtl_activated(q): return.
3. spin_lock_irq(q.queue_lock); rcu_read_lock.
4. blkg_for_each_descendant_post(blkg, pos_css, q.root_blkg):
   - ThrotlGrp::flush_bios(blkg_to_tg(blkg)).
5. rcu_unlock; spin_unlock_irq.

### Out of Scope

- `block/blk-cgroup.c` (blkcg core, `struct blkcg_gq` lifecycle) — covered separately under `block/00-overview.md` (cgroup core in `kernel/cgroup/`).
- `block/blk-iocost.c` (cost-model io.cost) — distinct policy, separate Tier-3 if expanded.
- `block/blk-iolatency.c` (io.latency QoS) — distinct policy, separate Tier-3 if expanded.
- `block/blk-mq.c` dispatch (covered in `blk-mq.md` Tier-3).
- `kernel/cgroup/cgroup.c` (covered separately).
- Implementation code.

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct throtl_data` | per-request_queue policy state | `ThrotlData` |
| `struct throtl_grp` | per-blkg policy data | `ThrotlGrp` |
| `struct throtl_service_queue` | per-level rb-dispatch tree | `ThrotlServiceQueue` |
| `struct throtl_qnode` | per-(group, rw) bio fifo | `ThrotlQnode` |
| `blkcg_policy_throtl` | per-policy blkcg-vtable | `THROTL_POLICY` |
| `throtl_pd_alloc` | per-blkg alloc | `Throtl::pd_alloc` |
| `throtl_pd_init` | per-blkg init (parent_sq linkage) | `Throtl::pd_init` |
| `throtl_pd_online` | per-blkg online (recompute has_rules) | `Throtl::pd_online` |
| `throtl_pd_offline` | per-blkg offline (flush bios) | `Throtl::pd_offline` |
| `throtl_pd_free` | per-blkg free | `Throtl::pd_free` |
| `tg_update_has_rules` | per-group recursive has_rules[] | `ThrotlGrp::update_has_rules` |
| `__blk_throtl_bio` | per-bio entry | `Throtl::throtl_bio` |
| `tg_within_limit` | per-bio gate | `ThrotlGrp::within_limit` |
| `tg_dispatch_time` | per-bio wait jiffies | `ThrotlGrp::dispatch_time` |
| `tg_within_bps_limit` | per-bio bps token-bucket | `ThrotlGrp::within_bps_limit` |
| `tg_within_iops_limit` | per-bio iops token-bucket | `ThrotlGrp::within_iops_limit` |
| `throtl_charge_bps_bio` | per-bio bytes charge | `ThrotlGrp::charge_bps` |
| `throtl_charge_iops_bio` | per-bio iops charge | `ThrotlGrp::charge_iops` |
| `throtl_add_bio_tg` | per-bio enqueue + WAS_EMPTY flag | `ThrotlGrp::add_bio` |
| `throtl_qnode_add_bio` / `_peek_queued` / `_pop_queued` | per-qnode fifo | `ThrotlQnode::{add,peek,pop}` |
| `tg_service_queue_add` | per-group rb-insert by disptime | `ThrotlGrp::sq_add` |
| `throtl_rb_first` / `_rb_erase` | per-tree leftmost / erase | `ThrotlServiceQueue::{first,erase}` |
| `throtl_enqueue_tg` / `_dequeue_tg` | per-group PENDING flag | `ThrotlGrp::{enqueue,dequeue}` |
| `tg_update_disptime` | per-group recompute disptime | `ThrotlGrp::update_disptime` |
| `update_min_dispatch_time` | per-sq first_pending_disptime | `ThrotlServiceQueue::update_min_disptime` |
| `throtl_schedule_pending_timer` | per-sq mod_timer (clamped 8*slice) | `ThrotlServiceQueue::schedule_pending_timer` |
| `throtl_schedule_next_dispatch` | per-sq arm-or-continue | `ThrotlServiceQueue::schedule_next_dispatch` |
| `throtl_pending_timer_fn` | per-sq timer callback | `Throtl::pending_timer_fn` |
| `throtl_select_dispatch` | per-sq round-robin dispatch | `ThrotlServiceQueue::select_dispatch` |
| `throtl_dispatch_tg` | per-group QUANTUM dispatch (R 75% / W 25%) | `ThrotlGrp::dispatch_one_round` |
| `tg_dispatch_one_bio` | per-bio transfer up one level | `ThrotlGrp::dispatch_one_bio` |
| `blk_throtl_dispatch_work_fn` | per-td final issue | `Throtl::dispatch_work_fn` |
| `throtl_start_new_slice` / `_extend_slice` / `_set_slice_end` | per-slice window | `ThrotlGrp::{start_new_slice,extend_slice,set_slice_end}` |
| `throtl_slice_used` | per-slice expired? | `ThrotlGrp::slice_used` |
| `throtl_trim_slice` / `throtl_trim_bps` / `throtl_trim_iops` | per-slice rolling-window decay | `ThrotlGrp::{trim_slice,trim_bps,trim_iops}` |
| `calculate_io_allowed` / `calculate_bytes_allowed` | per-(limit, jiffy) budget | `Throtl::{io_allowed,bytes_allowed}` |
| `__tg_update_carryover` / `tg_update_carryover` | per-limit-change carryover | `ThrotlGrp::{update_carryover,update_carryover_rw}` |
| `tg_set_conf_u64` / `_uint` | legacy `throttle.{read,write}_{bps,iops}_device` write | `Throtl::cgfile_set_conf_{u64,uint}` |
| `tg_set_limit` | v2 `io.max` write | `Throtl::cgfile_set_limit` |
| `tg_print_limit` | v2 `io.max` show | `Throtl::cgfile_print_limit` |
| `tg_print_conf_u64` / `_uint` / `tg_print_rwstat` / `_recursive` | per-rwstat seq_show | `Throtl::cgfile_print_*` |
| `tg_conf_updated` | per-cfg-change subtree update + restart slice | `Throtl::conf_updated` |
| `blk_throtl_init` | per-disk lazy init | `Throtl::init_disk` |
| `blk_throtl_exit` | per-disk teardown | `Throtl::exit_disk` |
| `blk_throtl_cancel_bios` | per-disk del_gendisk drain | `Throtl::cancel_bios` |
| `tg_flush_bios` | per-group set CANCELING + arm timer | `ThrotlGrp::flush_bios` |
| `throtl_shutdown_wq` | per-td cancel_work_sync | `Throtl::shutdown_wq` |
| `kthrotld_workqueue` | per-init WQ_MEM_RECLAIM | shared singleton |
| `DFL_THROTL_SLICE` | constant `HZ/10` | shared const |
| `THROTL_GRP_QUANTUM` (8) / `THROTL_QUANTUM` (32) | round caps | shared const |
| `THROTL_TG_PENDING` / `_WAS_EMPTY` / `_IOPS_WAS_EMPTY` / `_CANCELING` | tg->flags bits | shared bits |
| `BIO_BPS_THROTTLED` / `BIO_TG_BPS_THROTTLED` | bio bits | shared bits |

### compatibility contract

REQ-1: struct throtl_data:
- service_queue: top-level `throtl_service_queue` (no parent_sq).
- queue: back-pointer to owning `request_queue`.
- nr_queued[2]: per-rw count of bios sitting at the *top* service-queue's `qnode`s (ready-to-issue).
- dispatch_work: `work_struct` -> `blk_throtl_dispatch_work_fn` queued on `kthrotld_workqueue`.

REQ-2: struct throtl_grp:
- pd: embedded `struct blkg_policy_data`.
- service_queue: per-group `throtl_service_queue`; `parent_sq` set in `throtl_pd_init`.
- qnode_on_self[2]: per-rw fifo of bios queued *to this group* by its own descendants.
- qnode_on_parent[2]: per-rw fifo identity used when bios are transferred *up* to parent.
- rb_node: rb-tree node in `parent_sq->pending_tree`.
- disptime: jiffies-key for rb-tree (next eligible dispatch time).
- flags: bitset of `THROTL_TG_PENDING`, `THROTL_TG_WAS_EMPTY`, `THROTL_TG_IOPS_WAS_EMPTY`, `THROTL_TG_CANCELING`.
- bps[READ], bps[WRITE]: per-rw bytes-per-second limit; `U64_MAX` = no limit.
- iops[READ], iops[WRITE]: per-rw iops limit; `UINT_MAX` = no limit.
- has_rules_bps[2], has_rules_iops[2]: per-rw boolean — self or any ancestor has a limit; bypass fast-path when false.
- bytes_disp[2], io_disp[2]: per-rw counters within current slice (signed for carryover).
- slice_start[2], slice_end[2]: per-rw slice window in jiffies.
- stat_bytes, stat_ios: per-rw `blkg_rwstat` (used for `throttle.io_service_*` files).
- td: back-pointer to `throtl_data`.

REQ-3: struct throtl_service_queue:
- parent_sq: parent service-queue; NULL for `throtl_data->service_queue`.
- queued[2]: per-rw `list_head` of `throtl_qnode`s (bios ready to issue from *this* sq's children — at top sq these are dispatch-ready).
- nr_queued_bps[2], nr_queued_iops[2]: per-rw counters such that `sq_queued(sq, rw) = nr_queued_bps[rw] + nr_queued_iops[rw]`.
- pending_tree: `rb_root_cached` keyed by child `disptime`.
- nr_pending: count of `THROTL_TG_PENDING` children.
- first_pending_disptime: cached `disptime` of leftmost child (refreshed by `update_min_dispatch_time`).
- pending_timer: `timer_list` armed by `throtl_schedule_pending_timer` (callback = `throtl_pending_timer_fn`).

REQ-4: Hierarchy mode determination:
- per-`throtl_pd_init`:
  - `sq->parent_sq = &td->service_queue` (default flat).
  - if `cgroup_subsys_on_dfl(io_cgrp_subsys) ∧ blkg->parent`: `sq->parent_sq = &blkg_to_tg(blkg->parent)->service_queue` (true hierarchy on v2).
- per-`tg_bps_limit` / `tg_iops_limit`:
  - if `cgroup_subsys_on_dfl ∧ !blkg->parent`: return `U64_MAX` / `UINT_MAX` (root never throttles on v2).
  - else return `tg->bps[rw]` / `tg->iops[rw]`.

REQ-5: Slice (DFL_THROTL_SLICE = HZ/10):
- `throtl_start_new_slice(tg, rw, clear)`:
  - if clear: `tg->bytes_disp[rw] = 0`, `tg->io_disp[rw] = 0`.
  - `tg->slice_start[rw] = jiffies`; `tg->slice_end[rw] = jiffies + DFL_THROTL_SLICE`.
- `throtl_extend_slice(tg, rw, jiffy_end)`:
  - if `tg->slice_end[rw] < jiffy_end`: `tg->slice_end[rw] = roundup(jiffy_end, DFL_THROTL_SLICE)`.
- `throtl_slice_used(tg, rw)`: returns true iff `jiffies` is not in `[slice_start, slice_end]`.
- `throtl_trim_slice(tg, rw)` (called per-dispatch):
  - if slice still active: return.
  - `throtl_set_slice_end(tg, rw, jiffies + DFL_THROTL_SLICE)`.
  - `time_elapsed = rounddown(jiffies - slice_start, DFL_THROTL_SLICE) - DFL_THROTL_SLICE` (one slice preserved for deviation).
  - if `time_elapsed < DFL_THROTL_SLICE * 2`: return (need ≥ 2 slices used).
  - `bytes_trim = throtl_trim_bps(tg, rw, time_elapsed)`; `io_trim = throtl_trim_iops(tg, rw, time_elapsed)`.
  - `tg->slice_start[rw] += time_elapsed`.

REQ-6: Token-bucket budget calculation:
- `calculate_io_allowed(iops_limit, jiffy_elapsed) = min(iops_limit * jiffy_elapsed / HZ, UINT_MAX)`.
- `calculate_bytes_allowed(bps_limit, jiffy_elapsed)`:
  - if `ilog2(bps_limit) + ilog2(jiffy_elapsed) - ilog2(HZ) > 62`: return `U64_MAX` (overflow guard).
  - else return `mul_u64_u64_div_u64(bps_limit, jiffy_elapsed, HZ)`.

REQ-7: tg_within_iops_limit / tg_within_bps_limit:
- per-`tg_within_iops_limit(tg, bio, iops_limit)`:
  - `jiffy_elapsed = jiffies - slice_start[rw]`.
  - `jiffy_elapsed_rnd = roundup(jiffy_elapsed + 1, DFL_THROTL_SLICE)` (wait must be > 0).
  - `io_allowed = calculate_io_allowed(iops_limit, jiffy_elapsed_rnd)`.
  - if `io_disp[rw] + 1 ≤ io_allowed`: return 0 (immediate).
  - else `jiffy_wait = max(jiffy_elapsed_rnd - jiffy_elapsed, HZ/iops_limit + 1)`.
- per-`tg_within_bps_limit(tg, bio, bps_limit)`:
  - `jiffy_elapsed = jiffy_elapsed_rnd = jiffies - slice_start[rw]`.
  - if `!jiffy_elapsed`: `jiffy_elapsed_rnd = DFL_THROTL_SLICE`.
  - `jiffy_elapsed_rnd = roundup(jiffy_elapsed_rnd, DFL_THROTL_SLICE)`.
  - `bytes_allowed = calculate_bytes_allowed(bps_limit, jiffy_elapsed_rnd)`.
  - if `bytes_disp[rw] + bio_size ≤ bytes_allowed` ∨ `bytes_allowed < 0`: return 0.
  - else `extra = bytes_disp[rw] + bio_size - bytes_allowed`; `jiffy_wait = max(1, extra * HZ / bps_limit) + (jiffy_elapsed_rnd - jiffy_elapsed)`.

REQ-8: tg_dispatch_time:
- BUG_ON if `sq_queued(sq, rw) > 0` ∧ bio is not the head (only head consults dispatch_time).
- `wait = tg_dispatch_bps_time(tg, bio)`.
- if `wait != 0`: return wait.
- `throtl_charge_bps_bio(tg, bio)` (bio about to enter iops queue).
- return `tg_dispatch_iops_time(tg, bio)`.

REQ-9: throtl_charge_bps_bio / throtl_charge_iops_bio:
- per-charge_bps:
  - if `!bio_flagged(bio, BIO_BPS_THROTTLED) ∧ !bio_flagged(bio, BIO_TG_BPS_THROTTLED)`:
    - `bio_set_flag(bio, BIO_TG_BPS_THROTTLED)`.
    - `tg->bytes_disp[rw] += bio_size`.
- per-charge_iops:
  - `bio_clear_flag(bio, BIO_TG_BPS_THROTTLED)` (transitioning to iops queue).
  - `tg->io_disp[rw]++`.

REQ-10: __blk_throtl_bio(bio):
- Acquire `rcu_read_lock()` and `spin_lock_irq(&q->queue_lock)`.
- Loop climbing `parent_sq`:
  - if `tg_within_limit(tg, bio, rw)`:
    - `throtl_charge_iops_bio(tg, bio)`; `throtl_trim_slice(tg, rw)`.
  - else if `bio_issue_as_root_blkg(bio)`:
    - `throtl_charge_bps_bio(tg, bio)`; `throtl_charge_iops_bio(tg, bio)` (priority-inversion bypass — accumulates "debt").
  - else: break to queue.
  - climb: `qn = &tg->qnode_on_parent[rw]`; `sq = sq->parent_sq`; `tg = sq_to_tg(sq)`.
  - if !tg: `bio_set_flag(bio, BIO_BPS_THROTTLED)`; return false (passed all levels).
- Queue path:
  - `td->nr_queued[rw]++`; `throtl_add_bio_tg(bio, qn, tg)`; `throttled = true`.
  - if `tg->flags & (THROTL_TG_WAS_EMPTY | THROTL_TG_IOPS_WAS_EMPTY)`:
    - `tg_update_disptime(tg)`; `throtl_schedule_next_dispatch(parent_sq, true)`.
- Release locks; return `throttled`.

REQ-11: tg_within_limit:
- if `bio_flagged(bio, BIO_BPS_THROTTLED)`: return `sq->nr_queued_iops[rw] == 0 ∧ tg_dispatch_iops_time(tg, bio) == 0` (bps already paid).
- if `sq_queued(sq, rw) > 0`: (FIFO — must queue)
  - if `sq->nr_queued_bps[rw] == 0 ∧ tg_dispatch_bps_time(tg, bio) == 0`: `throtl_charge_bps_bio(tg, bio)` (place into iops queue path).
  - return false.
- else: return `tg_dispatch_time(tg, bio) == 0`.

REQ-12: throtl_pending_timer_fn:
- `spin_lock_irq(&q->queue_lock)`.
- if `!q->root_blkg`: unlock & return.
- Loop (label `again`):
  - dispatched = false; parent_sq = sq->parent_sq.
  - inner loop:
    - `ret = throtl_select_dispatch(sq)`.
    - if ret: dispatched = true.
    - if `throtl_schedule_next_dispatch(sq, false)`: break (next timer armed or empty).
    - unlock, cpu_relax(), relock — current window still open.
  - if !dispatched: unlock & return.
  - if parent_sq (we're inner-tier):
    - if `tg->flags & (THROTL_TG_WAS_EMPTY | THROTL_TG_IOPS_WAS_EMPTY)`:
      - `tg_update_disptime(tg)`.
      - if `!throtl_schedule_next_dispatch(parent_sq, false)`: sq = parent_sq; tg = sq_to_tg(sq); goto again (window open at parent).
  - else (top sq): `queue_work(kthrotld_workqueue, &td->dispatch_work)`.

REQ-13: throtl_select_dispatch(parent_sq):
- Loop:
  - if `!parent_sq->nr_pending`: break.
  - `tg = throtl_rb_first(parent_sq)` (leftmost — earliest `disptime`).
  - if `time_before(jiffies, tg->disptime)`: break (not yet).
  - `nr_disp += throtl_dispatch_tg(tg)`.
  - if `sq_queued(&tg->service_queue, READ) ∨ sq_queued(&tg->service_queue, WRITE)`:
    - `tg_update_disptime(tg)` (re-key in rb-tree).
  - else: `throtl_dequeue_tg(tg)` (clears `THROTL_TG_PENDING`).
  - if `nr_disp ≥ THROTL_QUANTUM (32)`: break.

REQ-14: throtl_dispatch_tg(tg) (THROTL_GRP_QUANTUM = 8):
- `max_nr_reads = THROTL_GRP_QUANTUM * 3 / 4 = 6`.
- `max_nr_writes = THROTL_GRP_QUANTUM - 6 = 2` (75%/25% R:W split).
- Drain READ until `nr_reads ≥ 6` or peek has non-zero `tg_dispatch_time`.
- Drain WRITE until `nr_writes ≥ 2` or peek has non-zero `tg_dispatch_time`.
- Each iteration: `tg_dispatch_one_bio(tg, rw)`.

REQ-15: tg_dispatch_one_bio:
- `bio = throtl_pop_queued(sq, &tg_to_put, rw)`.
- `throtl_charge_iops_bio(tg, bio)`.
- if `parent_tg = sq_to_tg(parent_sq)`:
  - `throtl_add_bio_tg(bio, &tg->qnode_on_parent[rw], parent_tg)` (transfer up).
  - `start_parent_slice_with_credit(tg, parent_tg, rw)`.
- else (parent is `throtl_data`):
  - `bio_set_flag(bio, BIO_BPS_THROTTLED)` (no more throttling above).
  - `throtl_qnode_add_bio(bio, &tg->qnode_on_parent[rw], parent_sq)`.
  - `td->nr_queued[rw]--`.
- `throtl_trim_slice(tg, rw)`.
- if tg_to_put: `blkg_put(tg_to_blkg(tg_to_put))`.

REQ-16: blk_throtl_dispatch_work_fn:
- Under `q->queue_lock`: drain `td->service_queue.queued[READ]` and `[WRITE]` into local `bio_list_on_stack` via `throtl_pop_queued(td_sq, NULL, rw)`.
- Unlock; `blk_start_plug(&plug)`; for each bio: `submit_bio_noacct_nocheck(bio, false)`; `blk_finish_plug`.

REQ-17: throtl_schedule_pending_timer:
- `max_expire = jiffies + 8 * DFL_THROTL_SLICE`.
- if `time_after(expires, max_expire)`: `expires = max_expire` (defends against stale long sleep after limit change).
- `mod_timer(&sq->pending_timer, expires)`.

REQ-18: throtl_schedule_next_dispatch(sq, force):
- if `!sq->nr_pending`: return true (nothing to do).
- `update_min_dispatch_time(sq)` (refresh `first_pending_disptime` from rb-tree leftmost).
- if `force ∨ time_after(first_pending_disptime, jiffies)`:
  - `throtl_schedule_pending_timer(sq, first_pending_disptime)`; return true.
- else: return false (caller continues dispatching now).

REQ-19: tg_update_has_rules:
- per-rw: `has_rules_iops[rw] = (parent_tg ∧ parent_tg->has_rules_iops[rw]) ∨ (tg_iops_limit(tg, rw) != UINT_MAX)`.
- per-rw: `has_rules_bps[rw] = (parent_tg ∧ parent_tg->has_rules_bps[rw]) ∨ (tg_bps_limit(tg, rw) != U64_MAX)`.
- Walks one level only (parent already correct).

REQ-20: Carryover (limit-change while bios queued):
- `__tg_update_carryover(tg, rw, *bytes, *ios)`:
  - if `sq_queued(sq, rw) == 0`: zero `bytes_disp` / `io_disp` and return.
  - if `bps_limit != U64_MAX`: `*bytes = calculate_bytes_allowed(bps_limit, jiffy_elapsed) - bytes_disp[rw]`.
  - if `iops_limit != UINT_MAX`: `*ios = calculate_io_allowed(iops_limit, jiffy_elapsed) - io_disp[rw]`.
  - `bytes_disp[rw] = -*bytes`; `io_disp[rw] = -*ios` (negative = credit).
- Invoked by `tg_set_conf` / `tg_set_limit` *before* installing new limits.

REQ-21: tg_conf_updated(tg, global):
- For each blkg in `blkg_for_each_descendant_pre(root_blkg or tg_to_blkg(tg))`:
  - `tg_update_has_rules(this_tg)`.
- `throtl_start_new_slice(tg, READ, false)`; `throtl_start_new_slice(tg, WRITE, false)` (clear=false preserves *_disp carryover).
- if `tg->flags & THROTL_TG_PENDING`: `tg_update_disptime(tg)`; `throtl_schedule_next_dispatch(sq->parent_sq, true)`.

REQ-22: Per-cgroup-v1 file set (`throtl_legacy_files`):
- `throttle.read_bps_device` (u64) → `tg->bps[READ]`; `0` writes as `U64_MAX` (unlimited).
- `throttle.write_bps_device` (u64) → `tg->bps[WRITE]`.
- `throttle.read_iops_device` (uint) → `tg->iops[READ]`.
- `throttle.write_iops_device` (uint) → `tg->iops[WRITE]`.
- `throttle.io_service_bytes` / `_recursive` (rwstat) → `tg->stat_bytes`.
- `throttle.io_serviced` / `_recursive` (rwstat) → `tg->stat_ios`.

REQ-23: Per-cgroup-v2 file set (`throtl_files`):
- `io.max` (CFTYPE_NOT_ON_ROOT):
  - Show: `<major>:<minor> rbps=<v|max> wbps=<v|max> riops=<v|max> wiops=<v|max>` (suppressed line if all at default).
  - Write: tokens parsed via `strsep("=")`; rejects `val == 0` (-ERANGE); iops clamped to `UINT_MAX`.

REQ-24: blk_throtl_init(disk) (lazy):
- Called from `tg_set_conf` / `tg_set_limit` if `!blk_throtl_activated(q)`.
- `kzalloc_node(throtl_data)`; `INIT_WORK(&td->dispatch_work, blk_throtl_dispatch_work_fn)`.
- `throtl_service_queue_init(&td->service_queue)`.
- `blk_mq_freeze_queue` + `blk_mq_quiesce_queue` (atomic policy install).
- `q->td = td`; `td->queue = q`.
- `blkcg_activate_policy(disk, &blkcg_policy_throtl)`.
- on error: `q->td = NULL`; `kfree(td)`.
- `blk_mq_unquiesce_queue`; `blk_mq_unfreeze_queue`.

REQ-25: blk_throtl_exit / blk_throtl_cancel_bios:
- per-`blk_throtl_exit(disk)`:
  - if `!q->td`: return.
  - `timer_delete_sync(&q->td->service_queue.pending_timer)`.
  - `throtl_shutdown_wq(q)` (`cancel_work_sync(&td->dispatch_work)`).
  - `kfree(q->td)`.
- per-`blk_throtl_cancel_bios(disk)` (called from `del_gendisk`):
  - `blkg_for_each_descendant_post(blkg, pos_css, q->root_blkg)`: `tg_flush_bios(blkg_to_tg(blkg))`.

REQ-26: tg_flush_bios:
- if `THROTL_TG_CANCELING`: return (already).
- `tg->flags |= THROTL_TG_CANCELING`.
- if `!(tg->flags & THROTL_TG_PENDING)`: return (no insert needed).
- `tg_update_disptime(tg)`; `throtl_schedule_pending_timer(sq, jiffies + 1)`.

REQ-27: kthrotld_workqueue:
- per-`throtl_init` (module_init): `alloc_workqueue("kthrotld", WQ_MEM_RECLAIM | WQ_PERCPU, 0)`.
- panic on failure (cannot fail late-init).

REQ-28: Locking discipline:
- `q->queue_lock` (spinlock, IRQ-disabled) protects: `throtl_data`, every `throtl_grp`, every `throtl_service_queue`'s rb-tree / pending_timer / qnode lists.
- `rcu_read_lock()` wraps tg_conf_updated subtree walks and `__blk_throtl_bio` outer scope.
- `down_write(&crypto_alg_sem)` — N/A here; throttle has no global sem.
- pending_timer fires in soft-IRQ context and reacquires `q->queue_lock`.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `bytes_allowed_no_overflow` | INVARIANT | per-calculate_bytes_allowed: `ilog2(bps)+ilog2(jiffy)-ilog2(HZ) > 62 ⟹ U64_MAX` short-circuit. |
| `io_allowed_clamped_to_uint_max` | INVARIANT | per-calculate_io_allowed: result ≤ UINT_MAX. |
| `disptime_monotonic_within_slice` | INVARIANT | per-tg_dispatch_time: returned wait ≤ DFL_THROTL_SLICE * (something bounded). |
| `pending_timer_expires_clamped` | INVARIANT | per-schedule_pending_timer: expires ≤ jiffies + 8*DFL_THROTL_SLICE. |
| `quantum_per_round_bounded` | INVARIANT | per-select_dispatch: nr_disp ≤ THROTL_QUANTUM (32). |
| `grp_quantum_rw_split` | INVARIANT | per-dispatch_tg: nr_reads ≤ 6, nr_writes ≤ 2 (per round). |
| `root_blkg_unlimited_on_dfl` | INVARIANT | per-tg_bps_limit / tg_iops_limit: cgroup_subsys_on_dfl ∧ !blkg->parent ⟹ unlimited. |
| `queue_lock_held_in_bio_path` | INVARIANT | per-__blk_throtl_bio: queue_lock held across all tg / sq mutations. |
| `pending_flag_balanced` | INVARIANT | per-enqueue_tg / dequeue_tg: THROTL_TG_PENDING toggled in lockstep with rb-tree insert/erase. |
| `nr_pending_matches_pending_flag_count` | INVARIANT | per-sq: nr_pending == |children with PENDING bit|. |
| `td_nr_queued_balanced` | INVARIANT | per-bio: td->nr_queued[rw] inc on enqueue at any tier, dec exactly when bio reaches top sq. |
| `has_rules_subtree_consistent` | INVARIANT | per-tg_update_has_rules: tg->has_rules ⟹ tg or some ancestor has finite limit. |

### Layer 2: TLA+

`block/blk-throttle.tla`:
- Per-bio-arrival + per-charge + per-queue + per-timer + per-dispatch + per-issue.
- Properties:
  - `safety_no_dispatch_before_disptime` — per-select_dispatch: never pops `tg` while `jiffies < tg.disptime`.
  - `safety_bytes_disp_consistent` — per-charge_bps: increments `bytes_disp` only when `BIO_TG_BPS_THROTTLED` not yet set.
  - `safety_pending_set_iff_in_rb_tree` — per-tg: `THROTL_TG_PENDING ⟺ tg.rb_node in parent_sq.pending_tree`.
  - `safety_v2_root_unlimited` — per-root_blkg: bps[rw] returned to tg_within_*_limit is `U64_MAX` regardless of cgroup write.
  - `liveness_bio_eventually_issued` — per-bio: if not throttled forever (limit eventually permits), bio reaches `submit_bio_noacct_nocheck`.
  - `liveness_cancel_drains` — per-cancel_bios: every queued bio is dispatched within finite jiffies after `THROTL_TG_CANCELING` is set.
  - `liveness_limit_decrease_honored` — per-tg_conf_updated: after decrease, future dispatch rate ≤ new limit (modulo carryover settlement).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `ThrotlGrp::within_limit` post: returns true ⟹ bio can be dispatched without exceeding bps/iops over current slice | `ThrotlGrp::within_limit` |
| `ThrotlGrp::dispatch_time` post: 0 ⟹ within both bps and iops; else returned jiffies > 0 | `ThrotlGrp::dispatch_time` |
| `Throtl::throtl_bio` post: returns false ⟹ BIO_BPS_THROTTLED set ∨ bio reached top sq | `Throtl::throtl_bio` |
| `Throtl::throtl_bio` post: returns true ⟹ bio queued at some tg, td->nr_queued[rw] incremented | `Throtl::throtl_bio` |
| `ThrotlServiceQueue::select_dispatch` post: ret ≤ THROTL_QUANTUM | `ThrotlServiceQueue::select_dispatch` |
| `ThrotlGrp::dispatch_one_round` post: nr_reads ≤ 6 ∧ nr_writes ≤ 2 | `ThrotlGrp::dispatch_one_round` |
| `Throtl::init_disk` post: ret=0 ⟹ q.td≠NULL ∧ policy activated; ret<0 ⟹ q.td=NULL ∧ td freed | `Throtl::init_disk` |
| `Throtl::exit_disk` post: q.td freed; pending_timer deleted synchronously; dispatch_work canceled synchronously | `Throtl::exit_disk` |
| `ThrotlGrp::update_carryover` post: bytes_disp[rw] = -*bytes; io_disp[rw] = -*ios (negative credit) | `ThrotlGrp::update_carryover` |

### Layer 4: Verus/Creusot functional

`Per-bio arrival → tg_within_limit → (charge | queue) → throtl_select_dispatch → tg_dispatch_one_bio (climb tier) → blk_throtl_dispatch_work_fn → submit_bio_noacct_nocheck` semantic equivalence: per-`Documentation/admin-guide/cgroup-v2.rst` (io.max) and `Documentation/admin-guide/cgroup-v1/blkio-controller.rst` (throttle.*_device).

`Per-limit change → tg_update_carryover (rebases *_disp) → tg_conf_updated → tg_update_has_rules subtree → throtl_start_new_slice(clear=false) → schedule_next_dispatch(parent_sq, true)` semantic equivalence: per-cgroup write atomicity.

### hardening

(Inherits row-1 features from `block/00-overview.md` § Hardening.)

Throttle reinforcement:

- **Per-`calculate_bytes_allowed` overflow guard (`ilog2` sum > 62 ⟹ `U64_MAX`)** — defense against per-cgroup writing pathological bps limits inducing 64-bit multiply overflow.
- **Per-`calculate_io_allowed` clamp to `UINT_MAX`** — defense against per-iops overflow.
- **Per-`throtl_schedule_pending_timer` expires clamp to `8 * DFL_THROTL_SLICE`** — defense against per-limit-change stranding bios in arbitrarily-long sleeps.
- **Per-`THROTL_QUANTUM = 32` per-round cap on `throtl_select_dispatch`** — defense against per-pending_timer-runaway dispatching the whole disk.
- **Per-`THROTL_GRP_QUANTUM = 8` (6 R / 2 W) per-group cap** — defense against per-tg-monopolizing-its-tier.
- **Per-`q->queue_lock` IRQ-disabled across all rb-tree / timer / qnode mutations** — defense against per-NMI / soft-IRQ race.
- **Per-`THROTL_TG_PENDING` flag strict pairing with rb-tree membership** — defense against per-double-insert (BUG_ON in `tg_service_queue_add`).
- **Per-`THROTL_TG_CANCELING` once-only set** — defense against per-`tg_flush_bios` double-arm of pending_timer.
- **Per-`bio_issue_as_root_blkg` accumulates debt (charges *_disp but bypasses queue)** — defense against per-priority-inversion deadlock without granting silent free bandwidth.
- **Per-v2 root blkg always unlimited via `tg_bps_limit` / `tg_iops_limit` guard** — defense against per-misconfiguration of root io.max.
- **Per-`blk_mq_freeze_queue` + `blk_mq_quiesce_queue` around `blk_throtl_init`** — defense against per-policy-install racing in-flight bios.
- **Per-`blk_throtl_cancel_bios` from `del_gendisk` (descendant_post order)** — defense against per-disk-removal leaking in-flight throttled bios.
- **Per-`cancel_work_sync(&td->dispatch_work)` in `throtl_shutdown_wq`** — defense against per-workqueue using freed `td` after `blk_throtl_exit`.
- **Per-`timer_delete_sync(&pending_timer)` in `blk_throtl_exit` and `throtl_pd_free`** — defense against per-timer-firing on freed `throtl_grp`.
- **Per-`__tg_update_carryover` rebases `bytes_disp`/`io_disp` before installing new limit** — defense against per-limit-decrease granting free burst from stale credit.
- **Per-`kthrotld_workqueue` with `WQ_MEM_RECLAIM`** — defense against per-memory-pressure stalling dispatch on writeback path.

