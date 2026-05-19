---
title: "Tier-5: syscall 463 — setxattrat(2)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`setxattrat(2)` is **x86_64 syscall 463**, the **directory-fd-relative, flag-bearing extended-attribute setter** introduced in Linux 6.13 (commit `f7b9d1cb6c5b`). It generalizes the four legacy variants — `setxattr(2)`, `lsetxattr(2)`, `fsetxattr(2)`, and a hypothetical fd-+-path combinator — into a single uniform entry point patterned after `openat2(2)`/`fchmodat2(2)`/`utimensat(2)`:

- Path resolution is anchored at `dfd` (a directory fd) plus `path`.
- Symlink behavior is selected by **`at_flags`** rather than by syscall identity.
- The setxattr-specific arguments (`value`, `size`, `flags`) are bundled into a versioned `struct xattr_args` user pointer rather than passed as registers, leaving room for future extensions (e.g. transactional flags, idmap overrides).

The `at_flags` argument accepts:

- `AT_SYMLINK_NOFOLLOW` (0x100) — terminal symlink is not followed. Equivalent to the `l*` variant.
- `AT_EMPTY_PATH` (0x1000) — `path == ""` operates on `dfd` itself (typically an `O_PATH` fd). Equivalent to the `f*` variant.
- Other bits ⟹ `-EINVAL`.

`dfd == AT_FDCWD` and non-empty `path` mirrors `setxattr(2)` (with terminal-symlink follow); the same `dfd` with `AT_SYMLINK_NOFOLLOW` mirrors `lsetxattr(2)`; any `dfd` with `AT_EMPTY_PATH` and `path == ""` mirrors `fsetxattr(2)`.

The `struct xattr_args` layout (in `include/uapi/linux/xattr.h`):

```c
struct xattr_args {
    __aligned_u64 value;   /* user pointer to value buffer (or 0) */
    __u32         size;    /* value length in bytes */
    __u32         flags;   /* XATTR_CREATE | XATTR_REPLACE */
};
```

The `size` argument outside the struct is the *struct length* (sizeof at the time of the call) — this is the `openat2`-style copy_struct_from_user pattern: future kernels may add fields; older callers pass smaller structs and the kernel zero-extends.

Critical for: every modern container runtime using `*at` discipline to set labels safely, every installer setting `security.capability` via `O_PATH` fd, every filesystem that wants per-call idmap or transactional semantics in the future, every Rookery xattr-vfs `*at`-discipline test.

### Acceptance Criteria

- [ ] AC-1: `setxattrat(AT_FDCWD, "f", 0, "user.foo", &args, sizeof(args))` ≡ `setxattr("f", "user.foo", ...)`.
- [ ] AC-2: `setxattrat(AT_FDCWD, "link", AT_SYMLINK_NOFOLLOW, ...)` ≡ `lsetxattr("link", ...)`.
- [ ] AC-3: `setxattrat(fd, "", AT_EMPTY_PATH, ...)` ≡ `fsetxattr(fd, ...)`.
- [ ] AC-4: Unknown `at_flags` bit → `-EINVAL`.
- [ ] AC-5: `size < 16` → `-EINVAL`; `size > known with trailing nonzero` → `-E2BIG`.
- [ ] AC-6: `args.flags == (XATTR_CREATE | XATTR_REPLACE)` → `-EINVAL`.
- [ ] AC-7: `args.size > XATTR_SIZE_MAX` → `-E2BIG`.
- [ ] AC-8: Unprivileged `trusted.*` write → `-EPERM`.
- [ ] AC-9: `AT_EMPTY_PATH` on non-empty path → `-ENOENT` or `-EINVAL` per path-lookup policy.
- [ ] AC-10: On success: `i_ctime` advanced and `fsnotify_xattr` emitted.

### Architecture

```
#[repr(C)]
struct XattrArgs {
    value: u64,
    size: u32,
    flags: u32,
}

struct SetxattratArgs {
    dfd: i32,
    path: UserPtr<u8>,
    at_flags: u32,
    name: UserPtr<u8>,
    args: UserPtr<XattrArgs>,
    size: usize,
}
```

`Xattr::sys_setxattrat(a) -> i32`:

1. `if a.at_flags & !(AT_SYMLINK_NOFOLLOW | AT_EMPTY_PATH) != 0 { return -EINVAL; }`
2. `let mut kargs = XattrArgs::default(); copy_struct_from_user(&mut kargs, mem::size_of::<XattrArgs>(), a.args, a.size)?;`
3. `if kargs.flags & !(XATTR_CREATE | XATTR_REPLACE) != 0 { return -EINVAL; }`
4. `if kargs.flags == (XATTR_CREATE | XATTR_REPLACE) { return -EINVAL; }`
5. `if kargs.size > XATTR_SIZE_MAX as u32 { return -E2BIG; }`
6. `let mut lookup = 0u32; if !(a.at_flags & AT_SYMLINK_NOFOLLOW) { lookup |= LOOKUP_FOLLOW; } if a.at_flags & AT_EMPTY_PATH { lookup |= LOOKUP_EMPTY; }`
7. `let path = Path::user_path_at(a.dfd, a.path, lookup)?;`
8. `let kname = NameBuf::copy_from_user(a.name, XATTR_NAME_MAX + 1)?;`
9. `if kname.len() == 0 || kname.len() > XATTR_NAME_MAX { path_put(path); return -ERANGE; }`
10. `let kvalue = if kargs.size > 0 { Some(UserBuf::memdup_from_user(UserPtr::new(kargs.value as *const u8), kargs.size as usize)?) } else { None };`
11. `mnt_want_write(path.mnt)?;`
12. `let mut delegated_inode = ptr::null_mut(); loop { let r = Xattr::setxattr(path.idmap(), path.dentry, &kname, kvalue.as_deref(), kargs.size as usize, kargs.flags as i32, &mut delegated_inode); if r == -EWOULDBLOCK { break_deleg_wait(&mut delegated_inode)?; continue; } break r; }`
13. `mnt_drop_write(path.mnt); path_put(path); return r;`

`Xattr::setxattr` and `Xattr::vfs_setxattr_locked` are shared with `setxattr(2)`; the only divergence is at the entry layer (path resolution flags + arg-struct copy).

### Out of Scope

- `getxattrat(2)` — separate Tier-5 (`getxattrat.md`).
- `removexattrat(2)` — separate Tier-5 (`removexattrat.md`) if expanded.
- `listxattrat(2)` — separate Tier-5 (`listxattrat.md`) if expanded.
- `struct xattr_args` future versions (v1+) — covered when added.
- POSIX ACL semantics — covered in `fs/posix_acl.md` Tier-3.
- LSM handler internals — covered in `security/00-overview.md`.
- Filesystem-specific xattr storage — covered in per-fs Tier-3.
- Implementation code.

### signature

C (POSIX-ish / man-pages):

```c
int setxattrat(int dfd, const char *path, unsigned int at_flags,
               const char *name, const struct xattr_args *args, size_t size);
```

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE6(setxattrat,
                int,                          dfd,
                const char       __user *,   pathname,
                unsigned int,                 at_flags,
                const char       __user *,   name,
                const struct xattr_args __user *, args,
                size_t,                       size);
```

Rookery dispatch:

```rust
pub fn sys_setxattrat(
    dfd: i32,
    path: UserPtr<u8>,
    at_flags: u32,
    name: UserPtr<u8>,
    args: UserPtr<XattrArgs>,
    size: usize,
) -> SyscallResult<i32>;
```

### parameters

| name      | type                              | constraints                                                                                       | errno-on-bad           |
|-----------|-----------------------------------|---------------------------------------------------------------------------------------------------|------------------------|
| dfd       | `int`                             | `AT_FDCWD` or open directory fd; with `AT_EMPTY_PATH` may be any fd type.                         | `EBADF` / `ENOTDIR`    |
| path      | `const char __user *`             | NUL-terminated; `""` only valid with `AT_EMPTY_PATH`.                                              | `EFAULT` / `ENOENT`    |
| at_flags  | `unsigned int`                    | Mask of `AT_SYMLINK_NOFOLLOW` and `AT_EMPTY_PATH`; other bits ⟹ `-EINVAL`.                         | `EINVAL`               |
| name      | `const char __user *`             | NUL-terminated; `1 ≤ strlen(name) ≤ XATTR_NAME_MAX`. Known namespace prefix required.             | `EFAULT` / `ERANGE` / `ENOTSUP` |
| args      | `const struct xattr_args __user *`| Pointer to user `xattr_args` struct of `size` bytes.                                              | `EFAULT` / `E2BIG`     |
| size      | `size_t`                          | `sizeof(struct xattr_args)` at user's call time. Must be ≥ `XATTR_ARGS_SIZE_VER0` (16).            | `EINVAL` / `E2BIG`     |

`args.value`, `args.size` (internal), and `args.flags` carry the actual value pointer, value length, and `XATTR_CREATE`/`XATTR_REPLACE` mask as in `setxattr(2)`.

### return value

- Success: `0`.
- Failure: `< 0` — negated errno.

### errors

| errno          | condition                                                                                       |
|----------------|-------------------------------------------------------------------------------------------------|
| `EFAULT`       | Any user pointer crosses kernel boundary illegally.                                             |
| `EBADF`        | `dfd` is not a valid open fd.                                                                   |
| `ENOTDIR`      | `dfd` is not a directory and `path` is non-empty.                                                |
| `EINVAL`       | `at_flags` has bits outside `{AT_SYMLINK_NOFOLLOW, AT_EMPTY_PATH}`; or `args.flags` invalid; or `size < XATTR_ARGS_SIZE_VER0`. |
| `E2BIG`        | `size` exceeds kernel's max known struct version; or `args.size > XATTR_SIZE_MAX`.              |
| `ENAMETOOLONG` | `path` `≥ PATH_MAX`, or `name` `> XATTR_NAME_MAX`.                                              |
| `ENOENT`       | `path` does not exist (and not `AT_EMPTY_PATH`).                                                 |
| `EACCES`       | Search/write permission denied.                                                                 |
| `ELOOP`        | Symlink chain exceeded `MAXSYMLINKS`.                                                           |
| `EEXIST`       | `XATTR_CREATE` and attribute already exists.                                                    |
| `ENODATA`      | `XATTR_REPLACE` and attribute does not exist.                                                   |
| `ENOTSUP`      | Filesystem does not support xattrs, or namespace prefix unrecognized.                           |
| `EPERM`        | Caller lacks `CAP_SYS_ADMIN` for `trusted.*`; LSM denies `security.*`; or file is immutable.    |
| `EROFS`        | Filesystem is read-only.                                                                        |

### abi surface

From `include/uapi/linux/xattr.h`:

- `XATTR_ARGS_SIZE_VER0 = 16` — sizeof(struct xattr_args) for the initial v0 layout.
- `XATTR_CREATE = 0x1`, `XATTR_REPLACE = 0x2` — mutually exclusive flags inside `args.flags`.
- `XATTR_NAME_MAX = 255`, `XATTR_SIZE_MAX = 65536`.

From `include/uapi/linux/fcntl.h`:

- `AT_SYMLINK_NOFOLLOW = 0x100`.
- `AT_EMPTY_PATH = 0x1000`.

`copy_struct_from_user(&kargs, sizeof(kargs), uargs, size)` handles forward-compat: callers may pass a smaller struct (kernel zero-extends) but the unknown-bits-must-be-zero rule applies — extra non-zero bytes at the tail ⟹ `-E2BIG`.

### compatibility contract

- REQ-1: Argument lowering: `%rdi=dfd`, `%rsi=path`, `%rdx=at_flags`, `%r10=name`, `%r8=args`, `%r9=size`.
- REQ-2: `at_flags & ~(AT_SYMLINK_NOFOLLOW | AT_EMPTY_PATH) ⟹ -EINVAL`.
- REQ-3: `size < XATTR_ARGS_SIZE_VER0 ⟹ -EINVAL`. `size > sizeof(known struct) && trailing non-zero ⟹ -E2BIG`. (Standard `copy_struct_from_user` semantics.)
- REQ-4: Compute lookup flags: `lookup = (at_flags & AT_SYMLINK_NOFOLLOW) ? 0 : LOOKUP_FOLLOW; lookup |= (at_flags & AT_EMPTY_PATH) ? LOOKUP_EMPTY : 0;`.
- REQ-5: `user_path_at(dfd, path, lookup, &p)`.
- REQ-6: `kargs.flags & ~(XATTR_CREATE | XATTR_REPLACE) ⟹ -EINVAL`. Both set ⟹ `-EINVAL`.
- REQ-7: `kargs.size > XATTR_SIZE_MAX ⟹ -E2BIG`.
- REQ-8: `strncpy_from_user(kname, name, XATTR_NAME_MAX + 1)`; length validation as in `setxattr`.
- REQ-9: `kvalue = memdup_user((void __user *)kargs.value, kargs.size)` when `kargs.size > 0`.
- REQ-10: `mnt_want_write(p.mnt)` taken; `mnt_drop_write` on every exit.
- REQ-11: `__vfs_setxattr_locked(p.idmap, p.dentry, kname, kvalue, kargs.size, kargs.flags, &delegated_inode)`.
- REQ-12: LSM hook `security_inode_setxattr` runs before fs handler.
- REQ-13: Namespace dispatch identical to `setxattr(2)`.
- REQ-14: On success: `fsnotify_xattr`, `inode_inc_iversion`, `i_ctime`, `mark_inode_dirty`.
- REQ-15: NFSv4 delegation break loop via `break_deleg_wait`.
- REQ-16: `path_put(p)` on every exit path.
- REQ-17: `AT_EMPTY_PATH` requires the caller to either own the fd's process or have `CAP_DAC_READ_SEARCH` (per `path_init` policy).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `at_flags_strict` | INVARIANT | Bits outside `{AT_SYMLINK_NOFOLLOW, AT_EMPTY_PATH}` ⟹ `-EINVAL`. |
| `args_size_strict` | INVARIANT | `size < XATTR_ARGS_SIZE_VER0` ⟹ `-EINVAL`; trailing nonzero ⟹ `-E2BIG`. |
| `args_flags_exclusive` | INVARIANT | `XATTR_CREATE` and `XATTR_REPLACE` mutually exclusive. |
| `name_length_bounded` | INVARIANT | `1 ≤ len(name) ≤ XATTR_NAME_MAX`. |
| `value_size_bounded` | INVARIANT | `0 ≤ kargs.size ≤ XATTR_SIZE_MAX`. |
| `mnt_write_paired` | INVARIANT | Every `mnt_want_write` paired with `mnt_drop_write`. |
| `at_empty_path_requires_empty_string` | INVARIANT | `AT_EMPTY_PATH` only meaningful when `path == ""`. |

### Layer 2: TLA+

`uapi/syscalls/setxattrat.tla`:
- Per-call → validate at_flags → copy_struct_from_user → validate kargs → lookup(dfd, path, lookup_flags) → mnt_want_write → LSM → handler → fsnotify → mnt_drop_write → path_put.
- Properties:
  - `safety_at_flags_validated`,
  - `safety_args_struct_versioned`,
  - `safety_equivalent_to_legacy_variants`,
  - `liveness_setxattrat_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post: `at_flags == 0` ≡ `setxattr` behavior | `Xattr::sys_setxattrat` |
| Post: `AT_SYMLINK_NOFOLLOW` ≡ `lsetxattr` behavior | `Xattr::sys_setxattrat` |
| Post: `AT_EMPTY_PATH` + `path == ""` ≡ `fsetxattr` behavior | `Xattr::sys_setxattrat` |
| Post: `copy_struct_from_user` zero-extends shorter structs | `Xattr::sys_setxattrat` |
| Post: success ⟹ `i_ctime` advanced and `fsnotify_xattr` emitted | `Xattr::vfs_setxattr_locked` |

### Layer 4: Verus/Creusot functional

`setxattrat(dfd, path, at_flags, name, args, size)` ≡ Linux setxattrat(2) per `man 2 setxattrat` and the original commit message; unifies the four legacy variants behind a single `*at` entry with `openat2`-style versioned struct.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`setxattrat(2)` reinforcement:

- **Per-`at_flags` strict mask** — defense against silent acceptance of unknown bits.
- **Per-`copy_struct_from_user` discipline** — defense against version-confusion attacks (kernel zero-extends; trailing nonzero is rejected).
- **Per-`AT_EMPTY_PATH` LOOKUP_EMPTY** — defense against accidentally operating on dfd when path is non-empty.
- **Per-`XATTR_NAME_MAX` cap** — defense against slab exhaustion.
- **Per-`XATTR_SIZE_MAX` cap** — defense against unbounded kvalue allocation.
- **Per-namespace handler dispatch** — defense against caller-controlled handler selection.
- **Per-LSM hook before fs handler** — defense against bypass of policy.
- **Per-`mnt_want_write` discipline** — defense against RO-bypass and unmount-races.
- **Per-`delegated_inode` break_deleg** — defense against NFSv4 inconsistent state.
- **Per-immutable/append-only `-EPERM`** — defense against rule bypass.

### grsecurity/pax-style reinforcement

- **PAX_UDEREF** — `path`, `name`, `args`, and `args.value` SMAP-guarded; `copy_struct_from_user` and `memdup_user` bound-checked under SMAP.
- **GRKERNSEC_TRUSTED** — only root sees and writes `trusted.*`; namespace-confined containers may still be denied.
- **GRKERNSEC_SECURITY_XATTR** — `security.*` writes LSM-gated; `security.capability` requires `CAP_SETFCAP`.
- **GRKERNSEC_SYSTEM_XATTR** — `system.*` writes mediated by fs (POSIX ACL semantics).
- **GRKERNSEC_PROC** — `/proc/<pid>` walks via `dfd` gated by proc-restrict.
- **PAX_REFCOUNT** — `struct path`, dentry, and fd refcounts saturating; UAF prevention on long `*at` chains.
- **GRKERNSEC_LINK** — symlink-followed `*at` resolution in protected directories gated. `AT_SYMLINK_NOFOLLOW` cooperates with link-protection by reading the link's xattr without traversing.
- **GRKERNSEC_FIFO** — FIFO with sticky-dir + `setxattrat` gated identically to data write.
- **GRKERNSEC_CHROOT_FCHDIR** — `dfd` referencing pre-chroot directory is gated against chroot-escape.
- **GRKERNSEC_HIDESYM** — LSM denial printks redact pointers.
- **PAX_RANDKSTACK** — kstack offset randomized at syscall entry.
- **PAX_MEMORY_SANITIZE** — `kvalue` and `kargs` zeroed on free; prevents info-leak of prior xattr bytes through slab reuse.
- **GRKERNSEC_AUDIT_XATTR** — every `security.*` and `trusted.*` write via `*at` logged with caller pid/uid/exe, dfd, and resolved path.
- **GRKERNSEC_HARDEN_USERCOPY** — `copy_struct_from_user` size-check enforced; trailing-byte non-zero check enforced.

