---
title: "Tier-2: fs/btrfs — Btrfs filesystem (b-trees + extent + chunk + RAID + send/receive + qgroups + snapshots + subvols + scrub + balance + zoned + raid-stripe-tree + extent-tree-v2)"
tags: ["tier-2", "fs", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 wrapper for Btrfs — copy-on-write multi-device filesystem with snapshots, subvolumes, native RAID, send/receive replication, qgroups (per-subvol quotas), scrub (online checksum verification), balance, defrag, zoned-storage support (HM-zoned + HA-zoned), Direct-IO, fs-verity, fsverity, raid-stripe-tree (newer chunk-tree replacement). Default for SUSE/openSUSE + Fedora Workstation desktop installs since F33; deployed at scale on Facebook for tier-2 storage; backbone of every Synology Linux NAS. Components (Btrfs has ~150 source files): **on-disk format** (`accessors.c` + `accessors.h` + `ctree.h` + `btrfs_inode.h`: superblock + b-tree key encoding + accessors), **b-tree core** (`ctree.c` + `tree-mod-log.c`: COW b-tree + tree modification log), **extent tree** (`extent-tree.c` + `extent_io.c` + `extent_map.c` + `extent-tree-v2.c`: per-extent reference tracking + extent-tree-v2 new-style reference), **block-group + chunk** (`block-group.c` + `volumes.c` + `volumes.h`: chunk allocator + RAID0/1/5/6/10/1c3/1c4 stripping + dev-management), **inode + dir + file** (`inode.c` + `inode-item.c` + `dir-item.c` + `file.c` + `file-item.c`: per-inode/file/dir items in fs-tree), **subvolume + snapshot** (`subvolume.c` (in `super.c`/`ioctl.c`) + `root-tree.c`: subvol create + snapshot CoW + cross-subvol references), **transaction** (`transaction.c`: COW transaction commit), **send/receive** (`send.c`: stream replication for backup), **scrub** (`scrub.c`: online checksum verification + auto-repair via mirror), **balance + relocation** (`relocation.c`: balance + chunk relocation for `btrfs balance`), **delayed-ref** (`delayed-ref.c` + `delayed-inode.c`: defer ref count updates), **defrag** (`defrag.c`: online defrag), **compression** (`compression.c` + `lzo.c` + `zlib.c` + `zstd.c`: per-extent compression), **checksum** (`checksum.c` + per-extent CRC32C/XXHASH/SHA256/BLAKE2 dispatch), **qgroups** (`qgroup.c`: per-subvol quotas), **disk-io + IO** (`disk-io.c` + `bio.c` + `direct-io.c`: bio submission + read-repair), **dev-replace** (`dev-replace.c`: hot-replace device with rebuild), **discard** (`discard.c`: TRIM/discard tracking), **block-rsv** (`block-rsv.c` + `space-info.c`: per-purpose space reservation), **xattr + ACL** (`xattr.c` + `acl.c`), **export** (`export.c`: NFS export support), **ioctl** (`ioctl.c`: BTRFS_IOC_* user UAPI), **sysfs** (`sysfs.c`: `/sys/fs/btrfs/<UUID>/`), **fast-tests** (KUnit `tests/`), **lru-cache + free-space + free-space-tree** (`lru_cache.c` + `free-space-cache.c` + `free-space-tree.c`), **misc-virtual-tree** (`uuid-tree.c` + `verity.c` + `raid-stripe-tree.c` + `super.c` + `messages.c` + `print-tree.c` + `props.c` + `reflink.c` + `tree-checker.c` + `tree-log.c`).

### Acceptance Criteria

- [ ] AC-O1: `mkfs.btrfs` + `xfstests-bld auto -g auto -F btrfs` test suite passes with same pass/fail manifest as upstream baseline.
- [ ] AC-O2: cross-mount: btrfs created on upstream-built kernel mounts read+write on Rookery + vice versa.
- [ ] AC-O3: `btrfs subvolume snapshot` + `btrfs send` → `btrfs receive` round-trip works.
- [ ] AC-O4: RAID6 test: degraded-mount with 2 missing devices replays correctly + scrub repairs.
- [ ] AC-O5: Zoned-storage: SMR drive used as Btrfs single device works.
- [ ] AC-O6: qgroup limits enforced: subvol exceeding qgroup limit returns -EDQUOT.
- [ ] AC-O7: Compression: zstd-3 vs lzo vs no-compress benchmark within 5% of upstream.

### Out of Scope

- Implementation code
- 32-bit-only paths
- btrfs-progs userspace (separate project)

### compatibility contract — outline

- On-disk format byte-identical (every feature flag / incompat-flag / RW-feature / RO-feature supported; cross-mount with btrfs-progs userspace `btrfs check`, `btrfs subvolume`, `btrfs send/receive`).
- Mount options byte-identical: `compress=`, `compress-force=`, `space_cache=v2`, `commit=`, `nodatacow`, `nodatasum`, `nobarrier`, `metadata_ratio=`, `enospc_debug`, `discard=async`, `subvol=`, `subvolid=`, `flushoncommit`, `notreelog`, `recovery`, `nologreplay`, `usebackuproot`, `inode_cache`, `noinode_cache`, `clear_cache`, `degraded`, `device=`, `defrag`, `autodefrag`, `noautodefrag`, `barrier`, `noenospc_debug`, `commit=`, `enospc_debug`, `nossd`/`ssd`/`ssd_spread`, `nospace_cache`, `recovery`, `rescan_uuid_tree`, `skip_balance`, `subvolrootid=`, `thread_pool=`, `treelog`/`notreelog`, `user_subvol_rm_allowed`, `noinode_cache`.
- BTRFS_IOC_* (~70 ioctls) byte-identical: `BTRFS_IOC_SNAP_CREATE`, `_SNAP_CREATE_V2`, `_SUBVOL_CREATE`, `_SUBVOL_CREATE_V2`, `_SNAP_DESTROY`, `_SNAP_DESTROY_V2`, `_DEFRAG`, `_DEFRAG_RANGE`, `_RESIZE`, `_SCAN_DEV`, `_FORGET_DEV`, `_TRANS_START`, `_TRANS_END`, `_SYNC`, `_CLONE`, `_CLONE_RANGE`, `_TREE_SEARCH`, `_TREE_SEARCH_V2`, `_INO_LOOKUP`, `_INO_LOOKUP_USER`, `_DEFAULT_SUBVOL`, `_SPACE_INFO`, `_START_SYNC`, `_WAIT_SYNC`, `_DEV_INFO`, `_FS_INFO`, `_BALANCE`, `_BALANCE_V2`, `_BALANCE_CTL`, `_BALANCE_PROGRESS`, `_DEV_REPLACE`, `_DEV_REPLACE_V2`, `_GET_FEATURES`, `_SET_FEATURES`, `_GET_SUPPORTED_FEATURES`, `_RM_DEV`, `_RM_DEV_V2`, `_QUOTA_CTL`, `_QGROUP_ASSIGN`, `_QGROUP_CREATE`, `_QGROUP_LIMIT`, `_QUOTA_RESCAN`, `_QUOTA_RESCAN_STATUS`, `_QUOTA_RESCAN_WAIT`, `_SCRUB`, `_SCRUB_CANCEL`, `_SCRUB_PROGRESS`, `_SEND`, `_SUBVOL_GETFLAGS`, `_SUBVOL_SETFLAGS`, `_RECEIVED_SUBVOL`, `_LOGICAL_INO`, `_LOGICAL_INO_V2`, `_GET_DEV_STATS`, `_FILE_EXTENT_SAME`, `_GET_VERITY`, `_ENABLE_VERITY`, `_ENCODED_READ`, `_ENCODED_WRITE`.
- `/sys/fs/btrfs/<UUID>/` topology byte-identical: per-FS attrs (`label`, `metadata_uuid`, `nodesize`, `sectorsize`, `clone_alignment`, `quota_override`, `bg_reclaim_threshold`, `commit_stats`, `read_policy`, `temp_fsid`, `last_balance_log`, `discardable_bytes`, `discardable_extents`, `discard/{discardable_bytes, discardable_extents, max_discard_size, ...}`), per-bg-allocator + per-device + per-feature subdirs.
- Tracepoints `events/btrfs/` byte-identical names + format.

### tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `fs/btrfs/super.md` | `super.c` + `messages.c`: mount + sb operations |
| `fs/btrfs/ctree.md` | `ctree.c` + `accessors.c` + `print-tree.c`: COW b-tree + accessors |
| `fs/btrfs/extent-tree.md` | `extent-tree.c` + `extent-tree-v2.c` + `delayed-ref.c`: extent ref tracking + delayed-ref |
| `fs/btrfs/extent-io.md` | `extent_io.c` + `extent_map.c`: per-page state + extent map |
| `fs/btrfs/inode.md` | `inode.c` + `inode-item.c` + `btrfs_inode.h`: per-inode lifecycle |
| `fs/btrfs/file.md` | `file.c` + `file-item.c` + `direct-io.c` + `bio.c`: file I/O paths |
| `fs/btrfs/dir-item.md` | `dir-item.c`: directory items |
| `fs/btrfs/transaction.md` | `transaction.c`: COW transaction commit |
| `fs/btrfs/volumes.md` | `volumes.c` + `block-group.c`: chunk allocator + RAID + dev-mgmt |
| `fs/btrfs/raid-stripe-tree.md` | `raid-stripe-tree.c`: RAID-stripe-tree |
| `fs/btrfs/zoned.md` | `zoned.c`: zoned-storage support |
| `fs/btrfs/subvolume-snapshot.md` | `subvolume.c` + `root-tree.c`: subvol + snapshot |
| `fs/btrfs/send-receive.md` | `send.c`: send stream + receive |
| `fs/btrfs/scrub.md` | `scrub.c`: online checksum verification + auto-repair |
| `fs/btrfs/balance-relocation.md` | `relocation.c`: balance + chunk relocation |
| `fs/btrfs/dev-replace.md` | `dev-replace.c`: hot-replace |
| `fs/btrfs/discard.md` | `discard.c`: async TRIM |
| `fs/btrfs/qgroup.md` | `qgroup.c`: per-subvol quotas |
| `fs/btrfs/compression.md` | `compression.c` + `lzo.c` + `zlib.c` + `zstd.c`: per-extent compression |
| `fs/btrfs/checksum.md` | `checksum.c` + `tree-checker.c`: per-extent CRC32C/XXHASH/SHA256/BLAKE2 |
| `fs/btrfs/free-space.md` | `free-space-cache.c` + `free-space-tree.c` + `block-rsv.c` + `space-info.c` |
| `fs/btrfs/tree-log.md` | `tree-log.c`: per-fsync tree-log fast-path |
| `fs/btrfs/disk-io.md` | `disk-io.c`: bio submission + read-repair |
| `fs/btrfs/defrag.md` | `defrag.c`: online defrag |
| `fs/btrfs/reflink.md` | `reflink.c`: COW reflink (FICLONE / FICLONERANGE) |
| `fs/btrfs/verity.md` | `verity.c`: fs-verity integration |
| `fs/btrfs/xattr-acl.md` | `xattr.c` + `acl.c` + `props.c` |
| `fs/btrfs/ioctl.md` | `ioctl.c`: BTRFS_IOC_* dispatch |
| `fs/btrfs/sysfs.md` | `sysfs.c`: per-FS sysfs |
| `fs/btrfs/uuid-tree.md` | `uuid-tree.c` |
| `fs/btrfs/export.md` | `export.c`: NFS export |

### compatibility outline

- REQ-O1: On-disk format byte-identical (cross-mount with upstream-built kernel + btrfs-progs).
- REQ-O2: All mount options + ioctls byte-identical.
- REQ-O3: `/sys/fs/btrfs/<UUID>/` byte-identical.
- REQ-O4: `events/btrfs/` tracepoints byte-identical.
- REQ-O5: send/receive stream format byte-identical (cross-version replication works).
- REQ-O6: RAID0/1/10/1c3/1c4/5/6 + raid-stripe-tree all read+write correctly.
- REQ-O7: Zoned-storage HM-zoned + HA-zoned modes work on supported HW.
- REQ-O8: TLA+ models (transaction commit pipeline, b-tree COW + tree-mod-log lookup invariant, extent-tree concurrent reference update + delayed-ref dispatch, scrub vs in-flight write race, balance + concurrent file I/O, qgroup accounting under reflink/snapshot).
- REQ-O9: Hardening: per-extent CRC verified on every read; corrupt-detect → auto-repair via mirror or RAID parity; subvol-create + snapshot user-ACL via LSM hooks.

### verification

| TLA+ Model | Owner |
|---|---|
| `models/btrfs/transaction.tla` | `fs/btrfs/transaction.md` (proves: transaction commit pipeline — running → committing → committed; concurrent dirty-inode + commit kthread + new-transaction-start serialized correctly) |
| `models/btrfs/ctree_cow.tla` | `fs/btrfs/ctree.md` (proves: COW b-tree concurrent lookup + insert + delete; tree-mod-log allows older lookup to see consistent snapshot during in-flight COW) |
| `models/btrfs/extent_ref.tla` | `fs/btrfs/extent-tree.md` (proves: extent reference tracking under concurrent reflink + snapshot + write; delayed-ref dispatch correctly serializes ref-add and ref-drop) |
| `models/btrfs/scrub_io_race.tla` | `fs/btrfs/scrub.md` (proves: scrub vs in-flight write — scrub re-reads the same extent that's being written; checksum verifies post-write or scrub re-issues; never reports false-positive corruption) |
| `models/btrfs/qgroup_accounting.tla` | `fs/btrfs/qgroup.md` (proves: qgroup accounting under reflink + snapshot + quota-rescan; per-qgroup byte counts converge to consistent post-rescan totals) |

### hardening

| Feature | Default |
|---|---|
| **REFCOUNT** | per-fs_info + per-root + per-block_group + per-transaction refcounts use `Refcount` | § Mandatory |
| **CONSTIFY** | per-FS `super_operations`, `inode_operations`, `file_operations` `static const` | § Mandatory |
| **SIZE_OVERFLOW** | per-extent length + per-bg used arithmetic checked | § Mandatory |
| **MEMORY_SANITIZE** | freed inode + xattr cleared (carries fs-crypt keys, ACL) | § Default-on configurable |

Btrfs-specific reinforcement: every read CRC-verified at extent level; corrupt-detect triggers mirror-read-repair (or RAID-parity rebuild for RAID5/6); subvol-create + snapshot user-ACL via LSM hooks (`security_inode_create` + `security_inode_link`); `dax` mode disabled (no fs-verity bypass); `device=` mount-time scan rate-limited (defense against device-id squatting attack).

