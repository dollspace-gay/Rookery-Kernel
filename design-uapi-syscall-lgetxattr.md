---
title: "Tier-5: syscall 192 — lgetxattr(2)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`lgetxattr(2)` is **x86_64 syscall 192**, the path-relative extended-attribute reader **that does not follow terminal symlinks**. It is identical to `getxattr(2)` in every respect except the `LOOKUP_FOLLOW` flag is omitted from `user_path_at`, so when `path` resolves to a symbolic link the xattr is read from the **link itself** rather than from the link's target.

Why a separate syscall: xattrs on symlinks are meaningful. SELinux, IMA, and tar/cpio archivers all need to read security labels and capabilities attached to the link's inode (not the target's inode). Most filesystems support only `security.*` and `trusted.*` namespaces on symlinks (the `user.*` namespace is restricted to regular files and directories per the VFS handler); reading `user.*` from a symlink typically returns `-ENODATA`.

Internally `lgetxattr` lowers to `path_getxattr(path, name, value, size, 0)` (note the **zero** lookup-flag argument vs `LOOKUP_FOLLOW` for `getxattr`). All other machinery — `vfs_getxattr`, LSM hook `security_inode_getxattr`, namespace handler dispatch, name-length cap, size cap, truncation `-ERANGE` discipline — is identical to `getxattr(2)`. Probe mode (`size == 0`) returns the required length without copying.

Critical for: every libc `lgetxattr`, every `getcap -h`, every `tar`/`rsync` preserving symlink xattrs, every SELinux relabeler iterating with `nftw(FTW_PHYS)`, every Rookery xattr-vfs symlink-read test.

### Acceptance Criteria

- [ ] AC-1: `lgetxattr(link, "security.selinux", buf, 64)` reads the link's own SELinux label.
- [ ] AC-2: `lgetxattr` on regular file behaves identically to `getxattr` on regular file.
- [ ] AC-3: `lgetxattr(link, "user.foo", buf, 64)` on a symlink-restricted fs → `-ENODATA` or `-EPERM`.
- [ ] AC-4: `lgetxattr` probe mode (`size == 0`) returns length without copying.
- [ ] AC-5: `lgetxattr` truncation → `-ERANGE`; buffer untouched.
- [ ] AC-6: Non-terminal symlinks still followed during path walk.
- [ ] AC-7: `lgetxattr` `strlen(name) > 255` → `-ERANGE`.
- [ ] AC-8: LSM denial mapped to `-EACCES`/`-EPERM`.
- [ ] AC-9: `i_atime` of link not advanced.

### Architecture

```
struct LgetxattrArgs {
    path: UserPtr<u8>,
    name: UserPtr<u8>,
    value: UserPtrMut<u8>,
    size: usize,
}
```

`Xattr::sys_lgetxattr(args) -> isize`:

1. `let path = Path::user_path_at(AT_FDCWD, args.path, 0 /* no follow */)?;`
2. `return Xattr::path_getxattr(&path, args.name, args.value, args.size);`

`Xattr::path_getxattr` is shared with `getxattr(2)`; the only divergence is the lookup-flags argument to `user_path_at` at the syscall entry. Internal flow:

1. `let kname = NameBuf::copy_from_user(name, XATTR_NAME_MAX + 1)?;`
2. `if kname.len() == 0 || kname.len() > XATTR_NAME_MAX { return -ERANGE; }`
3. `if size > XATTR_SIZE_MAX { return -E2BIG; }`
4. `let kvalue = if size > 0 { Some(KernBuf::kvzalloc(size, GFP_KERNEL)?) } else { None };`
5. `let r = Xattr::vfs_getxattr(path.idmap(), path.dentry, &kname, kvalue.as_deref_mut(), size);`
6. `if r > 0 && size > 0 { copy_to_user(value, kvalue.as_ref().unwrap(), r as usize)?; }`
7. `if let Some(buf) = kvalue { buf.zero_on_drop(); }`
8. `path_put(path); return r;`

### Out of Scope

- `getxattr(2)` — separate Tier-5 (`getxattr.md`).
- `fgetxattr(2)` — separate Tier-5 (`fgetxattr.md`).
- `getxattrat(2)` — separate Tier-5 (`getxattrat.md`).
- POSIX ACL semantics — covered in `fs/posix_acl.md` Tier-3 if expanded.
- LSM handler internals — covered in `security/00-overview.md`.
- Filesystem-specific symlink xattr storage — covered in per-fs Tier-3.
- Implementation code.

### signature

C (POSIX-ish / man-pages):

```c
ssize_t lgetxattr(const char *path, const char *name,
                  void *value, size_t size);
```

glibc wrapper: `__lgetxattr` → `INLINE_SYSCALL(lgetxattr, 4, path, name, value, size)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE4(lgetxattr,
                const char __user *, pathname,
                const char __user *, name,
                void        __user *, value,
                size_t,               size);
```

Rookery dispatch:

```rust
pub fn sys_lgetxattr(
    path: UserPtr<u8>,
    name: UserPtr<u8>,
    value: UserPtrMut<u8>,
    size: usize,
) -> SyscallResult<isize>;
```

### parameters

Identical to `getxattr(2)`. The only behavioral difference is symlink handling at path resolution time.

| name    | type                   | constraints                                                                                       | errno-on-bad           |
|---------|------------------------|---------------------------------------------------------------------------------------------------|------------------------|
| path    | `const char __user *`  | NUL-terminated; length `< PATH_MAX`. Terminal symlink **not** followed.                           | `EFAULT` / `ENAMETOOLONG` / `ENOENT` |
| name    | `const char __user *`  | NUL-terminated; `1 ≤ strlen(name) ≤ XATTR_NAME_MAX`. Known namespace prefix required.             | `EFAULT` / `ERANGE` / `ENOTSUP` |
| value   | `void __user *`        | Writable for `size` bytes; may be `NULL` when `size == 0`.                                         | `EFAULT`               |
| size    | `size_t`               | `0 ≤ size ≤ XATTR_SIZE_MAX`. `size == 0` ⟹ probe.                                                  | `ERANGE`               |

### return value

- Success: number of bytes copied (or required, in probe mode).
- Failure: `< 0` — negated errno.

### errors

| errno          | condition                                                                                       |
|----------------|-------------------------------------------------------------------------------------------------|
| `EFAULT`       | User pointer crosses kernel boundary illegally.                                                 |
| `ENAMETOOLONG` | `path` `≥ PATH_MAX`, or `name` `> XATTR_NAME_MAX`.                                              |
| `ENOENT`       | `path` does not exist.                                                                          |
| `EACCES`       | Search permission denied along non-terminal `path` components.                                  |
| `ELOOP`        | Non-terminal symlink chain exceeded `MAXSYMLINKS` (terminal link itself is fine — that's the point). |
| `ENOTDIR`      | Non-terminal component is not a directory.                                                      |
| `ENODATA`      | Attribute does not exist on the link's inode.                                                   |
| `ERANGE`       | `size` is non-zero and smaller than the actual value length. Buffer untouched.                  |
| `ENOTSUP`      | Filesystem does not support xattrs on symlinks, or namespace prefix is restricted for symlinks. |
| `EPERM`        | LSM denial.                                                                                     |

### abi surface (constants + flags)

Identical to `getxattr(2)`. Notable symlink-specific behavior:

- VFS `xattr_permission` rejects `user.*` reads on inodes that are neither regular files nor directories with `-EPERM` on some fs; many fs (ext4/xfs) restrict `user.*` to regular + dir only, so `user.*` on symlink ⟹ `-EPERM`/`-ENODATA`.
- `security.*` and `trusted.*` are routinely populated on symlinks (SELinux, IMA, capability inheritance).
- `system.posix_acl_*` is undefined for symlinks (no ACLs on links); returns `-ENOTSUP`.

### compatibility contract

- REQ-1: Argument lowering identical to `getxattr`: `%rdi=path`, `%rsi=name`, `%rdx=value`, `%r10=size`.
- REQ-2: `user_path_at(AT_FDCWD, path, 0 /* no LOOKUP_FOLLOW */, &p)` — terminal symlink retained.
- REQ-3: Non-terminal symlinks (intermediate path components) are still followed per normal lookup semantics; only the **terminal** symlink is preserved.
- REQ-4: `strncpy_from_user(kname, name, XATTR_NAME_MAX + 1)`; lengths `> XATTR_NAME_MAX` ⟹ `-ERANGE`; length `0` ⟹ `-ERANGE`.
- REQ-5: `size > XATTR_SIZE_MAX ⟹ -E2BIG`.
- REQ-6: `kvalue` allocation only when `size > 0`; zero-on-drop discipline.
- REQ-7: LSM hook `security_inode_getxattr(dentry, name)` runs before fs handler; `dentry` is the symlink's dentry.
- REQ-8: Namespace dispatch: filesystem handlers may reject `user.*` on symlinks via `xattr_permission` returning `-EPERM`.
- REQ-9: Truncation rejection: actual > size > 0 ⟹ `-ERANGE`; buffer untouched.
- REQ-10: `path_put(p)` on every exit path.
- REQ-11: `atime` not updated.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `nofollow_terminal` | INVARIANT | `lgetxattr` does not pass `LOOKUP_FOLLOW`; terminal symlink retained. |
| `name_length_bounded` | INVARIANT | `1 ≤ len(name) ≤ XATTR_NAME_MAX`. |
| `value_size_bounded` | INVARIANT | `0 ≤ size ≤ XATTR_SIZE_MAX`. |
| `truncation_rejected` | INVARIANT | Actual > size > 0 ⟹ `-ERANGE`; buffer untouched. |
| `nonterminal_links_followed` | INVARIANT | Intermediate symlinks still followed during path walk. |

### Layer 2: TLA+

`uapi/syscalls/lgetxattr.tla`:
- Per-call → validate → no-follow lookup → LSM → handler → optional copy_to_user → path_put.
- Properties:
  - `safety_nofollow_terminal`,
  - `safety_truncation_rejected`,
  - `safety_namespace_partitioned`,
  - `liveness_lgetxattr_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post: dentry resolved without following terminal link | `Xattr::sys_lgetxattr` |
| Post: `size == 0` ⟹ no allocation, returns required length | `Xattr::path_getxattr` |
| Post: success with `size > 0` ⟹ first `ret` bytes copied | `Xattr::path_getxattr` |
| Post: `-ERANGE` ⟹ user buffer unchanged | `Xattr::path_getxattr` |

### Layer 4: Verus/Creusot functional

`lgetxattr(path, name, value, size)` ≡ Linux lgetxattr(2) per `man 2 lgetxattr`; identical to `getxattr` except for terminal-symlink handling.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`lgetxattr(2)` reinforcement:

- **Per-no-follow terminal symlink** — defense against TOCTOU swap of link target between resolution and read.
- **Per-`XATTR_NAME_MAX` cap** — defense against slab exhaustion via long names.
- **Per-`XATTR_SIZE_MAX` cap** — defense against unbounded kernel buffer allocation.
- **Per-truncation `-ERANGE`** — defense against silent label/capability truncation.
- **Per-namespace handler dispatch** — defense against caller-controlled handler selection.
- **Per-`user.*` on symlinks rejected** — defense against namespace pollution.
- **Per-LSM hook before fs handler** — defense against bypass of policy.
- **Per-no `atime` update** — defense against existence side-channel.

### grsecurity/pax-style reinforcement

- **PAX_UDEREF** — `path`, `name`, `value` SMAP-guarded in `getname`/`copy_to_user`; sizes bound-checked.
- **GRKERNSEC_TRUSTED** — only root reads `trusted.*` on links; unprivileged readers receive `-ENODATA` to avoid existence-leak.
- **GRKERNSEC_SECURITY_XATTR** — `security.*` reads on links LSM-gated; SELinux/Smack denial silenced from unprivileged callers.
- **GRKERNSEC_SYSTEM_XATTR** — `system.*` on links rejected by fs handlers (POSIX ACL semantics do not apply to links).
- **GRKERNSEC_LINK** — symlink-handling hardened: read of an attacker-owned link in a protected directory still gated by the link-protection policy.
- **GRKERNSEC_FIFO** — applies analogously to FIFO xattr reads in sticky dirs.
- **GRKERNSEC_PROC** — proc-restrict applies to any `/proc`-routed symlinks (`/proc/<pid>/exe`, `/proc/<pid>/fd/N`).
- **PAX_REFCOUNT** — `struct path` and dentry refcounts saturating; UAF prevention on repeated probe-then-fetch.
- **GRKERNSEC_CHROOT_FCHDIR** — chroot'd `lgetxattr` cannot escape through a symlink to pre-chroot tree (the link is read, not followed, but the link's path components must still respect chroot).
- **GRKERNSEC_HIDESYM** — LSM denial printks redact pointers and labels.
- **PAX_RANDKSTACK** — kstack offset randomized at syscall entry.
- **PAX_MEMORY_SANITIZE** — kvalue buffer zeroed on free; symlinks frequently hold `security.capability` which must not leak through slab reuse.
- **GRKERNSEC_AUDIT_XATTR** — every `security.*` and `trusted.*` read on a symlink logged with caller pid/uid/exe and the link's inode.
- **PAX_USERCOPY** — `copy_to_user(value, kvalue, ret)` bound-checked, hardened against misaligned userspace.

