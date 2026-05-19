# Tier-5: syscall 194 — listxattr(2)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - arch/x86/entry/syscalls/syscall_64.tbl (`194  common  listxattr  sys_listxattr`)
  - fs/xattr.c (`SYSCALL_DEFINE3(listxattr, ...)`, `path_listxattr`, `listxattr`, `vfs_listxattr`)
  - fs/namei.c (`user_path_at`, `LOOKUP_FOLLOW`)
  - include/uapi/linux/xattr.h (`XATTR_LIST_MAX`)
  - security/security.c (`security_inode_listsecurity`)
-->

## Summary

`listxattr(2)` is **x86_64 syscall 194**, the path-relative extended-attribute name-enumerator. It returns the concatenated, NUL-separated list of xattr names attached to the inode identified by `path` (terminal symlinks **followed**). The output format is a sequence of NUL-terminated strings packed end-to-end into the caller's `list` buffer:

```
"user.foo\0user.bar\0security.selinux\0trusted.notes\0"
```

The total byte count of this packed list (including all interior NULs) is the return value. Callers iterate by scanning for NUL bytes between successive name starts. The function distinguishes from `read(2)`-style byte streams in two essential ways:

- **Probe-then-fetch.** `listxattr(path, NULL, 0)` returns the total byte length without copying; callers allocate exactly that much and call again.
- **Truncation rejection.** If the provided `size` is non-zero and smaller than the actual list, the kernel returns `-ERANGE` rather than producing a partial list. This is critical: a partial list with a truncated final name would be impossible to parse safely.

Internally `listxattr` lowers to `path_listxattr(path, list, size, LOOKUP_FOLLOW)` → `vfs_listxattr(dentry, list, size)`. The LSM hook `security_inode_listsecurity` is invoked to append `security.*` names that the fs does not store explicitly (SELinux/Smack synthesize their label names). The filesystem's `inode_operations->listxattr` (or the generic handler-walk) walks per-handler `.name`/`.list` predicates and assembles the buffer.

Critical for: every libc `listxattr`, every `getfattr` enumeration, every container labeler discovering existing labels, every backup tool preserving xattrs, every Rookery xattr-vfs enumeration test.

## Signature

C (POSIX-ish / man-pages):

```c
ssize_t listxattr(const char *path, char *list, size_t size);
```

glibc wrapper: `__listxattr` → `INLINE_SYSCALL(listxattr, 3, path, list, size)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE3(listxattr,
                const char __user *, pathname,
                char        __user *, list,
                size_t,               size);
```

Rookery dispatch:

```rust
pub fn sys_listxattr(
    path: UserPtr<u8>,
    list: UserPtrMut<u8>,
    size: usize,
) -> SyscallResult<isize>;
```

## Parameters

| name | type                  | constraints                                                                        | errno-on-bad           |
|------|-----------------------|------------------------------------------------------------------------------------|------------------------|
| path | `const char __user *` | NUL-terminated; length `< PATH_MAX`. Terminal symlink followed.                    | `EFAULT` / `ENAMETOOLONG` / `ENOENT` |
| list | `char __user *`       | Writable for `size` bytes; may be `NULL` when `size == 0` (probe mode).            | `EFAULT`               |
| size | `size_t`              | `0 ≤ size ≤ XATTR_LIST_MAX (65536)`. `size == 0` ⟹ probe.                           | `ERANGE`               |

## Return value

- Success: total number of bytes (including all interior NUL terminators) needed to hold the full list. When `size == 0` this is the required buffer length; when `size > 0` and successful, the list has been copied into `list`.
- `0` is a valid success result when the inode has no xattrs.
- Failure: `< 0` — negated errno.

## Errors

| errno          | condition                                                                                       |
|----------------|-------------------------------------------------------------------------------------------------|
| `EFAULT`       | `path` or `list` (with non-zero `size`) crosses the user/kernel boundary illegally.             |
| `ENAMETOOLONG` | `path` length `≥ PATH_MAX`.                                                                     |
| `ENOENT`       | `path` does not exist.                                                                          |
| `EACCES`       | Search permission denied along `path`, or read permission on terminal inode denied.             |
| `ELOOP`        | Symlink chain exceeded `MAXSYMLINKS`.                                                           |
| `ENOTDIR`      | Non-terminal component is not a directory.                                                      |
| `ERANGE`       | `size` is non-zero and smaller than the total list length. Buffer untouched.                    |
| `ENOTSUP`      | Filesystem does not implement `listxattr` (no `i_op->listxattr` and no handler table).          |
| `E2BIG`        | `size > XATTR_LIST_MAX`.                                                                        |
| `EPERM`        | LSM denial on overall list enumeration.                                                         |

## ABI surface

From `include/uapi/linux/xattr.h`:

- `XATTR_LIST_MAX = 65536` — maximum total list size (all names + NULs).
- Per-name length cap (`XATTR_NAME_MAX = 255`) still applies to each individual entry.
- LSMs may **filter out** names the caller is not entitled to see — the kernel never returns a name whose value the caller cannot read. (This avoids existence side-channels for `trusted.*` and protected `security.*`.)

The returned list is unordered. Callers must not assume:

- A particular name ordering (filesystem-defined, not guaranteed across remounts).
- Stability of total length between probe and fetch (concurrent writers may add or remove xattrs; the kernel may return `-ERANGE` on the fetch and the caller must retry).
- That the list is sorted, deduplicated against capabilities, or partitioned by namespace.

## Compatibility contract

- REQ-1: Argument lowering: `%rdi=path`, `%rsi=list`, `%rdx=size`.
- REQ-2: `user_path_at(AT_FDCWD, path, LOOKUP_FOLLOW, &p)` — terminal symlinks followed.
- REQ-3: `size > XATTR_LIST_MAX ⟹ -E2BIG`.
- REQ-4: When `size > 0`, kernel allocates `klist = kvmalloc(size, GFP_KERNEL)`; on success, `copy_to_user(list, klist, ret)`; klist is zeroed before free on every path.
- REQ-5: When `size == 0`, kernel does not allocate a list buffer; `vfs_listxattr(... NULL, 0)` returns the required length.
- REQ-6: `vfs_listxattr(dentry, klist, size)` dispatches:
  - If `i_op->listxattr` exists, call it directly.
  - Else walk `inode->i_sb->s_xattr` handler table; for each handler `h`, if `h->list(dentry)` returns truthy, append `h->name` (NUL-terminated) to the output.
- REQ-7: `security_inode_listsecurity(inode, klist, size)` appends LSM-synthesized `security.*` names (e.g. `security.selinux`).
- REQ-8: LSM may **filter**: handlers and synthesized names that the caller cannot read are omitted (`xattr_permission` check per-name).
- REQ-9: On actual-size > `size > 0`: `-ERANGE`; buffer untouched.
- REQ-10: `path_put(p)` on every exit path.
- REQ-11: `atime` not updated.
- REQ-12: No `mnt_want_write` — read-only.

## Acceptance Criteria

- [ ] AC-1: `listxattr("f", buf, 4096)` returns the NUL-separated list and the byte count.
- [ ] AC-2: `listxattr("f", NULL, 0)` returns required length without copying.
- [ ] AC-3: `listxattr` on inode with no xattrs returns `0`.
- [ ] AC-4: `listxattr` with `size` smaller than total → `-ERANGE`; buffer untouched.
- [ ] AC-5: `listxattr` on fs without xattr support → `-ENOTSUP`.
- [ ] AC-6: Terminal symlink followed (list is from target).
- [ ] AC-7: Unprivileged caller does not see `trusted.*` names.
- [ ] AC-8: LSM-filtered `security.*` names omitted for callers without read permission.
- [ ] AC-9: `i_atime` not advanced.
- [ ] AC-10: Each entry in the returned list is NUL-terminated; total bytes include all interior NULs.

## Architecture

```
struct ListxattrArgs {
    path: UserPtr<u8>,
    list: UserPtrMut<u8>,
    size: usize,
}
```

`Xattr::sys_listxattr(args) -> isize`:

1. `let path = Path::user_path_at(AT_FDCWD, args.path, LOOKUP_FOLLOW)?;`
2. `return Xattr::path_listxattr(&path, args.list, args.size);`

`Xattr::path_listxattr(path, list, size) -> isize`:

1. `if size > XATTR_LIST_MAX { return -E2BIG; }`
2. `let klist = if size > 0 { Some(KernBuf::kvzalloc(size, GFP_KERNEL)?) } else { None };`
3. `let r = Xattr::vfs_listxattr(path.dentry, klist.as_deref_mut(), size);`
4. `if r > 0 && size > 0 { copy_to_user(list, klist.as_ref().unwrap(), r as usize)?; }`
5. `if let Some(buf) = klist { buf.zero_on_drop(); }`
6. `path_put(path); return r;`

`Xattr::vfs_listxattr(dentry, list, size) -> isize`:

1. `let inode = dentry.d_inode;`
2. `let mut used: usize = 0;`
3. `if let Some(op) = inode.i_op.listxattr { used = op(dentry, list, size)?; }`
4. `else { used = Xattr::generic_listxattr(dentry, list, size)?; }`
5. /* LSM-synthesized security.* names */
6. `used += Security::inode_listsecurity(inode, list.offset_by(used), size - used)?;`
7. `if used > size && size > 0 { return -ERANGE; }`
8. `return used as isize;`

`Xattr::generic_listxattr(dentry, list, size) -> isize`:

1. For each handler `h` in `inode.i_sb.s_xattr`:
   - if `h.list != NULL && !h.list(dentry) { continue; }`
   - `name_len = strlen(h.name) + 1; if used + name_len > size && size > 0 { return -ERANGE; }`
   - `if size > 0 { memcpy(list + used, h.name, name_len); }`
   - `used += name_len;`

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `list_size_bounded` | INVARIANT | `0 ≤ size ≤ XATTR_LIST_MAX`. |
| `probe_no_copy` | INVARIANT | `size == 0` ⟹ no allocation, no `copy_to_user`. |
| `truncation_rejected` | INVARIANT | Actual total > size > 0 ⟹ `-ERANGE`; buffer untouched. |
| `each_name_nul_terminated` | INVARIANT | Every name in the output ends with a NUL. |
| `lsm_filtered_names_omitted` | INVARIANT | Names whose values the caller cannot read are not included. |
| `klist_zeroed_on_drop` | INVARIANT | Kernel list buffer zeroed before free. |

### Layer 2: TLA+

`uapi/syscalls/listxattr.tla`:
- Per-call → validate → lookup → vfs_listxattr (per-handler walk) → LSM-append → optional copy_to_user → path_put.
- Properties:
  - `safety_truncation_rejected`,
  - `safety_probe_no_userbuf_access`,
  - `safety_lsm_filtering_enforced`,
  - `liveness_listxattr_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post: `size == 0` ⟹ return ≥ 0 is total-length, user buffer untouched | `Xattr::path_listxattr` |
| Post: success with `size > 0` ⟹ first `ret` bytes of list copied; all names NUL-terminated | `Xattr::path_listxattr` |
| Post: `-ERANGE` ⟹ user buffer unchanged | `Xattr::path_listxattr` |
| Post: LSM-filtered names absent from output | `Xattr::vfs_listxattr` |
| Post: klist allocation zeroed on free | `Xattr::path_listxattr` |

### Layer 4: Verus/Creusot functional

`listxattr(path, list, size)` ≡ Linux listxattr(2) per `man 2 listxattr` and `Documentation/filesystems/xattr.rst`, including LSM filtering and truncation rejection.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`listxattr(2)` reinforcement:

- **Per-`XATTR_LIST_MAX` cap** — defense against unbounded kernel buffer allocation.
- **Per-truncation `-ERANGE`** — defense against partial-list parsing UB.
- **Per-LSM filtering of unreadable names** — defense against existence side-channel.
- **Per-handler `.list(dentry)` predicate** — defense against namespace-prefix leak for handlers that conditionally apply.
- **Per-no `mnt_want_write`** — defense against unnecessary RO-mount writer reservations.
- **Per-no `atime` update** — defense against side-channel atime probing.
- **Per-`kvfree` zero-on-drop** — defense against slab-reuse info-leak.
- **Per-probe-mode zero-alloc** — defense against denial-of-service via repeated zero-size listxattrs.

## Grsecurity/PaX-style Reinforcement

- **PAX_UDEREF** — `path` and `list` SMAP-guarded in `getname`/`copy_to_user`; output length bound-checked.
- **GRKERNSEC_TRUSTED** — unprivileged callers do not see `trusted.*` names at all; the kernel filters them out before assembling the list. Without this, the mere presence of `trusted.x` could leak information about a privileged labeling step.
- **GRKERNSEC_SECURITY_XATTR** — `security.*` names filtered per-LSM: SELinux/Smack omit names the caller cannot read.
- **GRKERNSEC_SYSTEM_XATTR** — `system.posix_acl_*` names visible only to callers entitled to read ACLs.
- **GRKERNSEC_PROC** — proc-restrict applies to `/proc`-routed paths: `/proc/<pid>/exe` and similar links filter xattr names per-owner.
- **PAX_REFCOUNT** — `struct path` refcounts saturating; UAF prevention on long probe-then-fetch.
- **GRKERNSEC_LINK** — symlink-followed list-resolution in protected dirs gated against attacker-owned link races.
- **GRKERNSEC_FIFO** — FIFO list reads in sticky dirs gated identically to data reads.
- **GRKERNSEC_CHROOT_FCHDIR** — chroot'd `listxattr` cannot traverse pre-chroot paths.
- **GRKERNSEC_HIDESYM** — LSM denial printks redact pointers; names that would leak kernel state suppressed.
- **PAX_RANDKSTACK** — kstack offset randomized at syscall entry.
- **PAX_MEMORY_SANITIZE** — klist buffer zeroed on free; prevents info-leak of prior xattr name lists through slab reuse.
- **GRKERNSEC_AUDIT_XATTR** — listxattr on protected directories logged with caller pid/uid/exe and inode.
- **PAX_USERCOPY** — `copy_to_user(list, klist, ret)` bound-checked; hardened against misaligned userspace.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `llistxattr(2)` — separate Tier-5 (`llistxattr.md`) if expanded.
- `flistxattr(2)` — separate Tier-5 (`flistxattr.md`) if expanded.
- `listxattrat(2)` — separate Tier-5 (`listxattrat.md`) if expanded.
- POSIX ACL list semantics — covered in `fs/posix_acl.md` Tier-3.
- LSM `inode_listsecurity` internals — covered in `security/00-overview.md`.
- Filesystem-specific xattr name storage — covered in per-fs Tier-3.
- Implementation code.
