---
title: "Tier-3: mm/swapfile.c — swap-file management"
tags: ["tier-3", "mm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

The **swap subsystem** manages per-device swap areas: backing storage for evicted anonymous and tmpfs pages. Per-`swapon(specialfile, flags)` syscall registers a swap area (block device or regular file with `SWAPSPACE2` magic) into `swap_info[MAX_SWAPFILES]`, priority-ordered in `swap_active_head` / `swap_avail_head` plist; per-`swapoff(specialfile)` reverses the operation after evicting (`try_to_unuse`) all in-use entries back into resident memory. Per-swap-slot identity = `swp_entry_t` = (SWP_TYPE | SWP_OFFSET) packed into a single `unsigned long`, with `swp_type()` selecting `swap_info[]` and `swp_offset()` selecting the in-area page. Per-`swap_info_struct` carries: backing file, plist priority, max/pages count, flags (SWP_USED | SWP_WRITEOK | SWP_SOLIDSTATE | SWP_DISCARDABLE | SWP_PAGE_DISCARD | SWP_AREA_DISCARD | SWP_FS_OPS | SWP_BLKDEV | SWP_STABLE_WRITES | SWP_SYNCHRONOUS_IO | SWP_ACTIVATED), rb-tree of `swap_extent` (page-offset → block-sector), array of `swap_cluster_info` (SWAPFILE_CLUSTER = 256-or-PMD-sized chunks with per-cluster spinlock + free/nonfull/full/frag/discard plist), per-CPU cluster cache `percpu_swap_cluster.si[SWAP_NR_ORDERS]`, percpu-ref liveness counter, discard + reclaim workqueues, zswap pool offset. Per-SSD-vs-HDD: SSD (SWP_SOLIDSTATE) uses per-CPU clusters for low contention; HDD uses `global_cluster` (single sequential pointer) for head-locality. Per-allocation entry point `folio_alloc_swap(folio)` → fast (per-CPU cluster hit) or slow (rotate `swap_avail_head` and pick another device). Per-zswap hook in swap_writepage / swap_read_folio; per-`zswap_swapon` / `zswap_swapoff` ties zswap allocator lifecycle to swap area. Per-hibernation slots: `swap_alloc_hibernation_slot` / `swap_free_hibernation_slot` for kernel-snapshot. Critical for: pageout/pagein, memcg swap accounting, hibernation, oom-pressure relief.

This Tier-3 covers `mm/swapfile.c` (~3788 lines).

### Acceptance Criteria

- [ ] AC-1: swapon('/swap', 0): creates swap_info_struct, plist-inserted at DEF_SWAP_PRIO, /proc/swaps shows entry.
- [ ] AC-2: swapon('/swap', SWAP_FLAG_PREFER | 5): plist priority = 5; preferred before lower-prio devices.
- [ ] AC-3: swapon on block device with !bdev_rot: si.flags has SWP_SOLIDSTATE; per-CPU clusters used.
- [ ] AC-4: swapon on rotational device: si.flags has !SWP_SOLIDSTATE; global_cluster used; nr_rotate_swap incremented.
- [ ] AC-5: swapon with SWAP_FLAG_DISCARD on discard-capable bdev: SWP_DISCARDABLE | _AREA_DISCARD | _PAGE_DISCARD set; one-shot discard issued at swapon time.
- [ ] AC-6: swapon on file without SWAPSPACE2 magic: -EINVAL.
- [ ] AC-7: swapon on already-active file: -EBUSY (IS_SWAPFILE).
- [ ] AC-8: folio_alloc_swap on locked uptodate order-0 folio: returns 0; folio.swap is valid swp_entry_t; folio in swap-cache.
- [ ] AC-9: folio_alloc_swap on PMD-order folio with CONFIG_THP_SWAP=n: returns -EAGAIN.
- [ ] AC-10: folio_alloc_swap when all devices full: returns -ENOMEM.
- [ ] AC-11: swap_put_entries_direct on a 4-page range: per-slot swap-count decremented; slots freed when count → 0.
- [ ] AC-12: swap_entry_to_info(swp_entry(type, offset)) == swap_info[type] for valid type.
- [ ] AC-13: swap_folio_sector on a swap-cache folio: returns sector matching swap_extent rb-tree lookup + (offset - se.start_page).
- [ ] AC-14: swapoff: try_to_unuse evicts all entries back into mm; percpu_ref_kill + synchronize_rcu drains readers; SWP_USED cleared.
- [ ] AC-15: swapoff while pages still pinned: re-inserted on -EBUSY; plist restored.
- [ ] AC-16: get_swap_device(entry) ↔ put_swap_device(si): refcount balanced; si stable during read.
- [ ] AC-17: /proc/swaps poll: EPOLLIN | EPOLLPRI fires on swapon/swapoff (proc_poll_event tick).
- [ ] AC-18: zswap_swapon/swapoff lifecycle: per-type tree created on swapon, drained on swapoff; orphaned entries impossible.

### Architecture

```
struct SwapInfo {
  ty: u32,                              // SWP_TYPE
  flags: u32,                           // SWP_USED | SWP_WRITEOK | SWP_SOLIDSTATE | ...
  prio: i32,                            // DEF_SWAP_PRIO = -1, or 0..SWAP_FLAG_PRIO_MASK
  list: PlistNode,                      // -prio key
  avail_list: PlistNode,                // -prio key
  swap_file: Arc<File>,
  bdev: Option<*BlockDevice>,
  max: u32,                             // total pages including header
  pages: u32,                           // usable pages
  inuse_pages: AtomicI64,               // | SWAP_USAGE_OFFLIST_BIT
  lock: SpinLock,
  cluster_info: Option<KVBox<[SwapCluster]>>,
  global_cluster: Option<KBox<GlobalCluster>>,  // HDD only
  global_cluster_lock: SpinLock,                // HDD only
  free_clusters: ListHead,
  full_clusters: ListHead,
  discard_clusters: ListHead,
  nonfull_clusters: [ListHead; SWAP_NR_ORDERS],
  frag_clusters: [ListHead; SWAP_NR_ORDERS],
  swap_extent_root: RbRoot,             // rb-tree of SwapExtent
  discard_work: WorkStruct,
  reclaim_work: WorkStruct,
  users: PercpuRef,                     // read-side liveness
  comp: Completion,                     // posted when users → 0
  zeromap: Option<KVBox<[u64]>>,        // bitmap len = BITS_TO_LONGS(max)
}

struct SwapCluster {
  lock: SpinLock,
  count: u16,                           // 0..SWAPFILE_CLUSTER
  flags: u8,                            // CLUSTER_FLAG_FREE | _NONFULL | _FRAG | _FULL | _DISCARD
  order: u8,                            // 0..PMD_ORDER-1 when count > 0
  list: ListHead,
  table: Rcu<Option<*SwapTable>>,
  extend_table: Option<KBox<[u8; SWAPFILE_CLUSTER]>>,  // overflow counters
}

struct SwapExtent {
  rb_node: RbNode,
  start_page: u32,
  nr_pages: u32,
  start_block: u64,                     // sector_t
}

struct PercpuSwapCluster {
  si: [Option<*SwapInfo>; SWAP_NR_ORDERS],
  offset: [u32; SWAP_NR_ORDERS],        // SWAP_ENTRY_INVALID = 0
  lock: LocalLock,
}

struct SwapEntry { val: u64 }
impl SwapEntry {
  pub fn new(ty: u32, offset: u32) -> Self { Self { val: ((ty as u64) << SWP_TYPE_SHIFT) | (offset as u64) } }
  pub fn ty(self) -> u32     { ((self.val >> SWP_TYPE_SHIFT) & SWP_TYPE_MASK) as u32 }
  pub fn offset(self) -> u32 { (self.val & SWP_OFFSET_MASK) as u32 }
}
```

`Sys::swapon(specialfile, swap_flags) -> Result<()>`:
1. if !capable(CAP_SYS_ADMIN): return Err(EPERM).
2. if swap_flags & !SWAP_FLAGS_VALID: return Err(EINVAL).
3. si = SwapInfo::alloc()?  /* reserves swap_info[] slot, percpu_ref dead */
4. WorkStruct::init(&si.discard_work, Self::discard_worker).
5. WorkStruct::init(&si.reclaim_work, Self::reclaim_worker).
6. swap_file = File::open(specialfile, O_RDWR | O_LARGEFILE | O_EXCL)?.
7. SwapInfo::claim(si, inode)?  /* S_ISBLK or S_ISREG */
8. inode_lock(inode); if IS_SWAPFILE(inode): return Err(EBUSY).
9. folio = read_mapping_folio(mapping, 0, swap_file)?.
10. maxpages = SwapInfo::read_header(si, swap_header, inode)?.
11. si.max = maxpages; si.pages = maxpages - 1.
12. SwapInfo::setup_extents(si, swap_file, &mut span)?.
13. SwapInfo::setup_clusters(si, swap_header, maxpages)?.
14. swap_cgroup_swapon(si.ty, maxpages)?.
15. si.zeromap = Some(kvmalloc_array(BITS_TO_LONGS(maxpages), ..., __GFP_ZERO)?).
16. /* SSD vs HDD */
17. if bdev_rot(si.bdev) == false: si.flags |= SWP_SOLIDSTATE.
18. else: nr_rotate_swap.fetch_add(1).
19. if bdev_stable_writes(si.bdev): si.flags |= SWP_STABLE_WRITES.
20. if bdev_synchronous(si.bdev): si.flags |= SWP_SYNCHRONOUS_IO.
21. if (swap_flags & SWAP_FLAG_DISCARD) ∧ bdev_max_discard_sectors(si.bdev) > 0:
    - si.flags |= SWP_DISCARDABLE | SWP_AREA_DISCARD | SWP_PAGE_DISCARD.
    - if SWAP_FLAG_DISCARD_ONCE: si.flags &= !SWP_PAGE_DISCARD.
    - if SWAP_FLAG_DISCARD_PAGES: si.flags &= !SWP_AREA_DISCARD.
    - if SWP_AREA_DISCARD: Swap::discard_all(si).
22. zswap_swapon(si.ty, maxpages)?.
23. inode.i_flags |= S_SWAPFILE.
24. inode_drain_writes(inode)?.
25. prio = if (swap_flags & SWAP_FLAG_PREFER) { swap_flags & SWAP_FLAG_PRIO_MASK } else { DEF_SWAP_PRIO }.
26. si.prio = prio; si.list.prio = -prio; si.avail_list.prio = -prio.
27. SwapInfo::enable(si).  /* plist + percpu_ref_resurrect */
28. proc_poll_event.fetch_add(1); wake_up_interruptible(&proc_poll_wait).
29. Ok(()).

`Sys::swapoff(specialfile) -> Result<()>`:
1. if !capable(CAP_SYS_ADMIN): return Err(EPERM).
2. victim = File::open(specialfile, O_RDWR | O_LARGEFILE)?.
3. /* Find si */
4. spin_lock(&swap_lock).
5. p = plist_for_each(swap_active_head).find(|si| (si.flags & SWP_WRITEOK) ∧ si.swap_file.mapping == victim.mapping)?.
6. if !p: return Err(EINVAL).
7. /* VM accounting */
8. if !security_vm_enough_memory_mm(current.mm, p.pages): return Err(ENOMEM).
9. /* Depublish */
10. SwapInfo::avail_del(p, swapoff=true).  /* clears SWP_WRITEOK */
11. plist_del(&p.list, &swap_active_head).
12. nr_swap_pages.fetch_sub(p.pages).
13. total_swap_pages -= p.pages.
14. /* Drain in-flight allocators */
15. SwapInfo::wait_for_allocation(p).
16. /* Evict */
17. set_current_oom_origin().
18. err = Swap::try_to_unuse(p.ty).
19. clear_current_oom_origin().
20. if err: SwapInfo::reinsert(p); return Err(err).
21. /* Drain readers */
22. percpu_ref_kill(&p.users); synchronize_rcu(); wait_for_completion(&p.comp).
23. flush_work(&p.discard_work); flush_work(&p.reclaim_work).
24. Swap::flush_percpu_cluster(p).
25. SwapInfo::destroy_extents(p, p.swap_file).
26. if !(p.flags & SWP_SOLIDSTATE): nr_rotate_swap.fetch_sub(1).
27. Swap::drain_mmlist().
28. swap_file = p.swap_file.take(); zeromap = p.zeromap.take(); cluster_info = p.cluster_info.take().
29. p.max = 0.
30. arch_swap_invalidate_area(p.ty); zswap_swapoff(p.ty).
31. drop(global_cluster, zeromap, cluster_info).
32. swap_cgroup_swapoff(p.ty).
33. inode.i_flags &= !S_SWAPFILE.
34. filp_close(swap_file).
35. p.flags = 0.  /* now reusable */
36. proc_poll_event.fetch_add(1); wake_up_interruptible(&proc_poll_wait).
37. Ok(()).

`Swap::folio_alloc(folio: &Folio) -> Result<()>`:
1. order = folio_order(folio).
2. assert(folio_test_locked(folio) ∧ folio_test_uptodate(folio)).
3. if order > 0:
   - if !cfg!(CONFIG_THP_SWAP): return Err(EAGAIN).
   - if (1 << order) > SWAPFILE_CLUSTER: return Err(EINVAL).
4. loop {
5.   percpu_swap_cluster.lock.local_lock();
6.   if !SwapAlloc::fast(folio):
7.     SwapAlloc::slow(folio).
8.   percpu_swap_cluster.lock.local_unlock();
9.   if order == 0 ∧ !folio_test_swapcache(folio):
10.    if Swap::sync_discard(): continue;
11.  break;
12.}
13. if mem_cgroup_try_charge_swap(folio, folio.swap).is_err():
    - swap_cache_del_folio(folio).
14. if !folio_test_swapcache(folio): return Err(ENOMEM).
15. Ok(()).

`SwapAlloc::fast(folio: &Folio) -> bool`:
1. order = folio_order(folio).
2. si = this_cpu_read(percpu_swap_cluster.si[order]).
3. offset = this_cpu_read(percpu_swap_cluster.offset[order]).
4. if si.is_none() ∨ offset == SWAP_ENTRY_INVALID ∨ !SwapInfo::get_device_info(si.unwrap()): return false.
5. ci = swap_cluster_lock(si.unwrap(), offset).
6. if cluster_is_usable(ci, order):
   - if cluster_is_empty(ci): offset = cluster_offset(si.unwrap(), ci).
   - SwapAlloc::scan_cluster(si.unwrap(), ci, folio, offset).
7. else: swap_cluster_unlock(ci).
8. put_swap_device(si.unwrap()).
9. folio_test_swapcache(folio).

`SwapAlloc::slow(folio: &Folio)`:
1. spin_lock(&swap_avail_lock).
2. 'start_over: loop {
3.   for (si, next) in plist_for_each_safe(&swap_avail_head):
4.     plist_requeue(&si.avail_list, &swap_avail_head).
5.     spin_unlock(&swap_avail_lock).
6.     if SwapInfo::get_device_info(si):
7.       SwapAlloc::cluster_alloc_entry(si, folio).
8.       put_swap_device(si).
9.       if folio_test_swapcache(folio): return;
10.      if folio_test_large(folio): return;
11.    spin_lock(&swap_avail_lock).
12.    if plist_node_empty(&next.avail_list): continue 'start_over;
13.  break;
14.}
15. spin_unlock(&swap_avail_lock).

`Swap::try_to_unuse(ty: u32) -> Result<()>`:
1. si = swap_info[ty as usize].
2. while SwapInfo::usage_in_pages(si) > 0:
3.   for each mm in init_mm.mmlist (mmget, drop lock, walk, mmput):
4.     Swap::unuse_mm(mm, ty)?.
5.   /* shmem path */
6.   offset = Swap::find_next_to_unuse(si, &mut offset)?.
7.   shmem_unuse(folio)?.
8.   if signal_pending(current): return Err(EINTR).
9. Ok(()).

`Swap::unuse_pte(vma, pmd, addr, entry, folio) -> Result<()>`:
1. (pte, ptl) = pte_offset_map_lock(mm, pmd, addr).
2. if !pte_same_as_swp(*pte, mk_swap_pte(entry)): return Ok(()).
3. page = folio_file_page(folio, swp_offset(entry)).
4. inc_mm_counter(mm, MM_ANONPAGES); dec_mm_counter(mm, MM_SWAPENTS).
5. new_pte = mk_pte(page, vma.vm_page_prot).
6. if pte_swp_soft_dirty(*pte): new_pte = pte_mksoft_dirty(new_pte).
7. if pte_swp_uffd_wp(*pte): new_pte = pte_mkuffd_wp(new_pte).
8. set_pte_at(mm, addr, pte, new_pte).
9. swap_free(entry).
10. drop((pte, ptl)).
11. Ok(()).

### Out of Scope

- mm/swap.c folio LRU / pagevec (covered in `swap.md` Tier-3)
- mm/swap_state.c swap-cache (covered separately if expanded)
- mm/swap_slots.c (folded into swapfile.c in 7.1.0-rc2; covered here)
- mm/zswap.c compressed cache (covered in `zswap.md` Tier-3)
- mm/page_io.c swap_writepage / swap_read_folio (covered separately)
- kernel/power/swap.c hibernation pipeline (covered in suspend/hibernate)
- block/blk-throttle.c blk-cgroup throttle (covered in block layer)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct swap_info_struct` | per-area state | `SwapInfo` |
| `struct swap_cluster_info` | per-cluster state | `SwapCluster` |
| `struct swap_extent` | per-page→block map (rb-tree node) | `SwapExtent` |
| `struct percpu_swap_cluster` | per-CPU cluster cache | `PercpuSwapCluster` |
| `swp_entry_t` | per-entry encoded type/offset | `SwapEntry` |
| `swp_type(e)` / `swp_offset(e)` / `swp_entry(t,o)` | encode helpers | `SwapEntry::ty` / `::offset` / `::new` |
| `SYSCALL_DEFINE2(swapon, ...)` | per-add | `Sys::swapon` |
| `SYSCALL_DEFINE1(swapoff, ...)` | per-remove | `Sys::swapoff` |
| `alloc_swap_info()` | per-area-alloc | `SwapInfo::alloc` |
| `claim_swapfile()` | per-S_ISBLK / S_ISREG claim | `SwapInfo::claim` |
| `read_swap_header()` | per-SWAPSPACE2 hdr | `SwapInfo::read_header` |
| `setup_swap_extents()` | per-build rb-tree | `SwapInfo::setup_extents` |
| `add_swap_extent()` | per-rb-insert | `SwapInfo::add_extent` |
| `offset_to_swap_extent()` | per-rb-lookup | `SwapInfo::find_extent` |
| `destroy_swap_extents()` | per-tree-free | `SwapInfo::destroy_extents` |
| `setup_swap_clusters_info()` | per-cluster-array alloc + init | `SwapInfo::setup_clusters` |
| `swap_cluster_alloc_table()` | per-cluster table-alloc (RCU) | `SwapCluster::alloc_table` |
| `enable_swap_info()` | per-activate (SWP_WRITEOK) | `SwapInfo::enable` |
| `_enable_swap_info()` | per-plist-insert | `SwapInfo::enable_inner` |
| `reinsert_swap_info()` | per-undo-swapoff | `SwapInfo::reinsert` |
| `del_from_avail_list()` / `add_to_avail_list()` | per-avail-head mutation | `SwapInfo::avail_del` / `::avail_add` |
| `swap_alloc_fast()` | per-CPU-cluster fast-path | `SwapAlloc::fast` |
| `swap_alloc_slow()` | per-rotate-device slow-path | `SwapAlloc::slow` |
| `cluster_alloc_swap_entry()` | per-device cluster pick | `SwapAlloc::cluster_alloc_entry` |
| `alloc_swap_scan_cluster()` | per-cluster scan-and-mark | `SwapAlloc::scan_cluster` |
| `alloc_swap_scan_list()` | per-list scan | `SwapAlloc::scan_list` |
| `__swap_cluster_alloc_entries()` | per-set-table + count | `SwapAlloc::cluster_alloc_entries` |
| `swap_range_alloc()` / `swap_range_free()` | per-slot-count accounting | `SwapAlloc::range_alloc` / `::range_free` |
| `swap_usage_add()` / `swap_usage_sub()` | per-inuse-pages atomic | `SwapInfo::usage_add` / `::usage_sub` |
| `folio_alloc_swap()` | per-folio entry point | `Swap::folio_alloc` |
| `folio_dup_swap()` / `folio_put_swap()` | per-folio count up/down | `Swap::folio_dup` / `::folio_put` |
| `swap_put_entries_direct()` | per-bulk-free | `Swap::put_entries_direct` |
| `swap_put_entries_cluster()` / `__swap_cluster_put_entry()` | per-decrement | `SwapCluster::put_entry` |
| `swap_dup_entries_cluster()` / `__swap_cluster_dup_entry()` | per-increment | `SwapCluster::dup_entry` |
| `__swap_count()` / `swp_swapcount()` | per-count query | `Swap::count` / `::full_count` |
| `swap_entry_swapped()` | per-presence test | `Swap::entry_present` |
| `folio_free_swap()` | per-release-if-unmapped | `Swap::folio_free` |
| `folio_swapcache_freeable()` | per-eligible-to-drop | `Swap::folio_freeable` |
| `__try_to_reclaim_swap()` | per-cache-reclaim | `Swap::try_reclaim_one` |
| `swap_reclaim_full_clusters()` | per-full-cluster cache-reclaim | `Swap::reclaim_full_clusters` |
| `discard_swap()` / `discard_swap_cluster()` | per-blkdev-discard | `Swap::discard_all` / `::discard_cluster` |
| `swap_do_scheduled_discard()` | per-pending-discard drain | `Swap::do_discard` |
| `swap_discard_work()` | per-workqueue | `Swap::discard_worker` |
| `swap_reclaim_work()` | per-cache-reclaim workqueue | `Swap::reclaim_worker` |
| `swap_sync_discard()` | per-sync-rotate-discard | `Swap::sync_discard` |
| `free_cluster()` / `__free_cluster()` | per-cluster-free | `SwapCluster::free` / `::free_inner` |
| `partial_free_cluster()` / `relocate_cluster()` | per-cluster-promote/demote | `SwapCluster::partial_free` / `::relocate` |
| `move_cluster()` | per-list-transfer | `SwapCluster::move` |
| `isolate_lock_cluster()` | per-pick-+-lock | `SwapCluster::isolate_lock` |
| `try_to_unuse()` | per-evict-all-back-to-mm | `Swap::try_to_unuse` |
| `unuse_mm()` / `unuse_vma()` / `unuse_p4d/pud/pmd/pte_range()` | per-mm-walk swap-in | `Swap::unuse_*` |
| `unuse_pte()` | per-PTE restore | `Swap::unuse_pte` |
| `find_next_to_unuse()` | per-iterate-in-use | `Swap::find_next_to_unuse` |
| `drain_mmlist()` | per-final-list-clear | `Swap::drain_mmlist` |
| `get_swap_device()` / `put_swap_device()` | per-percpu-ref pin | `SwapInfo::get_device` / `::put_device` |
| `get_swap_device_info()` | per-tryget-ref | `SwapInfo::get_device_info` |
| `swap_users_ref_free()` | per-percpu-ref-cb completion | `SwapInfo::users_ref_free` |
| `flush_percpu_swap_cluster()` | per-pcp-invalidate on swapoff | `Swap::flush_percpu_cluster` |
| `wait_for_allocation()` | per-cluster-quiescent | `SwapInfo::wait_for_allocation` |
| `swap_folio_sector()` | per-entry→sector | `Swap::folio_sector` |
| `swapdev_block()` | per-offset→sector | `Swap::dev_block` |
| `swap_alloc_hibernation_slot()` / `swap_free_hibernation_slot()` | per-hibernation | `Swap::alloc_hib_slot` / `::free_hib_slot` |
| `swap_type_of()` / `find_first_swap()` | per-hib lookup | `Swap::type_of` / `::find_first` |
| `count_swap_pages()` | per-hib accounting | `Swap::count_pages` |
| `si_swapinfo()` | per-/proc/meminfo | `Swap::si_swapinfo` |
| `__folio_throttle_swaprate()` | per-blk-cgroup throttle | `Swap::throttle_swaprate` |
| `swap_show()` / `swaps_proc_ops` | per-/proc/swaps | `Swap::proc_*` |
| `swaps_poll()` | per-poll proc_poll_event | `Swap::proc_poll` |
| `generic_max_swapfile_size()` / `arch_max_swapfile_size()` | per-pte-encoded limit | `Swap::generic_max_size` / `::arch_max_size` |
| `swap_dup_entry_direct()` / `swap_retry_table_alloc()` | per-table-extend retry | `Swap::dup_entry_direct` / `::retry_table_alloc` |
| `nr_swap_pages` / `total_swap_pages` / `nr_rotate_swap` | per-globals | shared |
| `swap_lock` / `swap_avail_lock` / `swapon_mutex` | per-globals lock | shared |
| `swap_active_head` / `swap_avail_head` | per-plist | shared |
| `proc_poll_wait` / `proc_poll_event` | per-/proc/swaps poll | shared |

### compatibility contract

REQ-1: struct swap_info_struct (selected per-area fields, in implementation order):
- type: per-area index into swap_info[].
- flags: SWP_USED | SWP_WRITEOK | SWP_SOLIDSTATE | SWP_DISCARDABLE | SWP_AREA_DISCARD | SWP_PAGE_DISCARD | SWP_STABLE_WRITES | SWP_SYNCHRONOUS_IO | SWP_BLKDEV | SWP_FS_OPS | SWP_ACTIVATED.
- prio: per-area swap priority (DEF_SWAP_PRIO = -1; or PREFER mask).
- list / avail_list: per-plist nodes (negated prio: high-to-low semantics).
- swap_file: per-`struct file *` backing.
- bdev: per-`struct block_device *` (for S_ISBLK; or sb->s_bdev for S_ISREG).
- max: per-total pages including header.
- pages: per-usable pages (max - 1 - badpages, minus EOF-cluster tail).
- inuse_pages: per-`atomic_long_t` with SWAP_USAGE_OFFLIST_BIT.
- lock: per-spinlock; held when mutating cluster lists or counts.
- cluster_info: per-array-of `swap_cluster_info`, length DIV_ROUND_UP(max, SWAPFILE_CLUSTER).
- global_cluster: per-HDD only — `struct swap_cluster*next[SWAP_NR_ORDERS]` head-pointer.
- global_cluster_lock: per-HDD only — serializes HDD allocation.
- free_clusters / full_clusters / discard_clusters: per-list (CLUSTER_FLAG_FREE | _FULL | _DISCARD).
- nonfull_clusters[SWAP_NR_ORDERS] / frag_clusters[SWAP_NR_ORDERS]: per-order lists.
- swap_extent_root: per-rb-tree of `swap_extent` page→block map.
- discard_work / reclaim_work: per-`struct work_struct`.
- users: per-`struct percpu_ref` for read-side liveness.
- comp: per-`struct completion` posted when users ref drops to zero.
- zeromap: per-bitmap of all-zero pages (kvmalloc_array, BITS_TO_LONGS(maxpages)).

REQ-2: struct swap_cluster_info:
- lock: per-cluster spinlock.
- count: per-allocated-slot count (0..SWAPFILE_CLUSTER).
- flags: CLUSTER_FLAG_NONE | _FREE | _NONFULL | _FRAG | _FULL | _DISCARD | _USABLE | _MAX.
- order: per-cluster-order (0..PMD_ORDER-1; only meaningful when count > 0).
- list: per-list-node (free/nonfull/frag/full/discard).
- table: per-`struct swap_table *` rcu-protected (NULL when freed).
- extend_table: per-overflow-counter array (kzalloc-on-demand when any slot count hits SWP_TB_COUNT_MAX).

REQ-3: struct swap_extent (rb-tree node):
- rb_node: per-`struct rb_node` linked into swap_info.swap_extent_root.
- start_page: per-page-offset into swap area.
- nr_pages: per-extent length in pages.
- start_block: per-disk-sector for start_page (PAGE_SIZE units).

REQ-4: struct percpu_swap_cluster:
- si[SWAP_NR_ORDERS]: per-order cached `*swap_info_struct`.
- offset[SWAP_NR_ORDERS]: per-order cached offset in si.
- lock: per-`local_lock_t`.
- /* per-CPU instance via `DEFINE_PER_CPU(percpu_swap_cluster, percpu_swap_cluster)` */
- /* invariant: si[i] non-NULL ⟺ offset[i] != SWAP_ENTRY_INVALID */

REQ-5: swp_entry_t encoding:
- per-`swp_entry_t.val`: per-unsigned-long packed (SWP_TYPE_SHIFT | SWP_OFFSET_MASK).
- swp_type(e) = (e.val >> SWP_TYPE_SHIFT) & SWP_TYPE_MASK.
- swp_offset(e) = e.val & SWP_OFFSET_MASK.
- swp_entry(type, offset) = (type << SWP_TYPE_SHIFT) | offset.
- /* PTE encoding: `__swp_entry_to_pte()` / `__pte_to_swp_entry()` per-arch */
- /* maximum offset: per-`arch_max_swapfile_size()` round-trip */

REQ-6: SYSCALL_DEFINE2(swapon, specialfile, swap_flags):
- /* Capability */
- if !capable(CAP_SYS_ADMIN): return -EPERM.
- if swap_flags & ~SWAP_FLAGS_VALID: return -EINVAL.
- /* Allocate swap_info_struct (or reuse first dead slot) */
- si = alloc_swap_info().
- if IS_ERR(si): return PTR_ERR(si).
- INIT_WORK(si.discard_work, swap_discard_work).
- INIT_WORK(si.reclaim_work, swap_reclaim_work).
- /* Open file O_RDWR | O_LARGEFILE | O_EXCL */
- swap_file = file_open_name(specialfile, ...).
- /* claim S_ISBLK or S_ISREG */
- claim_swapfile(si, inode):
  - if S_ISBLK: si.bdev = I_BDEV(inode); if bdev_is_zoned: return -EINVAL; si.flags |= SWP_BLKDEV.
  - if S_ISREG: si.bdev = inode.i_sb.s_bdev.
- /* Inode S_SWAPFILE flag mutex */
- inode_lock(inode).
- if IS_SWAPFILE(inode): return -EBUSY.
- /* Read SWAPSPACE2 header */
- folio = read_mapping_folio(mapping, 0, swap_file).
- maxpages = read_swap_header(si, swap_header, inode).
- si.max = maxpages; si.pages = maxpages - 1.
- /* Build swap_extent rb-tree */
- nr_extents = setup_swap_extents(si, swap_file, &span).
- /* Build cluster array */
- setup_swap_clusters_info(si, swap_header, maxpages).
- swap_cgroup_swapon(si.type, maxpages).
- /* zeromap bitmap */
- si.zeromap = kvmalloc_array(BITS_TO_LONGS(maxpages), ..., __GFP_ZERO).
- /* SSD detect */
- if si.bdev ∧ !bdev_rot(si.bdev): si.flags |= SWP_SOLIDSTATE.
- else: atomic_inc(&nr_rotate_swap).
- /* Stable-writes / sync-IO */
- if bdev_stable_writes: si.flags |= SWP_STABLE_WRITES.
- if bdev_synchronous: si.flags |= SWP_SYNCHRONOUS_IO.
- /* Discard policy */
- if (swap_flags & SWAP_FLAG_DISCARD) ∧ bdev_max_discard_sectors:
  - si.flags |= SWP_DISCARDABLE | SWP_AREA_DISCARD | SWP_PAGE_DISCARD.
  - if SWAP_FLAG_DISCARD_ONCE: clear SWP_PAGE_DISCARD.
  - if SWAP_FLAG_DISCARD_PAGES: clear SWP_AREA_DISCARD.
  - if SWP_AREA_DISCARD: discard_swap(si) one-shot.
- zswap_swapon(si.type, maxpages).
- /* Activate */
- inode.i_flags |= S_SWAPFILE.
- inode_drain_writes(inode).
- /* Priority */
- prio = (swap_flags & SWAP_FLAG_PREFER) ? (swap_flags & SWAP_FLAG_PRIO_MASK) : DEF_SWAP_PRIO.
- si.prio = prio; si.list.prio = -prio; si.avail_list.prio = -prio.
- /* Publish (plist-insert + percpu_ref_resurrect) */
- enable_swap_info(si).
- atomic_inc(&proc_poll_event); wake_up_interruptible(&proc_poll_wait).
- return 0.

REQ-7: SYSCALL_DEFINE1(swapoff, specialfile):
- if !capable(CAP_SYS_ADMIN): return -EPERM.
- /* Find si by mapping */
- victim = file_open_name(specialfile, O_RDWR | O_LARGEFILE, 0).
- plist_for_each_entry(p, &swap_active_head, list):
  - if (p.flags & SWP_WRITEOK) ∧ p.swap_file.f_mapping == victim.f_mapping: found.
- if !found: return -EINVAL.
- /* Account total */
- if !security_vm_enough_memory_mm(current.mm, p.pages): err = -ENOMEM; goto out.
- /* De-publish */
- del_from_avail_list(p, true).  /* clears SWP_WRITEOK */
- plist_del(&p.list, &swap_active_head).
- atomic_long_sub(p.pages, &nr_swap_pages); total_swap_pages -= p.pages.
- /* Wait for in-flight allocations */
- wait_for_allocation(p).
- /* Evict all in-use entries back to memory */
- set_current_oom_origin(); err = try_to_unuse(p.type); clear_current_oom_origin().
- if err: reinsert_swap_info(p); goto out.
- /* Wait for outstanding read-side users */
- percpu_ref_kill(&p.users); synchronize_rcu(); wait_for_completion(&p.comp).
- flush_work(&p.discard_work); flush_work(&p.reclaim_work).
- flush_percpu_swap_cluster(p).
- destroy_swap_extents(p, p.swap_file).
- if !(p.flags & SWP_SOLIDSTATE): atomic_dec(&nr_rotate_swap).
- /* Tear down */
- drain_mmlist().
- swap_file = p.swap_file; p.swap_file = NULL.
- zeromap = p.zeromap; p.zeromap = NULL.
- maxpages = p.max; cluster_info = p.cluster_info.
- p.max = 0; p.cluster_info = NULL.
- arch_swap_invalidate_area(p.type); zswap_swapoff(p.type).
- kfree(p.global_cluster).
- kvfree(zeromap).
- free_swap_cluster_info(cluster_info, maxpages).
- swap_cgroup_swapoff(p.type).
- inode.i_flags &= ~S_SWAPFILE.
- filp_close(swap_file, NULL).
- /* Re-arm slot for reuse */
- p.flags = 0.
- atomic_inc(&proc_poll_event); wake_up_interruptible(&proc_poll_wait).
- return 0.

REQ-8: alloc_swap_info():
- p = kvzalloc_obj(struct swap_info_struct).
- percpu_ref_init(&p.users, swap_users_ref_free, PERCPU_REF_INIT_DEAD, GFP_KERNEL).
- /* Find first SWP_USED-free slot in swap_info[MAX_SWAPFILES] */
- spin_lock(&swap_lock).
- for type in 0..nr_swapfiles: if !(swap_info[type].flags & SWP_USED): break.
- if type >= MAX_SWAPFILES: spin_unlock; return ERR_PTR(-EPERM).
- if type >= nr_swapfiles: p.type = type; smp_store_release(&swap_info[type], p); nr_swapfiles++.
- else: defer = p; p = swap_info[type].
- p.swap_extent_root = RB_ROOT.
- plist_node_init(&p.list, 0); plist_node_init(&p.avail_list, 0).
- p.flags = SWP_USED.
- spin_unlock(&swap_lock).
- atomic_long_set(&p.inuse_pages, SWAP_USAGE_OFFLIST_BIT).
- init_completion(&p.comp).
- return p.

REQ-9: read_swap_header(si, hdr, inode):
- if memcmp("SWAPSPACE2", hdr.magic.magic, 10) != 0: pr_err; return 0.
- if swab32(hdr.info.version) == 1: swab the version + last_page + nr_badpages + each badpages[i].
- if hdr.info.version != 1: return 0.
- maxpages = swapfile_maximum_size.
- last_page = hdr.info.last_page.
- if !last_page: return 0.
- if maxpages > last_page: maxpages = last_page + 1.
- if maxpages exceeds (unsigned int): clamp UINT_MAX.
- if i_size(inode) >> PAGE_SHIFT < maxpages: return 0 (shorter than signature).
- if hdr.info.nr_badpages ∧ S_ISREG(inode.i_mode): return 0.
- if hdr.info.nr_badpages > MAX_SWAP_BADPAGES: return 0.
- return maxpages.

REQ-10: setup_swap_extents(si, swap_file, *span):
- inode = swap_file.f_mapping.host.
- if S_ISBLK(inode.i_mode):
  - add_swap_extent(si, 0, si.max, 0).  /* single extent for blockdev */
  - *span = si.pages.
  - return 1.
- if mapping.a_ops.swap_activate:
  - ret = swap_activate(si, swap_file, span).
  - si.flags |= SWP_ACTIVATED.
  - if SWP_FS_OPS ∧ sio_pool_init() != 0: destroy + return -ENOMEM.
  - return ret.
- return generic_swapfile_activate(si, swap_file, span).  /* bmap walk → add_swap_extent per fragment */

REQ-11: add_swap_extent(si, start_page, nr_pages, start_block):
- /* Insert at right-most (called in ascending page order) */
- walk si.swap_extent_root from root following rb_right until NULL.
- if parent ∧ parent.start_page + parent.nr_pages == start_page ∧ parent.start_block + parent.nr_pages == start_block:
  - /* Contiguous: merge into parent */
  - parent.nr_pages += nr_pages.
  - return 0.
- /* Else allocate new extent */
- new_se = kmalloc_obj(struct swap_extent).
- new_se = { start_page, nr_pages, start_block }.
- rb_link_node + rb_insert_color(&new_se.rb_node, &si.swap_extent_root).
- return 1.

REQ-12: offset_to_swap_extent(si, offset):
- rb = si.swap_extent_root.rb_node.
- while rb:
  - se = rb_entry(rb, swap_extent, rb_node).
  - if offset < se.start_page: rb = rb.rb_left.
  - elif offset >= se.start_page + se.nr_pages: rb = rb.rb_right.
  - else: return se.
- BUG().  /* must always be present */

REQ-13: setup_swap_clusters_info(si, hdr, maxpages):
- nr_clusters = DIV_ROUND_UP(maxpages, SWAPFILE_CLUSTER).
- cluster_info = kvzalloc_objs(swap_cluster_info, nr_clusters).
- for i in 0..nr_clusters: spin_lock_init(&cluster_info[i].lock).
- /* HDD: single global pointer */
- if !(si.flags & SWP_SOLIDSTATE):
  - si.global_cluster = kmalloc_obj(struct swap_cluster).
  - for i in 0..SWAP_NR_ORDERS: si.global_cluster.next[i] = SWAP_ENTRY_INVALID.
  - spin_lock_init(&si.global_cluster_lock).
- /* Mark page-0 (header) + hdr.badpages + EOF-cluster-tail as bad */
- swap_cluster_setup_bad_slot(si, ci, 0, swapoff=false).
- for i in 0..hdr.info.nr_badpages: swap_cluster_setup_bad_slot(si, ci, hdr.badpages[i], false).
- for i in maxpages..round_up(maxpages, SWAPFILE_CLUSTER): swap_cluster_setup_bad_slot(si, ci, i, swapoff=true).
- INIT_LIST_HEAD(free_clusters, full_clusters, discard_clusters).
- for i in 0..SWAP_NR_ORDERS: INIT_LIST_HEAD(nonfull_clusters[i], frag_clusters[i]).
- /* Bucket clusters: free vs nonfull (only EOF-tail cluster + bad-slot clusters have count > 0 at init) */
- for ci in cluster_info[0..nr_clusters]:
  - if ci.count: ci.flags = CLUSTER_FLAG_NONFULL; list_add_tail(&ci.list, &nonfull_clusters[0]).
  - else: ci.flags = CLUSTER_FLAG_FREE; list_add_tail(&ci.list, &free_clusters).
- si.cluster_info = cluster_info.
- return 0.

REQ-14: enable_swap_info(si):
- percpu_ref_resurrect(&si.users).
- spin_lock(&swap_lock); spin_lock(&si.lock).
- _enable_swap_info(si):
  - atomic_long_add(si.pages, &nr_swap_pages).
  - total_swap_pages += si.pages.
  - plist_add(&si.list, &swap_active_head).
  - add_to_avail_list(si, swapon=true).
- spin_unlock(&si.lock); spin_unlock(&swap_lock).

REQ-15: folio_alloc_swap(folio):
- order = folio_order(folio).
- VM_BUG_ON_FOLIO(!folio_test_locked(folio), folio).
- VM_BUG_ON_FOLIO(!folio_test_uptodate(folio), folio).
- if order:
  - if !CONFIG_THP_SWAP: return -EAGAIN.
  - if (1 << order) > SWAPFILE_CLUSTER: WARN; return -EINVAL.
- again:
- local_lock(&percpu_swap_cluster.lock).
- if !swap_alloc_fast(folio): swap_alloc_slow(folio).
- local_unlock(&percpu_swap_cluster.lock).
- /* Sync discard retry on order-0 miss */
- if !order ∧ !folio_test_swapcache(folio):
  - if swap_sync_discard(): goto again.
- if mem_cgroup_try_charge_swap(folio, folio.swap): swap_cache_del_folio(folio).
- if !folio_test_swapcache(folio): return -ENOMEM.
- return 0.

REQ-16: swap_alloc_fast(folio):
- order = folio_order(folio).
- si = this_cpu_read(percpu_swap_cluster.si[order]).
- offset = this_cpu_read(percpu_swap_cluster.offset[order]).
- if !si ∨ !offset ∨ !get_swap_device_info(si): return false.
- ci = swap_cluster_lock(si, offset).
- if cluster_is_usable(ci, order):
  - if cluster_is_empty(ci): offset = cluster_offset(si, ci).
  - alloc_swap_scan_cluster(si, ci, folio, offset).
- else: swap_cluster_unlock(ci).
- put_swap_device(si).
- return folio_test_swapcache(folio).

REQ-17: swap_alloc_slow(folio):
- spin_lock(&swap_avail_lock).
- start_over:
- plist_for_each_entry_safe(si, next, &swap_avail_head, avail_list):
  - plist_requeue(&si.avail_list, &swap_avail_head).  /* round-robin */
  - spin_unlock(&swap_avail_lock).
  - if get_swap_device_info(si):
    - cluster_alloc_swap_entry(si, folio).
    - put_swap_device(si).
    - if folio_test_swapcache(folio): return.
    - if folio_test_large(folio): return.
  - spin_lock(&swap_avail_lock).
  - if plist_node_empty(&next.avail_list): goto start_over.
- spin_unlock(&swap_avail_lock).

REQ-18: cluster_alloc_swap_entry(si, folio):
- order = folio ? folio_order(folio) : 0.
- /* THP/large only on block-device-backed swap */
- if order ∧ !(si.flags & SWP_BLKDEV): return 0.
- /* HDD: hold global head pointer; SSD: lockless */
- if !(si.flags & SWP_SOLIDSTATE):
  - spin_lock(&si.global_cluster_lock).
  - offset = si.global_cluster.next[order].
  - if offset != SWAP_ENTRY_INVALID:
    - ci = swap_cluster_lock(si, offset).
    - if cluster_is_usable(ci, order):
      - if cluster_is_empty(ci): offset = cluster_offset(si, ci).
      - found = alloc_swap_scan_cluster(si, ci, folio, offset).
      - if found: goto done.
- new_cluster:
- /* Discard-aware: free → fresh; otherwise nonfull first */
- if si.flags & SWP_PAGE_DISCARD:
  - found = alloc_swap_scan_list(si, &si.free_clusters, folio, false); if found: done.
- if order < PMD_ORDER:
  - found = alloc_swap_scan_list(si, &si.nonfull_clusters[order], folio, true); if found: done.
- if !(si.flags & SWP_PAGE_DISCARD):
  - found = alloc_swap_scan_list(si, &si.free_clusters, folio, false); if found: done.
- /* Pressure: reclaim full clusters' swap-cache */
- if vm_swap_full(): swap_reclaim_full_clusters(si, force=false).
- if order < PMD_ORDER:
  - found = alloc_swap_scan_list(si, &si.frag_clusters[order], folio, false); if found: done.
- if order: goto done.
- /* Order-0 fallback: steal from higher-order frag/nonfull */
- for o in 1..SWAP_NR_ORDERS:
  - found = alloc_swap_scan_list(si, &si.frag_clusters[o], folio, true); if found: done.
  - found = alloc_swap_scan_list(si, &si.nonfull_clusters[o], folio, true); if found: done.
- done:
- if !(si.flags & SWP_SOLIDSTATE): spin_unlock(&si.global_cluster_lock).
- return found.

REQ-19: swap_cluster_alloc_table(si, ci):
- /* Lazy per-cluster table allocation guarded by ci.lock + percpu_swap_cluster.lock */
- lockdep_assert_held(percpu_swap_cluster.lock).
- if !(si.flags & SWP_SOLIDSTATE): lockdep_assert_held(global_cluster_lock).
- lockdep_assert_held(&ci.lock).
- /* Try atomic */
- table = swap_table_alloc(__GFP_HIGH | __GFP_NOMEMALLOC | __GFP_NOWARN).
- if table: rcu_assign_pointer(ci.table, table); return ci.
- /* Drop locks, sleeping allocation */
- spin_unlock(&ci.lock).
- if !(si.flags & SWP_SOLIDSTATE): spin_unlock(global_cluster_lock).
- local_unlock(percpu_swap_cluster.lock).
- table = swap_table_alloc(__GFP_HIGH | __GFP_NOMEMALLOC | GFP_KERNEL).
- /* Re-acquire */
- local_lock(percpu_swap_cluster.lock).
- if !(si.flags & SWP_SOLIDSTATE): spin_lock(global_cluster_lock).
- spin_lock(&ci.lock).
- if cluster_table_is_alloced(ci): /* another raced — drop our table */ free; return ci.
- if !table: move_cluster(si, ci, &si.free_clusters, CLUSTER_FLAG_FREE); spin_unlock(&ci.lock); return NULL.
- rcu_assign_pointer(ci.table, table).
- return ci.

REQ-20: discard / reclaim:
- discard_swap(si): for each swap_extent, blkdev_issue_discard(si.bdev, start_block, nr_blocks, GFP_KERNEL).
- discard_swap_cluster(si, start_page, nr_pages): walk swap_extent rb-tree from start_page; blkdev_issue_discard.
- swap_cluster_schedule_discard(si, ci): move ci → discard_clusters (CLUSTER_FLAG_DISCARD); schedule_work(&si.discard_work).
- swap_do_scheduled_discard(si): drain si.discard_clusters; for each ci → list_del, discard_swap_cluster, ci.flags = NONE, __free_cluster(si, ci).
- swap_sync_discard(): on order-0 alloc miss with no fast-path, rotate swap_active_head; if SWP_PAGE_DISCARD, do swap_do_scheduled_discard inline.
- swap_reclaim_full_clusters(si, force): iterate full_clusters; for each, __try_to_reclaim_swap to drop swap-cache.
- swap_reclaim_work: kthread wrapper for swap_reclaim_full_clusters(force=true).

REQ-21: swap_range_alloc / swap_range_free:
- swap_range_alloc(si, nr_entries): swap_usage_add(si, nr_entries); if !atomic_long_add_return_relaxed(nr_entries, &si.inuse_pages) result is si.pages: del_from_avail_list(si, swapoff=false).
- swap_range_free(si, offset, nr_entries): swap_usage_sub(si, nr_entries); if newly off-full: add_to_avail_list(si, swapon=false).

REQ-22: folio_dup_swap / folio_put_swap / swap_put_entries_direct:
- folio_dup_swap(folio, subpage): swap_dup_entries_cluster(si, swp_offset(folio.swap), nr_pages or 1-subpage).
- folio_put_swap(folio, subpage): swap_put_entries_cluster(si, swp_offset(folio.swap), nr_pages or 1-subpage, reclaim_cache=false).
- swap_put_entries_direct(entry, nr): walk entries cluster-by-cluster; swap_put_entries_cluster with reclaim_cache=true; called when entry is dropped from swap cache without folio backing.

REQ-23: get_swap_device / put_swap_device / get_swap_device_info:
- get_swap_device(entry): si = swap_entry_to_info(entry); if !si: return NULL; if !percpu_ref_tryget_live(&si.users): return NULL; smp_rmb (paired w/ enable_swap_info); return si.
- put_swap_device(si): percpu_ref_put(&si.users).
- get_swap_device_info(si): per-si variant of the above tryget, returns bool.
- /* Read-side guarantee: holding ref ⟹ si.flags / si.cluster_info / si.swap_file stable */

REQ-24: try_to_unuse(type) (per-swapoff eviction):
- si = swap_info[type].
- /* Outer loop over in-use slots */
- while swap_usage_in_pages(si) > 0:
  - /* Iterate mm list (every mm that has touched swap) */
  - spin_lock(&mmlist_lock).
  - for each mm in init_mm.mmlist:
    - mmget(mm).
    - spin_unlock(&mmlist_lock).
    - unuse_mm(mm, type).
    - spin_lock(&mmlist_lock).
    - mmput(mm).
  - spin_unlock(&mmlist_lock).
  - /* Iterate shmem inodes (per-swap_offset find) */
  - find_next_to_unuse(si, &offset).
  - shmem_unuse(folio).  /* if a tmpfs / shmem inode holds entry */
- return 0 on full drain; -EBUSY / -EINTR on signal.

REQ-25: unuse_pte(vma, pmd, addr, entry, folio):
- pte = pte_offset_map_lock(mm, pmd, addr, &ptl).
- /* Verify slot still references our swap entry */
- if !pte_same_as_swp(*pte, mk_swap_pte(entry)): unlock; return 0.
- /* Install folio's page; restore PTE */
- page = folio_file_page(folio, swp_offset(entry)).
- inc_mm_counter (MM_ANONPAGES or MM_FILEPAGES); dec_mm_counter(MM_SWAPENTS).
- new_pte = mk_pte(page, vma.vm_page_prot).
- if pte_swp_soft_dirty(*pte): new_pte = pte_mksoft_dirty(new_pte).
- if pte_swp_uffd_wp(*pte): new_pte = pte_mkuffd_wp(new_pte).
- set_pte_at(mm, addr, pte, new_pte).
- swap_free(entry).
- pte_unmap_unlock(pte, ptl).

REQ-26: zswap integration:
- zswap_swapon(type, maxpages) on swapon: allocate per-type tree, charge counters.
- zswap_swapoff(type) on swapoff: drain tree, free entries.
- /* On swap_writepage, zswap_store may absorb the folio (no disk I/O) */
- /* On swap_read_folio, zswap_load may serve the folio (no disk I/O) */
- /* No frontswap in 7.1.0-rc2 (removed); zswap is the sole compressed-cache path */

REQ-27: si_swapinfo(val):
- for type in 0..nr_swapfiles:
  - si = swap_info[type].
  - if !(si.flags & SWP_USED): continue.
  - val.freeswap += si.pages - swap_usage_in_pages(si).
  - val.totalswap += si.pages.

REQ-28: __folio_throttle_swaprate(folio, gfp):
- /* Per-blk-cgroup throttle on swap writes */
- if !__has_usable_swap: return.
- if !(gfp & __GFP_IO): return.
- if !blkcg_punt_bio_submit(...): /* throttle */ blkcg_schedule_throttle(...).

REQ-29: /proc/swaps:
- swap_show: emit `Filename Type Size Used Priority`.
- swaps_proc_ops: per-poll on proc_poll_event; readable seq_file.
- swaps_poll: poll_wait + EPOLLIN | EPOLLRDNORM | EPOLLERR | EPOLLPRI on swapon/swapoff events.

REQ-30: swap_alloc_hibernation_slot(type) (CONFIG_HIBERNATION):
- si = swap_info[type]; if !si: return {0}.
- get_swap_device_info(si).
- /* Try per-CPU cluster cached for this si first; else cluster_alloc_swap_entry */
- offset = ... .
- if offset: entry = swp_entry(type, offset).
- return entry.

REQ-31: per-cluster table extension:
- Each cluster has a packed swap_table mapping cluster-offset → (count + flags).
- When per-slot count reaches SWP_TB_COUNT_MAX (saturating), extend_table is allocated and overflow tracked in extend_table[ci_off].
- swap_retry_table_alloc(entry, gfp): swap_extend_table_alloc(si, ci, gfp); used when an atomic alloc failed and caller now has sleeping gfp.
- swap_extend_table_try_free(ci): if all extend_table slots are 0, free the table.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `swp_entry_roundtrip` | INVARIANT | per-encode: SwapEntry::new(t,o).ty() == t ∧ .offset() == o (for t < (1<<SWP_TYPE_BITS), o < (1<<SWP_OFFSET_BITS)). |
| `swap_info_slot_uniqueness` | INVARIANT | per-alloc_swap_info: each SWP_USED slot in swap_info[] is held by at most one live SwapInfo. |
| `percpu_ref_pair_balanced` | INVARIANT | per-get_swap_device/put_swap_device: refcount balanced; no orphaned ref on early return. |
| `cluster_flags_match_list` | INVARIANT | per-cluster: ci.flags determines which list (free/nonfull/frag/full/discard) ci is on. |
| `extent_rbtree_disjoint` | INVARIANT | per-rb-tree: extents are disjoint and ordered by start_page. |
| `inuse_pages_monotone_in_alloc` | INVARIANT | per-swap_range_alloc: inuse_pages strictly increases by nr_entries. |
| `inuse_pages_monotone_in_free` | INVARIANT | per-swap_range_free: inuse_pages strictly decreases by nr_entries. |
| `avail_list_membership` | INVARIANT | per-add_to_avail_list ⟺ SWAP_USAGE_OFFLIST_BIT cleared ∧ SWP_WRITEOK set. |
| `solidstate_no_global_cluster` | INVARIANT | per-SwapInfo: (flags & SWP_SOLIDSTATE) ⟹ global_cluster == None. |
| `hdd_global_cluster_lock_held` | INVARIANT | per-cluster_alloc_swap_entry: !SWP_SOLIDSTATE ⟹ global_cluster_lock held during scan. |
| `swap_extent_lookup_present` | INVARIANT | per-offset_to_swap_extent: offset < si.max ⟹ se exists and covers offset. |
| `cluster_table_alloc_under_lock` | INVARIANT | per-swap_cluster_alloc_table: ci.lock + percpu cluster local lock + global cluster lock (HDD) all held. |

### Layer 2: TLA+

`mm/swapfile.tla`:
- Per-swapon + per-swapoff lifecycle + per-allocate + per-free + per-unuse + per-percpu-ref drain.
- Properties:
  - `safety_swapon_swapoff_balanced` — per-area: swapon establishes ↔ swapoff drains.
  - `safety_users_ref_balanced` — per-percpu-ref: get_swap_device get'd, put'd on every path.
  - `safety_no_alloc_after_writeok_clear` — per-swapoff: after SWP_WRITEOK cleared, allocators see ci unusable.
  - `safety_no_double_free_of_entry` — per-cluster-table: count > 0 ⟹ no concurrent re-allocation.
  - `safety_extent_tree_disjoint` — per-rb-tree: extents are disjoint.
  - `safety_priority_ordering` — per-plist: swap_avail_head is high-to-low by prio.
  - `liveness_try_to_unuse_terminates` — per-swapoff: try_to_unuse drains in-use or returns -EINTR.
  - `liveness_swap_alloc_fast_or_slow` — per-folio_alloc_swap: returns or rotates devices.
  - `liveness_discard_drains` — per-swap_do_scheduled_discard: discard_clusters drains eventually.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Sys::swapon` post: si ∈ swap_info[]; SWP_USED ∧ SWP_WRITEOK set; plist-active | `Sys::swapon` |
| `Sys::swapoff` post: !SWP_USED; cluster_info/zeromap/swap_file/global_cluster freed | `Sys::swapoff` |
| `Swap::folio_alloc` post: Ok ⟹ folio.swap valid ∧ folio in swap-cache | `Swap::folio_alloc` |
| `Swap::folio_alloc` post: Err(-EAGAIN) ⟹ order > 0 ∧ !CONFIG_THP_SWAP | `Swap::folio_alloc` |
| `Swap::folio_alloc` post: Err(-ENOMEM) ⟹ all SWP_WRITEOK areas exhausted or mem_cgroup denied | `Swap::folio_alloc` |
| `Swap::folio_sector` post: returns sector matching extent rb-tree | `Swap::folio_sector` |
| `SwapAlloc::fast` post: true ⟹ folio_test_swapcache; ci was usable; cluster lock dropped | `SwapAlloc::fast` |
| `SwapAlloc::slow` post: visits each plist entry at most once per pass | `SwapAlloc::slow` |
| `SwapInfo::add_extent` post: rb-tree retains BST + disjoint-extent invariants | `SwapInfo::add_extent` |
| `Swap::try_to_unuse` post: Ok ⟹ inuse_pages == 0 | `Swap::try_to_unuse` |
| `Swap::unuse_pte` post: pte_same swap-entry ⟹ replaced with anon mapping; swap_free called | `Swap::unuse_pte` |
| `SwapInfo::get_device` post: Some(si) ⟹ percpu_ref+1; smp_rmb paired with enable_swap_info | `SwapInfo::get_device` |

### Layer 4: Verus/Creusot functional

`Per-swapon (alloc_swap_info → claim_swapfile → read_swap_header → setup_swap_extents → setup_swap_clusters_info → zswap_swapon → enable_swap_info) → per-folio_alloc_swap (swap_alloc_fast / swap_alloc_slow → cluster_alloc_swap_entry → alloc_swap_scan_cluster) → per-folio_dup_swap / folio_put_swap → per-swap_put_entries_direct → per-swapoff (try_to_unuse → percpu_ref_kill → drain_mmlist → destroy_swap_extents → free_swap_cluster_info → zswap_swapoff)` semantic equivalence: per-Documentation/admin-guide/mm/swap_numa.rst and per-Documentation/admin-guide/sysctl/vm.rst (swappiness, page-cluster).

### hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening.)

Swap-subsystem reinforcement:

- **Per-CAP_SYS_ADMIN check on swapon/swapoff** — defense against per-unprivileged swap-area mutation.
- **Per-SWAPSPACE2 magic check** — defense against per-non-swap file being misread.
- **Per-MAX_SWAP_BADPAGES bound** — defense against per-header-overflow exploits.
- **Per-bdev_is_zoned reject** — defense against per-zoned-write disorder.
- **Per-S_SWAPFILE inode flag** — defense against per-active-swap-file-edit corruption.
- **Per-percpu_ref get/put strict** — defense against per-swapoff UAF on swap_info_struct.
- **Per-synchronize_rcu + wait_for_completion** — defense against per-RCU-reader missing tear-down.
- **Per-cluster.lock + ci.lock + percpu local_lock layering** — defense against per-allocator deadlock.
- **Per-SWAP_USAGE_OFFLIST_BIT cmpxchg** — defense against per-double-list-insert/remove.
- **Per-swap_extent rb-tree disjoint + ascending insert** — defense against per-extent-overlap → wrong-sector.
- **Per-SWP_TB_COUNT_MAX overflow into extend_table** — defense against per-swap-count silent wrap.
- **Per-zswap_swapon/_swapoff paired** — defense against per-zswap-orphan after swapoff.
- **Per-swap_cgroup_swapon/_swapoff paired** — defense against per-memcg-charge orphan.

