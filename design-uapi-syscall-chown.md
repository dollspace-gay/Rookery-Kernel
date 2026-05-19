---
title: "Tier-5 syscall: chown(2) — syscall 92"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`chown(2)` changes the owner (uid) and group (gid) of the file
identified by `path`. Symbolic links are dereferenced (`lchown(2)`
modifies the link itself; `fchownat` with `AT_SYMLINK_NOFOLLOW` is the
modern path-based variant).

Changing the *owner* of a file requires `CAP_CHOWN` in the file's
user_ns. Changing the *group* of a file is permitted to the file's
owner ONLY if the new group is one of the caller's supplementary
groups (or the caller's egid), AND the caller is the file owner — or
the caller holds `CAP_CHOWN`. Passing `-1` (as `uid_t`/`gid_t`) for
either argument means "do not change".

As a security side-effect, a successful chown clears the setuid and
(if executable) setgid bits when the operation is performed by a
non-CAP_FSETID caller. The kernel also strips any file capabilities
(`security.capability` xattr) on chown of an executable.

Critical for: package installers, container layer extraction, /home
setup, the entire UNIX permission model.

### Acceptance Criteria

- [ ] AC-1: `chown("/tmp/f", 1000, 1000)` as root: succeeds.
- [ ] AC-2: `chown(path, 1000, -1)` as non-root, file owned by caller: `-EPERM` (owner change requires CAP_CHOWN).
- [ ] AC-3: `chown(path, -1, gid)` where gid in caller's supplementary set + file owned by caller: succeeds.
- [ ] AC-4: `chown(path, -1, gid)` where gid NOT in supplementary set + no CAP_CHOWN: `-EPERM`.
- [ ] AC-5: chown of suid binary by non-CAP_FSETID caller: setuid bit cleared in resulting mode.
- [ ] AC-6: chown of executable file with `security.capability`: xattr removed.
- [ ] AC-7: `chown` on symlink: follows; target's owner changed; symlink owner unchanged.
- [ ] AC-8: `chown` on RO-fs: `-EROFS`.
- [ ] AC-9: `chown` on immutable file: `-EPERM`.
- [ ] AC-10: ctime updated even when both args are `(uid_t)-1`/`(gid_t)-1`.
- [ ] AC-11: Quota transfer observed on filesystem with quota enabled.
- [ ] AC-12: Audit captures old/new uid+gid.
- [ ] AC-13: Idmapped mount: chown writes host-side uid after mapping.

### Architecture

```rust
#[syscall(nr = 92, abi = "sysv")]
pub fn sys_chown(path: UserPtr<u8>, uid: u32, gid: u32) -> isize {
    Fs::do_fchownat(AT_FDCWD, path, uid, gid, 0)
}
```

`Fs::do_fchownat(dfd, path, uid, gid, flags) -> isize`:
1. let name = getname(path)?;
2. let lookup = if (flags & AT_SYMLINK_NOFOLLOW) != 0 { 0 } else { LOOKUP_FOLLOW };
3. let p = user_path_at(dfd, &name, lookup)?;
4. let r = Fs::chown_common(&p, uid, gid);
5. path_put(p); putname(name);
6. r

`Fs::chown_common(path, uid, gid) -> isize`:
1. let inode = path.dentry.d_inode();
2. let mnt_w = mnt_want_write(path.mnt)?;
3. let mut ia = Iattr { ia_valid: ATTR_CTIME, ia_ctime: now(), .. Default::default() };
4. /* Owner */
5. if uid != u32::MAX { ia.ia_valid |= ATTR_UID; ia.ia_uid = map_uid(path.mnt_idmap, uid); }
6. if gid != u32::MAX { ia.ia_valid |= ATTR_GID; ia.ia_gid = map_gid(path.mnt_idmap, gid); }
7. /* Permission */
8. let perm = Fs::check_chown_perm(inode, &ia)?;       /* EPERM */
9. /* SUID/SGID clear */
10. if setattr_should_drop_suidgid(inode, &ia) { ia.ia_valid |= ATTR_KILL_SUID; if inode.mode & S_IXGRP != 0 { ia.ia_valid |= ATTR_KILL_SGID; } }
11. /* LSM */
12. security_inode_setattr(path.dentry, &ia)?;
13. inode_lock(inode);
14. let r = notify_change(path.mnt_idmap, path.dentry, &ia, &delegated);
15. inode_unlock(inode);
16. /* Strip security.capability */
17. if r == 0 && inode.mode & S_IXANY != 0 { vfs_remove_acl(inode, "security.capability"); }
18. mnt_drop_write(mnt_w);
19. if r == 0 { fsnotify_change(path.dentry, FS_ATTRIB); }
20. r

### Out of Scope

- `fchown(2)` (separate Tier-5 — fd variant).
- `lchown(2)` (separate Tier-5 — no symlink dereference).
- `fchownat(2)` (separate Tier-5 — at-relative variant).
- POSIX ACL handling (separate Tier-5).
- Implementation code.

### signature

```c
int chown(const char *path, uid_t owner, gid_t group);
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `path` | `const char *` | in | NUL-terminated path. Symlinks followed. |
| `owner` | `uid_t` | in | New owner uid, or `(uid_t)-1` to leave unchanged. |
| `group` | `gid_t` | in | New group gid, or `(gid_t)-1` to leave unchanged. |

### return value

| Value | Meaning |
|---|---|
| `0` | Ownership updated. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EFAULT` | `path` outside caller's address space. |
| `ENOENT` | `path` does not exist. |
| `ENOTDIR` | A path component is not a directory. |
| `EACCES` | Search permission denied on a component. |
| `ELOOP` | Symlink chain too long. |
| `ENAMETOOLONG` | Path too long. |
| `EPERM` | Caller lacks `CAP_CHOWN` for owner change; or new group not in supplementary set without CAP_CHOWN; or file is immutable. |
| `EROFS` | Read-only filesystem. |
| `EIO` | I/O error writing metadata. |
| `ENOMEM` | Out of memory. |

### abi surface

```text
__NR_chown   (x86_64) = 92
__NR_chown   (i386)   = 182    /* "chown32" — 32-bit uid/gid */
__NR_chown   (arm64)  — n/a; use fchownat (260)
__NR_chown   (riscv)  — n/a; use fchownat
```

### compatibility contract

REQ-1: Syscall number is **92** on x86_64. ABI-stable since v1.

REQ-2: Per-path resolution: `user_path_at(AT_FDCWD, path, LOOKUP_FOLLOW)`.
Symlinks dereferenced; final inode targeted.

REQ-3: Per-sentinel: `(uid_t)-1` or `(gid_t)-1` ⟹ field unchanged;
`iattr.ia_valid` reflects exactly which fields are being set.

REQ-4: Per-CAP_CHOWN for owner-change: any change of `inode.uid`
requires `CAP_CHOWN` in inode's user_ns.

REQ-5: Per-group-change permission: caller may set group iff
(a) caller owns file AND new group is in caller's supplementary set or
equals caller's egid, OR (b) caller has `CAP_CHOWN`. Otherwise `-EPERM`.

REQ-6: Per-suid/sgid clearing on chown: if the chown is performed by
a non-CAP_FSETID caller, ATTR_KILL_SUID and (for executables only)
ATTR_KILL_SGID are added to iattr. This drops the setuid/setgid bits
atomically with the ownership change. (See `setattr_should_drop_suidgid`.)

REQ-7: Per-file-capability-strip: a chown of an executable strips the
`security.capability` xattr unconditionally. This is part of the
inode's `setattr` finalization, not the iattr flow.

REQ-8: Per-RO-fs: `mnt_want_write` ⟹ `-EROFS` on RO mount.

REQ-9: Per-immutable: `S_IMMUTABLE` ⟹ `-EPERM` unless
`CAP_LINUX_IMMUTABLE`.

REQ-10: Per-LSM: `security_inode_setattr` invoked pre-change.

REQ-11: Per-notify_change with `ATTR_UID|ATTR_GID|ATTR_CTIME` plus
`ATTR_KILL_SUID|ATTR_KILL_SGID` as appropriate.

REQ-12: Per-ctime-update: ctime touched on success (even if uid/gid
are no-ops).

REQ-13: Per-fsnotify: `FS_ATTRIB` emitted post-success.

REQ-14: Per-audit: `AUDIT_SYSCALL` records old owner, new owner, old
group, new group, path.

REQ-15: Per-idmap: mount idmap and user_ns mapping applied — chown
writes the *unmapped* (host) uid into the inode after translation.

REQ-16: Per-disk-quota: on filesystems with quota enabled, transfer of
inode + disk-block counts from old owner's quota to new owner's quota
under `dquot_transfer`.

REQ-17: Per-cross-uid: chown to a different uid never preserves suid
when caller lacks CAP_FSETID — even if the target uid was the prior
owner of the suid bit.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `sentinel_minus_one` | INVARIANT | (uid_t)-1 / (gid_t)-1 ⟹ field not in ia_valid. |
| `cap_chown_for_owner_change` | INVARIANT | uid change ⟹ CAP_CHOWN present. |
| `group_in_supp_or_cap` | INVARIANT | gid change by owner ⟹ in supp set, else CAP_CHOWN. |
| `suid_cleared_no_fsetid` | INVARIANT | chown w/o CAP_FSETID ⟹ ATTR_KILL_SUID set. |
| `file_cap_stripped` | INVARIANT | executable + chown ⟹ security.capability removed. |

### Layer 2: TLA+

`fs/chown.tla`:
- States: LOOKUP → MNT_WRITE → BUILD_IATTR → PERM → SUID_FILTER → LSM → SETATTR → CAP_STRIP → FSNOTIFY.
- Properties:
  - `safety_cap_chown_owner` — uid change only with CAP_CHOWN.
  - `safety_group_membership` — gid change limited to supp-set or CAP_CHOWN.
  - `safety_suid_cleared` — no leftover suid after non-CAP_FSETID chown.
  - `safety_file_cap_stripped` — file caps gone after chown of exec.
  - `liveness_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_fchownat` post: inode.uid/.gid updated per iattr | `Fs::do_fchownat` |
| `chown_common` post: ATTR_KILL_SUID present when required | `Fs::chown_common` |
| `chown_common` post: security.capability stripped on success+executable | `Fs::chown_common` |

### Layer 4: Verus / Creusot functional

Per-`chown(2)` man-page, per-POSIX, per-Linux fs/open.c + fs/attr.c, LTP `chown01..05`.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`chown(2)` reinforcement:

- **Per-getname PAX_USERCOPY** — defense against per-string overflow.
- **Per-LOOKUP_FOLLOW + mnt_want_write** — defense against per-RO-fs bypass.
- **Per-CAP_CHOWN strict** — defense against per-DAC bypass.
- **Per-group-supp-set check** — defense against per-group-injection.
- **Per-setattr_should_drop_suidgid** — defense against per-suid retention.
- **Per-file-cap strip** — defense against per-fcap retention after chown.
- **Per-LSM hook** — defense against per-MAC bypass.
- **Per-quota dquot_transfer** — defense against per-quota-evasion.

### grsecurity / pax-style reinforcement

- **PaX UDEREF on `path`** — getname under SMAP+UDEREF.
- **CAP_CHOWN strict** — gradm RBAC may forbid CAP_CHOWN for non-policy
  subjects; default-deny for ownership change.
- **CAP_FOWNER / CAP_SETID strict** — RBAC discrete capability.
- **GRKERNSEC_SUIDDUMP on suid-mode-change** — chown that clears
  S_ISUID or S_ISGID via ATTR_KILL_SUID/SGID logs the transition with
  sender uid, target path, before/after mode. Coredump suppression flag
  recomputed.
- **GRKERNSEC_CHROOT_CHMOD** — chrooted process cannot chown to a uid
  outside its mapped user_ns; cross-user_ns chown denied even with
  CAP_CHOWN if RBAC policy lacks chroot-chown capability.
- **GRKERNSEC_TPE on setuid-mode set** — chown that would result in a
  setuid binary owned by an untrusted uid (combined with subsequent
  fchmod) is policy-denied at chown time.
- **File-cap-strip on chown ENFORCED** — `security.capability` xattr
  removed unconditionally on chown of an executable inode, regardless
  of caller capabilities. Grsec logs the cap-strip with uid, path, and
  the prior capability set.
- **GRKERNSEC_LINK on symlink chown traversal** — chown through a
  symlink owned by a different uid (or with safe-bit conflict) denied.
- **PaX MEMORY_SANITIZE on Iattr** — zero-init prevents leak of uninit
  ia_mode through ATTR_KILL paths.
- **PaX KERNEXEC** — `chown_common`, `notify_change`,
  `setattr_should_drop_suidgid` in read-only-after-init text.
- **PaX RANDSTRUCT on `struct iattr`** — randomized layout; corruption
  cannot deterministically smuggle uid via ia_uid swap.
- **GRKERNSEC_DMESG** — chown failure klog rate-limited, CAP_SYSLOG-gated.
- **GRKERNSEC_HIDESYM** — pointer leaks scrubbed.
- **gradm policy: chown REQUIRES policy** — discrete `o`/`O` mode RBAC
  capability; subjects without it cannot reparent files even as root.
- **GRKERNSEC_HARDEN_PTRACE** — tracee-driven chown via `/proc/<pid>/fd`
  requires PTRACE_MODE_ATTACH_FSCREDS.
- **Quota-policy enforcement** — dquot_transfer audited; failed transfer
  (over-quota) rejects chown before notify_change applies.

