---
title: "Tier-3: mm/mmu_gather.c — TLB flush batching (mmu_gather)"
tags: ["tier-3", "mm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

The **mmu_gather** structure batches page-table tear-down work — pages to free, page-table pages to RCU-free, and a single TLB-flush range — so that operations such as `munmap`, `exit`, `mremap`, `mprotect-cow`, and OOM-reap pay one TLB-shootdown rather than one per PTE. Per-call-site `struct mmu_gather` lives on the kernel stack; `tlb_gather_mmu(tlb, mm)` initializes it, the caller walks PTEs invoking `tlb_remove_page()` / `__tlb_remove_folio_pages()` and (for page-table pages) `tlb_remove_table()`, and `tlb_finish_mmu(tlb)` flushes the TLB once and frees everything. Per-`struct mmu_gather_batch` is a chain of `__get_free_page()`-allocated arrays of `encoded_page *` entries (encoding low-bit flags `ENCODED_PAGE_BIT_DELAY_RMAP` / `ENCODED_PAGE_BIT_NR_PAGES_NEXT`). Per-`struct mmu_table_batch` separately chains freed page-table pages and (under `CONFIG_MMU_GATHER_RCU_TABLE_FREE`) RCU-frees them so gup_fast / lockless walkers cannot observe a freed directory. Per-`tlb->fullmm` selects whole-mm flush (exit/execve fast path). Per-`mm_tlb_flush_nested(tlb->mm)` detects parallel batched flushers and force-promotes to fullmm to prevent stale TLB. Per-`tlb->delayed_rmap` defers rmap removal until after TLB flush. Critical for: correct TLB semantics under concurrent walkers, ZAP-fast-path performance, gup_fast safety, OOM-reaper progress.

This Tier-3 covers `mm/mmu_gather.c` (~555 lines).

### Acceptance Criteria

- [ ] AC-1: tlb_gather_mmu(tlb, mm) initializes tlb.active==&tlb.local, batch_count==0, increments tlb_flush_pending.
- [ ] AC-2: tlb_finish_mmu calls arch tlb_flush exactly once over [tlb.start, tlb.end] when !fullmm.
- [ ] AC-3: Per-fullmm: tlb_finish_mmu issues a fullmm flush, ignoring start/end.
- [ ] AC-4: Per-mm_tlb_flush_nested true: fullmm promotion + freed_tables=1.
- [ ] AC-5: __tlb_remove_page_size accumulates pages until batch full; returns true exactly when next_batch fails.
- [ ] AC-6: Encoded nr_pages cell follows the page cell when nr_pages > 1.
- [ ] AC-7: ENCODED_PAGE_BIT_DELAY_RMAP causes rmap removal deferred until tlb_flush_rmaps.
- [ ] AC-8: tlb_flush_rmaps visits each delayed encoded_page exactly once and clears tlb.delayed_rmap.
- [ ] AC-9: tlb_remove_table batches up to MAX_TABLE_BATCH then auto-flushes.
- [ ] AC-10: CONFIG_MMU_GATHER_RCU_TABLE_FREE: table batch freed via call_rcu (not directly).
- [ ] AC-11: OOM in tlb_remove_table allocates: falls back to single tlb_remove_table_one path.
- [ ] AC-12: tlb_batch_pages_flush iterates entire chain head-to-active, freeing in MAX_NR_FOLIOS_PER_FREE chunks with cond_resched.
- [ ] AC-13: tlb_batch_list_free frees only batches past &tlb.local (the on-stack one is not freed).
- [ ] AC-14: tlb_gather_mmu_vma initializes huge page_size for hugetlb VMA.
- [ ] AC-15: tlb_change_page_size on size change with pending batch flushes mmu before changing.

### Architecture

```
struct MmuGather {
  mm: *MmStruct,
  fullmm: u32,                            // 1 bit
  need_flush_all: u32,                    // 1 bit
  freed_tables: u32,                      // 1 bit
  delayed_rmap: u32,                      // 1 bit
  fully_unshared_tables: u32,             // 1 bit
  cleared_ptes: u32,                      // 1 bit
  cleared_pmds: u32,                      // 1 bit
  cleared_puds: u32,                      // 1 bit
  cleared_p4ds: u32,                      // 1 bit
  vma_exec: u32,                          // 1 bit
  vma_huge: u32,                          // 1 bit
  vma_pfn: u32,                           // 1 bit
  start: u64,                             // range start
  end: u64,                               // range end
  page_size: u64,                         // CONFIG_MMU_GATHER_PAGE_SIZE
  batch: *MmuTableBatch,                  // CONFIG_MMU_GATHER_TABLE_FREE
  local: MmuGatherBatch,                  // on-stack initial
  active: *MmuGatherBatch,
  batch_count: i32,
  __pages: [*EncodedPage; MMU_GATHER_BUNDLE],
}

struct MmuGatherBatch {
  next: *MmuGatherBatch,
  nr: u32,
  max: u32,
  encoded_pages: [*EncodedPage; 0],       // variable
}

struct MmuTableBatch {
  rcu: RcuHead,
  nr: u32,
  tables: [*void; MAX_TABLE_BATCH],
}

const MAX_GATHER_BATCH:        u32 = (PAGE_SIZE - sizeof(MmuGatherBatch)) / sizeof(*EncodedPage);
const MAX_GATHER_BATCH_COUNT:  u32 = 10000;
const MAX_TABLE_BATCH:         u32 = (PAGE_SIZE - sizeof(MmuTableBatch)) / sizeof(*void);
const MAX_NR_FOLIOS_PER_FREE:  u32 = 512;

const ENCODED_PAGE_BIT_DELAY_RMAP:    u64 = 1 << 0;
const ENCODED_PAGE_BIT_NR_PAGES_NEXT: u64 = 1 << 1;
```

`MmuGather::gather(self, mm)`:
1. self.gather_inner(mm, false).

`MmuGather::gather_fullmm(self, mm)`:
1. self.gather_inner(mm, true).

`MmuGather::gather_inner(self, mm, fullmm)`:
1. self.mm = mm; self.fullmm = fullmm as u32.
2. /* !MMU_GATHER_NO_GATHER */
3. self.need_flush_all = 0.
4. self.local.next = NULL; self.local.nr = 0; self.local.max = ARRAY_SIZE(self.__pages).
5. self.active = &self.local.
6. self.batch_count = 0.
7. self.delayed_rmap = 0.
8. self.table_init  /* self.batch = NULL */.
9. /* CONFIG_MMU_GATHER_PAGE_SIZE */ self.page_size = 0.
10. self.vma_pfn = 0; self.fully_unshared_tables = 0.
11. __tlb_reset_range(self)  /* start = TASK_SIZE; end = 0; cleared_* = 0; freed_tables = 0 */
12. inc_tlb_flush_pending(self.mm).

`MmuGather::gather_vma(self, vma)`:
1. self.gather(vma.vm_mm).
2. tlb_update_vma_flags(self, vma).
3. if is_vm_hugetlb_page(vma): tlb_change_page_size(self, huge_page_size(hstate_vma(vma))).

`MmuGather::finish(self)`:
1. VM_WARN_ON_ONCE(self.fully_unshared_tables).
2. if mm_tlb_flush_nested(self.mm):
   - /* Concurrent batched PTE changes detected — force full flush */
   - self.fullmm = 1; __tlb_reset_range(self); self.freed_tables = 1.
3. self.flush.
4. /* !MMU_GATHER_NO_GATHER */ self.batch_list_free.
5. dec_tlb_flush_pending(self.mm).

`MmuGather::flush(self)`:
1. tlb_flush_mmu_tlbonly(self)  /* per-arch tlb_flush(self) */
2. self.flush_free.

`MmuGather::flush_free(self)`:
1. self.table_flush.
2. /* !MMU_GATHER_NO_GATHER */ self.batch_pages_flush.

`MmuGather::remove_page_size(self, page, page_size) -> bool`:
1. return self.remove_folio_pages_size(page, 1, false, page_size).

`MmuGather::remove_folio_pages(self, page, nr_pages, delay_rmap) -> bool`:
1. return self.remove_folio_pages_size(page, nr_pages, delay_rmap, PAGE_SIZE).

`MmuGather::remove_folio_pages_size(self, page, nr_pages, delay_rmap, page_size) -> bool`:
1. VM_BUG_ON(self.end == 0)  /* must be in a range-clear */
2. /* CONFIG_MMU_GATHER_PAGE_SIZE */ VM_WARN_ON(self.page_size != page_size); VM_WARN_ON_ONCE(nr_pages != 1 ∧ page_size != PAGE_SIZE); VM_WARN_ON_ONCE(page_folio(page) != page_folio(page + nr_pages - 1)).
3. flags = if delay_rmap { ENCODED_PAGE_BIT_DELAY_RMAP } else { 0 }.
4. batch = self.active.
5. if nr_pages == 1:
   - batch.encoded_pages[batch.nr] = encode_page(page, flags); batch.nr += 1.
6. else:
   - flags |= ENCODED_PAGE_BIT_NR_PAGES_NEXT.
   - batch.encoded_pages[batch.nr] = encode_page(page, flags); batch.nr += 1.
   - batch.encoded_pages[batch.nr] = encode_nr_pages(nr_pages); batch.nr += 1.
7. if batch.nr >= batch.max - 1:
   - if !self.next_batch: return true  /* caller must flush */
   - batch = self.active.
8. VM_BUG_ON_PAGE(batch.nr > batch.max - 1, page).
9. return false.

`MmuGather::next_batch(self) -> bool`:
1. /* Delayed rmap requires staying on local batch */
2. if self.delayed_rmap ∧ self.active != &self.local: return false.
3. batch = self.active.
4. if batch.next: self.active = batch.next; return true.
5. if self.batch_count == MAX_GATHER_BATCH_COUNT: return false.
6. batch = __get_free_page(GFP_NOWAIT) as *MmuGatherBatch.
7. if !batch: return false.
8. self.batch_count += 1.
9. batch.next = NULL; batch.nr = 0; batch.max = MAX_GATHER_BATCH.
10. self.active.next = batch; self.active = batch.
11. return true.

`MmuGather::batch_pages_flush(self)`:
1. batch = &self.local.
2. while batch ∧ batch.nr:
   - __tlb_batch_free_encoded_pages(batch).
   - batch = batch.next.
3. self.active = &self.local.

`MmuGather::batch_free_encoded_pages(batch)`:
1. pages = batch.encoded_pages.
2. while batch.nr:
   - if !page_poisoning_enabled_static ∧ !want_init_on_free:
     - nr = min(MAX_NR_FOLIOS_PER_FREE, batch.nr).
     - /* avoid stranding nr_pages-cell */
     - if encoded_page_flags(pages[nr-1]) & ENCODED_PAGE_BIT_NR_PAGES_NEXT: nr += 1.
   - else:
     - /* poisoning/init: bound by pages, not folios */
     - nr = 0; nr_pages = 0.
     - while nr < batch.nr ∧ nr_pages < MAX_NR_FOLIOS_PER_FREE:
       - if encoded_page_flags(pages[nr]) & ENCODED_PAGE_BIT_NR_PAGES_NEXT:
         - nr += 1; nr_pages += encoded_nr_pages(pages[nr]).
       - else: nr_pages += 1.
       - nr += 1.
   - free_pages_and_swap_cache(pages, nr).
   - pages += nr; batch.nr -= nr.
   - cond_resched.

`MmuGather::batch_list_free(self)`:
1. batch = self.local.next.
2. while batch:
   - next = batch.next.
   - free_pages(batch as u64, 0).
   - batch = next.
3. self.local.next = NULL.

`MmuGather::flush_rmaps(self, vma)`:
1. if !self.delayed_rmap: return.
2. self.flush_rmap_batch(&self.local, vma).
3. if self.active != &self.local: self.flush_rmap_batch(self.active, vma).
4. self.delayed_rmap = 0.

`MmuGather::flush_rmap_batch(batch, vma)`:
1. pages = batch.encoded_pages.
2. for i in 0..batch.nr:
   - enc = pages[i].
   - if !(encoded_page_flags(enc) & ENCODED_PAGE_BIT_DELAY_RMAP): continue.
   - page = encoded_page_ptr(enc); nr_pages = 1.
   - if encoded_page_flags(enc) & ENCODED_PAGE_BIT_NR_PAGES_NEXT:
     - i += 1; nr_pages = encoded_nr_pages(pages[i]).
   - folio_remove_rmap_ptes(page_folio(page), page, nr_pages, vma).

`MmuGather::remove_table(self, table)`:  /* CONFIG_MMU_GATHER_TABLE_FREE */
1. batch = &self.batch.
2. if !*batch:
   - *batch = __get_free_page(GFP_NOWAIT) as *MmuTableBatch.
   - if !*batch:  /* OOM fallback: invalidate + free single */
     - self.table_invalidate.
     - self.remove_table_one(table).
     - return.
   - (*batch).nr = 0.
3. (*batch).tables[(*batch).nr] = table; (*batch).nr += 1.
4. if (*batch).nr == MAX_TABLE_BATCH: self.table_flush.

`MmuGather::table_flush(self)`:
1. batch = &self.batch.
2. if *batch:
   - self.table_invalidate.
   - tlb_remove_table_free(*batch)  /* call_rcu or direct */
   - *batch = NULL.

`MmuGather::table_invalidate(self)`:
1. if tlb_needs_table_invalidate(): tlb_flush_mmu_tlbonly(self).

`MmuGather::remove_table_free(batch)`:  /* CONFIG_MMU_GATHER_RCU_TABLE_FREE */
1. call_rcu(&batch.rcu, tlb_remove_table_rcu).

`MmuGather::remove_table_rcu(head)`:
1. batch = container_of(head, MmuTableBatch, rcu).
2. for t in batch.tables[0..batch.nr]: __tlb_remove_table(t)  /* per-arch free */.
3. free_page(batch).

`MmuGather::sync_one()`:
1. smp_call_function(tlb_remove_table_smp_sync, NULL, 1)  /* empty IPI handler — synchronize with IRQ-disabled lockless walkers (gup_fast) */.

`MmuGather::sync_rcu()`:
1. synchronize_rcu()  /* IRQ-disabled walkers are RCU read-side critical sections */.

`MmuGather::remove_table_one(table)`:
1. /* CONFIG_PT_RECLAIM */ ptdesc = table; call_rcu(&ptdesc.pt_rcu_head, __tlb_remove_table_one_rcu).
2. /* !CONFIG_PT_RECLAIM */ tlb_remove_table_sync_rcu(); __tlb_remove_table(table).

### Out of Scope

- arch/<arch>/mm/tlb*.c per-arch flush (covered separately if expanded)
- mm/memory.c zap_pte_range / unmap_page_range (covered in `virtual-memory.md` Tier-3)
- mm/mmap.c munmap / exit_mmap (covered in `mmap.md` Tier-3)
- mm/mremap.c / mprotect.c (covered in `mremap.md` / `mprotect.md` Tier-3)
- mm/gup.c gup_fast (covered in `gup.md` Tier-3)
- mm/hugetlb.c huge-page unmap (covered in `hugetlb.md` Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct mmu_gather` | per-tear-down context | `MmuGather` |
| `struct mmu_gather_batch` | per-page-batch array | `MmuGatherBatch` |
| `struct mmu_table_batch` | per-page-table-page batch | `MmuTableBatch` |
| `struct encoded_page` | per-flag-encoded `struct page *` | `EncodedPage` |
| `tlb_gather_mmu()` | per-init (non-full) | `MmuGather::gather` |
| `tlb_gather_mmu_fullmm()` | per-init (exit/execve) | `MmuGather::gather_fullmm` |
| `tlb_gather_mmu_vma()` | per-init (single VMA + huge page-size) | `MmuGather::gather_vma` |
| `__tlb_gather_mmu()` | per-init common | `MmuGather::gather_inner` |
| `tlb_finish_mmu()` | per-flush + free + destruct | `MmuGather::finish` |
| `tlb_flush_mmu()` | per-flush TLB + free batches | `MmuGather::flush` |
| `tlb_flush_mmu_free()` | per-free without TLB-flush | `MmuGather::flush_free` |
| `__tlb_remove_page_size()` | per-add page (single) | `MmuGather::remove_page_size` |
| `__tlb_remove_folio_pages()` | per-add page (folio range) | `MmuGather::remove_folio_pages` |
| `__tlb_remove_folio_pages_size()` | per-add core | `MmuGather::remove_folio_pages_size` |
| `tlb_next_batch()` | per-grow batch chain | `MmuGather::next_batch` |
| `tlb_batch_pages_flush()` | per-walk + free batches | `MmuGather::batch_pages_flush` |
| `tlb_batch_list_free()` | per-free batch nodes | `MmuGather::batch_list_free` |
| `__tlb_batch_free_encoded_pages()` | per-free batch entries | `MmuGather::batch_free_encoded_pages` |
| `tlb_flush_rmaps()` | per-deferred-rmap commit | `MmuGather::flush_rmaps` |
| `tlb_flush_rmap_batch()` | per-batch rmap removal | `MmuGather::flush_rmap_batch` |
| `tlb_remove_table()` | per-add page-table page | `MmuGather::remove_table` |
| `tlb_table_flush()` | per-flush page-table batch | `MmuGather::table_flush` |
| `tlb_table_invalidate()` | per-flush TLB pre-table-free | `MmuGather::table_invalidate` |
| `tlb_table_init()` | per-init table batch | `MmuGather::table_init` |
| `__tlb_remove_table_free()` | per-free table batch | `MmuGather::remove_table_free` |
| `tlb_remove_table_free()` | per-RCU-or-direct free | `MmuGather::table_free` |
| `tlb_remove_table_rcu()` | per-RCU callback | `MmuGather::remove_table_rcu` |
| `tlb_remove_table_sync_one()` | per-IPI sync | `MmuGather::sync_one` |
| `tlb_remove_table_sync_rcu()` | per-RCU sync | `MmuGather::sync_rcu` |
| `tlb_remove_table_one()` | per-fallback single | `MmuGather::remove_table_one` |
| `__tlb_remove_table_one()` | per-fallback single (PT_RECLAIM aware) | `MmuGather::remove_table_one_inner` |
| `tlb_remove_table_smp_sync()` | per-IPI handler (empty) | `MmuGather::smp_sync` |
| `encode_page()` / `encoded_page_ptr()` / `encoded_page_flags()` | per-low-bit pack/unpack | `EncodedPage::encode` / `ptr` / `flags` |
| `encode_nr_pages()` / `encoded_nr_pages()` | per-nr-pages-cell pack/unpack | `EncodedPage::encode_nr` / `nr_pages` |
| `MAX_GATHER_BATCH` / `MAX_GATHER_BATCH_COUNT` / `MAX_TABLE_BATCH` / `MAX_NR_FOLIOS_PER_FREE` | per-bound | `mmu_gather` constants |

### compatibility contract

REQ-1: struct mmu_gather:
- mm: per-target mm_struct.
- fullmm: per-tear-down-everything flag (exit/execve).
- need_flush_all: per-arch-extension flag.
- start / end: per-range covering all gathered PTE clears.
- vma_pfn: per-(packed) vma flags for tlb_start_vma/tlb_end_vma optimizations.
- freed_tables: per-page-table-pages-were-freed flag.
- cleared_ptes / cleared_pmds / cleared_puds / cleared_p4ds: per-which-level-was-cleared bitmap.
- vma_huge / vma_exec / vma_pfn (packed bits): per-VMA-flags-summary.
- batch: per-mmu_table_batch chain (page-table pages) (CONFIG_MMU_GATHER_TABLE_FREE).
- local: per-on-stack initial mmu_gather_batch.
- active: per-current batch (== &local or chained).
- batch_count: per-extra-batches-allocated.
- __pages[MMU_GATHER_BUNDLE]: per-local-array inside batch.local.
- delayed_rmap: per-defer-rmap-until-after-TLB-flush flag.
- fully_unshared_tables: per-pmd-unshare-was-done flag.
- page_size: per-CONFIG_MMU_GATHER_PAGE_SIZE: enforces single-size per batch.

REQ-2: struct mmu_gather_batch:
- next: per-link.
- nr / max: per-array-occupancy.
- encoded_pages[]: per-`encoded_page *` array (flag-encoded `struct page *`).

REQ-3: struct mmu_table_batch:
- rcu: per-rcu_head for call_rcu.
- nr / tables[MAX_TABLE_BATCH]: per-page-table-page pointers.

REQ-4: encoded_page encoding:
- Low bit ENCODED_PAGE_BIT_DELAY_RMAP: per-defer-rmap-removal until tlb_flush_rmaps.
- Low bit ENCODED_PAGE_BIT_NR_PAGES_NEXT: per-next-cell holds nr_pages encoded via encode_nr_pages.
- High bits: per-`struct page *` payload.

REQ-5: __tlb_gather_mmu(tlb, mm, fullmm):
- tlb.mm = mm; tlb.fullmm = fullmm.
- /* !MMU_GATHER_NO_GATHER */ tlb.need_flush_all = 0; tlb.local.next = NULL; tlb.local.nr = 0; tlb.local.max = ARRAY_SIZE(tlb.__pages); tlb.active = &tlb.local; tlb.batch_count = 0.
- tlb.delayed_rmap = 0.
- tlb_table_init(tlb)  /* tlb.batch = NULL */
- /* CONFIG_MMU_GATHER_PAGE_SIZE */ tlb.page_size = 0.
- tlb.vma_pfn = 0.
- tlb.fully_unshared_tables = 0.
- __tlb_reset_range(tlb)  /* start = TASK_SIZE; end = 0; freed_tables = 0; cleared_* = 0 */
- inc_tlb_flush_pending(tlb.mm)  /* paired with dec in tlb_finish_mmu */.

REQ-6: tlb_gather_mmu(tlb, mm):
- __tlb_gather_mmu(tlb, mm, false).

REQ-7: tlb_gather_mmu_fullmm(tlb, mm):
- /* exit_mmap / execve - mm has no users; tear down full address space */
- __tlb_gather_mmu(tlb, mm, true).

REQ-8: tlb_gather_mmu_vma(tlb, vma):
- tlb_gather_mmu(tlb, vma.vm_mm).
- tlb_update_vma_flags(tlb, vma)  /* set vma_huge/vma_exec/vma_pfn bits */
- if is_vm_hugetlb_page(vma): tlb_change_page_size(tlb, huge_page_size(hstate_vma(vma))).

REQ-9: tlb_finish_mmu(tlb):
- VM_WARN_ON_ONCE(tlb.fully_unshared_tables).
- if mm_tlb_flush_nested(tlb.mm):  /* parallel batched PTE changes detected */
  - tlb.fullmm = 1; __tlb_reset_range(tlb); tlb.freed_tables = 1.
- tlb_flush_mmu(tlb)  /* one TLB flush, then free batches */
- /* !MMU_GATHER_NO_GATHER */ tlb_batch_list_free(tlb).
- dec_tlb_flush_pending(tlb.mm).

REQ-10: tlb_flush_mmu(tlb):
- tlb_flush_mmu_tlbonly(tlb)  /* invokes per-arch tlb_flush(tlb) */
- tlb_flush_mmu_free(tlb)  /* table_flush + batch_pages_flush */

REQ-11: tlb_flush_mmu_free(tlb):
- tlb_table_flush(tlb)  /* page-table-page batch */
- /* !MMU_GATHER_NO_GATHER */ tlb_batch_pages_flush(tlb)  /* page batches */

REQ-12: __tlb_remove_page_size(tlb, page, page_size) -> bool:
- /* Returns true ⟹ caller must call tlb_flush_mmu before continuing */
- return __tlb_remove_folio_pages_size(tlb, page, 1, false, page_size).

REQ-13: __tlb_remove_folio_pages(tlb, page, nr_pages, delay_rmap) -> bool:
- return __tlb_remove_folio_pages_size(tlb, page, nr_pages, delay_rmap, PAGE_SIZE).

REQ-14: __tlb_remove_folio_pages_size(tlb, page, nr_pages, delay_rmap, page_size) -> bool:
- VM_BUG_ON(!tlb.end)  /* PTE clears must have already happened — tlb.end set by __tlb_adjust_range */
- /* CONFIG_MMU_GATHER_PAGE_SIZE */ VM_WARN_ON(tlb.page_size != page_size); VM_WARN_ON_ONCE(nr_pages != 1 ∧ page_size != PAGE_SIZE).
- flags = delay_rmap ? ENCODED_PAGE_BIT_DELAY_RMAP : 0.
- batch = tlb.active.
- if nr_pages == 1: batch.encoded_pages[batch.nr++] = encode_page(page, flags).
- else:
  - flags |= ENCODED_PAGE_BIT_NR_PAGES_NEXT.
  - batch.encoded_pages[batch.nr++] = encode_page(page, flags).
  - batch.encoded_pages[batch.nr++] = encode_nr_pages(nr_pages).
- /* leave room for next (page, nr_pages) pair */
- if batch.nr >= batch.max - 1:
  - if !tlb_next_batch(tlb): return true  /* caller must flush */
  - batch = tlb.active.
- return false.

REQ-15: tlb_next_batch(tlb) -> bool:
- /* Limit batching if delayed rmaps pending - those must flush together */
- if tlb.delayed_rmap ∧ tlb.active != &tlb.local: return false.
- batch = tlb.active.
- if batch.next: tlb.active = batch.next; return true.
- if tlb.batch_count == MAX_GATHER_BATCH_COUNT: return false.
- batch = (mmu_gather_batch *)__get_free_page(GFP_NOWAIT).
- if !batch: return false.
- tlb.batch_count += 1.
- batch.next = NULL; batch.nr = 0; batch.max = MAX_GATHER_BATCH.
- tlb.active.next = batch; tlb.active = batch.
- return true.

REQ-16: tlb_batch_pages_flush(tlb):
- /* Walk batch chain head-to-tail, free pages */
- for batch in &tlb.local..; batch ∧ batch.nr; batch = batch.next:
  - __tlb_batch_free_encoded_pages(batch).
- tlb.active = &tlb.local.

REQ-17: __tlb_batch_free_encoded_pages(batch):
- /* Iterate batch.encoded_pages in MAX_NR_FOLIOS_PER_FREE chunks; cond_resched between chunks */
- while batch.nr:
  - if !page_poisoning_enabled_static ∧ !want_init_on_free:
    - nr = min(MAX_NR_FOLIOS_PER_FREE, batch.nr).
    - /* don't leave a nr_pages-cell behind when capping */
    - if (encoded_page_flags(pages[nr-1]) & ENCODED_PAGE_BIT_NR_PAGES_NEXT): nr++.
  - else:
    - /* poisoning/init grows with size; cap by pages, not folios */
    - nr = 0; nr_pages = 0.
    - while nr < batch.nr ∧ nr_pages < MAX_NR_FOLIOS_PER_FREE:
      - if encoded_page_flags(pages[nr]) & ENCODED_PAGE_BIT_NR_PAGES_NEXT: nr_pages += encoded_nr_pages(pages[++nr]).
      - else: nr_pages += 1.
      - nr += 1.
  - free_pages_and_swap_cache(pages, nr).
  - pages += nr; batch.nr -= nr.
  - cond_resched.

REQ-18: tlb_batch_list_free(tlb):
- /* Free batch nodes (chained) after pages drained */
- for batch in tlb.local.next..; batch; batch = next:
  - next = batch.next.
  - free_pages((unsigned long)batch, 0).
- tlb.local.next = NULL.

REQ-19: tlb_flush_rmap_batch(batch, vma):
- for i in 0..batch.nr:
  - enc = batch.encoded_pages[i].
  - if encoded_page_flags(enc) & ENCODED_PAGE_BIT_DELAY_RMAP:
    - page = encoded_page_ptr(enc); nr_pages = 1.
    - if (encoded_page_flags(enc) & ENCODED_PAGE_BIT_NR_PAGES_NEXT): nr_pages = encoded_nr_pages(pages[++i]).
    - folio_remove_rmap_ptes(page_folio(page), page, nr_pages, vma).

REQ-20: tlb_flush_rmaps(tlb, vma):
- /* Called after TLB flush; commits deferred rmap removals so other walkers cannot reach a stale rmap */
- if !tlb.delayed_rmap: return.
- tlb_flush_rmap_batch(&tlb.local, vma).
- if tlb.active != &tlb.local: tlb_flush_rmap_batch(tlb.active, vma).
- tlb.delayed_rmap = 0.

REQ-21: tlb_remove_table(tlb, table):  /* CONFIG_MMU_GATHER_TABLE_FREE */
- batch = &tlb.batch.
- if *batch == NULL:
  - *batch = (mmu_table_batch *)__get_free_page(GFP_NOWAIT).
  - if *batch == NULL:  /* OOM-fallback: invalidate + free single */
    - tlb_table_invalidate(tlb).
    - tlb_remove_table_one(table).
    - return.
  - (*batch).nr = 0.
- (*batch).tables[(*batch).nr++] = table.
- if (*batch).nr == MAX_TABLE_BATCH: tlb_table_flush(tlb).

REQ-22: tlb_table_flush(tlb):
- batch = &tlb.batch.
- if *batch: tlb_table_invalidate(tlb); tlb_remove_table_free(*batch); *batch = NULL.

REQ-23: tlb_table_invalidate(tlb):
- if tlb_needs_table_invalidate(): tlb_flush_mmu_tlbonly(tlb)  /* flush TLB before allowing page-table page free, then RCU-wait for software walkers */.

REQ-24: tlb_remove_table_free(batch):  /* CONFIG_MMU_GATHER_RCU_TABLE_FREE */
- call_rcu(&batch.rcu, tlb_remove_table_rcu).

REQ-25: tlb_remove_table_rcu(head):
- __tlb_remove_table_free(container_of(head, mmu_table_batch, rcu)).

REQ-26: __tlb_remove_table_free(batch):
- for i in 0..batch.nr: __tlb_remove_table(batch.tables[i])  /* per-arch */
- free_page(batch).

REQ-27: tlb_remove_table_free non-RCU (!CONFIG_MMU_GATHER_RCU_TABLE_FREE):
- __tlb_remove_table_free(batch)  /* direct */.

REQ-28: tlb_remove_table_sync_one():  /* CONFIG_MMU_GATHER_RCU_TABLE_FREE */
- smp_call_function(tlb_remove_table_smp_sync, NULL, 1)  /* empty IPI handler — synchronizes against IRQ-disabled lockless walkers */.

REQ-29: tlb_remove_table_sync_rcu():
- synchronize_rcu()  /* equivalent to IPI broadcast but quiescence via grace period; use in sleepable slow paths */.

REQ-30: tlb_remove_table_one(table):
- __tlb_remove_table_one(table)
  - CONFIG_PT_RECLAIM: ptdesc = table; call_rcu(&ptdesc.pt_rcu_head, __tlb_remove_table_one_rcu).
  - !CONFIG_PT_RECLAIM: tlb_remove_table_sync_rcu(); __tlb_remove_table(table).

REQ-31: Per-mm_tlb_flush_nested force-flush (tlb_finish_mmu):
- /* PTE writes by other threads under shared mmap_lock + their own mmu_gather can race - force fullmm and freed_tables=1 */
- if mm_tlb_flush_nested(tlb.mm): tlb.fullmm = 1; __tlb_reset_range; tlb.freed_tables = 1.

REQ-32: Per-CONFIG_MMU_GATHER_NO_GATHER:
- Architectures with arch-side full-batching (e.g., powerpc) bypass page-batch chain entirely; only the table-batch path is used.

REQ-33: tlb_change_page_size(tlb, new_size) (CONFIG_MMU_GATHER_PAGE_SIZE):
- /* If pre-existing batches at different page_size: flush first */
- if tlb.page_size ∧ tlb.page_size != new_size: tlb_flush_mmu(tlb).
- tlb.page_size = new_size.

REQ-34: Force-flush from caller on full-batch:
- A caller that gets `true` from `__tlb_remove_page_size()` MUST call `tlb_flush_mmu(tlb)` before continuing to clear PTEs. This is the "force_flush" contract used by zap_pte_range.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `single_tlb_flush_per_finish` | INVARIANT | per-tlb_finish_mmu: tlb_flush invoked at most once on the gathered range. |
| `batch_chain_acyclic` | INVARIANT | per-next_batch: new batch always appended; no cycles in batch chain. |
| `nr_pages_cell_followed` | INVARIANT | per-remove_folio_pages_size: nr_pages > 1 ⟹ two cells written, NR_PAGES_NEXT flag on first. |
| `delay_rmap_keeps_active_local` | INVARIANT | per-next_batch: delayed_rmap ∧ active != local ⟹ false. |
| `flush_pending_paired` | INVARIANT | per-gather/finish: inc_tlb_flush_pending paired with dec. |
| `table_batch_rcu_freed` | INVARIANT | per-CONFIG_MMU_GATHER_RCU_TABLE_FREE: table batch reaches free only via call_rcu. |
| `force_flush_returns_true_on_full` | INVARIANT | per-remove_folio_pages_size: full batch + !next_batch ⟹ return true. |
| `fullmm_promotion_on_nested` | INVARIANT | per-finish: mm_tlb_flush_nested ⟹ fullmm=1, freed_tables=1. |
| `flush_rmaps_clears_delay` | INVARIANT | per-flush_rmaps: returns ⟹ tlb.delayed_rmap == 0. |
| `batch_count_bounded` | INVARIANT | per-next_batch: batch_count ≤ MAX_GATHER_BATCH_COUNT. |
| `cond_resched_per_chunk` | INVARIANT | per-batch_free_encoded_pages: cond_resched between MAX_NR_FOLIOS_PER_FREE chunks. |

### Layer 2: TLA+

`mm/mmu-gather.tla`:
- Per-tlb_gather + N × remove_page + tlb_finish, in concurrent presence of gup_fast walker.
- Properties:
  - `safety_no_freed_page_table_observed_by_gup_fast` — per-CONFIG_RCU_TABLE_FREE: page-table page freed only after RCU grace period or IPI-sync; gup_fast cannot observe freed PT.
  - `safety_one_tlb_flush_per_finish` — per-finish: arch tlb_flush invoked exactly once.
  - `safety_delayed_rmap_commit_after_flush` — per-delayed_rmap: rmap removal sequenced after TLB flush.
  - `safety_fullmm_promotion_on_nested` — per-mm_tlb_flush_nested true: finish promotes fullmm + freed_tables.
  - `safety_table_batch_separate_from_page_batch` — per-table batch path orthogonal to page batch path.
  - `liveness_batch_eventually_freed` — per-finish: every gathered page eventually free_pages_and_swap_cache'd.
  - `liveness_table_batch_eventually_rcu_freed` — per-call_rcu callback eventually fires.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `MmuGather::gather_inner` post: tlb.active==&tlb.local ∧ inc_tlb_flush_pending called | `MmuGather::gather_inner` |
| `MmuGather::finish` post: dec_tlb_flush_pending called; tlb invariant cleared | `MmuGather::finish` |
| `MmuGather::flush` post: arch tlb_flush + free batches both executed | `MmuGather::flush` |
| `MmuGather::remove_folio_pages_size` post: ret==true ⟹ batch full ∧ next_batch failed | `MmuGather::remove_folio_pages_size` |
| `MmuGather::next_batch` post: ret==true ⟹ active is new/next batch; ret==false ⟹ caller must flush | `MmuGather::next_batch` |
| `MmuGather::batch_pages_flush` post: all batches with nr>0 freed; active==&local | `MmuGather::batch_pages_flush` |
| `MmuGather::batch_list_free` post: only `>= &local.next` freed; on-stack local intact | `MmuGather::batch_list_free` |
| `MmuGather::flush_rmaps` post: delayed_rmap==0; each delayed page rmap-removed exactly once | `MmuGather::flush_rmaps` |
| `MmuGather::remove_table` post: table appended to batch ∨ OOM-fallback (single + invalidate) | `MmuGather::remove_table` |
| `MmuGather::table_flush` post: batch RCU-freed (or direct); tlb.batch reset to NULL | `MmuGather::table_flush` |

### Layer 4: Verus/Creusot functional

`Per-tear-down (tlb_gather_mmu → unmap_page_range/zap_pte_range with tlb_remove_page → free_pgtables with tlb_remove_table → tlb_finish_mmu)` semantic equivalence with upstream `mmu_gather` contract documented in Documentation/core-api/cachetlb.rst.

`Per-CONFIG_MMU_GATHER_RCU_TABLE_FREE` path provides the gup_fast-vs-pt-free synchronization: PT pages are not freed until either (a) one RCU grace period after batch handover (sleepable RCU walkers) or (b) an IPI completes against IRQ-disabled walkers — see Documentation/mm/process_vm_access.rst and asm-generic/tlb.h.

### hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening.)

mmu_gather reinforcement:

- **Per-VM_BUG_ON tlb.end > 0** — defense against per-add-before-clear ordering bug.
- **Per-VM_WARN_ON page_size mismatch** — defense against per-mixed-size-batch corruption.
- **Per-MAX_GATHER_BATCH_COUNT cap** — defense against per-runaway-batch memory blowup.
- **Per-GFP_NOWAIT for batch + table allocations** — defense against per-mid-tear-down reclaim-deadlock.
- **Per-OOM single-page fallback in tlb_remove_table** — defense against per-PT-page-leak on alloc failure.
- **Per-call_rcu for page-table-page free** — defense against per-gup_fast UAF on freed PT.
- **Per-smp_call_function empty IPI for sync_one** — defense against per-IRQ-disabled-walker race.
- **Per-synchronize_rcu in non-RCU-table-free fallback** — defense against per-walker-mid-deref free.
- **Per-mm_tlb_flush_nested fullmm promotion** — defense against per-concurrent-flusher stale-TLB.
- **Per-tlb_flush_rmaps after TLB flush only** — defense against per-rmap-removed-before-TLB-cleared stale-mapping.
- **Per-cond_resched per MAX_NR_FOLIOS_PER_FREE chunk** — defense against per-long-free soft-lockup.
- **Per-fully_unshared_tables WARN_ON in finish** — defense against per-pmd-unshare-miss accounting bug.
- **Per-inc/dec_tlb_flush_pending pairing** — defense against per-pending-count leak that breaks mm_tlb_flush_nested.

