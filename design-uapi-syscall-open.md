---
title: "Tier-5: syscall 2 — open(2)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`open(2)` is **x86_64 syscall 2**, the legacy AT_FDCWD-relative open entry point. Internally it constructs a `struct open_how` from the legacy `(flags, mode)` pair via `build_open_how`, optionally forces `O_LARGEFILE` on 64-bit, and delegates to `do_sys_openat2(AT_FDCWD, filename, &how)`. The pipeline is `do_sys_openat2` → `build_open_flags` → `do_file_open` → `path_openat` → `do_filp_open` → `vfs_open`. Returns a new file descriptor (lowest available, modulo `RLIMIT_NOFILE`) or a negative errno. `open` is the most flag-rich syscall in the kernel: every bit in `flags` independently steers the lookup, permission check, creation, truncation, and async semantics of the resulting file.

Critical for: every libc `fopen`/`open`, every shell redirect, every Rookery filesystem mount path, every container-runtime `pivot_root` bootstrap, every executable load (`/proc/self/exe`, `ld-linux.so` probes).

### Acceptance Criteria

- [ ] AC-1: `open("/no/such/path", O_RDONLY) == -ENOENT`.
- [ ] AC-2: `open(existing, O_CREAT|O_EXCL, 0600) == -EEXIST`.
- [ ] AC-3: `open(symlink, O_NOFOLLOW) == -ELOOP`.
- [ ] AC-4: `open(file, O_DIRECTORY|O_CREAT, 0700) == -EINVAL`.
- [ ] AC-5: `open(file, O_TMPFILE, 0600) == -EINVAL` (missing `O_DIRECTORY`); `open(dir, O_TMPFILE|O_WRONLY, 0600)` returns an anonymous fd.
- [ ] AC-6: `open(file, O_RDWR|O_PATH) ` strips access bits — resulting fd has `FMODE_PATH`, neither read nor write usable.
- [ ] AC-7: `open(file_owned_by_other, O_RDONLY|O_NOATIME)` as non-CAP_FOWNER → `-EPERM`.
- [ ] AC-8: `open(largefile, O_RDONLY)` on 32-bit without `O_LARGEFILE` → `-EOVERFLOW`; on 64-bit kernel forces `O_LARGEFILE`.
- [ ] AC-9: After successful open with `O_CLOEXEC`, `fcntl(fd, F_GETFD) & FD_CLOEXEC != 0` atomically (no race window).
- [ ] AC-10: `open` returns smallest unused fd; closing fd 3 then opening returns fd 3.
- [ ] AC-11: Unknown bits in `flags` are silently stripped (no `-EINVAL`).

### Architecture

```
struct OpenArgs { filename: UserPtr<u8>, flags: i32, mode: u16 }
```

`sys_open(args) -> i32`:

1. `let mut flags = args.flags as u32;`
2. If `force_o_largefile()` ⟹ `flags |= O_LARGEFILE;`
3. Return `do_sys_open(AT_FDCWD, args.filename, flags, args.mode)`.

`do_sys_open(dfd, filename, flags, mode) -> i32`:

1. `let how = build_open_how(flags, mode);`
2. Return `do_sys_openat2(dfd, filename, &how)`.

`build_open_how(flags, mode) -> OpenHow`:

1. `flags &= VALID_OPEN_FLAGS;`
2. `mode &= S_IALLUGO;`
3. If `flags & O_PATH` ⟹ `flags &= O_PATH_FLAGS;`
4. If `!WILL_CREATE(flags)` ⟹ `mode = 0;`
5. Return `{ flags, mode, resolve: 0 }`.

`do_sys_openat2(dfd, filename, how) -> i32`:

1. `let op = build_open_flags(how)?;`
2. `let name = getname(filename)?;`
3. `let f = do_file_open(dfd, name, &op)?;`
4. `let fd = get_unused_fd_flags(how.flags)?;`
5. `fd_install(fd, f);`
6. Return `fd`.

`build_open_flags(how, op) -> Result<(), errno>`:

1. Validate `flags & ~VALID_OPEN_FLAGS == 0` (only matters for `openat2`).
2. Validate `(resolve & RESOLVE_BENEATH) && (resolve & RESOLVE_IN_ROOT)` mutually exclusive.
3. Compute `acc_mode = ACC_MODE(flags)`.
4. Apply `O_DIRECTORY|O_CREAT` reject.
5. Apply `O_TMPFILE` requires `O_DIRECTORY|MAY_WRITE`.
6. Apply `O_PATH` strip.
7. `O_SYNC` ⟹ also set `O_DSYNC`.
8. `O_TRUNC` ⟹ `acc_mode |= MAY_WRITE`.
9. `O_APPEND` ⟹ `acc_mode |= MAY_APPEND`.
10. `O_NOFOLLOW` ⟹ clear `LOOKUP_FOLLOW`; `O_CREAT|O_EXCL` ⟹ implicit `O_NOFOLLOW`.
11. Translate `RESOLVE_*` to `LOOKUP_*` bits.
12. Write `op->open_flag / acc_mode / intent / lookup_flags`.

### Out of Scope

- `openat(2)` (separate Tier-5 — `openat.md`).
- `openat2(2)` (separate Tier-5 — `openat2.md`).
- `creat(2)` (legacy wrapper; equivalent to `open(path, O_CREAT|O_WRONLY|O_TRUNC, mode)`).
- `name_to_handle_at` / `open_by_handle_at` (covered in separate doc).
- LSM-internal policy machinery (covered in `security/00-overview.md`).
- Implementation code.

### signature

C (POSIX / man-pages):

```c
int open(const char *pathname, int flags, ... /* mode_t mode */ );
```

glibc wrapper: `__libc_open` → `INLINE_SYSCALL(open, 3, pathname, flags, mode)` (mode unused unless `O_CREAT|O_TMPFILE`).

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE3(open, const char __user *, filename, int, flags, umode_t, mode);
```

Rookery dispatch:

```rust
pub fn sys_open(filename: UserPtr<u8>, flags: i32, mode: u16) -> SyscallResult<i32>;
```

### parameters

| name     | type                   | constraints                                                  | errno-on-bad |
|----------|------------------------|--------------------------------------------------------------|--------------|
| filename | `const char __user *`  | NUL-terminated; length `< PATH_MAX`                          | `EFAULT` / `ENAMETOOLONG` |
| flags    | `int`                  | subset of `VALID_OPEN_FLAGS` (older syscall silently strips) | (silently stripped) |
| mode     | `umode_t`              | `S_IALLUGO` (`07777`); honored only if `O_CREAT` or `O_TMPFILE` | (silently stripped) |

### return value

- Success: `>= 0` — new file descriptor (smallest available index unless `O_PATH` etc.).
- Failure: `< 0` — negated errno.

### errors

| errno         | condition                                                                            |
|---------------|--------------------------------------------------------------------------------------|
| `EFAULT`      | `filename` outside userspace.                                                        |
| `ENOENT`      | Component of path does not exist (without `O_CREAT`), or empty-path without `AT_EMPTY_PATH`. |
| `ENOTDIR`     | A non-final component is not a directory, or `O_DIRECTORY` and target is not a directory. |
| `EISDIR`      | Target is a directory and access mode is not read-only (or `O_CREAT|O_WRONLY` etc.). |
| `EACCES`      | LSM/permission denial on lookup or open.                                             |
| `EROFS`       | Write access requested on read-only fs.                                              |
| `ETXTBSY`     | Write access to a currently-executing binary.                                        |
| `EEXIST`      | `O_CREAT|O_EXCL` and target exists.                                                  |
| `ELOOP`       | Symlink loop or `O_NOFOLLOW` and final component is a symlink.                       |
| `ENAMETOOLONG`| path length > `PATH_MAX` or component > `NAME_MAX`.                                  |
| `EMFILE`      | Per-process fd limit (`RLIMIT_NOFILE`) reached.                                      |
| `ENFILE`      | System-wide open-file table full.                                                    |
| `ENXIO`       | `O_NONBLOCK|O_WRONLY` on FIFO with no reader, or device absent.                      |
| `EINVAL`      | Invalid flag combination (e.g., `O_DIRECTORY|O_CREAT`, `O_TMPFILE` without `O_DIRECTORY`, `O_PATH` with disallowed extras). |
| `EOVERFLOW`   | Without `O_LARGEFILE`, file is larger than `MAX_NON_LFS`.                            |
| `EBUSY`       | `O_EXCL` on a block device already mounted.                                          |
| `ENODEV`      | Special file refers to non-existent device.                                          |
| `ENOSPC`      | Out of disk / inodes for `O_CREAT`.                                                  |
| `EFBIG`       | `O_LARGEFILE` not set + 32-bit overflow.                                             |
| `EPERM`       | `O_NOATIME` and caller is not owner/CAP_FOWNER.                                      |
| `EWOULDBLOCK` | `O_NONBLOCK` and file is locked / would block.                                       |
| `EOPNOTSUPP`  | `O_TMPFILE` on filesystem without tmpfile support.                                   |
| `EFAULT`      | mode/flag struct unreadable (n/a here; relevant for `openat2`).                      |

### abi surface (constants + flags)

Access mode (mutually exclusive, low 2 bits of `flags`):

- `O_RDONLY` = `0`
- `O_WRONLY` = `1`
- `O_RDWR`   = `2`
- (`O_ACCMODE` = `3` mask)

Creation flags (file-creation semantics; honored only at open-time):

- `O_CREAT`     `00000100` — create if absent (consults `mode`).
- `O_EXCL`      `00000200` — with `O_CREAT`, fail if exists; without `O_CREAT`, undefined (kernel treats as no-op or `EEXIST` for block devices).
- `O_NOCTTY`    `00000400` — terminal device does not become controlling tty.
- `O_TRUNC`     `00001000` — truncate to length 0 if a regular file and writable.
- `__O_TMPFILE` `020000000`; user constant `O_TMPFILE = __O_TMPFILE | O_DIRECTORY` — anonymous tmpfile in named directory.

Status flags (settable later via `F_SETFL`):

- `O_APPEND`    `00002000` — atomic seek-to-end before each write.
- `O_NONBLOCK`  `00004000` — non-blocking I/O (also `O_NDELAY`).
- `O_DSYNC`     `00010000` — data-only sync on each write.
- `__O_SYNC`    `04000000` (full `O_SYNC = __O_SYNC | O_DSYNC`) — sync data + metadata.
- `O_DIRECT`    `00040000` — bypass page cache.
- `O_LARGEFILE` `00100000` — allow file size > 2GiB on 32-bit ABIs.

Lookup-modifying flags:

- `O_DIRECTORY` `00200000` — must be a directory.
- `O_NOFOLLOW`  `00400000` — fail if final component is a symlink.
- `O_NOATIME`   `01000000` — skip atime updates.
- `O_CLOEXEC`   `02000000` — close-on-exec set on returned fd.
- `O_PATH`      `010000000` — open as a name handle only (no read/write); only `O_DIRECTORY|O_NOFOLLOW|O_CLOEXEC` allowed alongside.

Mode bits (`umode_t`, masked by `S_IALLUGO = 07777` after umask):

- `S_ISUID` (`04000`), `S_ISGID` (`02000`), `S_ISVTX` (`01000`).
- Owner: `S_IRUSR` (`0400`), `S_IWUSR` (`0200`), `S_IXUSR` (`0100`).
- Group: `S_IRGRP` (`0040`), `S_IWGRP` (`0020`), `S_IXGRP` (`0010`).
- Other: `S_IROTH` (`0004`), `S_IWOTH` (`0002`), `S_IXOTH` (`0001`).

`VALID_OPEN_FLAGS` covers all of the above; `open(2)` (unlike `openat2`) silently strips anything outside this set, never returning `-EINVAL` for unknown bits.

### compatibility contract

- REQ-1: Argument lowering: `%rdi=filename`, `%rsi=flags`, `%rdx=mode` (mode is `umode_t` = `unsigned short`, zero-extended).
- REQ-2: On 64-bit, `force_o_largefile()` ORs `O_LARGEFILE` into `flags` before dispatch.
- REQ-3: `build_open_how(flags, mode)` masks `flags &= VALID_OPEN_FLAGS` and `mode &= S_IALLUGO` (silent strip).
- REQ-4: If `O_PATH` is set, all flags outside `{O_DIRECTORY, O_NOFOLLOW, O_PATH, O_CLOEXEC}` are masked out (`O_PATH beats everything else`).
- REQ-5: `mode` is forced to `0` unless `O_CREAT` or `__O_TMPFILE` is set.
- REQ-6: `O_DIRECTORY | O_CREAT` is rejected with `-EINVAL` (and by extension protects `O_TMPFILE` which requires `O_DIRECTORY`).
- REQ-7: `__O_TMPFILE` requires `O_DIRECTORY` and write access (`MAY_WRITE`); else `-EINVAL`.
- REQ-8: `O_TRUNC` implies `MAY_WRITE` in the LSM permission check.
- REQ-9: `O_APPEND` implies `MAY_APPEND` in the LSM permission check.
- REQ-10: `__O_SYNC` implies `O_DSYNC` (kernel ORs `O_DSYNC` so all syncing code paths trigger).
- REQ-11: `O_NOATIME` requires owner or `CAP_FOWNER` — else `-EPERM` raised inside `may_open`.
- REQ-12: `O_CREAT|O_EXCL` ⟹ `O_NOFOLLOW` is implicitly added (kernel sets it so `EEXIST` on a symlink target is consistent).
- REQ-13: A successful `open` returns the smallest unused fd in `current->files`; `O_CLOEXEC` is set atomically with the fd allocation via `FD_ADD(O_CLOEXEC, fd)`.
- REQ-14: Without `O_LARGEFILE`, a regular file with `i_size > MAX_NON_LFS` ⟹ `-EOVERFLOW` (enforced in `generic_file_open`).
- REQ-15: `RLIMIT_NOFILE` is enforced at `__alloc_fd`; exceeding ⟹ `-EMFILE`.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_masked_to_valid` | INVARIANT | `flags & ~VALID_OPEN_FLAGS == 0` post-`build_open_how`. |
| `mode_zero_unless_create` | INVARIANT | `!WILL_CREATE(flags) ⟹ mode == 0`. |
| `o_path_strip` | INVARIANT | `O_PATH ⟹ flags & ~O_PATH_FLAGS == 0`. |
| `cloexec_atomic_with_fd` | INVARIANT | If `O_CLOEXEC` requested, `FD_CLOEXEC` set before `fd_install` returns. |
| `rlimit_nofile` | INVARIANT | `get_unused_fd_flags` returns `-EMFILE` once `current->files->next_fd >= RLIMIT_NOFILE`. |

### Layer 2: TLA+

`uapi/syscalls/open.tla`:
- Per-call → mask → getname → do_file_open → fd_install.
- Properties:
  - `safety_invalid_combinations_reject`,
  - `safety_o_path_strips_access`,
  - `safety_o_cloexec_atomic`,
  - `liveness_path_resolution_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Successful open ⟹ `current->files[fd]` ↦ new file | `Fs::open` |
| `O_CREAT|O_EXCL` ⟹ no pre-existing file at path | `Fs::path_openat` |
| `O_NOATIME` permission ⟹ owner or CAP_FOWNER | `Fs::may_open` |
| `O_LARGEFILE` ⟹ no `EOVERFLOW` for large files | `Fs::generic_file_open` |

### Layer 4: Verus/Creusot functional

`open(filename, flags, mode) ≡ POSIX.1-2024 open` semantic equivalence.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`open(2)` reinforcement:

- **Per-`VALID_OPEN_FLAGS` mask** — defense against attacker passing kernel-internal `FMODE_*` bits.
- **Per-`O_PATH` strip** — defense against `O_PATH`-with-RW confusion.
- **Per-`O_DIRECTORY|O_CREAT` reject** — defense against historical CVE pattern (regular file at directory path).
- **Per-`O_NOFOLLOW` implicit on `O_EXCL`** — defense against symlink-race CVEs.
- **Per-`MAY_APPEND` LSM hint** — defense against AppArmor/SELinux append-only policy bypass.
- **Per-`O_NOATIME` CAP_FOWNER gate** — defense against forensic-evasion via non-owner.
- **Per-`generic_file_open` `O_LARGEFILE`** — defense against silent truncation of legacy 32-bit callers.
- **Per-`fd_install` atomic with `O_CLOEXEC`** — defense against `exec`-leak race.
- **Per-`getname` length cap (`PATH_MAX`)** — defense against unbounded copy from userspace.
- **Per-`RLIMIT_NOFILE`** — defense against fd-exhaustion DoS.

### grsecurity/pax-style reinforcement

- **GRKERNSEC_CHROOT_FINDTASK / CHROOT_DOUBLE / CHROOT_FCHDIR** — chroot'd opens cannot escape the chroot via `..`, `/proc/self/root`, or fd inheritance.
- **GRKERNSEC_SYMLINKOWN** — symlink traversal denied if symlink and target have different owners in sticky world-writable directories.
- **GRKERNSEC_FIFO** — pipe/FIFO open denied if non-owner and sticky-dir.
- **GRKERNSEC_LINK** — hardlink-to-foreign-file blocked at lookup time.
- **GRKERNSEC_TPE** — Trusted-Path-Execution restricts executable opens to root-owned, root-writable paths.
- **GRKERNSEC_AUDIT_MOUNT** — opens that traverse a mount transition logged.
- **PAX_UDEREF** — `filename` user pointer SMAP-guarded in `getname`.
- **PAX_MEMORY_SANITIZE** — `struct file` slab freed at fput is zeroed.
- **PAX_REFCOUNT** — `file->f_count` saturating; underflow ⟹ `BUG()`.
- **GRKERNSEC_HIDESYM** — open failures redact kernel pointers in dmesg.

