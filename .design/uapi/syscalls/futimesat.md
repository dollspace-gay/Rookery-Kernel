# Tier-5 syscall: futimesat(2) — syscall 261

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/utimes.c (SYSCALL_DEFINE3(futimesat, ...), do_utimes)
  - include/uapi/linux/time.h (struct timeval)
  - include/linux/fs.h (struct iattr, ATTR_ATIME_SET, ATTR_MTIME_SET)
  - arch/x86/entry/syscalls/syscall_64.tbl (261  common  futimesat)
-->

## Summary

`futimesat(2)` sets the access and modification timestamps on a file with **dirfd-relative path resolution** plus **microsecond** precision. It is the transitional bridge between `utimes(2)` (no dirfd) and `utimensat(2)` (nanoseconds + dirfd + flags). Linux added it in 2.6.16, glibc deprecated it in 2.28 in favor of `utimensat()`. The kernel still implements it directly for legacy callers.

The dirfd argument resolves `path` against an open directory fd (or `AT_FDCWD`), allowing race-free directory traversal in unprivileged code.

Critical for: legacy GNU coreutils (`touch` pre-8.1), pre-utimensat archivers, container runtimes performing dirfd-anchored ops, FreeBSD/macOS portable code paths.

## Signature

```c
#include <sys/time.h>

int futimesat(int dirfd, const char *path, const struct timeval times[2]);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `dirfd` | `int` | in | Directory fd or `AT_FDCWD` for cwd-relative resolution. |
| `path` | `const char *` | in | Path; resolved against dirfd. May be NULL on some impls to mean dirfd itself (Linux: not supported; use `utimensat`). |
| `times` | `const struct timeval[2]` | in (optional) | NULL ⟹ both = now; else times[0]=atime, times[1]=mtime. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Success. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EACCES` | times == NULL and caller lacks owner/write/CAP_FOWNER. |
| `EPERM` | times != NULL and caller lacks owner/CAP_FOWNER. |
| `EINVAL` | tv_usec out of [0, 999_999]; or path NULL on Linux. |
| `EBADF` | dirfd is not a valid open fd (and not AT_FDCWD). |
| `ENOTDIR` | dirfd refers to a non-directory and path is relative. |
| `EFAULT` | path or times pointer faults. |
| `ENOENT` | Path component missing. |
| `ENAMETOOLONG` | path > PATH_MAX. |
| `EROFS` | Read-only filesystem. |
| `ELOOP` | Symlink loop. |
| `EIO` | Inode-update I/O failure. |

## ABI surface

```text
__NR_futimesat  (x86_64)   = 261
__NR_futimesat  (i386)     = 299
__NR_futimesat  (arm64)    = (not present — use utimensat)
__NR_futimesat  (riscv64)  = (not present — use utimensat)

AT_FDCWD  = -100   /* "interpret as cwd" sentinel for dirfd */
```

## Compatibility contract

REQ-1: Syscall number is **261** on x86_64. Absent on arm64/riscv64.

REQ-2: `dirfd == AT_FDCWD` ⟹ resolve path relative to cwd. Else dirfd MUST be an open directory fd; path is resolved against it (mount-namespace consistent).

REQ-3: `dirfd` referring to a non-directory + relative path ⟹ `ENOTDIR`. Absolute path ignores dirfd.

REQ-4: `path == NULL` on Linux is unsupported (returns EINVAL or EFAULT depending on path); use `utimensat(dirfd, NULL, ts, 0)` to operate on the dirfd itself.

REQ-5: Symlink semantics: symlinks **followed** (no NOFOLLOW flag exists for futimesat — that's a utimensat feature).

REQ-6: tv_usec ∈ [0, 999_999]; out-of-range ⟹ EINVAL.

REQ-7: When `times == NULL`: touch semantics — caller needs owner, write permission, OR CAP_FOWNER.

REQ-8: When `times != NULL`: explicit-timestamp semantics — caller needs owner OR CAP_FOWNER (write alone insufficient).

REQ-9: ctime always updated to `current_time(inode)` on success.

REQ-10: Internally: timeval → timespec by `nsec = usec * 1000`; then `do_utimes(dirfd, path, ts, 0)`.

REQ-11: LSM `security_inode_setattr` invoked.

REQ-12: Inotify IN_ATTRIB / fanotify FAN_ATTRIB on success.

REQ-13: Year-2038 on i386: 32-bit time_t — modern glibc bypasses futimesat for time64 by using utimensat.

REQ-14: EROFS returned before permission check on RO mount.

## Acceptance Criteria

- [ ] AC-1: `futimesat(dirfd, "x", NULL)` as owner: atime=mtime=ctime=now.
- [ ] AC-2: `futimesat(AT_FDCWD, "x", times)`: equivalent to utimes("x", times).
- [ ] AC-3: dirfd = open("/etc", O_DIRECTORY); futimesat(dirfd, "hostname", ts): updates /etc/hostname.
- [ ] AC-4: dirfd = open("/etc/hostname", O_RDONLY); futimesat(dirfd, "x", ts): -ENOTDIR.
- [ ] AC-5: dirfd = -1 (not AT_FDCWD): -EBADF.
- [ ] AC-6: tv_usec = 2_000_000: -EINVAL.
- [ ] AC-7: Non-owner with write, NULL times: success.
- [ ] AC-8: Non-owner with write, explicit times: -EPERM.
- [ ] AC-9: Non-owner without write, NULL times: -EACCES.
- [ ] AC-10: Symlink in path: target's timestamps updated.
- [ ] AC-11: Path absolute: dirfd ignored.
- [ ] AC-12: IN_ATTRIB fires on success.

## Architecture

```rust
#[syscall(nr = 261, abi = "sysv")]
pub fn sys_futimesat(dirfd: i32, path: UserPtr<u8>, times: UserPtr<[Timeval; 2]>) -> isize {
    Futimesat::do_futimesat(dirfd, path, times)
}
```

`Futimesat::do_futimesat(dirfd, path_ptr, times_ptr) -> isize`:
1. let path = Path::copy_from_user(path_ptr)?;
2. let kts: Option<[Timespec; 2]> = if times_ptr.is_null() {
3.    None
4.   } else {
5.    let tv: [Timeval; 2] = times_ptr.copy_in_array()?;
6.    if tv[0].tv_usec < 0 || tv[0].tv_usec >= 1_000_000 ||
7.       tv[1].tv_usec < 0 || tv[1].tv_usec >= 1_000_000 { return Err(EINVAL); }
8.    Some([Timespec::from_us(tv[0].tv_sec, tv[0].tv_usec),
9.          Timespec::from_us(tv[1].tv_sec, tv[1].tv_usec)])
10.  };
11. Futimesat::do_utimes_path(dirfd, &path, kts, 0)

`Futimesat::do_utimes_path(dirfd, path, kts, flags) -> isize`:
1. let lookup_flags = LOOKUP_FOLLOW;
2. let dentry = vfs::path_lookup_at(dirfd, path, lookup_flags)?;     // EBADF / ENOTDIR
3. Futimesat::utimes_common(&dentry, kts)

`Futimesat::utimes_common(dentry, kts) -> isize`:
1. let inode = dentry.d_inode;
2. let mut iattr = IAttr::default();
3. match kts {
4.   None => {
5.     iattr.ia_valid = ATTR_ATIME | ATTR_MTIME | ATTR_CTIME | ATTR_TOUCH;
6.     iattr.ia_atime = current_time(&inode);
7.     iattr.ia_mtime = iattr.ia_atime;
8.     iattr.ia_ctime = iattr.ia_atime;
9.   }
10.  Some([at, mt]) => {
11.    iattr.ia_valid = ATTR_ATIME | ATTR_ATIME_SET | ATTR_MTIME | ATTR_MTIME_SET | ATTR_CTIME;
12.    iattr.ia_atime = at;
13.    iattr.ia_mtime = mt;
14.    iattr.ia_ctime = current_time(&inode);
15.  }
16. }
17. Futimesat::check_permissions(&dentry, &iattr)?;
18. lsm::security_inode_setattr(&dentry, &iattr)?;
19. vfs::notify_change(&dentry, &iattr)?;
20. fsnotify::notify_attrib(&inode);
21. 0

`Futimesat::check_permissions(dentry, iattr) -> Result<()>`:
1. let inode = dentry.d_inode;
2. if iattr.ia_valid & ATTR_TOUCH != 0 {
3.   if !creds::is_owner(inode) && !creds::has_cap(CAP_FOWNER)
4.      && !inode::may_write(inode) { return Err(EACCES); }
5. } else {
6.   if !creds::is_owner(inode) && !creds::has_cap(CAP_FOWNER) { return Err(EPERM); }
7. }
8. Ok(())

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `dirfd_validate` | INVARIANT | dirfd != AT_FDCWD ⟹ fd table lookup; bad fd ⟹ EBADF. |
| `dirfd_must_be_dir_for_relative` | INVARIANT | non-dir + relative path ⟹ ENOTDIR. |
| `usec_range_check` | INVARIANT | tv_usec ∉ [0, 999_999] ⟹ EINVAL. |
| `path_user_copy_bounded` | INVARIANT | path ≤ PATH_MAX. |
| `null_times_owner_or_write` | INVARIANT | NULL times ⟹ owner ∨ write ∨ CAP_FOWNER. |
| `nonnull_times_owner_only` | INVARIANT | non-NULL ⟹ owner ∨ CAP_FOWNER. |

### Layer 2: TLA+

`fs/futimesat.tla`:
- States: per-dirfd-resolve, per-path-lookup, per-permission, per-iattr-commit, per-notify.
- Properties:
  - `safety_dirfd_validation` — EBADF/ENOTDIR precede any I/O.
  - `safety_usec_range_first` — EINVAL on bad usec precedes lookup.
  - `safety_ownership_for_explicit` — non-owner cannot set explicit times.
  - `safety_ctime_invariant` — ctime updated each success.
  - `liveness_terminates` — futimesat returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_futimesat` post: success ⟹ inode timestamps reflect inputs (or now) | `Futimesat::do_futimesat` |
| `path_lookup_at` post: dirfd refcounted; lookup begins at fd's inode | `vfs::path_lookup_at` |
| `usec_to_nsec` post: nsec = usec × 1000 | conversion |

### Layer 4: Verus / Creusot functional

Per-`futimesat(2)` man-page; POSIX *at family semantics. LTP `futimesat01` pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`futimesat(2)` reinforcement:

- **Per-dirfd validation** — defense against per-bogus-fd abuse.
- **Per-dirfd-is-directory check** — defense against per-non-dir relative-path resolution.
- **Per-tv_usec range** — defense against per-arg-injected nsec overflow.
- **Per-PATH_MAX bound** — defense against per-overlong-path DoS.
- **Per-LSM hook** — defense against per-policy bypass.
- **Per-permission split owner/write** — defense against per-non-owner timestamp poisoning.

## Grsecurity / PaX surface

- **PaX UDEREF on path + timeval copy_from_user** — SMAP enforced; per-kernel-deref-of-user-pointer hardening.
- **GRKERNSEC_CHROOT_FCHDIR** — dirfd anchored outside chroot is rejected for chrooted processes; futimesat cannot escape via leaked dirfd.
- **GRKERNSEC_LINK** — symlink-following in sticky-bit dirs blocked when symlink owner ≠ caller.
- **GRKERNSEC_AUDIT_SETATTR** — every successful futimesat is auditable.
- **CAP_FOWNER bounding** — bounded set respected; CAP_FOWNER dropped from bounding cannot be regained.
- **PAX_USERCOPY_HARDEN on timeval[2] copy** — bounded to whitelisted slab/stack.
- **Per-deprecated-syscall audit** — grsec may emit a warn that futimesat is legacy; userspace should use utimensat.
- **GRKERNSEC_TPE neutral** — no exec.
- **Mount-flag respected** — noatime/relatime cannot be silently overridden; only explicit utimes succeeds.
- **GRKERNSEC_PROC consistency** — auditing reflects dirfd-resolved final path, not the textual path argument.
- **No_new_privs neutral** — futimesat is not a privilege boundary.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `utime(2)`, `utimes(2)`, `utimensat(2)` (separate Tier-5 docs).
- Dirfd validation generic (Tier-3 `fs/namei.md`).
- `*at(2)` family generic semantics (Tier-3 `fs/uapi-at.md`).
- Implementation code.
