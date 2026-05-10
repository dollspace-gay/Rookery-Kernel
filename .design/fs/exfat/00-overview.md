# Tier-2: fs/exfat — exFAT filesystem (Microsoft exFAT — SD/SDXC/USB-stick interop)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - fs/exfat/
-->

## Summary

Tier-2 wrapper for exFAT — Microsoft's filesystem for high-capacity removable storage (SDXC ≥ 32GB, USB sticks, external SSDs); upstreamed in 5.7. Components: `exfat_fs.h`, `super.c`, `inode.c`, `dir.c`, `file.c`, `namei.c`, `cache.c`, `balloc.c`, `fatent.c`, `misc.c`, `nls.c` (NLS codepage), `xattr.c`.

## Compatibility contract — outline

- On-disk format per Microsoft exFAT spec byte-identical (cross-mount with Windows + macOS + Android consumers).
- Mount options byte-identical (uid, gid, umask, dmask, fmask, allow_utime, codepage, iocharset, namecase, errors, discard, time_offset).

## Tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `fs/exfat/super.md` | `super.c` + `exfat_fs.h` |
| `fs/exfat/inode-dir-file-namei.md` | `inode.c` + `dir.c` + `file.c` + `namei.c` |
| `fs/exfat/balloc-fatent.md` | `balloc.c` + `fatent.c` |
| `fs/exfat/cache-nls.md` | `cache.c` + `nls.c` |
| `fs/exfat/xattr.md` | `xattr.c` |

## Compatibility outline / AC / Verification / Hardening

- REQ-O1: On-disk format byte-identical (Win/macOS interop).
- REQ-O2: TLA+ models (cluster-allocator FAT-table consistency; bitmap allocation race).
- REQ-O3: AC: format SD card on Windows, mount on Linux, read+write, mount on macOS, files visible.
- Hardening: row-1; case-sensitive default off (matches Win); discard mode off by default for SD-card durability.

## Out of Scope
Implementation code; 32-bit-only paths.
