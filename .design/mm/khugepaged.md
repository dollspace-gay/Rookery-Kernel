# Tier-3: mm/khugepaged.c — Transparent Huge Page background collapse daemon

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: mm/00-overview.md
upstream-paths:
  - mm/khugepaged.c (~2909 lines)
  - mm/mm_slot.h
  - include/linux/khugepaged.h
  - include/trace/events/huge_memory.h
  - Documentation/admin-guide/mm/transhuge.rst
-->

## Summary

**khugepaged** is the kernel thread that scans mm address spaces for runs of base 4 KiB pages and **collapses** them into single 2 MiB PMD-mapped huge pages, so that THP benefits are reaped not only at first-fault time but for already-faulted memory. Per-`__khugepaged_enter(mm)` enrolls an mm by allocating a `struct mm_slot`, inserting it into the `mm_slots_hash` hash + `khugepaged_scan.mm_head` list, and setting `MMF_VM_HUGEPAGE`. Per-`khugepaged()` kthread loops `khugepaged_do_scan` → `collapse_scan_mm_slot` → `collapse_single_pmd`. Per-anon path: `collapse_scan_pmd` walks `HPAGE_PMD_NR` (=512) ptes under pte-lock, counts `none_or_zero` / `swap` / `shared` / `referenced`, and if all knob bounds satisfied, calls `collapse_huge_page` to allocate a folio, swapin missing pages, `pmdp_collapse_flush` the PMD, isolate + copy + map a PMD-mapped anon folio. Per-file path: `collapse_scan_file` walks the page-cache xarray, then `collapse_file` allocates a new PMD-order folio, replaces individual cache entries, copies data, retracts PTE tables via `retract_page_tables`. Per-`MADV_COLLAPSE` (`madvise_collapse`) drives the same collapse paths synchronously, bypassing the THP sysfs gates via `TVA_FORCED_COLLAPSE`. Per-tunables: `khugepaged_pages_to_scan` (default 8 * 512), `khugepaged_scan_sleep_millisecs` (10000), `khugepaged_alloc_sleep_millisecs` (60000), `khugepaged_max_ptes_none` (511), `khugepaged_max_ptes_swap` (64), `khugepaged_max_ptes_shared` (256). Critical for: post-fault THP density, defragmentation amortization, shmem/file THP hot-coalesce, MADV_COLLAPSE userspace contract.

This Tier-3 covers `mm/khugepaged.c` (~2909 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `enum scan_result` | per-collapse outcome (SCAN_SUCCEED, SCAN_FAIL, SCAN_EXCEED_*) | `ScanResult` |
| `struct collapse_control` | per-scan context (is_khugepaged, node_load, progress, alloc_nmask) | `CollapseControl` |
| `struct khugepaged_scan` | per-cursor (mm_head, mm_slot, address) | `KhugepagedScan` |
| `struct mm_slot` | per-mm tracking node | `MmSlot` |
| `khugepaged_init()` | per-boot init | `Khugepaged::init` |
| `khugepaged_destroy()` | per-shutdown | `Khugepaged::destroy` |
| `__khugepaged_enter()` | per-mm enroll | `Khugepaged::enter_mm` |
| `khugepaged_enter_vma()` | per-VMA-fault enroll | `Khugepaged::enter_vma` |
| `__khugepaged_exit()` | per-mm unenroll | `Khugepaged::exit_mm` |
| `hugepage_pmd_enabled()` | per-policy gate | `Khugepaged::pmd_enabled` |
| `collapse_test_exit()` / `_or_disable()` | per-mm liveness | `Khugepaged::test_exit` |
| `collapse_scan_pmd()` | per-PMD anon scan | `Khugepaged::scan_pmd` |
| `collapse_scan_file()` | per-PMD file scan | `Khugepaged::scan_file` |
| `collapse_single_pmd()` | per-PMD dispatch | `Khugepaged::collapse_single_pmd` |
| `collapse_huge_page()` | per-anon collapse | `Khugepaged::collapse_huge_page` |
| `collapse_file()` | per-file collapse | `Khugepaged::collapse_file` |
| `__collapse_huge_page_isolate()` | per-pte isolate | `Khugepaged::isolate_ptes` |
| `__collapse_huge_page_copy()` | per-pte copy | `Khugepaged::copy_ptes` |
| `__collapse_huge_page_swapin()` | per-pte swap-in | `Khugepaged::swapin_ptes` |
| `alloc_charge_folio()` | per-allocation + memcg charge | `Khugepaged::alloc_folio` |
| `try_collapse_pte_mapped_thp()` | per-PTE-mapped THP collapse | `Khugepaged::collapse_pte_mapped_thp` |
| `collapse_pte_mapped_thp()` | external entry | `Khugepaged::collapse_pte_mapped_thp_extern` |
| `retract_page_tables()` | per-file page-table retract | `Khugepaged::retract_page_tables` |
| `set_huge_pmd()` | per-PMD install | `Khugepaged::set_huge_pmd` |
| `find_pmd_or_thp_or_none()` | per-PMD probe | `Khugepaged::find_pmd_or_thp_or_none` |
| `check_pmd_state()` | per-PMD state classifier | `Khugepaged::check_pmd_state` |
| `check_pmd_still_valid()` | per-PMD validity after lock-drop | `Khugepaged::check_pmd_still_valid` |
| `hugepage_vma_revalidate()` | per-VMA recheck after lock-drop | `Khugepaged::hugepage_vma_revalidate` |
| `collapse_scan_abort()` | per-NUMA scan-abort | `Khugepaged::scan_abort` |
| `collapse_find_target_node()` | per-NUMA target-node pick | `Khugepaged::find_target_node` |
| `collect_mm_slot()` | per-iter mm_slot reclaim | `Khugepaged::collect_mm_slot` |
| `collapse_scan_mm_slot()` | per-iter mm-slot scan | `Khugepaged::scan_mm_slot` |
| `khugepaged_do_scan()` | per-pass scan | `Khugepaged::do_scan` |
| `khugepaged_wait_work()` | per-pass sleep | `Khugepaged::wait_work` |
| `khugepaged()` | kthread entry | `Khugepaged::thread` |
| `start_stop_khugepaged()` | per-policy-change kthread start/stop | `Khugepaged::start_stop` |
| `set_recommended_min_free_kbytes()` | per-policy min_free_kbytes raise | `Khugepaged::recommend_min_free_kbytes` |
| `khugepaged_min_free_kbytes_update()` | per-tunable recheck | `Khugepaged::min_free_kbytes_update` |
| `current_is_khugepaged()` | per-self test | `Khugepaged::current_is` |
| `hugepage_madvise()` | per-MADV_HUGEPAGE / NOHUGEPAGE handler | `Khugepaged::madvise` |
| `madvise_collapse()` | per-MADV_COLLAPSE handler | `Khugepaged::madvise_collapse` |
| `madvise_collapse_errno()` | per-scan_result → errno mapper | `Khugepaged::madvise_errno` |
| `khugepaged_pages_to_scan` | per-tunable | shared |
| `khugepaged_scan_sleep_millisecs` | per-tunable | shared |
| `khugepaged_alloc_sleep_millisecs` | per-tunable | shared |
| `khugepaged_max_ptes_none` | per-tunable | shared |
| `khugepaged_max_ptes_swap` | per-tunable | shared |
| `khugepaged_max_ptes_shared` | per-tunable | shared |
| `khugepaged_collapse_control` | per-singleton CC for kthread | `Khugepaged::CC_KTHREAD` |
| `khugepaged_attr_group` | per-sysfs group | `Khugepaged::SYSFS` |

## Compatibility contract

REQ-1: enum scan_result enumerates per-collapse outcome:
- SCAN_FAIL (default failure), SCAN_SUCCEED (happy path).
- SCAN_NO_PTE_TABLE / SCAN_PMD_MAPPED — PMD-level shape mismatches.
- SCAN_EXCEED_NONE_PTE / SCAN_EXCEED_SWAP_PTE / SCAN_EXCEED_SHARED_PTE — per-knob overruns.
- SCAN_PTE_NON_PRESENT / SCAN_PTE_UFFD_WP / SCAN_PTE_MAPPED_HUGEPAGE — PTE-level rejects.
- SCAN_LACK_REFERENCED_PAGE / SCAN_PAGE_NULL / SCAN_PAGE_COUNT / SCAN_PAGE_LRU / SCAN_PAGE_LOCK / SCAN_PAGE_ANON / SCAN_PAGE_LAZYFREE / SCAN_PAGE_COMPOUND / SCAN_PAGE_FILLED / SCAN_PAGE_HAS_PRIVATE / SCAN_PAGE_DIRTY_OR_WRITEBACK — per-page-state rejects.
- SCAN_SCAN_ABORT (cross-NUMA), SCAN_ANY_PROCESS (mm dying), SCAN_VMA_NULL / SCAN_VMA_CHECK / SCAN_ADDRESS_RANGE — VMA rejects.
- SCAN_DEL_PAGE_LRU, SCAN_ALLOC_HUGE_PAGE_FAIL, SCAN_CGROUP_CHARGE_FAIL — per-alloc fail.
- SCAN_TRUNCATED, SCAN_STORE_FAILED, SCAN_COPY_MC — per-file-collapse fail.

REQ-2: struct collapse_control:
- is_khugepaged: bool — true for kthread, false for MADV_COLLAPSE.
- node_load[MAX_NUMNODES]: u32 — per-node scan tally for NUMA target-node pick.
- progress: u32 — counts of ptes/folios processed against khugepaged_pages_to_scan.
- alloc_nmask: nodemask_t — fallback alloc nodemask.

REQ-3: struct khugepaged_scan (one global instance):
- mm_head: list_head — head of mm_slots enrolled with khugepaged.
- mm_slot: *mm_slot — current scan cursor (NULL between full scans).
- address: unsigned long — next VA inside current mm to scan.

REQ-4: khugepaged_init():
- mm_slot_cache = KMEM_CACHE(mm_slot, 0).
- if !mm_slot_cache: return -ENOMEM.
- khugepaged_pages_to_scan = HPAGE_PMD_NR * 8 (= 4096 ptes per pass).
- khugepaged_max_ptes_none = KHUGEPAGED_MAX_PTES_LIMIT (HPAGE_PMD_NR - 1 = 511).
- khugepaged_max_ptes_swap = HPAGE_PMD_NR / 8 (= 64).
- khugepaged_max_ptes_shared = HPAGE_PMD_NR / 2 (= 256).
- return 0.

REQ-5: __khugepaged_enter(mm):
- VM_BUG_ON_MM(collapse_test_exit(mm)).
- if mm_flags_test_and_set(MMF_VM_HUGEPAGE, mm): return (already enrolled).
- slot = mm_slot_alloc(mm_slot_cache).
- if !slot: return.
- spin_lock(&khugepaged_mm_lock).
- mm_slot_insert(mm_slots_hash, mm, slot).
- wakeup = list_empty(&khugepaged_scan.mm_head).
- list_add_tail(&slot->mm_node, &khugepaged_scan.mm_head).
- spin_unlock.
- mmgrab(mm); if wakeup: wake_up_interruptible(&khugepaged_wait).

REQ-6: khugepaged_enter_vma(vma, vm_flags):
- if !MMF_VM_HUGEPAGE ∧ hugepage_pmd_enabled() ∧ thp_vma_allowable_order(vma, vm_flags, TVA_KHUGEPAGED, PMD_ORDER):
  - __khugepaged_enter(vma.vm_mm).

REQ-7: __khugepaged_exit(mm):
- spin_lock(&khugepaged_mm_lock).
- slot = mm_slot_lookup(mm_slots_hash, mm).
- if slot ∧ khugepaged_scan.mm_slot != slot:
  - hash_del(&slot->hash); list_del(&slot->mm_node); free = 1.
- spin_unlock.
- if free: mm_flags_clear(MMF_VM_HUGEPAGE); mm_slot_free; mmdrop.
- else if slot: mmap_write_lock(mm); mmap_write_unlock(mm) /* barrier vs scan */.

REQ-8: collapse_scan_mm_slot(progress_max, *result, cc) under &khugepaged_mm_lock entry; releases + reacquires:
- pick slot = khugepaged_scan.mm_slot, else first of mm_head.
- spin_unlock(&khugepaged_mm_lock).
- mm = slot.mm.
- if !mmap_read_trylock(mm): goto breakouterloop_mmap_lock.
- cc.progress++; if exit_or_disable(mm): goto breakouterloop.
- vma_iter_init(&vmi, mm, khugepaged_scan.address).
- for_each_vma(vmi, vma):
  - if !thp_vma_allowable_order(vma, vma.vm_flags, TVA_KHUGEPAGED, PMD_ORDER): cc.progress++; continue.
  - hstart = round_up(vma.vm_start, HPAGE_PMD_SIZE); hend = round_down(vma.vm_end, HPAGE_PMD_SIZE).
  - if khugepaged_scan.address > hend: skip; else if < hstart: bump up.
  - while khugepaged_scan.address < hend:
    - *result = collapse_single_pmd(khugepaged_scan.address, vma, &lock_dropped, cc).
    - khugepaged_scan.address += HPAGE_PMD_SIZE.
    - if lock_dropped: goto breakouterloop_mmap_lock.
    - if cc.progress >= progress_max: goto breakouterloop.
- breakouterloop: mmap_read_unlock(mm).
- spin_lock(&khugepaged_mm_lock).
- if exit_or_disable(mm) ∨ !vma:
  - advance khugepaged_scan.mm_slot to next or NULL (khugepaged_full_scans++).
  - collect_mm_slot(slot).
- trace_mm_khugepaged_scan(mm, ...).

REQ-9: collapse_single_pmd(addr, vma, *lock_dropped, cc):
- mmap_assert_locked(mm).
- if vma_is_anonymous(vma): return collapse_scan_pmd(mm, vma, addr, lock_dropped, cc).
- file = get_file(vma.vm_file); pgoff = linear_page_index(vma, addr).
- mmap_read_unlock(mm); *lock_dropped = true.
- result = collapse_scan_file(mm, addr, file, pgoff, cc).
- if !cc.is_khugepaged ∧ result == SCAN_PAGE_DIRTY_OR_WRITEBACK ∧ !triggered_wb ∧ mapping_can_writeback: writeback + retry once.
- fput(file).
- if result == SCAN_PTE_MAPPED_HUGEPAGE: mmap_read_lock; try_collapse_pte_mapped_thp(mm, addr, !cc.is_khugepaged); mmap_read_unlock.
- if cc.is_khugepaged ∧ result == SCAN_SUCCEED: ++khugepaged_pages_collapsed.

REQ-10: collapse_scan_pmd(mm, vma, start_addr, *lock_dropped, cc):
- find_pmd_or_thp_or_none(mm, start_addr, &pmd).
- pte = pte_offset_map_lock(mm, pmd, start_addr, &ptl).
- for pte iter over HPAGE_PMD_NR entries:
  - count none_or_zero, swap, shared, referenced.
  - per-knob: if cc.is_khugepaged ∧ none_or_zero > khugepaged_max_ptes_none → SCAN_EXCEED_NONE_PTE.
  - per-knob: swap > khugepaged_max_ptes_swap → SCAN_EXCEED_SWAP_PTE.
  - per-knob: shared > khugepaged_max_ptes_shared → SCAN_EXCEED_SHARED_PTE.
  - check pte present, swap-entry (count swap-in cost), uffd-wp marker, page sanity.
- pte_unmap_unlock.
- if eligible: collapse_huge_page(mm, start_addr, referenced, unmapped, cc).

REQ-11: collapse_huge_page(mm, address, referenced, unmapped, cc):
- VM_BUG_ON(address & ~HPAGE_PMD_MASK).
- mmap_read_unlock(mm) /* alloc may block */.
- result = alloc_charge_folio(&folio, mm, cc).
- mmap_read_lock(mm).
- hugepage_vma_revalidate; find_pmd_or_thp_or_none.
- if unmapped: __collapse_huge_page_swapin (may release mmap_lock on failure → goto out_nolock).
- mmap_read_unlock; mmap_write_lock.
- hugepage_vma_revalidate; vma_start_write; check_pmd_still_valid.
- anon_vma_lock_write(vma.anon_vma).
- mmu_notifier_invalidate_range_start([address, address+HPAGE_PMD_SIZE)).
- pmd_ptl = pmd_lock; _pmd = pmdp_collapse_flush(vma, address, pmd); spin_unlock(pmd_ptl).
- mmu_notifier_invalidate_range_end; tlb_remove_table_sync_one.
- pte = pte_offset_map_lock(mm, &_pmd, address, &pte_ptl).
- result = __collapse_huge_page_isolate(vma, address, pte, cc, &compound_pagelist); spin_unlock(pte_ptl).
- if !SUCCEED: pte_unmap; restore via pmd_populate(mm, pmd, pmd_pgtable(_pmd)); anon_vma_unlock_write; out_up_write.
- anon_vma_unlock_write.
- __collapse_huge_page_copy(pte, folio, pmd, _pmd, vma, address, pte_ptl, &compound_pagelist); pte_unmap.
- __folio_mark_uptodate(folio); pgtable = pmd_pgtable(_pmd).
- spin_lock(pmd_ptl); pgtable_trans_huge_deposit; map_anon_folio_pmd_nopf(folio, pmd, vma, address); spin_unlock(pmd_ptl).
- folio = NULL; result = SCAN_SUCCEED.
- out_up_write: mmap_write_unlock(mm).
- if folio: folio_put(folio).
- trace_mm_collapse_huge_page.

REQ-12: collapse_scan_file(mm, addr, file, start, cc):
- xas_for_each over [start, start+HPAGE_PMD_NR-1]:
  - swap entry: swap += 1<<order; if cc.is_khugepaged ∧ swap > khugepaged_max_ptes_swap → SCAN_EXCEED_SWAP_PTE.
  - folio_try_get; xas_reload sanity.
  - if is_pmd_order(folio_order): SCAN_PTE_MAPPED_HUGEPAGE.
  - NUMA: cc.node_load[folio_nid(folio)]++; if collapse_scan_abort → SCAN_SCAN_ABORT.
  - per-folio_test_lru; folio_expected_ref_count+1 == folio_ref_count.
  - present += folio_nr_pages(folio).
- if cc.is_khugepaged ∧ present < HPAGE_PMD_NR - khugepaged_max_ptes_none: SCAN_EXCEED_NONE_PTE.
- else: collapse_file(mm, addr, file, start, cc).

REQ-13: collapse_file(mm, addr, file, start, cc):
- allocate + lock new PMD-order folio (alloc_charge_folio).
- xas walk + replace native cache entries; swap/gup-in needed pages.
- copy data to new folio; handle shmem holes (revalidate not filled, no uffd marker).
- finalize xarray updates: replace `HPAGE_PMD_NR` slots with single high-order folio.
- on success: unlock new folio; free old folios; retract_page_tables(mapping, pgoff) for all vmas mapping the range.
- on failure: unlock + free new folio; restore old folios.

REQ-14: retract_page_tables(mapping, pgoff):
- i_mmap_lock_read.
- vma_interval_tree_foreach(vma in mapping->i_mmap @ pgoff):
  - skip if addr unaligned; skip if !file_backed_vma_is_retractable (anon_vma ∨ userfaultfd_wp ∨ VMA_MAYBE_GUARD_BIT).
  - mmu_notifier_invalidate_range_start.
  - pml = pmd_lock; recheck check_pmd_state ⇒ SUCCEED.
  - ptl = pte_lockptr; nested-lock if distinct.
  - pgt_pmd = pmdp_collapse_flush(vma, addr, pmd); pmdp_get_lockless_sync; success = true.
  - mmu_notifier_invalidate_range_end.
  - if success: mm_dec_nr_ptes; page_table_check_pte_clear_range; pte_free_defer.
- i_mmap_unlock_read.

REQ-15: try_collapse_pte_mapped_thp(mm, addr, install_pmd):
- haddr = addr & HPAGE_PMD_MASK; vma = vma_lookup(mm, haddr).
- find_pmd_or_thp_or_none; if SCAN_PMD_MAPPED return early.
- folio = filemap_lock_folio(vma->vm_file->f_mapping, linear_page_index(vma, haddr)).
- if !is_pmd_order(folio_order): SCAN_PAGE_COMPOUND.
- iterate over HPAGE_PMD_NR ptes: validate each maps the expected folio offset; nr_mapped_ptes++.
- pml = pmd_lock; pgt_pmd = pmdp_collapse_flush; install replacement PMD if install_pmd via set_huge_pmd.
- mm_dec_nr_ptes; pte_free_defer.

REQ-16: alloc_charge_folio(&folio, mm, cc):
- gfp = alloc_hugepage_khugepaged_gfpmask() for kthread, GFP_TRANSHUGE for MADV_COLLAPSE.
- nid = collapse_find_target_node(cc).
- vma_alloc_folio / __folio_alloc_node + memcg charge.
- on alloc fail: SCAN_ALLOC_HUGE_PAGE_FAIL.
- on memcg fail: SCAN_CGROUP_CHARGE_FAIL.

REQ-17: khugepaged_do_scan(cc):
- progress_max = READ_ONCE(khugepaged_pages_to_scan).
- lru_add_drain_all().
- cc.progress = 0.
- loop until cc.progress >= progress_max ∨ kthread_should_stop:
  - spin_lock(&khugepaged_mm_lock).
  - if !mm_slot: pass_through_head++.
  - if has_work ∧ pass_through_head < 2: collapse_scan_mm_slot(progress_max, &result, cc).
  - else: cc.progress = progress_max.
  - spin_unlock.
  - if result == SCAN_ALLOC_HUGE_PAGE_FAIL: khugepaged_alloc_sleep on first failure, then bail.

REQ-18: khugepaged_wait_work():
- if has_work: scan_sleep_jiffies = msecs_to_jiffies(khugepaged_scan_sleep_millisecs).
  - khugepaged_sleep_expire = jiffies + scan_sleep_jiffies.
  - wait_event_freezable_timeout(khugepaged_wait, should_wakeup, scan_sleep_jiffies).
- else: if pmd_enabled(): wait_event_freezable(khugepaged_wait, wait_event()).

REQ-19: khugepaged(none) kthread:
- set_freezable(); set_user_nice(current, MAX_NICE).
- while !kthread_should_stop: khugepaged_do_scan(&khugepaged_collapse_control); khugepaged_wait_work().
- on stop: spin_lock; collect mm_slot; spin_unlock; return 0.

REQ-20: start_stop_khugepaged():
- mutex_lock(&khugepaged_mutex).
- if hugepage_pmd_enabled() ∧ khugepaged_has_work:
  - if !khugepaged_thread: kthread_run(khugepaged, NULL, "khugepaged"); set_recommended_min_free_kbytes.
- else if khugepaged_thread: kthread_stop; khugepaged_thread = NULL.
- mutex_unlock.

REQ-21: madvise_collapse(vma, start, end, *lock_dropped):
- BUG_ON(vma.vm_start > start ∨ vma.vm_end < end).
- if !thp_vma_allowable_order(vma, vma.vm_flags, TVA_FORCED_COLLAPSE, PMD_ORDER): return -EINVAL.
- cc = kmalloc(); cc.is_khugepaged = false; cc.progress = 0.
- mmgrab(mm); lru_add_drain_all.
- hstart = (start + ~HPAGE_PMD_MASK) & HPAGE_PMD_MASK; hend = end & HPAGE_PMD_MASK.
- for addr in [hstart, hend) step HPAGE_PMD_SIZE:
  - if mmap_unlocked: relock; revalidate; hend = min(hend, vma.vm_end & HPAGE_PMD_MASK).
  - result = collapse_single_pmd(addr, vma, &mmap_unlocked, cc).
  - per-result switch: SUCCEED/PMD_MAPPED→thps++; transient→last_fail set, continue; fatal→break.
- mmap_assert_locked; mmdrop; kfree(cc).
- return 0 iff thps == hpages, else madvise_collapse_errno(last_fail).

REQ-22: madvise_collapse_errno(r):
- SCAN_ALLOC_HUGE_PAGE_FAIL → -ENOMEM.
- SCAN_CGROUP_CHARGE_FAIL ∨ SCAN_EXCEED_* → -EBUSY.
- default → -EINVAL.

REQ-23: hugepage_madvise(vma, *vm_flags, advice):
- MADV_HUGEPAGE: clear VM_NOHUGEPAGE; set VM_HUGEPAGE; __khugepaged_enter(vma.vm_mm) if !MMF_VM_HUGEPAGE.
- MADV_NOHUGEPAGE: clear VM_HUGEPAGE; set VM_NOHUGEPAGE.

REQ-24: hugepage_pmd_enabled():
- true if any of: READ_ONLY_THP_FOR_FS ∧ hugepage_global_enabled(); PMD_ORDER in huge_anon_orders_always; PMD_ORDER in huge_anon_orders_madvise; PMD_ORDER in huge_anon_orders_inherit ∧ hugepage_global_enabled; CONFIG_SHMEM ∧ shmem_hpage_pmd_enabled.

REQ-25: per-NUMA target node:
- collapse_scan_abort(nid, cc): true if any node has > load * fraction crossing threshold ⇒ SCAN_SCAN_ABORT.
- collapse_find_target_node(cc): pick node with max cc.node_load (modulo NUMA_NO_NODE for non-NUMA).

REQ-26: set_recommended_min_free_kbytes():
- Raise `min_free_kbytes` so compaction can produce PMD-order pages.

REQ-27: khugepaged_min_free_kbytes_update():
- Called when user changes vm.min_free_kbytes; preserve recommendation if khugepaged active.

REQ-28: current_is_khugepaged():
- Returns current == khugepaged_thread (used to gate self-exclusion).

REQ-29: Sysfs tunables (`/sys/kernel/mm/transparent_hugepage/khugepaged/`):
- pages_to_scan, pages_collapsed (RO), full_scans (RO).
- scan_sleep_millisecs, alloc_sleep_millisecs.
- defrag (bool — controls gfp).
- max_ptes_none ∈ [0, KHUGEPAGED_MAX_PTES_LIMIT].
- max_ptes_swap ∈ [0, KHUGEPAGED_MAX_PTES_LIMIT].
- max_ptes_shared ∈ [0, KHUGEPAGED_MAX_PTES_LIMIT].

REQ-30: Locking discipline:
- khugepaged_mm_lock (spinlock): protects mm_slots_hash + khugepaged_scan.mm_head + mm_slot membership.
- khugepaged_mutex: serializes start_stop_khugepaged.
- Per-scan: mmap_read_lock then pte_offset_map_lock; collapse upgrades to mmap_write_lock.
- pmd_lock + pte ptl pair: nested order for retract_page_tables.

## Acceptance Criteria

- [ ] AC-1: __khugepaged_enter inserts mm_slot once; second call is a no-op (MMF_VM_HUGEPAGE set).
- [ ] AC-2: collapse_scan_mm_slot enters with khugepaged_mm_lock held, releases for mmap_read_trylock, reacquires before return.
- [ ] AC-3: collapse_scan_pmd respects max_ptes_none, max_ptes_swap, max_ptes_shared.
- [ ] AC-4: collapse_huge_page acquires mmap_write_lock + anon_vma_lock_write + pmd_lock in order; restores PMD via pmd_populate on isolate failure.
- [ ] AC-5: collapse_file replaces HPAGE_PMD_NR cache entries with one PMD-order folio; failures restore originals.
- [ ] AC-6: retract_page_tables skips VMAs with anon_vma, userfaultfd_wp, or VMA_MAYBE_GUARD_BIT.
- [ ] AC-7: try_collapse_pte_mapped_thp returns SCAN_PMD_MAPPED early if already PMD-mapped.
- [ ] AC-8: alloc_charge_folio returns SCAN_ALLOC_HUGE_PAGE_FAIL on alloc fail, SCAN_CGROUP_CHARGE_FAIL on memcg charge fail.
- [ ] AC-9: madvise_collapse uses TVA_FORCED_COLLAPSE, bypasses sysfs gates that block khugepaged.
- [ ] AC-10: madvise_collapse returns 0 only when every PMD in [hstart, hend) collapsed (or already PMD-mapped); else madvise_collapse_errno(last_fail).
- [ ] AC-11: khugepaged kthread sleeps khugepaged_scan_sleep_millisecs between passes when work pending; freezable.
- [ ] AC-12: khugepaged_do_scan caps work at khugepaged_pages_to_scan per pass; on SCAN_ALLOC_HUGE_PAGE_FAIL: one alloc_sleep retry then exit pass.
- [ ] AC-13: __khugepaged_exit removes slot unless scan cursor on it; in the cursor case, mmap_write_lock cycle synchronizes against scan release.
- [ ] AC-14: start_stop_khugepaged is idempotent w.r.t. THP policy changes; serialized by khugepaged_mutex.
- [ ] AC-15: hugepage_madvise(MADV_HUGEPAGE) sets VM_HUGEPAGE + enrolls mm if hugepage_pmd_enabled().

## Architecture

```
struct CollapseControl {
  is_khugepaged: bool,
  node_load: [u32; MAX_NUMNODES],
  progress: u32,
  alloc_nmask: NodeMask,
}

struct KhugepagedScan {
  mm_head: ListHead,
  mm_slot: Option<*MmSlot>,
  address: u64,
}

struct MmSlot {
  mm: *MmStruct,
  hash: HlistNode,
  mm_node: ListHead,
}

enum ScanResult {
  Fail, Succeed,
  NoPteTable, PmdMapped,
  ExceedNonePte, ExceedSwapPte, ExceedSharedPte,
  PteNonPresent, PteUffdWp, PteMappedHugepage,
  LackReferencedPage, PageNull, ScanAbort,
  PageCount, PageLru, PageLock, PageAnon, PageLazyfree,
  PageCompound, AnyProcess, VmaNull, VmaCheck, AddressRange,
  DelPageLru, AllocHugePageFail, CgroupChargeFail,
  Truncated, PageHasPrivate, StoreFailed, CopyMc,
  PageFilled, PageDirtyOrWriteback,
}
```

`Khugepaged::scan_mm_slot(progress_max, cc) -> ScanResult` (under khugepaged_mm_lock entry):
1. Pick slot (cursor or first); save khugepaged_scan.mm_slot = slot; spin_unlock.
2. mm = slot.mm. If !mmap_read_trylock(mm): breakouterloop_mmap_lock.
3. cc.progress++; if test_exit_or_disable(mm): break.
4. vma_iter_init(&vmi, mm, khugepaged_scan.address).
5. For each vma:
   - If !thp_vma_allowable_order(vma, vma.vm_flags, TVA_KHUGEPAGED, PMD_ORDER): cc.progress++; continue.
   - hstart = round_up(vma.vm_start, HPAGE_PMD_SIZE); hend = round_down(vma.vm_end, HPAGE_PMD_SIZE).
   - clamp khugepaged_scan.address into [hstart, hend).
   - While khugepaged_scan.address < hend:
     - result = collapse_single_pmd(khugepaged_scan.address, vma, &lock_dropped, cc).
     - khugepaged_scan.address += HPAGE_PMD_SIZE.
     - If lock_dropped: break outer (no read_unlock needed).
     - If cc.progress >= progress_max: break outer.
6. mmap_read_unlock(mm) only if not lock_dropped.
7. spin_lock(&khugepaged_mm_lock); advance cursor (next slot or NULL); on finish: khugepaged_full_scans++.
8. collect_mm_slot(slot) if mm dying or fully scanned.

`Khugepaged::scan_pmd(mm, vma, start_addr, cc) -> ScanResult`:
1. find_pmd_or_thp_or_none(mm, start_addr, &pmd).
2. pte = pte_offset_map_lock(mm, pmd, start_addr, &ptl).
3. For each of HPAGE_PMD_NR ptes:
   - cc.progress++.
   - Count none_or_zero (pte_none ∨ pte_zero), swap (pte_swap_entry), shared (folio_mapcount > 1), referenced (pte_young or folio_test_referenced).
   - Per-knob overrun: SCAN_EXCEED_*.
   - pte uffd-wp ⇒ SCAN_PTE_UFFD_WP.
   - swap entry ⇒ unmapped++ (later swapin).
   - !pte_present ∧ !swap ⇒ SCAN_PTE_NON_PRESENT.
4. pte_unmap_unlock.
5. If eligible: collapse_huge_page(mm, start_addr, referenced, unmapped, cc).

`Khugepaged::collapse_huge_page(mm, address, referenced, unmapped, cc) -> ScanResult`:
1. mmap_read_unlock(mm).
2. alloc_charge_folio(&folio, mm, cc) — may sleep.
3. mmap_read_lock(mm); hugepage_vma_revalidate; find_pmd_or_thp_or_none.
4. If unmapped: __collapse_huge_page_swapin (may drop mmap_lock; goto out_nolock on fail).
5. mmap_read_unlock(mm); mmap_write_lock(mm).
6. hugepage_vma_revalidate; vma_start_write; check_pmd_still_valid.
7. anon_vma_lock_write(vma.anon_vma).
8. mmu_notifier_invalidate_range_start([address, address+HPAGE_PMD_SIZE)).
9. pmd_ptl = pmd_lock; _pmd = pmdp_collapse_flush(vma, address, pmd); spin_unlock.
10. mmu_notifier_invalidate_range_end; tlb_remove_table_sync_one.
11. pte = pte_offset_map_lock(mm, &_pmd, address, &pte_ptl).
12. __collapse_huge_page_isolate(vma, address, pte, cc, &compound_pagelist); spin_unlock.
13. On isolate fail: pte_unmap; spin_lock(pmd_ptl); pmd_populate(mm, pmd, pmd_pgtable(_pmd)); spin_unlock; anon_vma_unlock_write; out_up_write.
14. anon_vma_unlock_write.
15. __collapse_huge_page_copy(pte, folio, pmd, _pmd, vma, address, pte_ptl, &compound_pagelist); pte_unmap.
16. __folio_mark_uptodate(folio).
17. pmd_lock; pgtable_trans_huge_deposit; map_anon_folio_pmd_nopf(folio, pmd, vma, address); spin_unlock.
18. folio = NULL; result = SCAN_SUCCEED.
19. out_up_write: mmap_write_unlock(mm). out_nolock: if folio: folio_put(folio); trace.

`Khugepaged::scan_file(mm, addr, file, start, cc) -> ScanResult`:
1. mapping = file.f_mapping; xas = XA_STATE(mapping.i_pages, start).
2. rcu_read_lock.
3. xas_for_each over [start, start+HPAGE_PMD_NR-1]:
   - xa_is_value(folio) ⇒ swap += 1<<order; check max_ptes_swap.
   - folio_try_get; xas_reload sanity.
   - is_pmd_order ⇒ SCAN_PTE_MAPPED_HUGEPAGE (caller will retry try_collapse_pte_mapped_thp).
   - per-NUMA: cc.node_load[folio_nid]++; collapse_scan_abort ⇒ SCAN_SCAN_ABORT.
   - !folio_test_lru ⇒ SCAN_PAGE_LRU.
   - folio_expected_ref_count + 1 != folio_ref_count ⇒ SCAN_PAGE_COUNT.
   - present += folio_nr_pages.
4. rcu_read_unlock.
5. If cc.is_khugepaged ∧ present < HPAGE_PMD_NR - khugepaged_max_ptes_none: SCAN_EXCEED_NONE_PTE.
6. Else: collapse_file(mm, addr, file, start, cc).

`Khugepaged::collapse_file(mm, addr, file, start, cc) -> ScanResult`:
1. Allocate + lock new PMD-order folio via alloc_charge_folio.
2. Lock-and-replace each cache entry under XA_STATE_ORDER(xas, mapping.i_pages, start, HPAGE_PMD_ORDER).
3. Swap-in / GUP-in missing entries; on SHMEM hole: re-validate not refilled, no uffd marker.
4. Copy data from native folios → new folio.
5. xas_store(new_folio); decrement old folio refs; isolate from LRU.
6. On success: unlock new folio; free old folios; retract_page_tables(mapping, start).
7. On failure: restore originals; unlock new folio; folio_put new folio.

`Khugepaged::retract_page_tables(mapping, pgoff)`:
1. i_mmap_lock_read(mapping).
2. For each vma in mapping.i_mmap @ pgoff:
   - addr = vma.vm_start + ((pgoff - vma.vm_pgoff) << PAGE_SHIFT).
   - Skip if addr not HPAGE_PMD-aligned or doesn't fit.
   - find_pmd_or_thp_or_none(mm, addr, &pmd).
   - Skip if collapse_test_exit(mm) ∨ !file_backed_vma_is_retractable(vma).
   - mmu_notifier_invalidate_range_start.
   - pml = pmd_lock; check_pmd_state(pmd) ⇒ SUCCEED.
   - ptl = pte_lockptr; nested spin_lock_nested if distinct.
   - Recheck retractable; pgt_pmd = pmdp_collapse_flush(vma, addr, pmd); pmdp_get_lockless_sync; success = true.
   - mmu_notifier_invalidate_range_end.
   - If success: mm_dec_nr_ptes; page_table_check_pte_clear_range; pte_free_defer.
3. i_mmap_unlock_read.

`Khugepaged::do_scan(cc)`:
1. lru_add_drain_all().
2. cc.progress = 0; pass_through_head = 0; wait = true.
3. Loop:
   - cond_resched.
   - if kthread_should_stop: break.
   - spin_lock(&khugepaged_mm_lock).
   - if !khugepaged_scan.mm_slot: pass_through_head++.
   - if has_work ∧ pass_through_head < 2: scan_mm_slot(progress_max, &result, cc).
   - else: cc.progress = progress_max.
   - spin_unlock.
   - if cc.progress >= progress_max: break.
   - if result == SCAN_ALLOC_HUGE_PAGE_FAIL: if wait: alloc_sleep; wait = false; else break.

`Khugepaged::thread()`:
1. set_freezable; set_user_nice(current, MAX_NICE).
2. While !kthread_should_stop: do_scan; wait_work.
3. On exit: spin_lock; collect leftover slot; spin_unlock; return 0.

`Khugepaged::madvise_collapse(vma, start, end) -> Result<bool, Errno>`:
1. BUG_ON misalignment.
2. If !thp_vma_allowable_order(vma, vma.vm_flags, TVA_FORCED_COLLAPSE, PMD_ORDER): return Err(EINVAL).
3. cc = box CollapseControl { is_khugepaged: false, progress: 0, .. }.
4. mmgrab(mm); lru_add_drain_all.
5. hstart = round_up(start, HPAGE_PMD_SIZE); hend = end & HPAGE_PMD_MASK.
6. thps = 0; last_fail = SCAN_FAIL.
7. For addr in [hstart, hend) step HPAGE_PMD_SIZE:
   - If lock dropped from previous iteration: mmap_read_lock; revalidate; clamp hend.
   - result = collapse_single_pmd(addr, vma, &mmap_unlocked, cc).
   - Per-result switch (whitelist transient errors).
8. mmap_assert_locked; mmdrop; drop cc.
9. Return Ok(true) iff thps == hpage count, else Err(madvise_errno(last_fail)).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `mm_slot_inserted_once` | INVARIANT | per-__khugepaged_enter: MMF_VM_HUGEPAGE test_and_set guards double-insert. |
| `mm_lock_acquired_for_scan` | INVARIANT | per-collapse_scan_mm_slot: mmap_read_trylock held when walking vmas. |
| `scan_address_pmd_aligned` | INVARIANT | per-while-loop: khugepaged_scan.address & ~HPAGE_PMD_MASK == 0. |
| `collapse_huge_page_lock_order` | INVARIANT | mmap_write_lock → anon_vma_lock_write → pmd_lock → pte ptl. |
| `pmd_restored_on_isolate_fail` | INVARIANT | per-collapse_huge_page: pmd_populate runs whenever isolate returns != SUCCEED. |
| `retract_pml_then_ptl` | INVARIANT | per-retract_page_tables: pml acquired before nested ptl. |
| `mm_slot_cursor_advances` | INVARIANT | per-scan_mm_slot: khugepaged_scan.mm_slot is next or NULL after slot fully scanned. |
| `progress_bound_per_pass` | INVARIANT | per-do_scan: cc.progress ≤ khugepaged_pages_to_scan on return. |
| `knob_bounds_honored` | INVARIANT | per-scan_pmd: none_or_zero ≤ max_ptes_none ∨ SCAN_EXCEED_NONE_PTE. |

### Layer 2: TLA+

`mm/khugepaged.tla`:
- Per-enroll + per-scan + per-collapse + per-exit lifecycle.
- Properties:
  - `safety_no_collapse_into_active_vma_without_lock` — per-collapse_huge_page: mmap_write_lock held.
  - `safety_no_dual_pmd_install` — per-collapse: PMD installed atomically under pmd_ptl.
  - `safety_retract_skips_anon_vma` — per-retract: anon_vma vmas excluded.
  - `safety_madv_collapse_returns_0_iff_all_collapsed` — per-madvise_collapse postcondition.
  - `liveness_per_scan_eventually_advances` — per-do_scan: khugepaged_scan.address strictly monotone within a vma.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Khugepaged::scan_mm_slot` pre: khugepaged_mm_lock held; post: also held | `Khugepaged::scan_mm_slot` |
| `Khugepaged::collapse_huge_page` post: SUCCEED ⟹ pmd maps PMD-order folio; FAIL ⟹ pmd points at original pgtable | `Khugepaged::collapse_huge_page` |
| `Khugepaged::collapse_file` post: SUCCEED ⟹ HPAGE_PMD_NR xarray slots replaced by single high-order folio | `Khugepaged::collapse_file` |
| `Khugepaged::retract_page_tables` post: per-vma SUCCEED ⟹ mm_dec_nr_ptes called once | `Khugepaged::retract_page_tables` |
| `Khugepaged::madvise_collapse` post: returns 0 ⟺ every PMD-aligned span collapsed | `Khugepaged::madvise_collapse` |
| `Khugepaged::do_scan` post: cc.progress ≤ progress_max | `Khugepaged::do_scan` |

### Layer 4: Verus/Creusot functional

`Per-anonymous PMD: scan ptes → eligible? → unlock-read → alloc_charge_folio → relock → revalidate → write-lock → collapse_flush → isolate → copy → map_pmd → unlock` semantic equivalence: per-Documentation/admin-guide/mm/transhuge.rst § "khugepaged".

`Per-file PMD: scan xarray → eligible? → unlock-read → collapse_file (alloc, replace, copy, retract) → done` semantic equivalence: per-Documentation/admin-guide/mm/transhuge.rst § "shmem/tmpfs huge pages".

## Hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening.)

khugepaged reinforcement:

- **Per-MMF_VM_HUGEPAGE test_and_set enroll** — defense against per-mm_slot double-insert and per-list corruption.
- **Per-mmap_read_trylock for scan path** — defense against per-blocking-on-foreign-mm starvation.
- **Per-mmap_write_lock for collapse_huge_page** — defense against per-concurrent-fault / per-rmap-walk races.
- **Per-pmd_populate restore on isolate failure** — defense against per-orphaned-PMD that points at freed pgtable.
- **Per-anon_vma_lock_write before pmd flush** — defense against per-rmap-walk seeing torn-down PMD.
- **Per-mmu_notifier_invalidate_range surrounding pmd flush** — defense against per-secondary-MMU (KVM, IOMMU) stale-mapping.
- **Per-file_backed_vma_is_retractable strict (anon_vma ∨ uffd-wp ∨ VMA_MAYBE_GUARD_BIT excluded)** — defense against per-COW / per-uffd / per-guard-region corruption.
- **Per-userfaultfd_armed bypasses max_ptes_none** — defense against per-uffd missing-page silent-fill.
- **Per-folio_expected_ref_count + 1 == folio_ref_count check** — defense against per-pinned-page (GUP-fast) collapse races.
- **Per-collapse_test_exit gate** — defense against per-mm-teardown-during-scan UAF.
- **Per-khugepaged_alloc_sleep on alloc fail** — defense against per-OOM-spin and per-CPU-monopolization.
- **Per-MADV_COLLAPSE TVA_FORCED_COLLAPSE strict-only path** — defense against per-unwanted bypass of THP sysfs policy by kthread.
- **Per-sysfs knob input validation (≤ KHUGEPAGED_MAX_PTES_LIMIT)** — defense against per-overflow / per-DoS.
- **Per-pte_offset_map_lock for pte iter** — defense against per-pte race with parallel teardown.
- **Per-cgroup-charge propagation (SCAN_CGROUP_CHARGE_FAIL)** — defense against per-cgroup-limit bypass via collapse.
- **Per-pmdp_collapse_flush atomic vs GUP-fast** — defense against per-stale-GUP-pin after collapse.
- **Per-pte_free_defer for retracted page tables** — defense against per-immediate-free vs concurrent gup_fast.

## Grsecurity/PaX-style Reinforcement

Baseline grsec/PaX features inherited project-wide:

- **PAX_USERCOPY** — collapse target pages copied via copy_highpage are slab-whitelist-respecting; metadata transfers boundary-checked.
- **PAX_KERNEXEC** — khugepaged kthread text W^X; collapse op dispatch table read-only post-init.
- **PAX_RANDKSTACK** — khugepaged scan loop entry-stack and MADV_COLLAPSE syscall entry randomized.
- **PAX_REFCOUNT** — folio refcount, mapcount, and mm_slot refcount saturate; expected-vs-actual ref mismatch aborts collapse before publish.
- **PAX_MEMORY_SANITIZE** — retracted page tables zero-poisoned via pte_free_defer; collapsed-out small pages zero-poisoned before release to buddy.
- **PAX_UDEREF** — MADV_COLLAPSE start/len validated against vma extent; out-of-range refused before scan.
- **PAX_RAP / kCFI** — vm_ops collapse-related callbacks type-tagged; cross-fs collapse handler swap refused.
- **GRKERNSEC_HIDESYM** — `khugepaged_do_scan`, `collapse_huge_page`, `khugepaged_scan_mm_slot` absent from `/proc/kallsyms` for unpriv.
- **GRKERNSEC_DMESG** — khugepaged collapse trace and SCAN_* failure reasons redacted from dmesg for non-CAP_SYSLOG.

khugepaged-specific reinforcements:

- **Collapse PAX_USERCOPY-respecting copy** — copy_highpage from 512 base pages into the THP target stays inside slab whitelist invariants; the destination folio is not exposed until copy verified.
- **MADV_COLLAPSE CAP_SYS_NICE gating** — forced/synchronous collapse on behalf of userspace requires CAP_SYS_NICE (treating compaction-class pressure as a niceness-style privilege) to prevent unpriv DoS via THP churn.
- **folio_expected_ref_count + 1 == folio_ref_count strict check** — any extra pin (GUP-fast / driver) aborts the collapse before pmd_populate; eliminates stale-pin-vs-collapse UAF class.
- **mmap_write_lock held across pmdp_collapse_flush** — VMA mutation cannot race with collapse; mmu_notifier invalidate brackets the flush for secondary MMUs.
- **mm_slot refcount + collapse_test_exit gate** — mm teardown during scan refused; kthread sees a quiescent or absent mm, never a partially-torn-down one.

Rationale: khugepaged collapses 512 base pages into a 2MB THP while the mm is potentially active across CPUs, GUP-fast, KVM, and IOMMU mirrors — any refcount slip or notifier gap is a multi-subsystem UAF. Grsec emphasis: saturate refcount checks (expected+1 == actual), gate user-driven collapse behind CAP_SYS_NICE so an attacker cannot trigger unbounded THP churn, and treat retracted page tables as quarantine-only via pte_free_defer.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- mm/huge_memory.c THP fault + split paths (covered in `thp.md` Tier-3).
- mm/shmem.c tmpfs THP allocation (covered in `swap.md` / shmem Tier-3 expansion).
- mm/memcontrol.c memcg charge logic for huge folios (covered in `memcg.md` Tier-3).
- mm/compaction.c defrag for PMD-order alloc (covered in `migration-compaction.md` Tier-3).
- mm/userfaultfd.c uffd-wp marker semantics (covered in `userfaultfd.md` Tier-3).
- Implementation code.
