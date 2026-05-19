# Tier-3: fs/vfs/super — struct super_block + super_operations + filesystem registration

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - fs/super.c
  - fs/init.c
  - fs/filesystems.c
  - fs/libfs.c
  - include/linux/fs.h
-->

## Summary
Tier-3 design for the per-mounted-filesystem-instance abstraction: `struct super_block`. Owns `super_operations` (sb-level vtable for alloc_inode, write_inode, sync_fs, statfs, freeze/thaw, remount, …), the registered-filesystem-types list (`/proc/filesystems`), the rootfs hand-off at boot (`fs/init.c`), generic libfs helpers, sb lifecycle (sget → kill_sb), per-sb shrinker registration, and freeze/thaw quiesce semantics.

Sub-tier-3 of `fs/vfs/00-overview.md`. The super_block is the per-mount-instance hub: every dentry, inode, and file ultimately points at one super_block; per-sb resources (LRU, writeback, dirty list) live here.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Superblock lifecycle (sget, deactivate_super, kill_sb) | `fs/super.c` |
| Boot-time rootfs init | `fs/init.c` |
| Filesystem-type registration (`register_filesystem`) | `fs/filesystems.c` |
| Generic libfs helpers (simple_super_block, mount_pseudo, …) | `fs/libfs.c` |
| Public types | `include/linux/fs.h` (struct super_block, super_operations, file_system_type) |

## Compatibility contract

### `struct super_block` layout

`include/linux/fs.h` defines `struct super_block`. Field-equivalent for first-cache-line + commonly-accessed fields:

- `s_list`, `s_dev`, `s_blocksize`, `s_blocksize_bits`, `s_maxbytes`
- `s_type` (file_system_type pointer)
- `s_op` (super_operations vtable)
- `s_dquot` (quota state)
- `s_active` (refcount; saturating)
- `s_security` (LSM blob)
- `s_xattr` (xattr_handler array)
- `s_root` (dentry — the FS's root)
- `s_umount` (rwsem)
- `s_count` (general refcount)
- `s_iflags` (kernel-internal flags, including SB_I_*)
- `s_flags` (mount flags: SB_RDONLY, SB_NOSUID, SB_NODEV, SB_NOEXEC, SB_SYNCHRONOUS, SB_MANDLOCK, SB_DIRSYNC, SB_NOATIME, SB_NODIRATIME, SB_SILENT, SB_POSIXACL, SB_INLINECRYPT, SB_KERNMOUNT, SB_I_VERSION, SB_LAZYTIME, SB_NOREMOTELOCK, SB_NOSEC, SB_BORN, SB_ACTIVE, SB_NOUSER)
- `s_magic` (FS-specific magic)
- `s_inodes` (per-sb inode list — cross-ref `inode.md`)
- `s_inodes_wb`, `s_inode_lru`
- `s_fs_info` (FS-private state)
- `s_id` (FS name + device)
- `s_uuid`, `s_subtype`
- `s_pre_writeback_inodes`, `s_dio_done_wq`
- `s_writers`, `s_min_blocksize`
- `s_remove_count` (deferred-remove counter)
- `s_dentry_lru`, `s_inode_lru` lists
- `s_remove_count`, `s_atime_quantum`, `s_time_min`, `s_time_max`

Layout-equivalent to upstream so out-of-tree FS implementations work unchanged.

### `super_operations` vtable

`alloc_inode`, `destroy_inode`, `dirty_inode`, `write_inode`, `evict_inode`, `put_super`, `sync_fs`, `freeze_super`, `freeze_fs`, `thaw_super`, `thaw_fs`, `statfs`, `remount_fs`, `umount_begin`, `show_options`, `show_devname`, `show_path`, `show_stats`, `quota_*`, `nr_cached_objects`, `free_cached_objects`, `shutdown`. Function-pointer table layout byte-identical.

### `/proc/filesystems`

Format: `<nodev?> <name>` per line. Identical.

### `/proc/mounts` (mount-table view)

Cross-ref `mount.md`; this Tier-3 owns the sb-side data, the format owned by `mount.md`.

### `statfs(2)` UAPI

`struct statfs`, `struct statfs64`, `struct statvfs` — byte-identical layout. Fields: `f_type`, `f_bsize`, `f_blocks`, `f_bfree`, `f_bavail`, `f_files`, `f_ffree`, `f_fsid`, `f_namelen`, `f_frsize`, `f_flags`, `f_spare`. UAPI MUST be byte-identical.

### Mount flags

`SB_*` flags (mode RDONLY/RW, NOEXEC, NODEV, NOSUID, etc.) — bit positions identical. `MS_*` (legacy mount(2) syscall) flags map to SB_*; numeric values byte-identical.

### Freeze / thaw

`freeze_super(sb)` / `thaw_super(sb)`: brings the FS into a quiescent state for snapshotting. Userspace via `FIFREEZE` / `FITHAW` ioctls (file-level) or `fsfreeze` userspace tool. Identical.

## Requirements

- REQ-1: `struct super_block` first-cache-line + commonly-accessed fields layout-equivalent to upstream.
- REQ-2: `super_operations` function-pointer table byte-identical (per `fs/vfs/00-overview.md` REQ-1).
- REQ-3: `struct file_system_type` layout byte-identical: `name`, `fs_flags`, `init_fs_context`, `parameters`, `mount`, `kill_sb`, `owner`, `next`, `fs_supers` (registered-supers list), `s_lock_key`, `s_umount_key`, `s_vfs_rename_key`, `s_writers_key`, `i_lock_key`, `i_mutex_key`, `invalidate_lock_key`, `i_mutex_dir_key`.
- REQ-4: `register_filesystem` / `unregister_filesystem` ABI preserved; `/proc/filesystems` format-identical.
- REQ-5: Superblock lifecycle: `sget` → `set_anon_super`/`get_tree_*` → `kill_sb`. `s_active` refcount transitions identical.
- REQ-6: `statfs` syscall + `struct statfs/statfs64/statvfs` UAPI byte-identical.
- REQ-7: Mount-flag (SB_*) bit positions identical; `MS_*` legacy compat preserved (`mount(2)` flag-mapping).
- REQ-8: Freeze/thaw quiesce: `FIFREEZE` ioctl quiesces the FS; subsequent `FITHAW` resumes. Atomic-quiesce semantics match upstream.
- REQ-9: Rootfs hand-off (`init_rootfs`, `populate_rootfs`, `init_mount_tree`) preserves boot-time semantics; cross-ref `init/00-overview.md` § rootfs-mount.md.
- REQ-10: Per-sb shrinker registered via `register_shrinker` participates in `mm/reclaim.md` shrinker dispatch; identical reclaim-priority + nr_to_scan semantics.
- REQ-11: `sb->s_inode_lru` LRU list participated in by every cache-resident inode (cross-ref `fs/vfs/inode.md`).
- REQ-12: libfs helpers (`simple_inode_init`, `simple_dir_inode_operations`, `simple_dir_operations`, `simple_lookup`, `mount_pseudo`, `simple_xattrs`, `simple_open`, `simple_get_link`, `simple_strtoul`, `simple_recursive_removal`, …) preserved — used by pseudo-FSes (proc, sysfs, devpts, debugfs).
- REQ-13: Idmapped-mount per-sb support: `s_user_ns` user-namespace + per-mount idmaps (cross-ref `fs/vfs/mount.md`).
- REQ-14: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `pahole struct super_block` first-cache-line layout byte-identical vs. upstream. (covers REQ-1)
- [ ] AC-2: `pahole struct file_system_type` byte-identical. (covers REQ-3)
- [ ] AC-3: `cat /proc/filesystems` byte-identical (modulo dynamically-loaded modules). (covers REQ-4)
- [ ] AC-4: A test mounts + unmounts ext4 100 times; `s_active` refcount transitions correctly throughout. (covers REQ-5)
- [ ] AC-5: `pahole struct statfs` + `struct statfs64` + `struct statvfs` byte-identical. (covers REQ-6)
- [ ] AC-6: An `mount(MS_RDONLY|MS_NOEXEC|MS_NODEV|MS_NOSUID, ...)` legacy-API call results in equivalent SB_* flags being set; observable via `/proc/<pid>/mountinfo`. (covers REQ-7)
- [ ] AC-7: An `fsfreeze -f /path` quiesces the FS; concurrent write blocks until `fsfreeze -u`. (covers REQ-8)
- [ ] AC-8: Boot Rookery; rootfs is correctly populated; `findmnt /` shows the boot-time rootfs. (covers REQ-9)
- [ ] AC-9: A test that creates 1M dentries + inodes on tmpfs; `echo 2 > /proc/sys/vm/drop_caches` triggers per-sb shrinker; LRU drops to expected level. (covers REQ-10, REQ-11)
- [ ] AC-10: A `mount_pseudo`-using internal FS (e.g., sockfs, pipefs) loads correctly. (covers REQ-12)
- [ ] AC-11: An idmapped-mount + chown test (cross-ref `inode.md`) verifies per-sb user-namespace integration. (covers REQ-13)
- [ ] AC-12: Hardening section present and follows template. (covers REQ-14)

## Architecture

### Rust module organization

- `kernel::fs::vfs::super_block::SuperBlock` — wrapper type
- `kernel::fs::vfs::super_block::ops::SuperOps` — super_operations vtable trait
- `kernel::fs::vfs::super_block::lifecycle` — sget / kill_sb / deactivate_super
- `kernel::fs::vfs::filesystems::FileSystemType` — registered-FS-type
- `kernel::fs::vfs::filesystems::register_filesystem` / `unregister_filesystem`
- `kernel::fs::vfs::libfs` — generic helpers (simple_*, mount_pseudo, ...)
- `kernel::fs::vfs::super_block::freeze` — freeze/thaw quiesce
- `kernel::fs::vfs::super_block::shrinker` — per-sb shrinker registration

### Locking and concurrency

- **`s_umount`** (rwsem): protects sb mount/unmount/remount; held write-side during umount/remount
- **`sb->s_inode_list_lock`** (spinlock): protects `s_inodes` list
- **`sb_writers->lock`** (per-write-class hash): freeze-protect machinery
- **`s_dentry_lru` lock**: per-LRU-list spinlock for dentry LRU
- **`file_systems_lock`** (rwlock): protects registered file-systems list

### Error handling

- `Err(EBUSY)` — sb in active use; cannot unmount
- `Err(ENOMEM)` — sb alloc / inode_cache alloc failed
- `Err(ENOENT)` — file_system_type not registered
- `Err(EROFS)` — write attempted on read-only FS
- `Err(EINVAL)` — bad mount option / bad SB flag combination

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| sb list manipulation | `kani::proofs::fs::vfs::super_block::list_safety` |
| sb refcount transitions (s_active, s_count) | `kani::proofs::fs::vfs::super_block::refcount_safety` |
| Freeze write-protect manipulation | `kani::proofs::fs::vfs::super_block::freeze_safety` |
| Filesystem-type registration | `kani::proofs::fs::vfs::filesystems::register_safety` |

### Layer 2: TLA+ models

(none mandatory at this sub-tier — sb lifecycle is mostly serialized by s_umount; concurrency models live in dcache.md, inode.md, mount.md)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-sb inode list (`s_inodes`) | Every linked inode is on its sb's list (mirror of `inode.md`'s invariant) | `kani::proofs::fs::vfs::super_block::inode_list_invariants` |
| Filesystem-type list | Every registered FS appears exactly once; order matches registration | `kani::proofs::fs::vfs::filesystems::registered_invariants` |
| Sb refcount + s_active interaction | s_active > 0 implies sb is on the global list; s_active == 0 implies kill_sb in progress | `kani::proofs::fs::vfs::super_block::refcount_invariants` |

### Layer 4: Functional correctness (opt-in)

- **Sb lifecycle state machine** via Verus — proves: every transition (alloc → init → SB_BORN → SB_ACTIVE → kill_sb) preserves invariants under concurrent operations.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | `s_active` + `s_count` use `Refcount` (saturating) | § Mandatory |
| **AUTOSLAB** | per-FS sb allocated via `KmemCache::<MyFsSb>::new()`; per-FS type-tagged in `/proc/slabinfo` | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT**, **AUTOSLAB**, **MEMORY_SANITIZE** — see above + per-sb dentry/inode caches
- **CONSTIFY**: super_operations vtables provided by per-FS modules are `static const`
- **SIZE_OVERFLOW**: sb statistics arithmetic uses checked operators

### Row-2 / GR-RBAC integration

LSM hooks fired from this component:
- `security_sb_alloc` / `security_sb_free` — sb lifecycle
- `security_sb_kern_mount` — kernel-side mount
- `security_sb_show_options` — readable per-sb options (LSM filter)
- `security_sb_remount` — remount permission check
- `security_sb_pivotroot` — pivotroot syscall hook

GR-RBAC (per its loaded policy) can deny mount/umount/remount operations.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — bounded copy_to/from_user on `struct statfs`/`statfs64`/`statvfs` out-buffers and remount-option in-buffers.
- **PAX_KERNEXEC** — W^X for super_operations vtables and any FS module text mapped via `register_filesystem`.
- **PAX_RANDKSTACK** — per-syscall kernel-stack randomization at statfs / mount / remount / umount entries.
- **PAX_REFCOUNT** — saturating refcount on `s_active`, `s_count`, `s_writers` per-class counters; wrap-to-zero UAF closed.
- **PAX_MEMORY_SANITIZE** — zero-on-free for super_block slabs, sb->s_fs_info FS-private tails, sb->s_security LSM blob.
- **PAX_UDEREF** — strict user-pointer access for fsconfig payloads + mount/remount option strings.
- **GRKERNSEC_HIDESYM** — hide super_block kernel pointers in /proc/mounts + tracepoints + audit records.
- **GRKERNSEC_LINK** — `fs.protected_hardlinks` policy plumbed through `s_flags`/SB_I_* iflags so per-mount can override.
- **GRKERNSEC_SYMLINKOWN** — symlink-owner restrictions evaluated against per-sb idmap.
- **GRKERNSEC_FIFO** — FIFO-creation restrictions checked against per-sb `s_flags` (SB_NOSUID/NODEV/NOEXEC) before mknod path.
- **GRKERNSEC_CHROOT_MOUNT** — `mount(2)` blocked inside chroot regardless of CAP_SYS_ADMIN at register_filesystem dispatch.
- **GRKERNSEC_TRUSTED** — `trusted.*` xattr namespace gated at per-sb `s_xattr` handler dispatch.
- **GRKERNSEC_PROC** — restrict `/proc/<pid>/mountinfo` to caller's mount namespace and CAP_SYS_ADMIN as policy dictates.
- **PAX_RANDFS** — per-FS layout-randomization seed stored in sb (consumed by tmpfs/ramfs/proc inode-number assignment).
- **PAX_SIZE_OVERFLOW** — statfs counter arithmetic + per-sb writer-class arithmetic uses checked operators.

Per-doc rationale: the super_block is the per-mount hub that materializes a mount's security personality (NOSUID/NODEV/NOEXEC/POSIXACL/idmap/LSM-label) and serves as the trust root for every dentry, inode, and file under it. PAX_REFCOUNT + PAX_MEMORY_SANITIZE guard the lifecycle; GRKERNSEC_CHROOT_MOUNT + GRKERNSEC_TRUSTED + PAX_UDEREF lock the configuration + ABI surface where mount-time evasions historically land.

## Open Questions

(none — sb semantics are exhaustively specified by the VFS contract)

## Out of Scope

- inode cache (cross-ref `fs/vfs/inode.md`)
- dcache (cross-ref `fs/vfs/dcache.md`)
- Per-FS-specific superblock extensions (cross-ref individual fs/* docs)
- 32-bit-only paths
- Implementation code
