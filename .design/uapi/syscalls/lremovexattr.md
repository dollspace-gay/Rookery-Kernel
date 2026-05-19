# Tier-5: syscall 198 — lremovexattr(2)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - arch/x86/entry/syscalls/syscall_64.tbl (`198  common  lremovexattr  sys_lremovexattr`)
  - fs/xattr.c (`SYSCALL_DEFINE2(lremovexattr, ...)`, `path_removexattr`, `removexattr`, `__vfs_removexattr_locked`)
  - fs/namei.c (`user_path_at` with no `LOOKUP_FOLLOW`)
  - include/uapi/linux/xattr.h (`XATTR_NAME_MAX`)
  - security/security.c (`security_inode_removexattr`)
-->

## Summary

`lremovexattr(2)` is **x86_64 syscall 198**, the **non-symlink-following** path-relative extended-attribute remover. It is identical to `removexattr(2)` except the terminal symlink, if present, is **not** dereferenced — removal targets the symlink inode itself. Together with `lsetxattr`/`lgetxattr`/`llistxattr` it forms the symlink-respecting xattr quartet.

This call matters specifically for the symlink's own xattrs: `security.*` labels that may have been applied to the link itself (SELinux symlink labeling, for example), and `trusted.*` annotations placed on links by privileged container infrastructure. Removing those without following the link is essential for tools that reset a link's labels while leaving the target unchanged.

Internally `lremovexattr` lowers to `path_removexattr(name, AT_FDCWD, path, 0 /* no LOOKUP_FOLLOW */)` → `removexattr(idmap, dentry, name)` → `__vfs_removexattr_locked(idmap, dentry, name, NULL)`.

Critical for: every `setfattr -h -x`, every container-image symlink-label-reset operation, every backup-restore tool that must clear stale symlink labels, every Rookery xattr-vfs symlink-respecting remove test.

## Signature

C (POSIX-ish / man-pages):

```c
int lremovexattr(const char *path, const char *name);
```

glibc wrapper: `__lremovexattr` → `INLINE_SYSCALL(lremovexattr, 2, path, name)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE2(lremovexattr,
                const char __user *, pathname,
                const char __user *, name);
```

Rookery dispatch:

```rust
pub fn sys_lremovexattr(
    path: UserPtr<u8>,
    name: UserPtr<u8>,
) -> SyscallResult<isize>;
```

## Parameters

| name | type                  | constraints                                                    | errno-on-bad           |
|------|-----------------------|----------------------------------------------------------------|------------------------|
| path | `const char __user *` | NUL-terminated; length `< PATH_MAX`. Terminal symlink kept.    | `EFAULT` / `ENAMETOOLONG` / `ENOENT` |
| name | `const char __user *` | NUL-terminated; length `≤ XATTR_NAME_MAX (255)`.               | `EFAULT` / `ERANGE`    |

## Return value

- Success: `0`.
- Failure: `< 0` — negated errno.

## Errors

| errno          | condition                                                                              |
|----------------|----------------------------------------------------------------------------------------|
| `EFAULT`       | `path` or `name` crosses the user/kernel boundary illegally.                           |
| `ENAMETOOLONG` | `path` length `≥ PATH_MAX`.                                                            |
| `ENOENT`       | `path` does not exist.                                                                 |
| `EACCES`       | Search permission denied along `path`.                                                 |
| `ENOTDIR`      | Non-terminal component is not a directory.                                             |
| `ERANGE`       | `name` longer than `XATTR_NAME_MAX`.                                                   |
| `ENODATA`      | Attribute does not exist on the symlink inode.                                         |
| `EOPNOTSUPP`   | Filesystem does not support removing xattrs in this namespace on symlinks.             |
| `EPERM`        | Namespace requires capability (e.g. `trusted.*` without `CAP_SYS_ADMIN`).              |
| `EROFS`        | Read-only filesystem or read-only mount.                                               |
| `EINVAL`       | Empty name; or unsupported namespace on symlink (e.g. `user.*` on most filesystems).   |

Note: `ELOOP` is not produced because the terminal symlink is not followed. Mid-path symlinks may still produce `ELOOP`.

## ABI surface

- `XATTR_NAME_MAX = 255`.
- Removal is per-name; no batch.
- Removal of a non-existent name → `-ENODATA`.
- On most filesystems, the only namespaces permitted on symlinks are `trusted.*` and `security.*`.

## Compatibility contract

- REQ-1: Argument lowering: `%rdi=path`, `%rsi=name`.
- REQ-2: `user_path_at(AT_FDCWD, path, 0 /* no LOOKUP_FOLLOW */, &p)` — terminal symlink **kept**.
- REQ-3: `strncpy_from_user(kname, name, XATTR_NAME_MAX+1)` — overflow → `-ERANGE`, fault → `-EFAULT`, empty → `-EINVAL`.
- REQ-4: Namespace prefix validated; unsupported-on-symlink ⟹ `-EOPNOTSUPP` or `-EINVAL`.
- REQ-5: Privilege check: `trusted.*` requires `CAP_SYS_ADMIN`; `security.*` mediated by LSM.
- REQ-6: `mnt_want_write(p.mnt)`; `mnt_drop_write(p.mnt)` on every exit edge.
- REQ-7: `__vfs_removexattr_locked(idmap, p.dentry, kname, NULL)` invoked; `idmap = mnt_idmap(p.mnt)`.
- REQ-8: LSM hooks `security_inode_removexattr` and `security_inode_post_removexattr`.
- REQ-9: Inode `i_ctime` of the symlink updated.
- REQ-10: Per-fsnotify: `FS_ATTRIB` event emitted on the symlink dentry.
- REQ-11: `path_put(p)` on every exit edge.

## Acceptance Criteria

- [ ] AC-1: `lremovexattr("link", "trusted.x")` removes xattr from the symlink, leaving target untouched.
- [ ] AC-2: `lremovexattr("link", "trusted.missing")` → `-ENODATA`.
- [ ] AC-3: Without `CAP_SYS_ADMIN`: `lremovexattr("link", "trusted.x")` → `-EPERM`.
- [ ] AC-4: For a regular file (non-symlink), behaves identically to `removexattr`.
- [ ] AC-5: `lremovexattr` with empty name: `-EINVAL`.
- [ ] AC-6: `lremovexattr` with name > 255: `-ERANGE`.
- [ ] AC-7: Read-only mount: `-EROFS`.
- [ ] AC-8: Symlink i_ctime advanced; target inode unchanged.
- [ ] AC-9: fsnotify `FS_ATTRIB` event observed on link dentry.
- [ ] AC-10: `lremovexattr("link", "user.x")` on a filesystem that disallows `user.*` on symlinks → `-EOPNOTSUPP`.

## Architecture

```rust
struct LremovexattrArgs {
    path: UserPtr<u8>,
    name: UserPtr<u8>,
}
```

`Xattr::sys_lremovexattr(args) -> isize`:

1. `let mut kname = [0u8; XATTR_NAME_MAX + 1];`
2. `let nlen = strncpy_from_user(&mut kname, args.name, XATTR_NAME_MAX + 1)?;`
3. `if nlen == 0 { return -EINVAL; }`
4. `if nlen > XATTR_NAME_MAX { return -ERANGE; }`
5. `let p = Path::user_path_at(AT_FDCWD, args.path, 0 /* no FOLLOW */)?;`
6. `let rc = Xattr::mnt_want_write(p.mnt);`
7. `if rc < 0 { path_put(&p); return rc; }`
8. `let r = Xattr::removexattr_locked(mnt_idmap(p.mnt), p.dentry, &kname[..nlen]);`
9. `Xattr::mnt_drop_write(p.mnt); path_put(&p);`
10. `return r;`

`Xattr::removexattr_locked(idmap, dentry, name) -> isize`:

1. `Xattr::xattr_permission(idmap, dentry.d_inode, name, MAY_WRITE)?;`
2. `Security::inode_removexattr(idmap, dentry, name)?;`
3. `inode_lock(dentry.d_inode);`
4. `let r = vfs_removexattr_locked(idmap, dentry, name);`
5. `inode_unlock(dentry.d_inode);`
6. `if r == 0 { fsnotify_xattr(dentry); Security::inode_post_removexattr(idmap, dentry, name); }`
7. `return r;`

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `no_lookup_follow` | INVARIANT | `user_path_at` flags do NOT include `LOOKUP_FOLLOW`. |
| `path_put_balanced` | INVARIANT | path resolved ⟹ path_put on every exit. |
| `mnt_writer_balanced` | INVARIANT | mnt_want_write ⟹ mnt_drop_write on every exit. |
| `inode_lock_balanced` | INVARIANT | inode_lock ⟹ inode_unlock on every exit. |
| `name_bounded` | INVARIANT | `nlen ≤ XATTR_NAME_MAX`. |
| `empty_name_rejected` | INVARIANT | `nlen == 0` ⟹ `-EINVAL`. |
| `lsm_veto_observed` | INVARIANT | LSM denial ⟹ no removal performed. |
| `link_inode_used` | INVARIANT | For symlinks, dentry's d_inode is the link inode. |

### Layer 2: TLA+

`uapi/syscalls/lremovexattr.tla`:
- Per-call → strncpy_from_user → user_path_at(no-follow) → mnt_want_write → xattr_permission → LSM → inode_lock → vfs_removexattr_locked → fsnotify → mnt_drop_write → path_put.
- Properties:
  - `safety_link_inode_targeted` — terminal-symlink inode is the removal target.
  - `safety_no_remove_without_perm`,
  - `safety_mnt_writer_balanced`,
  - `safety_inode_lock_balanced`,
  - `safety_lsm_veto_short_circuits`,
  - `liveness_lremovexattr_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post: dentry resolved from path without LOOKUP_FOLLOW | `Xattr::sys_lremovexattr` |
| Post: success ⟹ vfs_removexattr_locked returned 0 | `Xattr::removexattr_locked` |
| Post: LSM denial ⟹ no inode modification | `Xattr::removexattr_locked` |
| Post: path, writer, inode-lock all balanced | `Xattr::sys_lremovexattr` |
| Post: success ⟹ fsnotify FS_ATTRIB emitted on link dentry | `Xattr::removexattr_locked` |

### Layer 4: Verus/Creusot functional

`lremovexattr(path, name)` ≡ Linux lremovexattr(2) per `man 2 lremovexattr`. Per-symlink-inode semantics confirmed against `setfattr -h -x`.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`lremovexattr(2)` reinforcement:

- **Per-`XATTR_NAME_MAX` cap** — defense against unbounded kernel-side name buffer.
- **Per-namespace privilege gate** — defense against unprivileged `trusted.*` tampering.
- **Per-LSM veto** — defense against unprivileged label removal.
- **Per-no-follow on terminal** — defense against TOCTOU symlink-target swap during removal.
- **Per-mount writer-count balanced** — defense against `mnt_want_write` leak.
- **Per-inode lock balanced** — defense against double-lock or unlocked-modify.
- **Per-path_put balanced** — defense against path UAF.
- **Per-fsnotify FS_ATTRIB** — defense against silent symlink-label change.

## Grsecurity/PaX-style Reinforcement

- **PAX_UDEREF** — `path` and `name` SMAP-guarded in `getname`/`strncpy_from_user`.
- **GRKERNSEC_TRUSTED xattr LSM-mediation** — `trusted.*` removal on symlinks requires CAP_SYS_ADMIN AND LSM approval; defense against unprivileged tamper of privileged link annotations.
- **GRKERNSEC_SECURITY_XATTR** — `security.*` removal mediated by full LSM; SELinux refuses removal of `security.selinux` on links whose type does not allow re-labeling.
- **GRKERNSEC_LINK** — symlink-targeted xattr modification in protected sticky dirs gated against attacker-owned link races at the parent-directory level.
- **PAX_REFCOUNT** — `struct path` and mount writer-count refcounts saturating.
- **GRKERNSEC_FIFO** — sticky-dir FIFO link modification gated identically.
- **GRKERNSEC_CHROOT_FCHDIR** — chroot'd `lremovexattr` cannot traverse pre-chroot paths.
- **GRKERNSEC_HIDESYM** — LSM denial printks redact pointers.
- **PAX_RANDKSTACK** — kstack offset randomized at syscall entry.
- **PAX_MEMORY_SANITIZE** — kname buffer zeroed on stack-exit.
- **GRKERNSEC_AUDIT_XATTR** — lremovexattr logged with caller pid/uid/exe, inode, and name.
- **PAX_USERCOPY** — `strncpy_from_user` length bound-checked.

## Open Questions

(none at this Tier-5 level)
