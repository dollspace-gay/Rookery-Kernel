---
title: "Tier-5: syscall 191 — getxattr(2)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`getxattr(2)` is **x86_64 syscall 191**, the path-relative extended-attribute reader. It retrieves the value blob of a single named xattr attached to the inode identified by `path` (terminal symlinks **followed**). The call is the read-side mirror of `setxattr(2)`: identical namespace partitioning (`user.*`, `trusted.*`, `security.*`, `system.*`), identical name-length cap (`XATTR_NAME_MAX = 255`), identical value-size cap (`XATTR_SIZE_MAX = 65536`), and identical LSM-mediation discipline through `security_inode_getxattr`.

Two reader-side conventions matter:

- **Probe-then-fetch.** Callers may invoke `getxattr(path, name, NULL, 0)` to discover the attribute's current length without copying the value. The kernel returns the required buffer size; `EFAULT` is not raised on the `NULL` buffer when `size == 0`.
- **Truncation rejection.** If the provided `size` is non-zero and **smaller** than the actual value length, the kernel returns `-ERANGE` rather than truncating. The user buffer is not modified on truncation. This contrasts with read/write byte-stream semantics and is important for cryptographic xattrs (`security.ima`, `security.capability`) where a truncated read could be catastrophic.

Internally the call lowers to `path_getxattr(path, name, value, size, LOOKUP_FOLLOW)` → `getxattr(path_idmap, dentry, name, value, size)` → `vfs_getxattr(idmap, dentry, name, kvalue, size)`. The LSM hook `security_inode_getxattr` runs **before** the fs handler; on `security.*` reads, the LSM may rewrite or synthesize the value (e.g. SELinux returns the canonical context).

Critical for: every libc `getxattr`, every `getcap` reading `security.capability`, every container-runtime label reader, every IMA verifier, every Rookery xattr-vfs read test.

### Acceptance Criteria

- [ ] AC-1: `getxattr("f", "user.foo", buf, 64)` returns the byte count and `buf` contains the value.
- [ ] AC-2: `getxattr("f", "user.foo", NULL, 0)` returns the value length without copying.
- [ ] AC-3: `getxattr("f", "user.missing", buf, 64)` → `-ENODATA`.
- [ ] AC-4: `getxattr` with `size` smaller than actual value → `-ERANGE`; `buf` untouched.
- [ ] AC-5: `getxattr("f", "noprefix", ...)` → `-ENOTSUP`.
- [ ] AC-6: `getxattr` `strlen(name) > 255` → `-ERANGE`.
- [ ] AC-7: Terminal symlink is followed (xattr read from target).
- [ ] AC-8: LSM denial mapped to caller's errno (`-EACCES`/`-EPERM`).
- [ ] AC-9: Probe-then-fetch sequence is race-aware: between probe and fetch, value may grow; second call returns `-ERANGE` and caller retries.
- [ ] AC-10: `i_atime` is not advanced by `getxattr`.

### Architecture

```
struct GetxattrArgs {
    path: UserPtr<u8>,
    name: UserPtr<u8>,
    value: UserPtrMut<u8>,
    size: usize,
}
```

`Xattr::sys_getxattr(args) -> isize`:

1. `let path = Path::user_path_at(AT_FDCWD, args.path, LOOKUP_FOLLOW)?;`
2. `return Xattr::path_getxattr(&path, args.name, args.value, args.size);`

`Xattr::path_getxattr(path, name, value, size) -> isize`:

1. `let kname = NameBuf::copy_from_user(name, XATTR_NAME_MAX + 1)?;`
2. `if kname.len() == 0 || kname.len() > XATTR_NAME_MAX { return -ERANGE; }`
3. `if size > XATTR_SIZE_MAX { return -E2BIG; }`
4. `let kvalue = if size > 0 { Some(KernBuf::kvzalloc(size, GFP_KERNEL)?) } else { None };`
5. `let r = Xattr::vfs_getxattr(path.idmap(), path.dentry, &kname, kvalue.as_deref_mut(), size);`
6. `if r > 0 && size > 0 { copy_to_user(value, kvalue.as_ref().unwrap(), r as usize)?; }`
7. `if let Some(buf) = kvalue { buf.zero_on_drop(); }`
8. `path_put(path); return r;`

`Xattr::vfs_getxattr(idmap, dentry, name, value, size) -> isize`:

1. `Security::inode_getxattr(dentry, name)?;`
2. /* LSM may satisfy `security.*` reads directly */
3. `if name.starts_with("security.") { if let Some(r) = Security::xattr_getsecurity(idmap, dentry.d_inode, &name[9..], value, size) { return r; } }`
4. `let handler = xattr_resolve_name(inode.i_sb.s_xattr, name)?;`
5. `let r = handler.get(handler, dentry, inode, name, value, size);`
6. `return r;`

### Out of Scope

- `lgetxattr(2)` — separate Tier-5 (`lgetxattr.md`).
- `fgetxattr(2)` — separate Tier-5 (`fgetxattr.md`).
- `getxattrat(2)` — separate Tier-5 (`getxattrat.md`).
- POSIX ACL semantics (`system.posix_acl_*`) — covered in `fs/posix_acl.md` Tier-3 if expanded.
- LSM handler internals — covered in `security/00-overview.md`.
- Filesystem-specific xattr storage — covered in per-fs Tier-3.
- Implementation code.

### signature

C (POSIX-ish / man-pages):

```c
ssize_t getxattr(const char *path, const char *name,
                 void *value, size_t size);
```

glibc wrapper: `__getxattr` → `INLINE_SYSCALL(getxattr, 4, path, name, value, size)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE4(getxattr,
                const char __user *, pathname,
                const char __user *, name,
                void        __user *, value,
                size_t,               size);
```

Rookery dispatch:

```rust
pub fn sys_getxattr(
    path: UserPtr<u8>,
    name: UserPtr<u8>,
    value: UserPtrMut<u8>,
    size: usize,
) -> SyscallResult<isize>;
```

### parameters

| name    | type                   | constraints                                                                                       | errno-on-bad           |
|---------|------------------------|---------------------------------------------------------------------------------------------------|------------------------|
| path    | `const char __user *`  | NUL-terminated; length `< PATH_MAX`. Terminal symlink followed.                                   | `EFAULT` / `ENAMETOOLONG` / `ENOENT` |
| name    | `const char __user *`  | NUL-terminated; `1 ≤ strlen(name) ≤ XATTR_NAME_MAX (255)`. Must begin with a known namespace prefix. | `EFAULT` / `ERANGE` / `ENOTSUP` |
| value   | `void __user *`        | Writable for `size` bytes; may be `NULL` when `size == 0` (probe mode).                           | `EFAULT`               |
| size    | `size_t`               | `0 ≤ size ≤ XATTR_SIZE_MAX (65536)`. `size == 0` ⟹ probe.                                          | `ERANGE`               |

### return value

- Success: number of bytes copied into `value` (≥ 0). When `size == 0`, returns the required buffer length without copying.
- Failure: `< 0` — negated errno.

### errors

| errno          | condition                                                                                       |
|----------------|-------------------------------------------------------------------------------------------------|
| `EFAULT`       | `path`, `name`, or `value` (for non-zero `size`) crosses the user/kernel boundary illegally.    |
| `ENAMETOOLONG` | `path` length `≥ PATH_MAX`, or `name` length `> XATTR_NAME_MAX`.                                 |
| `ENOENT`       | `path` does not exist.                                                                          |
| `EACCES`       | Search permission denied along `path`, or read permission on terminal inode denied.             |
| `ELOOP`        | Symlink chain exceeded `MAXSYMLINKS`.                                                           |
| `ENOTDIR`      | Non-terminal component of `path` is not a directory.                                            |
| `ENODATA`      | Attribute does not exist (POSIX alias: `ENOATTR`).                                              |
| `ERANGE`       | `size` is non-zero and smaller than the actual value length. Buffer is not modified.            |
| `ENOTSUP`      | Filesystem does not support xattrs, or namespace prefix is unrecognized for this fs.            |
| `EPERM`        | LSM denies the read (e.g. unprivileged reading `trusted.*` on a fs that hides it).              |

### abi surface (constants + flags)

From `include/uapi/linux/xattr.h`:

- `XATTR_NAME_MAX = 255` — maximum name length (excluding NUL).
- `XATTR_SIZE_MAX = 65536` — maximum value length returned in any single call.
- Namespace prefixes (string literals, case-sensitive):
  - `XATTR_USER_PREFIX = "user."`.
  - `XATTR_TRUSTED_PREFIX = "trusted."`.
  - `XATTR_SECURITY_PREFIX = "security."`.
  - `XATTR_SYSTEM_PREFIX = "system."`.

A name lacking any of the four prefixes is rejected with `ENOTSUP` (or `EOPNOTSUPP`) by `xattr_resolve_name`. Probe mode (`size == 0`) is universal across namespaces.

### compatibility contract

- REQ-1: Argument lowering: `%rdi=path`, `%rsi=name`, `%rdx=value`, `%r10=size`.
- REQ-2: `user_path_at(AT_FDCWD, path, LOOKUP_FOLLOW, &p)` — terminal symlinks followed.
- REQ-3: `strncpy_from_user(kname, name, XATTR_NAME_MAX + 1)`; lengths `> XATTR_NAME_MAX` ⟹ `-ERANGE`; length `0` ⟹ `-ERANGE`.
- REQ-4: `size > XATTR_SIZE_MAX ⟹ -E2BIG` (treated as oversize request).
- REQ-5: When `size > 0`, kernel allocates `kvalue = kvmalloc(size, GFP_KERNEL)`; on success, `copy_to_user(value, kvalue, ret)`; on failure path, kvalue is zeroed and freed.
- REQ-6: When `size == 0`, kernel does **not** allocate a value buffer; `vfs_getxattr(... NULL, 0)` returns the required length directly from the fs handler.
- REQ-7: LSM hook `security_inode_getxattr(dentry, name)` runs **before** the fs handler; for `security.*` names, the LSM may itself satisfy the read (`xattr_getsecurity`).
- REQ-8: Namespace dispatch via `xattr_resolve_handler`:
  - `user.*` → `xattr_handler_user.get`.
  - `trusted.*` → `xattr_handler_trusted.get` (no `CAP_SYS_ADMIN` requirement to read on most fs, but LSM may add gating).
  - `security.*` → `xattr_handler_security.get`; LSM-synthesized when applicable.
  - `system.*` → `xattr_handler_posix_acl_*` etc.
- REQ-9: On actual-size > `size` (size > 0): return `-ERANGE`; user buffer untouched.
- REQ-10: Idmap (`mnt_idmap`) is threaded through for permission checks on user-namespace mounts.
- REQ-11: No mount-write reservation (`mnt_want_write`) is taken — this is a read.
- REQ-12: `ATIME` is **not** updated by `getxattr` (the inode's data was not read).
- REQ-13: Audit record emitted for `security.*` reads when audit is enabled.
- REQ-14: `path_put(p)` on every exit path.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `name_length_bounded` | INVARIANT | `1 ≤ len(name) ≤ XATTR_NAME_MAX`. |
| `value_size_bounded` | INVARIANT | `0 ≤ size ≤ XATTR_SIZE_MAX`. |
| `probe_no_copy` | INVARIANT | `size == 0` ⟹ no allocation, no `copy_to_user`. |
| `truncation_rejected` | INVARIANT | Actual length > size > 0 ⟹ `-ERANGE`; buffer untouched. |
| `prefix_required` | INVARIANT | `name` lacking known prefix ⟹ `-ENOTSUP`. |
| `kvalue_zeroed_on_drop` | INVARIANT | Kernel value buffer zeroed before free. |

### Layer 2: TLA+

`uapi/syscalls/getxattr.tla`:
- Per-call → validate → lookup → LSM → handler → optional copy_to_user → path_put.
- Properties:
  - `safety_truncation_rejected`,
  - `safety_probe_no_userbuf_access`,
  - `safety_namespace_partitioned`,
  - `liveness_getxattr_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post: `size == 0` ⟹ return ≥ 0 is actual-length, user buffer untouched | `Xattr::path_getxattr` |
| Post: success with `size > 0` ⟹ first `ret` bytes of value copied | `Xattr::path_getxattr` |
| Post: `-ERANGE` ⟹ user buffer unchanged | `Xattr::path_getxattr` |
| Post: kvalue allocation zeroed on free | `Xattr::path_getxattr` |

### Layer 4: Verus/Creusot functional

`getxattr(path, name, value, size)` ≡ Linux getxattr(2) per `man 2 getxattr` and `Documentation/filesystems/xattr.rst`, including probe semantics and truncation rejection.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`getxattr(2)` reinforcement:

- **Per-`XATTR_NAME_MAX` cap** — defense against slab exhaustion via long names.
- **Per-`XATTR_SIZE_MAX` cap** — defense against unbounded kernel buffer allocation on read.
- **Per-truncation `-ERANGE` (no partial copy)** — defense against silent crypto/label truncation.
- **Per-namespace handler dispatch** — defense against caller-controlled handler selection.
- **Per-LSM hook before fs handler** — defense against bypass of policy on `security.*` reads.
- **Per-no `mnt_want_write`** — defense against unnecessary RO-mount writer reservations.
- **Per-no `atime` update** — defense against side-channel atime probing of xattr existence.
- **Per-probe-mode zero-alloc** — defense against denial-of-service via repeated zero-size getxattrs.
- **Per-`kvfree` zero-on-drop** — defense against slab-reuse info-leak.

### grsecurity/pax-style reinforcement

- **PAX_UDEREF** — `path`, `name`, and `value` SMAP-guarded; `strncpy_from_user`/`copy_to_user` bound-checked under SMAP.
- **GRKERNSEC_TRUSTED** — only root sees `trusted.*` values; unprivileged readers receive `-ENODATA` to avoid existence-leak.
- **GRKERNSEC_SECURITY_XATTR** — `security.*` reads LSM-gated; SELinux/Smack denial silenced from unprivileged callers.
- **GRKERNSEC_SYSTEM_XATTR** — `system.*` (POSIX ACL) reads mediated by fs handlers; no raw blob exposure.
- **GRKERNSEC_PROC** — pid-namespace and proc-restrict apply: `/proc/<pid>/attr` xattr-equivalent reads hidden from non-owner.
- **PAX_REFCOUNT** — `struct path` refcounts saturating, preventing UAF on long probe-then-fetch loops.
- **GRKERNSEC_LINK** — symlink-followed path resolution in protected dirs gated against attacker-owned link races.
- **GRKERNSEC_FIFO** — FIFO xattr reads in sticky dirs gated identically to data reads.
- **GRKERNSEC_CHROOT_FCHDIR** — chroot'd `getxattr` cannot traverse pre-chroot paths.
- **GRKERNSEC_HIDESYM** — LSM denial printks redact kernel pointers and attribute values.
- **PAX_RANDKSTACK** — kstack offset randomized at syscall entry.
- **PAX_MEMORY_SANITIZE** — kvalue buffer zeroed on free, preventing info-leak of prior xattr bytes via slab reuse.
- **GRKERNSEC_AUDIT_XATTR** — every `security.*` and `trusted.*` read logged with caller pid/uid/exe and attribute name.
- **PAX_USERCOPY** — `copy_to_user(value, kvalue, ret)` bounded by `min(ret, size)`, hardened against overlapping/misaligned userspace.

