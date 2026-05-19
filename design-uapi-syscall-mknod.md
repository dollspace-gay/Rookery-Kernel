---
title: "Tier-5 syscall: mknod(2) — syscall 133"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`mknod(2)` creates a filesystem object of the kind specified by the upper bits of `mode`. It is the unified creation primitive behind regular files, named pipes (FIFOs), UNIX domain sockets, character device nodes, and block device nodes. The kernel resolves `path`'s parent directory, type-dispatches on `mode & S_IFMT`, and either calls `vfs_create` (regular file), `vfs_mkfifo` (FIFO), `vfs_mksock` (socket), or `vfs_mknod` (device). Creating character/block device nodes requires `CAP_MKNOD`. Critical for: `/dev` population, container runtime device pass-through, `tmpfs` device-node creation, classic FIFOs used by shells.

`mknod(2)` is the dirfd-less ancestor of `mknodat(2)`.

### Acceptance Criteria

- [ ] AC-1: `mknod("/tmp/f", S_IFREG | 0644, 0)`: creates regular file.
- [ ] AC-2: `mknod("/tmp/fifo", S_IFIFO | 0666, 0)`: creates FIFO; openable by reader+writer.
- [ ] AC-3: `mknod("/tmp/sock", S_IFSOCK | 0600, 0)`: creates socket inode.
- [ ] AC-4: `mknod("/dev/null2", S_IFCHR | 0666, makedev(1, 3))` as root: creates char node.
- [ ] AC-5: `mknod("/dev/foo", S_IFCHR | 0666, makedev(1, 3))` without CAP_MKNOD: `EPERM`.
- [ ] AC-6: `mknod("/tmp/blk", S_IFBLK | 0660, makedev(8, 0))` without CAP_MKNOD: `EPERM`.
- [ ] AC-7: `mknod("/tmp/x", S_IFMT, 0)`: `EINVAL`.
- [ ] AC-8: `mknod("/tmp/x", S_IFREG, 0)` when /tmp/x exists: `EEXIST`.
- [ ] AC-9: `mknod` on read-only fs: `EROFS`.
- [ ] AC-10: `mknod` with path too long: `ENAMETOOLONG`.
- [ ] AC-11: `mknod` with bad memory: `EFAULT`.
- [ ] AC-12: FIFO node created; readers/writers block until paired.

### Architecture

```rust
#[syscall(nr = 133, abi = "sysv")]
pub fn sys_mknod(path: UserPtr<u8>, mode: u32, dev: u64) -> isize {
    Mknod::do_mknodat(AT_FDCWD, path, mode, dev)
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

`Mknod::vfs_mknod(dir, leaf, mode, dev) -> Result<()>`:
1. if let Some(op) = dir.i_op.mknod {
2.   op(dir, leaf, mode, new_decode_dev(dev))?;
3.   Ok(())
4. } else {
5.   Err(EPERM)                                              // fs doesn't support device nodes
6. }

### Out of Scope

- `mknodat(2)` (covered in `mknodat.md` Tier-5).
- Device-driver chrdev/blkdev registration (covered in Tier-3 device-model docs).
- FIFO read/write semantics (covered in Tier-3 `fs/pipe.md`).
- Implementation code.

### signature

```c
int mknod(const char *path, mode_t mode, dev_t dev);
```

```c
/* mode & S_IFMT selects the kind: */
#define S_IFREG   0100000   /* regular file (dev ignored) */
#define S_IFCHR   0020000   /* character device (uses dev) */
#define S_IFBLK   0060000   /* block device (uses dev) */
#define S_IFIFO   0010000   /* FIFO/pipe (dev ignored) */
#define S_IFSOCK  0140000   /* UNIX socket (dev ignored) */
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `path` | `const char *` | in | NUL-terminated user-space pathname; intermediate symlinks followed. |
| `mode` | `mode_t` | in | Type bits (`S_IFMT`) + permission bits; masked by umask. Default type = `S_IFREG` if type bits absent. |
| `dev` | `dev_t` | in | For S_IFCHR/S_IFBLK: encoded major/minor (use `makedev()`). Ignored otherwise. |

### return value

| Value | Meaning |
|---|---|
| `0` | Success. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EACCES` | Write/search permission denied on parent dir. |
| `EDQUOT` | Quota exhausted. |
| `EEXIST` | `path` already exists. |
| `EFAULT` | `path` outside accessible memory. |
| `EINVAL` | `mode` requests an unsupported type, or `dev` has an invalid encoding. |
| `ELOOP` | Too many symbolic links. |
| `ENAMETOOLONG` | `path` exceeds `PATH_MAX`. |
| `ENOENT` | A directory component does not exist. |
| `ENOMEM` | Out of kernel memory. |
| `ENOSPC` | No space on filesystem. |
| `ENOTDIR` | A component of the prefix is not a directory. |
| `EPERM` | Type requires privilege (S_IFCHR / S_IFBLK ⟹ `CAP_MKNOD`); FS does not support requested type. |
| `EROFS` | Read-only filesystem. |

### abi surface

```text
__NR_mknod  (x86_64)  = 133
__NR_mknod  (i386)    = 14
/* arm64 / riscv: no __NR_mknod; userspace uses mknodat. */
```

### compatibility contract

REQ-1: Syscall number is **133** on x86_64; ABI-stable.

REQ-2: Type dispatch (`mode & S_IFMT`):
- `S_IFREG` or 0: create regular file (equivalent to `creat(path, mode)`).
- `S_IFCHR`: create character device node with `dev`.
- `S_IFBLK`: create block device node with `dev`.
- `S_IFIFO`: create FIFO.
- `S_IFSOCK`: create UNIX-domain socket inode (no listener until `bind`).
- Any other value ⟹ `EINVAL`.

REQ-3: Capability gating:
- `S_IFCHR`, `S_IFBLK` ⟹ require `CAP_MKNOD` in the user-namespace of the parent dir's filesystem.
- `S_IFREG`, `S_IFIFO`, `S_IFSOCK` ⟹ no extra capability beyond DAC.

REQ-4: `dev` encoding: `makedev(major, minor)` packs to `(major << MINORBITS) | minor`; on Linux `MINORBITS = 20`, supports 32-bit major/minor. Encoded value is filesystem-storable.

REQ-5: `mode` permission bits masked by `umask` BEFORE applied.

REQ-6: Filesystem may reject types it does not implement (e.g., ext2 supports all; some FUSE filesystems may not — returns `EPERM`).

REQ-7: Resolves `path`'s parent directory with `LOOKUP_PARENT | LOOKUP_FOLLOW`; the *leaf* must NOT exist.

REQ-8: LSM `inode_mknod` hook fires before creation; may deny with `EACCES`.

REQ-9: Path-string copy_from_user bounded by `PATH_MAX`.

REQ-10: Returned device node, when later opened, dispatches to the registered driver via `chrdev_open` / `blkdev_open`.

REQ-11: Inode created with ownership `(euid, egid)`; sgid-on-parent-dir inherited for `gid`.

REQ-12: No file descriptor is returned; node creation is name-only.

REQ-13: Forward-compat: future fs may extend `S_IFMT`; today kernel rejects unknown bits.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `type_dispatch_total` | INVARIANT | every S_IFMT bit pattern is either dispatched or rejected. |
| `cap_mknod_for_device` | INVARIANT | S_IFCHR / S_IFBLK ⟹ capable(CAP_MKNOD) checked before any work. |
| `umask_applied` | INVARIANT | applied mode == (mode & 07777) & ~umask. |
| `parent_path_no_create_on_error` | INVARIANT | any error ⟹ no inode created. |

### Layer 2: TLA+

`fs/mknod-syscall.tla`:
- States: per-copy_path, per-type-dispatch, per-cap-check, per-parent-resolve, per-lsm, per-fs-mknod.
- Properties:
  - `safety_cap_mknod_for_device` — device nodes require CAP_MKNOD.
  - `safety_unknown_type_rejected` — invalid S_IFMT ⟹ EINVAL.
  - `safety_no_partial_inode` — fail before fs-mknod ⟹ no inode left behind.
  - `liveness_mknod_terminates` — every call returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_mknodat` post: ok ⟹ node of correct type at leaf | `Mknod::do_mknodat` |
| `vfs_mknod` post: dev-encoded ⟹ filesystem sees major/minor | `Mknod::vfs_mknod` |

### Layer 4: Verus / Creusot functional

Per-`mknod(2)` POSIX semantics; selftests in `tools/testing/selftests/filesystems/`.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`mknod(2)` reinforcement:

- **Per-CAP_MKNOD gate on S_IFCHR / S_IFBLK** — defense against per-unprivileged-device-node creation.
- **Per-umask masking** — defense against per-permissive-default.
- **Per-LSM inode_mknod hook** — defense against per-MAC-bypass.
- **Per-type-dispatch exhaustive** — defense against per-unknown-S_IFMT smuggling.
- **Per-leaf-must-not-exist** — defense against per-EEXIST-overwrite race.

### grsecurity / pax-style reinforcement

- **PaX UDEREF on `path` copy_from_user** — defense against per-kernel-pointer path; SMAP forced.
- **CAP_MKNOD strict for S_IFCHR / S_IFBLK** — defense against per-container-escape via unauthorized device node; even root-in-userns without CAP_MKNOD-on-init-userns is rejected by grsec.
- **GRKERNSEC_CHROOT_MKNOD** — defense against per-chroot escape; mknod of any device node from within a chroot returns `EPERM` regardless of caps.
- **GRKERNSEC_DEVICE_SIDECHANNEL** — restricts device-node creation on tmpfs/ramfs mounted with `noexec,nosuid` to root in init_userns.
- **GRKERNSEC_LINK restricted symlink follow** — defense against per-mknod-via-symlink in sticky world-writable dirs.
- **GRKERNSEC_FIFO restrictive FIFO create** — FIFOs in sticky world-writable dirs require owner match (matches `fs.protected_fifos`).
- **PaX KERNEXEC** — generic kernel hardening; no direct mknod relevance.
- **PAX_USERCOPY_HARDEN on path copy** — bounded by PATH_MAX; whitelisted slab.
- **GRKERNSEC_TPE (Trusted Path Execution)** — limits where mknod-created executables may be invoked.
- **Per-dev_t encoding validated** — defense against per-bogus-dev_t fingerprint of internal majors.

