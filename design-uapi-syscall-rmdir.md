---
title: "Tier-5: syscall 84 — rmdir(2)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`rmdir(2)` is **x86_64 syscall 84**, the POSIX directory removal primitive. It removes the directory `pathname`, which MUST be empty (only `.` and `..` entries present) — otherwise `-ENOTEMPTY` (alias `-EEXIST` on some legacy ABIs; Linux returns `ENOTEMPTY`). On success the directory's `i_nlink` drops to 0 (a fresh directory starts at 2: itself + `.`), and the parent's `i_nlink` is decremented by 1 (removing the child's contribution from the parent's `..`-pointing count). Like `unlink`, if a process is currently using the directory (e.g., as its cwd), the inode persists until that reference is released, but the directory entry is removed immediately, so further lookups via path return `ENOENT`. `rmdir` of a non-directory returns `-ENOTDIR`. `rmdir` of `.` returns `-EINVAL`, and `rmdir` of `..` returns `-ENOTEMPTY`.

Critical for: every `rm -r` (combined with recursive unlink), every Rookery service-stop cleanup, every container layer teardown, every `mktemp -d` cleanup, every test-runner that creates/deletes scratch dirs.

### Acceptance Criteria

- [ ] AC-1: `rmdir("emptydir")` ⟹ `0`; subsequent `stat("emptydir") == -ENOENT`.
- [ ] AC-2: `rmdir("nonempty")` ⟹ `-ENOTEMPTY`.
- [ ] AC-3: `rmdir("regular_file")` ⟹ `-ENOTDIR`.
- [ ] AC-4: `rmdir("dir/.")` ⟹ `-EINVAL`.
- [ ] AC-5: `rmdir("dir/..")` ⟹ `-ENOTEMPTY` (POSIX-specified).
- [ ] AC-6: After successful rmdir, parent dir's `i_nlink` -= 1.
- [ ] AC-7: Removed dir's `i_nlink` becomes 0.
- [ ] AC-8: Sticky-bit dir + non-owner ⟹ `-EPERM`.
- [ ] AC-9: rmdir on mount point ⟹ `-EBUSY`.
- [ ] AC-10: Read-only FS ⟹ `-EROFS`.

### Architecture

```
struct RmdirArgs { pathname: UserPtr<u8> }
```

`sys_rmdir(args) -> i32`:

1. `let name = strncpy_from_user_pax(args.pathname, PATH_MAX)?;`
2. Return `Fs::do_rmdir(AT_FDCWD, name)`.

`Fs::do_rmdir(dfd, name) -> i32`:

1. `let (parent, dentry, last) = filename_parentat_with_last(dfd, name, LookupFlags::DIRECTORY)?;`
2. If `last == "." ⟹ -EINVAL;`
3. If `last == ".." ⟹ -ENOTEMPTY;`
4. If `dentry.inode().i_mode_type() != FileType::Directory ⟹ -ENOTDIR;`
5. `mnt_want_write(parent.mnt)?;`
6. `security_inode_rmdir(parent.dentry.inode(), &dentry)?;`
7. `inode_lock(parent.dentry.inode());`
8. `inode_lock_nested(dentry.inode(), I_MUTEX_CHILD);`
9. `vfs_rmdir(idmap, parent.dentry.inode(), &dentry)?;`
10. `fsnotify_rmdir(parent.dentry.inode(), &dentry);`
11. `dnotify_parent(&dentry, DnotifyFlags::DELETE);`
12. `d_delete(&dentry);`
13. `inode_unlock(dentry.inode());`
14. `inode_unlock(parent.dentry.inode());`
15. `mnt_drop_write(parent.mnt);`
16. Return `0`.

### Out of Scope

- `unlinkat(2)` with `AT_REMOVEDIR` flag (separate Tier-5 doc; alternative entry-point for dir removal).
- `unlink(2)` — covered in `unlink.md`.
- VFS `i_op->rmdir` per-filesystem (Tier-3 per FS).
- Implementation code.

### signature

C (POSIX-1.2008):

```c
int rmdir(const char *pathname);
```

glibc wrapper: `__rmdir` → `INLINE_SYSCALL(rmdir, 1, pathname)`. Linux exposes no `rmdirat`; the flag-bearing `unlinkat(..., AT_REMOVEDIR)` covers that role.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE1(rmdir, const char __user *, pathname);
```

Rookery dispatch:

```rust
pub fn sys_rmdir(pathname: UserPtr<u8>) -> SyscallResult<i32>;
```

### parameters

| name     | type           | constraints                                                   | errno-on-bad |
|----------|----------------|---------------------------------------------------------------|--------------|
| pathname | `const char *` | NUL-terminated; ≤ `PATH_MAX`; last component MUST be a directory and empty | `EFAULT` / `ENAMETOOLONG` / `ENOTDIR` / `ENOTEMPTY` |

### return value

- Success: `0`.
- Failure: `< 0` — negated errno.

### errors

| errno          | condition                                                                  |
|----------------|----------------------------------------------------------------------------|
| `EACCES`       | Write access to parent denied; or search permission on prefix denied.      |
| `EBUSY`        | `pathname` is the root of a mount, or is a working directory of a process. |
| `EFAULT`       | `pathname` outside the user's address space.                               |
| `EINVAL`       | `pathname` ends with `.` (refers to current/parent directly).              |
| `ELOOP`        | Too many symlinks in path resolution.                                      |
| `ENAMETOOLONG` | Path/component too long.                                                   |
| `ENOENT`       | A prefix component does not exist.                                         |
| `ENOMEM`       | Kernel memory exhausted.                                                   |
| `ENOTDIR`      | A component of the prefix is not a directory; or `pathname` itself is not a directory. |
| `ENOTEMPTY`    | `pathname` contains entries other than `.` and `..`; or ends with `..`.    |
| `EPERM`        | Sticky-bit dir DAC denial; or filesystem doesn't permit dir removal.       |
| `EROFS`        | Read-only filesystem.                                                      |

### abi surface (constants + flags)

`rmdir(2)` is flagless. Internally calls `do_rmdir(AT_FDCWD, pathname)`.

Related kernel symbols:

- `vfs_rmdir(idmap, dir, dentry)` — VFS layer; takes parent `i_mutex` + victim `i_mutex`, calls `dir->i_op->rmdir`.
- `may_delete(idmap, dir, victim, isdir=true)` — DAC + sticky-bit check.
- `security_inode_rmdir(dir, dentry)` — LSM hook.
- `fsnotify_rmdir(dir, dentry)` — inotify/fanotify event.
- `dnotify_parent(dentry, DN_DELETE)` — legacy.
- `shrink_dcache_parent(dentry)` — invalidate any cached children dentries.
- `d_delete(dentry)` — mark dentry as negative.

### compatibility contract

- REQ-1: Argument lowering: `%rdi=pathname (const char __user *)`.
- REQ-2: UDEREF on `pathname`.
- REQ-3: `strncpy_from_user_with_pax_check(pathname, PATH_MAX)`.
- REQ-4: Resolve parent (`LOOKUP_PARENT | LOOKUP_DIRECTORY`); last component must NOT be followed if symlink (rmdir of symlink to dir ⟹ `-ENOTDIR`).
- REQ-5: If last component is `.` ⟹ `-EINVAL`.
- REQ-6: If last component is `..` ⟹ `-ENOTEMPTY`.
- REQ-7: If `dentry.inode().i_mode_type() != S_IFDIR` ⟹ `-ENOTDIR`.
- REQ-8: `may_delete(idmap, parent.inode, victim, isdir=true)`:
  - parent must be writable,
  - sticky-bit rules same as `unlink`.
- REQ-9: `mnt_want_write(parent.mnt)?;`
- REQ-10: `security_inode_rmdir(parent.inode, &dentry);`
- REQ-11: `inode_lock(parent.inode); inode_lock_nested(dentry.inode, I_MUTEX_CHILD);`
- REQ-12: `vfs_rmdir(idmap, parent.inode, &dentry)`:
  - call `dir->i_op->rmdir(dir, dentry)` — FS layer verifies emptiness,
  - on success: `clear_nlink(dentry.inode); detach_mounts(&dentry); dont_mount(&dentry); dec_nlink(parent.inode);`
  - if not empty: `-ENOTEMPTY`.
- REQ-13: `fsnotify_rmdir(parent.inode, &dentry); dnotify_parent(&dentry, DELETE);`
- REQ-14: `d_delete(&dentry);`
- REQ-15: `inode_unlock(dentry.inode); inode_unlock(parent.inode);`
- REQ-16: `mnt_drop_write(parent.mnt);`
- REQ-17: On failure, no state mutation.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pax_uderef_pathname` | INVARIANT | pathname UDEREF-validated. |
| `non_dir_rejected` | INVARIANT | `S_IFDIR` check before mutation; `-ENOTDIR` otherwise. |
| `nonempty_rejected` | INVARIANT | FS-layer empty check before nlink mutation. |
| `nested_lock_order` | INVARIANT | parent lock acquired before child lock. |
| `mount_point_rejected` | INVARIANT | rmdir on mount root ⟹ `-EBUSY`. |

### Layer 2: TLA+

`uapi/syscalls/rmdir.tla`:
- Per-call → resolve(parent + last) → may_delete → vfs_rmdir → fsnotify → d_delete.
- Properties:
  - `safety_emptiness_invariant`,
  - `safety_parent_nlink_balanced`,
  - `safety_nested_lock_order`,
  - `liveness_rmdir_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post (success): `dentry.inode().i_nlink == 0` | `vfs_rmdir` |
| Post (success): parent `i_nlink` -= 1 | `vfs_rmdir` |
| Post (success): dentry marked negative; cached children invalidated | `Fs::do_rmdir` |
| Post (error): no state mutation; locks released | `Fs::do_rmdir` |

### Layer 4: Verus/Creusot functional

`rmdir(pathname)` semantics per POSIX-1.2008 + `man 2 rmdir`.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`rmdir(2)` reinforcement:

- **Per-emptiness check at FS layer** — defense against silent data loss (non-empty dir removal).
- **Per-`may_delete` sticky-bit DAC** — defense against `/tmp` cross-uid removal.
- **Per-`security_inode_rmdir` LSM** — SELinux/AppArmor mediation.
- **Per-nested-lock-order (parent before child)** — defense against ABBA deadlock.
- **Per-`detach_mounts`** — defense against UAF if a covered mount remained over the removed dir.
- **Per-`shrink_dcache_parent`** — defense against stale cached children dentries.
- **Per-`fsnotify_rmdir`** — auditable trail.
- **Per-`mnt_want_write`** — defense against rmdir during filesystem freeze.

### grsecurity/pax-style reinforcement

- **PAX_UDEREF on `pathname`** — strict user/kernel pointer split.
- **GRKERNSEC_LINK** — uniform with `link(2)` policy: cross-uid rmdir in sticky-bit dirs more strictly gated than DAC.
- **GRKERNSEC_SYMLINKOWN** — if the path traverses through a symlink owned by another UID in a sticky-bit dir, rmdir returns `-EACCES`. Defends `/tmp` race attacks.
- **Sticky-bit dir rules** — uniform with `unlink(2)`: caller must own removed dir OR sticky dir OR have `CAP_FOWNER`.
- **GRKERNSEC_FIFO** — analogous restriction on directory removal containing FIFOs of other UIDs.
- **PAX_MEMORY_SANITIZE** — removed directory's data blocks zeroed on free; metadata cleared.
- **GRKERNSEC_CHROOT_NICE** — chroot'd process cannot rmdir outside chroot via residual mount aliasing.
- **GRKERNSEC_HIDESYM** — rmdir-error dmesg redacts kernel pointers.
- **GRKERNSEC_AUDIT_PTRACE** — rmdir by ptracer on tracee FS auditable.
- **GRKERNSEC_DMESG** — rmdir-error dmesg lines CAP_SYSLOG-gated.

