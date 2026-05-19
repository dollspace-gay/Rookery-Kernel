# Tier-5 syscall: utimes(2) — syscall 235

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/utimes.c (SYSCALL_DEFINE2(utimes, ...), do_utimes, vfs_utimes)
  - include/uapi/linux/time.h (struct timeval)
  - include/linux/fs.h (struct iattr, ATTR_ATIME_SET, ATTR_MTIME_SET)
  - arch/x86/entry/syscalls/syscall_64.tbl (235  common  utimes)
-->

## Summary

`utimes(2)` sets the access and modification timestamps on a file named by `path` with **microsecond** precision (the `struct timeval` carries `tv_sec` + `tv_usec`). It superseded `utime(2)` historically (4.3BSD) and is itself superseded by `utimensat(2)` (nanosecond precision + dirfd + symlink-nofollow flag). glibc's `utimes()` wrapper today calls `utimensat(AT_FDCWD, path, ts2, 0)` after converting timeval → timespec.

Critical for: archivers using sub-second precision (cpio --extreme, dpkg), portable POSIX programs (FreeBSD/macOS compat shim), legacy build systems, `touch -t` GNU coreutils for sub-second touch.

## Signature

```c
#include <sys/time.h>

int utimes(const char *path, const struct timeval times[2]);

struct timeval {
    time_t      tv_sec;     /* seconds */
    suseconds_t tv_usec;    /* microseconds [0, 999_999] */
};

/* times[0] = atime, times[1] = mtime */
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `path` | `const char *` | in | Path; resolved against AT_FDCWD, symlinks followed. |
| `times` | `const struct timeval[2]` | in (optional) | NULL ⟹ both set to now; otherwise `times[0]=atime`, `times[1]=mtime`. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Success. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EACCES` | times == NULL and caller lacks write permission (and is not owner). |
| `EPERM` | times != NULL and caller is not owner and lacks CAP_FOWNER. |
| `EINVAL` | tv_usec outside [0, 999_999]. |
| `EFAULT` | path or times user-pointer faults. |
| `ENOENT` | Path component missing. |
| `ENAMETOOLONG` | path exceeds PATH_MAX. |
| `ENOTDIR` | Intermediate component not a directory. |
| `EROFS` | Read-only filesystem. |
| `ELOOP` | Symlink loop. |
| `EIO` | Inode-update I/O failure. |

## ABI surface

```text
__NR_utimes  (x86_64)   = 235
__NR_utimes  (i386)     = 271
__NR_utimes  (arm64)    = (not present — use utimensat)
__NR_utimes  (riscv64)  = (not present — use utimensat)

/* y2038-aware variants for compat-32: __NR_utimes_time64 family. */

struct timeval { __kernel_old_time_t tv_sec; __kernel_suseconds_t tv_usec; };
```

## Compatibility contract

REQ-1: Syscall number is **235** on x86_64. Not present on arm64/riscv64 generic-syscall arches.

REQ-2: Path resolved relative to AT_FDCWD; symlinks **followed**. Distinct from `lutimes()` (glibc wrapper around `utimensat(..., AT_SYMLINK_NOFOLLOW)`).

REQ-3: `tv_usec` validated in `[0, 999999]`. Out-of-range microseconds ⟹ `EINVAL`. tv_sec is signed and may be negative (pre-1970 timestamps) — implementation-defined whether the filesystem accepts.

REQ-4: When `times == NULL`: both atime and mtime set to `current_time(inode)`. Touch semantics: caller needs ownership, write permission, OR CAP_FOWNER.

REQ-5: When `times != NULL`: explicit timestamps. Caller MUST be owner OR hold CAP_FOWNER. Write permission alone is insufficient.

REQ-6: Internally: timeval → timespec by `tv_nsec = tv_usec * 1000`. The kernel-internal path is `do_utimes(AT_FDCWD, path, [ts0, ts1], 0)`.

REQ-7: ctime always updated to current_time(inode) on success.

REQ-8: Year-2038 on i386: 32-bit `time_t` in legacy timeval; modern glibc uses `__NR_utimes_time64` (syscall 412) for time64.

REQ-9: Filesystem mount flags (noatime, nodiratime, relatime) honored — explicit utimes overrides relatime suppression because it is an explicit set, not a stat-induced atime bump.

REQ-10: Inotify IN_ATTRIB and fanotify FAN_ATTRIB fire on success.

REQ-11: LSM `security_inode_setattr` invoked with ATTR_ATIME_SET | ATTR_MTIME_SET (or ATTR_TOUCH for null-times).

REQ-12: Path-resolution faults: ENAMETOOLONG > 4095 (PATH_MAX); ENOTDIR if intermediate is non-dir; ELOOP at MAXSYMLINKS (40).

REQ-13: EROFS returned before permission check when underlying mount is read-only.

## Acceptance Criteria

- [ ] AC-1: `utimes("/tmp/x", NULL)` as owner: atime=mtime=ctime=now.
- [ ] AC-2: `utimes("/tmp/x", [{1700,500000},{1701,123456}])` as owner: stat reflects those.
- [ ] AC-3: `tv_usec = 1_000_000`: -EINVAL.
- [ ] AC-4: `tv_usec = -1`: -EINVAL.
- [ ] AC-5: Non-owner with write: NULL times → success; explicit → -EPERM.
- [ ] AC-6: Non-owner without write, NULL times: -EACCES.
- [ ] AC-7: Symlink in path: target's timestamps updated.
- [ ] AC-8: Read-only mount: -EROFS.
- [ ] AC-9: Microsecond precision preserved on tmpfs/ext4/xfs/btrfs.
- [ ] AC-10: IN_ATTRIB fires after success.
- [ ] AC-11: Negative tv_sec accepted by ext4/xfs (pre-epoch) where filesystem supports.
- [ ] AC-12: Concurrent utimes from N threads: last-writer wins; no torn writes observable.

## Architecture

```rust
#[syscall(nr = 235, abi = "sysv")]
pub fn sys_utimes(path: UserPtr<u8>, times: UserPtr<[Timeval; 2]>) -> isize {
    Utimes::do_utimes_user(path, times)
}
```

`Utimes::do_utimes_user(path_ptr, times_ptr) -> isize`:
1. let path = Path::copy_from_user(path_ptr)?;
2. let kts: Option<[Timespec; 2]> = if times_ptr.is_null() {
3.    None
4.   } else {
5.    let tv: [Timeval; 2] = times_ptr.copy_in_array()?;
6.    if tv[0].tv_usec >= 1_000_000 || tv[1].tv_usec >= 1_000_000 ||
7.       tv[0].tv_usec < 0           || tv[1].tv_usec < 0           { return Err(EINVAL); }
8.    Some([Timespec { sec: tv[0].tv_sec, nsec: (tv[0].tv_usec as i64) * 1000 },
9.          Timespec { sec: tv[1].tv_sec, nsec: (tv[1].tv_usec as i64) * 1000 }])
10.  };
11. Utimes::do_utimes_path(AtFd::Cwd, &path, kts, /* flags = */ 0)

`Utimes::do_utimes_path(dfd, path, kts, flags) -> isize`:
1. let lookup_flags = LOOKUP_FOLLOW;
2. let dentry = vfs::path_lookup(dfd, path, lookup_flags)?;
3. Utimes::utimes_common(&dentry, kts)

`Utimes::utimes_common(dentry, kts) -> isize`:
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
17. Utimes::check_permissions(&dentry, &iattr)?;
18. lsm::security_inode_setattr(&dentry, &iattr)?;
19. vfs::notify_change(&dentry, &iattr)?;
20. fsnotify::notify_attrib(&inode);
21. 0

`Utimes::check_permissions(dentry, iattr) -> Result<()>`:
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
| `usec_range_check` | INVARIANT | tv_usec ∉ [0, 999_999] ⟹ EINVAL pre-permission. |
| `path_user_copy_bounded` | INVARIANT | path ≤ PATH_MAX. |
| `null_times_owner_or_write` | INVARIANT | NULL times ⟹ owner ∨ write ∨ CAP_FOWNER. |
| `nonnull_times_owner_only` | INVARIANT | non-NULL ⟹ owner ∨ CAP_FOWNER. |
| `usec_to_nsec_lossless` | INVARIANT | nsec = usec × 1000 ≤ 999_999_000. |

### Layer 2: TLA+

`fs/utimes.tla`:
- States: per-arg-validate, per-lookup, per-permission, per-iattr-commit, per-notify.
- Properties:
  - `safety_usec_range_first` — EINVAL precedes any I/O.
  - `safety_ownership_for_explicit` — explicit times require owner ∨ CAP_FOWNER.
  - `safety_ctime_invariant` — ctime updated every success.
  - `liveness_terminates` — utimes returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_utimes_user` post: success ⟹ inode timestamps reflect arg (or now) | `Utimes::do_utimes_user` |
| `usec_to_nsec` post: |nsec - usec*1000| == 0 | conversion |
| `utimes_common` post: notify_change ran with correct iattr | `Utimes::utimes_common` |

### Layer 4: Verus / Creusot functional

Per-`utimes(2)` man-page; POSIX.1-2008 retained; LTP `utimes01` pass; cpio/tar round-trip preserves microsecond mtime.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`utimes(2)` reinforcement:

- **Per-tv_usec range guard** — defense against per-arg-injected nsec overflow.
- **Per-PATH_MAX bound** — defense against per-overlong-path DoS.
- **Per-LSM hook gate** — defense against per-policy bypass.
- **Per-permission split owner/write** — defense against per-non-owner timestamp poisoning.
- **Per-EROFS-first ordering** — defense against per-RO-mount erroneous attempt.
- **Per-IN_ATTRIB notify** — defense against per-silent metadata change.

## Grsecurity / PaX surface

- **PaX UDEREF on path + timeval copy_from_user** — SMAP enforced; defense against per-kernel-deref-of-user-pointer bug.
- **GRKERNSEC_LINK / TPE-symlink** — utimes-via-symlink in sticky-bit dirs gated when symlink owner ≠ caller (mitigates symlink race + timestamp clobber).
- **GRKERNSEC_CHROOT** — chrooted task cannot reach files outside chroot.
- **CAP_FOWNER bounding** — bounded set respected; cannot regain dropped CAP_FOWNER for utimes.
- **PAX_USERCOPY_HARDEN on timeval[2] copy** — array copy must lie wholly within whitelisted slab/stack.
- **Year-2038 audit on i386** — grsec optionally audits negative or > INT_MAX tv_sec values to surface y2038-buggy callers.
- **Per-deprecated-syscall trace** — grsec may emit a one-shot warn that utimes is legacy; userspace recommended to use utimensat.
- **GRKERNSEC_AUDIT_SETATTR** — every successful utimes auditable.
- **No_new_privs neutral** — utimes is not a privilege boundary.
- **Mount-flag respected** — noatime/nodiratime cannot be overridden silently; only explicit utimes succeeds, which is auditable.
- **Microsecond-precision side-channel** — utimes precision (us) is coarser than nsec; grsec adds no extra blinding (already capped by FS resolution).

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `utime(2)` (separate Tier-5 doc — second precision).
- `futimesat(2)` (separate Tier-5 doc — dirfd-relative).
- `utimensat(2)` (separate Tier-5 doc — nanosecond + nofollow).
- `lutimes(2)` glibc wrapper (uses utimensat under the hood).
- Per-FS timestamp resolution (Tier-3 per FS).
- Implementation code.
