# Tier-5: syscall 197 — removexattr(2)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - arch/x86/entry/syscalls/syscall_64.tbl (`197  common  removexattr  sys_removexattr`)
  - fs/xattr.c (`SYSCALL_DEFINE2(removexattr, ...)`, `path_removexattr`, `removexattr`, `__vfs_removexattr_locked`, `__vfs_removexattr_noperm`)
  - fs/namei.c (`user_path_at`, `LOOKUP_FOLLOW`)
  - include/uapi/linux/xattr.h
  - security/security.c (`security_inode_removexattr`)
-->

## Summary

`removexattr(2)` is **x86_64 syscall 197**, the path-relative extended-attribute remover. It deletes a single named blob from the inode identified by `path` (terminal symlinks **followed**). The deletion is final on success: subsequent `getxattr` returns `-ENODATA`, the attribute disappears from `listxattr`, and `i_ctime` advances.

Three subtleties:

- **Namespace gating identical to `setxattr`.** Removing `trusted.*` requires `CAP_SYS_ADMIN`. Removing `security.capability` requires the same privilege as setting it; the LSM may further gate `security.*` deletions (some LSMs reject deletion of mandatory labels). `system.posix_acl_access` deletion clears the ACL and reverts the inode to mode-based permissions; `system.posix_acl_default` deletion clears the directory's default ACL.

- **`ENODATA` for missing.** Unlike `setxattr(... XATTR_REPLACE)`, which is a flag on a write, `removexattr` is unconditional. If the attribute does not exist it returns `-ENODATA` (POSIX alias `ENOATTR`).

- **NFSv4 delegation break.** Like `setxattr`, the call may return `-EWOULDBLOCK` from `__vfs_removexattr_locked` when an NFSv4 delegation must be broken; the retry loop calls `break_deleg_wait` and continues.

Internally `removexattr` lowers to `path_removexattr(path, name, LOOKUP_FOLLOW)` → `removexattr(idmap, dentry, name)` → `__vfs_removexattr_locked` under `inode_lock`. LSM hook `security_inode_removexattr` runs **before** the fs handler; on success, `fsnotify_xattr` is emitted and `i_ctime` advances.

Critical for: every libc `removexattr`, every `setcap -r` clearing `security.capability`, every container relabeler unsetting old labels, every ACL clearer, every Rookery xattr-vfs remove test.

## Signature

C (POSIX-ish / man-pages):

```c
int removexattr(const char *path, const char *name);
```

glibc wrapper: `__removexattr` → `INLINE_SYSCALL(removexattr, 2, path, name)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE2(removexattr,
                const char __user *, pathname,
                const char __user *, name);
```

Rookery dispatch:

```rust
pub fn sys_removexattr(
    path: UserPtr<u8>,
    name: UserPtr<u8>,
) -> SyscallResult<i32>;
```

## Parameters

| name | type                  | constraints                                                                                       | errno-on-bad           |
|------|-----------------------|---------------------------------------------------------------------------------------------------|------------------------|
| path | `const char __user *` | NUL-terminated; length `< PATH_MAX`. Terminal symlink followed.                                   | `EFAULT` / `ENAMETOOLONG` / `ENOENT` |
| name | `const char __user *` | NUL-terminated; `1 ≤ strlen(name) ≤ XATTR_NAME_MAX (255)`. Known namespace prefix required.       | `EFAULT` / `ERANGE` / `ENOTSUP` |

## Return value

- Success: `0`.
- Failure: `< 0` — negated errno.

## Errors

| errno          | condition                                                                                       |
|----------------|-------------------------------------------------------------------------------------------------|
| `EFAULT`       | `path` or `name` crosses the user/kernel boundary illegally.                                    |
| `ENAMETOOLONG` | `path` `≥ PATH_MAX`, or `name` `> XATTR_NAME_MAX`.                                              |
| `ENOENT`       | `path` does not exist.                                                                          |
| `EACCES`       | Search permission denied along `path`, or write permission on terminal inode denied.            |
| `ELOOP`        | Symlink chain exceeded `MAXSYMLINKS`.                                                           |
| `ENOTDIR`      | Non-terminal component is not a directory.                                                      |
| `ENODATA`      | Attribute does not exist (POSIX alias: `ENOATTR`).                                              |
| `ENOTSUP`      | Filesystem does not support xattrs, or namespace prefix unrecognized.                           |
| `EPERM`        | Caller lacks `CAP_SYS_ADMIN` for `trusted.*`; or LSM denies `security.*`; or file is immutable/append-only. |
| `EROFS`        | Filesystem is read-only.                                                                        |
| `EINVAL`       | Name is zero-length or lacks namespace prefix.                                                  |

## ABI surface

From `include/uapi/linux/xattr.h`:

- `XATTR_NAME_MAX = 255` — maximum name length.
- Namespace prefixes (case-sensitive): `user.`, `trusted.`, `security.`, `system.`.

No flags argument. Removal is unconditional once the name resolves to an existing attribute.

## Compatibility contract

- REQ-1: Argument lowering: `%rdi=path`, `%rsi=name`.
- REQ-2: `user_path_at(AT_FDCWD, path, LOOKUP_FOLLOW, &p)` — terminal symlinks followed.
- REQ-3: `mnt_want_write(p.mnt)` taken; `mnt_drop_write` on every exit path.
- REQ-4: `strncpy_from_user(kname, name, XATTR_NAME_MAX + 1)`; lengths `> XATTR_NAME_MAX` ⟹ `-ERANGE`; length `0` ⟹ `-ERANGE`.
- REQ-5: `__vfs_removexattr_locked(idmap, dentry, name, &delegated_inode)` acquires `inode_lock`.
- REQ-6: LSM hook `security_inode_removexattr(idmap, dentry, name)` runs **before** the fs handler.
- REQ-7: Namespace dispatch via `xattr_resolve_name`:
  - `user.*` → `xattr_handler_user.set(value=NULL, size=0)`.
  - `trusted.*` → `xattr_handler_trusted.set(NULL, 0)` — `capable(CAP_SYS_ADMIN)` gate.
  - `security.*` → LSM mediation; `security.capability` requires `CAP_SETFCAP`.
  - `system.*` → POSIX ACL clear path.
- REQ-8: On success: `fsnotify_xattr(dentry)`; `inode_inc_iversion(inode)`; `mark_inode_dirty(inode)`; `i_ctime = current_time(inode)`.
- REQ-9: Immutable (`S_IMMUTABLE`) / append-only (`S_APPEND`) inodes reject all xattr removals with `-EPERM`.
- REQ-10: `delegated_inode != NULL` ⟹ NFSv4 delegation must be broken via `break_deleg_wait`; loop retries.
- REQ-11: Read-only filesystem ⟹ `mnt_want_write` returns `-EROFS`.
- REQ-12: Caller's idmap is passed through.
- REQ-13: Audit record emitted for `security.*` removals.
- REQ-14: `path_put(p)` on every exit path.

## Acceptance Criteria

- [ ] AC-1: `removexattr("f", "user.foo")` removes the attribute; subsequent `getxattr` → `-ENODATA`.
- [ ] AC-2: `removexattr` on missing attribute → `-ENODATA`.
- [ ] AC-3: `removexattr("f", "noprefix")` → `-ENOTSUP`.
- [ ] AC-4: Unprivileged `removexattr("f", "trusted.x")` → `-EPERM`.
- [ ] AC-5: `removexattr` on RO mount → `-EROFS`.
- [ ] AC-6: `removexattr` on immutable inode → `-EPERM`.
- [ ] AC-7: Terminal symlink is followed (removal targets link's target).
- [ ] AC-8: On success: `i_ctime` advanced and `fsnotify_xattr` emitted.
- [ ] AC-9: LSM denial mapped to caller's errno (`-EACCES`/`-EPERM`).
- [ ] AC-10: `removexattr("dir", "system.posix_acl_default")` clears default ACL; subsequent `mkdir` uses mode-only.

## Architecture

```
struct RemovexattrArgs {
    path: UserPtr<u8>,
    name: UserPtr<u8>,
}
```

`Xattr::sys_removexattr(args) -> i32`:

1. `let path = Path::user_path_at(AT_FDCWD, args.path, LOOKUP_FOLLOW)?;`
2. `return Xattr::path_removexattr(&path, args.name);`

`Xattr::path_removexattr(path, name) -> i32`:

1. `let kname = NameBuf::copy_from_user(name, XATTR_NAME_MAX + 1)?;`
2. `if kname.len() == 0 || kname.len() > XATTR_NAME_MAX { return -ERANGE; }`
3. `mnt_want_write(path.mnt)?;`
4. `let mut delegated_inode: *mut Inode = ptr::null_mut();`
5. `loop { let r = Xattr::removexattr(path.idmap(), path.dentry, &kname, &mut delegated_inode); if r == -EWOULDBLOCK { break_deleg_wait(&mut delegated_inode)?; continue; } break r; }`
6. `mnt_drop_write(path.mnt);`
7. `path_put(path); return r;`

`Xattr::removexattr(idmap, dentry, name, delegated_inode) -> i32`:

1. `inode_lock(dentry.d_inode);`
2. `let r = Xattr::vfs_removexattr_locked(idmap, dentry, name, delegated_inode);`
3. `inode_unlock(dentry.d_inode); return r;`

`Xattr::vfs_removexattr_locked(idmap, dentry, name, delegated_inode) -> i32`:

1. `Security::inode_removexattr(idmap, dentry, name)?;`
2. `if inode.i_flags & (S_IMMUTABLE | S_APPEND) { return -EPERM; }`
3. /* Capability gates per-namespace */
4. `match XattrNs::classify(name) { Trusted => require_cap(CAP_SYS_ADMIN)?, Security => Security::inode_removesecurity(...)?, _ => () }`
5. `let handler = xattr_resolve_name(inode.i_sb.s_xattr, name)?;`
6. `let r = handler.set(handler, idmap, dentry, inode, name, ptr::null(), 0, XATTR_REPLACE);`
7. `if r == 0 { fsnotify_xattr(dentry); inode_inc_iversion(inode); inode.i_ctime = current_time(inode); mark_inode_dirty(inode); }`
8. `return r;`

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `name_length_bounded` | INVARIANT | `1 ≤ len(name) ≤ XATTR_NAME_MAX`. |
| `prefix_required` | INVARIANT | `name` lacking known prefix ⟹ `-ENOTSUP`. |
| `trusted_cap_gate` | INVARIANT | `trusted.*` removal ⟹ `capable(CAP_SYS_ADMIN)`. |
| `immutable_rejected` | INVARIANT | `S_IMMUTABLE`/`S_APPEND` ⟹ `-EPERM`. |
| `mnt_write_paired` | INVARIANT | Every `mnt_want_write` paired with `mnt_drop_write`. |
| `inode_lock_held` | INVARIANT | `__vfs_removexattr_locked` requires `inode_lock`. |
| `missing_returns_enodata` | INVARIANT | Removing nonexistent xattr ⟹ `-ENODATA`. |

### Layer 2: TLA+

`uapi/syscalls/removexattr.tla`:
- Per-call → validate → lookup → mnt_want_write → LSM → handler (set with NULL value) → fsnotify → mnt_drop_write → path_put.
- Properties:
  - `safety_namespace_capability_gated`,
  - `safety_missing_returns_enodata`,
  - `safety_immutable_rejected`,
  - `liveness_removexattr_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post: success ⟹ attribute removed; subsequent getxattr returns `-ENODATA` | `Xattr::vfs_removexattr_locked` |
| Post: success ⟹ `i_ctime` advanced and `fsnotify_xattr` emitted | `Xattr::vfs_removexattr_locked` |
| Post: any error ⟹ on-disk state unchanged | `Xattr::vfs_removexattr_locked` |
| Post: `trusted.*` removal without `CAP_SYS_ADMIN` ⟹ `-EPERM` | `Xattr::vfs_removexattr_locked` |

### Layer 4: Verus/Creusot functional

`removexattr(path, name)` ≡ Linux removexattr(2) per `man 2 removexattr` and `Documentation/filesystems/xattr.rst`, with namespace partitioning identical to upstream.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`removexattr(2)` reinforcement:

- **Per-`XATTR_NAME_MAX` cap** — defense against slab exhaustion via long names.
- **Per-namespace handler dispatch** — defense against caller-controlled handler selection.
- **Per-LSM hook before fs handler** — defense against bypass of policy on `security.*` deletion.
- **Per-`mnt_want_write` discipline** — defense against RO-bypass and unmount-races.
- **Per-`delegated_inode` break_deleg** — defense against NFSv4 inconsistent state.
- **Per-immutable/append-only `-EPERM`** — defense against rule bypass via xattr deletion.
- **Per-`inode_lock`** — defense against torn deletions interleaving with stat/getxattr/listxattr.
- **Per-`fsnotify_xattr` post-success only** — defense against false notification on failure.
- **Per-`ENODATA` on missing** — defense against silent success masking missing label.

## Grsecurity/PaX-style Reinforcement

- **PAX_UDEREF** — `path` and `name` SMAP-guarded; `getname`/`strncpy_from_user` bound-checked.
- **GRKERNSEC_TRUSTED** — only root removes `trusted.*`; namespace-confined containers may still be denied even with `CAP_SYS_ADMIN`.
- **GRKERNSEC_SECURITY_XATTR** — `security.*` removals LSM-gated; `security.capability` removal requires `CAP_SETFCAP` even with namespaces; mandatory labels may be unremovable.
- **GRKERNSEC_SYSTEM_XATTR** — `system.*` removals (POSIX ACL clear) mediated by filesystem, never bypassed.
- **GRKERNSEC_PROC** — pid-namespace and proc-restrict apply.
- **PAX_REFCOUNT** — `struct path` and dentry refcounts saturating; UAF prevention.
- **GRKERNSEC_LINK** — symlink-followed removexattr in protected directories gated against attacker-owned link races (TOCTOU on path resolution).
- **GRKERNSEC_FIFO** — FIFO with sticky-dir + xattr removal gated identically to data write.
- **GRKERNSEC_CHROOT_FCHDIR** — chroot'd `removexattr` cannot traverse pre-chroot paths.
- **GRKERNSEC_HIDESYM** — LSM denial printks redact pointers.
- **PAX_RANDKSTACK** — kstack offset randomized at syscall entry.
- **PAX_MEMORY_SANITIZE** — kernel name buffer zeroed on free; prevents info-leak through slab reuse.
- **GRKERNSEC_AUDIT_XATTR** — every `security.capability`, `trusted.*`, and `security.*` removal logged with caller pid/uid/exe and attribute name.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `lremovexattr(2)` — separate Tier-5 (`lremovexattr.md`) if expanded.
- `fremovexattr(2)` — separate Tier-5 (`fremovexattr.md`) if expanded.
- `removexattrat(2)` — separate Tier-5 (`removexattrat.md`) if expanded.
- POSIX ACL clear semantics — covered in `fs/posix_acl.md` Tier-3 if expanded.
- LSM handler internals — covered in `security/00-overview.md`.
- Filesystem-specific xattr deletion mechanics — covered in per-fs Tier-3.
- Implementation code.
