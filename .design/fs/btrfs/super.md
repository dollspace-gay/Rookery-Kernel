# Tier-3: fs/btrfs/super.c — btrfs superblock + mount + remount

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: fs/btrfs/00-overview.md
upstream-paths:
  - fs/btrfs/super.c (~2711 lines)
  - fs/btrfs/disk-io.c (per-fs init)
  - fs/btrfs/ctree.h
  - include/uapi/linux/btrfs.h
-->

## Summary

btrfs is a CoW (Copy-on-Write) filesystem with built-in snapshot + subvolumes + checksumming + RAID. Per-`btrfs_fs_info` holds per-fs in-memory state (trees, transactions, devices, raid-stripe info). Per-on-disk `struct btrfs_super_block` is 4096 bytes (replicated at multiple offsets). Per-`btrfs_init_fs_info` opens devices, reads superblock at primary location (64KB offset), validates magic (`_BHRfS_M`), opens trees (root tree, chunk tree, extent tree, fs tree). Per-mount supports per-subvolume (`subvol=`/`subvolid=`). Per-mount-options: compress, autodefrag, ssd, discard, etc. Critical for: modern Linux distros' default fs (Fedora, openSUSE), per-snapshot workflows, per-cross-device RAID.

This Tier-3 covers `super.c` (~2711 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct btrfs_fs_info` | per-fs in-memory state | `BtrfsFsInfo` |
| `struct btrfs_super_block` | on-disk format (4096 bytes) | `BtrfsSuperBlock` |
| `btrfs_init_fs_info()` | per-mount alloc | `Btrfs::init_fs_info` |
| `open_ctree()` | per-mount open trees | `Btrfs::open_ctree` |
| `close_ctree()` | per-umount close | `Btrfs::close_ctree` |
| `btrfs_mount()` / `btrfs_mount_root()` | per-mount entry | `Btrfs::mount` / `mount_root` |
| `btrfs_fill_super()` | per-vfs sb populate | `Btrfs::fill_super` |
| `btrfs_remount()` | per-remount | `Btrfs::remount` |
| `btrfs_statfs()` | per-statfs | `Btrfs::statfs` |
| `btrfs_show_options()` | per-/proc/mounts | `Btrfs::show_options` |
| `btrfs_get_subvol_name_from_objectid()` | per-subvol path | `Btrfs::get_subvol_name` |
| `btrfs_open_devices()` | per-multi-device | `Btrfs::open_devices` |
| `btrfs_freeze()` | per-freeze | `Btrfs::freeze` |
| `btrfs_validate_super()` | per-sb validate | `Btrfs::validate_super` |
| `BTRFS_MAGIC` (0x4D5F53664852425FULL or "_BHRfS_M") | per-magic | UAPI |
| `BTRFS_FEATURE_INCOMPAT_*` / `_COMPAT_RO_*` | per-feature-bitmap | UAPI |

## Compatibility contract

REQ-1: On-disk btrfs_super_block (4096 bytes; multiple copies):
- csum (32 bytes): per-superblock CRC32C / xxhash / sha256 / blake2.
- fsid (16 bytes): per-filesystem UUID.
- bytenr (u64): per-superblock disk offset.
- flags (u64).
- magic (u64): "_BHRfS_M".
- generation (u64): per-mount increment.
- root (u64): root-tree disk-pointer.
- chunk_root (u64): chunk-tree disk-pointer.
- log_root (u64): log-tree disk-pointer.
- log_root_transid (u64).
- total_bytes / bytes_used (u64).
- root_dir_objectid (u64): per-fs root dir.
- num_devices (u64): per-fs device count.
- sectorsize / nodesize / leafsize / stripesize (u32).
- sys_chunk_array_size (u32): per-system chunk inline array.
- chunk_root_generation (u64).
- compat_flags / compat_ro_flags / incompat_flags (u64): per-feature bitmaps.
- csum_type (u16): BTRFS_CSUM_TYPE_*.
- root_level (u8): root-tree depth.
- chunk_root_level (u8).
- log_root_level (u8).
- dev_item (per-device entry).
- label (256 chars).
- cache_generation (u64).
- uuid_tree_generation (u64).
- metadata_uuid (16 bytes).
- nr_global_roots (u8).
- sys_chunk_array (2048 bytes inline).

REQ-2: Per-superblock locations:
- Primary at 64KB.
- Secondary at 64MB.
- Tertiary at 256GB.
- All copies must validate.

REQ-3: Per-feature-incompat:
- MIXED_BACKREF, DEFAULT_SUBVOL, MIXED_GROUPS, COMPRESS_LZO, COMPRESS_ZSTD, BIG_METADATA, EXTENDED_IREF, RAID56, SKINNY_METADATA, NO_HOLES, METADATA_UUID, RAID1C34, ZONED, EXTENT_TREE_V2, RAID_STRIPE_TREE, SIMPLE_QUOTA.

REQ-4: btrfs_init_fs_info(fs_info):
- Allocate trees + locks + workqueues.
- Init transaction-list, delayed-iput-list.

REQ-5: open_ctree(fs_devices, ...):
- Read primary superblock.
- btrfs_validate_super: validate fields, magic, csum.
- Init chunk-tree (per-fs disk layout).
- Init root-tree.
- Init fs-tree (default subvolume).
- Init log-tree.
- Per-incompat-feature opens.
- Per-mount-option apply.

REQ-6: btrfs_fill_super(sb, fs_devices, ...):
- open_ctree.
- sb.s_root = d_make_root(root inode).
- sb.s_op = &btrfs_super_ops.

REQ-7: btrfs_mount(fs_type, flags, dev_name, raw_data):
- Per-subvol=/subvolid= mount-options: extract; mount root volume + traverse to subvol.

REQ-8: btrfs_remount:
- Per-rw↔ro: pause txns; flush; commit log-tree.

REQ-9: Per-features-incompat enforced:
- Unsupported INCOMPAT: -EOPNOTSUPP at mount.
- Unsupported COMPAT_RO: rw-mount fails; ro succeeds.

REQ-10: btrfs_validate_super:
- Magic check.
- csum-type ∈ {crc32c, xxhash, sha256, blake2}.
- csum verify.
- generation > 0.
- All offsets within disk size.

REQ-11: Per-fs flags:
- BTRFS_FS_OPEN, BTRFS_FS_BEEN_TRANSACTED, BTRFS_FS_QUOTA_ENABLED, BTRFS_FS_NEED_TRIM, etc.

## Acceptance Criteria

- [ ] AC-1: mount -t btrfs /dev/sda1 /mnt: superblock validated; trees opened.
- [ ] AC-2: Unsupported INCOMPAT: mount -EOPNOTSUPP.
- [ ] AC-3: mount -t btrfs -o subvol=foo: per-subvol mounted.
- [ ] AC-4: btrfs_statfs: per-bytes_used + total_bytes reported.
- [ ] AC-5: mount -o remount,ro: pauses transactions.
- [ ] AC-6: umount: trees closed; superblock commit.
- [ ] AC-7: Corrupt primary sb: secondary used.
- [ ] AC-8: Multi-device fs: per-dev superblock read.
- [ ] AC-9: btrfs_freeze: pauses txns for LVM snap.
- [ ] AC-10: /sys/fs/btrfs/<uuid>/: per-fs sysfs.

## Architecture

Per-fs_info:

```
struct BtrfsFsInfo {
  fs_devices: *BtrfsFsDevices,
  super_copy: *BtrfsSuperBlock,
  super_for_commit: *BtrfsSuperBlock,
  sb: *SuperBlock,
  tree_root: *BtrfsRoot,                          // root tree
  chunk_root: *BtrfsRoot,                         // chunk tree
  extent_root: *BtrfsRoot,                        // extent tree
  fs_root: *BtrfsRoot,                            // default subvol
  log_root: *BtrfsRoot,
  csum_root: *BtrfsRoot,
  uuid_root: *BtrfsRoot,
  global_roots_tree: RbRoot,
  fs_state: u64,                                  // BTRFS_FS_*
  flags: u64,
  fs_devices_lock: Mutex<()>,
  generation: AtomicU64,
  last_trans_committed: u64,
  running_transaction: *BtrfsTransaction,
  trans_lock: SpinLock,
  workers: WorkqueueStruct,
  delayed_workers: WorkqueueStruct,
  delalloc_workers: WorkqueueStruct,
  ...
}
```

Per-on-disk superblock:

```
struct BtrfsSuperBlock {
  csum: [u8; 32],
  fsid: [u8; 16],
  bytenr: __le64,
  flags: __le64,
  magic: __le64,                                  // 0x4D5F53664852425F
  generation: __le64,
  root: __le64,
  chunk_root: __le64,
  log_root: __le64,
  log_root_transid: __le64,
  total_bytes: __le64,
  bytes_used: __le64,
  root_dir_objectid: __le64,
  num_devices: __le64,
  sectorsize: __le32,
  nodesize: __le32,
  leafsize: __le32,
  stripesize: __le32,
  sys_chunk_array_size: __le32,
  chunk_root_generation: __le64,
  compat_flags: __le64,
  compat_ro_flags: __le64,
  incompat_flags: __le64,
  csum_type: __le16,
  root_level: u8,
  chunk_root_level: u8,
  log_root_level: u8,
  dev_item: BtrfsDevItem,
  label: [u8; 256],
  cache_generation: __le64,
  uuid_tree_generation: __le64,
  metadata_uuid: [u8; 16],
  nr_global_roots: u8,
  reserved: [u8; ...],
  sys_chunk_array: [u8; 2048],
  super_roots: [BtrfsRootBackup; 4],
  reserved2: [u8; 565],
}
```

`Btrfs::open_ctree(sb, fs_devices, options) -> Result<*BtrfsFsInfo>`:
1. fs_info = alloc + Btrfs::init_fs_info.
2. /* Read primary superblock */
3. bh = sb_bread(fs_devices.latest_bdev, BTRFS_SUPER_INFO_OFFSET / sectorsize).
4. fs_info.super_copy = bh.b_data.
5. /* Validate */
6. Btrfs::validate_super(fs_info.super_copy).
7. /* Check features */
8. unsupported_incompat = fs_info.super_copy.incompat_flags & ~BTRFS_FEATURE_INCOMPAT_SUPP.
9. if unsupported_incompat: return -EOPNOTSUPP.
10. /* Open chunk-tree → know disk layout */
11. fs_info.chunk_root = btrfs_read_tree_root(fs_info, ...).
12. /* Open root-tree */
13. fs_info.tree_root = btrfs_read_tree_root(fs_info, BTRFS_ROOT_TREE_OBJECTID).
14. /* Open fs-tree (default subvolume) */
15. fs_info.fs_root = btrfs_lookup_fs_root(fs_info, BTRFS_FS_TREE_OBJECTID).
16. /* Open extent / csum / uuid trees */
17. Return fs_info.

`Btrfs::validate_super(sb) -> Result<()>`:
1. if magic != BTRFS_MAGIC: return -EINVAL.
2. if csum_type > BTRFS_NR_CSUM_TYPES: return -EINVAL.
3. Compute checksum; compare against sb.csum.
4. if num_devices == 0: return -EINVAL.
5. if total_bytes == 0: return -EINVAL.

`Btrfs::mount(fs_type, flags, dev_name, raw_data) -> Result<*Dentry>`:
1. /* Per-options: parse subvol= / subvolid= */
2. fs_devices = btrfs_scan_one_device(dev_name).
3. /* Mount root volume */
4. dentry = mount_subvol(fs_type, flags, dev_name, raw_data, fs_devices).
5. /* Per-subvol traversal */
6. if subvol_name: traverse from root to subvol; return.

`Btrfs::fill_super(sb, fs_devices, data) -> Result<()>`:
1. fs_info = Btrfs::open_ctree(sb, fs_devices, data).
2. sb.s_fs_info = fs_info.
3. fs_info.sb = sb.
4. sb.s_op = &btrfs_super_ops.
5. inode = btrfs_iget(sb, BTRFS_FIRST_FREE_OBJECTID, fs_info.fs_root).
6. sb.s_root = d_make_root(inode).

`Btrfs::statfs(dentry, buf) -> Result<()>`:
1. fs_info = btrfs_sb(dentry.d_sb).
2. buf.f_type = BTRFS_SUPER_MAGIC.
3. buf.f_bsize = PAGE_SIZE.
4. buf.f_blocks = total_bytes / PAGE_SIZE.
5. buf.f_bfree = (total_bytes - bytes_used) / PAGE_SIZE.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `magic_matches` | INVARIANT | post-read: sb.magic == BTRFS_MAGIC. |
| `csum_validates` | INVARIANT | post-validate_super: computed_csum == sb.csum. |
| `incompat_supported` | INVARIANT | per-mount: incompat ⊆ FEATURE_INCOMPAT_SUPP. |
| `super_size_4kb` | INVARIANT | sizeof(BtrfsSuperBlock) == 4096. |
| `multi_dev_sb_consistent` | INVARIANT | per-multi-dev: all replicas matched. |

### Layer 2: TLA+

`fs/btrfs/super.tla`:
- Per-mount validate + open trees + per-feature gating.
- Properties:
  - `safety_no_mount_unsupported_incompat` — per-INCOMPAT not-in-supp ⟹ -EOPNOTSUPP.
  - `safety_csum_validates_per_mount` — per-mount-success: sb.csum matches.
  - `liveness_per_subvol_mountable` — per-subvol= option: dentry resolves.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Btrfs::open_ctree` post: tree_root + chunk_root + fs_root opened | `Btrfs::open_ctree` |
| `Btrfs::validate_super` post: returns Ok iff magic + csum + fields valid | `Btrfs::validate_super` |
| `Btrfs::fill_super` post: sb.s_root non-null; sb.s_fs_info set | `Btrfs::fill_super` |
| `Btrfs::statfs` post: per-stat fields populated | `Btrfs::statfs` |

### Layer 4: Verus/Creusot functional

`Per-mount on-disk superblock → per-tree validation → in-memory btrfs_fs_info; per-subvol = nested mount-point` semantic equivalence: per-btrfs on-disk format spec.

## Hardening

(Inherits row-1 features from `fs/btrfs/00-overview.md` § Hardening.)

btrfs-super reinforcement:

- **Per-magic + csum validated** — defense against per-misidentification + per-corruption.
- **Per-INCOMPAT enforced** — defense against per-feature mismatch.
- **Per-multi-dev sb consistency check** — defense against per-split-brain.
- **Per-options parsed strictly** — defense against per-malformed-option.
- **Per-trees opened RCU-protected** — defense against per-mount race.
- **Per-superblock secondary fallback** — defense against per-primary corruption.
- **Per-CAP_SYS_ADMIN for sysfs writes** — defense against per-unprivileged tweak.
- **Per-RAID feature gating** — defense against per-incomplete-config write-corruption.
- **Per-freeze pauses transactions** — defense against per-snapshot inconsistent.
- **Per-COW write-validation** — defense against per-checksum-fail data-loss.
- **Per-mount UUID per-fs unique** — defense against per-fsid collision.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- fs/btrfs/{tree-log, disk-io, transaction, ctree, extent_io, raid56, scrub, send, sysfs, qgroup, ...}.c (covered separately if expanded)
- btrfs-progs userspace (out-of-tree)
- Implementation code
