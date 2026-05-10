# Tier-3: fs/vfs/inode — struct inode + inode_operations + ACL + xattr

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - fs/inode.c
  - fs/bad_inode.c
  - fs/anon_inodes.c
  - fs/attr.c
  - fs/utimes.c
  - fs/posix_acl.c
  - fs/xattr.c
  - fs/libfs.c
  - include/linux/fs.h
  - include/linux/posix_acl.h
  - include/linux/xattr.h
  - include/uapi/linux/xattr.h
-->

## Summary
Tier-3 design for the VFS inode: the file-metadata data type. Owns `struct inode` lifecycle (alloc → initialize → use → evict → free), per-superblock inode list, the inode hash table, generic inode-operations helpers (libfs), POSIX ACLs, extended attributes (xattrs), file-attr manipulation (chmod, chown, utimes), bad-inode placeholder, and anon-inodes (used for various pseudo-FDs like eventfd, pipe, signalfd).

Sub-tier-3 of `fs/vfs/00-overview.md`. The inode is one of the central data types in the kernel; every file (regular, directory, symlink, device-node, fifo, socket) is represented by an inode.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| inode lifecycle + cache | `fs/inode.c` |
| Bad-inode placeholder | `fs/bad_inode.c` |
| Anon-inodes (used by eventfd / pipe / signalfd / etc.) | `fs/anon_inodes.c` |
| File attribute manipulation (chmod / chown / setattr) | `fs/attr.c` |
| utimes (atime/mtime modification) | `fs/utimes.c` |
| POSIX ACLs | `fs/posix_acl.c` |
| Extended attributes | `fs/xattr.c`, `include/linux/xattr.h`, `include/uapi/linux/xattr.h` |
| Generic libfs helpers | `fs/libfs.c` (selected functions for inode-only) |
| Public types | `include/linux/fs.h` (struct inode, inode_operations) |
| ACL types | `include/linux/posix_acl.h` |

## Compatibility contract

### `struct inode` layout

`include/linux/fs.h` defines `struct inode`. First-cache-line + commonly-accessed fields layout-equivalent to upstream:

- `i_mode`, `i_opflags`, `i_uid`, `i_gid`
- `i_flags`
- `i_acl`, `i_default_acl` (POSIX ACL pointers)
- `i_op` (inode_operations vtable)
- `i_sb` (super_block)
- `i_mapping` (address_space)
- `i_security` (LSM blob)
- `i_ino` (inode number)
- `i_count` (refcount; saturating)
- `i_nlink` (link count)
- `i_rdev` (device node major:minor for char/block devices)
- `i_size`, `i_blocks`, `i_bytes`
- `i_atime`, `i_mtime`, `i_ctime`, `i_btime` (timestamps)
- `i_lock` (spinlock)
- `i_rwsem` (rwsem)
- `i_state` (I_NEW, I_DIRTY_*, I_LOCK, I_FREEING, I_CLEAR, I_SYNC, …)
- `i_hash`, `i_io_list`, `i_lru`, `i_sb_list`, `i_wb_list` (list nodes)
- `i_dentry` (alias dentry list — hardlinks)
- `i_version` (NFS-style version counter)
- `i_writecount` (refcount of writers; for write-deny)
- `i_link` (inline symlink target for short symlinks)
- `i_dir_seq` (dir-readers seqcount)

### Userspace-visible inode metadata (via `stat(2)`, `statx(2)`)

`struct stat`, `struct stat64`, `struct statx` — byte-identical layout. Visible fields:
- `st_mode`, `st_uid`, `st_gid`, `st_size`, `st_blocks`, `st_blksize`, `st_dev`, `st_rdev`, `st_ino`, `st_nlink`, `st_atime`, `st_mtime`, `st_ctime`, `st_btime` (statx)

UAPI struct layout MUST be byte-identical so existing libc stat wrappers work unmodified.

### POSIX ACL UAPI

`include/uapi/linux/posix_acl.h` (or `_xattr.h`) — bit-identical entry layout: `e_tag` (ACL_USER, ACL_GROUP, etc.), `e_perm` (read/write/execute bits), `e_id` (UID/GID). xattr names: `system.posix_acl_access`, `system.posix_acl_default`. Identical.

### xattr UAPI

`syscalls`: `setxattr`, `lsetxattr`, `fsetxattr`, `getxattr`, `lgetxattr`, `fgetxattr`, `listxattr`, `llistxattr`, `flistxattr`, `removexattr`, `lremovexattr`, `fremovexattr`. Namespaces: `user.*`, `system.*`, `security.*`, `trusted.*`. ABI byte-identical.

### `/proc/sys/fs/{inode-nr,inode-state}`

Format: `<nr_inodes> <nr_unused>` and `<dummy_arg>` lines. Format-identical.

## Requirements

- REQ-1: `struct inode` first-cache-line + commonly-macro-accessed fields layout-equivalent to upstream so out-of-tree filesystems' inline-fastpath access works unchanged.
- REQ-2: `inode_operations` function-pointer table layout byte-identical (per `fs/vfs/00-overview.md` REQ-1).
- REQ-3: `struct stat` / `struct stat64` / `struct statx` UAPI layout byte-identical.
- REQ-4: Inode lifecycle:
  - `new_inode(sb)` allocates from sb's inode_cache, sets `I_NEW`, increments `sb->s_inode_list`
  - `mark_inode_dirty(inode)` adds to writeback list + sets `I_DIRTY_*`
  - `iput(inode)` decrements; if 0 and inode is unhashed, calls `evict_inode` from sb_op
  - `evict_inode` cleans up FS-private state; final `clear_inode` removes from all lists + frees via `destroy_inode` from sb_op
  - All transitions: `I_NEW` → in-use → `I_FREEING` → `I_CLEAR` per upstream
- REQ-5: Inode hash (per-inode-ino + per-sb global hash bucket) preserved; hashing function byte-identical.
- REQ-6: Per-superblock inode-list (`sb->s_inodes`) preserved; LRU + writeback lists ditto.
- REQ-7: POSIX ACL: `posix_acl_alloc`, `posix_acl_create`, `posix_acl_chmod`, `posix_acl_equiv_mode`, `posix_acl_permission` semantics byte-identical. ACL inheritance from parent dir matches upstream.
- REQ-8: Extended attributes:
  - `setxattr` / `getxattr` / `listxattr` / `removexattr` syscalls byte-identical
  - Per-FS xattr_handler tables match upstream: `XATTR_SYSTEM_PREFIX` ("system."), `XATTR_TRUSTED_PREFIX` ("trusted."), `XATTR_SECURITY_PREFIX` ("security."), `XATTR_USER_PREFIX` ("user.")
- REQ-9: File-attribute manipulation: `chmod` / `chown` / `setattr` semantics; capability-check (CAP_FOWNER, CAP_CHOWN); idmapped-mount uid/gid translation
- REQ-10: utimes / utimensat: ATIME, MTIME, CTIME update semantics; relatime + lazytime + noatime mount-flag honored
- REQ-11: anon_inodes: `anon_inode_getfd` / `anon_inode_create` returns an inode with no on-disk backing, used by eventfd/pipe/signalfd/timerfd/fanotify/inotify/userfaultfd.
- REQ-12: bad_inode: `make_bad_inode` returns an inode with all operations replaced by an EIO-returning vtable (used to mark filesystems as failed without freeing the inode).
- REQ-13: Layer-3 invariant harness: every linked inode appears on its superblock's `s_inodes` list. Cross-ref `fs/vfs/dcache.md` for parent-dentry consistency.
- REQ-14: TLA+ model `models/fs/inode_concurrent_link.tla` (per `fs/00-overview.md` Layer 2 list) proves concurrent link/unlink doesn't corrupt i_nlink counter.
- REQ-15: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `pahole struct inode` first-cache-line layout byte-identical vs. upstream. (covers REQ-1)
- [ ] AC-2: `pahole struct stat` + `struct stat64` + `struct statx` UAPI layouts byte-identical. (covers REQ-3)
- [ ] AC-3: A test creating 100K inodes via touch + unlinking verifies state transitions: `i_state` walks `I_NEW` → `0` → `I_FREEING` → `I_CLEAR`; lists are consistent. (covers REQ-4)
- [ ] AC-4: A `stat`-based file-system traversal benchmark (`find -ls /usr -size +10M`) on Rookery vs. upstream produces identical output. (covers REQ-3, REQ-4)
- [ ] AC-5: A POSIX ACL test sets default ACL on parent + creates child; child inherits default ACL with mode masked correctly. (covers REQ-7)
- [ ] AC-6: An xattr round-trip test on each namespace (user, security, trusted, system) byte-identical. (covers REQ-8)
- [ ] AC-7: An idmapped-mount + chown test: chown'ing a file via a uid_map'd mount translates correctly per upstream's idmap algebra. (covers REQ-9)
- [ ] AC-8: utimensat with mount mounted `noatime` doesn't update atime; with `relatime`, atime updates only when older than mtime. (covers REQ-10)
- [ ] AC-9: An eventfd test creates a 100-eventfd batch; each via `anon_inode_getfd`. lsof shows them as `anon_inode:[eventfd]`. (covers REQ-11)
- [ ] AC-10: A test that forces an FS into a "failed" state and replaces an inode with `make_bad_inode` causes subsequent reads to return EIO. (covers REQ-12)
- [ ] AC-11: `make verify` passes `kani::proofs::fs::inode::*` Kani harnesses. (covers REQ-13)
- [ ] AC-12: `make tla` passes `models/fs/inode_concurrent_link.tla`. (covers REQ-14)
- [ ] AC-13: Hardening section present and follows template. (covers REQ-15)

## Architecture

### Rust module organization

- `kernel::fs::vfs::inode::Inode` — `struct inode` wrapper
- `kernel::fs::vfs::inode::ops::InodeOps` — inode_operations vtable trait
- `kernel::fs::vfs::inode::lifecycle` — new_inode / iget / iput / evict / destroy
- `kernel::fs::vfs::inode::hash` — per-FS inode hash table
- `kernel::fs::vfs::inode::lists` — per-sb + per-mm + LRU + writeback list mgmt
- `kernel::fs::vfs::inode::attr` — chmod/chown/setattr
- `kernel::fs::vfs::inode::utimes` — atime/mtime update
- `kernel::fs::vfs::inode::acl::PosixAcl` — POSIX ACL
- `kernel::fs::vfs::inode::xattr` — xattr namespace dispatch
- `kernel::fs::vfs::inode::anon` — anon_inode helper
- `kernel::fs::vfs::inode::bad` — bad_inode placeholder
- `kernel::fs::vfs::inode::libfs` — generic helpers (read_dir, simple_inode, ...)

### Locking and concurrency

- **`inode->i_rwsem`** (sleepable rw): held during inode-modifying operations (rename, mkdir, unlink, write); read-side during read, getxattr.
- **`inode->i_lock`** (spinlock): atomic state-flag updates (I_NEW, I_DIRTY_*, I_LOCK), refcount adjustments, list manipulations.
- **`sb->s_inode_list_lock`**: per-sb inode-list manipulation.
- **`inode_hash_lock`**: per-bucket spinlock (hashed by ino).
- **`fsnotify_mark->lock`**: per-mark refcount updates.
- **`xattr_lock`** (per-inode): held during xattr mutation.

### Error handling

- `Err(EACCES)` — permission denied (DAC, MAC, ACL)
- `Err(ENOENT)` — inode lookup miss
- `Err(EXDEV)` — cross-device hardlink/rename
- `Err(EBUSY)` — inode busy during evict
- `Err(ESTALE)` — NFS-style stale handle
- `Err(EIO)` — bad-inode placeholder operation
- `Err(ENAMETOOLONG)` — name too long for FS
- `Err(EOPNOTSUPP)` — operation not supported by underlying FS
- `Err(EROFS)` — read-only filesystem

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Inode hash insertion + collision-list link | `kani::proofs::fs::inode::hash_link_safety` |
| Inode list (s_inodes) push/pop | `kani::proofs::fs::inode::sb_list_safety` |
| State-flag atomic updates | `kani::proofs::fs::inode::state_flags_safety` |
| ACL inheritance | `kani::proofs::fs::inode::acl_inherit_safety` |
| xattr namespace dispatch | `kani::proofs::fs::inode::xattr_dispatch_safety` |

### Layer 2: TLA+ models

- `models/fs/inode_concurrent_link.tla` (mandatory per `fs/00-overview.md` Layer 2) — proves concurrent link / unlink on the same target inode doesn't corrupt the i_nlink counter. Owned here.

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-sb inode list | "Every linked inode is on its superblock's `s_inodes` list" | `kani::proofs::fs::inode::sb_list_invariants` |
| Inode hash | "Every inode in the hash table is reachable via the hash bucket indexed by its inode number" | `kani::proofs::fs::inode::hash_invariants` |
| LRU list | "Every inode on the LRU has refcount==0 + I_REFERENCED bit set" | `kani::proofs::fs::inode::lru_invariants` |
| ACL list | "Per-inode ACL is either NULL or refcount-managed" | `kani::proofs::fs::inode::acl_refcount_invariants` |

### Layer 4: Functional correctness (opt-in)

- **POSIX ACL inheritance** via Creusot — proves: child inode's ACL after creation matches the upstream-defined formula `(parent_default & mode_mask) | new_inode_default`.
- **idmap algebra** via Creusot — proves: round-trip uid through `from_vfsuid` + `to_vfsuid` preserves identity.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | `inode->i_count`, `inode->i_writecount`, ACL refcounts all use `Refcount` | § Mandatory |
| **AUTOSLAB** | inode allocated via per-FS `KmemCache::<MyFsInode>::new()` (ext4_inode_cachep, etc.); per-FS type tagging visible in `/proc/slabinfo` | § Mandatory |
| **MEMORY_SANITIZE** | inode cache freed objects zeroed unless `SLAB_NO_ZERO_ON_FREE` per-cache; sensitive FS (e.g., ecryptfs) MUST mark caches as zero-on-free | § Default-on configurable off |

### Row-1 features consumed by this component

- **UDEREF**: never accepts user pointers directly; xattr name + value go through `getname` + `vmemdup_user` (cross-ref `lib/usercopy.md`)
- **SIZE_OVERFLOW**: i_size + i_blocks arithmetic uses checked operators
- **CONSTIFY**: inode_operations vtables provided by per-FS modules are `static const`

### Row-2 / GR-RBAC integration

Major LSM-hook integration site:
- `security_inode_alloc` / `security_inode_free` — inode lifecycle
- `security_inode_create` / `security_inode_link` / `security_inode_unlink` / `security_inode_symlink` / `security_inode_mkdir` / `security_inode_rmdir` / `security_inode_mknod` / `security_inode_rename` — mutation hooks
- `security_inode_permission` — permission check
- `security_inode_setattr` / `security_inode_getattr`
- `security_inode_setxattr` / `security_inode_getxattr` / `security_inode_listxattr` / `security_inode_removexattr`
- `security_inode_set_acl` / `security_inode_get_acl`
- `security_inode_listsecurity`

GR-RBAC's TPE + chroot-hardening + path-restriction policies all consume these hooks.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Open Questions

(none — inode semantics are exhaustively specified by POSIX + Linux extensions)

## Out of Scope

- dcache (cross-ref `fs/vfs/dcache.md`)
- super_block (cross-ref `fs/vfs/super.md`)
- Per-FS-specific inode extensions (cross-ref individual fs/* docs)
- 32-bit-only paths
- Implementation code
