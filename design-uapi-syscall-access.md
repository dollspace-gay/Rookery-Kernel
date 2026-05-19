---
title: "Tier-5 syscall: access(2) — syscall 21"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`access(2)` checks whether the calling process **would be permitted** to perform the operations specified by `mode` on the file at `path` — using the **real** UID/GID (not effective) and not consulting any other set-UID transitions. The `mode` is a bitmask of `R_OK` (read), `W_OK` (write), `X_OK` (execute / search), and `F_OK` (existence-only). The syscall has been functionally superseded by `faccessat(2)` (which adds dirfd and `AT_EACCESS` to switch real/effective semantics) and `faccessat2(2)` (which adds `AT_EMPTY_PATH`, `AT_SYMLINK_NOFOLLOW`, and a clean flags space), but `access(2)` remains a hot path because of its short signature, glibc compatibility, and the enormous footprint of legacy code that calls it before `open(2)` as a "may I" probe. Critical for: shell builtins (`test -r`, `test -w`, `[ -x ]`), setuid programs that drop privilege checks back to real-uid before touching user files, build-system file-existence probes, suid setuid-root sanity gates, package-manager preflight checks.

This Tier-5 covers the userspace ABI of syscall 21. The path-resolution and permission-check machinery is owned by `fs/namei.md` / `fs/open.md` (Tier-3).

### Acceptance Criteria

- [ ] AC-1: `access("/etc/passwd", R_OK)`: returns `0` for any user (world-readable).
- [ ] AC-2: `access("/etc/shadow", R_OK)`: returns `-EACCES` for non-root, `0` for root.
- [ ] AC-3: `access("/etc/shadow", F_OK)`: returns `0` (exists and search-perm OK).
- [ ] AC-4: `access("/no/such/file", F_OK)`: returns `-ENOENT`.
- [ ] AC-5: `access("/tmp", W_OK)`: returns `0` (world-writable).
- [ ] AC-6: `access("/proc/self/exe", W_OK)`: returns `-EACCES` or `-EROFS`.
- [ ] AC-7: `access(NULL, F_OK)`: returns `-EFAULT`.
- [ ] AC-8: `access("", F_OK)`: returns `-ENOENT`.
- [ ] AC-9: `access("/some/file", 0x10)` (invalid mode bit): returns `-EINVAL`.
- [ ] AC-10: `access("loop_symlink", F_OK)` where loop_symlink → loop_symlink: returns `-ELOOP` after `MAXSYMLINKS` (40).
- [ ] AC-11: Running as setuid-root, real uid is normal: `access("/root/secret", R_OK)` returns `-EACCES` (real-uid checked, not effective).
- [ ] AC-12: `access("/path", R_OK | W_OK)` where caller has only read: returns `-EACCES`.

### Architecture

Rookery surface in `fs/syscalls.rs`:

```rust
pub fn sys_access(path: UserCStr, mode: i32) -> isize {
    /* Validate mode bits */
    if (mode & !(F_OK | R_OK | W_OK | X_OK)) != 0 {
        return -EINVAL as isize;
    }
    /* Delegate to faccessat */
    Fs::do_faccessat(AT_FDCWD, path, mode, 0 /* flags */)
}
```

`Fs::do_faccessat(dirfd, path, mode, flags)`:

```rust
pub fn do_faccessat(dirfd: i32, path: UserCStr, mode: i32, flags: i32) -> isize {
    let path_str = match path.copy_into_kernel() {
        Ok(s)  => s,
        Err(_) => return -EFAULT as isize,
    };
    /* Determine cred set: real or effective */
    let creds = if flags & AT_EACCESS != 0 {
        current().effective_creds()
    } else {
        current().real_creds()
    };
    /* Resolve path under flags */
    let path = match Path::lookup(dirfd, &path_str, flags) {
        Ok(p)  => p,
        Err(e) => return -(e as isize),
    };
    /* Permission check */
    Path::may_access(&path, mode, &creds)
        .map(|_| 0)
        .unwrap_or_else(|e| -(e as isize))
}
```

`Path::may_access(path, mode, creds)`:

```rust
fn may_access(path: &Path, mode: i32, creds: &Creds) -> Result<(), Errno> {
    if mode == F_OK { return Ok(()); }
    let inode = path.inode();
    let needed = ((mode & R_OK) != 0).then_some(MAY_READ).unwrap_or(0)
               | ((mode & W_OK) != 0).then_some(MAY_WRITE).unwrap_or(0)
               | ((mode & X_OK) != 0).then_some(MAY_EXEC).unwrap_or(0);
    /* W_OK on read-only fs */
    if (mode & W_OK) != 0 && inode.sb().is_readonly() {
        return Err(EROFS);
    }
    /* W_OK on text-busy */
    if (mode & W_OK) != 0 && inode.deny_write_count() > 0 {
        return Err(ETXTBSY);
    }
    /* DAC + capability check */
    Path::inode_permission(inode, needed, creds)
}
```

### Out of Scope

- `fs/namei.md` Tier-3: path resolution, symlink traversal.
- `fs/open.md` Tier-3: `do_faccessat` shared implementation.
- `fs/inode.md` Tier-3: `inode_permission` DAC check.
- `faccessat.md` / `faccessat2.md` modern siblings.
- `kernel/cred.md` Tier-3: real vs effective creds.
- LSM `inode_permission` hook (covered separately).
- glibc / musl userspace wrappers.
- Implementation code.

### signature

```c
int access(const char *path, int mode);
```

Rust ABI shim:

```rust
pub fn sys_access(path: *const u8, mode: i32) -> isize;
```

Syscall number: **21** (x86_64). Generic syscall table has no direct entry: arm64 / generic platforms use `faccessat(AT_FDCWD, ...)` instead. The bare `access(2)` is x86_64 / i386 / x32 legacy.

### parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `path` | `const char *` | IN (UAPI ptr) | NUL-terminated pathname; copied in via `strncpy_from_user`. |
| `mode` | `i32` | IN | Bitwise OR of `F_OK` (0), `R_OK` (4), `W_OK` (2), `X_OK` (1). |

### return

- **Success**: `0` — all requested checks pass.
- **Failure**: `-1` and `errno`; Rust internal returns negated errno.

### errors

| errno | Trigger |
|---|---|
| `EACCES` | Permission denied for one or more requested modes; or search permission denied on a component of `path`. |
| `EINVAL` | `mode` contains bits outside `R_OK | W_OK | X_OK | F_OK`. |
| `ELOOP` | Too many symlinks during path resolution. |
| `ENAMETOOLONG` | Path component exceeds `NAME_MAX` (255) or total exceeds `PATH_MAX` (4096). |
| `ENOENT` | Component of `path` does not exist (or empty string when `AT_EMPTY_PATH` semantics not requested). |
| `ENOTDIR` | A non-final component of `path` is not a directory. |
| `EROFS` | `W_OK` requested on a file on a read-only filesystem. |
| `ETXTBSY` | `W_OK` requested on an executable that is currently being executed. |
| `EFAULT` | `path` points to unmapped userspace memory. |
| `EIO` | I/O error during path resolution. |
| `ENOMEM` | Insufficient kernel memory. |

### abi surface

```text
__NR_access (x86_64) = 21
__NR_access (i386)   = 33
__NR_access (arm64 / generic) = unavailable (use faccessat)

/* mode bits (include/uapi/linux/fcntl.h) */
F_OK = 0          /* existence */
X_OK = 1          /* execute / search */
W_OK = 2          /* write */
R_OK = 4          /* read */

/* Implementation dispatch */
sys_access(path, mode) === do_faccessat(AT_FDCWD, path, mode, 0 /* flags */);

/* The 0 flags argument means: */
/*   - real UID/GID used (NOT AT_EACCESS) */
/*   - symlinks followed (NOT AT_SYMLINK_NOFOLLOW) */
```

### compatibility contract

REQ-1: Syscall number is **21** on x86_64; not present on arm64/generic (glibc emulates via `faccessat(AT_FDCWD, ...)`). ABI-stable since 1.0.

REQ-2: Permission check uses the process's **real** UID and **real** GID (and supplementary groups), NOT effective UID/GID. This is the historical semantic of `access(2)` and the reason setuid programs call it.

REQ-3: `mode` validation: `(mode & ~(F_OK | R_OK | W_OK | X_OK)) != 0` ⟹ `-EINVAL`.

REQ-4: Implementation is `return do_faccessat(AT_FDCWD, path, mode, 0)`. Per-symlink follow is enabled.

REQ-5: `F_OK` (mode == 0): check only existence + search-permission on intervening components.

REQ-6: `R_OK | W_OK | X_OK`: each requested permission checked; all-or-nothing semantics — fail on any.

REQ-7: Capability override:
- `CAP_DAC_READ_SEARCH`: bypass DAC for read and directory search.
- `CAP_DAC_OVERRIDE`: bypass DAC for read, write, search; bypass partial executable check for non-superuser.

REQ-8: For executable check (`X_OK`): on a file, at least one execute bit must be set in the mode (for capable users, this is required to avoid the well-known footgun where `CAP_DAC_OVERRIDE` grants execute on a file with mode 644).

REQ-9: `W_OK` on a read-only filesystem ⟹ `-EROFS` even for `root`.

REQ-10: `W_OK` on a text-file currently in use as an executable ⟹ `-ETXTBSY`.

REQ-11: Symlink following: by default `access(2)` follows symlinks (no `AT_SYMLINK_NOFOLLOW` available; use `faccessat2` for that).

REQ-12: `path == NULL` ⟹ `-EFAULT`. `path == ""` ⟹ `-ENOENT`.

REQ-13: TOCTOU caveat documented: between `access(2)` succeeding and a subsequent `open(2)`, the file may change. Modern code should `open(2)` and handle errors rather than calling `access(2)` first.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `mode_bits_validated` | INVARIANT | mode ∉ {F_OK,R_OK,W_OK,X_OK}-mask ⟹ `-EINVAL`. |
| `real_uid_used` | INVARIANT | permission check uses real uid (not effective) for `access(2)`. |
| `rofs_blocks_W_OK` | INVARIANT | W_OK on RO-fs ⟹ `-EROFS`. |
| `txt_busy_blocks_W_OK` | INVARIANT | W_OK on busy text ⟹ `-ETXTBSY`. |
| `null_path_efault` | INVARIANT | NULL `path` ⟹ `-EFAULT`. |
| `symlink_followed` | INVARIANT | symlinks resolved (no `AT_SYMLINK_NOFOLLOW`). |

### Layer 2: TLA+

`uapi/access.tla`:
- Variables: `caller.real_creds`, `caller.eff_creds`, `path.inode.mode`, `path.inode.uid`, `path.inode.gid`, `fs.readonly`, `inode.deny_write`.
- Properties:
  - `safety_real_uid_used` — perm check uses `caller.real_creds`.
  - `safety_mode_validated` — invalid mode bits rejected.
  - `safety_rofs_blocks_W_OK` — RO-fs ⟹ `-EROFS` on W_OK.
  - `safety_capability_override` — `CAP_DAC_OVERRIDE` bypasses DAC where applicable.
  - `liveness_terminates` — path resolution terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_access` post: delegates to `do_faccessat(AT_FDCWD, ..., 0)` | `sys_access` |
| `do_faccessat(flags=0)` post: real_creds used | `Fs::do_faccessat` |
| `may_access` post: F_OK ⟹ no DAC check (only existence) | `Path::may_access` |
| `may_access` post: W_OK ∧ ro_fs ⟹ EROFS | `Path::may_access` |

### Layer 4: Verus/Creusot functional

Per-`access(2)` and `faccessat(2)` man pages; per-LTP `testcases/kernel/syscalls/access/*`; per-POSIX.1-2008; per-glibc `sysdeps/unix/syscalls.list`.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

access reinforcement:

- **Real-UID semantics strict** — defense against per-effective-uid-leak via `access`: callers explicitly want real-uid here.
- **Mode-bit validation strict** — defense against per-mode-bit-smuggling into permission machinery.
- **W_OK on RO-fs blocked** — defense against per-RO-bypass attempts.
- **W_OK on busy text blocked** — defense against per-executable-mutation race.
- **Symlink-follow explicit** — defense against per-AT_SYMLINK_NOFOLLOW assumption; modern code should use `faccessat2`.
- **TOCTOU caveat documented** — defense-in-depth via documentation; callers should prefer `open` + error handling.

### grsecurity/pax-style reinforcement

- **PaX UDEREF strict on `path` pointer** — `strncpy_from_user(path)` goes through the validated UDEREF helper; defense against per-attacker kernel-pointer aliased as user.
- **PAX_RANDKSTACK on every entry** — randomise kstack per `access` entry; defense against per-stack-spray of path-resolution code.
- **AT_EACCESS-equivalent strict opposite for `access(2)`** — `access(2)` must NOT silently switch to effective creds; defense against per-LSM-mistake that grants higher privilege through legacy `access`.
- **GRKERNSEC_HIDESYM on `path->dentry`, `inode` addresses** — path/inode addresses excluded from `/proc/kallsyms` and from leak through perm-check timing variance under `kptr_restrict ≥ 2`.
- **GRKERNSEC_CHROOT restrict `access` outside chroot** — chrooted process attempting `access("/etc/shadow")` outside the chroot bind: returns `-ENOENT` rather than a discriminating error code.
- **TOCTOU mitigation: audit `access`-then-`open`** — grsec hardened policy audit-logs sequences of `access(path, W_OK) == 0` followed by `open(path, O_WRONLY)` from the same task within 1 ms, flagging the well-known TOCTOU pattern.
- **Per-uid `access` rate-limit** — defense against per-`access`-floods used to enumerate filesystem layout (the historical `find` + `stat` reconnaissance pattern).
- **Refuse `access(.. R_OK | W_OK | X_OK)` cross-mount under sandbox** — under hardened policy, cross-mount `access` queries from sandboxed users are denied; defense against per-mount-recon.
- **Symlink-NOFOLLOW recommendation enforced** — grsec hardened policy may force `access(2)` to behave as `faccessat2(AT_SYMLINK_NOFOLLOW)` by default; defense against per-symlink-redirection toward attacker-controlled paths.
- **Deprecated-syscall ENOSYS gate (off by default)** — grsec hardened policy may force `access(2)` to `-ENOSYS`, forcing applications onto `faccessat2(2)` which has explicit flags for real-vs-effective and symlink-follow; defense against per-legacy-API surface.
- **Path-error-discrimination unified** — `EACCES` / `ENOENT` / `ENOTDIR` made indistinguishable via constant-time path lookup under hardened policy; defense against per-filesystem-enumeration through error-code-differential.
- **CAP_DAC_OVERRIDE strict in target user-ns** — capability checked in inode's user-ns, not caller's; defense against per-userns CAP-laundering.
- **CAP_DAC_READ_SEARCH similarly strict** — defense against per-userns CAP-laundering for read+search.

