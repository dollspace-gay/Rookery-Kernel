---
title: "Tier-2: fs/bcachefs — bcachefs (next-gen COW + RAID + replicas + tiering + checksum + erasure-coding + btree)"
tags: ["tier-2", "fs", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 wrapper for bcachefs — Kent Overstreet's next-gen filesystem (upstreamed 6.7); COW, native multi-device with replicas + erasure-coding + tiering (caching across SSD+HDD), per-extent checksum, snapshot, subvolume, native compression. Components: ~150 source files. Major: `bcachefs.h`, `super.c`, `super-io.c`, `super_types.h`, `bset.c` + `bset.h` (b-set in-memory representation), `btree_*.c` (~30 files: btree_io, btree_iter, btree_locking, btree_update_interior, btree_node_scan, btree_write_buffer, etc.), `alloc_*.c` + `alloc_background.c` + `alloc_foreground.c` (allocator), `chardev.c` (`/dev/bcachefs<N>` chardev), `checksum.c` + `chacha20.c` + `poly1305.c` (checksum + crypto), `compress.c` (LZ4/GZIP/ZSTD compression), `data_update.c` (per-update path), `dirent.c`, `disk_*.c` (disk-format), `ec.c` + `ec_format.h` (erasure-coding), `extent_update.c` + `extents.c` + `extents_types.h` (extent tree), `fs.c` + `fs-io*.c` + `fs-ioctl.c` (VFS + IO + ioctl), `inode.c` + `inode_format.h`, `journal_*.c` (journal), `keylist.c`, `migrate.c` (per-replica migration), `move.c` + `movinggc.c` (data movement + moving GC), `quota.c`, `rebalance.c` (balance bg work), `recovery.c` + `recovery_passes.c`, `replicas.c` + `replicas_format.h`, `snapshot.c` + `snapshot_format.h`, `subvolume.c` + `subvolume_format.h`, `xattr.c` + `acl.c`, `varint.c`, `vstructs.h`, `trace.c` + `trace.h`, `sysfs.c` + `sysfs.h`, `printbuf.c`, `bkey_methods.c` + `bkey_types.h` + `bkey_buf.h`.

### compatibility contract — outline

- On-disk format byte-identical (bcachefs-tools `bcachefs format`, `bcachefs fsck`, `bcachefs mount`, `bcachefs dump` consume).
- Mount options byte-identical (acl, compression={none,lz4,gzip,zstd}, background_compression, foreground_compression, errors={continue,ro,panic}, metadata_replicas, data_replicas, replicas, foreground_target, background_target, promote_target, metadata_target, fsck, fix_errors, ratelimit_errors, journal_flush_disabled, journal_reclaim_delay, …).
- BCH_IOC_* ioctls byte-identical.
- `/sys/fs/bcachefs/<UUID>/` byte-identical.

### tier-3 docs governed (selection — ~150 sources, deferring most to phase d)

| Tier-3 | Scope |
|---|---|
| `fs/bcachefs/super.md` | `super.c` + `super-io.c` |
| `fs/bcachefs/btree-core.md` | `btree_*.c` + `bset.c` + `bset.h` |
| `fs/bcachefs/alloc.md` | `alloc_*.c` + `alloc_background.c` + `alloc_foreground.c` |
| `fs/bcachefs/extent-data.md` | `extents.c` + `extent_update.c` + `data_update.c` |
| `fs/bcachefs/journal.md` | `journal_*.c` |
| `fs/bcachefs/replicas.md` | `replicas.c` |
| `fs/bcachefs/erasure.md` | `ec.c` |
| `fs/bcachefs/snapshot-subvol.md` | `snapshot.c` + `subvolume.c` |
| `fs/bcachefs/migrate-rebalance-gc.md` | `migrate.c` + `move.c` + `movinggc.c` + `rebalance.c` |
| `fs/bcachefs/checksum-compress.md` | `checksum.c` + `compress.c` + `chacha20.c` + `poly1305.c` |
| `fs/bcachefs/recovery.md` | `recovery.c` + `recovery_passes.c` |
| `fs/bcachefs/fs-vfs.md` | `fs.c` + `fs-io*.c` + `fs-ioctl.c` |
| `fs/bcachefs/dirent-inode.md` | `dirent.c` + `inode.c` |
| `fs/bcachefs/quota-xattr-acl.md` | `quota.c` + `xattr.c` + `acl.c` |
| `fs/bcachefs/chardev.md` | `chardev.c` + `sysfs.c` |

### compatibility outline / ac / verification / hardening

- REQ-O1: On-disk format byte-identical (bcachefs-tools consumes; cross-mount with upstream).
- REQ-O2: TLA+ models (btree COW + concurrent insert/lookup; journal replay; replicas write-quorum; erasure-coding stripe rebuild).
- REQ-O3: AC: bcachefs-tools format multi-device + mount + xfstests bcachefs subset passes.
- Hardening: row-1; per-extent checksum mandatory; ChaCha20-Poly1305 default for new mounts; replicas write-quorum required for ack.

