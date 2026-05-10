---
title: "Tier-3: mm/page_io.c — Swap page I/O"
tags: ["tier-3", "mm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

When a folio is selected for reclaim and is anonymous (or shmem) and backed by a swap entry, the **swap I/O path** writes its bytes to the swap area and reads them back on the next fault. Per-`swap_writeout(folio, swap_plug)` is the entry from reclaim: it short-circuits the I/O if (a) `folio_free_swap` reclaims the swap slot, (b) the folio is zero-filled and recorded in `sis->zeromap`, or (c) `zswap_store` accepts the folio into the in-memory compressed cache. Otherwise `__swap_writepage` dispatches to one of three transports: `swap_writepage_fs` for swapfile-via-FS (`SWP_FS_OPS`), `swap_writepage_bdev_sync` for synchronous block device (`SWP_SYNCHRONOUS_IO`), or `swap_writepage_bdev_async` for the default async-BIO path. Per-`swap_read_folio(folio, plug)` is the symmetric read entry from `do_swap_page`: checks zeromap, attempts `zswap_load`, then dispatches to `_fs` / `_bdev_sync` / `_bdev_async`. Per-`end_swap_bio_write` / `_read` are BIO endio handlers; write-error re-dirties the folio to prevent reclaim from dropping it, read-error leaves the folio !uptodate so the fault path delivers SIGBUS. Per-`struct swap_iocb` is the FS-path batching plug: up to `SWAP_CLUSTER_MAX` folios per kiocb. Per-`generic_swapfile_activate(sis, swap_file, span)` walks the swapfile's `bmap()` extents at swapon to populate `sis`'s extent tree, requiring PAGE_SIZE-aligned contiguous runs (a "hole" aborts swapon). Critical for: anonymous-memory reclaim survival, zswap promotion path, swapfile-on-FS compat (NFS swap, btrfs swap), and the BIO-level reclaim cgroup accounting.

This Tier-3 covers `mm/page_io.c` (~673 lines).

### Acceptance Criteria

- [ ] AC-1: swap_writeout: zero-filled folio sets `sis->zeromap` and skips IO.
- [ ] AC-2: swap_writeout: zswap_store accepting folio skips backing IO.
- [ ] AC-3: swap_writeout: !mem_cgroup_zswap_writeback_enabled returns AOP_WRITEPAGE_ACTIVATE.
- [ ] AC-4: __swap_writepage: SWP_FS_OPS path takes `swap_writepage_fs`.
- [ ] AC-5: __swap_writepage: SWP_SYNCHRONOUS_IO path takes `swap_writepage_bdev_sync` (submit_bio_wait).
- [ ] AC-6: __swap_writepage: default path takes `swap_writepage_bdev_async` and queues `end_swap_bio_write`.
- [ ] AC-7: swap_writepage_fs: SAWP_CLUSTER_MAX-batched plug; unplug on mismatch or full.
- [ ] AC-8: end_swap_bio_write: bi_status≠0 ⟹ folio_mark_dirty + folio_clear_reclaim.
- [ ] AC-9: end_swap_bio_read: bi_status≠0 ⟹ folio stays !uptodate (fault path emits SIGBUS).
- [ ] AC-10: swap_read_folio: zeromap-hit returns zero-filled folio uptodate without backing IO.
- [ ] AC-11: swap_read_folio: zswap_load ≠ -ENOENT returns without backing IO.
- [ ] AC-12: swap_read_folio_bdev_sync: get/put_task_struct brackets submit_bio_wait.
- [ ] AC-13: swap_read_folio: workingset ⟹ psi_memstall_enter/leave + delayacct_thrashing_start/end.
- [ ] AC-14: generic_swapfile_activate: bmap discontiguity ⟹ "swapon: swapfile has holes" + -EINVAL.
- [ ] AC-15: count_swpout_vm_event: PMD-mappable folio increments THP_SWPOUT.

### Architecture

```
struct SwapIocb {
  iocb: Kiocb,
  bvec: [BioVec; SWAP_CLUSTER_MAX],
  pages: i32,
  len: i32,
}
```

`PageIo::swap_writeout(folio, swap_plug) -> i32`:
1. if folio_free_swap(folio): folio_unlock; return 0.
2. ret = arch_prepare_to_swap(folio).
3. if ret != 0: folio_mark_dirty; folio_unlock; return ret.
4. if is_folio_zero_filled(folio):
   - swap_zeromap_folio_set(folio).
   - folio_unlock; return 0.
5. swap_zeromap_folio_clear(folio).
6. if zswap_store(folio):
   - count_mthp_stat(MTHP_STAT_ZSWPOUT).
   - folio_unlock; return 0.
7. rcu_read_lock.
8. if !mem_cgroup_zswap_writeback_enabled(folio_memcg(folio)):
   - rcu_read_unlock; folio_mark_dirty; return AOP_WRITEPAGE_ACTIVATE.
9. rcu_read_unlock.
10. PageIo::write_page_dispatch(folio, swap_plug).
11. return 0.

`PageIo::write_page_dispatch(folio, swap_plug)`:
1. sis = __swap_entry_to_info(folio.swap).
2. VM_BUG_ON_FOLIO(!folio_test_swapcache(folio)).
3. /* data_race-safe: SWP_FS_OPS never toggles after swapon */
4. if sis.flags & SWP_FS_OPS: PageIo::write_page_fs(folio, swap_plug).
5. else if sis.flags & SWP_SYNCHRONOUS_IO: PageIo::write_page_bdev_sync(folio, sis).
6. else: PageIo::write_page_bdev_async(folio, sis).

`PageIo::write_page_bdev_async(folio, sis)`:
1. bio = bio_alloc(sis.bdev, 1, REQ_OP_WRITE | REQ_SWAP, GFP_NOIO).
2. bio.bi_iter.bi_sector = swap_folio_sector(folio).
3. bio.bi_end_io = end_swap_bio_write.
4. bio_add_folio_nofail(bio, folio, folio_size(folio), 0).
5. PageIo::bio_associate_blkg(bio, folio).
6. PageIo::count_swpout_event(folio).
7. folio_start_writeback(folio).
8. folio_unlock(folio).
9. submit_bio(bio).

`PageIo::write_page_bdev_sync(folio, sis)`:
1. /* On-stack bio (struct bio) + on-stack bio_vec for atomic submit */
2. bio_init(&bio, sis.bdev, &bv, 1, REQ_OP_WRITE | REQ_SWAP).
3. bio.bi_iter.bi_sector = swap_folio_sector(folio).
4. bio_add_folio_nofail(&bio, folio, folio_size(folio), 0).
5. PageIo::bio_associate_blkg(&bio, folio).
6. PageIo::count_swpout_event(folio).
7. folio_start_writeback; folio_unlock.
8. submit_bio_wait(&bio).
9. PageIo::end_bio_write_inner(&bio).  /* __end_swap_bio_write equivalent, no bio_put */

`PageIo::write_page_fs(folio, swap_plug)`:
1. sis = __swap_entry_to_info(folio.swap); swap_file = sis.swap_file.
2. pos = swap_dev_pos(folio.swap).
3. PageIo::count_swpout_event(folio).
4. folio_start_writeback; folio_unlock.
5. sio = if swap_plug.is_some() { *swap_plug } else { None }.
6. if let Some(s) = sio:
   - if s.iocb.ki_filp != swap_file ∨ s.iocb.ki_pos + s.len != pos:
     - PageIo::write_unplug(s); sio = None.
7. if sio.is_none():
   - sio = mempool_alloc(sio_pool, GFP_NOIO).
   - init_sync_kiocb(&sio.iocb, swap_file).
   - sio.iocb.ki_complete = sio_write_complete.
   - sio.iocb.ki_pos = pos; sio.pages = 0; sio.len = 0.
8. bvec_set_folio(&sio.bvec[sio.pages], folio, folio_size(folio), 0).
9. sio.len += folio_size(folio); sio.pages += 1.
10. if sio.pages == SwapIocb::CAPACITY ∨ swap_plug.is_none():
    - PageIo::write_unplug(sio); sio = None.
11. if swap_plug.is_some(): *swap_plug = sio.

`PageIo::write_unplug(sio)`:
1. iov_iter_bvec(&from, ITER_SOURCE, sio.bvec, sio.pages, sio.len).
2. mapping = sio.iocb.ki_filp.f_mapping.
3. ret = mapping.a_ops.swap_rw(&sio.iocb, &from).
4. if ret != -EIOCBQUEUED: PageIo::sio_write_complete(&sio.iocb, ret).

`PageIo::end_bio_write(bio)` (BIO endio):
1. folio = bio_first_folio_all(bio).
2. if bio.bi_status != 0:
   - folio_mark_dirty(folio).
   - pr_alert_ratelimited("Write-error on swap-device (%u:%u:%llu)", MAJOR(bio_dev), MINOR(bio_dev), bio.bi_iter.bi_sector).
   - folio_clear_reclaim(folio).
3. folio_end_writeback(folio).
4. bio_put(bio).

`PageIo::read_folio(folio, plug)`:
1. sis = __swap_entry_to_info(folio.swap).
2. synchronous = sis.flags & SWP_SYNCHRONOUS_IO.
3. workingset = folio_test_workingset(folio).
4. VM_BUG_ON_FOLIO checks.
5. if workingset: delayacct_thrashing_start(&in_thrashing); psi_memstall_enter(&pflags).
6. delayacct_swapin_start.
7. if PageIo::read_zeromap(folio): folio_unlock; goto finish.
8. if zswap_load(folio) != -ENOENT: goto finish.
9. zswap_folio_swapin(folio).
10. if sis.flags & SWP_FS_OPS: PageIo::read_folio_fs(folio, plug).
11. else if synchronous: PageIo::read_folio_bdev_sync(folio, sis).
12. else: PageIo::read_folio_bdev_async(folio, sis).
13. finish: if workingset: delayacct_thrashing_end; psi_memstall_leave.
14. delayacct_swapin_end.

`PageIo::read_folio_bdev_sync(folio, sis)`:
1. bio_init(&bio, sis.bdev, &bv, 1, REQ_OP_READ).
2. bio.bi_iter.bi_sector = swap_folio_sector(folio).
3. bio_add_folio_nofail(&bio, folio, folio_size(folio), 0).
4. get_task_struct(current).      /* Pin task for OOM-fault-retry inspection */
5. count_mthp_stat(MTHP_STAT_SWPIN); count_memcg_folio_events(PSWPIN); count_vm_events(PSWPIN).
6. submit_bio_wait(&bio).
7. PageIo::end_bio_read_inner(&bio).
8. put_task_struct(current).

`PageIo::end_bio_read(bio)`:
1. folio = bio_first_folio_all(bio).
2. if bio.bi_status != 0:
   - pr_alert_ratelimited("Read-error on swap-device (%u:%u:%llu)", ...).
3. else: folio_mark_uptodate(folio).
4. folio_unlock(folio).
5. bio_put(bio).

`PageIo::read_zeromap(folio) -> bool`:
1. nr_pages = folio_nr_pages(folio).
2. if WARN_ON_ONCE(swap_zeromap_batch(folio.swap, nr_pages, &is_zeromap) != nr_pages):
   - return true (leaves folio !uptodate → SIGBUS).
3. if !is_zeromap: return false.
4. count_vm_events(SWPIN_ZERO, nr_pages).
5. if objcg: count_objcg_events(SWPIN_ZERO); obj_cgroup_put.
6. folio_zero_range(folio, 0, folio_size(folio)).
7. folio_mark_uptodate(folio).
8. return true.

`PageIo::swapfile_activate(sis, swap_file, *span) -> i32`:
1. mapping = swap_file.f_mapping; inode = mapping.host.
2. blkbits = inode.i_blkbits; blocks_per_page = PAGE_SIZE >> blkbits.
3. probe_block = 0; page_no = 0; last_block = i_size_read(inode) >> blkbits.
4. while probe_block + blocks_per_page <= last_block ∧ page_no < sis.max:
   - cond_resched.
   - first_block = probe_block; ret = bmap(inode, &first_block).
   - if ret ∨ !first_block: goto bad_bmap.
   - if first_block & (blocks_per_page - 1): probe_block++; continue.
   - for block_in_page in 1..blocks_per_page:
     - block = probe_block + block_in_page.
     - if bmap(inode, &block) fails ∨ block != first_block + block_in_page: probe_block++; goto reprobe.
   - first_block >>= (PAGE_SHIFT - blkbits).
   - if page_no > 0: update lowest_block, highest_block.
   - ret = add_swap_extent(sis, page_no, 1, first_block).
   - if ret < 0: goto out.
   - nr_extents += ret; page_no++; probe_block += blocks_per_page.
5. *span = 1 + highest_block - lowest_block.
6. sis.max = page_no; sis.pages = page_no - 1.
7. return nr_extents.
8. bad_bmap: pr_err("swapon: swapfile has holes"); return -EINVAL.

### Out of Scope

- mm/swap_state.c swap cache (covered in `swap-state.md` Tier-3 if expanded)
- mm/swapfile.c swap_info_struct / swapon/swapoff (covered in `swapfile.md` Tier-3 if expanded)
- mm/zswap.c compressed cache internals (covered in `zswap.md` Tier-3 if expanded)
- fs/crypto/ fscrypt swap encryption (covered separately if expanded)
- block layer BIO submission internals (covered in `block-bio.md` Tier-3 if expanded)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `swap_writeout()` | per-reclaim swap-write entry | `PageIo::swap_writeout` |
| `__swap_writepage()` | per-folio dispatch (FS / sync-bdev / async-bdev) | `PageIo::write_page_dispatch` |
| `swap_writepage_fs()` | per-SWP_FS_OPS batched kiocb | `PageIo::write_page_fs` |
| `swap_writepage_bdev_sync()` | per-SWP_SYNCHRONOUS_IO submit-and-wait | `PageIo::write_page_bdev_sync` |
| `swap_writepage_bdev_async()` | per-default async BIO | `PageIo::write_page_bdev_async` |
| `swap_read_folio()` | per-fault swap-read entry | `PageIo::read_folio` |
| `swap_read_folio_fs()` | per-SWP_FS_OPS batched read | `PageIo::read_folio_fs` |
| `swap_read_folio_bdev_sync()` | per-SWP_SYNCHRONOUS_IO submit-wait read | `PageIo::read_folio_bdev_sync` |
| `swap_read_folio_bdev_async()` | per-async BIO read | `PageIo::read_folio_bdev_async` |
| `end_swap_bio_write()` / `__end_swap_bio_write()` | per-write BIO endio | `PageIo::end_bio_write` |
| `end_swap_bio_read()` / `__end_swap_bio_read()` | per-read BIO endio | `PageIo::end_bio_read` |
| `sio_write_complete()` / `sio_read_complete()` | per-FS-batched-IO completion | `PageIo::sio_write_complete` / `sio_read_complete` |
| `swap_write_unplug()` / `__swap_read_unplug()` | per-flush batched plug | `PageIo::write_unplug` / `read_unplug` |
| `sio_pool_init()` | per-mempool for swap_iocb | `PageIo::sio_pool_init` |
| `swap_zeromap_folio_set()` / `_clear()` | per-zero-folio bitmap track | `PageIo::zeromap_set` / `zeromap_clear` |
| `is_folio_zero_filled()` | per-zero-detect | `PageIo::is_zero_filled` |
| `swap_read_folio_zeromap()` | per-zero-folio read short-circuit | `PageIo::read_zeromap` |
| `generic_swapfile_activate()` | per-swapon extent-tree build | `PageIo::swapfile_activate` |
| `count_swpout_vm_event()` | per-PSWPOUT / THP_SWPOUT stat | `PageIo::count_swpout_event` |
| `bio_associate_blkg_from_page()` | per-MEMCG+BLK_CGROUP CSS attach | `PageIo::bio_associate_blkg` |
| `struct swap_iocb` | per-FS-path batch (bvec[SWAP_CLUSTER_MAX]) | `SwapIocb` |

### compatibility contract

REQ-1: struct swap_iocb:
- iocb: per-kiocb bound to swap_file.
- bvec[SWAP_CLUSTER_MAX]: per-folio bvec array.
- pages: per-current populated count.
- len: per-byte sum.

REQ-2: swap_writeout(folio, swap_plug) -> i32:
- /* Reclaim swap slot if mapcount/refcount permits */
- if folio_free_swap(folio): goto out_unlock; return 0.
- /* Arch hook (e.g. memory tags) */
- ret = arch_prepare_to_swap(folio).
- if ret: folio_mark_dirty(folio); goto out_unlock; return ret.
- /* Zero-fill bitmap fast path */
- if is_folio_zero_filled(folio):
  - swap_zeromap_folio_set(folio).
  - goto out_unlock; return 0.
- /* Otherwise clear any stale zeromap bits */
- swap_zeromap_folio_clear(folio).
- /* zswap intercept (compressed in-memory) */
- if zswap_store(folio):
  - count_mthp_stat(folio_order(folio), MTHP_STAT_ZSWPOUT).
  - goto out_unlock; return 0.
- /* zswap writeback gate */
- rcu_read_lock.
- if !mem_cgroup_zswap_writeback_enabled(folio_memcg(folio)):
  - rcu_read_unlock; folio_mark_dirty; return AOP_WRITEPAGE_ACTIVATE.
- rcu_read_unlock.
- /* Real backing IO */
- __swap_writepage(folio, swap_plug).
- return 0.
- out_unlock: folio_unlock(folio); return ret.

REQ-3: __swap_writepage(folio, swap_plug):
- sis = __swap_entry_to_info(folio.swap).
- VM_BUG_ON_FOLIO(!folio_test_swapcache(folio)).
- if data_race(sis.flags & SWP_FS_OPS): swap_writepage_fs(folio, swap_plug).
- else if data_race(sis.flags & SWP_SYNCHRONOUS_IO): swap_writepage_bdev_sync(folio, sis).
- else: swap_writepage_bdev_async(folio, sis).

REQ-4: swap_writepage_bdev_async(folio, sis):
- bio = bio_alloc(sis.bdev, 1, REQ_OP_WRITE | REQ_SWAP, GFP_NOIO).
- bio.bi_iter.bi_sector = swap_folio_sector(folio).
- bio.bi_end_io = end_swap_bio_write.
- bio_add_folio_nofail(bio, folio, folio_size(folio), 0).
- bio_associate_blkg_from_page(bio, folio).
- count_swpout_vm_event(folio).
- folio_start_writeback(folio).
- folio_unlock(folio).
- submit_bio(bio).

REQ-5: swap_writepage_bdev_sync(folio, sis):
- /* On-stack bio + single bv */
- bio_init(&bio, sis.bdev, &bv, 1, REQ_OP_WRITE | REQ_SWAP).
- bio.bi_iter.bi_sector = swap_folio_sector(folio).
- bio_add_folio_nofail(&bio, folio, folio_size(folio), 0).
- bio_associate_blkg_from_page(&bio, folio).
- count_swpout_vm_event(folio).
- folio_start_writeback(folio); folio_unlock(folio).
- submit_bio_wait(&bio).
- __end_swap_bio_write(&bio).

REQ-6: swap_writepage_fs(folio, swap_plug):
- sio = *swap_plug (or NULL).
- sis = __swap_entry_to_info(folio.swap); swap_file = sis.swap_file; pos = swap_dev_pos(folio.swap).
- count_swpout_vm_event(folio); folio_start_writeback(folio); folio_unlock(folio).
- /* Flush plug if mismatched file or non-contiguous offset */
- if sio ∧ (sio.iocb.ki_filp != swap_file ∨ sio.iocb.ki_pos + sio.len != pos):
  - swap_write_unplug(sio); sio = NULL.
- if !sio:
  - sio = mempool_alloc(sio_pool, GFP_NOIO).
  - init_sync_kiocb(&sio.iocb, swap_file).
  - sio.iocb.ki_complete = sio_write_complete.
  - sio.iocb.ki_pos = pos; sio.pages = 0; sio.len = 0.
- bvec_set_folio(&sio.bvec[sio.pages], folio, folio_size(folio), 0).
- sio.len += folio_size(folio); sio.pages += 1.
- /* Flush if batch full or no plug requested */
- if sio.pages == ARRAY_SIZE(sio.bvec) ∨ !swap_plug:
  - swap_write_unplug(sio); sio = NULL.
- if swap_plug: *swap_plug = sio.

REQ-7: swap_write_unplug(sio):
- iov_iter_bvec(&from, ITER_SOURCE, sio.bvec, sio.pages, sio.len).
- ret = mapping.a_ops.swap_rw(&sio.iocb, &from).
- if ret != -EIOCBQUEUED: sio_write_complete(&sio.iocb, ret).

REQ-8: sio_write_complete(iocb, ret):
- sio = container_of(iocb, ...).
- if ret != sio.len:
  - pr_err_ratelimited.
  - for p in 0..sio.pages: set_page_dirty(bvec[p].bv_page); ClearPageReclaim(bvec[p].bv_page).
- for p in 0..sio.pages: end_page_writeback(bvec[p].bv_page).
- mempool_free(sio, sio_pool).

REQ-9: __end_swap_bio_write(bio):
- folio = bio_first_folio_all(bio).
- if bio.bi_status:
  - folio_mark_dirty(folio).
  - pr_alert_ratelimited("Write-error on swap-device (%u:%u:%llu)").
  - folio_clear_reclaim(folio).
- folio_end_writeback(folio).
- /* end_swap_bio_write also bio_puts; __end_ does not */

REQ-10: swap_read_folio(folio, plug):
- sis = __swap_entry_to_info(folio.swap).
- synchronous = sis.flags & SWP_SYNCHRONOUS_IO.
- workingset = folio_test_workingset(folio).
- VM_BUG_ON_FOLIO(!folio_test_swapcache(folio) ∧ !synchronous).
- VM_BUG_ON_FOLIO(!folio_test_locked(folio)).
- VM_BUG_ON_FOLIO(folio_test_uptodate(folio)).
- if workingset: delayacct_thrashing_start; psi_memstall_enter.
- delayacct_swapin_start.
- /* Zero-folio short-circuit */
- if swap_read_folio_zeromap(folio): folio_unlock(folio); goto finish.
- /* zswap promote */
- if zswap_load(folio) != -ENOENT: goto finish.
- /* Slower device: boost zswap protection */
- zswap_folio_swapin(folio).
- if data_race(sis.flags & SWP_FS_OPS): swap_read_folio_fs(folio, plug).
- else if synchronous: swap_read_folio_bdev_sync(folio, sis).
- else: swap_read_folio_bdev_async(folio, sis).
- finish: if workingset: delayacct_thrashing_end; psi_memstall_leave.
- delayacct_swapin_end.

REQ-11: swap_read_folio_bdev_sync(folio, sis):
- bio_init(&bio, sis.bdev, &bv, 1, REQ_OP_READ).
- bio.bi_iter.bi_sector = swap_folio_sector(folio).
- bio_add_folio_nofail(&bio, folio, folio_size(folio), 0).
- /* Pin task across submit_bio_wait — OOM may access during fault retry */
- get_task_struct(current).
- count_mthp_stat(SWPIN); count_memcg_folio_events(PSWPIN); count_vm_events(PSWPIN).
- submit_bio_wait(&bio).
- __end_swap_bio_read(&bio).
- put_task_struct(current).

REQ-12: swap_read_folio_bdev_async(folio, sis):
- bio = bio_alloc(sis.bdev, 1, REQ_OP_READ, GFP_KERNEL).
- bio.bi_end_io = end_swap_bio_read.
- bio_add_folio_nofail; count stats; submit_bio(bio).

REQ-13: __end_swap_bio_read(bio):
- folio = bio_first_folio_all(bio).
- if bio.bi_status: pr_alert_ratelimited("Read-error on swap-device").
- else: folio_mark_uptodate(folio).
- folio_unlock(folio).

REQ-14: swap_read_folio_zeromap(folio) -> bool:
- nr_pages = folio_nr_pages(folio).
- /* Partial-in-zeromap of a large folio is unsupported: return true without uptodate so SIGBUS fires */
- if WARN_ON_ONCE(swap_zeromap_batch(folio.swap, nr_pages, &is_zeromap) != nr_pages): return true.
- if !is_zeromap: return false.
- count_vm_events(SWPIN_ZERO); per-objcg event.
- folio_zero_range(folio, 0, folio_size(folio)).
- folio_mark_uptodate(folio).
- return true.

REQ-15: generic_swapfile_activate(sis, swap_file, *span) -> i32:
- /* Walk inode block map building PAGE_SIZE-aligned extents */
- blkbits = inode.i_blkbits; blocks_per_page = PAGE_SIZE >> blkbits.
- probe_block = 0; page_no = 0.
- last_block = i_size_read(inode) >> blkbits.
- while probe_block + blocks_per_page <= last_block ∧ page_no < sis.max:
  - first_block = probe_block; bmap(inode, &first_block).
  - if !first_block: goto bad_bmap.
  - if first_block & (blocks_per_page - 1): probe_block++; continue (reprobe).
  - for block_in_page in 1..blocks_per_page:
    - if bmap fails or non-contig: probe_block++; continue (reprobe).
  - add_swap_extent(sis, page_no, 1, first_block >> (PAGE_SHIFT - blkbits)).
  - page_no++; probe_block += blocks_per_page.
- *span = 1 + highest_block - lowest_block.
- sis.max = page_no; sis.pages = page_no - 1; return nr_extents.
- bad_bmap: pr_err("swapon: swapfile has holes"); return -EINVAL.

REQ-16: count_swpout_vm_event(folio):
- /* PMD-mappable THP */
- if folio_test_pmd_mappable(folio): count_memcg_folio_events(THP_SWPOUT, 1); count_vm_event(THP_SWPOUT).
- count_mthp_stat(folio_order(folio), MTHP_STAT_SWPOUT).
- count_memcg_folio_events(PSWPOUT, folio_nr_pages).
- count_vm_events(PSWPOUT, folio_nr_pages).

REQ-17: bio_associate_blkg_from_page(bio, folio):
- /* Per-CONFIG_MEMCG && CONFIG_BLK_CGROUP only */
- if !folio_memcg_charged(folio): return.
- rcu_read_lock; memcg = folio_memcg; css = cgroup_e_css(memcg.css.cgroup, &io_cgrp_subsys).
- bio_associate_blkg_from_css(bio, css); rcu_read_unlock.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `write_dispatch_partition` | INVARIANT | per-__swap_writepage: exactly one of FS / bdev_sync / bdev_async taken. |
| `write_error_redirties_folio` | INVARIANT | per-end_swap_bio_write: bi_status != 0 ⟹ folio_mark_dirty + folio_clear_reclaim before folio_end_writeback. |
| `read_error_leaves_not_uptodate` | INVARIANT | per-end_swap_bio_read: bi_status != 0 ⟹ folio !uptodate at folio_unlock. |
| `zeromap_skips_backing_io` | INVARIANT | per-swap_writeout: is_folio_zero_filled ⟹ no __swap_writepage call. |
| `zswap_store_skips_backing_io` | INVARIANT | per-swap_writeout: zswap_store true ⟹ no __swap_writepage call. |
| `sio_batch_bounded` | INVARIANT | per-swap_writepage_fs: sio.pages ≤ SWAP_CLUSTER_MAX always. |
| `sio_plug_contiguous` | INVARIANT | per-swap_writepage_fs: appended folio's pos equals sio.iocb.ki_pos + sio.len. |
| `bdev_sync_task_pinned` | INVARIANT | per-swap_read_folio_bdev_sync: get/put_task_struct paired across submit_bio_wait. |
| `swapfile_activate_aligned` | INVARIANT | per-generic_swapfile_activate: emitted extents are PAGE_SIZE-aligned. |
| `swapfile_activate_holes_rejected` | INVARIANT | per-generic_swapfile_activate: discontiguous bmap returns -EINVAL. |
| `workingset_psi_paired` | INVARIANT | per-swap_read_folio: workingset ⟹ psi_memstall_enter and _leave both called. |

### Layer 2: TLA+

`mm/page-io.tla`:
- Per-swap-write + per-swap-read + per-zeromap + per-zswap-intercept + per-BIO-endio.
- Properties:
  - `safety_zeromap_no_backing_io` — per-write: zero-folio ⟹ no BIO/iocb submitted.
  - `safety_zswap_no_backing_io` — per-write: zswap_store accept ⟹ no BIO/iocb submitted.
  - `safety_error_redirties` — per-write-bio fail ⟹ folio marked dirty.
  - `safety_no_uptodate_on_read_error` — per-read-bio fail ⟹ folio not marked uptodate.
  - `safety_fs_batch_unplug_on_mismatch` — per-FS-write: mismatched file/pos triggers unplug.
  - `liveness_per_write_eventually_endwriteback` — per-submitted-bio: folio_end_writeback eventually.
  - `liveness_per_read_eventually_unlocked` — per-submitted-bio: folio_unlock eventually.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `PageIo::swap_writeout` post: ret ∈ {0, AOP_WRITEPAGE_ACTIVATE, -errno} | `PageIo::swap_writeout` |
| `PageIo::write_page_dispatch` post: exactly one transport invoked | `PageIo::write_page_dispatch` |
| `PageIo::write_page_fs` post: sio.pages ≤ SWAP_CLUSTER_MAX ∧ contiguous | `PageIo::write_page_fs` |
| `PageIo::end_bio_write` post: bi_status ⟹ folio dirty + reclaim cleared | `PageIo::end_bio_write` |
| `PageIo::end_bio_read` post: bi_status ⟹ !uptodate; else uptodate | `PageIo::end_bio_read` |
| `PageIo::read_zeromap` post: returns true ⟹ folio uptodate ∨ partial-large-folio | `PageIo::read_zeromap` |
| `PageIo::swapfile_activate` post: nr_extents > 0 ∨ -EINVAL on hole | `PageIo::swapfile_activate` |

### Layer 4: Verus/Creusot functional

`Per-folio reclaim → swap_writeout → (zeromap | zswap_store | __swap_writepage → FS/bdev_sync/bdev_async) → BIO endio → folio_end_writeback` and `do_swap_page → swap_read_folio → (zeromap | zswap_load | __swap_readpage transport) → folio uptodate ∨ SIGBUS` semantic equivalence: per-Documentation/admin-guide/mm/swap_numa.rst + Documentation/admin-guide/mm/zswap.rst.

### hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening.)

Page-IO reinforcement:

- **Per-SWP_FS_OPS data_race read** — sis.flags is set at swapon and never toggled, so the data_race annotation is sound; defense against per-spurious-KCSAN noise.
- **Per-folio_test_swapcache assertion** — defense against per-non-swap-cache folio reaching __swap_writepage.
- **Per-zeromap bitmap atomic set/clear** — defense against per-zero-folio read-modify-write corruption (bits protected by swapcache folio lock, atomics on set_bit/clear_bit).
- **Per-write-error folio_mark_dirty + folio_clear_reclaim** — defense against per-reclaim-drop of unwritten data.
- **Per-read-error folio !uptodate** — defense against per-corruption-as-zero on read fail (fault path delivers SIGBUS instead).
- **Per-swap_iocb plug contiguity check** — defense against per-disordered swap-IO on FS (NFS/btrfs).
- **Per-SWAP_CLUSTER_MAX batch cap** — defense against per-unbounded plug growth.
- **Per-bio_associate_blkg under RCU** — defense against per-memcg/blkcg race during folio reclaim.
- **Per-swap_writepage_bdev_sync on-stack bio** — defense against per-OOM-during-OOM-write (no bio_alloc on the reclaim hot path).
- **Per-get_task_struct across submit_bio_wait** — defense against per-fault-retry UAF when OOM inspects current.
- **Per-generic_swapfile_activate hole rejection (-EINVAL)** — defense against per-non-contiguous swapfile corrupting on-disk slot mapping.
- **Per-zswap writeback gate via mem_cgroup_zswap_writeback_enabled** — defense against per-cgroup-policy bypass.
- **Per-arch_prepare_to_swap pre-IO hook** — defense against per-arch-state-loss (e.g. ARMv8 MTE tags) on swap-out.

