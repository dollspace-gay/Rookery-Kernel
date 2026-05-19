# Tier-5: syscall 199 — fremovexattr(2)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - arch/x86/entry/syscalls/syscall_64.tbl (`199  common  fremovexattr  sys_fremovexattr`)
  - fs/xattr.c (`SYSCALL_DEFINE2(fremovexattr, ...)`, `removexattr`, `__vfs_removexattr_locked`)
  - fs/file.c (`fdget`, `fdput`)
  - include/uapi/linux/xattr.h (`XATTR_NAME_MAX`)
  - security/security.c (`security_inode_removexattr`)
-->

## Summary

`fremovexattr(2)` is **x86_64 syscall 199**, the fd-relative extended-attribute remover. It deletes the xattr whose name is `name` from the inode referred to by `fd`. There is no path resolution, no symlink traversal, no per-component permission walk: the open `struct file` pins the dentry, the mount, and the credentials snapshot used to authorize the removal.

This is the call libcap uses to drop `security.capability` from a file just before promoting it to an installer slot, the call selinux-relabel uses to remove a stale `security.selinux` label before applying a new one (when the `setxattr REPLACE` path is unavailable), and the call container teardown code uses to wipe per-tenant labels on shared image layers.

Internally `fremovexattr` lowers to `removexattr(idmap, dentry, name)` with `dentry = f.f_path.dentry` and `idmap = file_mnt_idmap(f)`. The fd refcount holds the mount and dentry pinned, so `mnt_want_write_file` (rather than `mnt_want_write`) is used to take the writer count on the underlying mount.

Critical for: every libc `fremovexattr`, every `setcap -r` on an open fd, every container labeler that uses `O_PATH` + `fremovexattr`, every Rookery xattr-vfs fd-remove test.

## Signature

C (POSIX-ish / man-pages):

```c
int fremovexattr(int fd, const char *name);
```

glibc wrapper: `__fremovexattr` → `INLINE_SYSCALL(fremovexattr, 2, fd, name)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE2(fremovexattr,
                int,                 fd,
                const char __user *, name);
```

Rookery dispatch:

```rust
pub fn sys_fremovexattr(
    fd: i32,
    name: UserPtr<u8>,
) -> SyscallResult<isize>;
```

## Parameters

| name | type                  | constraints                                                   | errno-on-bad |
|------|-----------------------|---------------------------------------------------------------|--------------|
| fd   | `int`                 | Open file descriptor (regular fd, `O_PATH` fd, dirfd, etc.).  | `EBADF`      |
| name | `const char __user *` | NUL-terminated; length `≤ XATTR_NAME_MAX (255)`.              | `EFAULT` / `ERANGE` |

## Return value

- Success: `0`.
- Failure: `< 0` — negated errno.

## Errors

| errno      | condition                                                                              |
|------------|----------------------------------------------------------------------------------------|
| `EBADF`    | `fd` is not an open file descriptor.                                                   |
| `EFAULT`   | `name` crosses the user/kernel boundary illegally.                                     |
| `ERANGE`   | Name longer than `XATTR_NAME_MAX`.                                                     |
| `ENODATA`  | Attribute does not exist (`-ENOATTR` historically; same numeric value).                |
| `EOPNOTSUPP` | Filesystem does not support removing xattrs in this namespace.                       |
| `EACCES`   | Caller lacks permission to remove (e.g. user.* on file without write permission).      |
| `EPERM`    | Namespace requires capability (e.g. `trusted.*` without `CAP_SYS_ADMIN`).              |
| `EROFS`    | Read-only filesystem or read-only mount.                                               |
| `EINVAL`   | Empty name; or name without namespace prefix on filesystems that demand one.           |
| `ENOSPC`   | Removal triggers a journal commit that fills the journal.                              |

## ABI surface

- `XATTR_NAME_MAX = 255`.
- Removal is per-name; no batch interface.
- Removal of a non-existent name is `-ENODATA`, not a success.
- Removal of `security.capability` triggers re-evaluation of file capability state.

## Compatibility contract

- REQ-1: Argument lowering: `%edi=fd`, `%rsi=name`.
- REQ-2: `fdget(fd)` produces a refcounted `struct fd`; `fdput` on every exit edge.
- REQ-3: `strncpy_from_user(kname, name, XATTR_NAME_MAX+1)` — `> XATTR_NAME_MAX ⟹ -ERANGE`; faulting ⟹ `-EFAULT`.
- REQ-4: Empty `kname` ⟹ `-EINVAL`.
- REQ-5: Namespace prefix validated (`user.`, `trusted.`, `security.`, `system.`); unrecognized prefix may be accepted depending on filesystem (e.g. tmpfs accepts `user.*` only).
- REQ-6: Privilege check: `trusted.*` requires `CAP_SYS_ADMIN` in the filesystem's user-ns; `security.*` mediated by LSM.
- REQ-7: `mnt_want_write_file(f)`; `mnt_drop_write_file(f)` on every exit edge.
- REQ-8: `__vfs_removexattr_locked(idmap, dentry, kname, NULL)` — kname checked, then `inode->i_op->removexattr` or handler-walk invoked.
- REQ-9: LSM hook `security_inode_removexattr` veto before removal; success → `security_inode_post_removexattr` notification.
- REQ-10: Removal of `security.capability` triggers `cap_inode_killpriv` semantics consistent with `chown`.
- REQ-11: Inode `i_ctime` updated; `i_mtime` not updated (xattr removal is metadata change, not data change).
- REQ-12: Per-fsnotify: `FS_ATTRIB` event emitted.

## Acceptance Criteria

- [ ] AC-1: `fremovexattr(fd, "user.foo")` removes the xattr; subsequent `fgetxattr` returns `-ENODATA`.
- [ ] AC-2: `fremovexattr(fd, "user.missing")` → `-ENODATA`.
- [ ] AC-3: Without `CAP_SYS_ADMIN`: `fremovexattr(fd, "trusted.x")` → `-EPERM`.
- [ ] AC-4: `O_PATH` fd: succeeds (xattr permission independent of read/write).
- [ ] AC-5: `fremovexattr(-1, "user.foo")` → `-EBADF`.
- [ ] AC-6: Name `> XATTR_NAME_MAX`: `-ERANGE`.
- [ ] AC-7: Empty name: `-EINVAL`.
- [ ] AC-8: Read-only mount: `-EROFS`.
- [ ] AC-9: `security.capability` removed → file capability state re-evaluated; suid-cap stripped.
- [ ] AC-10: fsnotify `FS_ATTRIB` event observed.

## Architecture

```rust
struct FremovexattrArgs {
    fd: i32,
    name: UserPtr<u8>,
}
```

`Xattr::sys_fremovexattr(args) -> isize`:

1. `let mut kname = [0u8; XATTR_NAME_MAX + 1];`
2. `let nlen = strncpy_from_user(&mut kname, args.name, XATTR_NAME_MAX + 1)?;`
3. `if nlen == 0 { return -EINVAL; }`
4. `if nlen > XATTR_NAME_MAX { return -ERANGE; }`
5. `let f = file_table::fdget(args.fd).ok_or(-EBADF)?;`
6. `let rc = Xattr::mnt_want_write_file(&f);`
7. `if rc < 0 { file_table::fdput(f); return rc; }`
8. `let r = Xattr::removexattr_locked(file_mnt_idmap(&f), f.f_path.dentry, &kname[..nlen]);`
9. `Xattr::mnt_drop_write_file(&f); file_table::fdput(f);`
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
| `fd_fdget_fdput_balanced` | INVARIANT | fdget on entry ⟹ fdput on every exit. |
| `mnt_writer_balanced` | INVARIANT | mnt_want_write_file ⟹ mnt_drop_write_file on every exit. |
| `inode_lock_balanced` | INVARIANT | inode_lock ⟹ inode_unlock on every exit. |
| `name_bounded` | INVARIANT | `nlen ≤ XATTR_NAME_MAX`. |
| `empty_name_rejected` | INVARIANT | `nlen == 0` ⟹ `-EINVAL`. |
| `lsm_veto_observed` | INVARIANT | LSM denial ⟹ no removal performed. |
| `enodata_on_missing` | INVARIANT | Missing name ⟹ `-ENODATA`. |

### Layer 2: TLA+

`uapi/syscalls/fremovexattr.tla`:
- Per-call → strncpy_from_user → fdget → mnt_want_write_file → xattr_permission → LSM → inode_lock → vfs_removexattr_locked → fsnotify → mnt_drop_write_file → fdput.
- Properties:
  - `safety_no_remove_without_perm`,
  - `safety_fd_refcount_balanced`,
  - `safety_mnt_writer_balanced`,
  - `safety_inode_lock_balanced`,
  - `safety_lsm_veto_short_circuits`,
  - `liveness_fremovexattr_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post: success ⟹ vfs_removexattr_locked returned 0 | `Xattr::removexattr_locked` |
| Post: LSM denial ⟹ no inode modification | `Xattr::removexattr_locked` |
| Post: fd, writer, inode-lock all balanced | `Xattr::sys_fremovexattr` |
| Post: name length ≤ XATTR_NAME_MAX | `Xattr::sys_fremovexattr` |
| Post: success ⟹ fsnotify FS_ATTRIB emitted | `Xattr::removexattr_locked` |

### Layer 4: Verus/Creusot functional

`fremovexattr(fd, name)` ≡ Linux fremovexattr(2) per `man 2 fremovexattr`, including `security.capability` strip semantics and LSM mediation.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`fremovexattr(2)` reinforcement:

- **Per-`XATTR_NAME_MAX` cap** — defense against unbounded kernel-side name buffer.
- **Per-namespace privilege gate** — defense against unprivileged `trusted.*` tampering.
- **Per-LSM veto** — defense against unprivileged label removal in mandatory-access-control mode.
- **Per-`security.capability` cap-revoke** — defense against suid-cap reuse via xattr churn.
- **Per-fd fdget/fdput balanced** — defense against fd-refcount UAF.
- **Per-mount writer-count balanced** — defense against `mnt_want_write_file` leak.
- **Per-inode lock balanced** — defense against double-lock or unlocked-modify.
- **Per-fsnotify FS_ATTRIB** — defense against silent label change against monitoring.

## Grsecurity/PaX-style Reinforcement

- **PAX_UDEREF** — `name` SMAP-guarded in `strncpy_from_user`.
- **GRKERNSEC_TRUSTED xattr LSM-mediation** — `trusted.*` removal requires CAP_SYS_ADMIN AND LSM approval; defense against unprivileged tamper-and-reset cycles.
- **GRKERNSEC_SECURITY_XATTR** — `security.*` removal mediated by full LSM; SELinux refuses removal of `security.selinux` on inodes whose type does not allow re-labeling.
- **GRKERNSEC_SYSTEM_XATTR** — `system.posix_acl_*` removal requires owner-or-CAP_FOWNER.
- **PAX_REFCOUNT** — fd and mount writer-count refcounts saturating; UAF prevention on long calls.
- **GRKERNSEC_FORKBOMB** — fremovexattr does not bypass per-uid open-fd limits.
- **GRKERNSEC_CHROOT_FHANDLE** — fremovexattr on an fd obtained outside the chroot is denied when GRKERNSEC_CHROOT_FHANDLE is on.
- **GRKERNSEC_HIDESYM** — LSM denial printks redact pointers.
- **PAX_RANDKSTACK** — kstack offset randomized at syscall entry.
- **PAX_MEMORY_SANITIZE** — kname buffer zeroed on stack-exit (no info-leak through stack-residue).
- **GRKERNSEC_AUDIT_XATTR** — fremovexattr logged with caller pid/uid/exe, inode, and name.
- **PAX_USERCOPY** — `strncpy_from_user` length bound-checked.

## Open Questions

(none at this Tier-5 level)
