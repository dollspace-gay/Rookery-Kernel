---
title: "Tier-5: syscall 193 — fgetxattr(2)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`fgetxattr(2)` is **x86_64 syscall 193**, the fd-relative extended-attribute reader. It is the read-side mirror of `fsetxattr(2)`: the target inode is identified by an open file descriptor rather than a path. No path walk, no symlink decision, no race against unmount — the file's `struct file` pins the mount and dentry for the duration of the call.

Common use cases:

- Reading `security.capability` on an `O_PATH`-opened executable in an installer that already has the fd.
- Reading SELinux labels on a fd inherited from a parent process (no need to re-traverse the path namespace).
- Container runtimes inspecting xattrs on fds handed across `SCM_RIGHTS`.
- `fanotify` / `inotify` consumers that have an fd to the event target and need xattr metadata.

Internally `fgetxattr` lowers to `getxattr(idmap, dentry, name, value, size)` with `dentry = f.f_path.dentry` and `idmap = file_mnt_idmap(f)`. All policy machinery — `vfs_getxattr`, LSM hook `security_inode_getxattr`, namespace handler dispatch, name cap, size cap, truncation `-ERANGE`, probe semantics — is identical to `getxattr(2)`. The fd refcount guarantees lifetimes; the call cannot race with `umount`.

Notable: `O_PATH` fds are perfectly valid here — xattr read permission is decoupled from data read permission, so `fgetxattr` on an `O_PATH` fd reads xattrs even though `read(2)` would fail with `-EBADF`.

Critical for: every libc `fgetxattr`, every `getcap` reading an open-fd binary, every container inspector iterating `O_PATH` fds, every Rookery xattr-vfs fd-read test.

### Acceptance Criteria

- [ ] AC-1: `fgetxattr(fd, "user.foo", buf, 64)` returns count and `buf` contains value.
- [ ] AC-2: `fgetxattr(fd, "user.missing", buf, 64)` → `-ENODATA`.
- [ ] AC-3: `fgetxattr(-1, ...)` → `-EBADF`.
- [ ] AC-4: `fgetxattr(O_PATH_fd, "security.selinux", buf, 64)` succeeds.
- [ ] AC-5: `fgetxattr(fd, "noprefix", ...)` → `-ENOTSUP`.
- [ ] AC-6: `fgetxattr` probe mode returns length without copying.
- [ ] AC-7: `fgetxattr` truncation → `-ERANGE`; buffer untouched.
- [ ] AC-8: LSM denial mapped to `-EACCES`/`-EPERM`.
- [ ] AC-9: `i_atime` not advanced.
- [ ] AC-10: `fdput` paired with every `fdget`.

### Architecture

```
struct FgetxattrArgs {
    fd: i32,
    name: UserPtr<u8>,
    value: UserPtrMut<u8>,
    size: usize,
}
```

`Xattr::sys_fgetxattr(args) -> isize`:

1. `let f = Fdget::take(args.fd)?;`
2. `let dentry = f.f_path.dentry; let idmap = file_mnt_idmap(&f);`
3. `let kname = NameBuf::copy_from_user(args.name, XATTR_NAME_MAX + 1)?;`
4. `if kname.len() == 0 || kname.len() > XATTR_NAME_MAX { return -ERANGE; }`
5. `if args.size > XATTR_SIZE_MAX { return -E2BIG; }`
6. `let kvalue = if args.size > 0 { Some(KernBuf::kvzalloc(args.size, GFP_KERNEL)?) } else { None };`
7. `let r = Xattr::vfs_getxattr(idmap, dentry, &kname, kvalue.as_deref_mut(), args.size);`
8. `if r > 0 && args.size > 0 { copy_to_user(args.value, kvalue.as_ref().unwrap(), r as usize)?; }`
9. `if let Some(buf) = kvalue { buf.zero_on_drop(); }`
10. `Fdget::put(f); return r;`

`Xattr::vfs_getxattr` is shared with the path-relative variants:

1. `Security::inode_getxattr(dentry, name)?;`
2. `if name.starts_with("security.") { if let Some(r) = Security::xattr_getsecurity(idmap, dentry.d_inode, &name[9..], value, size) { return r; } }`
3. `let handler = xattr_resolve_name(inode.i_sb.s_xattr, name)?;`
4. `let r = handler.get(handler, dentry, inode, name, value, size);`
5. `return r;`

### Out of Scope

- `getxattr(2)` — separate Tier-5 (`getxattr.md`).
- `lgetxattr(2)` — separate Tier-5 (`lgetxattr.md`).
- `getxattrat(2)` — separate Tier-5 (`getxattrat.md`).
- POSIX ACL semantics — covered in `fs/posix_acl.md` Tier-3 if expanded.
- LSM handler internals — covered in `security/00-overview.md`.
- Filesystem-specific xattr storage — covered in per-fs Tier-3.
- Implementation code.

### signature

C (POSIX-ish / man-pages):

```c
ssize_t fgetxattr(int fd, const char *name,
                  void *value, size_t size);
```

glibc wrapper: `__fgetxattr` → `INLINE_SYSCALL(fgetxattr, 4, fd, name, value, size)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE4(fgetxattr,
                int,                  fd,
                const char __user *, name,
                void        __user *, value,
                size_t,               size);
```

Rookery dispatch:

```rust
pub fn sys_fgetxattr(
    fd: i32,
    name: UserPtr<u8>,
    value: UserPtrMut<u8>,
    size: usize,
) -> SyscallResult<isize>;
```

### parameters

| name    | type                  | constraints                                                                                       | errno-on-bad           |
|---------|-----------------------|---------------------------------------------------------------------------------------------------|------------------------|
| fd      | `int`                 | Open file descriptor in the caller's `fdtable`.                                                   | `EBADF`                |
| name    | `const char __user *` | NUL-terminated; `1 ≤ strlen(name) ≤ XATTR_NAME_MAX`. Known namespace prefix required.             | `EFAULT` / `ERANGE` / `ENOTSUP` |
| value   | `void __user *`       | Writable for `size` bytes; `NULL` when `size == 0`.                                                | `EFAULT`               |
| size    | `size_t`              | `0 ≤ size ≤ XATTR_SIZE_MAX`. `size == 0` ⟹ probe.                                                  | `ERANGE`               |

### return value

- Success: number of bytes copied (or required, in probe mode).
- Failure: `< 0` — negated errno.

### errors

| errno          | condition                                                                                       |
|----------------|-------------------------------------------------------------------------------------------------|
| `EBADF`        | `fd` is not a valid open file descriptor.                                                       |
| `EFAULT`       | `name` or `value` (with non-zero `size`) crosses the user/kernel boundary illegally.            |
| `ENAMETOOLONG` | `name` length `> XATTR_NAME_MAX`.                                                               |
| `ENODATA`      | Attribute does not exist (POSIX alias: `ENOATTR`).                                              |
| `ERANGE`       | `size` is non-zero and smaller than the actual value length. Buffer untouched.                  |
| `ENOTSUP`      | Filesystem does not support xattrs, or namespace prefix unrecognized.                           |
| `EPERM`        | LSM denial, or attribute requires a privilege the caller lacks.                                 |
| `EACCES`       | Permission check on inode denied.                                                               |
| `EINVAL`       | Name is zero-length.                                                                            |

### abi surface

Identical to `getxattr(2)`. The fd-relative entry has no `LOOKUP_*` flag and no `AT_*` flag space; symlink-vs-target is decided at `open(2)` time, not at `fgetxattr` time:

- `open(path, O_RDONLY)` follows links → fd to target → `fgetxattr` reads target's xattrs.
- `open(path, O_PATH | O_NOFOLLOW)` does not follow terminal link → fd to link → `fgetxattr` reads link's xattrs (equivalent to `lgetxattr`).
- `openat2` with `RESOLVE_NO_SYMLINKS` similarly determines the link/target binding.

### compatibility contract

- REQ-1: Argument lowering: `%rdi=fd`, `%rsi=name`, `%rdx=value`, `%r10=size`.
- REQ-2: `fdget(fd)` acquires reference; `fdput(f)` on every exit path.
- REQ-3: `strncpy_from_user(kname, name, XATTR_NAME_MAX + 1)`; lengths `> XATTR_NAME_MAX` ⟹ `-ERANGE`; length `0` ⟹ `-ERANGE`.
- REQ-4: `size > XATTR_SIZE_MAX ⟹ -E2BIG`.
- REQ-5: `kvalue` allocation only when `size > 0`; zero-on-drop discipline.
- REQ-6: LSM hook `security_inode_getxattr(f.f_path.dentry, name)` before fs handler.
- REQ-7: Namespace dispatch identical to `getxattr`:
  - `user.*`, `trusted.*`, `security.*`, `system.*` resolved through `xattr_resolve_name`.
- REQ-8: Idmap threaded via `file_mnt_idmap(f)`.
- REQ-9: `O_PATH` fd is accepted — `fgetxattr` is decoupled from data read permission.
- REQ-10: Truncation rejection: actual > size > 0 ⟹ `-ERANGE`; buffer untouched.
- REQ-11: No `mnt_want_write` (read-only operation).
- REQ-12: `i_atime` not updated.
- REQ-13: Audit record emitted for `security.*` reads.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `fdget_fdput_paired` | INVARIANT | Every `fdget` paired with `fdput`. |
| `name_length_bounded` | INVARIANT | `1 ≤ len(name) ≤ XATTR_NAME_MAX`. |
| `value_size_bounded` | INVARIANT | `0 ≤ size ≤ XATTR_SIZE_MAX`. |
| `truncation_rejected` | INVARIANT | Actual > size > 0 ⟹ `-ERANGE`; buffer untouched. |
| `o_path_accepted` | INVARIANT | `fgetxattr` does not require `FMODE_READ`. |
| `kvalue_zeroed_on_drop` | INVARIANT | Kernel value buffer zeroed before free. |

### Layer 2: TLA+

`uapi/syscalls/fgetxattr.tla`:
- Per-call → fdget → validate → LSM → handler → optional copy_to_user → fdput.
- Properties:
  - `safety_fdget_fdput_paired`,
  - `safety_truncation_rejected`,
  - `safety_namespace_partitioned`,
  - `liveness_fgetxattr_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post: `size == 0` ⟹ no allocation, returns required length | `Xattr::sys_fgetxattr` |
| Post: success with `size > 0` ⟹ first `ret` bytes copied | `Xattr::sys_fgetxattr` |
| Post: `-ERANGE` ⟹ user buffer unchanged | `Xattr::sys_fgetxattr` |
| Post: kvalue allocation zeroed on free | `Xattr::sys_fgetxattr` |
| Post: `O_PATH` fd succeeds for xattr read | `Xattr::sys_fgetxattr` |

### Layer 4: Verus/Creusot functional

`fgetxattr(fd, name, value, size)` ≡ Linux fgetxattr(2) per `man 2 fgetxattr`; identical to `getxattr` except inode is identified by fd.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`fgetxattr(2)` reinforcement:

- **Per-`fdget`/`fdput` discipline** — defense against fdtable UAF on long probe-then-fetch.
- **Per-`XATTR_NAME_MAX` cap** — defense against slab exhaustion via long names.
- **Per-`XATTR_SIZE_MAX` cap** — defense against unbounded kernel buffer allocation.
- **Per-truncation `-ERANGE`** — defense against silent label/capability truncation.
- **Per-namespace handler dispatch** — defense against caller-controlled handler selection.
- **Per-LSM hook before fs handler** — defense against bypass of policy on `security.*` reads.
- **Per-`O_PATH` decoupling** — defense against requiring excess data permission for metadata read.
- **Per-no `atime` update** — defense against existence side-channel.
- **Per-`kvfree` zero-on-drop** — defense against slab-reuse info-leak.

### grsecurity/pax-style reinforcement

- **PAX_UDEREF** — `name` and `value` SMAP-guarded; `strncpy_from_user`/`copy_to_user` bound-checked.
- **GRKERNSEC_TRUSTED** — only root sees `trusted.*` values; unprivileged fd holders receive `-ENODATA`.
- **GRKERNSEC_SECURITY_XATTR** — `security.*` reads LSM-gated; SELinux/Smack denial silenced from unprivileged callers.
- **GRKERNSEC_SYSTEM_XATTR** — `system.*` reads mediated by filesystem; no raw blob exposure.
- **GRKERNSEC_PROC** — proc-restrict applies to `/proc/<pid>/fd/N`-derived fds; non-owner cannot stat-and-fgetxattr to bypass dir-read restrictions.
- **PAX_REFCOUNT** — `struct file` refcount saturating; UAF prevention on long probe-then-fetch.
- **GRKERNSEC_LINK** / **GRKERNSEC_FIFO** — `fgetxattr` on a fd opened through a symlink/FIFO in a sticky dir inherits the open-time gating.
- **GRKERNSEC_CHROOT_FCHDIR** — fd inherited from outside chroot is gated by chroot-fd policy; xattr read also gated.
- **GRKERNSEC_HIDESYM** — LSM denial printks redact pointers and labels.
- **PAX_RANDKSTACK** — kstack offset randomized at syscall entry.
- **PAX_MEMORY_SANITIZE** — kvalue buffer zeroed on free; prevents info-leak of prior `security.capability` blobs through slab reuse.
- **GRKERNSEC_AUDIT_XATTR** — every `security.*` and `trusted.*` read on an fd logged with caller pid/uid/exe and the fd's underlying inode.
- **PAX_USERCOPY** — `copy_to_user(value, kvalue, ret)` bound-checked.

