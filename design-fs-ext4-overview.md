---
title: "Tier-2: fs/ext4 — ext4 filesystem (extent + htree + journaling jbd2 + fast-commit + verity + crypt + ACL/xattr + quota + zoning)"
tags: ["tier-2", "fs", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 wrapper for ext4 — the dominant Linux filesystem from kernel 2.6.28 onward; default for most distributions; remains the storage workhorse on Debian/Ubuntu/RHEL/Fedora server installs alongside XFS. Components: **on-disk format** (`ext4.h`, `ext4_extents.h`, `ext4_jbd2.h`: superblock + group-descriptor + inode + extent-tree + dir-htree + flex_bg layout), **inode + directory** (`inode.c` + `dir.c` + `namei.c` + `hash.c` + `htree.c`: inode lookup + directory hash-tree), **extent tree** (`extents.c` + `extents_status.c` + `extent_status_tree.c`: per-inode extent tree + extent-status cache), **block allocator** (`balloc.c` + `mballoc.c` + `bitmap.c` + `block_validity.c`: per-bg block bitmap + multi-block allocator + block validity), **inode allocator** (`ialloc.c`), **read/write** (`file.c` + `inline.c` + `fsmap.c` + `move_extent.c` + `truncate.c` + `page-io.c` + `readpage.c`: per-inode I/O paths + inline data + space mapping for `FS_IOC_FIEMAP`/`GETFSMAP`), **journaling integration** (`ext4_jbd2.c` — bridges to `fs/jbd2/` separate subsystem), **fast-commit** (`fast_commit.c` + `fast_commit.h`: NOFS-class commit-on-fsync acceleration via short-circuit transactions), **verity** (`verity.c`: fs-verity for read-only authenticity), **crypto** (`crypto.c`: fs-crypt for per-file encryption), **xattr + ACL** (`xattr.c` + `xattr_*.c` + `xattr_security.c` + `xattr_user.c` + `xattr_trusted.c` + `acl.c`), **quota** (`super.c` quota integration with `fs/quota/`), **resize** (`resize.c`: online filesystem grow), **mballoc** (`mballoc.c` + `mballoc-test.c`: multi-block allocator with locality-preserving group preference + stream allocator), **mmp** (`mmp.c`: Multi-Mount Protection — defeat double-mount over shared storage), **sysfs** (`sysfs.c`: `/sys/fs/ext4/<dev>/`), **orphan + recovery** (`orphan.c`), **symlink** (`symlink.c`), **migrate** (`migrate.c`: extent → indirect or vice versa), **super** (`super.c`: mount + sysctl + sb operations), **ext4-specific scrub-tests** (`extents-test.c`, `mballoc-test.c`).

`fs/jbd2/` is the journaling block device layer — distinct subsystem but always paired with ext4; in this Tier-2 the jbd2 is included since they're co-versioned.

### Acceptance Criteria

- [ ] AC-O1: `mkfs.ext4` + mount + extensive `xfstests-bld auto -g auto` test suite passes with same pass/fail manifest as upstream baseline.
- [ ] AC-O2: cross-mount: ext4 created on upstream-built kernel mounts read+write on Rookery + vice versa.
- [ ] AC-O3: `tune2fs -l` shows identical superblock fields.
- [ ] AC-O4: Online resize with `resize2fs` grows mounted filesystem.
- [ ] AC-O5: fs-crypt v2 policy applied to dir; per-file encryption works.
- [ ] AC-O6: fs-verity-protected file: corruption detected on read, returns -EIO.
- [ ] AC-O7: fast-commit benchmark: fsync latency within 5% upstream.
- [ ] AC-O8: MMP test: mounting same volume from two hosts simultaneously rejected.

### Out of Scope

- Implementation code
- 32-bit-only paths
- ext2/ext3 (separate `fs/ext2/` Tier-2 wrapper future, kept for compat)

### compatibility contract — outline

- On-disk format byte-identical to upstream — every feature bit (extents / 64bit / dir_index / dir_nlink / extra_isize / quota / large_file / huge_file / flex_bg / ext_attr / has_journal / inline_data / encrypt / verity / casefold / project / metadata_csum / mmp / fast_commit / orphan_present / sparse_super2 / orphan_file / sb_block_size_*) supported on read AND write side.
- mount(2) flags + ext4-specific mount options byte-identical: `data={journal,ordered,writeback}`, `journal_async_commit`, `journal_checksum`, `barrier`/`nobarrier`, `auto_da_alloc`/`noauto_da_alloc`, `discard`/`nodiscard`, `inode_readahead_blks=`, `init_itable=`, `noinit_itable`, `max_dir_size_kb=`, `min_batch_time=`, `max_batch_time=`, `stripe=`, `delalloc`/`nodelalloc`, `errors={continue,remount-ro,panic}`, `commit=`, `bsdgroups`/`grpid`/`nogrpid`, `acl`/`noacl`, `user_xattr`/`nouser_xattr`, `mblk_io_submit`, `block_validity`/`noblock_validity`, `dax`/`dax=always/never/inode`, `dioread_lock`/`dioread_nolock`, `i_version`, `prjquota`/`grpquota`/`usrquota`, `noload`, `lazytime`/`nolazytime`, `discard_unit_size`.
- ioctl(2): `EXT4_IOC_GETVERSION`, `_SETVERSION`, `_WAIT_FOR_READONLY`, `_GETVERSION_OLD`, `_SETVERSION_OLD`, `_GETRSVSZ`, `_SETRSVSZ`, `_GROUP_EXTEND`, `_GROUP_ADD`, `_MIGRATE`, `_ALLOC_DA_BLKS`, `_MOVE_EXT`, `_RESIZE_FS`, `_SWAP_BOOT`, `_PRECACHE_EXTENTS`, `_CLEAR_ES_CACHE`, `_GETSTATE`, `_GET_ES_CACHE`, `_CHECKPOINT`, `_GETFSUUID`, `_SETFSUUID`. Plus generic FS ioctls (`FS_IOC_FIEMAP`, `_GETFSLABEL`, `_SETFSLABEL`, `_FSGETXATTR`, `_FSSETXATTR`, `_GETFLAGS`, `_SETFLAGS`).
- `/sys/fs/ext4/<dev>/{...}` byte-identical (`mb_*`, `inode_goal`, `inode_readahead_blks`, `errors_count`, `last_error_*`, `first_error_*`, `extent_max_zeroout_kb`, `mb_stats`, `mb_max_to_scan`, `mb_min_to_scan`, `msg_ratelimit_*`, `warning_ratelimit_*`, `err_ratelimit_*`).
- jbd2 sysfs `/sys/fs/jbd2/<dev>/` byte-identical.
- Tracepoints `events/{ext4,jbd2}/` byte-identical.

### tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `fs/ext4/super.md` | `super.c`: mount + sb operations + sysctl |
| `fs/ext4/inode.md` | `inode.c` + `inline.c`: per-inode I/O |
| `fs/ext4/dir-htree.md` | `dir.c` + `namei.c` + `hash.c`: directory + hash-tree |
| `fs/ext4/extent.md` | `extents.c` + `extents_status.c`: extent tree |
| `fs/ext4/balloc-mballoc.md` | `balloc.c` + `mballoc.c` + `bitmap.c` + `block_validity.c`: block allocator |
| `fs/ext4/ialloc.md` | `ialloc.c`: inode allocator |
| `fs/ext4/file.md` | `file.c` + `readpage.c` + `page-io.c` + `truncate.c`: file I/O paths |
| `fs/ext4/fast-commit.md` | `fast_commit.c`: fast-commit |
| `fs/ext4/verity.md` | `verity.c`: fs-verity integration |
| `fs/ext4/crypt.md` | `crypto.c`: fs-crypt integration |
| `fs/ext4/xattr-acl.md` | `xattr*.c` + `acl.c` |
| `fs/ext4/resize.md` | `resize.c`: online grow |
| `fs/ext4/mmp.md` | `mmp.c`: Multi-Mount Protection |
| `fs/ext4/sysfs.md` | `sysfs.c`: per-mount sysfs |
| `fs/ext4/orphan.md` | `orphan.c`: orphan inode list + recovery |
| `fs/ext4/symlink.md` | `symlink.c`: symlink ops |
| `fs/ext4/migrate.md` | `migrate.c`: extent ↔ indirect |
| `fs/ext4/jbd2-core.md` | `fs/jbd2/transaction.c` + `commit.c` + `journal.c`: JBD2 journaling core |
| `fs/ext4/jbd2-recovery.md` | `fs/jbd2/recovery.c`: journal replay |
| `fs/ext4/jbd2-revoke.md` | `fs/jbd2/revoke.c`: revoke records |
| `fs/ext4/jbd2-checkpoint.md` | `fs/jbd2/checkpoint.c`: checkpoint |

### compatibility outline

- REQ-O1: On-disk format read + write byte-identical (e2fsprogs `tune2fs`, `dumpe2fs`, `debugfs`, `resize2fs` consume unchanged; cross-mount with upstream-built kernels round-trip).
- REQ-O2: All mount options + ioctls byte-identical.
- REQ-O3: `/sys/fs/ext4/<dev>/` + `/sys/fs/jbd2/<dev>/` sysfs byte-identical.
- REQ-O4: `events/{ext4,jbd2}/` tracepoints byte-identical.
- REQ-O5: fs-verity + fs-crypt integration identical (cross-ref `fs/verity/` + `fs/crypto/`).
- REQ-O6: Online resize + extent-migrate work identically.
- REQ-O7: TLA+ models declared (jbd2 commit-then-checkpoint ordering, mballoc concurrent-allocator + reserved-block accounting, htree concurrent insert/lookup, fast-commit replay correctness, mmp shared-storage protection).
- REQ-O8: Verus/Creusot Layer-4 contracts on extent-tree invariants + jbd2 transaction lifetime + htree invariants.
- REQ-O9: Hardening: per-FS LSM hooks, fs-verity required-on-boot for signed binaries, fs-crypt key-handling via keyring, MMP rate-limit.

### verification

| TLA+ Model | Owner |
|---|---|
| `models/ext4/jbd2_commit.tla` | `fs/ext4/jbd2-core.md` (proves: jbd2 commit pipeline — running → locked → flushed → committed; concurrent commit + new transaction never produce torn batch; checkpoint after commit completes correctly) |
| `models/ext4/mballoc.tla` | `fs/ext4/balloc-mballoc.md` (proves: mballoc per-bg + global allocator under N concurrent allocators; reserved-block accounting + locality preference + stream allocator never double-allocate or leak) |
| `models/ext4/htree.tla` | `fs/ext4/dir-htree.md` (proves: directory htree concurrent insert + lookup + split + collapse; hash-collision handling correctly preserves uniqueness; readdir cursor stable across split) |
| `models/ext4/fast_commit.tla` | `fs/ext4/fast-commit.md` (proves: fast-commit replay — partial fc-block list applied atomically; fallback to full-journal on inconsistency; recovery converges to consistent state) |
| `models/ext4/mmp.tla` | `fs/ext4/mmp.md` (proves: MMP heartbeat + foreign-write detection — concurrent multi-host mount detected within MMP_INTERVAL; race-free seqno bump + verify) |

### hardening

| Feature | Default |
|---|---|
| **REFCOUNT** | per-sb + per-inode + per-extent-status refcounts use `Refcount` | § Mandatory |
| **CONSTIFY** | per-FS `super_operations`, `inode_operations`, `file_operations`, `address_space_operations` `static const` | § Mandatory |
| **SIZE_OVERFLOW** | extent block-count + dir-htree depth + jbd2 transaction-size arithmetic checked | § Mandatory |
| **MEMORY_SANITIZE** | freed inode + xattr buffer cleared (carries fs-crypt master keys, ACL data) | § Default-on configurable |

ext4-specific reinforcement: fs-verity signed-only mode (CONFIG_FS_VERITY_BUILTIN_SIGNATURES + LSM mediation); fs-crypt keys stored in kernel keyring (cross-ref `security/keys/`); MMP rate-limit + audit on detection; on-disk corruption detected via metadata_csum triggers `errors=remount-ro` immediately + audit log; mount with `dax=always` requires CAP_SYS_ADMIN + LSM mediation (DAX bypasses page-cache → read-side bypass for fs-verity).

