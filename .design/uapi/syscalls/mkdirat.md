# Tier-5 syscall: mkdirat(2) — syscall 258

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/namei.c (SYSCALL_DEFINE3(mkdirat), do_mkdirat, vfs_mkdir, inode_operations::mkdir)
  - include/uapi/asm-generic/fcntl.h (AT_FDCWD)
  - arch/x86/entry/syscalls/syscall_64.tbl (258  common  mkdirat)
-->

## Summary

`mkdirat(2)` creates a new directory entry at a path interpreted relative to an open directory file descriptor `dirfd` (or to the calling task's cwd when `dirfd == AT_FDCWD`). It is the fd-relative form of `mkdir(2)` that allows a process to perform creation atomically scoped to a captured directory, avoiding TOCTOU races on the parent-directory chain.

Internally mkdirat resolves the relative path under the dirfd, performs DAC + MAC + grsec checks on the parent, then dispatches the filesystem's `->mkdir(dir, dentry, mode)` to allocate the on-disk directory and link it into the parent. Critical for: openat-family path safety, container build pipelines, atomic mkdir-then-rename patterns.

## Signature

```c
int mkdirat(int dirfd, const char *pathname, mode_t mode);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `dirfd` | `int` | in | Directory fd or `AT_FDCWD` (-100). Path is relative to this fd (or cwd). |
| `pathname` | `const char *` | in | NUL-terminated path to create. Absolute paths ignore `dirfd`. |
| `mode` | `mode_t` | in | Permissions; masked by the process umask: `effective = mode & ~current_umask & 0777` plus the directory bit. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Directory created. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EBADF` | `dirfd` invalid and `pathname` relative. |
| `ENOTDIR` | `dirfd` is not a directory and `pathname` relative; or a non-final path component is not a directory. |
| `EFAULT` | `pathname` outside the caller's address space. |
| `ENAMETOOLONG` | Path or component too long. |
| `ENOENT` | A path-component ancestor does not exist; or `pathname` is empty (Linux). |
| `EEXIST` | The final pathname already exists (file or directory). |
| `ENOSPC` | No space for the new directory's inode/extent. |
| `EDQUOT` | Quota exceeded. |
| `EROFS` | Parent is on a read-only filesystem. |
| `EACCES` | Write/search permission denied on parent or ancestor. |
| `ELOOP` | Symlink loop. |
| `EPERM` | LSM (SELinux/AppArmor/grsec) denies; or filesystem disallows creation here. |
| `EIO` | Low-level inode/extent allocation failed. |
| `ENOMEM` | Out of memory for the dentry/inode. |

## ABI surface

```text
__NR_mkdirat  (x86_64) = 258
__NR_mkdirat  (arm64)  = 34
__NR_mkdirat  (riscv)  = 34
__NR_mkdirat  (i386)   = 296
```

## Compatibility contract

REQ-1: Syscall number is **258** on x86_64. Added in Linux 2.6.16 ("openat family").

REQ-2: Path resolution: `user_path_create(dirfd, pathname, &path, LOOKUP_DIRECTORY)`. The path's parent is resolved with intermediate-symlink traversal; the final component is NOT dereferenced (since it is to be created).

REQ-3: `dirfd == AT_FDCWD`: resolution relative to `current->fs->pwd`. Otherwise dirfd's `f_path` provides the relative anchor; the fd must be a directory (`O_DIRECTORY` recommended) or -ENOTDIR.

REQ-4: Mode: `mode_t & ~umask & 0777` — the directory bit S_IFDIR is set by the kernel; setuid/setgid/sticky bits permitted if the underlying filesystem accepts them.

REQ-5: Per-DAC: caller must have write + search (MAY_WRITE | MAY_EXEC) on the parent directory.

REQ-6: Per-MAC: LSM hook `security_path_mkdir(&path, dentry, mode)` invoked; SELinux file_alloc + dir_mkdir transitions evaluated.

REQ-7: Per-vfs_mkdir: kernel checks `S_NOSEC` and audit-trail, then calls `dir.i_op->mkdir(idmap, dir, dentry, mode)`. Filesystem allocates the inode and inserts the dentry.

REQ-8: Per-`current->fs->umask`: applied to `mode` before vfs_mkdir.

REQ-9: Per-readonly-mount: -EROFS without dispatching ->mkdir.

REQ-10: Per-quota: dquot accounting applied during inode/block allocation.

REQ-11: Per-cgroup: I/O attributed to the calling task.

REQ-12: Per-inotify / fanotify: IN_CREATE / FAN_CREATE delivered to watchers on the parent.

REQ-13: Per-audit: AUDIT_PATH on the parent + the new dentry.

REQ-14: Per-bind-mount: mkdirat on a path traversing a bind-mount: the new dir lives in the underlying sb of the bind-mount endpoint.

REQ-15: Per-idmapped mount: vfs_mkdir uses the mount's idmap to map the caller's effective uid/gid to the on-disk credentials.

REQ-16: Per-`O_TMPFILE` mode does NOT apply to mkdirat (no anonymous-dir creation).

REQ-17: Per-EINVAL: invalid mode bits (e.g. S_IFREG / S_IFLNK in mode) typically silently ignored by vfs_mkdir (only S_ISVTX, S_ISGID, S_ISUID + 0777 honored).

## Acceptance Criteria

- [ ] AC-1: mkdirat(AT_FDCWD, "newdir", 0755) creates ./newdir with perms 0755 & ~umask.
- [ ] AC-2: dirfd = open("/tmp", O_RDONLY|O_DIRECTORY); mkdirat(dirfd, "x", 0700) creates /tmp/x.
- [ ] AC-3: mkdirat(AT_FDCWD, "existing", 0755) returns -EEXIST.
- [ ] AC-4: mkdirat(AT_FDCWD, "missing/sub", 0755) returns -ENOENT.
- [ ] AC-5: mkdirat(-1, "x", 0755) returns -EBADF.
- [ ] AC-6: mkdirat with regular-file dirfd on relative path: -ENOTDIR.
- [ ] AC-7: mkdirat with absolute path ignores dirfd.
- [ ] AC-8: mkdirat on read-only mount: -EROFS.
- [ ] AC-9: mkdirat with mode 07777: setuid/setgid/sticky bits honored where allowed.
- [ ] AC-10: SELinux denies the dir_mkdir transition: -EACCES (typed).
- [ ] AC-11: Inotify IN_CREATE delivered to parent watchers.
- [ ] AC-12: idmapped mount: on-disk uid reflects mount idmap.

## Architecture

```rust
#[syscall(nr = 258, abi = "sysv")]
pub fn sys_mkdirat(dirfd: i32, pathname: UserPtr<u8>, mode: u32) -> isize {
    Fs::do_mkdirat(dirfd, pathname, mode)
}
```

`Fs::do_mkdirat(dirfd, pathname, mode) -> isize`:
1. let name = getname(pathname)?;
2. /* Resolve parent and locked dentry for the final component */
3. let (path, dentry) = user_path_create(dirfd, &name, LOOKUP_DIRECTORY)?;
4. /* Apply umask */
5. let mode = (mode & !current.fs.umask) & 0o777 | u32::from(S_IFDIR);
6. /* LSM hook */
7. let r = security_path_mkdir(&path, &dentry, mode);
8. if r != 0 { done_path_create(&path, dentry); putname(name); return r; }
9. /* grsec link / chroot adjacency */
10. let g = Fs::check_mkdirat_grsec(&path, &dentry);
11. if g != 0 { done_path_create(&path, dentry); putname(name); return g; }
12. /* DAC + ROFS checks performed inside vfs_mkdir */
13. let r = vfs_mkdir(idmap_of(&path), path.dentry.d_inode(), dentry, mode);
14. /* Notify watchers */
15. if r == 0 { fsnotify_mkdir(path.dentry.d_inode(), dentry); }
16. done_path_create(&path, dentry);
17. putname(name);
18. r

`Fs::vfs_mkdir(idmap, dir, dentry, mode) -> isize`:
1. /* DAC + ROFS + immutable + LSM */
2. let r = may_create(idmap, dir, dentry);
3. if r != 0 { return r; }
4. let op = dir.i_op.mkdir.ok_or(EPERM)?;
5. /* Filesystem allocator */
6. let r = op(idmap, dir, dentry, mode);
7. if r == 0 { fsnotify_link_count(dir); }
8. r

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `dac_before_create` | ORDER | may_create checks before fs->mkdir dispatch. |
| `umask_applied` | INVARIANT | mode = (mode & ~umask) & 0777 before vfs_mkdir. |
| `path_create_locks_parent` | INVARIANT | user_path_create takes dir->i_rwsem write-lock. |
| `dentry_released_on_error` | INVARIANT | done_path_create called on every path. |

### Layer 2: TLA+

`fs/mkdirat.tla`:
- States: per-call, per-lookup, per-lsm, per-grsec, per-mkdir-dispatch, per-fsnotify.
- Properties:
  - `safety_parent_locked_during_create` — i_rwsem write-locked across vfs_mkdir.
  - `safety_mode_masked` — final mode strictly bounded by umask.
  - `safety_no_create_on_rofs` — EROFS returns before dispatch.
  - `liveness_mkdirat_terminates` — every mkdirat returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_mkdirat` post: success ⟹ dir exists with masked mode | `Fs::do_mkdirat` |
| `vfs_mkdir` post: may_create gate passed | `Fs::vfs_mkdir` |
| `done_path_create` post: dentry refcount balanced | `Fs::do_mkdirat` |

### Layer 4: Verus / Creusot functional

Per-`mkdirat(2)` man-page, per-POSIX `mkdirat`, per-Linux fs/namei.c, ltp syscalls/mkdirat suite.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`mkdirat(2)` reinforcement:

- **Per-getname under PAX_USERCOPY** — defense against per-userspace-string overflow.
- **Per-dirfd validation** — defense against per-bad-fd UAF.
- **Per-DAC + LSM strict** — defense against per-policy bypass.
- **Per-umask masking** — defense against per-mode privilege escalation.
- **Per-i_rwsem write-lock** — defense against per-parallel mkdir race.

## Grsecurity / PaX surface

- **PaX UDEREF on pathname getname** — defense against per-userspace-pointer kernel deref; SMAP forced.
- **GRKERNSEC_CHROOT_FCHDIR** — mkdirat where `dirfd` was opened pre-chroot and resolves outside the chroot is denied with -EPERM; closes the per-retained-dirfd chroot escape.
- **GRKERNSEC_LINK** — mkdirat through a symlink owned by a different uid (and not in a safe-by-uid mode) is denied.
- **CAP_DAC_OVERRIDE strict** — mkdirat does NOT honor CAP_DAC_OVERRIDE to bypass parent search/write permission; the DAC check is enforced fully.
- **PAX_REFCOUNT on dentry / inode refcounts during mkdirat** — defense against per-mkdirat-induced refcount overflow / UAF.
- **PaX KERNEXEC on i_op->mkdir dispatch** — indirect-call hardened.
- **Sync data-leak prevention** — mkdirat klog and audit records sanitized; no kernel-pointer leak.
- **GRKERNSEC_DMESG** — mkdirat error klog rate-limited and CAP_SYSLOG-gated; failed creates do not signal kernel-internal state to unprivileged tasks.
- **GRKERNSEC_HIDESYM in mkdirat klog** — pointers stripped.
- **Per-namespace scoping** — mkdirat across a mount namespace boundary (impossible without dirfd capture) yields -EPERM.
- **Per-quota DoS guard** — repeated mkdirat hitting EDQUOT rate-limited via gr-anti-fork policy.
- **PAX_USERCOPY_HARDEN on getname** — defense against per-string overflow.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- Per-mkdir(2) (covered separately if needed).
- Per-filesystem ->mkdir internals (covered in Tier-3 fs/<fs>-dir.md).
- Per-LSM hook semantics (covered in Tier-3 security/lsm.md).
- Implementation code.
