---
title: "Tier-5: syscall 265 — linkat(2)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`linkat(2)` is **x86_64 syscall 265**, the directory-relative hardlink creation primitive. It creates a new directory entry at `newpath` (relative to `newdirfd`) pointing at the same inode as `oldpath` (relative to `olddirfd`), incrementing `inode->i_nlink`. Two AT_* flags adjust resolution: `AT_SYMLINK_FOLLOW` causes a terminal symlink in `oldpath` to be dereferenced (default is NOT to follow, opposite of most syscalls); `AT_EMPTY_PATH` paired with `oldpath == ""` lets `olddirfd` directly reference the inode being linked (the canonical "publish an `O_TMPFILE`" pattern, gated by `CAP_DAC_READ_SEARCH` for non-owner cases). Source and target MUST live on the same filesystem (`-EXDEV` otherwise). Directories cannot be hard-linked by unprivileged users (and modern Linux refuses even with `CAP_SYS_ADMIN` to avoid loops).

Critical for: every `O_TMPFILE`-based atomic-publish (`open(O_TMPFILE)` → write → `linkat(AT_EMPTY_PATH)`), every libc `linkat`/`link`, every backup-tool hardlink farm, every Rookery VFS link test, every container-image dedup layer.

### Acceptance Criteria

- [ ] AC-1: `linkat(AT_FDCWD, "a", AT_FDCWD, "b", 0) == 0` ⟹ `stat("b").st_nlink == stat("a").st_nlink == 2`.
- [ ] AC-2: `linkat(AT_FDCWD, "/dir", AT_FDCWD, "newdir", 0) == -EPERM`.
- [ ] AC-3: Source on `/`, target on `/mnt/other` ⟹ `-EXDEV`.
- [ ] AC-4: Existing target ⟹ `-EEXIST`; source untouched.
- [ ] AC-5: `linkat(tmpfile_fd, "", AT_FDCWD, "published", AT_EMPTY_PATH) == 0` and the previously-anonymous file is now linked.
- [ ] AC-6: `linkat(.., AT_SYMLINK_FOLLOW)` dereferences a terminal symlink in oldpath.
- [ ] AC-7: `linkat(.., 0)` on a terminal symlink ⟹ hardlinks the symlink itself.
- [ ] AC-8: `protected_hardlinks=1` + non-owner + non-writable file ⟹ `-EPERM`.
- [ ] AC-9: `linkat(.., 0xFFFF) == -EINVAL`.
- [ ] AC-10: `linkat(AT_FDCWD, "", AT_FDCWD, "x", 0) == -ENOENT` (empty without `AT_EMPTY_PATH`).

### Architecture

```
struct LinkatArgs { olddirfd: i32, oldpath: UserPtr<u8>, newdirfd: i32, newpath: UserPtr<u8>, flags: i32 }
```

`sys_linkat(args) -> i32`:

1. If `args.flags & !(AT_SYMLINK_FOLLOW | AT_EMPTY_PATH)` ⟹ return `-EINVAL`.
2. Return `do_linkat(args)`.

`Fs::do_linkat(args) -> i32`:

1. `let old_name = getname_flags(args.oldpath, if AT_EMPTY_PATH { LOOKUP_EMPTY } else { 0 })?;`
2. `let new_name = getname(args.newpath)?;`
3. `let mut source = if (args.flags & AT_EMPTY_PATH) && old_name.is_empty() {
        let f = fdget(args.olddirfd)?;
        require_cap_dac_read_search_if_not_owner(&f)?;
        f.f_path.clone()
    } else {
        filename_lookup(args.olddirfd, old_name, lookup_flags_from(args.flags))?
    };`
4. `let (target_parent, last) = filename_parentat(args.newdirfd, new_name, LOOKUP_PARENT)?;`
5. If `source.mnt != target_parent.mnt` ⟹ `-EXDEV`.
6. `may_linkat(&source)?;` — protected_hardlinks gate.
7. If `S_ISDIR(source.dentry.inode.i_mode)` ⟹ `-EPERM`.
8. `inode_lock_nested(&target_parent.inode, I_MUTEX_PARENT);`
9. `let new_dentry = lookup_one(&target_parent, &last)?;`
10. If `new_dentry.is_positive()` ⟹ unlock; `-EEXIST`.
11. `let mut delegated = None;`
12. `vfs_link(source.dentry, idmap, target_parent.inode, new_dentry, &mut delegated)?;`
13. `inode_unlock(&target_parent.inode);`
14. If `delegated.is_some()` ⟹ `break_deleg_wait(&delegated); return -EWOULDBLOCK`.
15. Return `0`.

### Out of Scope

- `link(2)` legacy 2-arg wrapper (semantically `linkat(AT_FDCWD, .., AT_FDCWD, .., 0)`).
- Overlay-fs link semantics (covered in fs/overlayfs Tier-3).
- LSM link hook (`security/00-overview.md`).
- Implementation code.

### signature

C (POSIX):

```c
int linkat(int olddirfd, const char *oldpath,
           int newdirfd, const char *newpath, int flags);
```

glibc wrapper: `linkat` → `INLINE_SYSCALL(linkat, 5, olddirfd, oldpath, newdirfd, newpath, flags)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE5(linkat,
                int, olddfd, const char __user *, oldname,
                int, newdfd, const char __user *, newname, int, flags);
```

Rookery dispatch:

```rust
pub fn sys_linkat(
    olddirfd: i32, oldpath: UserPtr<u8>,
    newdirfd: i32, newpath: UserPtr<u8>,
    flags: i32,
) -> SyscallResult<i32>;
```

### parameters

| name      | type                  | constraints                                                                | errno-on-bad |
|-----------|-----------------------|----------------------------------------------------------------------------|--------------|
| olddirfd  | `int`                 | open directory fd or `AT_FDCWD`; or any fd with `AT_EMPTY_PATH`            | `EBADF` / `ENOTDIR` |
| oldpath   | `const char __user *` | NUL-terminated; `< PATH_MAX`; MAY be `""` only with `AT_EMPTY_PATH`        | `EFAULT` / `ENAMETOOLONG` / `ENOENT` |
| newdirfd  | `int`                 | open directory fd or `AT_FDCWD`                                            | `EBADF` / `ENOTDIR` |
| newpath   | `const char __user *` | NUL-terminated; `< PATH_MAX`; non-empty                                    | `EFAULT` / `ENAMETOOLONG` / `ENOENT` |
| flags     | `int`                 | subset of `{AT_SYMLINK_FOLLOW, AT_EMPTY_PATH}`                             | `EINVAL` |

### return value

- Success: `0`.
- Failure: `< 0` — negated errno.

### errors

| errno         | condition                                                                              |
|---------------|----------------------------------------------------------------------------------------|
| `EACCES`      | Write denied on target's parent directory; or read/search denied on source path.       |
| `EBADF`       | `olddirfd`/`newdirfd` not open and not `AT_FDCWD`.                                     |
| `EDQUOT`      | Quota exceeded creating the new dirent.                                                |
| `EEXIST`      | `newpath` already exists.                                                              |
| `EFAULT`      | Path pointer outside user address space.                                               |
| `EINVAL`      | Invalid `flags` (unknown bit, or `AT_EMPTY_PATH` without `CAP_DAC_READ_SEARCH` on non-owner fd). |
| `EIO`         | Filesystem I/O error.                                                                  |
| `ELOOP`       | Too many symlinks in resolution.                                                       |
| `EMLINK`      | Source inode at `inode->i_sb->s_max_links` (typically `LINK_MAX = 65535`).              |
| `ENAMETOOLONG`| Path too long.                                                                         |
| `ENOENT`      | Component missing; or empty `oldpath` without `AT_EMPTY_PATH`; or `oldpath` source has nlink=0 (`O_TMPFILE` not yet linked) and caller lacks privilege. |
| `ENOMEM`      | Allocation failed.                                                                     |
| `ENOSPC`      | No room for new dirent.                                                                |
| `ENOTDIR`     | Non-final component not a directory.                                                   |
| `EPERM`       | Source is a directory (hardlink to directory forbidden); or `protected_hardlinks` sysctl violated. |
| `EROFS`       | Target filesystem is read-only.                                                        |
| `EXDEV`       | Source and target on different filesystems.                                            |

### abi surface (constants + flags)

`flags` (`include/uapi/linux/fcntl.h`):

- `AT_SYMLINK_FOLLOW = 0x400` — dereference terminal symlink in `oldpath`. **Default for `linkat` is NOT to follow** (opposite of `openat`); set this flag if you want POSIX `link`-like behavior. (Compatibility note: POSIX leaves the symlink-follow behavior implementation-defined for `link(2)`; Linux historically chose "do not follow"; `linkat` makes it explicit.)
- `AT_EMPTY_PATH = 0x1000` — if `oldpath == ""`, use `olddirfd` directly as the source inode (regardless of file type, including `O_TMPFILE`-anonymous regular files with `i_nlink == 0`). For fds the caller did not open, `CAP_DAC_READ_SEARCH` is required.

`AT_FDCWD = -100`.

Related kernel symbols:

- `vfs_link(struct dentry *old_dentry, struct mnt_idmap *idmap, struct inode *dir, struct dentry *new_dentry, struct inode **delegated_inode)` — VFS link op.
- `may_linkat(struct path *link)` — `protected_hardlinks` sysctl gate plus DAC.
- `i_op->link` — fs-supplied link op.
- `LINK_MAX` / `s_sb->s_max_links` — per-inode link count ceiling.
- `protected_hardlinks` sysctl (`fs.protected_hardlinks`, default `1`) — restricts hardlinks to files the caller can read and either owns or has write access to.

### compatibility contract

- REQ-1: Argument lowering: `%rdi=olddirfd (i32)`, `%rsi=oldpath`, `%rdx=newdirfd (i32)`, `%r10=newpath`, `%r8=flags (i32)`.
- REQ-2: `flags & ~(AT_SYMLINK_FOLLOW|AT_EMPTY_PATH) ⟹ -EINVAL`.
- REQ-3: `getname(oldpath)` / `getname(newpath)` with empty-allowed = `(flags & AT_EMPTY_PATH ? on oldpath only)`.
- REQ-4: Resolve source:
  - If `AT_EMPTY_PATH` and `oldpath == ""`: source is `olddirfd`'s file → its `f_path.dentry` directly (and requires `CAP_DAC_READ_SEARCH` if not caller-opened).
  - Else: `filename_lookup(olddirfd, oldpath, LOOKUP_FOLLOW if AT_SYMLINK_FOLLOW else 0)`.
- REQ-5: Resolve target parent: `filename_parentat(newdirfd, newpath, LOOKUP_PARENT)`.
- REQ-6: Cross-fs check: `source.mnt != target.mnt ⟹ -EXDEV`.
- REQ-7: `may_linkat(&source_path)` — apply `protected_hardlinks` sysctl:
  - Caller is file owner ⟹ allowed.
  - Caller has write+read on source inode and source is "safe" (regular file, not setuid/setgid root) ⟹ allowed.
  - Else ⟹ `-EPERM`.
- REQ-8: Reject `S_ISDIR(source.inode.i_mode) ⟹ -EPERM` (no directory hardlinks).
- REQ-9: `inode_lock_nested(target_parent.inode, I_MUTEX_PARENT)`.
- REQ-10: `lookup_one(target_parent, last)`; if positive ⟹ `-EEXIST`.
- REQ-11: `vfs_link(source.dentry, idmap, target_parent.inode, new_dentry, &delegated_inode)`:
  - LSM `security_inode_link`.
  - `i_op->link(old_dentry, dir, new_dentry)` — fs implements by adding dirent and incrementing `inode.i_nlink`.
  - `fsnotify_link(dir, dentry)`.
- REQ-12: NFSv4 delegation: if `delegated_inode` set ⟹ `-EWOULDBLOCK`, retry after `break_deleg_wait`.
- REQ-13: `inode_unlock`; release paths; free names.
- REQ-14: Source `inode->i_nlink == 0` (`O_TMPFILE`): allowed iff `AT_EMPTY_PATH` is set and `CAP_DAC_READ_SEARCH` granted (or caller owns the fd). This is the canonical atomic-publish pattern.
- REQ-15: `inode->i_nlink >= s_max_links ⟹ -EMLINK` before link op.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_validated_first` | INVARIANT | `flags & ~VALID_LINKAT_FLAGS ⟹ -EINVAL` before lookups. |
| `at_empty_path_cap_gate` | INVARIANT | `AT_EMPTY_PATH` on non-owner fd ⟹ `CAP_DAC_READ_SEARCH`. |
| `no_directory_hardlink` | INVARIANT | `S_ISDIR(source) ⟹ -EPERM`. |
| `same_fs_required` | INVARIANT | `source.mnt != target.mnt ⟹ -EXDEV`. |
| `protected_hardlinks_enforced` | INVARIANT | `may_linkat` gate honored when sysctl=1. |
| `max_links_capped` | INVARIANT | `inode.nlink >= s_max_links ⟹ -EMLINK`. |

### Layer 2: TLA+

`uapi/syscalls/linkat.tla`:
- Per-call → validate → resolve → may_linkat → lock target_parent → vfs_link → unlock.
- Properties:
  - `safety_nlink_only_incremented_on_success`,
  - `safety_tmpfile_atomic_publish`,
  - `liveness_linkat_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post: success ⟹ source inode nlink incremented by 1 | `Fs::vfs_link` |
| Post: error ⟹ source inode nlink unchanged | `Fs::vfs_link` |
| Post: new_dentry is positive iff success | `Fs::do_linkat` |

### Layer 4: Verus/Creusot functional

`linkat(odfd, op, ndfd, np, flags)` ≡ POSIX.1-2024 linkat per `man 2 linkat`, with Linux-specific `AT_EMPTY_PATH` behavior.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`linkat(2)` reinforcement:

- **Per-`protected_hardlinks` gate** — defense against `/tmp`-pivot hardlink attacks.
- **Per-`AT_EMPTY_PATH` cap gate** — defense against publishing foreign fd inodes.
- **Per-no-directory-link** — defense against fs loop creation.
- **Per-`EXDEV` early reject** — defense against cross-fs partial link.
- **Per-`inode_lock_nested(I_MUTEX_PARENT)`** — defense against parent-race during target install.
- **Per-`delegated_inode` break** — defense against NFSv4 inconsistent state.
- **Per-`s_max_links` cap** — defense against link-count overflow corrupting filesystem.

### grsecurity/pax-style reinforcement

- **PAX_UDEREF** — `oldpath`/`newpath` SMAP-guarded in `getname`.
- **GRKERNSEC_LINK** — strict hardlink restrictions: caller must own the file or have write access, file must not be setuid/setgid, sticky-dir source rejected.
- **GRKERNSEC_CHROOT_FCHDIR** — chroot'd linkat cannot escape via dirfd inherited pre-chroot.
- **GRKERNSEC_FIFO** — FIFO hardlinks in sticky dirs gated.
- **GRKERNSEC_SYMLINKOWN** — symlink targets in protected dirs gated on owner.
- **AT_EMPTY_PATH / AT_SYMLINK_FOLLOW reject** — when grsec policy forbids cross-owner fd link or following symlinks for non-owner, returns `-EPERM` before any namei walk.
- **GRKERNSEC_DMESG** — link-error printks CAP_SYSLOG-gated.
- **GRKERNSEC_HIDESYM** — error printks redact kernel pointers.
- **PAX_REFCOUNT** — `inode.i_nlink` saturating; overflow before `s_max_links` triggers `BUG()`.
- **PAX_MEMORY_SANITIZE** — freed dentry slab zeroed on release.
- **PAX_RANDKSTACK** — kstack offset randomized at linkat syscall entry.
- **GRKERNSEC_AUDIT_CHDIR** — linkat across chroot boundary auditable.

