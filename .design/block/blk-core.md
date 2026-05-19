# Tier-3: block/blk-core.c — Block layer core (submit / queue lifecycle / poll)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: block/00-overview.md
upstream-paths:
  - block/blk-core.c (~1294 lines)
  - block/bio.c (submit_bio_wait, bio_await)
  - block/blk-flush.c (blkdev_issue_flush)
  - include/linux/blkdev.h
  - include/linux/blk_types.h
  - include/linux/blk-mq.h
-->

## Summary

`block/blk-core.c` is the **block-layer entry point**: it provides the bio submission funnel (`submit_bio` / `submit_bio_noacct`), request-queue allocation and lifecycle (`blk_alloc_queue`, refcount-based teardown via `blk_put_queue`, freeze/drain via `blk_queue_enter` / `blk_queue_exit`), the plugging machinery (`blk_start_plug` / `blk_finish_plug` / `__blk_flush_plug`), polled-completion (`bio_poll`, `iocb_bio_iopoll`), per-bdev I/O accounting (`bdev_start_io_acct` / `bdev_end_io_acct`), error-domain translation (`errno_to_blk_status` / `blk_status_to_errno`), and the kblockd workqueue + debugfs root used by the rest of the block layer. Per-`__submit_bio` dispatch chooses between **blk-mq** (`blk_mq_submit_bio` — used when the bdev does not declare `BD_HAS_SUBMIT_BIO`) and the **bio-only / stacking** path (`disk->fops->submit_bio` — used by dm/md/loop and other QUEUE_FLAG_NO_SCHED stacking drivers). Per-`__submit_bio_noacct` loop manages `current->bio_list` so a stacking driver's recursive `submit_bio_noacct` calls do not blow the kernel stack — submitted bios are sorted into "same queue" and "lower queue" lists and drained iteratively. Per-`bio_check_eod` / `bio_check_ro` / `blk_partition_remap` / `blk_check_zone_append` per-bio validation gates the path; per-`should_fail_bio` provides fault-injection. Per-`blk_throtl_bio` integrates blk-cgroup throttling at submit time. Critical for: every block I/O in the kernel passes through this file, container I/O QoS, zoned-device correctness, queue freeze (for elevator change / disk teardown), polled-NVMe latency.

This Tier-3 covers `block/blk-core.c` (~1294 lines) plus the small wrappers `submit_bio_wait` (in `block/bio.c`) and `blkdev_issue_flush` (in `block/blk-flush.c`) which form the synchronous-submit surface.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `submit_bio()` | per-fs/upper entry: account + ioprio + delegate | `BlkCore::submit_bio` |
| `submit_bio_noacct()` | per-stacking entry: validate + dispatch | `BlkCore::submit_bio_noacct` |
| `submit_bio_noacct_nocheck()` | per-validated dispatch | `BlkCore::submit_bio_noacct_nocheck` |
| `submit_bio_wait()` (bio.c) | per-sync wrap (waits for `bi_end_io`) | `BlkCore::submit_bio_wait` |
| `blkdev_issue_flush()` (blk-flush.c) | per-sync REQ_OP_WRITE|REQ_PREFLUSH | `BlkCore::issue_flush` |
| `__submit_bio()` | per-bdev dispatch (mq vs bio-only) | `BlkCore::submit_bio_inner` |
| `__submit_bio_noacct()` | per-bio-only stacking loop | `BlkCore::submit_bio_noacct_loop` |
| `__submit_bio_noacct_mq()` | per-mq stacking loop | `BlkCore::submit_bio_noacct_loop_mq` |
| `bio_check_eod()` | per-bio end-of-device check | `BlkCore::bio_check_eod` |
| `bio_check_ro()` | per-bio read-only check | `BlkCore::bio_check_ro` |
| `blk_partition_remap()` | per-bio partition remap | `BlkCore::partition_remap` |
| `blk_check_zone_append()` | per-zoned REQ_OP_ZONE_APPEND validate | `BlkCore::check_zone_append` |
| `blk_validate_atomic_write_op_size()` | per-REQ_ATOMIC size check | `BlkCore::validate_atomic_write` |
| `should_fail_bio()` | per-fault-injection gate | `BlkCore::should_fail_bio` |
| `blk_throtl_bio()` | per-blkcg throttle hook | shared (`BlkThrottle`) |
| `bio_set_ioprio()` | per-bio ioprio init | `BlkCore::bio_set_ioprio` |
| `bio_poll()` | per-bio polled completion | `BlkCore::bio_poll` |
| `iocb_bio_iopoll()` | per-kiocb iopoll helper | `BlkCore::iocb_bio_iopoll` |
| `struct request_queue` | per-queue handle | `RequestQueue` |
| `blk_alloc_queue()` | per-queue alloc + init | `RequestQueue::alloc` |
| `blk_put_queue()` | per-queue refcount drop | `RequestQueue::put` |
| `blk_get_queue()` | per-queue refcount get | `RequestQueue::get` |
| `blk_free_queue()` / `blk_free_queue_rcu()` | per-queue destructor | `RequestQueue::free` |
| `blk_queue_enter()` | per-IO mq_freeze counter inc | `RequestQueue::enter` |
| `__bio_queue_enter()` | per-bio mq_freeze counter inc | `RequestQueue::bio_enter` |
| `blk_queue_exit()` | per-IO counter dec | `RequestQueue::exit` |
| `blk_queue_start_drain()` | per-freeze begin | `RequestQueue::start_drain` |
| `blk_sync_queue()` | per-queue timer/work cancel | `RequestQueue::sync` |
| `blk_set_pm_only()` / `blk_clear_pm_only()` | per-runtime-PM gate | `RequestQueue::set_pm_only` |
| `blk_queue_flag_set()` / `blk_queue_flag_clear()` | per-`QUEUE_FLAG_*` atomic | `RequestQueue::flag_set` |
| `blk_lld_busy()` | per-stacked-driver busy probe | `RequestQueue::lld_busy` |
| `blk_start_plug()` / `blk_start_plug_nr_ios()` | per-task plug init | `BlkPlug::start` |
| `blk_finish_plug()` | per-task plug flush + clear | `BlkPlug::finish` |
| `__blk_flush_plug()` | per-plug drain (cb + mq + cached_rqs) | `BlkPlug::flush` |
| `blk_check_plugged()` | per-callback registration | `BlkPlug::check_plugged` |
| `errno_to_blk_status()` / `blk_status_to_errno()` / `blk_status_to_str()` | per-error translate | `BlkStatus::{from_errno, to_errno, as_str}` |
| `blk_op_str()` | per-`REQ_OP_*` name | `ReqOp::as_str` |
| `update_io_ticks()` | per-bdev io_ticks update | `BlkCore::update_io_ticks` |
| `bdev_start_io_acct()` / `bdev_end_io_acct()` | per-bdev acct | `BlkCore::io_acct_start` / `end` |
| `bio_start_io_acct()` / `bio_end_io_acct_remapped()` | per-bio acct wrappers | `BlkCore::bio_acct_start` / `end_remapped` |
| `blk_io_schedule()` | per-task io_schedule w/ hung-task ceiling | `BlkCore::io_schedule` |
| `kblockd_schedule_work()` / `kblockd_mod_delayed_work_on()` | per-kblockd dispatch | `KBlockD::schedule_work` |
| `blk_rq_timed_out_timer()` / `blk_timeout_work()` | per-queue timeout plumbing | `RequestQueue::timeout_*` |
| `blk_dev_init()` | per-boot init (workqueue + cache + debugfs) | `BlkCore::init` |
| `blk_debugfs_root` | per-block debugfs root dentry | shared global |

## Compatibility contract

REQ-1: struct request_queue — fields touched by blk-core.c:
- `queue_flags`: bitmap of `QUEUE_FLAG_*` (DYING, NO_SCHED, INIT_DONE, ...).
- `q_usage_counter`: percpu_ref tracking in-flight I/O (used for freeze).
- `mq_freeze_depth`: counter; non-zero ⟹ queue frozen.
- `mq_freeze_wq`: wait-queue for freeze/drain waiters.
- `mq_freeze_lock`: mutex serializing freeze depth changes.
- `pm_only`: atomic — non-zero ⟹ accept only `BLK_MQ_REQ_PM` requests.
- `refs`: refcount_t — queue lifetime.
- `id`: ida-allocated queue id.
- `stats`: blk_queue_stats pointer.
- `limits`: struct queue_limits (max_sectors, chunk_sectors, features, ...).
- `node`: NUMA node hint.
- `timeout`: timer_list (per-request timeout).
- `timeout_work`: work_struct dispatched from `blk_rq_timed_out_timer`.
- `last_merge`: per-queue merge hint pointer.
- `nr_requests` / `async_depth`: default `BLKDEV_DEFAULT_RQ`.
- `icq_list`: io-context queue list head.
- `queue_lock`, `debugfs_mutex`, `elevator_lock`, `sysfs_lock`, `limits_lock`, `rq_qos_mutex`: per-queue locks.
- `io_lockdep_map` / `q_lockdep_map` (+ `io_lock_cls_key` / `q_lock_cls_key`): lockdep annotations for queue_enter ordering.
- `nr_active_requests_shared_tags`: shared-tags counter.
- `rcu_head`: deferred-free linkage.
- `disk`: gendisk back-pointer (set later in `blk_register_queue`).
- `mq_ops`: blk_mq_ops (NULL ⟹ bio-only stacking driver).

REQ-2: submit_bio(bio):
- /* Accounting */
- if bio_op == REQ_OP_READ: `task_io_account_read(bi_size)`; `count_vm_events(PGPGIN, bio_sectors)`.
- else if bio_op == REQ_OP_WRITE: `count_vm_events(PGPGOUT, bio_sectors)`.
- /* Default ioprio from nice */
- `bio_set_ioprio(bio)`.
- `submit_bio_noacct(bio)`.

REQ-3: submit_bio_noacct(bio):
- `might_sleep()`.
- /* NOWAIT capability gate */
- if `bi_opf & REQ_NOWAIT` ∧ !`bdev_nowait(bdev)`: status = NOTSUPP; goto not_supported.
- /* Inline-encryption gate */
- if `bio_has_crypt_ctx(bio)`:
  - `WARN_ON_ONCE(!bio_has_data(bio))` ⟹ end_io.
  - !`blk_crypto_supported(bio)` ⟹ NOTSUPP.
- if `should_fail_bio(bio)`: end_io with bi_status=IOERR.
- `bio_check_ro(bio)`.
- if !BIO_REMAPPED:
  - `bio_check_eod` ⟹ end_io on overrun.
  - if partition: `blk_partition_remap(bio)` (sets BIO_REMAPPED, increments `bi_iter.bi_sector += bd_start_sect`).
- /* Flush filtering */
- if `op_is_flush(bi_opf)`:
  - require `REQ_OP_WRITE` or `REQ_OP_ZONE_APPEND` (else end_io).
  - if !`bdev_write_cache`: strip REQ_PREFLUSH|REQ_FUA; if no sectors ⟹ end_io OK.
- /* Per-op validation */
- switch bio_op:
  - READ: pass.
  - WRITE: if REQ_ATOMIC: `blk_validate_atomic_write_op_size`.
  - FLUSH: not_supported (FLUSH is request-level only).
  - DISCARD: require `bdev_max_discard_sectors`.
  - SECURE_ERASE: require `bdev_max_secure_erase_sectors`.
  - ZONE_APPEND: `blk_check_zone_append` (zone-start, ≤chunk_sectors, ≤max_zone_append_sectors).
  - WRITE_ZEROES: require `max_write_zeroes_sectors`.
  - ZONE_RESET / OPEN / CLOSE / FINISH / RESET_ALL: require `bdev_is_zoned`.
  - DRV_IN / DRV_OUT / default: not_supported.
- /* Throttle */
- if `blk_throtl_bio(bio)`: throttled; return (bio re-issued by throttle later).
- `submit_bio_noacct_nocheck(bio, false)`.

REQ-4: submit_bio_noacct_nocheck(bio, split):
- `blk_cgroup_bio_start(bio)`.
- if !BIO_TRACE_COMPLETION: `trace_block_bio_queue(bio)`; set BIO_TRACE_COMPLETION.
- /* Stacking-driver recursion guard */
- if `current->bio_list`:
  - if split: `bio_list_add_head(&current->bio_list[0], bio)`.
  - else: `bio_list_add(&current->bio_list[0], bio)`.
- else if !`bdev_test_flag(bdev, BD_HAS_SUBMIT_BIO)`: `__submit_bio_noacct_mq(bio)` (blk-mq direct).
- else: `__submit_bio_noacct(bio)` (bio-only stacking loop).

REQ-5: __submit_bio(bio):
- `blk_start_plug(&plug)` (local nested-plug allowed; outer plug wins).
- if !`bdev_test_flag(bdev, BD_HAS_SUBMIT_BIO)`:
  - `blk_mq_submit_bio(bio)` /* blk-mq path */.
- else if `bio_queue_enter(bio) == 0`:
  - disk = `bdev->bd_disk`.
  - if (REQ_POLLED ∧ !`(disk->queue->limits.features & BLK_FEAT_POLL)`):
    - bi_status = NOTSUPP; `bio_endio(bio)`.
  - else: `disk->fops->submit_bio(bio)` /* bio-only stacking */.
  - `blk_queue_exit(disk->queue)`.
- `blk_finish_plug(&plug)`.

REQ-6: __submit_bio_noacct(bio) — stacking-driver loop:
- `BUG_ON(bio->bi_next)`.
- `bio_list_init(&bio_list_on_stack[0])`.
- `current->bio_list = bio_list_on_stack`.
- do:
  - q = `bdev_get_queue(bio->bi_bdev)`.
  - /* Move pending to slot[1], fresh slot[0] for sub-bios from this submit */
  - `bio_list_on_stack[1] = bio_list_on_stack[0]`.
  - `bio_list_init(&bio_list_on_stack[0])`.
  - `__submit_bio(bio)`.
  - /* Partition sub-bios into same-queue vs lower-queue */
  - sort `bio_list_on_stack[0]` into `same` (q == bdev_get_queue) and `lower`.
  - reassemble: lower first, then same, then prior pending.
- while (bio = `bio_list_pop`).
- `current->bio_list = NULL`.

REQ-7: __submit_bio_noacct_mq(bio) — blk-mq stacking:
- bio_list[2] = {} on stack; `current->bio_list = bio_list`.
- do: `__submit_bio(bio)` while `bio_list_pop(&bio_list[0])`.
- `current->bio_list = NULL`.

REQ-8: submit_bio_wait(bio) (block/bio.c):
- `bio_await(bio, NULL, NULL)`:
  - `DECLARE_COMPLETION_ONSTACK(done)`.
  - bio.bi_private = &done; bio.bi_end_io = `bio_wait_end_io`; bi_opf |= REQ_SYNC.
  - `submit_bio(bio)`.
  - `blk_wait_io(&done)` /* completion wait */.
- return `blk_status_to_errno(bio->bi_status)`.
- /* NOTE: caller retains bio reference; submit_bio_wait does NOT consume it. */

REQ-9: blkdev_issue_flush(bdev) (block/blk-flush.c):
- `struct bio bio; bio_init(&bio, bdev, NULL, 0, REQ_OP_WRITE | REQ_PREFLUSH)`.
- return `submit_bio_wait(&bio)`.

REQ-10: blk_alloc_queue(lim, node_id):
- q = `kmem_cache_alloc_node(blk_requestq_cachep, GFP_KERNEL | __GFP_ZERO, node_id)`; -ENOMEM on failure.
- q.last_merge = NULL.
- q.id = `ida_alloc(&blk_queue_ida, GFP_KERNEL)`; fail_q on error.
- q.stats = `blk_alloc_queue_stats()`; fail_id on -ENOMEM.
- `blk_set_default_limits(lim)`; fail_stats on error.
- q.limits = *lim.
- q.node = node_id.
- `atomic_set(&q->nr_active_requests_shared_tags, 0)`.
- `timer_setup(&q->timeout, blk_rq_timed_out_timer, 0)`.
- `INIT_WORK(&q->timeout_work, blk_timeout_work)` /* no-op default */.
- `INIT_LIST_HEAD(&q->icq_list)`.
- `refcount_set(&q->refs, 1)`.
- init mutexes: debugfs_mutex, elevator_lock, sysfs_lock, limits_lock, rq_qos_mutex; spin_lock_init(queue_lock).
- `init_waitqueue_head(&q->mq_freeze_wq)`; `mutex_init(&q->mq_freeze_lock)`.
- `blkg_init_queue(q)` /* blkcg state */.
- `percpu_ref_init(&q->q_usage_counter, blk_queue_usage_counter_release, PERCPU_REF_INIT_ATOMIC, GFP_KERNEL)`; fail_stats on error.
- `lockdep_register_key(&q->io_lock_cls_key)`; `lockdep_register_key(&q->q_lock_cls_key)`.
- `lockdep_init_map(io_lockdep_map, ..., io_lock_cls_key, 0)`.
- `lockdep_init_map(q_lockdep_map, ..., q_lock_cls_key, 0)`.
- /* Teach lockdep about fs_reclaim ↔ q_usage_counter ordering */
- `fs_reclaim_acquire(GFP_KERNEL)`; `rwsem_acquire_read(io_lockdep_map, ...)`; `rwsem_release(...)`; `fs_reclaim_release(GFP_KERNEL)`.
- q.nr_requests = q.async_depth = `BLKDEV_DEFAULT_RQ`.
- return q.

REQ-11: blk_put_queue(q):
- if `refcount_dec_and_test(&q->refs)`: `blk_free_queue(q)`.

REQ-12: blk_free_queue(q):
- `blk_free_queue_stats(q->stats)`.
- if `queue_is_mq(q)`: `blk_mq_release(q)`.
- `ida_free(&blk_queue_ida, q->id)`.
- `lockdep_unregister_key(&q->io_lock_cls_key)`; `lockdep_unregister_key(&q->q_lock_cls_key)`.
- `call_rcu(&q->rcu_head, blk_free_queue_rcu)`.

REQ-13: blk_free_queue_rcu(rcu_head):
- q = container_of(rcu_head, struct request_queue, rcu_head).
- `percpu_ref_exit(&q->q_usage_counter)`.
- `kmem_cache_free(blk_requestq_cachep, q)`.

REQ-14: blk_get_queue(q):
- if `unlikely(blk_queue_dying(q))`: return false.
- `refcount_inc(&q->refs)`; return true.

REQ-15: blk_queue_start_drain(q):
- freeze = `__blk_freeze_queue_start(q, current)`.
- if `queue_is_mq(q)`: `blk_mq_wake_waiters(q)`.
- `wake_up_all(&q->mq_freeze_wq)` /* re-examine DYING */.
- return freeze.

REQ-16: blk_queue_enter(q, flags):
- pm = !!(flags & BLK_MQ_REQ_PM).
- while !`blk_try_enter_queue(q, pm)`:
  - if flags & BLK_MQ_REQ_NOWAIT: return -EAGAIN.
  - `smp_rmb()` /* pair w/ freeze barrier */.
  - `wait_event(q->mq_freeze_wq, (!q->mq_freeze_depth ∧ blk_pm_resume_queue(pm, q)) ∨ blk_queue_dying(q))`.
  - if `blk_queue_dying(q)`: return -ENODEV.
- /* Lockdep cookie pair */
- `rwsem_acquire_read(&q->q_lockdep_map, 0, 0, _RET_IP_)`; `rwsem_release(&q->q_lockdep_map, _RET_IP_)`.
- return 0.

REQ-17: __bio_queue_enter(q, bio):
- like blk_queue_enter but uses `bio->bi_opf & REQ_NOWAIT`, examines `GD_DEAD` on disk, and on dead/NOWAIT calls `bio_wouldblock_error(bio)` / `bio_io_error(bio)` then returns -EAGAIN / -ENODEV. Uses `io_lockdep_map` not `q_lockdep_map`.

REQ-18: blk_queue_exit(q):
- `percpu_ref_put(&q->q_usage_counter)`.
- /* On reaching zero with PERCPU_REF_DEAD: usage_counter_release wakes freeze waiters */.

REQ-19: blk_sync_queue(q):
- `timer_delete_sync(&q->timeout)`.
- `cancel_work_sync(&q->timeout_work)`.
- /* Does NOT cancel elevator / blkcg work */.

REQ-20: blk_set_pm_only / blk_clear_pm_only:
- set: `atomic_inc(&q->pm_only)`.
- clear: pm_only = `atomic_dec_return`; `WARN_ON_ONCE(pm_only < 0)`; if zero: `wake_up_all(&q->mq_freeze_wq)`.

REQ-21: blk_start_plug(plug):
- `blk_start_plug_nr_ios(plug, 1)`.
- nr_ios: capped to `BLK_MAX_REQUEST_COUNT`.
- if `current->plug` (nested): return without overwriting.
- else: zero plug; `rq_list_init(mq_list); rq_list_init(cached_rqs); INIT_LIST_HEAD(&cb_list); current->plug = plug`.

REQ-22: blk_finish_plug(plug):
- if plug == current->plug: `__blk_flush_plug(plug, false)`; current->plug = NULL.

REQ-23: __blk_flush_plug(plug, from_schedule):
- if cb_list non-empty: `flush_plug_callbacks(plug, from_schedule)`.
- `blk_mq_flush_plug_list(plug, from_schedule)`.
- if !rq_list_empty(cached_rqs): `blk_mq_free_plug_rqs(plug)`.
- plug.cur_ktime = 0; `current->flags &= ~PF_BLOCK_TS`.

REQ-24: blk_check_plugged(unplug, data, size):
- if !current->plug: return NULL.
- search `plug->cb_list` for (callback==unplug ∧ data==data); return if found.
- `BUG_ON(size < sizeof(*cb))`.
- `kzalloc(size, GFP_ATOMIC)`; init `cb->{data, callback}`; `list_add(&cb->list, &plug->cb_list)`.

REQ-25: bio_poll(bio, iob, flags):
- cookie = `READ_ONCE(bio->bi_cookie)`.
- bdev = `READ_ONCE(bio->bi_bdev)`; if !bdev: 0.
- if cookie == BLK_QC_T_NONE: 0.
- `blk_flush_plug(current->plug, false)` /* avoid plug-trapped polled IO */.
- if !`percpu_ref_tryget(&q->q_usage_counter)`: return 0.
- if `queue_is_mq(q)`: `blk_mq_poll(q, cookie, iob, flags)`.
- else if `(q->limits.features & BLK_FEAT_POLL) ∧ disk ∧ disk->fops->poll_bio`: `disk->fops->poll_bio(bio, iob, flags)`.
- `blk_queue_exit(q)`.

REQ-26: iocb_bio_iopoll(kiocb, iob, flags):
- `rcu_read_lock`; bio = `READ_ONCE(kiocb->private)`; if bio: bio_poll; rcu_read_unlock.
- /* SLAB_TYPESAFE_BY_RCU caveats documented inline. */

REQ-27: Error translation (blk_errors[]):
- BLK_STS_OK → 0; NOTSUPP → -EOPNOTSUPP; TIMEOUT → -ETIMEDOUT; NOSPC → -ENOSPC; TRANSPORT → -ENOLINK; TARGET → -EREMOTEIO; RESV_CONFLICT → -EBADE; MEDIUM → -ENODATA; PROTECTION → -EILSEQ; RESOURCE → -ENOMEM; DEV_RESOURCE → -EBUSY; AGAIN → -EAGAIN; OFFLINE → -ENODEV; DM_REQUEUE → -EREMCHG (internal); ZONE_OPEN_RESOURCE → -ETOOMANYREFS; ZONE_ACTIVE_RESOURCE → -EOVERFLOW; DURATION_LIMIT → -ETIME; INVAL → -EINVAL; IOERR → -EIO (catch-all).
- `errno_to_blk_status(errno)`: linear scan; default IOERR.
- `blk_status_to_errno(status)`: indexed; WARN if OOB → -EIO.

REQ-28: REQ_OP names (blk_op_name[]):
- READ, WRITE, FLUSH, DISCARD, SECURE_ERASE, ZONE_RESET, ZONE_RESET_ALL, ZONE_OPEN, ZONE_CLOSE, ZONE_FINISH, ZONE_APPEND, WRITE_ZEROES, DRV_IN, DRV_OUT.
- `blk_op_str(op)`: bounds-check + lookup; "UNKNOWN" fallback.

REQ-29: I/O accounting:
- `update_io_ticks(part, now, end)`: cmpxchg `bd_stamp` forward; if (end ∨ in-flight > 0): `__part_stat_add(part, io_ticks, now - stamp)`. Recurse for partitions to whole-disk.
- `bdev_start_io_acct(bdev, op, start_time)`: part_stat_lock; update_io_ticks(bdev, start_time, false); local_inc `in_flight[op_is_write(op)]`; part_stat_unlock; return start_time.
- `bdev_end_io_acct(bdev, op, sectors, start_time)`: now = jiffies; sgrp = `op_stat_group(op)`; part_stat_lock; update_io_ticks(now, true); inc `ios[sgrp]`; add `sectors[sgrp]`; add `nsecs[sgrp]` (jiffies_to_nsecs(duration)); local_dec `in_flight[op_is_write(op)]`; part_stat_unlock.
- `bio_start_io_acct(bio)`: wraps `bdev_start_io_acct(bio->bi_bdev, bio_op(bio), jiffies)`.
- `bio_end_io_acct_remapped(bio, start_time, orig_bdev)`: wraps `bdev_end_io_acct(orig_bdev, bio_op(bio), bio_sectors(bio), start_time)`.

REQ-30: blk_lld_busy(q):
- if `queue_is_mq(q) ∧ q->mq_ops->busy`: return `q->mq_ops->busy(q)`.
- else: 0.

REQ-31: kblockd workqueue:
- `kblockd_workqueue = alloc_workqueue("kblockd", WQ_MEM_RECLAIM | WQ_HIGHPRI | WQ_PERCPU, 0)` (panic on alloc fail).
- `kblockd_schedule_work(work)`: `queue_work(kblockd_workqueue, work)`.
- `kblockd_mod_delayed_work_on(cpu, dwork, delay)`: `mod_delayed_work_on(cpu, kblockd_workqueue, dwork, delay)`.

REQ-32: blk_io_schedule():
- timeout = `sysctl_hung_task_timeout_secs * HZ / 2`.
- if timeout: `io_schedule_timeout(timeout)`; else: `io_schedule()`.
- /* Prevents hung-task warnings on legitimate long I/O. */

REQ-33: blk_dev_init() — boot path:
- `BUILD_BUG_ON((__force u32)REQ_OP_LAST >= (1 << REQ_OP_BITS))`.
- `BUILD_BUG_ON(REQ_OP_BITS + REQ_FLAG_BITS > 8 * sizeof_field(struct request, cmd_flags))`.
- `BUILD_BUG_ON(REQ_OP_BITS + REQ_FLAG_BITS > 8 * sizeof_field(struct bio, bi_opf))`.
- alloc kblockd workqueue (panic on fail).
- `blk_requestq_cachep = KMEM_CACHE(request_queue, SLAB_PANIC)`.
- `blk_debugfs_root = debugfs_create_dir("block", NULL)`.

REQ-34: should_fail_bio(bio) (CONFIG_FAIL_MAKE_REQUEST):
- if `bdev_test_flag(bdev_whole(bio->bi_bdev), BD_MAKE_IT_FAIL)` ∧ `should_fail(&fail_make_request, bytes)`: -EIO.
- `ALLOW_ERROR_INJECTION(should_fail_bio, ERRNO)`.

REQ-35: bio_check_ro(bio):
- if `op_is_write` ∧ `bdev_read_only(bdev)`:
  - if `op_is_flush ∧ !bio_sectors`: silent (zero-length flush OK on RO).
  - else if `bdev_test_flag(bdev, BD_RO_WARNED)`: silent.
  - else: set BD_RO_WARNED; `pr_warn("Trying to write to read-only block-device %pg\n", bdev)`.

REQ-36: bio_check_eod(bio):
- maxsector = `bdev_nr_sectors(bdev)`; nr = `bio_sectors(bio)`.
- if nr ∧ (nr > maxsector ∨ `bi_iter.bi_sector > maxsector - nr`):
  - if !maxsector: -EIO (silent — fresh device).
  - else: `pr_info_ratelimited(...)`; -EIO.

REQ-37: blk_partition_remap(bio):
- if `should_fail_request(p, bi_size)`: -EIO.
- if `bio_sectors(bio)`: `bi_iter.bi_sector += p->bd_start_sect`; `trace_block_bio_remap`.
- `bio_set_flag(bio, BIO_REMAPPED)`.

REQ-38: blk_check_zone_append(q, bio):
- if !`bdev_is_zoned(bdev)`: NOTSUPP.
- if !`bdev_is_zone_start(bdev, bi_sector)`: IOERR.
- if nr > `q->limits.chunk_sectors`: IOERR.
- if nr > `q->limits.max_zone_append_sectors`: IOERR.
- bi_opf |= REQ_NOMERGE; return OK.

REQ-39: blk_validate_atomic_write_op_size(q, bio):
- if bi_size > `queue_atomic_write_unit_max_bytes(q)`: INVAL.
- if bi_size % `queue_atomic_write_unit_min_bytes(q)`: INVAL.
- OK.

REQ-40: Stacking-driver invariant (current->bio_list):
- Only one `->submit_bio` active per task at a time; recursive submissions are queued on `current->bio_list[0]` and drained iteratively in `__submit_bio_noacct`. `bio_list_on_stack[1]` holds pending pre-existing bios while [0] collects new sub-bios.

REQ-41: QUEUE_FLAG_NO_SCHED ↔ bio-only path:
- bdev with `BD_HAS_SUBMIT_BIO` set ⟹ dispatch through `disk->fops->submit_bio` (dm/md/loop/zram, etc.) — no blk-mq scheduler, queue typically has `QUEUE_FLAG_NO_SCHED` set elsewhere.
- bdev without `BD_HAS_SUBMIT_BIO` ⟹ `blk_mq_submit_bio` direct (full blk-mq pipeline with elevator).

## Acceptance Criteria

- [ ] AC-1: `submit_bio(bio)` with REQ_OP_READ accounts `task_io_account_read` and increments `PGPGIN` vm-event by `bio_sectors`.
- [ ] AC-2: `submit_bio_noacct` rejects REQ_NOWAIT on non-NOWAIT bdev with `bi_status = BLK_STS_NOTSUPP` via `bio_endio`.
- [ ] AC-3: Submitting an EOD-overrunning bio results in `pr_info_ratelimited` and `bio_endio(bio)` with -EIO; zero-sector bio on zero-sector bdev fails silently with -EIO.
- [ ] AC-4: Read-only bdev: write bio triggers one-shot `pr_warn` (BD_RO_WARNED idempotent); zero-length flush passes silently.
- [ ] AC-5: REQ_OP_FLUSH submitted via bio path is rejected (NOTSUPP) — FLUSH is only synthesized inside the flush state machine.
- [ ] AC-6: REQ_OP_ZONE_APPEND on non-zoned bdev → NOTSUPP; on zoned bdev with mid-zone sector → IOERR; with > chunk_sectors → IOERR.
- [ ] AC-7: `submit_bio_wait(bio)` returns 0 on success, `-EIO`/negative on `bi_status` failure; bio reference NOT consumed by wrapper.
- [ ] AC-8: `blkdev_issue_flush(bdev)` builds a `REQ_OP_WRITE|REQ_PREFLUSH` bio with zero sectors and returns the status via `submit_bio_wait`.
- [ ] AC-9: `blk_alloc_queue` returns `refs == 1`, `q_usage_counter` initialized atomic-mode, all mutexes/spinlocks init'd, queue id allocated from `blk_queue_ida`.
- [ ] AC-10: `blk_put_queue` on last reference (`refs == 0`) calls `blk_free_queue` which schedules `blk_free_queue_rcu` for the kmem_cache_free.
- [ ] AC-11: `blk_get_queue` returns false for queues with `QUEUE_FLAG_DYING`.
- [ ] AC-12: `blk_queue_enter` with `BLK_MQ_REQ_NOWAIT` on a frozen queue returns -EAGAIN; without NOWAIT, blocks on `mq_freeze_wq` until unfrozen or DYING.
- [ ] AC-13: `__bio_queue_enter` on a DEAD disk with REQ_NOWAIT bio calls `bio_wouldblock_error` and returns -EAGAIN; without NOWAIT but DEAD, calls `bio_io_error` and returns -ENODEV.
- [ ] AC-14: `blk_finish_plug` flushes plug only when `plug == current->plug` (nested plugs are no-ops).
- [ ] AC-15: `bio_poll` on a queue without `BLK_FEAT_POLL` and `disk->fops->poll_bio == NULL` returns 0 without dispatching.
- [ ] AC-16: `errno_to_blk_status(-EOPNOTSUPP)` returns `BLK_STS_NOTSUPP`; unknown errno returns `BLK_STS_IOERR`.
- [ ] AC-17: `blk_status_to_errno(out-of-range)` WARNs once and returns -EIO.
- [ ] AC-18: Stacking driver recursively calling `submit_bio_noacct` from inside its `submit_bio` queues sub-bios on `current->bio_list[0]` instead of recursing on the stack; drained iteratively.

## Architecture

```
struct RequestQueue {
  queue_flags: AtomicU64,                  // QUEUE_FLAG_*
  q_usage_counter: PercpuRef,              // in-flight counter; freeze gate
  mq_freeze_depth: AtomicI32,
  mq_freeze_wq: WaitQueueHead,
  mq_freeze_lock: Mutex<()>,
  pm_only: AtomicI32,
  refs: RefCount,
  id: i32,                                 // from blk_queue_ida
  stats: Box<BlkQueueStats>,
  limits: QueueLimits,                     // max_sectors, chunk_sectors, features, ...
  node: i32,
  timeout: TimerList,
  timeout_work: WorkStruct,
  last_merge: Option<*Request>,
  nr_requests: u32,                        // BLKDEV_DEFAULT_RQ
  async_depth: u32,
  icq_list: ListHead,
  queue_lock: SpinLock<()>,
  debugfs_mutex: Mutex<()>,
  elevator_lock: Mutex<()>,
  sysfs_lock: Mutex<()>,
  limits_lock: Mutex<()>,
  rq_qos_mutex: Mutex<()>,
  io_lockdep_map: LockdepMap,
  q_lockdep_map: LockdepMap,
  io_lock_cls_key: LockdepKey,
  q_lock_cls_key: LockdepKey,
  nr_active_requests_shared_tags: AtomicI32,
  rcu_head: RcuHead,
  disk: Option<*GenDisk>,
  mq_ops: Option<&'static BlkMqOps>,       // None ⟹ bio-only stacking
}

struct BlkPlug {
  mq_list: RqList,                         // pending mq requests
  cached_rqs: RqList,                      // pre-fetched request cache
  cb_list: ListHead,                       // blk_plug_cb chain
  cur_ktime: u64,
  nr_ios: u16,
  rq_count: u32,
  multiple_queues: bool,
  has_elevator: bool,
}
```

`BlkCore::submit_bio(bio)`:
1. /* VM/task accounting */
2. match bio.bi_opf.op() {
   - REQ_OP_READ: task_io_account_read(bi_size); count_vm_events(PGPGIN, bio_sectors).
   - REQ_OP_WRITE: count_vm_events(PGPGOUT, bio_sectors).
   - _: noop.
   }
3. BlkCore::bio_set_ioprio(bio).
4. BlkCore::submit_bio_noacct(bio).

`BlkCore::submit_bio_noacct(bio)`:
1. might_sleep().
2. bdev = bio.bi_bdev; q = bdev.queue().
3. if bio.bi_opf.has(REQ_NOWAIT) ∧ !bdev.supports_nowait(): goto not_supported.
4. if bio.has_crypt_ctx():
   - if !bio.has_data(): WARN; goto end_io with IOERR.
   - if !blk_crypto_supported(bio): goto not_supported.
5. if BlkCore::should_fail_bio(bio): goto end_io with IOERR.
6. BlkCore::bio_check_ro(bio).
7. if !bio.flag(BIO_REMAPPED):
   - if BlkCore::bio_check_eod(bio).is_err(): goto end_io.
   - if bdev.is_partition() ∧ BlkCore::partition_remap(bio).is_err(): goto end_io.
8. if op_is_flush(bio.bi_opf):
   - require op ∈ {REQ_OP_WRITE, REQ_OP_ZONE_APPEND}.
   - if !bdev.write_cache(): strip {REQ_PREFLUSH, REQ_FUA}; if zero sectors: end_io OK.
9. match bio.op() {
   - READ: pass.
   - WRITE: if REQ_ATOMIC: BlkCore::validate_atomic_write(q, bio).
   - FLUSH: not_supported.
   - DISCARD: require bdev.max_discard_sectors > 0.
   - SECURE_ERASE: require bdev.max_secure_erase_sectors > 0.
   - ZONE_APPEND: BlkCore::check_zone_append(q, bio).
   - WRITE_ZEROES: require q.limits.max_write_zeroes_sectors > 0.
   - ZONE_RESET | ZONE_OPEN | ZONE_CLOSE | ZONE_FINISH | ZONE_RESET_ALL: require bdev.is_zoned().
   - DRV_IN | DRV_OUT | _: not_supported.
   }
10. if BlkThrottle::throttle_bio(bio): return /* re-issued by throttle */.
11. BlkCore::submit_bio_noacct_nocheck(bio, /*split=*/ false).

`BlkCore::submit_bio_noacct_nocheck(bio, split)`:
1. blkcg_bio_start(bio).
2. if !bio.flag(BIO_TRACE_COMPLETION): trace_block_bio_queue(bio); bio.set_flag(BIO_TRACE_COMPLETION).
3. /* Stacking-recursion serialization */
4. if current().bio_list.is_some():
   - if split: current().bio_list[0].push_front(bio).
   - else: current().bio_list[0].push_back(bio).
5. else if !bdev.has_submit_bio(): BlkCore::submit_bio_noacct_loop_mq(bio).
6. else: BlkCore::submit_bio_noacct_loop(bio).

`BlkCore::submit_bio_noacct_loop(bio)`:
1. assert!(bio.bi_next.is_none()).
2. let stack: [BioList; 2] = [BioList::new(), BioList::new()].
3. current().bio_list = Some(&stack).
4. loop {
   - q = bio.bi_bdev.queue().
   - stack[1] = stack[0]; stack[0] = BioList::new().
   - BlkCore::submit_bio_inner(bio).
   - let (lower, same) = stack[0].drain().partition(|b| b.bi_bdev.queue() == q).
   - stack[0].extend(lower).extend(same).extend(stack[1]).
   - bio = stack[0].pop()?; // break if None
   }
5. current().bio_list = None.

`BlkCore::submit_bio_inner(bio)` (the mq-or-bio-only dispatcher):
1. let plug = BlkPlug::start_local(1).
2. if !bdev.has_submit_bio(): BlkMq::submit_bio(bio).
3. else if RequestQueue::bio_enter(q, bio) == Ok():
   - disk = bdev.disk.
   - if bio.flag(REQ_POLLED) ∧ !disk.queue.limits.features.has(BLK_FEAT_POLL):
     - bio.bi_status = BLK_STS_NOTSUPP; bio_endio(bio).
   - else: disk.fops.submit_bio(bio).
   - RequestQueue::exit(disk.queue).
4. BlkPlug::finish(plug).

`BlkCore::submit_bio_wait(bio) -> i32` (from bio.c):
1. let done = Completion::new_on_stack().
2. bio.bi_private = &done; bio.bi_end_io = bio_wait_end_io; bio.bi_opf |= REQ_SYNC.
3. BlkCore::submit_bio(bio).
4. BlkCore::wait_io(&done) /* completion_wait */.
5. return BlkStatus::to_errno(bio.bi_status).

`BlkCore::issue_flush(bdev) -> i32`:
1. let bio = Bio::init_on_stack(bdev, /*vecs=*/ None, /*nr=*/ 0, REQ_OP_WRITE | REQ_PREFLUSH).
2. BlkCore::submit_bio_wait(&bio).

`RequestQueue::alloc(lim, node_id) -> Result<Box<RequestQueue>, i32>`:
1. q = blk_requestq_cachep.alloc_node_zeroed(GFP_KERNEL, node_id)?.
2. q.last_merge = None.
3. q.id = blk_queue_ida.alloc(GFP_KERNEL).map_err(|_| ...)?.
4. q.stats = BlkQueueStats::alloc().ok_or(-ENOMEM)?.
5. blk_set_default_limits(lim)?.
6. q.limits = *lim.
7. q.node = node_id.
8. q.nr_active_requests_shared_tags = 0.
9. q.timeout.setup(BlkCore::timed_out_timer).
10. q.timeout_work.init(BlkCore::timeout_work).
11. q.icq_list.init().
12. q.refs.set(1).
13. init mutexes (debugfs_mutex, elevator_lock, sysfs_lock, limits_lock, rq_qos_mutex); init q.queue_lock.
14. q.mq_freeze_wq.init(); q.mq_freeze_lock.init().
15. BlkCgroup::init_queue(q).
16. q.q_usage_counter.init(RequestQueue::usage_counter_release, PERCPU_REF_INIT_ATOMIC, GFP_KERNEL)?.
17. q.io_lock_cls_key.register(); q.q_lock_cls_key.register().
18. q.io_lockdep_map.init("&q->q_usage_counter(io)", q.io_lock_cls_key).
19. q.q_lockdep_map.init("&q->q_usage_counter(queue)", q.q_lock_cls_key).
20. /* Teach lockdep: fs_reclaim ↔ q_usage_counter */
21. fs_reclaim_acquire(GFP_KERNEL); q.io_lockdep_map.acquire_read(); q.io_lockdep_map.release(); fs_reclaim_release(GFP_KERNEL).
22. q.nr_requests = BLKDEV_DEFAULT_RQ; q.async_depth = BLKDEV_DEFAULT_RQ.
23. return Ok(q).

`RequestQueue::put(q)`:
1. if q.refs.dec_and_test(): RequestQueue::free(q).

`RequestQueue::free(q)`:
1. BlkQueueStats::free(q.stats).
2. if q.is_mq(): BlkMq::release(q).
3. blk_queue_ida.free(q.id).
4. q.io_lock_cls_key.unregister(); q.q_lock_cls_key.unregister().
5. q.rcu_head.call(RequestQueue::free_rcu).

`RequestQueue::free_rcu(rcu_head)`:
1. q = container_of(rcu_head, RequestQueue, rcu_head).
2. q.q_usage_counter.exit().
3. blk_requestq_cachep.free(q).

`RequestQueue::enter(q, flags) -> Result<(), i32>`:
1. pm = flags.has(BLK_MQ_REQ_PM).
2. loop {
   - if RequestQueue::try_enter(q, pm): break.
   - if flags.has(BLK_MQ_REQ_NOWAIT): return Err(-EAGAIN).
   - smp_rmb().
   - wait_event(q.mq_freeze_wq, (q.mq_freeze_depth.load() == 0 ∧ blk_pm_resume_queue(pm, q)) ∨ q.is_dying()).
   - if q.is_dying(): return Err(-ENODEV).
   }
3. q.q_lockdep_map.acquire_read(); q.q_lockdep_map.release().
4. Ok(()).

`RequestQueue::bio_enter(q, bio) -> Result<(), i32>`:
1. loop {
   - if RequestQueue::try_enter(q, false): break.
   - disk = bio.bi_bdev.disk.
   - if bio.bi_opf.has(REQ_NOWAIT):
     - if disk.state.test(GD_DEAD): bio.io_error(); return Err(-ENODEV).
     - bio.wouldblock_error(); return Err(-EAGAIN).
   - smp_rmb().
   - wait_event(q.mq_freeze_wq, (q.mq_freeze_depth.load() == 0 ∧ blk_pm_resume_queue(false, q)) ∨ disk.state.test(GD_DEAD)).
   - if disk.state.test(GD_DEAD): bio.io_error(); return Err(-ENODEV).
   }
2. q.io_lockdep_map.acquire_read(); q.io_lockdep_map.release().
3. Ok(()).

`RequestQueue::exit(q)`:
1. q.q_usage_counter.put().
2. /* On final put: usage_counter_release fires (under PERCPU_REF_DEAD) and wakes q.mq_freeze_wq. */

`RequestQueue::usage_counter_release(ref)`:
1. q = container_of(ref, RequestQueue, q_usage_counter).
2. wake_up_all(&q.mq_freeze_wq).

`RequestQueue::sync(q)`:
1. q.timeout.delete_sync().
2. q.timeout_work.cancel_sync().

`RequestQueue::start_drain(q) -> bool`:
1. freeze = __blk_freeze_queue_start(q, current()).
2. if q.is_mq(): BlkMq::wake_waiters(q).
3. wake_up_all(&q.mq_freeze_wq).
4. return freeze.

`BlkPlug::start(plug)`:
1. BlkPlug::start_nr_ios(plug, 1).

`BlkPlug::start_nr_ios(plug, nr_ios)`:
1. tsk = current().
2. if tsk.plug.is_some(): return /* nested */.
3. plug.cur_ktime = 0; plug.mq_list.init(); plug.cached_rqs.init().
4. plug.nr_ios = min(nr_ios, BLK_MAX_REQUEST_COUNT).
5. plug.rq_count = 0; plug.multiple_queues = false; plug.has_elevator = false.
6. plug.cb_list.init().
7. tsk.plug = Some(plug) /* store after init; preempt implies full barrier */.

`BlkPlug::finish(plug)`:
1. if current().plug == Some(plug):
   - BlkPlug::flush(plug, /*from_schedule=*/ false).
   - current().plug = None.

`BlkPlug::flush(plug, from_schedule)`:
1. if !plug.cb_list.is_empty(): BlkPlug::flush_callbacks(plug, from_schedule).
2. BlkMq::flush_plug_list(plug, from_schedule).
3. if !plug.cached_rqs.is_empty(): BlkMq::free_plug_rqs(plug).
4. plug.cur_ktime = 0; current().flags.clear(PF_BLOCK_TS).

`BlkPlug::check_plugged(unplug, data, size) -> Option<&BlkPlugCb>`:
1. plug = current().plug?.
2. for cb in plug.cb_list.iter():
   - if cb.callback == unplug ∧ cb.data == data: return Some(cb).
3. assert!(size >= sizeof::<BlkPlugCb>()).
4. cb = kzalloc(size, GFP_ATOMIC)?.
5. cb.data = data; cb.callback = unplug.
6. plug.cb_list.push_front(&cb.list).
7. Some(cb).

`BlkCore::bio_poll(bio, iob, flags) -> i32`:
1. cookie = bio.bi_cookie.load().
2. bdev = bio.bi_bdev.load(); if bdev.is_null(): return 0.
3. q = bdev.queue().
4. if cookie == BLK_QC_T_NONE: return 0.
5. BlkPlug::flush(current().plug, /*from_schedule=*/ false).
6. if !q.q_usage_counter.try_get(): return 0.
7. ret = if q.is_mq():
   - BlkMq::poll(q, cookie, iob, flags).
   } else if q.limits.features.has(BLK_FEAT_POLL) ∧ q.disk.is_some() ∧ q.disk.fops.poll_bio.is_some():
   - q.disk.fops.poll_bio(bio, iob, flags).
   } else: 0.
8. RequestQueue::exit(q).
9. return ret.

`BlkStatus::from_errno(errno) -> BlkStatus`:
1. for (i, entry) in BLK_ERRORS.iter().enumerate():
   - if entry.errno == errno: return BlkStatus(i).
2. return BlkStatus::IOERR.

`BlkStatus::to_errno(status) -> i32`:
1. idx = status.0 as usize.
2. if idx >= BLK_ERRORS.len(): warn_once(); return -EIO.
3. return BLK_ERRORS[idx].errno.

`BlkCore::init()`:
1. const_assert!((REQ_OP_LAST as u32) < (1 << REQ_OP_BITS)).
2. const_assert!(REQ_OP_BITS + REQ_FLAG_BITS <= 8 * size_of::<cmd_flags>()).
3. const_assert!(REQ_OP_BITS + REQ_FLAG_BITS <= 8 * size_of::<bi_opf>()).
4. KBLOCKD_WORKQUEUE = WorkQueue::alloc("kblockd", WQ_MEM_RECLAIM | WQ_HIGHPRI | WQ_PERCPU, 0).expect_or_panic().
5. BLK_REQUESTQ_CACHEP = KmemCache::create("request_queue", SLAB_PANIC).
6. BLK_DEBUGFS_ROOT = debugfs::create_dir("block", None).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `submit_bio_noacct_validates_op` | INVARIANT | per-submit_bio_noacct: rejects FLUSH-via-bio, DISCARD without max_discard_sectors, etc. with NOTSUPP. |
| `bio_check_eod_rejects_overrun` | INVARIANT | per-bio_check_eod: nr > maxsector ∨ start > maxsector - nr ⟹ -EIO. |
| `partition_remap_bounded` | INVARIANT | per-blk_partition_remap: bi_sector += bd_start_sect; sets BIO_REMAPPED. |
| `bio_list_recursion_guard` | INVARIANT | per-submit_bio_noacct_nocheck: current->bio_list non-NULL ⟹ enqueue, no recursion. |
| `queue_enter_dying_returns_enodev` | INVARIANT | per-blk_queue_enter: queue DYING ⟹ -ENODEV without freeze-wait deadlock. |
| `queue_enter_nowait_returns_eagain` | INVARIANT | per-blk_queue_enter: frozen + NOWAIT ⟹ -EAGAIN; no wait. |
| `bio_queue_enter_dead_disk_errors_bio` | INVARIANT | per-__bio_queue_enter: GD_DEAD ⟹ bio_io_error or bio_wouldblock_error. |
| `refcount_get_rejects_dying` | INVARIANT | per-blk_get_queue: DYING ⟹ false; refs unchanged. |
| `free_via_rcu` | INVARIANT | per-blk_free_queue: kmem_cache_free deferred via call_rcu. |
| `plug_nested_no_overwrite` | INVARIANT | per-blk_start_plug_nr_ios: tsk.plug != NULL ⟹ no overwrite. |
| `plug_finish_only_own` | INVARIANT | per-blk_finish_plug: plug != current->plug ⟹ no-op. |
| `bio_poll_drops_qref_on_all_paths` | INVARIANT | per-bio_poll: percpu_ref_tryget ↔ blk_queue_exit balanced. |
| `errno_status_roundtrip` | INVARIANT | per-blk_errors[]: errno_to_blk_status ∘ blk_status_to_errno = id (on known entries). |

### Layer 2: TLA+

`block/blk-core.tla`:
- States: Bio { OWNED_BY_FS, ENQUEUED_ON_STACK, IN_FLIGHT_MQ, IN_FLIGHT_BIO_ONLY, COMPLETED_OK, COMPLETED_ERR }.
- States: Queue { LIVE, FREEZING, FROZEN, DYING, DRAINED }.
- States: PlugSlot { EMPTY, FLUSHING }.
- Actions: submit_bio_noacct, throttle_defer, stacking_recurse, queue_enter, queue_exit, freeze_start, freeze_end, plug_flush, bio_endio_complete.
- Properties:
  - `safety_no_stack_recursion` — per-stacking-driver submit: current->bio_list cap + iterative drain.
  - `safety_queue_enter_dying_returns_enodev` — per-blk_queue_enter: post(DYING) ⟹ -ENODEV.
  - `safety_freeze_quiesces_in_flight` — per-blk_freeze_queue_start: eventually q_usage_counter == 0.
  - `safety_nowait_never_blocks` — per-REQ_NOWAIT: enter returns immediately.
  - `liveness_unfreeze_wakes_waiters` — per-unfreeze: all wait_event waiters resume.
  - `liveness_bio_eventually_endio` — per-submit_bio: bi_end_io eventually invoked.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `submit_bio_noacct` post: validation rejects ⟹ bio_endio called exactly once | `BlkCore::submit_bio_noacct` |
| `submit_bio_wait` post: returns BLK_STS_OK ⟹ 0; else negative errno | `BlkCore::submit_bio_wait` |
| `blk_alloc_queue` post: q.refs == 1 ∧ percpu_ref initialized ∧ q.id ≥ 0 | `RequestQueue::alloc` |
| `blk_put_queue` post: refs.dec_and_test ⟹ free_queue called (RCU-deferred) | `RequestQueue::put` |
| `blk_queue_enter` post: returns 0 ⟹ q_usage_counter incremented exactly once | `RequestQueue::enter` |
| `blk_queue_exit` post: q_usage_counter decremented exactly once | `RequestQueue::exit` |
| `__blk_flush_plug` post: cached_rqs drained; mq_list dispatched | `BlkPlug::flush` |
| `bio_poll` post: return ≥ 0; q_usage_counter balanced | `BlkCore::bio_poll` |
| `errno_to_blk_status` post: returns a valid index ∈ [0, BLK_STS_*_LAST] | `BlkStatus::from_errno` |
| `__submit_bio_noacct` post: current->bio_list == NULL on return | `BlkCore::submit_bio_noacct_loop` |

### Layer 4: Verus/Creusot functional

`Per-submit_bio → submit_bio_noacct → validate (NOWAIT, crypt, fault-inject, RO, EOD, partition-remap, flush-filter, per-op) → blk_throtl_bio (or not) → submit_bio_noacct_nocheck → dispatch (mq vs bio-only stacking) → blk_mq_submit_bio | disk->fops->submit_bio → eventual bio_endio` semantic equivalence: per-Documentation/block/blk-mq.rst, Documentation/block/queue-sysfs.rst.

`Per-blk_alloc_queue → init counters/locks → percpu_ref_init(ATOMIC) → lockdep keys → BLKDEV_DEFAULT_RQ → return` semantic equivalence to upstream init order.

`Per-blk_queue_enter ↔ blk_queue_exit` refcount discipline (matched pair) holds across all callers in blk-core.

## Hardening

(Inherits row-1 features from `block/00-overview.md` § Hardening.)

blk-core reinforcement:

- **Per-`current->bio_list` recursion guard** — defense against per-stacking-driver kernel-stack overflow (dm-on-md-on-loop).
- **Per-`bio_check_eod` ratelimited error** — defense against per-attacker EOD probing log flood.
- **Per-`BD_RO_WARNED` one-shot** — defense against per-RO-write log flood.
- **Per-fault-injection scoped to `BD_MAKE_IT_FAIL` bdev** — defense against per-fault-injection production leakage.
- **Per-`REQ_NOWAIT` strict gate** — defense against per-io_uring nowait blocking regression.
- **Per-`blk_crypto_supported` validation** — defense against per-inline-encryption silent fallback.
- **Per-`blk_check_zone_append` (zone-start, chunk_sectors, max_zone_append_sectors)** — defense against per-zoned silent corruption.
- **Per-`blk_validate_atomic_write_op_size`** — defense against per-REQ_ATOMIC split.
- **Per-`bio_queue_enter` GD_DEAD short-circuit** — defense against per-disk-teardown UAF.
- **Per-`blk_get_queue` DYING reject** — defense against per-queue-teardown ref-leak.
- **Per-queue free via `call_rcu` + `percpu_ref_exit` deferred** — defense against per-queue UAF in concurrent readers.
- **Per-`blk_queue_enter` lockdep cookie pair (`io_lockdep_map` / `q_lockdep_map`)** — defense against per-freeze ↔ fs_reclaim deadlock.
- **Per-`blk_start_plug` nested-plug no-overwrite** — defense against per-inner-callee plug-clobber on outer.

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — `/sys/block/<dev>/queue/*` and `/sys/class/block/<dev>/*` reads bounded by `kobject` attr framework; no raw queue-pointer leak.
- **PAX_KERNEXEC** — `request_queue` text symbols (`blk_mq_submit_bio`, `blk_finish_plug`, `submit_bio_noacct`) RX-only; queue-ops vtables RAP-signed in module text.
- **PAX_RANDKSTACK** — submitting syscalls (`io_uring`, `pread`, `pwrite`, `aio`) re-randomize kernel-stack offset before entering `submit_bio`.
- **PAX_REFCOUNT** — `request_queue.usage_counter` (percpu_ref) and `disk->open_partitions` saturating; teardown ordering enforced.
- **PAX_MEMORY_SANITIZE** — freed `request_queue` objects (post `call_rcu` + `percpu_ref_exit`) zeroed before reuse so concurrent reader cannot resurrect.
- **PAX_UDEREF** — direct path does not touch user pointers; `BLKDISCARD` / `BLKZEROOUT` ioctl front-ends enforce SMAP/UDEREF in `block/ioctl.c`.
- **PAX_RAP / kCFI** — `request_queue.ops`, `make_request_fn` (legacy), `mq_ops.queue_rq`, `mq_ops.complete` all RAP-signed.
- **PAX_REFCOUNT_FULL** — `bio_list` recursion guard cannot be bypassed by stacking-driver replay attacks.
- **GRKERNSEC_KMEM** — `/dev/mem`, `/dev/kmem`, `/dev/port` accesses to PCI BARs and IOPORTs that bypass the block layer require lockdown=confidentiality.
- **GRKERNSEC_HIDESYM** — `/proc/diskstats`, `/proc/partitions`, `/sys/block/*` redact queue kaddrs; only CAP_SYSLOG sees raw pointers.
- **GRKERNSEC_DMESG** — queue freeze / dying transitions logged behind `dmesg_restrict`.
- **GRKERNSEC_IO** — raw `BLKRRPART`, `BLKFLSBUF`, `BLKDISCARD`, `BLKSECDISCARD` ioctls gated by CAP_SYS_ADMIN + gr-rbac role.
- **GRKERNSEC_CHROOT** — block-device ioctls inside a chroot rejected unless gr-rbac role explicitly permits.

Per-doc rationale: blk-core owns the queue lifecycle, request submission entry point, and the disk-state RCU teardown; KMEM lockdown + RAP on queue-ops + REFCOUNT-saturation on `usage_counter` + SANITIZE on freed queues close the disk-teardown UAF, queue-freeze race, and queue-ops-redirection attack classes that have shipped against upstream block-core.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `block/blk-mq.c` request-allocator and dispatch (covered in `blk-mq.md` Tier-3)
- `block/blk-flush.c` flush state machine (covered separately if expanded; `blkdev_issue_flush` summarized here)
- `block/blk-throttle.c` throttling internals (covered in `blk-throttle.md` Tier-3)
- `block/blk-mq-sched.c` elevator dispatch (covered in `io-schedulers.md` Tier-3)
- `block/bio.c` bio allocator / iov_iter import (covered in `bio.md` Tier-3; only `submit_bio_wait` summarized here)
- `block/blk-cgroup.c` blkcg policy (covered separately if expanded)
- `block/blk-zoned.c` zoned-device state machine (covered separately if expanded)
- `block/blk-crypto*.c` inline crypto (covered separately if expanded)
- Implementation code
