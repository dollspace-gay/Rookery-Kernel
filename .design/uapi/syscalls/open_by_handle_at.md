# Tier-5 syscall: open_by_handle_at(2) — syscall 304

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/fhandle.c (SYSCALL_DEFINE3(open_by_handle_at), do_handle_to_path)
  - include/linux/exportfs.h (struct export_operations, fh_to_dentry)
  - arch/x86/entry/syscalls/syscall_64.tbl (304  common  open_by_handle_at)
-->

## Summary

`open_by_handle_at(2)` resolves a file handle previously obtained from `name_to_handle_at(2)` into an open file descriptor. It is the inverse of `name_to_handle_at` — together they form the userspace `exportfs` interface. Because handles deliberately bypass path-based security and can refer to inodes the caller could not otherwise reach (e.g. via /proc, via a parent directory the caller has lost), this syscall requires `CAP_DAC_READ_SEARCH` in the user namespace owning the mount.

Critical for: NFS server, fanotify with FID, container-based snapshotting, file-content-addressed caches.

## Signature

```c
int open_by_handle_at(int mount_fd, struct file_handle *handle, int flags);
```

```c
/* `flags` is the same set as open(2): O_RDONLY, O_WRONLY, O_RDWR,
   O_NOFOLLOW, O_NONBLOCK, O_CLOEXEC, O_DIRECTORY, ... */
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `mount_fd` | `int` | in | fd referring to any object on the mount that originally produced the handle. `AT_FDCWD` is **not** accepted (handle is mount-scoped). |
| `handle` | `struct file_handle *` | in | Caller-supplied handle from `name_to_handle_at`. `handle_bytes` and `handle_type` must match what the kernel returned. |
| `flags` | `int` | in | `open(2)`-style flags. |

## Return value

| Value | Meaning |
|---|---|
| `>= 0` | New file descriptor. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EPERM` | Caller lacks `CAP_DAC_READ_SEARCH` in the mount's user namespace. |
| `EFAULT` | `handle` pointer faults. |
| `EBADF` | `mount_fd` invalid. |
| `EINVAL` | `handle->handle_bytes == 0`, unknown `handle_type` for the fs, or bad `flags`. |
| `ESTALE` | Handle no longer valid (inode unlinked & evicted, fs reformatted, etc.). |
| `EOPNOTSUPP` | Filesystem has no `.export_op->fh_to_dentry`. |
| `ELOOP` | `O_NOFOLLOW` and final component is a symlink. |
| `EACCES` | DAC denial on `flags`-requested access mode. |
| `ENOMEM` | Allocator failed. |
| `ENFILE` / `EMFILE` | fd table full. |

## ABI surface

```text
__NR_open_by_handle_at (x86_64) = 304
__NR_open_by_handle_at (arm64)  = 265
__NR_open_by_handle_at (riscv)  = 265
__NR_open_by_handle_at (i386)   = 342
```

## Compatibility contract

REQ-1: Syscall number is **304** on x86_64. ABI-stable since Linux 2.6.39.

REQ-2: `mount_fd` must reference an object whose mount has an `.export_op->fh_to_dentry`. The mount is the resolution context.

REQ-3: **Capability gate is mandatory** — `CAP_DAC_READ_SEARCH` in the mount's user namespace. This is the *security boundary* for the entire file-handle mechanism: with this capability the caller can effectively bypass directory traversal DAC.

REQ-4: `handle->handle_bytes` must equal what `name_to_handle_at` produced; the kernel passes the entire `f_handle[]` to `fh_to_dentry`.

REQ-5: `handle->handle_type` is interpreted only by the destination fs; mismatched type ⟹ `-EINVAL`.

REQ-6: `flags` accepts the full `open(2)` flag set except `O_CREAT`, `O_TMPFILE`, `O_EXCL` (cannot create via handle). Using forbidden flags ⟹ `-EINVAL`.

REQ-7: After dentry resolved, normal `vfs_open` runs: DAC and MAC checked against the resolved inode for the requested access mode. `CAP_DAC_READ_SEARCH` only grants the *lookup*, not the open-mode override.

REQ-8: `ESTALE` returned when the inode is gone or generation mismatches; caller may retry from `name_to_handle_at` or fall back to path-based open.

REQ-9: Handle re-encoding from a different mount must fail unless filesystems share the same superblock (e.g. bind-mounted): each handle is implicitly bound to the sb.

REQ-10: O_PATH valid: callers may obtain an `O_PATH` fd without ever exercising the DAC mode check on the inode — but `CAP_DAC_READ_SEARCH` is still required.

REQ-11: `O_NOFOLLOW`: if the resolved final dentry is a symlink, returns `-ELOOP` unless `O_PATH` is also set.

## Acceptance Criteria

- [ ] AC-1: Round-trip: `name_to_handle_at(/etc/hostname)` then `open_by_handle_at(mount_fd, h, O_RDONLY)` returns valid fd with same content.
- [ ] AC-2: Without `CAP_DAC_READ_SEARCH`, returns `-EPERM`.
- [ ] AC-3: Handle from a different sb returns `-ESTALE`.
- [ ] AC-4: `flags = O_CREAT` returns `-EINVAL`.
- [ ] AC-5: Inode evicted after handle obtained but file unlinked: returns `-ESTALE`.
- [ ] AC-6: `O_PATH | O_NOFOLLOW` returns fd to symlink (no follow).
- [ ] AC-7: Filesystem without `.fh_to_dentry`: returns `-EOPNOTSUPP`.

## Architecture

```rust
#[syscall(nr = 304, abi = "sysv")]
pub fn sys_open_by_handle_at(mount_fd: i32, handle: UserPtr<FileHandle>, flags: i32) -> isize {
    OpenByHandle::do_open_by_handle_at(mount_fd, handle, flags)
}
```

`OpenByHandle::do_open_by_handle_at(mount_fd, handle_ptr, flags) -> isize`:
1. /* Forbidden flags */
2. if flags & (O_CREAT | O_TMPFILE | O_EXCL) != 0 { return -EINVAL; }
3. /* Mount resolution */
4. let mnt_file = FdTable::get(mount_fd).ok_or(EBADF)?;
5. let mnt = mnt_file.path.mnt;
6. let sb = mnt.sb;
7. let ops = sb.s_export_op.as_ref().ok_or(EOPNOTSUPP)?;
8. /* Capability */
9. Capability::check_ns(CAP_DAC_READ_SEARCH, mnt.user_ns)?;       // EPERM
10. /* Handle copy */
11. let mut header = FileHandleHeader::zeroed();
12. unsafe { handle_ptr.copy_in_partial(&mut header, size_of::<FileHandleHeader>())?; }
13. if header.handle_bytes == 0 || header.handle_bytes as usize > MAX_HANDLE_SZ {
14.   return -EINVAL;
15. }
16. let mut fh = KernelBuf::<u8>::with_capacity(header.handle_bytes as usize);
17. unsafe { handle_ptr.f_handle_ptr().copy_in(&mut fh)?; }
18. /* Decode -> dentry */
19. let dentry = ops.fh_to_dentry(sb, header.handle_type, &fh).ok_or(ESTALE)?;
20. /* Build path and open */
21. let path = Path { mnt, dentry };
22. let acc_mode = open_to_acc_mode(flags);
23. let file = Vfs::open(&path, flags, acc_mode)?;                // EACCES, ELOOP, ENOMEM
24. let oflags = if flags & O_CLOEXEC != 0 { O_CLOEXEC } else { 0 };
25. FdTable::install(file, oflags) as isize

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cap_required` | INVARIANT | !CAP_DAC_READ_SEARCH in mnt.user_ns ⟹ EPERM. |
| `forbidden_flags_rejected` | INVARIANT | O_CREAT/O_TMPFILE/O_EXCL ⟹ EINVAL. |
| `fh_to_dentry_required` | INVARIANT | no .export_op ⟹ EOPNOTSUPP. |
| `stale_handle_estale` | INVARIANT | inode gone ⟹ ESTALE (not crash, not random fd). |
| `dac_macs_still_apply` | INVARIANT | open succeeds only if DAC/MAC allow requested mode. |

### Layer 2: TLA+

`fs/open-by-handle.tla`:
- States: flag-validate, mount-fetch, cap-check, copy-handle, fh-to-dentry, vfs-open, fd-install.
- Properties:
  - `safety_cap_before_decode` — capability check precedes inode lookup.
  - `safety_dac_macs_enforced_at_open` — open access mode validated against resolved inode.
  - `safety_estale_path` — stale handle never produces fd.
  - `liveness_returns` — every call returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_open_by_handle_at` post: success ⟹ CAP_DAC_READ_SEARCH was held | `OpenByHandle::do_open_by_handle_at` |
| `fh_to_dentry` post: returns dentry whose sb matches mount.sb | `ExportOp::fh_to_dentry` |
| `vfs_open` post: file.f_mode matches access mode | `Vfs::open` |

### Layer 4: Verus / Creusot functional

Per-`open_by_handle_at(2)` man-page semantic equivalence: `name_to_handle_at` → `open_by_handle_at` round-trips for live inodes; ESTALE for evicted; identical to `open(path)` content.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`open_by_handle_at(2)` reinforcement:

- **Per-CAP_DAC_READ_SEARCH** — the *only* security gate; defense against per-path-bypass.
- **Per-forbidden-flags whitelist** — defense against per-O_CREAT abuse via handle.
- **Per-ESTALE not crash** — defense against per-handle UAF.
- **Per-DAC/MAC at open time** — defense against per-cap-too-broad: capability grants lookup, not open mode.
- **Per-O_CLOEXEC strongly recommended** — defense against per-fd-leak across exec.

## Grsecurity / PaX surface

- **PaX UDEREF on `handle` copy_from_user** — defense against per-pointer kernel-deref bug; SMAP enforced.
- **CAP_DAC_READ_SEARCH STRICT** — grsec enforces `CAP_DAC_READ_SEARCH` in **init_user_ns** specifically for `open_by_handle_at` (not just mount's user_ns) when `GRKERNSEC_NO_HANDLE_BY_NAME` is set, raising the bar for container escape via handle.
- **GRKERNSEC_NO_HANDLE_BY_NAME** — pairs with `name_to_handle_at` lockdown to prevent unprivileged callers from ever assembling a usable handle.
- **GRKERNSEC_CHROOT_FCHDIR** — chrooted process: `open_by_handle_at` is denied entirely (handles can escape chroot).
- **PAX_USERCOPY_HARDEN on f_handle[] copy_from_user** — bounded copy with handle_bytes capped at MAX_HANDLE_SZ.
- **GRKERNSEC_PROC_RESTRICT** — even with handle in hand, mount_fd must belong to caller's namespace.
- **PAX_REFCOUNT on dentry and file refcount** — defense against per-decode UAF.
- **Audit log on every successful open_by_handle_at** (mandatory) — defense against silent inode access bypassing path DAC.
- **GRKERNSEC_DMESG-style rate limit on ESTALE/EPERM** — defense against per-handle brute-force scan.
- **Per-init_user_ns CAP_DAC_READ_SEARCH for cross-userns mount** — non-init userns cannot use handles produced under host userns.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `name_to_handle_at(2)` (own Tier-5 doc).
- Per-fs `fh_to_dentry` decoding (Tier-3 per-fs docs).
- NFS server export decisions (Tier-3 nfsd doc).
- Implementation code.
