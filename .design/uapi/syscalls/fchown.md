# Tier-5 syscall: fchown(2) — syscall 93

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/open.c (SYSCALL_DEFINE3(fchown), chown_common via fd)
  - fs/attr.c (notify_change, setattr_should_drop_suidgid)
  - include/linux/fs.h (struct file, struct iattr)
  - arch/x86/entry/syscalls/syscall_64.tbl (93  common  fchown)
-->

## Summary

`fchown(2)` changes the owner (uid) and group (gid) of the file
referred to by the open file descriptor `fd`. It is the fd-based
counterpart of `chown(2)` and shares all semantics of ownership
change, suid/sgid stripping, file-capability removal, and quota
transfer.

Because the fd was opened against a specific inode, `fchown` operates
TOCTOU-free: no path walk happens between the syscall and the
ownership change. Used by atomic-create workflows (`open` → `fchown`
→ `rename`), package installers, container layer extraction tools.

Permission rules: same as `chown` — owner change requires `CAP_CHOWN`;
group change to a supplementary group is permitted to the owner;
non-CAP_FSETID caller triggers ATTR_KILL_SUID / ATTR_KILL_SGID; file
capabilities (`security.capability` xattr) are stripped on chown of
an executable.

The fd does not need write access; `O_RDONLY` is sufficient. `O_PATH`
fds are accepted by `fchownat(fd, "", AT_EMPTY_PATH)` but not by
`fchown` directly (kernel design: `fchown` calls `fchownat` internally
on some paths).

Critical for: race-free ownership change on newly-created files,
container layer materialization, atomic-update flows.

## Signature

```c
int fchown(int fd, uid_t owner, gid_t group);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `fd` | `int` | in | Open file descriptor (any mode). |
| `owner` | `uid_t` | in | New owner uid, or `(uid_t)-1`. |
| `group` | `gid_t` | in | New group gid, or `(gid_t)-1`. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Ownership updated. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EBADF` | `fd` not a valid open file descriptor. |
| `EPERM` | Same as chown (no CAP_CHOWN for owner change; group not in supp set without CAP_CHOWN; immutable file). |
| `EROFS` | Read-only mount. |
| `EIO`  | Metadata I/O error. |
| `EACCES` | LSM denial. |
| `EDQUOT` | Disk quota would be exceeded by transfer. |

## ABI surface

```text
__NR_fchown   (x86_64)  = 93
__NR_fchown   (i386)    = 207   /* fchown32 */
__NR_fchown   (arm64)   = 55
__NR_fchown   (riscv)   = 55
__NR_fchown   (generic) = 55
```

## Compatibility contract

REQ-1: Syscall **93** on x86_64; **55** on generic. ABI-stable.

REQ-2: Per-fd resolution: `fdget(fd)` → struct file. EBADF if invalid.

REQ-3: Per-sentinel: `(uid_t)-1` / `(gid_t)-1` ⟹ field skipped.

REQ-4: Per-CAP_CHOWN for owner change: any uid change ⟹ require
`CAP_CHOWN` in inode's user_ns.

REQ-5: Per-group permission: owner may set gid to a supplementary
group; otherwise CAP_CHOWN required.

REQ-6: Per-suid/sgid clear: non-CAP_FSETID caller ⟹ ATTR_KILL_SUID
plus (executable-only) ATTR_KILL_SGID added to iattr.

REQ-7: Per-file-cap-strip: chown of an executable removes the
`security.capability` xattr unconditionally on success.

REQ-8: Per-RO-mount: `mnt_want_write_file(f.file)` ⟹ `-EROFS`. Uses
the fd's mount, not the inode's filesystem.

REQ-9: Per-immutable: `S_IMMUTABLE` ⟹ `-EPERM` unless
`CAP_LINUX_IMMUTABLE`.

REQ-10: Per-LSM: `security_inode_setattr` invoked.

REQ-11: Per-notify_change with appropriate ATTR_* flags.

REQ-12: Per-ctime: touched on success.

REQ-13: Per-fsnotify: `FS_ATTRIB` emitted.

REQ-14: Per-audit: AUDIT_SYSCALL records fd, resolved path, uid/gid.

REQ-15: Per-quota: `dquot_transfer` for inode + block counts; over-
quota ⟹ `-EDQUOT`.

REQ-16: Per-memfd: fchown on memfd succeeds; owner = caller initially,
chown changes anonymous inode's uid/gid.

REQ-17: Per-idmap: mount idmap and user_ns mapping applied; host-side
uid persisted.

REQ-18: Per-pipefd/sockfd: fchown on a pipe or socket succeeds (POSIX
allows). Anonymous inode metadata updated.

## Acceptance Criteria

- [ ] AC-1: `fchown(fd, 1000, 1000)` as root: succeeds.
- [ ] AC-2: `fchown(-1, ...)`: `-EBADF`.
- [ ] AC-3: `fchown(fd, 1000, -1)` as non-root, owner of file: `-EPERM`.
- [ ] AC-4: `fchown(fd, -1, gid_in_supp)` as owner: succeeds.
- [ ] AC-5: `fchown(fd, -1, gid_NOT_in_supp)` without CAP_CHOWN: `-EPERM`.
- [ ] AC-6: fchown of suid binary by non-CAP_FSETID caller: setuid cleared.
- [ ] AC-7: fchown of executable with `security.capability`: xattr removed.
- [ ] AC-8: fchown on RO-mount fd: `-EROFS`.
- [ ] AC-9: fchown on immutable file: `-EPERM`.
- [ ] AC-10: ctime updated even when both args are `-1`.
- [ ] AC-11: Quota transfer observed.
- [ ] AC-12: Audit captures fd→path + old/new uid/gid.
- [ ] AC-13: fchown on memfd succeeds.

## Architecture

```rust
#[syscall(nr = 93, abi = "sysv")]
pub fn sys_fchown(fd: c_int, uid: u32, gid: u32) -> isize {
    Fs::do_fchown(fd, uid, gid)
}
```

`Fs::do_fchown(fd, uid, gid) -> isize`:
1. let f = fdget(fd)?;                                       /* EBADF */
2. let r = Fs::chown_common(&f.file.f_path, uid, gid);       /* shared */
3. fdput(f);
4. r

`Fs::chown_common(path, uid, gid)`: identical to chown's (see chown.md
REQ-section + Architecture). Notable per-fchown points:
- `mnt_want_write_file(f.file)` instead of path-based variant.
- Inode obtained via `f.file.f_inode`, TOCTOU-immune.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `fd_valid_first` | ORDER | EBADF before inode deref. |
| `sentinel_minus_one` | INVARIANT | -1 ⟹ field skipped. |
| `cap_chown_for_owner_change` | INVARIANT | uid change ⟹ CAP_CHOWN. |
| `suid_cleared_no_fsetid` | INVARIANT | non-CAP_FSETID ⟹ ATTR_KILL_SUID. |
| `file_cap_stripped` | INVARIANT | exec + chown ⟹ security.capability removed. |
| `mnt_write_file_balanced` | INVARIANT | mnt_want_write_file balanced. |

### Layer 2: TLA+

`fs/fchown.tla`:
- States: FDGET → BUILD_IATTR → PERM → SUID_FILTER → LSM → SETATTR → CAP_STRIP.
- Properties:
  - `safety_cap_chown_owner` — same as chown.
  - `safety_group_membership` — owner-only group within supp set.
  - `safety_suid_cleared` — non-CAP_FSETID ⟹ ATTR_KILL_SUID applied.
  - `safety_file_cap_stripped` — fcap gone after exec chown.
  - `safety_no_toctou` — inode pinned by fd.
  - `liveness_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_fchown` post: ia.ia_valid contains UID iff uid != -1; same for GID | `sys_fchown` |
| `do_fchown` post: fdget/fdput refcount balanced | `Fs::do_fchown` |
| `chown_common` post: ATTR_KILL_SUID present per REQ-6 | `Fs::chown_common` |
| `chown_common` post: security.capability stripped on success+executable | `Fs::chown_common` |

### Layer 4: Verus / Creusot functional

Per-`fchown(2)` man-page, per-POSIX, per-Linux fs/open.c, LTP `fchown01..05`.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`fchown(2)` reinforcement:

- **Per-fdget atomic** — defense against per-fd-table race.
- **Per-mnt_want_write_file** — defense against per-RO-mount overlay.
- **Per-CAP_CHOWN strict** — defense against per-DAC bypass.
- **Per-group-supp-set check** — defense against per-group-injection.
- **Per-suid/sgid drop** — defense against per-suid retention.
- **Per-file-cap strip** — defense against per-fcap retention.
- **Per-LSM hook** — defense against per-MAC bypass.
- **Per-quota dquot_transfer** — defense against per-quota-evasion.
- **Per-TOCTOU immunity** — defense against per-rename race.

## Grsecurity / PaX-style Reinforcement

- **PaX UDEREF** — fchown has no user pointer in its main path; UDEREF
  policy applies uniformly across fd-based syscalls.
- **CAP_CHOWN strict** — gradm RBAC may forbid CAP_CHOWN for non-policy
  subjects; default-deny ownership change.
- **CAP_FOWNER / CAP_SETID strict** — discrete RBAC capabilities.
- **GRKERNSEC_SUIDDUMP on suid-mode-change** — fchown that triggers
  ATTR_KILL_SUID/SGID logs sender uid, fd-resolved path, before/after
  mode. Coredump suppression recomputed.
- **GRKERNSEC_CHROOT_CHMOD** — chrooted process cannot fchown to a uid
  outside its mapped user_ns, regardless of fd origin (even if fd was
  opened pre-chroot).
- **GRKERNSEC_TPE on setuid-mode set** — fchown that would result in a
  setuid binary owned by an untrusted uid is policy-denied (combined
  with fchmod scheduling).
- **File-cap-strip on chown ENFORCED** — `security.capability` xattr
  removed unconditionally; logged with uid, path, prior capability set.
  Grsec rate-limits per-uid against fcap-strip flooding.
- **PaX MEMORY_SANITIZE on Iattr** — zero-init prevents leak of uninit
  ia_mode through ATTR_KILL paths.
- **PaX KERNEXEC** — `chown_common`, `notify_change`,
  `setattr_should_drop_suidgid` in read-only-after-init text.
- **PaX RANDSTRUCT on `struct iattr` and `struct file`** — randomized
  layout; corruption cannot deterministically swap ia_uid or f_inode.
- **GRKERNSEC_DMESG** — fchown failure klog rate-limited, CAP_SYSLOG-gated.
- **GRKERNSEC_HIDESYM** — pointer leaks scrubbed.
- **gradm policy: fchown REQUIRES policy** — discrete `o`/`O` RBAC mode.
- **GRKERNSEC_HARDEN_PTRACE** — tracee-driven fchown via fd table
  requires PTRACE_MODE_ATTACH_FSCREDS.
- **GRKERNSEC_PROC restrictions** — `/proc/<pid>/fd/N` uid-filtered;
  foreign fds cannot be retargeted into fchown.
- **memfd-chown policy** — combined with MFD_NOEXEC_SEAL, RBAC may
  forbid `S_ISUID` on memfd via the chown -> chmod path.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `chown(2)` (separate Tier-5 — path variant).
- `lchown(2)` (separate Tier-5 — no symlink deref).
- `fchownat(2)` (separate Tier-5 — at-relative).
- POSIX ACL chown propagation (separate Tier-5).
- Implementation code.
