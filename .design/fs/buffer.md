# Tier-3: fs/buffer.c — Buffer-head (bh) subsystem

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: fs/00-overview.md
upstream-paths:
  - fs/buffer.c (~3078 lines)
  - include/linux/buffer_head.h
  - include/uapi/linux/fs.h (BH_* bits)
-->

## Summary

The **buffer-head** subsystem is the legacy block-aware abstraction layered on top of the page cache: a `struct buffer_head` (bh) represents one filesystem block within a folio, carries the (bdev, block-number, size) tuple needed to issue a bio, plus per-block state bits (Uptodate, Dirty, Locked, Mapped, New, Async_Read/Write, Delay, Unwritten, Boundary, Meta, Prio, ...). Per-folio bh chain is a circular list rooted at `folio->private`; each bh covers `bh->b_size` bytes at offset `bh_offset(bh)`. Per-CPU **bh_lru** (16 entries) caches recently-looked-up bhs to short-circuit page-cache walks. Per-`__getblk_slow` grows the folio + buffers for a (bdev, block) on miss; per-`__find_get_block` / `_nonatomic` does the cache lookup; per-`__bread` reads-if-not-uptodate. Per-`mark_buffer_dirty` propagates dirtiness up to the folio + inode dirty lists; per-`block_read_full_folio` / `__block_write_full_folio` drive the legacy aops; per-`submit_bh` builds a single-bh bio for the block layer; per-`end_buffer_async_read` / `_write` finalises folio uptodate / writeback state once the last bh in the chain completes. Used by: ext2, ext4 (jbd2 journal), reiserfs (deprecated), ntfs3, exfat, fat, hfsplus, jfs, udf, isofs, minix, sysv, qnx4/6, omfs, ufs, freevxfs, jffs2-bdev, block-device mapping itself.

This Tier-3 covers `fs/buffer.c` (~3078 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct buffer_head` | per-block descriptor over a folio range | `BufferHead` |
| `alloc_buffer_head()` | per-bh slab allocation + per-cpu accounting | `Bh::alloc` |
| `free_buffer_head()` | per-bh slab free + per-cpu accounting | `Bh::free` |
| `__lock_buffer()` / `unlock_buffer()` | per-bh BH_Lock bit-spin lock | `Bh::lock` / `unlock` |
| `__wait_on_buffer()` | per-bh wait on BH_Lock clear | `Bh::wait` |
| `end_buffer_read_sync()` | per-bh sync read end_io | `Bh::end_read_sync` |
| `end_buffer_write_sync()` | per-bh sync write end_io | `Bh::end_write_sync` |
| `end_buffer_async_read()` | per-bh async read end_io (folio uptodate) | `Bh::end_async_read` |
| `end_buffer_async_write()` | per-bh async write end_io (folio writeback) | `Bh::end_async_write` |
| `mark_buffer_dirty()` | per-bh BH_Dirty + folio dirty + inode dirty | `Bh::mark_dirty` |
| `mark_buffer_async_write()` / `_write_endio` | per-bh BH_Async_Write + end_io ptr | `Bh::mark_async_write` |
| `mark_buffer_write_io_error()` | per-bh BH_Write_EIO + mapping_set_error | `Bh::mark_write_io_error` |
| `__find_get_block()` / `_nonatomic` | per-(bdev,block) page-cache lookup + LRU | `Bh::find_get_block` |
| `__find_get_block_slow()` | per-folio walk + bh-chain match | `Bh::find_get_block_slow` |
| `__getblk_slow()` / `bdev_getblk()` | per-allocate-on-miss + retry-find | `Bh::getblk_slow` / `bdev_getblk` |
| `__bread_gfp()` / `__bread_slow()` | per-getblk + read-if-not-uptodate | `Bh::bread` |
| `__breadahead()` | per-readahead-one-block | `Bh::breadahead` |
| `__brelse()` / `__bforget()` | per-put_bh / per-discard-dirty | `Bh::release` / `forget` |
| `submit_bh()` / `submit_bh_wbc()` | per-bh single-bio submit | `Bh::submit` |
| `write_dirty_buffer()` | per-bh test+clear+submit-write | `Bh::write_dirty` |
| `sync_dirty_buffer()` / `__sync_dirty_buffer()` | per-bh write + wait | `Bh::sync_dirty` |
| `__bh_read()` / `__bh_read_batch()` | per-bh / per-batch read submit | `Bh::read` / `read_batch` |
| `bh_uptodate_or_lock()` | per-bh fast-path uptodate-or-acquire-lock | `Bh::uptodate_or_lock` |
| `block_read_full_folio()` | per-folio aops read driver | `Bh::block_read_full_folio` |
| `__block_write_full_folio()` / `block_write_full_folio` | per-folio aops writeback driver | `Bh::block_write_full_folio` |
| `__block_write_begin()` / `_int` | per-folio write-begin (zero/read holes) | `Bh::block_write_begin` |
| `block_write_end()` / `generic_write_end()` | per-folio write-end (dirty + uptodate) | `Bh::block_write_end` |
| `block_commit_write()` | per-folio dirty all touched bhs | `Bh::block_commit_write` |
| `block_is_partially_uptodate()` | per-folio range uptodate query | `Bh::is_partially_uptodate` |
| `block_invalidate_folio()` | per-folio bh-chain invalidate / drop | `Bh::invalidate_folio` |
| `block_dirty_folio()` | per-folio dirty all bhs + filemap_dirty_folio | `Bh::dirty_folio` |
| `block_truncate_page()` | per-folio tail-zero on truncate | `Bh::truncate_page` |
| `block_page_mkwrite()` | per-fault mkwrite via bh write-begin | `Bh::page_mkwrite` |
| `generic_cont_expand_simple()` / `cont_write_begin()` | per-expand into hole (no-holes FS) | `Bh::cont_write_begin` |
| `generic_block_bmap()` | per-bmap via get_block | `Bh::generic_bmap` |
| `folio_alloc_buffers()` / `alloc_page_buffers()` / `create_empty_buffers()` | per-folio bh-chain init | `Bh::folio_alloc_buffers` |
| `folio_set_bh()` / `folio_create_buffers()` | per-attach bh to folio | `Bh::folio_set_bh` |
| `folio_zero_new_buffers()` | per-folio zero BH_New bhs | `Bh::folio_zero_new` |
| `try_to_free_buffers()` / `drop_buffers()` | per-folio reclaim path | `Bh::try_to_free` |
| `clean_bdev_aliases()` | per-bdev invalidate aliased bhs | `Bh::clean_bdev_aliases` |
| `clean_bdev_bh_alias()` | per-bh alias-invalidate (one block) | `Bh::clean_bdev_bh_alias` |
| `iomap_to_bh()` | per-bh fill from `struct iomap` | `Bh::iomap_to_bh` |
| `bh_lru` / `lookup_bh_lru()` / `bh_lru_install()` | per-cpu 16-entry MRU cache | `Bh::Lru` |
| `invalidate_bh_lrus()` / `_cpu` | per-cpu LRU flush | `Bh::Lru::invalidate` |
| `mmb_init()` / `_sync` / `_fsync` / `_invalidate` / `_mark_buffer_dirty` | per-mapping metadata-bh list | `Bh::MappingMetadata` |
| `write_boundary_block()` | per-bh BH_Boundary forced flush | `Bh::write_boundary_block` |
| `buffer_check_dirty_writeback()` | per-folio MM reclaim hint | `Bh::check_dirty_writeback` |
| `touch_buffer()` | per-bh mark-accessed | `Bh::touch` |
| `buffer_heads_over_limit` | global pressure flag | `Bh::heads_over_limit` |
| `bh_cachep` | per-bh slab cache | `Bh::cache` |
| `max_buffer_heads` | per-init cap (10% of ZONE_NORMAL) | `Bh::max_heads` |
| `buffer_init()` | per-boot subsystem init | `Bh::init` |

## Compatibility contract

REQ-1: struct buffer_head — fixed UAPI-adjacent layout:
- b_state: per-bh atomic bit field (BH_Uptodate, BH_Dirty, BH_Lock, BH_Req, BH_Mapped, BH_New, BH_Async_Read, BH_Async_Write, BH_Delay, BH_Boundary, BH_Write_EIO, BH_Unwritten, BH_Quiet, BH_Meta, BH_Prio, BH_Defer_Completion, BH_Migrate, ...).
- b_this_page: per-folio circular-list next.
- b_folio: per-owning folio.
- b_blocknr: per-bh sector_t block number on b_bdev.
- b_size: per-bh size (multiple of bdev logical block size).
- b_data: per-bh kmap'd virtual pointer (folio data + bh_offset).
- b_bdev: per-bh `struct block_device *`.
- b_end_io: per-bh completion callback.
- b_private: per-bh user-data (jbd2 stashes journal-head here).
- b_assoc_buffers: per-bh list-head for mapping-metadata-bh list.
- b_mmb: per-bh back-pointer to owning `mapping_metadata_bhs`.
- b_count: per-bh atomic refcount.
- b_uptodate_lock: per-bh spinlock guarding end_io races on multi-bh folios.

REQ-2: __lock_buffer / unlock_buffer / __wait_on_buffer:
- BH_Lock is a bit-spin-lock — acquired via test_and_set_bit + sleep on contention.
- unlock_buffer: smp_mb__before_atomic; clear_bit; smp_mb__after_atomic; wake_up_bit(BH_Lock).
- __wait_on_buffer: wait_on_bit(BH_Lock) with TASK_UNINTERRUPTIBLE.
- All async I/O paths acquire BH_Lock before submit and release in end_io.

REQ-3: end_buffer_read_sync(bh, uptodate):
- if uptodate: set_buffer_uptodate.
- unlock_buffer.
- put_bh.

REQ-4: end_buffer_write_sync(bh, uptodate):
- if !uptodate: mark_buffer_write_io_error; print failure (rate-limited unless BH_Quiet).
- clear_buffer_uptodate if !uptodate, else leave as-is.
- unlock_buffer.
- put_bh.

REQ-5: end_buffer_async_read(bh, uptodate):
- Per-bh: set/clear BH_Uptodate; clear BH_Async_Read; unlock_buffer.
- Per-folio: traverse bh-chain under bh.b_uptodate_lock; if all bhs uptodate ∧ no async_read pending: folio_end_read(folio, true); else if any !uptodate: folio_end_read(folio, false).

REQ-6: end_buffer_async_write(bh, uptodate):
- Per-bh: if !uptodate: mark_buffer_write_io_error; mapping_set_error; clear BH_Uptodate.
- clear BH_Async_Write; unlock_buffer.
- Per-folio: traverse bh-chain under bh.b_uptodate_lock; if no async_write pending: folio_end_writeback(folio).

REQ-7: mark_buffer_dirty(bh):
- WARN if !buffer_uptodate.
- Fast-path: if BH_Dirty already set: smp_mb + re-check + early-return.
- test_set_buffer_dirty: if was-clear:
  - folio = bh.b_folio.
  - if !folio_test_set_dirty(folio):
    - mapping = folio.mapping.
    - if mapping: __folio_mark_dirty(folio, mapping, 0).
  - if mapping: __mark_inode_dirty(mapping.host, I_DIRTY_PAGES).

REQ-8: mark_buffer_async_write / _endio(bh, fn):
- bh.b_end_io = fn (or end_buffer_async_write).
- set_buffer_async_write.

REQ-9: __find_get_block_slow(bdev, block, atomic):
- index = block * (bdev.bd_inode.i_blkbits / bd_block_size).
- folio = filemap_get_folio(bdev.bd_mapping, index, FGP_ACCESSED) — atomic-variant uses _ACCESSED-only.
- if !folio: return NULL.
- folio_lock_if-blocking; walk folio_buffers chain matching (b_blocknr == block ∧ b_size == size).
- if hit: get_bh; touch_buffer.
- folio_unlock; folio_put.
- return bh or NULL.

REQ-10: bh_lru per-cpu MRU cache:
- BH_LRU_SIZE = 16 bhs per CPU.
- lookup_bh_lru: linear scan; on hit, rotate to slot 0; return with bh ref already held.
- bh_lru_install: insert at slot 0; evict last; bh_lru_lock = local_irq_disable.
- invalidate_bh_lrus: on_each_cpu_cond(has_bh_in_lru, invalidate_bh_lru).
- invalidate_bh_lrus_cpu: workqueue-context variant under bh_lru_lock.
- CPU hot-unplug: buffer_exit_cpu_dead releases all bhs in dead CPU's LRU.

REQ-11: __find_get_block(bdev, block, size):
- bh = lookup_bh_lru(bdev, block, size).
- if !bh: bh = __find_get_block_slow(bdev, block, size, atomic=true).
- if bh: bh_lru_install; touch_buffer.
- return bh.

REQ-12: __getblk_slow(bdev, block, size, gfp):
- WARN if !IS_ALIGNED(size, bdev_logical_block_size).
- Loop:
  - if !grow_buffers(bdev, block, size, gfp): return NULL.
  - bh = __find_get_block_nonatomic or __find_get_block.
  - if bh: return bh.

REQ-13: bdev_getblk(bdev, block, size, gfp):
- bh = __find_get_block (atomic if !gfp-blocking).
- if !bh: bh = __getblk_slow(bdev, block, size, gfp).
- return bh.

REQ-14: __bread_gfp(bdev, block, size, gfp):
- gfp |= mapping_gfp_constraint(bdev.bd_mapping, ~__GFP_FS) | __GFP_NOFAIL.
- bh = bdev_getblk(bdev, block, size, gfp).
- if bh ∧ !buffer_uptodate(bh): bh = __bread_slow(bh).
- return bh.

REQ-15: __bread_slow(bh):
- lock_buffer.
- if buffer_uptodate: unlock; return bh.
- get_bh; bh.b_end_io = end_buffer_read_sync; submit_bh(REQ_OP_READ, bh).
- wait_on_buffer.
- if buffer_uptodate: return bh.
- brelse; return NULL.

REQ-16: __breadahead(bdev, block, size):
- bh = __getblk(bdev, block, size).
- if buffer_uptodate: brelse; return.
- ll_rw_block-style submit REQ_OP_READ | REQ_RAHEAD; brelse.

REQ-17: __brelse(bh): if b_count > 0: put_bh; else: WARN.
REQ-18: __bforget(bh): clear_buffer_dirty; remove_assoc_queue; __brelse.

REQ-19: submit_bh / submit_bh_wbc(opf, bh, write_hint, wbc):
- BUG if !buffer_locked ∨ !buffer_mapped ∨ !b_end_io ∨ buffer_delay ∨ buffer_unwritten.
- test_set_buffer_req: if was-set ∧ op == WRITE: clear BH_Write_EIO.
- opf |= REQ_META if buffer_meta; |= REQ_PRIO if buffer_prio.
- bio = bio_alloc(bh.b_bdev, 1, opf, GFP_NOIO).
- buffer_set_crypto_ctx if FS_ENCRYPTION (fscrypt).
- bio.bi_iter.bi_sector = bh.b_blocknr * (bh.b_size >> 9).
- bio_add_folio_nofail(bio, bh.b_folio, bh.b_size, bh_offset(bh)).
- bio.bi_end_io = end_bio_bh_io_sync; bio.bi_private = bh.
- guard_bio_eod; wbc bind (wbc_init_bio + wbc_account_cgroup_owner).
- blk_crypto_submit_bio.

REQ-20: end_bio_bh_io_sync(bio):
- if bio_flagged BIO_QUIET: set BH_Quiet on bh.
- bh.b_end_io(bh, !bio.bi_status).
- bio_put.

REQ-21: write_dirty_buffer(bh, op_flags):
- lock_buffer.
- if !test_clear_buffer_dirty: unlock; return.
- bh.b_end_io = end_buffer_write_sync; get_bh.
- submit_bh(REQ_OP_WRITE | op_flags, bh).

REQ-22: __sync_dirty_buffer(bh, op_flags):
- WARN if b_count < 1.
- lock_buffer.
- if test_clear_buffer_dirty:
  - if !buffer_mapped: unlock; return -EIO.
  - get_bh; bh.b_end_io = end_buffer_write_sync.
  - submit_bh(REQ_OP_WRITE | op_flags, bh).
  - wait_on_buffer.
  - if !buffer_uptodate: return -EIO.
- else: unlock.
- return 0.

REQ-23: sync_dirty_buffer(bh) = __sync_dirty_buffer(bh, REQ_SYNC).

REQ-24: __bh_read(bh, op_flags, wait):
- BUG if !buffer_locked.
- get_bh; bh.b_end_io = end_buffer_read_sync; submit_bh(REQ_OP_READ | op_flags, bh).
- if wait: wait_on_buffer; if !uptodate: ret = -EIO.

REQ-25: __bh_read_batch(nr, bhs[], op_flags, force_lock):
- For each bh: skip if uptodate; lock (force or trylock); skip if became uptodate; bh.b_end_io = end_buffer_read_sync; get_bh; submit_bh(REQ_OP_READ | op_flags).

REQ-26: bh_uptodate_or_lock(bh):
- if !buffer_uptodate: lock_buffer; if !buffer_uptodate: return 0; unlock.
- return 1.

REQ-27: block_read_full_folio(folio, get_block):
- limit = i_size_read(inode); FS_VERITY: limit = s_maxbytes.
- head = folio_create_buffers(folio, inode, 0).
- iblock = folio_pos / blocksize; lblock = (limit + bs - 1) / bs.
- For each bh in chain:
  - skip if uptodate.
  - if !mapped: get_block(inode, iblock, bh, 0); if !mapped: folio_zero_range(bh region); set_uptodate; continue.
  - lock_buffer; if uptodate: unlock; continue.
  - mark_buffer_async_read; if prev: submit_bh(prev); prev = bh.
- if fully_mapped: folio_set_mappedtodisk.
- if prev: submit_bh(prev); else: folio_end_read(folio, !page_error).

REQ-28: __block_write_full_folio(inode, folio, get_block, wbc):
- head = folio_create_buffers(folio, inode, BH_Dirty|BH_Uptodate).
- block = folio_pos / bs; last_block = (i_size - 1) / bs.
- Pass-1 (map): for each bh: if block > last_block: clear dirty + set uptodate; else if (!mapped or delay) ∧ dirty: get_block(inode, block, bh, 1); clear delay; if BH_New: clear new + clean_bdev_bh_alias.
- Pass-2 (lock + transition): for each mapped bh:
  - if WB_SYNC_NONE: trylock_buffer or folio_redirty_for_writepage + continue.
  - else: lock_buffer.
  - if test_clear_buffer_dirty: mark_buffer_async_write_endio(bh, end_buffer_async_write).
  - else: unlock_buffer.
- BUG if folio_test_writeback; folio_start_writeback.
- Pass-3 (submit): for each BH_Async_Write bh: submit_bh_wbc(REQ_OP_WRITE | write_flags, bh, write_hint, wbc); nr_underway++.
- folio_unlock.
- if nr_underway == 0: folio_end_writeback.
- recover (on get_block err): re-walk; clear dirty on holes; mapping_set_error; submit remaining; folio_start_writeback then goto done.

REQ-29: block_write_full_folio(folio, wbc, get_block):
- Convenience: folio_zero_segment beyond i_size on tail folio; __block_write_full_folio.

REQ-30: __block_write_begin / _int(folio, pos, len, get_block, iomap):
- head = folio_create_buffers(folio, inode, 0).
- For each bh covering [pos, pos+len):
  - if !mapped: get_block (or iomap_to_bh from caller's iomap); if BH_New: clean_bdev_bh_alias; folio_zero_segments outside [from,to] for new bhs.
  - if !uptodate ∧ (overlap-with-existing): lock_buffer; submit READ; wait.
- Returns 0 on success; on error: folio_zero_new_buffers.

REQ-31: block_write_end(pos, len, copied, folio):
- start = pos & ~PAGE_MASK; end = start + copied.
- For each bh overlapping [start, end): if buffer_new: clear new; set_uptodate; mark_buffer_dirty.
- If short-copy (copied < len): folio_zero_new_buffers(folio, start+copied, start+len).
- Return copied.

REQ-32: generic_write_end(iocb, mapping, pos, len, copied, folio, fsdata):
- copied = block_write_end(pos, len, copied, folio).
- if pos + copied > i_size: i_size_write; mark_inode_dirty.
- folio_unlock; folio_put.
- Return copied.

REQ-33: block_commit_write(folio, from, to):
- For each bh overlapping [from, to): set_uptodate; mark_buffer_dirty.

REQ-34: block_is_partially_uptodate(folio, from, count):
- head = folio_buffers; if !head: return false.
- Walk bhs in [from, from+count); return false on first !uptodate.
- Return true.

REQ-35: block_invalidate_folio(folio, offset, length):
- partial = (offset ∨ offset+length < folio_size).
- Walk bh-chain; for each bh in invalidated range: discard_buffer (clear dirty + mapped + req + uptodate; unlock).
- If !partial: try_to_release_page (try_to_free_buffers).

REQ-36: block_dirty_folio(mapping, folio):
- head = folio_buffers; if head: walk chain set BH_Dirty on each bh.
- filemap_dirty_folio(mapping, folio); return true.

REQ-37: block_truncate_page(mapping, from, get_block):
- index = from >> PAGE_SHIFT; offset = from & (bs-1).
- folio = read_mapping_folio(mapping, index, NULL).
- head = folio_buffers; bh = nth covering offset.
- if !mapped: get_block(inode, iblock, bh, 0).
- if mapped ∧ !uptodate: __bh_read; wait.
- folio_zero_range(folio, offset_in_folio, bs - (offset & (bs-1))).
- mark_buffer_dirty; folio_unlock; folio_put.

REQ-38: block_page_mkwrite(vma, vmf, get_block):
- folio_lock; check folio.mapping == inode.i_mapping; check size.
- __block_write_begin(folio, 0, end, get_block).
- block_commit_write(folio, 0, end).
- folio_wait_stable.

REQ-39: cont_write_begin(iocb, mapping, pos, len, foliop, fsdata, get_block, bytes):
- cont_expand_zero into hole between *bytes and pos.
- block_write_begin(mapping, pos, len, foliop, get_block).

REQ-40: generic_block_bmap(mapping, block, get_block):
- tmp.b_state = 0; tmp.b_blocknr = 0; tmp.b_size = i_blocksize.
- get_block(mapping.host, block, &tmp, 0).
- return tmp.b_blocknr.

REQ-41: folio_alloc_buffers / alloc_page_buffers / create_empty_buffers(folio, size, state):
- Allocate N = folio_size / size bhs via alloc_buffer_head.
- Link into circular b_this_page list; folio_set_bh on each; folio_attach_private(folio, head).
- All bhs start with b_state == state (commonly 0 or BH_Dirty|BH_Uptodate).

REQ-42: folio_create_buffers(folio, inode, state):
- if folio_buffers(folio): return head.
- bs = i_blocksize(inode); create_empty_buffers(folio, bs, state).

REQ-43: folio_zero_new_buffers(folio, from, to):
- For each BH_New bh in [from,to): folio_zero_segment(folio, max(from,bs_start), min(to,bs_end)); set_uptodate; clear new; mark_buffer_dirty.

REQ-44: try_to_free_buffers(folio):
- if folio_test_writeback: return false.
- mapping = folio.mapping; spin_lock if mapping (i_private_lock).
- drop_buffers(folio, &buffers_to_free): walk; fail if any bh buffer_busy (b_count > 0 ∨ Dirty ∨ Lock).
- On success: remove_assoc_queue each bh; folio_detach_private; free chain (free_buffer_head loop).
- Return success.

REQ-45: clean_bdev_aliases(bdev, block, len):
- Walk bdev's mapping; for each cached folio overlapping [block, block+len): for each matching bh: clear_buffer_dirty; clear_buffer_uptodate; unmap.

REQ-46: iomap_to_bh(inode, block, bh, iomap):
- Fill bh.b_blocknr from iomap->addr + (offset within iomap).
- Set BH_Mapped; BH_New if IOMAP_F_NEW; BH_Unwritten if IOMAP_UNWRITTEN; BH_Delay if IOMAP_DELALLOC; BH_Boundary if IOMAP_F_BOUNDARY.

REQ-47: alloc_buffer_head(gfp):
- bh = kmem_cache_zalloc(bh_cachep, gfp).
- INIT_LIST_HEAD(&b_assoc_buffers); spin_lock_init(&b_uptodate_lock).
- preempt_disable; per-cpu bh_accounting.nr++; recalc_bh_state; preempt_enable.

REQ-48: free_buffer_head(bh):
- BUG if !list_empty(&b_assoc_buffers).
- kmem_cache_free(bh_cachep, bh).
- preempt_disable; per-cpu bh_accounting.nr--; recalc_bh_state; preempt_enable.

REQ-49: recalc_bh_state:
- Per-cpu ratelimit (4096 events).
- tot = sum per-cpu nr; buffer_heads_over_limit = (tot > max_buffer_heads).

REQ-50: mapping_metadata_bhs (mmb) — per-mapping metadata-bh list:
- mmb_init / mmb_has_buffers / mmb_sync / mmb_fsync / mmb_fsync_noflush / mmb_invalidate.
- mmb_mark_buffer_dirty: attach bh to mmb.list under mmb.lock.
- mmb_sync: lock_buffer + submit_bh(REQ_OP_WRITE) + wait + collect errors.
- Used by ext4 to track journal indirect-block buffers separate from data.

REQ-51: write_boundary_block(bdev, bblock, size):
- BH_Boundary forces a flush of the previous block's I/O when writing across an indirect-block boundary (extX backwards compat).

REQ-52: buffer_check_dirty_writeback(folio, dirty, writeback):
- Walk folio.buffers; *dirty = any BH_Dirty; *writeback = any BH_Async_Write.

REQ-53: buffer_init():
- bh_cachep = KMEM_CACHE(buffer_head, SLAB_RECLAIM_ACCOUNT|SLAB_PANIC).
- nrpages = nr_free_buffer_pages() * 10 / 100.
- max_buffer_heads = nrpages * (PAGE_SIZE / sizeof(buffer_head)).
- cpuhp_setup_state_nocalls(CPUHP_FS_BUFF_DEAD, "fs/buffer:dead", NULL, buffer_exit_cpu_dead).

REQ-54: Buffer state-bit semantics (BH_*):
- BH_Uptodate: contents match disk.
- BH_Dirty: must be written.
- BH_Lock: bit-spin-lock for I/O ownership.
- BH_Req: at least one I/O request has been issued.
- BH_Mapped: b_blocknr / b_bdev is valid.
- BH_New: just allocated on disk (must zero new in-folio range).
- BH_Async_Read / Async_Write: end_io will run folio-completion callback.
- BH_Delay: delayed-allocation (no disk addr yet).
- BH_Unwritten: extent allocated but not initialized (must be converted on write).
- BH_Boundary: indirect-block boundary — force flush.
- BH_Write_EIO: prior write returned EIO.
- BH_Meta / BH_Prio: hint flags for the block layer (REQ_META, REQ_PRIO).
- BH_Quiet: suppress per-bh I/O-error klog.
- BH_Defer_Completion: ext4 fast-commit defers folio-end to workqueue.
- BH_Migrate: page-migration in progress on owning folio.

## Acceptance Criteria

- [ ] AC-1: `__find_get_block(bdev, block, size)`: returns bh with refcount-elevated; bh installed in per-cpu LRU.
- [ ] AC-2: `__getblk_slow`: creates buffers if absent; retries find_get_block until success or grow failure.
- [ ] AC-3: `__bread`: returns NULL on read failure; returned bh has BH_Uptodate set on success.
- [ ] AC-4: `mark_buffer_dirty`: sets BH_Dirty; sets folio dirty; flags inode I_DIRTY_PAGES.
- [ ] AC-5: `submit_bh`: builds bio of exactly bh.b_size at bh.b_blocknr * (bh.b_size >> 9); attaches end_bio_bh_io_sync.
- [ ] AC-6: `end_buffer_async_read`: folio_end_read called exactly once when last async_read bh completes.
- [ ] AC-7: `end_buffer_async_write`: folio_end_writeback called exactly once when last async_write bh completes.
- [ ] AC-8: `block_read_full_folio`: handles holes by zeroing + set_uptodate without I/O.
- [ ] AC-9: `__block_write_full_folio` WB_SYNC_NONE: trylock failure → folio_redirty_for_writepage (no busy-wait).
- [ ] AC-10: `block_write_end`: short-copy zeroes BH_New region; returns bytes copied.
- [ ] AC-11: `try_to_free_buffers`: fails if any bh has b_count > 0 ∨ BH_Dirty ∨ BH_Lock; succeeds → folio.private cleared.
- [ ] AC-12: `sync_dirty_buffer`: submits write only if test_clear_buffer_dirty; waits; returns -EIO on !uptodate after wait.
- [ ] AC-13: `invalidate_bh_lrus`: post-call has_bh_in_lru returns false on all CPUs.
- [ ] AC-14: `alloc_buffer_head` / `free_buffer_head`: per-cpu bh_accounting.nr balanced; buffer_heads_over_limit toggles past max.
- [ ] AC-15: CPU hot-unplug: buffer_exit_cpu_dead releases dead CPU's LRU and folds accounting.

## Architecture

```
struct BufferHead {
  b_state: AtomicU64,           // BH_* bits
  b_this_page: *BufferHead,     // circular list within folio
  b_folio: *Folio,
  b_blocknr: u64,               // sector_t
  b_size: u32,
  b_data: *u8,                  // kmap'd pointer
  b_bdev: *BlockDevice,
  b_end_io: Option<fn(*BufferHead, bool)>,
  b_private: *mut (),           // jbd2 journal-head, etc
  b_assoc_buffers: ListHead,
  b_mmb: Option<*MappingMetadataBhs>,
  b_count: AtomicU32,
  b_uptodate_lock: SpinLock<()>,
}

struct BhLru { bhs: [Option<*BufferHead>; 16] }   // per-cpu
```

`Bh::find_get_block(bdev, block, size) -> Option<BhRef>`:
1. if bh = Bh::Lru::lookup(bdev, block, size): touch_buffer; return bh.
2. bh = Bh::find_get_block_slow(bdev, block, size, atomic=true).
3. if bh: Bh::Lru::install(bh); touch_buffer.
4. return bh.

`Bh::getblk_slow(bdev, block, size, gfp) -> Option<BhRef>`:
1. WARN if !IS_ALIGNED(size, bdev_logical_block_size).
2. loop:
   - if !Bh::grow_buffers(bdev, block, size, gfp): return None.
   - bh = Bh::find_get_block_(atomic|nonatomic).
   - if bh: return bh.

`Bh::bread(bdev, block, size, gfp) -> Option<BhRef>`:
1. gfp |= mapping_gfp_constraint(bdev.bd_mapping, ~__GFP_FS) | __GFP_NOFAIL.
2. bh = Bh::bdev_getblk(bdev, block, size, gfp).
3. if bh ∧ !buffer_uptodate(bh):
   - lock_buffer; if uptodate: unlock; return bh.
   - get_bh; bh.b_end_io = Bh::end_read_sync; Bh::submit(REQ_OP_READ, bh).
   - wait_on_buffer; if !uptodate: brelse; return None.
4. return bh.

`Bh::mark_dirty(bh)`:
1. WARN if !buffer_uptodate.
2. if buffer_dirty: smp_mb; if buffer_dirty: return.
3. if !test_set_buffer_dirty(bh):
   - folio = bh.b_folio.
   - if !folio_test_set_dirty(folio):
     - mapping = folio.mapping.
     - if mapping: __folio_mark_dirty(folio, mapping, 0).
   - if mapping: __mark_inode_dirty(mapping.host, I_DIRTY_PAGES).

`Bh::submit(opf, bh)`:
1. BUG !buffer_locked ∨ !buffer_mapped ∨ !b_end_io ∨ buffer_delay ∨ buffer_unwritten.
2. if test_set_buffer_req ∧ op == WRITE: clear_buffer_write_io_error.
3. opf |= REQ_META if buffer_meta; |= REQ_PRIO if buffer_prio.
4. bio = bio_alloc(bh.b_bdev, 1, opf, GFP_NOIO).
5. fscrypt_set_bio_crypt_ctx (if FS_ENCRYPTION).
6. bio.bi_iter.bi_sector = bh.b_blocknr * (bh.b_size >> 9).
7. bio_add_folio_nofail(bio, bh.b_folio, bh.b_size, bh_offset(bh)).
8. bio.bi_end_io = end_bio_bh_io_sync; bio.bi_private = bh.
9. guard_bio_eod.
10. if wbc: wbc_init_bio; wbc_account_cgroup_owner.
11. blk_crypto_submit_bio.

`Bh::block_read_full_folio(folio, get_block) -> Result<()>`:
1. inode = folio.mapping.host; limit = i_size_read; FS_VERITY: s_maxbytes.
2. head = folio_create_buffers(folio, inode, 0).
3. iblock = folio_pos / blocksize; lblock = (limit + bs - 1) / bs.
4. for bh in chain:
   - if uptodate: continue.
   - if !mapped:
     - fully_mapped = false.
     - if iblock < lblock: err = get_block(inode, iblock, bh, 0).
     - if !mapped: folio_zero_range(bh region); if !err: set_uptodate; continue.
   - lock_buffer; if uptodate: unlock; continue.
   - mark_buffer_async_read; if prev: Bh::submit(REQ_OP_READ, prev); prev = bh.
5. if fully_mapped: folio_set_mappedtodisk.
6. if prev: Bh::submit(REQ_OP_READ, prev).
7. else: folio_end_read(folio, !page_error).

`Bh::block_write_full_folio(inode, folio, get_block, wbc) -> Result<()>`:
1. head = folio_create_buffers(folio, inode, BH_Dirty|BH_Uptodate).
2. Pass-1 map: walk bhs; if past EOF: clear dirty + set uptodate; if (!mapped ∨ delay) ∧ dirty: get_block(block, 1); clear delay; BH_New → clean_bdev_bh_alias.
3. Pass-2 lock: walk bhs; WB_SYNC_NONE → trylock_buffer (else folio_redirty_for_writepage); WB_SYNC_* → lock_buffer.
   - if test_clear_buffer_dirty: mark_buffer_async_write_endio(bh, Bh::end_async_write).
   - else: unlock_buffer.
4. BUG if folio_test_writeback; folio_start_writeback.
5. Pass-3 submit: walk; if BH_Async_Write: Bh::submit_wbc(REQ_OP_WRITE | write_flags, bh, inode.i_write_hint, wbc); nr_underway++.
6. folio_unlock.
7. if nr_underway == 0: folio_end_writeback.
8. On recover (mid-loop get_block err): re-walk; mapping_set_error; submit mapped+dirty remainder; folio_start_writeback; goto done.

`Bh::sync_dirty(bh, op_flags) -> Result<()>`:
1. WARN b_count < 1.
2. lock_buffer.
3. if test_clear_buffer_dirty:
   - if !buffer_mapped: unlock; return Err(EIO).
   - get_bh; bh.b_end_io = Bh::end_write_sync; Bh::submit(REQ_OP_WRITE | op_flags, bh).
   - wait_on_buffer; if !uptodate: return Err(EIO).
4. else: unlock.
5. return Ok.

`Bh::end_async_read(bh, uptodate)`:
1. spin_lock_irqsave(bh.b_uptodate_lock).
2. if uptodate: set_buffer_uptodate; else: clear_buffer_uptodate; mark page_error on folio metadata.
3. clear_buffer_async_read; unlock_buffer.
4. tmp = bh.b_this_page.
5. while tmp != bh: if buffer_async_read(tmp): spin_unlock; return.
6. spin_unlock_irqrestore.
7. folio_end_read(bh.b_folio, all_uptodate).

`Bh::end_async_write(bh, uptodate)`:
1. if !uptodate: mark_buffer_write_io_error; mapping_set_error; clear_buffer_uptodate.
2. spin_lock_irqsave(bh.b_uptodate_lock).
3. clear_buffer_async_write; unlock_buffer.
4. tmp = bh.b_this_page.
5. while tmp != bh: if buffer_async_write(tmp): spin_unlock; return.
6. spin_unlock_irqrestore.
7. folio_end_writeback(bh.b_folio).

`Bh::try_to_free(folio) -> bool`:
1. if folio_test_writeback: return false.
2. mapping = folio.mapping; if mapping: spin_lock(&mapping.i_private_lock).
3. for bh in chain: if buffer_busy(bh): unlock; return false.
4. for bh in chain: remove_assoc_queue.
5. folio_detach_private; free bh chain (free_buffer_head).
6. unlock; return true.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `bh_lock_excludes_concurrent_io` | INVARIANT | per-submit: BH_Lock held; not released until end_io. |
| `bh_refcount_balanced` | INVARIANT | per-getblk/brelse + per-submit/end_io: b_count get'd and put'd. |
| `mark_dirty_propagates_to_folio` | INVARIANT | per-mark_buffer_dirty: BH_Dirty set ⟹ folio_test_dirty post. |
| `submit_bh_requires_mapped` | INVARIANT | per-submit_bh: buffer_mapped ∧ b_end_io ≠ NULL ∧ !delay ∧ !unwritten. |
| `async_read_completion_once` | INVARIANT | per-folio: folio_end_read called exactly once across all BH_Async_Read completions. |
| `async_write_completion_once` | INVARIANT | per-folio: folio_end_writeback called exactly once across all BH_Async_Write completions. |
| `lru_bh_ref_held` | INVARIANT | per-bh in bh_lru: b_count ≥ 1. |
| `try_to_free_rejects_busy` | INVARIANT | per-try_to_free_buffers: returns false on any busy bh. |
| `bh_accounting_balanced` | INVARIANT | per-alloc/free_buffer_head: per-cpu nr balanced. |
| `bh_size_aligned_to_bdev` | INVARIANT | per-getblk_slow: b_size % bdev_logical_block_size == 0. |
| `bio_sector_matches_blocknr` | INVARIANT | per-submit_bh: bio.bi_sector == bh.b_blocknr * (bh.b_size >> 9). |

### Layer 2: TLA+

`fs/buffer.tla`:
- States: per-(bdev, block) bh: Absent → Cached(clean/dirty) → Locked(reading/writing) → Cached → Freed.
- Per-folio bh-chain completion barrier.
- Properties:
  - `safety_lock_mutex` — per-bh: BH_Lock held by at most one writer.
  - `safety_dirty_implies_uptodate_or_new` — per-bh: BH_Dirty ⟹ BH_Uptodate ∨ BH_New (during write-begin).
  - `safety_submit_requires_mapped` — per-bh: submit_bh ⟹ BH_Mapped.
  - `safety_folio_completion_once` — per-folio: folio_end_read / folio_end_writeback called exactly once per I/O wave.
  - `safety_freed_only_when_clean_and_idle` — per-bh: free_buffer_head ⟹ !Dirty ∧ b_count == 0 ∧ !Lock.
  - `liveness_locked_eventually_unlocked` — per-bh: lock_buffer ⟹ ◇ unlock_buffer.
  - `liveness_dirty_eventually_clean` — per-bh under writeback: BH_Dirty cleared and submit_bh observed.
  - `liveness_async_io_completes` — per-bio: bi_end_io fires ⟹ bh.b_end_io fires.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Bh::find_get_block` post: bh ⟹ bh.b_bdev == bdev ∧ bh.b_blocknr == block ∧ bh.b_size == size ∧ b_count ≥ 1 | `Bh::find_get_block` |
| `Bh::getblk_slow` post: bh ⟹ buffer_mapped ∨ bh.b_blocknr assigned via grow_buffers | `Bh::getblk_slow` |
| `Bh::bread` post: Some(bh) ⟹ buffer_uptodate(bh) | `Bh::bread` |
| `Bh::mark_dirty` post: buffer_dirty(bh) ∧ folio_test_dirty(b_folio) ∧ inode I_DIRTY_PAGES | `Bh::mark_dirty` |
| `Bh::submit` post: bio queued; end_bio_bh_io_sync attached | `Bh::submit` |
| `Bh::block_read_full_folio` post: all bhs either uptodate or async_read submitted | `Bh::block_read_full_folio` |
| `Bh::block_write_full_folio` post: all dirty+mapped bhs submitted; folio writeback-started | `Bh::block_write_full_folio` |
| `Bh::end_async_read` post: folio_end_read called exactly when last async_read clears | `Bh::end_async_read` |
| `Bh::end_async_write` post: folio_end_writeback called exactly when last async_write clears | `Bh::end_async_write` |
| `Bh::try_to_free` post: success ⟹ folio.private == NULL ∧ bh chain freed | `Bh::try_to_free` |
| `Bh::sync_dirty` post: ret == 0 ⟹ buffer_uptodate ∧ !buffer_dirty | `Bh::sync_dirty` |
| `Bh::Lru::install` post: bh at slot 0 ∧ evicted-bh (if any) released | `Bh::Lru::install` |

### Layer 4: Verus/Creusot functional

`Per-(bdev,block,size) find_get_block → submit_bh → end_bio_bh_io_sync → b_end_io` semantic equivalence to upstream: per-Documentation/filesystems/buffer.rst (no longer in tree — derived from `fs/buffer.c` history) and per-extX behavioural tests under `tools/testing/selftests/filesystems`.

Per-folio `block_read_full_folio` / `__block_write_full_folio` round-trip equivalence: read after write returns identical bytes; folio_test_dirty ↔ any bh dirty; folio_test_writeback active throughout pass-2 → end-of-async.

## Hardening

(Inherits row-1 features from `fs/00-overview.md` § Hardening.)

Buffer-head reinforcement:

- **Per-BH_Lock bit-spin-lock with smp_mb on unlock** — defense against per-stale-data-after-unlock visibility bug.
- **Per-bh.b_size aligned to bdev_logical_block_size** — defense against per-partial-sector torn write.
- **Per-submit_bh BUG on !mapped / !end_io / delay / unwritten** — defense against per-corrupt-bio submit.
- **Per-bh BH_Quiet on BIO_QUIET** — defense against per-klog-flood from device hot-removal.
- **Per-bh BH_Write_EIO sticky until rewrite** — defense against per-silent-write-loss after I/O error.
- **Per-bh b_uptodate_lock around folio-completion vote** — defense against per-multi-bh end_io race.
- **Per-folio i_private_lock around bh-chain mutation** — defense against per-concurrent-truncate / reclaim race.
- **Per-cpu bh_lru bounded at BH_LRU_SIZE = 16** — defense against per-LRU-growth memory leak.
- **Per-cpu bh_accounting + max_buffer_heads = 10% ZONE_NORMAL** — defense against per-bh-explosion under cache pressure.
- **Per-buffer_exit_cpu_dead on CPU hot-unplug** — defense against per-orphaned-LRU memory leak.
- **Per-try_to_free_buffers refuses busy bh** — defense against per-UAF on reclaim.
- **Per-free_buffer_head BUG on non-empty b_assoc_buffers** — defense against per-mmb dangling list-head.
- **Per-clean_bdev_bh_alias on BH_New** — defense against per-stale-cached-alias after on-disk reallocation.
- **Per-guard_bio_eod** — defense against per-write-past-end-of-device.
- **Per-WB_SYNC_NONE trylock + redirty fallback** — defense against per-writeback-livelock with reclaim.
- **Per-FS_ENCRYPTION fscrypt_set_bio_crypt_ctx in submit_bh path** — defense against per-plaintext-leak on encrypted block writeback.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `fs/iomap/buffered-io.c` modern bh-free aops (covered in `fs/iomap.md` Tier-3)
- `fs/jbd2/` ext4 journal layered on bh (covered in `fs/ext4/` Tier-3s)
- `block/bio.c` bio submission backend (covered in `block/bio.md` Tier-3)
- `mm/filemap.c` folio / page-cache (covered in `mm/filemap.md` Tier-3)
- `mm/page-writeback.c` balance_dirty_pages / writeback throttling (covered in `mm/writeback.md` Tier-3)
- `fs/direct-io.c` legacy DIO (covered in `fs/direct-io.md` Tier-3)
- Implementation code
