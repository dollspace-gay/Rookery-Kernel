---
title: "Tier-5: syscall 82 — rename(2)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`rename(2)` is **x86_64 syscall 82**, the POSIX directory-entry rename primitive. It atomically removes the entry `oldpath` and creates a new entry `newpath` referring to the same inode, all on a single filesystem (`-EXDEV` otherwise). If `newpath` already exists, it is silently replaced — atomically, with no transient window where neither exists or both exist. If both `oldpath` and `newpath` refer to the same inode (hard links to one another), the call succeeds without effect. Renaming a directory across non-empty target directories returns `-ENOTEMPTY`. Renaming a directory into a path that would create a cycle (e.g., into its own descendant) returns `-EINVAL`. Internally `rename` is `renameat2(AT_FDCWD, oldpath, AT_FDCWD, newpath, 0)`.

Critical for: every editor's atomic save (`write tmp`; `rename tmp orig`), every package-manager file replacement, every Rookery init's atomic-config swap, every `mv` invocation, every `mktemp -d` then `mv` pattern, every git index swap.

### Acceptance Criteria

- [ ] AC-1: `rename("a", "b")` ⟹ `0`; `stat("b").st_ino == stat_prev("a").st_ino`; `stat("a") == -ENOENT`.
- [ ] AC-2: `rename("a", "existing_regular_file")` ⟹ atomic replace; old file unlinked.
- [ ] AC-3: `rename("dir", "newdir")` where `newdir` non-empty ⟹ `-ENOTEMPTY`.
- [ ] AC-4: `rename("dir", "dir/subdir/anything")` ⟹ `-EINVAL` (loop).
- [ ] AC-5: `rename("a", "a")` where both same dentry ⟹ `0`.
- [ ] AC-6: Cross-mount ⟹ `-EXDEV`.
- [ ] AC-7: `rename("dir", "regular")` ⟹ `-ENOTDIR`.
- [ ] AC-8: `rename("regular", "dir")` ⟹ `-EISDIR`.
- [ ] AC-9: Read-only FS ⟹ `-EROFS`.
- [ ] AC-10: Sticky-bit-dir cross-uid rename ⟹ `-EPERM`.

### Architecture

```
struct RenameArgs { oldpath: UserPtr<u8>, newpath: UserPtr<u8> }
```

`sys_rename(args) -> i32`:

1. `let oldname = strncpy_from_user_pax(args.oldpath, PATH_MAX)?;`
2. `let newname = strncpy_from_user_pax(args.newpath, PATH_MAX)?;`
3. Return `Fs::do_renameat2(AT_FDCWD, oldname, AT_FDCWD, newname, 0)`.

`Fs::do_renameat2(olddfd, oldname, newdfd, newname, flags) -> i32`:

1. `let (old_parent, old_dentry) = filename_parentat(olddfd, oldname, LookupFlags::empty())?;`
2. `let (new_parent, new_dentry) = filename_parentat(newdfd, newname, LookupFlags::empty())?;`
3. If `old_parent.mnt != new_parent.mnt` ⟹ `-EXDEV`.
4. If `old_dentry == new_dentry` ⟹ `0`.
5. Directory-loop / type-mismatch checks per REQ-7,8.
6. `may_delete(idmap, old_parent.dentry.inode(), &old_dentry, old_dentry.is_dir())?;`
7. If new_dentry exists: `may_delete(idmap, new_parent.dentry.inode(), &new_dentry, new_dentry.is_dir())?;`
   else: `may_create(idmap, new_parent.dentry.inode(), &new_dentry, old_dentry.is_dir())?;`
8. `mnt_want_write(new_parent.mnt)?;`
9. `security_inode_rename(old_parent.dentry.inode(), &old_dentry, new_parent.dentry.inode(), &new_dentry, flags)?;`
10. `lock_rename(&old_parent.dentry, &new_parent.dentry);`
11. `vfs_rename(&RenameData { old_dir: old_parent.dentry.inode(), old_dentry, new_dir: new_parent.dentry.inode(), new_dentry, flags })?;`
12. `fsnotify_move(old_parent.dentry.inode(), new_parent.dentry.inode(), &old_dentry.name(), old_dentry.is_dir(), &new_dentry, &old_dentry);`
13. `unlock_rename(&old_parent.dentry, &new_parent.dentry);`
14. `mnt_drop_write(new_parent.mnt);`
15. Return `0`.

### Out of Scope

- `renameat(2)` / `renameat2(2)` (separate Tier-5 docs).
- `RENAME_EXCHANGE`, `RENAME_NOREPLACE`, `RENAME_WHITEOUT` (covered with `renameat2.md`).
- VFS `i_op->rename` per-filesystem (Tier-3 per FS).
- Implementation code.

### signature

C (POSIX-1.2008):

```c
int rename(const char *oldpath, const char *newpath);
```

glibc wrapper: `__rename` → `INLINE_SYSCALL(rename, 2, oldpath, newpath)` on architectures exporting the bare syscall; otherwise forwarded to `renameat2(AT_FDCWD, oldpath, AT_FDCWD, newpath, 0)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE2(rename, const char __user *, oldname, const char __user *, newname);
```

Rookery dispatch:

```rust
pub fn sys_rename(oldpath: UserPtr<u8>, newpath: UserPtr<u8>) -> SyscallResult<i32>;
```

### parameters

| name    | type           | constraints                                                   | errno-on-bad |
|---------|----------------|---------------------------------------------------------------|--------------|
| oldpath | `const char *` | NUL-terminated; ≤ `PATH_MAX`; must exist                     | `EFAULT` / `ENAMETOOLONG` / `ENOENT` |
| newpath | `const char *` | NUL-terminated; ≤ `PATH_MAX`; parent must be writable        | `EFAULT` / `ENAMETOOLONG` / `EACCES` |

### return value

- Success: `0`.
- Failure: `< 0` — negated errno.

### errors

| errno          | condition                                                                  |
|----------------|----------------------------------------------------------------------------|
| `EACCES`       | Write access to parent of oldpath or newpath denied, or search permission on a prefix component denied. |
| `EBUSY`        | `oldpath` or `newpath` is in use as a mount point or working directory of a process. |
| `EDQUOT`       | Quota exhausted on the filesystem.                                         |
| `EFAULT`       | Either pointer is outside the caller's address space.                      |
| `EINVAL`       | New path is a subdirectory of old path, creating a loop; or `oldpath == newpath` and both are same inode (success) but invalid pair. |
| `EISDIR`       | `newpath` exists and is a directory while `oldpath` is not.                |
| `ELOOP`        | Too many symlinks in path resolution.                                      |
| `EMLINK`       | `oldpath` already has the maximum number of links, and would have to be linked into `newpath`. |
| `ENAMETOOLONG` | A path or component exceeds limits.                                        |
| `ENOENT`       | A prefix component does not exist or `oldpath` does not exist.             |
| `ENOMEM`       | Kernel memory exhausted.                                                   |
| `ENOSPC`       | The device has no room for the new directory entry.                        |
| `ENOTDIR`      | A prefix component of either path is not a directory; or `oldpath` is a directory and `newpath` exists but is not a directory. |
| `ENOTEMPTY`    | `newpath` is a non-empty directory.                                        |
| `EPERM`        | Sticky-bit dir DAC denial on either side; or filesystem does not support rename. |
| `EROFS`        | Either side is on a read-only filesystem.                                  |
| `EXDEV`        | `oldpath` and `newpath` are on different mounted filesystems.              |

### abi surface (constants + flags)

`rename(2)` is flagless. Internally calls `do_renameat2(AT_FDCWD, oldname, AT_FDCWD, newname, 0)`.

Related kernel symbols (most-used in the rename path):

- `vfs_rename(&rd)` — VFS layer (`struct renamedata`); coordinates locking of up to four inodes (old parent, new parent, old dentry inode, new dentry inode).
- `lock_rename(p1, p2)` — acquires both parent locks in canonical address order to prevent ABBA deadlock.
- `unlock_rename(p1, p2)` — releases.
- `may_delete(idmap, dir, victim, isdir)` — DAC + sticky check on victim removal side.
- `may_create(idmap, dir, dentry, lookup)` — DAC check on creation side.
- `security_inode_rename(old_dir, old_dentry, new_dir, new_dentry, flags)` — LSM hook.
- `fsnotify_move(old_dir, new_dir, old_name, is_dir, target_dentry, moved_dentry)` — inotify/fanotify event.

### compatibility contract

- REQ-1: Argument lowering: `%rdi=oldname (const char __user *)`, `%rsi=newname (const char __user *)`.
- REQ-2: UDEREF on both pointers.
- REQ-3: `strncpy_from_user_with_pax_check(oldname, PATH_MAX)`, same for newname.
- REQ-4: Resolve both parents + last components (`LOOKUP_PARENT | LOOKUP_RENAME_TARGET`).
- REQ-5: Cross-mount: `oldpath.mnt != newpath.mnt ⟹ -EXDEV`.
- REQ-6: Same-inode shortcut: if `old_dentry.inode() == new_dentry.inode()` AND `old_dentry == new_dentry` ⟹ return `0` (no-op).
- REQ-7: Directory loop: if `oldpath` is a directory AND `newpath`'s path traverses through `oldpath` ⟹ `-EINVAL`.
- REQ-8: Type mismatch:
  - `oldpath` dir, `newpath` non-dir ⟹ `-ENOTDIR`,
  - `oldpath` non-dir, `newpath` dir ⟹ `-EISDIR`.
- REQ-9: `may_delete(old_parent, old_dentry, isdir=old_is_dir)`.
- REQ-10: `may_create(new_parent, new_dentry, lookup=new_is_dir)` if newpath is fresh, else `may_delete` for the displaced one.
- REQ-11: `mnt_want_write(parent.mnt)?;` — `-EROFS`.
- REQ-12: `security_inode_rename(old_parent.inode, old_dentry, new_parent.inode, new_dentry, 0)`.
- REQ-13: `lock_rename(old_parent.dentry, new_parent.dentry);`
- REQ-14: `vfs_rename(&rd)` — atomic per-FS.
- REQ-15: `fsnotify_move(...);`
- REQ-16: `unlock_rename(...);`
- REQ-17: `mnt_drop_write(parent.mnt);`
- REQ-18: On failure, no state mutation.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pax_uderef_paths` | INVARIANT | Both pointers UDEREF-validated. |
| `xdev_blocks_mutation` | INVARIANT | `EXDEV` returned before any inode mutation. |
| `loop_detected` | INVARIANT | Renaming dir into descendant ⟹ `-EINVAL` before mutation. |
| `canonical_lock_order` | INVARIANT | `lock_rename` acquires parents in pointer-order; no ABBA. |
| `type_mismatch_rejected` | INVARIANT | dir↔non-dir mismatches `-ENOTDIR`/`-EISDIR` before mutation. |

### Layer 2: TLA+

`uapi/syscalls/rename.tla`:
- Per-call → resolve(both) → may_delete/may_create → lock_rename → vfs_rename → fsnotify → unlock.
- Properties:
  - `safety_atomic_replace`,
  - `safety_no_intermediate_state` — observers never see neither/both,
  - `safety_xdev_rejected`,
  - `safety_loop_rejected`,
  - `liveness_rename_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post (success): old name unresolvable; new name resolves to old's inode | `vfs_rename` |
| Post (success-replace): displaced inode `i_nlink` decremented; freed when no refs | `vfs_rename` |
| Post (error): no parent lock leak; refcounts balanced | `Fs::do_renameat2` |

### Layer 4: Verus/Creusot functional

`rename(oldpath, newpath)` ≡ `renameat2(AT_FDCWD, oldpath, AT_FDCWD, newpath, 0)` per POSIX-1.2008.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`rename(2)` reinforcement:

- **Per-`vfs_rename` atomicity** — defense against torn-state observation by inotify watchers.
- **Per-`lock_rename` canonical ordering** — defense against ABBA deadlock between two parents.
- **Per-`security_inode_rename`** — LSM mediates (SELinux/AppArmor scope enforcement).
- **Per-`fsnotify_move` paired old/new event** — auditable trail.
- **Per-`mnt_want_write` freeze barrier** — defense against rename during filesystem freeze.
- **Per-`-EXDEV`** — defense against cross-FS rename pretending to be atomic.
- **Per-directory-loop detection** — defense against UAF from cyclic dentry graph.

### grsecurity/pax-style reinforcement

- **PAX_UDEREF on `oldpath`/`newpath`** — strict user/kernel pointer split.
- **GRKERNSEC_LINK** — non-root caller cannot rename a file owned by another UID into a directory where they lack the corresponding link permission; uniform with `link(2)` policy.
- **GRKERNSEC_SYMLINKOWN** — rename of a symlink in a sticky-bit dir owned by another UID ⟹ `-EACCES`. Defends `/tmp` rename-race attacks.
- **Sticky-bit dir rules** — rename within sticky dir requires ownership of source file OR sticky dir OR `CAP_FOWNER`. This is core POSIX, reinforced by grsec to apply also to the **destination** side: caller must own destination too if destination's parent has sticky bit set.
- **GRKERNSEC_FIFO** — rename of FIFOs/sockets owned by another UID in sticky-bit dirs blocked.
- **PAX_MEMORY_SANITIZE** — displaced inode's pages zeroed on free.
- **GRKERNSEC_CHROOT_RENAME** — chroot'd process cannot rename across chroot boundary even via mount-namespace aliasing.
- **GRKERNSEC_HIDESYM** — rename-error dmesg redacts kernel pointers.
- **GRKERNSEC_AUDIT_PTRACE** — rename by ptracer on tracee FS auditable.
- **GRKERNSEC_DMESG** — rename-error dmesg lines CAP_SYSLOG-gated.

