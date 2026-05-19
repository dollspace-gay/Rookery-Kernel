---
title: "Tier-5 syscall: lstat(2) — syscall 6"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`lstat(2)` is the **link-stat** variant: it returns metadata for the file at `path`, but if the leaf path component is a symbolic link, the kernel returns metadata for the link itself rather than its target. Intermediate components are still followed normally. The kernel resolves `path` via `user_path_at(AT_FDCWD, path, 0, &p)` (note: `LOOKUP_FOLLOW` is **not** set), calls `vfs_getattr(&p, &kstat, ...)`, and translates `kstat` into the ABI-frozen `struct stat`. Critical for: `ls -l`, `find -type l`, archive tools (tar, cpio, rsync), symlink-aware backup utilities.

`lstat(2)` is the non-following sibling of `stat(2)`.

### Acceptance Criteria

- [ ] AC-1: `lstat` on a symlink "a -> /tmp/x": `S_ISLNK(sb.st_mode) == 1`, `sb.st_size == strlen("/tmp/x")`.
- [ ] AC-2: `stat` on the same symlink: returns metadata of `/tmp/x` (not the link).
- [ ] AC-3: `lstat` on a regular file: identical to `stat`.
- [ ] AC-4: `lstat` on a broken symlink: succeeds; describes the link.
- [ ] AC-5: `lstat("/no/such", &sb)`: `ENOENT`.
- [ ] AC-6: `lstat(NULL, &sb)`: `EFAULT`.
- [ ] AC-7: `lstat(path_4097, &sb)`: `ENAMETOOLONG`.
- [ ] AC-8: 50-symlink loop in path prefix: `ELOOP`.
- [ ] AC-9: `lstat("", &sb)`: `ENOENT`.
- [ ] AC-10: Caller without `+x` on intermediate dir: `EACCES`.

### Architecture

```rust
#[syscall(nr = 6, abi = "sysv")]
pub fn sys_lstat(path: UserPtr<u8>, statbuf: UserPtr<UapiStat>) -> isize {
    Stat::do_stat(path, statbuf, LookupFlags::empty())   // no FOLLOW
}
```

`Stat::do_stat(path, statbuf, flags) -> isize`:
1. let kpath = Stat::copy_path_from_user(path)?;             // EFAULT, ENAMETOOLONG
2. if kpath.is_empty() { return Err(ENOENT); }
3. let resolved = Path::resolve_at(AT_FDCWD, &kpath, flags)?;// ENOENT, ENOTDIR, ELOOP, EACCES
4. let kstat = Stat::vfs_getattr(&resolved, STATX_BASIC_STATS)?;
5. let ustat = Stat::cp_new_stat(&kstat)?;                   // EOVERFLOW
6. statbuf.write_to_user(&ustat)?;                           // EFAULT
7. Ok(0)

`Path::resolve_at(dirfd, path, flags) -> Result<Path>`:
1. nd = Nameidata::new(dirfd, path, flags);
2. loop until leaf:
3.   step = walk_next(&mut nd);                              // follows intermediate symlinks if any
4.   if leaf ∧ !(flags & FOLLOW) ∧ step.is_symlink:
5.     return step.path;                                     // stop on link
6.   if step.is_symlink:
7.     check_link_count_under_40()?;                         // ELOOP
8.     redirect_to_link_target(&mut nd);
9. return nd.final_path;

### Out of Scope

- `statx(AT_SYMLINK_NOFOLLOW)` extended variant (covered in `statx.md`).
- `readlink(2)` for the target string (covered in `readlink.md`).
- VFS getattr per-fs (covered in Tier-3 `fs/stat.md`).
- Implementation code.

### signature

```c
int lstat(const char *path, struct stat *statbuf);
```

```c
struct stat {
    dev_t      st_dev;
    ino_t      st_ino;
    mode_t     st_mode;       /* S_IFLNK possible here */
    nlink_t    st_nlink;
    uid_t      st_uid;
    gid_t      st_gid;
    dev_t      st_rdev;
    off_t      st_size;       /* for symlinks: length of target string */
    blksize_t  st_blksize;
    blkcnt_t   st_blocks;
    struct timespec st_atim;
    struct timespec st_mtim;
    struct timespec st_ctim;
};
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `path` | `const char *` | in | NUL-terminated user-space pathname; intermediate symbolic links are followed, **leaf is not**. |
| `statbuf` | `struct stat *` | out | User-space buffer of `sizeof(struct stat)` bytes; populated on success. |

### return value

| Value | Meaning |
|---|---|
| `0` | Success; `*statbuf` written. |
| `-1` + `errno` | Failure; `*statbuf` unchanged. |

### errors

| errno | Trigger |
|---|---|
| `EACCES` | Search permission denied on a component of the path prefix. |
| `EFAULT` | `path` or `statbuf` points outside accessible user memory. |
| `ELOOP` | Too many symbolic links resolved during path-prefix walk (>40 on Linux). |
| `ENAMETOOLONG` | `path` exceeds `PATH_MAX` (4096). |
| `ENOENT` | A component of the path prefix does not exist; `path` is the empty string. |
| `ENOMEM` | Out of kernel memory. |
| `ENOTDIR` | A component of the path prefix is not a directory. |
| `EOVERFLOW` | Field does not fit on a 32-bit ABI; use `lstat64`/`statx`. |

### abi surface

```text
__NR_lstat   (x86_64)  = 6
__NR_lstat   (i386)    = 107   (legacy 32-bit struct)
__NR_lstat64 (i386)    = 196   (LFS variant)
/* arm64 / riscv: no __NR_lstat; userspace uses statx with AT_SYMLINK_NOFOLLOW. */
```

### compatibility contract

REQ-1: Syscall number is **6** on x86_64; ABI-stable.

REQ-2: Path walk uses **no** `LOOKUP_FOLLOW` for the leaf; intermediate symlinks **are** followed.

REQ-3: If the leaf is a symlink, `st_mode & S_IFMT == S_IFLNK` and `st_size` equals the length of the target string (NOT a NUL-terminated buffer; just the byte count).

REQ-4: For non-symlink leaves, `lstat` is observationally indistinguishable from `stat`.

REQ-5: Mount points: traversal across a mount transparently follows the mount as part of the prefix walk; the leaf is the resolved dentry on the upper filesystem.

REQ-6: `vfs_getattr` dispatches to `i_op->getattr` of the leaf inode; for symlinks, that path returns the link's own metadata, not the target's.

REQ-7: Path-string copy_from_user is bounded by `PATH_MAX`.

REQ-8: `statbuf` is written **only** on success.

REQ-9: `lstat` honors the caller's namespace, chroot, and capability set; chrooted callers cannot probe outside their root.

REQ-10: `lstat` does **not** update `st_atim` of the symlink (Linux symlinks have no atime tracking by default).

REQ-11: On 64-bit ABI, `EOVERFLOW` is structurally impossible.

REQ-12: `lstat("", &sb)` returns `ENOENT` (use `newfstatat(.., AT_EMPTY_PATH)` for the dirfd-relative empty-path semantics).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `lookup_nofollow_for_leaf` | INVARIANT | `lstat` resolves leaf without LOOKUP_FOLLOW. |
| `path_copy_bounded` | INVARIANT | path copy ≤ PATH_MAX. |
| `no_statbuf_write_on_error` | INVARIANT | error ⟹ statbuf untouched. |
| `link_count_bounded` | INVARIANT | ≤ 40 symlinks resolved across the prefix walk. |

### Layer 2: TLA+

`fs/lstat-syscall.tla`:
- States: per-copy_path, per-resolve (no-follow-leaf), per-getattr, per-cp_new_stat, per-write_user.
- Properties:
  - `safety_leaf_link_preserved` — leaf symlink not dereferenced.
  - `safety_no_statbuf_write_on_error` — failure ⟹ statbuf untouched.
  - `liveness_lstat_terminates` — every call returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_stat(no-follow)` post: ok ∧ leaf-link ⟹ S_ISLNK(statbuf.st_mode) | `Stat::do_stat` |
| `resolve_at(no-follow)` post: returns leaf even if symlink | `Path::resolve_at` |

### Layer 4: Verus / Creusot functional

Per-`lstat(2)` POSIX semantics; selftests in `tools/testing/selftests/filesystems/`.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`lstat(2)` reinforcement:

- **Per-PATH_MAX-bounded copy_from_user** — defense against per-unbounded path.
- **Per-symlink loop counter (40)** — defense against per-symlink-DoS.
- **Per-leaf-no-follow** — defense against per-symlink-target-info-leak (returns the link, not the target).
- **Per-namespace-scoped resolution** — defense against per-chroot-escape.

### grsecurity / pax-style reinforcement

- **PaX UDEREF on `path` copy_from_user** — defense against per-kernel-pointer path; SMAP forced.
- **PaX UDEREF on `statbuf` copy_to_user** — defense against per-kernel-pointer destination.
- **GRKERNSEC_HIDESYM on st_ino / st_dev / st_rdev** — pseudo-FS inode / device numbers masked for unprivileged callers (procfs, sysfs).
- **GRKERNSEC_PROC_USERGROUP info-leak guard** — st_uid / st_gid on `/proc/<pid>/*` symlinks visible only to owner under `hidepid=4`.
- **PAX_USERCOPY_HARDEN on cp_new_stat copy_to_user** — bounded, whitelisted slab; structural size constant.
- **GRKERNSEC_LINK restricted symlink follow** — even for intermediate components, world-writable sticky-dir symlinks owned by another uid are not followed (matches `fs.protected_symlinks`).
- **Per-st_size of S_IFLNK bounded by PATH_MAX** — defense against per-oversized-link info-leak.
- **Per-symlink-readlink not implied** — defense against per-target-string leak via `lstat` (target bytes are NOT exposed; only length).
- **Per-namespace cross-mount-id masked** — defense against per-mount-id fingerprint leak.

