---
title: "Tier-5: syscall 466 — removexattrat(2)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`removexattrat(2)` is **x86_64 syscall 466**, the **directory-fd-relative, flag-bearing extended-attribute remover**, introduced in Linux 6.13 alongside the rest of the `*xattrat` family. It generalizes `removexattr(2)`, `lremovexattr(2)`, and `fremovexattr(2)` into a single uniform entry point patterned after `openat2(2)`:

- Path resolution is anchored at `dfd` (a directory fd) plus `path`.
- Symlink behavior is selected by **`at_flags`** rather than by syscall identity.
- The single `name` argument identifies the attribute (with mandatory namespace prefix).

The kernel locates the named xattr via the inode's xattr-handler table and deletes it from on-disk storage. The operation is **idempotent at the user-API surface only if the attribute exists** — removing a non-existent attribute returns `-ENODATA` (POSIX `-ENOATTR`). The same capability and LSM gates as `setxattr(2)` apply: `trusted.*` requires `CAP_SYS_ADMIN`; `security.*` is LSM-mediated; `system.*` is filesystem-mediated; `user.*` follows standard inode-write permissions.

`at_flags` accepts:

- `AT_SYMLINK_NOFOLLOW` (0x100) — terminal symlink is not followed (equivalent to `lremovexattr(2)`).
- `AT_EMPTY_PATH` (0x1000) — `path == ""` operates on `dfd` itself (typically an `O_PATH` fd, equivalent to `fremovexattr(2)`).
- Other bits ⟹ `-EINVAL`.

`dfd == AT_FDCWD` with non-empty `path` mirrors `removexattr(2)`; same `dfd` with `AT_SYMLINK_NOFOLLOW` mirrors `lremovexattr(2)`; any `dfd` with `AT_EMPTY_PATH` and `path == ""` mirrors `fremovexattr(2)`.

Critical for: every modern container runtime stripping labels, every `setfattr -x` user, every IMA/Integrity-Measurement teardown after kernel-policy change, every Rookery xattr-vfs `*at`-discipline removal test.

### Acceptance Criteria

- [ ] AC-1: `removexattrat(AT_FDCWD, "f", 0, "user.foo")` after prior set ⟹ `0`; subsequent `getxattr` ⟹ `-ENODATA`.
- [ ] AC-2: `removexattrat(..., 0, "user.foo")` on non-existent attribute ⟹ `-ENODATA`.
- [ ] AC-3: `removexattrat(AT_FDCWD, "f", 0, "noprefix")` ⟹ `-ENOTSUP` (or `-EINVAL` for empty/no-prefix).
- [ ] AC-4: Unprivileged `removexattrat(..., "trusted.x")` ⟹ `-EPERM`.
- [ ] AC-5: `at_flags = 0x200` (unknown bit) ⟹ `-EINVAL`.
- [ ] AC-6: `removexattrat(dfd, "", AT_EMPTY_PATH, name)` works as `fremovexattr`.
- [ ] AC-7: `removexattrat(AT_FDCWD, "", 0, name)` ⟹ `-ENOENT`.
- [ ] AC-8: `removexattrat(..., AT_SYMLINK_NOFOLLOW, ...)` on symlink ⟹ removes from symlink (not target).
- [ ] AC-9: RO mount ⟹ `-EROFS`.
- [ ] AC-10: On success: `i_ctime` advanced; `fsnotify_xattr` raised.
- [ ] AC-11: Immutable inode ⟹ `-EPERM`.
- [ ] AC-12: LSM denial ⟹ caller errno matches LSM verdict (`-EACCES`/`-EPERM`).
- [ ] AC-13: `name` length > `XATTR_NAME_MAX` ⟹ `-ERANGE`.

### Architecture

```
struct RemovexattratArgs {
    dfd: i32,
    path: UserPtr<u8>,
    at_flags: u32,
    name: UserPtr<u8>,
}
```

`Xattr::sys_removexattrat(args) -> i32`:

1. `if args.at_flags & !(AT_SYMLINK_NOFOLLOW | AT_EMPTY_PATH) != 0 { return -EINVAL; }`
2. `let kname = NameBuf::copy_from_user(args.name, XATTR_NAME_MAX + 1)?;`
3. `if kname.len() == 0 || kname.len() > XATTR_NAME_MAX { return -ERANGE; }`
4. `let lookup_flags = Xattr::lookup_flags_from_at(args.at_flags);`
5. `let path = Path::user_path_at(args.dfd, args.path, lookup_flags)?;`
6. `let res = Xattr::do_removexattr(&path, &kname);`
7. `path_put(path);`
8. `return res;`

`Xattr::do_removexattr(path, name) -> i32`:

1. `mnt_want_write(path.mnt)?;`
2. `let mut delegated: Option<*Inode> = None;`
3. `let res = loop {`
4. `    let r = Xattr::vfs_removexattr(path.idmap(), path.dentry, name, &mut delegated);`
5. `    if r == -EWOULDBLOCK { if let Err(e) = break_deleg_wait(&mut delegated) { break e; } continue; }`
6. `    break r;`
7. `};`
8. `mnt_drop_write(path.mnt);`
9. `return res;`

`Xattr::vfs_removexattr(idmap, dentry, name, delegated_inode) -> i32`:

1. `inode_lock(dentry.d_inode);`
2. `let r = Xattr::vfs_removexattr_locked(idmap, dentry, name, delegated_inode);`
3. `inode_unlock(dentry.d_inode);`
4. `return r;`

`Xattr::vfs_removexattr_locked(idmap, dentry, name, delegated_inode) -> i32`:

1. `Security::inode_removexattr(idmap, dentry, name)?;`
2. `let inode = dentry.d_inode;`
3. `if inode.i_flags & (S_IMMUTABLE | S_APPEND) != 0 { return -EPERM; }`
4. `match XattrNs::classify(name) { Trusted => require_cap(CAP_SYS_ADMIN)?, _ => () }`
5. `let handler = xattr_resolve_name(inode.i_sb.s_xattr, name).ok_or(-ENOTSUP)?;`
6. `let r = handler.set(handler, idmap, dentry, inode, name, None, 0, 0);`
7. `if r == 0 { fsnotify_xattr(dentry); inode_inc_iversion(inode); inode.i_ctime = current_time(inode); mark_inode_dirty(inode); }`
8. `return r;`

### Out of Scope

- `removexattr(2)` / `lremovexattr(2)` / `fremovexattr(2)` legacy — separate Tier-5 (`removexattr.md`).
- `getxattrat(2)` / `setxattrat(2)` / `listxattrat(2)` — separate Tier-5.
- LSM handler internals — covered in `security/00-overview.md`.
- Filesystem-specific xattr storage — covered in per-fs Tier-3.
- Implementation code.

### signature

C (POSIX-ish / man-pages):

```c
int removexattrat(int dfd, const char *path, unsigned int at_flags,
                  const char *name);
```

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE4(removexattrat,
                int,                 dfd,
                const char __user *, pathname,
                unsigned int,        at_flags,
                const char __user *, name);
```

Rookery dispatch:

```rust
pub fn sys_removexattrat(
    dfd: i32,
    path: UserPtr<u8>,
    at_flags: u32,
    name: UserPtr<u8>,
) -> SyscallResult<i32>;
```

### parameters

| name      | type                          | constraints                                                                                       | errno-on-bad           |
|-----------|-------------------------------|---------------------------------------------------------------------------------------------------|------------------------|
| dfd       | `int`                         | `AT_FDCWD` or open directory fd; with `AT_EMPTY_PATH` may be any fd type.                         | `EBADF` / `ENOTDIR`    |
| path      | `const char __user *`         | NUL-terminated; `""` only valid with `AT_EMPTY_PATH`.                                              | `EFAULT` / `ENOENT`    |
| at_flags  | `unsigned int`                | Mask of `AT_SYMLINK_NOFOLLOW` and `AT_EMPTY_PATH`; other bits ⟹ `-EINVAL`.                         | `EINVAL`               |
| name      | `const char __user *`         | NUL-terminated; `1 ≤ strlen(name) ≤ XATTR_NAME_MAX (255)`. Known namespace prefix required.       | `EFAULT` / `ERANGE` / `ENOTSUP` |

### return value

- Success: `0`.
- Failure: `-1` with `errno`; in-kernel: negated errno.

### errors

| errno          | condition                                                                                       |
|----------------|-------------------------------------------------------------------------------------------------|
| `EFAULT`       | `path` or `name` crosses the user/kernel boundary illegally.                                    |
| `EBADF`        | `dfd != AT_FDCWD` and `dfd` is not an open fd.                                                  |
| `ENOTDIR`      | `dfd` is not a directory and `path` is non-empty.                                               |
| `ENOENT`       | `path` does not exist; or `path == ""` without `AT_EMPTY_PATH`.                                 |
| `EINVAL`       | `at_flags` has unknown bits; or `name` has no namespace prefix; or `name` is empty.             |
| `EACCES`       | Search permission denied along `path`, or write denied on terminal inode.                       |
| `ELOOP`        | Symlink chain exceeded `MAXSYMLINKS`.                                                           |
| `ENAMETOOLONG` | `path` length ≥ `PATH_MAX`, or `name` length > `XATTR_NAME_MAX`.                                |
| `ERANGE`       | `name` exceeds `XATTR_NAME_MAX`.                                                                |
| `ENODATA`      | Named attribute does not exist (POSIX alias: `ENOATTR`).                                        |
| `ENOTSUP`      | Filesystem does not support xattrs, or namespace prefix unrecognized for this fs.               |
| `EPERM`        | Caller lacks `CAP_SYS_ADMIN` for `trusted.*`; or LSM denies `security.*`; or inode is immutable/append-only. |
| `EROFS`        | Filesystem is read-only.                                                                        |
| `EOPNOTSUPP`   | Synonym for `ENOTSUP` on some architectures.                                                    |

### abi surface (constants + flags)

From `include/uapi/linux/xattr.h` and `include/uapi/linux/fcntl.h`:

- `XATTR_NAME_MAX = 255` — maximum name length.
- `AT_FDCWD = -100`.
- `AT_SYMLINK_NOFOLLOW = 0x100`.
- `AT_EMPTY_PATH = 0x1000`.
- Namespace prefixes (case-sensitive): `user.`, `trusted.`, `security.`, `system.`.

### compatibility contract

- REQ-1: Argument lowering: `%rdi=dfd`, `%rsi=path`, `%rdx=at_flags`, `%r10=name`.
- REQ-2: `at_flags & ~(AT_SYMLINK_NOFOLLOW | AT_EMPTY_PATH) ⟹ -EINVAL`.
- REQ-3: `getname_flags(path, LOOKUP_EMPTY iff AT_EMPTY_PATH)`; empty path without `AT_EMPTY_PATH` ⟹ `-ENOENT`.
- REQ-4: `strncpy_from_user(kname, name, XATTR_NAME_MAX + 1)`; overflow ⟹ `-ERANGE`; length 0 ⟹ `-EINVAL`.
- REQ-5: Path resolution via `do_path_lookupat(dfd, path, lookup_flags, &path)`:
  - `LOOKUP_FOLLOW` set iff `!(at_flags & AT_SYMLINK_NOFOLLOW)`.
  - `LOOKUP_EMPTY` set iff `(at_flags & AT_EMPTY_PATH)`.
- REQ-6: `mnt_want_write(path.mnt)` taken; `mnt_drop_write` on every exit path. RO mount ⟹ `-EROFS`.
- REQ-7: LSM hook `security_inode_removexattr(idmap, dentry, name)` runs before fs handler.
- REQ-8: Immutable (`S_IMMUTABLE`) or append-only (`S_APPEND`) inodes reject xattr removal with `-EPERM` (LSM may permit specific cases).
- REQ-9: `__vfs_removexattr_locked(idmap, dentry, name, &delegated_inode)` acquires `inode_lock`.
- REQ-10: Namespace dispatch via `xattr_full_name`/handler table:
  - `user.*` → `xattr_handler_user.set(value=NULL)` (NULL value triggers remove).
  - `trusted.*` → `xattr_handler_trusted.set` — `capable(CAP_SYS_ADMIN)` gate.
  - `security.*` → `xattr_handler_security.set` — LSM-gated.
  - `system.*` → `xattr_handler_posix_acl_*` (or fs-specific).
- REQ-11: On success: `fsnotify_xattr(dentry)`; `inode_inc_iversion`; `i_ctime` updated; `mark_inode_dirty(inode)`.
- REQ-12: `delegated_inode != NULL` ⟹ NFSv4 delegation must be broken via `break_deleg_wait`; loop retried.
- REQ-13: Attribute not present ⟹ `-ENODATA`.
- REQ-14: Audit record emitted for `security.*` removals when audit enabled.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `at_flags_mask_strict` | INVARIANT | Only `AT_SYMLINK_NOFOLLOW | AT_EMPTY_PATH` accepted. |
| `name_length_bounded` | INVARIANT | `1 ≤ len(name) ≤ XATTR_NAME_MAX`. |
| `prefix_required` | INVARIANT | `name` lacking known prefix ⟹ `-ENOTSUP`. |
| `trusted_cap_gate` | INVARIANT | `trusted.*` removal ⟹ `capable(CAP_SYS_ADMIN)`. |
| `mnt_write_paired` | INVARIANT | Every `mnt_want_write` paired with `mnt_drop_write`. |
| `inode_lock_held` | INVARIANT | `__vfs_removexattr_locked` requires `inode_lock`. |
| `enodata_on_missing` | INVARIANT | Non-existent attribute ⟹ `-ENODATA`. |

### Layer 2: TLA+

`uapi/syscalls/removexattrat.tla`:
- Per-call → validate flags/name → lookup → mnt_want_write → LSM → handler → fsnotify → mnt_drop_write.
- Properties:
  - `safety_flags_validated`,
  - `safety_namespace_capability_gated`,
  - `safety_name_bounded`,
  - `liveness_removexattrat_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post: success ⟹ attribute removed; subsequent `getxattr` ⟹ `-ENODATA` | `Xattr::vfs_removexattr_locked` |
| Post: success ⟹ `i_ctime` advanced; `fsnotify_xattr` emitted | `Xattr::vfs_removexattr_locked` |
| Post: any error ⟹ on-disk state unchanged | `Xattr::vfs_removexattr_locked` |
| Post: `-ENODATA` ⟹ no fsnotify event, no ctime change | `Xattr::vfs_removexattr_locked` |

### Layer 4: Verus/Creusot functional

`removexattrat(dfd, path, at_flags, name)` ≡ unified semantics of `removexattr(2)`/`lremovexattr(2)`/`fremovexattr(2)` per `man 2 removexattrat` (Linux 6.13+) and `Documentation/filesystems/xattr.rst`.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`removexattrat(2)` reinforcement:

- **Per-`at_flags` strict mask** — defense against silent acceptance of unknown flag bits.
- **Per-`XATTR_NAME_MAX` cap** — defense against slab-allocator exhaustion via long names.
- **Per-namespace handler dispatch** — defense against caller-controlled handler selection.
- **Per-LSM hook before fs handler** — defense against bypass of label policy.
- **Per-`mnt_want_write` discipline** — defense against RO-bypass and unmount races.
- **Per-`delegated_inode` break_deleg** — defense against NFSv4 inconsistency.
- **Per-immutable/append-only `-EPERM`** — defense against append-rule bypass via xattr removal.
- **Per-`inode_lock`** — defense against torn read interleaving with set/get.
- **Per-`fsnotify_xattr` post-success only** — defense against false notification on failure.
- **Per-`AT_EMPTY_PATH` strict gate** — defense against accidental fd-as-path coercion.

### grsecurity/pax-style reinforcement

- **PAX_UDEREF** — `path` and `name` SMAP-guarded; `getname`/`strncpy_from_user` bounded by validated sizes.
- **GRKERNSEC_TRUSTED** — `trusted.*` removal denied to container-confined root even with `CAP_SYS_ADMIN` in user namespace; LSM-mediated to prevent label-stripping escapes.
- **GRKERNSEC_SECURITY_XATTR** — `security.*` removal LSM-gated; `security.capability` removal requires `CAP_SETFCAP` even with namespaces.
- **GRKERNSEC_SYSTEM_XATTR** — `system.posix_acl_*` removal mediated by filesystem; cannot be bypassed via direct VFS path.
- **GRKERNSEC_PROC** — `/proc/<pid>/attr` and xattr listings restricted to owner/root, preventing pre-removal reconnaissance.
- **PAX_REFCOUNT** — `path`, `dentry`, `inode` refcounts saturating.
- **GRKERNSEC_LINK** — symlink-traversal during path resolution gated in protected directories.
- **GRKERNSEC_FIFO** — FIFO with sticky-dir + xattr removal gated identically to data delete.
- **GRKERNSEC_CHROOT_FCHDIR** — chrooted `removexattrat` cannot traverse pre-chroot via `dfd`.
- **GRKERNSEC_HIDESYM** — LSM denial printks redact kernel pointers.
- **PAX_RANDKSTACK** — kstack offset randomized at syscall entry.
- **PAX_MEMORY_SANITIZE** — on-disk xattr block zeroed at remove (defense against later block-reuse leak).
- **GRKERNSEC_AUDIT_XATTR** — every `security.capability`, `trusted.*`, and `security.*` removal logged with caller pid/uid/exe and target inode.

