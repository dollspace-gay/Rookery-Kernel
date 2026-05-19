---
title: "Tier-5: syscall 196 — flistxattr(2)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`flistxattr(2)` is **x86_64 syscall 196**, the fd-relative extended-attribute name-enumerator. It is identical in semantics to `listxattr(2)` except the target inode is identified by an open file descriptor rather than a path: no `user_path_at`, no `LOOKUP_FOLLOW`, no symlink resolution, no per-component permission walk. The file descriptor pins the dentry and the mount, so the call is immune to rename / unmount races between two consecutive xattr operations.

The returned format is the same NUL-separated list as `listxattr`:

```
"user.foo\0user.bar\0security.selinux\0trusted.notes\0"
```

`flistxattr` is the call libcap uses to discover the present security/capability/POSIX-ACL xattrs of an already-open fd before deciding whether to set, replace, or remove them. It is also the call container-runtime label-discovery code uses on an `O_PATH` fd (which suffices because xattr permission is checked against the inode credentials, not the data-open mode).

Critical for: every libc `flistxattr`, every `getcap` on an open fd, every container labeler discovering existing labels on inherited fds, every backup tool preserving xattrs on streamed files, every Rookery xattr-vfs fd-enumeration test.

### Acceptance Criteria

- [ ] AC-1: `flistxattr(fd, buf, 4096)` returns the NUL-separated list and the byte count.
- [ ] AC-2: `flistxattr(fd, NULL, 0)` returns required length without copying.
- [ ] AC-3: `flistxattr` on inode with no xattrs returns `0`.
- [ ] AC-4: `flistxattr` with `size` smaller than total → `-ERANGE`; buffer untouched.
- [ ] AC-5: `flistxattr` on fs without xattr support → `-ENOTSUP`.
- [ ] AC-6: `O_PATH` fd: succeeds (read permission not required).
- [ ] AC-7: Unprivileged caller does not see `trusted.*` names.
- [ ] AC-8: `flistxattr` on an unlinked-but-open fd: still returns its xattrs.
- [ ] AC-9: `i_atime` not advanced.
- [ ] AC-10: `flistxattr(-1, ...)` → `-EBADF`.

### Architecture

```rust
struct FlistxattrArgs {
    fd: i32,
    list: UserPtrMut<u8>,
    size: usize,
}
```

`Xattr::sys_flistxattr(args) -> isize`:

1. `let f = file_table::fdget(args.fd).ok_or(-EBADF)?;`
2. `let r = Xattr::path_listxattr(&f.f_path, args.list, args.size);`
3. `file_table::fdput(f); return r;`

`Xattr::path_listxattr(path, list, size) -> isize`:

1. `if size > XATTR_LIST_MAX { return -E2BIG; }`
2. `let klist = if size > 0 { Some(KernBuf::kvzalloc(size, GFP_KERNEL)?) } else { None };`
3. `let r = Xattr::vfs_listxattr(path.dentry, klist.as_deref_mut(), size);`
4. `if r > 0 && size > 0 { copy_to_user(list, klist.as_ref().unwrap(), r as usize)?; }`
5. `if let Some(buf) = klist { buf.zero_on_drop(); }`
6. `return r;`

`Xattr::vfs_listxattr(dentry, list, size) -> isize`:

1. `let inode = dentry.d_inode;`
2. `let mut used: usize = 0;`
3. `if let Some(op) = inode.i_op.listxattr { used = op(dentry, list, size)?; }`
4. `else { used = Xattr::generic_listxattr(dentry, list, size)?; }`
5. `used += Security::inode_listsecurity(inode, list.offset_by(used), size - used)?;`
6. `if used > size && size > 0 { return -ERANGE; }`
7. `return used as isize;`

### signature

C (POSIX-ish / man-pages):

```c
ssize_t flistxattr(int fd, char *list, size_t size);
```

glibc wrapper: `__flistxattr` → `INLINE_SYSCALL(flistxattr, 3, fd, list, size)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE3(flistxattr,
                int,                  fd,
                char        __user *, list,
                size_t,               size);
```

Rookery dispatch:

```rust
pub fn sys_flistxattr(
    fd: i32,
    list: UserPtrMut<u8>,
    size: usize,
) -> SyscallResult<isize>;
```

### parameters

| name | type             | constraints                                                       | errno-on-bad |
|------|------------------|-------------------------------------------------------------------|--------------|
| fd   | `int`            | Open file descriptor (regular fd, `O_PATH` fd, dirfd, etc.).      | `EBADF`      |
| list | `char __user *`  | Writable for `size` bytes; may be `NULL` when `size == 0`.        | `EFAULT`     |
| size | `size_t`         | `0 ≤ size ≤ XATTR_LIST_MAX (65536)`.                              | `ERANGE` / `E2BIG` |

### return value

- Success: total number of bytes (including all interior NUL terminators) needed to hold the full list.
- `0` when the inode has no xattrs.
- Failure: `< 0` — negated errno.

### errors

| errno      | condition                                                                              |
|------------|----------------------------------------------------------------------------------------|
| `EBADF`    | `fd` is not an open file descriptor.                                                   |
| `EFAULT`   | `list` (with non-zero `size`) crosses the user/kernel boundary illegally.              |
| `ERANGE`   | `size` is non-zero and smaller than the total list length; buffer untouched.           |
| `ENOTSUP`  | Filesystem does not implement `listxattr`.                                             |
| `E2BIG`    | `size > XATTR_LIST_MAX`.                                                               |
| `EPERM`    | LSM denial on overall list enumeration.                                                |

### abi surface

- `XATTR_LIST_MAX = 65536`.
- `XATTR_NAME_MAX = 255` still applies to each name.
- LSMs filter names the caller cannot read.

The returned list is unordered. Callers must not assume:
- Stable ordering across kernel versions.
- Stable total length between probe and fetch (concurrent setters may extend).
- Sorted or deduplicated entries.

### compatibility contract

- REQ-1: Argument lowering: `%edi=fd`, `%rsi=list`, `%rdx=size`.
- REQ-2: `fdget(fd)` produces a refcounted `struct fd`; `fdput` on every exit edge.
- REQ-3: `size > XATTR_LIST_MAX ⟹ -E2BIG`.
- REQ-4: When `size > 0`, kernel allocates `klist = kvmalloc(size, GFP_KERNEL)`; on success, `copy_to_user(list, klist, ret)`; klist is zeroed before free on every path.
- REQ-5: When `size == 0`, kernel calls `vfs_listxattr(... NULL, 0)` to compute required length without allocation.
- REQ-6: `vfs_listxattr(f.f_path.dentry, klist, size)` dispatch identical to `listxattr` path.
- REQ-7: `security_inode_listsecurity` appends LSM-synthesized `security.*` names.
- REQ-8: LSM filters out names whose values the caller cannot read.
- REQ-9: Truncation: actual-size > `size > 0` ⟹ `-ERANGE`; buffer untouched.
- REQ-10: `i_atime` not updated.
- REQ-11: No `mnt_want_write_file` — read-only.
- REQ-12: An `O_PATH` fd is acceptable; xattr permission is independent of read permission.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `fd_fdget_fdput_balanced` | INVARIANT | fdget on entry ⟹ fdput on every exit. |
| `list_size_bounded` | INVARIANT | `0 ≤ size ≤ XATTR_LIST_MAX`. |
| `probe_no_copy` | INVARIANT | `size == 0` ⟹ no allocation, no `copy_to_user`. |
| `truncation_rejected` | INVARIANT | Actual total > size > 0 ⟹ `-ERANGE`; buffer untouched. |
| `each_name_nul_terminated` | INVARIANT | Every name in the output ends with a NUL. |
| `lsm_filtered_names_omitted` | INVARIANT | Names whose values the caller cannot read are not included. |
| `klist_zeroed_on_drop` | INVARIANT | Kernel list buffer zeroed before free. |

### Layer 2: TLA+

`uapi/syscalls/flistxattr.tla`:
- Per-call → fdget → vfs_listxattr (per-handler walk) → LSM-append → optional copy_to_user → fdput.
- Properties:
  - `safety_truncation_rejected`,
  - `safety_probe_no_userbuf_access`,
  - `safety_lsm_filtering_enforced`,
  - `safety_fd_refcount_balanced`,
  - `liveness_flistxattr_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post: `size == 0` ⟹ return ≥ 0 is total-length, user buffer untouched | `Xattr::path_listxattr` |
| Post: success with `size > 0` ⟹ first `ret` bytes of list copied; all names NUL-terminated | `Xattr::path_listxattr` |
| Post: `-ERANGE` ⟹ user buffer unchanged | `Xattr::path_listxattr` |
| Post: fd refcount balanced on every exit | `Xattr::sys_flistxattr` |
| Post: klist allocation zeroed on free | `Xattr::path_listxattr` |

### Layer 4: Verus/Creusot functional

`flistxattr(fd, list, size)` ≡ Linux flistxattr(2) per `man 2 flistxattr` and `Documentation/filesystems/xattr.rst`, including LSM filtering and truncation rejection.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`flistxattr(2)` reinforcement:

- **Per-`XATTR_LIST_MAX` cap** — defense against unbounded kernel buffer allocation.
- **Per-truncation `-ERANGE`** — defense against partial-list parsing UB.
- **Per-LSM filtering of unreadable names** — defense against existence side-channel.
- **Per-fd fdget/fdput balanced** — defense against fd-refcount UAF.
- **Per-no `mnt_want_write_file`** — defense against unnecessary RO writer reservations.
- **Per-no `atime` update** — defense against side-channel atime probing.
- **Per-`kvfree` zero-on-drop** — defense against slab-reuse info-leak.
- **Per-`O_PATH` accepted** — defense against double-open requirement that would broaden attack surface.

### grsecurity/pax-style reinforcement

- **PAX_UDEREF** — `list` SMAP-guarded in `copy_to_user`; output length bound-checked.
- **GRKERNSEC_TRUSTED xattr LSM-mediation** — unprivileged callers do not see `trusted.*` names at all; the kernel filters them out before assembling the list. Even existence-information is gated through the LSM mediation layer to prevent inferential side-channels.
- **GRKERNSEC_SECURITY_XATTR** — `security.*` names filtered per-LSM: SELinux/Smack omit names the caller cannot read.
- **GRKERNSEC_SYSTEM_XATTR** — `system.posix_acl_*` names visible only to callers entitled to read ACLs.
- **PAX_REFCOUNT** — fd refcounts saturating; UAF prevention on long probe-then-fetch.
- **GRKERNSEC_FORKBOMB** — flistxattr does not bypass per-uid open-fd limits.
- **GRKERNSEC_CHROOT_FHANDLE** — flistxattr on an fd obtained outside the chroot is denied when GRKERNSEC_CHROOT_FHANDLE is on.
- **GRKERNSEC_HIDESYM** — LSM denial printks redact pointers; names that would leak kernel state suppressed.
- **PAX_RANDKSTACK** — kstack offset randomized at syscall entry.
- **PAX_MEMORY_SANITIZE** — klist buffer zeroed on free; prevents info-leak of prior xattr name lists through slab reuse.
- **GRKERNSEC_AUDIT_XATTR** — flistxattr on protected fds logged with caller pid/uid/exe and inode.
- **PAX_USERCOPY** — `copy_to_user(list, klist, ret)` bound-checked.

