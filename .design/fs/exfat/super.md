# Tier-3: fs/exfat/super.c — exFAT superblock and mount

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: fs/exfat/00-overview.md
upstream-paths:
  - fs/exfat/super.c (~943 lines)
  - fs/exfat/exfat_raw.h
  - fs/exfat/exfat_fs.h
-->

## Summary

The exFAT (Extended FAT) superblock owns mount-time validation of the **Partition Boot Record (PBR)** at LBA 0, the boot-region checksum (sectors 0..11), and the in-core layout shadow that the rest of the driver uses to translate cluster numbers into device sectors. Per-PBR: `struct boot_sector` carries fs_name "EXFAT   ", BIOS-parameter-block region (`must_be_zero` 53 bytes), `vol_length` (sectors), `fat_offset`/`fat_length` (per-FAT region), `clu_offset`/`clu_count` (cluster heap), `root_cluster` (start cluster of root directory), `sect_size_bits` (9..12 → 512..4096 B/sector), `sect_per_clus_bits` (0..25-sect_size_bits → cluster size up to 32 MiB), `num_fats` (1 or 2), `vol_flags` (VOLUME_DIRTY 0x0002 / MEDIA_FAILURE 0x0004). Per-cluster-chain: walked via the 32-bit FAT table at FAT1_start_sector (entries 0/1 reserved, EOF_CLUSTER = 0xFFFFFFFF, BAD_CLUSTER = 0xFFFFFFF7). Per-allocation: the **ALLOCATION_BITMAP** directory entry (type 0x81) names the first cluster of the bitmap covering clusters EXFAT_FIRST_CLUSTER..num_clusters; bit-set ⟹ allocated. Per-upcase: the **UPCASE_TABLE** entry (type 0x82) names a per-volume Unicode upcase mapping used for case-insensitive name compare. Per-directory: dentries are 32-byte slots packed cluster-stream; primary types `EXFAT_VOLUME` 0x83 / `EXFAT_FILE` 0x85 + secondary `EXFAT_STREAM` 0xC0 + N × `EXFAT_NAME` 0xC1 (15 UTF-16 chars each, max 255). Critical for: removable-media interop (SD/SDXC, USB sticks, camera storage), persistent-flag tracking, large-cluster (>4 GiB) file support.

This Tier-3 covers `fs/exfat/super.c` (~943 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct boot_sector` | per-PBR on-disk layout | `ExfatBootSector` |
| `struct exfat_sb_info` | per-sb in-core state | `ExfatSbInfo` |
| `struct exfat_inode_info` | per-inode in-core state | `ExfatInodeInfo` |
| `struct exfat_mount_options` | per-mount options | `ExfatMountOptions` |
| `struct exfat_chain` | per-cluster chain descriptor | `ExfatChain` |
| `exfat_fill_super()` | per-mount | `ExfatSb::fill_super` |
| `__exfat_fill_super()` | per-mount inner | `ExfatSb::fill_super_inner` |
| `exfat_read_boot_sector()` | per-PBR parse | `ExfatSb::read_boot_sector` |
| `exfat_verify_boot_region()` | per-boot-checksum verify | `ExfatSb::verify_boot_region` |
| `exfat_calibrate_blocksize()` | per-logical-sector resize | `ExfatSb::calibrate_blocksize` |
| `exfat_create_upcase_table()` | per-UPCASE_TABLE load | `ExfatSb::create_upcase_table` |
| `exfat_load_bitmap()` | per-ALLOCATION_BITMAP load | `ExfatSb::load_bitmap` |
| `exfat_count_used_clusters()` | per-bitmap-popcount | `ExfatSb::count_used_clusters` |
| `exfat_count_num_clusters()` | per-chain walk | `ExfatSb::count_num_clusters` |
| `exfat_read_root()` | per-root-inode init | `ExfatSb::read_root` |
| `exfat_set_vol_flags()` | per-vol_flags write-through | `ExfatSb::set_vol_flags` |
| `exfat_set_volume_dirty()` | per-mark dirty | `ExfatSb::set_volume_dirty` |
| `exfat_clear_volume_dirty()` | per-clear dirty | `ExfatSb::clear_volume_dirty` |
| `exfat_put_super()` | per-umount | `ExfatSb::put_super` |
| `exfat_statfs()` | per-statfs(2) | `ExfatSb::statfs` |
| `exfat_show_options()` | per-/proc/mounts | `ExfatSb::show_options` |
| `exfat_force_shutdown()` | per-FS shutdown | `ExfatSb::force_shutdown` |
| `exfat_alloc_inode()` / `exfat_free_inode()` | per-inode slab | `ExfatSb::alloc_inode` / `free_inode` |
| `exfat_parse_param()` | per-mount-option parse | `ExfatSb::parse_param` |
| `exfat_reconfigure()` | per-remount | `ExfatSb::reconfigure` |
| `exfat_init_fs_context()` | per-fs_context init | `ExfatSb::init_fs_context` |
| `exfat_get_tree()` | per-get_tree_bdev | `ExfatSb::get_tree` |
| `exfat_kill_sb()` | per-kill_block_super + delayed-free | `ExfatSb::kill_sb` |
| `exfat_hash_init()` | per-inode-hashtable init | `ExfatSb::hash_init` |
| `BOOT_SIGNATURE` (0xAA55) | per-PBR validity | shared |
| `EXBOOT_SIGNATURE` (0xAA550000) | per-extended-boot validity | shared |
| `EXFAT_BITMAP` (0x81) / `EXFAT_UPCASE` (0x82) | per-primary system dentries | shared |
| `EXFAT_FILE` (0x85) / `EXFAT_STREAM` (0xC0) / `EXFAT_NAME` (0xC1) | per-file dentry triple | shared |
| `VOLUME_DIRTY` (0x0002) / `MEDIA_FAILURE` (0x0004) | per-vol_flags | shared |
| `EXFAT_RESERVED_CLUSTERS` (2) / `EXFAT_FIRST_CLUSTER` (2) | per-cluster-numbering | shared |
| `EXFAT_EOF_CLUSTER` (0xFFFFFFFF) / `EXFAT_BAD_CLUSTER` (0xFFFFFFF7) | per-FAT sentinels | shared |
| `EXFAT_MAX_NUM_CLUSTER` (0xFFFFFFF5) | per-spec cluster ceiling | shared |
| `ALLOC_FAT_CHAIN` (0x01) / `ALLOC_NO_FAT_CHAIN` (0x03) | per-chain mode | shared |

## Compatibility contract

REQ-1: struct boot_sector layout (512-byte PBR, first sector):
- jmp_boot[3] / fs_name[8] = "EXFAT   " / must_be_zero[53].
- partition_offset (u64 LE) / vol_length (u64 LE, sectors).
- fat_offset (u32 LE, sectors from PBR) / fat_length (u32 LE, sectors per FAT).
- clu_offset (u32 LE, cluster-heap start sector) / clu_count (u32 LE, total data clusters).
- root_cluster (u32 LE, root-dir start cluster).
- vol_serial (u32 LE) / fs_revision (u16 LE).
- vol_flags (u16 LE) — VOLUME_DIRTY | MEDIA_FAILURE | ...
- sect_size_bits (u8, 9..12) / sect_per_clus_bits (u8, 0..25 - sect_size_bits).
- num_fats (u8, 1 or 2) / drv_sel / percent_in_use / reserved[7].
- boot_code[390] / signature (u16 LE = 0xAA55).

REQ-2: struct exfat_sb_info fields used by super.c:
- options: exfat_mount_options.
- nls_io: per-non-UTF-8 charset NLS table.
- num_FAT_sectors / FAT1_start_sector / FAT2_start_sector / data_start_sector.
- num_sectors / num_clusters / root_dir / dentries_per_clu.
- sect_per_clus / sect_per_clus_bits / cluster_size / cluster_size_bits.
- vol_flags / vol_flags_persistent (retained across set: VOLUME_DIRTY | MEDIA_FAILURE).
- clu_srch_ptr (rolling cluster-allocator hint).
- used_clusters.
- boot_bh (cached PBR buffer_head).
- s_lock (sb mutex) / bitmap_lock (allocation-bitmap mutex).
- inode_hash_lock + inode_hashtable[EXFAT_HASH_SIZE] (per-(start_clu, entry) hash).
- ratelimit (per-error-log rate-limit).
- vol_type (FAT12/16/32 unused, always exFAT here).
- rcu (delayed-free head).
- s_exfat_flags (EXFAT_FLAGS_SHUTDOWN).

REQ-3: struct exfat_inode_info fields:
- dir (exfat_chain — start cluster + size + flags of containing dir).
- entry (dentry index within containing dir).
- start_clu / flags (ALLOC_FAT_CHAIN or ALLOC_NO_FAT_CHAIN).
- type (TYPE_FILE / TYPE_DIR / ...).
- attr / version / i_size_ondisk / i_size_aligned.
- truncate_lock (rw_semaphore — serializes truncate vs read).
- cache_lru / nr_caches / cache_valid_id / cache_lru_lock (per-fat-extent cache).
- hint_bmap / hint_stat / hint_femp (per-search hints).
- i_hash_fat (hlist node for sb hash).
- i_crtime (creation time).
- vfs_inode.

REQ-4: exfat_read_boot_sector(sb):
- sb_min_blocksize(sb, 512) → set initial block size.
- sbi.boot_bh = sb_bread(sb, 0). Cache PBR forever (released in put_super).
- p_boot = boot_bh.b_data.
- if le16_to_cpu(signature) != 0xAA55: -EINVAL.
- if memcmp(fs_name, "EXFAT   ", 8): -EINVAL.
- if memchr_inv(must_be_zero, 0, 53): -EINVAL (rejects FAT32 misidentification).
- if num_fats ∉ {1, 2}: -EINVAL.
- if sect_size_bits ∉ [9, 12]: -EINVAL.
- if sect_per_clus_bits > 25 - sect_size_bits: -EINVAL.
- Populate sbi from PBR:
  - sect_per_clus = 1 << sect_per_clus_bits.
  - cluster_size_bits = sect_per_clus_bits + sect_size_bits.
  - cluster_size = 1 << cluster_size_bits.
  - num_FAT_sectors = fat_length.
  - FAT1_start_sector = fat_offset.
  - FAT2_start_sector = fat_offset (+ num_FAT_sectors if num_fats == 2).
  - data_start_sector = clu_offset.
  - num_sectors = vol_length.
  - num_clusters = clu_count + EXFAT_RESERVED_CLUSTERS (2).
  - root_dir = root_cluster.
  - dentries_per_clu = 1 << (cluster_size_bits - DENTRY_SIZE_BITS) [DENTRY_SIZE_BITS = 5 → 32 B/dentry].
  - vol_flags = le16_to_cpu(p_boot.vol_flags).
  - vol_flags_persistent = vol_flags & (VOLUME_DIRTY | MEDIA_FAILURE).
  - clu_srch_ptr = EXFAT_FIRST_CLUSTER (2).
- Consistency:
  - if num_FAT_sectors << sect_size_bits < num_clusters * 4: -EINVAL (FAT too short for table).
  - if data_start_sector < FAT1_start_sector + num_FAT_sectors * num_fats: -EINVAL.
- if vol_flags & VOLUME_DIRTY: warn "Volume was not properly unmounted ... run fsck".
- if vol_flags & MEDIA_FAILURE: warn "Medium has reported failures".
- sb.s_maxbytes = min(MAX_LFS_FILESIZE, EXFAT_CLU_TO_B(EXFAT_MAX_NUM_CLUSTER, sbi)).
- exfat_calibrate_blocksize(sb, 1 << sect_size_bits) — bring sb block size up to logical sector if needed (release & re-read boot_bh).

REQ-5: exfat_verify_boot_region(sb):
- Sub-regions: sector 0 (PBR), sectors 1..8 (extended boot sectors), sector 9 (OEM parameters), sector 10 (reserved), sector 11 (boot checksum). 12 total.
- chksum = 0.
- for sn in 0..11:
  - bh = sb_bread(sb, sn).
  - if sn ∈ [1, 8] ∧ last 4 bytes != EXBOOT_SIGNATURE (0xAA550000): warn (non-fatal).
  - chksum = exfat_calc_chksum32(bh.b_data, blocksize, chksum, sn ? CS_DEFAULT : CS_BOOT_SECTOR).
    - CS_BOOT_SECTOR skips offsets 106 (vol_flags), 107, 112 (percent_in_use) per spec.
  - brelse(bh).
- bh = sb_bread(sb, 11).
- for i in 0..blocksize step sizeof(u32):
  - if le32_to_cpu(*(__le32 *)&bh.b_data[i]) != chksum: -EINVAL (whole sector must repeat the checksum).
- brelse(bh).

REQ-6: exfat_calibrate_blocksize(sb, logical_sect):
- if !is_power_of_2(logical_sect): -EIO.
- if logical_sect < sb.s_blocksize: -EIO.
- if logical_sect > sb.s_blocksize:
  - brelse(sbi.boot_bh).
  - if !sb_set_blocksize(sb, logical_sect): -EIO.
  - sbi.boot_bh = sb_bread(sb, 0). On NULL: -EIO.

REQ-7: __exfat_fill_super(sb, root_clu):
- ret = exfat_read_boot_sector(sb). On err: goto free_bh.
- ret = exfat_verify_boot_region(sb). On err: goto free_bh.
- exfat_chain_set(root_clu, sbi.root_dir, 0, ALLOC_FAT_CHAIN).
- ret = exfat_count_num_clusters(sb, root_clu, &root_clu.size).
  - Walks FAT chain starting at root_dir; detects loops via cluster count > sbi.num_clusters.
- ret = exfat_create_upcase_table(sb).
  - Scans root directory for EXFAT_UPCASE (0x82) dentry; reads upcase data via FAT chain; checksum-validates; loads into sbi.vol_utbl (UTF-16 → upper-case-UTF-16 map).
- ret = exfat_load_bitmap(sb).
  - Scans root directory for EXFAT_BITMAP (0x81) dentry; reads allocation-bitmap bytes via FAT chain (1 bit per data cluster).
- if !exfat_test_bitmap(sb, sbi.root_dir):
  - warn; exfat_set_bitmap(sb, sbi.root_dir, false) (recover — first cluster of root must be allocated).
- ret = exfat_count_used_clusters(sb, &sbi.used_clusters).
  - Population-count over allocation-bitmap.
- On any error: free_alloc_bitmap (exfat_free_bitmap) → free_bh (brelse sbi.boot_bh).

REQ-8: exfat_fill_super(sb, fc):
- if opts.allow_utime == (unsigned short)-1: opts.allow_utime = ~opts.fs_dmask & 0022.
- if opts.discard ∧ !bdev_max_discard_sectors(sb.s_bdev): warn; opts.discard = 0.
- sb.s_flags |= SB_NODIRATIME.
- sb.s_magic = EXFAT_SUPER_MAGIC (0x2011BAB0).
- sb.s_op = &exfat_sops.
- sb.s_time_gran = 10 * NSEC_PER_MSEC (exFAT timestamp granularity).
- sb.s_time_min/max = EXFAT_MIN_TIMESTAMP_SECS / EXFAT_MAX_TIMESTAMP_SECS (1980-01-01 .. 2107-12-31).
- err = __exfat_fill_super(sb, &root_clu). On err: goto check_nls_io.
- exfat_hash_init(sb): spin_lock_init + INIT_HLIST_HEAD × EXFAT_HASH_SIZE.
- if opts.utf8: set_default_d_op(sb, &exfat_utf8_dentry_ops).
- else: sbi.nls_io = load_nls(opts.iocharset); set_default_d_op(sb, &exfat_dentry_ops).
- root_inode = new_inode(sb).
- root_inode.i_ino = EXFAT_ROOT_INO.
- inode_set_iversion(root_inode, 1).
- err = exfat_read_root(root_inode, &root_clu).
- exfat_hash_inode(root_inode, i_pos) + insert_inode_hash.
- sb.s_root = d_make_root(root_inode).

REQ-9: exfat_read_root(inode, root_clu):
- exfat_chain_set(&ei.dir, sbi.root_dir, 0, ALLOC_FAT_CHAIN).
- ei.entry = -1 (root has no parent dentry).
- ei.start_clu = sbi.root_dir.
- ei.flags = ALLOC_FAT_CHAIN.
- ei.type = TYPE_DIR.
- ei.version = 0.
- ei.hint_bmap.off = EXFAT_EOF_CLUSTER.
- ei.hint_stat.{eidx, clu} = {0, sbi.root_dir}.
- ei.hint_femp.eidx = EXFAT_HINT_NONE.
- i_size = EXFAT_CLU_TO_B(root_clu.size, sbi).
- num_subdirs = exfat_count_dir_entries(sb, root_clu). On <0: -EIO.
- set_nlink(inode, num_subdirs + EXFAT_MIN_SUBDIR) [+2 for "." and ".."].
- inode.i_uid/gid = opts.fs_uid/fs_gid.
- inode_inc_iversion + i_generation = 0.
- inode.i_mode = exfat_make_mode(sbi, EXFAT_ATTR_SUBDIR, 0777) (subject to opts.fs_dmask).
- inode.i_op = &exfat_dir_inode_operations.
- inode.i_fop = &exfat_dir_operations.
- inode.i_blocks = round_up(i_size, cluster_size) >> 9.
- ei.i_pos = ((u64)sbi.root_dir << 32) | 0xffffffff (encodes "root").
- exfat_save_attr(inode, EXFAT_ATTR_SUBDIR).
- ei.i_crtime = simple_inode_init_ts(inode).
- exfat_truncate_inode_atime(inode).

REQ-10: exfat_set_vol_flags(sb, new_flags):
- new_flags |= sbi.vol_flags_persistent (retain VOLUME_DIRTY | MEDIA_FAILURE).
- if sbi.vol_flags == new_flags: return 0 (no change).
- sbi.vol_flags = new_flags.
- if sb_rdonly(sb): return 0 (skip disk update).
- p_boot.vol_flags = cpu_to_le16(new_flags).
- set_buffer_uptodate(sbi.boot_bh).
- mark_buffer_dirty(sbi.boot_bh).
- __sync_dirty_buffer(sbi.boot_bh, REQ_SYNC | REQ_FUA | REQ_PREFLUSH) — durability barrier.
- Wrappers: exfat_set_volume_dirty (vol_flags |= VOLUME_DIRTY), exfat_clear_volume_dirty (vol_flags &= ~VOLUME_DIRTY).

REQ-11: Mount-option parameters (exfat_parameters):
- uid (kuid), gid (kgid).
- umask / dmask / fmask (octal — file/dir creation-mask).
- allow_utime (octal & 0022 — non-owner utime permissions).
- iocharset (string — non-UTF-8 NLS; "utf8" sets opts.utf8 = 1).
- errors (enum: continue / panic / remount-ro [default]).
- discard / nodiscard (TRIM after delete).
- keep_last_dots (preserve trailing dots in name).
- sys_tz (use system TZ for ondisk-local timestamps).
- time_offset (s32, ±24*60 minutes).
- zero_size_dir / nozero_size_dir.
- Deprecated (parsed-but-ignored): utf8, debug, namecase, codepage.

REQ-12: exfat_show_options(m, root):
- ,uid=N if != root.
- ,gid=N if != root.
- ,fmask=0NNN,dmask=0NNN always.
- ,allow_utime=0NNN if set.
- ,iocharset=utf8 if opts.utf8 else nls_io.charset.
- ,errors=continue / panic / remount-ro.
- ,discard if set.
- ,keep_last_dots / ,sys_tz / ,time_offset=N / ,zero_size_dir.

REQ-13: exfat_statfs(dentry, buf):
- buf.f_type = sb.s_magic.
- buf.f_bsize = sbi.cluster_size.
- buf.f_blocks = sbi.num_clusters - 2 (subtract reserved clusters 0 and 1).
- buf.f_bfree = f_blocks - sbi.used_clusters.
- buf.f_bavail = f_bfree.
- buf.f_fsid = u64_to_fsid(huge_encode_dev(sb.s_bdev.bd_dev)).
- buf.f_namelen = EXFAT_MAX_FILE_LEN (255) * NLS_MAX_CHARSET_SIZE.

REQ-14: exfat_reconfigure(fc):
- fc.sb_flags |= SB_NODIRATIME.
- sync_filesystem(sb).
- mutex_lock(sbi.s_lock); exfat_clear_volume_dirty(sb); mutex_unlock.
- if new_opts.allow_utime == (u16)-1: new_opts.allow_utime = ~new_opts.fs_dmask & 0022.
- Reject changes to options cached in inodes/dentries: iocharset, keep_last_dots, sys_tz, time_offset, fs_uid, fs_gid, fs_fmask, fs_dmask, allow_utime → -EINVAL.
- if new_opts.discard ∧ !bdev_max_discard_sectors(sb.s_bdev): -EINVAL.
- swap(*cur_opts, *new_opts).

REQ-15: exfat_put_super(sb):
- mutex_lock(sbi.s_lock).
- exfat_clear_volume_dirty(sb) (clean unmount → no VOLUME_DIRTY persisted).
- exfat_free_bitmap(sbi).
- brelse(sbi.boot_bh).
- mutex_unlock.

REQ-16: exfat_kill_sb(sb):
- kill_block_super(sb).
- if sbi: call_rcu(&sbi.rcu, delayed_free).
- delayed_free: unload_nls(nls_io) + exfat_free_upcase_table(sbi) + exfat_free_sbi (= exfat_free_iocharset + kfree).

REQ-17: exfat_force_shutdown(sb, flags):
- if EXFAT_FLAGS_SHUTDOWN already: return 0.
- EXFAT_GOING_DOWN_DEFAULT / FULLSYNC: bdev_freeze(sb.s_bdev); bdev_thaw; set EXFAT_FLAGS_SHUTDOWN.
- EXFAT_GOING_DOWN_NOSYNC: just set EXFAT_FLAGS_SHUTDOWN.
- else: -EINVAL.
- if opts.discard: opts.discard = 0 (disable post-shutdown).

REQ-18: super_operations (exfat_sops):
- alloc_inode = exfat_alloc_inode (alloc_inode_sb(sb, exfat_inode_cachep, GFP_NOFS) + init_rwsem truncate_lock).
- free_inode = exfat_free_inode.
- write_inode = exfat_write_inode.
- evict_inode = exfat_evict_inode.
- put_super = exfat_put_super.
- statfs = exfat_statfs.
- show_options = exfat_show_options.
- shutdown = exfat_shutdown (= exfat_force_shutdown NOSYNC).

## Acceptance Criteria

- [ ] AC-1: mount("exfat", dev): sb_bread(0) returns PBR with fs_name "EXFAT   " and signature 0xAA55; otherwise -EINVAL.
- [ ] AC-2: PBR with must_be_zero non-zero: rejected with -EINVAL (catches FAT32 misidentification).
- [ ] AC-3: sect_size_bits = 8: rejected with -EINVAL (below EXFAT_MIN_SECT_SIZE_BITS).
- [ ] AC-4: sect_per_clus_bits + sect_size_bits > 25: rejected with -EINVAL (>32 MiB clusters not allowed).
- [ ] AC-5: num_fats = 3: rejected (must be 1 or 2).
- [ ] AC-6: boot-checksum sector (sn=11) byte-mismatch: mount fails -EINVAL.
- [ ] AC-7: mount with logical-sector 4096 on initially-512 sb: blocksize raised; boot_bh re-read.
- [ ] AC-8: vol_flags & VOLUME_DIRTY: warn issued; mount continues; exfat_clear_volume_dirty on put_super clears.
- [ ] AC-9: exfat_set_vol_flags persists vol_flags_persistent (VOLUME_DIRTY|MEDIA_FAILURE) across set/clear cycles.
- [ ] AC-10: statfs returns f_blocks = num_clusters - 2; f_bfree = f_blocks - used_clusters; f_bsize = cluster_size.
- [ ] AC-11: readdir of root directory: enumerates entries via FAT chain walk starting at root_cluster.
- [ ] AC-12: ALLOCATION_BITMAP (0x81) absent from root directory: mount fails -EIO (loaded by exfat_load_bitmap).
- [ ] AC-13: UPCASE_TABLE (0x82) absent: mount fails -EIO (loaded by exfat_create_upcase_table).
- [ ] AC-14: First cluster of root_dir not set in bitmap: warn + auto-set (recovery).
- [ ] AC-15: remount with different iocharset: -EINVAL (cached in dentries).
- [ ] AC-16: remount discard with non-discard bdev: -EINVAL.

## Architecture

```
struct ExfatBootSector {
  jmp_boot: [u8; 3],
  fs_name: [u8; 8],                    // "EXFAT   "
  must_be_zero: [u8; 53],
  partition_offset: u64,               // LE
  vol_length: u64,                     // LE — sectors
  fat_offset: u32, fat_length: u32,    // LE — sectors
  clu_offset: u32, clu_count: u32,     // LE — sector / cluster
  root_cluster: u32,                   // LE
  vol_serial: u32,                     // LE
  fs_revision: u16,                    // LE
  vol_flags: u16,                      // LE — VOLUME_DIRTY | MEDIA_FAILURE | ...
  sect_size_bits: u8,                  // 9..12
  sect_per_clus_bits: u8,              // 0..25 - sect_size_bits
  num_fats: u8,                        // 1 or 2
  drv_sel: u8, percent_in_use: u8,
  reserved: [u8; 7],
  boot_code: [u8; 390],
  signature: u16,                      // LE — 0xAA55
}

struct ExfatSbInfo {
  options: ExfatMountOptions,
  nls_io: Option<*NlsTable>,
  num_FAT_sectors: u32,
  FAT1_start_sector: u32,
  FAT2_start_sector: u32,
  data_start_sector: u32,
  num_sectors: u64,
  num_clusters: u32,                   // includes the 2 reserved
  root_dir: u32,                       // first cluster of root
  dentries_per_clu: u32,
  sect_per_clus: u32,
  sect_per_clus_bits: u8,
  cluster_size: u32,
  cluster_size_bits: u8,
  vol_flags: u16,
  vol_flags_persistent: u16,           // VOLUME_DIRTY | MEDIA_FAILURE
  clu_srch_ptr: u32,
  used_clusters: u32,
  boot_bh: *BufferHead,
  s_lock: Mutex<()>,
  bitmap_lock: Mutex<()>,
  inode_hash_lock: SpinLock,
  inode_hashtable: [HlistHead; EXFAT_HASH_SIZE],
  ratelimit: RatelimitState,
  rcu: RcuHead,
  s_exfat_flags: AtomicU64,            // EXFAT_FLAGS_SHUTDOWN
  vol_utbl: *Upcase,                   // upcase table
  bitmap: *Bitmap,                     // allocation-bitmap pages
}

struct ExfatInodeInfo {
  dir: ExfatChain,
  entry: i32,
  start_clu: u32,
  flags: u8,                           // ALLOC_FAT_CHAIN | ALLOC_NO_FAT_CHAIN
  type: u8,                            // TYPE_FILE / TYPE_DIR / ...
  attr: u16,                           // EXFAT_ATTR_*
  version: u64,
  i_size_ondisk: i64,
  i_size_aligned: i64,
  truncate_lock: RwSemaphore,
  cache_lru: ListHead,
  cache_lru_lock: SpinLock,
  nr_caches: u32,
  cache_valid_id: u32,
  hint_bmap: ExfatHintBmap,
  hint_stat: ExfatHintStat,
  hint_femp: ExfatHintFemp,
  i_hash_fat: HlistNode,
  i_pos: i64,
  i_crtime: Timespec64,
  vfs_inode: Inode,
}

struct ExfatChain {
  dir: u32,                            // start cluster
  size: u32,                           // chain length in clusters
  flags: u8,                           // ALLOC_FAT_CHAIN | ALLOC_NO_FAT_CHAIN
}
```

`ExfatSb::fill_super(sb, fc) -> Result<()>`:
1. /* Defaults */
2. if opts.allow_utime == (u16)-1: opts.allow_utime = !opts.fs_dmask & 0o022.
3. if opts.discard ∧ !bdev_max_discard_sectors(sb.s_bdev): warn; opts.discard = 0.
4. /* VFS bookkeeping */
5. sb.s_flags |= SB_NODIRATIME.
6. sb.s_magic = EXFAT_SUPER_MAGIC.
7. sb.s_op = &EXFAT_SOPS.
8. sb.s_time_gran = 10_000_000 ns; s_time_min/max = exFAT bounds.
9. /* PBR + checksum + system tables */
10. ExfatSb::fill_super_inner(sb, &mut root_clu)?  /* __exfat_fill_super */
11. /* Hash table for inode lookups */
12. ExfatSb::hash_init(sb).
13. /* NLS */
14. if opts.utf8: set_default_d_op(&EXFAT_UTF8_DENTRY_OPS).
15. else:
    - sbi.nls_io = load_nls(opts.iocharset).
    - if !nls_io: -EINVAL.
    - set_default_d_op(&EXFAT_DENTRY_OPS).
16. /* Root inode */
17. root_inode = new_inode(sb).
18. root_inode.i_ino = EXFAT_ROOT_INO; iversion = 1.
19. ExfatSb::read_root(root_inode, &root_clu)?
20. exfat_hash_inode + insert_inode_hash.
21. sb.s_root = d_make_root(root_inode).

`ExfatSb::fill_super_inner(sb, root_clu) -> Result<()>`:
1. ExfatSb::read_boot_sector(sb)?
2. ExfatSb::verify_boot_region(sb)?
3. ExfatChain::set(root_clu, sbi.root_dir, 0, ALLOC_FAT_CHAIN).
4. ExfatSb::count_num_clusters(sb, root_clu, &mut root_clu.size)? /* FAT walk */
5. ExfatSb::create_upcase_table(sb)?
6. ExfatSb::load_bitmap(sb)?
7. /* Self-heal: root cluster must be marked allocated */
8. if !ExfatSb::test_bitmap(sb, sbi.root_dir):
   - warn.
   - ExfatSb::set_bitmap(sb, sbi.root_dir, false).
9. ExfatSb::count_used_clusters(sb, &mut sbi.used_clusters)?

`ExfatSb::read_boot_sector(sb) -> Result<()>`:
1. sb_min_blocksize(sb, 512).
2. sbi.boot_bh = sb_bread(sb, 0)?
3. p_boot = boot_bh.b_data as *ExfatBootSector.
4. if le16(p_boot.signature) != BOOT_SIGNATURE (0xAA55): -EINVAL.
5. if p_boot.fs_name != "EXFAT   ": -EINVAL.
6. if any byte of p_boot.must_be_zero is non-zero: -EINVAL.
7. if p_boot.num_fats ∉ {1, 2}: -EINVAL.
8. if !(EXFAT_MIN_SECT_SIZE_BITS ≤ sect_size_bits ≤ EXFAT_MAX_SECT_SIZE_BITS): -EINVAL.
9. if sect_per_clus_bits > 25 - sect_size_bits: -EINVAL.
10. Populate sbi fields (REQ-4).
11. Consistency:
    - if num_FAT_sectors << sect_size_bits < num_clusters * 4: -EINVAL.
    - if data_start_sector < FAT1_start_sector + num_FAT_sectors * num_fats: -EINVAL.
12. Warnings: VOLUME_DIRTY / MEDIA_FAILURE flags.
13. sb.s_maxbytes = min(MAX_LFS_FILESIZE, cluster_to_bytes(EXFAT_MAX_NUM_CLUSTER)).
14. ExfatSb::calibrate_blocksize(sb, 1 << sect_size_bits)?

`ExfatSb::verify_boot_region(sb) -> Result<()>`:
1. chksum: u32 = 0.
2. for sn in 0..11:
   - bh = sb_bread(sb, sn)?
   - if 1 ≤ sn ≤ 8:
     - check last 4 bytes == EXBOOT_SIGNATURE (0xAA550000) — warn-not-fail.
   - chksum = exfat_calc_chksum32(bh.b_data, sb.s_blocksize, chksum,
                                  if sn == 0 { CS_BOOT_SECTOR } else { CS_DEFAULT }).
   - brelse(bh).
3. bh = sb_bread(sb, 11)?  /* Boot-checksum sector — entire sector is repeating u32 */
4. for off in 0..blocksize step 4:
   - if le32(bh.b_data[off..]) != chksum: -EINVAL.
5. brelse(bh).

`ExfatSb::calibrate_blocksize(sb, logical_sect) -> Result<()>`:
1. if !is_power_of_2(logical_sect): -EIO.
2. if logical_sect < sb.s_blocksize: -EIO.
3. if logical_sect > sb.s_blocksize:
   - brelse(sbi.boot_bh).
   - if !sb_set_blocksize(sb, logical_sect): -EIO.
   - sbi.boot_bh = sb_bread(sb, 0).  if NULL: -EIO.

`ExfatSb::read_root(inode, root_clu) -> Result<()>`:
1. ExfatChain::set(&ei.dir, sbi.root_dir, 0, ALLOC_FAT_CHAIN).
2. ei.entry = -1; ei.start_clu = sbi.root_dir; ei.flags = ALLOC_FAT_CHAIN; ei.type = TYPE_DIR.
3. ei.hint_bmap.off = EXFAT_EOF_CLUSTER; ei.hint_stat = {0, sbi.root_dir}; ei.hint_femp.eidx = EXFAT_HINT_NONE.
4. i_size_write(inode, root_clu.size << sbi.cluster_size_bits).
5. num_subdirs = ExfatSb::count_dir_entries(sb, root_clu)?
6. set_nlink(inode, num_subdirs + EXFAT_MIN_SUBDIR).
7. inode.i_uid/i_gid = opts.fs_uid/fs_gid.
8. inode.i_mode = exfat_make_mode(EXFAT_ATTR_SUBDIR, 0o777).
9. inode.i_op = &EXFAT_DIR_INODE_OPERATIONS.
10. inode.i_fop = &EXFAT_DIR_OPERATIONS.
11. inode.i_blocks = round_up(i_size, cluster_size) >> 9.
12. ei.i_pos = ((u64)sbi.root_dir << 32) | 0xffffffff.
13. exfat_save_attr(inode, EXFAT_ATTR_SUBDIR).
14. ei.i_crtime = simple_inode_init_ts(inode).

`ExfatSb::set_vol_flags(sb, new_flags) -> Result<()>`:
1. new_flags |= sbi.vol_flags_persistent.
2. if sbi.vol_flags == new_flags: return Ok(()).
3. sbi.vol_flags = new_flags.
4. if sb_rdonly(sb): return Ok(()).
5. (*p_boot).vol_flags = cpu_to_le16(new_flags).
6. set_buffer_uptodate(sbi.boot_bh); mark_buffer_dirty(sbi.boot_bh).
7. __sync_dirty_buffer(sbi.boot_bh, REQ_SYNC | REQ_FUA | REQ_PREFLUSH).

`ExfatSb::put_super(sb)`:
1. mutex_lock(&sbi.s_lock).
2. ExfatSb::clear_volume_dirty(sb).
3. ExfatSb::free_bitmap(sbi).
4. brelse(sbi.boot_bh).
5. mutex_unlock(&sbi.s_lock).

`ExfatSb::kill_sb(sb)`:
1. kill_block_super(sb).
2. if sbi: call_rcu(&sbi.rcu, delayed_free).
3. delayed_free: unload_nls(nls_io); ExfatSb::free_upcase_table(sbi); ExfatSb::free_sbi(sbi).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `boot_signature_required` | INVARIANT | per-read_boot_sector: signature == 0xAA55 or -EINVAL. |
| `must_be_zero_strict` | INVARIANT | per-read_boot_sector: any non-zero byte in must_be_zero rejects. |
| `sect_size_bits_in_range` | INVARIANT | per-read_boot_sector: 9 ≤ sect_size_bits ≤ 12 or -EINVAL. |
| `cluster_bits_in_range` | INVARIANT | per-read_boot_sector: sect_per_clus_bits ≤ 25 - sect_size_bits or -EINVAL. |
| `num_fats_strict` | INVARIANT | per-read_boot_sector: num_fats ∈ {1, 2}. |
| `fat_length_consistent` | INVARIANT | per-read_boot_sector: num_FAT_sectors << sect_size_bits ≥ num_clusters * 4. |
| `data_start_after_fats` | INVARIANT | per-read_boot_sector: data_start ≥ FAT1_start + fat_length * num_fats. |
| `vol_flags_persistent_retained` | INVARIANT | per-set_vol_flags: (new | persistent) preserves VOLUME_DIRTY|MEDIA_FAILURE. |
| `boot_checksum_strict` | INVARIANT | per-verify_boot_region: every u32 of sector 11 == computed chksum. |
| `boot_bh_lifetime` | INVARIANT | per-fill_super: boot_bh held until put_super or error-unwind. |
| `read_only_skips_disk_flag_update` | INVARIANT | per-set_vol_flags: sb_rdonly ⟹ no disk write. |

### Layer 2: TLA+

`fs/exfat/super.tla`:
- Per-mount-state machine: INIT → READ_BOOT → VERIFY_CHK → COUNT_CHAIN → LOAD_UPCASE → LOAD_BITMAP → COUNT_USED → ROOT_INODE → MOUNTED.
- Per-vol_flags transition: CLEAN ⇌ DIRTY (set_volume_dirty on first write; clear on put_super).
- Properties:
  - `safety_mounted_implies_bitmap_loaded` — per-MOUNTED: sbi.bitmap != NULL.
  - `safety_mounted_implies_upcase_loaded` — per-MOUNTED: sbi.vol_utbl != NULL.
  - `safety_dirty_persists_on_crash` — per-set_volume_dirty: VOLUME_DIRTY observable after sync.
  - `safety_clean_unmount_clears_dirty` — per-put_super: VOLUME_DIRTY cleared (unless sb_rdonly).
  - `safety_blocksize_monotone` — per-calibrate_blocksize: only raises sb.s_blocksize, never lowers.
  - `liveness_mount_terminates` — per-fill_super: returns 0 or -EINVAL/-EIO in finite steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `ExfatSb::read_boot_sector` post: all sbi layout fields populated; chains consistent | `ExfatSb::read_boot_sector` |
| `ExfatSb::verify_boot_region` post: chksum_repeats_sector_11 == true | `ExfatSb::verify_boot_region` |
| `ExfatSb::calibrate_blocksize` post: sb.s_blocksize ∈ {old, logical_sect} | `ExfatSb::calibrate_blocksize` |
| `ExfatSb::fill_super_inner` post: sbi.bitmap and sbi.vol_utbl non-null on Ok | `ExfatSb::fill_super_inner` |
| `ExfatSb::set_vol_flags` post: sbi.vol_flags == (new_flags | persistent) | `ExfatSb::set_vol_flags` |
| `ExfatSb::read_root` post: ei.start_clu == sbi.root_dir; ei.type == TYPE_DIR | `ExfatSb::read_root` |
| `ExfatSb::statfs` post: f_bsize == cluster_size; f_blocks == num_clusters - 2 | `ExfatSb::statfs` |
| `ExfatSb::put_super` post: VOLUME_DIRTY cleared if !sb_rdonly | `ExfatSb::put_super` |
| `ExfatSb::reconfigure` post: cache-bound options unchanged or -EINVAL | `ExfatSb::reconfigure` |

### Layer 4: Verus/Creusot functional

`Per-mount: read PBR → verify boot-checksum (12 sectors) → walk root-cluster FAT chain → locate EXFAT_UPCASE (0x82) → load upcase table → locate EXFAT_BITMAP (0x81) → load allocation bitmap → count used clusters via popcount → set sbi.vol_flags |= VOLUME_DIRTY (on first write) → exfat_clear_volume_dirty on put_super` semantic equivalence: per-Microsoft exFAT File System Specification (2019) and `Documentation/filesystems/exfat.rst`.

## Hardening

(Inherits row-1 features from `fs/exfat/00-overview.md` § Hardening.)

exFAT-superblock reinforcement:

- **Per-fs_name strict equality "EXFAT   "** — defense against per-mount-wrong-filesystem.
- **Per-must_be_zero memchr_inv** — defense against per-FAT32-misidentification (FAT32 BPB shares offset).
- **Per-sect_size_bits range [9, 12]** — defense against per-corrupted-PBR overflow.
- **Per-sect_per_clus_bits ≤ 25 - sect_size_bits** — defense against per-cluster-size > 32 MiB DoS.
- **Per-num_fats ∈ {1, 2}** — defense against per-bogus-FAT-region overrun.
- **Per-fat-length-vs-cluster-count consistency** — defense against per-FAT-too-short cluster-write OOB.
- **Per-data_start-after-FATs check** — defense against per-overlapping-region corruption.
- **Per-12-sector boot-checksum (CRC-style chksum32)** — defense against per-silent-bitrot on PBR.
- **Per-EXBOOT_SIGNATURE check on sectors 1..8** — defense against per-truncated boot region.
- **Per-vol_flags_persistent retention** — defense against per-MEDIA_FAILURE accidental-clear.
- **Per-VOLUME_DIRTY on first write + clear on clean umount** — defense against per-unclean-umount silent-data-loss.
- **Per-__sync_dirty_buffer FUA+PREFLUSH on PBR write** — defense against per-vol_flags-write-not-durable.
- **Per-sb_rdonly skips disk vol_flags update** — defense against per-read-only-mount disk-write.
- **Per-root-cluster bitmap-bit self-heal** — defense against per-corrupted-bitmap unmount-bricking.
- **Per-FAT-chain loop detection (cluster count > num_clusters)** — defense against per-malicious cyclic chain infinite-loop.
- **Per-iocharset/keep_last_dots/sys_tz/time_offset reconfigure rejection** — defense against per-cached-dentry-stale.
- **Per-call_rcu delayed free of sbi/nls/upcase** — defense against per-late-reference UAF after kill_sb.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- fs/exfat/dir.c — directory iteration + dentry parse (separate Tier-3)
- fs/exfat/fatent.c — FAT-entry chain walk + cluster allocate/free (separate Tier-3)
- fs/exfat/balloc.c — allocation bitmap (separate Tier-3)
- fs/exfat/inode.c — inode read/write + iomap (separate Tier-3)
- fs/exfat/file.c — file_operations + truncate (separate Tier-3)
- fs/exfat/namei.c — lookup / create / unlink / rename (separate Tier-3)
- fs/exfat/nls.c — name-conversion + upcase compare (separate Tier-3)
- fs/exfat/cache.c — FAT-extent LRU cache (separate Tier-3)
- fs/exfat/misc.c — timestamp encode/decode + helpers
- Implementation code
