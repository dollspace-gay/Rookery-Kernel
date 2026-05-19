# Tier-5: syscall 464 — getxattrat(2)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - arch/x86/entry/syscalls/syscall_64.tbl (`464  common  getxattrat  sys_getxattrat`)
  - fs/xattr.c (`SYSCALL_DEFINE6(getxattrat, ...)`, `do_getxattrat`, `vfs_getxattr`)
  - fs/namei.c (`user_path_at`, `do_path_lookupat`)
  - include/uapi/linux/xattr.h (`struct xattr_args`)
  - include/uapi/linux/fcntl.h (`AT_FDCWD`, `AT_SYMLINK_NOFOLLOW`, `AT_EMPTY_PATH`)
  - security/security.c (`security_inode_getxattr`)
-->

## Summary

`getxattrat(2)` is **x86_64 syscall 464**, the **directory-fd-relative, flag-bearing extended-attribute reader** introduced alongside `setxattrat(2)` in Linux 6.13. It is the read-side mirror of `setxattrat(2)` and unifies `getxattr(2)`, `lgetxattr(2)`, `fgetxattr(2)`, and any fd-+-path combinator behind one `openat2`-style entry point.

Routing matches `setxattrat`:

- `dfd == AT_FDCWD`, non-empty `path`, no `AT_SYMLINK_NOFOLLOW` ⟹ `getxattr(2)` semantics.
- `dfd == AT_FDCWD`, non-empty `path`, `AT_SYMLINK_NOFOLLOW` ⟹ `lgetxattr(2)` semantics.
- Any `dfd`, `path == ""`, `AT_EMPTY_PATH` ⟹ `fgetxattr(2)` semantics.
- Any `dfd` (directory), non-empty `path` ⟹ path is resolved relative to `dfd` (the genuine `*at` use case container runtimes need to avoid TOCTOU on `/proc/self/root`-style anchors).

The getter-specific arguments — output buffer pointer, output buffer size — are bundled into the same `struct xattr_args` user pointer used by `setxattrat`:

```c
struct xattr_args {
    __aligned_u64 value;   /* user pointer to output buffer (or 0 for probe) */
    __u32         size;    /* output buffer size in bytes */
    __u32         flags;   /* reserved, must be 0 for getxattrat */
};
```

`args.flags` must be **zero** for `getxattrat` (the `XATTR_CREATE`/`XATTR_REPLACE` flags are write-only). Non-zero ⟹ `-EINVAL`. Probe mode (`args.size == 0`, optionally `args.value == 0`) returns the required length without copying. Truncation rejection (`args.size > 0` and smaller than actual) returns `-ERANGE`.

The `size` argument outside the struct is the *struct length*, handled by `copy_struct_from_user` exactly as in `setxattrat`.

Critical for: every modern container runtime using `*at` discipline to read labels safely, every `getcap` reading via `O_PATH` fd, every backup tool walking a tree with a directory-fd anchor, every Rookery xattr-vfs `*at`-read test.

## Signature

C (POSIX-ish / man-pages):

```c
ssize_t getxattrat(int dfd, const char *path, unsigned int at_flags,
                   const char *name, struct xattr_args *args, size_t size);
```

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE6(getxattrat,
                int,                       dfd,
                const char    __user *,   pathname,
                unsigned int,              at_flags,
                const char    __user *,   name,
                struct xattr_args __user *, args,
                size_t,                    size);
```

Rookery dispatch:

```rust
pub fn sys_getxattrat(
    dfd: i32,
    path: UserPtr<u8>,
    at_flags: u32,
    name: UserPtr<u8>,
    args: UserPtrMut<XattrArgs>,
    size: usize,
) -> SyscallResult<isize>;
```

## Parameters

| name      | type                              | constraints                                                                                       | errno-on-bad           |
|-----------|-----------------------------------|---------------------------------------------------------------------------------------------------|------------------------|
| dfd       | `int`                             | `AT_FDCWD` or open directory fd; with `AT_EMPTY_PATH` may be any fd type.                         | `EBADF` / `ENOTDIR`    |
| path      | `const char __user *`             | NUL-terminated; `""` only valid with `AT_EMPTY_PATH`.                                              | `EFAULT` / `ENOENT`    |
| at_flags  | `unsigned int`                    | Mask of `AT_SYMLINK_NOFOLLOW` and `AT_EMPTY_PATH`; other bits ⟹ `-EINVAL`.                         | `EINVAL`               |
| name      | `const char __user *`             | NUL-terminated; `1 ≤ strlen(name) ≤ XATTR_NAME_MAX`. Known namespace prefix required.             | `EFAULT` / `ERANGE` / `ENOTSUP` |
| args      | `struct xattr_args __user *`      | Pointer to user `xattr_args` struct of `size` bytes; `args.flags` must be `0`.                    | `EFAULT` / `EINVAL`    |
| size      | `size_t`                          | `sizeof(struct xattr_args)` at user's call time. Must be ≥ `XATTR_ARGS_SIZE_VER0` (16).            | `EINVAL` / `E2BIG`     |

`args.value` is the output buffer pointer; `args.size` is its capacity in bytes (probe when `0`).

## Return value

- Success: number of bytes copied into `args.value` (≥ 0). When `args.size == 0`, returns the required buffer length without copying.
- Failure: `< 0` — negated errno.

## Errors

| errno          | condition                                                                                       |
|----------------|-------------------------------------------------------------------------------------------------|
| `EFAULT`       | Any user pointer crosses kernel boundary illegally.                                             |
| `EBADF`        | `dfd` is not a valid open fd.                                                                   |
| `ENOTDIR`      | `dfd` is not a directory and `path` is non-empty.                                                |
| `EINVAL`       | `at_flags` invalid; or `args.flags != 0`; or `size < XATTR_ARGS_SIZE_VER0`.                      |
| `E2BIG`        | `size` exceeds known struct max with trailing nonzero; or `args.size > XATTR_SIZE_MAX`.         |
| `ENAMETOOLONG` | `path` `≥ PATH_MAX`, or `name` `> XATTR_NAME_MAX`.                                              |
| `ENOENT`       | `path` does not exist.                                                                          |
| `EACCES`       | Search/read permission denied.                                                                  |
| `ELOOP`        | Symlink chain exceeded `MAXSYMLINKS`.                                                           |
| `ENODATA`      | Attribute does not exist.                                                                       |
| `ERANGE`       | `args.size` is non-zero and smaller than the actual value length. Buffer untouched.             |
| `ENOTSUP`      | Filesystem does not support xattrs, or namespace prefix unrecognized.                           |
| `EPERM`        | LSM denial.                                                                                     |

## ABI surface

From `include/uapi/linux/xattr.h`:

- `XATTR_ARGS_SIZE_VER0 = 16`.
- `XATTR_NAME_MAX = 255`, `XATTR_SIZE_MAX = 65536`.
- `args.flags` reserved for future read-mode flags; must be `0` in v0.

From `include/uapi/linux/fcntl.h`:

- `AT_SYMLINK_NOFOLLOW = 0x100`, `AT_EMPTY_PATH = 0x1000`.

`copy_struct_from_user` semantics identical to `setxattrat`. Future kernels may add fields; older callers pass smaller structs.

## Compatibility contract

- REQ-1: Argument lowering: `%rdi=dfd`, `%rsi=path`, `%rdx=at_flags`, `%r10=name`, `%r8=args`, `%r9=size`.
- REQ-2: `at_flags & ~(AT_SYMLINK_NOFOLLOW | AT_EMPTY_PATH) ⟹ -EINVAL`.
- REQ-3: `size < XATTR_ARGS_SIZE_VER0 ⟹ -EINVAL`. `copy_struct_from_user(&kargs, sizeof(kargs), uargs, size)` with trailing-nonzero check.
- REQ-4: `kargs.flags != 0 ⟹ -EINVAL` (no read-side flags defined in v0).
- REQ-5: `kargs.size > XATTR_SIZE_MAX ⟹ -E2BIG`.
- REQ-6: Compute lookup flags: `lookup = (at_flags & AT_SYMLINK_NOFOLLOW) ? 0 : LOOKUP_FOLLOW; lookup |= (at_flags & AT_EMPTY_PATH) ? LOOKUP_EMPTY : 0;`.
- REQ-7: `user_path_at(dfd, path, lookup, &p)`.
- REQ-8: `strncpy_from_user(kname, name, XATTR_NAME_MAX + 1)`; length validation as in `getxattr`.
- REQ-9: When `kargs.size > 0`, allocate `kvalue = kvzalloc(kargs.size, GFP_KERNEL)`. Probe mode (`kargs.size == 0`) skips allocation.
- REQ-10: LSM hook `security_inode_getxattr(p.dentry, kname)` runs before fs handler.
- REQ-11: `vfs_getxattr(p.idmap, p.dentry, kname, kvalue, kargs.size)` dispatches per-namespace.
- REQ-12: On `ret > 0 && kargs.size > 0`: `copy_to_user((void *)kargs.value, kvalue, ret)`.
- REQ-13: kvalue zeroed before free on every exit path.
- REQ-14: Truncation: actual > kargs.size > 0 ⟹ `-ERANGE`; user buffer untouched.
- REQ-15: No `mnt_want_write` (read-only).
- REQ-16: `path_put(p)` on every exit path.
- REQ-17: `atime` not updated.
- REQ-18: Audit record emitted for `security.*` reads.

## Acceptance Criteria

- [ ] AC-1: `getxattrat(AT_FDCWD, "f", 0, "user.foo", &args, sizeof(args))` ≡ `getxattr("f", "user.foo", ...)`.
- [ ] AC-2: `getxattrat(AT_FDCWD, "link", AT_SYMLINK_NOFOLLOW, ...)` ≡ `lgetxattr("link", ...)`.
- [ ] AC-3: `getxattrat(fd, "", AT_EMPTY_PATH, ...)` ≡ `fgetxattr(fd, ...)`.
- [ ] AC-4: Unknown `at_flags` bit → `-EINVAL`.
- [ ] AC-5: `args.flags != 0` → `-EINVAL`.
- [ ] AC-6: `args.size > XATTR_SIZE_MAX` → `-E2BIG`.
- [ ] AC-7: `getxattrat` probe mode (`args.size == 0`) returns required length without copying.
- [ ] AC-8: `getxattrat` truncation → `-ERANGE`; output buffer untouched.
- [ ] AC-9: LSM denial mapped to `-EACCES`/`-EPERM`.
- [ ] AC-10: `i_atime` not advanced.
- [ ] AC-11: `dfd`-anchored path resolution avoids `/proc/self/root` TOCTOU.

## Architecture

```
#[repr(C)]
struct XattrArgs {
    value: u64,
    size: u32,
    flags: u32,
}

struct GetxattratArgs {
    dfd: i32,
    path: UserPtr<u8>,
    at_flags: u32,
    name: UserPtr<u8>,
    args: UserPtrMut<XattrArgs>,
    size: usize,
}
```

`Xattr::sys_getxattrat(a) -> isize`:

1. `if a.at_flags & !(AT_SYMLINK_NOFOLLOW | AT_EMPTY_PATH) != 0 { return -EINVAL; }`
2. `let mut kargs = XattrArgs::default(); copy_struct_from_user(&mut kargs, mem::size_of::<XattrArgs>(), a.args, a.size)?;`
3. `if kargs.flags != 0 { return -EINVAL; }`
4. `if kargs.size > XATTR_SIZE_MAX as u32 { return -E2BIG; }`
5. `let mut lookup = 0u32; if !(a.at_flags & AT_SYMLINK_NOFOLLOW) { lookup |= LOOKUP_FOLLOW; } if a.at_flags & AT_EMPTY_PATH { lookup |= LOOKUP_EMPTY; }`
6. `let path = Path::user_path_at(a.dfd, a.path, lookup)?;`
7. `let kname = NameBuf::copy_from_user(a.name, XATTR_NAME_MAX + 1)?;`
8. `if kname.len() == 0 || kname.len() > XATTR_NAME_MAX { path_put(path); return -ERANGE; }`
9. `let kvalue = if kargs.size > 0 { Some(KernBuf::kvzalloc(kargs.size as usize, GFP_KERNEL)?) } else { None };`
10. `let r = Xattr::vfs_getxattr(path.idmap(), path.dentry, &kname, kvalue.as_deref_mut(), kargs.size as usize);`
11. `if r > 0 && kargs.size > 0 { copy_to_user(UserPtrMut::new(kargs.value as *mut u8), kvalue.as_ref().unwrap(), r as usize)?; }`
12. `if let Some(buf) = kvalue { buf.zero_on_drop(); }`
13. `path_put(path); return r;`

`Xattr::vfs_getxattr` is shared with the legacy variants — the only divergence is at the entry layer (path resolution flags + arg-struct copy).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `at_flags_strict` | INVARIANT | Bits outside `{AT_SYMLINK_NOFOLLOW, AT_EMPTY_PATH}` ⟹ `-EINVAL`. |
| `args_size_strict` | INVARIANT | `size < XATTR_ARGS_SIZE_VER0` ⟹ `-EINVAL`. |
| `args_flags_must_be_zero` | INVARIANT | `kargs.flags != 0` ⟹ `-EINVAL`. |
| `name_length_bounded` | INVARIANT | `1 ≤ len(name) ≤ XATTR_NAME_MAX`. |
| `value_size_bounded` | INVARIANT | `0 ≤ kargs.size ≤ XATTR_SIZE_MAX`. |
| `probe_no_copy` | INVARIANT | `kargs.size == 0` ⟹ no allocation, no `copy_to_user`. |
| `truncation_rejected` | INVARIANT | Actual > kargs.size > 0 ⟹ `-ERANGE`; buffer untouched. |
| `kvalue_zeroed_on_drop` | INVARIANT | Kernel value buffer zeroed before free. |

### Layer 2: TLA+

`uapi/syscalls/getxattrat.tla`:
- Per-call → validate at_flags → copy_struct_from_user → validate kargs → lookup(dfd, path, lookup_flags) → LSM → handler → optional copy_to_user → path_put.
- Properties:
  - `safety_at_flags_validated`,
  - `safety_args_struct_versioned`,
  - `safety_equivalent_to_legacy_variants`,
  - `safety_truncation_rejected`,
  - `liveness_getxattrat_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post: `at_flags == 0` ≡ `getxattr` behavior | `Xattr::sys_getxattrat` |
| Post: `AT_SYMLINK_NOFOLLOW` ≡ `lgetxattr` behavior | `Xattr::sys_getxattrat` |
| Post: `AT_EMPTY_PATH` + `path == ""` ≡ `fgetxattr` behavior | `Xattr::sys_getxattrat` |
| Post: `kargs.size == 0` ⟹ no allocation, returns required length | `Xattr::sys_getxattrat` |
| Post: `-ERANGE` ⟹ user buffer unchanged | `Xattr::sys_getxattrat` |
| Post: kvalue allocation zeroed on free | `Xattr::sys_getxattrat` |

### Layer 4: Verus/Creusot functional

`getxattrat(dfd, path, at_flags, name, args, size)` ≡ Linux getxattrat(2) per `man 2 getxattrat` and the original commit message; unifies the three legacy reader variants behind a single `*at` entry with `openat2`-style versioned struct.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`getxattrat(2)` reinforcement:

- **Per-`at_flags` strict mask** — defense against silent acceptance of unknown bits.
- **Per-`args.flags == 0` requirement** — defense against future-flag confusion on the read side.
- **Per-`copy_struct_from_user` discipline** — defense against version-confusion attacks.
- **Per-`XATTR_NAME_MAX` cap** — defense against slab exhaustion.
- **Per-`XATTR_SIZE_MAX` cap** — defense against unbounded kvalue allocation.
- **Per-truncation `-ERANGE`** — defense against silent label/capability truncation.
- **Per-namespace handler dispatch** — defense against caller-controlled handler selection.
- **Per-LSM hook before fs handler** — defense against bypass of policy on `security.*` reads.
- **Per-no `mnt_want_write`** — defense against unnecessary RO-mount writer reservations.
- **Per-no `atime` update** — defense against side-channel atime probing.
- **Per-`kvfree` zero-on-drop** — defense against slab-reuse info-leak.
- **Per-probe-mode zero-alloc** — defense against denial-of-service via repeated zero-size getxattrats.

## Grsecurity/PaX-style Reinforcement

- **PAX_UDEREF** — `path`, `name`, `args`, and the user output buffer SMAP-guarded; `copy_struct_from_user` and `copy_to_user` bound-checked under SMAP.
- **GRKERNSEC_TRUSTED** — only root sees `trusted.*` values; unprivileged callers receive `-ENODATA` to avoid existence-leak.
- **GRKERNSEC_SECURITY_XATTR** — `security.*` reads LSM-gated; SELinux/Smack denial silenced from unprivileged callers.
- **GRKERNSEC_SYSTEM_XATTR** — `system.*` reads mediated by fs; no raw blob exposure.
- **GRKERNSEC_PROC** — `/proc/<pid>` walks via `dfd` gated by proc-restrict; non-owner cannot probe `/proc/<pid>/exe` xattrs via dfd-relative anchor.
- **PAX_REFCOUNT** — `struct path`, dentry, and fd refcounts saturating; UAF prevention on long `*at` chains.
- **GRKERNSEC_LINK** — symlink-followed `*at` resolution in protected directories gated against attacker-owned link races. `AT_SYMLINK_NOFOLLOW` cooperates with link-protection.
- **GRKERNSEC_FIFO** — FIFO with sticky-dir + `getxattrat` gated identically to data read.
- **GRKERNSEC_CHROOT_FCHDIR** — `dfd` referencing pre-chroot directory is gated against chroot-escape; `getxattrat` cannot use an out-of-chroot dfd to read in-chroot xattrs.
- **GRKERNSEC_HIDESYM** — LSM denial printks redact pointers and labels.
- **PAX_RANDKSTACK** — kstack offset randomized at syscall entry.
- **PAX_MEMORY_SANITIZE** — `kvalue` and `kargs` zeroed on free; prevents info-leak of prior xattr bytes through slab reuse.
- **GRKERNSEC_AUDIT_XATTR** — every `security.*` and `trusted.*` read via `*at` logged with caller pid/uid/exe, dfd, and resolved path.
- **PAX_USERCOPY** — `copy_to_user(args.value, kvalue, ret)` bound-checked, hardened against misaligned userspace.
- **GRKERNSEC_HARDEN_USERCOPY** — `copy_struct_from_user` size-check and trailing-byte non-zero check enforced.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `setxattrat(2)` — separate Tier-5 (`setxattrat.md`).
- `removexattrat(2)` — separate Tier-5 (`removexattrat.md`) if expanded.
- `listxattrat(2)` — separate Tier-5 (`listxattrat.md`) if expanded.
- `struct xattr_args` future versions (v1+) — covered when added.
- POSIX ACL semantics — covered in `fs/posix_acl.md` Tier-3.
- LSM handler internals — covered in `security/00-overview.md`.
- Filesystem-specific xattr storage — covered in per-fs Tier-3.
- Implementation code.
