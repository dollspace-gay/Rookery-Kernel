# Tier-5 syscall: newfstatat(2) — syscall 262

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/stat.c (SYSCALL_DEFINE4(newfstatat), vfs_fstatat, cp_new_stat)
  - include/uapi/linux/fcntl.h (AT_SYMLINK_NOFOLLOW, AT_EMPTY_PATH, AT_NO_AUTOMOUNT)
  - include/linux/stat.h (struct kstat)
  - arch/x86/entry/syscalls/syscall_64.tbl (262 common newfstatat)
-->

## Summary

`newfstatat(2)` (a.k.a. `fstatat`) generalizes `stat`/`lstat` over an optional starting **directory file descriptor** and a set of `AT_*` flags. The kernel resolves `path` relative to `dirfd` (or the cwd if `dirfd == AT_FDCWD`), with leaf-symlink dereferencing controlled by `AT_SYMLINK_NOFOLLOW`. If `AT_EMPTY_PATH` is set and `path` is the empty string, the call returns the metadata of `dirfd` itself, allowing per-fd stat without re-resolving a name. The kernel calls `vfs_fstatat(dirfd, path, &kstat, flag_xlate)`, then translates to the ABI-frozen `struct stat`. Critical for: openat-based safe path resolution, container runtimes (avoid TOCTOU), `O_PATH` workflows, libc `fstatat`/`fstat`/`statx` underpinnings.

## Signature

```c
int newfstatat(int dirfd, const char *path, struct stat *statbuf, int flags);
```

```c
/* flags: */
#define AT_SYMLINK_NOFOLLOW  0x100   /* do not follow leaf symlink */
#define AT_NO_AUTOMOUNT      0x800   /* do not trigger automount */
#define AT_EMPTY_PATH        0x1000  /* operate on dirfd itself if path == "" */
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `dirfd` | `int` | in | Directory fd, or `AT_FDCWD` to use cwd. |
| `path` | `const char *` | in | Pathname relative to `dirfd`; absolute paths ignore `dirfd`. May be `""` if `AT_EMPTY_PATH`. |
| `statbuf` | `struct stat *` | out | Destination for `sizeof(struct stat)` bytes. |
| `flags` | `int` | in | Bitwise OR of `AT_SYMLINK_NOFOLLOW`, `AT_NO_AUTOMOUNT`, `AT_EMPTY_PATH`. Other bits ⟹ `EINVAL`. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Success; `*statbuf` written. |
| `-1` + `errno` | Failure; `*statbuf` unchanged. |

## Errors

| errno | Trigger |
|---|---|
| `EBADF` | `dirfd` is not a valid open fd and `path` is not absolute. |
| `ENOTDIR` | `dirfd` refers to a non-directory and `path` is relative. |
| `EACCES` | Search permission denied on a component of the path prefix. |
| `EFAULT` | `path` or `statbuf` outside accessible user memory. |
| `EINVAL` | Invalid bit in `flags`; or `path == ""` without `AT_EMPTY_PATH`. |
| `ELOOP` | Too many symbolic links resolved (>40). |
| `ENAMETOOLONG` | `path` exceeds `PATH_MAX`. |
| `ENOENT` | A component of `path` does not exist. |
| `ENOMEM` | Out of kernel memory. |
| `EOVERFLOW` | Field does not fit in legacy 32-bit struct; use `statx`. |

## ABI surface

```text
__NR_newfstatat (x86_64)  = 262
__NR_fstatat64  (i386)    = 300
__NR_newfstatat (arm64)   = 79
__NR_newfstatat (riscv)   = 79
```

## Compatibility contract

REQ-1: Syscall number is **262** on x86_64; ABI-stable.

REQ-2: Resolution semantics:
- `dirfd == AT_FDCWD` ⟹ resolve relative to cwd.
- absolute `path` ⟹ `dirfd` is ignored (but must still be `AT_FDCWD` or valid; not validated when path is absolute on Linux).
- relative `path` ⟹ resolve relative to `dirfd`.

REQ-3: `flags`:
- `AT_SYMLINK_NOFOLLOW`: leaf symlink not dereferenced (lstat-like).
- `AT_EMPTY_PATH`: if `path == ""`, operate on the file referenced by `dirfd` itself (allows fstat-via-fstatat).
- `AT_NO_AUTOMOUNT`: do not trigger automount of leaf.
- Other bits ⟹ `EINVAL`.

REQ-4: `AT_EMPTY_PATH` makes `dirfd` legal for non-directory fds (regular files, sockets, anon_inodes) — provides `fstat` semantics through this path.

REQ-5: `AT_EMPTY_PATH` requires kernel-`CAP_DAC_READ_SEARCH` for some `O_PATH`-only fds in restricted modes? No — Linux mainline does not gate `AT_EMPTY_PATH` on capability; the fd ownership check is sufficient.

REQ-6: `vfs_fstatat(dirfd, path, &kstat, flags)` is the in-kernel entry; it converts AT_* into LOOKUP_* flags (`LOOKUP_FOLLOW`, `LOOKUP_AUTOMOUNT`, `LOOKUP_EMPTY`).

REQ-7: Path-string copy_from_user is bounded by `PATH_MAX`.

REQ-8: `statbuf` is written **only** on success.

REQ-9: `dirfd` reference uses `fdget(dirfd)` for the duration of the resolve; fdput on return.

REQ-10: `O_PATH` fds are valid as `dirfd`.

REQ-11: Permissions: `dirfd` must be openable for search (directory `+x`); intermediate dirs must permit search.

REQ-12: Forward-compat: kernel rejects unknown `flags` bits — clients should mask appropriately or accept `EINVAL`.

## Acceptance Criteria

- [ ] AC-1: `newfstatat(AT_FDCWD, "/etc/hostname", &sb, 0)` succeeds; `S_ISREG`.
- [ ] AC-2: `newfstatat(dirfd, "child", &sb, 0)` resolves relative to `dirfd`.
- [ ] AC-3: `newfstatat(dirfd, "", &sb, AT_EMPTY_PATH)` returns metadata of `dirfd` itself.
- [ ] AC-4: `newfstatat(AT_FDCWD, "link", &sb, AT_SYMLINK_NOFOLLOW)`: `S_ISLNK`.
- [ ] AC-5: `newfstatat(AT_FDCWD, "link", &sb, 0)`: follows; returns target's mode.
- [ ] AC-6: `newfstatat(AT_FDCWD, "", &sb, 0)`: `EINVAL` (empty path without AT_EMPTY_PATH).
- [ ] AC-7: `newfstatat(-1, "rel", &sb, 0)`: `EBADF`.
- [ ] AC-8: `newfstatat(AT_FDCWD, "x", &sb, 0xdeadbeef)`: `EINVAL`.
- [ ] AC-9: `newfstatat(file_fd /* not dir */, "x", &sb, 0)`: `ENOTDIR`.
- [ ] AC-10: 50-symlink loop in prefix: `ELOOP`.
- [ ] AC-11: `newfstatat` on regular-file dirfd with `AT_EMPTY_PATH`: succeeds (fstat semantics).
- [ ] AC-12: `newfstatat` honors chroot: cannot escape with `..`.

## Architecture

```rust
#[syscall(nr = 262, abi = "sysv")]
pub fn sys_newfstatat(dirfd: i32, path: UserPtr<u8>, statbuf: UserPtr<UapiStat>, flags: i32) -> isize {
    Stat::do_newfstatat(dirfd, path, statbuf, flags)
}
```

`Stat::do_newfstatat(dirfd, path_uptr, statbuf, flags) -> isize`:
1. const ALLOWED: i32 = AT_SYMLINK_NOFOLLOW | AT_NO_AUTOMOUNT | AT_EMPTY_PATH;
2. if (flags & !ALLOWED) != 0 { return Err(EINVAL); }
3. let kpath = Stat::copy_path_from_user(path_uptr)?;            // EFAULT, ENAMETOOLONG
4. if kpath.is_empty() ∧ (flags & AT_EMPTY_PATH) == 0 { return Err(EINVAL); }
5. let lookup = Stat::at_flags_to_lookup(flags);
6. let resolved = Path::resolve_at(dirfd, &kpath, lookup)?;      // EBADF, ENOTDIR, ENOENT, ELOOP, EACCES
7. let kstat = Stat::vfs_getattr(&resolved, STATX_BASIC_STATS)?;
8. let ustat = Stat::cp_new_stat(&kstat)?;                       // EOVERFLOW
9. statbuf.write_to_user(&ustat)?;                               // EFAULT
10. Ok(0)

`Stat::at_flags_to_lookup(flags) -> LookupFlags`:
1. let mut lf = LookupFlags::empty();
2. if (flags & AT_SYMLINK_NOFOLLOW) == 0 { lf |= LookupFlags::FOLLOW; }
3. if (flags & AT_NO_AUTOMOUNT)    == 0 { lf |= LookupFlags::AUTOMOUNT; }
4. if (flags & AT_EMPTY_PATH)      != 0 { lf |= LookupFlags::EMPTY; }
5. lf

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_validated` | INVARIANT | unknown flag bits ⟹ EINVAL before any work. |
| `empty_path_requires_at_empty_path` | INVARIANT | empty path && !AT_EMPTY_PATH ⟹ EINVAL. |
| `lookup_flags_total` | INVARIANT | AT_* → LOOKUP_* mapping covers all legal combinations. |
| `no_statbuf_write_on_error` | INVARIANT | error ⟹ statbuf untouched. |
| `dirfd_get_balanced` | INVARIANT | fdget/fdput paired across resolve. |

### Layer 2: TLA+

`fs/newfstatat-syscall.tla`:
- States: per-flag-validate, per-copy_path, per-dirfd_get, per-resolve, per-getattr, per-cp_new_stat, per-write_user, per-fdput.
- Properties:
  - `safety_flag_check_before_work` — invalid flags rejected without work.
  - `safety_dirfd_lifetime` — fdput follows fdget on every path.
  - `safety_at_empty_path_obeyed` — empty path semantics correctly gated.
  - `liveness_newfstatat_terminates` — every call returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_newfstatat` post: ok ⟹ statbuf valid | `Stat::do_newfstatat` |
| `at_flags_to_lookup` post: bijective mapping | `Stat::at_flags_to_lookup` |

### Layer 4: Verus / Creusot functional

Per-`fstatat(2)` POSIX-2008 + Linux-extended semantics; selftests in `tools/testing/selftests/openat2/`.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`newfstatat(2)` reinforcement:

- **Per-flags strict-mask validation** — defense against per-future-flag smuggling.
- **Per-empty-path requires AT_EMPTY_PATH** — defense against per-accidental-fstat semantics.
- **Per-dirfd fdget/fdput balanced** — defense against per-dirfd-UAF.
- **Per-namespace-scoped resolution** — defense against per-chroot-escape.
- **Per-LOOKUP_AUTOMOUNT optional via AT_NO_AUTOMOUNT** — defense against per-automount-DoS.

## Grsecurity / PaX-style Reinforcement

- **PaX UDEREF on `path` and `statbuf`** — defense against per-kernel-pointer params; SMAP forced.
- **GRKERNSEC_HIDESYM on st_ino / st_dev / st_rdev** — pseudo-FS device / inode numbers masked for unprivileged.
- **GRKERNSEC_PROC_USERGROUP info-leak guard** — `/proc/<pid>/*` ownership masked under `hidepid=4`.
- **PAX_USERCOPY_HARDEN on cp_new_stat copy_to_user** — bounded, whitelisted slab.
- **GRKERNSEC_LINK restricted symlink follow** — defends prefix-component symlink-races.
- **Per-AT_EMPTY_PATH on O_PATH fd permitted only for fd owner** — defense against per-cross-process O_PATH stat leak.
- **Per-AT_NO_AUTOMOUNT default-on for restricted mode** — defense against per-automount-trigger DoS by unprivileged.
- **Per-dirfd ownership table strict** — defense against per-cross-process fd injection.
- **Per-flags reserved bits rejected** — defense against per-future-flag forward-compat smuggling.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `statx(2)` extended interface (covered in `statx.md` Tier-5).
- `openat2(2)` resolution-flag superset (covered in `openat2.md`).
- VFS getattr per-fs (covered in Tier-3 `fs/stat.md`).
- Implementation code.
