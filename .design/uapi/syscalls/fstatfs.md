# Tier-5 syscall: fstatfs(2) — syscall 138

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/statfs.c (SYSCALL_DEFINE2(fstatfs), vfs_statfs)
  - fs/file.c (fget)
  - include/uapi/linux/statfs.h (struct statfs, struct statfs64)
  - arch/x86/entry/syscalls/syscall_64.tbl (138  common  fstatfs)
-->

## Summary

`fstatfs(2)` is the file-descriptor variant of `statfs(2)`. It returns filesystem-level metadata for the filesystem containing the file referenced by `fd`, bypassing pathname resolution entirely. Used by tools that already hold an open fd (often inherited from a more-privileged parent, or obtained via `openat`+`O_PATH`) and want filesystem statistics without re-walking the path. Also the canonical primitive when the calling process has chroot'd or unshared mounts in a way that makes path lookup unreliable. Critical for: container runtimes resolving FS info via fd-inheritance, sandboxed processes querying their cwd-FS, observability without TOCTOU on pathname.

## Signature

```c
int fstatfs(int fd, struct statfs *buf);
```

```c
struct statfs {
    __SWORD_TYPE f_type;
    __SWORD_TYPE f_bsize;
    fsblkcnt_t   f_blocks;
    fsblkcnt_t   f_bfree;
    fsblkcnt_t   f_bavail;
    fsfilcnt_t   f_files;
    fsfilcnt_t   f_ffree;
    __kernel_fsid_t f_fsid;
    __SWORD_TYPE f_namelen;
    __SWORD_TYPE f_frsize;
    __SWORD_TYPE f_flags;
    __SWORD_TYPE f_spare[4];
};
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `fd` | `int` | in | Open file descriptor (any type: regular, dir, symlink-via-O_PATH, socket, pipe). |
| `buf` | `struct statfs *` | out | Caller-allocated; kernel fills with FS metadata of fd's superblock. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Success; `buf` filled. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EBADF` | `fd` is not a valid open fd. |
| `EFAULT` | `buf` userptr fault. |
| `EOVERFLOW` | Some FS counter doesn't fit in 32-bit `statfs` field. |
| `ENOSYS` | FS does not implement `statfs`. |
| `EIO` | Lower-layer FS error. |
| `ENOENT` | (rare) FD refers to a deleted-and-cleaned-up superblock. |

## ABI surface

```text
__NR_fstatfs (x86_64) = 138
__NR_fstatfs (arm64)  = N/A (uses fstatfs64 unified)
__NR_fstatfs (i386)   = 100

/* 64-bit kernels alias fstatfs to fstatfs64 internally. */
/* fstatfs operates on the SUPERBLOCK of the file's filesystem, not on the file. */
/* fstatfs on a pipe/socket fd returns the appropriate synthetic FS magic
   (e.g., PIPEFS_MAGIC, SOCKFS_MAGIC). */
```

## Compatibility contract

REQ-1: Syscall number is **138** on x86_64. ABI-stable.

REQ-2: `fd` MUST be open in caller's fdtable; otherwise `-EBADF`. O_PATH fds ARE valid for fstatfs (this is the canonical race-free FS-stat primitive).

REQ-3: No permission check on the file itself: holding an fd is the authorization. (Distinct from `fstat(2)` which has the same semantics.)

REQ-4: Superblock dispatch: `vfs_statfs(&file_path, &st) -> sb->s_op->statfs(dentry, &st)`.

REQ-5: O_PATH semantics: an fd opened with `O_PATH` is sufficient; the FS is queried via the dentry's superblock without needing read/write rights.

REQ-6: Per-pipe/socket/anon-inode FD: returns the synthetic-FS magic (PIPEFS_MAGIC = 0x50495045, SOCKFS_MAGIC = 0x534F434B, ANON_INODE_FS_MAGIC = 0x09041934).

REQ-7: `f_flags` reports the mount flags of the **mount** the fd was opened through, not the global superblock flags (a bind-mount with `ro,nosuid` overlay reports those flags here, not the underlying FS's).

REQ-8: Per-overlayfs: same overlay semantics as `statfs(2)` (lower-layer underlying FS by default).

REQ-9: Per-`statfs` vs `statfs64`: same overflow story as `statfs(2)`; use `fstatfs64` on 32-bit for >4TB.

REQ-10: Per-mount-namespace: the fd carries its own mount reference; mount-namespace of caller is NOT consulted (the mount that the fd was opened against is what's used).

REQ-11: Per-deleted-FS: if the superblock has been torn down (unmount in progress), returns `-EIO` or `-ENOENT`.

REQ-12: Per-statx-with-AT_STATX_FORCE_SYNC: `fstatfs` does not honor statx-style sync flags; it is a metadata-only read.

## Acceptance Criteria

- [ ] AC-1: `fd = open("/")`; `fstatfs(fd, &st)`: returns 0; f_type matches root FS.
- [ ] AC-2: `fd = open(".", O_PATH)`; `fstatfs(fd, &st)`: returns 0 (O_PATH valid).
- [ ] AC-3: `fd = pipe[0]`; `fstatfs(fd, &st)`: `st.f_type == PIPEFS_MAGIC`.
- [ ] AC-4: `fd = socket(AF_INET, SOCK_STREAM, 0)`; `fstatfs(fd, &st)`: `st.f_type == SOCKFS_MAGIC`.
- [ ] AC-5: `fd = -1`: `-EBADF`.
- [ ] AC-6: `fd = 999` (closed): `-EBADF`.
- [ ] AC-7: NULL buf: `-EFAULT`.
- [ ] AC-8: 32-bit caller, >4TB FS: `-EOVERFLOW`.
- [ ] AC-9: Bind-mount ro overlay: `st.f_flags & ST_RDONLY` reflects the mount, not underlying FS.
- [ ] AC-10: fd opened then FS lazy-unmounted: subsequent fstatfs returns 0 (sb still alive via fd ref).

## Architecture

```rust
#[syscall(nr = 138, abi = "sysv")]
pub fn sys_fstatfs(fd: i32, buf: UserPtr<Statfs>) -> isize {
    Fstatfs::do_fstatfs(fd, buf)
}
```

`Fstatfs::do_fstatfs(fd, buf_ptr) -> isize`:
1. let file = FdTable::fget(fd).ok_or(-EBADF)?;
2. let mut st = KernelStatfs::default();
3. Fstatfs::vfs_statfs_file(&file, &mut st)?;       // ENOSYS/EIO
4. let user_st = UserStatfs::from_kernel(&st).ok_or(-EOVERFLOW)?;
5. buf_ptr.write(&user_st)?;                         // EFAULT
6. 0

`Fstatfs::vfs_statfs_file(file, st) -> Result<()>`:
1. let path = &file.f_path;
2. let sb = path.dentry.d_sb;
3. if sb.s_op.statfs.is_none() { return Err(ENOSYS); }
4. (sb.s_op.statfs)(&path.dentry, st)?;
5. /* Per-mount flags overlay */
6. st.f_flags = mount_flags_to_st(path.mnt.mnt_flags);
7. Ok(())

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `fd_validity` | INVARIANT | invalid fd ⟹ EBADF. |
| `buf_write_bounded` | INVARIANT | buf write bounded by sizeof(Statfs). |
| `o_path_accepted` | INVARIANT | O_PATH fd accepted. |
| `mount_flags_from_mnt` | INVARIANT | f_flags derived from path.mnt, not sb. |
| `overflow_detected` | INVARIANT | narrow field overflow ⟹ EOVERFLOW. |

### Layer 2: TLA+

`fs/fstatfs.tla`:
- States: per-fget, per-sb-dispatch, per-narrow-convert, per-write-user.
- Properties:
  - `safety_no_path_walk` — fstatfs never performs path resolution.
  - `safety_o_path_accepted` — O_PATH fd succeeds.
  - `safety_overflow_detected` — narrow fields never silently truncate.
  - `liveness_returns` — every call terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_fstatfs` post: success ⟹ buf filled | `Fstatfs::do_fstatfs` |
| `vfs_statfs_file` post: f_flags reflects mount | `Fstatfs::vfs_statfs_file` |

### Layer 4: Verus / Creusot functional

Per-`fstatfs(2)` man-page semantic equivalence. LTP `fstatfs01..02` pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`fstatfs(2)` reinforcement:

- **Per-fget refcount strict** — defense against per-file UAF during dispatch.
- **Per-O_PATH accepted (no read perm required)** — defense against per-TOCTOU on pathname re-walk.
- **Per-mount-flags overlay (mnt, not sb)** — defense against per-flag-leak from underlying FS.
- **Per-overflow strict EOVERFLOW** — defense against per-silent counter truncation.
- **Per-userptr SMAP/UDEREF buf write** — defense against per-pointer kernel deref.

## Grsecurity / PaX surface

- **PaX UDEREF on `buf` copy_to_user** — defense against per-buf-pointer kernel deref; SMAP forced.
- **PAX_USERCOPY_HARDEN on Statfs write** — bounded into whitelisted slab/stack; rejects oversized writes to wrong slab.
- **GRKERNSEC_PROC restrictions for fstatfs** — synthetic /proc and /sys fstatfs returns zero counters for non-root callers; defense against per-procfs fingerprint via fd.
- **GRKERNSEC_HIDESYM on f_fsid** — `f_fsid` zeroed for non-root callers; defense against per-FS instance fingerprint leak via fd-inherited handle.
- **PaX KERNEXEC on `sb->s_op->statfs` indirect call** — RAP-tagged; defense against per-ROP gadget.
- **PAX_REFCOUNT on superblock refcount during dispatch** — defense against per-sb refcount overflow UAF.
- **GRKERNSEC_AUDIT_FSTATFS** — kernel.audit emits ANOM_FSTATFS record per call (configurable allowlist for sensitive mounts).
- **GRKERNSEC_FD_PROTECT on closed-fd race** — defense against per-fdt-race where another thread closes during fget; fget already strict.
- **PaX MEMORY_SANITIZE on `KernelStatfs` stack buffer** — defense against per-uninit-data leak from FS impl that fails to fill all fields.
- **GRKERNSEC_CHROOT** consistency — fd inherited from outside chroot does NOT leak parent-FS counters when fstatfs called inside chroot (the fd carries its own ref; this is by design but logged).
- **GRKERNSEC_LOG_RATE_LIMIT on EBADF** — defense against per-log-flood probing for valid fds.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- Per-FS `s_op->statfs` implementations (covered in per-FS Tier-3 docs).
- Per-`statx(2)` (covered in `statx.md`).
- O_PATH semantics (covered in `open.md` Tier-5).
- Implementation code.
