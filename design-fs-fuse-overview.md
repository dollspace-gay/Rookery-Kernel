---
title: "Tier-2: fs/fuse — FUSE (Filesystem in USErspace + virtio-fs)"
tags: ["tier-2", "fs", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 wrapper for FUSE — userspace filesystem framework underneath sshfs, gocryptfs, encfs, ntfs-3g, ifuse, gvfsd-fuse, mergerfs, mhddfs, mountpoint-s3, ec2cli/aws-cli s3-mount, podman/docker overlay-via-fuse, every dev-build-only userspace FS. Plus virtio-fs (kernel→host shared FS for cloud VMs). Components: `fuse_i.h`, `inode.c`, `dev.c` (`/dev/fuse` chardev — kernel↔userspace handler dispatch), `dir.c`, `file.c`, `xattr.c`, `acl.c`, `readdir.c`, `control.c`, `cuse.c` (CUSE — Char Devices in USErspace; lets userspace impl chardevs e.g. fuse-pulseaudio-virtual-snd), `dax.c` (FUSE+DAX), `iomode.c`, `passthrough.c` (passthrough mode — bypass userspace handler for direct backing-file IO; performance optimization), `virtio_fs.c` (virtio-fs guest-side; pairs with virtiofsd userspace daemon on host).

### compatibility contract — outline

- `/dev/fuse` chardev byte-identical (libfuse + libfuse3 consume).
- FUSE protocol message format (struct fuse_in/out_header + per-op payload) byte-identical.
- Mount options + `FUSE_INIT` features negotiated identically.
- virtio-fs FUSE-over-virtio wire format byte-identical (virtiofsd consumes).

### tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `fs/fuse/inode.md` | `inode.c`: per-inode + sb |
| `fs/fuse/dev.md` | `dev.c`: /dev/fuse chardev + msg dispatch |
| `fs/fuse/file-dir-readdir.md` | `file.c` + `dir.c` + `readdir.c` |
| `fs/fuse/xattr-acl.md` | `xattr.c` + `acl.c` |
| `fs/fuse/control.md` | `control.c`: per-mount control fs at /sys/fs/fuse/connections/ |
| `fs/fuse/cuse.md` | `cuse.c`: CUSE chardev-in-userspace |
| `fs/fuse/dax-iomode.md` | `dax.c` + `iomode.c`: FUSE+DAX |
| `fs/fuse/passthrough.md` | `passthrough.c`: bypass userspace for direct backing IO |
| `fs/fuse/virtio-fs.md` | `virtio_fs.c`: virtio-fs guest-side |

### compatibility outline / ac / verification / hardening

- REQ-O1: /dev/fuse + FUSE protocol byte-identical (sshfs + libfuse2 + libfuse3 consume unchanged).
- REQ-O2: virtio-fs interop with virtiofsd.
- REQ-O3: TLA+ models (per-request unique-id allocation + concurrent reply; FORGET refcount; abort propagation on user-daemon crash).
- REQ-O4: AC: sshfs mount + read+write works; virtio-fs cloud-hypervisor + virtiofsd works; xfstests subset for FUSE passes.
- Hardening: per-mount user-allow-other gated by /etc/fuse.conf + LSM hook; passthrough requires CAP_SYS_ADMIN per-mount.

