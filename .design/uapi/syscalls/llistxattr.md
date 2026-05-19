# Tier-5: syscall 195 — llistxattr(2)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - arch/x86/entry/syscalls/syscall_64.tbl (`195  common  llistxattr  sys_llistxattr`)
  - fs/xattr.c (`SYSCALL_DEFINE3(llistxattr, ...)`, `path_listxattr`, `listxattr`, `vfs_listxattr`)
  - fs/namei.c (`user_path_at` with no `LOOKUP_FOLLOW`)
  - include/uapi/linux/xattr.h (`XATTR_LIST_MAX`)
  - security/security.c (`security_inode_listsecurity`)
-->

## Summary

`llistxattr(2)` is **x86_64 syscall 195**, the **non-symlink-following** path-relative extended-attribute name-enumerator. It is identical to `listxattr(2)` except that the terminal path component, if it is a symlink, is **not** dereferenced — the call enumerates the xattrs of the symlink itself rather than those of the link target. This is essential for tools that need to discover xattrs on the symlink (the `trusted.*` and `security.*` namespaces are permitted on symlinks; the `user.*` namespace is not, per filesystem support).

Output format is the same NUL-separated list:

```
"security.selinux\0trusted.label\0"
```

The split between `listxattr` (follow) and `llistxattr` (no-follow) is the same convention as `stat`/`lstat`, `getxattr`/`lgetxattr`, `setxattr`/`lsetxattr`. Callers operating on symlinks (backup tools, `cp -a --no-dereference`, container image archivers preserving link labels) MUST use `llistxattr` to avoid silently reading xattrs from the target inode.

Critical for: every `tar`, `cpio`, `cp -a` symlink preservation; every container image extractor that reproduces link labels; every `getfattr -h` invocation; every Rookery xattr-vfs symlink-respecting enumeration test.

## Signature

C (POSIX-ish / man-pages):

```c
ssize_t llistxattr(const char *path, char *list, size_t size);
```

glibc wrapper: `__llistxattr` → `INLINE_SYSCALL(llistxattr, 3, path, list, size)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE3(llistxattr,
                const char __user *, pathname,
                char        __user *, list,
                size_t,               size);
```

Rookery dispatch:

```rust
pub fn sys_llistxattr(
    path: UserPtr<u8>,
    list: UserPtrMut<u8>,
    size: usize,
) -> SyscallResult<isize>;
```

## Parameters

| name | type                  | constraints                                                    | errno-on-bad           |
|------|-----------------------|----------------------------------------------------------------|------------------------|
| path | `const char __user *` | NUL-terminated; length `< PATH_MAX`. Terminal symlink kept.    | `EFAULT` / `ENAMETOOLONG` / `ENOENT` |
| list | `char __user *`       | Writable for `size` bytes; may be `NULL` when `size == 0`.     | `EFAULT`               |
| size | `size_t`              | `0 ≤ size ≤ XATTR_LIST_MAX (65536)`.                           | `ERANGE` / `E2BIG`     |

## Return value

- Success: total number of bytes (including all interior NUL terminators) needed to hold the full list.
- `0` when the inode (symlink) has no xattrs.
- Failure: `< 0` — negated errno.

## Errors

| errno          | condition                                                                              |
|----------------|----------------------------------------------------------------------------------------|
| `EFAULT`       | `path` or `list` (with non-zero `size`) crosses the user/kernel boundary illegally.    |
| `ENAMETOOLONG` | `path` length `≥ PATH_MAX`.                                                            |
| `ENOENT`       | `path` does not exist.                                                                 |
| `EACCES`       | Search permission denied along `path`.                                                 |
| `ENOTDIR`      | Non-terminal component is not a directory.                                             |
| `ERANGE`       | `size` is non-zero and smaller than the total list length; buffer untouched.           |
| `ENOTSUP`      | Filesystem does not implement `listxattr` for the symlink inode.                       |
| `E2BIG`        | `size > XATTR_LIST_MAX`.                                                               |
| `EPERM`        | LSM denial on overall list enumeration.                                                |

Note: `ELOOP` is not produced because the terminal symlink is not followed. Mid-path symlinks may still cause `ELOOP` if their chain exceeds `MAXSYMLINKS`.

## ABI surface

- `XATTR_LIST_MAX = 65536`.
- `XATTR_NAME_MAX = 255` still applies to each name.
- LSMs filter out names whose values the caller cannot read.

The symlink inode supports a narrow xattr namespace:
- `security.*`: allowed (used by SELinux, capability propagation through certain link types).
- `trusted.*`: allowed (CAP_SYS_ADMIN only).
- `system.posix_acl_access`: NOT allowed (symlinks have no ACL).
- `user.*`: NOT allowed on most filesystems.

## Compatibility contract

- REQ-1: Argument lowering: `%rdi=path`, `%rsi=list`, `%rdx=size`.
- REQ-2: `user_path_at(AT_FDCWD, path, 0 /* no LOOKUP_FOLLOW */, &p)` — terminal symlink **kept**.
- REQ-3: `size > XATTR_LIST_MAX ⟹ -E2BIG`.
- REQ-4: When `size > 0`, kernel allocates `klist = kvmalloc(size, GFP_KERNEL)`; on success, `copy_to_user(list, klist, ret)`; klist is zeroed before free.
- REQ-5: When `size == 0`, kernel does not allocate; `vfs_listxattr(... NULL, 0)` returns required length.
- REQ-6: `vfs_listxattr(dentry, klist, size)` dispatch identical to `listxattr` path; dentry's inode is the symlink itself.
- REQ-7: `security_inode_listsecurity` appends LSM-synthesized `security.*` names.
- REQ-8: LSM may filter names the caller cannot read.
- REQ-9: `i_atime` not updated.
- REQ-10: No `mnt_want_write` — read-only.

## Acceptance Criteria

- [ ] AC-1: `llistxattr("link", buf, 4096)` returns xattrs of the symlink itself, not its target.
- [ ] AC-2: `llistxattr("link", NULL, 0)` returns required length without copying.
- [ ] AC-3: `llistxattr` on symlink with no xattrs returns `0`.
- [ ] AC-4: `llistxattr` with `size` smaller than total → `-ERANGE`; buffer untouched.
- [ ] AC-5: `llistxattr` on fs without xattr support → `-ENOTSUP`.
- [ ] AC-6: For a regular file (non-symlink), behaves identically to `listxattr`.
- [ ] AC-7: Unprivileged caller does not see `trusted.*` names on the link.
- [ ] AC-8: LSM-filtered `security.*` names omitted for callers without read permission.
- [ ] AC-9: `i_atime` not advanced on either symlink or target.
- [ ] AC-10: Each entry in the returned list is NUL-terminated.

## Architecture

```rust
struct LlistxattrArgs {
    path: UserPtr<u8>,
    list: UserPtrMut<u8>,
    size: usize,
}
```

`Xattr::sys_llistxattr(args) -> isize`:

1. `let path = Path::user_path_at(AT_FDCWD, args.path, 0 /* no FOLLOW */)?;`
2. `return Xattr::path_listxattr(&path, args.list, args.size);`

`Xattr::path_listxattr(path, list, size) -> isize`:

1. `if size > XATTR_LIST_MAX { return -E2BIG; }`
2. `let klist = if size > 0 { Some(KernBuf::kvzalloc(size, GFP_KERNEL)?) } else { None };`
3. `let r = Xattr::vfs_listxattr(path.dentry, klist.as_deref_mut(), size);`
4. `if r > 0 && size > 0 { copy_to_user(list, klist.as_ref().unwrap(), r as usize)?; }`
5. `if let Some(buf) = klist { buf.zero_on_drop(); }`
6. `path_put(path); return r;`

`Xattr::vfs_listxattr(dentry, list, size) -> isize`:

1. `let inode = dentry.d_inode; // for symlinks: the link inode, not target */`
2. `let mut used: usize = 0;`
3. `if let Some(op) = inode.i_op.listxattr { used = op(dentry, list, size)?; }`
4. `else { used = Xattr::generic_listxattr(dentry, list, size)?; }`
5. `used += Security::inode_listsecurity(inode, list.offset_by(used), size - used)?;`
6. `if used > size && size > 0 { return -ERANGE; }`
7. `return used as isize;`

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `no_lookup_follow` | INVARIANT | `user_path_at` flags do NOT include `LOOKUP_FOLLOW`. |
| `list_size_bounded` | INVARIANT | `0 ≤ size ≤ XATTR_LIST_MAX`. |
| `probe_no_copy` | INVARIANT | `size == 0` ⟹ no allocation, no `copy_to_user`. |
| `truncation_rejected` | INVARIANT | Actual total > size > 0 ⟹ `-ERANGE`; buffer untouched. |
| `lsm_filtered_names_omitted` | INVARIANT | Names whose values the caller cannot read are not included. |
| `klist_zeroed_on_drop` | INVARIANT | Kernel list buffer zeroed before free. |
| `link_inode_used` | INVARIANT | For symlinks, dentry's d_inode is the link inode. |

### Layer 2: TLA+

`uapi/syscalls/llistxattr.tla`:
- Per-call → validate → lookup-no-follow → vfs_listxattr → LSM-append → optional copy_to_user → path_put.
- Properties:
  - `safety_link_inode_enumerated` — terminal-symlink inode is the xattr source.
  - `safety_truncation_rejected`,
  - `safety_probe_no_userbuf_access`,
  - `safety_lsm_filtering_enforced`,
  - `liveness_llistxattr_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post: dentry resolved from path without LOOKUP_FOLLOW | `Xattr::sys_llistxattr` |
| Post: `size == 0` ⟹ return ≥ 0 is total-length, user buffer untouched | `Xattr::path_listxattr` |
| Post: success with `size > 0` ⟹ first `ret` bytes of list copied | `Xattr::path_listxattr` |
| Post: `-ERANGE` ⟹ user buffer unchanged | `Xattr::path_listxattr` |
| Post: klist allocation zeroed on free | `Xattr::path_listxattr` |

### Layer 4: Verus/Creusot functional

`llistxattr(path, list, size)` ≡ Linux llistxattr(2) per `man 2 llistxattr` and `Documentation/filesystems/xattr.rst`. Per-symlink-inode semantics confirmed against `tar --xattrs --no-dereference`.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`llistxattr(2)` reinforcement:

- **Per-`XATTR_LIST_MAX` cap** — defense against unbounded kernel buffer allocation.
- **Per-truncation `-ERANGE`** — defense against partial-list parsing UB.
- **Per-LSM filtering of unreadable names** — defense against existence side-channel.
- **Per-no-follow on terminal** — defense against TOCTOU symlink-target swap during enumeration.
- **Per-no `mnt_want_write`** — defense against unnecessary RO writer reservations.
- **Per-no `atime` update** — defense against side-channel atime probing.
- **Per-`kvfree` zero-on-drop** — defense against slab-reuse info-leak.
- **Per-probe-mode zero-alloc** — defense against denial-of-service via repeated zero-size enumerations.

## Grsecurity/PaX-style Reinforcement

- **PAX_UDEREF** — `path` and `list` SMAP-guarded in `getname`/`copy_to_user`; output length bound-checked.
- **GRKERNSEC_TRUSTED xattr LSM-mediation** — unprivileged callers do not see `trusted.*` names on symlinks at all; the kernel filters them through the LSM layer. Without this, the mere presence of `trusted.x` on a link could leak information about a privileged labeling step performed against the link inode.
- **GRKERNSEC_SECURITY_XATTR** — `security.*` names filtered per-LSM: SELinux/Smack omit names the caller cannot read.
- **GRKERNSEC_LINK** — symlink-enumeration in protected sticky dirs gated against attacker-owned link races at the parent-directory level.
- **PAX_REFCOUNT** — `struct path` refcounts saturating; UAF prevention on long probe-then-fetch.
- **GRKERNSEC_FIFO** — sticky-dir FIFO link enumeration gated identically.
- **GRKERNSEC_CHROOT_FCHDIR** — chroot'd `llistxattr` cannot traverse pre-chroot paths.
- **GRKERNSEC_HIDESYM** — LSM denial printks redact pointers; names that would leak kernel state suppressed.
- **PAX_RANDKSTACK** — kstack offset randomized at syscall entry.
- **PAX_MEMORY_SANITIZE** — klist buffer zeroed on free; prevents info-leak.
- **GRKERNSEC_AUDIT_XATTR** — llistxattr on protected paths logged with caller pid/uid/exe and inode.
- **PAX_USERCOPY** — `copy_to_user(list, klist, ret)` bound-checked.

## Open Questions

(none at this Tier-5 level)
