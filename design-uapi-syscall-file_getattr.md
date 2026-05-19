---
title: "Tier-5 syscall: file_getattr(2) — syscall 467"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`file_getattr(2)` retrieves the filesystem-level inode attributes (extended attribute *flags* of the inode, project id, extent-size hints) for a file identified by `dirfd` + `path`, without opening the file. It is the syscall successor to the long-standing `FS_IOC_FSGETXATTR` and `FS_IOC_GETFLAGS` ioctls, exposing a single typed `struct file_attr` and adopting the modern `dirfd, path, at_flags` parameter pattern of `openat2(2)` / `fstatat(2)`.

The `usize` parameter is the **caller-declared size** of `struct file_attr`, allowing forward/backward compatibility: a future kernel may add fields, an older userspace gets the prefix it understands.

Critical for: XFS / ext4 / Btrfs project-quota tooling (`xfs_quota`, `chattr`/`lsattr`), immutable / append-only flag introspection, `FS_XFLAG_DAX` and CoW-extent management, replacement of the per-fs ioctl maze with a single ABI.

### Acceptance Criteria

- [ ] AC-1: `file_getattr(AT_FDCWD, "f", ...)` on an immutable file returns `fa_xflags & FS_XFLAG_IMMUTABLE`.
- [ ] AC-2: Path with no project quota: `fa_projid == 0`.
- [ ] AC-3: `usize = 24` (V0) succeeds.
- [ ] AC-4: `usize = 8` returns `-EINVAL`.
- [ ] AC-5: `usize = 1024` succeeds; kernel zeros trailing 1000 bytes.
- [ ] AC-6: `at_flags = 0x4` (unknown) returns `-EINVAL`.
- [ ] AC-7: `path == ""` without `AT_EMPTY_PATH` returns `-ENOENT`.
- [ ] AC-8: Symlink final with `AT_SYMLINK_NOFOLLOW` returns the symlink's own attrs.
- [ ] AC-9: tmpfs path: `-EOPNOTSUPP`.
- [ ] AC-10: No path search permission: `-EACCES`.

### Architecture

```rust
#[syscall(nr = 467, abi = "sysv")]
pub fn sys_file_getattr(
    dirfd: i32,
    path: UserPtr<u8>,
    attr_p: UserPtr<u8>,
    usize: usize,
    at_flags: u32,
) -> isize {
    FileAttr::get(dirfd, path, attr_p, usize, at_flags)
}
```

`FileAttr::get(dirfd, path_p, attr_p, usize, at_flags) -> isize`:
1. if (at_flags & !(AT_SYMLINK_NOFOLLOW | AT_EMPTY_PATH)) != 0 { return Err(EINVAL); }
2. const MIN: usize = FILE_ATTR_SIZE_VER0;
3. const MAX: usize = size_of::<FileAttr>();
4. if usize < MIN { return Err(EINVAL); }
5. let path = path_p.read_cstr_max(PATH_MAX)?;                  // EFAULT / ENAMETOOLONG
6. let target = Path::resolve(current, dirfd, &path, at_flags)?; // ENOENT / EACCES / ELOOP
7. let inode = target.inode();
8. let getattr = inode.fileattr_get_op().ok_or(EOPNOTSUPP)?;
9. let kattr: FileAttr = getattr(&inode)?;                      // filesystem call
10. let copy_n = usize.min(MAX);
11. attr_p.copy_out(&kattr, copy_n)?;
12. if usize > MAX {
13.    attr_p.add(MAX).zero(usize - MAX)?;
14. }
15. Ok(0)

`Path::resolve(current, dirfd, path, at_flags) -> Result<Path>`:
1. /* Same kern_path_at family used by fstatat(2) */
2. kern_path_at(current, dirfd, path, at_flags)

### Out of Scope

- `file_setattr(2)` (see `file_setattr.md`).
- Per-filesystem `fileattr_get` implementation (Tier-3 `fs/xfs/`, `fs/ext4/`).
- Legacy `FS_IOC_FSGETXATTR` / `FS_IOC_GETFLAGS` ioctls (Tier-4 `fs/ioctl.md`).
- Implementation code.

### signature

```c
int file_getattr(
    int dirfd,
    const char *path,
    struct file_attr *attr,
    size_t usize,
    unsigned int at_flags
);
```

```c
struct file_attr {
    __u64 fa_xflags;        /* FS_XFLAG_* */
    __u32 fa_extsize;       /* extent size hint, bytes */
    __u32 fa_nextents;      /* extents in use */
    __u32 fa_projid;        /* project id */
    __u32 fa_cowextsize;    /* CoW extent size hint */
};

/* fa_xflags bits */
#define FS_XFLAG_IMMUTABLE       0x00000008
#define FS_XFLAG_APPEND          0x00000010
#define FS_XFLAG_SYNC            0x00000020
#define FS_XFLAG_NOATIME         0x00000040
#define FS_XFLAG_NODUMP          0x00000080
#define FS_XFLAG_PROJINHERIT     0x00000200
#define FS_XFLAG_DAX             0x00008000
#define FS_XFLAG_COWEXTSIZE      0x00010000
/* ... see linux/fs.h ... */

/* at_flags */
#define AT_SYMLINK_NOFOLLOW   0x100
#define AT_EMPTY_PATH         0x1000
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `dirfd` | `int` | in | Directory fd anchoring `path`. `AT_FDCWD` uses the cwd. |
| `path` | `const char *` | in | Path relative to `dirfd`. May be `""` with `AT_EMPTY_PATH`. |
| `attr` | `struct file_attr *` | out | Receives the inode attributes. |
| `usize` | `size_t` | in | Caller-declared `sizeof(*attr)`. Kernel uses `min(usize, sizeof(struct file_attr))` for the copy and validates trailing kernel bytes are returned zero. |
| `at_flags` | `unsigned int` | in | `AT_SYMLINK_NOFOLLOW`, `AT_EMPTY_PATH`. Unknown bits ⟹ `-EINVAL`. |

### return value

| Value | Meaning |
|---|---|
| `0` | Success; `*attr` populated. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EFAULT` | `path` or `attr` invalid. |
| `EINVAL` | Bad `at_flags`, `usize` smaller than the minimum supported `struct file_attr`. |
| `EBADF` | `dirfd` not a valid dirfd (and `path` is relative). |
| `ENOENT` | `path` does not exist. |
| `EACCES` | Search permission denied along `path`. |
| `ENOTDIR` | Component of `path` is not a directory. |
| `ELOOP` | Symlink loop or `AT_SYMLINK_NOFOLLOW` and final is a symlink. |
| `ENAMETOOLONG` | Path too long. |
| `EOPNOTSUPP` | Filesystem does not implement the `getattr` op. |
| `EPERM` | Caller lacks capability to read the requested attributes (some projects). |

### abi surface

```text
__NR_file_getattr  (x86_64) = 467
__NR_file_getattr  (arm64)  = 467
__NR_file_getattr  (riscv)  = 467
__NR_file_getattr  (i386)   = 467

/* struct file_attr is forward-extensible. Caller passes usize. */
```

### compatibility contract

REQ-1: Syscall number is **467** on x86_64. ABI-stable.

REQ-2: `usize` MUST be at least `FILE_ATTR_SIZE_VER0` (current minimum is `sizeof(__u64) + 4*sizeof(__u32) = 24`).

REQ-3: `usize > sizeof(struct file_attr)`: kernel fills `sizeof(struct file_attr)` bytes and writes zeros to the trailing `(usize - sizeof)` bytes so userspace seeing a too-large struct gets a clean zeroed tail.

REQ-4: `usize < sizeof(struct file_attr)`: kernel writes only the prefix the caller declared; trailing kernel-known fields are discarded.

REQ-5: `at_flags` MUST contain only known bits; unknown ⟹ `-EINVAL`.

REQ-6: `AT_EMPTY_PATH` permits `path == ""` and uses `dirfd` as the target inode.

REQ-7: `AT_SYMLINK_NOFOLLOW` operates on the symlink itself rather than its target.

REQ-8: The syscall does NOT require opening the file. It performs a `path_lookupat` and calls `inode->i_op->fileattr_get`.

REQ-9: Filesystems without `fileattr_get` (rare; most tmpfs/proc paths) ⟹ `-EOPNOTSUPP`.

REQ-10: Read of the attributes is governed by the same DAC/MAC rules as a `stat(2)`: search permission on the directory path; no read permission on the file itself.

REQ-11: `FS_XFLAG_PROJINHERIT` and `fa_projid` are returned 0 on filesystems without project-quota support.

REQ-12: `fa_nextents` is informational; kernel may return 0 if the filesystem does not cheaply expose extent count.

REQ-13: Reads via this call do NOT update atime.

REQ-14: Concurrent `file_setattr(2)` calls produce snapshot semantics: each `file_getattr` returns a consistent view as of one fs-level locked instant.

REQ-15: This syscall is a SUPERSET of `FS_IOC_FSGETXATTR` and `FS_IOC_GETFLAGS`; the ioctls remain for backwards compat.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `at_flags_validated` | INVARIANT | unknown bits ⟹ `-EINVAL`. |
| `usize_min` | INVARIANT | `usize < FILE_ATTR_SIZE_VER0` ⟹ `-EINVAL`. |
| `tail_zeroed` | INVARIANT | `usize > sizeof(FileAttr)` ⟹ tail zeroed. |
| `no_partial_write` | INVARIANT | failure ⟹ no bytes written to user. |
| `fileattr_op_present` | INVARIANT | missing op ⟹ `-EOPNOTSUPP`. |
| `path_resolution_isolated` | INVARIANT | resolution honors `AT_SYMLINK_NOFOLLOW`. |

### Layer 2: TLA+

`fs/file-getattr.tla`:
- Per-call transitions: parse, path-resolve, op-dispatch, copy-out.
- Properties:
  - `safety_no_priv_op` — `EOPNOTSUPP` returned before any partial write.
  - `safety_atomic_out` — output buffer fully written or untouched.
  - `safety_tail_zeroed` — extra trailing bytes always zeroed.
  - `liveness_terminates` — call returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `FileAttr::get` post: success ⟹ buf populated, tail zero | `FileAttr::get` |
| `Path::resolve` post: respects AT_EMPTY_PATH, AT_SYMLINK_NOFOLLOW | `Path::resolve` |
| `FileAttr::get` post: -EOPNOTSUPP iff no fileattr_get op | `FileAttr::get` |
| `FileAttr::get` post: usize >= MIN invariant | `FileAttr::get` |

### Layer 4: Verus / Creusot functional

Per-`file_getattr(2)` man page; XFS `xfs_fs_fileattr_get` / ext4 `ext4_fileattr_get` semantic equivalence; matches FS_IOC_FSGETXATTR + FS_IOC_GETFLAGS.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`file_getattr(2)` reinforcement:

- **Per-`at_flags` allow-list** — defense against per-future-flag smuggle.
- **Per-`usize` minimum** — defense against per-too-small ABI confusion.
- **Per-`usize` tail-zero** — defense against per-info-leak when caller passes oversize.
- **Per-`-EOPNOTSUPP` for non-fileattr filesystems** — defense against per-ABI-leak from filesystems that lack the concept.
- **Per-path-resolution isolated** — defense against per-symlink-race info-leak.

### grsecurity / pax surface

- **PaX UDEREF on `path` and `attr`** — defense against per-pointer smuggling; SMAP forced for both copy_from_user and copy_to_user.
- **file_getattr capability gating** — `GRKERNSEC_FILEATTR_PRIV` requires `CAP_SYS_ADMIN` (or `CAP_FOWNER` for files owned by caller) to read `FS_XFLAG_IMMUTABLE`, `FS_XFLAG_APPEND`, and `FS_XFLAG_PROJINHERIT`; defense against fingerprinting protected-attribute layout by unprivileged callers.
- **PAX_USERCOPY on `struct file_attr` copy_to_user** — bounded copy uses whitelisted slab; rejects oversize.
- **Per-`usize` tail-zero strict** — defense against per-uninitialized-kmem leak via oversize write; tail bytes always zeroed before the user copy completes.
- **GRKERNSEC_PROC_GETPID identity** — path resolution honors caller's DAC/MAC; no `/proc/<other>` shortcuts.
- **GRKERNSEC_CHROOT_FCHDIR style isolation** — `AT_FDCWD` resolved within caller's chroot; defense against per-chroot escape.
- **PAX_REFCOUNT on `inode` during op call** — defense against per-fs-eject UAF.
- **GRKERNSEC_HIDESYM on inode internal pointers** — only ABI-visible fields written; internal pointers never leak.
- **Per-symlink-resolution audit** — `GRKERNSEC_AUDIT_PATH_RESOLVE` records the resolved path when symlinks are traversed; defense against per-symlink-confusion attacks.
- **Per-`AT_EMPTY_PATH` requires `CAP_DAC_READ_SEARCH`** — `GRKERNSEC_AT_EMPTY_PATH_PRIV` blocks unprivileged use of empty-path lookups against arbitrary fds.
- **Per-`fa_nextents` exposure restricted** — extent count can fingerprint storage layout; `GRKERNSEC_FILEATTR_NEXTENTS_HIDE` zeroes `fa_nextents` for unprivileged callers.
- **Per-`-EOPNOTSUPP` no-write guarantee** — caller's buffer untouched on this error; defense against per-stale-data leak.

