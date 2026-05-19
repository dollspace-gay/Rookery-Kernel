---
title: "Tier-5 syscall: mknodat(2) — syscall 259"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`mknodat(2)` generalizes `mknod(2)` over an optional starting **directory file descriptor**: `path` is resolved relative to `dirfd` (or the cwd if `dirfd == AT_FDCWD`), eliminating the TOCTOU window that exists in path-only `mknod`. All other semantics — type dispatch, capability gating, dev encoding, LSM hooks — are identical to `mknod`. The kernel and `mknod` share the same internal `do_mknodat` implementation; `mknod(path, mode, dev)` is literally `mknodat(AT_FDCWD, path, mode, dev)`. Critical for: openat-style safe creation, container runtimes setting up `/dev`, race-free directory-node creation.

### Acceptance Criteria

- [ ] AC-1: `mknodat(AT_FDCWD, "/tmp/f", S_IFREG | 0644, 0)`: equivalent to `mknod`.
- [ ] AC-2: `mknodat(dirfd, "child", S_IFIFO | 0666, 0)`: FIFO created under dirfd.
- [ ] AC-3: `mknodat(-1, "rel", ..., ...)`: `EBADF`.
- [ ] AC-4: `mknodat(filefd, "rel", ..., ...)`: `ENOTDIR`.
- [ ] AC-5: `mknodat(AT_FDCWD, "/tmp/c", S_IFCHR | 0666, makedev(1,3))` w/o CAP_MKNOD: `EPERM`.
- [ ] AC-6: `mknodat(AT_FDCWD, "/tmp/b", S_IFBLK | 0660, makedev(8,0))` w/o CAP_MKNOD: `EPERM`.
- [ ] AC-7: `mknodat(AT_FDCWD, "/tmp/x", S_IFMT, 0)`: `EINVAL`.
- [ ] AC-8: `mknodat(dirfd, "exists", S_IFREG, 0)`: `EEXIST`.
- [ ] AC-9: `mknodat` on read-only fs: `EROFS`.
- [ ] AC-10: `mknodat` with bad path memory: `EFAULT`.
- [ ] AC-11: O_PATH dirfd valid: success on a sub-path.
- [ ] AC-12: Chrooted caller: cannot escape via `..`.

### Architecture

```rust
#[syscall(nr = 259, abi = "sysv")]
pub fn sys_mknodat(dirfd: i32, path: UserPtr<u8>, mode: u32, dev: u64) -> isize {
    Mknod::do_mknodat(dirfd, path, mode, dev)
}
```

`Mknod::do_mknodat(dirfd, path_uptr, mode, dev) -> isize`:
1. let kpath = Mknod::copy_path_from_user(path_uptr)?;       // EFAULT, ENAMETOOLONG
2. let kind = mode & S_IFMT;
3. let perm = mode & 07777 & !current_umask();
4. match kind {
5.   0 | S_IFREG | S_IFCHR | S_IFBLK | S_IFIFO | S_IFSOCK => {}
6.   _ => return Err(EINVAL),
7. }
8. if matches!(kind, S_IFCHR | S_IFBLK) ∧ !capable(CAP_MKNOD) { return Err(EPERM); }
9. let (parent, leaf) = Path::resolve_parent_at(dirfd, &kpath, LOOKUP_FOLLOW)?;
10. parent.may_create_under(&current_creds())?;              // EACCES, EROFS
11. lsm_inode_mknod(parent.inode, leaf, mode, dev)?;         // EACCES
12. match kind {
13.   0 | S_IFREG  => Mknod::vfs_create(parent.inode, leaf, perm)?,
14.   S_IFCHR | S_IFBLK => Mknod::vfs_mknod(parent.inode, leaf, kind | perm, dev)?,
15.   S_IFIFO  => Mknod::vfs_mkfifo(parent.inode, leaf, perm)?,
16.   S_IFSOCK => Mknod::vfs_mksock(parent.inode, leaf, perm)?,
17.   _ => unreachable!(),
18. }
19. Ok(0)

`Path::resolve_parent_at(dirfd, path, flags) -> Result<(Path, &str)>`:
1. let mut nd = Nameidata::new(dirfd, path, flags | LOOKUP_PARENT);
2. while !nd.is_at_leaf() { walk_next(&mut nd)?; }
3. Ok((nd.parent_path, nd.leaf_name))

### Out of Scope

- `mknod(2)` (covered in `mknod.md` Tier-5).
- Device-driver chrdev/blkdev registration (covered in Tier-3 device-model docs).
- Implementation code.

### signature

```c
int mknodat(int dirfd, const char *path, mode_t mode, dev_t dev);
```

```c
/* mode & S_IFMT selects the kind: */
#define S_IFREG   0100000
#define S_IFCHR   0020000
#define S_IFBLK   0060000
#define S_IFIFO   0010000
#define S_IFSOCK  0140000
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `dirfd` | `int` | in | Directory fd, or `AT_FDCWD` for cwd. |
| `path` | `const char *` | in | NUL-terminated pathname relative to `dirfd`; absolute paths ignore `dirfd`. |
| `mode` | `mode_t` | in | Type bits (`S_IFMT`) + permission bits; masked by umask. |
| `dev` | `dev_t` | in | For `S_IFCHR`/`S_IFBLK`: encoded major/minor. Ignored otherwise. |

### return value

| Value | Meaning |
|---|---|
| `0` | Success. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EACCES` | Search/write permission denied on parent. |
| `EBADF` | `dirfd` invalid and `path` is not absolute. |
| `EDQUOT` | Quota exhausted. |
| `EEXIST` | Path already exists. |
| `EFAULT` | `path` outside accessible memory. |
| `EINVAL` | Unsupported type in `mode`, or bad `dev` encoding. |
| `ELOOP` | Too many symbolic links. |
| `ENAMETOOLONG` | `path` exceeds `PATH_MAX`. |
| `ENOENT` | A component does not exist. |
| `ENOMEM` | Out of kernel memory. |
| `ENOSPC` | No space on filesystem. |
| `ENOTDIR` | `dirfd` refers to a non-directory and `path` is relative; or a path component is not a directory. |
| `EPERM` | Type requires CAP_MKNOD; or filesystem does not support type. |
| `EROFS` | Read-only filesystem. |

### abi surface

```text
__NR_mknodat (x86_64)  = 259
__NR_mknodat (i386)    = 297
__NR_mknodat (arm64)   = 33
__NR_mknodat (riscv)   = 33
```

### compatibility contract

REQ-1: Syscall number is **259** on x86_64; ABI-stable.

REQ-2: Resolution semantics:
- `dirfd == AT_FDCWD` ⟹ resolve relative to cwd.
- absolute `path` ⟹ `dirfd` is not consulted for resolution.
- relative `path` ⟹ resolve relative to `dirfd`.

REQ-3: Equivalent to `mknod(path, mode, dev)` when `dirfd == AT_FDCWD`.

REQ-4: Type dispatch (`mode & S_IFMT`): identical to `mknod` — `S_IFREG`/`S_IFCHR`/`S_IFBLK`/`S_IFIFO`/`S_IFSOCK`; unknown ⟹ `EINVAL`.

REQ-5: `S_IFCHR` / `S_IFBLK` require `CAP_MKNOD` in the userns of the parent's filesystem.

REQ-6: `dev` encoding: `makedev(major, minor)`; on Linux MINORBITS = 20.

REQ-7: `mode` permission bits masked by umask.

REQ-8: `dirfd` ref via `fdget(dirfd)`; `fdput` on return; `O_PATH` fds are valid as `dirfd`.

REQ-9: Path-string copy_from_user bounded by `PATH_MAX`.

REQ-10: LSM `inode_mknod` hook fires before creation.

REQ-11: Resolves `path`'s parent directory with `LOOKUP_PARENT | LOOKUP_FOLLOW`; leaf must not exist.

REQ-12: Inode ownership inherits `(euid, egid)` (or sgid-on-parent-dir).

REQ-13: No file descriptor returned; node creation is name-only.

REQ-14: Forward-compat: no `flags` argument; future extension via `openat2`-style or new syscall.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `type_dispatch_total` | INVARIANT | every S_IFMT bit pattern dispatched or rejected. |
| `cap_mknod_for_device` | INVARIANT | S_IFCHR/S_IFBLK ⟹ capable(CAP_MKNOD) checked. |
| `umask_applied` | INVARIANT | applied mode == (mode & 07777) & ~umask. |
| `dirfd_get_balanced` | INVARIANT | fdget/fdput paired. |
| `no_partial_inode` | INVARIANT | any error ⟹ no leaf inode. |

### Layer 2: TLA+

`fs/mknodat-syscall.tla`:
- States: per-copy_path, per-type-dispatch, per-cap-check, per-dirfd_get, per-resolve, per-lsm, per-vfs-mknod, per-dirfd_put.
- Properties:
  - `safety_cap_mknod_for_device` — device nodes require CAP_MKNOD.
  - `safety_dirfd_lifetime` — fdput follows fdget.
  - `safety_no_partial_inode` — fail before vfs-mknod ⟹ no inode.
  - `liveness_mknodat_terminates` — every call returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_mknodat` post: ok ⟹ node of correct type at leaf | `Mknod::do_mknodat` |
| `resolve_parent_at` post: returns directory parent + leaf | `Path::resolve_parent_at` |

### Layer 4: Verus / Creusot functional

Per-`mknodat(2)` POSIX-2008 semantics; selftests in `tools/testing/selftests/filesystems/`.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`mknodat(2)` reinforcement:

- **Per-CAP_MKNOD gate on S_IFCHR / S_IFBLK** — defense against per-unprivileged-device-node creation.
- **Per-dirfd fdget/fdput balanced** — defense against per-dirfd-UAF.
- **Per-umask masking** — defense against per-permissive-default.
- **Per-LSM inode_mknod hook** — defense against per-MAC-bypass.
- **Per-leaf-must-not-exist** — defense against per-EEXIST-overwrite race.
- **Per-relative-path-resolved-under-dirfd** — defense against per-TOCTOU between resolve and create.

### grsecurity / pax-style reinforcement

- **PaX UDEREF on `path` copy_from_user** — defense against per-kernel-pointer path; SMAP forced.
- **CAP_MKNOD strict for S_IFCHR / S_IFBLK** — defense against per-container-escape via unauthorized device node; root-in-userns without init-userns CAP_MKNOD is rejected.
- **GRKERNSEC_CHROOT_MKNOD** — defense against per-chroot escape; mknodat of any device node from within a chroot returns `EPERM` regardless of caps.
- **GRKERNSEC_DEVICE_SIDECHANNEL** — restricts device-node creation on tmpfs/ramfs with `nosuid`.
- **GRKERNSEC_LINK restricted symlink follow** — defense against per-mknodat-via-symlink in sticky world-writable dirs.
- **GRKERNSEC_FIFO restrictive FIFO create** — owner-match enforcement.
- **GRKERNSEC_TPE** — limits where mknodat-created executables may be invoked.
- **PAX_USERCOPY_HARDEN on path copy** — bounded by PATH_MAX; whitelisted slab.
- **Per-dirfd-ownership table strict** — defense against per-cross-process fd injection.
- **Per-dev_t encoding validated** — defense against per-bogus-dev_t fingerprint of internal majors.

