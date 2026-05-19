---
title: "Tier-3: fs/btrfs/raid56.c — Btrfs RAID5/RAID6 stripe engine"
tags: ["tier-3", "fs", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

Btrfs **RAID5/RAID6** parity stripe engine. One full stripe = `nr_data` data stripes (64 KiB each, `BTRFS_STRIPE_LEN`) plus 1 (RAID5: P) or 2 (RAID6: P+Q) parity stripes. The central object `struct btrfs_raid_bio` ("rbio") aggregates one or more higher-layer bios for the same full stripe so the RMW (read-modify-write) path can compute P/Q over the whole stripe. Three operations are supported: `BTRFS_RBIO_WRITE` (RMW: read missing data sectors, recompute P/Q, write back the changed sectors + P/Q), `BTRFS_RBIO_READ_REBUILD` (a higher-layer read hit a missing-device or csum-mismatch sector and needs reconstruction from parity), and `BTRFS_RBIO_PARITY_SCRUB` (scrubber asks "regenerate P/Q from data and compare to what's on disk"). P is computed by `xor_gen`; Q is computed by `raid6_call.gen_syndrome` (Reed-Solomon over GF(2^8) from `lib/raid6/`). Single-disk loss in RAID5 / dual-disk loss in RAID6 is reconstructed via `xor_gen` (P-only path) or `raid6_2data_recov`/`raid6_datap_recov`. A per-fs stripe hash table (`struct btrfs_stripe_hash_table`) serializes concurrent rbios on the same full stripe (`lock_stripe_add` / `unlock_stripe`) and an LRU **full-stripe cache** keeps recent rbios alive so a back-to-back sub-stripe write can skip re-reading data sectors. Btrfs RAID5/6 still has a known **write-hole** (data + parity update is not atomic across a crash) — the code mitigates by csum-verifying data sectors during RMW reads and refusing to compute parity from a torn sub-stripe write.

This Tier-3 covers `fs/btrfs/raid56.c` (~3044 lines).

### Acceptance Criteria

- [ ] AC-1: alloc_rbio: ENOMEM path frees all partially-allocated arrays + rbio.
- [ ] AC-2: raid56_parity_write: full-stripe write skips plug, queues RMW directly.
- [ ] AC-3: raid56_parity_write: sub-stripe write attaches to per-task plug, then unplug submits all rbios sorted by full_stripe_logical.
- [ ] AC-4: rmw_rbio: RAID5 P stripe computed via memcpy + xor_gen over nr_data-1 sources.
- [ ] AC-5: rmw_rbio: RAID6 P+Q stripes computed via raid6_call.gen_syndrome over real_stripes sources.
- [ ] AC-6: recover_vertical: RAID5 single-failure rebuilt from P (memcpy + xor_gen rotated).
- [ ] AC-7: recover_vertical: RAID6 dual-data failure rebuilt via raid6_2data_recov; data+P failure via raid6_datap_recov.
- [ ] AC-8: get_rbio_vertical_errors > max_errors → -EIO, no false data returned.
- [ ] AC-9: fill_data_csums: rebuilt data sector verified against csum tree before being returned.
- [ ] AC-10: lock_stripe_add: two concurrent rbios on same full_stripe_logical — second is either merged or queued on plug_list.
- [ ] AC-11: unlock_stripe: stripe lock handed off to next pending rbio without dropping data.
- [ ] AC-12: cache_rbio: LRU evicts when cache_size exceeds threshold; eviction is idempotent.
- [ ] AC-13: scrub_rbio: parity mismatch between on-disk and regenerated P/Q triggers parity overwrite, not data overwrite.
- [ ] AC-14: scrub_rbio: parity-only error tolerated (dfail == 0) — only parity rewritten.
- [ ] AC-15: raid56_parity_cache_data_folios: pre-fills stripe_pages from known-good data folios to avoid re-read.

### Architecture

```
struct Rbio {
  bioc: NonNull<BtrfsIoContext>,
  hash_list: ListHead,
  stripe_cache: ListHead,
  work: WorkStruct,
  bio_list: BioList,
  bio_list_lock: SpinLock,
  plug_list: ListHead,
  flags: AtomicUlong,                  // RBIO_*_BIT
  operation: RbioOp,                   // Write / ReadRebuild / ParityScrub
  nr_pages: u16,
  nr_sectors: u16,
  nr_data: u8,
  real_stripes: u8,
  stripe_npages: u8,
  stripe_nsectors: u8,
  sector_nsteps: u8,
  scrubp: u8,
  bio_list_bytes: i32,
  refs: RefCount,
  stripes_pending: AtomicI32,
  io_wait: WaitQueueHead,
  dbitmap: BitMap,                     // stripe_nsectors bits
  finish_pbitmap: BitMap,
  stripe_pages: Box<[Option<*Page>]>,  // nr_pages
  bio_paddrs: Box<[PhysAddr]>,         // nr_sectors * sector_nsteps, INVALID_PADDR sentinel
  stripe_paddrs: Box<[PhysAddr]>,      // same
  stripe_uptodate_bitmap: BitMap,
  finish_pointers: Box<[*mut c_void]>, // real_stripes
  error_bitmap: BitMap,                // nr_sectors
  csum_buf: Option<Box<[u8]>>,
  csum_bitmap: Option<BitMap>,
}

enum RbioOp { Write, ReadRebuild, ParityScrub }

struct StripeHash { hash_list: ListHead, lock: SpinLock }
struct StripeHashTable {
  stripe_cache: ListHead,
  cache_lock: SpinLock,
  cache_size: i32,
  table: Box<[StripeHash]>,            // 1 << BTRFS_STRIPE_HASH_TABLE_BITS
}
```

`Rbio::alloc(fs_info, bioc) -> Result<Self, Errno>`:
1. real_stripes = bioc.num_stripes - bioc.replace_nr_stripes.
2. stripe_npages = BTRFS_STRIPE_LEN / PAGE_SIZE; stripe_nsectors = BTRFS_STRIPE_LEN / sectorsize; sector_nsteps = sectorsize / min(sectorsize, PAGE_SIZE).
3. Validate alignment: PAGE_SIZE ‖ sectorsize ∨ sectorsize ‖ PAGE_SIZE; stripe_nsectors ≤ BITS_PER_LONG; 2 ≤ real_stripes ≤ 255.
4. Allocate rbio + 6 sidecar arrays (stripe_pages, bio_paddrs, stripe_paddrs, finish_pointers, error_bitmap, stripe_uptodate_bitmap); GFP_NOFS.
5. Initialize bio_paddrs / stripe_paddrs to INVALID_PADDR.
6. nr_data = real_stripes - btrfs_nr_parity_stripes(bioc.map_type).
7. refcount_set(refs, 1); atomic_set(stripes_pending, 0); init bio_list / plug_list / hash_list / stripe_cache / io_wait / bio_list_lock.
8. btrfs_get_bioc(bioc).
9. Return Ok(rbio).

`Rbio::write_entry(bio, bioc)`:
1. rbio = Rbio::alloc(fs_info, bioc).
2. rbio.operation = Write; Rbio::add_bio(rbio, bio).
3. if !Rbio::is_full(rbio): try blk_check_plugged(raid_unplug, fs_info); if plug → append rbio to plug.rbio_list; return.
4. start_async_work(rbio, rmw_rbio_work).

`Rbio::recover_entry(bio, bioc, mirror_num)`:
1. rbio = Rbio::alloc; rbio.operation = ReadRebuild.
2. Rbio::add_bio(rbio, bio); set_rbio_range_error(rbio, bio).
3. if mirror_num > 2: set_rbio_raid6_extra_error(rbio, mirror_num).
4. start_async_work(rbio, recover_rbio_work).

`Rbio::rmw(rbio)`:
1. alloc_rbio_parity_pages(rbio).
2. if !Rbio::is_full(rbio) ∧ Rbio::need_read_stripe_sectors(rbio):
   - alloc_rbio_data_pages(rbio); Rbio::index_rbio_pages(rbio); Rbio::read_wait_recover(rbio).
3. spin_lock(bio_list_lock); set_bit(RBIO_RMW_LOCKED_BIT, flags); spin_unlock.
4. bitmap_clear(error_bitmap, 0, nr_sectors); Rbio::index_rbio_pages(rbio).
5. if !Rbio::is_full(rbio): Rbio::cache_rbio_pages(rbio); else clear_bit(RBIO_CACHE_READY_BIT, flags).
6. for sectornr in 0..stripe_nsectors: Rbio::generate_pq_vertical(rbio, sectornr).
7. Rbio::assemble_write_bios(rbio, &bio_list).
8. Rbio::submit_writes(rbio, &bio_list).
9. wait_event(io_wait, stripes_pending == 0).
10. For each vertical stripe: if Rbio::get_vertical_errors > max_errors: ret = -EIO.
11. Rbio::orig_end_io(rbio, ret).

`Rbio::recover(rbio)`:
1. fill_data_csums(rbio) (only for data block groups).
2. alloc_rbio_pages(rbio); index_rbio_pages(rbio).
3. Rbio::assemble_scrub_reads-like loop to read every needed sector.
4. submit_read_wait_bio_list.
5. recover_sectors(rbio).
6. Rbio::orig_end_io(rbio, ret).

`Rbio::recover_sectors(rbio)`:
1. pointers = alloc(real_stripes); unmap_array = alloc(real_stripes).
2. if operation == ReadRebuild: set RBIO_RMW_LOCKED_BIT atomically under bio_list_lock.
3. Rbio::index_rbio_pages(rbio).
4. for sectornr in 0..stripe_nsectors: Rbio::recover_vertical(rbio, sectornr, pointers, unmap_array).

`Rbio::generate_pq_vertical_step(rbio, sector_nr, step_nr)`:
1. has_qstripe = (bioc.map_type & BTRFS_BLOCK_GROUP_RAID6).
2. for stripe in 0..nr_data: pointers[stripe] = kmap_local_paddr(sector_paddr_in_rbio(rbio, stripe, sector_nr, step_nr, 0)).
3. pointers[nr_data] = kmap_local_paddr(rbio_pstripe_paddr(rbio, sector_nr, step_nr)).
4. if has_qstripe:
   - pointers[nr_data+1] = kmap_local_paddr(rbio_qstripe_paddr(rbio, sector_nr, step_nr)).
   - raid6_call.gen_syndrome(real_stripes, step, pointers).
5. else: memcpy(pointers[nr_data], pointers[0], step); xor_gen(pointers[nr_data], pointers + 1, nr_data - 1, step).
6. kunmap_local in reverse.

`Rbio::recover_vertical_step(rbio, sector_nr, step_nr, faila, failb, pointers, unmap_array)`:
1. Setup pointers from bio (ReadRebuild) or stripe paddrs (other ops).
2. if RAID6:
   - if failb < 0: if faila == nr_data: cleanup; else goto pstripe.
   - if failb == real_stripes - 1: if faila == real_stripes - 2: cleanup; else goto pstripe.
   - if failb == real_stripes - 2: raid6_datap_recov(real_stripes, step, faila, pointers).
   - else: raid6_2data_recov(real_stripes, step, faila, failb, pointers).
3. else /* RAID5 or RAID6 single-data via P */:
   - pstripe: memcpy(pointers[faila], pointers[nr_data], step); rotate so faila holds parity slot; xor_gen(p, pointers, nr_data - 1, step).
4. cleanup: kunmap_local each unmap_array entry.

`Rbio::lock_stripe_add(rbio)` (return: 0 owns / 1 plugged / merge sentinel):
1. bucket = hash_64(bioc.full_stripe_logical >> 16, BTRFS_STRIPE_HASH_TABLE_BITS).
2. h = stripe_hash_table.table[bucket].
3. spin_lock(h.lock).
4. for found in h.hash_list:
   - if found.full_stripe_logical == bioc.full_stripe_logical:
     - if Rbio::can_merge(found, rbio): Rbio::merge(found, rbio); spin_unlock; free rbio; return MERGED.
     - else list_add_tail(rbio.plug_list, found.plug_list); spin_unlock; return PLUGGED.
5. list_add(rbio.hash_list, h.hash_list); spin_unlock; return OWNED.

`Rbio::unlock_stripe(rbio)`:
1. spin_lock(h.lock).
2. list_del_init(rbio.hash_list).
3. next = first(rbio.plug_list).
4. if next: list_del(next.plug_list); spin_unlock; start_async_work(next, locked-variant).
5. else if RBIO_CACHE_READY_BIT set ∧ !error: cache_rbio(rbio); spin_unlock.
6. else: spin_unlock; free_raid_bio(rbio).

`Rbio::scrub(rbio)`:
1. alloc_rbio_essential_pages(rbio).
2. bitmap_clear(error_bitmap, 0, nr_sectors).
3. Rbio::assemble_scrub_reads(rbio).
4. Rbio::recover_scrub(rbio) — refuse repair if scrubp is the corrupted parity.
5. Rbio::finish_parity_scrub(rbio) — regenerate P/Q, compare, overwrite mismatched parity.
6. wait_event(io_wait, stripes_pending == 0).
7. For each vertical stripe: max_errors check.
8. Rbio::orig_end_io(rbio, ret).

`Rbio::finish_parity_scrub(rbio)` (algorithmic core):
1. has_qstripe = (real_stripes - nr_data == 2).
2. is_replace = (bioc.replace_nr_stripes ∧ bioc.replace_stripe_src == scrubp) ? copy dbitmap to finish_pbitmap.
3. clear_bit(RBIO_CACHE_READY_BIT, flags).
4. Allocate temp P page; if RAID6 also temp Q page; kmap pointers[nr_data] and pointers[real_stripes-1].
5. for each sectornr ∈ dbitmap: Rbio::verify_one_parity_sector(rbio, pointers, sectornr).
6. kunmap + free temp pages.
7. for each sectornr ∈ dbitmap: rbio_add_io_paddrs(WRITE) to scrubp.
8. if is_replace: for each sectornr ∈ finish_pbitmap: also write to real_stripes (replace target).
9. submit_write_bios.

### Out of Scope

- `lib/xor.c` per-CPU XOR ops (per-arch xor_block_*) — separate Tier-3 if expanded.
- `lib/raid6/` Reed-Solomon implementation (per-arch tables, SSE/AVX/NEON kernels) — separate Tier-3 if expanded.
- `fs/btrfs/scrub.c` (covered in sibling `scrub.md`).
- `fs/btrfs/volumes.c` chunk allocator + `btrfs_io_context` construction (covered in `volumes.md` if expanded).
- `fs/btrfs/dev-replace.c` device replace orchestration (covered in `dev-replace.md` if expanded).
- `fs/btrfs/raid-stripe-tree.c` write-hole-free per-stripe tracking (covered separately if expanded).
- `fs/btrfs/disk-io.c` bio submission helpers (covered in `disk-io.md`).
- Implementation code.

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct btrfs_raid_bio` | per-rbio state (data + P/Q paddrs, bitmaps, refs) | `Rbio` |
| `enum btrfs_rbio_ops` | WRITE / READ_REBUILD / PARITY_SCRUB | `RbioOp` |
| `struct btrfs_stripe_hash` | per-bucket hash + lock | `StripeHash` |
| `struct btrfs_stripe_hash_table` | per-fs full-stripe hash + LRU | `StripeHashTable` |
| `btrfs_alloc_stripe_hash_table()` | per-fs init | `StripeHashTable::alloc` |
| `btrfs_free_stripe_hash_table()` | per-fs teardown | `StripeHashTable::free` |
| `alloc_rbio()` | per-rbio constructor | `Rbio::alloc` |
| `free_raid_bio()` / `free_raid_bio_pointers()` | per-rbio teardown via refcount | `Rbio::drop` |
| `rbio_add_bio()` | per-bio merge into rbio + dbitmap update | `Rbio::add_bio` |
| `rbio_is_full()` | per-full-stripe check | `Rbio::is_full` |
| `rbio_can_merge()` | per-merge eligibility | `Rbio::can_merge` |
| `merge_rbio()` | per-bio_list splice into existing rbio | `Rbio::merge` |
| `lock_stripe_add()` | per-hash insert + serialize | `Rbio::lock_stripe_add` |
| `unlock_stripe()` | per-hash remove + hand-off | `Rbio::unlock_stripe` |
| `cache_rbio()` / `cache_rbio_pages()` | per-LRU cache insert | `Rbio::cache` |
| `steal_rbio()` / `steal_rbio_page()` | per-cache-hit page reuse | `Rbio::steal` |
| `__remove_rbio_from_cache()` / `remove_rbio_from_cache()` | per-LRU evict | `Rbio::cache_evict` |
| `btrfs_clear_rbio_cache()` | per-fs flush | `StripeHashTable::clear` |
| `raid56_parity_write()` | per-FS-write entry | `Rbio::write_entry` |
| `raid56_parity_recover()` | per-rebuild-read entry | `Rbio::recover_entry` |
| `raid56_parity_alloc_scrub_rbio()` | per-scrub rbio constructor | `Rbio::alloc_scrub` |
| `raid56_parity_submit_scrub_rbio()` | per-scrub submit | `Rbio::submit_scrub` |
| `raid56_parity_cache_data_folios()` | per-scrub pre-fill known-good data | `Rbio::cache_data_folios` |
| `rmw_rbio()` | per-RMW state machine | `Rbio::rmw` |
| `rmw_rbio_work()` / `_locked()` | per-workqueue entry | `Rbio::rmw_work` |
| `rmw_assemble_write_bios()` | per-write-bio assembly | `Rbio::assemble_write_bios` |
| `rmw_read_wait_recover()` | per-RMW read + recover | `Rbio::read_wait_recover` |
| `recover_rbio()` | per-recovery driver | `Rbio::recover` |
| `recover_sectors()` | per-stripe recovery loop | `Rbio::recover_sectors` |
| `recover_vertical()` / `recover_vertical_step()` | per-vertical (P/P+Q) reconstruction | `Rbio::recover_vertical` |
| `recover_rbio_work()` / `_locked()` | per-workqueue recovery | `Rbio::recover_work` |
| `generate_pq_vertical()` / `_step()` | per-P/Q gen (XOR + raid6 syndrome) | `Rbio::generate_pq_vertical` |
| `verify_one_sector()` | per-csum verify of rebuilt sector | `Rbio::verify_sector` |
| `verify_one_parity_step()` / `_sector()` | per-scrub: compare gen'd parity vs disk | `Rbio::verify_parity` |
| `finish_parity_scrub()` | per-scrub: write regenerated parity | `Rbio::finish_parity_scrub` |
| `scrub_rbio()` | per-scrub state machine | `Rbio::scrub` |
| `recover_scrub_rbio()` | per-scrub: recover data before parity check | `Rbio::recover_scrub` |
| `scrub_assemble_read_bios()` | per-scrub read assembly | `Rbio::assemble_scrub_reads` |
| `fill_data_csums()` | per-rbio: load data csums from csum-tree | `Rbio::fill_data_csums` |
| `set_rbio_range_error()` | per-bio: mark missing-device sectors | `Rbio::set_range_error` |
| `rbio_update_error_bitmap()` | per-endio error tracking | `Rbio::update_error_bitmap` |
| `get_rbio_vertical_errors()` | per-vertical-stripe error count + faila/failb | `Rbio::get_vertical_errors` |
| `rbio_orig_end_io()` | per-rbio: complete higher-layer bios | `Rbio::orig_end_io` |
| `rbio_endio_bio_list()` | per-bio-list completion | `Rbio::endio_bio_list` |
| `submit_read_wait_bio_list()` | per-read submit + wait | `Rbio::submit_read_wait` |
| `submit_write_bios()` | per-write submit | `Rbio::submit_writes` |
| `raid_wait_read_end_io()` / `raid_wait_write_end_io()` | per-bio endio | `Rbio::endio_read` / `endio_write` |
| `struct btrfs_plug_cb` | per-task plug for batching small writes | `RbioPlug` |
| `raid_unplug()` | per-plug flush on schedule | `RbioPlug::unplug` |
| `plug_cmp()` | per-plug sort by full_stripe_logical | `RbioPlug::cmp` |
| `xor_gen` | per-P-stripe XOR (`lib/xor.c`) | re-exported |
| `raid6_call.gen_syndrome` | per-Q-stripe RS syndrome (`lib/raid6/`) | re-exported |
| `raid6_datap_recov` / `raid6_2data_recov` | per-RAID6 reconstruction | re-exported |
| `RAID5_P_STRIPE` / `RAID6_Q_STRIPE` | sentinel stripe numbers | constants |
| `INVALID_PADDR` | sentinel phys_addr_t (~0) | constant |
| `BTRFS_STRIPE_LEN` (64 KiB) | per-stripe length | constant |
| `BTRFS_STRIPE_HASH_TABLE_BITS` | per-hash size | constant |

### compatibility contract

REQ-1: struct btrfs_raid_bio:
- bioc: per-btrfs_io_context (chunk mapping + stripes[] + map_type + replace_*).
- hash_list, stripe_cache: per-stripe-hash + per-LRU-cache linkage.
- work: per-workqueue work_struct.
- bio_list, bio_list_lock: per-higher-layer-bio queue.
- plug_list: per-stripe-hand-off chain (also reused by per-task plug).
- flags: per-rbio bits (RBIO_RMW_LOCKED_BIT, RBIO_CACHE_READY_BIT, RBIO_CACHE_BIT, RBIO_HOLD_BBIO_MAP_BIT, RBIO_*_BIT).
- operation: per-`BTRFS_RBIO_WRITE` / `_READ_REBUILD` / `_PARITY_SCRUB`.
- nr_pages, nr_sectors: per-full-stripe count (data + P/Q).
- nr_data, real_stripes: per-data count, per-total count (real_stripes = nr_data + 1 RAID5 or + 2 RAID6).
- stripe_npages, stripe_nsectors: per-stripe page / sector count.
- sector_nsteps: per-sector step count (1 when sectorsize ≤ PAGE_SIZE; sectorsize/PAGE_SIZE for bs > ps).
- scrubp: per-PARITY_SCRUB: which parity stripe is being scrubbed.
- bio_list_bytes: per-rbio total bytes from higher-layer bios (for rbio_is_full).
- refs: per-rbio refcount.
- stripes_pending: per-rbio in-flight bio count.
- io_wait: per-rbio wait-queue (drained when stripes_pending == 0).
- dbitmap: per-vertical-stripe data-present bitmap (stripe_nsectors bits).
- finish_pbitmap: per-finish_parity_scrub temp bitmap.
- stripe_pages: per-rbio internal pages (nr_pages entries; can be NULL when bio supplies the page).
- bio_paddrs: per-bio sector paddr lookup (nr_sectors * sector_nsteps entries; INVALID_PADDR when uncovered).
- stripe_paddrs: per-internal-page paddr lookup (same dimensions; INVALID_PADDR when unallocated).
- stripe_uptodate_bitmap: per-sector uptodate bit.
- finish_pointers: per-stripe void* array for P/Q generation kmap.
- error_bitmap: per-sector error bit (IO + csum + missing device).
- csum_buf, csum_bitmap: per-data-sector csum cache (only for data block groups, not metadata).

REQ-2: alloc_rbio(fs_info, bioc) -> rbio:
- real_stripes = bioc.num_stripes - bioc.replace_nr_stripes.
- stripe_npages = BTRFS_STRIPE_LEN >> PAGE_SHIFT.
- num_pages = stripe_npages * real_stripes.
- stripe_nsectors = BTRFS_STRIPE_LEN >> sectorsize_bits.
- num_sectors = stripe_nsectors * real_stripes.
- step = min(sectorsize, PAGE_SIZE).
- sector_nsteps = sectorsize / step.
- ASSERT IS_ALIGNED(PAGE_SIZE, sectorsize) ∨ IS_ALIGNED(sectorsize, PAGE_SIZE).
- ASSERT stripe_nsectors ≤ BITS_PER_LONG.
- ASSERT 2 ≤ real_stripes ≤ U8_MAX.
- Allocate rbio + stripe_pages + bio_paddrs + stripe_paddrs + finish_pointers + error_bitmap + stripe_uptodate_bitmap (GFP_NOFS).
- Initialize all bio_paddrs / stripe_paddrs to INVALID_PADDR.
- nr_data = real_stripes - btrfs_nr_parity_stripes(bioc.map_type); ASSERT nr_data > 0.
- refcount_set(refs, 1); atomic_set(stripes_pending, 0).
- Return rbio or ERR_PTR(-ENOMEM).

REQ-3: rbio_add_bio(rbio, orig_bio):
- ASSERT orig_logical ∈ [full_stripe_logical, full_stripe_logical + nr_data * BTRFS_STRIPE_LEN).
- bio_list_add(rbio.bio_list, orig_bio).
- bio_list_bytes += orig_bio.bi_iter.bi_size.
- For each sector covered: set bit in dbitmap.

REQ-4: raid56_parity_write(bio, bioc):
- rbio = alloc_rbio(fs_info, bioc).
- if IS_ERR(rbio): set bio status, bio_endio, return.
- rbio.operation = BTRFS_RBIO_WRITE.
- rbio_add_bio(rbio, bio).
- /* Per-plug batching for sub-stripe writes */
- if !rbio_is_full(rbio):
  - cb = blk_check_plugged(raid_unplug, ...).
  - if cb: append rbio to plug.rbio_list; return.
- /* Full stripe or no plug — queue RMW */
- start_async_work(rbio, rmw_rbio_work).

REQ-5: raid56_parity_recover(bio, bioc, mirror_num):
- rbio = alloc_rbio(fs_info, bioc).
- rbio.operation = BTRFS_RBIO_READ_REBUILD.
- rbio_add_bio(rbio, bio).
- set_rbio_range_error(rbio, bio) — marks bio's sectors as errored to drive recovery.
- if mirror_num > 2: set_rbio_raid6_extra_error(rbio, mirror_num) — retries with selected extra failed stripe.
- start_async_work(rbio, recover_rbio_work).

REQ-6: rmw_rbio(rbio) — RMW state machine:
- alloc_rbio_parity_pages(rbio) — P/Q pages always needed.
- if !rbio_is_full(rbio) ∧ need_read_stripe_sectors(rbio):
  - alloc_rbio_data_pages(rbio).
  - index_rbio_pages(rbio).
  - rmw_read_wait_recover(rbio) — read every sector, csum-verify, recover failures.
- /* Lock the stripe — no further bios may merge */
- set_bit(RBIO_RMW_LOCKED_BIT, rbio.flags).
- bitmap_clear(rbio.error_bitmap, 0, nr_sectors).
- index_rbio_pages(rbio).
- if !rbio_is_full(rbio): cache_rbio_pages(rbio) /* full-stripe cache */
- else: clear_bit(RBIO_CACHE_READY_BIT, rbio.flags).
- for sectornr in 0..stripe_nsectors: generate_pq_vertical(rbio, sectornr).
- rmw_assemble_write_bios(rbio, &bio_list).
- submit_write_bios(rbio, &bio_list).
- wait_event(io_wait, stripes_pending == 0).
- /* Final tolerance check */
- for each vertical sector: if get_rbio_vertical_errors > max_errors → ret = -EIO.
- rbio_orig_end_io(rbio, errno_to_blk_status(ret)).

REQ-7: rmw_read_wait_recover(rbio):
- fill_data_csums(rbio) — load CRC32C/XXHASH/SHA256/BLAKE2 from csum tree for the data range.
- For total_sector_nr ∈ 0..nr_sectors:
  - paddrs = rbio_stripe_paddrs(rbio, stripe, sectornr).
  - rbio_add_io_paddrs(rbio, &bio_list, paddrs, stripe, sectornr, REQ_OP_READ).
- submit_read_wait_bio_list(rbio, &bio_list).
- recover_sectors(rbio) — reconstruct any errored sectors.

REQ-8: fill_data_csums(rbio):
- Skip if !(bioc.map_type & BTRFS_BLOCK_GROUP_DATA) ∨ (bioc.map_type & BTRFS_BLOCK_GROUP_METADATA) /* avoids deadlock holding full stripe lock */.
- Allocate csum_buf (nr_data * stripe_nsectors * csum_size) + csum_bitmap.
- csum_root = btrfs_csum_root(fs_info, full_stripe_logical).
- btrfs_lookup_csums_bitmap(csum_root, NULL, start, start + len - 1, csum_buf, csum_bitmap).
- On error: warn `sub-stripe write for full stripe %llu is not safe`, free both, continue without csum.

REQ-9: generate_pq_vertical(rbio, sectornr):
- For each step ∈ 0..sector_nsteps: generate_pq_vertical_step(rbio, sectornr, step).
- set stripe_uptodate_bitmap bit for P sector (nr_data, sectornr); if RAID6 also Q sector (nr_data+1).

REQ-10: generate_pq_vertical_step(rbio, sector_nr, step_nr):
- has_qstripe = (bioc.map_type & BTRFS_BLOCK_GROUP_RAID6).
- For stripe ∈ 0..nr_data: pointers[stripe] = kmap_local_paddr(sector_paddr_in_rbio(stripe, sector_nr, step_nr, 0)).
- pointers[nr_data] = kmap_local_paddr(rbio_pstripe_paddr(rbio, sector_nr, step_nr)).
- if has_qstripe:
  - pointers[nr_data+1] = kmap_local_paddr(rbio_qstripe_paddr(rbio, sector_nr, step_nr)).
  - raid6_call.gen_syndrome(real_stripes, step, pointers) /* lib/raid6 Reed-Solomon-Q + P */.
- else /* RAID5 */:
  - memcpy(pointers[nr_data], pointers[0], step).
  - xor_gen(pointers[nr_data], pointers + 1, nr_data - 1, step) /* lib/xor.c */.
- kunmap_local each pointer.

REQ-11: recover_vertical(rbio, sector_nr, pointers, unmap_array):
- found_errors, faila, failb = get_rbio_vertical_errors(rbio, sector_nr).
- if found_errors == 0: return 0.
- if found_errors > bioc.max_errors: return -EIO.
- For each step ∈ 0..sector_nsteps: recover_vertical_step(rbio, sector_nr, step, faila, failb, pointers, unmap_array).
- if faila ≥ 0: verify_one_sector(rbio, faila, sector_nr); set stripe_uptodate_bitmap bit.
- if failb ≥ 0: verify_one_sector(rbio, failb, sector_nr); set stripe_uptodate_bitmap bit.

REQ-12: recover_vertical_step (algorithmic core):
- Setup pointers/unmap_array from READ_REBUILD bio paddrs or stripe paddrs.
- if RAID6:
  - if failb < 0 (single failure):
    - if faila == nr_data (P only): skip (no data lost).
    - else: goto pstripe (rebuild from P via XOR).
  - if failb == real_stripes - 1 (Q failed):
    - if faila == real_stripes - 2 (P+Q both failed): skip (data is fine).
    - else: goto pstripe (single data failure + good P).
  - if failb == real_stripes - 2 (data + P): raid6_datap_recov(real_stripes, step, faila, pointers).
  - else (2 data failures): raid6_2data_recov(real_stripes, step, faila, failb, pointers).
- else /* RAID5 */:
  - ASSERT failb == -1.
  - pstripe: memcpy(pointers[faila], pointers[nr_data], step).
  - Rotate pointers so faila slot now holds parity, run xor_gen(p, pointers, nr_data - 1, step).
- kunmap_local each unmap_array entry.

REQ-13: verify_one_sector(rbio, stripe_nr, sector_nr):
- if !csum_bitmap ∨ !csum_buf: return 0.
- if stripe_nr ≥ nr_data: return 0 /* P/Q not covered by data csum */.
- paddrs = (operation == READ_REBUILD) ? sector_paddrs_in_rbio(stripe, sector, 0) : rbio_stripe_paddrs.
- csum_expected = csum_buf + (stripe_nr * stripe_nsectors + sector_nr) * csum_size.
- btrfs_calculate_block_csum_pages(fs_info, paddrs, csum_buf_local).
- if memcmp(csum_buf_local, csum_expected, csum_size) != 0: return -EIO.

REQ-14: rmw_assemble_write_bios(rbio, &bio_list):
- ASSERT bitmap_weight(dbitmap, stripe_nsectors) ≥ 1.
- bitmap_clear(error_bitmap, 0, nr_sectors).
- For total_sector_nr ∈ 0..nr_sectors:
  - stripe = total_sector_nr / stripe_nsectors; sectornr = total_sector_nr % stripe_nsectors.
  - if !test_bit(sectornr, dbitmap): continue.
  - paddrs = (stripe < nr_data) ? sector_paddrs_in_rbio(stripe, sectornr, 1) : rbio_stripe_paddrs.
  - if paddrs == NULL (data sector not covered by bio): continue.
  - rbio_add_io_paddrs(rbio, bio_list, paddrs, stripe, sectornr, REQ_OP_WRITE).
- if bioc.replace_nr_stripes: duplicate writes to replace_stripe_src target.

REQ-15: lock_stripe_add(rbio):
- bucket = rbio_bucket(rbio) = hash_64(full_stripe_logical >> 16, BTRFS_STRIPE_HASH_TABLE_BITS).
- h = stripe_hash_table.table[bucket].
- spin_lock(h.lock).
- Scan h.hash_list for an existing rbio matching same full_stripe_logical:
  - If found ∧ can_merge: merge_rbio(found, rbio); free new rbio; return -EEXIST-equivalent (caller treats as "absorbed").
  - If found ∧ !can_merge: list_add_tail(rbio.plug_list, found.plug_list); return 1 (queued for hand-off).
- Otherwise: list_add(rbio.hash_list, h.hash_list); return 0 (rbio owns the stripe).

REQ-16: unlock_stripe(rbio):
- /* Hand the stripe lock to the next pending rbio if any */
- spin_lock(h.lock).
- list_del_init(rbio.hash_list).
- /* Walk plug_list: pick first plugged rbio that wants this stripe */
- if any pending: dequeue and start_async_work(next, next.operation == READ_REBUILD ? recover_rbio_work_locked : rmw_rbio_work_locked).
- /* Otherwise add to LRU cache if eligible */
- if test_bit(RBIO_CACHE_READY_BIT, rbio.flags) ∧ !(error in stripe): cache_rbio(rbio).
- spin_unlock(h.lock).

REQ-17: cache_rbio_pages(rbio):
- alloc_rbio_pages(rbio).
- For i ∈ 0..nr_sectors:
  - if bio_paddrs[i*sector_nsteps] == INVALID_PADDR: skip (sub-stripe sector not covered).
  - memcpy_from_bio_to_stripe(rbio, i) — copy data from higher-layer bio into rbio's internal pages.
  - set_bit(i, stripe_uptodate_bitmap).
- set_bit(RBIO_CACHE_READY_BIT, rbio.flags).

REQ-18: scrub path — raid56_parity_alloc_scrub_rbio + scrub_rbio:
- alloc_scrub_rbio: alloc_rbio + operation = BTRFS_RBIO_PARITY_SCRUB; locate scrubp by scanning bioc.stripes[nr_data..real_stripes] for scrub_dev; copy caller's dbitmap.
- raid56_parity_submit_scrub_rbio: if !lock_stripe_add: start_async_work(rbio, scrub_rbio_work_locked).
- scrub_rbio:
  - alloc_rbio_essential_pages — only allocate pages for vertical stripes with dbitmap set.
  - scrub_assemble_read_bios — read every data + parity sector needed, submit + wait.
  - recover_scrub_rbio — recover failed sectors (with the constraint that the scrubbed parity cannot itself be used to repair).
  - finish_parity_scrub — recompute P/Q for the dbitmap range; if disk differs, overwrite parity; submit writes.
  - Final max_errors check; rbio_orig_end_io.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `rbio_refcount_balanced` | INVARIANT | per-alloc_rbio: refs initialized to 1; per-free_raid_bio: dec-and-test only frees on zero. |
| `paddrs_invalid_sentinel` | INVARIANT | per-alloc_rbio: every entry of bio_paddrs / stripe_paddrs is INVALID_PADDR until index_*_pages runs. |
| `nr_data_positive` | INVARIANT | per-alloc_rbio: nr_data ≥ 1 ∧ real_stripes ≥ 2. |
| `stripe_lock_held_for_rmw` | INVARIANT | per-rmw_rbio: RBIO_RMW_LOCKED_BIT set under bio_list_lock before generate_pq_vertical. |
| `hash_lock_held_for_lock_stripe_add` | INVARIANT | per-lock_stripe_add: stripe_hash.lock held across the scan + insert. |
| `max_errors_not_exceeded` | INVARIANT | per-recover_vertical: found_errors > max_errors ⟹ -EIO, no reconstruction attempted. |
| `csum_only_for_data` | INVARIANT | per-verify_one_sector: stripe_nr ≥ nr_data ⟹ return 0 (P/Q not csum-protected). |
| `kmap_balanced` | INVARIANT | per-generate_pq_vertical_step / recover_vertical_step: every kmap_local_paddr is matched by kunmap_local. |
| `parity_overwrite_only` | INVARIANT | per-finish_parity_scrub: only scrubp parity gets rewritten, never data sectors. |
| `replace_dup_only_to_target` | INVARIANT | per-rmw_assemble_write_bios / finish_parity_scrub: replace duplicates target stripe == real_stripes (replace target). |

### Layer 2: TLA+

`fs/btrfs/raid56.tla`:
- Per-rbio lifecycle: ALLOC → ADD_BIO → LOCK → (RMW | RECOVER | SCRUB) → SUBMIT → ENDIO → UNLOCK → CACHE | FREE.
- Per-hash bucket: at-most-one OWNED rbio for a given full_stripe_logical; others MERGED or PLUGGED.
- Properties:
  - `safety_no_concurrent_rmw_same_stripe` — per-lock_stripe_add: two rbios with same full_stripe_logical never both reach OWNED simultaneously.
  - `safety_handoff_no_loss` — per-unlock_stripe: every PLUGGED rbio eventually transitions to OWNED.
  - `safety_max_errors_bounds_recovery` — per-recover_vertical: found_errors > max_errors ⟹ EIO, never silent.
  - `safety_csum_verify_before_endio` — per-rmw_read_wait_recover: every rebuilt data sector csum-verified before endio.
  - `safety_parity_scrub_no_data_clobber` — per-scrub_rbio: write set is a subset of {scrubp, replace_target}.
  - `liveness_per_rbio_terminates` — every rbio enters ENDIO in finite steps.
  - `liveness_per_plug_drains` — every PLUGGED rbio eventually OWNS or MERGES.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Rbio::alloc` post: rbio.refs == 1 ∧ all paddrs == INVALID_PADDR ∧ nr_data ≥ 1 | `Rbio::alloc` |
| `Rbio::add_bio` post: dbitmap bit set for every sector in [orig_logical, orig_logical+orig_len) ∧ bio_list_bytes incremented | `Rbio::add_bio` |
| `Rbio::generate_pq_vertical_step` post: P = XOR(data); Q = raid6 gen_syndrome(data) | `Rbio::generate_pq_vertical_step` |
| `Rbio::recover_vertical` post: faila/failb sectors reconstructed ∧ csum-verified ∧ uptodate bits set | `Rbio::recover_vertical` |
| `Rbio::lock_stripe_add` post: ret ∈ {OWNED, PLUGGED, MERGED} ∧ exactly one rbio per stripe in OWNED | `Rbio::lock_stripe_add` |
| `Rbio::rmw` post: stripes_pending == 0 ∧ rbio_orig_end_io called | `Rbio::rmw` |
| `Rbio::finish_parity_scrub` post: writes go only to scrubp (+replace_target if is_replace) | `Rbio::finish_parity_scrub` |
| `Rbio::cache_rbio_pages` post: RBIO_CACHE_READY_BIT set ∧ stripe_uptodate_bitmap covers all bio-covered sectors | `Rbio::cache_rbio_pages` |

### Layer 4: Verus/Creusot functional

`Per-write (raid56_parity_write) → alloc_rbio → optional plug → rmw_rbio (read missing data, csum-verify, generate P/Q, write changes + parity) → end_io` semantic equivalence: per Documentation/filesystems/btrfs.rst (RAID5/6) and `Documentation/admin-guide/btrfs.rst`.

`Per-recover (raid56_parity_recover) → alloc_rbio → recover_sectors → recover_vertical (XOR for RAID5 / raid6_2data_recov | raid6_datap_recov for RAID6) → verify_one_sector → end_io` matches `lib/raid6/algos.c` and `lib/xor.c` semantics.

`Per-scrub (raid56_parity_alloc_scrub_rbio → raid56_parity_submit_scrub_rbio → scrub_rbio → finish_parity_scrub)` semantic equivalence: parity mismatch ⟹ parity overwrite; data corruption + scrubbed parity ⟹ EIO.

### hardening

(Inherits row-1 features from `fs/btrfs/00-overview.md` § Hardening.)

RAID5/6 reinforcement:

- **Per-csum verify of every reconstructed sector** — defense against per-undetected silent corruption returned to upper layer (verify_one_sector before endio).
- **Per-INVALID_PADDR sentinel for unallocated sectors** — defense against per-NULL-page kmap and per-uninitialized-stripe XOR.
- **Per-max_errors hard ceiling in recover_vertical** — defense against per-overconfident reconstruction on RAID5 dual-disk loss or RAID6 triple-disk loss.
- **Per-stripe-hash serialization (lock_stripe_add)** — defense against per-concurrent-RMW racing P/Q updates on the same full stripe.
- **Per-RBIO_RMW_LOCKED_BIT before parity gen** — defense against per-late-arriving bio mutating data after csum-verify.
- **Per-data-only csum loading (skip metadata block groups)** — defense against per-deadlock on csum-tree read while holding stripe lock.
- **Per-replace target written only when replace_nr_stripes set** — defense against per-stray-write to dev-replace target.
- **Per-parity-only repair in scrub** — defense against per-data-clobber by parity-mismatch repair (writes restricted to scrubp + replace_target).
- **Per-write-hole acknowledgement** — Documentation/filesystems/btrfs.rst explicitly warns; RMW csum-verifies inputs, scrub re-verifies P/Q; user advised to combine with `raid1c3`/`raid1c4` metadata.
- **Per-plug sort by full_stripe_logical** — defense against per-cache-miss thrash during bulk writes (plug_cmp orders rbios before unplug).
- **Per-LRU stripe cache eviction is idempotent** — defense against per-double-free or per-UAF on cache replacement.
- **Per-bio_list_bytes vs rbio_is_full check** — defense against per-treat-partial-as-full RMW path skip (would compute parity over stale data).
- **Per-stripe_uptodate_bitmap gate on cache reuse** — defense against per-stale-cache reuse for steal_rbio after partial RMW.

