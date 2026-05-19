---
title: "Tier-5: syscall 188 — setxattr(2)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`setxattr(2)` is **x86_64 syscall 188**, the path-relative extended-attribute setter. It writes (creates or replaces) a single named blob attached to the inode identified by `path` (terminal symlinks **followed**). Extended attributes (xattrs) are filesystem-supported key/value pairs partitioned into four mandatory namespaces:

- `user.*` — unprivileged, fs-policy-controlled (only regular files and directories on most fs).
- `trusted.*` — root-only (`CAP_SYS_ADMIN`), readable and writable only by privileged tasks.
- `security.*` — LSM-managed (SELinux labels, IMA digests, Smack labels, capability sets). LSMs gate every write; some prefixes (`security.capability`) require `CAP_FSETID`/`CAP_SETFCAP`.
- `system.*` — filesystem-managed semantics (POSIX ACLs `system.posix_acl_access`/`_default`, NFSv4 ACLs); writes mediated by the filesystem rather than the generic VFS table.

The `flags` argument is a strict exclusive-choice mask: `XATTR_CREATE` (fail with `EEXIST` if attribute already present) or `XATTR_REPLACE` (fail with `ENODATA`/`ENOATTR` if absent). `flags == 0` means create-or-replace. Internally the call lowers to `path_setxattr(path, name, value, size, flags, LOOKUP_FOLLOW)` → `setxattr(path_idmap, dentry, name, value, size, flags)` → `__vfs_setxattr_locked` under `inode_lock`. LSM hook `security_inode_setxattr` runs before the fs handler; on success, `fsnotify_xattr` is emitted and `i_ctime` is updated.

Critical for: every libc `setxattr`, every container-runtime label assignment, every `setcap` setting `security.capability`, every ACL writer, every IMA appraiser, every Rookery xattr-vfs test.

### Acceptance Criteria

- [ ] AC-1: `setxattr("f", "user.foo", "v", 1, 0)` creates xattr; subsequent `getxattr` returns `"v"`.
- [ ] AC-2: `setxattr(... XATTR_CREATE)` on existing attribute → `-EEXIST`.
- [ ] AC-3: `setxattr(... XATTR_REPLACE)` on missing attribute → `-ENODATA`.
- [ ] AC-4: `setxattr` with both flags set → `-EINVAL`.
- [ ] AC-5: `setxattr("f", "noprefix", ...) → -ENOTSUP`.
- [ ] AC-6: Unprivileged `setxattr("f", "trusted.x", ...) → -EPERM`.
- [ ] AC-7: `setxattr` `size > XATTR_SIZE_MAX → -E2BIG`.
- [ ] AC-8: `setxattr` `strlen(name) > 255 → -ERANGE`.
- [ ] AC-9: Terminal symlink is followed (xattr lands on target, not link).
- [ ] AC-10: RO mount → `-EROFS`.
- [ ] AC-11: On success: `i_ctime` advanced and `fsnotify_xattr` raised.
- [ ] AC-12: LSM denial mapped to caller's errno (`-EACCES`/`-EPERM`).

### Architecture

```
struct SetxattrArgs {
    path: UserPtr<u8>,
    name: UserPtr<u8>,
    value: UserPtr<u8>,
    size: usize,
    flags: i32,
}
```

`Xattr::sys_setxattr(args) -> i32`:

1. /* Validate flags eagerly */ `if args.flags & !(XATTR_CREATE | XATTR_REPLACE) != 0 { return -EINVAL; }`
2. `if args.flags == (XATTR_CREATE | XATTR_REPLACE) { return -EINVAL; }`
3. `let path = Path::user_path_at(AT_FDCWD, args.path, LOOKUP_FOLLOW)?;`
4. `return Xattr::path_setxattr(&path, args.name, args.value, args.size, args.flags);`

`Xattr::path_setxattr(path, name, value, size, flags) -> i32`:

1. `let kname = NameBuf::copy_from_user(name, XATTR_NAME_MAX + 1)?;`
2. `if kname.len() == 0 || kname.len() > XATTR_NAME_MAX { return -ERANGE; }`
3. `if size > XATTR_SIZE_MAX { return -E2BIG; }`
4. `let kvalue = if size > 0 { UserBuf::memdup_from_user(value, size)? } else { None };`
5. `mnt_want_write(path.mnt)?;`
6. `loop { let r = Xattr::setxattr(path.idmap(), path.dentry, &kname, kvalue.as_deref(), size, flags, &mut delegated_inode); if r == -EWOULDBLOCK { break_deleg_wait(&mut delegated_inode)?; continue; } break r; }`
7. `mnt_drop_write(path.mnt);`
8. `path_put(path); return r;`

`Xattr::setxattr(idmap, dentry, name, value, size, flags, delegated_inode) -> i32`:

1. `inode_lock(dentry.d_inode);`
2. `let r = Xattr::vfs_setxattr_locked(idmap, dentry, name, value, size, flags, delegated_inode);`
3. `inode_unlock(dentry.d_inode); return r;`

`Xattr::vfs_setxattr_locked(idmap, dentry, name, value, size, flags, delegated_inode) -> i32`:

1. `Security::inode_setxattr(idmap, dentry, name, value, size, flags)?;`
2. `if inode.i_flags & (S_IMMUTABLE | S_APPEND) { return -EPERM; }`
3. /* Capability gates per-namespace */
4. `match XattrNs::classify(name) { Trusted => require_cap(CAP_SYS_ADMIN)?, Security => Security::inode_setsecurity_or_setxattr(...)?, System => fs_acl_set(...)?, User => () }`
5. `let handler = xattr_resolve_name(inode.i_sb.s_xattr, name)?;`
6. `let r = handler.set(handler, idmap, dentry, inode, name, value, size, flags);`
7. /* Post-action */
8. `if r == 0 { fsnotify_xattr(dentry); inode_inc_iversion(inode); inode.i_ctime = current_time(inode); mark_inode_dirty(inode); }`
9. `return r;`

### Out of Scope

- `lsetxattr(2)` — separate Tier-5 (`lsetxattr.md`).
- `fsetxattr(2)` — separate Tier-5 (`fsetxattr.md`).
- `setxattrat(2)` — separate Tier-5 (`setxattrat.md`).
- POSIX ACL semantics (`system.posix_acl_*`) — covered in `fs/posix_acl.md` Tier-3 if expanded.
- LSM handler internals — covered in `security/00-overview.md`.
- Filesystem-specific xattr storage (ext4/xfs/btrfs/tmpfs) — covered in per-fs Tier-3.
- Implementation code.

### signature

C (POSIX-ish / man-pages):

```c
int setxattr(const char *path, const char *name,
             const void *value, size_t size, int flags);
```

glibc wrapper: `__setxattr` → `INLINE_SYSCALL(setxattr, 5, path, name, value, size, flags)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE5(setxattr,
                const char __user *, pathname,
                const char __user *, name,
                const void __user *, value,
                size_t,              size,
                int,                 flags);
```

Rookery dispatch:

```rust
pub fn sys_setxattr(
    path: UserPtr<u8>,
    name: UserPtr<u8>,
    value: UserPtr<u8>,
    size: usize,
    flags: i32,
) -> SyscallResult<i32>;
```

### parameters

| name    | type                  | constraints                                                                                       | errno-on-bad           |
|---------|-----------------------|---------------------------------------------------------------------------------------------------|------------------------|
| path    | `const char __user *` | NUL-terminated; length `< PATH_MAX`. Terminal symlink followed.                                   | `EFAULT` / `ENAMETOOLONG` / `ENOENT` |
| name    | `const char __user *` | NUL-terminated; `1 ≤ strlen(name) ≤ XATTR_NAME_MAX (255)`. Must begin with a known namespace prefix. | `EFAULT` / `ERANGE` / `ENOTSUP` |
| value   | `const void __user *` | Readable for `size` bytes; may be `NULL` only if `size == 0`.                                     | `EFAULT`               |
| size    | `size_t`              | `0 ≤ size ≤ XATTR_SIZE_MAX (65536)`. Filesystem may impose a smaller cap.                          | `E2BIG`                |
| flags   | `int`                 | `0`, `XATTR_CREATE (0x1)`, or `XATTR_REPLACE (0x2)` — never both.                                  | `EINVAL`               |

### return value

- Success: `0`.
- Failure: `< 0` — negated errno.

### errors

| errno          | condition                                                                                       |
|----------------|-------------------------------------------------------------------------------------------------|
| `EFAULT`       | Any of `path`, `name`, or `value` (for non-zero `size`) crosses the user/kernel boundary illegally. |
| `ENAMETOOLONG` | `path` length `≥ PATH_MAX`, or `name` length `> XATTR_NAME_MAX`.                                 |
| `ENOENT`       | `path` does not exist.                                                                          |
| `EACCES`       | Search permission denied along `path`, or write permission on terminal inode denied.            |
| `ELOOP`        | Symlink chain exceeded `MAXSYMLINKS`.                                                           |
| `ENOTDIR`      | Non-terminal component of `path` is not a directory.                                            |
| `EINVAL`       | `flags` has unknown bits, or both `XATTR_CREATE`+`XATTR_REPLACE`, or `name` has no namespace prefix, or name is "" (zero-length). |
| `EEXIST`       | `XATTR_CREATE` and attribute already exists.                                                    |
| `ENODATA`      | `XATTR_REPLACE` and attribute does not exist (POSIX alias: `ENOATTR`).                          |
| `ENOTSUP`      | Filesystem does not support xattrs, or namespace prefix is unrecognized for this fs.            |
| `EPERM`        | Caller lacks `CAP_SYS_ADMIN` for `trusted.*`; or LSM denies `security.*`; or file is immutable/append-only. |
| `E2BIG`        | `size` exceeds the filesystem's or kernel's per-attribute limit (`XATTR_SIZE_MAX`).             |
| `EROFS`        | Filesystem is read-only.                                                                        |
| `EDQUOT`       | Disk quota exceeded (xattr storage counts against quota on some fs).                            |
| `ENOSPC`       | No space left for xattr block.                                                                  |
| `EOPNOTSUPP`   | Synonym for `ENOTSUP` on some architectures.                                                    |

### abi surface (constants + flags)

From `include/uapi/linux/xattr.h`:

- `XATTR_CREATE = 0x1` — fail if attribute exists.
- `XATTR_REPLACE = 0x2` — fail if attribute does not exist.
- `XATTR_NAME_MAX = 255` — maximum name length (excluding NUL).
- `XATTR_SIZE_MAX = 65536` — maximum value length (per filesystem).
- `XATTR_LIST_MAX = 65536` — applies to listxattr; documented here for completeness.
- Namespace prefixes (string literals, case-sensitive):
  - `XATTR_USER_PREFIX = "user."` (len 5).
  - `XATTR_TRUSTED_PREFIX = "trusted."` (len 8).
  - `XATTR_SECURITY_PREFIX = "security."` (len 9).
  - `XATTR_SYSTEM_PREFIX = "system."` (len 7).

A name lacking any of the four prefixes is rejected with `ENOTSUP` (or `EOPNOTSUPP`) by `xattr_full_name` / namespace-handler lookup.

### compatibility contract

- REQ-1: Argument lowering: `%rdi=path`, `%rsi=name`, `%rdx=value`, `%r10=size`, `%r8=flags`.
- REQ-2: `flags & ~(XATTR_CREATE | XATTR_REPLACE) ⟹ -EINVAL`. `flags == (XATTR_CREATE | XATTR_REPLACE) ⟹ -EINVAL`.
- REQ-3: `user_path_at(AT_FDCWD, path, LOOKUP_FOLLOW, &p)` — terminal symlinks are followed.
- REQ-4: `mnt_want_write(p.mnt)` taken; `mnt_drop_write` on every exit path.
- REQ-5: `strncpy_from_user(kname, name, XATTR_NAME_MAX + 1)`; lengths `> XATTR_NAME_MAX` ⟹ `-ERANGE`; length `0` ⟹ `-ERANGE`.
- REQ-6: `size > XATTR_SIZE_MAX ⟹ -E2BIG`. `value` copy is bounded: `memdup_user(value, size)` allocates kernel buffer.
- REQ-7: `__vfs_setxattr_locked(idmap, dentry, name, kvalue, size, flags, &delegated_inode)` acquires `inode_lock`.
- REQ-8: LSM hook `security_inode_setxattr(idmap, dentry, name, kvalue, size, flags)` runs **before** the fs handler.
- REQ-9: Namespace dispatch via `xattr_full_name`/handler table:
  - `user.*` → `xattr_handler_user.set`.
  - `trusted.*` → `xattr_handler_trusted.set` — `capable(CAP_SYS_ADMIN)` gate.
  - `security.*` → `xattr_handler_security.set` — `security_inode_setsecurity` for `security.selinux` etc.; `security.capability` writes require `CAP_SETFCAP`.
  - `system.*` → `xattr_handler_posix_acl_*` (for ACLs); the fs may also expose other `system.*` handlers.
- REQ-10: On success: `fsnotify_xattr(dentry)`; `inode_inc_iversion`; `mark_inode_dirty(inode)`; `i_ctime` updated to `current_time(inode)`.
- REQ-11: Immutable (`S_IMMUTABLE`) or append-only (`S_APPEND`) inodes reject all xattr writes with `-EPERM` (except where the LSM explicitly permits).
- REQ-12: `delegated_inode != NULL` ⟹ NFSv4 delegation must be broken via `break_deleg_wait` and the loop retried.
- REQ-13: Read-only filesystem ⟹ `mnt_want_write` returns `-EROFS`.
- REQ-14: Caller's mnt_userns/idmap is passed through (`idmap = mnt_idmap(p.mnt)`).
- REQ-15: Audit record emitted for `security.*` writes when audit is enabled.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_strict` | INVARIANT | Any bit outside `{XATTR_CREATE, XATTR_REPLACE}` ⟹ `-EINVAL`; mutually exclusive. |
| `name_length_bounded` | INVARIANT | `1 ≤ len(name) ≤ XATTR_NAME_MAX`. |
| `value_size_bounded` | INVARIANT | `0 ≤ size ≤ XATTR_SIZE_MAX`. |
| `prefix_required` | INVARIANT | `name` lacking known prefix ⟹ `-ENOTSUP`. |
| `trusted_cap_gate` | INVARIANT | `trusted.*` write ⟹ `capable(CAP_SYS_ADMIN)`. |
| `mnt_write_paired` | INVARIANT | Every `mnt_want_write` paired with `mnt_drop_write`. |
| `inode_lock_held` | INVARIANT | `__vfs_setxattr_locked` requires `inode_lock`. |

### Layer 2: TLA+

`uapi/syscalls/setxattr.tla`:
- Per-call → validate → lookup → mnt_want_write → LSM → handler → fsnotify → mnt_drop_write → path_put.
- Properties:
  - `safety_flags_validated`,
  - `safety_namespace_capability_gated`,
  - `safety_size_within_XATTR_SIZE_MAX`,
  - `liveness_setxattr_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post: `XATTR_CREATE` success ⟹ attribute newly created | `Xattr::vfs_setxattr_locked` |
| Post: `XATTR_REPLACE` success ⟹ attribute existed and is overwritten | `Xattr::vfs_setxattr_locked` |
| Post: success ⟹ `i_ctime` advanced and `fsnotify_xattr` emitted | `Xattr::vfs_setxattr_locked` |
| Post: any error ⟹ on-disk state unchanged | `Xattr::vfs_setxattr_locked` |

### Layer 4: Verus/Creusot functional

`setxattr(path, name, value, size, flags)` ≡ Linux setxattr(2) per `man 2 setxattr` and `Documentation/filesystems/xattr.rst`, with namespace partitioning identical to upstream.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`setxattr(2)` reinforcement:

- **Per-`flags` strict mask** — defense against silent forward-compat acceptance of unknown bits.
- **Per-`XATTR_NAME_MAX` cap** — defense against slab-allocator exhaustion via long names.
- **Per-`XATTR_SIZE_MAX` cap** — defense against unbounded kvalue allocation.
- **Per-namespace handler dispatch** — defense against caller-controlled handler selection.
- **Per-LSM hook before fs handler** — defense against bypass of policy via direct fs ioctl-like paths.
- **Per-`mnt_want_write` discipline** — defense against RO-bypass and unmount-races.
- **Per-`delegated_inode` break_deleg** — defense against NFSv4 inconsistent state.
- **Per-immutable/append-only `-EPERM`** — defense against ime/append rule bypass via xattr metadata.
- **Per-`inode_lock`** — defense against torn writes interleaving with stat/getxattr.
- **Per-`fsnotify_xattr` post-success only** — defense against false notification on failure.

### grsecurity/pax-style reinforcement

- **PAX_UDEREF** — `path`, `name`, and `value` SMAP-guarded in `getname`/`memdup_user`; copy bounded by validated sizes.
- **GRKERNSEC_TRUSTED** — only root sees and writes `trusted.*`; even with `CAP_SYS_ADMIN`, namespace-confined containers may be denied.
- **GRKERNSEC_SECURITY_XATTR** — `security.*` writes LSM-gated; `security.capability` requires `CAP_SETFCAP` even with namespaces.
- **GRKERNSEC_SYSTEM_XATTR** — `system.*` writes mediated by filesystem (POSIX ACL semantics), never bypassed.
- **GRKERNSEC_PROC** — `/proc/<pid>/attr` and xattr listings restricted to owner/root.
- **PAX_REFCOUNT** — `struct path` and `struct dentry` refcounts saturating, preventing UAF on long xattr walks.
- **GRKERNSEC_LINK** — symlink-traversal during `setxattr` path resolution gated in protected directories.
- **GRKERNSEC_FIFO** — FIFO with sticky-dir + xattr write gated identically to data write.
- **GRKERNSEC_CHROOT_FCHDIR** — chroot'd `setxattr` cannot traverse pre-chroot paths.
- **GRKERNSEC_HIDESYM** — LSM denial printks redact kernel pointers.
- **PAX_RANDKSTACK** — kstack offset randomized at syscall entry.
- **PAX_MEMORY_SANITIZE** — kvalue buffer zeroed on free, preventing leak of prior xattr bytes through slab reuse.
- **GRKERNSEC_AUDIT_XATTR** — every `security.capability`, `trusted.*`, and `security.*` write logged with caller pid/uid/exe.

