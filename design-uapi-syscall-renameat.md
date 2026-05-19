---
title: "Tier-5 syscall: renameat(2) — syscall 264"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`renameat(2)` atomically renames a file or directory whose path is interpreted relative to an open directory file descriptor `olddirfd`, to a new path interpreted relative to a (possibly different) open directory file descriptor `newdirfd`. It is the fd-relative form of `rename(2)`, immune to TOCTOU races on the parent-directory chains of either source or destination.

renameat is dispatched internally through the renameat2-engine with `flags=0`. The atomic guarantee: at no point is the source absent from the namespace; at no point is the destination both present-as-old and present-as-new. The fs-specific `->rename` implementation is invoked under a carefully ordered locking dance (lock_rename) that pins both parents and (if a target exists) the target dentry against concurrent unlink/rename. Critical for: atomic file replacement, mailbox swap, write-temp-then-rename durability pattern.

### Acceptance Criteria

- [ ] AC-1: write data to "tmp"; renameat(AT_FDCWD, "tmp", AT_FDCWD, "final"): tmp gone, final has the data.
- [ ] AC-2: renameat across different fds: fdA = open("/a", O_DIRECTORY); fdB = open("/b", O_DIRECTORY); renameat(fdA, "x", fdB, "y"): /a/x → /b/y.
- [ ] AC-3: renameat across mounts: -EXDEV.
- [ ] AC-4: renameat directory into its own descendant: -EINVAL.
- [ ] AC-5: renameat onto an existing non-empty dir: -ENOTEMPTY.
- [ ] AC-6: renameat onto a file when source is a dir: -EISDIR.
- [ ] AC-7: renameat where dest dir has sticky bit and caller does not own source: -EPERM.
- [ ] AC-8: renameat on a read-only mount: -EROFS.
- [ ] AC-9: renameat on identical paths: returns 0.
- [ ] AC-10: renameat with bad dirfd and relative path: -EBADF.
- [ ] AC-11: Inotify watchers see IN_MOVED_FROM + IN_MOVED_TO with matching cookie.
- [ ] AC-12: Atomic: a concurrent reader observing either old or new path never observes both gone.

### Architecture

```rust
#[syscall(nr = 264, abi = "sysv")]
pub fn sys_renameat(olddfd: i32, oldpath: UserPtr<u8>, newdfd: i32, newpath: UserPtr<u8>) -> isize {
    Fs::do_renameat2(olddfd, oldpath, newdfd, newpath, 0)
}
```

`Fs::do_renameat2(olddfd, oldpath, newdfd, newpath, flags) -> isize`:
1. let oldname = getname(oldpath)?;
2. let newname = getname(newpath)?;
3. /* Resolve parents + locked target dentries */
4. let (old_path, old_dentry) = user_path_parent(olddfd, &oldname, LOOKUP_RENAME_SOURCE)?;
5. let (new_path, new_dentry) = user_path_parent(newdfd, &newname, LOOKUP_RENAME_TARGET)?;
6. /* Cross-fs check */
7. if old_path.mnt != new_path.mnt { rc = -EXDEV; goto out; }
8. /* Same-path early-exit */
9. if old_dentry == new_dentry { rc = 0; goto out; }
10. /* LSM */
11. rc = security_path_rename(&old_path, &old_dentry, &new_path, &new_dentry, flags);
12. if rc != 0 { goto out; }
13. /* grsec link / chroot adjacency */
14. rc = Fs::check_renameat_grsec(&old_path, &old_dentry, &new_path, &new_dentry);
15. if rc != 0 { goto out; }
16. /* Acquire ordered locks */
17. let trap = lock_rename(&new_path.dentry, &old_path.dentry);
18. /* Disallow cycles */
19. if trap == old_dentry { rc = -EINVAL; unlock_rename; goto out; }
20. /* Dispatch to fs ->rename */
21. rc = vfs_rename(idmap_of(&old_path), old_path.dentry.d_inode(), old_dentry, new_path.dentry.d_inode(), new_dentry, flags);
22. /* fsnotify */
23. if rc == 0 {
24.   let cookie = fsnotify_get_cookie();
25.   fsnotify(old_path.dentry.d_inode(), FS_MOVED_FROM, old_dentry, cookie);
26.   fsnotify(new_path.dentry.d_inode(), FS_MOVED_TO, new_dentry, cookie);
27. }
28. unlock_rename(&new_path.dentry, &old_path.dentry);
29. out: cleanup paths and names; return rc.

`Fs::vfs_rename(idmap, old_dir, old_dentry, new_dir, new_dentry, flags) -> isize`:
1. let rc = may_rename(idmap, old_dir, old_dentry, new_dir, new_dentry, flags);
2. if rc != 0 { return rc; }
3. let op = old_dir.i_op.rename.ok_or(EPERM)?;
4. let rc = op(idmap, old_dir, old_dentry, new_dir, new_dentry, flags);
5. if rc == 0 { d_move(old_dentry, new_dentry); }
6. rc

### Out of Scope

- Per-rename(2) (covered separately if needed).
- Per-renameat2(2) flags (RENAME_NOREPLACE / EXCHANGE / WHITEOUT) (covered in Tier-5 renameat2.md).
- Per-filesystem ->rename internals (covered in Tier-3 fs/<fs>-rename.md).
- Per-LSM hook semantics (covered in Tier-3 security/lsm.md).
- Implementation code.

### signature

```c
int renameat(int olddirfd, const char *oldpath,
             int newdirfd, const char *newpath);
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `olddirfd` | `int` | in | Source directory fd or `AT_FDCWD` (-100). |
| `oldpath` | `const char *` | in | NUL-terminated source path; absolute paths ignore `olddirfd`. |
| `newdirfd` | `int` | in | Destination directory fd or `AT_FDCWD`. |
| `newpath` | `const char *` | in | NUL-terminated destination path; absolute paths ignore `newdirfd`. |

### return value

| Value | Meaning |
|---|---|
| `0` | Rename completed. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EBADF` | `olddirfd` or `newdirfd` invalid and the corresponding path is relative. |
| `ENOTDIR` | A dirfd is not a directory; or a non-final path component is not a directory; or `oldpath` is a directory and `newpath` exists as non-directory; or vice versa. |
| `EFAULT` | path outside the caller's address space. |
| `ENAMETOOLONG` | path or component too long. |
| `ENOENT` | source does not exist; or a path-component ancestor missing. |
| `EEXIST` | with `RENAME_NOREPLACE`: destination exists. |
| `EXDEV` | source and destination on different mounts/filesystems (rename(2) is single-filesystem). |
| `ENOTEMPTY` | destination is a non-empty directory. |
| `EISDIR` | destination is a directory and source is not. |
| `EBUSY` | source or destination is a mount point. |
| `EROFS` | filesystem read-only. |
| `EACCES` | DAC permission denied on either parent. |
| `EPERM` | LSM denies; immutable/append-only; sticky-bit on parent rejects (RENAME of file not owned in sticky dir). |
| `ELOOP` | symlink loop. |
| `EINVAL` | newpath descended into oldpath subtree; or invalid combination of dirfds (zero `oldpath`/`newpath`). |
| `ENOMEM` | OOM during dentry/inode work. |
| `ENOSPC` | journal/extent allocation failed. |

### abi surface

```text
__NR_renameat   (x86_64) = 264
__NR_renameat   (arm64)  = (not exposed; arm64 uses renameat2 = 38)
__NR_renameat   (riscv)  = (not exposed; riscv uses renameat2 = 38)
__NR_renameat   (i386)   = 302
```

Note: arm64/riscv expose only renameat2(2) (with flags); kernel renameat(olddfd, old, newdfd, new) reduces to renameat2(olddfd, old, newdfd, new, 0).

### compatibility contract

REQ-1: Syscall number is **264** on x86_64. Added in Linux 2.6.16.

REQ-2: Atomic semantics: per-POSIX, rename is atomic — at no instant is the destination missing if it existed at start; the source is unobservable post-rename. Linux implements this via `lock_rename(old_dir, new_dir)` which acquires `s_vfs_rename_mutex` for cross-directory and the appropriate `i_rwsem` write-locks.

REQ-3: Per-vfs_rename: kernel-side wrapper invokes `old_dir.i_op->rename(idmap, old_dir, old_dentry, new_dir, new_dentry, flags)`. Filesystem must:
- Re-evaluate that source still exists and target state matches expectations.
- Update directory entries atomically (typically a journaled inode-table + dirent update).
- Bump i_nlink correctly for directory-rename (parent of new dir gets ++; parent of old dir gets --).

REQ-4: Per-EXDEV: rename(2) is single-fs/single-mount. Cross-fs requires copy + unlink (userspace).

REQ-5: Per-`oldpath == newpath`: returns 0 (no-op) per POSIX.

REQ-6: Per-EBUSY: source or destination is a vfsmount root: -EBUSY.

REQ-7: Per-directory rename: source dir's parent's "..\" inside the source dir is updated to point to the new parent (for fs that track it).

REQ-8: Per-cycle detection: if newpath descends inside oldpath subtree (directory rename into its own descendant), -EINVAL.

REQ-9: Per-DAC + sticky:
- Caller needs MAY_WRITE | MAY_EXEC on both parents.
- Sticky-bit on parent (`S_ISVTX`): caller must own the source, or own the parent, or be root.

REQ-10: Per-LSM: `security_path_rename(&old_path, &old_dentry, &new_path, &new_dentry, flags)` invoked.

REQ-11: Per-fsnotify: IN_MOVED_FROM (old parent), IN_MOVED_TO (new parent), and per-inode IN_MOVE_SELF emitted under a shared `move_cookie`.

REQ-12: Per-audit: AUDIT_PATH records on both old and new dentries.

REQ-13: Per-idmapped mount: vfs_rename respects the idmap of the relevant mounts; cross-idmap rename requires CAP_FSETID and CAP_CHOWN considerations.

REQ-14: Per-quota: rename across users (chown-via-rename impossible since rename doesn't change owner) does not engage dquot.

REQ-15: Per-readonly-mount: -EROFS.

REQ-16: Per-O_PATH dirfds: accepted for olddirfd/newdirfd resolution.

REQ-17: renameat(olddfd, "", newdfd, ""): -ENOENT (empty path).

REQ-18: Per-grsec_link: rename through a symlink owned by a different uid is denied (no symlink-as-target rename allowed).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `lock_rename_order_invariant` | INVARIANT | lock_rename takes locks in a globally-consistent address order (ABBA-free). |
| `same_mount_required` | INVARIANT | cross-mount ⟹ -EXDEV before dispatch. |
| `dac_lsm_grsec_before_rename` | ORDER | DAC + LSM + grsec gates evaluated BEFORE vfs_rename. |
| `atomic_rename` | INVARIANT | at no observable instant is both old and new absent. |
| `cycle_rejected` | INVARIANT | dir rename into descendant ⟹ -EINVAL. |

### Layer 2: TLA+

`fs/renameat.tla`:
- States: per-call, per-lookup-old, per-lookup-new, per-lock-rename, per-dispatch, per-fsnotify, per-unlock.
- Properties:
  - `safety_lock_ordering` — lock_rename order globally consistent.
  - `safety_atomic_rename` — atomicity preserved against concurrent readers.
  - `safety_no_cross_fs` — EXDEV enforced.
  - `safety_no_dir_into_self` — cycle rejected.
  - `liveness_renameat_terminates` — every renameat returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_renameat2` post: success ⟹ source absent, target present with old inode | `Fs::do_renameat2` |
| `vfs_rename` post: may_rename passed; d_move applied | `Fs::vfs_rename` |
| `lock_rename` post: both parents' i_rwsem held; trap dentry recorded | `Fs::do_renameat2` |

### Layer 4: Verus / Creusot functional

Per-`renameat(2)` man-page, per-POSIX `renameat`, per-Linux fs/namei.c, ltp syscalls/renameat suite.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`renameat(2)` reinforcement:

- **Per-getname under PAX_USERCOPY** — defense against per-userspace-string overflow.
- **Per-dirfd validation (both)** — defense against per-bad-fd UAF.
- **Per-DAC + LSM + sticky-bit strict** — defense against per-policy bypass.
- **Per-lock_rename strict ordering** — defense against per-ABBA-deadlock.
- **Per-cross-mount EXDEV** — defense against per-cross-fs corruption.
- **Per-cycle rejection** — defense against per-rename-into-self orphan tree.

### grsecurity / pax surface

- **PaX UDEREF on getname for both oldpath/newpath** — defense against per-userspace-pointer kernel deref; SMAP forced.
- **GRKERNSEC_CHROOT_FCHDIR adjacency** — renameat with olddirfd or newdirfd whose path resolves outside the caller's chroot is denied with -EPERM. Closes the "open-dir-pre-chroot, then renameat with retained fd" escape.
- **GRKERNSEC_LINK** — renameat where either oldpath or newpath traverses a symlink owned by a different uid (and not under the safe-by-uid policy) is denied; also blocks rename-onto-symlink-target attacks.
- **CAP_DAC_OVERRIDE strict** — renameat does NOT honor CAP_DAC_OVERRIDE to bypass parent search/write or sticky-bit checks. DAC + sticky enforced fully.
- **PAX_REFCOUNT on dentry/inode refcounts during renameat** — defense against per-rename-induced refcount overflow / UAF, especially on the move_cookie / dentry-graft path.
- **PaX KERNEXEC on i_op->rename dispatch** — indirect-call hardened.
- **Sync data-leak prevention** — rename audit and klog records sanitized.
- **GRKERNSEC_DMESG** — renameat error klog rate-limited and CAP_SYSLOG-gated; renaming failures do not signal kernel-internal state to unprivileged tasks.
- **GRKERNSEC_HIDESYM in renameat klog** — pointers stripped.
- **Per-namespace scoping** — renameat across a mount namespace boundary (impossible without dirfd capture) yields -EPERM.
- **Per-fsnotify cookie sanitized** — cookie is a random nonce per-rename, not a kernel pointer; defense against per-cookie info leak.
- **PAX_USERCOPY_HARDEN on getname** — defense against per-string overflow.
- **Per-immutable / append-only honored** — rename of immutable file or onto append-only target denied with -EPERM regardless of caps.

