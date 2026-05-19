# Tier-5: syscall 263 — unlinkat(2)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - arch/x86/entry/syscalls/syscall_64.tbl (`263  common  unlinkat  sys_unlinkat`)
  - fs/namei.c (`SYSCALL_DEFINE3(unlinkat, ...)`, `do_unlinkat`, `do_rmdir`, `vfs_unlink`, `vfs_rmdir`, `may_delete`)
  - include/uapi/linux/fcntl.h (`AT_FDCWD`, `AT_REMOVEDIR`)
-->

## Summary

`unlinkat(2)` is **x86_64 syscall 263**, the directory-relative file/directory removal primitive. With `flags == 0` it behaves like `unlink(2)` relative to `dirfd`: removes a non-directory dentry and decrements its inode's link count (freeing the inode if `nlink` reaches `0` and no open fds remain). With `flags == AT_REMOVEDIR` it behaves like `rmdir(2)`: removes an empty directory. The two modes are mutually exclusive and selected by a single flag bit; `AT_REMOVEDIR` is also the only flag accepted.

Critical for: every libc `unlinkat`/`unlink`/`remove`/`rmdir`, every Rookery VFS removal test, every container-runtime image-prune, every `rm -rf` traversal, every `O_TMPFILE → linkat` atomic-publish race-avoidance pattern.

## Signature

C (POSIX):

```c
int unlinkat(int dirfd, const char *pathname, int flags);
```

glibc wrapper: `unlinkat` → `INLINE_SYSCALL(unlinkat, 3, dirfd, pathname, flags)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE3(unlinkat, int, dfd, const char __user *, pathname, int, flag);
```

Rookery dispatch:

```rust
pub fn sys_unlinkat(dirfd: i32, pathname: UserPtr<u8>, flags: i32) -> SyscallResult<i32>;
```

## Parameters

| name      | type                  | constraints                                                                | errno-on-bad |
|-----------|-----------------------|----------------------------------------------------------------------------|--------------|
| dirfd     | `int`                 | open directory fd or `AT_FDCWD`                                            | `EBADF` / `ENOTDIR` |
| pathname  | `const char __user *` | NUL-terminated; `< PATH_MAX`; non-empty                                    | `EFAULT` / `ENAMETOOLONG` / `ENOENT` |
| flags     | `int`                 | `0` or `AT_REMOVEDIR`                                                      | `EINVAL` |

## Return value

- Success: `0`.
- Failure: `< 0` — negated errno.

## Errors

| errno         | condition                                                                              |
|---------------|----------------------------------------------------------------------------------------|
| `EACCES`      | Write permission denied on parent directory; or sticky bit forbids removal.            |
| `EBADF`       | `dirfd` is not open and not `AT_FDCWD`.                                                |
| `EBUSY`       | Target is a mountpoint, or in-use special file.                                        |
| `EFAULT`      | `pathname` outside user address space.                                                 |
| `EINVAL`      | `flags` contains any bit other than `AT_REMOVEDIR`.                                    |
| `EIO`         | Filesystem I/O error during dentry/inode update.                                       |
| `EISDIR`      | Target is a directory and `flags` lacks `AT_REMOVEDIR`.                                |
| `ELOOP`       | Too many symlinks in resolution.                                                       |
| `ENAMETOOLONG`| Path too long.                                                                         |
| `ENOENT`      | Component missing, or `pathname` empty/trailing slash without directory.               |
| `ENOMEM`      | Allocation failed.                                                                     |
| `ENOTDIR`     | Non-final component not a directory; or `AT_REMOVEDIR` and target not a directory.     |
| `ENOTEMPTY`   | `AT_REMOVEDIR` and target directory not empty.                                         |
| `EPERM`       | Filesystem doesn't permit unlink (e.g., immutable / append-only); or system has `_POSIX_CHOWN_RESTRICTED` and caller lacks privilege for special files. |
| `EROFS`       | Read-only filesystem.                                                                  |

## ABI surface (constants + flags)

`flags` (`include/uapi/linux/fcntl.h`):

- `AT_REMOVEDIR = 0x200` — perform `rmdir`-semantics instead of `unlink`-semantics. With this flag, target MUST be a directory (else `-ENOTDIR`) and MUST be empty (else `-ENOTEMPTY`). Without this flag, target MUST NOT be a directory (else `-EISDIR`).

`AT_FDCWD = -100`.

Related kernel symbols:

- `vfs_unlink(struct mnt_idmap *, struct inode *dir, struct dentry *dentry, struct inode **delegated_inode)` — non-directory removal entry; emits `fsnotify_unlink`, calls LSM, then `dir->i_op->unlink`.
- `vfs_rmdir(struct mnt_idmap *, struct inode *dir, struct dentry *dentry)` — directory removal entry; `dir->i_op->rmdir`.
- `may_delete(idmap, dir, victim, is_dir)` — DAC + sticky + immutable / append checks.
- `filename_parentat` — resolves `(dirfd, pathname)` to `(parent_dentry, last_component)`.
- `dont_mount` / `is_local_mountpoint(dentry)` — refuse to unlink mountpoint.

## Compatibility contract

- REQ-1: Argument lowering: `%rdi=dirfd (i32)`, `%rsi=pathname`, `%rdx=flags (i32)`.
- REQ-2: `flags & ~AT_REMOVEDIR ⟹ -EINVAL`.
- REQ-3: `getname(pathname)` ⟹ `-EFAULT/-ENAMETOOLONG/-ENOENT` (empty rejected even with `AT_REMOVEDIR`).
- REQ-4: Branch on `flags & AT_REMOVEDIR`:
  - Set ⟹ `do_rmdir(dirfd, name)`.
  - Clear ⟹ `do_unlinkat(dirfd, name)`.
- REQ-5: `do_unlinkat`:
  - Resolve parent via `filename_parentat(dirfd, name, lookup_flags=LOOKUP_PARENT)`.
  - `inode_lock_nested(parent_inode, I_MUTEX_PARENT)`.
  - `lookup_one(parent, last)`.
  - Reject if `S_ISDIR(victim_inode.i_mode) ⟹ -EISDIR`.
  - Reject mountpoint ⟹ `-EBUSY`.
  - `vfs_unlink(idmap, parent_inode, victim, &delegated_inode)`.
  - `inode_unlock(parent_inode)`.
- REQ-6: `do_rmdir`:
  - Same parent resolution and locking.
  - Reject if `!S_ISDIR(victim_inode.i_mode) ⟹ -ENOTDIR`.
  - Reject mountpoint ⟹ `-EBUSY`.
  - Reject non-empty (per `i_op->rmdir`) ⟹ `-ENOTEMPTY`.
  - `vfs_rmdir(idmap, parent_inode, victim)`.
- REQ-7: `vfs_unlink`:
  - LSM `security_inode_unlink(parent, dentry)`.
  - `may_delete(parent, dentry, S_ISDIR(inode))` — DAC + sticky.
  - `dir->i_op->unlink(dir, dentry)`.
  - `fsnotify_unlink(parent, dentry)`.
  - `d_delete(dentry)` — turn dentry negative; if other refs hold positive form, mark unhashed.
- REQ-8: `vfs_rmdir`:
  - LSM `security_inode_rmdir`.
  - `may_delete(..., is_dir=true)`.
  - `dir->i_op->rmdir`.
  - `fsnotify_rmdir`.
- REQ-9: Inode link-count decrement happens inside `i_op->unlink` (typical fs: `drop_nlink(inode)` then sb-dependent durability).
- REQ-10: Open fds on the unlinked file retain access until last `fput`; inode is reclaimed at `nlink == 0 && i_count == 0`.
- REQ-11: Delegations (NFSv4): if `delegated_inode` is set, the syscall returns `-EWOULDBLOCK`, the delegation is broken via `break_deleg_wait`, and userspace retries.
- REQ-12: Releasing path refs and freeing the kernel-side `name`.

## Acceptance Criteria

- [ ] AC-1: `unlinkat(AT_FDCWD, "regfile", 0) == 0` and `regfile` is removed.
- [ ] AC-2: `unlinkat(AT_FDCWD, "regfile", AT_REMOVEDIR) == -ENOTDIR`.
- [ ] AC-3: `unlinkat(AT_FDCWD, "emptydir", AT_REMOVEDIR) == 0`.
- [ ] AC-4: `unlinkat(AT_FDCWD, "nonemptydir", AT_REMOVEDIR) == -ENOTEMPTY`.
- [ ] AC-5: `unlinkat(AT_FDCWD, "regfile", 0x80) == -EINVAL`.
- [ ] AC-6: `unlinkat(AT_FDCWD, "", 0) == -ENOENT`.
- [ ] AC-7: `unlinkat(non_dir_fd, "foo", 0) == -ENOTDIR` when path is relative.
- [ ] AC-8: Unlinking a file with open fds keeps the file readable via those fds until close.
- [ ] AC-9: `unlinkat(AT_FDCWD, "mountpoint", AT_REMOVEDIR) == -EBUSY`.
- [ ] AC-10: Sticky-dir + non-owner ⟹ `-EACCES` (or `-EPERM` per fs).

## Architecture

```
struct UnlinkatArgs { dirfd: i32, pathname: UserPtr<u8>, flags: i32 }
```

`sys_unlinkat(args) -> i32`:

1. If `args.flags & !AT_REMOVEDIR` ⟹ return `-EINVAL`.
2. If `args.flags & AT_REMOVEDIR` ⟹ return `do_rmdir(args.dirfd, args.pathname)`.
3. Else ⟹ return `do_unlinkat(args.dirfd, args.pathname)`.

`Fs::do_unlinkat(dfd, name_uptr) -> i32`:

1. `let name = getname(name_uptr)?;`
2. `let (parent, last) = filename_parentat(dfd, name, LOOKUP_PARENT)?;`
3. `inode_lock_nested(&parent.inode, I_MUTEX_PARENT);`
4. `let dentry = lookup_one(&parent, &last)?;`
5. If `dentry.is_negative()` ⟹ `inode_unlock; return -ENOENT`.
6. If `S_ISDIR(dentry.inode.i_mode)` ⟹ `inode_unlock; return -EISDIR`.
7. If `is_local_mountpoint(&dentry)` ⟹ `inode_unlock; return -EBUSY`.
8. `let res = vfs_unlink(idmap, &parent.inode, &dentry, &mut delegated)?;`
9. `inode_unlock(&parent.inode);`
10. If `delegated.is_some()` ⟹ `break_deleg_wait(&delegated); return -EWOULDBLOCK`.
11. Return `res`.

`Fs::do_rmdir(dfd, name_uptr) -> i32`:

1. Mirror of `do_unlinkat` with `vfs_rmdir`, `!S_ISDIR ⟹ -ENOTDIR`, non-empty ⟹ `-ENOTEMPTY`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_strict` | INVARIANT | Any bit outside `AT_REMOVEDIR` ⟹ `-EINVAL`. |
| `unlink_rejects_dir` | INVARIANT | Without `AT_REMOVEDIR`, target=directory ⟹ `-EISDIR`. |
| `rmdir_rejects_nondir` | INVARIANT | With `AT_REMOVEDIR`, target≠directory ⟹ `-ENOTDIR`. |
| `mountpoint_busy` | INVARIANT | `is_local_mountpoint ⟹ -EBUSY`. |
| `delegation_retry` | INVARIANT | NFSv4 delegated_inode set ⟹ `-EWOULDBLOCK` after break. |
| `parent_lock_held` | INVARIANT | `vfs_unlink`/`vfs_rmdir` runs with parent inode lock held. |

### Layer 2: TLA+

`uapi/syscalls/unlinkat.tla`:
- Per-call → validate → resolve parent → lock → vfs_unlink/vfs_rmdir → unlock.
- Properties:
  - `safety_dir_unlink_mutex`,
  - `safety_mountpoint_never_removed`,
  - `liveness_unlinkat_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post: `vfs_unlink` ⟹ dentry is negative or unhashed | `Fs::vfs_unlink` |
| Post: `vfs_rmdir` ⟹ target inode unreachable from parent | `Fs::vfs_rmdir` |
| Post: error ⟹ no dentry change visible | `Fs::do_unlinkat` |

### Layer 4: Verus/Creusot functional

`unlinkat(dirfd, path, flags)` ≡ POSIX.1-2024 unlinkat plus Linux `AT_REMOVEDIR` semantics per `man 2 unlinkat`.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`unlinkat(2)` reinforcement:

- **Per-flag-strict mask** — defense against silent forward-compat acceptance.
- **Per-`is_local_mountpoint` reject** — defense against mountpoint-removal data-loss.
- **Per-`may_delete` sticky check** — defense against `/tmp`-style cross-user removal.
- **Per-`inode_lock_nested(I_MUTEX_PARENT)`** — defense against parent-mutation race during lookup-then-delete.
- **Per-`delegated_inode` break** — defense against NFSv4 inconsistent client view.
- **Per-`getname` length cap** — defense against long-path DoS.

## Grsecurity/PaX-style Reinforcement

- **PAX_UDEREF** — `pathname` SMAP-guarded in `getname`.
- **GRKERNSEC_CHROOT_FCHDIR** — chroot'd process cannot unlink via dirfd inherited pre-chroot.
- **GRKERNSEC_LINK** — hardlink removal in protected dirs respects owner check.
- **GRKERNSEC_FIFO** — FIFO unlink in sticky dirs gated on owner.
- **GRKERNSEC_SYMLINKOWN** — symlink unlink in protected dirs gated.
- **AT_REMOVEDIR enforcement** — without `AT_REMOVEDIR`, directories are never unlinked; with it, only directories.
- **GRKERNSEC_DMESG** — unlink-error printks CAP_SYSLOG-gated.
- **GRKERNSEC_HIDESYM** — error printks redact kernel pointers.
- **PAX_REFCOUNT** — dentry/inode refcounts saturating across removal.
- **PAX_MEMORY_SANITIZE** — freed dentry / inode slab zeroed on release.
- **PAX_RANDKSTACK** — kstack offset randomized at unlinkat syscall entry.
- **GRKERNSEC_AUDIT_CHDIR** — unlink on chroot boundary auditable.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `unlink(2)` and `rmdir(2)` legacy wrappers (semantically `unlinkat(AT_FDCWD, .., 0/AT_REMOVEDIR)`).
- Inode reclaim / orphan-list handling (covered in fs/<fstype> Tier-3).
- LSM hook details (`security/00-overview.md`).
- Implementation code.
