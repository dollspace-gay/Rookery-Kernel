---
title: "Tier-2: fs/erofs — Enhanced Read-Only Filesystem (compressed RO + Android system + container layers)"
tags: ["tier-2", "fs", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 wrapper for EROFS — read-only filesystem with on-disk LZ4/LZMA/DEFLATE/ZSTD compression, designed for Android system partition + container image layers + read-only system installs. Components: `erofs_fs.h`, `super.c`, `inode.c`, `dir.c`, `data.c`, `decompressor.c` + `decompressor_lz4.c` + `decompressor_lzma.c` + `decompressor_deflate.c` + `decompressor_zstd.c` + `decompressor_*.c`, `compress.h`, `decompressor_lzip.c`, `xattr.c`, `namei.c`, `fscache.c`, `zdata.c`, `zmap.c`, `zutil.c`.

### compatibility contract — outline

- On-disk format byte-identical (mkfs.erofs from erofs-utils consumes; cross-mount works).
- Mount options byte-identical (cache_strategy, dax, fsid, domain_id).
- fscache integration for over-IPFS or NFS-backing.

### tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `fs/erofs/super.md` | `super.c` |
| `fs/erofs/inode-dir-namei.md` | `inode.c` + `dir.c` + `namei.c` |
| `fs/erofs/data.md` | `data.c` |
| `fs/erofs/decompressor.md` | `decompressor*.c` |
| `fs/erofs/zdata-zmap.md` | `zdata.c` + `zmap.c` + `zutil.c` |
| `fs/erofs/xattr.md` | `xattr.c` |
| `fs/erofs/fscache.md` | `fscache.c` |

### compatibility outline / ac / verification / hardening

- REQ-O1: On-disk format byte-identical.
- REQ-O2: All decompressors (LZ4, LZMA, DEFLATE, ZSTD) work.
- REQ-O3: TLA+ models (per-cluster decompress under concurrent reads; fscache fetch race).
- REQ-O4: AC: mkfs.erofs + mount + read works; xfstests RO subset passes; container runtime overlayfs-on-erofs lower-dir works.
- Hardening: row-1; per-cluster decompression bounds-checked.

