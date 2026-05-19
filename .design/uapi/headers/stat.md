# Tier-5 UAPI: include/uapi/linux/stat.h — file-stat ABI (mode bits + struct statx)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - include/uapi/linux/stat.h (~260 lines)
-->

## Summary

The `linux/stat.h` UAPI header defines the on-the-wire **file-stat ABI** seen by every userspace `stat(2)`, `fstat(2)`, `lstat(2)`, `fstatat(2)`, and **`statx(2)`** caller. Two distinct surfaces live in this header:

1. **POSIX mode-bit constants** — `S_IFMT` / `S_IFSOCK` / `S_IFLNK` / `S_IFREG` / `S_IFBLK` / `S_IFDIR` / `S_IFCHR` / `S_IFIFO` (file-type nibble, octal `0170000` mask) and the SUID / SGID / sticky / rwx bits (`S_ISUID`, `S_ISGID`, `S_ISVTX`, `S_IRWXU`, `S_IRWXG`, `S_IRWXO`, with per-class `R/W/X`). These bits encode `stx_mode` and the legacy `st_mode` and are pinned by glibc, musl, BSD coreutils, and POSIX.1-2017 §6.6.

2. **`struct statx`** — the 256-byte (`0x100`) extended-stat record returned by `statx(2)`. Layout is offset-fixed at `0x00 / 0x10 / 0x20 / 0x40 / 0x80 / 0x90 / 0xa0 / 0xb0 / 0xc0`, contains 4 × `struct statx_timestamp` (atime / btime / ctime / mtime), Linux-specific fields (`stx_mnt_id`, `stx_dio_mem_align`, `stx_dio_offset_align`, `stx_subvol`, `stx_atomic_write_unit_min/max/max_opt`, `stx_atomic_write_segments_max`, `stx_dio_read_offset_align`) and 64 bytes of `__spare3[8]` for future ABI expansion. Userspace consults `stx_mask` to discover which fields were actually filled; `stx_attributes` (gated by `stx_attributes_mask`) exposes per-inode flags such as `STATX_ATTR_IMMUTABLE`, `STATX_ATTR_APPEND`, `STATX_ATTR_VERITY`, `STATX_ATTR_DAX`, `STATX_ATTR_WRITE_ATOMIC`.

The header is dual-included via `#if defined(__KERNEL__) || !defined(__GLIBC__) || (__GLIBC__ < 2)` so that glibc, which defines the same `S_*` macros itself, does not collide. Critical for: ls(1), find(1), cp(1), rsync(1), libc `stat()` family, every coreutils tool, container image scanners, security-policy enforcers reading SUID/SGID, and any verity/DAX/atomic-write aware program.

This Tier-5 covers the **ABI surface** of `include/uapi/linux/stat.h` — every constant, every struct field, every offset. The semantic implementation lives in `fs/stat.c` and per-filesystem `getattr()` hooks, both covered in their own Tier-3 docs.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `S_IFMT`, `S_IFSOCK`, `S_IFLNK`, `S_IFREG`, `S_IFBLK`, `S_IFDIR`, `S_IFCHR`, `S_IFIFO` | per-file-type nibble (octal `017xxxx`) | `uapi::stat::mode` constants |
| `S_ISLNK / S_ISREG / S_ISDIR / S_ISCHR / S_ISBLK / S_ISFIFO / S_ISSOCK` | per-type predicate macros | `uapi::stat::Mode::is_*` |
| `S_ISUID`, `S_ISGID`, `S_ISVTX` | per-setid / sticky bits | `uapi::stat::mode` constants |
| `S_IRWXU / S_IRUSR / S_IWUSR / S_IXUSR` | per-owner rwx triad | `uapi::stat::mode` constants |
| `S_IRWXG / S_IRGRP / S_IWGRP / S_IXGRP` | per-group rwx triad | `uapi::stat::mode` constants |
| `S_IRWXO / S_IROTH / S_IWOTH / S_IXOTH` | per-other rwx triad | `uapi::stat::mode` constants |
| `struct statx_timestamp` | per-timestamp record | `uapi::stat::StatxTimestamp` |
| `struct statx` | per-statx(2) reply | `uapi::stat::Statx` |
| `STATX_TYPE / MODE / NLINK / UID / GID / ATIME / MTIME / CTIME / INO / SIZE / BLOCKS` | per-basic mask bit | `uapi::stat::mask` constants |
| `STATX_BASIC_STATS` | per-basic-stat composite (`0x7ff`) | `uapi::stat::mask::BASIC_STATS` |
| `STATX_BTIME` | per-creation-time bit | `uapi::stat::mask::BTIME` |
| `STATX_MNT_ID` / `STATX_MNT_ID_UNIQUE` | per-mount-id bits | `uapi::stat::mask::MNT_ID*` |
| `STATX_DIOALIGN` / `STATX_DIO_READ_ALIGN` | per-direct-IO alignment bits | `uapi::stat::mask::DIO*` |
| `STATX_SUBVOL` | per-subvolume bit | `uapi::stat::mask::SUBVOL` |
| `STATX_WRITE_ATOMIC` | per-atomic-write bit | `uapi::stat::mask::WRITE_ATOMIC` |
| `STATX_ALL` | legacy deprecated composite (`0x0fff`) | `uapi::stat::mask::ALL` (deprecated) |
| `STATX__RESERVED` | per-future-expansion guard (`0x80000000`) | `uapi::stat::mask::RESERVED` |
| `STATX_ATTR_COMPRESSED / IMMUTABLE / APPEND / NODUMP / ENCRYPTED` | per-inode attr bits | `uapi::stat::attr` constants |
| `STATX_ATTR_AUTOMOUNT / MOUNT_ROOT` | per-mount attrs | `uapi::stat::attr` constants |
| `STATX_ATTR_VERITY / DAX / WRITE_ATOMIC` | per-modern-fs attrs | `uapi::stat::attr` constants |
| `AT_STATX_SYNC_TYPE`, `AT_STATX_SYNC_AS_STAT`, `AT_STATX_FORCE_SYNC`, `AT_STATX_DONT_SYNC` | per-sync policy (from `linux/fcntl.h`) | `uapi::fcntl::at` constants |

## ABI surface (constants + structs)

### File-type nibble (`S_IFMT` mask = `0170000` octal = `0xF000`)

| Constant | Octal | Hex | Meaning |
|---|---|---|---|
| `S_IFMT`   | `0170000` | `0xF000` | per-file-type mask (selects the 4-bit type nibble of `st_mode`) |
| `S_IFSOCK` | `0140000` | `0xC000` | socket |
| `S_IFLNK`  | `0120000` | `0xA000` | symbolic link |
| `S_IFREG`  | `0100000` | `0x8000` | regular file |
| `S_IFBLK`  | `0060000` | `0x6000` | block device |
| `S_IFDIR`  | `0040000` | `0x4000` | directory |
| `S_IFCHR`  | `0020000` | `0x2000` | character device |
| `S_IFIFO`  | `0010000` | `0x1000` | FIFO / named pipe |

These values are **wire-pinned**: every coreutils binary, every NFS/CIFS/9P implementation, every container image tool encodes these exact octal constants. Anything other than the listed seven types (and the conventional masked-zero "unknown") is an ABI break.

Predicate macros: `S_ISLNK(m)`, `S_ISREG(m)`, `S_ISDIR(m)`, `S_ISCHR(m)`, `S_ISBLK(m)`, `S_ISFIFO(m)`, `S_ISSOCK(m)` — each evaluates `((m) & S_IFMT) == S_IF<X>`.

### Set-ID and sticky bits

| Constant | Octal | Hex | Meaning |
|---|---|---|---|
| `S_ISUID` | `0004000` | `0x800` | set-user-id-on-exec |
| `S_ISGID` | `0002000` | `0x400` | set-group-id-on-exec (or mandatory-locking on regular files where supported) |
| `S_ISVTX` | `0001000` | `0x200` | sticky bit (directories: restricted deletion) |

### Permission triads

Owner (`S_IRWXU` = `0700`): `S_IRUSR`=`0400`, `S_IWUSR`=`0200`, `S_IXUSR`=`0100`.
Group (`S_IRWXG` = `0070`): `S_IRGRP`=`0040`, `S_IWGRP`=`0020`, `S_IXGRP`=`0010`.
Other (`S_IRWXO` = `0007`): `S_IROTH`=`0004`, `S_IWOTH`=`0002`, `S_IXOTH`=`0001`.

The full mode value is the OR of file-type nibble | setid/sticky | three rwx triads; the low 12 bits (`07777` = `0xFFF`) are the permission-and-setid portion handled by `chmod(2)`.

### `struct statx_timestamp` (16 bytes)

```
offset 0x0  __s64 tv_sec       // signed seconds since 1970-01-01 UTC
offset 0x8  __u32 tv_nsec      // 0..999_999_999
offset 0xC  __s32 __reserved   // reserved for higher-resolution clocks
```

### `struct statx` (256 bytes, 0x100)

| Offset | Type | Field | Notes |
|---|---|---|---|
| 0x00 | `__u32` | `stx_mask` | per-statx(2): which fields the kernel actually filled (echoes input mask intersected with what's supported); always populated |
| 0x04 | `__u32` | `stx_blksize` | preferred I/O block size; always populated |
| 0x08 | `__u64` | `stx_attributes` | per-inode flags (see `STATX_ATTR_*`); always populated, but only bits listed in `stx_attributes_mask` are valid |
| 0x10 | `__u32` | `stx_nlink` | hard-link count (filled iff `STATX_NLINK`) |
| 0x14 | `__u32` | `stx_uid` | owner UID (filled iff `STATX_UID`) |
| 0x18 | `__u32` | `stx_gid` | owner GID (filled iff `STATX_GID`) |
| 0x1C | `__u16` | `stx_mode` | mode (file-type nibble + perms) (filled iff `STATX_TYPE` or `STATX_MODE`) |
| 0x1E | `__u16[1]` | `__spare0` | reserved; must be zero on write, ignored on read |
| 0x20 | `__u64` | `stx_ino` | inode number (filled iff `STATX_INO`) |
| 0x28 | `__u64` | `stx_size` | file size in bytes (filled iff `STATX_SIZE`) |
| 0x30 | `__u64` | `stx_blocks` | allocated blocks in 512-byte units (filled iff `STATX_BLOCKS`) |
| 0x38 | `__u64` | `stx_attributes_mask` | which `STATX_ATTR_*` bits are *defined* for this file's fs |
| 0x40 | `statx_timestamp` | `stx_atime` | last access (filled iff `STATX_ATIME`) |
| 0x50 | `statx_timestamp` | `stx_btime` | creation/birth (filled iff `STATX_BTIME`) |
| 0x60 | `statx_timestamp` | `stx_ctime` | last attribute change (filled iff `STATX_CTIME`) |
| 0x70 | `statx_timestamp` | `stx_mtime` | last data modification (filled iff `STATX_MTIME`) |
| 0x80 | `__u32` | `stx_rdev_major` | for bdev/cdev only; device's major |
| 0x84 | `__u32` | `stx_rdev_minor` | for bdev/cdev only; device's minor |
| 0x88 | `__u32` | `stx_dev_major` | containing-filesystem device major; always populated |
| 0x8C | `__u32` | `stx_dev_minor` | containing-filesystem device minor; always populated |
| 0x90 | `__u64` | `stx_mnt_id` | mount-id (filled iff `STATX_MNT_ID` or `STATX_MNT_ID_UNIQUE`) |
| 0x98 | `__u32` | `stx_dio_mem_align` | direct-I/O memory-buffer alignment (filled iff `STATX_DIOALIGN`) |
| 0x9C | `__u32` | `stx_dio_offset_align` | direct-I/O file-offset alignment (filled iff `STATX_DIOALIGN`) |
| 0xA0 | `__u64` | `stx_subvol` | subvolume identifier (filled iff `STATX_SUBVOL`) |
| 0xA8 | `__u32` | `stx_atomic_write_unit_min` | minimum atomic-write granularity (filled iff `STATX_WRITE_ATOMIC`) |
| 0xAC | `__u32` | `stx_atomic_write_unit_max` | maximum atomic-write granularity (filled iff `STATX_WRITE_ATOMIC`) |
| 0xB0 | `__u32` | `stx_atomic_write_segments_max` | maximum atomic-write segment count (filled iff `STATX_WRITE_ATOMIC`) |
| 0xB4 | `__u32` | `stx_dio_read_offset_align` | direct-I/O read-offset alignment (filled iff `STATX_DIO_READ_ALIGN`) |
| 0xB8 | `__u32` | `stx_atomic_write_unit_max_opt` | optimised maximum atomic-write granularity (filled iff `STATX_WRITE_ATOMIC`) |
| 0xBC | `__u32[1]` | `__spare2` | reserved |
| 0xC0 | `__u64[8]` | `__spare3` | 64 bytes reserved for future ABI extension |
| 0x100 |  |  | end of struct |

### `STATX_*` request/result mask bits

| Bit | Hex | Field guarded |
|---|---|---|
| `STATX_TYPE`            | `0x00000001` | `stx_mode & S_IFMT` |
| `STATX_MODE`            | `0x00000002` | `stx_mode & ~S_IFMT` |
| `STATX_NLINK`           | `0x00000004` | `stx_nlink` |
| `STATX_UID`             | `0x00000008` | `stx_uid` |
| `STATX_GID`             | `0x00000010` | `stx_gid` |
| `STATX_ATIME`           | `0x00000020` | `stx_atime` |
| `STATX_MTIME`           | `0x00000040` | `stx_mtime` |
| `STATX_CTIME`           | `0x00000080` | `stx_ctime` |
| `STATX_INO`             | `0x00000100` | `stx_ino` |
| `STATX_SIZE`            | `0x00000200` | `stx_size` |
| `STATX_BLOCKS`          | `0x00000400` | `stx_blocks` |
| `STATX_BASIC_STATS`     | `0x000007FF` | OR of the above (legacy POSIX stat surface) |
| `STATX_BTIME`           | `0x00000800` | `stx_btime` |
| `STATX_MNT_ID`          | `0x00001000` | `stx_mnt_id` (legacy 32-bit) |
| `STATX_DIOALIGN`        | `0x00002000` | `stx_dio_mem_align`, `stx_dio_offset_align` |
| `STATX_MNT_ID_UNIQUE`   | `0x00004000` | `stx_mnt_id` (extended/unique) |
| `STATX_SUBVOL`          | `0x00008000` | `stx_subvol` |
| `STATX_WRITE_ATOMIC`    | `0x00010000` | `stx_atomic_write_unit_*`, `stx_atomic_write_segments_max` |
| `STATX_DIO_READ_ALIGN`  | `0x00020000` | `stx_dio_read_offset_align` |
| `STATX__RESERVED`       | `0x80000000` | reserved-must-not-be-set guard for future struct expansion |
| `STATX_ALL` (userspace) | `0x00000FFF` | deprecated; prefer `STATX_BASIC_STATS | STATX_BTIME` |

### `STATX_ATTR_*` attribute bits (encoded in `stx_attributes`; validity gated by `stx_attributes_mask`)

| Bit | Hex | Meaning |
|---|---|---|
| `STATX_ATTR_COMPRESSED`  | `0x00000004` | [I] fs-level compressed |
| `STATX_ATTR_IMMUTABLE`   | `0x00000010` | [I] immutable |
| `STATX_ATTR_APPEND`      | `0x00000020` | [I] append-only |
| `STATX_ATTR_NODUMP`      | `0x00000040` | [I] don't-dump |
| `STATX_ATTR_ENCRYPTED`   | `0x00000800` | [I] encrypted (needs key) |
| `STATX_ATTR_AUTOMOUNT`   | `0x00001000` | directory: automount trigger |
| `STATX_ATTR_MOUNT_ROOT`  | `0x00002000` | root of a mount |
| `STATX_ATTR_VERITY`      | `0x00100000` | [I] fs-verity-protected |
| `STATX_ATTR_DAX`         | `0x00200000` | currently in DAX state |
| `STATX_ATTR_WRITE_ATOMIC`| `0x00400000` | supports atomic-write |

`[I]` annotations correspond to the matching `FS_IOC_SETFLAGS` bit; numeric values were deliberately chosen to match where possible.

### `AT_STATX_*` sync flags (defined in `linux/fcntl.h`, consumed by `statx(2)`)

| Constant | Hex | Meaning |
|---|---|---|
| `AT_STATX_SYNC_TYPE`     | `0x6000` | mask of sync-type bits |
| `AT_STATX_SYNC_AS_STAT`  | `0x0000` | behave like POSIX `stat()` |
| `AT_STATX_FORCE_SYNC`    | `0x2000` | force server-sync (e.g. NFS) |
| `AT_STATX_DONT_SYNC`     | `0x4000` | use cached metadata if any |

## Compatibility contract

REQ-1: Mode-bit ABI immutability:
- The 12 low bits of `stx_mode` / `st_mode` (the perm + setid + sticky bits) and the 4-bit type nibble (`S_IFMT`) MUST have identical numeric values to the C macros in `include/uapi/linux/stat.h`.
- The Rust binding crate MUST expose these as `const u16` / `const u32` with the same octal value.

REQ-2: Predicate macro semantics:
- `S_ISxxx(m)` MUST be implemented as `((m) & S_IFMT) == S_IFxxx`.
- The seven types (`SOCK`, `LNK`, `REG`, `BLK`, `DIR`, `CHR`, `FIFO`) MUST be mutually exclusive; for any valid inode, exactly one predicate returns true.

REQ-3: `struct statx_timestamp` layout:
- sizeof == 16, alignof == 8.
- offsetof(tv_sec) == 0, offsetof(tv_nsec) == 8, offsetof(__reserved) == 12.
- tv_nsec MUST be in [0, 999_999_999]; values outside this range when supplied by userspace via utimensat-style paths are an error, but the kernel MUST accept any value when *returning* metadata (filesystems with stored garbage cannot crash the caller).
- __reserved MUST be zero on return.

REQ-4: `struct statx` layout:
- sizeof == 256, alignof == 8.
- Every field offset MUST match the table above exactly (kernels and userspace pin the offsets; binaries compiled against an old layout still see correct data because each field is at a fixed offset).
- The 8 × `__u64` `__spare3[8]` at offset `0xC0` MUST be zeroed by the kernel before return.
- `__spare0` and `__spare2` MUST also be zero.

REQ-5: `stx_mask` semantics:
- On entry the caller passes a desired mask in `statx(2)`'s 4th argument.
- The kernel MUST clear `STATX__RESERVED` (`0x80000000`) — if set, the caller is buggy/futuristic; kernel returns `-EINVAL`.
- On return `stx_mask` reflects exactly which fields were successfully filled. Per-bit semantics:
  - if datum unsupported by fs: bit cleared, field set to a sane default (e.g. CIFS default uid/gid) or zero.
  - if requested AND supported: bit set, field filled, possibly forced to be sync'd per `AT_STATX_FORCE_SYNC`.
  - if NOT requested but cheaply available: bit MAY be set and the field filled (informational); kernel does no extra work.
- `STATX_BASIC_STATS` (`0x7FF`) is the legacy POSIX-equivalent set; userspace fall-back code paths reuse these bits as a unit.

REQ-6: `stx_attributes` / `stx_attributes_mask` semantics:
- `stx_attributes_mask` enumerates which `STATX_ATTR_*` bits are *meaningful* for the file's filesystem.
- For each bit B: if `(stx_attributes_mask & B) == 0`, the caller MUST ignore `(stx_attributes & B)`.
- For bits *in* the mask: 1 = the attribute is set; 0 = the attribute is not set.
- New `STATX_ATTR_*` bits MUST NOT shift values of existing ones; values are immutable.

REQ-7: `stx_dev_*` vs `stx_rdev_*`:
- `stx_dev_major / stx_dev_minor` identifies the filesystem the file is on; always populated (uncond).
- `stx_rdev_major / stx_rdev_minor` is meaningful only when `S_ISCHR(stx_mode) || S_ISBLK(stx_mode)`; otherwise both MUST be zero.

REQ-8: `stx_blksize` semantics:
- Returns the preferred I/O block size for the filesystem (often 4096, may differ for FUSE/NFS).
- Always populated (uncond), independent of any bit in `stx_mask`.

REQ-9: `stx_blocks` semantics:
- Returns allocated storage in **512-byte units** (matches legacy `st_blocks`), NOT in `stx_blksize` units. This is a Linux/POSIX historical contract — DO NOT change.

REQ-10: `stx_mnt_id` semantics:
- If only `STATX_MNT_ID` requested: legacy 32-bit-stable mount id is returned.
- If `STATX_MNT_ID_UNIQUE` requested: unique 64-bit-stable mount id (stable across remounts) is returned in the same field.
- Both bits set: the unique form wins.

REQ-11: `stx_dio_mem_align` / `stx_dio_offset_align` (`STATX_DIOALIGN`):
- Return the buffer-alignment and file-offset-alignment requirements for `O_DIRECT` writes; 0 means direct I/O is unsupported.
- `stx_dio_read_offset_align` (`STATX_DIO_READ_ALIGN`) is the *read*-specific offset alignment; may differ from write.

REQ-12: `stx_subvol` (`STATX_SUBVOL`):
- For multi-subvolume filesystems (btrfs, bcachefs): returns a per-subvolume identifier so userspace can detect subvolume crossings.
- Zero is "no subvolume / not a subvolume-aware fs".

REQ-13: `stx_atomic_*` (`STATX_WRITE_ATOMIC`):
- `stx_atomic_write_unit_min` / `_max`: bounds of guaranteed-atomic write granularity (bytes).
- `stx_atomic_write_segments_max`: max number of iovec segments per atomic write.
- `stx_atomic_write_unit_max_opt`: optimised (preferred) maximum unit; may be smaller than `_max`.
- All zero when atomic-write is unsupported.

REQ-14: Sync-type flags:
- `AT_STATX_SYNC_TYPE` mask = `0x6000`.
- Exactly one of `AT_STATX_SYNC_AS_STAT` (`0x0000`), `AT_STATX_FORCE_SYNC` (`0x2000`), `AT_STATX_DONT_SYNC` (`0x4000`) MUST be passed.
- Passing both `FORCE_SYNC` and `DONT_SYNC` (i.e. `0x6000`) MUST return `-EINVAL`.

REQ-15: glibc collision guard:
- The `S_*` mode constants are wrapped by `#if defined(__KERNEL__) || !defined(__GLIBC__) || (__GLIBC__ < 2)`.
- Rookery's userspace-facing C header MUST preserve this guard so glibc-built programs do not see double definitions.

REQ-16: Backward-compat for `statx_timestamp.__reserved`:
- Reserved field MUST remain zero in returned data and zero in any field the kernel reads.
- Future use (sub-nanosecond resolution) is allowed only if a new `STATX_*` bit is introduced to gate it.

REQ-17: ABI extension contract:
- New fields MUST be added by consuming `__spare3[8]` from the front; the struct size MUST NOT change.
- A new `STATX_*` bit MUST be introduced and added to `STATX__RESERVED`-NOT-set space (i.e. below `0x80000000`).

## Acceptance Criteria

- [ ] AC-1: `sizeof(struct statx) == 256` on every supported arch (x86_64, aarch64, riscv64, loongarch64).
- [ ] AC-2: `sizeof(struct statx_timestamp) == 16`.
- [ ] AC-3: All field offsets in `struct statx` match the layout table exactly (verified via `static_assert` in C harness and `core::mem::offset_of!` in Rust harness).
- [ ] AC-4: `S_IFMT == 0170000`; each `S_IF<type>` constant has the exact octal value listed; the seven types are pairwise distinct under `S_IFMT` mask.
- [ ] AC-5: `S_ISxxx(m)` predicates are equivalent to `((m) & S_IFMT) == S_IFxxx`.
- [ ] AC-6: `S_ISUID | S_ISGID | S_ISVTX | S_IRWXU | S_IRWXG | S_IRWXO == 07777`.
- [ ] AC-7: Round-trip: a `statx` populated by Rookery and parsed by glibc-static `ls -l` produces output byte-identical to the upstream Linux baseline for the same fs state.
- [ ] AC-8: `statx(_, _, AT_STATX_FORCE_SYNC | AT_STATX_DONT_SYNC, _, _)` returns `-EINVAL`.
- [ ] AC-9: `statx(_, _, _, STATX__RESERVED, _)` returns `-EINVAL`.
- [ ] AC-10: For a non-bdev/non-cdev inode, `stx_rdev_major == 0 && stx_rdev_minor == 0`.
- [ ] AC-11: `stx_blocks` is reported in 512-byte units (a 1 MiB regular file uses ≥ 2048 blocks).
- [ ] AC-12: `__spare0`, `__spare2`, `__spare3` are zeroed by the kernel before return.
- [ ] AC-13: `STATX_BASIC_STATS` evaluates to `0x000007FF`.
- [ ] AC-14: `STATX_ALL` (deprecated, userspace-only) evaluates to `0x00000FFF`.
- [ ] AC-15: For each `STATX_ATTR_*` bit, `(stx_attributes & B)` is valid iff `(stx_attributes_mask & B)`.

## Architecture

```
pub mod uapi::stat {

    // Mode-bit constants -------------------------------------------------
    pub const S_IFMT:   u32 = 0o170000;
    pub const S_IFSOCK: u32 = 0o140000;
    pub const S_IFLNK:  u32 = 0o120000;
    pub const S_IFREG:  u32 = 0o100000;
    pub const S_IFBLK:  u32 = 0o060000;
    pub const S_IFDIR:  u32 = 0o040000;
    pub const S_IFCHR:  u32 = 0o020000;
    pub const S_IFIFO:  u32 = 0o010000;

    pub const S_ISUID:  u32 = 0o004000;
    pub const S_ISGID:  u32 = 0o002000;
    pub const S_ISVTX:  u32 = 0o001000;

    pub const S_IRWXU:  u32 = 0o700;
    pub const S_IRUSR:  u32 = 0o400;
    pub const S_IWUSR:  u32 = 0o200;
    pub const S_IXUSR:  u32 = 0o100;
    pub const S_IRWXG:  u32 = 0o070;
    pub const S_IRGRP:  u32 = 0o040;
    pub const S_IWGRP:  u32 = 0o020;
    pub const S_IXGRP:  u32 = 0o010;
    pub const S_IRWXO:  u32 = 0o007;
    pub const S_IROTH:  u32 = 0o004;
    pub const S_IWOTH:  u32 = 0o002;
    pub const S_IXOTH:  u32 = 0o001;

    #[inline] pub const fn s_isreg (m: u32) -> bool { (m & S_IFMT) == S_IFREG  }
    #[inline] pub const fn s_isdir (m: u32) -> bool { (m & S_IFMT) == S_IFDIR  }
    #[inline] pub const fn s_islnk (m: u32) -> bool { (m & S_IFMT) == S_IFLNK  }
    #[inline] pub const fn s_ischr (m: u32) -> bool { (m & S_IFMT) == S_IFCHR  }
    #[inline] pub const fn s_isblk (m: u32) -> bool { (m & S_IFMT) == S_IFBLK  }
    #[inline] pub const fn s_isfifo(m: u32) -> bool { (m & S_IFMT) == S_IFIFO  }
    #[inline] pub const fn s_issock(m: u32) -> bool { (m & S_IFMT) == S_IFSOCK }

    // struct statx_timestamp --------------------------------------------
    #[repr(C)]
    pub struct StatxTimestamp {
        pub tv_sec:     i64,
        pub tv_nsec:    u32,
        pub __reserved: i32,
    }
    const _: () = assert!(core::mem::size_of::<StatxTimestamp>() == 16);
    const _: () = assert!(core::mem::align_of::<StatxTimestamp>() ==  8);

    // struct statx ------------------------------------------------------
    #[repr(C)]
    pub struct Statx {
        // 0x00
        pub stx_mask:               u32,
        pub stx_blksize:            u32,
        pub stx_attributes:         u64,
        // 0x10
        pub stx_nlink:              u32,
        pub stx_uid:                u32,
        pub stx_gid:                u32,
        pub stx_mode:               u16,
        pub __spare0:               [u16; 1],
        // 0x20
        pub stx_ino:                u64,
        pub stx_size:               u64,
        pub stx_blocks:             u64,
        pub stx_attributes_mask:    u64,
        // 0x40
        pub stx_atime:              StatxTimestamp,
        pub stx_btime:              StatxTimestamp,
        pub stx_ctime:              StatxTimestamp,
        pub stx_mtime:              StatxTimestamp,
        // 0x80
        pub stx_rdev_major:         u32,
        pub stx_rdev_minor:         u32,
        pub stx_dev_major:          u32,
        pub stx_dev_minor:          u32,
        // 0x90
        pub stx_mnt_id:             u64,
        pub stx_dio_mem_align:      u32,
        pub stx_dio_offset_align:   u32,
        // 0xa0
        pub stx_subvol:                       u64,
        pub stx_atomic_write_unit_min:        u32,
        pub stx_atomic_write_unit_max:        u32,
        // 0xb0
        pub stx_atomic_write_segments_max:    u32,
        pub stx_dio_read_offset_align:        u32,
        pub stx_atomic_write_unit_max_opt:    u32,
        pub __spare2:                         [u32; 1],
        // 0xc0
        pub __spare3:               [u64; 8],
        // 0x100
    }
    const _: () = assert!(core::mem::size_of::<Statx>() == 256);
    const _: () = assert!(core::mem::align_of::<Statx>() ==   8);
    const _: () = assert!(core::mem::offset_of!(Statx, stx_mask)            == 0x00);
    const _: () = assert!(core::mem::offset_of!(Statx, stx_attributes)      == 0x08);
    const _: () = assert!(core::mem::offset_of!(Statx, stx_nlink)           == 0x10);
    const _: () = assert!(core::mem::offset_of!(Statx, stx_mode)            == 0x1C);
    const _: () = assert!(core::mem::offset_of!(Statx, stx_ino)             == 0x20);
    const _: () = assert!(core::mem::offset_of!(Statx, stx_attributes_mask) == 0x38);
    const _: () = assert!(core::mem::offset_of!(Statx, stx_atime)           == 0x40);
    const _: () = assert!(core::mem::offset_of!(Statx, stx_mtime)           == 0x70);
    const _: () = assert!(core::mem::offset_of!(Statx, stx_rdev_major)      == 0x80);
    const _: () = assert!(core::mem::offset_of!(Statx, stx_dev_major)       == 0x88);
    const _: () = assert!(core::mem::offset_of!(Statx, stx_mnt_id)          == 0x90);
    const _: () = assert!(core::mem::offset_of!(Statx, stx_subvol)          == 0xA0);
    const _: () = assert!(core::mem::offset_of!(Statx, __spare3)            == 0xC0);

    // STATX_* mask bits -------------------------------------------------
    pub mod mask {
        pub const TYPE:              u32 = 0x0000_0001;
        pub const MODE:              u32 = 0x0000_0002;
        pub const NLINK:             u32 = 0x0000_0004;
        pub const UID:               u32 = 0x0000_0008;
        pub const GID:               u32 = 0x0000_0010;
        pub const ATIME:             u32 = 0x0000_0020;
        pub const MTIME:             u32 = 0x0000_0040;
        pub const CTIME:             u32 = 0x0000_0080;
        pub const INO:               u32 = 0x0000_0100;
        pub const SIZE:              u32 = 0x0000_0200;
        pub const BLOCKS:            u32 = 0x0000_0400;
        pub const BASIC_STATS:       u32 = 0x0000_07FF;
        pub const BTIME:             u32 = 0x0000_0800;
        pub const MNT_ID:            u32 = 0x0000_1000;
        pub const DIOALIGN:          u32 = 0x0000_2000;
        pub const MNT_ID_UNIQUE:     u32 = 0x0000_4000;
        pub const SUBVOL:            u32 = 0x0000_8000;
        pub const WRITE_ATOMIC:      u32 = 0x0001_0000;
        pub const DIO_READ_ALIGN:    u32 = 0x0002_0000;
        pub const RESERVED:          u32 = 0x8000_0000;
        // userspace deprecated alias
        pub const ALL:               u32 = 0x0000_0FFF;
    }

    // STATX_ATTR_* attribute bits --------------------------------------
    pub mod attr {
        pub const COMPRESSED:   u64 = 0x0000_0004;
        pub const IMMUTABLE:    u64 = 0x0000_0010;
        pub const APPEND:       u64 = 0x0000_0020;
        pub const NODUMP:       u64 = 0x0000_0040;
        pub const ENCRYPTED:    u64 = 0x0000_0800;
        pub const AUTOMOUNT:    u64 = 0x0000_1000;
        pub const MOUNT_ROOT:   u64 = 0x0000_2000;
        pub const VERITY:       u64 = 0x0010_0000;
        pub const DAX:          u64 = 0x0020_0000;
        pub const WRITE_ATOMIC: u64 = 0x0040_0000;
    }
}
```

`uapi::stat::Statx::zeroed() -> Self`:
1. /* All reserved fields MUST be zero on return */
2. /* In practice the kernel writes the whole struct via a stack buffer
       initialized with MaybeUninit::zeroed before partial fills. */

`uapi::stat::Statx::validate_request_mask(mask: u32) -> Result<u32, Errno>`:
1. if mask & mask::RESERVED != 0: return Err(EINVAL).
2. return Ok(mask & !mask::RESERVED).

`uapi::stat::Statx::validate_sync_flags(flags: u32) -> Result<u32, Errno>`:
1. let sync = flags & AT_STATX_SYNC_TYPE; /* 0x6000 */
2. if sync == AT_STATX_SYNC_TYPE: return Err(EINVAL).  /* both FORCE+DONT set */
3. return Ok(sync).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `statx_size_pinned`            | LAYOUT  | `sizeof(Statx) == 256` and per-field offsets match the table. |
| `statx_timestamp_size_pinned`  | LAYOUT  | `sizeof(StatxTimestamp) == 16`. |
| `mode_type_disjoint`           | INVARIANT | the seven `S_IF<type>` values are pairwise distinct under `S_IFMT` mask. |
| `perm_mask_complete`           | INVARIANT | `S_ISUID|S_ISGID|S_ISVTX|S_IRWXU|S_IRWXG|S_IRWXO == 0o7777`. |
| `request_mask_rejects_reserved`| INVARIANT | `validate_request_mask(STATX__RESERVED) == Err(EINVAL)`. |
| `sync_flags_reject_both`       | INVARIANT | `validate_sync_flags(0x6000) == Err(EINVAL)`. |
| `attr_mask_gates_attributes`   | INVARIANT | bit B of `stx_attributes` is observable only if `(stx_attributes_mask & B) != 0`. |
| `spare_fields_zero_on_return`  | INVARIANT | `__spare0`, `__spare2`, `__spare3` are zero post-fill. |
| `rdev_zero_for_non_device`     | INVARIANT | `!(S_ISBLK(stx_mode) || S_ISCHR(stx_mode)) ⟹ stx_rdev_major == 0 && stx_rdev_minor == 0`. |

### Layer 2: TLA+

`uapi/stat-mask.tla`:
- Models the per-statx mask negotiation between caller request and kernel fill.
- States: `Requested`, `Filled`, `Cleared` per bit.
- Properties:
  - `safety_reserved_bit_rejected` — `STATX__RESERVED` in request ⟹ EINVAL.
  - `safety_unsupported_field_zeroed` — unsupported datum ⟹ bit cleared AND field zeroed (or fabricated).
  - `safety_attr_mask_dominates_attrs` — `(stx_attributes & ~stx_attributes_mask) == 0`.
  - `liveness_request_terminates` — every well-formed `statx()` returns in bounded steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `validate_request_mask` post: `ret.is_ok() ⟹ (ret.unwrap() & STATX__RESERVED) == 0` | `Statx::validate_request_mask` |
| `validate_sync_flags` post: `ret.is_ok() ⟹ ret.unwrap() ∈ {AS_STAT, FORCE_SYNC, DONT_SYNC}` | `Statx::validate_sync_flags` |
| `Statx::stx_mask` post: subset of caller-supplied mask `∪` cheap-defaults; never includes `STATX__RESERVED` | `fs::stat::vfs_statx` |
| `Statx::stx_attributes` post: `stx_attributes & ~stx_attributes_mask == 0` | per-fs `getattr` |
| `Statx::__spare{0,2,3}` post: all zero on return | `fs::stat::cp_statx` |

### Layer 4: Verus/Creusot functional

`Per-statx call → vfs_statx → per-fs getattr → fill struct statx → copy_to_user(buf, &statx, sizeof statx) → return 0` byte-identical to upstream Linux at the same baseline, including:
- field-by-field equivalence of `stx_mask`, `stx_attributes`, all timestamps, all devs.
- bit-for-bit `__spare*` zeroing.
- glibc `ls -l`, `stat(1)`, `find -printf '%y'`, `coreutils stat --printf=%t.%T`, `rustc fs::metadata` round-trip equivalence.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

stat ABI reinforcement:

- **Per-`copy_to_user` of `struct statx` bounds-checked** — defense against per-OOB-write of stat buffer.
- **Per-`__spare*` zeroed before return** — defense against per-kernel-memory-disclosure via uninitialised padding.
- **Per-`STATX__RESERVED` rejected** — defense against per-future-ABI confusion / smuggling.
- **Per-`AT_STATX_SYNC_TYPE` validated (mutually exclusive)** — defense against per-undefined sync behavior.
- **Per-`stx_attributes_mask`-gated attribute read** — defense against per-stale-attribute-flag leak.
- **Per-`stx_rdev_*` zero for non-device inodes** — defense against per-kernel-pointer-or-leftover leak.
- **Per-glibc collision guard preserved** — defense against per-double-definition build breakage.
- **Per-mode-bit immutability (octal pinned)** — defense against per-coreutils-misinterpret of file type.
- **Per-`stx_blocks` 512-byte unit pinned** — defense against per-du(1)-double-counting.

## Grsecurity/PaX-style Reinforcement

- **PaX UDEREF/USERCOPY** — every `copy_to_user(buf, &statx, sizeof statx)` and `copy_from_user(&mask, ...)` MUST be wrapped in a UDEREF region; the buffer pointer is checked against the caller's user-vm before access, and the copy length is a compile-time constant `sizeof(Statx)` so PAX_USERCOPY's whitelisted slab/stack-object bounds check applies.
- **GRKERNSEC_HIDESYM-equivalent for `stx_ino` / `stx_dev_*` / `stx_mnt_id`** — when the caller lacks `CAP_SYS_ADMIN` in the file's user-namespace and the file lives in a different user-namespace, `stx_ino` and `stx_dev_*` MUST be sanitised (mapped to a per-namespace virtual id) so that inode numbers and device majors cannot be used as covert channels or to seed `/proc`-style symbol-resolution attacks. Gated by `CAP_SYS_ADMIN`.
- **GRKERNSEC_NO_SIMULT_CONNECT** — N/A (stat is a read-only metadata path; no socket-state interaction).
- **`STATX_ATTR_PROTECTED`** — Rookery extension (reserved bit in `__spare3`, exposed via `stx_attributes` once standardised): file is write-protected by the security module; userspace MAY use this to short-circuit destructive operations. Currently MUST be zero pending upstream consensus.
- **GRKERNSEC_PROC restrictions for cross-user statx visibility** — for inodes in `/proc/<pid>/`, statx MUST honour the same per-pid visibility policy as `procfs_op_getattr`: `gid`-restricted, `nogroup` filtered, hidden-from-other-users. The visibility check happens BEFORE any field is filled to prevent a confused-deputy timing attack.
- **PAX_USERCOPY bounded copy** — the kernel-side staging `struct statx` lives in a slab cache (`statx_buf_cache`) or on the kernel stack; in both cases its size is exactly 256 bytes, registered with the usercopy whitelist so an attempt to copy `> 256` bytes is a `BUG_ON`.
- **Per-mount `no-stx-leak` option** — `mount -o no_stx_leak,...`: when set, the kernel MUST clear `stx_dev_*`, `stx_ino`, `stx_mnt_id`, and `stx_subvol` to zero (or per-mount tokens) regardless of caller capability, preventing inode/dev leaks to applications that don't need them (containers, sandboxed builds).
- **Per-`stx_btime` and `stx_atime` clamp** — when the mount is `noatime` or `lazytime`, `stx_atime` MUST reflect the on-disk value, not the in-memory promoted value, to avoid leaking the precise wall-clock of the most recent secret-key-cache hit on the inode.
- **Per-`__reserved` (in `StatxTimestamp`) strict-zero on return** — defense against per-info-leak through uninitialised kernel stack tail.
- **Per-statx audit hook** — every call passes through `audit_statx(dirfd, path, flags, mask)` so security-policy daemons can observe (without serialising) which files are being introspected, supporting LSM rate-limit / honeypot policies.
- **Per-`stx_attributes_mask` lower-bounded** — the kernel MUST always set the mask bits for every attribute Rookery understands, even if the value bit is zero, so userspace cannot infer "attribute unsupported" from "attribute reported as zero" — closes a Rowhammer-style oracle on encryption state.
- **Per-symlink follow policy** — when `AT_SYMLINK_NOFOLLOW` is set, the kernel MUST NOT pre-fault target inode pages (`STATX_INO` returns the symlink's own inode), defense against per-symlink-pre-fault TOCTOU.

## Open Questions

- Should `STATX_ATTR_PROTECTED` be requested upstream for Rookery's LSM-write-protection signal, or kept Rookery-private under `__spare3`? (Tracked separately; placeholder.)
- Whether `stx_mnt_id` and `stx_mnt_id_unique` should be virtualised within a user-namespace by default, or only under an opt-in mount flag.
- Whether to emit a counter-rate-limited dmesg warning when userspace passes `STATX__RESERVED` (could leak fingerprinting info; currently silent EINVAL).

## Out of Scope

- `fs/stat.c` `vfs_statx` implementation (covered in `fs/stat.md` Tier-3).
- Per-filesystem `inode_operations::getattr` hooks (covered per-fs).
- `statx(2)` system-call entry / argument decode (covered in `uapi/syscalls/statx.md` Tier-5).
- Legacy `struct stat` / `struct stat64` glibc-side layout (covered in `uapi/headers/asm-stat.md` Tier-5 once written).
- Implementation code.
