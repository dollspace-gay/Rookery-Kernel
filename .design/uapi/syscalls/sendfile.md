# Tier-5 syscall: sendfile(2) — transfer data between file descriptors (syscall 40)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/read_write.c (__do_sys_sendfile, do_sendfile, sys_sendfile, sys_sendfile64)
  - fs/splice.c (do_splice_direct, splice_direct_to_actor, do_splice_from)
  - include/linux/fs.h (file_operations.splice_read, file_operations.splice_write)
  - net/socket.c (sock_splice_read - typical out_fd target)
-->

## Summary

`sendfile(2)` transfers data between two file descriptors **inside the kernel**, avoiding the user-space round-trip of `read`+`write`. The source (`in_fd`) must be a file that supports `splice_read` (regular file / block dev), and the destination (`out_fd`) must be a file that supports `splice_write` — historically restricted to sockets, but since 2.6.33 any `splice_write`-capable fd (regular file, pipe, socket). It is syscall number **40** on x86-64.

The path is `sys_sendfile(out_fd, in_fd, offset, count)` → `do_sendfile()` → `do_splice_direct(in, &pos, out, &out_pos, count, SPLICE_F_MOVE | SPLICE_F_NONBLOCK?)` → `splice_direct_to_actor()` plumbs page cache pages from `in_fd` straight to `out_fd`'s `splice_write` handler, without per-byte copy through userspace.

Critical for: HTTP servers (Apache, nginx, FreeBSD/Linux compat), `tar`/`cp` accelerators, sendfile-based file caches. Reduces CPU cost dramatically for large file → socket transfers (the original use case).

This Tier-5 covers syscall **40** `sendfile(int out_fd, int in_fd, off_t *offset, size_t count)` and the 64-bit superset `sendfile64`.

## Signature

```c
ssize_t sendfile  (int out_fd, int in_fd, off_t   *offset, size_t count);
ssize_t sendfile64(int out_fd, int in_fd, off64_t *offset, size_t count);
```

```rust
pub fn sys_sendfile(out_fd: i32, in_fd: i32, offset: UserPtr<i64>, count: usize)
    -> SyscallResult<isize>;
```

x86-64 entry: `__x64_sys_sendfile64` → `do_sendfile(out_fd, in_fd, &pos_kernel, count, max_count)`.

## Parameters

- **`out_fd`** — destination file descriptor (socket, regular file, pipe).
- **`in_fd`** — source file descriptor (regular file / block dev / anything implementing `splice_read`). Must be readable and support `mmap`-style page cache access.
- **`offset`** — IN/OUT, userland pointer: if non-NULL, read starts at `*offset` and the source fd's `f_pos` is NOT modified; on return, `*offset` reflects bytes-past-the-last-read position. If NULL, read starts at the source fd's `f_pos` and `f_pos` is advanced by bytes-sent.
- **`count`** — maximum bytes to transfer. The kernel caps this internally at `MAX_RW_COUNT = 0x7ffff000` (~2 GiB).

## Return

Number of bytes transferred (≥ 0) on success; `-1`/`-errno` on error. May be less than `count` (partial transfer for non-blocking sockets, short reads, etc.).

## Errors

- `EBADF` — out_fd or in_fd not open / not readable / not writable.
- `EINVAL` — fd type cross-validation failed (in_fd lacks `splice_read`; out_fd lacks `splice_write`; in_fd not seekable when offset != NULL); count is negative (when reinterpreted signed).
- `EFAULT` — offset pointer fault.
- `EIO` — low-level I/O error.
- `ENOMEM` — page-cache or pipe-buffer alloc failure.
- `EAGAIN` — out_fd is non-blocking and would block.
- `EPIPE` — out_fd is a socket whose peer closed (+ SIGPIPE if applicable).
- `EOVERFLOW` — sendfile (not sendfile64) and offset doesn't fit in off_t (32-bit on 32-bit archs).
- `ESPIPE` — in_fd is a pipe and offset != NULL (pipes are not seekable).
- `EACCES` / `EPERM` — LSM denial on source or destination.

## ABI surface

- The legacy `sendfile(2)` used `off_t` (32-bit on some archs); `sendfile64(2)` always uses `off64_t`. Modern glibc maps both to the same syscall on 64-bit. We implement only the 64-bit form (`__x64_sys_sendfile64`).
- `count` is `size_t`; capped to `MAX_RW_COUNT` (0x7FFFF000).
- `offset == NULL` ⟹ advance `in_fd.f_pos`; `offset != NULL` ⟹ do NOT touch `in_fd.f_pos`.
- Internally calls `do_splice_direct` with default flags `SPLICE_F_MOVE | (out_nonblock ? SPLICE_F_NONBLOCK : 0)`.
- Source must implement `file_operations.splice_read`; destination must implement either `file_operations.splice_write` (regular file) or `file_operations.write_iter` (socket via `generic_splice_sendpage` adapter).

## Compatibility contract

REQ-1: fd lookup:
- in_file = fget_light(in_fd, &fput_needed_in); if !in_file: return `-EBADF`.
- out_file = fget_light(out_fd, &fput_needed_out); if !out_file: return `-EBADF`.

REQ-2: read/write capability:
- if !(in_file.f_mode & FMODE_READ): return `-EBADF`.
- if !(out_file.f_mode & FMODE_WRITE): return `-EBADF`.

REQ-3: cross-type validation (the **sendfile fd-cross-type validation** rule):
- if in_file.f_op.splice_read == NULL: return `-EINVAL`.
- if out_file.f_op.splice_write == NULL ∧ out_file.f_op.write_iter == NULL: return `-EINVAL`.
- if in_file is a pipe ∧ offset != NULL: return `-ESPIPE`.

REQ-4: offset handling:
- if offset:
  - copy_from_user(&pos, offset, sizeof(off_t)).
  - if pos < 0: return `-EINVAL`.
  - if !(in_file.f_mode & FMODE_PREAD): return `-ESPIPE`.
- else:
  - pos = in_file.f_pos.

REQ-5: count clamp:
- if (ssize_t)count < 0: return `-EINVAL`.
- count = min(count, MAX_RW_COUNT).
- if pos + count > MAX_LFS_FILESIZE: count = MAX_LFS_FILESIZE - pos.

REQ-6: LSM:
- err = security_file_permission(in_file, MAY_READ); if err: return err.
- err = security_file_permission(out_file, MAY_WRITE); if err: return err.

REQ-7: splice direct:
- flags = SPLICE_F_MOVE.
- if (out_file.f_flags & O_NONBLOCK): flags |= SPLICE_F_NONBLOCK.
- ret = do_splice_direct(in_file, &pos, out_file, &out_pos, count, flags).

REQ-8: offset writeback:
- if offset:
  - copy_to_user(offset, &pos, sizeof(off_t)). // pos advanced by ret
- else:
  - in_file.f_pos = pos.

REQ-9: Audit:
- audit_log(SYS_SENDFILE, in_fd, out_fd, count, ret).

## Acceptance Criteria

- [ ] AC-1: sendfile(socket, regfile, NULL, 1MB): transfers 1MB from regfile's f_pos to socket; f_pos advances.
- [ ] AC-2: sendfile(socket, regfile, &offset=4096, 1MB): transfers starting at offset 4096; offset writeback = 4096 + 1MB; regfile.f_pos unchanged.
- [ ] AC-3: sendfile(regfile-out, regfile-in, NULL, 1MB): success (post-2.6.33; cross-regfile sendfile).
- [ ] AC-4: sendfile(pipe-in as in_fd, socket, NULL, 1KB): -EINVAL (pipes don't `splice_read` to non-pipe?). Actually pipes DO splice_read; check in_fd-is-pipe ∧ offset != NULL is -ESPIPE.
- [ ] AC-5: sendfile(socket-as-in_fd, socket-out, NULL, 1KB): -EINVAL (sockets lack splice_read in default).
- [ ] AC-6: sendfile with offset = -1: -EINVAL.
- [ ] AC-7: sendfile with count = SSIZE_MAX (reinterpreted signed negative): -EINVAL.
- [ ] AC-8: sendfile to non-blocking socket with full sndbuf: -EAGAIN; partial transfer returned if any progress.
- [ ] AC-9: sendfile to closed peer socket: -EPIPE + SIGPIPE.
- [ ] AC-10: sendfile with in_fd lacking FMODE_READ: -EBADF.
- [ ] AC-11: sendfile with out_fd lacking FMODE_WRITE: -EBADF.
- [ ] AC-12: sendfile LSM denial on either fd: -EACCES.
- [ ] AC-13: sendfile with count = 0: returns 0.

## Architecture

```
SysFs::sys_sendfile(out_fd, in_fd, offset, count) -> SyscallResult<isize>
  1. let in_file = FdTable::fget_light(in_fd)?;
  2. let out_file = FdTable::fget_light(out_fd)?;
  3. if !in_file.f_mode.contains(FileMode::READ) { return Err(EBADF); }
  4. if !out_file.f_mode.contains(FileMode::WRITE) { return Err(EBADF); }
  5. if in_file.f_op.splice_read.is_none() { return Err(EINVAL); }
  6. if out_file.f_op.splice_write.is_none() && out_file.f_op.write_iter.is_none() {
        return Err(EINVAL);
     }
  7. let mut pos: i64 = if !offset.is_null() {
        if !in_file.f_mode.contains(FileMode::PREAD) { return Err(ESPIPE); }
        let p = UserCopy::get_user_i64(offset)?;
        if p < 0 { return Err(EINVAL); }
        p
     } else {
        in_file.f_pos.load()
     };
  8. if (count as isize) < 0 { return Err(EINVAL); }
  9. let count = core::cmp::min(count, MAX_RW_COUNT);
 10. Lsm::file_permission(&in_file, MayMode::READ)?;
 11. Lsm::file_permission(&out_file, MayMode::WRITE)?;
 12. let flags = SpliceFlags::MOVE
        | if out_file.f_flags.contains(FileFlags::NONBLOCK) {
              SpliceFlags::NONBLOCK
          } else { SpliceFlags::empty() };
 13. let mut out_pos: i64 = out_file.f_pos.load();
 14. let ret = SpliceCore::do_splice_direct(&in_file, &mut pos, &out_file, &mut out_pos,
                                            count, flags)?;
 15. if !offset.is_null() {
        UserCopy::put_user_i64(pos, offset)?;
     } else {
        in_file.f_pos.store(pos);
     }
 16. Audit::log_sendfile(in_fd, out_fd, count, ret);
 17. Ok(ret)
```

`do_splice_direct(in_file, in_pos, out_file, out_pos, count, flags)`:
1. Allocate ephemeral splice_desc { len = count, total_len = 0, flags, pos = in_pos, ... }.
2. while count > 0:
   - read pages via in_file.f_op.splice_read into ephemeral pipe.
   - drain ephemeral pipe via out_file.f_op.splice_write (or generic_file_splice_write for write_iter targets).
   - update *in_pos and *out_pos by consumed/produced bytes.
   - if non-blocking and out would block: return partial.
3. return total_len.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `sendfile_in_readable` | INVARIANT | per-sys_sendfile: in_file.f_mode has FMODE_READ. |
| `sendfile_out_writable` | INVARIANT | per-sys_sendfile: out_file.f_mode has FMODE_WRITE. |
| `sendfile_in_has_splice_read` | INVARIANT | per-sys_sendfile: in_file.f_op.splice_read != NULL. |
| `sendfile_out_has_splice_write_or_write_iter` | INVARIANT | per-sys_sendfile: out_file supports splice_write OR write_iter. |
| `sendfile_offset_nonneg` | INVARIANT | per-sys_sendfile: pos ≥ 0 pre-splice. |
| `sendfile_count_clamped` | INVARIANT | per-sys_sendfile: count ≤ MAX_RW_COUNT. |
| `sendfile_offset_writeback_advances` | INVARIANT | per-sys_sendfile: offset != NULL ∧ ret > 0 ⟹ *offset_out = pos_in + ret. |
| `sendfile_fpos_writeback_when_no_offset` | INVARIANT | per-sys_sendfile: offset == NULL ∧ ret > 0 ⟹ in_file.f_pos advanced by ret. |

### Layer 2: TLA+

`uapi/syscalls/sendfile.tla`:
- States: validate, lsm, splice-loop, offset-writeback, return.
- Models page-by-page splice through the ephemeral pipe; tracks pos/total_len.
- Properties:
  - `safety_no_pos_skew` — pos advances by exactly the bytes-transferred sum.
  - `safety_offset_or_fpos_exclusive` — offset != NULL ⟹ f_pos unchanged; offset == NULL ⟹ f_pos advanced.
  - `safety_count_clamp` — count never exceeds MAX_RW_COUNT.
  - `safety_partial_transfer_consistent` — ret < count ⟹ ret bytes actually delivered; pos updated accordingly.
  - `liveness_terminates` — sendfile returns within bounded splice iterations.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| pre: in_fd, out_fd both refer to open file refs | `sys_sendfile` |
| pre: in_file has splice_read; out_file has splice_write/write_iter | `sys_sendfile` |
| pre: pos ≥ 0; count ≤ MAX_RW_COUNT | `sys_sendfile` |
| post: ret ≥ 0 ⟹ ret ≤ count | `sys_sendfile` |
| post: offset != NULL ⟹ *offset += ret; in_file.f_pos unchanged | `sys_sendfile` |
| post: offset == NULL ⟹ in_file.f_pos += ret | `sys_sendfile` |
| post: ret == -EPIPE ∧ out_is_socket ⟹ SIGPIPE delivered unless MSG_NOSIGNAL semantics inherited | `sys_sendfile` |

### Layer 4: Verus/Creusot functional

Per-`do_sendfile → do_splice_direct` semantic equivalence with `fs/read_write.c` and `fs/splice.c`. `splice_read` page-extraction and `splice_write` page-consumption interlock matches upstream. Pos advance per-iteration matches `splice_direct_to_actor` exactly.

## Hardening

- **Cross-fd-type validation: in_file MUST support splice_read; out_file MUST support splice_write or write_iter** — defense against per-confused-fd-type (e.g. sendfile from a netlink fd to a regular file). **This is the central sendfile fd-cross-type validation rule.**
- **`offset != NULL` requires `FMODE_PREAD` on in_file** — defense against per-non-seekable read with positional offset.
- **`count` clamped to MAX_RW_COUNT (0x7FFFF000)** — defense against per-overflow on length arithmetic.
- **`(ssize_t)count` signed-check** — defense against per-overflow when caller passes count = SIZE_MAX.
- **`pos ≥ 0` enforced** — defense against per-negative-offset filesystem corruption.
- **`fget_light` + matching `fput_light`** — defense against per-fd-refleak on error.
- **LSM `file_permission(MAY_READ)` on in_file and `MAY_WRITE` on out_file** — defense against per-MAC bypass; both must pass.
- **`f_pos` updated only on offset==NULL path** — defense against per-state-skew when offset is given.
- **Offset writeback uses `copy_to_user`** — defense against per-OOB-write into caller's off_t.
- **Non-blocking propagated via SPLICE_F_NONBLOCK** — defense against per-deadlock on full sndbuf with O_NONBLOCK out_fd.
- **`do_splice_direct` partial-progress accounting** — defense against per-state-skew on partial transfer.

## Grsecurity-PaX

- **PAX_RANDKSTACK** — kstack base randomized; `splice_desc` and `pos` scratch placement varies.
- **PaX UDEREF** — `offset` deref only via `copy_from_user` / `copy_to_user`; direct kernel deref traps.
- **GRKERNSEC_NO_SIMULT_CONNECT** — irrelevant on sendfile (no connect).
- **GRKERNSEC_BLACKHOLE** — irrelevant on sendfile (sock egress already accounted for).
- **GRKERNSEC_RANDNET** — irrelevant on sendfile.
- **CAP_NET_RAW / CAP_NET_ADMIN gates** — sendfile to a `SOCK_RAW` out_fd: hardened policy revalidates `CAP_NET_RAW` per call (defeats post-cap-drop raw-emit).
- **GRKERNSEC_HARDEN_IPC for SOCK_*UNIX** — sendfile to UNIX-domain out_fd is allowed; SCM_RIGHTS cannot be injected via sendfile path (sendfile bypasses cmsg).
- **SOCK_CLOEXEC mandatory under suid** — irrelevant on sendfile (no fd creation).
- **SCM_RIGHTS recursion bound** — irrelevant on sendfile (no scm).
- **sendmmsg compound-bound limit** — irrelevant.
- **splice fd-permission boundary** — **the central grsec rule for sendfile**: in_fd MUST have read permission as enforced by `security_file_permission(in_file, MAY_READ)`; out_fd MUST have write permission via `MAY_WRITE`. Hardened build additionally forbids sendfile from `/proc/self/mem` / `/proc/*/mem` / `kcore` / sysfs sensitive files to any out_fd whose target is **outside the caller's task / namespace** (defeats classic memory-introspection-exfiltration via sendfile).
- **`grsec.deny_sendfile_to_socket_unprivileged`** — optional hardened policy: unprivileged sendfile from regfile to **external** socket (non-loopback) requires `CAP_NET_ADMIN`; deters file-exfil DoS.
- **getsockopt info-leak prevention** — sendfile does not read sockopts; `pos` writeback path is bounded; no info leak.

## Open Questions

- For sendfile from `tmpfs` to a UNIX-domain socket with `SCM_RIGHTS` already in flight: ordering vs. cmsg is undefined upstream. We document the sendfile bypasses cmsg entirely; SCM is a sendmsg-only concept.
- Should sendfile to a pipe out_fd (very common pattern) be optimized through the zero-copy pipe path? Upstream: yes; already covered by `do_splice_direct`.

## Out of Scope

- `splice(2)` — separate Tier-5 (more general fd-to-fd pipe variant).
- `tee(2)`, `vmsplice(2)` — separate Tier-5s.
- `copy_file_range(2)` — separate Tier-5 (modern cp accelerator).
- Per-fs `splice_read` implementations — Tier-3 per filesystem.
- `do_splice_direct` internals — Tier-3 `fs/splice.md`.
- Implementation code.
