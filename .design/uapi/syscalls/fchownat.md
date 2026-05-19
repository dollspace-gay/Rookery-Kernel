# Tier-5 syscall: fchownat(2) — syscall 260

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/open.c (SYSCALL_DEFINE5(fchownat), do_fchownat, chown_common)
  - fs/namei.c (user_path_at, LOOKUP_AT_EMPTY_PATH)
  - fs/attr.c (notify_change, setattr_should_drop_suidgid)
  - include/uapi/linux/fcntl.h (AT_FDCWD, AT_SYMLINK_NOFOLLOW, AT_EMPTY_PATH)
  - arch/x86/entry/syscalls/syscall_64.tbl (260  common  fchownat)
-->

## Summary

`fchownat(2)` is the *at-relative* and *flag-aware* variant of
`chown(2)`. It resolves `pathname` relative to a directory fd
`dirfd` (or AT_FDCWD), with flags controlling symlink behavior:
`AT_SYMLINK_NOFOLLOW` selects `lchown` semantics, `AT_EMPTY_PATH`
selects fd-direct semantics (like `fchown` but accepting `O_PATH`).

`fchownat` is the modern POSIX-canonical interface; on architectures
without legacy `chown(2)`/`lchown(2)`/`fchown(2)` (arm64, riscv), it
is the ONLY ownership-change syscall. glibc implements all three
classic variants on top of `fchownat` where present.

Same permission and side-effect semantics as `chown`: `CAP_CHOWN` for
owner change, group-in-supp-set for owner-driven group change,
suid/sgid stripping by non-CAP_FSETID caller, `security.capability`
xattr removal on chown of an executable, quota transfer, LSM hook,
fsnotify, audit, idmap.

Critical for: container/sandbox runtimes that hold "real root" dirfd,
atomic install patterns, race-free per-tree ownership reset.

## Signature

```c
int fchownat(int dirfd, const char *pathname,
             uid_t owner, gid_t group, int flags);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `dirfd` | `int` | in | Directory fd or `AT_FDCWD`. |
| `pathname` | `const char *` | in | Path relative to `dirfd`; absolute bypasses dirfd. |
| `owner` | `uid_t` | in | New uid or `(uid_t)-1`. |
| `group` | `gid_t` | in | New gid or `(gid_t)-1`. |
| `flags` | `int` | in | Combination of `AT_SYMLINK_NOFOLLOW`, `AT_EMPTY_PATH`. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Ownership updated. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EFAULT` | `pathname` outside caller's address space. |
| `EBADF` | `dirfd` invalid (and path non-absolute). |
| `ENOTDIR` | `dirfd` not a directory; or path component not directory. |
| `EINVAL` | `flags` has unknown bits. |
| `ENOENT` | path missing (and `pathname != ""` or no `AT_EMPTY_PATH`). |
| `EACCES` | search denied. |
| `ELOOP` | symlink loop. |
| `ENAMETOOLONG` | path too long. |
| `EPERM` | no CAP_CHOWN for owner change; group not in supp set without CAP_CHOWN; immutable. |
| `EROFS` | read-only fs. |
| `EIO`  | metadata I/O error. |

## ABI surface

```text
__NR_fchownat (x86_64)  = 260
__NR_fchownat (i386)    = 298
__NR_fchownat (arm64)   = 54
__NR_fchownat (riscv)   = 54
__NR_fchownat (generic) = 54
```

## Compatibility contract

REQ-1: Syscall number is **260** on x86_64; **54** on generic. ABI-stable.

REQ-2: Per-AT_FDCWD: resolution relative to `current.fs.pwd`.

REQ-3: Per-absolute pathname: dirfd ignored; no EBADF on bad dirfd.

REQ-4: Per-dirfd validation: non-absolute path + invalid dirfd ⟹ EBADF.

REQ-5: Per-flags strict: `flags & ~(AT_SYMLINK_NOFOLLOW|AT_EMPTY_PATH) != 0`
⟹ `-EINVAL`.

REQ-6: Per-AT_SYMLINK_NOFOLLOW: lookup with `LOOKUP_FOLLOW=0`; final
symlink NOT dereferenced — the link itself is chowned. This is the
"lchown" semantic.

REQ-7: Per-AT_EMPTY_PATH: if `pathname == ""` and `AT_EMPTY_PATH` set,
operate on `dirfd` itself (acts like `fchown`). Caller must have
`CAP_DAC_READ_SEARCH` for foreign-uid dirfd cases.

REQ-8: Per-sentinel: `(uid_t)-1` / `(gid_t)-1` ⟹ field skipped.

REQ-9: Per-CAP_CHOWN: owner change ⟹ CAP_CHOWN in inode's user_ns.

REQ-10: Per-group permission: owner-driven gid change limited to caller's
supp set (or egid); else CAP_CHOWN required.

REQ-11: Per-suid/sgid clear: non-CAP_FSETID caller triggers ATTR_KILL_SUID
+ (executable only) ATTR_KILL_SGID. Note for AT_SYMLINK_NOFOLLOW on a
symlink: the symlink has no mode bits to clear.

REQ-12: Per-file-cap-strip: chown of an executable removes
`security.capability` xattr unconditionally on success. NOT applied to
symlink-direct chowns.

REQ-13: Per-RO-mount: EROFS from `mnt_want_write`.

REQ-14: Per-immutable: EPERM unless CAP_LINUX_IMMUTABLE.

REQ-15: Per-LSM: `security_inode_setattr` invoked.

REQ-16: Per-notify_change with ATTR_UID/ATTR_GID/ATTR_CTIME and
optionally ATTR_KILL_SUID/SGID.

REQ-17: Per-fsnotify: `FS_ATTRIB`.

REQ-18: Per-audit: AUDIT_SYSCALL records dirfd, path, owner, group, flags.

REQ-19: Per-idmap: mount idmap and user_ns mapping applied.

REQ-20: Per-quota: `dquot_transfer`.

## Acceptance Criteria

- [ ] AC-1: `fchownat(AT_FDCWD, "f", 1000, 1000, 0)`: succeeds.
- [ ] AC-2: `fchownat(dirfd, "sub/f", uid, gid, 0)`: resolves under dirfd.
- [ ] AC-3: Absolute path with invalid dirfd: succeeds (dirfd ignored).
- [ ] AC-4: Non-absolute path + invalid dirfd: `-EBADF`.
- [ ] AC-5: `fchownat(_, _, _, _, 0x100000)`: `-EINVAL`.
- [ ] AC-6: `fchownat(_, "f", uid, gid, AT_SYMLINK_NOFOLLOW)` on symlink: symlink itself chowned; target unchanged.
- [ ] AC-7: `fchownat(dirfd, "", uid, gid, AT_EMPTY_PATH)` w/ CAP_DAC_READ_SEARCH: operates on dirfd inode.
- [ ] AC-8: `fchownat(dirfd, "", uid, gid, AT_EMPTY_PATH)` no CAP: EPERM or EACCES per LSM/policy.
- [ ] AC-9: `fchownat` of suid binary by non-CAP_FSETID: setuid cleared.
- [ ] AC-10: `fchownat` of executable: `security.capability` xattr removed.
- [ ] AC-11: Owner-change without CAP_CHOWN: `-EPERM`.
- [ ] AC-12: Idmap honored.
- [ ] AC-13: Audit captures dirfd, path, flags, old/new uid+gid.

## Architecture

```rust
#[syscall(nr = 260, abi = "sysv")]
pub fn sys_fchownat(dirfd: c_int, path: UserPtr<u8>,
                    uid: u32, gid: u32, flags: c_int) -> isize {
    if flags & !(AT_SYMLINK_NOFOLLOW | AT_EMPTY_PATH) != 0 { return -EINVAL; }
    Fs::do_fchownat(dirfd, path, uid, gid, flags)
}
```

`Fs::do_fchownat(dfd, path, uid, gid, flags) -> isize`:
1. let name = getname_flags(path, (flags & AT_EMPTY_PATH))?;       /* EFAULT, ENAMETOOLONG */
2. let lookup = if (flags & AT_SYMLINK_NOFOLLOW) != 0 { 0 } else { LOOKUP_FOLLOW };
3. let lookup = lookup | (if (flags & AT_EMPTY_PATH) != 0 { LOOKUP_EMPTY } else { 0 });
4. let p = user_path_at(dfd, &name, lookup)?;
5. let r = Fs::chown_common(&p, uid, gid);                          /* shared */
6. path_put(p); putname(name);
7. r

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_strict` | INVARIANT | unknown flag bits ⟹ EINVAL. |
| `nofollow_targets_link` | INVARIANT | AT_SYMLINK_NOFOLLOW ⟹ chown applies to symlink. |
| `empty_path_caps` | INVARIANT | AT_EMPTY_PATH + foreign-uid dirfd ⟹ CAP_DAC_READ_SEARCH. |
| `cap_chown_for_owner_change` | INVARIANT | uid change ⟹ CAP_CHOWN. |
| `suid_cleared_no_fsetid` | INVARIANT | non-CAP_FSETID ⟹ ATTR_KILL_SUID. |
| `file_cap_stripped` | INVARIANT | regular-file + executable + chown ⟹ security.capability removed. |
| `link_chown_no_fcap` | INVARIANT | symlink chown does NOT touch security.capability. |

### Layer 2: TLA+

`fs/fchownat.tla`:
- States: PARSE_FLAGS → DIRFD_RESOLVE → LOOKUP (FOLLOW vs NOT) → BUILD_IATTR → PERM → SUID_FILTER → LSM → SETATTR → CAP_STRIP.
- Properties:
  - `safety_cap_chown_owner` — uid change only with CAP_CHOWN.
  - `safety_flags_strict` — invalid flag bits rejected.
  - `safety_nofollow_link` — AT_SYMLINK_NOFOLLOW chowns the link.
  - `safety_suid_cleared` — non-CAP_FSETID ⟹ ATTR_KILL_SUID.
  - `safety_file_cap_stripped` — fcap gone after exec chown.
  - `liveness_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_fchownat` post: flags validated BEFORE lookup | `sys_fchownat` |
| `do_fchownat` post: AT_SYMLINK_NOFOLLOW ⟹ lookup_flags omits FOLLOW | `Fs::do_fchownat` |
| `chown_common` post: ATTR_KILL_SUID per REQ-11 | `Fs::chown_common` |
| `chown_common` post: security.capability stripped per REQ-12 | `Fs::chown_common` |

### Layer 4: Verus / Creusot functional

Per-`fchownat(2)` man-page, per-POSIX, per-Linux fs/open.c + fs/attr.c, LTP `fchownat01..03`.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`fchownat(2)` reinforcement:

- **Per-getname PAX_USERCOPY** — defense against per-string overflow.
- **Per-dirfd validation atomic** — defense against per-fd-race.
- **Per-AT_SYMLINK_NOFOLLOW correctly targets link** — defense against
  per-symlink-target chown bypass.
- **Per-AT_EMPTY_PATH CAP_DAC_READ_SEARCH** — defense against per-empty-
  path inode access by unprivileged.
- **Per-flags strict** — defense against per-future-flag smuggling.
- **Per-CAP_CHOWN strict** — defense against per-DAC bypass.
- **Per-group-supp-set check** — defense against per-group-injection.
- **Per-suid/sgid drop** — defense against per-suid retention.
- **Per-file-cap strip** — defense against per-fcap retention.
- **Per-LSM hook** — defense against per-MAC bypass.

## Grsecurity / PaX-style Reinforcement

- **PaX UDEREF on `pathname`** — getname under SMAP+UDEREF.
- **CAP_CHOWN strict** — gradm RBAC may forbid CAP_CHOWN for non-policy
  subjects.
- **CAP_FOWNER / CAP_SETID strict** — discrete RBAC capabilities.
- **GRKERNSEC_SUIDDUMP on suid-mode-change** — fchownat that triggers
  ATTR_KILL_SUID/SGID logs sender uid, dirfd-resolved path, flags,
  before/after mode. Coredump suppression recomputed.
- **GRKERNSEC_CHROOT_CHMOD** — chrooted process cannot fchownat to a
  uid outside its mapped user_ns, regardless of dirfd origin. Cross-
  user_ns chown silently denied even with CAP_CHOWN if RBAC lacks
  chroot-chown.
- **GRKERNSEC_TPE on setuid-mode set** — fchownat that would create a
  setuid binary owned by an untrusted uid is policy-denied at chown
  time (combined with subsequent fchmodat sequencing).
- **File-cap-strip on chown ENFORCED** — `security.capability` xattr
  removed unconditionally on regular-file chown. Symlink-direct chown
  via AT_SYMLINK_NOFOLLOW does not touch the xattr (symlinks cannot
  carry file caps). Grsec logs the strip with uid, path, prior caps.
- **GRKERNSEC_LINK on symlink-traversal chown** — fchownat traversal
  through a symlink owned by a different uid denied per sticky-dir
  rules (unless AT_SYMLINK_NOFOLLOW which targets the link itself).
- **PaX MEMORY_SANITIZE on Iattr** — zero-init.
- **PaX KERNEXEC** — do_fchownat, chown_common, notify_change in
  read-only-after-init text.
- **PaX RANDSTRUCT on `struct iattr` and `struct path`** — randomized.
- **GRKERNSEC_DMESG** — fchownat failure klog rate-limited, CAP_SYSLOG-
  gated.
- **GRKERNSEC_HIDESYM** — pointer leaks scrubbed.
- **gradm policy: fchownat REQUIRES policy** — discrete `o`/`O` RBAC
  mode; subjects without it cannot reparent files even as root.
- **GRKERNSEC_HARDEN_PTRACE** — tracee-driven fchownat via fd table
  requires PTRACE_MODE_ATTACH_FSCREDS.
- **AT_EMPTY_PATH hardening** — even with CAP_DAC_READ_SEARCH, RBAC
  policy may forbid AT_EMPTY_PATH for unprivileged subjects, blocking
  the "dirfd-to-arbitrary-chown" pattern.
- **Idmap policy** — mount idmap and user_ns translation audited; an
  attempt to chown across user_ns boundaries denied unless RBAC grants
  cross-namespace `O` mode.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `chown(2)` (separate Tier-5).
- `fchown(2)` (separate Tier-5).
- `lchown(2)` (separate Tier-5).
- POSIX ACL propagation (separate Tier-5).
- Implementation code.
