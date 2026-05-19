---
title: "Tier-5: syscall 266 — symlinkat(2)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`symlinkat(2)` is **x86_64 syscall 266**, the directory-relative symbolic-link creation primitive. It creates a new symlink at `linkpath` (relative to `newdirfd`) whose link-text content is the string `target`. The target string is stored verbatim and is NOT validated to refer to an existing path — symlinks may dangle, may be relative to the *symlink's* directory at resolution time (not to `newdirfd`), and may contain any byte except embedded NUL. `symlinkat` has no flags (the third argument is `int newdirfd`, not `flags`); symlink permissions are filesystem-defined (most fs use `0777` for the link itself).

Critical for: every libc `symlinkat`/`symlink`, every container-runtime image-extract producing layer symlinks, every Rookery VFS symlink test, every `ln -s` in shell scripts, every `/dev/<host>` → real-device alias setup.

### Acceptance Criteria

- [ ] AC-1: `symlinkat("foo", AT_FDCWD, "bar") == 0` and `readlink("bar")` yields `"foo"`.
- [ ] AC-2: `symlinkat("", AT_FDCWD, "x") == -ENOENT`.
- [ ] AC-3: `symlinkat("foo", AT_FDCWD, "") == -ENOENT`.
- [ ] AC-4: `symlinkat("foo", AT_FDCWD, "existing") == -EEXIST`.
- [ ] AC-5: `symlinkat("/nonexistent", AT_FDCWD, "dangling") == 0` (dangling is legal).
- [ ] AC-6: `symlinkat("foo", closed_fd, "bar") == -EBADF`.
- [ ] AC-7: `symlinkat("foo", non_dir_fd, "bar") == -ENOTDIR` (relative linkpath).
- [ ] AC-8: Symlink inode mode is `S_IFLNK | 0777` (independent of umask).
- [ ] AC-9: `symlinkat(very_long_target, AT_FDCWD, "x") == -ENAMETOOLONG` when target ≥ PATH_MAX.
- [ ] AC-10: `symlinkat` on FAT-like fs returns `-EPERM`.

### Architecture

```
struct SymlinkatArgs { target: UserPtr<u8>, newdirfd: i32, linkpath: UserPtr<u8> }
```

`sys_symlinkat(args) -> i32`:

1. `let target_name = getname(args.target)?;` — `-EFAULT`/`-ENAMETOOLONG`/`-ENOENT`.
2. Return `do_symlinkat(target_name, args.newdirfd, args.linkpath)`.

`Fs::do_symlinkat(target, dfd, linkpath_uptr) -> i32`:

1. `let new_name = getname(linkpath_uptr)?;`
2. `let (parent, last) = filename_parentat(dfd, new_name, LOOKUP_PARENT)?;`
3. `inode_lock_nested(&parent.inode, I_MUTEX_PARENT);`
4. `let new_dentry = lookup_one(&parent, &last)?;`
5. If `new_dentry.is_positive()` ⟹ unlock; return `-EEXIST`.
6. `may_create(idmap, &parent.inode, &new_dentry)?;`
7. `vfs_symlink(idmap, &parent.inode, &new_dentry, target.name())?;`
8. `inode_unlock(&parent.inode);`
9. Release paths and `target`.
10. Return `0`.

`Fs::vfs_symlink(idmap, dir, dentry, oldname) -> Result<(), errno>`:

1. `security_inode_symlink(dir, dentry, oldname)?;`
2. `dir.i_op.symlink.ok_or(EPERM)?(dir, dentry, oldname)?;`
3. `fsnotify_create(dir, dentry);`
4. `Ok(())`.

### Out of Scope

- `symlink(2)` legacy 2-arg wrapper (semantically `symlinkat(target, AT_FDCWD, linkpath)`).
- `readlinkat(2)` (separate Tier-5 if expanded).
- `fast`/`slow` symlink storage tradeoffs (covered in fs/<fstype> Tier-3).
- LSM symlink hook details (`security/00-overview.md`).
- Implementation code.

### signature

C (POSIX):

```c
int symlinkat(const char *target, int newdirfd, const char *linkpath);
```

glibc wrapper: `symlinkat` → `INLINE_SYSCALL(symlinkat, 3, target, newdirfd, linkpath)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE3(symlinkat,
                const char __user *, oldname,
                int, newdfd, const char __user *, newname);
```

Rookery dispatch:

```rust
pub fn sys_symlinkat(
    target: UserPtr<u8>,
    newdirfd: i32,
    linkpath: UserPtr<u8>,
) -> SyscallResult<i32>;
```

### parameters

| name      | type                  | constraints                                                                | errno-on-bad |
|-----------|-----------------------|----------------------------------------------------------------------------|--------------|
| target    | `const char __user *` | NUL-terminated; `< PATH_MAX`; non-empty; arbitrary string (need not exist) | `EFAULT` / `ENAMETOOLONG` / `ENOENT` (empty) |
| newdirfd  | `int`                 | open directory fd or `AT_FDCWD`                                            | `EBADF` / `ENOTDIR` |
| linkpath  | `const char __user *` | NUL-terminated; `< PATH_MAX`; non-empty                                    | `EFAULT` / `ENAMETOOLONG` / `ENOENT` (empty) |

### return value

- Success: `0`. A new symlink exists at `linkpath`.
- Failure: `< 0` — negated errno.

### errors

| errno         | condition                                                                              |
|---------------|----------------------------------------------------------------------------------------|
| `EACCES`      | Write permission denied on the directory containing `linkpath`.                        |
| `EBADF`       | `newdirfd` is not open and not `AT_FDCWD`.                                             |
| `EDQUOT`      | Quota exhausted creating the symlink inode.                                            |
| `EEXIST`      | `linkpath` already exists.                                                             |
| `EFAULT`      | `target` or `linkpath` outside user address space.                                     |
| `EIO`         | Filesystem I/O error.                                                                  |
| `ELOOP`       | Too many symlinks in resolving `linkpath`'s parent.                                    |
| `ENAMETOOLONG`| `target` or `linkpath` longer than `PATH_MAX-1`.                                       |
| `ENOENT`      | Component of `linkpath`'s parent path is missing; or `target`/`linkpath` is empty.     |
| `ENOMEM`      | Allocation failed.                                                                     |
| `ENOSPC`      | No space for new dirent or inode.                                                      |
| `ENOTDIR`     | Non-final component of `linkpath` not a directory; or `newdirfd` not a directory.      |
| `EPERM`       | Filesystem (e.g., FAT) doesn't support symlinks.                                       |
| `EROFS`       | Read-only filesystem.                                                                  |

### abi surface (constants + flags)

`symlinkat` has **no flags argument** — the third positional argument is `newdirfd`, not a flag mask. This is unusual in the `*at` family.

`AT_FDCWD = -100` (`include/uapi/linux/fcntl.h`).

Related kernel symbols:

- `vfs_symlink(struct mnt_idmap *idmap, struct inode *dir, struct dentry *dentry, const char *oldname)` — VFS symlink entry; calls LSM and `i_op->symlink`.
- `may_create(idmap, dir, dentry)` — DAC + sticky check.
- `i_op->symlink(dir, dentry, symname)` — fs-supplied symlink op; stores link text in inode (fast symlink) or dedicated block (slow symlink).
- `filename_parentat` — resolves `(newdirfd, linkpath)` to `(parent_dentry, last_component)`.
- `LSM security_inode_symlink` — security hook.

The symlink's link-text limit is `PATH_MAX-1` bytes (`getname` enforces `PATH_MAX`). The symlink **inode** does not have meaningful permission bits in POSIX terms — Linux's `vfs_symlink` always sets `S_IFLNK | 0777` (mode unused for access checks; resolution uses parent-directory permissions).

### compatibility contract

- REQ-1: Argument lowering: `%rdi=target`, `%rsi=newdirfd (i32)`, `%rdx=linkpath`.
- REQ-2: `getname(target)` — produces a `struct filename` containing the verbatim link text; empty ⟹ `-ENOENT`; > `PATH_MAX-1` ⟹ `-ENAMETOOLONG`.
- REQ-3: `getname(linkpath)` — similarly.
- REQ-4: `filename_parentat(newdirfd, linkpath, LOOKUP_PARENT)` — resolves the parent directory and last component.
- REQ-5: `inode_lock_nested(parent_inode, I_MUTEX_PARENT)`.
- REQ-6: `lookup_one(parent, last)` — if positive ⟹ `-EEXIST`.
- REQ-7: `may_create(idmap, parent_inode, new_dentry)` — DAC + sticky.
- REQ-8: `vfs_symlink(idmap, parent_inode, new_dentry, target_text)`:
  - `security_inode_symlink(parent_inode, new_dentry, target_text)`.
  - `parent_inode->i_op->symlink(parent_inode, new_dentry, target_text)`.
  - `fsnotify_create(parent_inode, new_dentry)`.
- REQ-9: `inode_unlock(parent_inode)`; release paths; free names.
- REQ-10: The created symlink inode has mode `S_IFLNK | 0777` regardless of caller umask. Resolution traverses the parent directory's mode for access checks.
- REQ-11: The link text is stored byte-for-byte; the kernel does NOT canonicalize it.
- REQ-12: A successful `symlinkat` does NOT validate that `target` resolves — dangling symlinks are legal and common.
- REQ-13: `target` may contain `..` and absolute paths; resolution time uses caller's filesystem view, not `newdirfd`.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `empty_target_rejected` | INVARIANT | `target == "" ⟹ -ENOENT`. |
| `empty_linkpath_rejected` | INVARIANT | `linkpath == "" ⟹ -ENOENT`. |
| `existing_target_rejected` | INVARIANT | `new_dentry.is_positive() ⟹ -EEXIST`. |
| `link_text_stored_verbatim` | INVARIANT | `target` bytes written into inode without canonicalization. |
| `parent_lock_held` | INVARIANT | `vfs_symlink` runs under `parent_inode` lock. |
| `dangling_allowed` | INVARIANT | Non-existent `target` path content ⟹ still `0`. |

### Layer 2: TLA+

`uapi/syscalls/symlinkat.tla`:
- Per-call → getname → resolve_parent → lock → vfs_symlink → unlock.
- Properties:
  - `safety_no_existing_overwrite`,
  - `safety_link_text_verbatim`,
  - `liveness_symlinkat_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post: `new_dentry.inode.i_mode & S_IFMT == S_IFLNK` | `Fs::vfs_symlink` |
| Post: `readlink(new_dentry) == target` byte-for-byte | `Fs::vfs_symlink` |
| Post: error ⟹ no new dentry installed | `Fs::do_symlinkat` |

### Layer 4: Verus/Creusot functional

`symlinkat(target, dfd, linkpath)` ≡ POSIX.1-2024 symlinkat per `man 2 symlinkat`.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`symlinkat(2)` reinforcement:

- **Per-empty-string reject** — defense against silent zero-byte symlink creation.
- **Per-`PATH_MAX` enforcement** — defense against long-string DoS in symlink storage.
- **Per-`inode_lock_nested(I_MUTEX_PARENT)`** — defense against parent-race during create.
- **Per-`may_create` sticky/DAC** — defense against `/tmp`-style cross-user link planting.
- **Per-`fsnotify_create` post-hook** — defense against missed audit/inotify event.
- **Per-no-flag-mask** — symlinkat is the one *at-family member with no flag arg; no parsing required.

### grsecurity/pax-style reinforcement

- **PAX_UDEREF** — `target` and `linkpath` SMAP-guarded in `getname`.
- **GRKERNSEC_CHROOT_FCHDIR** — chroot'd symlinkat cannot escape via newdirfd inherited pre-chroot.
- **GRKERNSEC_SYMLINKOWN** — symlink creation in protected (e.g., world-writable sticky) dirs follows owner-match rules: caller must own the parent or be root.
- **GRKERNSEC_LINK** — symlink in sticky dir to file caller doesn't own ⟹ `-EPERM`.
- **GRKERNSEC_FIFO** — irrelevant for symlinks but mode-mask precedence preserved.
- **AT_EMPTY_PATH / AT_SYMLINK_NOFOLLOW reject** — symlinkat has no flag arg; any attempt to overload the slot returns `-EBADF` (since the value would be parsed as `newdirfd`).
- **GRKERNSEC_DMESG** — symlink-error printks CAP_SYSLOG-gated.
- **GRKERNSEC_HIDESYM** — error printks redact kernel pointers.
- **PAX_REFCOUNT** — dentry/inode refcounts saturating.
- **PAX_MEMORY_SANITIZE** — slab freed on error path zeroed.
- **PAX_RANDKSTACK** — kstack offset randomized at symlinkat syscall entry.
- **GRKERNSEC_AUDIT_CHDIR** — symlinkat across chroot boundary auditable.

