---
title: "Tier-5 syscall: name_to_handle_at(2) — syscall 303"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`name_to_handle_at(2)` encodes a pathname into an opaque, persistent **file handle** plus the underlying mount id. The handle can later be opened with `open_by_handle_at(2)` even after the original path has been renamed or unlinked (as long as the inode is reachable from the export). It is the userspace projection of the in-kernel `exportfs` interface used by NFS, knfsd, fanotify, and overlayfs.

Critical for: NFS server-side export, fanotify with `FAN_REPORT_FID`, container snapshotting, file-content addressed caches, and any program that needs a stable, mount-independent file id.

### Acceptance Criteria

- [ ] AC-1: `name_to_handle_at(AT_FDCWD, "/etc/hostname", h, &mid, 0)` with `h->handle_bytes >= MAX_HANDLE_SZ` returns 0 and populates handle.
- [ ] AC-2: `h->handle_bytes = 0` then call: returns `-EOVERFLOW` and `h->handle_bytes` updated to required size.
- [ ] AC-3: Unknown flag bit returns `-EINVAL`.
- [ ] AC-4: Non-exportable fs (e.g. some FUSE) returns `-EOPNOTSUPP`.
- [ ] AC-5: `AT_HANDLE_FID` returns a shorter handle compared to default mode.
- [ ] AC-6: `AT_HANDLE_MNT_ID_UNIQUE` populates a 64-bit unique mount id.
- [ ] AC-7: Subsequent `open_by_handle_at` on the handle (with `CAP_DAC_READ_SEARCH`) returns valid fd.

### Architecture

```rust
#[syscall(nr = 303, abi = "sysv")]
pub fn sys_name_to_handle_at(
    dirfd: i32, pathname: UserPtr<u8>,
    handle: UserPtr<FileHandle>, mount_id: UserPtr<i32>,
    flags: i32,
) -> isize {
    NameToHandle::do_name_to_handle_at(dirfd, pathname, handle, mount_id, flags)
}
```

`NameToHandle::do_name_to_handle_at(dirfd, pathname, handle, mid, flags) -> isize`:
1. const ACCEPTED = AT_EMPTY_PATH | AT_SYMLINK_FOLLOW | AT_HANDLE_FID | AT_HANDLE_MNT_ID_UNIQUE;
2. if flags & !ACCEPTED != 0 { return -EINVAL; }
3. let lookup = LookupFlags::from_name_to_handle(flags);
4. let path = Fs::resolve_at(dirfd, pathname, lookup)?;
5. let inode = path.dentry.d_inode;
6. let ops = inode.sb.s_export_op.as_ref().ok_or(EOPNOTSUPP)?;
7. /* Read caller capacity */
8. let mut header = FileHandleHeader::zeroed();
9. unsafe { handle.copy_in_partial(&mut header, size_of::<FileHandleHeader>())?; }
10. if header.handle_bytes == 0 { return -EINVAL; }
11. let mut kbuf = KernelBuf::<u8>::with_capacity(header.handle_bytes as usize);
12. let res = ops.encode_fh(inode, &mut kbuf, flags & AT_HANDLE_FID != 0);
13. let needed = res.bytes;
14. if needed > header.handle_bytes as usize {
15.   header.handle_bytes = needed as u32;
16.   unsafe { handle.copy_out_header(&header)?; }
17.   return -EOVERFLOW;
18. }
19. header.handle_bytes = needed as u32;
20. header.handle_type = res.handle_type;
21. unsafe { handle.copy_out_header(&header)?; }
22. unsafe { handle.f_handle_ptr().copy_out(&kbuf[..needed])?; }
23. /* Mount id */
24. let mid_val = Mount::id_of(path.mnt, flags & AT_HANDLE_MNT_ID_UNIQUE != 0);
25. unsafe { mid.copy_out(&mid_val)?; }
26. 0

### Out of Scope

- `open_by_handle_at(2)` (own Tier-5 doc).
- Per-filesystem `.encode_fh` formats (Tier-3 per-fs docs).
- fanotify FID semantics (Tier-3 fanotify doc).
- Implementation code.

### signature

```c
int name_to_handle_at(int dirfd, const char *pathname,
                      struct file_handle *handle, int *mount_id,
                      int flags);
```

```c
struct file_handle {
    __u32 handle_bytes;   /* in: capacity; out: actual size used */
    int   handle_type;    /* fs-specific (FILEID_*) */
    unsigned char f_handle[];
};

#define AT_EMPTY_PATH        0x1000
#define AT_SYMLINK_FOLLOW    0x400
#define AT_HANDLE_FID        0x200   /* return FID-only handle (no inode in dst) */
#define AT_HANDLE_MNT_ID_UNIQUE 0x1   /* *mount_id holds 64-bit unique id, not legacy 32-bit */
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `dirfd` | `int` | in | Resolution base; `AT_FDCWD`, dirfd, or with `AT_EMPTY_PATH` any fd. |
| `pathname` | `const char *` | in | Path to encode. |
| `handle` | `struct file_handle *` | in/out | Caller-owned; `handle_bytes` field declares capacity. Filled on success. |
| `mount_id` | `int *` | out | Receives mount id of resolved path's mount. |
| `flags` | `int` | in | Bitmask of `AT_*`. |

### return value

| Value | Meaning |
|---|---|
| `0` | Success; `handle` and `*mount_id` populated. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EINVAL` | Unknown `flags` bit; `handle_bytes == 0`. |
| `EOVERFLOW` | `handle_bytes` too small to hold the encoded handle. Kernel updates `handle_bytes` with required size; caller may retry. |
| `EFAULT` | Pointer fault on `handle`, `mount_id`, or `pathname`. |
| `ENOENT` | Path missing. |
| `ENOTDIR` | Non-final path component is not a directory. |
| `ELOOP` | Symlink loop. |
| `EACCES` | DAC / MAC traversal denial. |
| `EOPNOTSUPP` | Filesystem does not implement `.export_op` (no encode_fh). |
| `EBADF` | `dirfd` invalid. |
| `EPERM` | Hardened policy (e.g. `fs.handle_lookup` disabled or grsec NO_HANDLE_BY_NAME). |

### abi surface

```text
__NR_name_to_handle_at (x86_64) = 303
__NR_name_to_handle_at (arm64)  = 264
__NR_name_to_handle_at (riscv)  = 264
__NR_name_to_handle_at (i386)   = 341
```

### compatibility contract

REQ-1: Syscall number is **303** on x86_64. ABI-stable since Linux 2.6.39.

REQ-2: `flags` validation: only documented `AT_*` bits accepted; others return `-EINVAL`.

REQ-3: `handle->handle_bytes` is the **capacity** declared by caller on entry and the **actual size** written on exit. EOVERFLOW returns with `handle_bytes` updated to required size (allows two-step caller pattern).

REQ-4: `handle->handle_type` is opaque to the syscall; assigned by `fs->export_op->encode_fh`. Common values: `FILEID_INO32_GEN`, `FILEID_INO32_GEN_PARENT`, `FILEID_BTRFS_*`, `FILEID_NFS_*`. Caller must not interpret.

REQ-5: `AT_EMPTY_PATH`: with `pathname = ""`, encode the inode `dirfd` directly.

REQ-6: `AT_SYMLINK_FOLLOW`: follow final symlink (default is no-follow for this syscall — unlike most `*at` syscalls).

REQ-7: `AT_HANDLE_FID` (Linux 6.5+): encode an FID-only handle suitable for `fanotify(FAN_REPORT_FID)` — no need to resolve back to inode via `open_by_handle_at`. Saves space.

REQ-8: `AT_HANDLE_MNT_ID_UNIQUE`: `*mount_id` is the 64-bit unique mount id (same as exposed by `statmount`/`listmount`), stored in two 32-bit halves at `mount_id` and `mount_id+1` — caller must declare a `__u64`. Without flag, legacy 32-bit truncated id.

REQ-9: Capability gate: the **encode** operation itself does not require CAP_DAC_READ_SEARCH (only path traversal DAC is enforced). The capability gate exists on `open_by_handle_at(2)`.

REQ-10: Per-filesystem `.export_op` is mandatory; tmpfs, ext4, xfs, btrfs, nfsd, overlayfs implement it. Filesystems lacking it return `-EOPNOTSUPP`.

REQ-11: Handles are stable across rename and unlink-while-linked, **but not across filesystem reformat**. Caller responsible for revalidation.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flag_bits_validated` | INVARIANT | unknown bit ⟹ EINVAL. |
| `overflow_updates_size` | INVARIANT | EOVERFLOW ⟹ handle_bytes set to required size. |
| `encode_fh_required` | INVARIANT | no .export_op ⟹ EOPNOTSUPP. |
| `no_kernel_leak_on_overflow` | INVARIANT | EOVERFLOW path does not write f_handle bytes. |
| `mount_id_truncation_documented` | INVARIANT | without UNIQUE flag, mid is truncated 32-bit. |

### Layer 2: TLA+

`fs/name-to-handle.tla`:
- States: flag-validate, resolve, encode-fh, check-capacity, copy-out.
- Properties:
  - `safety_no_partial_handle` — EOVERFLOW ⟹ no handle bytes written.
  - `safety_handle_type_set_iff_success` — successful path sets type.
  - `liveness_returns` — every call returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_name_to_handle_at` post: success ⟹ handle.handle_bytes ≤ caller-capacity | `NameToHandle::do_name_to_handle_at` |
| `encode_fh` post: bytes ≤ MAX_HANDLE_SZ | `ExportOp::encode_fh` |

### Layer 4: Verus / Creusot functional

Per-`name_to_handle_at(2)` man-page semantic equivalence with nfs-server export tests and fanotify FID tests.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`name_to_handle_at(2)` reinforcement:

- **Per-flag whitelist** — defense against per-extension-flag smuggling.
- **Per-EOVERFLOW no-partial-write** — defense against per-fhandle-info-leak.
- **Per-handle opaque** — caller must not interpret; defense against per-cross-fs interpretation bug.
- **Per-EOPNOTSUPP for non-exportable fs** — defense against per-FUSE-confusion.
- **Per-mount-id-truncation documented** — defense against per-cross-32/64 id collisions.

### grsecurity / pax surface

- **PaX UDEREF on `pathname`, `handle`, `mount_id` user pointers** — defense against per-pointer kernel-deref bug; SMAP enforced.
- **GRKERNSEC_NO_HANDLE_BY_NAME** — when enabled, `name_to_handle_at` returns -EPERM for callers without `CAP_DAC_READ_SEARCH`. Pairs with `open_by_handle_at` always requiring CAP_DAC_READ_SEARCH so unprivileged callers can never construct usable handles.
- **GRKERNSEC_CHROOT_FINDTASK / GRKERNSEC_CHROOT_FCHDIR** — chrooted process restricted: cannot encode handles to inodes outside its chroot subtree.
- **PAX_USERCOPY_HARDEN on `f_handle[]` copy_to_user** — bounded copy uses whitelisted slab; truncation = EOVERFLOW.
- **GRKERNSEC_HIDESYM** — handle bytes never contain kernel addresses; encoder must use only on-disk identifiers.
- **PAX_REFCOUNT on inode refcount during encode** — defense against per-inode-vanish race.
- **Per-init_user_ns required for cross-userns encode** — non-init userns cannot encode handles for inodes in host filesystems unless mounted with id-mapped support.
- **Audit log on every name_to_handle_at** (rate-limited) — defense against silent handle-harvest.
- **GRKERNSEC_DMESG-style rate limit on EOVERFLOW** — defense against per-handle-size probing.
- **GRKERNSEC_PROC_RESTRICT** — mount_id values are namespace-local, not kernel pointers.

