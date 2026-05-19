# Tier-5: syscall 87 — unlink(2)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - arch/x86/entry/syscalls/syscall_64.tbl (`87  common  unlink  sys_unlink`)
  - fs/namei.c (`SYSCALL_DEFINE1(unlink, ...)`, `do_unlinkat`, `vfs_unlink`)
  - include/linux/namei.h
  - Documentation/filesystems/sticky.rst
-->

## Summary

`unlink(2)` is **x86_64 syscall 87**, the POSIX directory-entry removal primitive. It removes the directory entry `pathname`, decrements the referenced inode's `i_nlink` count, and — if `i_nlink` reaches zero AND no process holds the file open — frees the inode and reclaims its data blocks. If processes hold the file open, the inode persists until the last close (the classic Unix "unlink-while-open" pattern that allows `tmpfile(3)` and the "lost+found-free temp file" idiom). The directory entry MUST refer to a non-directory file — `unlink` of a directory returns `-EISDIR`; use `rmdir(2)` instead. Sticky-bit (`+t`) directories require ownership of either the file or the directory.

Critical for: every `rm` invocation, every `tempfile`/`mkstemp` cleanup, every package-manager file replacement, every Rookery init `runlevel.d/*` cleanup, every `inotify` IN_DELETE_SELF user.

## Signature

C (POSIX-1.2008):

```c
int unlink(const char *pathname);
```

glibc wrapper: `__unlink` → `INLINE_SYSCALL(unlink, 1, pathname)` on architectures exporting the bare syscall; otherwise forwarded to `unlinkat(AT_FDCWD, pathname, 0)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE1(unlink, const char __user *, pathname);
```

Rookery dispatch:

```rust
pub fn sys_unlink(pathname: UserPtr<u8>) -> SyscallResult<i32>;
```

## Parameters

| name     | type           | constraints                                                   | errno-on-bad |
|----------|----------------|---------------------------------------------------------------|--------------|
| pathname | `const char *` | NUL-terminated; ≤ `PATH_MAX`; final component MUST exist and be non-directory | `EFAULT` / `ENAMETOOLONG` / `ENOENT` / `EISDIR` |

## Return value

- Success: `0`.
- Failure: `< 0` — negated errno.

## Errors

| errno          | condition                                                                  |
|----------------|----------------------------------------------------------------------------|
| `EACCES`       | Write permission denied on parent dir, or search permission denied on prefix. |
| `EBUSY`        | `pathname` refers to a file that is in use (e.g., a mount point).         |
| `EFAULT`       | `pathname` outside the caller's address space.                            |
| `EIO`          | I/O error on the filesystem.                                              |
| `EISDIR`       | `pathname` refers to a directory.                                         |
| `ELOOP`        | Too many symlinks in path resolution.                                     |
| `ENAMETOOLONG` | Path > `PATH_MAX` or component > `NAME_MAX`.                              |
| `ENOENT`       | A component of `pathname` does not exist.                                 |
| `ENOMEM`       | Kernel memory exhausted.                                                  |
| `ENOTDIR`      | A component of the prefix is not a directory.                             |
| `EPERM`        | Sticky-bit dir and caller is not owner of file/dir; or filesystem does not permit unlink; or caller lacks `CAP_DAC_OVERRIDE`/`CAP_FOWNER`. |
| `EROFS`        | `pathname` is on a read-only filesystem.                                  |

## ABI surface (constants + flags)

`unlink(2)` is flagless. Internally calls `do_unlinkat(AT_FDCWD, pathname)`.

The unlink path-walk uses `LOOKUP_PARENT | LOOKUP_DIRECTORY` on the prefix, and the last component is NOT followed if it is a symlink — the symlink itself is unlinked (consistent with `link(2)` and `lstat(2)`).

Related kernel symbols:

- `vfs_unlink(idmap, dir, dentry, &delegated_inode)` — VFS layer; takes `i_mutex` on parent.
- `may_delete(idmap, dir, victim, isdir=false)` — DAC + sticky-bit check.
- `security_inode_unlink(dir, dentry)` — LSM hook.
- `fsnotify_unlink(dir, dentry)` — inotify/fanotify event emission.
- `dnotify_parent(dentry, DN_DELETE)` — legacy dnotify event.
- `inode_lock` / `inode_unlock` — parent inode lock.
- `dput(dentry)` — dentry release.
- `iput(inode)` — inode release; triggers `i_op->evict_inode` if `i_nlink == 0` and not in use.

## Compatibility contract

- REQ-1: Argument lowering: `%rdi=pathname (const char __user *)`.
- REQ-2: UDEREF: `pathname` must be a user pointer.
- REQ-3: `strncpy_from_user_with_pax_check(pathname, PATH_MAX)`.
- REQ-4: Resolve parent (`LOOKUP_PARENT | LOOKUP_DIRECTORY`); last component NOT followed.
- REQ-5: If last component is a directory (`S_IFDIR`) ⟹ `-EISDIR`.
- REQ-6: `may_delete(idmap, parent.dentry.inode(), victim, isdir=false)`:
  - parent must be writable for caller,
  - if sticky bit set on parent: caller must own file OR parent OR have `CAP_FOWNER`.
- REQ-7: `mnt_want_write(parent.mnt)?;` — `-EROFS`.
- REQ-8: `security_inode_unlink(parent.dentry.inode(), dentry)`.
- REQ-9: `inode_lock(parent.dentry.inode());`
- REQ-10: `vfs_unlink(idmap, parent.dentry.inode(), dentry, &delegated_inode)`:
  - call `dir->i_op->unlink(dir, dentry)`,
  - if NFS-delegated: `-EWOULDBLOCK`-then-retry path,
  - `drop_nlink(inode)`,
  - `inode->i_ctime = current_time(inode);`.
- REQ-11: `fsnotify_unlink(parent.dentry.inode(), dentry);`
- REQ-12: `dnotify_parent(dentry, DN_DELETE);`
- REQ-13: `inode_unlock(parent.dentry.inode());`
- REQ-14: `mnt_drop_write(parent.mnt);`
- REQ-15: `dput(dentry); iput(inode);` — inode evicted when last reference drops AND `i_nlink == 0`.
- REQ-16: On failure, no inode mutation; ref counts balanced.

## Acceptance Criteria

- [ ] AC-1: `unlink("regfile")` ⟹ `0`, `stat("regfile") == -ENOENT`.
- [ ] AC-2: `unlink("dir")` ⟹ `-EISDIR`.
- [ ] AC-3: `unlink("symlink")` ⟹ `0`, removes the symlink itself, target untouched.
- [ ] AC-4: `unlink("nonexistent")` ⟹ `-ENOENT`.
- [ ] AC-5: Sticky-bit dir, file owned by other UID, caller not `CAP_FOWNER` ⟹ `-EPERM`.
- [ ] AC-6: File open by another process: `unlink` returns 0; inode persists until close.
- [ ] AC-7: Read-only mount ⟹ `-EROFS`.
- [ ] AC-8: NULL pathname ⟹ `-EFAULT`.
- [ ] AC-9: `unlink("/")` ⟹ `-EISDIR` (or `-EBUSY` for root mount).
- [ ] AC-10: After `unlink`, `inotify` IN_DELETE event fires on parent.

## Architecture

```
struct UnlinkArgs { pathname: UserPtr<u8> }
```

`sys_unlink(args) -> i32`:

1. `let name = strncpy_from_user_pax(args.pathname, PATH_MAX)?;`
2. Return `Fs::do_unlinkat(AT_FDCWD, name)`.

`Fs::do_unlinkat(dfd, name) -> i32`:

1. `let (parent, dentry) = filename_parentat(dfd, name, LookupFlags::DIRECTORY)?;`
2. If `dentry.inode().i_mode_type() == FileType::Directory` ⟹ `-EISDIR`.
3. `mnt_want_write(parent.mnt)?;`
4. `security_inode_unlink(parent.dentry.inode(), &dentry)?;`
5. `inode_lock(parent.dentry.inode());`
6. Loop (NFS delegation retry):
   - `let mut delegated_inode = None;`
   - `vfs_unlink(idmap, parent.dentry.inode(), &dentry, &mut delegated_inode)?;`
   - `if let Some(d) = delegated_inode { break_deleg_wait(d)?; continue; }`
   - break.
7. `fsnotify_unlink(parent.dentry.inode(), &dentry);`
8. `dnotify_parent(&dentry, DnotifyFlags::DELETE);`
9. `inode_unlock(parent.dentry.inode());`
10. `mnt_drop_write(parent.mnt);`
11. `dput(dentry);`
12. Return `0`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pax_uderef_pathname` | INVARIANT | pathname UDEREF before deref. |
| `directory_rejected` | INVARIANT | `EISDIR` returned before any state mutation. |
| `parent_lock_held_during_vfs_unlink` | INVARIANT | parent inode lock held during `vfs_unlink`. |
| `sticky_bit_obeyed` | INVARIANT | sticky-dir + non-owner ⟹ `-EPERM`. |
| `nlink_underflow_blocked` | INVARIANT | `drop_nlink` cannot decrement below 0. |

### Layer 2: TLA+

`uapi/syscalls/unlink.tla`:
- Per-call → resolve(parent + last) → may_delete → vfs_unlink → fsnotify → iput.
- Properties:
  - `safety_isdir_rejected_before_mutation`,
  - `safety_sticky_bit_enforced`,
  - `safety_open_file_unlink_keeps_inode_alive`,
  - `liveness_unlink_terminates_unless_busy`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post (success): `i_nlink == prev - 1` | `vfs_unlink` |
| Post (success): parent dir's `i_mtime`/`i_ctime` updated | `vfs_unlink` |
| Post (error): no inode mutation, refcounts balanced | `Fs::do_unlinkat` |

### Layer 4: Verus/Creusot functional

`unlink(pathname)` ≡ `unlinkat(AT_FDCWD, pathname, 0)` per POSIX-1.2008.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`unlink(2)` reinforcement:

- **Per-sticky-bit DAC check** — defense against `/tmp` cross-uid unlink attacks.
- **Per-`-EISDIR` check before any mutation** — defense against accidental directory removal via `unlink`.
- **Per-`security_inode_unlink` LSM** — SELinux/AppArmor mediates.
- **Per-`mnt_want_write` freeze barrier** — defense against unlink during filesystem freeze.
- **Per-`fsnotify_unlink`** — auditable inotify event.
- **Per-NFS delegation retry loop** — defense against client-server-state-divergence.
- **Per-parent `i_mutex`** — defense against race-unlinking the same dentry twice.
- **Per-`iput` deferred-eviction discipline** — defense against unlinked-but-open inode UAF.

## Grsecurity/PaX-style Reinforcement

- **PAX_UDEREF on `pathname`** — strict user/kernel pointer split; kernel pointer in argv rejected.
- **GRKERNSEC_LINK** — uniform with `link(2)`: cross-uid unlinks in sticky-bit dirs blocked beyond stock sticky-bit rules.
- **GRKERNSEC_SYMLINKOWN** — when last component is a symlink owned by another UID in a sticky-bit directory, unlink returns `-EACCES`. Defends `/tmp` race scenarios where an attacker plants a symlink so the victim's unlink hits the symlink rather than the intended file.
- **GRKERNSEC_FIFO** — unlinking a FIFO owned by another UID in a sticky-bit dir blocked.
- **PAX_MEMORY_SANITIZE** — unlinked inode and data blocks zeroed on free before reuse, defending against information disclosure via reallocated blocks.
- **GRKERNSEC_HIDESYM** — unlink-error dmesg redacts kernel pointers.
- **GRKERNSEC_CHROOT_UNIX** — chroot'd process cannot unlink AF_UNIX socket nodes owned by host.
- **GRKERNSEC_AUDIT_PTRACE** — unlink by ptracer on tracee FS auditable.
- **GRKERNSEC_DMESG** — unlink-error dmesg lines CAP_SYSLOG-gated.
- **GRKERNSEC_PROC_USER** — unlink in `/proc/$pid/` constrained by ownership.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `unlinkat(2)` (separate Tier-5 doc; this is the AT_FDCWD-only ancestor).
- `rmdir(2)` — covered in `rmdir.md`.
- `rename(2)` — covered in `rename.md`.
- VFS `i_op->unlink` per-filesystem (Tier-3 per FS).
- Implementation code.
