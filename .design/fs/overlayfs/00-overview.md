# Tier-2: fs/overlayfs — overlayfs (lower + upper + work + merged; backbone of every container layer stack)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - fs/overlayfs/
-->

## Summary

Tier-2 wrapper for overlayfs — union mount filesystem underneath every container runtime (containerd, podman, docker, cri-o, lxc, systemd-nspawn) layered image. Core mechanism: lower-dirs (read-only) + upper-dir (writable) + work-dir (overlayfs work area) → merged view. CoW on first write; whiteout-files mark deletions. Components: `super.c`, `inode.c`, `dir.c`, `file.c`, `readdir.c`, `copy_up.c`, `namei.c`, `util.c`, `ovl_entry.h`, `overlayfs.h`, `params.c` (mount-option parser), `xattrs.c`, `export.c` (NFS export), `range.c`.

## Compatibility contract — outline

- `mount -t overlay -o lowerdir=...,upperdir=...,workdir=...,index=,xino=,redirect_dir=,metacopy=,nfs_export=,uuid=,verity=` mount options byte-identical (containerd / podman / mount.overlay consume).
- Whiteout-file format (char dev with major=0,minor=0; or trusted.overlay.whiteout xattr) byte-identical.
- Per-inode `trusted.overlay.*` xattrs byte-identical (origin, redirect, metacopy, impure, nlink, upper, opaque).
- xino (inode-number-stable across upper/lower) mode byte-identical.
- index (handle-stable) mode byte-identical.
- metacopy (copy-up metadata only, not data) byte-identical.
- nfs_export mode byte-identical.

## Tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `fs/overlayfs/super.md` | `super.c` + `params.c` |
| `fs/overlayfs/inode-dir-file.md` | `inode.c` + `dir.c` + `file.c` + `readdir.c` + `namei.c` |
| `fs/overlayfs/copy-up.md` | `copy_up.c`: CoW upon write |
| `fs/overlayfs/xattr.md` | `xattrs.c` + per-inode trusted.overlay.* |
| `fs/overlayfs/export.md` | `export.c`: NFS export |
| `fs/overlayfs/util-range.md` | `util.c` + `range.c` |

## Compatibility outline / AC / Verification / Hardening

- REQ-O1: All mount options + whiteout format + xattr format byte-identical.
- REQ-O2: TLA+ models (whiteout precedence in lookup; copy-up atomicity for partial-write; redirect_dir reference correctness).
- REQ-O3: AC: containerd image-pull + container start works; xfstests overlay subset passes; podman/docker run reference image works.
- Hardening: row-1; per-mount LSM stacking (overlay sets per-inode LSM context from upper or lower per policy); userns-mounting requires CAP_SYS_ADMIN in user-ns + LSM mediation.

## Out of Scope
Implementation code; 32-bit-only paths.
