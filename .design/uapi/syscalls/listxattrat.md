# Tier-5: syscall 465 — listxattrat(2)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - arch/x86/entry/syscalls/syscall_64.tbl (`465  common  listxattrat  sys_listxattrat`)
  - fs/xattr.c (`SYSCALL_DEFINE4(listxattrat, ...)`, `do_listxattrat`, `vfs_listxattr`)
  - fs/namei.c (`user_path_at`, `do_path_lookupat`)
  - include/uapi/linux/xattr.h (`XATTR_LIST_MAX`)
  - include/uapi/linux/fcntl.h (`AT_FDCWD`, `AT_SYMLINK_NOFOLLOW`, `AT_EMPTY_PATH`)
  - security/security.c (`security_inode_listxattr`)
-->

## Summary

`listxattrat(2)` is **x86_64 syscall 465**, the **directory-fd-relative, flag-bearing extended-attribute name lister**, introduced in Linux 6.13 alongside the rest of the `*xattrat` family. It generalizes `listxattr(2)`, `llistxattr(2)`, and `flistxattr(2)` into a single uniform entry point patterned after `openat2(2)`:

- Path resolution is anchored at `dfd` (a directory fd) plus `path`.
- Symlink behavior is selected by **`at_flags`** rather than by syscall identity.
- Output buffer (`list`, `size`) is the same NUL-separated name list as the legacy variants.

The kernel walks the inode's xattr handlers and emits a NUL-separated list of attribute names (`"user.foo\0trusted.bar\0security.selinux\0"` etc.) into the caller's buffer. If `size == 0`, the kernel returns the *required* length without writing the buffer (a sizing probe). If the populated list would exceed `size`, the call returns `-ERANGE` and the buffer is undefined; the caller must call again with a larger buffer (or with `size == 0` first).

`at_flags` accepts:

- `AT_SYMLINK_NOFOLLOW` (0x100) — terminal symlink is not followed (equivalent to `llistxattr(2)`).
- `AT_EMPTY_PATH` (0x1000) — `path == ""` operates on `dfd` itself (typically an `O_PATH` fd, equivalent to `flistxattr(2)`).
- Other bits ⟹ `-EINVAL`.

`dfd == AT_FDCWD` and non-empty `path` mirrors `listxattr(2)`; same `dfd` with `AT_SYMLINK_NOFOLLOW` mirrors `llistxattr(2)`; any `dfd` with `AT_EMPTY_PATH` and `path == ""` mirrors `flistxattr(2)`.

Critical for: every modern container runtime introspecting labels safely, every `getfattr -d` user, every backup tool enumerating xattrs, every Rookery xattr-vfs `*at`-discipline list test.

## Signature

C (POSIX-ish / man-pages):

```c
ssize_t listxattrat(int dfd, const char *path, unsigned int at_flags,
                    char *list, size_t size);
```

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE5(listxattrat,
                int,                    dfd,
                const char __user *,    pathname,
                unsigned int,           at_flags,
                char __user *,          list,
                size_t,                 size);
```

Rookery dispatch:

```rust
pub fn sys_listxattrat(
    dfd: i32,
    path: UserPtr<u8>,
    at_flags: u32,
    list: UserPtr<u8>,
    size: usize,
) -> SyscallResult<isize>;
```

## Parameters

| name      | type                          | constraints                                                                                       | errno-on-bad           |
|-----------|-------------------------------|---------------------------------------------------------------------------------------------------|------------------------|
| dfd       | `int`                         | `AT_FDCWD` or open directory fd; with `AT_EMPTY_PATH` may be any fd type.                         | `EBADF` / `ENOTDIR`    |
| path      | `const char __user *`         | NUL-terminated; `""` only valid with `AT_EMPTY_PATH`.                                              | `EFAULT` / `ENOENT`    |
| at_flags  | `unsigned int`                | Mask of `AT_SYMLINK_NOFOLLOW` and `AT_EMPTY_PATH`; other bits ⟹ `-EINVAL`.                         | `EINVAL`               |
| list      | `char __user *`               | Writable for `size` bytes; may be `NULL` only if `size == 0`.                                     | `EFAULT`               |
| size      | `size_t`                      | `0 ≤ size ≤ XATTR_LIST_MAX (65536)`. `0` ⟹ sizing probe (returns required length).                | `E2BIG`                |

## Return value

- Success (`size > 0`, fits): non-negative count of bytes written to `list`.
- Success (`size == 0`): non-negative count of bytes that *would* be written (sizing probe).
- Failure: `-1` with `errno`; in-kernel: negated errno.

## Errors

| errno          | condition                                                                                       |
|----------------|-------------------------------------------------------------------------------------------------|
| `EFAULT`       | `path` or `list` (for `size > 0`) crosses the user/kernel boundary illegally.                   |
| `EBADF`        | `dfd != AT_FDCWD` and `dfd` is not an open fd.                                                  |
| `ENOTDIR`      | `dfd` is not a directory and `path` is non-empty.                                               |
| `ENOENT`       | `path` does not exist; or `path == ""` without `AT_EMPTY_PATH`.                                 |
| `EINVAL`       | `at_flags` has unknown bits; or both `AT_EMPTY_PATH | AT_SYMLINK_NOFOLLOW` semantics conflict.  |
| `EACCES`       | Search permission denied along `path`, or read denied on terminal inode.                        |
| `ELOOP`        | Symlink chain exceeded `MAXSYMLINKS`.                                                           |
| `ENAMETOOLONG` | `path` length ≥ `PATH_MAX`.                                                                     |
| `ERANGE`       | `size > 0` and not large enough to hold the full list.                                          |
| `E2BIG`        | `size > XATTR_LIST_MAX` rejected eagerly.                                                       |
| `ENOTSUP`      | Filesystem does not support xattrs.                                                             |
| `EOPNOTSUPP`   | Synonym for `ENOTSUP` on some architectures.                                                    |

## ABI surface (constants + flags)

From `include/uapi/linux/xattr.h` and `include/uapi/linux/fcntl.h`:

- `XATTR_LIST_MAX = 65536` — maximum list length.
- `AT_FDCWD = -100` — relative to current working directory.
- `AT_SYMLINK_NOFOLLOW = 0x100` — do not follow terminal symlink.
- `AT_EMPTY_PATH = 0x1000` — operate on `dfd` itself when `path == ""`.

## Compatibility contract

- REQ-1: Argument lowering: `%rdi=dfd`, `%rsi=path`, `%rdx=at_flags`, `%r10=list`, `%r8=size`.
- REQ-2: `at_flags & ~(AT_SYMLINK_NOFOLLOW | AT_EMPTY_PATH) ⟹ -EINVAL`.
- REQ-3: `size > XATTR_LIST_MAX ⟹ -E2BIG`.
- REQ-4: `getname_flags(path, LOOKUP_EMPTY iff AT_EMPTY_PATH)` — empty path requires `AT_EMPTY_PATH`; otherwise `-ENOENT`.
- REQ-5: Path resolution via `do_path_lookupat(dfd, name, lookup_flags, &path)`:
  - `LOOKUP_FOLLOW` set iff `!(at_flags & AT_SYMLINK_NOFOLLOW)`.
  - `LOOKUP_EMPTY` set iff `(at_flags & AT_EMPTY_PATH)`.
- REQ-6: For `AT_EMPTY_PATH` with `path == ""`: `dfd` must be a valid fd; inode is `fdget(dfd).file->f_path.dentry->d_inode`. May be a non-directory.
- REQ-7: LSM hook `security_inode_listxattr(dentry)` runs before fs handler.
- REQ-8: Allocate kernel buffer `klist` of `min(size, XATTR_LIST_MAX)` bytes (or none if `size == 0`).
- REQ-9: Invoke `vfs_listxattr(dentry, klist, size)`:
  - Iterates `inode->i_sb->s_xattr` handler table.
  - Filters out handlers whose `list()` callback rejects (e.g., `system.*` ACL handlers gate on inode type).
  - Returns total bytes that would be written; if `size > 0`, also writes into `klist`.
- REQ-10: If `vfs_listxattr` returns `> size` and `size > 0` ⟹ `-ERANGE` to caller.
- REQ-11: If `size > 0` and result fits: `copy_to_user(list, klist, result)`.
- REQ-12: Return value is `ssize_t`: bytes used (or would-use for `size == 0`).
- REQ-13: No `i_ctime` update; this is a read-side call.
- REQ-14: No `mnt_want_write` — read-only operation; permitted on RO filesystems.

## Acceptance Criteria

- [ ] AC-1: `listxattrat(AT_FDCWD, "f", 0, buf, sizeof(buf))` after `setxattr("f", "user.a")` and `setxattr("f", "user.b")` ⟹ buf contains `"user.a\0user.b\0"`, return = 14.
- [ ] AC-2: `listxattrat(..., 0, NULL, 0)` ⟹ returns required size, no write.
- [ ] AC-3: `listxattrat(..., 0, buf, 1)` when full list is 14 ⟹ `-ERANGE`.
- [ ] AC-4: `listxattrat(dfd, "", AT_EMPTY_PATH, buf, sizeof(buf))` with `dfd` an `O_PATH` fd works as `flistxattr`.
- [ ] AC-5: `listxattrat(AT_FDCWD, "", 0, buf, sizeof(buf))` (empty path without `AT_EMPTY_PATH`) ⟹ `-ENOENT`.
- [ ] AC-6: `listxattrat(..., AT_SYMLINK_NOFOLLOW, ...)` on symlink ⟹ lists the symlink's xattrs (not target's).
- [ ] AC-7: `at_flags = 0x200` (unknown bit) ⟹ `-EINVAL`.
- [ ] AC-8: `size > XATTR_LIST_MAX` ⟹ `-E2BIG`.
- [ ] AC-9: Filesystem without xattr support ⟹ `-ENOTSUP` (or `0` with empty list per fs convention).
- [ ] AC-10: LSM denial ⟹ caller errno matches LSM verdict (`-EACCES`/`-EPERM`).
- [ ] AC-11: No xattrs ⟹ returns `0`, no write.
- [ ] AC-12: Multiple namespaces present ⟹ all visible to caller's perms (unprivileged sees `user.*` always; sees `trusted.*` only if `CAP_SYS_ADMIN`).

## Architecture

```
struct ListxattratArgs {
    dfd: i32,
    path: UserPtr<u8>,
    at_flags: u32,
    list: UserPtr<u8>,
    size: usize,
}
```

`Xattr::sys_listxattrat(args) -> isize`:

1. `if args.at_flags & !(AT_SYMLINK_NOFOLLOW | AT_EMPTY_PATH) != 0 { return -EINVAL; }`
2. `if args.size > XATTR_LIST_MAX { return -E2BIG; }`
3. `let lookup_flags = Xattr::lookup_flags_from_at(args.at_flags);`
4. `let path = Path::user_path_at(args.dfd, args.path, lookup_flags)?;`
5. `let res = Xattr::do_listxattr(&path, args.list, args.size);`
6. `path_put(path);`
7. `return res;`

`Xattr::do_listxattr(path, ulist, usize) -> isize`:

1. `let dentry = path.dentry;`
2. `Security::inode_listxattr(dentry)?;`
3. `let klist = if usize > 0 { Some(KernelBuf::alloc(usize)?) } else { None };`
4. `let needed = vfs_listxattr(dentry, klist.as_mut().map(|b| b.as_mut_slice()).unwrap_or(&mut []), usize);`
5. `if needed < 0 { return needed; }`
6. `if usize == 0 { return needed as isize; }`
7. `if (needed as usize) > usize { return -ERANGE; }`
8. `copy_to_user(ulist, klist.as_ref().unwrap().as_slice(), needed as usize)?;`
9. `return needed as isize;`

`Xattr::lookup_flags_from_at(at_flags) -> u32`:

1. `let mut f = 0;`
2. `if (at_flags & AT_SYMLINK_NOFOLLOW) == 0 { f |= LOOKUP_FOLLOW; }`
3. `if (at_flags & AT_EMPTY_PATH) != 0 { f |= LOOKUP_EMPTY; }`
4. `return f;`

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `at_flags_mask_strict` | INVARIANT | Only `AT_SYMLINK_NOFOLLOW | AT_EMPTY_PATH` accepted. |
| `size_bounded` | INVARIANT | `0 ≤ size ≤ XATTR_LIST_MAX`. |
| `sizing_probe_no_write` | INVARIANT | `size == 0` ⟹ no `copy_to_user`. |
| `erange_on_overflow` | INVARIANT | `needed > size > 0` ⟹ `-ERANGE`. |
| `empty_path_requires_flag` | INVARIANT | `path == ""` without `AT_EMPTY_PATH` ⟹ `-ENOENT`. |
| `no_mnt_write_taken` | INVARIANT | Read-side call; `mnt_want_write` never invoked. |

### Layer 2: TLA+

`uapi/syscalls/listxattrat.tla`:
- Per-call → validate flags/size → lookup → LSM → vfs_listxattr → copy_to_user.
- Properties:
  - `safety_flags_validated`,
  - `safety_size_within_XATTR_LIST_MAX`,
  - `safety_no_write_on_sizing_probe`,
  - `liveness_listxattrat_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post: success ⟹ return = bytes filled = sum of name lengths + NULs | `Xattr::do_listxattr` |
| Post: sizing probe ⟹ no user-buffer write | `Xattr::do_listxattr` |
| Post: `-ERANGE` ⟹ user buffer untouched | `Xattr::do_listxattr` |
| Post: any error ⟹ no kernel state mutation | `Xattr::sys_listxattrat` |

### Layer 4: Verus/Creusot functional

`listxattrat(dfd, path, at_flags, list, size)` ≡ unified semantics of `listxattr(2)`/`llistxattr(2)`/`flistxattr(2)` per `man 2 listxattrat` (Linux 6.13+) and `Documentation/filesystems/xattr.rst`.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`listxattrat(2)` reinforcement:

- **Per-`at_flags` strict mask** — defense against silent acceptance of unknown flag bits.
- **Per-`XATTR_LIST_MAX` cap** — defense against unbounded kernel allocation.
- **Per-sizing-probe contract** — defense against forced large allocations from naive callers.
- **Per-LSM hook before fs handler** — defense against bypass of label-visibility policy.
- **Per-`AT_EMPTY_PATH` strict gate** — defense against accidental fd-as-path coercion.
- **Per-`LOOKUP_FOLLOW`/`LOOKUP_EMPTY` derivation** — defense against caller-controlled lookup-flags injection.
- **No-`mnt_want_write` policy** — preserved; read-only on RO mounts.

## Grsecurity/PaX-style Reinforcement

- **PAX_UDEREF** — `path` and `list` SMAP-guarded; `getname`/`copy_to_user` bounded by validated sizes.
- **GRKERNSEC_TRUSTED** — `trusted.*` names elided from list emitted to non-CAP_SYS_ADMIN callers (LSM-mediated; the list is the kernel's filtered view, not the raw inode view).
- **GRKERNSEC_SECURITY_XATTR** — `security.*` names visible per LSM verdict (e.g., SELinux contexts visible only to authorized peers).
- **GRKERNSEC_SYSTEM_XATTR** — `system.posix_acl_*` names visible only when fs supports ACLs; namespace boundary enforced.
- **PAX_REFCOUNT** — `path`, `dentry`, `inode` refcounts saturating across the walk.
- **GRKERNSEC_LINK** — symlink traversal under `*at` rules; chrooted task cannot escape via `AT_SYMLINK_NOFOLLOW` semantics.
- **GRKERNSEC_FIFO** — FIFO/special-file xattr listing gated identically to regular file.
- **GRKERNSEC_CHROOT_FCHDIR** — chrooted `listxattrat` cannot traverse pre-chroot via `dfd`.
- **GRKERNSEC_HIDESYM** — LSM denial printks redact kernel pointers (no inode addresses).
- **PAX_RANDKSTACK** — kstack offset randomized at syscall entry.
- **PAX_MEMORY_SANITIZE** — `klist` buffer zeroed on free, preventing slab-reuse leak of xattr names.
- **GRKERNSEC_AUDIT_XATTR** — listing on inodes carrying `security.*` or `trusted.*` xattrs logged with caller uid/exe.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `listxattr(2)` / `llistxattr(2)` / `flistxattr(2)` legacy — separate Tier-5 (`listxattr.md`).
- `getxattrat(2)` / `setxattrat(2)` / `removexattrat(2)` — separate Tier-5.
- LSM handler internals — covered in `security/00-overview.md`.
- Filesystem-specific xattr storage — covered in per-fs Tier-3.
- Implementation code.
