# Tier-3: fs/vfs/00-overview — VFS core hub

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - fs/inode.c
  - fs/dcache.c
  - fs/super.c
  - fs/file.c
  - fs/file_table.c
  - fs/namei.c
  - fs/namespace.c
  - fs/pnode.c
  - fs/mount.h
  - fs/d_path.c
  - fs/libfs.c
  - fs/bad_inode.c
  - fs/anon_inodes.c
  - fs/mnt_idmapping.c
  - fs/file_attr.c
  - fs/init.c
  - fs/filesystems.c
  - fs/fs_context.c
  - fs/fs_parser.c
  - fs/fs_pin.c
  - fs/fs_struct.c
  - fs/fsopen.c
  - fs/attr.c
  - fs/posix_acl.c
  - fs/utimes.c
  - fs/xattr.c
  - include/linux/fs.h
  - include/linux/dcache.h
  - include/linux/path.h
  - include/linux/file.h
  - include/linux/mount.h
  - include/linux/namei.h
  - include/linux/posix_acl.h
  - include/linux/xattr.h
  - include/linux/fsnotify.h
-->

## Summary
Tier-3 hub for the Virtual Filesystem (VFS) core: the abstraction layer that every concrete filesystem plugs into. Owns the data model (inode, dentry, file, super_block, vfsmount, path), the lifecycle of each, the path-resolution machinery, the file-table per-task ownership, the mount-namespace propagation, the file-system context (modern mount API), the file_operations / inode_operations / address_space_operations / super_operations / dentry_operations / vfsmount-operations vtables, ACL + xattr + idmapped-mount integration, and the cross-arch helpers (libfs).

This 00-overview hub spawns sub-tier docs: `inode.md`, `dcache.md` (already written), `super.md`, `file-table.md`, `mount.md`, `path-resolution.md`, `fs-context.md`. Sub-tier-3 docs detail per-component implementation; this 00-overview establishes shared concepts.

## Upstream references in scope

| Subsystem | Upstream paths | Sub-tier-3 doc |
|---|---|---|
| inode lifecycle + cache | `fs/inode.c`, `fs/bad_inode.c`, `fs/anon_inodes.c` | `inode.md` |
| dcache (already authored) | `fs/dcache.c`, `fs/d_path.c` | `dcache.md` (existing) |
| super_block lifecycle + sb operations | `fs/super.c`, `fs/init.c`, `fs/filesystems.c` | `super.md` |
| File-table + per-task fdtable | `fs/file.c`, `fs/file_table.c`, `fs/file_attr.c` | `file-table.md` |
| Mount namespace + propagation | `fs/namespace.c`, `fs/mount.h`, `fs/pnode.c`, `fs/fs_pin.c`, `fs/fs_struct.c`, `fs/mnt_idmapping.c` | `mount.md` |
| Path resolution (namei) | `fs/namei.c` | `path-resolution.md` |
| Modern mount API (fsopen/fsmount/fsconfig) | `fs/fs_context.c`, `fs/fs_parser.c`, `fs/fsopen.c` | `fs-context.md` |
| Generic libfs helpers | `fs/libfs.c` | folded into `inode.md` + `super.md` |
| File attribute manipulation | `fs/attr.c`, `fs/utimes.c` | folded into `inode.md` |
| POSIX ACL | `fs/posix_acl.c` | folded into `inode.md` |
| Extended attributes | `fs/xattr.c` | folded into `inode.md` |
| Public types | `include/linux/{fs,dcache,path,file,mount,namei,posix_acl,xattr,fsnotify}.h` | (referenced from each sub-doc) |

## Compatibility contract

### VFS data model

Every concrete filesystem (ext4, btrfs, xfs, tmpfs, …) plugs into the VFS via:

- **`super_operations`** — per-FS-instance: `alloc_inode`, `destroy_inode`, `dirty_inode`, `write_inode`, `evict_inode`, `put_super`, `sync_fs`, `freeze_fs`, `thaw_fs`, `statfs`, `remount_fs`, `umount_begin`, `show_options`, `quota_*`, `nr_cached_objects`, `free_cached_objects`
- **`inode_operations`** — per-inode: `lookup`, `link`, `unlink`, `symlink`, `mkdir`, `rmdir`, `mknod`, `rename`, `readlink`, `permission`, `get_acl`, `setattr`, `getattr`, `listxattr`, `fileattr_get`, `fileattr_set`, `tmpfile`, `set_acl`, `get_offset_ctx`
- **`file_operations`** — per-open-file: `llseek`, `read_iter`, `write_iter`, `iopoll`, `iterate_shared`, `poll`, `unlocked_ioctl`, `compat_ioctl`, `mmap`, `mmap_supported_flags`, `open`, `flush`, `release`, `fsync`, `fasync`, `lock`, `get_unmapped_area`, `check_flags`, `flock`, `splice_write`, `splice_read`, `splice_eof`, `setlease`, `fallocate`, `show_fdinfo`, `copy_file_range`, `remap_file_range`, `fadvise`, `uring_cmd`, `uring_cmd_iopoll`
- **`address_space_operations`** — per-mapping: `read_folio`, `writepage`, `writepages`, `dirty_folio`, `release_folio`, `invalidate_folio`, `direct_IO`, `migrate_folio`, `launder_folio`, `is_partially_uptodate`, `error_remove_folio`, `swap_activate`, `swap_deactivate`, `swap_rw`, `validate_folio`
- **`dentry_operations`** — per-dentry: `d_revalidate`, `d_weak_revalidate`, `d_hash`, `d_compare`, `d_delete`, `d_init`, `d_release`, `d_prune`, `d_iput`, `d_dname`, `d_automount`, `d_manage`, `d_real`

Function-pointer table layouts MUST be byte-identical so existing fs/* implementations + out-of-tree filesystems work unchanged (recompile-from-source compat per `00-overview.md` REQ-4 / D2).

### `/proc/sys/fs/*` sysctls

`file-max`, `nr_open`, `dir-notify-enable`, `inode-nr`, `inode-state`, `dentry-state`, `lease-break-time`, `pipe-max-size`, `pipe-user-pages-{soft,hard}`, `protected_*`, `binfmt_misc/*`, `mqueue/*`, `quota/*`. Format-identical.

### Userspace tooling compat

Every existing userspace tool that interacts with the VFS (find, ls, stat, du, df, lsof, ftruncate, mount, umount, fstab, systemd-mount, getcwd, getpwd) works unmodified.

## Requirements

- REQ-1: Every operation table layout (super_operations, inode_operations, file_operations, address_space_operations, dentry_operations) byte-identical with upstream so out-of-tree filesystems work unchanged.
- REQ-2: VFS data-model invariants (cross-ref sub-docs):
  - One `inode` per file (across all hardlinks); refcount-managed
  - One `dentry` per name in the dcache (cached path component)
  - One `file` per open(2) call; multiple `file`s can share an `inode`
  - One `super_block` per mounted filesystem instance
  - One `vfsmount` per mountpoint in a mount namespace
- REQ-3: Sub-tier-3 docs cover every component (inode, dcache, super, file-table, mount, path-resolution, fs-context).
- REQ-4: Idmapped mounts: per-mount uid/gid mapping; vfs operations honor `vfsuid_t`/`vfsgid_t` typed handles vs. raw uid/gid; `from_vfsuid` / `to_vfsuid` / `vfsuid_eq_kuid` semantics byte-identical.
- REQ-5: Modern mount API (fsopen + fsconfig + fsmount + move_mount) preserves syscall ABI per Tier-5 docs.
- REQ-6: Path resolution: RCU-walk + ref-walk paths, LOOKUP_* flag set, mount-traversal, symlink-resolution, deep-symlink limits all match upstream.
- REQ-7: File-locking (flock + POSIX + OFD + leases) byte-identical (cross-ref `fs/00-overview.md` Compatibility contract).
- REQ-8: ACL + xattr + posix_acl semantics match upstream byte-for-byte.
- REQ-9: fsnotify integration: every mutation invokes appropriate fsnotify hook so inotify/fanotify/dnotify event streams match upstream.
- REQ-10: LSM hooks: every mutation invokes appropriate `security_inode_*` / `security_file_*` / `security_path_*` / `security_sb_*` hook.
- REQ-11: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: A canonical filesystem (ext4) built from upstream source against Rookery's vmlinux loads + mounts + reads/writes correctly without recompilation. (covers REQ-1)
- [ ] AC-2: VFS namei selftest (`tools/testing/selftests/filesystems/`) passes. (covers REQ-2, REQ-6)
- [ ] AC-3: Each sub-tier-3 doc (inode, super, file-table, mount, path-resolution, fs-context) is drafted with its own REQ/AC/Verification/Hardening sections. (covers REQ-3)
- [ ] AC-4: Idmapped-mount selftest under `tools/testing/selftests/mount_setattr/` passes. (covers REQ-4)
- [ ] AC-5: Modern mount API test (`fsopen`+`fsmount`+`fsconfig`) round-trips a tmpfs mount. (covers REQ-5)
- [ ] AC-6: `flock` + `fcntl(F_SETLK)` + `fcntl(F_OFD_SETLK)` selftests pass. (covers REQ-7)
- [ ] AC-7: ACL/xattr selftest round-trips POSIX ACLs, security xattrs, user xattrs. (covers REQ-8)
- [ ] AC-8: An inotify-based file-monitor produces byte-identical event stream vs. upstream after curated workload. (covers REQ-9)
- [ ] AC-9: An LSM-policy denying `inode_create` translates to `EACCES` from the mkdir/touch syscall. (covers REQ-10)
- [ ] AC-10: Hardening section present and follows template. (covers REQ-11)

## Architecture

### Sub-tier-3 doc layout

```
.design/fs/vfs/
  00-overview.md       ← this hub
  inode.md             ← struct inode + inode_operations + alloc_inode/destroy_inode + ACLs + xattrs + attr/utimes
  dcache.md            ← (already drafted) struct dentry + dentry_operations + dcache hash + LRU + RCU-walk
  super.md             ← struct super_block + super_operations + filesystem registration + libfs helpers
  file-table.md        ← struct file + file_operations + per-task fdtable + close_on_exec + close_range + file_attr
  mount.md             ← struct vfsmount + mount namespace + propagation rules + idmapped mounts + fs_pin
  path-resolution.md   ← namei.c + LOOKUP_* + symlink resolution + RCU-walk vs ref-walk + deep-symlink loop
  fs-context.md        ← modern mount API: fsopen + fsmount + fsconfig + fs_parser + move_mount
```

### Locking landscape (umbrella view; sub-docs detail)

| Lock | Type | Owner | Sub-doc |
|---|---|---|---|
| `inode->i_rwsem` | rwsem | per-inode | `inode.md` |
| `inode->i_lock` | spinlock | per-inode | `inode.md` |
| `dentry->d_lock` | spinlock | per-dentry | `dcache.md` |
| `s_umount` | rwsem | per-superblock | `super.md` |
| `sb->s_inode_list_lock` | spinlock | per-superblock | `inode.md` |
| `namespace_sem` | rwsem | global | `mount.md` |
| `mount_lock` | seqlock | global | `mount.md` |
| `flc_lock` | spinlock | per-file | `file-table.md` |
| `files->file_lock` | spinlock | per-task | `file-table.md` |

### Rust module organization

- `kernel::fs::vfs::Inode` — inode wrapper
- `kernel::fs::vfs::Dentry` — dentry wrapper (cross-ref `dcache.md`)
- `kernel::fs::vfs::SuperBlock` — superblock wrapper
- `kernel::fs::vfs::File` — file wrapper
- `kernel::fs::vfs::VfsMount` — vfsmount wrapper
- `kernel::fs::vfs::Path` — (vfsmount, dentry) pair
- `kernel::fs::vfs::ops` — vtable traits (InodeOps, FileOps, AddressSpaceOps, SuperOps, DentryOps)

### Error handling

VFS uses standard errno set; sub-docs detail per-operation errors. Most-common: `EACCES`, `ENOENT`, `EISDIR`, `ENOTDIR`, `EXDEV`, `EBUSY`, `ENOSPC`, `EDQUOT`, `EROFS`, `ESTALE`, `ENAMETOOLONG`, `ELOOP`, `EFBIG`.

## Verification

This 00-overview is a hub; verification artifacts are owned by the sub-tier-3 docs:

- Layer 1 SAFETY proofs per sub-component
- Layer 2 TLA+ models: `dcache_rcu_walk.tla` (existing), `mount_propagation.tla`, `inode_concurrent_link.tla`, `file_lock_compat.tla`, `fsnotify_event_order.tla`, `jbd2_journal.tla` (per `fs/00-overview.md` Layer 2 list)
- Layer 3 invariants: dcache (already drafted as MANDATORY L3); other sub-doc invariants per their declarations
- Layer 4 opt-ins per sub-doc

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

This hub doc doesn't directly own row-1 features; sub-tier-3 docs own their portions:

- `inode.md` — REFCOUNT for i_count + i_writecount + ACL refcounts; AUTOSLAB for inode struct
- `dcache.md` — REFCOUNT for d_lockref; AUTOSLAB for dentry struct (already covered in dcache.md)
- `super.md` — REFCOUNT for s_active; AUTOSLAB for superblock struct
- `file-table.md` — REFCOUNT for f_count; close-on-exec + close_range hardening for fd inheritance
- `mount.md` — idmapped-mount uid/gid validation; chroot hardening primitives
- `path-resolution.md` — symlink-loop-bound enforcement; RCU-walk safety
- `fs-context.md` — fsconfig parameter validation; LSM hook for sb mount

### LSM hooks dispatched from this subsystem

- `security_inode_*` (mkdir, rmdir, create, unlink, symlink, link, rename, ...)
- `security_file_*` (open, ioctl, mmap, fcntl, fadvise, ...)
- `security_path_*` (chroot, chdir, ...)
- `security_sb_*` (mount, umount, kern_mount, ...)
- `security_inode_setxattr` / `security_inode_getxattr` / `security_inode_listxattr` / `security_inode_removexattr`
- `security_inode_set_acl` / `security_inode_get_acl`

GR-RBAC's TPE + chroot-hardening + path-restriction policies all consume these hooks.

### Userspace-visible behavior changes

None at this hub level; sub-doc-specific behavior changes (if any) appear in sub-doc Hardening sections.

## Open Questions

(none — VFS hub establishes shared concepts; per-component questions surface in sub-tier-3 docs as they're authored)

## Out of Scope

- Per-filesystem implementations (cross-ref individual fs/* docs in `00-index.md`)
- VFS test infrastructure (cross-ref sub-doc per-test sections)
- 32-bit-only VFS paths
- Implementation code
