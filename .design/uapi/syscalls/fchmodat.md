# Tier-5 syscall: fchmodat(2) — syscall 268

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/open.c (SYSCALL_DEFINE4(fchmodat2), SYSCALL_DEFINE3(fchmodat), do_fchmodat)
  - fs/namei.c (user_path_at, LOOKUP_AT_*)
  - fs/attr.c (notify_change)
  - include/uapi/linux/fcntl.h (AT_FDCWD, AT_SYMLINK_NOFOLLOW, AT_EMPTY_PATH)
  - arch/x86/entry/syscalls/syscall_64.tbl (268  common  fchmodat)
  - arch/x86/entry/syscalls/syscall_64.tbl (452  common  fchmodat2)
-->

## Summary

`fchmodat(2)` is the *at-relative* variant of `chmod(2)`: it resolves
`pathname` relative to an open directory fd `dirfd` (or AT_FDCWD for
cwd-relative), then changes the file's mode. The mode-change semantics
are identical to `chmod(2)` and `fchmod(2)`.

POSIX-2008 specifies a `flags` argument with `AT_SYMLINK_NOFOLLOW`,
but the Linux kernel's `fchmodat(2)` historically IGNORED `flags`
entirely — and the new `fchmodat2(2)` (syscall 452, since 6.6) added
proper `flags` handling. glibc 2.39+ uses `fchmodat2` transparently
where available and emulates `AT_SYMLINK_NOFOLLOW` via `O_PATH` +
`/proc/self/fd` for older kernels. This document covers the original
`fchmodat` (268) with notes on `fchmodat2`.

Critical for: openat-relative workflows (containers/sandboxes that
have a directory fd of the "real root"), atomic file creation patterns
(`openat` → `fchmodat` → `renameat`), portable POSIX code.

## Signature

```c
int fchmodat(int dirfd, const char *pathname, mode_t mode, int flags);
/* Linux 6.6+ also offers: */
int fchmodat2(int dirfd, const char *pathname, mode_t mode, int flags);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `dirfd` | `int` | in | Directory fd or `AT_FDCWD`. |
| `pathname` | `const char *` | in | Path relative to `dirfd`; absolute paths bypass `dirfd`. |
| `mode` | `mode_t` | in | New mode bits (low 12 used). |
| `flags` | `int` | in | `0` or `AT_SYMLINK_NOFOLLOW` (only honored by `fchmodat2`); `AT_EMPTY_PATH` available on `fchmodat2`. |

## Return value

| Value | Meaning |
|---|---|
| `0` | mode updated. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EFAULT` | `pathname` outside caller's address space. |
| `EBADF`  | `dirfd` not valid (and `pathname` is not absolute). |
| `ENOTDIR` | `dirfd` not a directory; or path component not directory. |
| `EINVAL` | `flags` has unsupported bits (fchmodat2 only; original fchmodat IGNORED flags). |
| `ENOENT` | path component missing. |
| `EACCES` | search permission denied. |
| `ELOOP` | symlink chain > 40. |
| `EPERM` | not owner and no CAP_FOWNER; or immutable; or symlink target on `AT_SYMLINK_NOFOLLOW` (kernel returns `EOPNOTSUPP` on chmod-of-symlink — Linux). |
| `EROFS` | read-only filesystem. |
| `ENAMETOOLONG` | path too long. |
| `EOPNOTSUPP` | `AT_SYMLINK_NOFOLLOW` set and path resolves to a symlink (Linux does not support chmod-on-symlink). |

## ABI surface

```text
__NR_fchmodat   (x86_64) = 268
__NR_fchmodat   (i386)   = 306
__NR_fchmodat   (arm64)  = 53
__NR_fchmodat   (riscv)  = 53
__NR_fchmodat   (generic)= 53
__NR_fchmodat2  (x86_64) = 452       /* since 6.6 */
__NR_fchmodat2  (generic)= 452
```

## Compatibility contract

REQ-1: Syscall number is **268** on x86_64; **53** on generic. ABI-stable.

REQ-2: Per-AT_FDCWD: `dirfd == AT_FDCWD (-100)` ⟹ resolve relative to
`current.fs.pwd`.

REQ-3: Per-absolute pathname: leading `/` ⟹ `dirfd` ignored (no EBADF
even if dirfd invalid).

REQ-4: Per-dirfd validation: non-absolute path + invalid dirfd ⟹ EBADF.
non-absolute path + dirfd-is-not-dir ⟹ ENOTDIR.

REQ-5: Per-flags=0: lookup with `LOOKUP_FOLLOW` (final symlink chased).

REQ-6: Per-AT_SYMLINK_NOFOLLOW (fchmodat2 only): lookup with
`LOOKUP_FOLLOW=0`; if target is symlink ⟹ EOPNOTSUPP (Linux does not
support chmod on symlinks — symlink modes are unused).

REQ-7: Per-AT_EMPTY_PATH (fchmodat2 only): if `pathname == ""` and
caller has `CAP_DAC_READ_SEARCH`, operate on `dirfd` itself (acts like
`fchmod(dirfd)`).

REQ-8: Per-original-fchmodat ignored-flags: old kernels silently
accept any `flags` value; glibc had to emulate AT_SYMLINK_NOFOLLOW.

REQ-9: Per-mode masking: low 12 bits only.

REQ-10: Per-owner-check: same as chmod — owner OR CAP_FOWNER.

REQ-11: Per-setgid-restrict: silently clear S_ISGID if not in group set
and no CAP_FSETID.

REQ-12: Per-RO-mount: EROFS from `mnt_want_write`.

REQ-13: Per-LSM: `security_inode_setattr` hook invoked.

REQ-14: Per-notify_change with `ATTR_MODE | ATTR_CTIME`.

REQ-15: Per-fsnotify: `FS_ATTRIB` post-success.

REQ-16: Per-audit: AUDIT_SYSCALL records dirfd, pathname, mode, flags.

REQ-17: Per-idmap: mount idmap applied to ownership check.

## Acceptance Criteria

- [ ] AC-1: `fchmodat(AT_FDCWD, "f", 0644, 0)` on owned file: succeeds.
- [ ] AC-2: `fchmodat(dirfd, "subdir/f", 0644, 0)` resolves under dirfd.
- [ ] AC-3: Absolute path with invalid dirfd: succeeds (dirfd ignored).
- [ ] AC-4: Non-absolute path with invalid dirfd: `-EBADF`.
- [ ] AC-5: dirfd points at a regular file, non-absolute pathname: `-ENOTDIR`.
- [ ] AC-6: `fchmodat2` with `AT_SYMLINK_NOFOLLOW`, target is symlink: `-EOPNOTSUPP`.
- [ ] AC-7: Original `fchmodat` ignores `flags`; symlink dereferenced.
- [ ] AC-8: `fchmodat2(dirfd, "", 0644, AT_EMPTY_PATH)` with CAP_DAC_READ_SEARCH: succeeds (operates on dirfd inode).
- [ ] AC-9: `fchmodat(_, _, 0, 0xDEADBEEF)`: returns 0 on old fchmodat (ignored flags); `-EINVAL` on fchmodat2.
- [ ] AC-10: Owner check identical to chmod (EPERM, setgid silent-drop).
- [ ] AC-11: Per-LSM denial: `-EACCES`.
- [ ] AC-12: Mount idmap honored in owner check.

## Architecture

```rust
#[syscall(nr = 268, abi = "sysv")]
pub fn sys_fchmodat(dirfd: c_int, path: UserPtr<u8>, mode: u16) -> isize {
    /* Linux 3-arg fchmodat: flags-less */
    Fs::do_fchmodat(dirfd, path, mode, 0)
}

#[syscall(nr = 452, abi = "sysv")]
pub fn sys_fchmodat2(dirfd: c_int, path: UserPtr<u8>, mode: u16, flags: c_int) -> isize {
    if flags & !(AT_SYMLINK_NOFOLLOW | AT_EMPTY_PATH) != 0 { return -EINVAL; }
    Fs::do_fchmodat(dirfd, path, mode, flags)
}
```

`Fs::do_fchmodat(dirfd, path, mode, flags) -> isize`:
1. let name = getname_flags(path, (flags & AT_EMPTY_PATH))?;       /* EFAULT, ENAMETOOLONG */
2. let lookup_flags = if (flags & AT_SYMLINK_NOFOLLOW) != 0 { 0 } else { LOOKUP_FOLLOW };
3. let p = user_path_at(dirfd, &name, lookup_flags | LOOKUP_AT_EMPTY_PATH_if(flags))?;
4. /* Linux: chmod-on-symlink not supported */
5. if (flags & AT_SYMLINK_NOFOLLOW) && p.dentry.d_inode().is_lnk() {
6.     path_put(p); putname(name); return -EOPNOTSUPP;
7. }
8. let r = Fs::chmod_common(&p, mode & 0o7777);
9. path_put(p); putname(name);
10. r

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_validated_fchmodat2` | INVARIANT | fchmodat2 rejects unknown flag bits with EINVAL. |
| `nofollow_symlink_eopnotsupp` | INVARIANT | AT_SYMLINK_NOFOLLOW + symlink target ⟹ EOPNOTSUPP. |
| `absolute_path_ignores_dirfd` | INVARIANT | absolute pathname bypasses dirfd validation. |
| `at_empty_path_caps` | INVARIANT | AT_EMPTY_PATH requires CAP_DAC_READ_SEARCH. |
| `mode_masked_07777` | INVARIANT | Only low 12 bits applied. |

### Layer 2: TLA+

`fs/fchmodat.tla`:
- States: PARSE_FLAGS → DIRFD_RESOLVE → LOOKUP (FOLLOW or not) → SYMLINK_GUARD → OWNER → SETGID_FILTER → LSM → SETATTR.
- Properties:
  - `safety_no_chmod_on_symlink` — NOFOLLOW path never reaches notify_change.
  - `safety_owner_or_capable` — owner-only or CAP_FOWNER.
  - `safety_flags_strict_fchmodat2` — invalid flags rejected before lookup.
  - `liveness_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_fchmodat` post: inode.mode updated; symlink-final never modified | `sys_fchmodat` |
| `sys_fchmodat2` post: flag validation precedes lookup | `sys_fchmodat2` |
| `do_fchmodat` post: chmod_common semantics hold | `Fs::do_fchmodat` |

### Layer 4: Verus / Creusot functional

Per-`fchmodat(2)` man-page, per-POSIX, per-Linux fs/open.c, LTP `fchmodat01..05`.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`fchmodat(2)` reinforcement:

- **Per-getname PAX_USERCOPY** — defense against per-string overflow.
- **Per-dirfd validation atomic** — defense against per-fd-race.
- **Per-AT_SYMLINK_NOFOLLOW EOPNOTSUPP** — defense against per-symlink-
  mode confusion.
- **Per-AT_EMPTY_PATH CAP_DAC_READ_SEARCH** — defense against per-empty-
  path inode access by unprivileged.
- **Per-flags strict (fchmodat2)** — defense against per-future-flag
  smuggling.
- **Per-LSM hook** — defense against per-MAC bypass.

## Grsecurity / PaX-style Reinforcement

- **PaX UDEREF on `pathname`** — getname under SMAP+UDEREF; user pointer
  cannot point into kernel memory.
- **CAP_FOWNER strict** — RBAC may forbid CAP_FOWNER for non-policy
  subjects; default-deny for suid creation.
- **GRKERNSEC_SUIDDUMP on suid-mode-change** — `S_ISUID` / `S_ISGID`
  add/remove logged with dirfd-resolved path, sender uid, old/new mode.
- **GRKERNSEC_CHROOT_CHMOD** — chrooted process cannot fchmodat-set
  `S_ISUID` / `S_ISGID` inside its chroot, regardless of dirfd origin
  (even if dirfd was opened pre-chroot, the dirfd-relative resolution
  is still confined to the chroot tree). Setuid bits silently cleared.
- **GRKERNSEC_TPE on setuid-mode set** — fchmodat setting `S_ISUID` on
  a binary whose containing directory is world/group-writable returns
  `-EPERM`. TPE_GID-aware.
- **File-cap-strip on chown (cross-reference)** — a subsequent
  fchownat will strip file capabilities via setattr_should_drop_suidgid;
  rate-limited.
- **GRKERNSEC_LINK on symlink path** — fchmodat traversal through a
  symlink owned by a different uid is rejected when the destination is
  owned by a different uid (defaults: yes in /tmp-like sticky dirs).
- **PaX MEMORY_SANITIZE on Iattr** — zero-init; no leak.
- **GRKERNSEC_PROC restrictions** — `/proc/<pid>/fd/<dirfd>` resolution
  uid-filtered; foreign dirfd cannot be retargeted.
- **PaX KERNEXEC** — `do_fchmodat`, `chmod_common`, `notify_change` in
  read-only-after-init text.
- **PaX RANDSTRUCT on `struct iattr` and `struct path`** — randomized
  layout; corruption cannot deterministically swap ia_mode or dentry.
- **gradm policy: setuid-bit set REQUIRES policy** — discrete `O` mode
  RBAC capability; subjects without policy cannot add `S_ISUID` even
  as root, regardless of which chmod variant is used.
- **GRKERNSEC_DMESG** — fchmodat failure klog rate-limited and
  CAP_SYSLOG-gated.
- **GRKERNSEC_HIDESYM** — pointer leaks scrubbed.
- **GRKERNSEC_HARDEN_PTRACE** — fchmodat via tracee's fd table requires
  PTRACE_MODE_ATTACH_FSCREDS.
- **AT_EMPTY_PATH hardening** — even with CAP_DAC_READ_SEARCH, grsec
  policy may forbid AT_EMPTY_PATH for unprivileged subjects, blocking
  the "fd-to-inode-without-path" disclosure pattern.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `chmod(2)` (separate Tier-5).
- `fchmod(2)` (separate Tier-5).
- `fchmodat2(2)` flag-extension details (covered inline).
- `fchownat(2)` (separate Tier-5).
- Implementation code.
