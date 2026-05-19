# Tier-3: fs/iomap/buffered-io.c — iomap buffered I/O

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: fs/00-overview.md
upstream-paths:
  - fs/iomap/buffered-io.c (~1976 lines)
  - include/linux/iomap.h
  - fs/iomap/internal.h
  - fs/iomap/trace.h
-->

## Summary

The **iomap** subsystem is the modern, buffer-head-free abstraction for buffered page-cache I/O, used by ext4 (extent-based regular files), xfs, gfs2, zonefs, and erofs. Per-`struct iomap` describes one extent of a file (offset, length, type ∈ {HOLE, DELALLOC, MAPPED, UNWRITTEN, INLINE}, disk addr, bdev, flags). Per-`struct iomap_ops` exposes the filesystem's `iomap_begin` / `iomap_end` callbacks; per-`struct iomap_iter` is the generic iterator that walks a (pos, len) range converting it into a sequence of iomaps. Per-`struct iomap_folio_state` (ifs) tracks **per-block sub-folio state** as two adjacent bitmaps inside `folio->private`: `[0..nblocks)` = uptodate, `[nblocks..2*nblocks)` = dirty — needed when blocksize < folio size (e.g. large folios with 4K filesystem blocks). Per-`iomap_file_buffered_write` drives the write side via `iomap_write_iter` → `iomap_write_begin` → `copy_folio_from_iter_atomic` → `iomap_write_end`. Per-`iomap_read_folio` and `iomap_readahead` drive the read side via `iomap_read_folio_iter`. Per-`iomap_zero_range` / `iomap_truncate_page` zero a sub-block tail on truncate. Per-`iomap_dirty_folio` / `iomap_release_folio` / `iomap_invalidate_folio` are the modern aops hooks. Per-`iomap_writepages` / `iomap_writeback_folio` drive writeback via filesystem-supplied `writeback_range` (XFS only at present; ext4 still uses bh writeback). Per-`IOMAP_F_BUFFER_HEAD` is the legacy bridge: when a filesystem (gfs2, ext4 sub-page) needs bh chains, iomap will create them via `iomap_to_bh` and the bh path takes over.

This Tier-3 covers `fs/iomap/buffered-io.c` (~1976 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct iomap` | per-extent descriptor (offset/length/type/addr/flags) | `Iomap` |
| `struct iomap_ops` | per-FS iomap_begin / iomap_end vtable | `IomapOps` |
| `struct iomap_iter` | per-call iterator (inode, pos, len, flags, iomap, srcmap, private, fbatch) | `IomapIter` |
| `iomap_iter()` | per-step iterator advance + iomap_begin invocation | `IomapIter::next` |
| `iomap_iter_advance()` / `_advance_full()` | per-step length consumption | `IomapIter::advance` |
| `iomap_iter_srcmap()` | per-iter pick srcmap vs iomap (COW source) | `IomapIter::srcmap` |
| `iomap_length()` | per-iter remaining bytes in current iomap | `IomapIter::length` |
| `struct iomap_write_ops` | per-write get_folio / put_folio / read_folio_range vtable | `IomapWriteOps` |
| `struct iomap_writepage_ctx` | per-writeback ctx (inode, wbc, iomap, writeback_range, ops) | `IomapWritepageCtx` |
| `struct iomap_read_folio_ctx` | per-read submit ctx (rac, cur_folio, read_ctx, ops) | `IomapReadFolioCtx` |
| `struct iomap_folio_state` | per-folio sub-block uptodate+dirty bitmaps + IO counters | `IomapFolioState` |
| `ifs_alloc()` / `ifs_free()` | per-folio ifs attach/detach (folio.private) | `Ifs::alloc` / `free` |
| `ifs_is_fully_uptodate()` | per-folio all-blocks-uptodate query | `Ifs::is_fully_uptodate` |
| `ifs_set_range_uptodate()` / `iomap_set_range_uptodate()` | per-folio uptodate bitmap set | `Ifs::set_range_uptodate` |
| `ifs_set_range_dirty()` / `iomap_set_range_dirty()` | per-folio dirty bitmap set | `Ifs::set_range_dirty` |
| `ifs_clear_range_dirty()` / `iomap_clear_range_dirty()` | per-folio dirty bitmap clear | `Ifs::clear_range_dirty` |
| `ifs_find_dirty_range()` / `iomap_find_dirty_range()` | per-folio next dirty range | `Ifs::find_dirty_range` |
| `ifs_next_uptodate_block()` / `_nonuptodate_block()` / `_dirty_block()` / `_clean_block()` | per-folio block iterator helpers | `Ifs::next_*_block` |
| `iomap_adjust_read_range()` | per-folio trim read to needed sub-range | `Iomap::adjust_read_range` |
| `iomap_block_needs_zeroing()` | per-block hole / unwritten / past-EOF check | `Iomap::block_needs_zeroing` |
| `iomap_read_inline_data()` | per-IOMAP_INLINE copy into folio | `Iomap::read_inline_data` |
| `iomap_read_folio_iter()` | per-iter read submit per folio range | `Iomap::read_folio_iter` |
| `iomap_read_folio()` | per-folio aops .read_folio | `Iomap::read_folio` |
| `iomap_readahead()` / `iomap_readahead_iter()` | per-readahead aops .readahead | `Iomap::readahead` |
| `iomap_read_init()` / `iomap_read_end()` | per-folio read submit init / completion bridge | `Iomap::read_init` / `read_end` |
| `iomap_finish_folio_read()` | per-bio-end read accounting | `Iomap::finish_folio_read` |
| `iomap_is_partially_uptodate()` | per-folio aops .is_partially_uptodate | `Iomap::is_partially_uptodate` |
| `iomap_get_folio()` | per-write filemap_get_folio + FGP_WRITEBEGIN | `Iomap::get_folio` |
| `iomap_dirty_folio()` | per-folio aops .dirty_folio + ifs_alloc + set-dirty | `Iomap::dirty_folio` |
| `iomap_release_folio()` | per-folio aops .release_folio (refuses dirty) | `Iomap::release_folio` |
| `iomap_invalidate_folio()` | per-folio aops .invalidate_folio (cancel_dirty + ifs_free) | `Iomap::invalidate_folio` |
| `__iomap_write_begin()` | per-write per-folio read-around / zero-around | `Iomap::write_begin_inner` |
| `__iomap_get_folio()` / `__iomap_put_folio()` | per-write folio acquire/release with FS hooks | `Iomap::get_folio_inner` / `put_folio_inner` |
| `iomap_trim_folio_range()` | per-folio range clamp to iomap length | `Iomap::trim_folio_range` |
| `iomap_write_begin_inline()` / `iomap_write_end_inline()` | per-IOMAP_INLINE write paths | `Iomap::write_(begin|end)_inline` |
| `iomap_write_begin()` / `iomap_write_end()` | per-write per-folio public hooks | `Iomap::write_begin` / `write_end` |
| `__iomap_write_end()` | per-write per-folio uptodate + dirty bookkeeping | `Iomap::write_end_inner` |
| `iomap_write_iter()` | per-write loop copying iov_iter into folios | `Iomap::write_iter` |
| `iomap_file_buffered_write()` | per-call file_op write entry | `Iomap::file_buffered_write` |
| `iomap_write_failed()` | per-failure pagecache truncate beyond i_size | `Iomap::write_failed` |
| `iomap_write_delalloc_release()` | per-failure delalloc reservation rollback | `Iomap::write_delalloc_release` |
| `iomap_write_delalloc_scan()` / `_punch()` / `_ifs_punch()` | per-failure scan dirty + punch holes | `Iomap::write_delalloc_*` |
| `iomap_file_unshare()` / `iomap_unshare_iter()` | per-call reflink unshare | `Iomap::file_unshare` |
| `iomap_zero_iter()` / `iomap_zero_iter_flush_and_stale()` | per-zero per-iter step | `Iomap::zero_iter` |
| `iomap_zero_range()` | per-call hole-aware zero | `Iomap::zero_range` |
| `iomap_truncate_page()` | per-call sub-block tail zero on truncate | `Iomap::truncate_page` |
| `iomap_fill_dirty_folios()` | per-call gather dirty folios (folio_batch) | `Iomap::fill_dirty_folios` |
| `iomap_folio_mkwrite_iter()` / `iomap_page_mkwrite()` | per-fault VM_FAULT mkwrite | `Iomap::page_mkwrite` |
| `iomap_writeback_init()` | per-folio writeback start: prime write_bytes_pending | `Iomap::writeback_init` |
| `iomap_writeback_range()` | per-iomap delegate to FS writeback_range | `Iomap::writeback_range` |
| `iomap_writeback_handle_eof()` | per-folio truncate-EOF interaction | `Iomap::writeback_handle_eof` |
| `iomap_writeback_folio()` | per-folio aops writepage equivalent | `Iomap::writeback_folio` |
| `iomap_writepages()` | per-mapping aops .writepages | `Iomap::writepages` |
| `iomap_finish_folio_write()` | per-bio-end write accounting | `Iomap::finish_folio_write` |

## Compatibility contract

REQ-1: struct iomap_folio_state — folio.private layout when blocksize < folio_size:
- state_lock: per-ifs spinlock guarding bitmap mutations.
- read_bytes_pending: unsigned int — outstanding read bytes; XOR semantics with folio_end_read.
- write_bytes_pending: atomic_t — outstanding write bytes; folio_end_writeback when reaches 0.
- state[]: flex-array of unsigned long; first `BITS_TO_LONGS(nblocks)` words = uptodate map; next `BITS_TO_LONGS(nblocks)` = dirty map; allocated as `BITS_TO_LONGS(2 * nblocks)` longs.

REQ-2: ifs_alloc(inode, folio, flags) -> ifs:
- if folio.private ∨ nblocks ≤ 1: return existing.
- gfp = (IOMAP_NOWAIT ? GFP_NOWAIT : GFP_NOFS | __GFP_NOFAIL).
- ifs = kzalloc_flex(...).
- spin_lock_init.
- if folio_test_uptodate: bitmap_set(state, 0, nblocks).
- if folio_test_dirty: bitmap_set(state, nblocks, nblocks).
- folio_attach_private(folio, ifs).

REQ-3: ifs_free(folio):
- if !folio.private: return.
- WARN if folio_test_uptodate ∧ !ifs_is_fully_uptodate.
- WARN if folio_test_dirty ∧ !ifs_is_fully_dirty.
- folio_detach_private; kfree.

REQ-4: iomap_set_range_uptodate(folio, off, len):
- if folio_test_uptodate: return.
- if ifs:
  - spin_lock_irqsave(ifs.state_lock).
  - mark_uptodate = ifs_set_range_uptodate(folio, ifs, off, len) ∧ !ifs.read_bytes_pending.
  - spin_unlock.
- if mark_uptodate: folio_mark_uptodate(folio).
- Subtlety: if a read with read_bytes_pending > 0 is in flight, defer the folio-mark to iomap_read_end which uses XOR-based folio_end_read.

REQ-5: iomap_set_range_dirty / clear_range_dirty:
- if ifs: under state_lock toggle bits [nblocks + first_blk .. + nr_blks).
- if !ifs: no-op (folio-level dirty bit is authoritative).

REQ-6: iomap_find_dirty_range(folio, *range_start, range_end):
- if *range_start ≥ range_end: return 0.
- if ifs: ifs_find_dirty_range (next BH_dirty bit; nblks = next clean - first dirty).
- else: range_end - *range_start (whole range).

REQ-7: iomap_adjust_read_range(inode, folio, *block_start, length, *poff, *plen):
- Trim [block_start, block_start+length) down to the contiguous range of non-uptodate blocks that intersect the operation; account first/last sub-folio offsets.

REQ-8: iomap_block_needs_zeroing(iter, block_start):
- srcmap = iomap_iter_srcmap.
- Returns true iff: srcmap.type == IOMAP_HOLE ∨ srcmap.type == IOMAP_UNWRITTEN ∨ block_start ≥ i_size_read(iter.inode).

REQ-9: iomap_read_inline_data(iter, folio):
- BUG if iomap.length > PAGE_SIZE - iomap.inline_data_offset.
- memcpy iomap.inline_data into folio[offset]; folio_zero_segment beyond; iomap_set_range_uptodate(folio, 0, folio_size).

REQ-10: iomap_read_folio_iter(iter, ctx, *bytes_submitted):
- For each [pos, pos+plen) within current iomap covering ctx.cur_folio:
  - if iomap_block_needs_zeroing: folio_zero_segments + iomap_set_range_uptodate; advance.
  - else: ifs.read_bytes_pending += plen (under state_lock); call ctx.ops.read_folio_range(iter, ctx, folio, pos, plen) — FS submits bio.
- Hand off cur_folio = NULL when full folio submitted or no ifs attached.

REQ-11: iomap_read_folio(ops, ctx, private):
- iter = {inode, pos = folio_pos(ctx.cur_folio), len = folio_size, private}.
- while iomap_iter(iter, ops) > 0: iter.status = iomap_read_folio_iter(iter, ctx, &bytes_submitted).
- if ctx.read_ctx ∧ ctx.ops.submit_read: ctx.ops.submit_read(iter, ctx).
- if ctx.cur_folio: iomap_read_end(ctx.cur_folio, bytes_submitted).

REQ-12: iomap_readahead(ops, ctx, private):
- iter = {inode = rac.mapping.host, pos = readahead_pos, len = readahead_length, private}.
- while iomap_iter > 0: iter.status = iomap_readahead_iter — pulls successive folios from readahead_folio.
- Submit + read_end as in REQ-11.

REQ-13: iomap_read_end(folio, bytes_submitted):
- folio_end_read uses XOR — if bytes_submitted == 0 (all zeroed in REQ-10): folio_end_read(folio, true) directly.
- Otherwise per-bio iomap_finish_folio_read drains read_bytes_pending; when it hits 0, the last completion XORs the uptodate bit.

REQ-14: iomap_finish_folio_read(folio, off, len, err):
- if ifs:
  - spin_lock_irqsave(state_lock).
  - if !err: uptodate = ifs_set_range_uptodate(folio, ifs, off, len).
  - ifs.read_bytes_pending -= len.
  - finished = (!ifs.read_bytes_pending).
  - spin_unlock.
  - if finished: folio_end_read(folio, uptodate ∧ !err).
- else: folio_end_read(folio, !err).

REQ-15: iomap_is_partially_uptodate(folio, from, count):
- if !ifs: return false.
- first = from >> blkbits; last = (from + count - 1) >> blkbits.
- return ifs_next_nonuptodate_block(folio, first, last) > last.

REQ-16: iomap_get_folio(iter, pos, len):
- fgp = FGP_WRITEBEGIN | FGP_NOFS.
- if IOMAP_NOWAIT: fgp |= FGP_NOWAIT.
- if IOMAP_DONTCACHE: fgp |= FGP_DONTCACHE.
- fgp |= fgf_set_order(len) (request large folio).
- __filemap_get_folio.

REQ-17: iomap_dirty_folio(mapping, folio):
- ifs_alloc(mapping.host, folio, 0).
- iomap_set_range_dirty(folio, 0, folio_size).
- filemap_dirty_folio(mapping, folio).

REQ-18: iomap_release_folio(folio, gfp):
- if folio_test_dirty: return false (refuse — can't drop partially dirty ifs).
- ifs_free; return true.

REQ-19: iomap_invalidate_folio(folio, offset, len):
- if (offset, len) covers whole folio: WARN if writeback; folio_cancel_dirty; ifs_free.
- else: no-op (sub-folio invalidate not supported via ifs drop; dirty bitmap retained).

REQ-20: __iomap_write_begin(iter, write_ops, len, folio):
- If write entirely overlaps folio ∧ !IOMAP_UNSHARE: return 0 (no per-block tracking needed; will dirty whole).
- ifs = ifs_alloc(inode, folio, flags); if NOWAIT ∧ !ifs ∧ nblocks > 1: return -EAGAIN.
- if folio_test_uptodate: return 0.
- Per-block loop covering [block_start, block_end):
  - iomap_adjust_read_range to skip uptodate sub-runs.
  - if write covers full sub-range (from ≤ poff ∧ to ≥ poff+plen): skip (will overwrite).
  - if iomap_block_needs_zeroing:
    - if IOMAP_UNSHARE: return -EIO (must read for COW).
    - folio_zero_segments around [from, to].
  - else: write_ops.read_folio_range(iter, folio, block_start, plen) — synchronous read.

REQ-21: __iomap_get_folio(iter, write_ops, pos, len):
- If write_ops.get_folio: delegate (FS hook).
- else: iomap_get_folio.

REQ-22: __iomap_put_folio(iter, write_ops, written, folio):
- if write_ops.put_folio: delegate.
- else: folio_unlock; folio_put.

REQ-23: iomap_trim_folio_range(iter, folio, *offset, *bytes):
- Clamp (*offset, *bytes) so range fits within iomap_length(iter) and within folio bounds.

REQ-24: iomap_write_begin(iter, write_ops, &folio, &offset, &bytes):
- folio = __iomap_get_folio(iter, write_ops, pos, len).
- if IS_ERR(folio): return.
- iomap_trim_folio_range.
- if IOMAP_F_BUFFER_HEAD on iter.iomap: __block_write_begin_int(folio, pos, len, NULL, iter.iomap) (bridge to bh path).
- elif srcmap.type == IOMAP_INLINE: iomap_write_begin_inline.
- else: __iomap_write_begin.

REQ-25: iomap_write_end(iter, len, copied, folio):
- srcmap = iomap_iter_srcmap.
- if srcmap.type == IOMAP_INLINE: iomap_write_end_inline.
- elif srcmap.flags & IOMAP_F_BUFFER_HEAD: bh = block_write_end(pos, len, copied, folio); WARN bh ≠ copied ∧ bh ≠ 0; return bh == copied.
- else: __iomap_write_end(inode, pos, len, copied, folio).

REQ-26: __iomap_write_end(inode, pos, len, copied, folio):
- if copied < len ∧ !folio_test_uptodate: return false (caller treats as short-copy fail).
- iomap_set_range_uptodate(folio, offset_in_folio(pos), copied).
- iomap_set_range_dirty(folio, offset_in_folio(pos), copied).
- filemap_dirty_folio_post if first dirty.
- Return true.

REQ-27: iomap_write_iter(iter, iov_iter, write_ops):
- chunk = mapping_max_folio_size.
- bdp_flags = (IOMAP_NOWAIT ? BDP_ASYNC : 0).
- Loop while iov_iter_count > 0 ∧ iomap_length > 0:
  - balance_dirty_pages_ratelimited_flags(mapping, bdp_flags); break on error.
  - bytes = min(chunk - (pos & chunk-1), iomap_length, iov_iter_count).
  - fault_in_iov_iter_readable; break -EFAULT if full miss.
  - iomap_write_begin(iter, write_ops, &folio, &offset, &bytes).
  - if iomap.flags & IOMAP_F_STALE: break (iomap revalidate).
  - if mapping_writably_mapped: flush_dcache_folio.
  - copied = copy_folio_from_iter_atomic(folio, offset, bytes, iov_iter).
  - written = iomap_write_end ? copied : 0.
  - i_size_write if pos + written > old_size; set IOMAP_F_SIZE_CHANGED.
  - __iomap_put_folio(iter, write_ops, written, folio).
  - if old_size < pos: pagecache_isize_extended.
  - cond_resched.
  - if written == 0: iomap_write_failed; iov_iter_revert(copied); halve chunk if > PAGE_SIZE; retry with bytes = copied.
  - else: total_written += written; iomap_iter_advance(iter, written).

REQ-28: iomap_file_buffered_write(iocb, iov_iter, ops, write_ops, private):
- iter = {inode = iocb.ki_filp.f_mapping.host, pos = iocb.ki_pos, len = iov_iter_count, flags = IOMAP_WRITE, private}.
- if IOCB_NOWAIT: flags |= IOMAP_NOWAIT.
- if IOCB_DONTCACHE: flags |= IOMAP_DONTCACHE.
- while iomap_iter(iter, ops) > 0: iter.status = iomap_write_iter(iter, iov_iter, write_ops).
- if iter.pos == iocb.ki_pos: return ret (no progress).
- ret = iter.pos - iocb.ki_pos; iocb.ki_pos = iter.pos; return ret.

REQ-29: iomap_write_failed(inode, pos, len):
- if pos + len > i_size: truncate_pagecache_range(inode, max(pos, i_size), pos + len - 1).

REQ-30: iomap_write_delalloc_release(inode, start_byte, end_byte, iomap, punch):
- Walk pagecache in [start, end); for clean folios (or clean sub-ranges via ifs): punch delalloc reservation back.
- Used by XFS to release speculative delalloc on failed write.

REQ-31: iomap_zero_range(inode, pos, len, *did_zero, ops, write_ops, private):
- fbatch + iter {flags = IOMAP_ZERO}.
- range_dirty = filemap_range_needs_writeback.
- Loop iomap_iter:
  - if iomap.type == HOLE ∨ UNWRITTEN (and no folio batch): if dirty ∧ UNWRITTEN: flush_and_stale (return -EAGAIN to revalidate); else: advance.
  - else: iomap_zero_iter.

REQ-32: iomap_zero_iter(iter, *did_zero, write_ops):
- For range:
  - if iomap_block_needs_zeroing: skip if folio not present (hole stays hole); else folio_zero_range + iomap_set_range_uptodate + iomap_set_range_dirty; *did_zero = true.
  - else: get folio; iomap_write_begin (with len = zero-len); folio_zero_range; iomap_write_end.

REQ-33: iomap_truncate_page(inode, pos, *did_zero, ops, write_ops, private):
- blocksize = i_blocksize; off = pos & (blocksize - 1).
- if !off: return 0 (aligned, no work).
- return iomap_zero_range(inode, pos, blocksize - off, did_zero, ops, write_ops, private).

REQ-34: iomap_fill_dirty_folios(iter, *start, end, *iomap_flags):
- if !iter.fbatch: *start = end; return 0.
- count = filemap_get_folios_dirty(mapping, &pstart, pend, iter.fbatch).
- *start = pstart << PAGE_SHIFT; *iomap_flags |= IOMAP_F_FOLIO_BATCH.

REQ-35: iomap_page_mkwrite(vmf, ops, private):
- iter = {inode = file_inode(vma.vm_file), flags = IOMAP_WRITE | IOMAP_FAULT, private}.
- folio = page_folio(vmf.page); folio_lock; folio_mkwrite_check_truncate.
- iter.pos = folio_pos; iter.len = check_truncate-return.
- while iomap_iter > 0: iomap_folio_mkwrite_iter (bh or iomap path).
- folio_wait_stable; return VM_FAULT_LOCKED.

REQ-36: iomap_folio_mkwrite_iter(iter, folio):
- if iomap.flags & IOMAP_F_BUFFER_HEAD: __block_write_begin_int + block_commit_write.
- else: WARN if !uptodate; folio_mark_dirty.

REQ-37: iomap_writeback_init(inode, folio):
- WARN if nblocks > 1 ∧ !ifs.
- if ifs: WARN if write_bytes_pending != 0; atomic_set(write_bytes_pending, folio_size).

REQ-38: iomap_writeback_range(wpc, folio, pos, rlen, end_pos, *bytes_submitted):
- Loop: ret = wpc.ops.writeback_range(wpc, folio, pos, rlen, end_pos); WARN ret == 0 ∨ ret > rlen; rlen -= ret; pos += ret.
- if wpc.iomap.type != HOLE: *bytes_submitted += ret.

REQ-39: iomap_writeback_handle_eof(folio, inode, *end_pos):
- isize = i_size_read.
- if *end_pos > isize: zero (poff = offset_in_folio(isize); folio_zero_segment(folio, poff, folio_size)); *end_pos = isize.
- Return false if folio entirely past isize.

REQ-40: iomap_writeback_folio(wpc, folio):
- iomap_writeback_init.
- iomap_writeback_handle_eof; return -EIO ∨ skip per-handle.
- Loop iomap_find_dirty_range; for each dirty range: iomap_writeback_range; iomap_clear_range_dirty.
- folio_start_writeback (once); if bytes_submitted == 0: iomap_finish_folio_write(folio_size) (no I/O → finish immediately).
- folio_unlock.

REQ-41: iomap_writepages(wpc):
- folio_batch iter; writeback_iter loop: iomap_writeback_folio.
- Returns error from any folio; wbc.nr_to_write decremented.

REQ-42: iomap_finish_folio_write(inode, folio, len):
- WARN if nblocks > 1 ∧ !ifs.
- WARN if ifs ∧ write_bytes_pending ≤ 0.
- if !ifs ∨ atomic_sub_and_test(len, &write_bytes_pending): folio_end_writeback.

REQ-43: IOMAP_F_* flags (returned by FS in iomap.flags):
- F_NEW (0x01): newly allocated blocks — caller may need to zero.
- F_DIRTY (0x02): inode has uncommitted metadata.
- F_SHARED (0x04): blocks are shared (reflink) — write needs COW.
- F_MERGED (0x08): iomap is the merge of multiple extents.
- F_BUFFER_HEAD (0x10, if CONFIG_BUFFER_HEAD): use bh path.
- F_XATTR (0x20): mapping is for an xattr extent.
- F_BOUNDARY (0x40): I/O completion must be ordered (zonefs).
- F_ANON_WRITE (0x80): no target block addr (write-redirect).
- F_ATOMIC_BIO (0x100): submit as atomic torn-write-protected bio.
- F_INTEGRITY (0x200, if CONFIG_FS_VERITY / DIX): integrity metadata handled.
- F_PRIVATE (0x1000): FS-private use.
- F_FOLIO_BATCH (0x2000): iter.fbatch carries pre-selected dirty folios.
- F_SIZE_CHANGED (0x4000): i_size updated during write_iter → iomap_end must record.
- F_STALE (0x8000): iomap invalid — caller must revalidate (XFS extent change).

REQ-44: IOMAP_* iter.flags (passed by caller):
- WRITE (1<<0): writing.
- ZERO (1<<1): zeroing (may skip pure holes).
- REPORT (1<<2): FIEMAP-style report.
- FAULT (1<<3): page-fault path.
- DIRECT (1<<4): direct-I/O caller.
- NOWAIT (1<<5): non-blocking; return -EAGAIN on contention.
- OVERWRITE_ONLY (1<<6): no allocations.
- UNSHARE (1<<7): reflink unshare.
- DAX (1<<8): DAX path.
- ATOMIC (1<<9): atomic-write request.
- DONTCACHE (1<<10): folio_dontcache hint.

REQ-45: iomap_iter() — generic iterator step:
- if iter.status < 0: call ops.iomap_end (if present) to unwind; return iter.status.
- if first call: ops.iomap_begin(inode, pos, len, flags, &iter.iomap, &iter.srcmap); return 1 if non-empty iomap.
- if iter.status == 0 (caller finished step): advance pos by iomap_length consumed; call ops.iomap_end; if more left: re-iomap_begin; else return 0.

REQ-46: iomap_iter_srcmap(iter):
- Returns iter.srcmap if srcmap.type ≠ HOLE (COW source); else iter.iomap.

REQ-47: iomap_iter_advance(iter, count) / _advance_full:
- iter.pos += count; iter.len -= count; iter.iomap.offset += count; iter.iomap.length -= count.
- _advance_full = advance by remaining iomap_length.

## Acceptance Criteria

- [ ] AC-1: `iomap_file_buffered_write`: total bytes returned == iter.pos - iocb.ki_pos; iocb.ki_pos updated; partial write returns positive count.
- [ ] AC-2: `iomap_read_folio` over a hole: zeros folio, marks uptodate, no bio submitted.
- [ ] AC-3: `iomap_readahead`: for-each readahead folio, ops.read_folio_range called exactly once per mapped sub-range; ctx.cur_folio drained at end.
- [ ] AC-4: `iomap_dirty_folio`: ifs allocated if nblocks > 1; dirty bitmap fully set; filemap_dirty_folio returns true.
- [ ] AC-5: `iomap_release_folio` on dirty folio: returns false; ifs retained.
- [ ] AC-6: `iomap_invalidate_folio` (whole folio): folio_cancel_dirty + ifs_free; WARN if writeback in flight.
- [ ] AC-7: `iomap_write_begin` covering full folio: skips per-block ifs alloc; returns 0.
- [ ] AC-8: `__iomap_write_begin` partial overlap with hole: zeroes around [from, to]; no synchronous read.
- [ ] AC-9: `iomap_write_end` short-copy: returns false; iomap_write_iter retries with smaller chunk.
- [ ] AC-10: `iomap_zero_range` over IOMAP_HOLE: no folio touched (pure hole skipped); *did_zero unchanged.
- [ ] AC-11: `iomap_truncate_page` on aligned pos: returns 0 immediately.
- [ ] AC-12: `iomap_writeback_folio`: folio_end_writeback called exactly once after all bytes_submitted complete.
- [ ] AC-13: `iomap_finish_folio_write`: atomic_sub_and_test → folio_end_writeback only on last byte.
- [ ] AC-14: `iomap_page_mkwrite`: returns VM_FAULT_LOCKED; folio_wait_stable observed.
- [ ] AC-15: IOMAP_F_BUFFER_HEAD bridge: iomap_write_begin uses __block_write_begin_int; iomap_write_end uses block_write_end; ifs not allocated for bh path.

## Architecture

```
struct Iomap {
  offset: u64,
  length: u64,
  ty: u8,                       // IOMAP_HOLE | DELALLOC | MAPPED | UNWRITTEN | INLINE
  flags: u16,                   // IOMAP_F_*
  addr: u64,                    // device addr or IOMAP_NULL_ADDR
  bdev: *BlockDevice,
  dax_dev: *DaxDevice,
  inline_data: *u8,
  validity_cookie: u64,         // FS revalidation token
}

struct IomapIter {
  inode: *Inode,
  pos: u64,
  len: u64,
  status: i32,
  flags: u32,                   // IOMAP_WRITE | _ZERO | _FAULT | ...
  iomap: Iomap,
  srcmap: Iomap,
  private: *mut (),
  fbatch: Option<*FolioBatch>,
}

struct IomapFolioState {           // folio.private when nblocks > 1
  state_lock: SpinLock<()>,
  read_bytes_pending: u32,
  write_bytes_pending: AtomicU32,
  state: [usize; BITS_TO_LONGS(2 * nblocks)],
  //  [0 .. nblocks)         → per-block uptodate
  //  [nblocks .. 2*nblocks) → per-block dirty
}
```

`Iomap::file_buffered_write(iocb, ii, ops, write_ops, private) -> ssize_t`:
1. iter = IomapIter::new(iocb.ki_filp.f_mapping.host, iocb.ki_pos, iov_iter_count(ii), IOMAP_WRITE).
2. iter.flags |= IOMAP_NOWAIT if IOCB_NOWAIT; |= IOMAP_DONTCACHE if IOCB_DONTCACHE.
3. while ret = iomap_iter(&iter, ops) > 0: iter.status = Iomap::write_iter(&iter, ii, write_ops).
4. if iter.pos == iocb.ki_pos: return ret.
5. ret = iter.pos - iocb.ki_pos; iocb.ki_pos = iter.pos; return ret.

`Iomap::write_iter(iter, ii, write_ops) -> Result<usize, i32>`:
1. chunk = mapping_max_folio_size(mapping); bdp_flags = (NOWAIT ? BDP_ASYNC : 0).
2. loop:
   - bytes = iov_iter_count(ii).
   - balance_dirty_pages_ratelimited_flags(mapping, bdp_flags) → break on err.
   - bytes = min(chunk - (pos & chunk-1), iomap_length(iter), bytes).
   - if fault_in_iov_iter_readable(ii, bytes) == bytes: break -EFAULT.
   - status = Iomap::write_begin(iter, write_ops, &folio, &offset, &bytes); on err: iomap_write_failed; break.
   - if iter.iomap.flags & IOMAP_F_STALE: break.
   - if mapping_writably_mapped(mapping): flush_dcache_folio.
   - copied = copy_folio_from_iter_atomic(folio, offset, bytes, ii).
   - written = Iomap::write_end(iter, bytes, copied, folio) ? copied : 0.
   - if pos + written > i_size: i_size_write; iomap.flags |= IOMAP_F_SIZE_CHANGED.
   - __iomap_put_folio(iter, write_ops, written, folio).
   - if old_size < pos: pagecache_isize_extended.
   - cond_resched.
   - if written == 0: iomap_write_failed; iov_iter_revert(copied); halve chunk if > PAGE_SIZE; if copied: retry; else break.
   - else: total_written += written; iomap_iter_advance(iter, written).

`Iomap::write_begin(iter, write_ops, &folio, &offset, &bytes) -> Result<()>`:
1. folio = __iomap_get_folio(iter, write_ops, pos, len).
2. iomap_trim_folio_range(iter, folio, offset, bytes).
3. if iter.iomap.flags & IOMAP_F_BUFFER_HEAD: __block_write_begin_int(folio, pos, len, NULL, &iter.iomap).
4. elif srcmap.ty == IOMAP_INLINE: iomap_write_begin_inline.
5. else: __iomap_write_begin(iter, write_ops, len, folio).

`Iomap::write_end_inner(inode, pos, len, copied, folio) -> bool`:
1. if copied < len ∧ !folio_test_uptodate: return false.
2. iomap_set_range_uptodate(folio, offset_in_folio(folio, pos), copied).
3. iomap_set_range_dirty(folio, offset_in_folio(folio, pos), copied).
4. return true.

`Iomap::read_folio(ops, ctx, private)`:
1. iter = IomapIter::new(ctx.cur_folio.mapping.host, folio_pos(ctx.cur_folio), folio_size(ctx.cur_folio), 0); iter.private = private.
2. while iomap_iter > 0: iter.status = Iomap::read_folio_iter(iter, ctx, &bytes_submitted).
3. if ctx.read_ctx ∧ ctx.ops.submit_read: ctx.ops.submit_read(iter, ctx).
4. if ctx.cur_folio: Iomap::read_end(ctx.cur_folio, bytes_submitted).

`Iomap::dirty_folio(mapping, folio) -> bool`:
1. ifs_alloc(mapping.host, folio, 0).
2. iomap_set_range_dirty(folio, 0, folio_size(folio)).
3. return filemap_dirty_folio(mapping, folio).

`Iomap::release_folio(folio, gfp) -> bool`:
1. if folio_test_dirty(folio): return false.
2. ifs_free(folio); return true.

`Iomap::invalidate_folio(folio, offset, len)`:
1. if (offset == 0 ∧ len == folio_size(folio)):
   - WARN if folio_test_writeback.
   - folio_cancel_dirty(folio); ifs_free(folio).

`Iomap::writeback_folio(wpc, folio) -> Result<()>`:
1. iomap_writeback_init(inode, folio).
2. end_pos = folio_pos(folio) + folio_size(folio); if !iomap_writeback_handle_eof(folio, inode, &end_pos): folio_unlock; return Ok.
3. folio_start_writeback(folio).
4. loop iomap_find_dirty_range(folio, &range_start, end_pos):
   - iomap_writeback_range(wpc, folio, range_start, rlen, end_pos, &bytes_submitted).
   - iomap_clear_range_dirty(folio, off, rlen).
5. folio_unlock.
6. if bytes_submitted == 0: iomap_finish_folio_write(inode, folio, folio_size(folio)).

`Iomap::zero_range(inode, pos, len, did_zero, ops, write_ops, private) -> Result<()>`:
1. iter = IomapIter::new(inode, pos, len, IOMAP_ZERO); iter.fbatch = &fbatch.
2. range_dirty = filemap_range_needs_writeback(mapping, pos, pos+len-1).
3. loop iomap_iter:
   - if !F_FOLIO_BATCH ∧ (srcmap.ty == HOLE ∨ UNWRITTEN):
     - if range_dirty ∧ UNWRITTEN: range_dirty = false; status = iomap_zero_iter_flush_and_stale.
     - else: status = iomap_iter_advance_full.
     - continue.
   - status = Iomap::zero_iter(iter, did_zero, write_ops).

`Iomap::page_mkwrite(vmf, ops, private) -> vm_fault_t`:
1. iter = IomapIter::new(file_inode(vmf.vma.vm_file), 0, 0, IOMAP_WRITE | IOMAP_FAULT); iter.private = private.
2. folio = page_folio(vmf.page); folio_lock.
3. ret = folio_mkwrite_check_truncate(folio, iter.inode); if < 0: goto unlock.
4. iter.pos = folio_pos(folio); iter.len = ret.
5. while iomap_iter > 0: iter.status = Iomap::folio_mkwrite_iter(iter, folio).
6. if ret < 0: goto unlock.
7. folio_wait_stable(folio); return VM_FAULT_LOCKED.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `ifs_state_lock_held_for_bitmap_mutation` | INVARIANT | per-ifs_set/clear_range_(uptodate|dirty): ifs.state_lock held. |
| `ifs_size_matches_blocks_per_folio` | INVARIANT | per-ifs: state[] sized BITS_TO_LONGS(2 * nblocks). |
| `read_bytes_pending_balanced` | INVARIANT | per-folio: sum of iomap_read_folio_iter submits == sum of iomap_finish_folio_read drains. |
| `write_bytes_pending_balanced` | INVARIANT | per-folio: writeback_init primes to folio_size; finish_folio_write drains exactly to zero before folio_end_writeback. |
| `release_refuses_dirty` | INVARIANT | per-iomap_release_folio: folio_test_dirty ⟹ return false. |
| `invalidate_full_clears_dirty` | INVARIANT | per-iomap_invalidate_folio: full-folio ⟹ folio_cancel_dirty + ifs_free. |
| `iter_advance_monotonic` | INVARIANT | per-iomap_iter_advance: iter.pos non-decreasing; iter.len non-increasing. |
| `inline_size_bound` | INVARIANT | per-iomap_read_inline_data: BUG if iomap.length > PAGE_SIZE - inline_data_offset. |
| `nowait_returns_eagain_on_alloc` | INVARIANT | per-__iomap_write_begin: NOWAIT ∧ !ifs ∧ nblocks>1 ⟹ -EAGAIN. |
| `write_end_short_copy_returns_false` | INVARIANT | per-__iomap_write_end: copied<len ∧ !uptodate ⟹ false. |
| `writeback_handle_eof_clamps_end` | INVARIANT | per-writeback_handle_eof: *end_pos ≤ i_size after call. |
| `iter_status_propagated_to_iomap_end` | INVARIANT | per-iomap_iter: ops.iomap_end called with iter.status. |
| `iomap_f_stale_aborts_iter` | INVARIANT | per-write_iter: IOMAP_F_STALE ⟹ loop break and revalidate. |

### Layer 2: TLA+

`fs/iomap-buffered.tla`:
- States per-folio: NoIfs → IfsAllocated → SomeBlocksDirty/Uptodate → AllDirty → Writeback → Clean → Freed.
- Per-write loop: Begin → CopyFromUser → End → AdvanceOrRetry.
- Per-read loop: Begin → SubmitOrZero → End.
- Per-writeback: Init → ForEachDirtyRange(Submit) → Drain(write_bytes_pending → 0) → EndWriteback.
- Properties:
  - `safety_ifs_alloc_only_when_needed` — nblocks > 1 ⟹ ifs allocated; nblocks == 1 ⟹ ifs absent (folio-level bits authoritative).
  - `safety_uptodate_bitmap_monotone_during_read` — per-folio: uptodate bits only set, never cleared, while a read is pending.
  - `safety_dirty_bitmap_cleared_before_writeback_end` — per-folio: every block submitted has its dirty bit cleared before folio_end_writeback.
  - `safety_release_preserves_dirty` — per-folio: iomap_release_folio refuses when any dirty bit is set.
  - `safety_unshare_requires_read` — per-IOMAP_UNSHARE: __iomap_write_begin reads any non-uptodate sub-range (no zero shortcut).
  - `safety_write_end_size_changed_recorded` — per-iomap_iter: F_SIZE_CHANGED implies iomap_end observes new size.
  - `liveness_pending_io_completes` — per-folio: read/write_bytes_pending eventually drains; folio_end_(read|writeback) fires.
  - `liveness_iter_advances_or_aborts` — per-iomap_iter: each call either advances iter.pos or returns ≤ 0.
  - `liveness_stale_iomap_revalidates` — per-F_STALE: caller breaks; outer iomap_iter re-invokes iomap_begin.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Iomap::file_buffered_write` post: ret > 0 ⟹ iocb.ki_pos advanced by ret | `Iomap::file_buffered_write` |
| `Iomap::write_iter` post: total_written ≤ original iov_iter_count | `Iomap::write_iter` |
| `Iomap::write_begin` post: folio locked ∧ FGP_WRITEBEGIN semantics | `Iomap::write_begin` |
| `Iomap::write_end_inner` post: copied ≤ len ∧ (return ⟹ uptodate ∧ dirty over [pos, pos+copied)) | `Iomap::write_end_inner` |
| `Iomap::read_folio` post: folio_uptodate ∨ folio_end_read(false) | `Iomap::read_folio` |
| `Iomap::readahead` post: every readahead folio either submitted or skipped (hole→zero) | `Iomap::readahead` |
| `Iomap::dirty_folio` post: ifs present (nblocks>1) ∧ dirty bitmap fully set ∧ filemap_dirty_folio returned | `Iomap::dirty_folio` |
| `Iomap::release_folio` post: success ⟹ folio.private == NULL ∧ !folio_test_dirty pre | `Iomap::release_folio` |
| `Iomap::invalidate_folio` post: full-folio ⟹ !folio_test_dirty post | `Iomap::invalidate_folio` |
| `Iomap::zero_range` post: every (HOLE,non-folio) skipped; (UNWRITTEN+range_dirty) flushed; rest zeroed | `Iomap::zero_range` |
| `Iomap::truncate_page` post: aligned pos ⟹ no work; misaligned ⟹ tail zeroed | `Iomap::truncate_page` |
| `Iomap::writeback_folio` post: all dirty bits cleared ∧ folio_start_writeback called exactly once | `Iomap::writeback_folio` |
| `Iomap::finish_folio_write` post: atomic_sub_and_test true ⟹ folio_end_writeback | `Iomap::finish_folio_write` |
| `Iomap::page_mkwrite` post: success ⟹ VM_FAULT_LOCKED ∧ folio stable | `Iomap::page_mkwrite` |
| `Ifs::alloc` post: nblocks ≤ 1 ⟹ NULL; else state_lock initialised ∧ bitmaps reflect folio_test_(uptodate|dirty) | `Ifs::alloc` |

### Layer 4: Verus/Creusot functional

`Per-(inode, pos, len) iomap_iter → ops.iomap_begin → write_iter / read_folio_iter / zero_iter → ops.iomap_end` semantic equivalence to upstream: per-Documentation/filesystems/iomap/index.rst and per-`xfstests` generic group (010, 074, 091, 127, 285, 286, 466) passing on Rookery-iomap-backed ext4 / xfs / gfs2.

Per-folio invariant: ifs uptodate bitmap matches authoritative folio_test_uptodate when fully set; ifs dirty bitmap matches authoritative folio_test_dirty when fully set; sub-folio partial state captured exclusively by ifs.

## Hardening

(Inherits row-1 features from `fs/00-overview.md` § Hardening.)

iomap buffered-I/O reinforcement:

- **Per-ifs state[] flex-array sized BITS_TO_LONGS(2*nblocks)** — defense against per-out-of-bounds bitmap access on large folios.
- **Per-ifs state_lock around all bitmap mutations** — defense against per-sub-folio race between read-end and dirty-set.
- **Per-read read_bytes_pending counter + XOR folio_end_read** — defense against per-double-folio-uptodate vote on multi-bio read completion.
- **Per-write write_bytes_pending primed to folio_size + atomic_sub_and_test drain** — defense against per-premature folio_end_writeback.
- **Per-iomap_release_folio refuses dirty** — defense against per-data-loss on reclaim of partially-dirty folio.
- **Per-iomap_invalidate_folio full-folio guard (WARN if writeback)** — defense against per-invalidate-during-IO race.
- **Per-NOWAIT ifs-alloc returns -EAGAIN** — defense against per-NOWAIT-deadlock.
- **Per-IOMAP_F_STALE revalidation** — defense against per-stale-extent write to wrong block.
- **Per-iomap_write_failed truncate_pagecache_range beyond i_size** — defense against per-stale-cached-page after failed write.
- **Per-iomap_write_delalloc_release** — defense against per-leaked-delalloc-reservation on write error.
- **Per-iomap_writeback_handle_eof clamps end_pos and zeroes tail** — defense against per-uninitialised-data-past-EOF leak.
- **Per-iomap_block_needs_zeroing includes past-EOF** — defense against per-stale-disk-content read on extended file.
- **Per-IOMAP_UNSHARE forbids zero-around (must read)** — defense against per-reflink-COW data loss.
- **Per-IOMAP_F_BUFFER_HEAD bridge isolates bh path** — defense against per-mixed-state ifs+bh corruption.
- **Per-folio_wait_stable in page_mkwrite** — defense against per-write-vs-writeback torn page.
- **Per-balance_dirty_pages_ratelimited in write_iter** — defense against per-runaway-dirty-pages memory pressure.
- **Per-flush_dcache_folio on mapping_writably_mapped** — defense against per-aliased-mapping stale dcache on architectures w/ VIVT.
- **Per-fault_in_iov_iter_readable before copy_folio_from_iter_atomic** — defense against per-page-fault-under-folio-lock deadlock.

## Grsecurity/PaX-style Reinforcement

The iomap buffered-I/O core is the user/kernel data boundary for ext4/xfs/gfs2/zonefs/btrfs reflink; any iov_iter mismatch or folio-state-poisoning here turns into a data-integrity or info-leak bug. The following PaX/grsecurity-style controls augment the in-tree iomap_iter and folio_state defenses above.

- **PAX_USERCOPY** — `iomap_folio_state` and the per-iter scratch buffers declare zero-byte usercopy whitelists; every `copy_folio_to_iter` / `copy_folio_from_iter_atomic` is bounded against `folio_size()` and refuses partial-folio over-runs.
- **PAX_KERNEXEC** — `iomap_*_ops` dispatch and the per-folio block-mapping callbacks live in `__ro_after_init` text; W^X is enforced over the per-fs override tables.
- **PAX_RANDKSTACK** — `iomap_write_iter`, `iomap_read_iter`, and the page-fault-driven `iomap_page_mkwrite` enter on randomized stacks so locals in `iomap_iter` cannot be groomed.
- **PAX_REFCOUNT** — `folio->_refcount` and the per-folio `iomap_folio_state->read_bytes_pending` / `write_bytes_pending` are saturating atomics that BUG on wrap.
- **PAX_MEMORY_SANITIZE** — freed `iomap_folio_state` and short-read tail pads are poisoned so residue between read and copy_to_iter cannot leak prior page contents.
- **PAX_UDEREF** — `iomap_iter` validates that the iov_iter's user pointers reside in user-space before every fault-in and copy.
- **PAX_RAP / kCFI** — `iomap_ops->iomap_begin` and `->iomap_end` are typed-call-site-verified; a corrupted ops pointer cannot redirect into a gadget.
- **GRKERNSEC_HIDESYM** — `iomap_*`, `iomap_folio_state`, and per-fs `iomap_ops` instances are stripped from `/proc/kallsyms` for non-CAP_SYSLOG callers.
- **GRKERNSEC_DMESG** — iomap warn-prints (mapping mismatch, holepunch race, folio state invariant) are gated behind CAP_SYSLOG.
- **iomap_iter PAX_USERCOPY** — every call into iov_iter from `iomap_write_iter`/`iomap_dio_iter` is wrapped in PAX_USERCOPY checks that re-derive the destination folio offset and length and refuse mismatched iters.
- **iomap_folio_state validation** — on `iomap_finish_folio_read` / `write`, the per-folio uptodate and dirty bitmaps are reverified against the iomap extent's block count; any mismatch fences the I/O with `EIO` and marks the inode `IOMAP_INVALID`.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `fs/iomap/direct-io.c` direct-I/O path (covered in `fs/iomap-direct.md` Tier-3)
- `fs/iomap/fiemap.c` extent reporting (covered in `fs/iomap-fiemap.md` Tier-3)
- `fs/iomap/seek.c` SEEK_HOLE / SEEK_DATA (covered in `fs/iomap-seek.md` Tier-3)
- `fs/iomap/swapfile.c` swap-on-iomap (covered in `fs/iomap-swapfile.md` Tier-3)
- `fs/iomap/ioend.c` writeback completion machinery (covered in `fs/iomap-ioend.md` Tier-3)
- `fs/buffer.c` legacy bh path (covered in `fs/buffer.md` Tier-3)
- `fs/xfs/`, `fs/gfs2/`, `fs/zonefs/`, `fs/ext4/` filesystem-specific iomap_ops (covered in per-FS Tier-3s)
- `mm/filemap.c` / `mm/readahead.c` folio cache + readahead control (covered in `mm/filemap.md` / `mm/readahead.md` Tier-3s)
- Implementation code
