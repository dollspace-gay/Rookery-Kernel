---
title: "Tier-5: syscall 190 — fsetxattr(2)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`fsetxattr(2)` is **x86_64 syscall 190**, the fd-relative extended-attribute setter. It is identical to `setxattr(2)` except the target inode is identified by an open file descriptor `fd` rather than a path. There is no path resolution, no symlink traversal, no `LOOKUP_*` flag, and no `mnt_want_write` race against unmount — the file's `struct file` pins the mount.

Use cases:
- Setting xattrs on a file opened with `O_TMPFILE` before linking it into the filesystem.
- Setting `security.capability` on a file just opened for write by a privileged installer (avoids opening twice).
- Setting xattrs on a file referred to by an `O_PATH` fd (read-only or trusted-namespace writes; `O_PATH` fd does not require read/write permission since xattr permission is independent of data permission).
- Setting xattrs on a fd inherited from a parent process (container init handing labeled fds to children).

Internally `fsetxattr` lowers to `setxattr(idmap, dentry, name, value, size, flags)` with `dentry = f.f_path.dentry` and `idmap = file_mnt_idmap(f)`. The fd refcount holds the mount and dentry pinned, so `mnt_want_write_file` (rather than `mnt_want_write`) is used.

Critical for: every libc `fsetxattr`, every `setcap`-on-open-fd installer, every container labeler that uses `O_PATH` + `fsetxattr`, every Rookery xattr-vfs fd test.

### Acceptance Criteria

- [ ] AC-1: `fsetxattr(fd, "user.foo", "v", 1, 0)` writes xattr to the inode behind `fd`.
- [ ] AC-2: `fsetxattr(-1, ...) → -EBADF`.
- [ ] AC-3: `fsetxattr(closed_fd, ...) → -EBADF`.
- [ ] AC-4: `fsetxattr(o_path_fd, "trusted.x", ...)` succeeds for root (`O_PATH` does not block xattr writes).
- [ ] AC-5: `fsetxattr(fd, ..., XATTR_CREATE)` on existing attribute → `-EEXIST`.
- [ ] AC-6: `fsetxattr(fd, ..., XATTR_REPLACE)` on missing attribute → `-ENODATA`.
- [ ] AC-7: Both flags set → `-EINVAL`.
- [ ] AC-8: `name` length > `XATTR_NAME_MAX` → `-ERANGE`.
- [ ] AC-9: `size` > `XATTR_SIZE_MAX` → `-E2BIG`.
- [ ] AC-10: Unprivileged `trusted.*` → `-EPERM`.
- [ ] AC-11: RO mount → `-EROFS`.
- [ ] AC-12: Immutable inode → `-EPERM`.
- [ ] AC-13: On success: `i_ctime` advanced; `fsnotify_xattr` emitted.
- [ ] AC-14: `O_TMPFILE` file + `fsetxattr` + `linkat`: xattr persists after link.

### Architecture

```
struct FsetxattrArgs {
    fd: i32,
    name: UserPtr<u8>,
    value: UserPtr<u8>,
    size: usize,
    flags: i32,
}
```

`Xattr::sys_fsetxattr(args) -> i32`:

1. `if args.flags & !(XATTR_CREATE | XATTR_REPLACE) != 0 { return -EINVAL; }`
2. `if args.flags == (XATTR_CREATE | XATTR_REPLACE) { return -EINVAL; }`
3. `let f = Fd::fdget(args.fd)?;`  // -EBADF if absent
4. `let kname = NameBuf::copy_from_user(args.name, XATTR_NAME_MAX + 1)?;`
5. `if kname.len() == 0 || kname.len() > XATTR_NAME_MAX { fdput(f); return -ERANGE; }`
6. `if args.size > XATTR_SIZE_MAX { fdput(f); return -E2BIG; }`
7. `let kvalue = if args.size > 0 { Some(UserBuf::memdup_from_user(args.value, args.size)?) } else { None };`
8. `mnt_want_write_file(&f)?;`
9. `let r = loop { let mut deleg = None; let r = Xattr::setxattr(file_mnt_idmap(&f), f.dentry(), &kname, kvalue.as_deref(), args.size, args.flags, &mut deleg); if r == -EWOULDBLOCK { break_deleg_wait(&mut deleg)?; continue; } break r; };`
10. `mnt_drop_write_file(&f);`
11. `fdput(f); return r;`

`Xattr::setxattr` body is the shared helper from `setxattr.md` (LSM hook → namespace gate → handler dispatch → post-update).

### Out of Scope

- `setxattr(2)` — separate Tier-5 (`setxattr.md`).
- `lsetxattr(2)` — separate Tier-5 (`lsetxattr.md`).
- `setxattrat(2)` — separate Tier-5 (`setxattrat.md`).
- `O_PATH` semantics — covered in `open.md` Tier-5.
- LSM handler internals — covered in `security/00-overview.md`.
- Filesystem-specific xattr storage — covered in per-fs Tier-3.
- Implementation code.

### signature

C (POSIX-ish / man-pages):

```c
int fsetxattr(int fd, const char *name,
              const void *value, size_t size, int flags);
```

glibc wrapper: `__fsetxattr` → `INLINE_SYSCALL(fsetxattr, 5, fd, name, value, size, flags)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE5(fsetxattr,
                int,                 fd,
                const char __user *, name,
                const void __user *, value,
                size_t,              size,
                int,                 flags);
```

Rookery dispatch:

```rust
pub fn sys_fsetxattr(
    fd: i32,
    name: UserPtr<u8>,
    value: UserPtr<u8>,
    size: usize,
    flags: i32,
) -> SyscallResult<i32>;
```

### parameters

| name    | type                  | constraints                                                                                                | errno-on-bad           |
|---------|-----------------------|------------------------------------------------------------------------------------------------------------|------------------------|
| fd      | `int`                 | Open file descriptor; any open mode acceptable, including `O_PATH`.                                        | `EBADF`                |
| name    | `const char __user *` | NUL-terminated; `1 ≤ strlen(name) ≤ XATTR_NAME_MAX (255)`; must begin with a known namespace prefix.        | `EFAULT` / `ERANGE` / `ENOTSUP` |
| value   | `const void __user *` | Readable for `size` bytes; may be `NULL` only if `size == 0`.                                              | `EFAULT`               |
| size    | `size_t`              | `0 ≤ size ≤ XATTR_SIZE_MAX (65536)`.                                                                       | `E2BIG`                |
| flags   | `int`                 | `0`, `XATTR_CREATE (0x1)`, or `XATTR_REPLACE (0x2)`; never both.                                            | `EINVAL`               |

### return value

- Success: `0`.
- Failure: `< 0` — negated errno.

### errors

Same as `setxattr(2)` plus:

| errno          | condition                                                                                       |
|----------------|-------------------------------------------------------------------------------------------------|
| `EBADF`        | `fd` is not open.                                                                               |
| All of `setxattr(2)` non-path-related | `EFAULT`, `EINVAL`, `EEXIST`, `ENODATA`, `ENOTSUP`, `EPERM`, `E2BIG`, `EROFS`, `EDQUOT`, `ENOSPC`, `ERANGE`, `EACCES`. |

Note: no `ENOENT`, no `ENOTDIR`, no `ELOOP`, no `ENAMETOOLONG`-from-path (those are path-resolution errors). `ERANGE` still applies to the `name` argument.

### abi surface (constants + flags)

Identical to `setxattr(2)`: `XATTR_CREATE`, `XATTR_REPLACE`, `XATTR_NAME_MAX = 255`, `XATTR_SIZE_MAX = 65536`, namespace prefixes `user.`/`trusted.`/`security.`/`system.`.

`fd` may be `O_PATH` — xattr permission is checked separately from data read/write permission (in fact, `security.capability` is commonly written to `O_PATH | O_NOFOLLOW` fds to avoid following symlinks).

### compatibility contract

- REQ-1: Argument lowering: `%rdi=fd`, `%rsi=name`, `%rdx=value`, `%r10=size`, `%r8=flags`.
- REQ-2: `flags` validation identical to `setxattr(2)`: `flags & ~(XATTR_CREATE | XATTR_REPLACE) ⟹ -EINVAL`; mutually exclusive.
- REQ-3: `f = fdget(fd)` — `EBADF` if not open.
- REQ-4: Use `dentry = f.f_path.dentry`, `mnt = f.f_path.mnt`, `inode = file_inode(f)`, `idmap = file_mnt_idmap(f)`.
- REQ-5: `mnt_want_write_file(f)` taken; `mnt_drop_write_file(f)` on every exit path.
- REQ-6: Name and value copy-in identical to `setxattr(2)`: `strncpy_from_user(kname, name, XATTR_NAME_MAX + 1)`, `memdup_user(value, size)`, with same size/length caps.
- REQ-7: LSM hook `security_inode_setxattr(idmap, dentry, name, kvalue, size, flags)` runs before the fs handler.
- REQ-8: `__vfs_setxattr_locked(idmap, dentry, name, kvalue, size, flags, &delegated_inode)` under `inode_lock`.
- REQ-9: Namespace dispatch and capability gates identical to `setxattr(2)`.
- REQ-10: Immutable / append-only inode ⟹ `-EPERM` (note: `O_APPEND` data writes are still allowed; xattr writes are stricter).
- REQ-11: `delegated_inode` break-loop applies (NFSv4).
- REQ-12: On success: `fsnotify_xattr(dentry)`, `inode_inc_iversion`, `i_ctime` updated.
- REQ-13: `O_PATH` fd is acceptable; xattr writes do not require `FMODE_WRITE` on the file (LSM and capability gates still apply).
- REQ-14: `fdput(f)` on exit; refcount strict.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `fd_validated` | INVARIANT | `fdget` succeeded; refcount get/put balanced. |
| `flags_strict` | INVARIANT | Any bit outside `{XATTR_CREATE, XATTR_REPLACE}` ⟹ `-EINVAL`; mutually exclusive. |
| `name_length_bounded` | INVARIANT | `1 ≤ len(name) ≤ XATTR_NAME_MAX`. |
| `value_size_bounded` | INVARIANT | `0 ≤ size ≤ XATTR_SIZE_MAX`. |
| `mnt_write_file_paired` | INVARIANT | Every `mnt_want_write_file` paired with `mnt_drop_write_file`. |
| `o_path_permits_xattr` | INVARIANT | `O_PATH` fd is acceptable; LSM and CAP_* gates still authoritative. |

### Layer 2: TLA+

`uapi/syscalls/fsetxattr.tla`:
- Per-call → validate → fdget → mnt_want_write_file → LSM → handler → fsnotify → mnt_drop_write_file → fdput.
- Properties:
  - `safety_fd_refcount_balanced`,
  - `safety_flags_validated`,
  - `safety_namespace_capability_gated`,
  - `liveness_fsetxattr_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post: success ⟹ xattr installed on `file_inode(f)` | `Xattr::vfs_setxattr_locked` |
| Post: any error ⟹ inode unchanged | `Xattr::vfs_setxattr_locked` |
| Post: success ⟹ `i_ctime` advanced; `fsnotify_xattr` emitted | `Xattr::vfs_setxattr_locked` |
| Post: `fdget` ref get/put balanced regardless of exit path | `Xattr::sys_fsetxattr` |

### Layer 4: Verus/Creusot functional

`fsetxattr(fd, name, value, size, flags)` ≡ Linux fsetxattr(2) per `man 2 fsetxattr`: identical to `setxattr(2)` modulo target-by-fd vs target-by-path.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening; inherits all `setxattr(2)` hardening.)

`fsetxattr(2)` reinforcement:

- **Per-`fdget` refcount discipline** — defense against UAF / fd-reuse during xattr handler dispatch.
- **Per-`mnt_want_write_file` discipline** — defense against unmount-races (the fd pins the mount, but the freeze state can change).
- **Per-`O_PATH` accepted** — defense against capability-laundering by requiring `FMODE_WRITE`: the kernel uses LSM + CAP_* rather than data permission to gate xattr writes, which is the documented and audited mechanism.
- **Per-immutable / append-only `-EPERM`** — defense against xattr-laundering bypass.
- **Per-`delegated_inode` break_deleg** — defense against NFSv4 inconsistent state.

### grsecurity/pax-style reinforcement

- **PAX_UDEREF** — `name` and `value` SMAP-guarded; copy bounded by validated sizes.
- **GRKERNSEC_TRUSTED** — `trusted.*` writes via fd restricted to root or `CAP_SYS_ADMIN` over the namespace owning the mount.
- **GRKERNSEC_SECURITY_XATTR** — `security.*` writes LSM-gated; `security.capability` via `fsetxattr` requires `CAP_SETFCAP`. The `O_PATH | O_NOFOLLOW` + `fsetxattr` idiom used by `setcap(8)` is preserved.
- **GRKERNSEC_SYSTEM_XATTR** — `system.*` writes mediated by filesystem ACL handler.
- **GRKERNSEC_PROC** — listings of xattrs on `/proc/<pid>/fd/*` restricted to owner/root.
- **PAX_REFCOUNT** — `struct file` refcount saturating; `fdget`/`fdput` ladder strict.
- **GRKERNSEC_LINK** — even if the symlink reached via the `fd`'s `O_PATH` came from a tagged sticky directory, the symlink-protection ruling still applies.
- **GRKERNSEC_FIFO** — FIFO + xattr write gated identically to FIFO data write.
- **GRKERNSEC_CHROOT_FCHDIR** — chroot'd process cannot use a pre-chroot `fd` to escape; the fd-owning inode's mount must still be reachable in the chroot.
- **GRKERNSEC_HIDESYM** — LSM denial printks redact kernel pointers.
- **PAX_RANDKSTACK** — kstack offset randomized at syscall entry.
- **PAX_MEMORY_SANITIZE** — kvalue buffer zeroed on free.
- **GRKERNSEC_AUDIT_XATTR** — every `security.capability`, `trusted.*`, and `security.*` write via fd logged with caller pid/uid/exe and inode identity.

