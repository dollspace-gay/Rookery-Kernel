# Tier-5 syscall: chdir(2) — syscall 80

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/open.c (SYSCALL_DEFINE1(chdir), set_fs_pwd)
  - fs/namei.c (user_path_at, path lookup with LOOKUP_DIRECTORY)
  - include/linux/fs_struct.h (struct fs_struct, set_fs_pwd, set_fs_root)
  - arch/x86/entry/syscalls/syscall_64.tbl (80  common  chdir)
-->

## Summary

`chdir(2)` changes the calling thread group's current working directory (the "pwd" or "cwd") to the directory specified by a path string. The cwd is per-`fs_struct`; threads created via `clone(CLONE_FS)` share the same fs_struct and observe each other's cwd changes; threads without CLONE_FS have private fs_structs.

The cwd is used as the resolution starting point for every relative path the process subsequently opens. Critical for: shell `cd`, process-relative path resolution, container entrypoint setup.

## Signature

```c
int chdir(const char *path);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `path` | `const char *` | in | NUL-terminated UTF-8 path; absolute or relative to the current cwd. |

## Return value

| Value | Meaning |
|---|---|
| `0` | cwd updated. |
| `-1` + `errno` | Failure; cwd unchanged. |

## Errors

| errno | Trigger |
|---|---|
| `EFAULT` | `path` points outside the caller's address space. |
| `ENAMETOOLONG` | `path` exceeds PATH_MAX (4096) or a component exceeds NAME_MAX (255). |
| `ENOENT` | `path` does not exist. |
| `ENOTDIR` | A component of `path` is not a directory; or the final target is not a directory. |
| `EACCES` | Search permission denied on a component of `path`, or read+execute denied on the final dir (per LOOKUP_DIRECTORY DAC check). |
| `ELOOP` | Too many symlinks (>40) encountered during resolution. |
| `EIO` | Low-level I/O error reading a directory inode. |
| `ENOMEM` | Out of memory during path lookup. |
| `EPERM` | Hidden by grsec_chroot_chdir policy (target not reachable from chroot). |

## ABI surface

```text
__NR_chdir  (x86_64) = 80
__NR_chdir  (arm64)  = 49
__NR_chdir  (riscv)  = 49
__NR_chdir  (i386)   = 12
```

## Compatibility contract

REQ-1: Syscall number is **80** on x86_64. ABI-stable since v1.

REQ-2: Per-LOOKUP_DIRECTORY: path must resolve to a directory. Final component dereferenced through symlinks; the final inode must be a directory or -ENOTDIR.

REQ-3: Per-DAC: caller must have execute (search) permission on every directory component of the path, AND on the final directory.

REQ-4: Per-fs_struct: chdir updates `current->fs->pwd` (a struct path = {mnt, dentry}). Refcount on the previous path dropped; refcount on the new path incremented under `current->fs->lock`.

REQ-5: Per-CLONE_FS: tasks sharing fs_struct see the chdir effect; tasks with private fs_struct (default) do not.

REQ-6: Per-chroot: chdir resolution is constrained by `current->fs->root`. Per-CAP_SYS_CHROOT: caller cannot ascend above their root (the dotdot at root resolves to itself); per-grsec, even pointing chdir at a vfsmount outside the chroot is rejected.

REQ-7: Per-bind-mount: chdir into a bind-mount: the mnt of `current->fs->pwd` reflects the bind-mounted vfsmount; subsequent getcwd produces the bind-mount-relative path.

REQ-8: Per-symbolic link: each symlink in the path counts toward the SYMLINK_MAX (40) budget; final symlink dereferenced (no AT_SYMLINK_NOFOLLOW for chdir).

REQ-9: Per-EFAULT: if path is in unmapped memory, the strncpy_from_user returns -EFAULT.

REQ-10: Per-empty string: `path == ""` returns -ENOENT (Linux) or -EINVAL (some kernels — but Linux is ENOENT).

REQ-11: Per-LOOKUP_REVAL: lookup flag set to revalidate cached dentries for cwd transitions (NFS, FUSE).

REQ-12: Per-audit: AUDIT_SYSCALL chdir event records path and pwd-change for ausearch.

REQ-13: Per-fanotify FAN_CHDIR: parent watchers see the chdir event (rarely).

REQ-14: Per-/proc/self/cwd symlink: kernel exposes the cwd via this magic symlink; updated atomically on chdir.

REQ-15: Per-thread-local cwd: NOT supported by Linux directly; per-thread cwd requires CLONE_FS=0 and per-thread fs_struct (uncommon).

## Acceptance Criteria

- [ ] AC-1: chdir("/tmp") returns 0; subsequent open("foo") opens /tmp/foo.
- [ ] AC-2: chdir("/nonexistent") returns -ENOENT.
- [ ] AC-3: chdir("/etc/passwd") returns -ENOTDIR.
- [ ] AC-4: chdir to a directory without execute permission: -EACCES.
- [ ] AC-5: chdir within a chroot to a sibling of the chroot via ../..: stays at chroot.
- [ ] AC-6: chdir on a thread without CLONE_FS: only that thread's cwd changes.
- [ ] AC-7: chdir under symlink chain of 50: -ELOOP.
- [ ] AC-8: chdir to a bind-mounted dir: /proc/self/cwd reflects bind mount.
- [ ] AC-9: chdir followed by getcwd: round-trips path (modulo bind mounts).
- [ ] AC-10: Audit subsystem records chdir with full path and uid.

## Architecture

```rust
#[syscall(nr = 80, abi = "sysv")]
pub fn sys_chdir(path: UserPtr<u8>) -> isize {
    Fs::do_chdir(path)
}
```

`Fs::do_chdir(uptr) -> isize`:
1. let name = getname(uptr).map_err(|e| e)?;            /* EFAULT, ENAMETOOLONG */
2. let path = user_path_at(AT_FDCWD, &name, LOOKUP_FOLLOW | LOOKUP_DIRECTORY)?;
3. let r = Fs::check_chdir_chroot(&path);                /* grsec hook */
4. if r != 0 { path_put(path); putname(name); return r; }
5. /* DAC */
6. let dac = inode_permission(path.dentry.d_inode(), MAY_EXEC | MAY_CHDIR);
7. if dac != 0 { path_put(path); putname(name); return dac; }
8. /* Update fs_struct */
9. Fs::set_fs_pwd(current.fs, &path);
10. path_put(path);
11. putname(name);
12. 0

`Fs::set_fs_pwd(fs, path)`:
1. spin_lock(&fs.lock);
2. let old = fs.pwd.clone();
3. fs.pwd = path.clone();           /* increments path refcount */
4. spin_unlock(&fs.lock);
5. path_put(&old);

`Fs::check_chdir_chroot(path) -> isize`:
1. if !current.has_chroot() { return 0; }
2. if !path_is_under(&path, &current.fs.root) { return -EPERM; }
3. 0

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `path_is_dir` | INVARIANT | LOOKUP_DIRECTORY enforced; non-dir returns ENOTDIR. |
| `pwd_atomic_swap` | INVARIANT | set_fs_pwd uses fs.lock; old refcount dropped post-swap. |
| `dac_before_swap` | ORDER | DAC check passes BEFORE fs.pwd updated. |
| `chroot_confines_pwd` | INVARIANT | pwd always under fs.root. |

### Layer 2: TLA+

`fs/chdir.tla`:
- States: per-call, per-lookup, per-dac, per-grsec-chroot, per-swap.
- Properties:
  - `safety_pwd_dir_only` — pwd inode always a directory.
  - `safety_pwd_under_root` — pwd reachable from root under chroot.
  - `safety_atomic_swap` — pwd transitions are seqlock-consistent.
  - `liveness_chdir_terminates` — every chdir returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_chdir` post: success ⟹ pwd is the canonical path of arg | `Fs::do_chdir` |
| `set_fs_pwd` post: refcount balanced | `Fs::set_fs_pwd` |
| `check_chdir_chroot` post: -EPERM if outside chroot | `Fs::check_chdir_chroot` |

### Layer 4: Verus / Creusot functional

Per-`chdir(2)` man-page, per-POSIX, per-Linux fs/namei.c, ltp syscalls/chdir test suite.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`chdir(2)` reinforcement:

- **Per-path getname under PAX_USERCOPY** — defense against per-userspace-string overflow.
- **Per-LOOKUP_DIRECTORY** — defense against per-non-dir cwd corruption.
- **Per-DAC strict** — defense against per-DAC bypass.
- **Per-chroot adjacency check** — defense against per-chroot escape via chdir.
- **Per-fs.lock atomic swap** — defense against per-pwd-tearing.

## Grsecurity / PaX surface

- **PaX UDEREF on path string getname** — defense against per-userspace-pointer kernel deref; SMAP forced.
- **GRKERNSEC_CHROOT_CHDIR** — chdir(2) into a directory not reachable from the chroot is rejected with -EPERM, even if the path was constructed with absolute components. Closes the "chdir-to-mountpoint-then-fchdir-out" pattern.
- **GRKERNSEC_CHROOT_DOUBLE** — chrooted task that attempts chdir to a path resolving to the chroot's parent inode is denied.
- **GRKERNSEC_LINK** — chdir through a symlink owned by a different uid (or different group with safe-bit) is denied unless the destination is owned by the same uid.
- **CAP_DAC_OVERRIDE strict** — chdir does not honor CAP_DAC_OVERRIDE to bypass the search-permission check on intermediate components; standard DAC applies in full.
- **PAX_REFCOUNT on path refcount during set_fs_pwd** — defense against per-fs_struct UAF.
- **PaX KERNEXEC on path-lookup dispatch** — indirect call hardened.
- **Sync data-leak prevention** — chdir audit record sanitizes secrets; no leak through audit format.
- **GRKERNSEC_DMESG** — chdir failure klog (e.g. chroot-escape attempt) rate-limited and CAP_SYSLOG-gated.
- **GRKERNSEC_HIDESYM in chdir klog** — kernel pointers stripped.
- **PAX_USERCOPY_HARDEN on getname** — defense against per-string-copy overflow.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- Per-fchdir(2) variant (covered in Tier-5 fchdir.md).
- Per-getcwd(2) inverse (covered in Tier-5 getcwd.md).
- Per-chroot(2) (covered in Tier-5 chroot.md).
- Implementation code.
