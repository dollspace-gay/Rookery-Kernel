# Tier-5: syscall 161 — chroot(2)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/open.c (sys_chroot, ksys_chroot)
  - fs/fs_struct.c (set_fs_root)
  - include/uapi/linux/capability.h (CAP_SYS_CHROOT)
-->

## Summary

`chroot(2)` changes the calling process's notional `/`. Subsequent absolute path lookups, `..` past the root, and getcwd traverse are clamped to a subtree rooted at `path`. Per-CAP_SYS_CHROOT required. Per-`chroot` does **not** change cwd: callers traditionally `chdir(path)` before `chroot(path)` (else cwd escapes). Per-chroot is **not** a security boundary in the kernel's classic sense — root inside chroot can escape via `chdir + ..`, ptrace, mount, or pivot — but Rookery's grsecurity-style hardening converts the upstream loophole into a hard wall (CHROOT_* family). Critical for: pre-namespace daemon containment, mock build roots, hardened legacy isolation.

This Tier-5 covers `syscall 161 chroot`.

## Signature

```c
int chroot(const char *path);
```

Per-x86_64-syscall-table: `__NR_chroot = 161`. Per-glibc: `unistd.h`. Per-vDSO: not vDSO-vectored.

## Parameters

| Name | Type | Direction | Meaning |
|---|---|---|---|
| `path` | `const char *` | in | New root directory; must be a directory accessible to the caller. |

## Return

- 0 on success.
- -1 with `errno` on failure.

## Errors

| errno | Cause |
|---|---|
| EPERM | Caller lacks CAP_SYS_CHROOT in the user-namespace. |
| EACCES | Search permission denied on a component, or no X on final directory. |
| ENOENT | Path component does not exist. |
| ENOTDIR | A path component is not a directory; or final is not a directory. |
| ENAMETOOLONG | Path > PATH_MAX. |
| ELOOP | Symlink loop. |
| EFAULT | `path` invalid pointer. |
| EIO | I/O on path lookup. |
| ENOMEM | Kernel allocation failed. |

## ABI surface

- Syscall number: `__NR_chroot = 161`.
- No flags. Single path arg.
- Per-task scope: affects only current's fs_struct.root (CLONE_FS sharing makes it visible to peers sharing fs_struct).
- Glibc and musl pass through; not vDSO.

## Compatibility contract

REQ-1: Lookup:
- path resolution via user_path_at(AT_FDCWD, path, LOOKUP_FOLLOW | LOOKUP_DIRECTORY).
- LOOKUP_FOLLOW: final component symlink followed.
- LOOKUP_DIRECTORY: must resolve to directory.

REQ-2: Capability:
- ns_capable(current.cred.user_ns, CAP_SYS_CHROOT).
- May_chroot enforces; in older kernels, CAP_SYS_CHROOT in init_user_ns; modern: namespace-aware.

REQ-3: Permission:
- inode_permission(inode, MAY_EXEC) on target.

REQ-4: Side effects:
- current.fs_struct.root = (path, mnt).
- mntget(path.mnt), dget(path.dentry); old root ref dropped via Drop.
- cwd unchanged.

REQ-5: Per-CLONE_FS:
- If multiple tasks share fs_struct (CLONE_FS), set_fs_root affects them all atomically under fs->lock.

REQ-6: Per-double-chroot:
- Allowed by POSIX semantics: nested chroots collapse upward unconditionally.
- Grsecurity GRKERNSEC_CHROOT_DOUBLE refuses second chroot from within a chroot.

REQ-7: Per-`..` semantics:
- Path lookup intercepted at root: `..` at root returns root itself.
- This is the only structural barrier; not a security barrier without grsec hardening.

REQ-8: Per-cwd-not-under-root:
- chroot does not chdir. cwd may dangle outside new root.
- POSIX behavior: subsequent absolute lookups clamped; relative lookups from outside-root cwd can still resolve outside.
- Userspace pattern: chdir(target); chroot(".");

REQ-9: Per-mount-propagation:
- chroot does not change mount namespace. The new fs.root anchors lookups but mnt_ns retains all mounts.

REQ-10: Per-success: returns 0; subsequent path-walks see clamped root.

REQ-11: Per-thread visibility:
- chroot effect on current. Other tasks in same fs_struct see it; others do not.

## Acceptance Criteria

- [ ] AC-1: chdir("/jail"); chroot("/jail"): subsequent open("/etc/passwd") opens /jail/etc/passwd.
- [ ] AC-2: Non-CAP_SYS_CHROOT: EPERM.
- [ ] AC-3: chroot to non-directory: ENOTDIR.
- [ ] AC-4: chroot to nonexistent path: ENOENT.
- [ ] AC-5: path = NULL: EFAULT.
- [ ] AC-6: cd("/jail/sub"); chroot("/jail"): cwd still "/jail/sub" relative to new root.
- [ ] AC-7: Inside chroot, open("/.."): resolves to new root, not the old one.
- [ ] AC-8: CLONE_FS sibling: also sees new root.
- [ ] AC-9: Second chroot from within: succeeds (POSIX) unless GRKERNSEC_CHROOT_DOUBLE: EPERM.
- [ ] AC-10: chroot does not change cwd: getcwd returns previous path (may dangle).

## Architecture

```
Chroot::sys_chroot(path: UserPtr<u8>) -> Result<i32, Errno>
```

1. /* Capability */
2. if !ns_capable(current().cred.user_ns, CAP_SYS_CHROOT): return Err(EPERM).
3. /* Lookup */
4. let new_path = user_path_at(AT_FDCWD, path, LOOKUP_FOLLOW | LOOKUP_DIRECTORY)?;
5. /* Inode permission */
6. inode_permission(new_path.dentry.d_inode, MAY_EXEC)?;
7. /* Optional grsec checks */
8. grsec_chroot_double_check()?;     // GRKERNSEC_CHROOT_DOUBLE
9. /* Apply */
10. set_fs_root(current().fs, &new_path);
11. /* Mark task as in-chroot for downstream grsec hooks */
12. current().mark_in_chroot();
13. path_put(&new_path).
14. Ok(0).
```

`set_fs_root(fs, path)`:
1. write_lock(&fs.lock).
2. let old = fs.root.replace(path.clone()).
3. write_unlock(&fs.lock).
4. mntget(path.mnt); dget(path.dentry).
5. old.put().                              // drops mnt + dentry refs

`current().mark_in_chroot()`:
- task.flags |= TASK_IN_CHROOT (Rookery additional flag, used by grsec gates).

`grsec_chroot_double_check()`:
- if config.GRKERNSEC_CHROOT_DOUBLE && task.in_chroot(): Err(EPERM).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cap_sys_chroot_required` | INVARIANT | per-sys_chroot: CAP_SYS_CHROOT precedes any state change. |
| `target_is_directory` | INVARIANT | per-sys_chroot: LOOKUP_DIRECTORY ⟹ final inode is S_IFDIR. |
| `exec_perm_on_target` | INVARIANT | per-sys_chroot: MAY_EXEC checked. |
| `set_fs_root_under_fs_lock` | INVARIANT | per-set_fs_root: fs.lock held during root replace. |
| `old_root_ref_balanced` | INVARIANT | per-set_fs_root: old root dropped exactly once. |
| `new_root_ref_acquired` | INVARIANT | per-set_fs_root: mntget + dget called on new. |
| `grsec_chroot_double` | INVARIANT | per-config.CHROOT_DOUBLE: nested chroot ⟹ EPERM. |

### Layer 2: TLA+

`uapi/chroot.tla`:
- States: task.fs.root, task.in_chroot flag, refcounts.
- Properties:
  - `safety_caps` — per-call: CAP_SYS_CHROOT enforced.
  - `safety_refcount_balance` — per-set_fs_root: old ref put exactly once; new ref get exactly once.
  - `safety_grsec_double` — per-config: nested chroot blocked.
  - `safety_clone_fs_visibility` — per-CLONE_FS group: change visible to all share-members atomically.
  - `liveness_terminates` — per-call: returns success or definite errno.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Chroot::sys_chroot` post: success ⟹ current.fs.root == new_path | `Chroot::sys_chroot` |
| `set_fs_root` post: old refcount decremented, new incremented | `set_fs_root` |
| `mark_in_chroot` post: TASK_IN_CHROOT set | `task.mark_in_chroot` |
| `Chroot::sys_chroot` post: cwd unchanged | `Chroot::sys_chroot` |

### Layer 4: Verus/Creusot functional

Per-chroot semantic equivalence with upstream `sys_chroot` + `set_fs_root`: per-POSIX-1.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

chroot-syscall reinforcement:

- **Per-CAP_SYS_CHROOT in namespace** — defense against per-unprivileged chroot.
- **Per-GRKERNSEC_CHROOT_DOUBLE** — defense against per-nested chroot escape pattern.
- **Per-GRKERNSEC_CHROOT_PIVOT** — chroot'd task forbidden from pivot_root.
- **Per-GRKERNSEC_CHROOT_MOUNT** — chroot'd task forbidden from mount/umount.
- **Per-GRKERNSEC_CHROOT_FCHDIR** — chroot'd task cannot fchdir to outside-root fd.
- **Per-GRKERNSEC_CHROOT_MKNOD** — chroot'd task forbidden from mknod.
- **Per-GRKERNSEC_CHROOT_NICE** — chroot'd task cannot renice / change rt-policy outside chroot.
- **Per-GRKERNSEC_CHROOT_EXECLOG** — log all exec() from chroot.
- **Per-GRKERNSEC_CHROOT_CAPS** — strip capabilities (CAP_SYS_ADMIN, CAP_SYS_MODULE, CAP_NET_ADMIN, etc.) on chroot.
- **Per-GRKERNSEC_CHROOT_SYSCTL** — chroot'd task forbidden writing /proc/sys.
- **Per-GRKERNSEC_CHROOT_UNIX** — chroot'd task cannot connect to abstract unix sockets outside chroot.
- **Per-GRKERNSEC_CHROOT_FINDTASK** — chroot'd task can only see tasks in same chroot via /proc, kill, etc.
- **Per-PAX UDEREF on path copy_from_user** — defense against per-kernel-pointer-fixup race.
- **Per-PAX_RANDKSTACK on syscall entry** — defense against per-ROP-via-chroot.
- **Per-LOOKUP_DIRECTORY enforced** — defense against per-non-dir chroot (regular-file as root).
- **Per-fs.lock write-lock around root replace** — defense against per-CLONE_FS race.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `pivot_root` namespace-level root swap (covered in `pivot_root.md`).
- mount-namespace creation (covered in `unshare.md`).
- per-chroot capability stripping policy implementation (covered in `kernel/grsec-chroot.md`).
- Implementation code.
