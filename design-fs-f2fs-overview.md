---
title: "Tier-2: fs/f2fs — Flash-Friendly Filesystem (log-structured + zoned + segment + multi-device)"
tags: ["tier-2", "fs", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 wrapper for F2FS — Flash-Friendly Filesystem (Samsung-developed, default on most Android devices, also useful for SD-card / SSD storage where flash-aware allocation matters). Log-structured with append-only writes inside per-segment regions; zoned-device aware (HM-SMR). Components: `f2fs.h`, `super.c`, `inode.c`, `dir.c`, `file.c`, `data.c`, `node.c`, `segment.c`, `gc.c`, `recovery.c`, `checkpoint.c`, `xattr.c`, `acl.c`, `extent_cache.c`, `inline.c`, `compress.c`, `compress_lzo.c`, `compress_lz4.c`, `compress_zstd.c`, `compress_lzorle.c`, `verity.c`, `iostat.c`, `shrinker.c`, `sysfs.c`, `debug.c`, `dax.c`.

### compatibility contract — outline

- On-disk format byte-identical (cross-mount with f2fs-tools).
- Mount options byte-identical (background_gc, inline_xattr, inline_data, inline_dentry, flush_merge, data_flush, fastboot, extent_cache, mode, active_logs, alloc_mode, fsync_mode, whint_mode, test_dummy_encryption, compress_*, atgc, gc_merge).
- F2FS_IOC_* ioctls byte-identical (f2fs-tools consumes).
- `/sys/fs/f2fs/<dev>/{...}` byte-identical.

### tier-3 docs governed (selection)

| Tier-3 | Scope |
|---|---|
| `fs/f2fs/super.md` | `super.c` + `f2fs.h` |
| `fs/f2fs/inode-dir-file.md` | `inode.c` + `dir.c` + `file.c` |
| `fs/f2fs/data.md` | `data.c`: data-block IO |
| `fs/f2fs/node.md` | `node.c`: node-block + nat |
| `fs/f2fs/segment-gc.md` | `segment.c` + `gc.c`: segment alloc + GC |
| `fs/f2fs/checkpoint-recovery.md` | `checkpoint.c` + `recovery.c` |
| `fs/f2fs/extent-inline.md` | `extent_cache.c` + `inline.c` |
| `fs/f2fs/compress.md` | `compress.c` + `compress_*.c` |
| `fs/f2fs/verity-crypt.md` | `verity.c` + fs-crypt integration |
| `fs/f2fs/xattr-acl.md` | `xattr.c` + `acl.c` |
| `fs/f2fs/sysfs-debug-iostat.md` | `sysfs.c` + `debug.c` + `iostat.c` |

### compatibility outline / ac / verification / hardening

- REQ-O1: On-disk format + mount options + ioctls byte-identical (f2fs-tools `mkfs.f2fs`, `fsck.f2fs`, `dump.f2fs` consume).
- REQ-O2: TLA+ models (segment GC + concurrent file write; checkpoint pipeline; node-block reference under concurrent inode update).
- REQ-O3: AC: xfstests f2fs subset passes; cross-mount with upstream kernel works.
- Hardening: row-1; fs-crypt + fs-verity integration; metadata-csum on (defense against corruption).

