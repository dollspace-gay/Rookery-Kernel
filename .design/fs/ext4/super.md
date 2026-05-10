# Tier-3: fs/ext4/super.c — ext4 superblock + mount/remount

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: fs/ext4/00-overview.md
upstream-paths:
  - fs/ext4/super.c (~7606 lines)
  - fs/ext4/ext4.h
  - fs/ext4/ext4_jbd2.c
  - include/uapi/linux/ext4.h
-->

## Summary

ext4 superblock provides per-mount filesystem state: per-`ext4_sb_info` holds in-memory cached metadata (free blocks/inodes, block-group descriptors, journal handle, mount options, feature bits). Per-on-disk `struct ext4_super_block` is 1024 bytes at offset 1024 of block-device. Per-`ext4_fill_super` parses on-disk superblock + validates magic (0xEF53) + features (compat, incompat, ro_compat) + bg-descriptors + opens jbd2 journal + mounts root inode. Per-`ext4_remount` handles remount transitions (rw↔ro, journaling toggle). Critical for: most Linux distros' root-fs.

This Tier-3 covers `super.c` (~7606 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct ext4_sb_info` | per-mount in-memory state | `Ext4SbInfo` |
| `struct ext4_super_block` | on-disk format (1024 bytes) | `Ext4SuperBlock` |
| `ext4_fill_super()` | per-mount entry | `Ext4::fill_super` |
| `ext4_remount()` | per-remount | `Ext4::remount` |
| `ext4_put_super()` | per-umount | `Ext4::put_super` |
| `ext4_load_super()` | per-block-device superblock-read | `Ext4::load_super` |
| `ext4_setup_super()` | per-superblock validate | `Ext4::setup_super` |
| `ext4_calculate_overhead()` | per-overhead compute | `Ext4::calculate_overhead` |
| `ext4_commit_super()` | per-fsync write superblock | `Ext4::commit_super` |
| `ext4_validate_options()` | per-mount-option parse | `Ext4::validate_options` |
| `ext4_load_journal()` | per-mount load jbd2 | `Ext4::load_journal` |
| `ext4_register_li_request()` | per-lazy-init request | `Ext4::register_li_request` |
| `ext4_freeze()` / `ext4_unfreeze()` | per-freeze for snapshots | `Ext4::freeze` / `unfreeze` |
| `ext4_statfs()` | per-statfs() syscall | `Ext4::statfs` |
| `ext4_show_options()` | per-/proc/mounts | `Ext4::show_options` |
| `ext4_sync_fs()` | per-sync(2) | `Ext4::sync_fs` |
| `EXT4_SUPER_MAGIC` (0xEF53) | per-magic | UAPI |
| `EXT4_FEATURE_*` (COMPAT/INCOMPAT/RO_COMPAT) | per-feature-bitmap | UAPI |

## Compatibility contract

REQ-1: On-disk ext4_super_block (1024 bytes):
- s_magic (u16): 0xEF53.
- s_log_block_size, s_log_cluster_size (u32): per-fs block size log2.
- s_inodes_count, s_blocks_count_lo (u32).
- s_free_blocks_count_lo (u32).
- s_free_inodes_count (u32).
- s_first_data_block (u32).
- s_log_groups_per_flex (u8): flex-bg log2.
- s_feature_compat (u32): COMPAT-feature-bitmap.
- s_feature_incompat (u32): INCOMPAT-feature-bitmap.
- s_feature_ro_compat (u32): RO_COMPAT-feature-bitmap.
- s_uuid (16 bytes).
- s_volume_name (16 bytes).
- s_journal_inum (u32): journal inode # (typically 8).
- s_journal_dev (u32): external-journal device.
- s_def_mount_opts (u32): default mount-options.
- s_kbytes_written (u64): per-lifetime stat.
- s_orphan_file_inum (u32): orphan-file inode.
- s_first_error_time, s_last_error_time, s_first_error_block, s_last_error_block (u64): per-error history.
- s_csum_seed (u32) + s_metadata_csum (CRC32C): per-checksum.
- s_checksum (u32): superblock-itself CRC32C.

REQ-2: Feature bits:
- COMPAT: DIR_PREALLOC, IMAGIC_INODES, HAS_JOURNAL, EXT_ATTR, RESIZE_INO, DIR_INDEX, SPARSE_SUPER2, ORPHAN_FILE.
- INCOMPAT: FILETYPE, RECOVER, JOURNAL_DEV, META_BG, EXTENTS, 64BIT, MMP, FLEX_BG, EA_INODE, INLINE_DATA, ENCRYPT, CASEFOLD, LARGEDIR.
- RO_COMPAT: SPARSE_SUPER, LARGE_FILE, BTREE_DIR, HUGE_FILE, GDT_CSUM, DIR_NLINK, EXTRA_ISIZE, QUOTA, BIGALLOC, METADATA_CSUM, READONLY, PROJECT, VERITY, ORPHAN_PRESENT.

REQ-3: ext4_fill_super(sb, fc, ...):
- ext4_load_super: read block 0 of device; validate magic.
- Validate feature-bits: per-INCOMPAT not-supported = -EOPNOTSUPP.
- Per-RO_COMPAT not-supported + !mount-ro = -EOPNOTSUPP.
- Compute fs.block_size = 1 << (s_log_block_size + 10).
- Init ext4_sb_info.
- Load block-group descriptors.
- ext4_load_journal: open jbd2.
- ext4_setup_super: mark fs not-clean (s_state |= EXT4_ERROR_FS); finalize.
- iget the root inode.
- d_make_root(root_inode).

REQ-4: ext4_remount(fc, flags):
- Acquire s_lock.
- Per-MS_RDONLY transition: flush journal; mark clean.
- Per-MS_RDWR: mark dirty.
- Update mount-options.

REQ-5: ext4_put_super(sb):
- ext4_unregister_li_request.
- ext4_flush_completed_IO.
- jbd2_journal_destroy.
- Flush superblock.
- Free ext4_sb_info.

REQ-6: ext4_validate_options:
- Per-fs_parameter_spec parse.
- Validate combinations (e.g. data=journal forbids dioread_nolock).

REQ-7: Per-error-handling:
- s_errors = panic / continue / remount-ro.
- On per-corrupt: ext4_error → action per s_errors.

REQ-8: ext4_freeze:
- Stop new writes; flush; commit journal.
- Used by LVM snapshots.

REQ-9: ext4_statfs:
- Returns per-fs stats: f_bsize, f_blocks, f_bfree, f_files, f_ffree, f_namelen, f_fsid.

REQ-10: Per-encryption / per-quota / per-verity:
- ENCRYPT enabled: per-inode encryption-policy.
- QUOTA enabled: per-uid/gid quota.
- VERITY enabled: per-file fs-verity Merkle-tree.

REQ-11: Per-sysfs:
- /sys/fs/ext4/<dev>/: per-mount attributes.
- mb_*, max_writeback_pages, etc.

## Acceptance Criteria

- [ ] AC-1: mount -t ext4 /dev/sda1 /mnt: superblock read; magic validates.
- [ ] AC-2: Unsupported INCOMPAT feature: mount fails -EOPNOTSUPP.
- [ ] AC-3: Unsupported RO_COMPAT + rw mount: fails; ro succeeds.
- [ ] AC-4: Journal-replay on mount: per-journal recovery.
- [ ] AC-5: ext4_statfs: per-blocks/files counts accurate.
- [ ] AC-6: mount -o remount,ro: writeable→ro transitions; journal committed.
- [ ] AC-7: mount -o data=ordered → journal-mode set.
- [ ] AC-8: umount: per-resources released.
- [ ] AC-9: Per-corrupt block detected: ext4_error → remount-ro per s_errors.
- [ ] AC-10: /sys/fs/ext4/<dev> attributes accessible.

## Architecture

Per-sb_info:

```
struct Ext4SbInfo {
  s_desc_size: u32,                                // group-desc size
  s_inodes_per_block: u32,
  s_blocks_per_group: u32,
  s_inodes_per_group: u32,
  s_itb_per_group: u32,
  s_gdb_count: u32,
  s_desc_per_block: u32,
  s_groups_count: u32,
  s_flex_groups: Vec<FlexGroups>,
  s_blockgroup_lock: BlockgroupLock,
  s_mount_opt: u64,
  s_mount_opt2: u32,
  s_mount_flags: u64,
  s_def_mount_opt: u64,
  s_uuid: [u8; 16],
  s_volume_name: [u8; 16],
  s_resgid: KgidT,
  s_resuid: KuidT,
  s_inode_readahead_blks: u32,
  s_mb_max_to_scan: u32,
  s_mb_stats: u32,
  s_journal: *JournalT,                             // jbd2 handle
  s_journal_bdev: *BlockDevice,
  s_li_request: *Ext4LiRequest,                     // lazy init
  s_es: *Ext4SuperBlock,                            // on-disk SB
  s_sbh: *BufferHead,                                // SB buffer
  s_group_desc: Vec<*BufferHead>,
  s_stripe: u32,
  s_inode_size: u32,
  s_first_ino: u32,
  s_chksum_driver: *CryptoShash,
  s_csum_seed: u32,
  s_es_lock: SpinLock,
  s_max_writeback_pages: u32,
  s_freeclusters_counter: PercpuCounter,
  s_freeinodes_counter: PercpuCounter,
  ...
}
```

`Ext4::fill_super(sb, fc) -> Result<()>`:
1. /* Load + validate superblock */
2. err = Ext4::load_super(sb, &ext4_sbi).
3. if !ext4_sbi.s_es.s_magic == EXT4_SUPER_MAGIC: return -EINVAL.
4. Ext4::check_features(sb, ext4_sbi, fc.flags).
5. Ext4::validate_options(sb, fc, ext4_sbi).
6. /* Load block-group descriptors */
7. ext4_load_group_descriptors(sb).
8. /* Load journal */
9. if EXT4_HAS_COMPAT_FEATURE(HAS_JOURNAL):
   - Ext4::load_journal(sb, fc).
10. Ext4::setup_super(sb).
11. /* Root inode */
12. root = iget_locked(sb, EXT4_ROOT_INO).
13. sb.s_root = d_make_root(root).

`Ext4::load_super(sb, &ext4_sbi) -> Result<()>`:
1. /* Read first block (offset 1024 = block 0 / block 1 depending on block size) */
2. bh = sb_bread(sb, log_block_size_to_first_sb_block).
3. ext4_sbi.s_es = bh.b_data + offset.
4. ext4_sbi.s_sbh = bh.

`Ext4::check_features(sb, ext4_sbi, mount_flags) -> Result<()>`:
1. es = ext4_sbi.s_es.
2. /* INCOMPAT */
3. unsupported = es.s_feature_incompat & ~EXT4_FEATURE_INCOMPAT_SUPP.
4. if unsupported: return -EOPNOTSUPP.
5. /* RO_COMPAT */
6. unsupported = es.s_feature_ro_compat & ~EXT4_FEATURE_RO_COMPAT_SUPP.
7. if unsupported ∧ !(mount_flags & MS_RDONLY): return -EOPNOTSUPP.

`Ext4::setup_super(sb)`:
1. es.s_mnt_count++.
2. es.s_state &= ~EXT4_VALID_FS.
3. ext4_commit_super(sb).

`Ext4::remount(fc, flags) -> Result<()>`:
1. ext4_sbi = EXT4_SB(sb).
2. /* Parse new options */
3. Ext4::parse_options(fc, ext4_sbi, &new_opt).
4. /* RO/RW transition */
5. if rw → ro:
   - jbd2_journal_lock_updates; commit.
   - ext4_setup_super_writeback (mark clean).
6. if ro → rw:
   - Ext4::load_journal.
   - mark dirty.

`Ext4::statfs(dentry, buf) -> Result<()>`:
1. sb = dentry.d_sb.
2. ext4_sbi = EXT4_SB(sb).
3. buf.f_type = EXT4_SUPER_MAGIC.
4. buf.f_bsize = sb.s_blocksize.
5. buf.f_blocks = ext4_blocks_count(ext4_sbi.s_es).
6. buf.f_bfree = percpu_counter_sum_positive(&ext4_sbi.s_freeclusters_counter).
7. buf.f_files = ext4_sbi.s_es.s_inodes_count.
8. buf.f_ffree = percpu_counter_sum_positive(&ext4_sbi.s_freeinodes_counter).
9. buf.f_namelen = EXT4_NAME_LEN.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `magic_matches` | INVARIANT | post-load_super: s_magic == EXT4_SUPER_MAGIC. |
| `block_size_power_of_2` | INVARIANT | sb.s_blocksize = 1 << log; ∈ {1024, 2048, 4096, 65536}. |
| `feature_incompat_supported` | INVARIANT | per-mount-rw: s_feature_incompat ⊆ EXT4_FEATURE_INCOMPAT_SUPP. |
| `groups_count_consistent` | INVARIANT | ext4_sbi.s_groups_count == ceil(blocks_count / blocks_per_group). |
| `journal_inode_valid` | INVARIANT | per-HAS_JOURNAL: s_journal_inum valid + journal loaded. |

### Layer 2: TLA+

`fs/ext4/super.tla`:
- Per-mount load + feature-check + journal-load.
- Properties:
  - `safety_no_mount_unsupported_features` — per-INCOMPAT not-supported ⟹ -EOPNOTSUPP.
  - `safety_journal_loaded_iff_HAS_JOURNAL` — per-mount with HAS_JOURNAL ⟹ jbd2 journal.
  - `liveness_per_remount_consistent` — per-rw↔ro: state transitioned correctly.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Ext4::fill_super` post: sb.s_root != NULL; sb.s_fs_info = ext4_sbi | `Ext4::fill_super` |
| `Ext4::check_features` post: returns OK iff features supported | `Ext4::check_features` |
| `Ext4::setup_super` post: s_state's VALID_FS cleared; mnt_count++ | `Ext4::setup_super` |
| `Ext4::statfs` post: per-stat fields populated | `Ext4::statfs` |
| `Ext4::remount` post: per-transition state correct | `Ext4::remount` |

### Layer 4: Verus/Creusot functional

`Per-mount on-disk superblock → in-memory ext4_sb_info; per-feature gating; per-journal opened for HAS_JOURNAL filesystems` semantic equivalence: per-ext4 on-disk format spec.

## Hardening

(Inherits row-1 features from `fs/ext4/00-overview.md` § Hardening.)

ext4-super reinforcement:

- **Per-magic validated** — defense against per-misidentification.
- **Per-INCOMPAT enforced** — defense against per-fs-corruption from old kernel.
- **Per-RO_COMPAT rw-block** — defense against per-write-incompatible fs corruption.
- **Per-csum CRC32C verified** — defense against per-bit-rot in superblock.
- **Per-mount validates groups_count** — defense against per-overflow in alloc.
- **Per-options parsed strictly** — defense against per-malformed-option crash.
- **Per-journal-replay required iff dirty** — defense against per-stale-journal corruption.
- **Per-error policy honored** — defense against per-corrupt continued mutation.
- **Per-freeze locks updates** — defense against per-snapshot inconsistent.
- **Per-sysfs CAP_SYS_ADMIN** — defense against unprivileged tweak.
- **Per-superblock-checksum integrity** — defense against per-mount-time corruption attack.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- fs/ext4/{inode, balloc, ialloc, namei, file, dir, extents, mballoc, page-io, sysfs, ...}.c (covered separately if expanded)
- jbd2 journaling layer (covered in `fs/jbd2.md` separately)
- e2fsprogs userspace (out-of-tree)
- Implementation code
