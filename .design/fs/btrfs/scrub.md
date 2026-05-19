# Tier-3: fs/btrfs/scrub.c — Btrfs online scrub + auto-repair

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: fs/btrfs/00-overview.md
upstream-paths:
  - fs/btrfs/scrub.c (~3330 lines)
  - fs/btrfs/scrub.h
  - fs/btrfs/raid56.c (scrub_rbio path used by RAID56 parity scrub)
  - include/uapi/linux/btrfs.h (struct btrfs_scrub_progress, BTRFS_IOC_SCRUB_*)
-->

## Summary

Btrfs **scrub** is the online consistency checker: it walks every chunk on a chosen device, reads every extent, verifies each sector against its stored checksum (CRC32C / XXHASH / SHA256 / BLAKE2), verifies every tree-block header (bytenr / fsid / chunk-tree-uuid / csum / generation), and **auto-repairs** any sector that fails verification by sourcing a good copy from another mirror (DUP, RAID1, RAID1C3/4, RAID10) or from RAID5/6 parity reconstruction. The entry point `btrfs_scrub_dev()` builds a per-device `struct scrub_ctx` (with 128 inline `struct scrub_stripe` slots in `SCRUB_GROUPS_PER_SCTX (16) * SCRUB_STRIPES_PER_GROUP (8)`), drains scrub-pause requests from the transaction commit path, scrubs the device's super blocks once via `scrub_supers()`, then iterates chunks via `scrub_enumerate_chunks()`. For non-RAID56 profiles `scrub_simple_mirror()` enqueues each extent as a `scrub_stripe`, the `scrub_workers` workqueue runs `scrub_stripe_read_repair_worker` which verifies via packed sub-bitmaps (has_extent / is_metadata / error / io_error / csum_error / meta_error / meta_gen_error) and, on mismatch, walks every mirror via `calc_next_mirror`, writing back the recovered data via `scrub_write_sectors`. RAID56 takes a different path: `scrub_raid56_parity_stripe()` reads all data stripes via the regular `scrub_stripe` engine, verifies they are clean, then hands a `BTRFS_RBIO_PARITY_SCRUB` rbio to `raid56_parity_submit_scrub_rbio()` to regenerate and compare P/Q (covered in sibling `raid56.md`). `fs_info->scrub_lock` (mutex) plus `fs_info->scrub_workers` (workqueue, ref-counted) plus the `scrub_pause_*` / `scrubs_running` / `scrub_cancel_req` atomics coordinate pause/resume/cancel. Statistics are accumulated in `sctx->stat` (`struct btrfs_scrub_progress`) under `stat_lock`.

This Tier-3 covers `fs/btrfs/scrub.c` (~3330 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct scrub_ctx` | per-scrub context (128 stripes + path + stats) | `ScrubCtx` |
| `struct scrub_stripe` | per-64KiB stripe state (folios, bitmaps, sectors) | `ScrubStripe` |
| `struct scrub_sector_verification` | per-sector csum/generation | `ScrubSectorVerification` |
| `enum scrub_stripe_flags` | INITIALIZED / REPAIR_DONE / NO_REPORT | `ScrubStripeFlag` |
| `struct scrub_warning` | per-error path-resolution context | `ScrubWarning` |
| `struct scrub_error_records` | per-stripe snapshot for reporting | `ScrubErrorRecords` |
| `scrub_setup_ctx()` | per-scrub ctx ctor (kvzalloc, 128 stripes) | `ScrubCtx::setup` |
| `scrub_free_ctx()` / `scrub_put_ctx()` | per-scrub teardown | `ScrubCtx::drop` |
| `init_scrub_stripe()` / `release_scrub_stripe()` | per-stripe init/release | `ScrubStripe::init` / `drop` |
| `scrub_reset_stripe()` | per-stripe reset before reuse | `ScrubStripe::reset` |
| `scrub_bitmap_set_##name` / `_clear_##name` / `_test_bit_##name` / `_read_##name` / `_weight_##name` | packed sub-bitmap ops | `ScrubStripe::bitmap_*` |
| `btrfs_scrub_dev()` | per-device scrub entry (UAPI) | `Scrub::scrub_dev` |
| `btrfs_scrub_cancel()` | per-fs cancel (all running) | `Scrub::cancel` |
| `btrfs_scrub_cancel_dev()` | per-device cancel | `Scrub::cancel_dev` |
| `btrfs_scrub_pause()` / `_continue()` | per-fs pause/resume (used by transaction commit) | `Scrub::pause` / `continue_` |
| `btrfs_scrub_progress()` | per-device progress query | `Scrub::progress` |
| `scrub_workers_get()` / `_put()` | per-fs workqueue refcount | `Scrub::workers_get` / `put` |
| `scrub_enumerate_chunks()` | per-device chunk-by-chunk loop | `Scrub::enumerate_chunks` |
| `scrub_chunk()` | per-chunk dispatch (lookup chunk_map, call scrub_stripe) | `Scrub::scrub_chunk` |
| `scrub_stripe()` | per-stripe profile dispatch | `Scrub::scrub_stripe` |
| `scrub_simple_mirror()` | SINGLE/DUP/RAID1/RAID1Cx (+ RAID0/10/56 inner) | `Scrub::simple_mirror` |
| `scrub_simple_stripe()` | RAID0/RAID10 outer loop | `Scrub::simple_stripe` |
| `scrub_raid56_parity_stripe()` | RAID56 per-full-stripe parity verify | `Scrub::raid56_parity_stripe` |
| `scrub_raid56_cached_parity()` | RAID56: hand cached data + dbitmap to raid56_parity_submit_scrub_rbio | `Scrub::raid56_cached_parity` |
| `raid56_scrub_wait_endio()` | per-completion callback for RAID56 scrub | `Scrub::raid56_endio` |
| `queue_scrub_stripe()` | per-extent enqueue, submit on group full | `Scrub::queue_stripe` |
| `submit_initial_group_read()` | per-group submit (8 stripes) | `Scrub::submit_group` |
| `scrub_submit_initial_read()` | per-stripe submit | `Scrub::submit_stripe` |
| `scrub_submit_extent_sector_read()` | per-stripe sector-by-sector read (fallback) | `Scrub::submit_sector_read` |
| `flush_scrub_stripes()` | per-ctx flush (drain remaining < group) | `Scrub::flush_stripes` |
| `scrub_find_fill_first_stripe()` | per-stripe extent search + fill sectors[] | `Scrub::find_fill_first_stripe` |
| `find_first_extent_item()` | per-extent-tree lookup | `Scrub::find_first_extent_item` |
| `fill_one_extent_info()` | per-extent populate has_extent/is_metadata/csum/generation | `Scrub::fill_extent_info` |
| `scrub_read_endio()` | per-bio read endio (queues repair work) | `Scrub::read_endio` |
| `scrub_write_endio()` | per-bio write endio (tracks write_error_bitmap) | `Scrub::write_endio` |
| `scrub_repair_read_endio()` | per-mirror repair endio | `Scrub::repair_endio` |
| `scrub_stripe_read_repair_worker()` | per-stripe verify + cross-mirror repair worker | `Scrub::stripe_repair_worker` |
| `scrub_stripe_submit_repair_read()` | per-mirror repair read | `Scrub::stripe_submit_repair_read` |
| `scrub_verify_one_stripe()` / `_sector()` / `_metadata()` | per-stripe verification | `Scrub::verify_stripe` / `verify_sector` / `verify_metadata` |
| `scrub_write_sectors()` | per-stripe writeback after repair | `Scrub::write_sectors` |
| `scrub_submit_write_bio()` | per-bio write submit (+zoned write-pointer mgmt) | `Scrub::submit_write_bio` |
| `scrub_throttle_dev_io()` | per-device bandwidth limit (scrub_speed_max) | `Scrub::throttle_io` |
| `scrub_stripe_report_errors()` | per-stripe error report + stat accumulation | `Scrub::report_errors` |
| `scrub_print_warning_inode()` / `_common_warning()` | per-error path resolution | `Scrub::print_warning_*` |
| `scrub_supers()` / `scrub_one_super()` | per-device super block verify | `Scrub::supers` / `one_super` |
| `should_cancel_scrub()` | per-loop cancel/freeze/signal check | `Scrub::should_cancel` |
| `scrub_pause_on()` / `_off()` / `__scrub_blocked_if_needed()` / `scrub_blocked_if_needed()` | per-pause synchronization | `Scrub::pause_*` |
| `calc_next_mirror()` | per-mirror cycle | `Scrub::calc_next_mirror` |
| `calc_sector_number()` | per-bio first-bvec → sector_nr | `Scrub::calc_sector_number` |
| `get_raid56_logic_offset()` | per-RAID56 phys → logical (+stripe_start) | `Scrub::get_raid56_logic_offset` |
| `fill_writer_pointer_gap()` | per-zoned writer-pointer gap fill | `Scrub::fill_writer_pointer_gap` |
| `sync_write_pointer_for_zoned()` | per-zoned final writer-pointer sync | `Scrub::sync_write_pointer_for_zoned` |
| `finish_extent_writes_for_zoned()` | per-zoned flush before chunk end | `Scrub::finish_extent_writes_for_zoned` |
| `fs_info->scrub_lock` | per-fs scrub mutex | `FsInfo::scrub_lock` |
| `fs_info->scrub_workers` | per-fs scrub workqueue | `FsInfo::scrub_workers` |
| `fs_info->scrub_workers_refcnt` | per-fs scrub workqueue refcount | `FsInfo::scrub_workers_refcnt` |
| `fs_info->scrub_pause_req` / `scrubs_paused` / `scrubs_running` / `scrub_pause_wait` / `scrub_cancel_req` | per-fs pause/cancel coordination | shared atomic fields |
| `struct btrfs_scrub_progress` | UAPI per-device statistics | `BtrfsScrubProgress` |
| `SCRUB_STRIPES_PER_GROUP` (8) / `SCRUB_GROUPS_PER_SCTX` (16) / `SCRUB_TOTAL_STRIPES` (128) | per-ctx sizing | constants |
| `SCRUB_MAX_SECTORS_PER_BLOCK` | per-block max sectors | constant |

## Compatibility contract

REQ-1: struct scrub_ctx:
- stripes[SCRUB_TOTAL_STRIPES]: per-stripe slots reused round-robin.
- raid56_data_stripes: per-RAID56 extra data-stripe scratch (nr_data_stripes(map)).
- fs_info: per-fs back-pointer.
- extent_path, csum_path: per-walk btrfs_path (search_commit_root + skip_locking).
- first_free, cur_stripe: per-ctx slot bookkeeping.
- cancel_req: per-ctx cancel atomic.
- readonly: per-scrub mode flag.
- throttle_deadline, throttle_sent: per-bandwidth-limit window.
- is_dev_replace: per-replace mode.
- write_pointer: per-zoned writer pointer.
- wr_lock, wr_tgtdev: per-replace target.
- stat (struct btrfs_scrub_progress), stat_lock: per-stats spinlock.
- refs: per-ctx refcount.

REQ-2: struct scrub_stripe:
- sctx, bg: per-stripe back-pointers.
- folios[SCRUB_STRIPE_MAX_FOLIOS]: per-stripe folio array (BTRFS_STRIPE_LEN / PAGE_SIZE max).
- sectors: per-sector verification data (csum or generation).
- dev: per-stripe device.
- logical, physical: per-stripe addresses.
- mirror_num: per-stripe mirror used for initial read.
- nr_sectors: BTRFS_STRIPE_LEN / sectorsize.
- nr_data_extents, nr_meta_extents: per-stripe reporting counters.
- pending_io: per-stripe in-flight bio atomic.
- io_wait, repair_wait: per-stripe wait queues.
- state: SCRUB_STRIPE_FLAG_INITIALIZED / _REPAIR_DONE / _NO_REPORT bitmask.
- bitmaps[]: packed per-sector sub-bitmaps (has_extent, is_metadata, error, io_error, csum_error, meta_error, meta_gen_error).
- write_error_bitmap, write_error_lock: per-stripe writeback failure tracking.
- csums: per-stripe data csum buffer.
- work: per-stripe work_struct.

REQ-3: btrfs_scrub_dev(fs_info, devid, start, end, progress, readonly, is_dev_replace):
- if btrfs_fs_closing(fs_info): return -EAGAIN.
- ASSERT nodesize ≤ BTRFS_STRIPE_LEN ∧ nodesize ≤ SCRUB_MAX_SECTORS_PER_BLOCK << sectorsize_bits.
- sctx = scrub_setup_ctx(fs_info, is_dev_replace); if IS_ERR(sctx): return PTR_ERR(sctx).
- sctx.stat.last_physical = start.
- scrub_workers_get(fs_info) — ref-count up the per-fs workqueue, allocate if first.
- Lock device_list_mutex; lookup device by devid; reject if missing (unless dev_replace) or read-only target.
- Lock scrub_lock; reject IN_FS_METADATA cleared or REPLACE_TGT device; reject if dev.scrub_ctx already set, or another dev_replace ongoing (unless we are it).
- sctx.readonly = readonly; dev.scrub_ctx = sctx.
- __scrub_blocked_if_needed(fs_info) — wait on scrub_pause_wait if scrub_pause_req > 0.
- atomic_inc(scrubs_running); unlock scrub_lock.
- memalloc_nofs_save (avoid reclaim-deadlock vs transaction commit).
- if !is_dev_replace: scrub_supers(sctx, dev) — verify the 3 super-block copies.
- if !ret: scrub_enumerate_chunks(sctx, dev, start, end).
- memalloc_nofs_restore.
- atomic_dec(scrubs_running); wake_up(scrub_pause_wait).
- if progress: memcpy progress from sctx.stat.
- dev.scrub_ctx = NULL.
- scrub_workers_put(fs_info); scrub_put_ctx(sctx).
- If super_errors discovered: force a transaction commit to persist counters.

REQ-4: scrub_setup_ctx(fs_info, is_dev_replace):
- kvzalloc the sctx (with inlined 128 stripes).
- refcount_set(refs, 1); is_dev_replace = arg; fs_info = arg.
- extent_path/csum_path: search_commit_root = true, skip_locking = true.
- For i in 0..SCRUB_TOTAL_STRIPES: init_scrub_stripe (alloc folios + sectors + csums); stripes[i].sctx = sctx.
- first_free = 0; atomic_set(cancel_req, 0); spin_lock_init(stat_lock); throttle_deadline = 0; mutex_init(wr_lock).
- if is_dev_replace: sctx.wr_tgtdev = fs_info.dev_replace.tgtdev.

REQ-5: scrub_workers_get(fs_info):
- if refcount_inc_not_zero(scrub_workers_refcnt): return 0.
- alloc_workqueue("btrfs-scrub", WQ_FREEZABLE | WQ_UNBOUND, thread_pool_size).
- Lock scrub_lock; if refcount still 0: set scrub_workers, refcount_set(1).
- Else: refcount_inc and destroy our raced workqueue.

REQ-6: scrub_enumerate_chunks(sctx, dev, start, end):
- Loop over device's dev-extent items in [start, end), call scrub_chunk(sctx, bg, dev, dev_offset, dev_extent_len).
- Per chunk: lookup btrfs_chunk_map, enter scrub_stripe for each device-side stripe_index belonging to this dev.

REQ-7: scrub_stripe(sctx, bg, map, scrub_dev, stripe_index):
- profile = map.type & BTRFS_BLOCK_GROUP_PROFILE_MASK.
- ASSERT sctx.extent_path.nodes[0] == NULL.
- scrub_blocked_if_needed(fs_info).
- if is_dev_replace ∧ zoned-sequential target: mutex_lock(wr_lock); sctx.write_pointer = physical; unlock.
- if profile & RAID56_MASK: ASSERT sctx.raid56_data_stripes == NULL; allocate nr_data_stripes(map) extra stripes; init each.
- Dispatch:
  - !(RAID0|RAID10|RAID56_MASK): scrub_simple_mirror(sctx, bg, bg.start, bg.length, scrub_dev, physical, stripe_index + 1).
  - RAID0|RAID10: scrub_simple_stripe(sctx, bg, map, scrub_dev, stripe_index).
  - RAID56: iterate by physical, for each get_raid56_logic_offset → if parity stripe: scrub_raid56_parity_stripe; else: scrub_simple_mirror(logical, BTRFS_STRIPE_LEN, scrub_dev, physical, mirror_num=1).
- flush_scrub_stripes(sctx); release paths.
- if raid56_data_stripes: release + kfree all + NULL.
- if is_dev_replace ∧ ret ≥ 0: sync_write_pointer_for_zoned.

REQ-8: scrub_simple_mirror(sctx, bg, logical_start, logical_length, device, physical, mirror_num):
- ASSERT logical_start ∈ bg ∧ logical_end ≤ btrfs_block_group_end(bg).
- While cur_logical < logical_end:
  - if should_cancel_scrub: break.
  - if scrub_pause_req: scrub_blocked_if_needed.
  - if BLOCK_GROUP_FLAG_REMOVED: break.
  - queue_scrub_stripe(sctx, bg, device, mirror_num, cur_logical, end-cur, cur_physical, &found_logical).
  - if ret > 0 (no more extents): update sctx.stat.last_physical; ret = 0; break.
  - cur_logical = found_logical + BTRFS_STRIPE_LEN; cond_resched().

REQ-9: queue_scrub_stripe(sctx, bg, dev, mirror_num, logical, length, physical, &found_logical):
- ASSERT cur_stripe < SCRUB_TOTAL_STRIPES.
- stripe = &sctx.stripes[cur_stripe]; scrub_reset_stripe(stripe).
- scrub_find_fill_first_stripe(bg, extent_path, csum_path, dev, physical, mirror_num, logical, length, stripe).
- if ret ≠ 0 (>0 no extent / <0 error): return ret.
- found_logical = stripe.logical; cur_stripe++.
- if cur_stripe % SCRUB_STRIPES_PER_GROUP == 0: submit_initial_group_read(sctx, cur_stripe - 8, 8).
- if cur_stripe == SCRUB_TOTAL_STRIPES: flush_scrub_stripes(sctx).

REQ-10: scrub_read_endio (per-bio completion):
- sector_nr = calc_sector_number(stripe, first_bvec).
- if bbio.bio.bi_status: scrub_bitmap_set_io_error + scrub_bitmap_set_error.
- else: scrub_bitmap_clear_io_error.
- bio_put.
- if atomic_dec_and_test(&stripe.pending_io): wake_up(io_wait); INIT_WORK(scrub_stripe_read_repair_worker); queue_work(scrub_workers).

REQ-11: scrub_stripe_read_repair_worker (per-stripe verify + repair):
- wait_scrub_stripe_io(stripe).
- scrub_verify_one_stripe(stripe, scrub_bitmap_read_has_extent(stripe)).
- Snapshot init_error_bitmap + nr_io_errors + nr_csum_errors + nr_meta_errors + nr_meta_gen_errors into errors.
- if bitmap_empty(init_error_bitmap): goto out.
- /* First repair pass: try every other mirror with BTRFS_STRIPE_LEN block size */
- for mirror = calc_next_mirror(stripe.mirror_num, num_copies); mirror ≠ stripe.mirror_num; mirror = calc_next_mirror(...):
  - scrub_stripe_submit_repair_read(stripe, mirror, BTRFS_STRIPE_LEN, false).
  - wait_scrub_stripe_io; scrub_verify_one_stripe(old_error_bitmap).
  - if empty_error: goto out.
- /* Last-resort: sector-by-sector across all mirrors */
- for i in 0..num_copies; mirror cycles via calc_next_mirror:
  - scrub_stripe_submit_repair_read(stripe, mirror, sectorsize, true).
  - wait + verify; if empty_error: goto out.
- out:
  - error = scrub_bitmap_read_error(stripe).
  - repaired = init_error_bitmap & ~error.
  - if !readonly ∧ !empty(repaired):
    - if zoned: btrfs_repair_one_zone (queue block group for relocation).
    - else: scrub_write_sectors(sctx, stripe, repaired, false=dev_replace); wait.
  - scrub_stripe_report_errors(sctx, stripe, &errors).
  - set_bit(SCRUB_STRIPE_FLAG_REPAIR_DONE, stripe.state); wake_up(repair_wait).

REQ-12: scrub_verify_one_sector(stripe, sector_nr):
- if !has_extent: return.
- if io_error: return (already accounted).
- if is_metadata:
  - if sector_nr + sectors_per_tree > stripe.nr_sectors: warn (crossed boundary, cannot verify); return.
  - scrub_verify_one_metadata(stripe, sector_nr).
- else (data):
  - if !sector.csum: clear error bit (no csum ⟹ trust); return.
  - btrfs_check_block_csum(fs_info, paddr, csum_buf, sector.csum).
  - on mismatch: set csum_error + error bits; else clear.

REQ-13: scrub_verify_one_metadata (tree block verify):
- Read header from first sector kaddr.
- Compare logical vs btrfs_stack_header_bytenr; if mismatch: set meta_error + error.
- Compare header.fsid vs metadata_uuid; mismatch: set meta_error + error.
- Compare header.chunk_tree_uuid vs fs_info.chunk_tree_uuid; mismatch: set meta_error + error.
- btrfs_csum_init(&csum, csum_type); btrfs_csum_update over first sector minus header csum field; then update over sectors[1..sectors_per_tree]; btrfs_csum_final.
- Compare against on_disk_csum; mismatch: set meta_error + error.
- Compare sectors[sector_nr].generation vs header.generation; mismatch: set meta_gen_error + error.
- All match: clear all 4 error sub-bits.

REQ-14: scrub_write_sectors(sctx, stripe, write_bitmap, dev_replace):
- For each set bit in write_bitmap:
  - ASSERT has_extent bit set.
  - If non-contiguous with previous: submit current bbio, alloc new.
  - scrub_bio_add_sector(bbio, stripe, sector_nr).
- Final submit via scrub_submit_write_bio (which fills writer-pointer gap on zoned, btrfs_submit_repair_write with mirror_num + dev_replace flag, waits if zoned).

REQ-15: scrub_raid56_parity_stripe(sctx, scrub_dev, bg, map, full_stripe_start):
- should_cancel_scrub / scrub_pause_req checks.
- For i in 0..data_stripes:
  - Compute stripe_index = (i + rot) % map.num_stripes; physical accordingly.
  - scrub_reset_stripe(raid56_data_stripes[i]); set SCRUB_STRIPE_FLAG_NO_REPORT.
  - scrub_find_fill_first_stripe(...); if no extent: mark INITIALIZED manually.
- if all_empty: return 0.
- For each: scrub_submit_initial_read.
- For each: wait_event(repair_wait, REPAIR_DONE).
- ASSERT !zoned (RAID56 + zoned not supported).
- For each: if error & has_extent ≠ 0: log "unrepaired sectors detected" + return; else extent_bitmap |= has_extent.
- return scrub_raid56_cached_parity(sctx, scrub_dev, map, full_stripe_start, &extent_bitmap).

REQ-16: scrub_raid56_cached_parity(sctx, scrub_dev, map, full_stripe_start, &extent_bitmap):
- DECLARE_COMPLETION_ONSTACK(io_done).
- bio_init local bio; bi_iter.bi_sector = full_stripe_start >> SECTOR_SHIFT; bi_private = io_done; bi_end_io = raid56_scrub_wait_endio.
- btrfs_bio_counter_inc_blocked.
- btrfs_map_block(BTRFS_MAP_WRITE, full_stripe_start, &length, &bioc, ...).
- rbio = raid56_parity_alloc_scrub_rbio(&bio, bioc, scrub_dev, extent_bitmap, BTRFS_STRIPE_LEN >> sectorsize_bits).
- For each data stripe: raid56_parity_cache_data_folios(rbio, stripe.folios, full_stripe_start + (i << BTRFS_STRIPE_LEN_SHIFT)).
- raid56_parity_submit_scrub_rbio(rbio); wait_for_completion_io(&io_done).
- btrfs_bio_counter_dec; bio_uninit; return blk_status_to_errno(bio.bi_status).

REQ-17: scrub_supers(sctx, dev):
- For each of the 3 super copies: scrub_one_super(sctx, dev, &sb, bytenr, gen) — read, csum-verify, increment super_errors counter on mismatch.

REQ-18: btrfs_scrub_pause / continue / cancel / cancel_dev:
- pause: atomic_inc(scrub_pause_req); wait until scrubs_paused == scrubs_running.
- continue: atomic_dec(scrub_pause_req); wake_up(scrub_pause_wait).
- cancel: if !scrubs_running: return -ENOTCONN. atomic_inc(scrub_cancel_req); wait until scrubs_running == 0; atomic_dec(scrub_cancel_req).
- cancel_dev: if !dev.scrub_ctx: return -ENOTCONN. atomic_inc(sctx.cancel_req); wait until dev.scrub_ctx == NULL.

REQ-19: btrfs_scrub_progress(fs_info, devid, progress):
- Lock device_list_mutex; lookup dev; sctx = dev.scrub_ctx.
- if sctx: memcpy(progress, &sctx.stat, sizeof(*progress)).
- Return -ENODEV / -ENOTCONN / 0.

REQ-20: should_cancel_scrub(sctx):
- if scrub_cancel_req ∨ sctx.cancel_req: return -ECANCELED.
- if sb.s_writers.frozen > SB_UNFROZEN ∨ freezing(current) ∨ signal_pending(current): return -EINTR.
- Else: 0.

REQ-21: scrub_throttle_dev_io(sctx, device, bio_size):
- bwlimit = READ_ONCE(device.scrub_speed_max); if 0: return.
- div = clamp(bwlimit / (16 MiB), 1, 64).
- 1-second slice: if within deadline ∧ over limit, sleep until deadline.

## Acceptance Criteria

- [ ] AC-1: btrfs_scrub_dev: rejects when fs is closing (-EAGAIN), missing dev (-ENODEV), non-writable non-replace target (-EROFS).
- [ ] AC-2: btrfs_scrub_dev: rejects concurrent scrub on same device (-EINPROGRESS) and rejects concurrent dev_replace.
- [ ] AC-3: scrub_setup_ctx: kvzalloc successful ⟹ all 128 stripes init'd; failure path frees partial state.
- [ ] AC-4: scrub_workers_get/put: refcount reaches 0 ⟹ destroy_workqueue; refcount > 0 ⟹ reuse.
- [ ] AC-5: scrub_enumerate_chunks: skips removed block groups; honors pause_req at chunk boundary.
- [ ] AC-6: scrub_stripe: SINGLE/DUP/RAID1/RAID1C* dispatched to scrub_simple_mirror; RAID0/10 to scrub_simple_stripe; RAID5/6 to scrub_raid56_parity_stripe + scrub_simple_mirror inner.
- [ ] AC-7: scrub_stripe_read_repair_worker: csum mismatch on mirror 1 ⟹ retries every other mirror; first clean mirror returned and written back to original.
- [ ] AC-8: scrub_verify_one_metadata: bytenr / fsid / chunk_tree_uuid / csum / generation all checked; any mismatch ⟹ meta_error counted.
- [ ] AC-9: scrub_write_sectors: writeback restricted to set bits of write_bitmap; never touches non-extent sectors.
- [ ] AC-10: scrub_raid56_parity_stripe: data stripes verified clean before P/Q regenerated; unrepaired data ⟹ EIO (no parity overwrite).
- [ ] AC-11: scrub_raid56_cached_parity: parity mismatch ⟹ parity overwrite via finish_parity_scrub; data sectors never overwritten by parity scrub.
- [ ] AC-12: btrfs_scrub_pause + transaction commit: scrub blocks at chunk boundary; resumes on continue.
- [ ] AC-13: btrfs_scrub_cancel: pending stripe reads drain; sctx.stat returned to user; ret = -ECANCELED.
- [ ] AC-14: btrfs_scrub_progress: returns sctx.stat under stat_lock without racing concurrent updates.
- [ ] AC-15: scrub_throttle_dev_io: scrub_speed_max = 0 ⟹ no throttle; > 0 ⟹ bps ≤ limit averaged over slice.

## Architecture

```
struct ScrubCtx {
  stripes: [ScrubStripe; SCRUB_TOTAL_STRIPES],   // 128
  raid56_data_stripes: Option<Box<[ScrubStripe]>>,
  fs_info: NonNull<FsInfo>,
  extent_path: BtrfsPath,
  csum_path: BtrfsPath,
  first_free: i32,
  cur_stripe: i32,
  cancel_req: AtomicI32,
  readonly: bool,
  throttle_deadline: KTime,
  throttle_sent: u64,
  is_dev_replace: bool,
  write_pointer: u64,
  wr_lock: Mutex,
  wr_tgtdev: Option<NonNull<BtrfsDevice>>,
  stat: BtrfsScrubProgress,                      // UAPI struct
  stat_lock: SpinLock,
  refs: RefCount,
}

struct ScrubStripe {
  sctx: NonNull<ScrubCtx>,
  bg: NonNull<BtrfsBlockGroup>,
  folios: [Option<*Folio>; SCRUB_STRIPE_MAX_FOLIOS],
  sectors: Box<[ScrubSectorVerification]>,
  dev: NonNull<BtrfsDevice>,
  logical: u64,
  physical: u64,
  mirror_num: u16,
  nr_sectors: u16,
  nr_data_extents: u16,
  nr_meta_extents: u16,
  pending_io: AtomicI32,
  io_wait: WaitQueueHead,
  repair_wait: WaitQueueHead,
  state: AtomicUlong,                            // ScrubStripeFlag bits
  bitmaps: BitArray<{ scrub_bitmap_nr_last * 16 }>,
  write_error_bitmap: BitMap,
  write_error_lock: SpinLock,
  csums: Option<Box<[u8]>>,
  work: WorkStruct,
}

enum ScrubStripeFlag { Initialized, RepairDone, NoReport }
enum ScrubBitmapKind { HasExtent, IsMetadata, Error, IoError, CsumError, MetaError, MetaGenError }
```

`Scrub::scrub_dev(fs_info, devid, start, end, progress, readonly, is_dev_replace) -> Result<(), Errno>`:
1. if fs_info.is_closing: return -EAGAIN.
2. ASSERT nodesize ≤ BTRFS_STRIPE_LEN.
3. sctx = ScrubCtx::setup(fs_info, is_dev_replace)?.
4. sctx.stat.last_physical = start.
5. Scrub::workers_get(fs_info)?.
6. Lock device_list_mutex; dev = btrfs_find_device(...); validate state; release lock.
7. Lock scrub_lock; reject if dev.scrub_ctx ∨ concurrent dev_replace; set dev.scrub_ctx = sctx; release.
8. Scrub::pause_blocked_if_needed(fs_info).
9. atomic_inc(fs_info.scrubs_running).
10. memalloc_nofs_save.
11. if !is_dev_replace: ret = Scrub::supers(sctx, dev).
12. if !ret: ret = Scrub::enumerate_chunks(sctx, dev, start, end).
13. memalloc_nofs_restore.
14. atomic_dec(scrubs_running); wake_up(scrub_pause_wait).
15. if progress: copy sctx.stat.
16. dev.scrub_ctx = None.
17. Scrub::workers_put(fs_info); ScrubCtx::put(sctx).
18. If super_errors > old: btrfs_start_transaction + btrfs_commit_transaction.
19. Return ret.

`Scrub::enumerate_chunks(sctx, dev, start, end)`:
1. For each dev-extent item in [start, end):
   - lookup chunk_map; bg = btrfs_lookup_block_group(fs_info, chunk_offset).
   - Scrub::scrub_chunk(sctx, bg, dev, dev_offset, dev_extent_len).
   - btrfs_put_block_group(bg).

`Scrub::scrub_chunk(sctx, bg, dev, dev_offset, dev_extent_len)`:
1. map = btrfs_get_chunk_map(fs_info, bg.start, ...).
2. For each stripe_index in 0..map.num_stripes whose stripes[i].dev == dev: Scrub::scrub_stripe(sctx, bg, map, dev, stripe_index).
3. btrfs_free_chunk_map(map).

`Scrub::scrub_stripe(sctx, bg, map, scrub_dev, stripe_index)`:
1. profile = map.type & BTRFS_BLOCK_GROUP_PROFILE_MASK.
2. ASSERT sctx.extent_path empty.
3. Scrub::pause_blocked_if_needed(fs_info).
4. if is_dev_replace ∧ zoned-sequential target: sctx.write_pointer = physical.
5. if profile & RAID56_MASK: allocate + init raid56_data_stripes[nr_data_stripes(map)].
6. Dispatch by profile:
   - Simple (SINGLE|DUP|RAID1|RAID1Cx): Scrub::simple_mirror(sctx, bg, bg.start, bg.length, scrub_dev, physical, stripe_index + 1).
   - RAID0|RAID10: Scrub::simple_stripe(sctx, bg, map, scrub_dev, stripe_index).
   - RAID56: while physical < physical_end: get_raid56_logic_offset → parity ⟹ Scrub::raid56_parity_stripe; data ⟹ Scrub::simple_mirror(logical, BTRFS_STRIPE_LEN, scrub_dev, physical, 1); update last_physical; physical += BTRFS_STRIPE_LEN.
7. Scrub::flush_stripes(sctx); release paths.
8. if raid56_data_stripes: release_scrub_stripe each + kfree.
9. if is_dev_replace ∧ ret ≥ 0: sync_write_pointer_for_zoned.

`Scrub::simple_mirror(sctx, bg, logical_start, logical_length, device, physical, mirror_num)`:
1. While cur_logical < logical_end:
   - if Scrub::should_cancel: break.
   - if scrub_pause_req: pause_blocked_if_needed.
   - if BLOCK_GROUP_FLAG_REMOVED: break (ret = 0).
   - Scrub::queue_stripe(sctx, bg, device, mirror_num, cur_logical, end - cur, cur_physical, &found_logical).
   - if ret > 0: stat.last_physical = physical + logical_length; ret = 0; break.
   - cur_logical = found_logical + BTRFS_STRIPE_LEN; cond_resched.

`Scrub::queue_stripe(sctx, bg, dev, mirror_num, logical, length, physical, &found_logical)`:
1. ASSERT cur_stripe < 128.
2. stripe = &sctx.stripes[cur_stripe]; ScrubStripe::reset(stripe).
3. Scrub::find_fill_first_stripe(bg, extent_path, csum_path, dev, physical, mirror_num, logical, length, stripe).
4. if ret ≠ 0: return ret.
5. found_logical = stripe.logical; cur_stripe++.
6. if cur_stripe % SCRUB_STRIPES_PER_GROUP == 0: Scrub::submit_group(sctx, cur_stripe - 8, 8).
7. if cur_stripe == SCRUB_TOTAL_STRIPES: Scrub::flush_stripes(sctx).

`Scrub::stripe_repair_worker(stripe)` (workqueue handler):
1. wait_scrub_stripe_io(stripe).
2. Scrub::verify_stripe(stripe, scrub_bitmap_read_has_extent(stripe)).
3. errors = ScrubErrorRecords { init_error_bitmap, nr_io_errors, nr_csum_errors, nr_meta_errors, nr_meta_gen_errors }.
4. if init_error_bitmap empty: goto out.
5. for mirror = calc_next_mirror(stripe.mirror_num, num_copies) until back to stripe.mirror_num:
   - Scrub::stripe_submit_repair_read(stripe, mirror, BTRFS_STRIPE_LEN, false).
   - wait + Scrub::verify_stripe(old_error_bitmap).
   - if empty_error: goto out.
6. /* Sector-by-sector last resort */
7. for i in 0..num_copies; mirror cycles:
   - Scrub::stripe_submit_repair_read(stripe, mirror, sectorsize, true).
   - wait + Scrub::verify_stripe.
   - if empty_error: goto out.
8. out:
   - repaired = init_error_bitmap & !current_error.
   - if !readonly ∧ !empty(repaired):
     - if zoned: btrfs_repair_one_zone(fs_info, bg.start) /* queues for relocation */.
     - else: Scrub::write_sectors(sctx, stripe, repaired, false); wait.
   - Scrub::report_errors(sctx, stripe, &errors).
   - set_bit(RepairDone, stripe.state); wake_up(repair_wait).

`Scrub::verify_sector(stripe, sector_nr)`:
1. if !has_extent: return.
2. if io_error: return (counted by endio).
3. if is_metadata:
   - if sector_nr + sectors_per_tree > nr_sectors: warn (cross-boundary); return.
   - Scrub::verify_metadata(stripe, sector_nr).
4. else (data):
   - if !sector.csum: clear error_bit; return.
   - btrfs_check_block_csum(fs_info, paddr, csum_buf, sector.csum).
   - mismatch ⟹ set csum_error + error; else clear both.

`Scrub::raid56_parity_stripe(sctx, scrub_dev, bg, map, full_stripe_start)`:
1. should_cancel + pause checks.
2. For each data stripe i ∈ 0..data_stripes:
   - Compute rotated stripe_index + physical.
   - ScrubStripe::reset(raid56_data_stripes[i]); set NoReport flag.
   - Scrub::find_fill_first_stripe; if no extent: mark INITIALIZED manually.
3. if all stripes empty: return 0.
4. For each: Scrub::submit_stripe(sctx, stripe).
5. For each: wait_event(repair_wait, RepairDone bit set).
6. ASSERT !zoned.
7. For each: if error & has_extent ≠ 0: return EIO (refuse to corrupt P/Q with bad data).
8. Aggregate extent_bitmap |= has_extent.
9. Scrub::raid56_cached_parity(sctx, scrub_dev, map, full_stripe_start, &extent_bitmap).

`Scrub::raid56_cached_parity(sctx, scrub_dev, map, full_stripe_start, &extent_bitmap)`:
1. completion = ON_STACK; bio_init local; bi_end_io = raid56_scrub_wait_endio.
2. btrfs_bio_counter_inc_blocked.
3. btrfs_map_block(BTRFS_MAP_WRITE, full_stripe_start, ..., &bioc).
4. rbio = raid56_parity_alloc_scrub_rbio(bio, bioc, scrub_dev, extent_bitmap, BTRFS_STRIPE_LEN / sectorsize).
5. For each data stripe i: raid56_parity_cache_data_folios(rbio, stripe.folios, full_stripe_start + (i << BTRFS_STRIPE_LEN_SHIFT)).
6. raid56_parity_submit_scrub_rbio(rbio); wait_for_completion_io.
7. btrfs_bio_counter_dec; bio_uninit; return blk_status_to_errno(bio.bi_status).

`Scrub::supers(sctx, dev)`:
1. For each of the 3 super block copies (BTRFS_SUPER_INFO_OFFSET, mirror_2, mirror_3): Scrub::one_super(sctx, dev, sb, bytenr, gen) — read, csum-verify, increment sctx.stat.super_errors on mismatch.

`Scrub::pause(fs_info)`:
1. mutex_lock(scrub_lock); atomic_inc(scrub_pause_req).
2. while scrubs_paused ≠ scrubs_running:
   - mutex_unlock; wait_event(scrub_pause_wait, ...); mutex_lock.
3. mutex_unlock.

`Scrub::continue_(fs_info)`:
1. atomic_dec(scrub_pause_req); wake_up(scrub_pause_wait).

`Scrub::cancel(fs_info)`:
1. mutex_lock(scrub_lock); if !scrubs_running: unlock + return -ENOTCONN.
2. atomic_inc(scrub_cancel_req).
3. while scrubs_running > 0: unlock + wait + lock.
4. atomic_dec(scrub_cancel_req); unlock.

`Scrub::cancel_dev(dev)`:
1. mutex_lock(scrub_lock); sctx = dev.scrub_ctx; if !sctx: unlock + return -ENOTCONN.
2. atomic_inc(sctx.cancel_req).
3. while dev.scrub_ctx ≠ NULL: unlock + wait + lock.
4. unlock.

`Scrub::progress(fs_info, devid, progress)`:
1. Lock device_list_mutex; dev = btrfs_find_device.
2. sctx = dev ? dev.scrub_ctx : NULL.
3. if sctx: memcpy(progress, &sctx.stat, sizeof(*progress)).
4. unlock; return dev ? (sctx ? 0 : -ENOTCONN) : -ENODEV.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `sctx_refcount_balanced` | INVARIANT | per-scrub_setup_ctx + scrub_put_ctx: refs ≥ 1; scrub_free_ctx only on zero. |
| `stripe_state_repair_done_only_after_verify` | INVARIANT | per-stripe_repair_worker: SCRUB_STRIPE_FLAG_REPAIR_DONE set only after verify_one_stripe completes. |
| `pause_count_bounded` | INVARIANT | per-pause/continue: scrubs_paused ≤ scrubs_running ∧ scrub_pause_req ≥ 0. |
| `workqueue_refcount_balanced` | INVARIANT | per-workers_get + workers_put: scrub_workers_refcnt reaches 0 ⟹ scrub_workers destroyed exactly once. |
| `cur_stripe_in_range` | INVARIANT | per-queue_stripe: 0 ≤ cur_stripe ≤ SCRUB_TOTAL_STRIPES. |
| `write_bitmap_subset_of_has_extent` | INVARIANT | per-write_sectors: every set bit in write_bitmap has has_extent bit set. |
| `metadata_boundary_check` | INVARIANT | per-verify_one_sector: sectors_per_tree crossings warn-and-skip, never verify partial. |
| `raid56_no_data_clobber` | INVARIANT | per-raid56_parity_stripe: data sectors verified ∧ no error before parity regen; otherwise EIO. |
| `stat_lock_held_for_stat_mutation` | INVARIANT | per-stat updates: stat_lock held across read-modify-write. |
| `bbio_pending_io_balanced` | INVARIANT | per-stripe: atomic_inc(pending_io) before submit; atomic_dec_and_test in endio wakes io_wait. |
| `scrub_lock_held_for_dev_scrub_ctx_mutation` | INVARIANT | per-scrub_dev: dev.scrub_ctx ← sctx / NULL under scrub_lock. |

### Layer 2: TLA+

`fs/btrfs/scrub.tla`:
- Per-device scrub lifecycle: SETUP → SUPERS → ENUMERATE → (per chunk) STRIPE → (per group) READ → REPAIR → WRITE_BACK → ENDIO.
- Per-fs pause/cancel: PAUSE_REQ → DRAIN → BLOCK → CONTINUE_REQ → RESUME.
- Properties:
  - `safety_no_concurrent_scrub_per_device` — per-dev.scrub_ctx slot: at most one ScrubCtx at a time.
  - `safety_no_scrub_during_replace` — per-fs: scrub and dev_replace are mutually exclusive (except the scrub initiated by replace itself).
  - `safety_repaired_subset_of_init_error` — per-stripe: repaired bitmap ⊆ init_error_bitmap.
  - `safety_writeback_only_extent_sectors` — per-write_sectors: write_bitmap ⊆ has_extent.
  - `safety_raid56_parity_only_overwrite` — per-raid56 parity scrub: writes target only parity stripe (+replace target).
  - `safety_pause_drain` — per-pause: when scrub_pause_req > 0, scrubs eventually reach scrubs_paused == scrubs_running.
  - `liveness_per_scrub_terminates` — per-btrfs_scrub_dev: terminates with ret ∈ {0, ECANCELED, EINTR, EAGAIN, EROFS, ENODEV, EINPROGRESS, EIO, ENOMEM}.
  - `liveness_pause_eventually_resumes` — per-continue: every scrub blocked in pause eventually resumes after continue.
  - `liveness_cancel_eventually_finishes` — per-cancel: scrubs_running drops to 0 after atomic_inc(scrub_cancel_req).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `ScrubCtx::setup` post: sctx.refs == 1 ∧ 128 stripes initialized ∧ first_free == 0 | `Scrub::setup` |
| `Scrub::scrub_dev` post: dev.scrub_ctx restored to NULL on every exit path | `Scrub::scrub_dev` |
| `Scrub::queue_stripe` post: cur_stripe incremented iff stripe filled successfully | `Scrub::queue_stripe` |
| `Scrub::stripe_repair_worker` post: state.RepairDone set ∧ repair_wait woken | `Scrub::stripe_repair_worker` |
| `Scrub::verify_metadata` post: every bytenr/fsid/chunk_uuid/csum/generation mismatch marks an error sub-bitmap | `Scrub::verify_metadata` |
| `Scrub::write_sectors` post: writes restricted to bits ∈ write_bitmap | `Scrub::write_sectors` |
| `Scrub::raid56_parity_stripe` post: parity regen invoked only when all data verified clean | `Scrub::raid56_parity_stripe` |
| `Scrub::pause` post: scrub_pause_req incremented ∧ scrubs_paused == scrubs_running on return | `Scrub::pause` |
| `Scrub::cancel_dev` post: dev.scrub_ctx == NULL on return | `Scrub::cancel_dev` |
| `Scrub::progress` post: progress contents == sctx.stat snapshot under stat_lock | `Scrub::progress` |

### Layer 4: Verus/Creusot functional

`Per-scrub_dev → scrub_setup_ctx → scrub_supers → scrub_enumerate_chunks → (per chunk) scrub_stripe → (per profile) scrub_simple_mirror | scrub_simple_stripe | scrub_raid56_parity_stripe → queue_scrub_stripe → submit_initial_group_read → scrub_read_endio → scrub_stripe_read_repair_worker → scrub_verify_one_stripe → scrub_stripe_submit_repair_read (per mirror) → scrub_write_sectors → scrub_stripe_report_errors → end` semantic equivalence: per `Documentation/filesystems/btrfs.rst` (scrub), `Documentation/btrfs-scrub(8)`, and `include/uapi/linux/btrfs.h` (`BTRFS_IOC_SCRUB_*`, `struct btrfs_scrub_progress`).

`Per-scrub_pause / _continue / _cancel / _cancel_dev / _progress` semantic equivalence: behaviour matches userspace `btrfs scrub start/cancel/pause/resume/status` expectations.

`Per-RAID56-parity-scrub` semantic equivalence: data stripes verified clean ⟹ regenerate P/Q via `raid56_parity_submit_scrub_rbio` ⟹ parity mismatch ⟹ parity-only overwrite, matching the contract from sibling `raid56.md` REQ-18.

## Hardening

(Inherits row-1 features from `fs/btrfs/00-overview.md` § Hardening.)

Scrub reinforcement:

- **Per-mirror cycle for repair (calc_next_mirror)** — defense against per-fast-fail repair attempts; every mirror tried before giving up.
- **Per-sector-by-sector last-resort read** — defense against per-drive-internal-csum-failure that taints a whole large read.
- **Per-write-bitmap = init_error & ~final_error** — defense against per-write-clean-sector (only rewrite sectors that were broken and are now repaired).
- **Per-readonly mode honored** — defense against per-corrupt-mirror overwriting good data when admin requested verification only.
- **Per-zoned: queue block group for relocation instead of in-place rewrite** — defense against per-zone-append-required violation.
- **Per-RAID56: refuse parity regen if data stripes still have errors** — defense against per-bad-data persisted as good parity.
- **Per-NO_REPORT flag for RAID56 data stripes** — defense against per-double-count of errors found via parity-driven recovery.
- **Per-cancel + pause + freeze + signal checks at chunk boundary** — defense against per-runaway scrub blocking transaction commit / fsfreeze / suspend.
- **Per-stat_lock around every stat mutation** — defense against per-torn-counter on concurrent progress query.
- **Per-scrub_workers refcount via mutex** — defense against per-workqueue destroy while jobs in flight.
- **Per-dev_replace mutual exclusion** — defense against per-replace racing scrub on same device.
- **Per-throttle (scrub_speed_max)** — defense against per-scrub-saturates-disk starving foreground IO.
- **Per-super-block scrub triggers transaction commit when errors detected** — defense against per-counter-loss across crash.
- **Per-memalloc_nofs_save for the whole scrub** — defense against per-reclaim-deadlock vs transaction commit attempting to pause scrub.

## Grsecurity/PaX-style Reinforcement

Beyond the upstream hardening above, Rookery layers the following grsec/PaX-style controls onto `fs/btrfs/scrub.c`:

- **PAX_USERCOPY** — bounds-checks every `BTRFS_IOC_SCRUB*` payload (start, cancel, progress) so a malformed ioctl cannot drive a slab overrun in the scrub argument parser.
- **PAX_KERNEXEC** — keeps scrub worker callbacks and the per-stripe endio dispatch in read-only memory so a memory bug cannot rewrite the repair-path callbacks.
- **PAX_RANDKSTACK** — randomizes kernel-stack offset on each scrub-progress / scrub-start entry so an attacker cannot deterministically probe the long mirror-cycle path.
- **PAX_REFCOUNT** — wraps `scrub_ctx.refs` and `scrub_workers` refs so a torn cancel/pause sequence cannot wrap and trigger early free.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `scrub_ctx`, per-sector csum scratch, and write-bitmap buffers so leftover csum bytes cannot be recovered from the freelist.
- **PAX_UDEREF** — enforces user/kernel pointer separation across the scrub ioctl boundary.
- **PAX_RAP / kCFI** — forward-edge CFI on scrub endio and worker callbacks so a corrupted worker pointer cannot redirect repair into a gadget.
- **GRKERNSEC_HIDESYM** — strips kernel pointers from scrub-repair/error printks so a crafted bad mirror cannot leak kASLR offsets via dmesg.
- **GRKERNSEC_DMESG** — gates dmesg on `CAP_SYSLOG` so scrub repair-success/failure diagnostics do not leak pointer material to unprivileged users.
- **CAP_SYS_ADMIN on csum-verify ioctls** — `BTRFS_IOC_SCRUB`, `BTRFS_IOC_SCRUB_CANCEL`, `BTRFS_IOC_SCRUB_PROGRESS`, and any per-device csum-verify path are hard-gated so unprivileged callers cannot initiate or pause a full-fs csum verification and stress the repair-write path.
- **`scrub_workers` PAX_REFCOUNT** — the per-fs `scrub_workers` reference count (mutex-guarded against workqueue destroy while jobs in flight) is wrapped with the hardened-refcount layer so a malicious cancel/start storm cannot wrap and prematurely free the workqueue.
- **GRKERNSEC_HIDESYM on RAID56 parity-regen refuse paths** — RAID56 "refuse parity regen if data stripes still have errors" warning traces sanitize embedded pointers so a crafted parity-driven repair attack cannot extract scrub-ctx or stripe-cache addresses via dmesg.
- **PAX_MEMORY_SANITIZE on write-bitmap buffers** — `init_error & ~final_error` write-bitmap scratch is zeroed on free so a partial-repair crash cannot leave residual sector-bitmap material recoverable from the freelist.

Rationale: scrub is a privilege-relevant integrity verifier that issues writes against the very mirrors it is verifying; a `scrub_ctx` UAF or a write-bitmap stride overrun translates directly into silent on-disk corruption (a "repair" that overwrites good data), so capability gating + refcount hardening on the workqueue is layered on top of the upstream mirror-cycle / RAID56-no-parity-regen defenses.

## Open Questions

- Should Rookery support concurrent multi-device scrub (current upstream serializes per-device via dev.scrub_ctx) for faster verification across N drives?
- Whether to expose a per-fs `scrub_repair_max_attempts` sysfs knob (currently hard-coded by num_copies).
- Should RAID56 + zoned be supported (currently `ASSERT(!btrfs_is_zoned(...))` in `scrub_raid56_parity_stripe`)?

## Out of Scope

- `fs/btrfs/dev-replace.c` device-replace orchestration (it reuses `btrfs_scrub_dev` with `is_dev_replace = true` but adds its own state machine; covered in `dev-replace.md` if expanded).
- `fs/btrfs/raid56.c` rbio + parity engine internals (covered in sibling `raid56.md`).
- `fs/btrfs/checksum.c` per-CRC32C/XXHASH/SHA256/BLAKE2 implementations (covered in `checksum.md` if expanded).
- `fs/btrfs/extent-tree.c` extent-item search helpers (covered in `extent-tree.md`).
- `fs/btrfs/zoned.c` zoned-device support (covered separately if expanded).
- `fs/btrfs/relocation.c` `btrfs_repair_one_zone` block-group relocation (covered in `relocation.md` if expanded).
- `fs/btrfs/super.c` UAPI `BTRFS_IOC_SCRUB_*` ioctl plumbing (covered in `super.md`).
- Implementation code.
