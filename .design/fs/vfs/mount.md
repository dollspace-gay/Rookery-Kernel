# Tier-3: fs/vfs/mount — vfsmount, mount namespace, propagation, idmapped mounts

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - fs/namespace.c
  - fs/mount.h
  - fs/pnode.c
  - fs/fs_pin.c
  - fs/fs_struct.c
  - fs/mnt_idmapping.c
  - include/linux/mount.h
  - include/linux/path.h
  - include/uapi/linux/mount.h
-->

## Summary
Tier-3 design for the mount namespace, propagation rules, and idmapped mounts. Owns `struct vfsmount` (the mountpoint reference), `struct mount` (kernel-internal full mount state), the per-task mount-namespace tree, propagation types (shared, private, slave, unbindable), idmapped-mount uid/gid translation, fs_struct (per-task root + cwd), and fs_pin (mount-pinning machinery used to prevent disk-detach mid-use).

Sub-tier-3 of `fs/vfs/00-overview.md`. The mount namespace is the substrate for containers — every container runtime (Docker, Kubernetes, systemd-nspawn, Podman, LXC) relies on this code path being byte-identical.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Mount namespace + tree mgmt | `fs/namespace.c` |
| Mount-internal types | `fs/mount.h` |
| Propagation rules (shared/private/slave) | `fs/pnode.c` |
| Mount pinning (anti-detach-during-use) | `fs/fs_pin.c` |
| Per-task root + cwd | `fs/fs_struct.c` |
| Idmapped mount uid/gid translation | `fs/mnt_idmapping.c` |
| Public types | `include/linux/mount.h`, `include/linux/path.h` |
| UAPI | `include/uapi/linux/mount.h` |

## Compatibility contract

### Syscalls

`mount`, `umount`, `umount2`, `pivot_root`, `chroot`, `chdir`, `fchdir`, `fsmount`, `fsopen`, `fsconfig`, `fspick`, `move_mount`, `open_tree`, `mount_setattr`. Each gets a Tier-5 doc.

### `struct vfsmount` + `struct mount`

`include/linux/mount.h` (`struct vfsmount` — short, public-facing) and `fs/mount.h` (`struct mount` — kernel-internal, full state). Layout-equivalent for first-cache-line so out-of-tree code accessing fields via macro works.

`struct vfsmount`:
- `mnt_root` (root dentry of the FS this is a mount of)
- `mnt_sb` (super_block)
- `mnt_flags` (MNT_*: NOSUID, NODEV, NOEXEC, NOATIME, NODIRATIME, RELATIME, READONLY, ...)

`struct mount` (private):
- `mnt_hash`, `mnt_parent` (parent mount; for the mount tree)
- `mnt_mountpoint` (dentry where this mount is attached in parent)
- `mnt` (embedded struct vfsmount)
- `mnt_mp` (mountpoint structure for shared dentry-pinning)
- `mnt_rcu`, `mnt_llist` (rcu+llist for batched put)
- `mnt_count`, `mnt_writers`, `mnt_master`, `mnt_ns` (refcount + writers + propagation chain)
- `mnt_pinned`, `mnt_ghosts`
- `mnt_idmap` (idmapped-mount mapping pointer; cross-ref `mnt_idmapping.c`)
- `mnt_share`, `mnt_slave_list`, `mnt_slave`
- `mnt_expire`
- per-mount cgroup info

### `/proc/<pid>/{mountinfo,mounts,mountstats}`

`/proc/<pid>/mountinfo` line format per `Documentation/filesystems/proc.rst`:
```
<mount id> <parent id> <major:minor> <root> <mount point> <mount options> <optional fields> - <fs type> <mount source> <super options>
```

Example: `26 21 0:24 / /sys/fs/cgroup ro,nosuid,nodev,noexec shared:8 - tmpfs tmpfs ro,mode=755`

Format-identical to upstream. Existing tools (findmnt, systemd-mount, container-runtime mount-table parsers) work unmodified.

`/proc/<pid>/mounts` (legacy 5-column format) and `/proc/<pid>/mountstats` (per-mount statistics) — both format-identical.

### Mount flags (`MS_*` legacy + `MOUNT_ATTR_*` modern)

Numeric values byte-identical:
- `MS_RDONLY=1`, `MS_NOSUID=2`, `MS_NODEV=4`, `MS_NOEXEC=8`, `MS_SYNCHRONOUS=16`, `MS_REMOUNT=32`, `MS_MANDLOCK=64`, `MS_DIRSYNC=128`, `MS_NOATIME=1024`, `MS_NODIRATIME=2048`, `MS_BIND=4096`, `MS_MOVE=8192`, `MS_REC=16384`, `MS_VERBOSE=32768`, `MS_SILENT=32768`, `MS_POSIXACL=0x10000`, `MS_UNBINDABLE=0x20000`, `MS_PRIVATE=0x40000`, `MS_SLAVE=0x80000`, `MS_SHARED=0x100000`, `MS_RELATIME=0x200000`, `MS_KERNMOUNT=0x400000`, `MS_I_VERSION=0x800000`, `MS_STRICTATIME=0x1000000`, `MS_LAZYTIME=0x2000000`, `MS_NOREMOTELOCK=0x8000000`, `MS_NOSEC=0x10000000`, `MS_BORN=0x20000000`, `MS_ACTIVE=0x40000000`, `MS_NOUSER=0x80000000`.

`MOUNT_ATTR_*` (modern; for `mount_setattr(2)`): `MOUNT_ATTR_RDONLY=1`, `MOUNT_ATTR_NOSUID=2`, `MOUNT_ATTR_NODEV=4`, `MOUNT_ATTR_NOEXEC=8`, `MOUNT_ATTR_RELATIME=0`, `MOUNT_ATTR_NOATIME=0x10`, `MOUNT_ATTR_STRICTATIME=0x20`, `MOUNT_ATTR_NODIRATIME=0x80`, `MOUNT_ATTR__ATIME=0x70`, `MOUNT_ATTR_IDMAP=0x100000`, `MOUNT_ATTR_NOSYMFOLLOW=0x200000`.

### Propagation types

`MS_SHARED`, `MS_PRIVATE`, `MS_SLAVE`, `MS_UNBINDABLE` — propagation behavior matches upstream byte-for-byte. Existing systemd `MountFlags=`, container-runtime mount-propagation conventions work unchanged.

### Idmapped mounts (`mount_setattr(MOUNT_ATTR_IDMAP)`)

`include/linux/mnt_idmapping.h` defines `vfsuid_t` and `vfsgid_t` typed handles; per-mount idmap maps host uid/gid → mount-visible uid/gid. Userspace API: `mount_setattr(2)` with `MOUNT_ATTR_IDMAP` and an fd to `/proc/<pid>/uid_map`-style struct (`struct mount_attr.userns_fd`). Identical.

## Requirements

- REQ-1: All mount-related syscalls byte-identical entry/exit ABI per upstream.
- REQ-2: `struct vfsmount` + `struct mount` layouts equivalent for first-cache-line + commonly-macro-accessed fields.
- REQ-3: `/proc/<pid>/{mountinfo,mounts,mountstats}` content format-identical.
- REQ-4: Mount-flag (`MS_*` legacy + `MOUNT_ATTR_*` modern) numeric values + semantics byte-identical.
- REQ-5: Propagation rules (shared / private / slave / unbindable) match upstream's algorithm:
  - When mounting on a shared mount, the new mount becomes shared with the parent
  - When mounting on a private mount, the new mount is private
  - When mounting on a slave mount, the new mount is slave to the parent's slave-master
  - Unbindable mounts cannot be bind-mounted
- REQ-6: Mount namespace isolation (`CLONE_NEWNS` / `unshare(CLONE_NEWNS)`): per-namespace mount tree; `/proc/<pid>/mountinfo` shows only the calling task's namespace's mounts.
- REQ-7: `pivot_root(2)` semantics: swap rootfs of the current mount namespace; identical to upstream.
- REQ-8: `chroot(2)` + per-task `fs_struct->root` semantics identical.
- REQ-9: Idmapped mounts: `mount_setattr(MOUNT_ATTR_IDMAP, ...)` semantics identical; `vfsuid_t` / `vfsgid_t` typed handles enforce no-mixing of host uid + mount-visible uid in compile time.
- REQ-10: Mount-pinning (fs_pin): drivers can pin a mount to prevent unmount during use; identical semantics.
- REQ-11: Modern mount API: `fsopen` returns fsfd; `fsconfig` configures (FSCONFIG_SET_STRING/PATH/FD/FLAG, FSCONFIG_CMD_CREATE, FSCONFIG_CMD_RECONFIGURE); `fsmount` materializes; `move_mount` attaches. Identical (cross-ref `fs/vfs/fs-context.md` for the fsopen/fsmount-flow details).
- REQ-12: `umount(2)` with MNT_DETACH (lazy unmount) preserves upstream semantics: unmounted but still-referenced mounts are detached from the tree but kept alive until refcount drops.
- REQ-13: `findmnt`, `mount`, `umount`, `systemd-mount` userspace tools work unmodified against Rookery.
- REQ-14: TLA+ model `models/fs/mount_propagation.tla` (per `fs/00-overview.md` Layer 2 list) proves shared/private/slave propagation preserves mount-tree invariants under racing mount/umount.
- REQ-15: Layer-3 invariant harnesses (mandatory per `fs/00-overview.md` REQ-15+16): mount tree acyclic; every mountpoint dentry has corresponding `vfsmount->mnt_mountpoint`.
- REQ-16: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `strace -e trace=mount,umount,umount2,pivot_root,move_mount,open_tree,fsopen,fsconfig,fsmount,mount_setattr` of a systemd boot byte-identical vs. upstream. (covers REQ-1)
- [ ] AC-2: `pahole struct vfsmount` + `pahole struct mount` byte-identical first-cache-line layout. (covers REQ-2)
- [ ] AC-3: `cat /proc/self/mountinfo` after a curated container-startup workload byte-identical. (covers REQ-3)
- [ ] AC-4: A `mount(MS_BIND|MS_REC, ...)` test produces a recursively-bind-mounted subtree; verifiable via mountinfo. (covers REQ-4, REQ-5)
- [ ] AC-5: A propagation test: mount shared / private / slave / unbindable variants; subsequent bind-mounts propagate (or don't) per upstream's rules. (covers REQ-5)
- [ ] AC-6: An `unshare -m` shell sees its own mount namespace; subsequent mounts in the unshared namespace don't leak to host. (covers REQ-6)
- [ ] AC-7: `pivot_root` test in a namespace: rootfs swaps correctly; old root reachable as expected. (covers REQ-7)
- [ ] AC-8: A `chroot(2)` + read in chroot returns expected content (matches upstream). (covers REQ-8)
- [ ] AC-9: An idmapped-mount selftest under `tools/testing/selftests/mount_setattr/` passes with same set as upstream. (covers REQ-9)
- [ ] AC-10: A driver test that takes a `fs_pin` reference + concurrent `umount` returns EBUSY. (covers REQ-10)
- [ ] AC-11: A `fsopen + fsconfig + fsmount + move_mount` round-trip test mounts tmpfs successfully. (covers REQ-11)
- [ ] AC-12: `umount /mnt --lazy` (MNT_DETACH) on a busy mount detaches the mount; the underlying FS remains valid for in-flight operations. (covers REQ-12)
- [ ] AC-13: `findmnt`, `mount -a`, `systemd --user-units mount` operate identically. (covers REQ-13)
- [ ] AC-14: `make tla` passes `models/fs/mount_propagation.tla`. (covers REQ-14)
- [ ] AC-15: `make verify` passes mandatory L3 harnesses for mount-tree invariants. (covers REQ-15)
- [ ] AC-16: Hardening section present and follows template. (covers REQ-16)

## Architecture

### Rust module organization

- `kernel::fs::vfs::mount::Mount` / `Vfsmount` — wrapper types
- `kernel::fs::vfs::mount::namespace::MountNs` — per-task mount namespace
- `kernel::fs::vfs::mount::tree` — mount-tree manipulation (graft, ungraft, attach, detach)
- `kernel::fs::vfs::mount::propagation` — shared/private/slave/unbindable rules
- `kernel::fs::vfs::mount::idmap::Idmap` — idmapped-mount uid/gid translation
- `kernel::fs::vfs::mount::pivot_root` — pivot_root + chroot
- `kernel::fs::vfs::mount::fs_pin::FsPin` — mount pinning
- `kernel::fs::vfs::mount::fs_struct::FsStruct` — per-task root + cwd
- `kernel::fs::vfs::mount::sysctl` — per-namespace sysctl knobs

### Locking and concurrency

- **`namespace_sem`** (rwsem): per-namespace; held write-side during mount/umount; read-side during read of mountinfo
- **`mount_lock`** (seqlock): protects mount tree; readers retry on writer
- **Per-mount `mnt_count`** + **`mnt_writers`** atomics: refcount management
- **Per-mountpoint `mp->m_dentry->d_lock`**: held during mount-attach to dentry

TLA+ model `models/fs/mount_propagation.tla` proves the propagation algorithm under racing operations.

### Error handling

- `Err(EBUSY)` — mount in active use; cannot unmount without MNT_DETACH
- `Err(ENOENT)` — mountpoint not found
- `Err(EINVAL)` — bad flags; bad propagation type combination
- `Err(EPERM)` — capability check failed (CAP_SYS_ADMIN required)
- `Err(EXDEV)` — cross-device pivot_root
- `Err(EOVERFLOW)` — mount-counter overflow

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Mount-tree graft / ungraft (parent-child link manipulation) | `kani::proofs::fs::vfs::mount::tree_safety` |
| Propagation list traversal under namespace_sem | `kani::proofs::fs::vfs::mount::propagation_safety` |
| Idmap apply/revert (uid_map xform) | `kani::proofs::fs::vfs::mount::idmap_safety` |
| fs_pin take/release under mount_lock | `kani::proofs::fs::vfs::mount::pin_safety` |
| pivot_root atomic-swap | `kani::proofs::fs::vfs::mount::pivot_root_safety` |

### Layer 2: TLA+ models

- `models/fs/mount_propagation.tla` (mandatory per `fs/00-overview.md` Layer 2) — proves shared/private/slave propagation preserves mount-tree invariants. Owned here.

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Mount tree | Every mountpoint dentry has the corresponding `vfsmount->mnt_mountpoint` field; mount tree is acyclic | `kani::proofs::fs::vfs::mount::tree_invariants` |
| Propagation chains | Slave list per master; shared list per peer; no cycles in slave/master graph | `kani::proofs::fs::vfs::mount::propagation_invariants` |
| Mount namespaces | Every namespace has a unique mnt_ns_id; init_mnt_ns is global | `kani::proofs::fs::vfs::mount::namespace_invariants` |
| Idmap chain | Per-mount idmap is either NULL or points to a valid mnt_idmap with refcount ≥ 1 | `kani::proofs::fs::vfs::mount::idmap_invariants` |

### Layer 4: Functional correctness (opt-in)

- **Idmap algebra correctness** via Creusot — proves: round-trip uid through `from_vfsuid` + `to_vfsuid` preserves identity; idmap composition is associative.
- **Propagation algorithm correctness** via Verus — proves: under any sequence of mount/umount operations on shared/private/slave subtrees, the resulting mount tree remains consistent.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | `mnt_count`, `mnt_writers`, idmap refcounts use `Refcount` (saturating) | § Mandatory |
| **Mount NOSUID/NODEV/NOEXEC** (per-mount flags) | Userspace can opt-in via `MS_NOSUID|MS_NODEV|MS_NOEXEC` to drop privilege within a mount | (existing upstream feature; preserved) |
| **MOUNT_ATTR_NOSYMFOLLOW** (modern alternative to chroot-symlink hardening) | Per-mount; new in modern mount API | § Default-on configurable off |
| **Idmapped mounts as drop-privilege primitive** | Container runtimes use idmapped mounts to drop privilege; preserved per upstream | (existing) |

### Row-1 features consumed by this component

- **UDEREF**: mount syscall args (path strings, options strings) come through `UserPtr<...>`; raw deref forbidden
- **AUTOSLAB**: mount + vfsmount + idmap + fs_struct allocated via per-type slab caches
- **MEMORY_SANITIZE**: freed mount objects zeroed
- **CONSTIFY**: per-FS file_system_type structures provided by FS modules are `static const`
- **SIZE_OVERFLOW**: refcount + bitmap arithmetic uses checked operators

### Row-2 / GR-RBAC integration

LSM hooks dispatched:
- `security_sb_mount` — pre-mount permission check (path, fs_type, flags, data)
- `security_sb_umount` — pre-umount permission check
- `security_sb_pivotroot` — pre-pivot_root permission check
- `security_sb_remount` — pre-remount permission check
- `security_move_mount` — modern mount API hook
- `security_path_chroot` — chroot syscall

GR-RBAC's chroot-hardening + mount-restriction policies (cross-ref `references/grsec-pax-notes.md`) all consume these hooks. The grsec GRKERNSEC_CHROOT_* feature suite becomes loadable settings of the GR-RBAC LSM (per the layered absorption model in `00-security-principles.md`).

### Userspace-visible behavior changes

Per Axiom 4 of `00-security-principles.md`:
- **MOUNT_ATTR_NOSYMFOLLOW default-on**: when MS_NOSUID|MS_NODEV|MS_NOEXEC is selected, MOUNT_ATTR_NOSYMFOLLOW is also added unless explicitly cleared. Container runtimes have already migrated to opt-in NOSYMFOLLOW; this Rookery default extends to host-mounts.

### Verification

(See § Verification above.)

## Open Questions

(none — mount semantics are exhaustively specified by upstream + container-runtime conventions)

## Out of Scope

- Modern mount API (cross-ref `fs/vfs/fs-context.md` for fsopen/fsmount-flow detail)
- 32-bit-only paths
- Implementation code
