# Tier-5: syscall 217 — getdents64(2)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - arch/x86/entry/syscalls/syscall_64.tbl (`217  common  getdents64  sys_getdents64`)
  - fs/readdir.c (`SYSCALL_DEFINE3(getdents64, ...)`, `iterate_dir`, `filldir64`, `verify_dirent_name`)
  - include/uapi/linux/dirent.h (`struct linux_dirent64`)
  - include/linux/fs.h (`DT_*` file-type constants, `struct dir_context`)
-->

## Summary

`getdents64(2)` is **x86_64 syscall 217**, the 64-bit directory-entry retrieval primitive. It reads zero or more `struct linux_dirent64` records from an open directory fd into a user-supplied buffer, each record carrying `d_ino` (64-bit inode number), `d_off` (cookie for next-call resumption), `d_reclen` (variable record length), `d_type` (file-type hint, `DT_*`), and a NUL-terminated `d_name`. The directory `f_pos` cursor is advanced by the filesystem's `iterate_shared`/`iterate` op via `dir_context::pos`. Userspace MUST iterate by `d_reclen` byte strides — record lengths are not fixed — until the syscall returns `0` (EOF) or `-errno`.

Critical for: every libc `readdir`/`readdir_r`, every `ls`/`find`/`fts`, every Rookery VFS dirent iteration test, every container-runtime image-extract that walks layer trees, every `glob`/`fnmatch` shell expansion.

## Signature

C (Linux-specific):

```c
ssize_t getdents64(unsigned int fd, void *dirp, unsigned int count);
```

glibc wrapper: `__getdents64` → `INLINE_SYSCALL(getdents64, 3, fd, dirp, count)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE3(getdents64, unsigned int, fd,
                struct linux_dirent64 __user *, dirent, unsigned int, count);
```

Rookery dispatch:

```rust
pub fn sys_getdents64(fd: u32, dirp: UserPtr<u8>, count: u32) -> SyscallResult<i64>;
```

## Parameters

| name   | type            | constraints                                                              | errno-on-bad |
|--------|-----------------|--------------------------------------------------------------------------|--------------|
| fd     | `unsigned int`  | open in `current->files->fdt`; underlying file MUST be a directory       | `EBADF` / `ENOTDIR` |
| dirp   | `void __user *` | writable for `count` bytes                                               | `EFAULT` |
| count  | `unsigned int`  | `> 0`; SHOULD be `>= sizeof(struct linux_dirent64) + NAME_MAX + 1`       | `EINVAL` (if too small for one entry) |

## Return value

- Success: `> 0` — bytes written into `dirp`. The contents are a concatenation of variable-length `struct linux_dirent64` records.
- `0` — end of directory.
- Failure: `< 0` — negated errno.

## Errors

| errno         | condition                                                                                |
|---------------|------------------------------------------------------------------------------------------|
| `EBADF`       | `fd` is not open.                                                                        |
| `EFAULT`      | `dirp` is not writable for `count` bytes.                                                |
| `EINVAL`      | `count` too small to hold even one record (the smallest possible entry overflows).       |
| `ENOTDIR`     | `fd` does not refer to a directory.                                                      |
| `ENOENT`      | The directory was unlinked while open (unusual; some fs return success-EOF instead).     |
| `EOVERFLOW`   | Resulting `d_off` cannot be represented (legacy `getdents`-only; not for getdents64).    |
| `EIO`         | Filesystem read error during directory traversal.                                        |

## ABI surface (constants + flags)

`struct linux_dirent64` (`include/uapi/linux/dirent.h`):

```c
struct linux_dirent64 {
    __u64          d_ino;     /* 64-bit inode number */
    __s64          d_off;     /* opaque cursor for next call */
    unsigned short d_reclen;  /* record length (multiple of 8) */
    unsigned char  d_type;    /* DT_* file-type hint */
    char           d_name[];  /* NUL-terminated filename */
};
```

`d_type` values (`include/linux/fs.h`):

- `DT_UNKNOWN = 0` — fs cannot determine type cheaply (caller must `stat` to learn).
- `DT_FIFO    = 1` — named pipe.
- `DT_CHR     = 2` — character device.
- `DT_DIR     = 4` — directory.
- `DT_BLK     = 6` — block device.
- `DT_REG     = 8` — regular file.
- `DT_LNK     = 10` — symbolic link.
- `DT_SOCK    = 12` — UNIX-domain socket.
- `DT_WHT     = 14` — whiteout (overlayfs / union mounts).

`NAME_MAX = 255`. `d_reclen` is rounded up to an 8-byte alignment so the next record starts aligned.

Related kernel symbols:

- `struct dir_context` — readdir iterator carrying `actor` callback and current `pos`.
- `iterate_dir(file, ctx)` — VFS-level entry point that locks the directory inode, calls `f_op->iterate_shared` (or `iterate`), and updates `file->f_pos`.
- `filldir64` — `actor` callback that emits one `linux_dirent64` into the user buffer.
- `verify_dirent_name(name, len)` — rejects entries with embedded NUL or invalid names.

## Compatibility contract

- REQ-1: Argument lowering: `%rdi=fd (u32)`, `%rsi=dirp ptr`, `%rdx=count (u32)`. Return in `%rax`.
- REQ-2: `let f = fdget_pos(fd)?;` — fails with `-EBADF`. `FMODE_ATOMIC_POS` is honored so `f_pos` is mutated under `f_pos_lock`.
- REQ-3: If `!(f.f_mode & FMODE_READ) ⟹ -EBADF`.
- REQ-4: If `!S_ISDIR(file_inode(f).i_mode) ⟹ -ENOTDIR`.
- REQ-5: Construct `struct getdents_callback64 { ctx: { actor: filldir64, pos: f.f_pos }, count, error: 0, current_dir: dirp }`.
- REQ-6: Call `iterate_dir(f, &ctx)`; the fs walks entries starting at `ctx.pos`, calling `filldir64(name, len, offset, ino, type)` for each.
- REQ-7: `filldir64`:
  - Compute `reclen = ALIGN(offsetof(linux_dirent64, d_name) + len + 1, 8)`.
  - If `count_remaining < reclen` ⟹ set `ctx.error = -EINVAL` if no entry has been written yet **and** the first entry is too large; else set `ctx.error = 0` and return `-EINVAL` to stop iteration (the buffer is full).
  - `verify_dirent_name(name, len)` ⟹ rejects embedded NUL.
  - `unsafe_put_user` each field; on `EFAULT` ⟹ `ctx.error = -EFAULT`, abort.
  - Decrement `count_remaining` by `reclen`; advance `dirp`.
- REQ-8: After `iterate_dir` returns, the syscall returns:
  - `ctx.error` if set negative,
  - else `(initial_count - ctx.count_remaining)` (bytes written),
  - else `0` if EOF reached without writing anything.
- REQ-9: `d_off` is opaque — userspace MUST NOT compute it. It is whatever the fs chooses (often the file-position cookie for the next entry).
- REQ-10: `d_reclen` MAY differ between consecutive entries; userspace iterates by adding `d_reclen` to the current record pointer.
- REQ-11: Race: concurrent `unlink`/`rename` may produce stale entries or skip entries — POSIX-allowed; the directory itself remains consistent (no torn `linux_dirent64`).
- REQ-12: `fdput_pos(f)` on exit.

## Acceptance Criteria

- [ ] AC-1: `getdents64(closed_fd, buf, 4096) == -EBADF`.
- [ ] AC-2: `getdents64(reg_file_fd, buf, 4096) == -ENOTDIR`.
- [ ] AC-3: `getdents64(NULL pointer, count) == -EFAULT`.
- [ ] AC-4: `getdents64(dir_fd, buf, 1) == -EINVAL` if the first entry doesn't fit.
- [ ] AC-5: A fresh directory traversal produces records with `d_reclen` multiple of 8 and `d_name` NUL-terminated.
- [ ] AC-6: `d_type ∈ {DT_UNKNOWN, DT_FIFO, DT_CHR, DT_DIR, DT_BLK, DT_REG, DT_LNK, DT_SOCK, DT_WHT}`.
- [ ] AC-7: Repeated `getdents64` calls eventually return `0` after enumerating all entries.
- [ ] AC-8: `lseek(dir_fd, 0, SEEK_SET)` followed by `getdents64` restarts enumeration.
- [ ] AC-9: Returned buffer contains no embedded-NUL `d_name`s.
- [ ] AC-10: Concurrent `unlink` during iteration does not corrupt buffer or crash; entries may legitimately be missing.

## Architecture

```
struct Getdents64Args { fd: u32, dirp: UserPtr<u8>, count: u32 }
```

`sys_getdents64(args) -> i64`:

1. `let f = fdget_pos(args.fd).ok_or(EBADF)?;`
2. If `!S_ISDIR(file_inode(&f).i_mode) ⟹ fdput_pos; return -ENOTDIR`.
3. `let mut ctx = DirentCallback64 { actor: filldir64, pos: f.f_pos, current: args.dirp, remaining: args.count, error: 0, prev: None };`
4. `iterate_dir(&f, &mut ctx)`;
5. `f.f_pos = ctx.pos;` (already updated by iterate_dir)
6. `fdput_pos(f);`
7. If `ctx.error < 0` ⟹ return `ctx.error`.
8. Return `(args.count - ctx.remaining) as i64`.

`Fs::filldir64(ctx, name, namlen, offset, ino, dtype) -> i32`:

1. `let reclen = align_up(offsetof(linux_dirent64, d_name) + namlen + 1, 8);`
2. If `ctx.remaining < reclen`:
   - If `ctx.prev.is_none()` ⟹ `ctx.error = -EINVAL; return -EINVAL;`
   - else `return -EINVAL;` (signal "buffer full, stop")
3. `verify_dirent_name(name, namlen).map_err(|_| { ctx.error = -EIO; -EIO })?;`
4. Atomic user-stores via `unsafe_put_user`:
   - `d_ino = ino`, `d_off = offset`, `d_reclen = reclen`, `d_type = dtype`, `d_name = name`, trailing `\0`.
5. `ctx.prev = Some(ctx.current); ctx.current += reclen; ctx.remaining -= reclen;`
6. `ctx.pos = offset;`
7. Return `0`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `reclen_aligned_8` | INVARIANT | Every emitted `d_reclen` is a multiple of 8. |
| `name_nul_terminated` | INVARIANT | Every `d_name` ends with `\0`. |
| `no_buffer_overrun` | INVARIANT | Cumulative bytes written ≤ `count`. |
| `first_entry_einval` | INVARIANT | First record too large ⟹ `-EINVAL` with no bytes written. |
| `efault_short_circuits` | INVARIANT | `unsafe_put_user` failure ⟹ `-EFAULT` and iteration stops. |
| `name_has_no_embedded_nul` | INVARIANT | `verify_dirent_name` rejects names with `\0` interior. |

### Layer 2: TLA+

`uapi/syscalls/getdents64.tla`:
- Per-call → fdget_pos → iterate_dir loop → filldir64 → fdput_pos.
- Properties:
  - `safety_no_buffer_overrun`,
  - `safety_reclen_alignment`,
  - `liveness_getdents64_eventually_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post: `f.f_pos` monotonically non-decreasing across successful calls | `Fs::sys_getdents64` |
| Post: bytes_written ≤ count | `Fs::filldir64` |
| Post: every written record satisfies `d_reclen >= minimum && d_reclen % 8 == 0` | `Fs::filldir64` |

### Layer 4: Verus/Creusot functional

`getdents64(fd, dirp, count)` ≡ Linux getdents64(2) per `man 2 getdents64`; POSIX equivalent is `readdir(3)` (library-level).

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`getdents64(2)` reinforcement:

- **Per-`verify_dirent_name`** — defense against name-injection (`/`, `\0`) reaching userspace.
- **Per-`reclen` alignment enforcement** — defense against unaligned parser surprises in libc.
- **Per-`first_entry_einval`** — defense against silent buffer-too-small data loss.
- **Per-`unsafe_put_user` short-circuit** — defense against partial-fault leaving inconsistent buffer.
- **Per-`fdget_pos`/`fdput_pos`** — defense against `f_pos` race with concurrent `getdents`.
- **Per-`iterate_dir` shared lock** — defense against torn directory traversal during `rename`.

## Grsecurity/PaX-style Reinforcement

- **PAX_UDEREF** — `dirp` SMAP-guarded across every `unsafe_put_user`.
- **GRKERNSEC_CHROOT_FCHDIR** — chroot'd process cannot escape via dirfd inherited pre-chroot.
- **GRKERNSEC_PROC_USER** — `/proc/<pid>` directory listing filtered per-uid.
- **GRKERNSEC_LINK** — hardlink visibility in protected dirs follows owner check.
- **GRKERNSEC_FIFO** — FIFO entries in sticky dirs surface DT_FIFO but creation gated.
- **GRKERNSEC_SYMLINKOWN** — symlink entries surface DT_LNK; traversal gated separately.
- **GRKERNSEC_HIDESYM** — getdents64 error printks redact kernel pointers.
- **PAX_REFCOUNT** — directory inode refcount saturating.
- **PAX_MEMORY_SANITIZE** — `dirent_callback64` stack scrubbed on return.
- **PAX_RANDKSTACK** — kstack offset randomized at getdents64 syscall entry.
- **GRKERNSEC_DMESG** — getdents64-error dmesg lines CAP_SYSLOG-gated.
- **GRKERNSEC_AUDIT_CHDIR** — directory-fd reads on chroot boundary auditable.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `getdents(2)` (legacy 32-bit-inode variant; superseded by getdents64).
- `readdir(2)` (single-entry legacy; long deprecated).
- Filesystem-specific `iterate_shared` (covered in fs/<fstype> Tier-3 if expanded).
- Implementation code.
