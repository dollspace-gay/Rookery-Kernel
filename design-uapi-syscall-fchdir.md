---
title: "Tier-5 syscall: fchdir(2) — syscall 81"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`fchdir(2)` changes the calling thread group's current working directory (cwd) to the directory referenced by an open file descriptor. It is the fd-based counterpart of `chdir(2)`: no path string is involved, so it is immune to TOCTOU races against a path's parent-directory components and to filesystem ordering quirks during the resolution.

Historically the most dangerous escape vector from a chroot: a process retains a directory fd opened BEFORE chroot(2) (when the fd referred to a dir outside the chroot), then after chroot(2) calls fchdir(fd) — pre-grsec the kernel happily moves the cwd to the outside-chroot dir, after which "../" walks escape the chroot. GRKERNSEC_CHROOT_FCHDIR closes this hole. Critical for: container security, fd-based-path-walk APIs, root-relative chdir.

### Acceptance Criteria

- [ ] AC-1: fd = open("/tmp", O_RDONLY|O_DIRECTORY); fchdir(fd) returns 0; cwd is /tmp.
- [ ] AC-2: fd = open("/etc/passwd", O_RDONLY); fchdir(fd) returns -ENOTDIR.
- [ ] AC-3: fchdir(-1) returns -EBADF.
- [ ] AC-4: O_PATH fd on a dir without search bit: -EACCES at fchdir.
- [ ] AC-5: Pre-chroot: open("/", O_RDONLY|O_DIRECTORY); chroot("/jail"); fchdir(saved_fd): vanilla = success+escape; grsec = -EPERM.
- [ ] AC-6: fchdir on an fd whose dir was unlinked: succeeds; getcwd shows "(deleted)".
- [ ] AC-7: fchdir on a bind-mounted dir fd: cwd reflects bind mount.
- [ ] AC-8: fchdir in a thread without CLONE_FS: only that thread's cwd changes.
- [ ] AC-9: Audit logs fchdir with fd resolution.

### Architecture

```rust
#[syscall(nr = 81, abi = "sysv")]
pub fn sys_fchdir(fd: i32) -> isize {
    Fs::do_fchdir(fd)
}
```

`Fs::do_fchdir(fd) -> isize`:
1. let file = fdget(fd).ok_or(EBADF)?;
2. let inode = file.f_inode();
3. if !inode.is_dir() { fdput(file); return -ENOTDIR; }
4. /* DAC re-check */
5. let r = inode_permission(inode, MAY_EXEC | MAY_CHDIR);
6. if r != 0 { fdput(file); return r; }
7. /* grsec chroot-escape check */
8. let g = Fs::check_fchdir_chroot(&file.f_path);
9. if g != 0 { fdput(file); return g; }
10. /* Update fs_struct */
11. Fs::set_fs_pwd(current.fs, &file.f_path);
12. fdput(file);
13. 0

`Fs::check_fchdir_chroot(path) -> isize`:
1. /* GRKERNSEC_CHROOT_FCHDIR */
2. if !grsec_chroot_fchdir { return 0; }
3. if !current.has_chroot() { return 0; }
4. if !path_is_under(&path, &current.fs.root) {
5.   gr_log_chroot_fchdir(&path);
6.   return -EPERM;
7. }
8. 0

### Out of Scope

- Per-chdir(2) string variant (covered in Tier-5 chdir.md).
- Per-getcwd(2) inverse (covered in Tier-5 getcwd.md).
- Per-openat AT_FDCWD semantics (covered in Tier-5 openat.md).
- Implementation code.

### signature

```c
int fchdir(int fd);
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `fd` | `int` | in | Open file descriptor referring to a directory. |

### return value

| Value | Meaning |
|---|---|
| `0` | cwd updated. |
| `-1` + `errno` | Failure; cwd unchanged. |

### errors

| errno | Trigger |
|---|---|
| `EBADF` | `fd` invalid. |
| `ENOTDIR` | `fd` is not a directory. |
| `EACCES` | Search permission on the directory denied (DAC re-check at fchdir time). |
| `EPERM` | grsec_chroot_fchdir rejects an attempted chroot escape. |

### abi surface

```text
__NR_fchdir  (x86_64) = 81
__NR_fchdir  (arm64)  = 50
__NR_fchdir  (riscv)  = 50
__NR_fchdir  (i386)   = 133
```

### compatibility contract

REQ-1: Syscall number is **81** on x86_64. ABI-stable.

REQ-2: Per-fd: must be open and refer to a directory inode (S_ISDIR). Any other inode type returns -ENOTDIR.

REQ-3: Per-DAC: caller must have execute (search) permission on the directory at fchdir time (re-evaluated, not cached from open). This catches the case where the open succeeded under elevated creds and the caller later dropped privileges.

REQ-4: Per-fs_struct: fchdir updates `current->fs->pwd` atomically with the spin-lock as in chdir(2). path = file.f_path (mnt + dentry).

REQ-5: Per-chroot:
- Vanilla Linux: fchdir to a dir outside chroot succeeds; this is a known escape vector when combined with an fd carried across chroot.
- grsec GRKERNSEC_CHROOT_FCHDIR: fchdir on an fd whose path is NOT under `current->fs->root` is rejected with -EPERM.

REQ-6: Per-CLONE_FS: tasks sharing fs_struct see the cwd change.

REQ-7: Per-mount-namespace: fchdir on a dir whose vfsmount is not visible in the current mount namespace returns -ENOENT (path resolution fails to canonicalize the mnt).

REQ-8: Per-/proc/self/cwd: updated atomically.

REQ-9: Per-audit: AUDIT_SYSCALL fchdir event with fd and resolved path.

REQ-10: fchdir does NOT dereference symlinks (no path resolution); the fd already designates a concrete dentry/inode.

REQ-11: Per-bind-mount: fchdir uses the file's `f_path.mnt`, so the cwd inherits the bind-mount; getcwd reflects the bind-mount path.

REQ-12: Per-O_PATH fd: O_PATH fds may be passed to fchdir even though they do not permit read/write. The DAC re-check still applies (search bit on the dir).

REQ-13: Per-unlinked-dir fd: if the directory has been unlinked since the fd was opened (but the fd holds the inode alive), fchdir succeeds; subsequent relative opens may fail because the dentry is now negative or detached. /proc/self/cwd shows the path as "<deleted>".

REQ-14: Per-fanotify FAN_CHDIR: emitted (rarely).

REQ-15: fchdir is interruptible: not interrupted by signals (kernel path; pure pointer swap).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `fd_dir_only` | INVARIANT | non-dir fd ⟹ ENOTDIR. |
| `dac_re_evaluated` | INVARIANT | DAC checked at fchdir time, not cached from open. |
| `grsec_chroot_fchdir_enforced` | INVARIANT | chroot escape attempt ⟹ EPERM. |
| `pwd_atomic_swap` | INVARIANT | set_fs_pwd holds fs.lock; old refcount dropped post-swap. |

### Layer 2: TLA+

`fs/fchdir.tla`:
- States: per-call, per-fdget, per-dir-check, per-dac, per-grsec, per-swap.
- Properties:
  - `safety_pwd_dir_only` — pwd inode always a directory.
  - `safety_chroot_no_escape_via_fchdir` — grsec mode: pwd always under root post-fchdir.
  - `safety_dac_at_call_time` — dropped creds re-evaluated.
  - `liveness_fchdir_terminates` — every fchdir returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_fchdir` post: success ⟹ pwd = file.f_path | `Fs::do_fchdir` |
| `check_fchdir_chroot` post: chroot mode ⟹ -EPERM on escape | `Fs::check_fchdir_chroot` |
| `set_fs_pwd` post: refcount balanced | `Fs::set_fs_pwd` |

### Layer 4: Verus / Creusot functional

Per-`fchdir(2)` man-page, per-Linux fs/open.c, ltp syscalls/fchdir test suite.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`fchdir(2)` reinforcement:

- **Per-fd validation pre-use** — defense against per-bad-fd kernel UAF.
- **Per-DAC re-check at fchdir** — defense against per-dropped-priv-fd reuse.
- **Per-chroot fchdir escape check** — defense against per-chroot escape via retained dir-fd.
- **Per-fs.lock atomic swap** — defense against per-pwd tearing.
- **Per-O_PATH DAC honored** — defense against per-search-bit bypass.

### grsecurity / pax surface

- **PaX UDEREF on fd table lookup** — defense against per-fd-table TOCTOU; SMAP forced.
- **GRKERNSEC_CHROOT_FCHDIR** — primary defense. fchdir(2) on an fd whose path is not under `current->fs->root` is rejected with -EPERM when the caller is chrooted. Defense against the classical "open-dir-pre-chroot then chroot then fchdir(saved_fd) to escape" exploit. Logged via gr_log_chroot_fchdir.
- **GRKERNSEC_CHROOT_DOUBLE** — fchdir to a dir whose inode equals the chroot's parent is denied.
- **GRKERNSEC_LINK** — fchdir on an fd opened through a symlink owned by a different uid is denied unless safe-by-uid match.
- **CAP_DAC_OVERRIDE strict** — fchdir does NOT honor CAP_DAC_OVERRIDE; the search-bit DAC re-check is enforced even for CAP_DAC_OVERRIDE holders to prevent priv-drop fd reuse.
- **PAX_REFCOUNT on file refcount and fs_struct pwd path refcount** — defense against per-fchdir-induced UAF.
- **PaX KERNEXEC on dispatch** — indirect-call hardened.
- **Sync data-leak prevention** — fchdir audit and klog records sanitized; no kernel-pointer leak.
- **GRKERNSEC_DMESG** — fchdir error klog (including chroot-escape attempts) rate-limited and CAP_SYSLOG-gated.
- **GRKERNSEC_HIDESYM in fchdir klog** — kernel pointers and dentry addresses stripped.
- **Per-namespace scoping** — fchdir on an fd whose path resolves to a vfsmount outside the current mount namespace returns -EPERM.

