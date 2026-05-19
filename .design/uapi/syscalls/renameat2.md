# Tier-5: syscall 316 — renameat2(2)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - arch/x86/entry/syscalls/syscall_64.tbl (`316  common  renameat2  sys_renameat2`)
  - fs/namei.c (`SYSCALL_DEFINE5(renameat2, ...)`, `do_renameat2`, `vfs_rename`, `lock_rename_child`, `may_delete`, `may_create`)
  - include/uapi/linux/fs.h (`RENAME_NOREPLACE`, `RENAME_EXCHANGE`, `RENAME_WHITEOUT`)
-->

## Summary

`renameat2(2)` is **x86_64 syscall 316**, the flag-bearing generalization of `rename(2)` and `renameat(2)`. It atomically renames `oldpath` (relative to `olddirfd` or absolute) to `newpath` (relative to `newdirfd` or absolute), with three new behaviors selectable via `flags`: `RENAME_NOREPLACE` (fail if `newpath` exists), `RENAME_EXCHANGE` (atomically swap two existing paths), and `RENAME_WHITEOUT` (rename and leave a whiteout char-device at the source, used by overlay/union filesystems). All operations are atomic with respect to crash and concurrent readers: a partial state is never observable, even across two different directories on the same filesystem.

Critical for: every libc `renameat2`/`rename`, every overlay-fs copy-up that needs whiteout, every database commit-via-rename, every container-runtime that performs atomic config swap, every Rookery VFS atomic-publish test.

## Signature

C (Linux-specific):

```c
int renameat2(int olddirfd, const char *oldpath,
              int newdirfd, const char *newpath,
              unsigned int flags);
```

glibc wrapper: `renameat2` → `INLINE_SYSCALL(renameat2, 5, olddirfd, oldpath, newdirfd, newpath, flags)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE5(renameat2,
                int, olddfd, const char __user *, oldname,
                int, newdfd, const char __user *, newname,
                unsigned int, flags);
```

Rookery dispatch:

```rust
pub fn sys_renameat2(
    olddirfd: i32, oldpath: UserPtr<u8>,
    newdirfd: i32, newpath: UserPtr<u8>,
    flags: u32,
) -> SyscallResult<i32>;
```

## Parameters

| name      | type                  | constraints                                                                | errno-on-bad |
|-----------|-----------------------|----------------------------------------------------------------------------|--------------|
| olddirfd  | `int`                 | open directory fd or `AT_FDCWD`                                            | `EBADF` / `ENOTDIR` |
| oldpath   | `const char __user *` | NUL-terminated; `< PATH_MAX`                                               | `EFAULT` / `ENAMETOOLONG` / `ENOENT` |
| newdirfd  | `int`                 | open directory fd or `AT_FDCWD`                                            | `EBADF` / `ENOTDIR` |
| newpath   | `const char __user *` | NUL-terminated; `< PATH_MAX`                                               | `EFAULT` / `ENAMETOOLONG` |
| flags     | `unsigned int`        | `0`, or subset of `{RENAME_NOREPLACE, RENAME_EXCHANGE, RENAME_WHITEOUT}` with documented exclusions | `EINVAL` |

## Return value

- Success: `0`.
- Failure: `< 0` — negated errno; filesystem state unchanged.

## Errors

| errno         | condition                                                                              |
|---------------|----------------------------------------------------------------------------------------|
| `EBADF`       | `olddirfd` or `newdirfd` not open and not `AT_FDCWD`.                                  |
| `EFAULT`      | `oldpath` or `newpath` outside user address space.                                     |
| `EINVAL`      | Invalid `flags` (unknown bit; `EXCHANGE | NOREPLACE`; `EXCHANGE | WHITEOUT`; `WHITEOUT` without `CAP_MKNOD`); source or target is a directory child of the other; same path. |
| `EBUSY`       | Source or target is mountpoint; or fs reports busy state.                              |
| `EDQUOT`      | Quota exceeded creating whiteout.                                                      |
| `EEXIST`      | `RENAME_NOREPLACE` and `newpath` exists.                                               |
| `EISDIR`      | `newpath` is a directory but `oldpath` is not (no `RENAME_EXCHANGE`).                  |
| `ELOOP`       | Too many symlinks during resolution.                                                   |
| `EMLINK`      | Target dir at max link count (Linux 7.1 generally avoids this).                        |
| `ENAMETOOLONG`| Path too long.                                                                         |
| `ENOENT`      | `oldpath` (or under `RENAME_EXCHANGE`, `newpath`) doesn't exist; or empty.             |
| `ENOMEM`      | Kernel allocation failed.                                                              |
| `ENOSPC`      | No space for new whiteout / new dirent.                                                |
| `ENOTDIR`     | `oldpath` is a directory but `newpath` exists and is not (no `RENAME_EXCHANGE`); or non-final component not a directory. |
| `ENOTEMPTY`   | `newpath` is a non-empty directory.                                                    |
| `EPERM`       | Filesystem doesn't support requested operation (e.g., `RENAME_EXCHANGE` on FAT).      |
| `EROFS`       | Read-only filesystem.                                                                  |
| `EXDEV`       | Source and target on different filesystems / mounts.                                   |

## ABI surface (constants + flags)

`flags` (`include/uapi/linux/fs.h`):

- `RENAME_NOREPLACE = (1 << 0)` — fail with `-EEXIST` if `newpath` already exists. Cannot combine with `RENAME_EXCHANGE`.
- `RENAME_EXCHANGE  = (1 << 1)` — atomically swap two existing paths; both must exist; either may be a directory; no implicit unlink. Cannot combine with `RENAME_NOREPLACE` or `RENAME_WHITEOUT`.
- `RENAME_WHITEOUT  = (1 << 2)` — rename `oldpath` to `newpath`, then leave a whiteout character-device (major 0, minor 0) at the source. Requires `CAP_MKNOD` and an fs that supports it (`s_op` flag `FS_RENAME_WHITEOUT`). Cannot combine with `RENAME_EXCHANGE`.

`AT_FDCWD = -100` (`include/uapi/linux/fcntl.h`).

Related kernel symbols:

- `vfs_rename(struct renamedata *)` — VFS entrypoint; serializes via `lock_rename` or `lock_rename_child`.
- `lock_rename(p1, p2)` — locks two parents in inode-pointer order to avoid AB-BA deadlock.
- `may_delete` / `may_create` — permission and stickiness checks.
- `s_op->rename` — fs-supplied rename op (handles EXCHANGE/WHITEOUT support).
- `WHITEOUT_DEV` — sentinel char-device for whiteouts.

## Compatibility contract

- REQ-1: Argument lowering: `%rdi=olddirfd (i32)`, `%rsi=oldpath`, `%rdx=newdirfd (i32)`, `%r10=newpath`, `%r8=flags (u32)`.
- REQ-2: `flags & ~(RENAME_NOREPLACE|RENAME_EXCHANGE|RENAME_WHITEOUT) ⟹ -EINVAL`.
- REQ-3: `RENAME_EXCHANGE | RENAME_NOREPLACE ⟹ -EINVAL`.
- REQ-4: `RENAME_EXCHANGE | RENAME_WHITEOUT ⟹ -EINVAL`.
- REQ-5: `RENAME_WHITEOUT && !capable(CAP_MKNOD) ⟹ -EPERM`.
- REQ-6: `getname(oldpath)` and `getname(newpath)` ⟹ `-EFAULT/-ENAMETOOLONG`.
- REQ-7: `kern_path_parent` for each (resolve to `(parent_dentry, last_component)` under `olddirfd`/`newdirfd`).
- REQ-8: Determine cross-fs: parents on different `vfsmount` ⟹ `-EXDEV`.
- REQ-9: Reject `oldpath == newpath` (same dentry) ⟹ `0` (no-op) unless `RENAME_EXCHANGE` (which is meaningful only for distinct paths).
- REQ-10: `lock_rename(old_parent, new_parent)` — global rename-mutex serializing all renames involving directories; otherwise inode mutex on parents.
- REQ-11: `may_delete(old_dir, old_dentry, is_dir)`; `may_create(new_dir, new_dentry, exchange_or_replace?)` — DAC, capability, sticky.
- REQ-12: Build `struct renamedata { old_mnt_userns, old_dir, old_dentry, new_mnt_userns, new_dir, new_dentry, flags, delegated_inode: NULL }` and call `vfs_rename`.
- REQ-13: `vfs_rename` performs LSM `security_inode_rename`, fsnotify, audit, dquot transfer, then `s_op->rename`.
- REQ-14: `RENAME_EXCHANGE`: both `new_dentry` and `old_dentry` must be positive; no implicit unlink; both directories' nlink semantics preserved.
- REQ-15: `RENAME_WHITEOUT`: after successful rename, `s_op->rename` (or VFS helper) creates a `WHITEOUT_DEV` char-device at the old path within the same atomic operation.
- REQ-16: `unlock_rename`; release path refs.
- REQ-17: Atomicity guarantee: a concurrent observer sees either the old layout or the new layout — never an intermediate state with both or neither dentry present.

## Acceptance Criteria

- [ ] AC-1: `renameat2(AT_FDCWD, "a", AT_FDCWD, "b", 0)` ≡ `rename("a", "b")`.
- [ ] AC-2: `renameat2(..., RENAME_NOREPLACE)` on existing target ⟹ `-EEXIST`; source untouched.
- [ ] AC-3: `renameat2(..., RENAME_EXCHANGE)`: both paths must exist, returns `0`, contents are atomically swapped.
- [ ] AC-4: `renameat2(..., RENAME_EXCHANGE | RENAME_NOREPLACE) == -EINVAL`.
- [ ] AC-5: `renameat2(..., RENAME_WHITEOUT)` without `CAP_MKNOD` ⟹ `-EPERM`.
- [ ] AC-6: `renameat2(..., RENAME_WHITEOUT)` on supported fs leaves a `S_IFCHR` whiteout at source.
- [ ] AC-7: Cross-fs ⟹ `-EXDEV` regardless of flags.
- [ ] AC-8: Concurrent rename + readdir sees either layout, never both.
- [ ] AC-9: `renameat2(olddirfd, "", newdirfd, "y", 0) == -ENOENT` (empty source rejected).
- [ ] AC-10: `flags == 0x10000000` ⟹ `-EINVAL`.

## Architecture

```
struct Renameat2Args { olddirfd: i32, oldpath: UserPtr<u8>, newdirfd: i32, newpath: UserPtr<u8>, flags: u32 }
```

`sys_renameat2(args) -> i32`:

1. If `args.flags & !VALID_RENAME_FLAGS` ⟹ return `-EINVAL`.
2. If `args.flags & RENAME_EXCHANGE` && (args.flags & (RENAME_NOREPLACE | RENAME_WHITEOUT)) ⟹ return `-EINVAL`.
3. If `args.flags & RENAME_WHITEOUT && !capable(CAP_MKNOD)` ⟹ return `-EPERM`.
4. Return `do_renameat2(args)`.

`Fs::do_renameat2(args) -> i32`:

1. `let old_name = getname(args.oldpath)?;`
2. `let new_name = getname(args.newpath)?;`
3. `let (old_parent, old_last) = filename_parentat(args.olddirfd, old_name)?;`
4. `let (new_parent, new_last) = filename_parentat(args.newdirfd, new_name)?;`
5. If `old_parent.mnt != new_parent.mnt` ⟹ return `-EXDEV`.
6. `lock_rename(old_parent.dentry, new_parent.dentry);`
7. `let old_dentry = lookup_one(old_parent, old_last)?;`
8. `let new_dentry = lookup_one(new_parent, new_last)?;`
9. Permission checks: `may_delete(old_parent, old_dentry, ...)` and `may_create(new_parent, new_dentry, ...)`.
10. `let rd = RenameData { old_dir: old_parent.inode, old_dentry, new_dir: new_parent.inode, new_dentry, flags: args.flags, delegated_inode: None };`
11. `vfs_rename(&rd)?;`
12. `unlock_rename(old_parent.dentry, new_parent.dentry);`
13. Release names and path refs.
14. Return `0`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_validated_first` | INVARIANT | Unknown / mutually-exclusive flags ⟹ `-EINVAL` before any path walk. |
| `exchange_requires_both_exist` | INVARIANT | `RENAME_EXCHANGE && (!old_dentry.positive || !new_dentry.positive) ⟹ -ENOENT`. |
| `whiteout_cap_mknod` | INVARIANT | `RENAME_WHITEOUT ⟹ capable(CAP_MKNOD)`. |
| `cross_fs_exdev` | INVARIANT | `old_parent.mnt != new_parent.mnt ⟹ -EXDEV`. |
| `atomic_lock_rename` | INVARIANT | `vfs_rename` runs with both parent inodes locked. |
| `partial_state_unobservable` | INVARIANT | No observer sees both source and target absent. |

### Layer 2: TLA+

`uapi/syscalls/renameat2.tla`:
- Per-call → validate → resolve parents → lock_rename → vfs_rename → unlock.
- Properties:
  - `safety_atomic_rename`,
  - `safety_exchange_symmetric`,
  - `safety_whiteout_left_only_on_success`,
  - `liveness_renameat2_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post: `RENAME_EXCHANGE` ⟹ both dentries now point at the previously-other inode | `Fs::vfs_rename` |
| Post: `RENAME_WHITEOUT` success ⟹ a `WHITEOUT_DEV` exists at old path | `Fs::vfs_rename` |
| Post: error ⟹ no fs state change visible to concurrent readers | `Fs::vfs_rename` |

### Layer 4: Verus/Creusot functional

`renameat2(odfd, op, ndfd, np, flags)` ≡ Linux renameat2(2) per `man 2 renameat2`.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`renameat2(2)` reinforcement:

- **Per-flag-strict mask + mutual-exclusion** — defense against silent semantic drift.
- **Per-`RENAME_WHITEOUT` cap gate** — defense against unprivileged whiteout fabrication.
- **Per-`lock_rename` discipline** — defense against rename-rename deadlock.
- **Per-`may_delete`/`may_create`** — defense against sticky-bit / DAC bypass.
- **Per-`EXDEV` early reject** — defense against cross-fs partial state.
- **Per-`vfs_rename` atomicity** — defense against torn-directory observers.
- **Per-`getname` length cap** — defense against long-path DoS.

## Grsecurity/PaX-style Reinforcement

- **PAX_UDEREF** — `oldpath`/`newpath` SMAP-guarded in `getname`.
- **GRKERNSEC_CHROOT_FCHDIR** — chroot'd rename rejects dirfd escaping the jail.
- **GRKERNSEC_LINK** — hardlink protections enforce ownership/world-writable rules during rename of links.
- **GRKERNSEC_FIFO** — FIFO rename in sticky dirs follows owner gating.
- **GRKERNSEC_SYMLINKOWN** — symlink rename in protected dirs gated on owner match.
- **AT_EMPTY_PATH / AT_SYMLINK_NOFOLLOW reject** — renameat2 does not accept those AT_* flags; passing them ⟹ `-EINVAL`.
- **GRKERNSEC_AUDIT_MOUNT** — rename across mount boundary is auditable.
- **GRKERNSEC_DMESG** — rename-error printks CAP_SYSLOG-gated.
- **GRKERNSEC_HIDESYM** — error printks redact kernel pointers.
- **PAX_REFCOUNT** — dentry/inode refcount saturating across the rename.
- **PAX_MEMORY_SANITIZE** — freed dentry slab zeroed on release.
- **PAX_RANDKSTACK** — kstack offset randomized at renameat2 syscall entry.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `rename(2)` / `renameat(2)` legacy wrappers (semantically `renameat2(.., flags=0)`).
- Overlay-fs copy-up internals (covered in fs/overlayfs Tier-3).
- LSM rename hook details (`security/00-overview.md`).
- Implementation code.
