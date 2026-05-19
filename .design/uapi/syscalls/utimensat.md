# Tier-5 syscall: utimensat(2) — syscall 280

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/utimes.c (SYSCALL_DEFINE4(utimensat), do_utimes, vfs_utimes, utimes_common)
  - include/uapi/linux/time.h (UTIME_NOW, UTIME_OMIT)
  - include/uapi/asm-generic/fcntl.h (AT_SYMLINK_NOFOLLOW)
  - arch/x86/entry/syscalls/syscall_64.tbl (280 common  utimensat)
-->

## Summary

`utimensat(2)` sets the access and modification timestamps of a file with nanosecond resolution. It supersedes `utime(2)` and `utimes(2)` (which only support seconds and microseconds). The kernel resolves `path` relative to `dirfd`, accepts a 2-element `struct timespec` array (`times[0]` = atime, `times[1]` = mtime), and supports two sentinel `tv_nsec` values: `UTIME_NOW` (use current wall-clock time) and `UTIME_OMIT` (do not modify this timestamp). Passing `times == NULL` is equivalent to passing both as `UTIME_NOW`. Critical for: `touch(1)`, archive extractors (tar, cpio, rsync) preserving timestamps, build systems comparing mtimes.

`utimensat(2)` is the primary modern timestamp-setting interface; `futimens(3)` is a libc wrapper that calls `utimensat(fd, NULL, times, 0)`.

## Signature

```c
int utimensat(int dirfd, const char *path, const struct timespec times[2], int flags);
```

```c
struct timespec {
    time_t tv_sec;
    long   tv_nsec;
};

/* tv_nsec sentinel values: */
#define UTIME_NOW    ((1l << 30) - 1l)
#define UTIME_OMIT   ((1l << 30) - 2l)

/* flags: */
#define AT_SYMLINK_NOFOLLOW  0x100
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `dirfd` | `int` | in | Directory fd, or `AT_FDCWD`. If `path == NULL`, `dirfd` itself is the target. |
| `path` | `const char *` | in | NUL-terminated pathname. May be `NULL` to operate on `dirfd` itself. |
| `times` | `const struct timespec[2]` | in | Two timespecs: `times[0]` = atime, `times[1]` = mtime. May be `NULL` (both = UTIME_NOW). |
| `flags` | `int` | in | `AT_SYMLINK_NOFOLLOW` (don't follow leaf symlink). Other bits ⟹ `EINVAL`. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Success; timestamps updated (or omitted per UTIME_OMIT). |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EACCES` | Caller is not the owner, has no `CAP_FOWNER`, and: times == NULL OR both are UTIME_NOW, AND no write permission. |
| `EBADF` | `dirfd` invalid and `path` is not absolute (or `path == NULL` with invalid `dirfd`). |
| `EFAULT` | `path` or `times` outside accessible memory. |
| `EINVAL` | Invalid `flags`; `tv_nsec` not in [0, 10^9) and not a sentinel; `tv_sec` out of range. |
| `ELOOP` | Too many symbolic links. |
| `ENAMETOOLONG` | `path` exceeds `PATH_MAX`. |
| `ENOENT` | A component does not exist. |
| `ENOTDIR` | A path component is not a directory. |
| `EPERM` | `tv_nsec != UTIME_OMIT` and caller is neither owner nor has `CAP_FOWNER`. |
| `EROFS` | Read-only filesystem. |
| `ESRCH` | (rare) per-fs failure. |

## ABI surface

```text
__NR_utimensat (x86_64)  = 280
__NR_utimensat (i386)    = 320
__NR_utimensat (arm64)   = 88
__NR_utimensat (riscv)   = 88

/* y2038-safe: 64-bit tv_sec on all 64-bit ABIs.
   32-bit archs additionally provide __NR_utimensat_time64 = 412. */
```

## Compatibility contract

REQ-1: Syscall number is **280** on x86_64; ABI-stable.

REQ-2: `flags`: only `AT_SYMLINK_NOFOLLOW` accepted; other bits ⟹ `EINVAL`.

REQ-3: `times == NULL`: treated as both atime and mtime = UTIME_NOW. Requires only write permission on file.

REQ-4: `tv_nsec` validation:
- Range `[0, 10^9)`: valid wall-clock nanoseconds.
- `UTIME_NOW` (1<<30 - 1): use current time. `tv_sec` ignored.
- `UTIME_OMIT` (1<<30 - 2): leave timestamp unchanged. `tv_sec` ignored.
- Any other value ⟹ `EINVAL`.

REQ-5: Permission rules:
- Caller is file owner OR has `CAP_FOWNER` in fs's userns ⟹ allowed regardless.
- Otherwise, only `UTIME_NOW`/NULL/equivalent permitted, AND only if caller has write permission on file.
- `UTIME_OMIT` on both ⟹ no-op (no permission check at all per fs/utimes.c, though path resolution may still fault).

REQ-6: `dirfd == AT_FDCWD` ⟹ cwd; absolute path ⟹ ignore dirfd; relative path ⟹ resolve under dirfd.

REQ-7: If `path == NULL`: operate on `dirfd` itself (legacy `futimens` semantics expressed via this syscall).

REQ-8: `AT_SYMLINK_NOFOLLOW`: don't follow leaf symlink — set timestamps on the link itself. Most filesystems do not support setting symlink timestamps and will return success or `EOPNOTSUPP` depending on fs.

REQ-9: Timestamps copied via `copy_from_user(&kts, times, 2 * sizeof(timespec))` if non-NULL.

REQ-10: Path-string copy_from_user bounded by `PATH_MAX`.

REQ-11: `ctime` of the inode is also updated implicitly (always — POSIX requires).

REQ-12: LSM `inode_setattr` hook fires.

REQ-13: Forward-compat: y2038 — 64-bit `tv_sec` on 64-bit ABIs; 32-bit archs use `_time64` variants for new code.

## Acceptance Criteria

- [ ] AC-1: `utimensat(AT_FDCWD, "/tmp/x", {{1234567890,0},{1234567890,0}}, 0)`: atime + mtime = 1234567890.
- [ ] AC-2: `utimensat(AT_FDCWD, "/tmp/x", NULL, 0)`: both = current time.
- [ ] AC-3: `utimensat(..., {{UTIME_NOW,UTIME_NOW},{UTIME_OMIT,UTIME_OMIT}}, ...)`: atime updated, mtime untouched.
- [ ] AC-4: `utimensat(AT_FDCWD, "x", times_invalid_nsec, 0)`: `EINVAL`.
- [ ] AC-5: `utimensat(-1, "rel", times, 0)`: `EBADF`.
- [ ] AC-6: `utimensat(AT_FDCWD, NULL, times, 0)` with valid dirfd: timestamps on dirfd's target.
- [ ] AC-7: `utimensat(AT_FDCWD, "x", times, 0xdeadbeef)`: `EINVAL`.
- [ ] AC-8: Non-owner without CAP_FOWNER, explicit times: `EPERM`.
- [ ] AC-9: Non-owner with write perm, NULL times: succeeds.
- [ ] AC-10: Both UTIME_OMIT: success, no change, no perm check beyond path resolution.
- [ ] AC-11: `utimensat` on read-only fs: `EROFS`.
- [ ] AC-12: `utimensat(AT_FDCWD, "link", times, AT_SYMLINK_NOFOLLOW)`: targets link, not target.
- [ ] AC-13: ctime is updated whenever atime or mtime is updated.

## Architecture

```rust
#[syscall(nr = 280, abi = "sysv")]
pub fn sys_utimensat(
    dirfd: i32,
    path: UserPtr<u8>,
    times: UserPtr<[UapiTimespec; 2]>,
    flags: i32,
) -> isize {
    Utimes::do_utimensat(dirfd, path, times, flags)
}
```

`Utimes::do_utimensat(dirfd, path_uptr, times_uptr, flags) -> isize`:
1. if (flags & !AT_SYMLINK_NOFOLLOW) != 0 { return Err(EINVAL); }
2. /* Copy times if non-NULL. */
3. let kts: Option<[KTimespec; 2]> = if times_uptr.is_null() {
4.   None
5. } else {
6.   let raw = times_uptr.copy_from_user()?;                    // EFAULT
7.   Some([Utimes::validate(raw[0])?, Utimes::validate(raw[1])?]) // EINVAL
8. };
9. /* Resolve. */
10. let resolved = if path_uptr.is_null() {
11.   Path::from_fd(dirfd)?                                     // EBADF
12. } else {
13.   let kpath = Utimes::copy_path_from_user(path_uptr)?;     // EFAULT, ENAMETOOLONG
14.   let lf = if (flags & AT_SYMLINK_NOFOLLOW) != 0 { LookupFlags::empty() } else { LookupFlags::FOLLOW };
15.   Path::resolve_at(dirfd, &kpath, lf)?                     // EBADF, ENOENT, ELOOP
16. };
17. /* Permission. */
18. Utimes::check_perms(&resolved, kts.as_ref())?;             // EACCES, EPERM, EROFS
19. /* Apply. */
20. Utimes::vfs_utimes(&resolved, kts.as_ref())?;
21. Ok(0)

`Utimes::validate(ts) -> Result<KTimespec>`:
1. match ts.tv_nsec {
2.   UTIME_NOW | UTIME_OMIT => Ok(KTimespec { tv_sec: 0, tv_nsec: ts.tv_nsec }),
3.   0..=999_999_999       => Ok(KTimespec { tv_sec: ts.tv_sec, tv_nsec: ts.tv_nsec }),
4.   _                     => Err(EINVAL),
5. }

`Utimes::check_perms(path, kts) -> Result<()>`:
1. /* Both UTIME_OMIT: no perm check. */
2. if let Some([a, m]) = kts {
3.   if a.tv_nsec == UTIME_OMIT ∧ m.tv_nsec == UTIME_OMIT { return Ok(()); }
4. }
5. /* Owner or CAP_FOWNER: always allowed. */
6. if current_uid_owns(path.inode) ∨ capable(CAP_FOWNER) { return Ok(()); }
7. /* Else: only UTIME_NOW/NULL/equivalent allowed. */
8. let now_only = kts.is_none() ∨ matches!(kts, Some([a, m]) if (a.tv_nsec == UTIME_NOW ∨ a.tv_nsec == UTIME_OMIT) ∧ (m.tv_nsec == UTIME_NOW ∨ m.tv_nsec == UTIME_OMIT));
9. if !now_only { return Err(EPERM); }
10. /* And caller needs write perm. */
11. inode_permission(path.inode, MAY_WRITE)?;                  // EACCES
12. if path.mount.is_readonly() { return Err(EROFS); }
13. Ok(())

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_validated` | INVARIANT | unknown flag bits ⟹ EINVAL before work. |
| `tv_nsec_validated` | INVARIANT | every tv_nsec is in [0,10^9) or UTIME_NOW or UTIME_OMIT. |
| `owner_or_cap_fowner_for_explicit_times` | INVARIANT | non-NOW/OMIT times ⟹ owner or CAP_FOWNER. |
| `both_omit_is_noop` | INVARIANT | UTIME_OMIT/UTIME_OMIT skips perm check + mutation. |
| `ctime_updated_when_atime_or_mtime_set` | INVARIANT | ctime tracks any non-OMIT update. |

### Layer 2: TLA+

`fs/utimensat-syscall.tla`:
- States: per-flag-validate, per-times-copy, per-resolve, per-perm-check, per-vfs-utimes.
- Properties:
  - `safety_omit_omit_noop` — both UTIME_OMIT ⟹ no mutation.
  - `safety_owner_or_cap_for_explicit` — explicit times require owner/CAP_FOWNER.
  - `safety_now_under_write_perm` — UTIME_NOW/NULL requires MAY_WRITE.
  - `safety_ro_mount_blocks_update` — readonly mount ⟹ EROFS.
  - `liveness_utimensat_terminates` — every call returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_utimensat` post: ok ⟹ inode times set per kts | `Utimes::do_utimensat` |
| `validate` post: total mapping nsec → KTimespec ∨ EINVAL | `Utimes::validate` |
| `check_perms` post: enforces matrix above | `Utimes::check_perms` |

### Layer 4: Verus / Creusot functional

Per-`utimensat(2)` POSIX-2008 semantics; selftests in `tools/testing/selftests/timestamps/`.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`utimensat(2)` reinforcement:

- **Per-tv_nsec validation strict** — defense against per-out-of-range timestamp.
- **Per-UTIME_OMIT/UTIME_OMIT no-op** — defense against per-needless-perm-check side-effect.
- **Per-owner-or-CAP_FOWNER for explicit times** — defense against per-non-owner-mtime-forgery.
- **Per-NULL-times-needs-MAY_WRITE** — defense against per-touch-without-write-perm.
- **Per-mount-policy enforcement (ro)** — defense against per-mount-bypass.
- **Per-LSM inode_setattr hook** — defense against per-MAC-bypass.

## Grsecurity / PaX-style Reinforcement

- **PaX UDEREF on `path` and `times` copy_from_user** — defense against per-kernel-pointer params; SMAP forced.
- **UTIME_NOW / UTIME_OMIT sentinel validated** — defense against per-nsec-overflow info-leak; sentinels match exact constants and nothing else.
- **CAP_FOWNER strict on cross-userns** — defense against per-userns CAP_FOWNER escape; grsec enforces that CAP_FOWNER over an inode requires the cap in the fs's userns, not just any userns.
- **GRKERNSEC_LINK restricted symlink follow** — defense against per-utimensat-via-symlink in sticky world-writable dirs.
- **AT_SYMLINK_NOFOLLOW explicit** — defense against per-symlink-leaf-TOCTOU.
- **Per-ctime always updated on change** — defense against per-stat-anti-forensic (an attacker cannot silently update mtime without also touching ctime).
- **Per-mount-policy ro/noatime/lazytime honored** — defense against per-mount-bypass timestamp write.
- **PAX_USERCOPY_HARDEN on path + times copy** — bounded; whitelisted slab.
- **Per-dirfd ownership table strict** — defense against per-cross-process fd injection.
- **GRKERNSEC_PROC_USERGROUP info-leak guard** — `/proc/<pid>/*` utimensat from non-owner under `hidepid=4` is rejected at resolution.
- **Per-flags reserved bits rejected** — defense against per-future-flag smuggling.
- **GRKERNSEC_CHROOT consistency** — chrooted caller cannot reach files outside root.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `utime(2)` / `utimes(2)` legacy variants (covered in separate Tier-5 docs).
- `futimens(3)` libc wrapper (no separate syscall).
- y2038 32-bit `utimensat_time64` (covered in arch-compat docs).
- VFS setattr per-fs (covered in Tier-3 `fs/attr.md`).
- Implementation code.
