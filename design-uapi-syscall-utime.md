---
title: "Tier-5 syscall: utime(2) — syscall 132"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`utime(2)` sets the access and modification timestamps on a file named by `path` to either the current time (when `times` is `NULL`) or to caller-supplied values in a `struct utimbuf`. It is the original Unix mtime/atime API, predates `utimensat(2)` by decades, and has been deprecated in favor of `utimensat()` since glibc 2.6 (which redirects `utime()` to `utimensat()` internally for sub-second precision). The kernel still implements `utime(2)` directly for legacy binaries and as a low-overhead `time_t`-precision path.

Critical for: legacy archivers (`tar`, `cpio`, `rsync` pre-3.0), legacy installers, package managers retaining file timestamps, build tools using `time_t`-precision touch operations, libc back-compat tests.

### Acceptance Criteria

- [ ] AC-1: `utime("/tmp/x", NULL)` as owner: success; atime=mtime=ctime=now.
- [ ] AC-2: `utime("/tmp/x", {1700000000, 1700000001})` as owner: stat shows those values.
- [ ] AC-3: `utime("/tmp/x", {1, 2})` as non-owner with write permission: -EPERM.
- [ ] AC-4: `utime("/tmp/x", NULL)` as non-owner with write permission: success (touch semantics).
- [ ] AC-5: `utime("/tmp/x", NULL)` as non-owner without write permission: -EACCES.
- [ ] AC-6: `utime("/nonexistent", NULL)`: -ENOENT.
- [ ] AC-7: `utime("/proc/1/cmdline", NULL)`: -EROFS or -EPERM (procfs read-only attrs).
- [ ] AC-8: `utime` on symlink: follows symlink; updates target's timestamps.
- [ ] AC-9: ctime updated to current time regardless of supplied actime/modtime.
- [ ] AC-10: IN_ATTRIB inotify event fires after successful utime.
- [ ] AC-11: utime on RO bind-mount: -EROFS.
- [ ] AC-12: Concurrent utime + read: atime reflects most recent setter.

### Architecture

```rust
#[syscall(nr = 132, abi = "sysv")]
pub fn sys_utime(path: UserPtr<u8>, times: UserPtr<Utimbuf>) -> isize {
    Utime::do_utime(path, times)
}
```

`Utime::do_utime(path_ptr, times_ptr) -> isize`:
1. let path = Path::copy_from_user(path_ptr)?;
2. let kts: Option<[Timespec; 2]> = if times_ptr.is_null() {
3.    None
4.   } else {
5.    let ub: Utimbuf = times_ptr.copy_in_struct()?;
6.    Some([Timespec::from_secs(ub.actime), Timespec::from_secs(ub.modtime)])
7.   };
8. Utime::do_utimes_path(AtFd::Cwd, &path, kts, /* flags = */ 0)

`Utime::do_utimes_path(dfd, path, kts, flags) -> isize`:
1. let lookup_flags = LOOKUP_FOLLOW;
2. let dentry = vfs::path_lookup(dfd, path, lookup_flags)?;
3. Utime::utimes_common(&dentry, kts)

`Utime::utimes_common(dentry, kts) -> isize`:
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
17. Utime::check_permissions(&dentry, &iattr)?;
18. lsm::security_inode_setattr(&dentry, &iattr)?;
19. vfs::notify_change(&dentry, &iattr)?;
20. fsnotify::notify_attrib(&inode);
21. 0

`Utime::check_permissions(dentry, iattr) -> Result<()>`:
1. let inode = dentry.d_inode;
2. if iattr.ia_valid & ATTR_TOUCH != 0 {
3.   /* times == NULL: touch semantics */
4.   if !creds::is_owner(inode) && !creds::has_cap(CAP_FOWNER)
5.      && !inode::may_write(inode) { return Err(EACCES); }
6. } else {
7.   /* explicit timestamps: ownership required */
8.   if !creds::is_owner(inode) && !creds::has_cap(CAP_FOWNER) { return Err(EPERM); }
9. }
10. Ok(())

### Out of Scope

- `utimes(2)` (separate Tier-5 doc — microsecond precision).
- `futimesat(2)` (separate Tier-5 doc — dirfd-relative).
- `utimensat(2)` (Tier-5 covered separately — nanosecond + AT_SYMLINK_NOFOLLOW + UTIME_NOW/OMIT).
- Filesystem-internal mtime/atime update (covered per-FS Tier-3).
- Implementation code.

### signature

```c
#include <utime.h>

int utime(const char *path, const struct utimbuf *times);

struct utimbuf {
    time_t actime;     /* access time */
    time_t modtime;    /* modification time */
};
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `path` | `const char *` | in | Null-terminated absolute or relative path resolved against AT_FDCWD. |
| `times` | `const struct utimbuf *` | in (optional) | When NULL: set both timestamps to `current_time(inode)`. When non-NULL: set atime=actime, mtime=modtime. |

### return value

| Value | Meaning |
|---|---|
| `0` | Success; atime and mtime updated; ctime set to current_time(inode). |
| `-1` + `errno` | Failure (see Errors). |

### errors

| errno | Trigger |
|---|---|
| `EACCES` | times == NULL and caller lacks write permission on file (and is not owner, and lacks CAP_FOWNER). |
| `EPERM` | times != NULL and caller is not file owner and lacks CAP_FOWNER. |
| `EFAULT` | path or times user pointer faults. |
| `ENOENT` | A path component does not exist. |
| `ENAMETOOLONG` | path exceeds PATH_MAX. |
| `ENOTDIR` | An intermediate component of path is not a directory. |
| `EROFS` | File is on a read-only filesystem. |
| `ELOOP` | Too many symlinks during path resolution. |
| `EIO` | Inode-update I/O failure. |

### abi surface

```text
__NR_utime  (x86_64)    = 132
__NR_utime  (i386)      = 30
__NR_utime  (arm64)     = (not defined — deprecated; use utimensat)
__NR_utime  (riscv64)   = (not defined — deprecated; use utimensat)

/* x86_64 retains utime for legacy ELF binaries; modern arches drop it. */

struct utimbuf {
    __kernel_old_time_t actime;     /* signed 32-bit on x86; 64-bit on most arches */
    __kernel_old_time_t modtime;
};
```

### compatibility contract

REQ-1: Syscall number is **132** on x86_64. Not defined on arm64/riscv64 generic-syscall arches; userspace there uses `utimensat(AT_FDCWD, path, ts, 0)`.

REQ-2: `path` is resolved relative to AT_FDCWD, with symlinks **followed** (unlike `lutimes`).

REQ-3: When `times == NULL`: both atime and mtime set to `current_time(inode)`. Permission rule: caller must either own the file, OR have write permission on the file, OR hold `CAP_FOWNER`. This is the "touch" semantics path.

REQ-4: When `times != NULL`: explicit timestamps supplied. Permission rule: caller MUST be file owner OR hold `CAP_FOWNER`. Write permission is not sufficient (this differs from times==NULL).

REQ-5: ctime is always updated to `current_time(inode)` regardless of times argument — ctime tracks metadata change, which an atime/mtime set is.

REQ-6: Internally implemented in fs/utimes.c via `do_utimes(AT_FDCWD, path, kts, 0)` where `kts[0] = utimbuf.actime`, `kts[1] = utimbuf.modtime`. Sub-second fields are zero.

REQ-7: Filesystem must support timestamp update; tmpfs, ext4, xfs, btrfs all support. fuse forwards to userspace handler.

REQ-8: `utime()` honors filesystem `noatime`, `nodiratime`, `relatime` mount flags by clearing/forcing atime updates as appropriate — but explicit utime overrides relatime suppression.

REQ-9: Inotify/fanotify `IN_ATTRIB` event fires on the inode.

REQ-10: LSM hook `security_inode_setattr` invoked with ATTR_ATIME | ATTR_MTIME | ATTR_CTIME. SELinux / AppArmor / SMACK may deny.

REQ-11: Audit subsystem records `AUDIT_PATH` for path and `AUDIT_SYSCALL` with utimbuf values (if audit enabled).

REQ-12: Year-2038 concern on i386: `time_t` is 32-bit; `actime`/`modtime` overflow at 2038-01-19 03:14:07 UTC. x86_64 is 64-bit-clean.

REQ-13: `EROFS` returned before permission checks when the filesystem is read-only — POSIX-required ordering.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `path_user_copy_bounded` | INVARIANT | path copy bounded by PATH_MAX. |
| `null_times_owner_or_write` | INVARIANT | times==NULL ⟹ owner ∨ write ∨ CAP_FOWNER. |
| `nonnull_times_owner_only` | INVARIANT | times!=NULL ⟹ owner ∨ CAP_FOWNER. |
| `ctime_always_updated` | INVARIANT | every success ⟹ ATTR_CTIME set. |
| `symlink_followed` | INVARIANT | LOOKUP_FOLLOW always set (no AT_SYMLINK_NOFOLLOW analogue). |

### Layer 2: TLA+

`fs/utime.tla`:
- States: per-path lookup, per-permission gate, per-iattr-commit, per-notify.
- Properties:
  - `safety_ownership_for_explicit` — explicit times require ownership.
  - `safety_write_or_owner_for_null` — NULL times require write-or-owner.
  - `safety_ctime_invariant` — ctime updated every success.
  - `liveness_terminates` — utime returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_utime` post: success ⟹ inode timestamps reflect inputs (or now) | `Utime::do_utime` |
| `check_permissions` post: returns Err unless rule satisfied | `Utime::check_permissions` |
| `utimes_common` post: iattr posted via vfs::notify_change | `Utime::utimes_common` |

### Layer 4: Verus / Creusot functional

Per-`utime(2)` man-page semantic equivalence; POSIX.1-2001 retained API; LTP `utime01..utime03` pass.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`utime(2)` reinforcement:

- **Per-path PATH_MAX bound** — defense against per-overlong-name DoS.
- **Per-permission split owner/write** — defense against per-non-owner timestamp poisoning.
- **Per-LSM hook gate** — defense against per-policy bypass.
- **Per-ATTR_CTIME forced** — defense against per-stat-cache poisoning (metadata change must be visible).
- **Per-EROFS first-class** — defense against per-write-attempt on RO mount.
- **Per-fsnotify IN_ATTRIB** — defense against per-attribute change going unobserved.

### grsecurity / pax surface

- **PaX UDEREF on path + utimbuf copy_from_user** — SMAP enforced; defense against per-kernel-deref-of-user-pointer bug.
- **GRKERNSEC_LINK / TPE** — symlink-following utime in /tmp blocked if the symlink owner differs from the dereferencing task (sticky-bit dir + non-owner symlink classic attack class).
- **GRKERNSEC_CHROOT** — chrooted task cannot utime files outside its chroot (path resolution stays in jail).
- **GRKERNSEC_AUDIT_CHMOD analog (audit_setattr)** — every successful utime is auditable for security-sensitive groups.
- **CAP_FOWNER bounding** — grsec respects file-cap bounding set; a process with CAP_FOWNER dropped from its bounding set cannot regain it for utime.
- **PAX_USERCOPY_HARDEN on utimbuf copy** — utimbuf struct must lie wholly within a whitelisted slab/stack range.
- **Year-2038 hardening on i386** — grsec treats negative time_t values as suspect and audits (configurable).
- **No_new_privs honors** — utime is not a privilege boundary; NNP has no effect.
- **GRKERNSEC_TPE on path execution** — irrelevant (utime never executes).
- **Per-mount nosuid / noatime / relatime preserved** — utime cannot resurrect atime updates suppressed by mount options unless explicit times are passed.
- **Per-deprecated-syscall audit** — grsec may emit a warning (configurable) when legacy utime is invoked, recommending utimensat migration.

