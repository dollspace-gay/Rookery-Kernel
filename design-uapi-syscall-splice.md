---
title: "Tier-5 syscall: splice(2) — splice data to/from a pipe (syscall 275)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`splice(2)` moves data between two file descriptors **without** copying through user space — using a kernel pipe (real or implicit) as the intermediate buffer. At least one of `fd_in` or `fd_out` must be a pipe. When both are pipes it short-circuits to `splice_pipe_to_pipe`. When one is a regular file/socket and the other is a pipe, the kernel "splices" page references through the pipe, optionally `SPLICE_F_MOVE`-ing the pages (steal-and-attach) rather than copying. It is syscall number **275** on x86-64.

The path is `sys_splice(fd_in, off_in, fd_out, off_out, len, flags)` → `do_splice()` → branches: `splice_pipe_to_pipe`, `do_splice_to` (pipe target), or `do_splice_from` (pipe source) → file_operations.splice_read / .splice_write at the non-pipe end.

Critical for: `tee`, `cat`, `pv`, log-shippers, HTTP/2 proxies, container `runc` stdio passthrough; any chain of pipes through which large data flows.

This Tier-5 covers syscall **275** `splice(int fd_in, off_t *off_in, int fd_out, off_t *off_out, size_t len, unsigned int flags)`.

### Acceptance Criteria

- [ ] AC-1: splice(regfile, NULL, pipe, NULL, 1MB, 0): 1MB moved through pipe; regfile f_pos advanced.
- [ ] AC-2: splice(regfile, &off=4096, pipe, NULL, 1MB, 0): offset writeback = 4096 + 1MB; regfile f_pos unchanged.
- [ ] AC-3: splice(pipe, NULL, regfile, NULL, 1MB, 0): 1MB drained from pipe to regfile.
- [ ] AC-4: splice(pipe-A, NULL, pipe-B, NULL, 1MB, SPLICE_F_MOVE): pages moved via splice_pipe_to_pipe.
- [ ] AC-5: splice(regfile, NULL, regfile, NULL, ...): -EINVAL (neither is pipe).
- [ ] AC-6: splice(pipe, &off=...): -ESPIPE.
- [ ] AC-7: splice(..., flags=0x100): -EINVAL (unknown flag).
- [ ] AC-8: splice(... SPLICE_F_NONBLOCK) with empty pipe: -EAGAIN.
- [ ] AC-9: splice(pipe-A, NULL, pipe-A, NULL, ...): -EINVAL (same pipe both ends).
- [ ] AC-10: splice out to socket peer closed: -EPIPE + SIGPIPE.
- [ ] AC-11: splice in_fd lacks FMODE_READ: -EBADF.
- [ ] AC-12: splice out_fd lacks FMODE_WRITE: -EBADF.
- [ ] AC-13: splice O_APPEND on out_fd ∧ off_out != NULL: -EINVAL.
- [ ] AC-14: splice with len = 0: returns 0.
- [ ] AC-15: splice LSM denial on either fd: -EACCES.

### Architecture

```
SysFs::sys_splice(fd_in, off_in, fd_out, off_out, len, flags) -> SyscallResult<isize>
  1. if flags & !VALID_SPLICE_FLAGS { return Err(EINVAL); }
  2. let in_file = FdTable::fget_light(fd_in)?;
  3. let out_file = FdTable::fget_light(fd_out)?;
  4. if !in_file.f_mode.contains(FileMode::READ)  { return Err(EBADF); }
  5. if !out_file.f_mode.contains(FileMode::WRITE) { return Err(EBADF); }
  6. let in_pipe = PipeInfo::from_file(&in_file);
  7. let out_pipe = PipeInfo::from_file(&out_file);
  8. if in_pipe.is_none() && out_pipe.is_none() { return Err(EINVAL); }
  9. if in_pipe.is_some() && off_in.is_some()  { return Err(ESPIPE); }
 10. if out_pipe.is_some() && off_out.is_some() { return Err(ESPIPE); }
 11. let mut pos_in: i64 = if in_pipe.is_none() {
        if off_in.is_some() {
            let p = UserCopy::get_user_i64(off_in.unwrap())?;
            if p < 0 { return Err(EINVAL); }
            p
        } else { in_file.f_pos.load() }
     } else { 0 };
 12. let mut pos_out: i64 = if out_pipe.is_none() {
        if out_file.f_flags.contains(FileFlags::APPEND) && off_out.is_some() {
            return Err(EINVAL);
        }
        if off_out.is_some() {
            let p = UserCopy::get_user_i64(off_out.unwrap())?;
            if p < 0 { return Err(EINVAL); }
            p
        } else { out_file.f_pos.load() }
     } else { 0 };
 13. let len = core::cmp::min(len, MAX_RW_COUNT);
 14. Lsm::file_permission(&in_file, MayMode::READ)?;
 15. Lsm::file_permission(&out_file, MayMode::WRITE)?;
 16. let ret = match (in_pipe, out_pipe) {
        (Some(ip), Some(op)) if ip.same_pipe(op) => return Err(EINVAL),
        (Some(ip), Some(op)) => SpliceCore::pipe_to_pipe(ip, op, len, flags),
        (Some(ip), None)     => SpliceCore::do_splice_from(ip, &out_file, &mut pos_out, len, flags),
        (None,     Some(op)) => SpliceCore::do_splice_to(&in_file, &mut pos_in, op, len, flags),
        (None,     None)     => unreachable!(),
     }?;
 17. // Writeback positions
     if in_pipe.is_none() {
        if let Some(off_in_ptr) = off_in.option() {
            UserCopy::put_user_i64(pos_in, off_in_ptr)?;
        } else { in_file.f_pos.store(pos_in); }
     }
     if out_pipe.is_none() {
        if let Some(off_out_ptr) = off_out.option() {
            UserCopy::put_user_i64(pos_out, off_out_ptr)?;
        } else { out_file.f_pos.store(pos_out); }
     }
 18. Audit::log_splice(fd_in, fd_out, len, ret, flags);
 19. Ok(ret)
```

`splice_pipe_to_pipe(in_pipe, out_pipe, len, flags)`:
1. lock_ordered(in_pipe, out_pipe).  // ptr-order to avoid deadlock.
2. while bytes_moved < len:
   - if in_pipe.empty(): if NONBLOCK return partial; else wait.
   - if out_pipe.full(): if NONBLOCK return partial; else wait.
   - if SPLICE_F_MOVE: detach pipe_buffer from in; attach to out.
   - else: alloc new buffer, copy contents.
3. unlock_ordered.
4. return bytes_moved.

### Out of Scope

- `sendfile(2)` — separate Tier-5 (single-call variant; implicit pipe).
- `tee(2)`, `vmsplice(2)` — separate Tier-5s.
- `copy_file_range(2)` — separate Tier-5 (file-to-file no-pipe).
- `pipe2(2)` — separate Tier-5 (pipe creation).
- Per-fs `splice_read` / `splice_write` implementations — Tier-3 per fs.
- `do_splice_direct` / `pipe_to_pipe` internals — Tier-3 `fs/splice.md`.
- Implementation code.

### signature

```c
ssize_t splice(int fd_in,  off64_t *off_in,
               int fd_out, off64_t *off_out,
               size_t len, unsigned int flags);
```

```rust
pub fn sys_splice(fd_in: i32, off_in: UserPtr<i64>,
                  fd_out: i32, off_out: UserPtr<i64>,
                  len: usize, flags: u32) -> SyscallResult<isize>;
```

x86-64 entry: `__x64_sys_splice` → `do_splice(in_file, off_in_kern, out_file, off_out_kern, len, flags)`.

### parameters

- **`fd_in`** — source fd. At least one of {fd_in, fd_out} must be a pipe.
- **`off_in`** — IN/OUT: if fd_in is NOT a pipe and non-NULL, start at `*off_in`, do not advance `f_pos`; writeback consumed. Must be NULL if fd_in IS a pipe.
- **`fd_out`** — destination fd.
- **`off_out`** — IN/OUT: if fd_out is NOT a pipe and non-NULL, start at `*off_out`, do not advance `f_pos`; writeback advanced. Must be NULL if fd_out IS a pipe.
- **`len`** — maximum bytes to transfer; clamped at `MAX_RW_COUNT` (0x7FFFF000).
- **`flags`** — bitmask of:
  - `SPLICE_F_MOVE` (0x01): try to move (steal) pages rather than copy. Falls back to copy if move infeasible.
  - `SPLICE_F_NONBLOCK` (0x02): never block on pipe operations. Note: does NOT affect blocking of the non-pipe end; that uses the non-pipe fd's O_NONBLOCK.
  - `SPLICE_F_MORE` (0x04): hint to socket out_fd that more data follows (Nagle-corking).
  - `SPLICE_F_GIFT` (0x08): pages are "gifted" — caller promises not to modify them after the call; allows page-table fast path. Only meaningful for `vmsplice` input.

### return

Number of bytes spliced (≥ 0) on success; `-1`/`-errno` on error. May be less than `len` (partial).

### errors

- `EBADF` — fd_in / fd_out invalid; not readable / not writable.
- `EINVAL` — neither fd is a pipe; offset given for pipe-typed fd; appendmode out_fd with off_out != NULL; flags contain unknown bits; len overflow.
- `EFAULT` — off_in or off_out pointer fault.
- `EAGAIN` — non-blocking and would block.
- `ENOMEM` — pipe-buffer alloc failure.
- `EPIPE` — out_fd's pipe peer closed (+ SIGPIPE).
- `ESPIPE` — offset given for a non-seekable non-pipe fd.
- `EIO` — low-level I/O error.
- `EACCES` / `EPERM` — LSM denial on either fd.

### abi surface

- Pipe is detected by `f_op == &pipefifo_fops` (or via `S_ISFIFO(inode.i_mode)`).
- The "neither is a pipe" case returns `-EINVAL` (use `sendfile` or `copy_file_range`).
- `SPLICE_F_NONBLOCK` only affects the pipe side; the non-pipe side uses its own `O_NONBLOCK`.
- `flags & ~(MOVE|NONBLOCK|MORE|GIFT)` returns `-EINVAL` (mask check).
- `off_in` / `off_out` use `loff_t` (64-bit signed). NULL means "use f_pos".

### compatibility contract

REQ-1: fd lookup:
- in_file = fget_light(fd_in); if !in_file: return `-EBADF`.
- out_file = fget_light(fd_out); if !out_file: return `-EBADF`.

REQ-2: capability checks:
- if !(in_file.f_mode & FMODE_READ): return `-EBADF`.
- if !(out_file.f_mode & FMODE_WRITE): return `-EBADF`.

REQ-3: flag mask:
- if flags & ~(SPLICE_F_MOVE | SPLICE_F_NONBLOCK | SPLICE_F_MORE | SPLICE_F_GIFT):
  - return `-EINVAL`.

REQ-4: pipe presence:
- in_pipe = get_pipe_info(in_file).
- out_pipe = get_pipe_info(out_file).
- if !in_pipe ∧ !out_pipe: return `-EINVAL`.

REQ-5: offset rules:
- if in_pipe ∧ off_in != NULL: return `-ESPIPE`.
- if out_pipe ∧ off_out != NULL: return `-ESPIPE`.
- if !in_pipe ∧ off_in == NULL: pos_in = in_file.f_pos.
- if !in_pipe ∧ off_in != NULL: copy_from_user(&pos_in, off_in, 8); if pos_in < 0: -EINVAL.
- if !out_pipe ∧ off_out == NULL: pos_out = out_file.f_pos.
- if !out_pipe ∧ off_out != NULL: copy_from_user(&pos_out, off_out, 8); if pos_out < 0: -EINVAL.
- if !out_pipe ∧ (out_file.f_flags & O_APPEND) ∧ off_out != NULL: return `-EINVAL`.

REQ-6: len clamp:
- if len > MAX_RW_COUNT: len = MAX_RW_COUNT.

REQ-7: LSM:
- err = security_file_permission(in_file, MAY_READ); if err: return err.
- err = security_file_permission(out_file, MAY_WRITE); if err: return err.

REQ-8: Dispatch:
- if in_pipe ∧ out_pipe: ret = splice_pipe_to_pipe(in_pipe, out_pipe, len, flags).
- elif in_pipe: ret = do_splice_from(in_pipe, out_file, &pos_out, len, flags).
- else: ret = do_splice_to(in_file, &pos_in, out_pipe, len, flags).

REQ-9: offset writeback:
- if !in_pipe ∧ off_in != NULL: copy_to_user(off_in, &pos_in, 8).
- if !in_pipe ∧ off_in == NULL: in_file.f_pos = pos_in.
- if !out_pipe ∧ off_out != NULL: copy_to_user(off_out, &pos_out, 8).
- if !out_pipe ∧ off_out == NULL: out_file.f_pos = pos_out.

REQ-10: Audit:
- audit_log(SYS_SPLICE, fd_in, fd_out, len, ret, flags).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `splice_at_least_one_pipe` | INVARIANT | per-sys_splice: in_pipe || out_pipe (else EINVAL). |
| `splice_pipe_offset_rejected` | INVARIANT | per-sys_splice: pipe-typed fd ∧ off != NULL ⟹ ESPIPE. |
| `splice_flags_mask` | INVARIANT | per-sys_splice: flags & ~VALID_FLAGS == 0. |
| `splice_len_clamped` | INVARIANT | per-sys_splice: len ≤ MAX_RW_COUNT. |
| `splice_offset_nonneg` | INVARIANT | per-sys_splice: pos_in ≥ 0 ∧ pos_out ≥ 0. |
| `splice_in_readable_out_writable` | INVARIANT | per-sys_splice: in.FMODE_READ ∧ out.FMODE_WRITE. |
| `splice_no_same_pipe_self_loop` | INVARIANT | per-sys_splice: in_pipe == out_pipe ⟹ EINVAL. |
| `splice_append_offset_rejected` | INVARIANT | per-sys_splice: !out_pipe ∧ O_APPEND ∧ off_out != NULL ⟹ EINVAL. |

### Layer 2: TLA+

`uapi/syscalls/splice.tla`:
- States: validate, lookup, classify-pipes, lsm, dispatch (3-way), offset-writeback, return.
- Models pipe buffer page-move vs page-copy under SPLICE_F_MOVE.
- Properties:
  - `safety_at_least_one_pipe` — return success ⟹ at least one of fds was a pipe.
  - `safety_no_self_loop_deadlock` — same-pipe both ends ⟹ EINVAL before any locking.
  - `safety_ordered_pipe_locking` — pipe_to_pipe locks in deterministic order to prevent ABBA deadlock.
  - `safety_move_or_copy` — every byte either page-moved or content-copied, never both.
  - `safety_offset_or_fpos_exclusive` — offset != NULL ⟹ f_pos unchanged.
  - `liveness_terminates` — splice returns within bounded steps (when blocking or NONBLOCK).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| pre: flags ⊆ {MOVE,NONBLOCK,MORE,GIFT} | `sys_splice` |
| pre: in_pipe.is_some() \|\| out_pipe.is_some() | `sys_splice` |
| pre: pos_in ≥ 0; pos_out ≥ 0; len ≤ MAX_RW_COUNT | `sys_splice` |
| pre: in_pipe != out_pipe (no self-loop) | `sys_splice` |
| post: ret ≥ 0 ⟹ ret ≤ len | `sys_splice` |
| post: ret > 0 ⟹ exactly ret bytes either page-moved or copied | `splice_pipe_to_pipe` |
| post: offset writeback to user ⟹ pos advanced by ret | `sys_splice` |

### Layer 4: Verus/Creusot functional

Per-`do_splice → splice_pipe_to_pipe / do_splice_from / do_splice_to` semantic equivalence with `fs/splice.c`. Pipe-buffer ref-count / page-move semantics under `SPLICE_F_MOVE` match upstream byte-for-byte. ABBA-deadlock-avoidance lock-ordering matches `pipe_double_lock`.

### hardening

- **At least one fd must be a pipe** — defense against per-arbitrary-fd-to-fd copy bypassing copy_file_range's policy checks.
- **No self-loop (`in_pipe == out_pipe`)** — defense against per-deadlock infinite splice.
- **Pipe-typed fd with non-NULL offset rejected** — defense against per-bogus-offset-on-pipe (pipes have no seek).
- **`flags` strict mask check** — defense against per-future-flag-collision causing surprising semantics.
- **`O_APPEND` out_fd + `off_out != NULL` rejected** — defense against per-append-vs-offset semantic contradiction.
- **`pos_in ≥ 0 ∧ pos_out ≥ 0`** — defense against per-negative-offset fs corruption.
- **`len` clamp to `MAX_RW_COUNT`** — defense against per-overflow on length math.
- **LSM `file_permission(MAY_READ)` on in, `MAY_WRITE` on out** — defense against per-MAC bypass. **This is the central splice fd-permission boundary** check.
- **`pipe_double_lock` ordered by pointer** — defense against per-ABBA-deadlock between two pipes.
- **`SPLICE_F_MOVE` falls back to copy when move infeasible** — defense against per-page-corruption when caller can't actually gift pages.
- **`SPLICE_F_GIFT` only honored for vmsplice input** — defense against per-illusory-page-stability promise on regular files.
- **Non-blocking on pipe side respects `SPLICE_F_NONBLOCK`** — defense against per-deadlock when caller expects non-blocking.
- **Offset writeback bounded under UDEREF** — defense against per-OOB-write into caller's loff_t.

### grsecurity-pax

- **PAX_RANDKSTACK** — kstack base randomized; pos_in/pos_out stack slots randomized.
- **PaX UDEREF** — `off_in`, `off_out` derefs only via `copy_from_user` / `copy_to_user`; direct kernel deref traps.
- **GRKERNSEC_NO_SIMULT_CONNECT** — irrelevant on splice.
- **GRKERNSEC_BLACKHOLE** — irrelevant on splice (no on-wire emission directly; underlying socket emit is gated separately).
- **GRKERNSEC_RANDNET** — irrelevant on splice.
- **CAP_NET_RAW / CAP_NET_ADMIN gates** — splice to a `SOCK_RAW` out_fd revalidates `CAP_NET_RAW` per call under hardened policy.
- **GRKERNSEC_HARDEN_IPC for SOCK_*UNIX** — splice from a regular file to a UNIX-domain socket: hardened policy denies when target is an abstract-namespace socket created in a different user-ns.
- **SOCK_CLOEXEC mandatory under suid** — irrelevant on splice (no fd creation).
- **SCM_RIGHTS recursion bound** — irrelevant on splice (no cmsg; SCM_RIGHTS is sendmsg-only).
- **sendmmsg compound-bound limit** — irrelevant on splice.
- **splice fd-permission boundary** — **the central grsec rule for splice**: both `security_file_permission` calls must succeed; in_fd needs `MAY_READ`, out_fd needs `MAY_WRITE`. Hardened build additionally forbids splice from `/proc/*/mem`, `/proc/kcore`, `/dev/mem`, `/dev/kmem`, and sensitive `/sys` files when the out_fd is **external** to the caller (defeats classic in-kernel-memory-exfil-via-splice attack). Configurable via `grsec.splice_deny_kmem_external`.
- **`grsec.splice_pipe_ownership`** — hardened policy: pipe-to-pipe splice across processes with different `current.real_cred.uid` requires `CAP_SYS_ADMIN`; defeats abuse of inherited pipes for stealthy data exfil between privsep'd processes.
- **getsockopt info-leak prevention** — splice does not read sockopts; pos writeback is bounded; no info leak through this path.

