# Tier-5 syscall: faccessat2(2) — syscall 439

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/open.c (SYSCALL_DEFINE4(faccessat2), do_faccessat, inode_permission)
  - include/uapi/asm-generic/fcntl.h (AT_EACCESS, AT_SYMLINK_NOFOLLOW, AT_EMPTY_PATH)
  - include/uapi/linux/fcntl.h (F_OK, R_OK, W_OK, X_OK)
  - arch/x86/entry/syscalls/syscall_64.tbl (439 common  faccessat2)
-->

## Summary

`faccessat2(2)` is the 4-argument flag-bearing variant of `faccessat`. It adds `AT_EACCESS` (check using effective uid/gid instead of real), `AT_SYMLINK_NOFOLLOW` (do not follow leaf symlink), and `AT_EMPTY_PATH` (operate on `dirfd` itself if `path` is empty). This was added in Linux 5.8 specifically so glibc could stop emulating these AT-flags in userspace (the original `faccessat` didn't accept flags, forcing glibc into multi-syscall sequences that had subtle TOCTOU windows). The kernel resolves `path` per the flags, then performs an `inode_permission` check using either the real or effective uid as determined by `AT_EACCESS`. Critical for: modern glibc, setuid binaries doing effective-uid checks, container runtimes, openat-style safe-access workflows.

## Signature

```c
int faccessat2(int dirfd, const char *path, int mode, int flags);
```

```c
/* mode bits: */
#define F_OK   0
#define X_OK   1
#define W_OK   2
#define R_OK   4

/* flags: */
#define AT_EACCESS           0x200   /* check using effective ids */
#define AT_SYMLINK_NOFOLLOW  0x100   /* do not follow leaf symlink */
#define AT_EMPTY_PATH        0x1000  /* operate on dirfd if path == "" */
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `dirfd` | `int` | in | Directory fd, or `AT_FDCWD`. |
| `path` | `const char *` | in | NUL-terminated pathname; may be `""` if `AT_EMPTY_PATH`. |
| `mode` | `int` | in | Bitwise OR of `F_OK`, `R_OK`, `W_OK`, `X_OK`. Other bits ⟹ `EINVAL`. |
| `flags` | `int` | in | Bitwise OR of `AT_EACCESS`, `AT_SYMLINK_NOFOLLOW`, `AT_EMPTY_PATH`. Other bits ⟹ `EINVAL`. |

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
| `EINVAL` | Invalid bit in `mode` or `flags`; empty `path` without `AT_EMPTY_PATH`. |
| `EIO` | I/O error during resolution. |
| `ELOOP` | Too many symbolic links. |
| `ENAMETOOLONG` | `path` exceeds `PATH_MAX`. |
| `ENOENT` | A component does not exist. |
| `ENOMEM` | Out of kernel memory. |
| `ENOTDIR` | A path component is not a directory. |
| `EROFS` | `W_OK` on a read-only filesystem. |
| `ETXTBSY` | `W_OK` on currently-executing file. |

## ABI surface

```text
__NR_faccessat2 (x86_64)  = 439
__NR_faccessat2 (i386)    = 439
__NR_faccessat2 (arm64)   = 439
__NR_faccessat2 (riscv)   = 439
/* Same number across arches (post-5.8 unified numbering). */
```

## Compatibility contract

REQ-1: Syscall number is **439** on all archs; ABI-stable.

REQ-2: Resolution semantics identical to other `*at` syscalls; `dirfd == AT_FDCWD` ⟹ cwd.

REQ-3: `flags`:
- `AT_EACCESS`: check using **effective** uid/gid (default behavior is **real** uid/gid).
- `AT_SYMLINK_NOFOLLOW`: leaf symlink not dereferenced (lstat-like semantics).
- `AT_EMPTY_PATH`: if `path == ""`, operate on `dirfd` itself.
- Other bits ⟹ `EINVAL`.

REQ-4: `mode` bits: only `F_OK`, `R_OK`, `W_OK`, `X_OK` (bitmask `0x7`); any other ⟹ `EINVAL`.

REQ-5: `AT_EACCESS` ⟹ no creds swap; use current effective creds for `inode_permission`.

REQ-6: `AT_SYMLINK_NOFOLLOW` ⟹ symlink leaf: returns 0 if mode requested is `F_OK` (link itself exists); for R/W/X on a symlink, the kernel reports the link's own permissions (symlinks are typically lrwxrwxrwx — wide-open by convention).

REQ-7: `AT_EMPTY_PATH` ⟹ permits non-directory `dirfd`; provides per-fd permission check without re-resolving a name.

REQ-8: Mount policy: `W_OK` on read-only mount ⟹ `EROFS`; `X_OK` on noexec ⟹ `EACCES`.

REQ-9: LSM hooks fire normally.

REQ-10: `dirfd` ref via `fdget(dirfd)`; `fdput` on return.

REQ-11: Path-string copy_from_user bounded by `PATH_MAX`.

REQ-12: Forward-compat: only the four listed AT-flag bits are accepted; future extensions get explicit syscall.

REQ-13: Sysctl `kernel.faccessat2_default_eaccess` (hardened-build optional, default off): if set, treat absent `AT_EACCESS` as the default. Not in mainline.

## Acceptance Criteria

- [ ] AC-1: `faccessat2(AT_FDCWD, "/etc/hostname", R_OK, 0)`: 0 (real-uid view).
- [ ] AC-2: `faccessat2(AT_FDCWD, "/etc/hostname", R_OK, AT_EACCESS)`: 0 (effective-uid view).
- [ ] AC-3: `faccessat2(AT_FDCWD, "link", R_OK, AT_SYMLINK_NOFOLLOW)`: tests link's perms.
- [ ] AC-4: `faccessat2(dirfd, "", F_OK, AT_EMPTY_PATH)`: tests existence of dirfd's target.
- [ ] AC-5: `faccessat2(AT_FDCWD, "", F_OK, 0)`: `EINVAL`.
- [ ] AC-6: `faccessat2(AT_FDCWD, "x", 0xff, 0)`: `EINVAL`.
- [ ] AC-7: `faccessat2(AT_FDCWD, "x", R_OK, 0xdeadbeef)`: `EINVAL`.
- [ ] AC-8: `faccessat2(-1, "rel", F_OK, 0)`: `EBADF`.
- [ ] AC-9: `faccessat2` from setuid binary with `AT_EACCESS`: euid is checked (proper privilege).
- [ ] AC-10: `faccessat2` from setuid binary without `AT_EACCESS`: ruid is checked (legacy safe).
- [ ] AC-11: `faccessat2(..., W_OK, ...)` on ro mount: `EROFS`.
- [ ] AC-12: `faccessat2(..., X_OK, ...)` on noexec mount: `EACCES`.

## Architecture

```rust
#[syscall(nr = 439, abi = "sysv")]
pub fn sys_faccessat2(dirfd: i32, path: UserPtr<u8>, mode: i32, flags: i32) -> isize {
    Access::do_faccessat(dirfd, path, mode, flags)
}
```

`Access::do_faccessat(dirfd, path_uptr, mode, flags) -> isize`:
1. const ALLOWED_FLAGS: i32 = AT_EACCESS | AT_SYMLINK_NOFOLLOW | AT_EMPTY_PATH;
2. if (flags & !ALLOWED_FLAGS) != 0 { return Err(EINVAL); }
3. if (mode & !(F_OK | R_OK | W_OK | X_OK)) != 0 { return Err(EINVAL); }
4. let kpath = Access::copy_path_from_user(path_uptr)?;        // EFAULT, ENAMETOOLONG
5. if kpath.is_empty() ∧ (flags & AT_EMPTY_PATH) == 0 { return Err(EINVAL); }
6. /* Build creds: euid (AT_EACCESS) or ruid (default). */
7. let creds = if (flags & AT_EACCESS) != 0 {
8.   current_creds().clone()
9. } else {
10.  current_creds().with_real_as_effective()                  // override euid <- ruid
11. };
12. let lookup = if (flags & AT_SYMLINK_NOFOLLOW) != 0 {
13.   LookupFlags::empty()
14. } else { LookupFlags::FOLLOW };
15. let lookup = lookup | if (flags & AT_EMPTY_PATH) != 0 { LookupFlags::EMPTY } else { LookupFlags::empty() };
16. let resolved = Path::resolve_at(dirfd, &kpath, lookup)?;
17. if mode == F_OK { return Ok(0); }
18. let inode_perm = mode_to_mask(mode);
19. Access::inode_permission(&resolved.inode, inode_perm, &creds)?;
20. if (mode & W_OK) != 0 ∧ resolved.mount.is_readonly() { return Err(EROFS); }
21. if (mode & X_OK) != 0 ∧ resolved.mount.is_noexec()   { return Err(EACCES); }
22. lsm_file_permission(&resolved, inode_perm)?;
23. Ok(0)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_validated` | INVARIANT | unknown flag bits ⟹ EINVAL before work. |
| `mode_validated` | INVARIANT | invalid mode bits ⟹ EINVAL. |
| `at_eaccess_swaps_creds` | INVARIANT | AT_EACCESS ⟹ effective creds; default ⟹ real-as-effective override. |
| `empty_path_requires_at_empty_path` | INVARIANT | empty path && !AT_EMPTY_PATH ⟹ EINVAL. |
| `mount_policy_enforced` | INVARIANT | ro mount + W_OK ⟹ EROFS; noexec + X_OK ⟹ EACCES. |

### Layer 2: TLA+

`fs/faccessat2-syscall.tla`:
- States: per-flag-validate, per-mode-validate, per-copy_path, per-creds-build, per-resolve, per-inode-perm, per-mount-policy, per-lsm.
- Properties:
  - `safety_at_eaccess_obeyed` — correct creds selected per AT_EACCESS.
  - `safety_at_symlink_nofollow_obeyed` — leaf symlink resolved per flag.
  - `safety_at_empty_path_obeyed` — empty path semantics correctly gated.
  - `liveness_faccessat2_terminates` — every call returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_faccessat` post: ok ⟺ all mode bits granted under selected creds | `Access::do_faccessat` |
| `creds-build` post: AT_EACCESS ⟹ euid; default ⟹ ruid-as-euid | inline |

### Layer 4: Verus / Creusot functional

Per-`faccessat2(2)` post-5.8 semantics; selftests in `tools/testing/selftests/openat2/`.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`faccessat2(2)` reinforcement:

- **Per-flags strict-mask validation** — defense against per-future-flag smuggling.
- **Per-mode strict-mask validation** — defense against per-mode-bit smuggling.
- **Per-empty-path requires AT_EMPTY_PATH** — defense against per-accidental-fstat-style semantics.
- **Per-real-uid default, per-AT_EACCESS opt-in** — defense against per-setuid-program privilege-leak.
- **Per-mount-policy enforcement (ro/noexec)** — defense against per-mount-bypass.
- **Per-LSM hook** — defense against per-MAC-bypass.

## Grsecurity / PaX-style Reinforcement

- **PaX UDEREF on `path` copy_from_user** — defense against per-kernel-pointer path; SMAP forced.
- **CAP_DAC_OVERRIDE strict on faccessat2** — defense against per-cap-override leaking access; root-in-userns is gated when GRKERNSEC_PROC_ADD or similar policies are in effect.
- **AT_EACCESS for setuid-binary safe-check** — setuid binary explicitly opts in to euid-check via `AT_EACCESS`; absence preserves real-uid safe behavior (this is the documented anti-confused-deputy pattern). Grsec hardens this default and never silently switches.
- **GRKERNSEC_LINK restricted symlink follow** — defense against per-faccessat2-via-symlink in sticky world-writable dirs (matches `fs.protected_symlinks`).
- **AT_SYMLINK_NOFOLLOW provides explicit no-follow** — defense against per-symlink-TOCTOU at the leaf.
- **AT_EMPTY_PATH allowed only for fd owner** — defense against per-cross-process O_PATH leak.
- **GRKERNSEC_PROC_USERGROUP info-leak guard** — `/proc/<pid>/*` honored under `hidepid=4`.
- **PAX_USERCOPY_HARDEN on path copy** — bounded by PATH_MAX; whitelisted slab.
- **Per-dirfd ownership table strict** — defense against per-cross-process fd injection.
- **Per-flags reserved bits rejected** — defense against per-future-flag smuggling; kernel-side immune to unknown bits.
- **GRKERNSEC_CHROOT consistency** — chrooted caller cannot escape with `..` flags.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `faccessat(2)` 3-arg variant (covered in `faccessat.md`).
- `access(2)` legacy (covered in `access.md` Tier-5).
- VFS inode_permission internals (covered in Tier-3 `fs/namei.md`).
- Implementation code.
