---
title: "Tier-5: syscall 437 — openat2(2)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`openat2(2)` is **x86_64 syscall 437**, the extensible directory-relative open. Unlike `open`/`openat`, it accepts a versioned, growable `struct open_how` packaged as a userspace pointer plus size, and applies **strict** validation: unknown `flags` bits, unknown `resolve` bits, or `mode != 0` on non-creating opens all yield `-EINVAL`. `openat2` is the only entry point that exposes the `RESOLVE_*` lookup-restriction flags — the modern, container-safe path-resolution hardening (no symlinks, no magic links, no mount crossings, beneath/in-root scoping, cached-only lookup). Implementation: `copy_struct_from_user` (per-extensible-struct ABI) into a kernel `struct open_how`, then `do_sys_openat2(dfd, filename, &how)`.

Critical for: container runtimes (`runc`, `crun`), sandboxed file-server / web-server static-asset paths, fuzz-resistant path traversal, `openat2` libraries that need symlink-free / mount-free resolution, kernel-tested test suites (`tools/testing/selftests/openat2/`).

### Acceptance Criteria

- [ ] AC-1: `openat2(AT_FDCWD, "foo", &how, 24)` with `how.flags=O_RDONLY` ≡ `open("foo", O_RDONLY)`.
- [ ] AC-2: Unknown `how.flags` bit ⟹ `-EINVAL`.
- [ ] AC-3: Unknown `how.resolve` bit ⟹ `-EINVAL`.
- [ ] AC-4: `how.mode != 0` with `O_RDONLY` ⟹ `-EINVAL`.
- [ ] AC-5: `usize == 23` ⟹ `-EINVAL`; `usize == 4097` ⟹ `-E2BIG`.
- [ ] AC-6: `usize > sizeof(open_how)` with non-zero trailing bytes ⟹ `-E2BIG`.
- [ ] AC-7: `RESOLVE_NO_SYMLINKS` on a symlinked path ⟹ `-ELOOP`.
- [ ] AC-8: `RESOLVE_NO_XDEV` crossing a mount ⟹ `-EXDEV`.
- [ ] AC-9: `RESOLVE_BENEATH | RESOLVE_IN_ROOT` ⟹ `-EINVAL`.
- [ ] AC-10: `RESOLVE_CACHED` with `O_CREAT|O_TRUNC|__O_TMPFILE` ⟹ `-EAGAIN`.
- [ ] AC-11: `RESOLVE_IN_ROOT` with `dirfd` as `/srv`: `openat2(dirfd, "/etc/passwd", ...)` resolves to `/srv/etc/passwd`.
- [ ] AC-12: `RESOLVE_NO_MAGICLINKS` through `/proc/self/fd/N` ⟹ `-ELOOP`.

### Architecture

```
struct OpenHow { flags: u64, mode: u64, resolve: u64 }
struct Openat2Args { dirfd: i32, filename: UserPtr<u8>, how: UserPtr<OpenHow>, usize_: usize }
```

`sys_openat2(args) -> i32`:

1. `BUILD_BUG_ON(size_of::<OpenHow>() != OPEN_HOW_SIZE_LATEST);`
2. If `args.usize_ < OPEN_HOW_SIZE_VER0` ⟹ return `-EINVAL`.
3. If `args.usize_ > PAGE_SIZE` ⟹ return `-E2BIG`.
4. `let mut tmp: OpenHow = Default::default();`
5. `copy_struct_from_user(&mut tmp, size_of::<OpenHow>(), args.how, args.usize_)?;`
6. `audit_openat2_how(&tmp);`
7. If `!(tmp.flags & O_PATH) && force_o_largefile()` ⟹ `tmp.flags |= O_LARGEFILE;`
8. Return `do_sys_openat2(args.dirfd, args.filename, &tmp)`.

`Fs::do_sys_openat2(dfd, filename, how) -> i32`:

1. `let op = build_open_flags(how)?;`
2. `let name = getname(filename)?;`
3. `let f = do_file_open(dfd, name, &op)?;`
4. `let fd = get_unused_fd_flags(how.flags as u32)?;`
5. `fd_install(fd, f);`
6. Return `fd | FD_ADD(how.flags)`.

`Fs::build_open_flags` extended for `openat2`:

1. Strict-reject unknown `flags` (no silent strip).
2. Validate `resolve` bits.
3. Reject `BENEATH | IN_ROOT`.
4. Validate `mode`.
5. Apply legacy `open` flag-combo rejects (`O_DIRECTORY|O_CREAT`, `O_TMPFILE`, `O_PATH` constraints).
6. Translate `RESOLVE_*` ⟹ `LOOKUP_*`.
7. `RESOLVE_CACHED` exclusion with truncate/create/tmpfile ⟹ early `-EAGAIN`.

### Out of Scope

- `open(2)` / `openat(2)` (separate Tier-5 docs).
- Future `OPEN_HOW_SIZE_VER1+` fields (none defined as of 7.1.0-rc2).
- LSM internals.
- Path-resolution internals (covered in `fs/namei.md` Tier-3 if expanded).
- Implementation code.

### signature

C (Linux-specific):

```c
int openat2(int dirfd, const char *pathname, struct open_how *how, size_t size);
```

glibc wrapper: `__libc_openat2` (since glibc 2.34) → `INLINE_SYSCALL(openat2, 4, dirfd, pathname, how, size)`. Older glibc requires `syscall(SYS_openat2, ...)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE4(openat2, int, dfd, const char __user *, filename,
                 struct open_how __user *, how, size_t, usize);
```

Rookery dispatch:

```rust
pub fn sys_openat2(dirfd: i32, filename: UserPtr<u8>,
                   how: UserPtr<OpenHow>, usize_: usize) -> SyscallResult<i32>;
```

### parameters

| name     | type                      | constraints                                                       | errno-on-bad |
|----------|---------------------------|-------------------------------------------------------------------|--------------|
| dirfd    | `int`                     | open directory fd or `AT_FDCWD`                                    | `EBADF` / `ENOTDIR` |
| filename | `const char __user *`     | NUL-terminated; length `< PATH_MAX`                                | `EFAULT` / `ENAMETOOLONG` |
| how      | `struct open_how __user *`| valid userspace `struct open_how`                                  | `EFAULT`     |
| usize    | `size_t`                  | `>= OPEN_HOW_SIZE_VER0 (24)`; `<= PAGE_SIZE` (4096)                | `EINVAL` / `E2BIG` |

`struct open_how` layout (UAPI):

```c
struct open_how {
    __u64 flags;    /* O_* flags */
    __u64 mode;     /* file mode for O_CREAT / O_TMPFILE; else must be 0 */
    __u64 resolve;  /* RESOLVE_* flags */
};
```

### return value

- Success: `>= 0` — new file descriptor.
- Failure: `< 0` — negated errno.

### errors

| errno      | condition                                                                                            |
|------------|------------------------------------------------------------------------------------------------------|
| `EFAULT`   | `filename` or `how` is outside user space, or trailing bytes of `how` (beyond `OPEN_HOW_SIZE_VER0`) are non-NULL when kernel doesn't know them. |
| `EINVAL`   | `usize < OPEN_HOW_SIZE_VER0`, unknown bit in `how->flags`, unknown bit in `how->resolve`, `(resolve & RESOLVE_BENEATH) && (resolve & RESOLVE_IN_ROOT)`, `mode != 0` without `O_CREAT|O_TMPFILE`, `mode & ~S_IALLUGO`, `O_DIRECTORY|O_CREAT`, `O_TMPFILE` without `O_DIRECTORY`, `O_PATH` with disallowed extras, `O_TMPFILE` without write access. |
| `E2BIG`    | `usize > PAGE_SIZE`.                                                                                 |
| `EAGAIN`   | `RESOLVE_CACHED` and resolution would have required a non-cached I/O (e.g., a stat-cache miss).      |
| `EXDEV`    | `RESOLVE_NO_XDEV` and path traversal crossed a mount.                                                |
| `ELOOP`    | `RESOLVE_NO_SYMLINKS` and resolution hit a symlink, OR `RESOLVE_NO_MAGICLINKS` and resolution hit a procfs-style magic link. |
| `EBADF`    | `dirfd` invalid (non-`AT_FDCWD` and not an open fd).                                                 |
| `ENOTDIR`  | `dirfd` not a directory and `filename` is relative.                                                  |
| All `openat(2)` errors | propagate from `do_sys_openat2`.                                                          |

### abi surface (constants + flags)

`how->flags` — any subset of `VALID_OPEN_FLAGS` (see `open.md`). **Unknown bits ⟹ `-EINVAL`.** (Contrast `open(2)` / `openat(2)` which silently strip.)

`how->mode` — `S_IALLUGO (07777)`. **Must be `0` unless `O_CREAT` or `O_TMPFILE` is set; else `-EINVAL`.**

`how->resolve` — bitfield from `include/uapi/linux/openat2.h`:

- `RESOLVE_NO_XDEV` (`0x01`) — block mount-point crossings (includes bind mounts).
- `RESOLVE_NO_MAGICLINKS` (`0x02`) — block traversal through procfs-style "magic links" (e.g., `/proc/$pid/fd/*`, `/proc/$pid/cwd`, `/proc/$pid/root`).
- `RESOLVE_NO_SYMLINKS` (`0x04`) — block any symlink traversal (implies `RESOLVE_NO_MAGICLINKS`).
- `RESOLVE_BENEATH` (`0x08`) — block `..`/absolute-path escapes above `dirfd`'s starting point. Mutually exclusive with `RESOLVE_IN_ROOT`.
- `RESOLVE_IN_ROOT` (`0x10`) — treat `dirfd` as `/`: `..` at root stays at root; absolute paths are interpreted relative to `dirfd` (`chroot`-like, in-process, no privilege required). Mutually exclusive with `RESOLVE_BENEATH`.
- `RESOLVE_CACHED` (`0x20`) — fail with `-EAGAIN` if resolution would require non-cached I/O. Forbidden when `flags & (O_TRUNC | O_CREAT | __O_TMPFILE)` — return `-EAGAIN` immediately.

`VALID_RESOLVE_FLAGS` = `RESOLVE_NO_XDEV | RESOLVE_NO_MAGICLINKS | RESOLVE_NO_SYMLINKS | RESOLVE_BENEATH | RESOLVE_IN_ROOT | RESOLVE_CACHED`. **Any other bit ⟹ `-EINVAL`.**

Version-size constants:

- `OPEN_HOW_SIZE_VER0 = 24` (first publication).
- `OPEN_HOW_SIZE_LATEST = 24` (currently same as ver0).
- `BUILD_BUG_ON(sizeof(struct open_how) < OPEN_HOW_SIZE_VER0)` and `... != OPEN_HOW_SIZE_LATEST` — kernel asserts struct stability at compile time.

### compatibility contract

- REQ-1: Argument lowering: `%rdi=dirfd (i32)`, `%rsi=filename`, `%rdx=how`, `%rcx`/`%r10=usize`.
- REQ-2: `usize < OPEN_HOW_SIZE_VER0` ⟹ `-EINVAL`.
- REQ-3: `usize > PAGE_SIZE` ⟹ `-E2BIG`.
- REQ-4: `copy_struct_from_user(&tmp, sizeof(tmp), how, usize)` — extensible-struct ABI:
  - If `usize > sizeof(struct open_how)`, trailing bytes MUST be all-zero else `-E2BIG`.
  - If `usize < sizeof(struct open_how)`, missing bytes are zero-filled.
- REQ-5: `audit_openat2_how(&tmp)` records the `how` for syscall auditing.
- REQ-6: If `!(tmp.flags & O_PATH) && force_o_largefile()` ⟹ `tmp.flags |= O_LARGEFILE`. (Note: unlike `open`/`openat`, `O_PATH` opens never get `O_LARGEFILE` auto-added.)
- REQ-7: `build_open_flags(&tmp, &op)`:
  - `flags & ~VALID_OPEN_FLAGS ⟹ -EINVAL` (strict, unlike legacy syscalls).
  - `resolve & ~VALID_RESOLVE_FLAGS ⟹ -EINVAL`.
  - `(resolve & RESOLVE_BENEATH) && (resolve & RESOLVE_IN_ROOT) ⟹ -EINVAL`.
  - If `WILL_CREATE(flags)` ⟹ `mode & ~S_IALLUGO ⟹ -EINVAL`; else `mode != 0 ⟹ -EINVAL`.
  - All `open(2)` flag-combo rejections apply (`O_DIRECTORY|O_CREAT`, `O_TMPFILE` without `O_DIRECTORY`, `O_PATH` with disallowed extras).
- REQ-8: `RESOLVE_*` translates to `LOOKUP_*`:
  - `RESOLVE_NO_XDEV ⟹ LOOKUP_NO_XDEV`
  - `RESOLVE_NO_MAGICLINKS ⟹ LOOKUP_NO_MAGICLINKS`
  - `RESOLVE_NO_SYMLINKS ⟹ LOOKUP_NO_SYMLINKS`
  - `RESOLVE_BENEATH ⟹ LOOKUP_BENEATH`
  - `RESOLVE_IN_ROOT ⟹ LOOKUP_IN_ROOT`
  - `RESOLVE_CACHED`: forbidden alongside `O_TRUNC|O_CREAT|__O_TMPFILE` ⟹ `-EAGAIN`; else `LOOKUP_CACHED`.
- REQ-9: Path resolution failures from RESOLVE bits surface as:
  - `RESOLVE_NO_XDEV` violation ⟹ `-EXDEV`.
  - `RESOLVE_NO_SYMLINKS`/`RESOLVE_NO_MAGICLINKS` violation ⟹ `-ELOOP`.
  - `RESOLVE_BENEATH`/`RESOLVE_IN_ROOT` violation ⟹ `-EXDEV` (escape) or normal `-ENOENT` (in-root reflection).
- REQ-10: Forward-compatibility: kernels supporting only `OPEN_HOW_SIZE_VER0` reject larger structs containing non-zero new fields (`-E2BIG`); future versions read additional fields if `usize` covers them.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `usize_bounds_enforced` | INVARIANT | `OPEN_HOW_SIZE_VER0 <= usize <= PAGE_SIZE`. |
| `unknown_flags_rejected` | INVARIANT | `flags & ~VALID_OPEN_FLAGS != 0 ⟹ -EINVAL`. |
| `unknown_resolve_rejected` | INVARIANT | `resolve & ~VALID_RESOLVE_FLAGS != 0 ⟹ -EINVAL`. |
| `mode_zero_without_create` | INVARIANT | `!WILL_CREATE(flags) && mode != 0 ⟹ -EINVAL`. |
| `beneath_in_root_exclusive` | INVARIANT | `BENEATH && IN_ROOT ⟹ -EINVAL`. |
| `cached_excludes_create_trunc_tmpfile` | INVARIANT | `RESOLVE_CACHED && (O_TRUNC|O_CREAT|__O_TMPFILE) ⟹ -EAGAIN`. |
| `copy_struct_from_user_zero_extension` | INVARIANT | Missing bytes treated as zero; trailing non-zero ⟹ `-E2BIG`. |

### Layer 2: TLA+

`uapi/syscalls/openat2.tla`:
- Per-call → copy_struct_from_user → audit → build_open_flags → path_openat → fd_install.
- Properties:
  - `safety_strict_flag_validation`,
  - `safety_resolve_to_lookup_translation`,
  - `safety_versioned_struct_forward_compat`,
  - `liveness_openat2_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `flags ⊆ VALID_OPEN_FLAGS` post-validation | `Fs::build_open_flags_strict` |
| `resolve ⊆ VALID_RESOLVE_FLAGS` post-validation | `Fs::build_open_flags_strict` |
| `LOOKUP_*` bits derived bijectively from `RESOLVE_*` | `Fs::resolve_to_lookup` |
| Path traversal honoring `RESOLVE_BENEATH` never escapes `dirfd.path` | `Path::walk_with_beneath` |
| Path traversal honoring `RESOLVE_IN_ROOT` never escapes (reflects at root) | `Path::walk_with_in_root` |

### Layer 4: Verus/Creusot functional

`openat2(dirfd, filename, how, usize) ≡ Linux openat2 man-page` semantic equivalence, including all `RESOLVE_*` constraint enforcement.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening; inherits all `openat(2)` hardening.)

`openat2(2)` reinforcement:

- **Per-strict-flag-validation** — defense against attackers passing kernel-internal `FMODE_*` or future-reserved bits.
- **Per-`copy_struct_from_user` extensible ABI** — defense against version-skew silent corruption.
- **Per-`usize` bounds (`>= VER0`, `<= PAGE_SIZE`)** — defense against oversized-struct DoS.
- **Per-`audit_openat2_how`** — every `how` recorded for forensics.
- **Per-`RESOLVE_NO_SYMLINKS`** — defense against symlink-race CVEs (classic TOCTOU).
- **Per-`RESOLVE_NO_MAGICLINKS`** — defense against `/proc/$pid/fd/*` re-open escapes.
- **Per-`RESOLVE_NO_XDEV`** — defense against bind-mount escape (containers).
- **Per-`RESOLVE_BENEATH`** — defense against `..`/absolute-path escape from a sandboxed dirfd.
- **Per-`RESOLVE_IN_ROOT`** — defense against absolute-path leakage (chroot-like without privilege).
- **Per-`RESOLVE_CACHED`** — defense against cache-miss-driven DoS in latency-sensitive paths.
- **Per-`BENEATH|IN_ROOT` exclusion** — defense against semantic ambiguity.
- **Per-`RESOLVE_CACHED` excludes create-flags** — defense against semantic ambiguity (cache cannot create).

### grsecurity/pax-style reinforcement

- **GRKERNSEC_CHROOT_FCHDIR / FINDTASK** — chroot'd `openat2` with `RESOLVE_IN_ROOT` cannot escape chroot.
- **GRKERNSEC_SYMLINKOWN / FIFO / LINK** — additional kernel-policy symlink/FIFO/hardlink protections layered atop `RESOLVE_*`.
- **GRKERNSEC_TPE** — Trusted-Path-Execution applies to `openat2`-loaded executables.
- **PAX_UDEREF** — `how` and `filename` SMAP-guarded.
- **PAX_REFCOUNT** — `dirfd` and `struct file` saturating refcounts.
- **PAX_MEMORY_SANITIZE** — `struct file` slab zeroed on free.
- **GRKERNSEC_AUDIT_MOUNT** — `openat2` traversing a mount logged (in addition to native `RESOLVE_NO_XDEV` enforcement).
- **GRKERNSEC_HIDESYM** — error printks redact kernel pointers.
- **PAX_RANDKSTACK** — kstack offset randomized at syscall entry.

