# Tier-2: fs/xfs — XFS filesystem (b+tree + AG + inode + extent + log + reflink + rmap + scrub + parent-pointers + zoned)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - fs/xfs/
  - include/uapi/linux/dqblk_xfs.h
  - include/uapi/linux/xfs.h
-->

## Summary

Tier-2 wrapper for XFS — high-performance journaling filesystem with native delayed-allocation, allocation-group parallelism, b+tree extent indexing, reflink (CoW shared extents), rmap (reverse-mapping for online repair), parent-pointers (parent-inode tracking for online repair), online scrub + fsck (xfs_scrub + xfs_repair), real-time subvolume, project quotas, growfs, fs-verity, fs-crypt (recently added), DAX, zoned-storage. Default for RHEL/CentOS Stream/Rocky/Alma + most enterprise + cloud installs since RHEL7. Two physical layouts: "v4" (older) + "v5" (CRC + reflink — current default since 5.15).

Components (XFS has ~120 source files): **userspace-shared library** (`libxfs/` — `xfs_alloc.c`, `xfs_alloc_btree.c`, `xfs_attr.c`, `xfs_attr_*.c`, `xfs_bmap.c`, `xfs_bmap_btree.c`, `xfs_btree.c`, `xfs_btree_staging.c`, `xfs_da_btree.c`, `xfs_da_format.c`, `xfs_defer.c`, `xfs_dir2.c`, `xfs_dir2_*.c`, `xfs_dquot_buf.c`, `xfs_format.h`, `xfs_health.c`, `xfs_iext_tree.c`, `xfs_inode_buf.c`, `xfs_inode_fork.c`, `xfs_log_format.h`, `xfs_log_recover.h`, `xfs_metafile.c`, `xfs_quota_defs.h`, `xfs_refcount.c`, `xfs_refcount_btree.c`, `xfs_rmap.c`, `xfs_rmap_btree.c`, `xfs_rtbitmap.c`, `xfs_rtgroup.c`, `xfs_rtrefcount_btree.c`, `xfs_rtrmap_btree.c`, `xfs_sb.c`, `xfs_shared.h`, `xfs_symlink_remote.c`, `xfs_trans_inode.c`, `xfs_trans_resv.c`, `xfs_trans_space.h`, `xfs_types.h` — shared with xfsprogs userspace), **kernel-only**: `xfs_super.c` (mount + sb), `xfs_aops.c` (address-space ops), `xfs_inode.c` + `xfs_iops.c` (inode ops), `xfs_buf.c` + `xfs_buf_item.c` (buffer cache), `xfs_log.c` + `xfs_log_recover.c` + `xfs_log_cil.c` (log + CIL committed-item-list), `xfs_trans.c` + `xfs_trans_*.c` (transaction), `xfs_bmap_util.c` + `xfs_bmap_item.c` (block-map util + bmap-update intent), `xfs_attr_*.c` (xattr), `xfs_acl.c`, `xfs_quota.c` + `xfs_qm.c` + `xfs_dquot.c`, `xfs_xattr.c`, `xfs_export.c` (NFS export), `xfs_extfree_item.c` (extent-free intent), `xfs_extent_busy.c` (busy-extent tracking), `xfs_filestream.c` (file-stream allocator), `xfs_fsmap.c` (`FS_IOC_GETFSMAP`), `xfs_fsops.c` (growfs etc.), `xfs_handle.c` (file-handle ops), `xfs_icache.c` (inode cache), `xfs_ioctl.c` + `xfs_ioctl32.c` (ioctl dispatch), `xfs_mount.c` (mount struct), `xfs_mru_cache.c` (most-recently-used cache), `xfs_notify_failure.c` (memory-failure notification), `xfs_pnfs.c` (pNFS layout server), `xfs_pwork.c` (parallel work), `xfs_qm_bhv.c` + `xfs_qm_syscalls.c` (quota syscall), `xfs_refcount_item.c` (refcount intent), `xfs_reflink.c` (reflink), `xfs_rmap_item.c` (rmap intent), `xfs_rtalloc.c` (real-time allocation), `xfs_stats.c` + `xfs_stats.h` + `xfs_sysctl.c` + `xfs_sysctl.h` + `xfs_sysfs.c` + `xfs_sysfs.h`, `xfs_symlink.c`, `xfs_trace.c` + `xfs_trace.h`, `xfs_xchgrange.c` (FIDEDUPERANGE+exchange), `xfs_zone_alloc.c` + `xfs_zone_*.c` (zoned-storage), **scrub** (`scrub/` — online scrub + repair via `xfs_scrub`).

## Compatibility contract — outline

- On-disk format byte-identical for v5 (default) — superblock + AG-headers + inode + b+tree pointers + log records (xfs-progs `xfs_db`, `xfs_repair`, `xfs_metadump`, `xfs_growfs`, `xfs_quota` consume).
- v4 (legacy, no CRC) preserved for backward-compat read.
- Mount options byte-identical: `allocsize=`, `attr2`/`noattr2`, `barrier`/`nobarrier`, `discard`/`nodiscard`, `dax`/`dax=always/never/inode`, `filestreams`, `grpid`/`bsdgroups`/`nogrpid`/`sysvgroups`, `inode32`/`inode64`, `largeio`/`nolargeio`, `logbsize=`, `logbufs=`, `logdev=`, `noalign`, `noattr2`, `noquota`, `norecovery`, `nouuid`, `pquota`, `prjquota`, `pqnoenforce`, `qnoenforce`, `quota`, `rtdev=`, `swalloc`, `sunit=`, `swidth=`, `swidth=`, `uquota`, `uqnoenforce`, `usrquota`, `wsync`.
- XFS_IOC_* ioctls (~50): `_FSGEOMETRY`, `_FSGEOMETRY_V4`, `_FSGEOMETRY_V5`, `_FSCOUNTS`, `_SET_RESBLKS`, `_GET_RESBLKS`, `_GROWFSDATA`, `_GROWFSLOG`, `_GROWFSRT`, `_FSGROWFSDATA`, `_GETBMAP`, `_GETBMAPA`, `_GETBMAPX`, `_FSCOUNTS_V2`, `_DIOINFO`, `_FSGETXATTR`, `_FSSETXATTR`, `_FSGETXATTRA`, `_GETXATTR`, `_PATH_TO_FSHANDLE`, `_PATH_TO_HANDLE`, `_FD_TO_HANDLE`, `_OPEN_BY_HANDLE`, `_READLINK_BY_HANDLE`, `_ATTRMULTI_BY_HANDLE`, `_ATTRLIST_BY_HANDLE`, `_FSSETDM_BY_HANDLE`, `_FSBULKSTAT`, `_FSBULKSTAT_SINGLE`, `_FSINUMBERS`, `_SCRUB_METADATA`, `_REPAIR_METADATA`, `_AG_GEOMETRY`, `_RTGROUP_GEOMETRY`, `_GETPARENTS`, `_GETPARENTS_BY_HANDLE`, `_EXCHANGE_RANGE`, `_FSCOMMITS`, `_GETFSREFCOUNTS`, `_FSGEOMETRY_V5_RT`, `_FREE_EOFBLOCKS`, `_FSCOMMIT_DEFAULT_VALUES`, `_GET_RESBLKS_V2`.
- `/sys/fs/xfs/<dev>/{stats,error,log,...}` byte-identical.
- `/proc/sys/fs/xfs/{stats_clear,xfssyncd_centisecs,xfsbufd_centisecs,xfsbufd_age_centisecs,filestream_centisecs,inherit_sync,inherit_nodump,inherit_noatime,inherit_nosymlinks,inherit_nodefrag,rotorstep,error_level,panic_mask,irix_symlink_mode,irix_sgid_inherit}` byte-identical.
- `events/xfs/` tracepoints byte-identical (huge: ~600+ tracepoints).

## Tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `fs/xfs/super.md` | `xfs_super.c` + `xfs_mount.c`: mount + sb |
| `fs/xfs/aops.md` | `xfs_aops.c`: address-space ops |
| `fs/xfs/inode.md` | `xfs_inode.c` + `xfs_iops.c` + `xfs_icache.c`: inode lifecycle + cache |
| `fs/xfs/buf.md` | `xfs_buf.c` + `xfs_buf_item.c`: buffer cache + buf-item |
| `fs/xfs/log.md` | `xfs_log.c` + `xfs_log_recover.c` + `xfs_log_cil.c`: log + CIL |
| `fs/xfs/trans.md` | `xfs_trans.c` + `xfs_trans_*.c`: transaction |
| `fs/xfs/bmap.md` | `libxfs/xfs_bmap.c` + `xfs_bmap_util.c` + `xfs_bmap_item.c`: block-map + bmap intent |
| `fs/xfs/alloc.md` | `libxfs/xfs_alloc.c` + `xfs_alloc_btree.c` + `xfs_extent_busy.c`: AG block allocator |
| `fs/xfs/dir.md` | `libxfs/xfs_dir2*.c`: directory format + ops |
| `fs/xfs/attr.md` | `libxfs/xfs_attr*.c` + `xfs_xattr.c` + `xfs_acl.c`: xattr + ACL |
| `fs/xfs/quota.md` | `xfs_qm.c` + `xfs_quota.c` + `xfs_dquot.c` + `xfs_qm_*.c`: quota |
| `fs/xfs/rmap.md` | `libxfs/xfs_rmap.c` + `xfs_rmap_item.c`: reverse-mapping b+tree |
| `fs/xfs/refcount-reflink.md` | `libxfs/xfs_refcount.c` + `xfs_refcount_item.c` + `xfs_reflink.c`: refcount + reflink |
| `fs/xfs/rt.md` | `libxfs/xfs_rtbitmap.c` + `xfs_rtgroup.c` + `xfs_rtalloc.c` + `libxfs/xfs_rtrefcount_btree.c` + `xfs_rtrmap_btree.c`: real-time subvolume |
| `fs/xfs/zoned.md` | `xfs_zone_alloc.c` + `xfs_zone_*.c`: zoned-storage |
| `fs/xfs/scrub.md` | `scrub/`: online scrub + repair |
| `fs/xfs/fsmap.md` | `xfs_fsmap.c`: `FS_IOC_GETFSMAP` |
| `fs/xfs/fsops.md` | `xfs_fsops.c`: growfs |
| `fs/xfs/export.md` | `xfs_export.c`: NFS export |
| `fs/xfs/pnfs.md` | `xfs_pnfs.c`: pNFS layout server |
| `fs/xfs/parent-pointers.md` | `libxfs/xfs_parent.c`: parent-pointers (online repair) |
| `fs/xfs/sysfs.md` | `xfs_sysfs.c`: per-FS sysfs |
| `fs/xfs/sysctl.md` | `xfs_sysctl.c`: `/proc/sys/fs/xfs/...` |
| `fs/xfs/notify-failure.md` | `xfs_notify_failure.c`: DAX memory-failure |
| `fs/xfs/xchgrange.md` | `xfs_xchgrange.c`: FIDEDUPERANGE + atomic-swap-range |
| `fs/xfs/ioctl.md` | `xfs_ioctl.c` + `xfs_ioctl32.c`: ioctl dispatch |

## Compatibility outline

- REQ-O1: On-disk format v5 (default) byte-identical (cross-mount with upstream-built kernel + xfsprogs).
- REQ-O2: All mount options + ioctls byte-identical.
- REQ-O3: `/sys/fs/xfs/<dev>/` + `/proc/sys/fs/xfs/` byte-identical.
- REQ-O4: `events/xfs/` tracepoints byte-identical.
- REQ-O5: Online scrub + repair via `xfs_scrub` works.
- REQ-O6: pNFS layout server functions correctly.
- REQ-O7: Reflink + rmap + parent-pointers all work.
- REQ-O8: Zoned-storage (HM-zoned + HA-zoned) works.
- REQ-O9: TLA+ models (CIL log-commit pipeline + intent-deferred-completion + AG-busy-extent + reflink share+unshare race + log-recovery replay-correctness).
- REQ-O10: Hardening: per-extent intent-replay correctness; growfs CAP_SYS_ADMIN.

## Acceptance Criteria

- [ ] AC-O1: `mkfs.xfs` + `xfstests-bld auto -g auto -F xfs` test suite passes with same pass/fail manifest as upstream baseline.
- [ ] AC-O2: cross-mount: xfs created on upstream-built kernel mounts read+write on Rookery + vice versa.
- [ ] AC-O3: Reflink test: `cp --reflink=always` shares extent; modify-via-CoW unshares correctly.
- [ ] AC-O4: `xfs_scrub /` runs full scrub + reports no errors.
- [ ] AC-O5: Online growfs: `xfs_growfs /` extends mounted FS to fill new partition size.
- [ ] AC-O6: pNFS test: NFSv4.1+ client with pNFS-block layout works against XFS-backed export.
- [ ] AC-O7: Zoned: HM-SMR drive used as XFS rt-device works.

## Verification

| TLA+ Model | Owner |
|---|---|
| `models/xfs/log_cil.tla` | `fs/xfs/log.md` (proves: CIL committed-item-list → log-commit pipeline; concurrent transactions buffer items in CIL; log-force flushes batch atomically; log-recovery replays in correct order) |
| `models/xfs/intent_defer.tla` | `fs/xfs/trans.md` (proves: deferred-intent (bmap-update / extent-free / refcount / rmap) intent items committed before consequent updates; recovery replays intents in order; partial completion handled correctly) |
| `models/xfs/refcount_share.tla` | `fs/xfs/refcount-reflink.md` (proves: reflink share+unshare under concurrent write; per-extent refcount b+tree consistent; CoW happens before write to shared extent) |
| `models/xfs/scrub_consistency.tla` | `fs/xfs/scrub.md` (proves: online scrub vs concurrent in-flight transaction; scrub uses snapshot of metadata; never reports false-positive on actively-changing AG) |

## Hardening

| Feature | Default |
|---|---|
| **REFCOUNT** | per-mount + per-inode + per-buf refcounts use `Refcount` | § Mandatory |
| **CONSTIFY** | per-FS `super_operations`, `inode_operations` `static const` | § Mandatory |
| **SIZE_OVERFLOW** | per-extent length + log-record arithmetic checked | § Mandatory |
| **MEMORY_SANITIZE** | freed inode + xattr cleared | § Default-on configurable |

XFS-specific reinforcement: every metadata block CRC-verified at read; corrupt-detect → log-error + remount-readonly + audit; `xfs_repair` requires offline + CAP_SYS_RAWIO; `dax=always` requires CAP_SYS_ADMIN; growfs + scrub-repair require CAP_SYS_ADMIN; per-LSM hook on quota config.

## Open Questions
(none at Tier-2)

## Out of Scope
- Implementation code
- 32-bit-only paths (XFS no longer supported on 32-bit)
- xfsprogs userspace (separate)
