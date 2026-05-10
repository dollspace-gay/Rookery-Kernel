---
title: "Tier-2: fs/zonefs — ZoneFS (per-zone file abstraction over zoned storage HM-SMR / ZNS)"
tags: ["tier-2", "fs", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 wrapper for ZoneFS — exposes a zoned block device (HM-SMR HDD or NVMe-ZNS SSD) as a small filesystem with a fixed file-per-zone tree (`/mnt/zonefs/seq/<N>` for sequential-zone files, `/mnt/zonefs/cnv/<N>` for conventional-zone files). Each file's i_size = zone wp; writes are append-only; truncate=zone-reset. Designed for direct application use (RocksDB, Ceph's BlueStore, custom append-only DBs) without ext4/btrfs/f2fs intermediation. Components: `super.c` + `inode.c` + `file.c` + `sysfs.c` + `zonefs.h`.

### compatibility contract — outline

- On-disk format byte-identical (zonefs-tools `mkzonefs` consumes).
- File tree layout `/seq/<N>` + `/cnv/<N>` byte-identical.
- Mount options byte-identical (errors, explicit-open).
- Per-file ioctls + standard file-API operations (open, read, write, fsync, truncate→zone-reset, ftruncate→zone-finish) byte-identical.

### tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `fs/zonefs/super.md` | `super.c` |
| `fs/zonefs/inode-file.md` | `inode.c` + `file.c` |
| `fs/zonefs/sysfs.md` | `sysfs.c` |

### compatibility outline / ac / verification / hardening

- REQ-O1: On-disk format + file tree byte-identical.
- REQ-O2: Append-only write semantics + truncate=zone-reset behavior identical.
- REQ-O3: TLA+ models (per-zone wp tracking under concurrent append; zone-reset atomicity).
- REQ-O4: AC: mkzonefs on HM-SMR HDD; mount; write-append + read; RocksDB-on-zonefs benchmark works.
- Hardening: row-1; per-zone access mediated by file-LSM hooks.

