---
title: "Tier-5: syscall 332 — statx(2)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`statx(2)` is **x86_64 syscall 332**, the modern extended file-status retrieval interface introduced in Linux 4.11. It supersedes `stat(2)`, `lstat(2)`, and `fstatat(2)` by exposing a versioned, mask-driven `struct statx` containing nanosecond-resolution timestamps, btime (creation time), DOS-style file attributes, mount ID, filesystem ID, and inode generation alongside the classical fields. The mask (`STATX_*`) tells the kernel which fields the caller wants — the kernel returns at least those, plus whatever else it can supply cheaply, and reports back through `stx_mask` which fields were actually populated. The `flags` argument carries `AT_*` directory-resolution semantics plus a synchronization hint (`AT_STATX_SYNC_*`) for network filesystems, and `AT_EMPTY_PATH` enables fstat-via-fd lookup against `dirfd`.

Critical for: every libc `statx` and modern stat backend, every Rookery filesystem capability probe, every `coreutils`/`stat`/`ls` invocation built against glibc ≥ 2.28, every container image-build cache invalidation, every backup tool that needs btime.

### Acceptance Criteria

- [ ] AC-1: `statx(AT_FDCWD, "/etc/passwd", 0, STATX_BASIC_STATS, &buf)` returns `0` and `buf.stx_mask & STATX_BASIC_STATS == STATX_BASIC_STATS`.
- [ ] AC-2: `statx(fd, "", AT_EMPTY_PATH, STATX_INO, &buf) == 0` with valid `stx_ino`.
- [ ] AC-3: `statx(.., 0, STATX__RESERVED, &buf) == -EINVAL`.
- [ ] AC-4: Multiple `AT_STATX_SYNC_*` bits set ⟹ `-EINVAL`.
- [ ] AC-5: `AT_SYMLINK_NOFOLLOW` returns info about a terminal symlink, not its target.
- [ ] AC-6: Unsupported `STATX_BTIME` ⟹ reply mask bit cleared, no error.
- [ ] AC-7: Buffer is zeroed for any field not selected in `mask` (no kernel-stack leak).
- [ ] AC-8: `pathname == ""` without `AT_EMPTY_PATH` ⟹ `-ENOENT`.
- [ ] AC-9: Empty path with `O_PATH` dirfd + `AT_EMPTY_PATH` ⟹ inode info for the underlying object.

### Architecture

```
struct StatxArgs { dirfd: i32, filename: UserPtr<u8>, flags: u32, mask: u32, buffer: UserPtr<Statx> }
```

`sys_statx(args) -> i32`:

1. Reject `args.mask & STATX__RESERVED` ⟹ `-EINVAL`.
2. Reject `args.flags & !VALID_STATX_AT_FLAGS` ⟹ `-EINVAL`.
3. Reject `popcount(args.flags & AT_STATX_SYNC_TYPE) > 1` ⟹ `-EINVAL`.
4. `let lookup = build_lookup_flags(args.flags);`
5. `let mut kstat = KernelStat::default();`
6. `vfs_statx(args.dirfd, args.filename, lookup, &mut kstat, args.mask)?;`
7. `cp_statx(&kstat, args.buffer)?;`
8. Return `0`.

`Fs::vfs_statx(dfd, name, lookup, kstat, mask) -> Result<(), errno>`:

1. `let path = filename_lookup(dfd, name, lookup)?;`
2. `vfs_getattr(&path, kstat, mask, query_flags_from(lookup))?;`
3. `path_put(path);`
4. `Ok(())`.

`Fs::cp_statx(kstat, ubuf) -> Result<(), errno>`:

1. `let mut tmp = Statx::zeroed();`
2. `tmp.stx_mask = kstat.result_mask;`
3. Populate selected fields from `kstat`.
4. `copy_to_user(ubuf, &tmp, size_of::<Statx>()).map_err(|_| EFAULT)`.

### Out of Scope

- `stat(2)`, `lstat(2)`, `fstatat(2)` legacy interfaces (separate Tier-5 docs if expanded).
- Filesystem-specific `getattr` internals (covered in fs/ Tier-3 docs).
- LSM hook details (`security/00-overview.md`).
- Implementation code.

### signature

C (POSIX-influenced; Linux-specific):

```c
int statx(int dirfd, const char *pathname, int flags,
          unsigned int mask, struct statx *statxbuf);
```

glibc wrapper: `statx` → `INLINE_SYSCALL(statx, 5, dirfd, pathname, flags, mask, statxbuf)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE5(statx,
                int, dfd, const char __user *, filename,
                unsigned int, flags, unsigned int, mask,
                struct statx __user *, buffer);
```

Rookery dispatch:

```rust
pub fn sys_statx(
    dirfd: i32,
    filename: UserPtr<u8>,
    flags: u32,
    mask: u32,
    buffer: UserPtr<Statx>,
) -> SyscallResult<i32>;
```

### parameters

| name     | type                    | constraints                                                              | errno-on-bad |
|----------|-------------------------|--------------------------------------------------------------------------|--------------|
| dirfd    | `int`                   | open directory fd, `AT_FDCWD`, or any fd with `AT_EMPTY_PATH`            | `EBADF` / `ENOTDIR` |
| filename | `const char __user *`   | NUL-terminated; `< PATH_MAX`; may be `""` with `AT_EMPTY_PATH`           | `EFAULT` / `ENAMETOOLONG` / `ENOENT` |
| flags    | `unsigned int`          | subset of `AT_SYMLINK_NOFOLLOW | AT_EMPTY_PATH | AT_NO_AUTOMOUNT | AT_STATX_*` | `EINVAL` |
| mask     | `unsigned int`          | subset of valid `STATX_*` bits; `STATX__RESERVED` MUST be zero          | `EINVAL` |
| buffer   | `struct statx __user *` | writable for `sizeof(struct statx) == 256` bytes; 8-byte aligned         | `EFAULT` |

### return value

- Success: `0`. `statxbuf->stx_mask` reports which fields are valid.
- Failure: `< 0` — negated errno; `*statxbuf` is left untouched on error.

### errors

| errno          | condition                                                                              |
|----------------|----------------------------------------------------------------------------------------|
| `EBADF`        | `dirfd` is not open and not `AT_FDCWD`.                                                |
| `EFAULT`       | `filename` or `buffer` points outside the caller's address space.                      |
| `EINVAL`       | Reserved bits set in `mask` or `flags`; mutually-exclusive `AT_STATX_*` sync flags.    |
| `ELOOP`        | Too many symlinks during resolution.                                                   |
| `ENAMETOOLONG` | `pathname` is too long.                                                                |
| `ENOENT`       | Component of path is missing; or empty `pathname` without `AT_EMPTY_PATH`.             |
| `ENOMEM`       | Kernel ran out of memory for the lookup.                                               |
| `ENOTDIR`      | A non-final path component is not a directory; or relative path with non-directory `dirfd` and no `AT_EMPTY_PATH`. |
| `EACCES`       | Search permission denied on a path component.                                          |
| `EOVERFLOW`    | A field cannot be represented (e.g., size > 63-bit on 32-bit reduced view — not on 64-bit). |

### abi surface (constants + flags)

`AT_*` directory-resolution flags (`include/uapi/linux/fcntl.h`):

- `AT_FDCWD = -100`.
- `AT_SYMLINK_NOFOLLOW = 0x100` — do not dereference a terminal symlink (return info about the link itself).
- `AT_NO_AUTOMOUNT = 0x800` — do not trigger automount on terminal component.
- `AT_EMPTY_PATH = 0x1000` — if `pathname == ""`, operate on `dirfd` (allows fstat-via-fd, including `O_PATH` fds; for fds the caller doesn't own, `CAP_DAC_READ_SEARCH` gates use).

Synchronization-hint flags (`include/uapi/linux/stat.h`):

- `AT_STATX_SYNC_AS_STAT = 0x0000` — same behavior as `stat(2)`: filesystem decides.
- `AT_STATX_FORCE_SYNC   = 0x2000` — force the fs to round-trip to the server / refresh inode (NFS, FUSE, ceph).
- `AT_STATX_DONT_SYNC    = 0x4000` — return cached attributes only; do not refresh.
- `AT_STATX_SYNC_TYPE    = 0x6000` — mask covering the three above; at most one bit may be set.

`STATX_*` mask bits (`stx_mask` request and reply):

- `STATX_TYPE = 0x0001`, `STATX_MODE = 0x0002`, `STATX_NLINK = 0x0004`, `STATX_UID = 0x0008`, `STATX_GID = 0x0010`, `STATX_ATIME = 0x0020`, `STATX_MTIME = 0x0040`, `STATX_CTIME = 0x0080`, `STATX_INO = 0x0100`, `STATX_SIZE = 0x0200`, `STATX_BLOCKS = 0x0400`, `STATX_BASIC_STATS = 0x07ff`.
- `STATX_BTIME = 0x0800` — creation time (best-effort; not all fs).
- `STATX_MNT_ID = 0x1000` — mount ID (since 5.8).
- `STATX_DIOALIGN = 0x2000` — direct-I/O alignment requirements (since 6.1).
- `STATX_MNT_ID_UNIQUE = 0x4000` — unique mount ID across reboots (since 6.8).
- `STATX_SUBVOL = 0x8000` — subvolume identifier (btrfs etc.; since 6.10).
- `STATX_WRITE_ATOMIC = 0x10000` — atomic-write boundary (since 6.11).
- `STATX__RESERVED` (high bits) — MUST be zero.

`stx_attributes` / `stx_attributes_mask` bits — `STATX_ATTR_COMPRESSED`, `_IMMUTABLE`, `_APPEND`, `_NODUMP`, `_ENCRYPTED`, `_AUTOMOUNT`, `_MOUNT_ROOT`, `_VERITY`, `_DAX`, `_WRITE_ATOMIC`.

`struct statx` layout (`sizeof == 256`): `stx_mask`, `stx_blksize`, `stx_attributes`, `stx_nlink`, `stx_uid`, `stx_gid`, `stx_mode`, `stx_ino`, `stx_size`, `stx_blocks`, `stx_attributes_mask`, timestamps (`stx_atime`, `stx_btime`, `stx_ctime`, `stx_mtime` — each `struct statx_timestamp { __s64 tv_sec; __u32 tv_nsec; __s32 __reserved; }`), `stx_rdev_{major,minor}`, `stx_dev_{major,minor}`, `stx_mnt_id`, `stx_dio_*`, `stx_subvol`, `stx_atomic_write_*`, plus reserved trailing space.

### compatibility contract

- REQ-1: Argument lowering: `%rdi=dirfd (i32)`, `%rsi=filename`, `%rdx=flags (u32)`, `%r10=mask (u32)`, `%r8=buffer`.
- REQ-2: `mask & STATX__RESERVED ⟹ -EINVAL` (forward-compatibility lockout).
- REQ-3: At most one `AT_STATX_SYNC_*` bit set; multiple ⟹ `-EINVAL`.
- REQ-4: `flags` outside `{AT_SYMLINK_NOFOLLOW, AT_EMPTY_PATH, AT_NO_AUTOMOUNT, AT_STATX_SYNC_TYPE}` ⟹ `-EINVAL`.
- REQ-5: Lookup flags built from `AT_*`: `LOOKUP_FOLLOW` unless `AT_SYMLINK_NOFOLLOW`; `LOOKUP_AUTOMOUNT` unless `AT_NO_AUTOMOUNT`; `LOOKUP_EMPTY` iff `AT_EMPTY_PATH`.
- REQ-6: `vfs_statx(dirfd, filename, lookup_flags, &kstat, request_mask) -> Result<()>` walks the path, then calls `vfs_getattr(&path, &kstat, request_mask, query_flags)`.
- REQ-7: `vfs_getattr` MAY call `inode->i_op->getattr` if the fs supports statx-extensions, else falls back to generic `generic_fillattr`.
- REQ-8: `cp_statx(&kstat, buffer)` zeroes the user buffer, then populates only requested fields, then `__copy_to_user(buffer, &tmp, sizeof tmp)`.
- REQ-9: `stx_mask` on reply contains the subset of requested bits the kernel populated; the kernel MAY also populate unrequested bits (and bit them into `stx_mask`).
- REQ-10: `STATX_BTIME` unsupported by an fs ⟹ bit cleared in reply, field is zero — NOT an error.
- REQ-11: With `AT_EMPTY_PATH` and `pathname == ""`: operate on the file referenced by `dirfd`, even if it is not a directory; permits `O_PATH` fds.
- REQ-12: Returned timestamps use `struct statx_timestamp` with `tv_nsec` in `[0, 1_000_000_000)`.
- REQ-13: Future ABI growth: kernel writes `sizeof(struct statx) == 256` bytes; userspace MUST allocate at least that. New fields appear at the tail.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `reserved_mask_rejected` | INVARIANT | `mask & STATX__RESERVED ⟹ -EINVAL` before any path walk. |
| `at_sync_exclusive` | INVARIANT | At most one `AT_STATX_SYNC_*` bit set. |
| `buffer_zeroed_on_success` | INVARIANT | Unrequested fields read as zero. |
| `empty_path_requires_flag` | INVARIANT | `name == ""` without `AT_EMPTY_PATH` ⟹ `-ENOENT`. |
| `no_kstack_leak` | INVARIANT | `cp_statx` writes from a stack-init'd `tmp`, never raw kernel memory. |

### Layer 2: TLA+

`uapi/syscalls/statx.tla`:
- Per-call → validate → lookup → getattr → cp_statx.
- Properties:
  - `safety_no_kstack_leak`,
  - `safety_reserved_bits_zero`,
  - `liveness_statx_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `stx_mask ⊆ requested_mask ∪ free_fields` | `Fs::cp_statx` |
| `stx_nsec ∈ [0, 1_000_000_000)` for every timestamp | `Fs::cp_statx` |
| `vfs_statx` never returns success with `kstat` partially-uninit'd | `Fs::vfs_statx` |

### Layer 4: Verus/Creusot functional

`statx(dirfd, path, flags, mask, buf)` ≡ Linux statx(2) semantics, with mask masking and AT_* lookup behavior matching `man 2 statx`.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`statx(2)` reinforcement:

- **Per-`STATX__RESERVED` rejection** — defense against silent forward-compat semantic drift.
- **Per-`AT_STATX_SYNC_TYPE` mutual-exclusion** — defense against ambiguous freshness contract.
- **Per-buffer zero-init** — defense against kernel-stack-info leak via unrequested fields.
- **Per-`AT_EMPTY_PATH` cap gate** — defense against statx-via-foreign-fd.
- **Per-`AT_NO_AUTOMOUNT`** — defense against accidental NFS automount probe leaking topology.
- **Per-`PATH_MAX` cap in `getname`** — defense against long-path DoS.

### grsecurity/pax-style reinforcement

- **PAX_UDEREF** — `filename` and `buffer` SMAP-guarded in `getname` and `copy_to_user`.
- **GRKERNSEC_CHROOT_FCHDIR** — statx on dirfd inherited pre-chroot rejected if it escapes the jail.
- **GRKERNSEC_SYMLINKOWN** — symlink-owner check applies during path resolution.
- **GRKERNSEC_LINK** — hardlink-traversal restrictions apply.
- **GRKERNSEC_FIFO** — FIFO-in-sticky-dir restrictions apply to terminal nodes.
- **AT_EMPTY_PATH / AT_SYMLINK_NOFOLLOW reject** — when grsec policy forbids cross-owner fd stat, the syscall returns `-EPERM` before any inode probe.
- **GRKERNSEC_DMESG** — statx error printks CAP_SYSLOG-gated.
- **PAX_REFCOUNT** — `path` dentry/inode refcount saturating across `filename_lookup`/`path_put`.
- **PAX_MEMORY_SANITIZE** — `KernelStat` and `Statx` stack temporaries scrubbed on return.
- **PAX_RANDKSTACK** — kstack offset randomized at statx syscall entry.
- **GRKERNSEC_HIDESYM** — error printks redact kernel pointers.
- **GRKERNSEC_AUDIT_MOUNT** — statx traversing a mount boundary is auditable.

