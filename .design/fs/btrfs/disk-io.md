# Tier-3: fs/btrfs/disk-io.c — Btrfs disk I/O, tree-block cache, mount/umount core

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: fs/btrfs/00-overview.md
upstream-paths:
  - fs/btrfs/disk-io.c (~4955 lines)
  - fs/btrfs/disk-io.h
  - fs/btrfs/dev-replace.c (btrfs_init_dev_replace_tgtdev)
  - fs/btrfs/extent_io.c (btree_writepages, read_extent_buffer_pages)
  - include/uapi/linux/btrfs_tree.h (root objectids)
-->

## Summary

`fs/btrfs/disk-io.c` is the **tree-block I/O and mount/umount core** of btrfs. It owns the in-memory representation of metadata: per-`extent_buffer` (eb) cache backed by the per-`btree_inode` `address_space` (folio cache) plus the per-FS `xa_array` `buffer_tree` index. It serves: per-tree-block read (`read_tree_block`) with parent-transid + level + key + checksum verification; per-tree-block write submission (`btree_csum_one_bio` + `btree_writepages`); per-mount **open_ctree** (super → sys-chunk-array → chunk-tree → tree-root → backup-root fallback → all global roots → block-groups → log-replay → fs-tree); per-umount **close_ctree** (drain delalloc/iputs → commit-super → kthread-stop → free-roots); and per-super-write `write_dev_supers` + `wait_dev_supers` + `barrier_all_devices` + `write_all_supers` (per-mirror, primary FUA, secondaries lazy). Tree-block checksums use one of CRC32C/XXHASH/SHA256/BLAKE2 (`btrfs_supported_super_csum`). Per-`btrfs_init_dev_replace_tgtdev` registers a write-only replace-target device, sharing geometry from the srcdev. Critical for: per-mount integrity, per-corruption detection, per-dev-replace target setup, per-shutdown clean commit, per-multi-device atomic super commit.

This Tier-3 covers `fs/btrfs/disk-io.c` (~4955 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `open_ctree()` | per-mount tree-init | `DiskIo::open_ctree` |
| `close_ctree()` | per-umount teardown | `DiskIo::close_ctree` |
| `init_tree_roots()` | per-mount backup-root retry loop | `DiskIo::init_tree_roots` |
| `load_super_root()` | per-root read root-node eb | `DiskIo::load_super_root` |
| `load_important_roots()` | per-mount load tree-root + remap-root | `DiskIo::load_important_roots` |
| `btrfs_read_roots()` | per-mount load extent/dev/csum/uuid/free-space/quota/raid-stripe roots | `DiskIo::read_roots` |
| `load_global_roots()` | per-mount per-objectid extent/csum/free-space global roots (extent-tree-v2) | `DiskIo::load_global_roots` |
| `read_tree_block()` | per-eb read+verify | `DiskIo::read_tree_block` |
| `btrfs_find_create_tree_block()` | per-eb alloc-or-find | `DiskIo::find_create_tree_block` |
| `btrfs_validate_extent_buffer()` | per-eb post-read verify | `DiskIo::validate_extent_buffer` |
| `btrfs_read_extent_buffer()` | per-eb retry-mirrors read | `DiskIo::read_extent_buffer` |
| `btrfs_buffer_uptodate()` | per-eb uptodate + parent-transid match | `DiskIo::buffer_uptodate` |
| `csum_tree_block()` | per-eb checksum compute | `DiskIo::csum_tree_block` |
| `btree_csum_one_bio()` | per-write checksum + node/leaf check | `DiskIo::btree_csum_one_bio` |
| `btrfs_check_super_csum()` | per-super checksum verify | `DiskIo::check_super_csum` |
| `btrfs_validate_super()` | per-super sanity check | `DiskIo::validate_super` |
| `btree_writepages()` | per-mapping flush dirty ebs | `DiskIo::btree_writepages` |
| `btree_release_folio()` / `btree_invalidate_folio()` | per-folio reclaim | `DiskIo::btree_release_folio` / `_invalidate_folio` |
| `btree_aops` | per-btree_inode address_space_operations | `DiskIo::BTREE_AOPS` |
| `btrfs_init_btree_inode()` | per-mount fake btree-inode init | `DiskIo::init_btree_inode` |
| `write_dev_supers()` | per-device per-mirror super write | `DiskIo::write_dev_supers` |
| `wait_dev_supers()` | per-device super-write wait | `DiskIo::wait_dev_supers` |
| `write_dev_flush()` | per-device REQ_PREFLUSH submit | `DiskIo::write_dev_flush` |
| `wait_dev_flush()` | per-device flush wait | `DiskIo::wait_dev_flush` |
| `barrier_all_devices()` | per-FS-commit barrier broadcast | `DiskIo::barrier_all_devices` |
| `write_all_supers()` | per-trans-commit cross-device super | `DiskIo::write_all_supers` |
| `btrfs_init_dev_replace_tgtdev()` (dev-replace.c) | per-replace target alloc | `DiskIo::init_dev_replace_tgtdev` |
| `cleaner_kthread()` / `transaction_kthread()` | per-FS kthreads | `DiskIo::cleaner_kthread` / `transaction_kthread` |
| `btrfs_commit_super()` | per-FS commit on unmount | `DiskIo::commit_super` |
| `backup_super_roots()` | per-trans-commit backup-root array fill | `DiskIo::backup_super_roots` |
| `find_newest_super_backup()` / `read_backup_root()` | per-backup-root selection | `DiskIo::find_newest_backup` / `read_backup_root` |
| `btrfs_global_root_insert/_delete/_global()` | per-global-root rb_tree management | `DiskIo::global_root_insert/_delete/_lookup` |

## Compatibility contract

REQ-1: struct extent_buffer cache integration:
- Per-FS `fs_info->buffer_tree` (xarray, XA_FLAGS_LOCK_IRQ): per-bytenr → `extent_buffer *` lookup.
- Per-`btree_inode->i_mapping` folio cache: per-page-or-folio backs eb content; eb→folio pointers held in `eb->folios[INLINE_EXTENT_BUFFER_PAGES]` (or `eb->addr` for contiguous).
- Per-`BTRFS_I(btree_inode)->io_tree` (`extent_io_tree`, type `IO_TREE_BTREE_INODE_IO`): per-range state bits (UPTODATE, LOCKED, DIRTY, WRITEBACK).
- Per-`btrfs_find_create_tree_block(fs_info, bytenr, owner_root, level)` → calls `alloc_extent_buffer(fs_info, bytenr, owner_root, level)` (or test-buffer when `btrfs_is_testing(fs_info)`).
- Per-subpage (sectorsize < PAGE_SIZE) layout: folio's `private` = `btrfs_subpage *` bitmaps for per-sector dirty/uptodate/writeback/locked.
- Per-nodesize >= PAGE_SIZE: folio's `private` = eb pointer directly.

REQ-2: btrfs_init_btree_inode(sb):
- /* Per-mount fake-inode init for tree-block cache */
- inode = new_inode(sb).
- `btrfs_set_inode_number(BTRFS_I(inode), BTRFS_BTREE_INODE_OBJECTID)`; set_nlink(inode, 1).
- inode->i_size = OFFSET_MAX (entire logical address space).
- inode->i_mapping->a_ops = &btree_aops.
- mapping_set_gfp_mask(inode->i_mapping, GFP_NOFS).
- `btrfs_extent_io_tree_init(fs_info, &BTRFS_I(inode)->io_tree, IO_TREE_BTREE_INODE_IO)`.
- `btrfs_extent_map_tree_init(&BTRFS_I(inode)->extent_tree)`.
- BTRFS_I(inode)->root = `btrfs_grab_root(fs_info->tree_root)`.
- set BTRFS_INODE_DUMMY runtime flag.
- __insert_inode_hash(inode, hash); set AS_KERNEL_FILE.
- fs_info->btree_inode = inode.

REQ-3: btree_aops = { writepages = btree_writepages, release_folio = btree_release_folio, invalidate_folio = btree_invalidate_folio, migrate_folio = btree_migrate_folio, dirty_folio = btree_dirty_folio (else filemap_dirty_folio) }.
- Per-`btree_release_folio(folio)`: if folio_test_writeback ∨ folio_test_dirty: return false; else `try_release_extent_buffer(folio)`.
- Per-`btree_invalidate_folio(folio, off, len)`: extent_invalidate_folio + btree_release_folio + folio_detach_private if leftover.
- Per-`btree_migrate_folio(mapping, dst, src, mode)`: if folio_test_dirty(src): -EAGAIN; if private and !filemap_release_folio: -EAGAIN; else migrate_folio.
- Per-`btree_writepages(mapping, wbc)`: flush dirty extent_buffers via writeback path (`extent_io.c:btree_writepages`).

REQ-4: csum_tree_block(eb, result):
- /* Compute csum over (eb-content - first BTRFS_CSUM_SIZE bytes) */
- `btrfs_csum_init(&csum, fs_info->csum_type)`.
- If eb->addr (contiguous): kaddr = eb->addr; first_page_part = nodesize; num_pages = 1.
- Else: kaddr = folio_address(eb->folios[0]); first_page_part = min(PAGE_SIZE, nodesize); num_pages = num_extent_pages(eb).
- `btrfs_csum_update(&csum, kaddr + BTRFS_CSUM_SIZE, first_page_part - BTRFS_CSUM_SIZE)`.
- For i in 1..num_pages: csum_update on each folio page.
- `btrfs_csum_final(&csum, result)`; memset(result, 0, BTRFS_CSUM_SIZE) prior to final write.

REQ-5: btrfs_supported_super_csum(t): t ∈ {CRC32, XXHASH, SHA256, BLAKE2}.

REQ-6: btrfs_check_super_csum(fs_info, disk_sb):
- result[BTRFS_CSUM_SIZE].
- `btrfs_csum(fs_info->csum_type, (u8 *)disk_sb + BTRFS_CSUM_SIZE, BTRFS_SUPER_INFO_SIZE - BTRFS_CSUM_SIZE, result)`.
- return memcmp(disk_sb->csum, result, fs_info->csum_size) ≠ 0 ? 1 : 0.

REQ-7: btrfs_validate_extent_buffer(eb, check):
- Per-post-read verify: fsid match (`check_tree_block_fsid`) + level matches `check->level` + key in `check->first_key` + transid >= `check->transid` + nodesize match.
- For interior nodes: `btrfs_check_node(eb)`; for leaves: `btrfs_check_leaf(eb)`.
- On mismatch: clear_extent_buffer_uptodate; return -EIO or -EUCLEAN.

REQ-8: btrfs_buffer_uptodate(eb, parent_transid, check):
- If !extent_buffer_uptodate(eb): return 0.
- If parent_transid == 0 ∨ btrfs_header_generation(eb) == parent_transid:
  - if check ∧ btrfs_verify_level_key(eb, check) fails: return -EUCLEAN.
  - return 1.
- Else (transid mismatch): clear_extent_buffer_uptodate; return 0.

REQ-9: btrfs_read_extent_buffer(eb, check):
- Per-retry-all-mirrors loop:
  - ret = `read_extent_buffer_pages(eb, mirror_num, check)`.
  - if !ret: break (success).
  - num_copies = `btrfs_num_copies(fs_info, eb->start, eb->len)`.
  - if num_copies == 1: break.
  - mirror_num++ (skipping failed); if > num_copies: break.
- On success after failure, repair via `btrfs_repair_eb_io_failure(eb, failed_mirror)`.

REQ-10: read_tree_block(fs_info, bytenr, check):
- eb = `btrfs_find_create_tree_block(fs_info, bytenr, check->owner_root, check->level)`.
- if IS_ERR(eb): return ERR_CAST.
- if `btrfs_buffer_uptodate(eb, check->transid, check) == 1`: return eb.
- ret = `btrfs_read_extent_buffer(eb, check)`.
- if ret: `free_extent_buffer_stale(eb)`; return ERR_PTR(ret).
- return eb.

REQ-11: btree_csum_one_bio(bbio):
- /* Pre-write: csum + node/leaf check + generation check */
- eb = bbio->private; WARN if bbio->file_offset != eb->start or bi_size != eb->len.
- if EXTENT_BUFFER_ZONED_ZEROOUT set: memzero_extent_buffer; return 0.
- ASSERT metadata_uuid matches.
- `csum_tree_block(eb, result)`.
- if btrfs_header_level(eb) > 0: `btrfs_check_node(eb)`; else `btrfs_check_leaf(eb)`.
- last_trans = `btrfs_get_last_trans_committed(fs_info)`.
- if btrfs_header_generation(eb) <= last_trans: -EUCLEAN (write-time stale gen detection).
- `write_extent_buffer(eb, result, 0, fs_info->csum_size)`.

REQ-12: load_super_root(root, bytenr, gen, level):
- check = { level, transid = gen, owner_root = btrfs_root_id(root) }.
- root->node = `read_tree_block(root->fs_info, bytenr, &check)`.
- if IS_ERR: return PTR_ERR.
- `btrfs_set_root_node(&root->root_item, root->node)`; root->commit_root = `btrfs_root_node(root)`; refs = 1.

REQ-13: load_important_roots(fs_info):
- sb = fs_info->super_copy.
- bytenr = `btrfs_super_root(sb)`; gen = `btrfs_super_generation(sb)`; level = `btrfs_super_root_level(sb)`.
- ret = `load_super_root(fs_info->tree_root, bytenr, gen, level)`.
- if `btrfs_fs_incompat(fs_info, REMAP_TREE)`: also load `remap_root` from `btrfs_super_remap_root[_generation/_level]`.

REQ-14: init_tree_roots(fs_info):
- backup_index = `find_newest_super_backup(info)`.
- For i in 0..BTRFS_NUM_BACKUP_ROOTS (4):
  - if handle_error and USEBACKUPROOT mount opt set: free_root_pointers + read_backup_root(i).
  - ret = `load_important_roots(fs_info)`; on err set handle_error, continue.
  - `btrfs_init_root_free_objectid(tree_root)`.
  - ret = `btrfs_read_roots(fs_info)`; on err set handle_error, continue.
  - fs_info->generation = btrfs_header_generation(tree_root->node).
  - `btrfs_set_last_trans_committed(fs_info, fs_info->generation)`.
  - backup_root_index = (backup_index + 1) % NUM_BACKUP_ROOTS.
  - break (success).

REQ-15: btrfs_read_roots(fs_info) mount-stage tree load order:
- /* extent-tree-v2: global roots split per-objectid */
- `load_global_roots(tree_root)` → per-`BTRFS_EXTENT_TREE_OBJECTID` + `BTRFS_CSUM_TREE_OBJECTID` + `BTRFS_FREE_SPACE_TREE_OBJECTID` via `load_global_roots_objectid`.
- if `BLOCK_GROUP_TREE` compat-ro flag: load `BTRFS_BLOCK_GROUP_TREE_OBJECTID` → `fs_info->block_group_root`.
- `BTRFS_DEV_TREE_OBJECTID` → `fs_info->dev_root` (always); then `btrfs_init_devices_late(fs_info)`.
- if REMAP_TREE incompat: validate no data_reloc_tree.
- Else: load `BTRFS_DATA_RELOC_TREE_OBJECTID` → `fs_info->data_reloc_root`.
- `BTRFS_QUOTA_TREE_OBJECTID` → `fs_info->quota_root` (optional).
- `BTRFS_UUID_TREE_OBJECTID` → `fs_info->uuid_root` (-ENOENT tolerated).
- if RAID_STRIPE_TREE incompat: `BTRFS_RAID_STRIPE_TREE_OBJECTID` → `fs_info->stripe_root`.
- Each loaded root gets `BTRFS_ROOT_TRACK_DIRTY` state-bit.

REQ-16: open_ctree(sb, fs_devices) per-mount sequence:
1. `init_mount_fs_info(fs_info, sb)`.
2. Alloc `tree_root` (BTRFS_ROOT_TREE_OBJECTID) + `chunk_root` (BTRFS_CHUNK_TREE_OBJECTID) via `btrfs_alloc_root`.
3. `btrfs_init_btree_inode(sb)`.
4. invalidate_bdev(latest_dev->bdev); read disk-super; verify csum_type ∈ supported set; `btrfs_check_super_csum`.
5. memcpy super_copy + super_for_commit; `btrfs_validate_mount_super`.
6. Set sectorsize/nodesize/csum-size/csums_per_leaf etc into fs_info.
7. `btrfs_set_free_space_cache_settings` + `btrfs_check_options` + `btrfs_check_features(fs_info, !sb_rdonly)`.
8. If REMAP_TREE incompat: alloc remap_root (BTRFS_REMAP_TREE_OBJECTID).
9. `btrfs_alloc_compress_wsm` + `btrfs_init_workqueues`.
10. sb->s_bdi->ra_pages adjust; sb->s_blocksize = sectorsize; copy fsid.
11. `btrfs_read_sys_array(fs_info)` (system chunk array — bootstrap for chunk-tree lookup).
12. `load_super_root(chunk_root, super_chunk_root, super_chunk_root_generation, super_chunk_root_level)`.
13. `btrfs_read_chunk_tree(fs_info)` (per-DEV_ITEM + CHUNK_ITEM).
14. `btrfs_free_extra_devids(fs_devices)`.
15. `init_tree_roots(fs_info)` — REQ-14 (loads tree-root + REQ-15 cascade).
16. `btrfs_get_dev_zone_info_all_devices`.
17. UUID_TREE_GEN check vs `btrfs_super_uuid_tree_generation`; verify dev_items + dev_extents; recover_balance; init dev_stats; `btrfs_init_dev_replace`; check zoned mode.
18. sysfs add fsid + mounted; `btrfs_init_space_info`; `btrfs_read_block_groups`.
19. if REMAP_TREE: `btrfs_populate_fully_remapped_bgs_list`.
20. Free zone cache; check active zone reservation; refuse rw mount if missing devices.
21. kthread_run cleaner_kthread + transaction_kthread.
22. `btrfs_zoned_reserve_data_reloc_bg`; `btrfs_read_qgroup_config`; `btrfs_build_ref_tree`.
23. If `btrfs_super_log_root(disk_super) != 0` ∧ !NOLOGREPLAY: `btrfs_replay_log(fs_info, fs_devices)`.
24. `fs_info->fs_root = btrfs_get_fs_root(fs_info, BTRFS_FS_TREE_OBJECTID, true)`.
25. if !sb_rdonly: `btrfs_start_pre_rw_mount` + `btrfs_discard_resume`.
26. if uuid_root rescan needed: `btrfs_check_uuid_tree`.
27. Set BTRFS_FS_OPEN.
28. If BTRFS_FS_UNFINISHED_DROPS: wake_up_process(cleaner_kthread).

REQ-17: close_ctree(fs_info) per-umount sequence:
1. set BTRFS_FS_CLOSING_START.
2. `btrfs_wake_unfinished_drop`.
3. cancel_work_sync(reclaim_bgs_work).
4. kthread_park(cleaner_kthread) (so it can resume later for delayed iputs).
5. `btrfs_qgroup_wait_for_completion`; uuid_tree_rescan_sem down/up; `btrfs_pause_balance`; `btrfs_dev_replace_suspend_for_unmount`; `btrfs_scrub_cancel`.
6. wait_event(transaction_wait, defrag_running == 0); `btrfs_cleanup_defrag_inodes`.
7. If FS_ERROR: `btrfs_error_commit_super`.
8. Flush fixup_workers, delalloc_workers, workers, endio_workers, endio_write_workers, endio_freespace_worker.
9. `btrfs_run_delayed_iputs` (twice: once before, once after async-reclaim cancels).
10. cancel_work_sync(async_reclaim_work, async_data_reclaim_work, preempt_reclaim_work, em_shrinker_work).
11. set BTRFS_FS_STATE_NO_DELAYED_IPUT; `btrfs_discard_cleanup`.
12. if !sb_rdonly ∧ !shutdown: `btrfs_delete_unused_bgs`; flush delayed_workers; `btrfs_commit_super(fs_info)`.
13. kthread_stop(transaction_kthread, cleaner_kthread).
14. set BTRFS_FS_CLOSING_DONE.
15. `btrfs_check_quota_leak`; `btrfs_free_qgroup_config`.
16. WARN if `percpu_counter_sum(&fs_info->delalloc_bytes)` or `ordered_bytes` non-zero.
17. sysfs_remove_mounted/_fsid; put_block_group_cache; `warn_about_uncommitted_trans`; clear BTRFS_FS_OPEN.
18. `free_root_pointers(fs_info, true)` + `btrfs_free_fs_roots(fs_info)`.
19. invalidate_inode_pages2(btree_inode->i_mapping); `btrfs_stop_all_workers`; `btrfs_free_block_groups`; iput(btree_inode); `btrfs_mapping_tree_free`.

REQ-18: write_dev_supers(device, sb, max_mirrors):
- For mirror i in 0..max_mirrors:
  - bytenr = `btrfs_sb_offset(i)`; if `btrfs_sb_log_location(device, i, WRITE, &bytenr) == -ENOENT`: skip; else negative: count error.
  - if bytenr + SUPER_INFO_SIZE >= commit_total_bytes: break.
  - `btrfs_set_super_bytenr(sb, bytenr_orig)`.
  - `btrfs_csum(fs_info->csum_type, (u8 *)sb + CSUM_SIZE, SUPER_INFO_SIZE - CSUM_SIZE, sb->csum)`.
  - `folio = __filemap_get_folio(mapping, bytenr >> PAGE_SHIFT, FGP_LOCK|FGP_ACCESSED|FGP_CREAT, GFP_NOFS)`.
  - memcpy(folio_address(folio) + offset_in_folio, sb, SUPER_INFO_SIZE).
  - bio_alloc(device->bdev, 1, REQ_OP_WRITE | REQ_SYNC | REQ_META | REQ_PRIO, GFP_NOFS); bio_add_folio_nofail; bi_end_io = btrfs_end_super_write.
  - **Per-i == 0 ∧ !NOBARRIER: bio->bi_opf |= REQ_FUA** (primary durable; secondaries lazy).
  - submit_bio; `btrfs_advance_sb_log(device, i)`.

REQ-19: wait_dev_supers(device, max_mirrors):
- For each mirror: `filemap_get_folio(bdev->bd_mapping, bytenr >> PAGE_SHIFT)`; `folio_wait_locked(folio)`; folio_put.
- Return -1 if primary failed (BTRFS_SUPER_PRIMARY_WRITE_ERROR).

REQ-20: write_dev_flush(device): bio_init(flush_bio, bdev, REQ_OP_WRITE | REQ_SYNC | REQ_PREFLUSH); bi_end_io = btrfs_end_empty_barrier; init_completion(flush_wait); submit_bio. wait_dev_flush(device): wait_for_completion_io; on error set BTRFS_DEV_STATE_FLUSH_FAILED + dev_stat_inc FLUSH_ERRS.

REQ-21: barrier_all_devices(info):
- lockdep_assert_held(device_list_mutex).
- list_for_each_entry(dev, devices, dev_list) skip MISSING: write_dev_flush(dev).
- Then list_for_each_entry: wait_dev_flush(dev); count errors_wait.
- Return >= `btrfs_get_num_tolerated_disk_barrier_failures(fs_info->fs_devices->fs_type)` → error.

REQ-22: write_all_supers(trans):
- Per-fsync (trans state < UNBLOCKED): max_mirrors = 1.
- Per-trans-commit: max_mirrors = BTRFS_SUPER_MIRROR_MAX (3); `backup_super_roots(fs_info)`.
- Lock fs_devices->device_list_mutex.
- if do_barriers: `barrier_all_devices(fs_info)`; on err abort_transaction.
- Set BTRFS_HEADER_FLAG_WRITTEN in super flags.
- For each dev with bdev ∧ IN_FS_METADATA ∧ WRITEABLE: fill dev_item; `btrfs_validate_write_super`; `write_dev_supers(dev, sb, max_mirrors)`.
- if total_errors > max_errors: abort_transaction(-EIO); return -EIO.
- Second pass: `wait_dev_supers`; ditto error check.

REQ-23: btrfs_init_dev_replace_tgtdev(fs_info, device_path, srcdev, *device_out):
- /* Per-write-only target dev for online dev-replace */
- if srcdev->fs_devices->seeding: -EINVAL.
- bdev_file = `bdev_file_open_by_path(device_path, BLK_OPEN_WRITE, fs_info->sb, &fs_holder_ops)`.
- `btrfs_check_device_zone_type(fs_info, bdev)`.
- sync_blockdev(bdev).
- Reject if bdev already in fs_devices->devices.
- Reject if `bdev_nr_bytes(bdev) < btrfs_device_get_total_bytes(srcdev)`.
- device = `btrfs_alloc_device(NULL, &devid=BTRFS_DEV_REPLACE_DEVID, NULL, device_path)`.
- `lookup_bdev(device_path, &device->devt)`.
- Set BTRFS_DEV_STATE_WRITEABLE.
- Copy from srcdev: io_width = sectorsize, io_align = sectorsize, sector_size = sectorsize, total_bytes, disk_total_bytes, bytes_used, commit_total_bytes, commit_bytes_used.
- Set BTRFS_DEV_STATE_IN_FS_METADATA + BTRFS_DEV_STATE_REPLACE_TGT.
- `set_blocksize(bdev_file, BTRFS_BDEV_BLOCKSIZE)`.
- `btrfs_get_dev_zone_info(device, false)`.
- list_add(&device->dev_list, &fs_devices->devices); fs_devices->num_devices++; open_devices++.

REQ-24: cleaner_kthread(arg=fs_info):
- Per-loop: kthread_should_stop test; `btrfs_run_delayed_iputs`; `btrfs_clean_one_deleted_snapshot` (drops snapshots); `btrfs_delete_unused_bgs`; `btrfs_reclaim_bgs`; sleep on TASK_INTERRUPTIBLE for `commit_interval`.

REQ-25: transaction_kthread(arg=tree_root):
- Per-loop: periodic `btrfs_commit_transaction` based on `fs_info->commit_interval` (default 30s).

REQ-26: Backup-root rotation (`backup_super_roots`):
- Per-trans-commit: copy {tree_root, extent_root, chunk_root, dev_root, csum_root, fs_root + generations + levels + nbytes} into `fs_info->super_for_commit->super_roots[backup_root_index]`.
- backup_root_index = (idx + 1) % BTRFS_NUM_BACKUP_ROOTS (4 slots).

REQ-27: Per-global-roots rbtree (`fs_info->global_root_tree`):
- Keyed by (objectid, offset). `btrfs_global_root_insert`/`_delete` operate under `fs_info->global_root_lock`.
- `btrfs_csum_root(fs_info, bytenr)` → global_root_id mapping.
- `btrfs_extent_root(fs_info, bytenr)` → global_root_id mapping.

## Acceptance Criteria

- [ ] AC-1: Mount: open_ctree completes all stages → BTRFS_FS_OPEN set.
- [ ] AC-2: Mount with valid super: csum_type validated; -EINVAL if unsupported.
- [ ] AC-3: Mount with stale tree_root: USEBACKUPROOT cycles all 4 backup slots.
- [ ] AC-4: Mount loads tree-root → extent-tree (global) → dev-tree → csum-tree → uuid-tree → free-space-tree (global) → fs-tree.
- [ ] AC-5: read_tree_block returns EUCLEAN on parent-transid/level/key mismatch.
- [ ] AC-6: read_tree_block retries all mirrors on csum failure; repairs failed mirror.
- [ ] AC-7: btree_csum_one_bio refuses generation <= last_committed (-EUCLEAN).
- [ ] AC-8: write_all_supers: primary super written with REQ_FUA; secondaries no FUA.
- [ ] AC-9: write_all_supers: barrier_all_devices issues REQ_PREFLUSH to all devs before super write.
- [ ] AC-10: write_all_supers tolerates up to `btrfs_get_num_tolerated_disk_barrier_failures()` flush errors.
- [ ] AC-11: btrfs_init_dev_replace_tgtdev rejects target smaller than source (-EINVAL).
- [ ] AC-12: close_ctree commits super (unless shutdown ∨ FS_ERROR) before kthread_stop.
- [ ] AC-13: close_ctree leaves no delalloc/ordered bytes (WARN otherwise).
- [ ] AC-14: btree_release_folio refuses to release dirty/writeback folios.
- [ ] AC-15: btree_writepages flushes all dirty extent_buffers in mapping.

## Architecture

```
struct DiskIo {
  fs_info: *FsInfo,
}

struct FsInfo {
  super_copy: *BtrfsSuperBlock,
  super_for_commit: *BtrfsSuperBlock,
  tree_root: *BtrfsRoot,           // BTRFS_ROOT_TREE_OBJECTID
  chunk_root: *BtrfsRoot,          // BTRFS_CHUNK_TREE_OBJECTID
  dev_root: Option<*BtrfsRoot>,
  block_group_root: Option<*BtrfsRoot>,
  quota_root: Option<*BtrfsRoot>,
  uuid_root: Option<*BtrfsRoot>,
  data_reloc_root: Option<*BtrfsRoot>,
  remap_root: Option<*BtrfsRoot>,
  stripe_root: Option<*BtrfsRoot>,
  fs_root: Option<*BtrfsRoot>,     // BTRFS_FS_TREE_OBJECTID
  global_root_tree: RbRoot,        // per-(objectid, offset) — extent/csum/free-space
  global_root_lock: Spinlock,
  buffer_tree: XArray,             // per-bytenr → ExtentBuffer
  btree_inode: *Inode,
  csum_type: u16,
  csum_size: u32,
  sectorsize: u32,
  nodesize: u32,
  generation: u64,
  last_trans_committed: AtomicU64,
  backup_root_index: u32,
  flags: AtomicBitFlags<BtrfsFsFlag>,
  cleaner_kthread: *TaskStruct,
  transaction_kthread: *TaskStruct,
  fs_devices: *FsDevices,
  ...
}
```

`DiskIo::open_ctree(sb, fs_devices) -> Result<()>`:
1. /* Stage 1: alloc roots + btree-inode */
2. init_mount_fs_info(fs_info, sb).
3. fs_info.tree_root = btrfs_alloc_root(fs_info, BTRFS_ROOT_TREE_OBJECTID, GFP_KERNEL).
4. fs_info.chunk_root = btrfs_alloc_root(fs_info, BTRFS_CHUNK_TREE_OBJECTID, GFP_KERNEL).
5. DiskIo::init_btree_inode(sb).
6. /* Stage 2: read + verify disk super */
7. disk_super = btrfs_read_disk_super(latest_dev->bdev, 0, false).
8. csum_type = btrfs_super_csum_type(disk_super); if !btrfs_supported_super_csum(csum_type): -EINVAL.
9. btrfs_init_csum_hash(fs_info, csum_type).
10. if DiskIo::check_super_csum(fs_info, disk_super): -EINVAL.
11. memcpy(fs_info.super_copy, disk_super, sizeof(SuperBlock)); memcpy(super_for_commit, super_copy, ...).
12. DiskIo::validate_mount_super(fs_info).
13. Cache sectorsize/nodesize/csum_size/etc.
14. /* Stage 3: features + opts */
15. btrfs_check_features(fs_info, !sb_rdonly(sb)).
16. if REMAP_TREE incompat: fs_info.remap_root = btrfs_alloc_root(BTRFS_REMAP_TREE_OBJECTID).
17. /* Stage 4: chunk + tree-root bootstrap */
18. btrfs_init_workqueues(fs_info).
19. btrfs_read_sys_array(fs_info).
20. DiskIo::load_super_root(chunk_root, super_chunk_root, gen, level).
21. btrfs_read_chunk_tree(fs_info).
22. /* Stage 5: tree-root + cascading roots */
23. DiskIo::init_tree_roots(fs_info).  // REQ-14
24. /* Stage 6: zoned + dev + block groups */
25. btrfs_get_dev_zone_info_all_devices; verify dev_items/extents; recover_balance; init dev_stats; btrfs_init_dev_replace; check_zoned_mode.
26. btrfs_init_space_info; btrfs_read_block_groups; populate_fully_remapped_bgs_list (if remap).
27. /* Stage 7: kthreads */
28. fs_info.cleaner_kthread = kthread_run(cleaner_kthread, fs_info, "btrfs-cleaner").
29. fs_info.transaction_kthread = kthread_run(transaction_kthread, tree_root, "btrfs-transaction").
30. /* Stage 8: qgroup + ref-tree + log-replay + fs-root */
31. btrfs_read_qgroup_config; btrfs_build_ref_tree.
32. if super_log_root != 0 ∧ !NOLOGREPLAY: btrfs_replay_log(fs_info, fs_devices).
33. fs_info.fs_root = btrfs_get_fs_root(fs_info, BTRFS_FS_TREE_OBJECTID, true).
34. if !sb_rdonly(sb): btrfs_start_pre_rw_mount(fs_info); btrfs_discard_resume(fs_info).
35. if uuid rescan: btrfs_check_uuid_tree(fs_info).
36. set BTRFS_FS_OPEN; return Ok(()).

`DiskIo::init_tree_roots(fs_info) -> Result<()>`:
1. backup_index = DiskIo::find_newest_backup(fs_info).
2. For i in 0..BTRFS_NUM_BACKUP_ROOTS:
   - if handle_error ∧ USEBACKUPROOT: free_root_pointers(fs_info, 0); btrfs_set_super_log_root(sb, 0); DiskIo::read_backup_root(fs_info, i); backup_index = ret.
   - DiskIo::load_important_roots(fs_info).  /* tree-root + remap-root */
   - btrfs_init_root_free_objectid(tree_root).
   - DiskIo::read_roots(fs_info).  /* extent/dev/csum/uuid/free-space/quota/raid-stripe */
   - fs_info.generation = btrfs_header_generation(tree_root->node).
   - btrfs_set_last_trans_committed(fs_info, generation).
   - backup_root_index = (backup_index + 1) % NUM_BACKUP_ROOTS.
   - break Ok.

`DiskIo::read_roots(fs_info) -> Result<()>`:
1. DiskIo::load_global_roots(tree_root). /* extent/csum/free-space global roots */
2. if BLOCK_GROUP_TREE compat-ro: read BTRFS_BLOCK_GROUP_TREE_OBJECTID → block_group_root.
3. read BTRFS_DEV_TREE_OBJECTID → dev_root; btrfs_init_devices_late(fs_info).
4. if REMAP_TREE: validate data_reloc tree absent; else: get BTRFS_DATA_RELOC_TREE_OBJECTID → data_reloc_root.
5. read BTRFS_QUOTA_TREE_OBJECTID → quota_root (optional).
6. read BTRFS_UUID_TREE_OBJECTID → uuid_root (-ENOENT tolerated).
7. if RAID_STRIPE_TREE: read BTRFS_RAID_STRIPE_TREE_OBJECTID → stripe_root.
8. Each loaded root: set BTRFS_ROOT_TRACK_DIRTY.

`DiskIo::read_tree_block(fs_info, bytenr, check) -> Result<*ExtentBuffer>`:
1. eb = btrfs_find_create_tree_block(fs_info, bytenr, check.owner_root, check.level).
2. if let Ok(1) = DiskIo::buffer_uptodate(eb, check.transid, check): return eb. /* cache hit */
3. ret = DiskIo::read_extent_buffer(eb, check). /* retry-mirrors + repair */
4. if ret < 0: free_extent_buffer_stale(eb); return Err(ret).
5. return eb.

`DiskIo::btree_csum_one_bio(bbio) -> Result<()>`:
1. eb = bbio.private.
2. WARN bbio.file_offset == eb.start ∧ bi_size == eb.len.
3. if EXTENT_BUFFER_ZONED_ZEROOUT set: memzero_extent_buffer; return Ok.
4. ASSERT metadata_uuid match.
5. DiskIo::csum_tree_block(eb, result).
6. if btrfs_header_level(eb) > 0: btrfs_check_node(eb); else: btrfs_check_leaf(eb).
7. last_trans = btrfs_get_last_trans_committed(fs_info).
8. if btrfs_header_generation(eb) <= last_trans: return Err(-EUCLEAN).
9. write_extent_buffer(eb, result, 0, fs_info.csum_size); return Ok.

`DiskIo::write_all_supers(trans) -> Result<()>`:
1. do_barriers = !NOBARRIER.
2. if trans.state < TRANS_STATE_UNBLOCKED: max_mirrors = 1. /* fsync path */
3. else: max_mirrors = BTRFS_SUPER_MIRROR_MAX; backup_super_roots(fs_info).
4. lock fs_devices.device_list_mutex.
5. max_errors = num_devices - 1.
6. if do_barriers: DiskIo::barrier_all_devices(fs_info).
7. Set BTRFS_HEADER_FLAG_WRITTEN.
8. For each dev with bdev ∧ IN_FS_METADATA ∧ WRITEABLE:
   - Fill dev_item (devid, generation, type, total_bytes, bytes_used, io_align, io_width, sector_size, uuid, fsid).
   - btrfs_validate_write_super(fs_info, sb).
   - DiskIo::write_dev_supers(dev, sb, max_mirrors).
9. if total_errors > max_errors: abort_transaction(-EIO); return -EIO.
10. For each dev: DiskIo::wait_dev_supers(dev, max_mirrors).
11. Unlock; return Ok.

`DiskIo::close_ctree(fs_info)`:
1. set BTRFS_FS_CLOSING_START.
2. btrfs_wake_unfinished_drop; cancel_work_sync(reclaim_bgs_work); kthread_park(cleaner_kthread).
3. btrfs_qgroup_wait_for_completion; pause_balance; dev_replace_suspend_for_unmount; scrub_cancel.
4. wait_event(defrag_running == 0); cleanup_defrag_inodes.
5. if FS_ERROR: btrfs_error_commit_super.
6. flush_workqueue (fixup, delalloc, workers, endio, endio_write, endio_freespace).
7. btrfs_run_delayed_iputs.
8. cancel_work_sync (async_reclaim, async_data_reclaim, preempt_reclaim, em_shrinker).
9. btrfs_run_delayed_iputs (again).
10. set BTRFS_FS_STATE_NO_DELAYED_IPUT.
11. btrfs_discard_cleanup.
12. if rw ∧ !shutdown: delete_unused_bgs; flush(delayed_workers); btrfs_commit_super.
13. kthread_stop(transaction_kthread); kthread_stop(cleaner_kthread).
14. set BTRFS_FS_CLOSING_DONE.
15. free_qgroup_config; sysfs_remove_mounted/_fsid; put_block_group_cache; warn_about_uncommitted_trans; clear BTRFS_FS_OPEN.
16. free_root_pointers(fs_info, true); btrfs_free_fs_roots(fs_info).
17. invalidate_inode_pages2(btree_inode->i_mapping); btrfs_stop_all_workers; btrfs_free_block_groups; iput(btree_inode); btrfs_mapping_tree_free.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `super_csum_match` | INVARIANT | per-check_super_csum: csum mismatch ⟹ returns 1. |
| `parent_transid_match` | INVARIANT | per-buffer_uptodate: parent_transid ≠ 0 ⟹ header_gen == parent_transid for return 1. |
| `read_tree_block_owner_root` | INVARIANT | per-read_tree_block: check.owner_root passed to find_create_tree_block. |
| `btree_csum_one_bio_no_stale_gen` | INVARIANT | per-write: header_gen > last_trans_committed. |
| `write_dev_supers_primary_fua` | INVARIANT | per-mirror 0 ∧ !NOBARRIER: REQ_FUA set on primary. |
| `write_all_supers_barrier_first` | INVARIANT | per-do_barriers: barrier_all_devices precedes write_dev_supers. |
| `init_tree_roots_backup_idx_modulo` | INVARIANT | per-init_tree_roots: backup_root_index < BTRFS_NUM_BACKUP_ROOTS. |
| `close_ctree_kthreads_after_commit` | INVARIANT | per-close: btrfs_commit_super precedes kthread_stop. |
| `dev_replace_tgtdev_size_ok` | INVARIANT | per-init_tgtdev: bdev_nr_bytes(bdev) >= total_bytes(srcdev). |
| `btree_release_folio_clean` | INVARIANT | per-release_folio: dirty/writeback ⟹ false. |

### Layer 2: TLA+

`fs/btrfs/disk-io.tla`:
- Per-mount-state-machine: ALLOC → BOOTSTRAP → TREE_ROOTS → DEV_INIT → BLOCK_GROUPS → KTHREADS → LOG_REPLAY → FS_ROOT → OPEN.
- Per-umount: OPEN → CLOSING_START → DRAIN → COMMIT → KTHREADS_STOP → CLOSING_DONE → FREED.
- Per-super-write: WRITE_BARRIER → WRITE_PRIMARY → WRITE_SECONDARIES → WAIT_ALL.
- Properties:
  - `safety_backup_root_array_bounded` — per-init_tree_roots: never exceeds 4 slots.
  - `safety_primary_super_durable` — per-write_all_supers: REQ_FUA on primary always.
  - `safety_barrier_precedes_super_write` — per-do_barriers: flush bio waited before super bio.
  - `safety_mount_failure_calls_close_ctree` — per-failure-after-FS_OPEN-set: close_ctree invoked.
  - `safety_csum_type_supported_only` — per-mount: csum_type ∈ {CRC32, XXHASH, SHA256, BLAKE2}.
  - `liveness_open_ctree_terminates` — per-mount: either OPEN reached or open_ctree returns error.
  - `liveness_close_ctree_terminates` — per-umount: CLOSING_DONE reached.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `DiskIo::open_ctree` post: BTRFS_FS_OPEN set ∨ Err returned | `DiskIo::open_ctree` |
| `DiskIo::close_ctree` post: BTRFS_FS_CLOSING_DONE set; btree_inode put | `DiskIo::close_ctree` |
| `DiskIo::init_tree_roots` post: tree_root->node valid; generation set | `DiskIo::init_tree_roots` |
| `DiskIo::read_tree_block` post: eb verified by check (level, transid, owner, key) | `DiskIo::read_tree_block` |
| `DiskIo::btree_csum_one_bio` post: csum at offset 0; gen > last_trans | `DiskIo::btree_csum_one_bio` |
| `DiskIo::write_dev_supers` post: primary FUA when do_barriers; mirrors written contiguously | `DiskIo::write_dev_supers` |
| `DiskIo::write_all_supers` post: max_errors not exceeded; HEADER_FLAG_WRITTEN set | `DiskIo::write_all_supers` |
| `DiskIo::init_dev_replace_tgtdev` post: BTRFS_DEV_STATE_REPLACE_TGT + WRITEABLE + IN_FS_METADATA set | `DiskIo::init_dev_replace_tgtdev` |
| `DiskIo::load_super_root` post: root->node, root->commit_root non-NULL; refs=1 | `DiskIo::load_super_root` |
| `DiskIo::read_roots` post: dev_root non-NULL; track_dirty on all loaded | `DiskIo::read_roots` |

### Layer 4: Verus/Creusot functional

`Per-mount: open_ctree → init_tree_roots(USEBACKUPROOT fallback) → read_roots(extent+dev+csum+uuid+free-space) → start kthreads → log_replay → fs_root` semantic equivalence: per-Documentation/filesystems/btrfs.rst + per-`btrfs-progs/check`.

`Per-trans-commit: backup_super_roots → barrier_all_devices → write_dev_supers (primary FUA, secondaries lazy) → wait_dev_supers` semantic equivalence to upstream.

`Per-umount: drain_workqueues → run_delayed_iputs → btrfs_commit_super → kthread_stop → free_root_pointers` ordering semantic equivalence.

## Hardening

(Inherits row-1 features from `fs/btrfs/00-overview.md` § Hardening.)

Disk-IO reinforcement:

- **Per-super-csum verify before any tree-block read** — defense against per-superblock-corruption mount.
- **Per-parent-transid check on every eb read** — defense against per-stale-block read of stale-superblock-pointer.
- **Per-write-time generation check (gen > last_committed)** — defense against per-write-of-stale-cached-eb.
- **Per-tree-block level + key + fsid post-read verify** — defense against per-misdirected-write or per-wrong-mirror read.
- **Per-mirror retry with explicit repair** — defense against per-single-mirror corruption.
- **Per-USEBACKUPROOT 4-slot fallback** — defense against per-tree-root corruption with bounded retry.
- **Per-primary super FUA** — defense against per-power-loss between bio-ack and disk-cache flush.
- **Per-PREFLUSH barrier before any super write** — defense against per-out-of-order persistence violating commit order.
- **Per-tolerated barrier failure count** — defense against per-degraded-RAID losing all flush guarantees.
- **Per-dev-replace target size check** — defense against per-replace into smaller dev silently truncating data.
- **Per-dev-replace target seeding rejection** — defense against per-replace targeting seed FS.
- **Per-csum-type allowlist (CRC32C/XXHASH/SHA256/BLAKE2)** — defense against per-unknown-csum mount.
- **Per-folio release refuses dirty/writeback** — defense against per-data-loss on reclaim.
- **Per-umount flushed delalloc/ordered before commit** — defense against per-trailing-write-loss.
- **Per-shutdown skip commit-super** — defense against per-write-to-failed-device on umount.
- **Per-FS_ERROR commit_super skip** — defense against per-error-amplification on cleanup.

## Grsecurity/PaX-style Reinforcement

Beyond the upstream hardening above, Rookery layers the following grsec/PaX-style controls onto `fs/btrfs/disk-io.c`:

- **PAX_USERCOPY** — bounds-checks every super-block / tree-block buffer copied between user and kernel space (e.g. `BTRFS_IOC_GET_SUPER`-style consumers) so a malformed read cannot drive a slab overrun.
- **PAX_KERNEXEC** — keeps `btrfs_super_ops`, `btree_aops`, and `extent_io_ops` in read-only memory so a memory bug cannot rewrite the I/O-completion or readahead callbacks.
- **PAX_RANDKSTACK** — randomizes kernel-stack offset on entry to `open_ctree`/`close_ctree` and the tree-block read path, defeating deterministic stack-spray against the long mount-time parser.
- **PAX_REFCOUNT** — wraps `btrfs_root.refs` and `btree_inode` reference counts so a malicious image driving extreme root churn cannot wrap counts and trigger early free.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `btrfs_fs_info`, `btrfs_super_block` scratch copies, and tree-block buffers so leftover device UUIDs / csum-tree bytes cannot be recovered from the freelist.
- **PAX_UDEREF** — enforces user/kernel pointer separation across the long mount-time super parser.
- **PAX_RAP / kCFI** — forward-edge CFI on `btrfs_super_ops` and `extent_io_ops` so a corrupted ops pointer cannot redirect tree-block read completion to a userspace gadget.
- **GRKERNSEC_HIDESYM** — strips kernel pointers from btrfs mount/commit error printks so a crafted on-disk superblock cannot leak kASLR offsets via dmesg.
- **GRKERNSEC_DMESG** — gates dmesg on `CAP_SYSLOG` so disk-IO error traces do not leak pointer material to unprivileged users.
- **`write_dev_supers` PAX_USERCOPY** — the per-device super-write path validates the bounce buffer length against `BTRFS_SUPER_INFO_SIZE` via the hardened usercopy path, so a corrupted in-memory super cannot stride past the super-block bounce buffer.
- **`btree_inode` PAX_REFCOUNT** — the synthetic `btree_inode` reference count (held across every tree-block read/write) is wrapped with the hardened-refcount layer so a malicious image driving extreme tree-block churn cannot wrap and prematurely free the btree inode.
- **GRKERNSEC_HIDESYM on parent-transid / mirror-retry failures** — `verify_parent_transid` and `validate_extent_buffer` warning paths sanitize embedded pointers so a crafted stale-block attack cannot extract eb/page addresses via dmesg.
- **CAP_SYS_ADMIN on `BTRFS_IOC_DEV_REPLACE`** — dev-replace ioctls are hard-gated, blocking unprivileged callers from initiating a device-replace into a smaller or seed-flagged target.

Rationale: `disk-io.c` is the trust boundary between the on-disk image and the in-memory btrfs structures; any UAF on `btree_inode` or a write-side overrun in `write_dev_supers` translates directly into persistent on-disk corruption, so refcount + usercopy hardening is layered on top of the upstream csum/parent-transid/USEBACKUPROOT defenses.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- fs/btrfs/extent_io.c per-extent_buffer alloc/find/folio plumbing (covered separately in `extent-io.md` Tier-3 candidate)
- fs/btrfs/transaction.c commit_transaction core (covered in `transaction.md` Tier-3)
- fs/btrfs/super.c mount-API plumbing (covered in `super.md` Tier-3)
- fs/btrfs/tree-checker.c node/leaf structural checks (covered separately)
- fs/btrfs/tree-log.c log replay (covered separately)
- fs/btrfs/volumes.c chunk-tree + sys-chunk-array (covered separately)
- fs/btrfs/zoned.c zoned-block-device specifics (covered separately)
- Implementation code
