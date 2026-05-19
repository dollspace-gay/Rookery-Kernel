---
title: "Tier-5: syscall 8 — lseek(2)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`lseek(2)` is **x86_64 syscall 8**, the file-offset reposition primitive. It updates `file->f_pos` of an open seekable fd by anchoring to start (`SEEK_SET`), the current position (`SEEK_CUR`), end-of-file (`SEEK_END`), or to the next data extent / next hole (`SEEK_DATA`/`SEEK_HOLE`) for sparse-file aware filesystems. Pipes, sockets, FIFOs, character devices without seek support, and `epollfd`-like anonymous-inode fds return `-ESPIPE`. The 64-bit kernel uses a 64-bit `off_t` natively; 32-bit ABIs go through `llseek(2)` for 64-bit offsets.

Critical for: every libc `lseek`/`ftello`, every `mmap`-based loader rewinding ELF headers, every database WAL writer adjusting position after fsync, every sparse-file copy tool (`cp --sparse=always`, `rsync --sparse`), every Rookery VFS sparse-aware tests.

### Acceptance Criteria

- [ ] AC-1: `lseek(closed_fd, 0, SEEK_SET) == -EBADF`.
- [ ] AC-2: `lseek(pipe_fd, 0, SEEK_SET) == -ESPIPE`.
- [ ] AC-3: `lseek(file_fd, 100, SEEK_SET) == 100` and `file_fd`'s `f_pos == 100`.
- [ ] AC-4: `lseek(file_fd, 0, SEEK_END) == i_size`.
- [ ] AC-5: `lseek(file_fd, -1, SEEK_SET) == -EINVAL` and `f_pos` unchanged.
- [ ] AC-6: `lseek(file_fd, 999, SEEK_END)` succeeds and returns `i_size + 999` (sparse hole-creation, no I/O).
- [ ] AC-7: `lseek(sparse_fd, offset_inside_hole, SEEK_DATA)` returns position of next data extent, or `-ENXIO` if none.
- [ ] AC-8: `lseek(sparse_fd, offset_inside_data, SEEK_HOLE)` returns position of next hole (or EOF).
- [ ] AC-9: `whence == 5 ⟹ -EINVAL`.
- [ ] AC-10: Concurrent `lseek` + `read` on the same fd see consistent `f_pos` (per `f_pos_lock`).

### Architecture

```
struct LseekArgs { fd: u32, offset: i64, whence: u32 }
```

`sys_lseek(args) -> i64`:

1. If `args.whence > SEEK_MAX` ⟹ return `-EINVAL`.
2. `let f = fdget_pos(args.fd)?;` — returns `-EBADF` if absent.
3. `let retval = vfs_llseek(&f, args.offset, args.whence);`
4. `fdput_pos(f);`
5. Return `retval`.

`Fs::vfs_llseek(file, offset, whence) -> i64`:

1. If `!(file.f_mode & FMODE_LSEEK) ⟹ return -ESPIPE`.
2. `let llseek = file.f_op.llseek.ok_or(ESPIPE)?;`
3. Return `llseek(file, offset, whence)`.

`Fs::generic_file_llseek(file, offset, whence) -> i64`:

1. `let inode = file_inode(file);`
2. `let maxsize = inode.i_sb.s_maxbytes;`
3. Compute new_pos per `whence`:
   - `SEEK_SET ⟹ offset`
   - `SEEK_CUR ⟹ file.f_pos + offset` (overflow ⟹ `-EOVERFLOW`)
   - `SEEK_END ⟹ inode.i_size + offset` (overflow ⟹ `-EOVERFLOW`)
   - `SEEK_DATA|SEEK_HOLE ⟹ delegate to inode->i_op->fiemap`/fs-specific
4. If `new_pos < 0 || new_pos > maxsize ⟹ -EINVAL/EOVERFLOW`.
5. `file.f_pos = new_pos;`
6. Return `new_pos`.

### Out of Scope

- `llseek(2)` (32-bit-compat helper; separate Tier-5 if expanded).
- `pread`/`pwrite` positional I/O (these do not touch `f_pos`).
- `lseek` on `O_APPEND` fds — POSIX allows `lseek` but writes still go to EOF; covered in fs/ Tier-3.
- Implementation code.

### signature

C (POSIX):

```c
off_t lseek(int fd, off_t offset, int whence);
```

glibc wrapper: `__libc_lseek64` → `INLINE_SYSCALL(lseek, 3, fd, offset, whence)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE3(lseek, unsigned int, fd, off_t, offset, unsigned int, whence);
```

Rookery dispatch:

```rust
pub fn sys_lseek(fd: u32, offset: i64, whence: u32) -> SyscallResult<i64>;
```

### parameters

| name   | type           | constraints                                                                | errno-on-bad |
|--------|----------------|----------------------------------------------------------------------------|--------------|
| fd     | `unsigned int` | open in `current->files->fdt`; underlying file must be seekable            | `EBADF` / `ESPIPE` |
| offset | `off_t (i64)`  | within `[-S_MAX, S_MAX]` where `S_MAX` is fs-specific (typically `LLONG_MAX`); SEEK_HOLE/SEEK_DATA bound by EOF | `EINVAL` / `ENXIO` / `EOVERFLOW` |
| whence | `unsigned int` | one of `SEEK_SET`, `SEEK_CUR`, `SEEK_END`, `SEEK_DATA`, `SEEK_HOLE`        | `EINVAL` |

### return value

- Success: new `file->f_pos` as a non-negative `off_t`.
- Failure: `< 0` — negated errno.

### errors

| errno       | condition                                                                                |
|-------------|------------------------------------------------------------------------------------------|
| `EBADF`     | `fd` is not open.                                                                        |
| `EINVAL`    | `whence` is not one of the recognized values; or computed offset is negative; or `SEEK_DATA`/`SEEK_HOLE` requested on fs that lacks support. |
| `ENXIO`     | `SEEK_DATA` past EOF, or `SEEK_HOLE` past EOF without an implicit trailing hole.         |
| `EOVERFLOW` | Resulting offset cannot be represented in `off_t` (relevant for 32-bit compat).         |
| `ESPIPE`    | `fd` refers to a pipe, socket, FIFO, or other non-seekable object.                       |
| `EFAULT`    | (None — `lseek` has no user pointers.)                                                   |

### abi surface (constants + flags)

`whence` constants (`include/uapi/linux/fs.h`):

- `SEEK_SET  = 0` — `new_pos = offset`.
- `SEEK_CUR  = 1` — `new_pos = f_pos + offset`.
- `SEEK_END  = 2` — `new_pos = i_size + offset`.
- `SEEK_DATA = 3` — `new_pos = next data byte at or after offset` (sparse-aware).
- `SEEK_HOLE = 4` — `new_pos = next hole byte at or after offset` (sparse-aware); EOF counts as a virtual hole.
- `SEEK_MAX  = SEEK_HOLE` — upper bound for validation.

Related kernel symbols / file-op helpers:

- `f_op->llseek` — fs-supplied implementation. NULL ⟹ `no_llseek` ⟹ `-ESPIPE`.
- `generic_file_llseek` — handles `SET/CUR/END` for ordinary files.
- `generic_file_llseek_size` — variant taking explicit `maxsize`.
- `no_seek_end_llseek` — used by procfs/sysfs where size is dynamic.
- `noop_llseek` — fd whose offset is irrelevant (returns current `f_pos`).
- `FMODE_LSEEK` (`file->f_mode` bit) — set during `__fdget_pos` for files with non-null `llseek`.
- `f_pos_lock` (`file->f_pos_lock`) — mutex held across `lseek` + `read`/`write` to keep `f_pos` consistent for shared fds.

### compatibility contract

- REQ-1: Argument lowering: `%rdi=fd (u32)`, `%rsi=offset (i64)`, `%rdx=whence (u32)`. Return in `%rax`.
- REQ-2: `whence > SEEK_MAX ⟹ -EINVAL` before any fd lookup.
- REQ-3: `let f = fdget_pos(fd)?;` — grabs reference plus `f_pos_lock` if `FMODE_ATOMIC_POS` is set (regular files with `pos_lock`).
- REQ-4: If `!(f.f_mode & FMODE_LSEEK) || f.f_op.llseek.is_none() ⟹ -ESPIPE`.
- REQ-5: Call `f_op->llseek(file, offset, whence)`; receive the new `f_pos` or negative errno.
- REQ-6: On success, the returned `f_pos` is also written into `file->f_pos` by the `llseek` implementation under whatever locking that fs uses (usually `f_pos_lock`).
- REQ-7: `SEEK_DATA`/`SEEK_HOLE` semantics:
  - Returned position is in `[offset, i_size]`.
  - If no data after `offset` ⟹ `-ENXIO` (`SEEK_DATA`).
  - If no further holes (EOF acts as final virtual hole) ⟹ `new_pos = i_size`.
  - If `offset > i_size ⟹ -ENXIO`.
- REQ-8: `SEEK_END` with positive offset is allowed (no implicit allocation — writing later will extend / sparse-grow).
- REQ-9: Negative resulting position ⟹ `-EINVAL`; `file->f_pos` unchanged.
- REQ-10: `lseek` MUST NOT trigger I/O on ordinary files (it manipulates only metadata).
- REQ-11: Pipe/socket/FIFO ⟹ `-ESPIPE` regardless of `whence`.
- REQ-12: `fput(file)` on exit; if `FMODE_ATOMIC_POS`, `f_pos_lock` is dropped.
- REQ-13: 32-bit compat: `lseek` is limited to `i32` offsets; use `llseek(2)` (syscall 140) for 64-bit. On 64-bit `lseek` covers the full `i64` range.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `whence_validated_first` | INVARIANT | `whence > SEEK_MAX` rejected before any fd lookup. |
| `nonseekable_yields_espipe` | INVARIANT | `f_op->llseek == NULL ⟹ -ESPIPE`. |
| `negative_pos_rejected` | INVARIANT | Resulting `f_pos < 0` ⟹ `-EINVAL`, state unchanged. |
| `no_io_on_metadata_only` | INVARIANT | No `read_iter`/`write_iter` invocation during `lseek`. |
| `f_pos_lock_held_for_atomic` | INVARIANT | `FMODE_ATOMIC_POS ⟹ f_pos_lock` held across update. |

### Layer 2: TLA+

`uapi/syscalls/lseek.tla`:
- Per-call → fdget_pos → llseek → fdput_pos.
- Properties:
  - `safety_f_pos_monotonic_under_set`,
  - `safety_espipe_on_nonseekable`,
  - `liveness_lseek_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post: `file.f_pos == new_pos` on success | `Fs::generic_file_llseek` |
| Post: `file.f_pos` unchanged on error | `Fs::generic_file_llseek` |
| Post: `SEEK_DATA` result ∈ `[offset, i_size]` ∪ `-ENXIO` | `Fs::generic_file_llseek` |

### Layer 4: Verus/Creusot functional

`lseek(fd, offset, whence)` ≡ POSIX.1-2024 `lseek` plus Linux extensions `SEEK_DATA`/`SEEK_HOLE` per `man 2 lseek`.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`lseek(2)` reinforcement:

- **Per-`whence` validation first** — defense against fd-lookup side effects on invalid input.
- **Per-`f_pos_lock` discipline** — defense against torn-read of `f_pos` for shared fds.
- **Per-`SEEK_DATA`/`SEEK_HOLE` capability** — defense against fs that lacks support returning bogus `f_pos`.
- **Per-overflow guard on `SEEK_CUR`/`SEEK_END`** — defense against integer-wrap producing negative `f_pos`.
- **Per-`FMODE_LSEEK` check** — defense against calling `llseek` on noop / null pointers.
- **Per-`fdget_pos`/`fdput_pos`** — defense against fd-reuse race during seek.

### grsecurity/pax-style reinforcement

- **PAX_REFCOUNT** — `struct file` refcount saturating across `fdget_pos`/`fdput_pos`.
- **GRKERNSEC_CHROOT_FCHDIR** — chroot'd processes cannot escape via inherited fd lseek.
- **GRKERNSEC_HIDESYM** — `lseek` error printks redact kernel pointers.
- **PAX_MEMORY_SANITIZE** — `struct file` slab zeroed on free.
- **PAX_RANDKSTACK** — kstack offset randomized at lseek syscall entry.
- **GRKERNSEC_DMESG** — lseek-error dmesg lines CAP_SYSLOG-gated.
- **GRKERNSEC_PROC_USER** — `/proc/$pid/fdinfo/<fd>` `pos:` line only visible to owner.
- **PAX_UDEREF** — no user pointers in lseek; uderef inapplicable but stack discipline retained.
- **GRKERNSEC_AUDIT_CHDIR** — lseek on chroot-boundary fd auditable.

