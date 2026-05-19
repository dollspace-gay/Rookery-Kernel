---
title: "Tier-5 syscall: lchown(2) — syscall 94"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`lchown(2)` changes the owner (uid) and group (gid) of the file
identified by `path`, but unlike `chown(2)`, it does NOT dereference
symbolic links — the link itself is chowned. This matters because the
ownership of a symlink controls whether a user may unlink it from a
sticky directory (e.g. `/tmp`), and chmod/chown of the target are
distinct operations.

`lchown` is the only path-based way to change a symlink's owner;
without it (or `fchownat(_, _, _, _, AT_SYMLINK_NOFOLLOW)`), a user
running `chown -h` would not be able to express the operation. The
classic example is `tar -p` (preserve) restoring a tree with symlinks
owned by various users.

Since symlinks have no mode bits used for permission checking on
Linux (the link's mode is always `0777`), `lchown` skips the
ATTR_KILL_SUID / ATTR_KILL_SGID dance — a symlink can never carry a
setuid bit, and the `security.capability` xattr is not applied to
symlink inodes.

Permission rules are identical to `chown`: `CAP_CHOWN` for owner
change; group change must remain within caller's supplementary group
set (or hold CAP_CHOWN).

Critical for: `tar -p` extraction, `cp -a` clones, `rsync -a`
permission preservation, container layer materialization.

### Acceptance Criteria

- [ ] AC-1: `lchown("/tmp/symlink", 1000, 1000)` as root: symlink's owner updated; target's owner unchanged.
- [ ] AC-2: `lchown("/tmp/file", 1000, 1000)` on regular file (no symlink): behaves like chown.
- [ ] AC-3: lchown on dangling symlink: succeeds (symlink itself exists; ENOENT only for missing path).
- [ ] AC-4: lchown of symlink does NOT touch target's mode bits or fcap.
- [ ] AC-5: lchown with `(uid_t)-1, gid_in_supp_set` as link-owner: succeeds.
- [ ] AC-6: lchown owner-change without CAP_CHOWN: `-EPERM`.
- [ ] AC-7: lchown on RO-fs: `-EROFS`.
- [ ] AC-8: Audit captures "lchown" syscall name, link path, uid/gid.
- [ ] AC-9: SELinux context on symlink considered by LSM hook.
- [ ] AC-10: `lchown` of symlink whose final target is on a different mount: link's mount governs RO-check.
- [ ] AC-11: Idmap honored — uid stored is host-side after mapping.
- [ ] AC-12: ctime of symlink updated even when both args are `(uid_t)-1`.

### Architecture

```rust
#[syscall(nr = 94, abi = "sysv")]
pub fn sys_lchown(path: UserPtr<u8>, uid: u32, gid: u32) -> isize {
    Fs::do_fchownat(AT_FDCWD, path, uid, gid, AT_SYMLINK_NOFOLLOW)
}
```

`Fs::do_fchownat(...)` (shared with chown / fchownat): lookup with
`LOOKUP_FOLLOW=0`, all other steps identical. Filesystem layer:
`notify_change(mnt_idmap, dentry, &iattr, &delegated)` dispatches to
the symlink inode's `setattr` op (most FS: `simple_setattr` for
in-memory FS; ext4/xfs/btrfs have a dedicated path that handles
symlink inodes).

`Fs::chown_common(path, uid, gid)`: identical to chown.md — the key
difference is REQ-6 / REQ-7 short-circuit when `inode.is_lnk()`.

### Out of Scope

- `chown(2)` (separate Tier-5).
- `fchown(2)` (separate Tier-5).
- `fchownat(2)` (separate Tier-5 — modern superset).
- POSIX ACL propagation (separate Tier-5).
- Implementation code.

### signature

```c
int lchown(const char *path, uid_t owner, gid_t group);
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `path` | `const char *` | in | Path string. Symlinks NOT dereferenced. |
| `owner` | `uid_t` | in | New uid, or `(uid_t)-1`. |
| `group` | `gid_t` | in | New gid, or `(gid_t)-1`. |

### return value

| Value | Meaning |
|---|---|
| `0` | Ownership updated on the link / file itself. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EFAULT` | `path` outside caller's address space. |
| `ENOENT` | `path` does not exist. |
| `ENOTDIR` | A component is not a directory. |
| `EACCES` | search denied. |
| `ELOOP` | Symlink chain too long during intermediate components. |
| `ENAMETOOLONG` | Path too long. |
| `EPERM` | No CAP_CHOWN for owner change; group not in supp set without CAP_CHOWN; immutable. |
| `EROFS` | Read-only fs. |
| `EIO` | I/O error. |

### abi surface

```text
__NR_lchown (x86_64)  = 94
__NR_lchown (i386)    = 198    /* lchown32 */
__NR_lchown (arm64)   — n/a; use fchownat(AT_SYMLINK_NOFOLLOW)
__NR_lchown (riscv)   — n/a; use fchownat
```

### compatibility contract

REQ-1: Syscall number is **94** on x86_64. ABI-stable.

REQ-2: Per-path resolution: `user_path_at(AT_FDCWD, path, 0)` — NO
`LOOKUP_FOLLOW`. Final symlink NOT dereferenced. Intermediate
components ARE still followed.

REQ-3: Per-sentinel: `(uid_t)-1` / `(gid_t)-1` ⟹ field skipped.

REQ-4: Per-CAP_CHOWN: owner change ⟹ CAP_CHOWN in inode's user_ns.

REQ-5: Per-group permission: owner-driven gid change limited to supp
set or egid; else CAP_CHOWN.

REQ-6: Per-suid/sgid clear: ATTR_KILL_SUID / SGID NOT applied when the
target is a symlink (symlinks have no setuid semantics). For regular
files reached via lchown (no intermediate symlink as final), behavior
matches chown — ATTR_KILL_SUID applied for non-CAP_FSETID.

REQ-7: Per-file-cap-strip: NOT applied to symlink inodes (xattr storage
not used for symlink security.capability). Applied normally to regular
files reached without symlink final component.

REQ-8: Per-RO-mount: EROFS from mnt_want_write.

REQ-9: Per-immutable: EPERM unless CAP_LINUX_IMMUTABLE.

REQ-10: Per-LSM: `security_inode_setattr` invoked on symlink inode.
SELinux may have specific contexts for symlinks.

REQ-11: Per-notify_change with ATTR_UID/ATTR_GID/ATTR_CTIME. Filesystem
must support symlink setattr (most do via `simple_setattr` or
filesystem-specific op).

REQ-12: Per-ctime: symlink's ctime touched.

REQ-13: Per-fsnotify: `FS_ATTRIB` on symlink dentry.

REQ-14: Per-audit: AUDIT_SYSCALL records path, old/new uid+gid; audit
trail distinguishes lchown from chown.

REQ-15: Per-idmap: mount idmap + user_ns mapping applied.

REQ-16: Per-quota: dquot_transfer on the symlink inode (symlink uses
at most one block).

REQ-17: Per-non-symlink: if final component is NOT a symlink, lchown
behaves identically to chown — but caller asked for lchown semantics
explicitly.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `lookup_flag_no_follow` | INVARIANT | LOOKUP_FOLLOW not set in lookup. |
| `target_inode_is_link_or_file` | INVARIANT | final inode = path-named inode (no deref). |
| `no_suid_clear_on_link` | INVARIANT | symlink inode never gets ATTR_KILL_SUID. |
| `no_fcap_strip_on_link` | INVARIANT | symlink inode never has security.capability removed. |
| `cap_chown_for_owner_change` | INVARIANT | uid change ⟹ CAP_CHOWN. |

### Layer 2: TLA+

`fs/lchown.tla`:
- States: LOOKUP_NOFOLLOW → BUILD_IATTR → PERM → SUID_FILTER (skip-if-link) → LSM → SETATTR.
- Properties:
  - `safety_link_targeted` — final inode is the named inode, not the dereferenced target.
  - `safety_no_suid_clear_on_link` — symlink ATTR_KILL_SUID never applied.
  - `safety_cap_chown_owner` — uid change ⟹ CAP_CHOWN.
  - `liveness_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_lchown` post: target inode == name's inode (not deref) | `sys_lchown` |
| `do_fchownat` post: LOOKUP_FOLLOW=0 in lookup_flags | `Fs::do_fchownat` |
| `chown_common` post: ATTR_KILL_SUID skipped if S_ISLNK | `Fs::chown_common` |
| `chown_common` post: security.capability strip skipped if S_ISLNK | `Fs::chown_common` |

### Layer 4: Verus / Creusot functional

Per-`lchown(2)` man-page, per-POSIX (XSI), per-Linux fs/open.c +
fs/attr.c, LTP `lchown01..03`.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`lchown(2)` reinforcement:

- **Per-getname PAX_USERCOPY** — defense against per-string overflow.
- **Per-lookup-no-follow** — defense against per-symlink-target chown
  confusion.
- **Per-CAP_CHOWN strict** — defense against per-DAC bypass.
- **Per-group-supp-set check** — defense against per-group-injection.
- **Per-no-suid-clear-on-link** — defense against per-symlink suid
  confusion (symlinks cannot carry suid).
- **Per-LSM hook** — defense against per-MAC bypass on symlink contexts.

### grsecurity / pax-style reinforcement

- **PaX UDEREF on `path`** — getname under SMAP+UDEREF.
- **CAP_CHOWN strict** — gradm RBAC may forbid CAP_CHOWN for non-policy
  subjects.
- **CAP_FOWNER / CAP_SETID strict** — discrete RBAC capabilities.
- **GRKERNSEC_SUIDDUMP (cross-reference)** — lchown of a regular file
  (path resolved without an intermediate symlink as final) that
  triggers ATTR_KILL_SUID logs the event. lchown of a symlink itself
  does NOT trigger SUIDDUMP since symlinks carry no setuid bits.
- **GRKERNSEC_CHROOT_CHMOD** — chrooted process cannot lchown to a uid
  outside its mapped user_ns; cross-user_ns lchown silently denied
  even with CAP_CHOWN if RBAC lacks chroot-chown.
- **GRKERNSEC_TPE on setuid-mode set** — N/A for symlink-final lchown
  (no mode bits). Regular-file lchown follows chown's TPE policy.
- **File-cap-strip on chown (cross-reference)** — symlink lchown does
  NOT touch `security.capability` (symlinks cannot hold fcaps).
  Regular-file lchown strips fcap per chown semantics.
- **GRKERNSEC_LINK** — lchown changing a symlink owner to a different
  uid inside a sticky directory (`/tmp`, `/var/tmp`) is policy-checked:
  RBAC may require the caller to own both the symlink and the parent
  directory before chowning. Closes the "lchown to plant attacker-owned
  symlink" pattern.
- **GRKERNSEC_FIFO equivalent for symlink** — analogous policy: a
  symlink in a world-writable sticky directory whose target is a setuid
  binary cannot be re-owned to bypass owner check (`/tmp/foo -> /usr/bin/passwd`
  retargeting blocked unless lchown owner is the parent-directory owner).
- **PaX MEMORY_SANITIZE on Iattr** — zero-init.
- **PaX KERNEXEC** — `do_fchownat`, `chown_common`, `notify_change` in
  read-only-after-init text; cannot be patched to chase the symlink.
- **PaX RANDSTRUCT on `struct iattr`** — randomized; corruption cannot
  deterministically swap target inode pointer.
- **GRKERNSEC_DMESG** — lchown failure klog rate-limited, CAP_SYSLOG-gated.
- **GRKERNSEC_HIDESYM** — pointer leaks scrubbed.
- **gradm policy: lchown REQUIRES policy** — discrete RBAC mode.
- **GRKERNSEC_HARDEN_PTRACE** — tracee-driven lchown via fd-resolved
  path requires PTRACE_MODE_ATTACH_FSCREDS.
- **Symlink-restriction (sysctl: fs.protected_symlinks)** — though
  primarily about traversal, grsec extends to lchown: a symlink in a
  sticky directory cannot be lchowned to a different uid by anyone
  other than the symlink's owner or the parent-dir's owner. Closes
  the "rename-then-lchown" privilege-laundering pattern.

