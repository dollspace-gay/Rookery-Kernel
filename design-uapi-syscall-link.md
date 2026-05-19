---
title: "Tier-5: syscall 86 — link(2)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`link(2)` is **x86_64 syscall 86**, the classic POSIX hard-link primitive. It creates a new directory entry `newpath` that refers to the **same inode** as `oldpath`, incrementing `inode->i_nlink` atomically. Both paths are resolved with `AT_FDCWD` semantics (relative to current working directory). The two paths MUST reside on the same filesystem mount (`-EXDEV` otherwise) and the source MUST be a regular file or, with `CAP_SYS_ADMIN`, a directory (most filesystems forbid hard-linking directories outright with `-EPERM` regardless of capability). `link(2)` is the unflagged ancestor of `linkat(2)` and is implemented as `do_linkat(AT_FDCWD, oldpath, AT_FDCWD, newpath, 0)` internally.

Critical for: every package manager that uses content-addressed storage with hard links, every backup tool emitting hard-link de-duplication (`rsync --link-dest`), every shell `ln` invocation without `-s`, every container layer that hard-links shared files across overlay branches.

### Acceptance Criteria

- [ ] AC-1: `link("a", "b")` with `a` a regular file ⟹ `b` exists, `stat(b).st_ino == stat(a).st_ino`, `stat(a).st_nlink == 2`.
- [ ] AC-2: `link("a", "b")` with `b` already existing ⟹ `-EEXIST`.
- [ ] AC-3: `link("dir", "newdir")` ⟹ `-EPERM` on ext4/xfs/btrfs/Rookery-vfs.
- [ ] AC-4: Cross-mount `link("/mnt1/a", "/mnt2/b")` ⟹ `-EXDEV`.
- [ ] AC-5: `link("symlink", "b")` ⟹ `b` is a hard link to the **symlink itself** (not the target), because `LOOKUP_FOLLOW` is off.
- [ ] AC-6: `fs.protected_hardlinks = 1` + non-owner caller + non-writable source ⟹ `-EPERM`.
- [ ] AC-7: Reaching `i_nlink == s_max_links` ⟹ `-EMLINK`.
- [ ] AC-8: Read-only mount ⟹ `-EROFS`.
- [ ] AC-9: NULL pointer in either arg ⟹ `-EFAULT`.

### Architecture

```
struct LinkArgs { oldpath: UserPtr<u8>, newpath: UserPtr<u8> }
```

`sys_link(args) -> i32`:

1. `let oldname = strncpy_from_user_pax(args.oldpath, PATH_MAX)?;` — `-EFAULT` / `-ENAMETOOLONG`.
2. `let newname = strncpy_from_user_pax(args.newpath, PATH_MAX)?;`.
3. Return `Fs::do_linkat(AT_FDCWD, oldname, AT_FDCWD, newname, 0)`.

`Fs::do_linkat(olddfd, oldname, newdfd, newname, flags) -> i32`:

1. `let old_path = filename_lookup(olddfd, oldname, LookupFlags::empty())?;` — does NOT follow trailing symlink for plain `link`.
2. `let (new_parent, new_dentry) = filename_create(newdfd, newname, LookupFlags::DIRECTORY)?;`
3. If `old_path.mnt != new_parent.mnt` ⟹ `-EXDEV`.
4. `may_linkat(idmap, &old_path)?;` — `fs.protected_hardlinks` policy check.
5. `mnt_want_write(new_parent.mnt)?;` — `-EROFS` if read-only.
6. `security_inode_link(old_path.dentry, new_parent.dentry.inode(), new_dentry)?;` — LSM gate.
7. `inode_lock(new_parent.dentry.inode());`
8. `vfs_link(old_path.dentry, idmap, new_parent.dentry.inode(), new_dentry, None)?;`
9. `fsnotify_link(new_parent.dentry.inode(), old_path.dentry.inode(), new_dentry);`
10. `inode_unlock(new_parent.dentry.inode());`
11. `mnt_drop_write(new_parent.mnt);`
12. Return `0`.

### Out of Scope

- `linkat(2)` (separate Tier-5 doc; `link` is the AT_FDCWD-only ancestor).
- `symlink(2)` / `symlinkat(2)` — covered in `symlink.md`.
- `rename(2)` — covered in `rename.md`.
- VFS `i_op->link` per-filesystem implementations (Tier-3 per FS).
- Implementation code.

### signature

C (POSIX-1.2008):

```c
int link(const char *oldpath, const char *newpath);
```

glibc wrapper: `__link` → `INLINE_SYSCALL(link, 2, oldpath, newpath)` on architectures that still export the bare syscall; on others, glibc forwards to `linkat(AT_FDCWD, oldpath, AT_FDCWD, newpath, 0)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE2(link, const char __user *, oldname, const char __user *, newname);
```

Rookery dispatch:

```rust
pub fn sys_link(oldpath: UserPtr<u8>, newpath: UserPtr<u8>) -> SyscallResult<i32>;
```

### parameters

| name    | type             | constraints                                                       | errno-on-bad |
|---------|------------------|-------------------------------------------------------------------|--------------|
| oldpath | `const char *`   | NUL-terminated path; ≤ `PATH_MAX`; resolved with `LOOKUP_FOLLOW` off (does not follow trailing symlink) | `EFAULT` / `ENAMETOOLONG` / `ENOENT` |
| newpath | `const char *`   | NUL-terminated path; ≤ `PATH_MAX`; parent must exist and be writable | `EFAULT` / `ENAMETOOLONG` / `ENOENT` / `EACCES` |

### return value

- Success: `0`.
- Failure: `< 0` — negated errno.

### errors

| errno          | condition                                                                |
|----------------|--------------------------------------------------------------------------|
| `EACCES`       | Search permission denied on directory component of either path; or write permission denied on parent directory of `newpath`. |
| `EDQUOT`       | User's disk quota on `newpath`'s filesystem exhausted.                  |
| `EEXIST`       | `newpath` already exists.                                                |
| `EFAULT`       | Either pointer is outside the user's address space.                      |
| `EIO`          | An I/O error occurred reading/writing directory blocks.                  |
| `ELOOP`        | Too many symbolic links encountered while resolving either path.         |
| `EMLINK`       | `oldpath`'s inode already has the maximum `i_nlink` count.               |
| `ENAMETOOLONG` | A path exceeds `PATH_MAX` or a component exceeds `NAME_MAX`.             |
| `ENOENT`       | A component of either path prefix does not exist.                        |
| `ENOMEM`       | Kernel memory exhausted.                                                 |
| `ENOSPC`       | The device containing `newpath`'s parent has no room for the new entry.  |
| `ENOTDIR`      | A component of either path prefix is not a directory.                    |
| `EPERM`        | `oldpath` is a directory and the filesystem does not permit hard-linking directories. |
| `EPERM`        | The filesystem containing `oldpath`/`newpath` does not support hard links. |
| `EROFS`        | The file is on a read-only filesystem.                                   |
| `EXDEV`        | `oldpath` and `newpath` are on different mounted filesystems.            |

### abi surface (constants + flags)

`link(2)` takes no flags. Internally the kernel calls `do_linkat(AT_FDCWD, oldname, AT_FDCWD, newname, 0)`. The `0` translates to `LOOKUP_FOLLOW = 0` for the source — i.e., **trailing symlinks in `oldpath` are NOT followed**. This is the historical POSIX behavior that diverges from `linkat(... AT_SYMLINK_FOLLOW)`.

Related kernel symbols:

- `struct path` — `{vfsmount, dentry}` tuple for the resolved target.
- `vfs_link(old_dentry, idmap, dir, new_dentry, inode_out)` — VFS layer; takes `i_mutex` on parent of `new_dentry`, calls `dir->i_op->link`.
- `may_linkat(idmap, &path)` — enforces sysctl `fs.protected_hardlinks`.
- `mnt_want_write(mnt)` / `mnt_drop_write(mnt)` — freeze-aware write reservation on the target mount.
- `security_inode_link(old_dentry, dir, new_dentry)` — LSM hook (SELinux, AppArmor, Smack, Tomoyo, Yama, BPF-LSM).
- `fsnotify_link(dir, inode, new_dentry)` — inotify/fanotify event emission.
- `audit_inode(name, dentry, flags)` — auditd record on linked path.

### compatibility contract

- REQ-1: Argument lowering: `%rdi=oldname (const char __user *)`, `%rsi=newname (const char __user *)`.
- REQ-2: Both pointers MUST pass `access_ok(USER_DS, p, 1)`; PaX-UDEREF guarantees the strict user/kernel split so a kernel pointer cannot be passed in.
- REQ-3: `strncpy_from_user_with_pax_check(oldname, PATH_MAX)` → `-EFAULT` / `-ENAMETOOLONG`.
- REQ-4: Same for `newname`.
- REQ-5: Resolve `oldname` with `LOOKUP_FOLLOW = false` — last component symlink NOT dereferenced (differs from `linkat`'s `AT_SYMLINK_FOLLOW`).
- REQ-6: Resolve `newname`'s parent (`LOOKUP_PARENT | LOOKUP_DIRECTORY`).
- REQ-7: Enforce `oldpath` and `newpath` on the same `vfsmount` ⟹ `-EXDEV` otherwise.
- REQ-8: `may_linkat(&old_path)` per `fs.protected_hardlinks` policy.
- REQ-9: Call `vfs_link(old_path.dentry, mnt_idmap(...), new_dir, new_dentry, NULL)`.
- REQ-10: On success, fire `fsnotify_link` + `audit_inode`; return `0`.
- REQ-11: On failure, no inode mutation; refcounts on both paths balanced.
- REQ-12: SUID/SGID bits on `oldpath`'s inode are **preserved** (links share the inode).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pax_uderef_paths` | INVARIANT | Both `oldname` / `newname` pass UDEREF before deref. |
| `same_mount_enforced` | INVARIANT | `EXDEV` returned before any inode mutation when mounts differ. |
| `no_follow_old_symlink` | INVARIANT | `LOOKUP_FOLLOW` clear on `oldname` resolution. |
| `i_nlink_overflow_blocked` | INVARIANT | `i_nlink == s_max_links` ⟹ `-EMLINK` before increment. |
| `parent_lock_held` | INVARIANT | `new_parent` inode lock held during `vfs_link`. |

### Layer 2: TLA+

`uapi/syscalls/link.tla`:
- Per-call → resolve(old) → resolve(new-parent) → may_linkat → vfs_link → fsnotify → audit.
- Properties:
  - `safety_xdev_blocks_inode_mutation`,
  - `safety_protected_hardlinks_obeyed`,
  - `liveness_link_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post (success): `new_dentry.inode == old_path.dentry.inode` | `Fs::do_linkat` |
| Post (success): `inode.i_nlink == prev + 1` | `vfs_link` |
| Post (error): refcounts balanced; no mutation | `Fs::do_linkat` |

### Layer 4: Verus/Creusot functional

`link(oldpath, newpath)` ≡ `linkat(AT_FDCWD, oldpath, AT_FDCWD, newpath, 0)` per POSIX-1.2008 + `man 2 link`.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`link(2)` reinforcement:

- **Per-`fs.protected_hardlinks` sysctl** — non-owner cannot link to a file they cannot read/write.
- **Per-mount `EXDEV` enforcement** — defense against cross-filesystem hard-link bypass of mount-isolation.
- **Per-`security_inode_link` LSM** — SELinux/AppArmor mediate link creation.
- **Per-`i_nlink` saturating counter** — defense against `i_nlink` integer-overflow leading to UAF on unlink-to-zero.
- **Per-`mnt_want_write` freeze barrier** — defense against link during filesystem freeze.
- **Per-`fsnotify_link` event** — auditable inotify/fanotify trail.
- **Per-`new_parent` inode-lock** — defense against concurrent rmdir/rename eating the parent.

### grsecurity/pax-style reinforcement

- **PAX_UDEREF on `oldpath`/`newpath`** — kernel pointer in argv rejected before any path walk.
- **GRKERNSEC_LINK** — non-root caller cannot hard-link to a file owned by another UID; `EPERM` even when DAC would otherwise permit.
- **GRKERNSEC_SYMLINKOWN** — when last component of `oldpath` is a symlink whose `uid != caller.uid`, `link(2)` returns `-EACCES` (defends `/tmp` race).
- **GRKERNSEC_FIFO** — analogous restrictions on linking to FIFOs/sockets owned by other UIDs in sticky-bit dirs.
- **GRKERNSEC_HIDESYM** — link-related dmesg redacts kernel pointers.
- **PAX_MEMORY_SANITIZE** — failed-link transient path buffer zeroed on free.
- **GRKERNSEC_CHROOT_NICE** — link inside chroot cannot reference a file outside the chroot via residual fd-walk.
- **GRKERNSEC_PROC_USER** — `/proc/$pid/fd` reflecting the linked path is owner-only.
- **GRKERNSEC_AUDIT_PTRACE** — link by ptracer onto tracee filesystem auditable.
- **GRKERNSEC_DMESG** — link-error dmesg lines CAP_SYSLOG-gated.

