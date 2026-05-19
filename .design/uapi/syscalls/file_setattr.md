# Tier-5 syscall: file_setattr(2) — syscall 468

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/inode.c (SYSCALL_DEFINE5(file_setattr), do_file_setattr)
  - fs/ioctl.c (FS_IOC_FSSETXATTR — predecessor)
  - include/uapi/linux/fs.h (struct file_attr, FS_XFLAG_*)
  - include/linux/file_attr.h (vfs_fileattr_set)
  - arch/x86/entry/syscalls/syscall_64.tbl (468  common  file_setattr)
-->

## Summary

`file_setattr(2)` writes inode-level filesystem attributes (xflags, project id, extent-size hints) for a file identified by `dirfd` + `path`, without requiring the caller to open the file. It is the typed-syscall replacement for `FS_IOC_FSSETXATTR` and `FS_IOC_SETFLAGS` ioctls. Like `file_getattr(2)`, it uses the `dirfd, path, at_flags` pattern and a forward-extensible `struct file_attr` with caller-declared `usize`.

Setting attributes is privileged: immutable / append / project-quota changes require `CAP_LINUX_IMMUTABLE` (for the lock flags) and `CAP_SYS_RESOURCE` (for the project id), in addition to the file's standard owner / DAC rules.

Critical for: `chattr +i`/`+a`, `xfs_quota project`, container-builder image immutability, replacing the per-fs ioctl surface with a typed ABI.

## Signature

```c
int file_setattr(
    int dirfd,
    const char *path,
    struct file_attr *attr,
    size_t usize,
    unsigned int at_flags
);
```

(struct definitions match `file_getattr.md`)

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `dirfd` | `int` | in | Directory fd anchoring `path`. |
| `path` | `const char *` | in | Path relative to `dirfd`. |
| `attr` | `struct file_attr *` | in | The attribute values to set. |
| `usize` | `size_t` | in | Caller-declared `sizeof(*attr)`. |
| `at_flags` | `unsigned int` | in | `AT_SYMLINK_NOFOLLOW`, `AT_EMPTY_PATH`. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Success; attributes applied. |
| `-1` + `errno` | Failure; no change. |

## Errors

| errno | Trigger |
|---|---|
| `EFAULT` | `path` or `attr` invalid. |
| `EINVAL` | Unknown `at_flags` bits, `usize < FILE_ATTR_SIZE_VER0`, unknown / mutually-exclusive `fa_xflags` bits, trailing bytes (`usize > sizeof(struct file_attr)`) non-zero. |
| `EBADF` | `dirfd` not valid. |
| `ENOENT` | Path missing. |
| `EACCES` | Search permission denied. |
| `EPERM` | Missing capability for the requested flag (e.g. `CAP_LINUX_IMMUTABLE` for `FS_XFLAG_IMMUTABLE`, `CAP_SYS_RESOURCE` for `fa_projid`). |
| `EROFS` | Read-only filesystem. |
| `EOPNOTSUPP` | Filesystem lacks `fileattr_set`. |
| `EUCLEAN` | XFS / ext4 detected on-disk inconsistency in extent hints. |
| `E2BIG` | Trailing bytes past `sizeof(struct file_attr)` non-zero. |

## ABI surface

```text
__NR_file_setattr  (x86_64) = 468
__NR_file_setattr  (arm64)  = 468
__NR_file_setattr  (riscv)  = 468
__NR_file_setattr  (i386)   = 468
```

## Compatibility contract

REQ-1: Syscall number is **468** on x86_64. ABI-stable.

REQ-2: `usize` MUST be at least `FILE_ATTR_SIZE_VER0`.

REQ-3: If `usize > sizeof(struct file_attr)`, kernel reads `sizeof(struct file_attr)` bytes and verifies the trailing bytes are all zero; non-zero ⟹ `-E2BIG`. This is the extension-probe convention.

REQ-4: `attr->fa_xflags` MUST contain only known bits and MUST NOT contain mutually-exclusive combinations (e.g. `FS_XFLAG_REALTIME` with `FS_XFLAG_RTINHERIT` for non-XFS filesystems).

REQ-5: Capability gating per-flag (typical):
- `FS_XFLAG_IMMUTABLE`, `FS_XFLAG_APPEND`: `CAP_LINUX_IMMUTABLE`.
- `fa_projid`, `FS_XFLAG_PROJINHERIT`: `CAP_SYS_RESOURCE`.
- `FS_XFLAG_DAX`: `CAP_LINUX_IMMUTABLE` plus filesystem support.
- Other flags: the file's owner or `CAP_FOWNER`.

REQ-6: The kernel reads existing attrs first (vfs_fileattr_get), then applies the caller's diff to the writable subset, then calls `inode->i_op->fileattr_set` under inode lock.

REQ-7: Filesystems may reject flag changes that are illegal on populated inodes (e.g. setting `FS_XFLAG_DAX` on a file that already has data); error returned per-fs.

REQ-8: Read-only filesystems return `-EROFS` before any state change.

REQ-9: Extent-size hints (`fa_extsize`, `fa_cowextsize`) are clamped per-fs to legal ranges; out-of-range ⟹ `-EINVAL` (kernel does not silently clamp).

REQ-10: `fa_nextents` is read-only at the ABI level; setting a non-zero value MAY return `-EINVAL`. Kernel currently ignores incoming `fa_nextents`.

REQ-11: Changes are atomic at the inode level: success ⟹ all bits applied; failure ⟹ none.

REQ-12: ctime is updated on success; mtime is NOT.

REQ-13: Audit subsystem records every successful `file_setattr` call when `CONFIG_AUDIT_FS` is on.

REQ-14: `AT_SYMLINK_NOFOLLOW` operates on the symlink itself; many filesystems forbid setting flags on symlinks and return `-EPERM`.

REQ-15: This syscall is a SUPERSET of `FS_IOC_FSSETXATTR` and `FS_IOC_SETFLAGS`.

## Acceptance Criteria

- [ ] AC-1: Caller with `CAP_LINUX_IMMUTABLE` sets `FS_XFLAG_IMMUTABLE`: returns 0; `file_getattr` confirms.
- [ ] AC-2: Caller without `CAP_LINUX_IMMUTABLE` setting immutable: `-EPERM`.
- [ ] AC-3: `at_flags = 0x4` (unknown): `-EINVAL`.
- [ ] AC-4: `usize = 8`: `-EINVAL`.
- [ ] AC-5: `usize = sizeof(file_attr) + 4` with non-zero tail: `-E2BIG`.
- [ ] AC-6: `usize = sizeof(file_attr) + 4` with zero tail: succeeds.
- [ ] AC-7: `fa_xflags = 0x80000000` (unknown bit): `-EINVAL`.
- [ ] AC-8: Read-only fs: `-EROFS`.
- [ ] AC-9: tmpfs: `-EOPNOTSUPP`.
- [ ] AC-10: Setting `fa_projid` without `CAP_SYS_RESOURCE`: `-EPERM`.
- [ ] AC-11: Out-of-range `fa_extsize`: `-EINVAL`.
- [ ] AC-12: Successful call updates ctime, not mtime.

## Architecture

```rust
#[syscall(nr = 468, abi = "sysv")]
pub fn sys_file_setattr(
    dirfd: i32,
    path: UserPtr<u8>,
    attr_p: UserPtr<u8>,
    usize: usize,
    at_flags: u32,
) -> isize {
    FileAttr::set(dirfd, path, attr_p, usize, at_flags)
}
```

`FileAttr::set(dirfd, path_p, attr_p, usize, at_flags) -> isize`:
1. if (at_flags & !(AT_SYMLINK_NOFOLLOW | AT_EMPTY_PATH)) != 0 { return Err(EINVAL); }
2. const MIN: usize = FILE_ATTR_SIZE_VER0;
3. const MAX: usize = size_of::<FileAttr>();
4. if usize < MIN { return Err(EINVAL); }
5. let mut kattr: FileAttr = FileAttr::zeroed();
6. let copy_n = usize.min(MAX);
7. attr_p.copy_in(&mut kattr, copy_n)?;                          // EFAULT
8. if usize > MAX {
9.     let mut probe = [0u8; 64];
10.    let mut off = MAX;
11.    while off < usize {
12.       let n = (usize - off).min(64);
13.       attr_p.add(off).copy_in_partial(&mut probe[..n])?;
14.       if probe[..n].iter().any(|b| *b != 0) { return Err(E2BIG); }
15.       off += n;
16.    }
17. }
18. FileAttr::validate(&kattr)?;                                  // EINVAL
19. let path = path_p.read_cstr_max(PATH_MAX)?;
20. let target = Path::resolve(current, dirfd, &path, at_flags)?;
21. let inode = target.inode();
22. FileAttr::check_capabilities(&kattr, &inode)?;               // EPERM
23. let setattr_op = inode.fileattr_set_op().ok_or(EOPNOTSUPP)?;
24. inode.with_write_lock(|| setattr_op(&inode, &kattr))?;       // EROFS / EUCLEAN
25. audit::file_setattr(&target, &kattr);
26. Ok(0)

`FileAttr::validate(kattr) -> Result<()>`:
1. const KNOWN_XFLAGS: u64 = FS_XFLAG_IMMUTABLE | FS_XFLAG_APPEND | FS_XFLAG_SYNC
2.     | FS_XFLAG_NOATIME | FS_XFLAG_NODUMP | FS_XFLAG_PROJINHERIT
3.     | FS_XFLAG_DAX | FS_XFLAG_COWEXTSIZE | /* ... */;
4. if kattr.fa_xflags & !KNOWN_XFLAGS != 0 { return Err(EINVAL); }
5. /* Mutually exclusive bits */
6. let excl_pairs = [(FS_XFLAG_REALTIME, FS_XFLAG_DAX), /* ... */];
7. for (a, b) in excl_pairs {
8.    if kattr.fa_xflags & a != 0 && kattr.fa_xflags & b != 0 { return Err(EINVAL); }
9. }
10. Ok(())

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `at_flags_validated` | INVARIANT | unknown bits ⟹ `-EINVAL`. |
| `usize_min` | INVARIANT | `usize < FILE_ATTR_SIZE_VER0` ⟹ `-EINVAL`. |
| `tail_zero_check` | INVARIANT | `usize > MAX` with non-zero tail ⟹ `-E2BIG`. |
| `xflag_allow_list` | INVARIANT | unknown `fa_xflags` bits ⟹ `-EINVAL`. |
| `cap_check_before_op` | INVARIANT | EPERM before any inode mutation. |
| `atomic_inode_update` | INVARIANT | success ⟹ all bits applied; failure ⟹ none. |

### Layer 2: TLA+

`fs/file-setattr.tla`:
- Per-call transitions: parse, validate, path-resolve, cap-check, op-dispatch.
- Properties:
  - `safety_cap_before_change` — capability denied ⟹ no on-disk state mutated.
  - `safety_atomic` — partial application impossible.
  - `safety_tail_zero` — `-E2BIG` for non-zero tail.
  - `safety_audit_on_success` — successful set always audited.
  - `liveness_terminates` — call returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `FileAttr::set` post: success ⟹ inode flags updated | `FileAttr::set` |
| `FileAttr::validate` post: rejects unknown / exclusive xflags | `FileAttr::validate` |
| `FileAttr::check_capabilities` post: EPERM on missing cap | `FileAttr::check_capabilities` |
| `FileAttr::set` post: ctime updated on success | `FileAttr::set` |

### Layer 4: Verus / Creusot functional

Per-`file_setattr(2)` man page; XFS `xfs_fs_fileattr_set` / ext4 `ext4_fileattr_set` semantic equivalence; matches FS_IOC_FSSETXATTR + FS_IOC_SETFLAGS.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`file_setattr(2)` reinforcement:

- **Per-`at_flags` allow-list** — defense against per-future-flag smuggle.
- **Per-`fa_xflags` allow-list** — defense against per-flag smuggle.
- **Per-`usize` extension-probe (E2BIG on non-zero tail)** — defense against per-extension-field smuggling.
- **Per-capability gating per-flag** — defense against per-privilege escalation via mass-flag change.
- **Per-atomic inode update** — defense against per-partial-state corruption.
- **Per-`-EROFS` precheck** — defense against per-state-change-on-readonly-fs.

## Grsecurity / PaX surface

- **PaX UDEREF on `path` and `attr`** — defense against per-pointer smuggling; SMAP forced for copy_from_user.
- **file_setattr capability gating** — `GRKERNSEC_FILEATTR_PRIV` makes capability checks STRICT:
  - `FS_XFLAG_IMMUTABLE` / `FS_XFLAG_APPEND` require `CAP_LINUX_IMMUTABLE` in **init userns** (not just current userns); defense against per-userns escape of write-lock bypass.
  - `fa_projid` change requires `CAP_SYS_RESOURCE` in init userns plus `GRKERNSEC_PROJID_RESTRICT` (project-id change must be within caller's quota).
  - `FS_XFLAG_DAX` requires `CAP_SYS_ADMIN` in init userns (DAX exposes timing side channels).
- **Per-`at_flags` allow-list** — defense against per-future-flag smuggle.
- **PAX_USERCOPY on `struct file_attr` copy_from_user** — bounded copy uses whitelisted slab.
- **Per-`E2BIG` on non-zero tail** — defense against per-extension-field smuggling.
- **PAX_REFCOUNT on `inode` during write-lock** — defense against per-fs-eject UAF.
- **GRKERNSEC_AUDIT_MOUNT mandatory** — every successful setattr audited (cannot be silenced).
- **Per-symlink-target `EPERM`** — `GRKERNSEC_FILEATTR_SYMLINK_DENY` rejects setattr on symlinks entirely; defense against per-symlink-confusion escalation.
- **GRKERNSEC_CHROOT** — path resolved within caller's chroot; defense against per-chroot escape.
- **Per-`fa_nextents` strict zero on input** — defense against per-future-field smuggling; non-zero rejected.
- **Per-flag mutually-exclusive enforcement** — defense against per-fs UB.
- **Per-`fa_projid` ID-space restricted** — `GRKERNSEC_PROJID_RANGE` clamps caller-supplied projid to caller's allowed range when not init-userns root.
- **PaX KERNEXEC on `fileattr_set` callsite** — defense against per-W^X violation via fs-driver function pointer overwrite.
- **GRKERNSEC_HIDESYM on inode flags echoed back** — internal pointer cookies never leak via the per-fs op return path.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `file_getattr(2)` (see `file_getattr.md`).
- Per-filesystem `fileattr_set` implementation (Tier-3 `fs/xfs/`, `fs/ext4/`).
- Legacy ioctls FS_IOC_FSSETXATTR / FS_IOC_SETFLAGS (Tier-4 `fs/ioctl.md`).
- Implementation code.
