# Tier-5 syscall: statfs(2) — syscall 137

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/statfs.c (SYSCALL_DEFINE2(statfs), do_statfs_native, vfs_statfs)
  - fs/namei.c (user_path_at)
  - include/uapi/linux/statfs.h (struct statfs, struct statfs64)
  - arch/x86/entry/syscalls/syscall_64.tbl (137  common  statfs)
-->

## Summary

`statfs(2)` returns filesystem-level metadata about the filesystem containing the named path: block size, total / free / available blocks, total / free inodes, filesystem type magic, mount flags, and filesystem ID. Used by tooling that needs whole-filesystem statistics (`df`, container quota probes, package managers checking install-target free space, monitoring agents). It is path-based; the kernel walks the path, dereferences the final dentry, locates the superblock, and dispatches to `sb->s_op->statfs`. Critical for: storage observability, install/upgrade safety checks, container free-space accounting.

## Signature

```c
int statfs(const char *path, struct statfs *buf);
```

```c
struct statfs {
    __SWORD_TYPE f_type;     /* filesystem magic (see <linux/magic.h>) */
    __SWORD_TYPE f_bsize;    /* optimal block size */
    fsblkcnt_t   f_blocks;
    fsblkcnt_t   f_bfree;
    fsblkcnt_t   f_bavail;   /* free for unprivileged */
    fsfilcnt_t   f_files;
    fsfilcnt_t   f_ffree;
    __kernel_fsid_t f_fsid;
    __SWORD_TYPE f_namelen;
    __SWORD_TYPE f_frsize;
    __SWORD_TYPE f_flags;    /* ST_RDONLY, ST_NOSUID, ... */
    __SWORD_TYPE f_spare[4];
};
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `path` | `const char *` | in | NUL-terminated pathname; AT_FDCWD-rooted for relative paths. |
| `buf` | `struct statfs *` | out | Caller-allocated; kernel fills with FS metadata. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Success; `buf` filled. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EFAULT` | `path` or `buf` userptr fault. |
| `ENAMETOOLONG` | `path` exceeds `PATH_MAX`. |
| `ENOENT` | A path component does not exist. |
| `ENOTDIR` | A non-final component is not a directory. |
| `EACCES` | Search permission denied on a path component. |
| `ELOOP` | Symlink loop. |
| `EOVERFLOW` | Some FS counter doesn't fit in 32-bit `statfs` field (use `statfs64`). |
| `ENOSYS` | FS does not implement `statfs`. |
| `EIO` | Lower-layer FS error reading superblock. |

## ABI surface

```text
__NR_statfs (x86_64) = 137
__NR_statfs (arm64)  = N/A (uses statfs64 unified)
__NR_statfs (i386)   = 99

/* 64-bit kernels alias statfs to statfs64 internally. */
/* On 32-bit, fsblkcnt_t may be 32-bit ⟹ EOVERFLOW for >4TB filesystems; use statfs64. */
```

## Compatibility contract

REQ-1: Syscall number is **137** on x86_64. ABI-stable.

REQ-2: `path` is resolved relative to AT_FDCWD for relative paths; absolute paths root at caller's chroot.

REQ-3: Final component is dereferenced (follows symlinks). Use `lstat`+`statfs` chain or explicit AT_SYMLINK_NOFOLLOW via `fstatat`+`fstatfs` for no-follow.

REQ-4: Permission: search permission (x bit) required on every directory in path; no permission required on the target itself.

REQ-5: Kernel walks path via `user_path_at(AT_FDCWD, path, LOOKUP_FOLLOW, &path_struct)`.

REQ-6: Superblock dispatch: `vfs_statfs(&path_struct, &st) -> sb->s_op->statfs(dentry, &st)`.

REQ-7: Common per-FS implementations:
- ext4: `ext4_statfs` — reads superblock counters.
- xfs: `xfs_fs_statfs` — reads AG headers.
- btrfs: `btrfs_statfs` — accounts profile (raid0/1/10 multiplier).
- tmpfs: `shmem_statfs` — uses internal sb_info counters.
- procfs / sysfs / cgroupfs: synthetic, zero counters.

REQ-8: Per-`statfs` vs `statfs64`: on 64-bit kernels they are unified; on 32-bit, `statfs` may EOVERFLOW for >4TB fields.

REQ-9: `f_fsid` is FS-implementation-defined; some FS leave it zero for security (avoids fingerprinting).

REQ-10: `f_flags` reports mount flags: `ST_RDONLY`, `ST_NOSUID`, `ST_NODEV`, `ST_NOEXEC`, `ST_SYNCHRONOUS`, `ST_MANDLOCK`, `ST_NOATIME`, `ST_NODIRATIME`, `ST_RELATIME`.

REQ-11: Per-mount-namespace: path resolution is in caller's mount namespace; sibling-mnt-ns paths invisible.

REQ-12: Per-quota: `f_bavail` reflects unprivileged-reservation deduction (root sees `f_bfree`).

REQ-13: Per-overlayfs: returns lower-layer underlying FS statfs by default; some overlay configs report overlay-merged free.

REQ-14: Per-statx: modern callers prefer `statx(2)` for file-level metadata; `statfs(2)` remains the FS-level primitive.

## Acceptance Criteria

- [ ] AC-1: `statfs("/", &st)`: returns 0; `st.f_blocks > 0`; `st.f_type == EXT4_SUPER_MAGIC` (or actual root FS magic).
- [ ] AC-2: `statfs("/tmp", &st)` on tmpfs: `st.f_type == TMPFS_MAGIC`; `st.f_bsize == PAGE_SIZE`.
- [ ] AC-3: Nonexistent path: `-ENOENT`.
- [ ] AC-4: Path > PATH_MAX: `-ENAMETOOLONG`.
- [ ] AC-5: Symlink loop: `-ELOOP`.
- [ ] AC-6: NULL buf pointer: `-EFAULT`.
- [ ] AC-7: 32-bit caller, >4TB FS: `-EOVERFLOW` (must use statfs64).
- [ ] AC-8: Path with no search perm on intermediate dir: `-EACCES`.
- [ ] AC-9: Read-only mount: `st.f_flags & ST_RDONLY` set.
- [ ] AC-10: f_bavail < f_bfree on quota-reserved FS.

## Architecture

```rust
#[syscall(nr = 137, abi = "sysv")]
pub fn sys_statfs(path: UserPtr<u8>, buf: UserPtr<Statfs>) -> isize {
    Statfs::do_statfs(path, buf)
}
```

`Statfs::do_statfs(path_ptr, buf_ptr) -> isize`:
1. let path = UserPath::import(path_ptr)?;          // EFAULT/ENAMETOOLONG
2. let p = Namei::user_path_at(AT_FDCWD, &path, LookupFlags::FOLLOW)?; // ENOENT/ENOTDIR/EACCES/ELOOP
3. let mut st = KernelStatfs::default();
4. Statfs::vfs_statfs(&p, &mut st)?;                // ENOSYS/EIO
5. /* On 32-bit narrow fields: detect overflow. */
6. let user_st = UserStatfs::from_kernel(&st).ok_or(-EOVERFLOW)?;
7. buf_ptr.write(&user_st)?;                        // EFAULT
8. 0

`Statfs::vfs_statfs(path, st) -> Result<()>`:
1. let sb = path.dentry.d_sb;
2. let dentry = &path.dentry;
3. if sb.s_op.statfs.is_none() { return Err(ENOSYS); }
4. (sb.s_op.statfs)(dentry, st)?;
5. /* Fill kernel-side common fields */
6. st.f_namelen = st.f_namelen.max(sb.s_max_links_or_default());
7. st.f_flags = mount_flags_to_st(path.mnt.mnt_flags);
8. Ok(())

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `buf_writable` | INVARIANT | buf pointer access bounded by sizeof(Statfs). |
| `path_lookup_follow` | INVARIANT | statfs uses LOOKUP_FOLLOW on final component. |
| `overflow_detected` | INVARIANT | 32-bit narrow field overflow ⟹ EOVERFLOW. |
| `superblock_dispatched` | INVARIANT | sb->s_op->statfs called when present; ENOSYS else. |
| `permission_search` | INVARIANT | search perm checked on each intermediate dir. |

### Layer 2: TLA+

`fs/statfs.tla`:
- States: per-import-path, per-path-walk, per-permission-check, per-sb-dispatch, per-narrow-convert, per-write-user.
- Properties:
  - `safety_path_search_perm` — every intermediate dir checked.
  - `safety_overflow_detected` — narrow fields never silently truncate.
  - `safety_user_buf_write_bounded` — buf write bounded.
  - `liveness_returns` — every call terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_statfs` post: success ⟹ buf filled in full | `Statfs::do_statfs` |
| `vfs_statfs` post: f_type set; f_blocks ≥ f_bfree ≥ f_bavail | `Statfs::vfs_statfs` |

### Layer 4: Verus / Creusot functional

Per-`statfs(2)` man-page semantic equivalence. LTP `statfs01..03` pass; xfstests pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`statfs(2)` reinforcement:

- **Per-path strict length bound (PATH_MAX)** — defense against per-pathwalk-DoS.
- **Per-search-perm enforced** — defense against per-path-component-bypass.
- **Per-overflow strict EOVERFLOW** — defense against per-silent counter truncation.
- **Per-userptr SMAP/UDEREF buf write** — defense against per-pointer kernel deref.
- **Per-mount-ns scoped** — defense against per-cross-ns path leak.
- **Per-fsid zero-by-default** — defense against per-FS-fingerprint leak.

## Grsecurity / PaX surface

- **PaX UDEREF on `path` and `buf` copy** — defense against per-user-pointer kernel deref; SMAP forced.
- **PAX_USERCOPY_HARDEN on Statfs write to user** — bounded into whitelisted slab/stack range; rejects oversized writes mapped to wrong slab.
- **GRKERNSEC_PROC restrictions for statfs** — synthetic /proc and /sys statfs returns ENOSYS or zero counters for non-root callers; defense against per-procfs-fingerprint leak (uptime, mem state).
- **GRKERNSEC_CHROOT_FCHMOD-style restrictions** — chroot'd callers cannot statfs paths outside the chroot via parent traversal; same gating reused.
- **GRKERNSEC_MOUNT_NS_STRICT** — statfs in sibling mount-ns returns ENOENT (not EACCES) to avoid existence leak.
- **PaX KERNEXEC on `vfs_statfs` indirect call to `sb->s_op->statfs`** — RAP-tagged dispatch; defense against per-ROP gadget.
- **GRKERNSEC_HIDESYM on f_fsid** — `f_fsid` is zeroed for non-root callers; defense against per-FS-instance fingerprint.
- **PAX_REFCOUNT on superblock refcount during dispatch** — defense against per-sb refcount overflow UAF.
- **GRKERNSEC_AUDIT_STATFS** — kernel.audit emits ANOM_STATFS record per call to sensitive paths (configurable allowlist).
- **GRKERNSEC_FIFO/SOCKET strict** — statfs on a fifo/socket path follows symlink rules same as other syscalls; no special bypass.
- **PaX MEMORY_SANITIZE on `KernelStatfs` stack buffer** — defense against per-uninit-data leak from kernel statfs implementation that fails to fill all fields.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- Per-FS `s_op->statfs` implementations (covered in per-FS Tier-3 docs).
- Per-statx(2) (covered in `statx.md`).
- Per-mount-namespace details (covered in Tier-3 `fs/namespace.md`).
- Implementation code.
