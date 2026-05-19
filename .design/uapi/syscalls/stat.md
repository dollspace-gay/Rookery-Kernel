# Tier-5 syscall: stat(2) — syscall 4

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/stat.c (SYSCALL_DEFINE2(stat), vfs_stat, vfs_getattr, cp_old_stat)
  - include/uapi/asm-generic/stat.h (struct stat layout)
  - include/linux/stat.h (struct kstat)
  - arch/x86/entry/syscalls/syscall_64.tbl (4  common  stat)
-->

## Summary

`stat(2)` retrieves the file-metadata block for the file at `path`, dereferencing all symbolic links along the way and at the leaf. The kernel resolves `path` via `user_path_at(AT_FDCWD, path, LOOKUP_FOLLOW, &p)`, calls `vfs_getattr(&p, &kstat, STATX_BASIC_STATS, AT_NO_AUTOMOUNT_DISABLED)`, then translates the architecture-neutral `struct kstat` into the legacy ABI-frozen `struct stat` (containing st_dev, st_ino, st_mode, st_nlink, st_uid, st_gid, st_rdev, st_size, st_atim, st_mtim, st_ctim, st_blksize, st_blocks). Critical for: shells (`test -f`), make/build systems, libc `stat()`/`fstat()` family.

`stat(2)` is the symlink-following sibling of `lstat(2)`; both predate the modern `statx(2)` interface but remain part of the permanent POSIX ABI surface.

## Signature

```c
int stat(const char *path, struct stat *statbuf);
```

```c
struct stat {
    dev_t      st_dev;       /* ID of device containing file */
    ino_t      st_ino;       /* inode number */
    mode_t     st_mode;      /* protection */
    nlink_t    st_nlink;     /* number of hard links */
    uid_t      st_uid;       /* user ID of owner */
    gid_t      st_gid;       /* group ID of owner */
    dev_t      st_rdev;      /* device ID (if special file) */
    off_t      st_size;      /* total size, in bytes */
    blksize_t  st_blksize;   /* blocksize for filesystem I/O */
    blkcnt_t   st_blocks;    /* number of 512B blocks allocated */
    struct timespec st_atim; /* time of last access */
    struct timespec st_mtim; /* time of last modification */
    struct timespec st_ctim; /* time of last status change */
};
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `path` | `const char *` | in | NUL-terminated user-space pathname; symbolic links are followed at every component including the leaf. |
| `statbuf` | `struct stat *` | out | User-space buffer of `sizeof(struct stat)` bytes; populated on success. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Success; `*statbuf` written. |
| `-1` + `errno` | Failure; `*statbuf` is unchanged. |

## Errors

| errno | Trigger |
|---|---|
| `EACCES` | Search permission denied on a component of the path prefix. |
| `EBADF` | (Not used for `stat`; see `fstat`.) |
| `EFAULT` | `path` or `statbuf` points outside accessible user memory. |
| `ELOOP` | Too many symbolic links resolved during path walk (>40 on Linux). |
| `ENAMETOOLONG` | `path` exceeds `PATH_MAX` (4096). |
| `ENOENT` | A component of `path` does not exist; `path` is the empty string. |
| `ENOMEM` | Out of kernel memory. |
| `ENOTDIR` | A component of the path prefix is not a directory. |
| `EOVERFLOW` | `st_size`, `st_blocks`, or `st_ino` does not fit in the legacy struct on a 32-bit ABI (use `stat64` / `statx`). |

## ABI surface

```text
__NR_stat   (x86_64)  = 4
__NR_stat   (i386)    = 106   (legacy 32-bit struct)
__NR_stat64 (i386)    = 195   (LFS variant)
/* arm64 / riscv: no __NR_stat; userspace uses statx or newfstatat. */

/* The on-the-wire struct stat differs per-arch; the kernel's
   architecture-neutral working type is struct kstat. */
```

## Compatibility contract

REQ-1: Syscall number is **4** on x86_64; ABI-stable.

REQ-2: Path walk uses `LOOKUP_FOLLOW` semantics (follow leaf symlink).

REQ-3: `vfs_getattr` is dispatched on the resolved `struct path`; the inode operations `i_op->getattr` may override default behavior.

REQ-4: Returned `st_mode` encodes file type (`S_IFMT` mask) and permission bits; type values are `S_IFREG`, `S_IFDIR`, `S_IFLNK` (never, since we followed), `S_IFCHR`, `S_IFBLK`, `S_IFIFO`, `S_IFSOCK`.

REQ-5: `st_dev` is the **mount-point** device, not the underlying block device; bind mounts may shift it.

REQ-6: `st_ino` is filesystem-internal; not guaranteed unique across mounts.

REQ-7: `st_blocks` counts **512-byte blocks** regardless of `st_blksize`; per POSIX.

REQ-8: `st_atim` may be omitted if mount option `noatime`/`relatime`/`lazyatime` defers update.

REQ-9: Path resolution is performed under the caller's namespace (mount, user, pid as relevant); chrooted callers see paths relative to their root.

REQ-10: `EFAULT` is returned even before any work begins if `path` is non-NUL but unreadable; SMAP / PaX UDEREF is in force.

REQ-11: All path-string copy_from_user operations are bounded by `PATH_MAX`.

REQ-12: On a 64-bit ABI, `EOVERFLOW` is structurally impossible (all fields are 64-bit).

## Acceptance Criteria

- [ ] AC-1: `stat("/etc/hostname", &sb)` returns 0 with `S_ISREG(sb.st_mode) == 1`.
- [ ] AC-2: `stat("/proc/self/exe", &sb)` follows the symlink and returns the target's metadata.
- [ ] AC-3: `stat("/no/such/path", &sb)` returns -1 with `errno == ENOENT`.
- [ ] AC-4: `stat(NULL, &sb)` returns -1 with `errno == EFAULT`.
- [ ] AC-5: `stat(path_4097_chars, &sb)` returns -1 with `errno == ENAMETOOLONG`.
- [ ] AC-6: `stat(path, (void*)0x1)` returns -1 with `errno == EFAULT`.
- [ ] AC-7: Loop of 50 symlinks: `errno == ELOOP`.
- [ ] AC-8: `stat` on directory: `S_ISDIR == 1`, `st_size` = dir block size.
- [ ] AC-9: `stat` on a UNIX socket file: `S_ISSOCK == 1`.
- [ ] AC-10: Caller without `+x` on intermediate dir: `errno == EACCES`.

## Architecture

```rust
#[syscall(nr = 4, abi = "sysv")]
pub fn sys_stat(path: UserPtr<u8>, statbuf: UserPtr<UapiStat>) -> isize {
    Stat::do_stat(path, statbuf, LookupFlags::FOLLOW)
}
```

`Stat::do_stat(path, statbuf, flags) -> isize`:
1. let kpath = Stat::copy_path_from_user(path)?;             // EFAULT, ENAMETOOLONG
2. let resolved = Path::resolve_at(AT_FDCWD, &kpath, flags)?;// ENOENT, ENOTDIR, ELOOP, EACCES
3. let kstat = Stat::vfs_getattr(&resolved, STATX_BASIC_STATS)?;
4. let ustat = Stat::cp_new_stat(&kstat)?;                   // EOVERFLOW
5. statbuf.write_to_user(&ustat)?;                           // EFAULT
6. Ok(0)

`Stat::cp_new_stat(kstat) -> Result<UapiStat>`:
1. let mut u = UapiStat::zeroed();
2. u.st_dev      = encode_dev(kstat.dev)?;
3. u.st_ino      = kstat.ino.try_into().map_err(|_| EOVERFLOW)?;
4. u.st_mode     = kstat.mode;
5. u.st_nlink    = kstat.nlink;
6. u.st_uid      = kstat.uid;
7. u.st_gid      = kstat.gid;
8. u.st_rdev     = encode_dev(kstat.rdev)?;
9. u.st_size     = kstat.size.try_into().map_err(|_| EOVERFLOW)?;
10. u.st_blksize = kstat.blksize;
11. u.st_blocks  = kstat.blocks.try_into().map_err(|_| EOVERFLOW)?;
12. u.st_atim    = kstat.atime;
13. u.st_mtim    = kstat.mtime;
14. u.st_ctim    = kstat.ctime;
15. Ok(u)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `path_copy_bounded` | INVARIANT | path copy ≤ PATH_MAX. |
| `lookup_follow_set` | INVARIANT | `stat` always passes LOOKUP_FOLLOW. |
| `no_user_write_on_error` | INVARIANT | error ⟹ statbuf untouched. |
| `kstat_to_ustat_total` | INVARIANT | every kstat field has a defined translation or EOVERFLOW. |

### Layer 2: TLA+

`fs/stat-syscall.tla`:
- States: per-copy_path, per-resolve, per-getattr, per-cp_new_stat, per-write_user.
- Properties:
  - `safety_no_statbuf_write_on_error` — failure path never touches statbuf.
  - `safety_symlink_followed` — leaf symlink resolved.
  - `liveness_stat_terminates` — every call returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_stat` post: ok ⟹ statbuf valid | `Stat::do_stat` |
| `cp_new_stat` post: overflow ⟹ EOVERFLOW | `Stat::cp_new_stat` |

### Layer 4: Verus/Creusot functional

Per-`stat(2)` POSIX semantics; selftests in `tools/testing/selftests/filesystems/`.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`stat(2)` reinforcement:

- **Per-PATH_MAX-bounded copy_from_user on `path`** — defense against per-unbounded path-copy.
- **Per-LOOKUP_FOLLOW symlink loop counter** — defense against per-symlink-DoS.
- **Per-namespace-scoped resolution** — defense against per-chroot-escape.
- **Per-statbuf write last, on success only** — defense against per-partial-write info-leak.

## Grsecurity / PaX-style Reinforcement

- **PaX UDEREF on `path` copy_from_user** — defense against per-kernel-pointer path; SMAP enforced.
- **PaX UDEREF on `statbuf` copy_to_user** — defense against per-kernel-pointer destination.
- **GRKERNSEC_HIDESYM on st_ino / st_dev / st_rdev** — KASLR-relevant inode / device numbers in pseudo-filesystems are masked for unprivileged readers; only CAP_SYS_ADMIN sees raw values for `/proc`, `/sys`, kernfs.
- **GRKERNSEC_PROC_USERGROUP info-leak guard** — restricts visibility of `st_uid` / `st_gid` on `/proc/<pid>/*` for non-owning callers (matches /proc dir-mode 700).
- **PAX_USERCOPY_HARDEN on cp_new_stat copy_to_user** — bounded, whitelisted slab; structural memcpy size constant.
- **GRKERNSEC_LINK restricted-symlink-follow** — symlinks owned by other uids in world-writable sticky dirs are not followed by `stat` for unprivileged callers (matches sysctl `fs.protected_symlinks`).
- **Per-st_atime suppression under noatime / relatime** — defense against per-timestamp side-channel leak.
- **Per-LOOKUP_NO_XDEV optional** — defense against per-cross-mount inode-leak in chroot scenarios.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `statx(2)` extended interface (covered in `statx.md` Tier-5).
- VFS getattr per-fs (covered in Tier-3 `fs/stat.md`).
- 32-bit `stat64` ABI quirks (covered in arch-compat docs).
- Implementation code.
