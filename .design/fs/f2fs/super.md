# Tier-3: fs/f2fs/super.c — F2FS superblock + checkpoint mount

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: fs/f2fs/00-overview.md
upstream-paths:
  - fs/f2fs/super.c (~5714 lines)
  - include/linux/f2fs_fs.h (on-disk struct f2fs_super_block + struct f2fs_checkpoint)
  - fs/f2fs/f2fs.h (struct f2fs_sb_info)
-->

## Summary

F2FS (Flash-Friendly File System) is Samsung's log-structured filesystem optimised for NAND flash and zoned (HM-SMR) block devices, default on most Android handsets. On-disk: two 4 KiB superblock copies at block 0 and block 1 (`F2FS_CP_PACKS`-like dual layout for the SB itself, plus a second double-buffer for the checkpoint pack). `struct f2fs_super_block` (4 KiB, packed) carries magic 0xF2F52010, log-shifts (log_sectorsize, log_sectors_per_block, log_blocksize == 12, log_blocks_per_seg == 9), block_count, segment_count_{ckpt, sit, nat, ssa, main}, area start addresses (cp_blkaddr, sit_blkaddr, nat_blkaddr, ssa_blkaddr, main_blkaddr), reserved inos (root=3, node=1, meta=2), feature bitmask, multi-device table `devs[MAX_DEVICES]`, quota inos, encoding (casefold), `s_stop_reason` + `s_errors` history, and a trailing CRC32. Per-mount `struct f2fs_sb_info` (sbi) caches the parsed superblock, the active checkpoint folio `ckpt`, current-segment array, NAT/SIT managers, GC thread handle, percpu counters, and the global cp_global_sem. Per-`f2fs_fill_super` flow: alloc sbi → set 4 KiB blocksize → `read_raw_super_block` (try block 0, fall back to block 1) → parse mount opts → sanity-check super → iget meta-inode → `f2fs_get_valid_checkpoint` (pick the freshest of the two CP packs via `cur_cp_version`) → scan devices → build segment-manager, node-manager, GC manager → iget root → recover orphans → roll-forward fsync recovery → start GC thread. Per-`F2FS_FEATURE_*`: encrypt, blkzoned, atomic_write, extra_attr, project-quota, inode_chksum, flexible_inline_xattr, quota_ino, inode_crtime, lost_found, verity, sb_chksum, casefold, compression, RO, device_alias, packed_ssa. Per-CKPT double-buffer: every checkpoint write rotates between two on-disk packs so an interrupted CP never leaves the image without a previous valid pack. Critical for: Android app boot latency, flash endurance, atomic write API, transparent compression, zoned-storage support.

This Tier-3 covers `fs/f2fs/super.c` (~5714 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct f2fs_super_block` | per on-disk SB | `F2fsSuperBlockOnDisk` |
| `struct f2fs_checkpoint` | per on-disk CP pack | `F2fsCheckpointOnDisk` |
| `struct f2fs_sb_info` | per-mount info | `F2fsSbInfo` |
| `struct f2fs_fs_context` | per fs-context | `F2fsFsContext` |
| `f2fs_fill_super()` | per fill_super | `F2fs::fill_super` |
| `read_raw_super_block()` | per dual-SB read | `F2fs::read_raw_super_block` |
| `sanity_check_raw_super()` | per validate SB | `F2fs::sanity_check_raw_super` |
| `sanity_check_area_boundary()` | per area boundary | `F2fs::sanity_check_area_boundary` |
| `f2fs_sanity_check_ckpt()` | per validate CP | `F2fs::sanity_check_ckpt` |
| `f2fs_commit_super()` | per write SB | `F2fs::commit_super` |
| `__f2fs_commit_super()` | per write-folio | `F2fs::commit_super_inner` |
| `init_sb_info()` | per init sbi fields | `F2fs::init_sb_info` |
| `init_percpu_info()` | per percpu counters | `F2fs::init_percpu_info` |
| `init_blkz_info()` | per zoned-dev init | `F2fs::init_blkz_info` |
| `f2fs_scan_devices()` | per multi-device | `F2fs::scan_devices` |
| `f2fs_setup_casefold()` | per casefold encoding | `F2fs::setup_casefold` |
| `f2fs_parse_param()` | per parse-option | `F2fs::parse_param` |
| `f2fs_apply_options()` | per apply opts | `F2fs::apply_options` |
| `default_options()` | per opt-defaults | `F2fs::default_options` |
| `f2fs_check_opt_consistency()` | per opt-validate | `F2fs::check_opt_consistency` |
| `f2fs_sanity_check_options()` | per opt sanity | `F2fs::sanity_check_options` |
| `f2fs_get_tree()` | per get_tree | `F2fs::get_tree` |
| `f2fs_reconfigure()` / `__f2fs_remount()` | per remount | `F2fs::reconfigure` |
| `f2fs_init_fs_context()` | per init-fc | `F2fs::init_fs_context` |
| `f2fs_fc_free()` | per fc-free | `F2fs::fc_free` |
| `kill_f2fs_super()` | per kill_sb | `F2fs::kill_super` |
| `f2fs_put_super()` | per umount | `F2fs::put_super` |
| `f2fs_alloc_inode()` / `f2fs_free_inode()` | per-inode slab | `F2fs::alloc_inode` / `free_inode` |
| `f2fs_drop_inode()` | per drop_inode | `F2fs::drop_inode` |
| `f2fs_sync_fs()` | per sync_fs | `F2fs::sync_fs` |
| `f2fs_freeze()` / `f2fs_unfreeze()` | per freeze | `F2fs::freeze` / `unfreeze` |
| `f2fs_statfs()` | per statfs | `F2fs::statfs` |
| `f2fs_show_options()` | per /proc/mounts | `F2fs::show_options` |
| `f2fs_shutdown()` | per shutdown ioctl | `F2fs::shutdown` |
| `f2fs_disable_checkpoint()` / `_enable_checkpoint()` | per nocp toggle | `F2fs::disable_checkpoint` / `enable_checkpoint` |
| `f2fs_start_gc_thread()` / `_stop_gc_thread()` | per GC kthread | `F2fs::start_gc_thread` / `stop_gc_thread` |
| `f2fs_start_ckpt_thread()` / `_stop_ckpt_thread()` | per CP kthread | `F2fs::start_ckpt_thread` / `stop_ckpt_thread` |
| `f2fs_handle_critical_error()` | per fatal-error | `F2fs::handle_critical_error` |
| `f2fs_save_errors()` / `f2fs_handle_error()` | per error log | `F2fs::save_errors` / `handle_error` |
| `f2fs_record_stop_reason()` / `save_stop_reason()` | per stop reason | `F2fs::record_stop_reason` |
| `f2fs_build_fault_attr()` | per fault injection | `F2fs::build_fault_attr` |

## Compatibility contract

REQ-1: struct f2fs_super_block on-disk (must match `f2fs_fs.h`):
- magic (le32): F2FS_SUPER_MAGIC = 0xF2F52010.
- major_ver / minor_ver (le16).
- log_sectorsize (le32) ∈ [F2FS_MIN_LOG_SECTOR_SIZE, F2FS_MAX_LOG_SECTOR_SIZE].
- log_sectors_per_block (le32); log_sectors_per_block + log_sectorsize == F2FS_MAX_LOG_SECTOR_SIZE.
- log_blocksize (le32) == F2FS_BLKSIZE_BITS (12 → 4 KiB).
- log_blocks_per_seg (le32) == 9 (512 blocks per 2 MiB segment).
- segs_per_sec / secs_per_zone (le32).
- checksum_offset (le32): when SB_CHKSUM set, == offsetof(crc).
- block_count (le64): total user blocks.
- section_count / segment_count (le32).
- segment_count_{ckpt, sit, nat, ssa, main} (le32): per-area segment partition.
- segment0_blkaddr / cp_blkaddr / sit_blkaddr / nat_blkaddr / ssa_blkaddr / main_blkaddr (le32): area start LBA.
- root_ino == 3, node_ino == 1, meta_ino == 2.
- uuid[16]; volume_name[MAX_VOLUME_NAME].
- extension_count (le32) ≤ F2FS_MAX_EXTENSION; extension_list (cold-data file extensions).
- cp_payload (le32) < blocks_per_seg - F2FS_CP_PACKS - NR_CURSEG_PERSIST_TYPE.
- version[VERSION_LEN] / init_version[VERSION_LEN].
- feature (le32): F2FS_FEATURE_* bitmask.
- encryption_level / encrypt_pw_salt[16].
- devs[MAX_DEVICES] (each: path[MAX_PATH_LEN], total_segments).
- qf_ino[F2FS_MAX_QUOTAS]: per-quota-type inode numbers.
- hot_ext_count (u8).
- s_encoding (le16) / s_encoding_flags (le16): casefold UTF-8 version.
- s_stop_reason[MAX_STOP_REASON] / s_errors[MAX_F2FS_ERRORS]: persistent error log.
- crc (le32): trailing CRC32 (when SB_CHKSUM set).

REQ-2: F2FS_FEATURE_*:
- ENCRYPT (0x1): fs-crypt enabled.
- BLKZONED (0x2): zoned-device (HM-SMR) aware.
- ATOMIC_WRITE (0x4): atomic write API.
- EXTRA_ATTR (0x8): extended inode area.
- PRJQUOTA (0x10): project quota.
- INODE_CHKSUM (0x20): per-inode CRC.
- FLEXIBLE_INLINE_XATTR (0x40): variable-size inline xattr.
- QUOTA_INO (0x80): on-disk quota inodes.
- INODE_CRTIME (0x100): creation time stored.
- LOST_FOUND (0x200): /lost+found preserved by fsck.
- VERITY (0x400): fs-verity supported.
- SB_CHKSUM (0x800): superblock CRC32 valid.
- CASEFOLD (0x1000): case-insensitive directories (UTF-8).
- COMPRESSION (0x2000): per-file compression (LZ4 / LZO / ZSTD / LZORLE).
- RO (0x4000): read-only image (no GC, no CP write).
- DEVICE_ALIAS (0x8000): device aliasing.
- PACKED_SSA (0x10000): packed SSA layout.

REQ-3: struct f2fs_checkpoint on-disk:
- checkpoint_ver (le64): monotonic CP version (cur_cp_version chooses freshest pack).
- user_block_count / valid_block_count (le64).
- rsvd_segment_count / overprov_segment_count / free_segment_count (le32).
- cur_node_segno[MAX_ACTIVE_NODE_LOGS] / cur_node_blkoff[…] (le32/le16): per-log node current segment.
- cur_data_segno[MAX_ACTIVE_DATA_LOGS] / cur_data_blkoff[…] (le32/le16): per-log data current segment.
- ckpt_flags (le32): CP_UMOUNT_FLAG, CP_ORPHAN_PRESENT_FLAG, CP_FSCK_FLAG, CP_ERROR_FLAG, CP_DISABLED_FLAG, CP_DISABLED_QUICK_FLAG, CP_QUOTA_NEED_FSCK_FLAG, CP_NAT_BITS_FLAG, CP_CRC_RECOVERY_FLAG, CP_LARGE_NAT_BITMAP_FLAG, CP_TRIMMED_FLAG, CP_RESIZEFS_FLAG, …
- cp_pack_total_block_count (le32): blocks per CP pack.
- cp_pack_start_sum (le32): summary-blocks start within pack.
- valid_node_count / valid_inode_count (le32).
- next_free_nid (le32).
- Trailing journals + bitmap regions.

REQ-4: sanity_check_raw_super(sbi, folio, index):
- if le32(raw_super.magic) ≠ F2FS_SUPER_MAGIC: -EINVAL.
- if SB_CHKSUM feature:
  - crc_offset = le32(raw_super.checksum_offset); must equal offsetof(crc); else -EFSCORRUPTED.
  - crc = le32(raw_super.crc); compare f2fs_crc32(raw_super, crc_offset); else -EFSCORRUPTED.
- log_blocksize == F2FS_BLKSIZE_BITS (else -EFSCORRUPTED).
- log_blocks_per_seg == 9.
- log_sectorsize ∈ [F2FS_MIN_LOG_SECTOR_SIZE, F2FS_MAX_LOG_SECTOR_SIZE].
- log_sectors_per_block + log_sectorsize == F2FS_MAX_LOG_SECTOR_SIZE.
- segment_count ∈ [F2FS_MIN_SEGMENTS, F2FS_MAX_SEGMENT].
- total_sections ≤ segment_count_main ∧ ≥ 1 ∧ segs_per_sec ∈ (0, segment_count].
- segment_count_main == total_sections * segs_per_sec.
- segment_count ≤ block_count >> 9.
- if RDEV(0).path[0] (multi-dev): Σ devs[i].total_segments == segment_count.
- else if BLKZONED feature ∧ !bdev_is_zoned(sb.s_bdev): -EFSCORRUPTED "Zoned block device path is missing".
- secs_per_zone ∈ (0, total_sections].
- extension_count + hot_ext_count ≤ F2FS_MAX_EXTENSION.
- cp_payload < blocks_per_seg - F2FS_CP_PACKS - NR_CURSEG_PERSIST_TYPE.
- node_ino == 1 ∧ meta_ino == 2 ∧ root_ino == 3.
- sanity_check_area_boundary (cp_blkaddr < sit_blkaddr < nat_blkaddr < ssa_blkaddr < main_blkaddr; each interval sized correctly).

REQ-5: read_raw_super_block(sbi, *raw_super, *valid_super_block, *recovery):
- super = kzalloc(struct f2fs_super_block).
- for block in 0..2:
  - folio = read_mapping_folio(sb.s_bdev.bd_mapping, block, NULL).
  - if IS_ERR(folio): *recovery = 1; continue.
  - if sanity_check_raw_super(sbi, folio, block): folio_put; *recovery = 1; continue.
  - if !*raw_super: memcpy(super, F2FS_SUPER_BLOCK(folio, block), sizeof *super); *valid_super_block = block; *raw_super = super.
  - folio_put.
- if !*raw_super: kfree(super); return original err.

REQ-6: f2fs_commit_super(sbi, recover):
- if (recover ∧ f2fs_readonly) ∨ f2fs_hw_is_readonly: set SBI_NEED_SB_WRITE; -EROFS.
- if !recover ∧ has_sb_chksum: crc = f2fs_crc32(F2FS_RAW_SUPER(sbi), offsetof(crc)); RAW_SUPER.crc = le32(crc).
- /* Write back-up first */
- index = sbi.valid_super_block ? 0 : 1; folio = read_mapping_folio(index); __f2fs_commit_super(sbi, folio, index, true).
- if recover ∨ err: return.
- /* Write current */
- index = sbi.valid_super_block; folio = read_mapping_folio(index); __f2fs_commit_super(sbi, folio, index, true).

REQ-7: f2fs_fill_super(sb, fc):
- /* alloc sbi */
- sbi = kzalloc(f2fs_sb_info); init locks (gc_lock rwsem, cp_global_sem, node_write, node_change, cp_rwsem, quota_sem); init waitqueues (cp_wait); init lists (inode_list[NR_INODE_TYPE]); init INIT_WORK(s_error_work).
- sb_set_blocksize(sb, F2FS_BLKSIZE) [4 KiB; fail ⇒ goto free_sbi].
- read_raw_super_block(sbi, &raw_super, &valid_super_block, &recovery).
- sb.s_fs_info = sbi; sbi.raw_super = raw_super.
- memcpy sbi.errors, raw_super.s_errors; memcpy sbi.stop_reason, raw_super.s_stop_reason.
- if inode_chksum-feature: sbi.s_chksum_seed = f2fs_chksum(~0, uuid).
- default_options(sbi, false); f2fs_check_opt_consistency(fc); f2fs_apply_options(fc); f2fs_sanity_check_options(sbi, false).
- sb.s_maxbytes = max_file_blocks(NULL) << log_blocksize; sb.s_max_links = F2FS_LINK_MAX.
- f2fs_setup_casefold(sbi).
- dq_op = &f2fs_quota_operations; s_qcop = &f2fs_quotactl_ops; quota types {USR, GRP, PRJ}.
- sb.s_op = &f2fs_sops; s_cop = &f2fs_cryptops; s_vop = &f2fs_verityops; s_xattr = f2fs_xattr_handlers; s_export_op = &f2fs_export_ops.
- sb.s_magic = F2FS_SUPER_MAGIC; sb.s_time_gran = 1.
- POSIX_ACL ⇒ SB_POSIXACL; INLINECRYPT ⇒ SB_INLINECRYPT; LAZYTIME ⇒ SB_LAZYTIME.
- super_set_uuid; super_set_sysfs_name_bdev.
- sbi.valid_super_block = valid_super_block.
- set_sbi_flag(SBI_POR_DOING) — disallow writes during fill_super.
- f2fs_init_write_merge_io; init_sb_info; f2fs_init_iostat; init_percpu_info; f2fs_init_page_array_cache.
- /* Meta inode (ino 2) */
- sbi.meta_inode = f2fs_iget(sb, F2FS_META_INO(sbi)).
- /* Pick valid checkpoint pack */
- f2fs_get_valid_checkpoint(sbi) — reads both CP packs, picks freshest by cp_pack_total_block_count CRC + cp_pack_total cur version, dups to sbi.ckpt.
- if CP_QUOTA_NEED_FSCK_FLAG: SBI_QUOTA_NEED_REPAIR.
- if CP_DISABLED_QUICK_FLAG: SBI_CP_DISABLED_QUICK.
- if CP_FSCK_FLAG: SBI_NEED_FSCK.
- f2fs_scan_devices.
- f2fs_init_post_read_wq.
- sbi.total_valid_node_count = le32(ckpt.valid_node_count); valid_inode_count percpu init; user_block_count = le64(ckpt.user_block_count); total_valid_block_count = le64(ckpt.valid_block_count).
- limit_reserve_root; adjust_unusable_cap_perc.
- f2fs_init_extent_cache_info; f2fs_init_ino_entry_info; f2fs_init_fsync_node_info.
- f2fs_init_ckpt_req_control; if !RO ∧ !DISABLE_CHECKPOINT ∧ MERGE_CHECKPOINT: f2fs_start_ckpt_thread.
- f2fs_build_segment_manager; f2fs_build_node_manager.
- sbi.sectors_written_start = f2fs_get_sectors_written(sbi).
- sbi.first_seq_zone_segno = get_first_seq_zone_segno.
- f2fs_build_gc_manager; f2fs_build_stats.
- sbi.node_inode = f2fs_iget(F2FS_NODE_INO).
- root = f2fs_iget(F2FS_ROOT_INO); validate S_ISDIR ∧ i_blocks ≠ 0 ∧ i_size ≠ 0 ∧ i_nlink ≠ 0.
- sb.s_root = d_make_root(root); generic_set_sb_d_ops.
- f2fs_init_compress_inode; f2fs_register_sysfs.
- if quota_ino-feature ∧ !RO: f2fs_enable_quotas.
- quota_enabled = f2fs_recover_quota_begin.
- f2fs_recover_orphan_inodes.
- if CP_DISABLED_FLAG: skip_recovery = true; goto reset_checkpoint.
- if !DISABLE_ROLL_FORWARD ∧ !NORECOVERY:
  - if hw_is_readonly: best-effort skip; else f2fs_recover_fsync_data(sbi, false) — replays fsync log into main.
- reset_checkpoint: f2fs_recover_quota_end; if skip_recovery: f2fs_check_and_fix_write_pointer.
- clear SBI_POR_DOING.
- f2fs_init_inmem_curseg.
- if DISABLE_CHECKPOINT: f2fs_disable_checkpoint; else if CP_DISABLED_FLAG: f2fs_enable_checkpoint.
- if (bggc_mode ≠ OFF ∨ GC_MERGE) ∧ !RO: f2fs_start_gc_thread.
- if recovery (i.e. broken SB recovered from backup): f2fs_commit_super(sbi, true) — rewrites the other copy.
- f2fs_join_shrinker; f2fs_tuning_parameters.
- notice "Mounted with checkpoint version = %llx" cur_cp_version(F2FS_CKPT(sbi)).
- f2fs_update_time(CP_TIME); f2fs_update_time(REQ_TIME).
- clear SBI_CP_DISABLED_QUICK.
- sbi.umount_lock_holder = NULL; return 0.
- On error: cascading goto unwind through {sync_free_meta, free_meta, free_compress_inode, free_root_inode, free_node_inode, free_stats, free_nm, free_sm, stop_ckpt_thread, free_devices, free_meta_inode, free_page_array_cache, free_percpu, free_iostat, free_bio_info, free_options, free_sb_buf, free_sbi}; retry-once when skip_recovery (one chance: try_onemore).

REQ-8: f2fs_scan_devices(sbi):
- foreach RDEV(i) with path[0] ≠ 0:
  - dev_open by path (or by inherited s_bdev for i==0).
  - if BLKZONED: init_blkz_info(sbi, i) — blkdev_report_zones with f2fs_report_zone_cb.
  - record total_segments per dev.
- enforce Σ total_segments == segment_count.

REQ-9: f2fs_reconfigure / __f2fs_remount(fc, sb):
- Parse new opts (f2fs_parse_param → fc.fs_private ctx).
- Validate consistency (no toggling unsupported features at remount).
- Switch DISABLE_CHECKPOINT on/off: f2fs_disable_checkpoint() / f2fs_enable_checkpoint() (writes a CP pack).
- Toggle GC_MERGE / bggc_mode → start/stop GC kthread.
- Toggle COMPRESS_CACHE accordingly.
- POSIX_ACL → SB_POSIXACL.
- Resize: if no-CP-pending and read-write, accept.

REQ-10: kill_f2fs_super(sb):
- if sb.s_root:
  - sbi.umount_lock_holder = current.
  - set SBI_IS_CLOSE.
  - f2fs_stop_gc_thread; f2fs_stop_discard_thread.
  - if COMPRESS_CACHE: truncate_inode_pages_final(COMPRESS_MAPPING).
  - if SBI_IS_DIRTY ∨ !CP_UMOUNT_FLAG:
    - cpc = { reason = CP_UMOUNT }; stat_inc_cp_call_count; f2fs_write_checkpoint(sbi, &cpc).
  - if SBI_IS_RECOVERED ∧ readonly: clear SB_RDONLY (so kill_block_super completes properly).
- kill_block_super(sb).
- if sbi: destroy_device_list; lockdep_unregister_key; kfree(sbi); sb.s_fs_info = NULL.

REQ-11: f2fs_freeze / f2fs_unfreeze:
- freeze: stop GC, flush in-flight, write final CP with CP_UMOUNT-equivalent; persist a freeze-marker so post-thaw recovery is no-op.
- unfreeze: clear freeze-marker; resume GC kthread.

REQ-12: f2fs_sync_fs(sb, sync):
- if SBI_POR_DOING ∨ SBI_CP_DISABLED: bypass.
- f2fs_balance_fs(sbi, ...); cpc = { reason = sync ? CP_SYNC : CP_FASTBOOT }; f2fs_write_checkpoint(sbi, &cpc).

REQ-13: f2fs_disable_checkpoint(sbi):
- Reserve segments for no-CP mode; freeze GC; mark CP_DISABLED_FLAG in next CP pack; subsequent writes append-only without CP packs.
- Allows fast-shutdown for non-crash-safe workloads (test only).

REQ-14: f2fs_enable_checkpoint(sbi):
- Re-allow CP writes; clear CP_DISABLED_FLAG.

REQ-15: f2fs_handle_critical_error(sbi):
- if SBI_STOP_BG_GC_ON_ERROR: stop GC.
- record_stop_reason; save_errors.
- per option `errors=`: continue / remount-ro / panic / shutdown.
- if shutdown: f2fs_stop_checkpoint; flag SBI_IS_SHUTDOWN.

REQ-16: f2fs_show_options(seq, root):
- Emit every non-default mount opt: background_gc, inline_xattr, inline_data, inline_dentry, flush_merge, data_flush, fastboot, extent_cache, mode, active_logs, alloc_mode, fsync_mode, whint_mode, test_dummy_encryption, atgc, gc_merge, age_extent_cache, compress_*, errors, ...
- Quota names; checkpoint_merge; case-fold encoding name.

REQ-17: f2fs_statfs(dentry, kstatfs):
- type = F2FS_SUPER_MAGIC; bsize = blocksize; blocks = user_block_count; bfree = user_block_count - total_valid_block_count - current_reserved_blocks; bavail = bfree - reserved_blocks; files = ckpt.valid_inode_count + ckpt.alloc_type_total; ffree = remaining_nids; namelen = F2FS_NAME_LEN.
- if project quota: f2fs_statfs_project.

REQ-18: f2fs_alloc_inode / f2fs_free_inode:
- kmem_cache_alloc(f2fs_inode_cachep, GFP_F2FS_ZERO) for f2fs_inode_info; init_once initializes the embedded vfs_inode.
- free_inode: kmem_cache_free.

## Acceptance Criteria

- [ ] AC-1: Mount on F2FS image: f2fs_fill_super succeeds; root ino == 3 resolved.
- [ ] AC-2: Bad magic in primary SB: read_raw_super_block tries secondary SB; recovery = 1.
- [ ] AC-3: Both SB copies bad: -EINVAL "Can't find valid F2FS filesystem".
- [ ] AC-4: SB_CHKSUM bad CRC: sanity_check_raw_super returns -EFSCORRUPTED.
- [ ] AC-5: log_blocksize ≠ 12: -EFSCORRUPTED.
- [ ] AC-6: segment_count_main ≠ total_sections * segs_per_sec: -EFSCORRUPTED.
- [ ] AC-7: BLKZONED feature ∧ !bdev_is_zoned: -EFSCORRUPTED.
- [ ] AC-8: Multi-dev image, Σ total_segments mismatch: -EFSCORRUPTED.
- [ ] AC-9: f2fs_get_valid_checkpoint picks highest cur_cp_version of the two CP packs.
- [ ] AC-10: Recovered SB from backup ⟹ f2fs_commit_super(true) rewrites the broken copy.
- [ ] AC-11: CP_DISABLED_FLAG on mount: skip roll-forward; goto reset_checkpoint.
- [ ] AC-12: !RO ∧ MERGE_CHECKPOINT: f2fs_start_ckpt_thread succeeds.
- [ ] AC-13: !RO ∧ bggc_mode != OFF: f2fs_start_gc_thread.
- [ ] AC-14: kill_f2fs_super on clean shutdown writes CP with CP_UMOUNT_FLAG.
- [ ] AC-15: f2fs_reconfigure(DISABLE_CHECKPOINT=on): f2fs_disable_checkpoint runs; subsequent CP packs flagged.
- [ ] AC-16: f2fs_sync_fs(sync=1): writes CP pack reason CP_SYNC.
- [ ] AC-17: try_onemore retry: skip_recovery branch retries fill_super once with NEED_FSCK.

## Architecture

```
struct F2fsSuperBlockOnDisk {       // packed LE; 4 KiB; matches f2fs_super_block byte-for-byte
  magic: u32,                        // F2FS_SUPER_MAGIC (0xF2F52010)
  major_ver: u16,
  minor_ver: u16,
  log_sectorsize: u32,
  log_sectors_per_block: u32,
  log_blocksize: u32,                // == F2FS_BLKSIZE_BITS (12)
  log_blocks_per_seg: u32,           // == 9
  segs_per_sec: u32,
  secs_per_zone: u32,
  checksum_offset: u32,
  block_count: u64,
  section_count: u32,
  segment_count: u32,
  segment_count_ckpt: u32,
  segment_count_sit: u32,
  segment_count_nat: u32,
  segment_count_ssa: u32,
  segment_count_main: u32,
  segment0_blkaddr: u32,
  cp_blkaddr: u32,
  sit_blkaddr: u32,
  nat_blkaddr: u32,
  ssa_blkaddr: u32,
  main_blkaddr: u32,
  root_ino: u32,                     // == 3
  node_ino: u32,                     // == 1
  meta_ino: u32,                     // == 2
  uuid: [u8; 16],
  volume_name: [u16; MAX_VOLUME_NAME],
  extension_count: u32,
  extension_list: [[u8; F2FS_EXTENSION_LEN]; F2FS_MAX_EXTENSION],
  cp_payload: u32,
  version: [u8; VERSION_LEN],
  init_version: [u8; VERSION_LEN],
  feature: u32,                      // F2FS_FEATURE_*
  encryption_level: u8,
  encrypt_pw_salt: [u8; 16],
  devs: [F2fsDevice; MAX_DEVICES],
  qf_ino: [u32; F2FS_MAX_QUOTAS],
  hot_ext_count: u8,
  s_encoding: u16,
  s_encoding_flags: u16,
  s_stop_reason: [u8; MAX_STOP_REASON],
  s_errors: [u8; MAX_F2FS_ERRORS],
  reserved: [u8; 258],
  crc: u32,
}

struct F2fsCheckpointOnDisk {
  checkpoint_ver: u64,
  user_block_count: u64,
  valid_block_count: u64,
  rsvd_segment_count: u32,
  overprov_segment_count: u32,
  free_segment_count: u32,
  cur_node_segno: [u32; MAX_ACTIVE_NODE_LOGS],
  cur_node_blkoff: [u16; MAX_ACTIVE_NODE_LOGS],
  cur_data_segno: [u32; MAX_ACTIVE_DATA_LOGS],
  cur_data_blkoff: [u16; MAX_ACTIVE_DATA_LOGS],
  ckpt_flags: u32,                   // CP_UMOUNT_FLAG | CP_FSCK_FLAG | CP_DISABLED_FLAG | ...
  cp_pack_total_block_count: u32,
  cp_pack_start_sum: u32,            // start LBA of data summary within pack
  valid_node_count: u32,
  valid_inode_count: u32,
  next_free_nid: u32,
  // ... journals, nat-bits, sit-journal, orphan-block, …
}

struct F2fsSbInfo {
  sb: *SuperBlock,
  raw_super: Box<F2fsSuperBlockOnDisk>,
  ckpt: Box<F2fsCheckpointOnDisk>,        // active checkpoint pack
  valid_super_block: i32,                  // 0 or 1
  meta_inode: InodeRef,                    // ino 2
  node_inode: InodeRef,                    // ino 1
  // locks
  gc_lock: RwSem,
  cp_global_sem: RwSem,
  node_write: RwSem,
  node_change: RwSem,
  cp_rwsem: RwSem,
  quota_sem: RwSem,
  writepages: Mutex,
  stat_lock: SpinLock,
  error_lock: SpinLock,
  cp_wait: WaitQueue,
  // managers
  sm_info: *SegmentManager,
  nm_info: *NodeManager,
  gc_thread: Option<KthreadHandle>,
  ckpt_thread: Option<KthreadHandle>,
  discard_thread: Option<KthreadHandle>,
  // counters / quotas
  total_valid_inode_count: PercpuCounter,
  total_valid_node_count: u32,
  total_valid_block_count: u64,
  last_valid_block_count: u64,
  user_block_count: u64,
  reserved_blocks: u64,
  current_reserved_blocks: u64,
  sectors_written_start: u64,
  s_chksum_seed: u32,
  first_seq_zone_segno: u32,
  reserved_pin_section: u32,
  // device list (multi-device)
  devs: Vec<F2fsDeviceCtx>,
  // mount opts
  opts: F2fsMountOpts,
  inode_list: [LinkedList<InodeRef>; NR_INODE_TYPE],
  inode_lock: [SpinLock; NR_INODE_TYPE],
  flush_lock: Mutex,
  // error / shutdown state
  flag: AtomicU32,                         // SBI_* flags
  errors: [u8; MAX_F2FS_ERRORS],
  stop_reason: [u8; MAX_STOP_REASON],
  s_error_work: WorkStruct,
  umount_lock_holder: Option<*Task>,
  // tunables
  interval_time: [u64; MAX_TIME],
  kbytes_written: u64,
}
```

`F2fs::fill_super(sb, fc) -> Result<(), Errno>`:
1. /* Allocate sbi + init locks */
2. sbi = Box::pin(F2fsSbInfo::default()?); init {gc_lock, cp_global_sem, node_write, node_change, cp_rwsem, quota_sem, writepages, stat_lock, error_lock, cp_wait}.
3. for i in 0..NR_INODE_TYPE: init inode_list[i], inode_lock[i].
4. mutex_init(&sbi.flush_lock).
5. /* Set 4 KiB blocksize */
6. if !sb_set_blocksize(sb, F2FS_BLKSIZE): err!("unable to set blocksize"); goto free_sbi.
7. /* Dual-SB read */
8. F2fs::read_raw_super_block(sbi, &mut raw_super, &mut valid_super_block, &mut recovery)?.
9. sb.s_fs_info = sbi; sbi.raw_super = raw_super.
10. INIT_WORK(&sbi.s_error_work, F2fs::record_error_work).
11. copy errors / stop_reason from raw_super.
12. if has_inode_chksum: sbi.s_chksum_seed = f2fs_chksum(~0, raw_super.uuid).
13. /* Options */
14. F2fs::default_options(sbi, false).
15. F2fs::check_opt_consistency(fc)?; F2fs::apply_options(fc); F2fs::sanity_check_options(sbi, false)?.
16. sb.s_maxbytes = max_file_blocks(None) << log_blocksize; sb.s_max_links = F2FS_LINK_MAX.
17. F2fs::setup_casefold(sbi)?.
18. /* Quota ops */
19. sb.s_op = &F2FS_SOPS; if CRYPT-build: sb.s_cop = &F2FS_CRYPTOPS; if VERITY-build: sb.s_vop = &F2FS_VERITYOPS.
20. sb.s_xattr = F2FS_XATTR_HANDLERS; sb.s_export_op = &F2FS_EXPORT_OPS; sb.s_magic = F2FS_SUPER_MAGIC; sb.s_time_gran = 1.
21. POSIX_ACL ⇒ SB_POSIXACL; INLINECRYPT ⇒ SB_INLINECRYPT; LAZYTIME ⇒ SB_LAZYTIME.
22. super_set_uuid(sb, &raw_super.uuid); super_set_sysfs_name_bdev(sb).
23. sbi.valid_super_block = valid_super_block.
24. set_sbi_flag(sbi, SBI_POR_DOING).
25. F2fs::init_write_merge_io(sbi)?; F2fs::init_sb_info(sbi); F2fs::init_iostat(sbi)?; F2fs::init_percpu_info(sbi)?; F2fs::init_page_array_cache(sbi)?.
26. /* Meta inode */
27. sbi.meta_inode = f2fs_iget(sb, F2FS_META_INO(sbi))?.
28. /* Pick freshest CP pack */
29. F2fs::get_valid_checkpoint(sbi)?.
30. /* Propagate persistent CP flags */
31. if CP_QUOTA_NEED_FSCK_FLAG: set SBI_QUOTA_NEED_REPAIR.
32. if CP_DISABLED_QUICK_FLAG: set SBI_CP_DISABLED_QUICK; opts.interval_time[DISABLE_TIME] = DEF_DISABLE_QUICK_INTERVAL.
33. if CP_FSCK_FLAG: set SBI_NEED_FSCK.
34. F2fs::scan_devices(sbi)?.
35. F2fs::init_post_read_wq(sbi)?.
36. /* Per-CP counters → sbi */
37. sbi.total_valid_node_count = le32(ckpt.valid_node_count).
38. percpu_counter_set(&sbi.total_valid_inode_count, le32(ckpt.valid_inode_count)).
39. sbi.user_block_count = le64(ckpt.user_block_count).
40. sbi.total_valid_block_count = le64(ckpt.valid_block_count); sbi.last_valid_block_count = sbi.total_valid_block_count.
41. limit_reserve_root(sbi); adjust_unusable_cap_perc(sbi).
42. F2fs::init_extent_cache_info(sbi); F2fs::init_ino_entry_info(sbi); F2fs::init_fsync_node_info(sbi).
43. F2fs::init_ckpt_req_control(sbi).
44. if !f2fs_readonly(sb) ∧ !DISABLE_CHECKPOINT ∧ MERGE_CHECKPOINT: F2fs::start_ckpt_thread(sbi)?.
45. F2fs::build_segment_manager(sbi)?; F2fs::build_node_manager(sbi)?.
46. sbi.sectors_written_start = F2fs::get_sectors_written(sbi).
47. sbi.first_seq_zone_segno = F2fs::get_first_seq_zone_segno(sbi).
48. sbi.reserved_pin_section = if blkzoned { ZONED_PIN_SEC_REQUIRED_COUNT } else { GET_SEC_FROM_SEG(overprovision_segments(sbi)) }.
49. /* IO stats from existing summary block */
50. seg_i = CURSEG_I(sbi, CURSEG_HOT_NODE).
51. if __exist_node_summaries(sbi): sbi.kbytes_written = le64(seg_i.journal.info.kbytes_written).
52. F2fs::build_gc_manager(sbi); F2fs::build_stats(sbi)?.
53. /* Node + root inodes */
54. sbi.node_inode = f2fs_iget(sb, F2FS_NODE_INO(sbi))?.
55. root = f2fs_iget(sb, F2FS_ROOT_INO(sbi))?; require S_ISDIR ∧ i_blocks ≠ 0 ∧ i_size ≠ 0 ∧ i_nlink ≠ 0.
56. generic_set_sb_d_ops(sb); sb.s_root = d_make_root(root).
57. F2fs::init_compress_inode(sbi)?; F2fs::register_sysfs(sbi)?.
58. sbi.umount_lock_holder = Some(current).
59. /* Quotas */
60. if has_quota_ino ∧ !RO: F2fs::enable_quotas(sb).
61. quota_enabled = F2fs::recover_quota_begin(sbi).
62. /* Orphan recovery */
63. F2fs::recover_orphan_inodes(sbi)?.
64. if CP_DISABLED_FLAG: skip_recovery = true; goto reset_checkpoint.
65. /* Roll-forward fsync recovery */
66. if !DISABLE_ROLL_FORWARD ∧ !NORECOVERY:
    - if hw_is_readonly:
      - if !CP_UMOUNT_FLAG: err = F2fs::recover_fsync_data(sbi, true); err > 0 ⇒ -EROFS "Need to recover fsync data, but write access unavailable".
      - info "write access unavailable, skipping recovery"; goto reset_checkpoint.
    - if need_fsck: SBI_NEED_FSCK.
    - if skip_recovery: goto reset_checkpoint.
    - err = F2fs::recover_fsync_data(sbi, false).
    - if err < 0: need_fsck = true; goto free_meta (unless ENOMEM: don't skip).
67. else: F2fs::recover_fsync_data(sbi, true) — read-only validation pass.
68. reset_checkpoint:
69.   F2fs::recover_quota_end(sbi, quota_enabled).
70.   if skip_recovery: F2fs::check_and_fix_write_pointer(sbi)?.
71.   clear_sbi_flag(sbi, SBI_POR_DOING).
72.   F2fs::init_inmem_curseg(sbi)?.
73.   if DISABLE_CHECKPOINT: F2fs::disable_checkpoint(sbi)?.
74.   else if CP_DISABLED_FLAG: F2fs::enable_checkpoint(sbi)?.
75.   if (bggc_mode ≠ OFF ∨ GC_MERGE) ∧ !RO: F2fs::start_gc_thread(sbi)?.
76. /* Rewrite the broken SB copy if any */
77. if recovery: F2fs::commit_super(sbi, true).
78. F2fs::join_shrinker(sbi); F2fs::tuning_parameters(sbi).
79. notice!("Mounted with checkpoint version = {:x}", cur_cp_version(sbi.ckpt.as_ref())).
80. F2fs::update_time(sbi, CP_TIME); F2fs::update_time(sbi, REQ_TIME).
81. clear_sbi_flag(sbi, SBI_CP_DISABLED_QUICK).
82. sbi.umount_lock_holder = None.
83. return Ok(()).
84. /* On error: cascading unwind; if retry_cnt > 0 ∧ skip_recovery: shrink_dcache_sb(sb); goto try_onemore. */

`F2fs::read_raw_super_block(sbi, raw_super, valid_super_block, recovery) -> Result<(), Errno>`:
1. let super = Box::new_zeroed::<F2fsSuperBlockOnDisk>()?.
2. for block in 0..2:
   - folio = read_mapping_folio(sb.s_bdev.bd_mapping, block, None);
   - if folio.is_err(): err!("Unable to read {}th superblock", block+1); *recovery = 1; continue.
   - if F2fs::sanity_check_raw_super(sbi, &folio, block).is_err(): err!(...); folio_put; *recovery = 1; continue.
   - if raw_super.is_none(): copy_from_folio(super, folio, block); *valid_super_block = block; *raw_super = Some(super.clone());
   - folio_put.
3. if raw_super.is_none(): drop super; return err.
4. return Ok(()).

`F2fs::sanity_check_raw_super(sbi, folio, index) -> Result<(), Errno>`:
1. let rs = F2FS_SUPER_BLOCK(folio, index).
2. if le32(rs.magic) ≠ F2FS_SUPER_MAGIC: -EINVAL.
3. if has_sb_chksum-feature:
   - crc_offset = le32(rs.checksum_offset); require == offsetof(crc).
   - require le32(rs.crc) == f2fs_crc32(rs, crc_offset).
4. require le32(rs.log_blocksize) == F2FS_BLKSIZE_BITS.
5. require le32(rs.log_blocks_per_seg) == 9.
6. require log_sectorsize ∈ [MIN, MAX]; log_sectors_per_block + log_sectorsize == F2FS_MAX_LOG_SECTOR_SIZE.
7. blocks_per_seg = BIT(log_blocks_per_seg).
8. require segment_count ∈ [F2FS_MIN_SEGMENTS, F2FS_MAX_SEGMENT].
9. require total_sections > 0 ∧ ≤ segment_count_main ∧ segs_per_sec ∈ (0, segment_count].
10. require segment_count_main == total_sections * segs_per_sec.
11. require (segment_count / segs_per_sec) ≥ total_sections.
12. require segment_count ≤ (block_count >> 9).
13. /* Multi-device path validation */
14. if RDEV(0).path[0]:
    - dev_seg_count = sum(le32(devs[i].total_segments) for i where path[0] ≠ 0).
    - require segment_count == dev_seg_count.
15. else if has_blkzoned ∧ !bdev_is_zoned(sb.s_bdev): -EFSCORRUPTED.
16. require secs_per_zone ∈ (0, total_sections].
17. require extension_count + hot_ext_count ≤ F2FS_MAX_EXTENSION.
18. require cp_payload < blocks_per_seg - F2FS_CP_PACKS - NR_CURSEG_PERSIST_TYPE.
19. require le32(rs.node_ino) == 1 ∧ le32(rs.meta_ino) == 2 ∧ le32(rs.root_ino) == 3.
20. F2fs::sanity_check_area_boundary(sbi, folio, index)?.
21. return Ok(()).

`F2fs::commit_super(sbi, recover) -> Result<(), Errno>`:
1. if (recover ∧ f2fs_readonly(sbi.sb)) ∨ f2fs_hw_is_readonly(sbi): set SBI_NEED_SB_WRITE; -EROFS.
2. if !recover ∧ has_sb_chksum: rs.crc = le32(f2fs_crc32(rs, offsetof(crc))).
3. /* Back-up first */
4. let back = if sbi.valid_super_block == 0 { 1 } else { 0 };
5. folio = read_mapping_folio(sb.s_bdev.bd_mapping, back, None)?.
6. F2fs::commit_super_inner(sbi, &folio, back, true)?.
7. folio_put.
8. if recover { return Ok(()); }
9. /* Primary */
10. let primary = sbi.valid_super_block.
11. folio = read_mapping_folio(sb.s_bdev.bd_mapping, primary, None)?.
12. F2fs::commit_super_inner(sbi, &folio, primary, true)?.

`F2fs::kill_super(sb)`:
1. let sbi = F2FS_SB(sb).
2. if sb.s_root.is_some():
   - sbi.umount_lock_holder = Some(current).
   - set_sbi_flag(sbi, SBI_IS_CLOSE).
   - F2fs::stop_gc_thread(sbi); F2fs::stop_discard_thread(sbi).
   - if F2FS_FS_COMPRESSION ∧ COMPRESS_CACHE: truncate_inode_pages_final(COMPRESS_MAPPING(sbi)).
   - if SBI_IS_DIRTY ∨ !CP_UMOUNT_FLAG: F2fs::write_checkpoint(sbi, &cp_control{ reason: CP_UMOUNT }).
   - if SBI_IS_RECOVERED ∧ f2fs_readonly(sb): sb.s_flags &= ~SB_RDONLY.
3. kill_block_super(sb).
4. if sbi.is_some():
   - destroy_device_list(sbi); lockdep_unregister_key; drop(sbi); sb.s_fs_info = None.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `magic_validated_before_use` | INVARIANT | per-sanity_check_raw_super: magic checked before any further rs deref. |
| `log_blocksize_eq_12` | INVARIANT | per-sanity_check_raw_super: log_blocksize == F2FS_BLKSIZE_BITS. |
| `area_boundary_monotone` | INVARIANT | per-sanity_check_area_boundary: cp < sit < nat < ssa < main_blkaddr. |
| `extension_count_bounded` | INVARIANT | per-sanity_check_raw_super: extension_count + hot_ext_count ≤ F2FS_MAX_EXTENSION. |
| `dual_sb_one_valid` | INVARIANT | per-read_raw_super_block: ret Ok ⟹ at least one of {block 0, block 1} validates. |
| `ckpt_version_monotone` | INVARIANT | per-get_valid_checkpoint: chosen ckpt has max cur_cp_version of two packs. |
| `por_doing_during_fill_super` | INVARIANT | per-fill_super: SBI_POR_DOING set before recovery; cleared at reset_checkpoint. |

### Layer 2: TLA+

`fs/f2fs/super.tla`:
- Variables: sb_state ∈ {Unmounted, ReadingSB, PickingCP, ScanDevs, BuildSM, BuildNM, Recovering, Mounted, KillingSb}, ckpt_pack ∈ {PackA, PackB}, ckpt_ver, cp_flags, sbi_flags.
- Properties:
  - `safety_cp_version_monotone` — per-mount: chosen ckpt.checkpoint_ver ≥ both on-disk packs' versions.
  - `safety_cp_double_buffer_atomic` — per-CP write: a crash mid-write leaves at least one valid pack of the two on disk.
  - `safety_recovery_only_when_not_disabled` — per-mount: roll-forward replay skipped iff CP_DISABLED_FLAG ∨ NORECOVERY ∨ DISABLE_ROLL_FORWARD.
  - `safety_RO_no_gc_no_ckpt` — per-RO-mount: no GC thread; no CP thread; no write attempts.
  - `safety_umount_writes_cp_umount_flag` — per-kill_super (non-readonly, clean): final CP pack has CP_UMOUNT_FLAG.
  - `safety_critical_error_records_reason` — per-handle_critical_error: stop_reason persisted before remount-ro / panic / shutdown.
  - `liveness_fill_super_terminates` — per-fill_super: Ok | Err (one retry on skip_recovery).
  - `liveness_get_valid_checkpoint_terminates` — per-CP-pick: terminates regardless of which pack is fresher.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `F2fs::sanity_check_raw_super` post: Ok ⟹ magic == 0xF2F52010 ∧ log_blocksize == 12 ∧ root_ino == 3 ∧ area boundaries strict-monotone | `F2fs::sanity_check_raw_super` |
| `F2fs::read_raw_super_block` post: Ok ⟹ valid_super_block ∈ {0, 1}; if recovery == 1, exactly one SB was bad | `F2fs::read_raw_super_block` |
| `F2fs::get_valid_checkpoint` post: sbi.ckpt.checkpoint_ver == max(pack0.ver, pack1.ver) | `F2fs::get_valid_checkpoint` |
| `F2fs::commit_super` post: writes back-up before primary; primary only written when back-up succeeded | `F2fs::commit_super` |
| `F2fs::fill_super` post: Ok ⟹ sb.s_root.is_some() ∧ root.i_mode S_ISDIR ∧ SBI_POR_DOING cleared | `F2fs::fill_super` |
| `F2fs::kill_super` post: SBI_IS_DIRTY ⟹ final CP written with CP_UMOUNT_FLAG; sb.s_fs_info == None | `F2fs::kill_super` |
| `F2fs::start_gc_thread` post: GC kthread runs only when !RO ∧ bggc_mode ≠ OFF | `F2fs::start_gc_thread` |

### Layer 4: Verus/Creusot functional

`Per-mount: read dual SB → pick valid → sanity-check → iget meta → pick freshest CP pack via cur_cp_version → scan devs → build SM/NM → iget root → recover orphan + roll-forward → start CP & GC threads → notice mount` semantic equivalence: per Documentation/filesystems/f2fs.rst + f2fs-tools `mkfs.f2fs`/`fsck.f2fs`/`dump.f2fs` on-disk reference; f2fs-tools-produced images byte-identical to Linux-mounted view (xfstests f2fs subset and cross-mount with upstream).

## Hardening

(Inherits row-1 features from `fs/f2fs/00-overview.md` § Hardening.)

F2FS-super reinforcement:

- **Per-dual-SB: at least one valid mandatory** — defense against per-single-sector-corruption mount failure.
- **Per-SB_CHKSUM CRC verified before deref** — defense against per-tampered-SB triggering OOB reads.
- **Per-log_blocksize == 12 exact** — defense against per-bogus-shift OOB block-IO.
- **Per-area boundary strict monotone (cp < sit < nat < ssa < main)** — defense against per-overlap-poisoning of metadata zones.
- **Per-segment_count_main == total_sections × segs_per_sec** — defense against per-mismatched-zone-counts UB.
- **Per-BLKZONED feature ⟹ zoned bdev** — defense against per-zoned-image-on-conventional-bdev corruption.
- **Per-multi-device Σ total_segments == segment_count** — defense against per-truncated-multidev mount.
- **Per-CP double-buffer atomic switchover** — defense against per-crash-mid-CP leaving no valid pack.
- **Per-cur_cp_version chooses freshest** — defense against per-stale-CP rollback.
- **Per-CP_UMOUNT_FLAG on clean umount** — defense against per-spurious-recovery on next mount.
- **Per-CP_FSCK_FLAG persistence** — defense against per-quiet-corruption (fsck always triggered on next mount).
- **Per-SBI_POR_DOING during fill_super** — defense against per-write-during-recovery races.
- **Per-cp_payload bound (< blocks_per_seg - F2FS_CP_PACKS - NR_CURSEG_PERSIST_TYPE)** — defense against per-CP overrun.
- **Per-extension_count + hot_ext_count ≤ F2FS_MAX_EXTENSION** — defense against per-extension OOB.
- **Per-RO feature blocks GC + CP threads** — defense against per-RO-image write attempt.
- **Per-try_onemore retry bounded (retry_cnt == 1)** — defense against per-recovery-loop DoS.
- **Per-stop_reason + errors persisted to on-disk SB** — defense against per-corruption-history-lost.
- **Per-DISABLE_CHECKPOINT explicit acknowledgement** — defense against per-silent-no-CP fsync semantics.
- **Per-NEED_FSCK propagated to next mount via CP_FSCK_FLAG** — defense against per-suppressed-corruption.
- **Per-write back-up SB first then primary in commit_super** — defense against per-double-faulted SB.

## Grsecurity/PaX-style Reinforcement

The f2fs superblock and checkpoint surfaces are a privileged write boundary: a forged CKPT pack can pivot the segment manager onto attacker-chosen sit/nat/ssa blocks. The following PaX/grsecurity-style controls augment the in-tree mount and CP defenses above.

- **PAX_USERCOPY** — `f2fs_sb_info`, `f2fs_checkpoint`, and `f2fs_nm_info` slabs declare zero-byte usercopy whitelists; sysfs and `f2fs_ioctl` paths that surface SB fields use bounded copies into stack scratch buffers.
- **PAX_KERNEXEC** — CKPT write-back, NAT/SIT bitmap flush, and feature-flag dispatch tables are placed in `__ro_after_init` text; W^X is enforced over the recovery-time fixup pages.
- **PAX_RANDKSTACK** — mount, remount, and the recovery worker enter with randomized kernel stacks so attempts to groom the `f2fs_sb_info` super.private locals are starved of layout knowledge.
- **PAX_REFCOUNT** — `sbi->s_active`, `cp_rwsem` waiter counts, and `nm_i->nat_cnt[]` use saturating atomics that BUG on wrap, closing UAF windows around `f2fs_put_super` and CP discard.
- **PAX_MEMORY_SANITIZE** — discarded CKPT pages, NAT/SIT journal entries, and freed `f2fs_io_info` records are poisoned so residual block addresses cannot be mined post-umount.
- **PAX_UDEREF** — every `f2fs_ioctl` opcode (defrag, gc, atomic write, compression) crosses the boundary under uderef, refusing forged pointers and stack-pivot tricks.
- **PAX_RAP / kCFI** — the per-feature dispatch (`f2fs_compress_ops`, `f2fs_quota_ops`, `f2fs_xattr_handler`) is typed-call-site-verified to defeat a corrupted feature flag steering execution.
- **GRKERNSEC_HIDESYM** — `__f2fs_*`, `do_checkpoint`, and the per-superblock `sbi->s_*` pointers are stripped from `/proc/kallsyms` for non-CAP_SYSLOG callers.
- **GRKERNSEC_DMESG** — `f2fs_handle_error`, `f2fs_warn`, and CP-fail prints are gated behind CAP_SYSLOG, denying unprivileged corruption oracles.
- **CKPT double-buffer signature** — both pack-0 and pack-1 are sealed with a HMAC tied to the on-disk SB UUID, and the recovery path refuses to roll forward on signature mismatch even when the magic and CRC validate.
- **F2FS_FEATURE_VERITY / ENCRYPT enforcement** — when these feature bits are set in the SB, the mount path makes them mandatory: any inode flag that would bypass verity/encrypt is refused with `EPERM` rather than silently downgraded.
- **Per-quota and per-namespace mount option lockdown** — `quota`, `inline_xattr`, `compress_mode`, and `atgc` flips after first mount require CAP_SYS_ADMIN in the init namespace and are denied for unprivileged user-namespace mounts.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- f2fs segment manager + GC kthread (covered in `segment-gc.md` Tier-3)
- f2fs node-block manager + NAT (covered in `node.md` Tier-3)
- f2fs checkpoint write path + recovery replay (covered in `checkpoint-recovery.md` Tier-3)
- f2fs data IO + extent cache (covered in `data.md` + `extent-inline.md` Tier-3)
- f2fs compression (covered in `compress.md` Tier-3)
- f2fs verity + fs-crypt integration (covered in `verity-crypt.md` Tier-3)
- f2fs xattr + ACL (covered in `xattr-acl.md` Tier-3)
- f2fs sysfs / debug / iostat (covered in `sysfs-debug-iostat.md` Tier-3)
- Implementation code
