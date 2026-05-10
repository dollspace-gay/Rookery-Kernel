---
title: "Tier-2: fs/ntfs3 — NTFS3 filesystem (Paragon-contributed read+write NTFS, replacing legacy fs/ntfs/ + ntfs-3g FUSE)"
tags: ["tier-2", "fs", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 wrapper for ntfs3 — Paragon-contributed kernel-side read+write NTFS driver (upstreamed 5.15), replaces the legacy `fs/ntfs/` (read-only) + ntfs-3g (FUSE). Supports NTFS 3.x format (NT 3.51 onward). Components: `super.c`, `inode.c`, `dir.c`, `file.c`, `namei.c`, `attrib.c`, `attrlist.c`, `bitfunc.c`, `bitmap.c`, `dir.c`, `file.c`, `frecord.c`, `fslog.c`, `fsntfs.c`, `index.c`, `lib/decompress_common.c`, `lib/lzx_common.c`, `lib/lzx_decompress.c`, `lib/lzx_constants.h`, `lib/xpress_decompress.c`, `lznt.c`, `record.c`, `run.c`, `upcase.c`, `xattr.c`, `ntfs.h`, `ntfs_fs.h`, `lib/`. Compression decompressors: LZX (file-system compression), XPRESS, LZNT.

### compatibility contract — outline

- On-disk format byte-identical to Microsoft NTFS spec (cross-mount with Windows works).
- Mount options byte-identical (uid, gid, umask, dmask, fmask, prealloc, no_acs_rules, acl, noatime, sparse, hidden, sys_immutable, discard, force, sparse, hide_dot_files, windows_names, hidedotfiles).
- File compression (LZX/XPRESS/LZNT) decompressed on read; compressed-write supported.
- ADS (Alternate Data Streams) accessible via `streams_interface=` mount option.
- Reparse points + symlinks + junctions handled.

### tier-3 docs governed (selection)

| Tier-3 | Scope |
|---|---|
| `fs/ntfs3/super.md` | `super.c` + `fsntfs.c` + `ntfs.h` |
| `fs/ntfs3/inode-dir-file-namei.md` | `inode.c` + `dir.c` + `file.c` + `namei.c` |
| `fs/ntfs3/attrib.md` | `attrib.c` + `attrlist.c` + `record.c` + `frecord.c` + `run.c` + `index.c` |
| `fs/ntfs3/fslog.md` | `fslog.c`: NTFS log replay |
| `fs/ntfs3/bitmap.md` | `bitmap.c` + `bitfunc.c` |
| `fs/ntfs3/decompress.md` | `lib/`: LZX + XPRESS + LZNT decompressors |
| `fs/ntfs3/upcase-xattr.md` | `upcase.c` + `xattr.c` |

### compatibility outline / ac / verification / hardening

- REQ-O1: On-disk format byte-identical (Windows interop).
- REQ-O2: TLA+ models (NTFS log replay correctness; MFT entry concurrent update; sparse-file run-list management).
- REQ-O3: AC: cross-mount NTFS volume Win10↔Linux, read+write, no corruption; chkdsk on Win clean post-Linux-mount.
- Hardening: row-1; LZX/XPRESS/LZNT decompressor bounds-checked.

