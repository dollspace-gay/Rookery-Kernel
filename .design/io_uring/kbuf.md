# Tier-3: io_uring/kbuf.c — Provided buffers and ring-mapped buffer pools

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: io_uring/00-overview.md
upstream-paths:
  - io_uring/kbuf.c (~762 lines)
  - io_uring/kbuf.h
  - include/uapi/linux/io_uring.h (struct io_uring_buf, io_uring_buf_ring, io_uring_buf_reg, io_uring_buf_status, IOU_PBUF_RING_*)
-->

## Summary

io_uring **provided buffers** let an application register a pool of receive/read buffers in advance, identified by a **BGID** (buffer group id, 16-bit), and have the kernel pick a buffer at execution time when the SQE carries `IOSQE_BUFFER_SELECT`. Two flavors share the BGID namespace: the **legacy** path (`IORING_OP_PROVIDE_BUFFERS` / `IORING_OP_REMOVE_BUFFERS`) maintains a `struct list_head` of `struct io_buffer` entries; the **ring-mapped** path (`IORING_REGISTER_PBUF_RING` / `_UNREGISTER_PBUF_RING`) maintains a single-producer / single-consumer `struct io_uring_buf_ring` shared memory ring whose **tail** is advanced by userspace as it (re)posts buffers and whose **head** is advanced by the kernel as it consumes them. The on-completion CQE returns the chosen **BID** (buffer id, 16-bit) in `cqe->flags >> IORING_CQE_BUFFER_SHIFT` with `IORING_CQE_F_BUFFER` set. Each `struct io_buffer_list` is indexed in `ctx->io_bl_xa` (XArray keyed by BGID) and may have the `IOBL_BUF_RING` flag (ring-mode) and optionally `IOBL_INC` flag (incremental consumption — one underlying ring buffer can satisfy multiple short reads until depleted). The kernel-allocated `IOU_PBUF_RING_MMAP` flavor lets userspace `mmap(2)` the ring back at offset `IORING_OFF_PBUF_RING | (bgid << IORING_OFF_PBUF_SHIFT)`. Critical for: zero-copy-friendly recv/read fast path, recv-multishot, scaling to thousands of in-flight sockets without per-request user-buffer pinning, and the `IOU_PBUF_RING_INC` "donate a big slab" idiom.

This Tier-3 covers `io_uring/kbuf.c` (~762 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct io_buffer_list` | per-BGID descriptor (list or ring) | `BufferList` |
| `struct io_buffer` | per-legacy-buffer node | `LegacyBuffer` |
| `struct io_uring_buf` | per-ring-slot UAPI | `RingBuf` |
| `struct io_uring_buf_ring` | per-ring header + flex slots | `RingBufRing` |
| `struct io_uring_buf_reg` | per-register UAPI arg | `BufReg` |
| `struct io_uring_buf_status` | per-status UAPI arg | `BufStatus` |
| `struct io_br_sel` | per-select result (buf_list + addr/val union) | `BrSel` |
| `struct buf_sel_arg` | per-multi-buffer-peek arg | `BufSelArg` |
| `struct io_provide_buf` | per-OP_PROVIDE/REMOVE_BUFFERS request cmd | `ProvideBufCmd` |
| `io_buffer_select()` | per-recv/read hot path single-buf select | `Kbuf::buffer_select` |
| `io_buffers_select()` | per-multi-buf select (committing) | `Kbuf::buffers_select` |
| `io_buffers_peek()` | per-multi-buf peek (no commit) | `Kbuf::buffers_peek` |
| `io_ring_buffer_select()` | per-ring single-buf inner | `Kbuf::ring_buffer_select` |
| `io_ring_buffers_peek()` | per-ring multi-buf inner | `Kbuf::ring_buffers_peek` |
| `io_provided_buffer_select()` | per-legacy single-buf inner | `Kbuf::legacy_buffer_select` |
| `io_provided_buffers_select()` | per-legacy multi-buf inner | `Kbuf::legacy_buffers_select` |
| `io_kbuf_commit()` | per-ring head advance | `Kbuf::commit` |
| `io_kbuf_inc_commit()` | per-INC partial head advance | `Kbuf::inc_commit` |
| `__io_put_kbufs()` | per-CQE `IORING_CQE_F_BUFFER`/`F_BUF_MORE` flag construction | `Kbuf::put_kbufs` |
| `io_kbuf_drop_legacy()` | per-legacy drop on cancel | `Kbuf::drop_legacy` |
| `io_kbuf_recycle_legacy()` | per-legacy recycle to head of list | `Kbuf::recycle_legacy` |
| `io_kbuf_recycle_ring()` | per-ring recycle (clear flags) | `Kbuf::recycle_ring` |
| `io_register_pbuf_ring()` | per-IORING_REGISTER_PBUF_RING | `Kbuf::register_ring` |
| `io_unregister_pbuf_ring()` | per-IORING_UNREGISTER_PBUF_RING | `Kbuf::unregister_ring` |
| `io_register_pbuf_status()` | per-IORING_REGISTER_PBUF_STATUS | `Kbuf::register_status` |
| `io_provide_buffers_prep()` | per-OP_PROVIDE_BUFFERS SQE parse | `Kbuf::provide_prep` |
| `io_remove_buffers_prep()` | per-OP_REMOVE_BUFFERS SQE parse | `Kbuf::remove_prep` |
| `io_manage_buffers_legacy()` | per-OP_PROVIDE/REMOVE_BUFFERS issue | `Kbuf::manage_legacy` |
| `io_add_buffers()` | per-legacy add-N walk | `Kbuf::add_legacy` |
| `io_remove_buffers_legacy()` | per-legacy remove-N walk | `Kbuf::remove_legacy` |
| `io_buffer_get_list()` | per-XArray lookup by BGID | `Kbuf::get_list` |
| `io_buffer_add_list()` | per-XArray insert | `Kbuf::add_list` |
| `io_put_bl()` | per-BufferList tear-down | `Kbuf::put_list` |
| `io_destroy_bl()` | per-erase-then-put | `Kbuf::destroy_list` |
| `io_destroy_buffers()` | per-ctx full sweep on exit | `Kbuf::destroy_all` |
| `io_pbuf_get_region()` | per-mmap region lookup | `Kbuf::get_region` |
| `MAX_BIDS_PER_BGID` | per-BGID cap = 1 << 16 | constant |
| `IOBL_BUF_RING` / `IOBL_INC` | per-`bl->flags` | flags |
| `IOU_PBUF_RING_MMAP` / `IOU_PBUF_RING_INC` | per-reg UAPI flags | flags |
| `IORING_CQE_F_BUFFER` / `_F_BUF_MORE` | per-CQE delivery flags | flags |
| `IORING_CQE_BUFFER_SHIFT` | per-CQE BID shift | constant |
| `IORING_OFF_PBUF_RING` / `IORING_OFF_PBUF_SHIFT` | per-mmap offset encoding | constants |
| `PEEK_MAX_IMPORT` (= 256) | per-peek iov cap | constant |

## Compatibility contract

REQ-1: struct io_buffer_list:
- Either `buf_list` (legacy `struct list_head` of `io_buffer`) OR `buf_ring` (shared `io_uring_buf_ring*`), discriminated by `flags & IOBL_BUF_RING`.
- `nbufs`: per-legacy current count.
- `bgid`: 16-bit buffer group id (also XArray key).
- `head`: per-ring consumer index (kernel-private; mod-`mask`+1 size).
- `mask`: `ring_entries - 1` (must be power-of-2).
- `flags`: `IOBL_BUF_RING` | `IOBL_INC`.
- `min_left_sub_one`: per-`IOBL_INC` minimum-left threshold minus one.
- `region`: `struct io_mapped_region` backing the ring (user-pinned or kernel-allocated).
- Stored in `ctx->io_bl_xa` (XArray) under `ctx->uring_lock` for writers; readers from mmap path use `ctx->mmap_lock`.

REQ-2: struct io_buffer (legacy node):
- `list`: per-`bl->buf_list` linkage.
- `addr`: user virtual address.
- `len`: byte length (capped to `MAX_RW_COUNT`).
- `bid`: 16-bit BID.
- `bgid`: 16-bit BGID (for sanity).

REQ-3: struct io_uring_buf (UAPI ring slot):
- `addr`: user virtual address of buffer.
- `len`: usable length.
- `bid`: 16-bit BID.
- `resv`: must be zero on add; aliased with `tail` field of the ring header in slot 0 (see REQ-4).

REQ-4: struct io_uring_buf_ring (UAPI ring header):
- Slot 0 is overlaid as the header: 14 bytes reserved, then `__u16 tail`.
- Slots 1..N (where N+1 = ring_entries) carry the `io_uring_buf` entries.
- Producer (userspace) writes tail with `smp_store_release` after populating slots.
- Consumer (kernel) reads tail with `smp_load_acquire`, advances private `bl->head`.

REQ-5: io_buffer_select(req, *len, buf_group, issue_flags) → io_br_sel:
- Acquire `ctx->uring_lock` (via `io_ring_submit_lock`).
- bl = `xa_load(io_bl_xa, buf_group)`.
- if bl is NULL → sel = empty (returns `addr=NULL`).
- if `bl->flags & IOBL_BUF_RING` → io_ring_buffer_select.
- else → io_provided_buffer_select (legacy single).
- Release `ctx->uring_lock`.
- Returns `{ buf_list, addr }`.

REQ-6: io_ring_buffer_select(req, *len, bl, issue_flags) inner:
- tail = `smp_load_acquire(&bl->buf_ring->tail)`.
- head = `bl->head`.
- if tail == head → empty `io_br_sel`.
- buf = `&bl->buf_ring->bufs[head & bl->mask]`.
- buf_len = `READ_ONCE(buf->len)`; if `*len == 0 || *len > buf_len` → `*len = buf_len`.
- if `head + 1 == tail` → `req->flags |= REQ_F_BL_EMPTY` (caller can refrain from arming poll-rearm).
- `req->flags |= REQ_F_BUFFER_RING | REQ_F_BUFFERS_COMMIT`.
- `req->buf_index = READ_ONCE(buf->bid)`.
- sel = `{ buf_list = bl, addr = u64_to_user_ptr(READ_ONCE(buf->addr)) }`.
- if `io_should_commit(req, issue_flags)`:
  - `io_kbuf_commit(req, bl, *len, 1)`.
  - if commit returns false → `req->flags |= REQ_F_BUF_MORE` (INC mode signaled more bytes pending on the same slot).
  - sel.buf_list = NULL (caller no longer owns it).

REQ-7: io_provided_buffer_select(req, *len, bl) inner:
- if `bl->buf_list` empty → return NULL.
- kbuf = `list_first_entry(&bl->buf_list, io_buffer, list)`.
- `list_del(&kbuf->list); bl->nbufs--`.
- if `*len == 0 || *len > kbuf->len` → `*len = kbuf->len`.
- if list now empty → `req->flags |= REQ_F_BL_EMPTY`.
- `req->flags |= REQ_F_BUFFER_SELECTED`.
- `req->kbuf = kbuf; req->buf_index = kbuf->bid`.
- return `u64_to_user_ptr(kbuf->addr)`.

REQ-8: io_buffers_peek(req, arg, sel):
- lockdep_assert held(uring_lock).
- bl = `xa_load(io_bl_xa, arg->buf_group)`.
- if !bl → -ENOENT.
- if ring: ret = io_ring_buffers_peek; if ret > 0 → `REQ_F_BUFFERS_COMMIT` set; `sel->buf_list = bl`.
- else: legacy single-iov fallback via `io_provided_buffers_select`.

REQ-9: io_buffers_select(req, arg, sel, issue_flags):
- Acquire uring_lock.
- sel->buf_list = `xa_load(io_bl_xa, arg->buf_group)`; if NULL → -ENOENT.
- if ring: ret = io_ring_buffers_peek; on ret > 0 → `REQ_F_BUFFERS_COMMIT | REQ_F_BL_NO_RECYCLE`; commit immediately (cannot recycle once peeked under unlocked / async).
- else: legacy single-iov.
- Special: if `issue_flags & IO_URING_F_UNLOCKED` → drop uring_lock + `sel->buf_list = NULL` (caller no longer references list).

REQ-10: io_ring_buffers_peek(req, arg, bl):
- tail = `smp_load_acquire(&bl->buf_ring->tail)`.
- head = bl->head.
- nr_avail = min(tail - head, UIO_MAXIOV).
- if !nr_avail → -ENOBUFS.
- if arg->max_len: needed = ceil(max_len / first-buf-len); cap needed to `PEEK_MAX_IMPORT`; if nr_avail > needed → nr_avail = needed.
- if `arg->mode & KBUF_MODE_EXPAND` ∧ nr_avail > nr_iovs ∧ arg->max_len:
  - allocate new `iov[nr_avail]`.
  - if `KBUF_MODE_FREE` set → free old.
- else if nr_avail < nr_iovs → nr_iovs = nr_avail.
- if !arg->max_len → arg->max_len = INT_MAX.
- req->buf_index = first BID.
- Loop nr_iovs times:
  - len = READ_ONCE(buf->len); truncate to `arg->max_len`; if non-INC and truncated → `arg->partial_map = 1` and (if same array) shorten WRITE_ONCE(buf->len) — else stop.
  - iov->iov_base = `u64_to_user_ptr(READ_ONCE(buf->addr))`.
  - iov->iov_len = len.
  - arg->out_len += len; arg->max_len -= len; if 0 → break.
  - advance head (local, not bl->head — commit will handle).
- if local-head == tail → `REQ_F_BL_EMPTY`.
- `REQ_F_BUFFER_RING` set.
- return iov - arg->iovs.

REQ-11: io_kbuf_commit(req, bl, len, nr):
- if `!(req->flags & REQ_F_BUFFERS_COMMIT)` → return true (already committed or N/A).
- Clear `REQ_F_BUFFERS_COMMIT`.
- if len < 0 → return true (error path; nothing consumed but we must not leak the flag).
- if `bl->flags & IOBL_INC` → tail-call io_kbuf_inc_commit(bl, len).
- else: `bl->head += nr`.
- return true.

REQ-12: io_kbuf_inc_commit(bl, len):
- if len == 0 → false.
- while (len > 0):
  - buf = `&bl->buf_ring->bufs[bl->head & bl->mask]`.
  - buf_len = READ_ONCE(buf->len).
  - this_len = min(len, buf_len).
  - buf_len -= this_len.
  - if buf_len > `bl->min_left_sub_one` ∨ !this_len:
    - WRITE_ONCE(buf->addr, buf->addr + this_len).
    - WRITE_ONCE(buf->len, buf_len) — slot stays usable.
    - return false (slot not yet retired).
  - WRITE_ONCE(buf->len, 0); bl->head++; len -= this_len (slot retired).
- return true (all consumed within retired slots).

REQ-13: io_should_commit(req, issue_flags):
- if `IO_URING_F_UNLOCKED` → true (io-wq context; commit now or nobody will).
- if !file_can_poll ∧ !is_uring_cmd → true.
- else false.

REQ-14: __io_put_kbufs(req, bl, len, nbufs):
- Build CQE flags: `IORING_CQE_F_BUFFER | (req->buf_index << IORING_CQE_BUFFER_SHIFT)`.
- if `!(req->flags & REQ_F_BUFFER_RING)` → `io_kbuf_drop_legacy(req)` and return.
- ret = `__io_put_kbuf_ring(req, bl, len, nbufs)`:
  - if bl → `io_kbuf_commit(req, bl, len, nbufs)`.
  - if returned-true ∧ `req->flags & REQ_F_BUF_MORE` → force false.
  - Clear `REQ_F_BUFFER_RING | REQ_F_BUF_MORE` from req->flags.
- if !ret → flags |= `IORING_CQE_F_BUF_MORE` (peer: more data pending on the same buffer slot).

REQ-15: io_kbuf_drop_legacy(req):
- WARN if `!(req->flags & REQ_F_BUFFER_SELECTED)`.
- Clear flag, kfree(req->kbuf), kbuf = NULL.

REQ-16: io_kbuf_recycle_legacy(req, issue_flags):
- Acquire uring_lock.
- buf = req->kbuf.
- bl = `io_buffer_get_list(ctx, buf->bgid)`.
- if bl exists ∧ `!(bl->flags & IOBL_BUF_RING)` → `list_add(&buf->list, &bl->buf_list); bl->nbufs++`.
- else (list converted to ring, or removed) → kfree(buf).
- Clear `REQ_F_BUFFER_SELECTED`, req->kbuf = NULL.
- Release uring_lock.

REQ-17: io_provide_buffers_prep(SQE):
- Reject `sqe->rw_flags | sqe->splice_fd_in`.
- nbufs = `sqe->fd`; 1 ≤ nbufs ≤ `MAX_BIDS_PER_BGID` else -E2BIG.
- addr = sqe->addr; len = sqe->len (must != 0).
- check_mul_overflow(len, nbufs) → -EOVERFLOW.
- check_add_overflow(addr, size) → -EOVERFLOW.
- access_ok(addr, size) → else -EFAULT.
- bgid = sqe->buf_group.
- bid = sqe->off; ≤ USHRT_MAX; bid + nbufs ≤ MAX_BIDS_PER_BGID.

REQ-18: io_remove_buffers_prep(SQE):
- Reject `sqe->rw_flags | sqe->addr | sqe->len | sqe->off | sqe->splice_fd_in`.
- nbufs = sqe->fd; 1 ≤ nbufs ≤ MAX_BIDS_PER_BGID.
- bgid = sqe->buf_group.

REQ-19: io_manage_buffers_legacy(req, issue_flags):
- Acquire uring_lock.
- bl = io_buffer_get_list(ctx, p->bgid).
- if !bl:
  - if opcode != PROVIDE → -ENOENT.
  - kzalloc bl; INIT_LIST_HEAD(&bl->buf_list); io_buffer_add_list (XArray insert under mmap_lock).
- if `bl->flags & IOBL_BUF_RING` → -EINVAL (cannot mix legacy / ring on the same BGID).
- if PROVIDE → io_add_buffers (loop kmalloc N io_buffer, list_add_tail, addr += len, bid++, cond_resched).
- else REMOVE → io_remove_buffers_legacy (loop list_del_front+kfree, cond_resched).
- Release uring_lock; req_set_fail on ret < 0; io_req_set_res(ret); return IOU_COMPLETE.

REQ-20: io_register_pbuf_ring(ctx, arg):
- lockdep_assert_held(uring_lock).
- copy_from_user(reg).
- reg.resv must be zero.
- reg.flags must subset of `IOU_PBUF_RING_MMAP | IOU_PBUF_RING_INC`.
- reg.ring_entries must be power-of-2.
- reg.ring_entries < 65536.
- if !IOU_PBUF_RING_INC ∧ reg.min_left != 0 → -EINVAL.
- bl = get existing for reg.bgid: if BUF_RING ∨ buf_list nonempty → -EEXIST; else io_destroy_bl(ctx, bl) (replace empty legacy stub).
- kzalloc new bl.
- mmap_offset = `(unsigned long)reg.bgid << IORING_OFF_PBUF_SHIFT`.
- ring_size = `flex_array_size(br, bufs, reg.ring_entries)`.
- rd.size = PAGE_ALIGN(ring_size).
- if !IOU_PBUF_RING_MMAP → rd.user_addr = reg.ring_addr; rd.flags |= `IORING_MEM_REGION_TYPE_USER` (pin user range).
- io_create_region(ctx, &bl->region, &rd, mmap_offset).
- br = io_region_get_ptr(&bl->region).
- (SHM_COLOUR arches) if !MMAP ∧ (ring_addr|br) misaligned vs SHM_COLOUR → -EINVAL.
- bl->mask = ring_entries - 1.
- bl->flags |= IOBL_BUF_RING.
- bl->buf_ring = br.
- if reg.min_left → bl->min_left_sub_one = min_left - 1.
- if IOU_PBUF_RING_INC → bl->flags |= IOBL_INC.
- io_buffer_add_list(ctx, bl, reg.bgid) under mmap_lock.
- fail path frees region + bl.

REQ-21: io_unregister_pbuf_ring(ctx, arg):
- lockdep_assert_held(uring_lock).
- copy_from_user(reg); reg.resv zero, reg.flags zero.
- bl = io_buffer_get_list(ctx, reg.bgid); !bl → -ENOENT.
- if `!(bl->flags & IOBL_BUF_RING)` → -EINVAL (won't tear down legacy via this op).
- scoped_guard(mmap_lock) xa_erase.
- io_put_bl(ctx, bl).

REQ-22: io_register_pbuf_status(ctx, arg):
- copy_from_user(buf_status); resv zero.
- bl = io_buffer_get_list(ctx, buf_status.buf_group); !bl → -ENOENT.
- !BUF_RING → -EINVAL.
- buf_status.head = bl->head.
- copy_to_user.

REQ-23: io_pbuf_get_region(ctx, bgid) (mmap-path lookup):
- lockdep_assert_held(mmap_lock).
- bl = xa_load(io_bl_xa, bgid).
- if !bl ∨ !BUF_RING → NULL.
- return &bl->region.
- This is the only reader path that can race with register/unregister, hence the mmap_lock discipline (writers always grab mmap_lock around xa_store/xa_erase).

REQ-24: io_destroy_buffers(ctx) (on ring teardown):
- Loop: scoped_guard(mmap_lock) { xa_find first present; xa_erase }; if found → io_put_bl.

REQ-25: io_put_bl(ctx, bl):
- if BUF_RING → io_free_region(ctx->user, &bl->region).
- else → io_remove_buffers_legacy(ctx, bl, -1U) (drain).
- kfree(bl).

REQ-26: Memory ordering invariants:
- Userspace producer: write `bufs[t & mask]` fields, then `smp_store_release(tail, t+1)`.
- Kernel consumer: `smp_load_acquire(tail)`; index `bufs[head & mask]`; `bl->head++` later.
- Kernel-mutation of slot (INC partial consume, peek truncate of buf->len) uses WRITE_ONCE on individual fields; userspace must not mutate slots between tail-publish and head-advance (kernel does not synchronize with userspace mutation post-publish other than via tail/head).

REQ-27: CQE encoding:
- `IORING_CQE_F_BUFFER` set whenever a buffer was selected.
- `(req->buf_index << IORING_CQE_BUFFER_SHIFT)` carries the BID in the high bits of `cqe->flags`.
- `IORING_CQE_F_BUF_MORE` set iff this completion did not retire the slot (INC mode, more bytes pending on same slot).

REQ-28: Locking discipline:
- Writers to `ctx->io_bl_xa`: hold `ctx->uring_lock`; inside that take `ctx->mmap_lock` around xa_store / xa_erase to fence the mmap reader path.
- Reader from mmap path: hold `ctx->mmap_lock` only.
- bl->head is mutated only from the kernel side under uring_lock.
- bl->buf_list is mutated only under uring_lock (legacy).
- bl->buf_ring is shared memory; ordering per REQ-26.

## Acceptance Criteria

- [ ] AC-1: IORING_REGISTER_PBUF_RING with `ring_entries=8`, `IOU_PBUF_RING_MMAP` set: kernel allocates region; subsequent `mmap(IORING_OFF_PBUF_RING | (bgid<<16))` returns the same `io_uring_buf_ring`.
- [ ] AC-2: `ring_entries` not power-of-2 → -EINVAL.
- [ ] AC-3: `ring_entries >= 65536` → -EINVAL.
- [ ] AC-4: Setting `min_left` without `IOU_PBUF_RING_INC` → -EINVAL.
- [ ] AC-5: Re-register existing BGID with active ring → -EEXIST.
- [ ] AC-6: IOSQE_BUFFER_SELECT on recv with empty ring → completion `-ENOBUFS`.
- [ ] AC-7: Single-buf select: CQE carries `IORING_CQE_F_BUFFER` and `flags >> IORING_CQE_BUFFER_SHIFT == bid`.
- [ ] AC-8: `head + 1 == tail` at select time → `REQ_F_BL_EMPTY` propagated (no auto-rearm).
- [ ] AC-9: INC mode partial consume of len < (slot_len - min_left) leaves slot at advanced addr/len; head unchanged.
- [ ] AC-10: INC mode consume of remainder retires slot; head advances by 1; CQE has no `F_BUF_MORE`.
- [ ] AC-11: Legacy PROVIDE_BUFFERS then REMOVE_BUFFERS round-trip: counts symmetric.
- [ ] AC-12: Legacy and ring on same BGID rejected (-EEXIST or -EINVAL).
- [ ] AC-13: nbufs > `MAX_BIDS_PER_BGID` (65536) → -E2BIG on PROVIDE prep.
- [ ] AC-14: `bid + nbufs > MAX_BIDS_PER_BGID` → -EINVAL.
- [ ] AC-15: addr/len overflow → -EOVERFLOW.
- [ ] AC-16: unmapped user range → -EFAULT.
- [ ] AC-17: IORING_REGISTER_PBUF_STATUS returns `bl->head`.
- [ ] AC-18: io_destroy_buffers on ring exit reclaims every BGID under both flavors.

## Architecture

```
struct BufferList {
  // discriminated by flags & IOBL_BUF_RING
  inner: BufferListInner,
  nbufs: i32,                 // legacy count
  bgid: u16,
  head: u16,                  // ring consumer
  mask: u16,                  // ring_entries - 1
  flags: u16,                 // IOBL_BUF_RING | IOBL_INC
  min_left_sub_one: u32,      // IOBL_INC threshold - 1
  region: MappedRegion,       // pinned-user or kernel-alloc region
}

enum BufferListInner {
  Legacy { buf_list: ListHead<LegacyBuffer> },
  Ring   { buf_ring: *mut RingBufRing },
}

struct LegacyBuffer { list: ListNode, addr: u64, len: u32, bid: u16, bgid: u16 }
struct RingBuf    { addr: u64, len: u32, bid: u16, resv: u16 }
struct RingBufRing { /* slot[0] header: tail at offset 14; slots[1..] flex */ }

struct BrSel { buf_list: Option<*BufferList>, addr_or_val: union { addr: UserPtr, val: isize } }

struct BufSelArg {
  iovs: *mut Iovec, out_len: usize, max_len: usize,
  nr_iovs: u16, mode: u16, buf_group: u16, partial_map: u16,
}

struct ProvideBufCmd { file: *File, addr: u64, len: u32, bgid: u32, nbufs: u32, bid: u16 }
```

`Kbuf::buffer_select(req, len, buf_group, issue_flags) -> BrSel`:
1. ring_submit_lock(ctx, issue_flags).
2. bl = Kbuf::get_list(ctx, buf_group).
3. if bl.is_none() → drop lock, return empty BrSel.
4. if bl.flags & IOBL_BUF_RING → sel = Kbuf::ring_buffer_select(req, len, bl, issue_flags).
5. else → sel.addr = Kbuf::legacy_buffer_select(req, len, bl); sel.buf_list = None.
6. ring_submit_unlock(ctx, issue_flags).
7. return sel.

`Kbuf::ring_buffer_select(req, len, bl, issue_flags) -> BrSel`:
1. tail = smp_load_acquire(&bl.buf_ring.tail).
2. head = bl.head.
3. if tail == head → return BrSel::EMPTY.
4. if head + 1 == tail → req.flags |= REQ_F_BL_EMPTY.
5. buf = &bl.buf_ring.bufs[head & bl.mask].
6. buf_len = READ_ONCE(buf.len); if *len == 0 ∨ *len > buf_len → *len = buf_len.
7. req.flags |= REQ_F_BUFFER_RING | REQ_F_BUFFERS_COMMIT.
8. req.buf_index = READ_ONCE(buf.bid).
9. sel = BrSel { buf_list: Some(bl), addr: u64_to_user_ptr(READ_ONCE(buf.addr)) }.
10. if Kbuf::should_commit(req, issue_flags):
    - if !Kbuf::commit(req, bl, *len, 1) → req.flags |= REQ_F_BUF_MORE.
    - sel.buf_list = None.
11. return sel.

`Kbuf::commit(req, bl, len, nr) -> bool`:
1. if !(req.flags & REQ_F_BUFFERS_COMMIT) → return true.
2. req.flags &= !REQ_F_BUFFERS_COMMIT.
3. if len < 0 → return true.
4. if bl.flags & IOBL_INC → return Kbuf::inc_commit(bl, len).
5. bl.head = bl.head.wrapping_add(nr).
6. return true.

`Kbuf::inc_commit(bl, len) -> bool`:
1. if len == 0 → return false.
2. while len > 0:
   - buf = &bl.buf_ring.bufs[bl.head & bl.mask].
   - buf_len = READ_ONCE(buf.len).
   - this_len = min(len, buf_len).
   - buf_len -= this_len.
   - if buf_len > bl.min_left_sub_one ∨ this_len == 0:
     - WRITE_ONCE(buf.addr, buf.addr + this_len).
     - WRITE_ONCE(buf.len, buf_len).
     - return false.
   - WRITE_ONCE(buf.len, 0).
   - bl.head = bl.head.wrapping_add(1).
   - len -= this_len.
3. return true.

`Kbuf::register_ring(ctx, arg) -> Result<()>`:
1. lockdep_assert_held(ctx.uring_lock).
2. reg = copy_from_user::<BufReg>(arg)?.
3. Validate reg.resv == 0, reg.flags ⊆ {MMAP, INC}, ring_entries.is_power_of_two(), ring_entries < 65536.
4. if !(reg.flags & INC) ∧ reg.min_left != 0 → -EINVAL.
5. existing = Kbuf::get_list(ctx, reg.bgid).
6. if let Some(bl) = existing:
   - if bl.flags & BUF_RING ∨ !bl.buf_list.is_empty() → -EEXIST.
   - Kbuf::destroy_list(ctx, bl).
7. bl = BufferList::zeroed().
8. mmap_offset = (reg.bgid as u64) << IORING_OFF_PBUF_SHIFT.
9. ring_size = flex_array_size(RingBufRing, bufs, reg.ring_entries).
10. rd = RegionDesc { size: PAGE_ALIGN(ring_size), .. }.
11. if !(reg.flags & MMAP) → rd.user_addr = reg.ring_addr; rd.flags |= MEM_REGION_TYPE_USER.
12. io_create_region(ctx, &bl.region, &rd, mmap_offset)?.
13. br = io_region_get_ptr(&bl.region).
14. (SHM_COLOUR arches) if !(reg.flags & MMAP) ∧ ((reg.ring_addr|br) & (SHM_COLOUR-1)) != 0 → fail -EINVAL.
15. bl.mask = reg.ring_entries - 1.
16. bl.flags |= IOBL_BUF_RING.
17. bl.buf_ring = br.
18. if reg.min_left != 0 → bl.min_left_sub_one = reg.min_left - 1.
19. if reg.flags & INC → bl.flags |= IOBL_INC.
20. Kbuf::add_list(ctx, bl, reg.bgid) (under mmap_lock).

`Kbuf::manage_legacy(req, issue_flags) -> i32`:
1. ring_submit_lock(ctx, issue_flags).
2. bl = Kbuf::get_list(ctx, p.bgid).
3. ret = __manage_legacy(req, bl):
   - if bl.is_none():
     - if req.opcode != PROVIDE_BUFFERS → -ENOENT.
     - bl = new BufferList::legacy(); Kbuf::add_list.
   - if bl.flags & BUF_RING → -EINVAL.
   - if PROVIDE → Kbuf::add_legacy(ctx, p, bl) loop.
   - else REMOVE → Kbuf::remove_legacy(ctx, bl, p.nbufs) loop.
4. ring_submit_unlock(ctx, issue_flags).
5. if ret < 0 → req_set_fail(req).
6. io_req_set_res(req, ret, 0).
7. return IOU_COMPLETE.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `ring_entries_pow2` | INVARIANT | per-register_ring: post: bl.mask + 1 is power of 2. |
| `bgid_not_aliased` | INVARIANT | per-register_ring success: existing entry for bgid implies destroyed or rejected. |
| `tail_acquire_head_release` | INVARIANT | per-ring_buffer_select: tail loaded with acquire; head advance happens-after slot field reads. |
| `inc_commit_head_monotone` | INVARIANT | per-inc_commit: bl.head only increases (modulo 2^16) within a call. |
| `select_returns_addr_or_empty` | INVARIANT | per-buffer_select: returns {NULL, .} or valid user_ptr aliased to a published ring slot. |
| `legacy_drop_kfree_once` | INVARIANT | per-drop_legacy: req.kbuf kfree'd exactly once on cancel path. |
| `recycle_legacy_returns_to_correct_bgid` | INVARIANT | per-recycle_legacy: kbuf re-listed only if matching bgid bl still exists and is legacy. |
| `pbuf_status_head_consistency` | INVARIANT | per-register_status: reported head equals bl.head at call time under uring_lock. |

### Layer 2: TLA+

`io_uring/kbuf.tla`:
- Variables: bl.head (kernel), tail (userspace), slots[0..ring_entries], req.flags, cqe_f_buffer, cqe_f_buf_more, partial_map.
- Actions: UserspaceProvide(slot, addr, len, bid), KernelSelect(req), KernelCommit(req, len), UserspaceMmapRead(bgid).
- Properties:
  - `safety_no_use_after_retire` — per-slot: once head crosses slot, kernel does not deref slot until userspace re-provides.
  - `safety_inc_partial_keeps_slot_owned` — per-INC: if commit returns false, head unchanged, slot.addr/len mutated by kernel only.
  - `safety_cqe_bid_matches_select` — per-completion: CQE's BID == buf->bid at select time.
  - `safety_mmap_consistent_with_register` — per-mmap: io_pbuf_get_region returns region iff bl in xa under mmap_lock.
  - `liveness_per_provide_eventually_selectable` — per-tail-publish: eventually a peer select consumes a slot (under non-empty + reader present).
  - `safety_no_legacy_ring_mix` — per-BGID: never both buf_list nonempty AND IOBL_BUF_RING.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Kbuf::register_ring` post: bl in io_bl_xa[bgid]; bl.flags & IOBL_BUF_RING; bl.region pinned | `Kbuf::register_ring` |
| `Kbuf::unregister_ring` post: bl removed; region freed | `Kbuf::unregister_ring` |
| `Kbuf::buffer_select` post: returns addr ⟹ req.buf_index < 2^16 ∧ CQE will carry F_BUFFER | `Kbuf::buffer_select` |
| `Kbuf::commit` post: REQ_F_BUFFERS_COMMIT cleared on req | `Kbuf::commit` |
| `Kbuf::inc_commit` post: ∀ slot retired: WRITE_ONCE(buf.len, 0) happened | `Kbuf::inc_commit` |
| `Kbuf::manage_legacy` post: PROVIDE — bl.nbufs += nbufs; REMOVE — bl.nbufs -= min(nbufs, original) | `Kbuf::manage_legacy` |
| `Kbuf::destroy_all` post: ctx.io_bl_xa empty | `Kbuf::destroy_all` |

### Layer 4: Verus/Creusot functional

`Per-userspace ring_provide(N slots) → kernel buffer_select × N (consume) → buffer_commit × N → ring drained` is observationally equivalent to upstream `io_uring/kbuf.c` per `Documentation/io_uring/buffer-ring.rst` and `Documentation/io_uring/io_uring.rst`. INC-mode trace `provide(1 big slot, min_left=4096) → N×recv(small len)` retires the slot exactly when residual ≤ min_left, with `F_BUF_MORE` emitted on every CQE except the retirement.

## Hardening

(Inherits row-1 features from `io_uring/00-overview.md` § Hardening.)

provided-buffers reinforcement:

- **Per-power-of-2 ring_entries enforced** — defense against per-mask-arithmetic-OOB on head/tail indexing.
- **Per-ring_entries < 65536** — defense against per-tail-wrap ambiguity (head/tail are 16-bit; equality means empty, not "full minus one").
- **Per-min_left only with IOU_PBUF_RING_INC** — defense against per-misinterpreted-residual.
- **Per-existing-BGID-with-active-content rejected** — defense against per-silent-clobber of in-use buffer set.
- **Per-buf_group XArray under uring_lock + mmap_lock** — defense against per-mmap-race vs register/unregister.
- **Per-mem_is_zero(resv) check** — defense against per-future-flag-collision.
- **Per-access_ok + check_mul/add_overflow on legacy PROVIDE** — defense against per-user-pointer integer overflow.
- **Per-MAX_BIDS_PER_BGID cap (1 << 16)** — defense against per-BID-aliasing exhaustion.
- **Per-SHM_COLOUR alignment check** — defense against per-cache-alias divergence between kernel and user views.
- **Per-IORING_MEM_REGION_TYPE_USER pinning** — defense against per-user-unmap during ring lifetime.
- **Per-bl->head kernel-private** — defense against per-userspace-monkeying with consumer index.
- **Per-RW_ONCE on slot.addr/slot.len reads in select / peek** — defense against per-tearing under concurrent userspace stores between tail-publish and head-advance (best-effort).
- **Per-IOU_PBUF_RING_INC partial retire respects min_left_sub_one** — defense against per-too-small-residual buffer re-use.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY on provided-buffer ring** — bounds-checked copy on the user-mapped `io_uring_buf_ring` head/tail/mask and on every `slot.addr` / `slot.len` field cracked open for receive selection.
- **PAX_KERNEXEC** — write-protects `io_buffer_select`, `io_buffer_recycle`, and the kbuf op table; rejects late patching of the buffer-select hot path.
- **PAX_RANDKSTACK** — per-call stack-offset randomization on `io_provide_buffers`, `io_register_pbuf_ring`, and `io_buffer_select`.
- **PAX_REFCOUNT** — saturating trap on `bl->refs` (per-bgid buffer list) and on the `io_buffer.list` chain held under `ctx->uring_lock`.
- **PAX_MEMORY_SANITIZE** — zeroes `struct io_buffer_list` and `struct io_buffer` slab objects on free; scrubs unregistered pbuf-ring backing pages before unpin.
- **PAX_UDEREF** — every provided-buffer-ring slot is read with `READ_ONCE` against the kernel-side `kvaddr` mapping; ioctl-style `addr` writes from user are user-pointer-faulted at prep.
- **PAX_RAP / kCFI** — forward-edge CFI on `io_kbuf_recycle`/`io_kbuf_drop` callbacks invoked from per-op cleanup.
- **GRKERNSEC_HIDESYM** — strips kbuf helpers and `bl->head`/`bl->ring` pointers from kallsyms-leaking debug paths.
- **GRKERNSEC_DMESG** — restricts `pr_warn` buffer-ring shrink/exhaustion traces to CAP_SYSLOG.
- **BID/BGID bounds** — `bid` is masked by `bl->mask` (power-of-2 ring); `bgid` is `< IORING_MAX_REG_BUFFERS`; both bounds are checked in `__io_remove_buffers` and `io_buffer_select` under `ctx->uring_lock`; defends against per-BID-aliasing exhaustion and per-BGID OOB write to `ctx->io_bl[]`.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `io_uring/memmap.c` mapping helpers (covered in `io_uring-core.md` Tier-3)
- `io_uring/zcrx.c` zero-copy receive buffer pools (covered in `net.md` / dedicated zcrx Tier-3 if expanded)
- `IORING_REGISTER_PBUF_RING` from the registration multiplex (covered in `register.md` Tier-3)
- recv / read consumer hot paths (covered in `net.md` / `io_uring-core.md`)
- Implementation code
