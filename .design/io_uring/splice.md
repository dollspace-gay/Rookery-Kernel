# Tier-3: io_uring/splice.c — io_uring splice and tee operations

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: io_uring/00-overview.md
upstream-paths:
  - io_uring/splice.c (~149 lines)
  - io_uring/splice.h
  - io_uring/filetable.h (io_rsrc_node, io_rsrc_node_lookup, io_slot_file, io_put_rsrc_node)
  - include/linux/splice.h (do_splice, do_tee, SPLICE_F_*)
  - include/uapi/linux/io_uring.h (IORING_OP_SPLICE, IORING_OP_TEE)
-->

## Summary

`io_uring/splice.c` implements the async wrappers for the zero-copy pipe-mediated data-transfer family as io_uring SQE opcodes: `IORING_OP_SPLICE` (move bytes between file and pipe, or between two pipes, optionally with explicit offsets) and `IORING_OP_TEE` (duplicate bytes between two pipes without consuming the source). Both opcodes share a single command payload, `struct io_splice`, holding `file_out` (resolved by the io_uring core from `sqe->fd`), `off_out`, `off_in`, `len`, `splice_fd_in` (the *other* fd, raw or registered), `flags` (the `splice(2)` SPLICE_F_* bitmask plus the io_uring-private `SPLICE_F_FD_IN_FIXED`), and `rsrc_node` (per-registered-fd refcount handle when fixed-file lookup occurred). The two-phase split is: a shared `__io_splice_prep` reads len/flags/splice_fd_in/rsrc_node, plus per-op offset reads (`io_splice_prep` reads off_in/off_out, `io_tee_prep` rejects nonzero offsets); both mark `REQ_F_FORCE_ASYNC`. At issue, `io_splice_get_file` resolves the *second* fd via either `io_file_get_normal` (raw fd path) or the registered-file table (`io_rsrc_node_lookup` under submit lock, refs++, `io_slot_file`, with REQ_F_NEED_CLEANUP set). The actual zero-copy call is `do_splice(in, poff_in, out, poff_out, len, flags)` or `do_tee(in, out, len, flags)`. After the call, the raw-fd path calls `fput(in)`; the fixed-file path defers release to `io_splice_cleanup` which calls `io_put_rsrc_node`. The op explicitly does not allow IOSQE_BUFFER_SELECT (no provided-buffer semantics — splice operates fd-to-fd). Critical for: zero-copy stream forwarding, proxy/relay servers, `tee(2)`-based fanout, and high-throughput pipeline patterns without per-byte memcpy.

This Tier-3 covers `io_uring/splice.c` (~149 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct io_splice` | per-SPLICE / TEE cmd payload | `IoSplice` |
| `__io_splice_prep()` | per-shared prep core | `IoSplice::prep_common` |
| `io_splice_prep()` | per-SPLICE prep wrapper | `IoSplice::splice_prep` |
| `io_tee_prep()` | per-TEE prep wrapper | `IoSplice::tee_prep` |
| `io_splice_get_file()` | per-issue: resolve fd_in (raw or fixed) | `IoSplice::get_in_file` |
| `io_splice()` | per-SPLICE issue: do_splice | `IoSplice::splice` |
| `io_tee()` | per-TEE issue: do_tee | `IoSplice::tee` |
| `io_splice_cleanup()` | per-fixed-file release | `IoSplice::cleanup` |
| `do_splice()` | per-VFS zero-copy fd↔pipe | extern from `vfs/splice.md` |
| `do_tee()` | per-VFS pipe→pipe duplication | extern from `vfs/splice.md` |
| `io_file_get_normal()` | per-raw-fd lookup (fget) | extern from `io_uring/io_uring-core.md` |
| `io_rsrc_node_lookup()` | per-fixed-file slot lookup | extern from `io_uring/rsrc.md` |
| `io_slot_file()` | per-rsrc_node → struct file extractor | extern from `io_uring/rsrc.md` |
| `io_put_rsrc_node()` | per-rsrc_node release | extern from `io_uring/rsrc.md` |
| `io_ring_submit_lock()` / `_unlock()` | per-ctx submit-side lock | shared core |
| `SPLICE_F_FD_IN_FIXED` | per-flag: fd_in is a registered slot | shared UAPI (io_uring-private bit) |
| `SPLICE_F_ALL` | per-flag mask: all valid splice(2) flags | shared UAPI |
| `SPLICE_F_MOVE` / `_NONBLOCK` / `_MORE` / `_GIFT` | per-`splice(2)` flags | shared UAPI |
| `REQ_F_FORCE_ASYNC` | per-req: forces io-wq dispatch | shared req-flag |
| `REQ_F_NEED_CLEANUP` | per-req: cleanup callback required | shared req-flag |
| `IO_URING_F_NONBLOCK` | per-issue flag (must be cleared) | shared issue-flag |
| `IOU_COMPLETE` | per-issue: completion sentinel | shared issue-ret |
| `io_req_set_res()` / `req_set_fail()` | per-req: stamp result / mark failure | shared req helper |
| `fput()` | per-raw-fd release | shared file refcount |

## Compatibility contract

REQ-1: `struct io_splice` — per-op cmd payload, inline in io_kiocb cmd area:
- `file_out: *struct file` — per-output file (resolved by io_uring core from `sqe->fd`).
- `off_out: loff_t` — per-output-offset (-1 ⟹ NULL, follow current file pos).
- `off_in:  loff_t` — per-input-offset (-1 ⟹ NULL, follow current file pos).
- `len:     u64`   — per-transfer length.
- `splice_fd_in: i32` — per-second-fd: raw fd or registered-slot index.
- `flags:   u32`   — per-op flags: `SPLICE_F_FD_IN_FIXED | SPLICE_F_MOVE | SPLICE_F_NONBLOCK | SPLICE_F_MORE | SPLICE_F_GIFT`.
- `rsrc_node: *io_rsrc_node` — per-fixed-file refcount handle (NULL for raw-fd path).

REQ-2: `__io_splice_prep(req, sqe)` — shared prep core:
- `valid_flags = SPLICE_F_FD_IN_FIXED | SPLICE_F_ALL`.
- `sp->len = READ_ONCE(sqe->len)`.
- `sp->flags = READ_ONCE(sqe->splice_flags)`.
- /* Reject any bit outside the valid mask */
- if `sp->flags & ~valid_flags`: return -EINVAL.
- `sp->splice_fd_in = READ_ONCE(sqe->splice_fd_in)`.
- `sp->rsrc_node = NULL`.
- /* Always blocks; force worker dispatch */
- `req->flags |= REQ_F_FORCE_ASYNC`.
- return 0.

REQ-3: `io_tee_prep(req, sqe)` — per-TEE prep:
- /* TEE has no offset semantics (pipe-to-pipe only) */
- if `READ_ONCE(sqe->splice_off_in)` ∨ `READ_ONCE(sqe->off)`: return -EINVAL.
- return __io_splice_prep(req, sqe).

REQ-4: `io_splice_prep(req, sqe)` — per-SPLICE prep:
- `sp->off_in  = READ_ONCE(sqe->splice_off_in)`.
- `sp->off_out = READ_ONCE(sqe->off)`.
- return __io_splice_prep(req, sqe).

REQ-5: `io_splice_get_file(req, issue_flags) -> *file` — per-input-file resolver:
- /* Raw-fd path */
- if `!(sp->flags & SPLICE_F_FD_IN_FIXED)`:
  - return `io_file_get_normal(req, sp->splice_fd_in)`.
- /* Fixed-file path */
- `io_ring_submit_lock(ctx, issue_flags)`.
- `node = io_rsrc_node_lookup(&ctx->file_table.data, sp->splice_fd_in)`.
- if node:
  - `node->refs++`.
  - `sp->rsrc_node = node`.
  - `file = io_slot_file(node)`.
  - `req->flags |= REQ_F_NEED_CLEANUP`.
- `io_ring_submit_unlock(ctx, issue_flags)`.
- return file.

REQ-6: `io_tee(req, issue_flags)` — per-TEE issue:
- `WARN_ON_ONCE(issue_flags & IO_URING_F_NONBLOCK)`.
- `out = sp->file_out`.
- `flags = sp->flags & ~SPLICE_F_FD_IN_FIXED`.   /* strip io_uring-private bit */
- `in = io_splice_get_file(req, issue_flags)`.
- if `!in`:
  - `ret = -EBADF`.
  - goto done.
- if `sp->len`: `ret = do_tee(in, out, sp->len, flags)`.
- else `ret = 0`.
- /* Raw-fd path: release input file ref */
- if `!(sp->flags & SPLICE_F_FD_IN_FIXED)`: `fput(in)`.
- done:
- if `ret != sp->len`: `req_set_fail(req)`.
- `io_req_set_res(req, ret, 0)`.
- return `IOU_COMPLETE`.

REQ-7: `io_splice(req, issue_flags)` — per-SPLICE issue:
- `WARN_ON_ONCE(issue_flags & IO_URING_F_NONBLOCK)`.
- `out = sp->file_out`.
- `flags = sp->flags & ~SPLICE_F_FD_IN_FIXED`.
- `in = io_splice_get_file(req, issue_flags)`.
- if `!in`:
  - `ret = -EBADF`.
  - goto done.
- /* Offset -1 ⟹ NULL pointer (follow file position) */
- `poff_in  = (sp->off_in  == -1) ? NULL : &sp->off_in`.
- `poff_out = (sp->off_out == -1) ? NULL : &sp->off_out`.
- if `sp->len`: `ret = do_splice(in, poff_in, out, poff_out, sp->len, flags)`.
- else `ret = 0`.
- if `!(sp->flags & SPLICE_F_FD_IN_FIXED)`: `fput(in)`.
- done:
- if `ret != sp->len`: `req_set_fail(req)`.
- `io_req_set_res(req, ret, 0)`.
- return `IOU_COMPLETE`.

REQ-8: `io_splice_cleanup(req)` — per-cleanup:
- if `sp->rsrc_node`: `io_put_rsrc_node(req->ctx, sp->rsrc_node)`.

REQ-9: Per-IOSQE_BUFFER_SELECT incompatibility:
- Splice / tee operate fd-to-fd; no provided-buffer semantics apply.
- The opdef for IORING_OP_SPLICE / TEE does *not* set the `buffer_select` flag, so the io_uring core rejects an SQE with IOSQE_BUFFER_SELECT before reaching this file.
- (Confirmed by absence of `.buffer_select = 1` in `opdef.c` rows for these ops.)

REQ-10: `SPLICE_F_FD_IN_FIXED` is the io_uring-private repurpose of an otherwise-undefined bit in `sqe->splice_flags`:
- When set, `splice_fd_in` is interpreted as a registered-file slot index.
- When clear, it is a raw fd consumed via `io_file_get_normal` (fget-equivalent).
- The bit is *stripped* before passing flags to `do_splice` / `do_tee`, since the VFS does not recognize it.

REQ-11: `SPLICE_F_ALL` is the union of all UAPI-public splice flags: `SPLICE_F_MOVE | SPLICE_F_NONBLOCK | SPLICE_F_MORE | SPLICE_F_GIFT`. Any bit outside `SPLICE_F_ALL | SPLICE_F_FD_IN_FIXED` is rejected at prep with -EINVAL.

REQ-12: `sp->len == 0` short-circuit:
- Both issue paths skip the `do_splice` / `do_tee` call when len is zero.
- Result is 0 (no bytes transferred); `ret != sp->len` ⟹ 0 != 0 is false, so no req_set_fail.

REQ-13: Per-short-transfer flagging:
- If `do_splice` / `do_tee` returns less than `sp->len` (partial transfer, EAGAIN, EOF, signal), `req_set_fail(req)` is called.
- The CQE still reports the actual byte count in `res`, but the request is marked as failed for the linked-request chain.

REQ-14: Per-output-file resolution:
- `sp->file_out` is the io_uring-core-resolved file for `sqe->fd`. The standard SQE flags (`IOSQE_FIXED_FILE` for fd_out) apply here and are honored by the core; this file is *not* tied to `SPLICE_F_FD_IN_FIXED`.
- `SPLICE_F_FD_IN_FIXED` controls *only* the second fd (`sqe->splice_fd_in`).

REQ-15: Per-locking for fixed-file lookup:
- `io_ring_submit_lock` is held around `io_rsrc_node_lookup` / refs++ to prevent a concurrent unregister from freeing the node mid-lookup.
- The submit_lock is per-ctx; issue_flags carries whether the caller already holds it.

REQ-16: Per-cleanup invariant:
- `rsrc_node` is non-NULL iff the fixed-file path was taken successfully.
- If `io_rsrc_node_lookup` returned NULL (invalid slot), no node ref is held; cleanup is a no-op.
- The raw-fd path consumes its `fput` synchronously in the issue handler, so cleanup is also a no-op there.

REQ-17: Per-completion semantics:
- Return is the bytes-transferred count from `do_splice` / `do_tee` (>=0), or a negative errno.
- `io_req_set_res(req, ret, 0)` stamps `res = ret, cflags = 0`.

## Acceptance Criteria

- [ ] AC-1: IORING_OP_SPLICE: do_splice called with (in, &off_in?NULL, out, &off_out?NULL, len, flags & ~SPLICE_F_FD_IN_FIXED).
- [ ] AC-2: IORING_OP_TEE: do_tee called with (in, out, len, flags & ~SPLICE_F_FD_IN_FIXED).
- [ ] AC-3: IORING_OP_TEE with nonzero sqe->splice_off_in or sqe->off: -EINVAL at prep.
- [ ] AC-4: Prep with flags outside SPLICE_F_ALL | SPLICE_F_FD_IN_FIXED: -EINVAL.
- [ ] AC-5: off_in == -1: do_splice receives NULL for poff_in.
- [ ] AC-6: off_out == -1: do_splice receives NULL for poff_out.
- [ ] AC-7: SPLICE_F_FD_IN_FIXED set: io_rsrc_node_lookup under submit_lock; rsrc_node->refs++; REQ_F_NEED_CLEANUP set.
- [ ] AC-8: SPLICE_F_FD_IN_FIXED clear: io_file_get_normal called; fput(in) at end of issue; no cleanup needed.
- [ ] AC-9: Invalid splice_fd_in (lookup returns NULL): CQE res = -EBADF.
- [ ] AC-10: ret < sp->len: req_set_fail(req) called.
- [ ] AC-11: sp->len == 0: do_splice / do_tee not called; ret = 0.
- [ ] AC-12: Issue with IO_URING_F_NONBLOCK: WARN_ON_ONCE fires.
- [ ] AC-13: io_splice_cleanup: io_put_rsrc_node only if rsrc_node != NULL.
- [ ] AC-14: IOSQE_BUFFER_SELECT on SPLICE / TEE SQE: rejected by core (opdef lacks buffer_select).
- [ ] AC-15: SPLICE_F_FD_IN_FIXED bit stripped from flags passed to do_splice / do_tee.

## Architecture

```
struct IoSplice {
  file_out: *File,        // resolved by io_uring core from sqe->fd
  off_out:  i64,          // loff_t, -1 sentinel ⟹ NULL ptr to VFS
  off_in:   i64,
  len:      u64,
  splice_fd_in: i32,      // raw fd or registered slot index
  flags:    u32,          // SPLICE_F_FD_IN_FIXED | SPLICE_F_ALL bits
  rsrc_node: Option<*IoRsrcNode>,
}
```

`IoSplice::prep_common(req, sqe) -> Result<(), Errno>`:
1. let valid_flags = SPLICE_F_FD_IN_FIXED | SPLICE_F_ALL.
2. sp.len = read_once(&sqe.len).
3. sp.flags = read_once(&sqe.splice_flags).
4. if sp.flags & !valid_flags != 0: return Err(EINVAL).
5. sp.splice_fd_in = read_once(&sqe.splice_fd_in).
6. sp.rsrc_node = None.
7. req.flags |= REQ_F_FORCE_ASYNC.
8. Ok(()).

`IoSplice::tee_prep(req, sqe) -> Result<(), Errno>`:
1. if read_once(&sqe.splice_off_in) != 0 || read_once(&sqe.off) != 0: return Err(EINVAL).
2. IoSplice::prep_common(req, sqe).

`IoSplice::splice_prep(req, sqe) -> Result<(), Errno>`:
1. sp.off_in  = read_once(&sqe.splice_off_in) as i64.
2. sp.off_out = read_once(&sqe.off) as i64.
3. IoSplice::prep_common(req, sqe).

`IoSplice::get_in_file(req, issue_flags) -> Option<*File>`:
1. let ctx = req.ctx.
2. if !(sp.flags & SPLICE_F_FD_IN_FIXED):
   - return io_file_get_normal(req, sp.splice_fd_in).
3. io_ring_submit_lock(ctx, issue_flags).
4. let node = io_rsrc_node_lookup(&ctx.file_table.data, sp.splice_fd_in).
5. let file = if let Some(n) = node {
   -   n.refs += 1.
   -   sp.rsrc_node = Some(n).
   -   req.flags |= REQ_F_NEED_CLEANUP.
   -   Some(io_slot_file(n))
   - } else { None }.
6. io_ring_submit_unlock(ctx, issue_flags).
7. file.

`IoSplice::tee(req, issue_flags) -> IouRet`:
1. warn_on_once(issue_flags & IO_URING_F_NONBLOCK).
2. let out = sp.file_out.
3. let flags = sp.flags & !SPLICE_F_FD_IN_FIXED.   /* strip private bit */
4. let in = match IoSplice::get_in_file(req, issue_flags) {
   -   Some(f) => f,
   -   None => { ret = -EBADF; goto done; }
   - }.
5. let ret = if sp.len > 0 { vfs::do_tee(in, out, sp.len, flags) } else { 0 }.
6. if !(sp.flags & SPLICE_F_FD_IN_FIXED): fput(in).   /* raw-fd path only */
7. done:
8. if ret as u64 != sp.len: req_set_fail(req).
9. io_req_set_res(req, ret, 0).
10. IouRet::Complete.

`IoSplice::splice(req, issue_flags) -> IouRet`:
1. warn_on_once(issue_flags & IO_URING_F_NONBLOCK).
2. let out = sp.file_out.
3. let flags = sp.flags & !SPLICE_F_FD_IN_FIXED.
4. let in = match IoSplice::get_in_file(req, issue_flags) {
   -   Some(f) => f, None => { ret = -EBADF; goto done; }
   - }.
5. let poff_in  = if sp.off_in  == -1 { None } else { Some(&mut sp.off_in)  }.
6. let poff_out = if sp.off_out == -1 { None } else { Some(&mut sp.off_out) }.
7. let ret = if sp.len > 0 {
   -   vfs::do_splice(in, poff_in, out, poff_out, sp.len, flags)
   - } else { 0 }.
8. if !(sp.flags & SPLICE_F_FD_IN_FIXED): fput(in).
9. done:
10. if ret as u64 != sp.len: req_set_fail(req).
11. io_req_set_res(req, ret, 0).
12. IouRet::Complete.

`IoSplice::cleanup(req)`:
1. if let Some(node) = sp.rsrc_node: io_put_rsrc_node(req.ctx, node).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `prep_rejects_unknown_flags` | INVARIANT | per-prep_common: any bit outside SPLICE_F_FD_IN_FIXED|SPLICE_F_ALL ⟹ -EINVAL. |
| `tee_prep_rejects_offsets` | INVARIANT | per-tee_prep: nonzero sqe.splice_off_in or sqe.off ⟹ -EINVAL. |
| `prep_sets_force_async` | INVARIANT | per-prep_common: REQ_F_FORCE_ASYNC set on success. |
| `issue_in_blocking_context` | INVARIANT | per-tee/splice issue: IO_URING_F_NONBLOCK never observed. |
| `fixed_file_path_takes_submit_lock` | INVARIANT | per-get_in_file: lookup/refs++ inside submit_lock critical section. |
| `fixed_file_path_sets_cleanup` | INVARIANT | per-get_in_file: success on fixed path ⟹ REQ_F_NEED_CLEANUP set. |
| `raw_fd_path_fputs` | INVARIANT | per-tee/splice issue: raw path ⟹ fput(in) before return. |
| `private_bit_stripped_to_vfs` | INVARIANT | per-issue: SPLICE_F_FD_IN_FIXED cleared in flags passed to do_splice/do_tee. |
| `off_neg1_means_null` | INVARIANT | per-splice issue: off_in == -1 ⟹ poff_in = NULL. |
| `short_xfer_marks_fail` | INVARIANT | per-issue: ret != sp.len ⟹ req_set_fail. |
| `len_zero_no_call` | INVARIANT | per-issue: sp.len == 0 ⟹ do_splice/do_tee not invoked. |

### Layer 2: TLA+

`io_uring/splice.tla`:
- Per-prep + per-issue + per-cleanup states for {SPLICE, TEE} × {raw-fd, fixed-fd}.
- Properties:
  - `safety_no_buffer_select` — per-opdef: IORING_OP_SPLICE/TEE never have `buffer_select` set.
  - `safety_fixed_refcount_balanced` — per-fixed path: refs++ in get_in_file always matched by io_put_rsrc_node in cleanup.
  - `safety_raw_fput_paired` — per-raw path: io_file_get_normal always matched by fput on the same issue.
  - `safety_private_flag_never_to_vfs` — per-issue: do_splice/do_tee called with flags & ~SPLICE_F_FD_IN_FIXED.
  - `safety_no_nonblock_issue` — per-issue: IO_URING_F_NONBLOCK ⟹ WARN.
  - `liveness_each_issue_completes` — per-issue: terminates with IOU_COMPLETE.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `IoSplice::prep_common` post: sp.flags ⊆ (SPLICE_F_FD_IN_FIXED \| SPLICE_F_ALL) | `IoSplice::prep_common` |
| `IoSplice::tee_prep` post: sp.off_in untouched (TEE has no offsets) | `IoSplice::tee_prep` |
| `IoSplice::get_in_file` post fixed: rsrc_node = Some(n) ⟺ REQ_F_NEED_CLEANUP set | `IoSplice::get_in_file` |
| `IoSplice::splice` post: poff_in = NULL ⟺ off_in == -1 | `IoSplice::splice` |
| `IoSplice::tee` post raw: fput(in) called exactly once | `IoSplice::tee` |
| `IoSplice::cleanup` post: rsrc_node released iff non-NULL | `IoSplice::cleanup` |

### Layer 4: Verus/Creusot functional

`Per-SQE submit → prep validates flag mask + per-op offset rule → REQ_F_FORCE_ASYNC ⟹ worker dispatch → io_splice_get_file (raw fget or fixed-slot lookup under submit_lock) → do_splice / do_tee call → req_set_fail on short xfer → io_req_set_res ⟹ CQE → cleanup releases rsrc_node` semantic equivalence: per-`splice(2)` and `tee(2)` man pages, per-`io_uring_enter(2)` for IORING_OP_SPLICE / TEE.

## Hardening

(Inherits row-1 features from `io_uring/00-overview.md` § Hardening.)

Splice/Tee reinforcement:

- **Per-flag mask strict (SPLICE_F_FD_IN_FIXED | SPLICE_F_ALL only)** — defense against per-forward-flag UAPI drift.
- **Per-TEE rejects offsets** — defense against per-UAPI confusion (TEE is pipe-only, no seekable offsets).
- **Per-SPLICE_F_FD_IN_FIXED bit stripped before VFS call** — defense against per-VFS rejection of unknown flag bits.
- **Per-fixed-file lookup under io_ring_submit_lock** — defense against per-concurrent-unregister UAF.
- **Per-rsrc_node refs++ paired with io_put_rsrc_node** — defense against per-refcount leak / underflow.
- **Per-raw-fd path fput synchronous in issue** — defense against per-deferred-release fd leak.
- **Per-REQ_F_NEED_CLEANUP gated on rsrc_node != NULL** — defense against per-spurious cleanup.
- **Per-IOSQE_BUFFER_SELECT not in opdef** — defense against per-provided-buffer-on-fd-op semantic violation.
- **Per-REQ_F_FORCE_ASYNC enforced** — defense against per-blocking-in-submit-path stall.
- **Per-WARN_ON_ONCE on IO_URING_F_NONBLOCK** — defense against per-misdispatch (splice in submit thread).
- **Per-short-transfer marked req_set_fail** — defense against per-linked-chain silently continuing on partial xfer.
- **Per-len == 0 short-circuit** — defense against per-VFS edge-case zero-len divergence.
- **Per-off == -1 sentinel ⟹ NULL** — defense against per-loff_t -1 reaching VFS as a real offset.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — splice SQE fields are read through the user-mapped SQ, which is mmap'd; the `sqe->splice_off_in`, `splice_fd_in`, `len`, and `off_in/_out` reads must be one-shot snapshots through SMAP-enforced accessors, never re-read post-validation.
- **PAX_KERNEXEC** — splice op table is `__ro_after_init`; no runtime patching of `do_splice` dispatch.
- **PAX_RANDKSTACK** — `io_splice` runs under full stack randomization; no stack-pointer caching across the pipe-side blocking point.
- **PAX_REFCOUNT** — both `file_in` and `file_out` use the hardened `fget`/`fput` refcount path; on `SPLICE_F_FD_IN_FIXED` the registered-files node refcount is the saturating wrapper.
- **PAX_MEMORY_SANITIZE** — splice does not own a kmalloc'd per-op buffer (pipe pages are the medium); on cancellation the request slab is zero-poisoned before kmem_cache_free.
- **PAX_UDEREF** — SQE field reads on the user-mapped SQ pass through SMAP gates; pipe-side `iov_iter` setup uses the hardened accessor variants.
- **PAX_RAP / kCFI** — `file->f_op->splice_read` / `splice_write` indirect calls carry kCFI type tags; mismatched f_op (e.g., character-device without splice support) traps rather than silently zero-returning.
- **GRKERNSEC_HIDESYM** — splice failure paths must not emit `file_in`/`file_out` kernel pointers into audit.
- **GRKERNSEC_DMESG** — EOPNOTSUPP/EINVAL diagnostics on the splice fast path are ratelimited; bulk-splice failure storms cannot drown out dmesg.
- **fd permission boundary** — both endpoints honour FMODE_READ/FMODE_WRITE; the pipe-side validation refuses non-pipe-on-pipe-side (one end must be a pipe in vanilla splice); grsec adds a refuse-cross-mount-namespace check when both endpoints are real files and CAP_SYS_ADMIN is absent.
- **Pipe-side validation** — splice into a pipe whose `i_pipe->files == 0` after read-side close traps instead of proceeding; defense against TOCTOU pipe-reader exit.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `do_splice` / `do_tee` VFS implementation (covered in `vfs/splice.md` Tier-3)
- `io_file_get_normal` / `io_rsrc_node_lookup` / `io_slot_file` / `io_put_rsrc_node` (covered in `io_uring/rsrc.md` and `io_uring/io_uring-core.md` Tier-3)
- io_uring core SQE→io_kiocb dispatch and IOSQE_FIXED_FILE handling for fd_out (covered in `io_uring/io_uring-core.md` Tier-3)
- Pipe internals (covered in `fs/pipe.md` Tier-3)
- `sendfile(2)` (separate path, not an io_uring op)
- Implementation code
