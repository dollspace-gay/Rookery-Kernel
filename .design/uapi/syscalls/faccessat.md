# Tier-5 syscall: faccessat(2) — syscall 269

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/open.c (SYSCALL_DEFINE3(faccessat), do_faccessat, inode_permission)
  - include/uapi/asm-generic/fcntl.h (AT_FDCWD, AT_EACCESS, AT_SYMLINK_NOFOLLOW)
  - include/uapi/linux/fcntl.h (F_OK, R_OK, W_OK, X_OK)
  - arch/x86/entry/syscalls/syscall_64.tbl (269 common  faccessat)
-->

## Summary

`faccessat(2)` tests whether the calling process can access the file at `path` relative to `dirfd` for the modes specified in `mode` (`R_OK`, `W_OK`, `X_OK`, or `F_OK` for existence-only). Unlike `access(2)` it accepts a starting fd. **Glibc's** `faccessat()` simulates `AT_EACCESS` and `AT_SYMLINK_NOFOLLOW` flags by calling other syscalls; the **kernel** `faccessat(2)` does **not** accept a `flags` argument — the variant that does is `faccessat2(2)` (syscall 439). This document covers the 3-argument kernel syscall. The kernel resolves `path`, then calls `inode_permission` against the inode using the caller's **real** uid/gid (not effective), to mirror set-uid-binary semantics for the underlying real user. Critical for: shells (`test -r`), setuid programs verifying caller's real access, glibc compatibility shims.

## Signature

```c
int faccessat(int dirfd, const char *path, int mode);
```

```c
/* mode bits (combinable): */
#define F_OK   0   /* existence only */
#define X_OK   1   /* execute / search */
#define W_OK   2   /* write */
#define R_OK   4   /* read */
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `dirfd` | `int` | in | Directory fd, or `AT_FDCWD`. |
| `path` | `const char *` | in | NUL-terminated pathname; intermediate AND leaf symlinks followed. |
| `mode` | `int` | in | Bitwise OR of `F_OK`, `R_OK`, `W_OK`, `X_OK`. Other bits ⟹ `EINVAL`. |

## Return value

| Value | Meaning |
|---|---|
| `0` | All requested accesses are permitted. |
| `-1` + `errno` | Access denied or other failure. |

## Errors

| errno | Trigger |
|---|---|
| `EACCES` | At least one mode bit denied. |
| `EBADF` | `dirfd` invalid and `path` is not absolute. |
| `EFAULT` | `path` outside accessible memory. |
| `EINVAL` | Invalid bit in `mode`. |
| `EIO` | I/O error during path resolution. |
| `ELOOP` | Too many symbolic links. |
| `ENAMETOOLONG` | `path` exceeds `PATH_MAX`. |
| `ENOENT` | A component does not exist; or path is empty. |
| `ENOMEM` | Out of kernel memory. |
| `ENOTDIR` | A component of the path is not a directory. |
| `EROFS` | `W_OK` requested on a read-only filesystem. |
| `ETXTBSY` | `W_OK` requested on an executable currently running. |

## ABI surface

```text
__NR_faccessat (x86_64)  = 269
__NR_faccessat (i386)    = 307
__NR_faccessat (arm64)   = 48
__NR_faccessat (riscv)   = 48
```

## Compatibility contract

REQ-1: Syscall number is **269** on x86_64; ABI-stable.

REQ-2: Resolution semantics: identical to other `*at` syscalls; `dirfd == AT_FDCWD` ⟹ cwd; absolute path ⟹ `dirfd` ignored.

REQ-3: Symlink behavior: **leaf symlinks are followed** (no `AT_SYMLINK_NOFOLLOW` flag in this 3-arg variant). Use `faccessat2` for non-follow.

REQ-4: Permission check uses caller's **real** uid/gid (not euid/egid). This mirrors `access(2)` legacy semantics: a setuid program calling `faccessat` checks "would the real user have been allowed?" — useful for safe-check before performing privileged operation.

REQ-5: `mode` bits: only `F_OK`, `R_OK`, `W_OK`, `X_OK` (bitmask `0x7`); any other bit ⟹ `EINVAL`.

REQ-6: `F_OK` alone tests existence; on success returns 0; on missing returns `ENOENT`.

REQ-7: For `X_OK` on a directory: tests "search" permission (the same bit; effectively the same as +x on a dir).

REQ-8: For `X_OK` on a non-executable file system mount (`noexec`): returns `EACCES` even if the file's mode bits allow.

REQ-9: For `W_OK` on a read-only mount: returns `EROFS`.

REQ-10: `dirfd` ref via `fdget(dirfd)`; `fdput` on return.

REQ-11: Path-string copy_from_user bounded by `PATH_MAX`.

REQ-12: LSM hooks (SELinux `file_permission`) fire and may deny with `EACCES`.

REQ-13: Forward-compat: this 3-arg syscall is frozen; new flags require `faccessat2`.

REQ-14: An empty `path` returns `ENOENT` (no `AT_EMPTY_PATH` semantics in this variant).

## Acceptance Criteria

- [ ] AC-1: `faccessat(AT_FDCWD, "/etc/hostname", R_OK)`: returns 0.
- [ ] AC-2: `faccessat(AT_FDCWD, "/etc/shadow", R_OK)` as non-root: `EACCES`.
- [ ] AC-3: `faccessat(AT_FDCWD, "/missing", F_OK)`: `ENOENT`.
- [ ] AC-4: `faccessat(dirfd, "child", R_OK)`: resolves under dirfd.
- [ ] AC-5: `faccessat(-1, "rel", F_OK)`: `EBADF`.
- [ ] AC-6: `faccessat(AT_FDCWD, "x", 0xff)`: `EINVAL`.
- [ ] AC-7: `faccessat` on read-only mount with `W_OK`: `EROFS`.
- [ ] AC-8: `faccessat` on noexec mount with `X_OK`: `EACCES`.
- [ ] AC-9: `faccessat` from setuid binary uses **real** uid (caller, not setuid target).
- [ ] AC-10: 50-symlink loop: `ELOOP`.
- [ ] AC-11: `faccessat(AT_FDCWD, "", F_OK)`: `ENOENT`.
- [ ] AC-12: Path too long: `ENAMETOOLONG`.

## Architecture

```rust
#[syscall(nr = 269, abi = "sysv")]
pub fn sys_faccessat(dirfd: i32, path: UserPtr<u8>, mode: i32) -> isize {
    Access::do_faccessat(dirfd, path, mode, 0)
}
```

`Access::do_faccessat(dirfd, path_uptr, mode, flags) -> isize`:
1. if (mode & !(F_OK | R_OK | W_OK | X_OK)) != 0 { return Err(EINVAL); }
2. let kpath = Access::copy_path_from_user(path_uptr)?;        // EFAULT, ENAMETOOLONG
3. if kpath.is_empty() { return Err(ENOENT); }
4. /* Build creds: real-uid view (not effective). */
5. let creds = if (flags & AT_EACCESS) != 0 {
6.   current_creds().clone()
7. } else {
8.   current_creds().with_real_as_effective()                 // override euid <- ruid
9. };
10. let resolved = Path::resolve_at(dirfd, &kpath, LOOKUP_FOLLOW)?;
11. if mode == F_OK { return Ok(0); }
12. let inode_perm = mode_to_mask(mode);
13. Access::inode_permission(&resolved.inode, inode_perm, &creds)?;
14. /* Mount-policy enforcement */
15. if (mode & W_OK) != 0 ∧ resolved.mount.is_readonly() { return Err(EROFS); }
16. if (mode & X_OK) != 0 ∧ resolved.mount.is_noexec()   { return Err(EACCES); }
17. lsm_file_permission(&resolved, inode_perm)?;
18. Ok(0)

`Access::mode_to_mask(mode) -> u32`:
1. let mut m = 0;
2. if (mode & R_OK) != 0 { m |= MAY_READ; }
3. if (mode & W_OK) != 0 { m |= MAY_WRITE; }
4. if (mode & X_OK) != 0 { m |= MAY_EXEC; }
5. m

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `mode_validated` | INVARIANT | invalid bits ⟹ EINVAL before work. |
| `real_uid_check_by_default` | INVARIANT | this 3-arg variant always uses ruid (no AT_EACCESS). |
| `empty_path_rejected` | INVARIANT | empty path ⟹ ENOENT. |
| `mount_policy_enforced` | INVARIANT | noexec mount ⟹ X_OK denied; ro mount ⟹ W_OK denied. |

### Layer 2: TLA+

`fs/faccessat-syscall.tla`:
- States: per-copy_path, per-mode-validate, per-creds-build, per-resolve, per-inode-perm, per-mount-policy, per-lsm.
- Properties:
  - `safety_real_uid_for_check` — euid not used when AT_EACCESS unset (impossible here without flags).
  - `safety_mount_ro_blocks_W_OK` — readonly mount ⟹ W_OK ⟹ EROFS.
  - `safety_mount_noexec_blocks_X_OK` — noexec mount ⟹ X_OK ⟹ EACCES.
  - `liveness_faccessat_terminates` — every call returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_faccessat` post: ok ⟺ all mode bits granted under real-uid creds | `Access::do_faccessat` |
| `mode_to_mask` post: total mapping | `Access::mode_to_mask` |

### Layer 4: Verus / Creusot functional

Per-`faccessat(2)` POSIX-2008 semantics; selftests in `tools/testing/selftests/openat2/`.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`faccessat(2)` reinforcement:

- **Per-real-uid check** — defense against per-setuid-program privilege-leak (caller's real id is checked, not setuid'd effective).
- **Per-mount-policy enforcement** — defense against per-mount-bypass (noexec / ro).
- **Per-mode-bit strict mask** — defense against per-future-flag smuggling.
- **Per-LSM hook** — defense against per-MAC-bypass.
- **Per-dirfd fdget/fdput balanced** — defense against per-dirfd-UAF.

## Grsecurity / PaX-style Reinforcement

- **PaX UDEREF on `path` copy_from_user** — defense against per-kernel-pointer path; SMAP forced.
- **CAP_DAC_OVERRIDE strict on faccessat** — defense against per-cap-override leaking access: even with CAP_DAC_OVERRIDE, grsec under `GRKERNSEC_PROC_ADD` restricts override on /proc/<pid> paths owned by other users. Pure DAC override is gated to root-in-init-userns.
- **CAP_DAC_READ_SEARCH similarly strict** — defense against per-cap-read-leak in restricted-mode chroot/userns.
- **GRKERNSEC_LINK restricted symlink follow** — defense against per-faccessat-via-symlink in sticky world-writable dirs.
- **AT_EACCESS for setuid-binary safe-check** — `faccessat` defaults to real-uid; setuid programs must opt-in to AT_EACCESS via `faccessat2` for euid-based check. This 3-arg variant is intentionally real-uid-only.
- **GRKERNSEC_PROC_USERGROUP info-leak guard** — `faccessat` on `/proc/<pid>/*` honors `hidepid=4` and returns `ENOENT` to non-owning caller for hidden entries.
- **PAX_USERCOPY_HARDEN on path copy** — bounded by PATH_MAX; whitelisted slab.
- **Per-dirfd ownership table strict** — defense against per-cross-process fd injection.
- **Per-LSM file_permission hook** — SELinux/AppArmor checks layered on top.
- **GRKERNSEC_CHROOT consistency** — chrooted caller cannot escape with `..`.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `faccessat2(2)` 4-arg flag-bearing variant (covered in `faccessat2.md`).
- `access(2)` legacy (covered in `access.md` Tier-5).
- VFS inode_permission (covered in Tier-3 `fs/namei.md`).
- Implementation code.
