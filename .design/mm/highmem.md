# Tier-3: mm/highmem.c — High-memory (HIGHMEM) infrastructure

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: mm/00-overview.md
upstream-paths:
  - mm/highmem.c (~826 lines)
  - include/linux/highmem.h
  - include/linux/highmem-internal.h
  - include/asm-generic/kmap_size.h
  - include/linux/sched.h (struct kmap_ctrl)
-->

## Summary

On 32-bit kernels (i386 with PAE, ARM with LPAE, MIPS-32, ...) physical RAM exceeds the kernel's direct-map window (`PAGE_OFFSET..high_memory`, typically 896 MiB on i386). Pages above the direct-map cannot be dereferenced by a `struct page *` alone — they are flagged `__GFP_HIGHMEM` and require an on-demand temporary virtual mapping. `mm/highmem.c` provides three flavors of that mapping: (1) the **persistent kmap** (`kmap_high` / `kunmap_high`) — a fixed virtual window `PKMAP_ADDR(0..LAST_PKMAP)` of LAST_PKMAP refcounted slots, sleeps when full, used for long-lived mappings of arbitrary highmem pages; (2) the **atomic/local kmap** (`kmap_local_page` / `kunmap_local`) — per-CPU fixmap stack `FIX_KMAP_BEGIN..FIX_KMAP_END` indexed by `task->kmap_ctrl.idx`, never sleeps, scoped to a single context, saved/restored across `schedule()`; (3) the **page_address hash** — a `HASHED_PAGE_VIRTUAL` table mapping `struct page *` to its current pkmap virtual address (the "where is this highmem page mapped right now" oracle). On 64-bit kernels (x86_64, arm64, riscv64) the direct map already covers all RAM; `CONFIG_HIGHMEM=n`, `kmap_high` etc. compile out, and `kmap_local_page()` reduces to `page_address(page)`. Rookery is x86_64-first, but the `Highmem::*` surface is part of the kernel-wide page-mapping ABI and is required to plug HIGHMEM-aware drivers/filesystems unchanged. Critical for: cross-arch ABI completeness, kmap_local sleeping-allowed semantics, page_address oracle correctness, preempt/migrate-disable invariants during local kmap.

This Tier-3 covers `mm/highmem.c` (~826 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `pkmap_count[LAST_PKMAP]` | per-pkmap-slot refcount | `Highmem::pkmap_count` |
| `pkmap_page_table` | per-pkmap PTE array | arch-supplied |
| `kmap_lock` | per-pkmap spinlock | `Highmem::kmap_lock` |
| `pkmap_map_wait` (colored) | per-color waitqueue | `Highmem::pkmap_wait` |
| `__kmap_to_page()` | per-vaddr→page reverse | `Highmem::kmap_to_page` |
| `__nr_free_highpages()` | per-stat | `Highmem::nr_free_highpages` |
| `__totalhigh_pages()` | per-stat | `Highmem::totalhigh_pages` |
| `flush_all_zero_pkmaps()` | per-flush stale slots | `Highmem::flush_all_zero_pkmaps` |
| `__kmap_flush_unused()` | per-explicit-flush | `Highmem::flush_unused` |
| `map_new_virtual()` | per-alloc pkmap slot | `Highmem::map_new_virtual` |
| `kmap_high()` | per-page persistent kmap | `Highmem::kmap_high` |
| `kmap_high_get()` | per-page atomic-context pin | `Highmem::kmap_high_get` |
| `kunmap_high()` | per-page persistent kunmap | `Highmem::kunmap_high` |
| `zero_user_segments()` | per-page segment-zeroing | `Highmem::zero_user_segments` |
| `__kmap_local_pfn_prot()` | per-pfn local-kmap install | `Highmem::kmap_local_pfn_prot` |
| `__kmap_local_page_prot()` | per-page local-kmap install | `Highmem::kmap_local_page_prot` |
| `kunmap_local_indexed()` | per-vaddr local-kunmap | `Highmem::kunmap_local_indexed` |
| `__kmap_local_sched_out()` | per-task save on switch-out | `Highmem::sched_out` |
| `__kmap_local_sched_in()` | per-task restore on switch-in | `Highmem::sched_in` |
| `kmap_local_fork()` | per-fork clear-child-stack | `Highmem::fork_clear` |
| `struct page_address_map` | per-(page,vaddr) record | `PageAddressMap` |
| `page_address_maps[LAST_PKMAP]` | per-slot record-pool | `Highmem::page_address_pool` |
| `page_address_htable[]` | per-hash bucket | `Highmem::page_address_htable` |
| `page_slot()` | per-hash lookup | `Highmem::page_slot` |
| `page_address()` | per-page→vaddr oracle | `Highmem::page_address` |
| `set_page_address()` | per-page→vaddr install/remove | `Highmem::set_page_address` |
| `page_address_init()` | per-boot init | `Highmem::page_address_init` |
| `struct kmap_ctrl` (task_struct) | per-task local-kmap stack | embedded in `TaskStruct` |
| `KM_MAX_IDX` | per-arch slot ceiling | const |
| `LAST_PKMAP` | per-arch pkmap count | const |

## Compatibility contract

REQ-1: pkmap address window:
- Persistent kmap region: `PKMAP_BASE..PKMAP_BASE + LAST_PKMAP * PAGE_SIZE`.
- LAST_PKMAP arch-defined (i386-PAE: 512; i386-non-PAE: 1024).
- `PKMAP_ADDR(nr) = PKMAP_BASE + (nr << PAGE_SHIFT)`.
- `PKMAP_NR(vaddr) = (vaddr - PKMAP_BASE) >> PAGE_SHIFT`.

REQ-2: pkmap_count[LAST_PKMAP]:
- 0: slot free.
- 1: slot mapped but unreferenced; awaiting TLB flush.
- ≥2: slot mapped and referenced (count - 1 users).
- A count must never decrement to 0 without a TLB flush (`flush_all_zero_pkmaps` invariant).

REQ-3: kmap_high(page):
- /* May sleep — must not be called from interrupt context */
- spin_lock_irq(&kmap_lock).
- vaddr = page_address(page).
- if !vaddr: vaddr = map_new_virtual(page).
- pkmap_count[PKMAP_NR(vaddr)]++.
- BUG_ON(pkmap_count[nr] < 2).
- spin_unlock_irq.
- return vaddr.

REQ-4: map_new_virtual(page):
- color = get_pkmap_color(page).
- loop:
  - last_pkmap_nr = get_next_pkmap_nr(color).  /* per-color round-robin */
  - if no_more_pkmaps(last_pkmap_nr, color): flush_all_zero_pkmaps().
  - if pkmap_count[last_pkmap_nr] == 0: break (found slot).
  - if --count == 0:
    - add_wait_queue(pkmap_map_wait, &wait); TASK_UNINTERRUPTIBLE.
    - unlock_kmap(); schedule(); lock_kmap().
    - if page_address(page): return existing.  /* someone else mapped while we slept */
    - restart.
- vaddr = PKMAP_ADDR(last_pkmap_nr).
- set_pte_at(&init_mm, vaddr, &pkmap_page_table[last_pkmap_nr], mk_pte(page, kmap_prot)).
- pkmap_count[last_pkmap_nr] = 1.
- set_page_address(page, (void *)vaddr).

REQ-5: flush_all_zero_pkmaps():
- /* Reap slots with pkmap_count == 1 (mapped-but-unreferenced) */
- for nr in 0..LAST_PKMAP:
  - if pkmap_count[nr] != 1: continue.
  - pkmap_count[nr] = 0.
  - pte_clear(&init_mm, PKMAP_ADDR(nr), &pkmap_page_table[nr]).
  - set_page_address(pte_page(...), NULL).
- flush_tlb_kernel_range(PKMAP_ADDR(0), PKMAP_ADDR(LAST_PKMAP)).

REQ-6: kunmap_high(page):
- spin_lock_irqsave(&kmap_lock, flags).
- vaddr = page_address(page); BUG_ON(!vaddr).
- nr = PKMAP_NR(vaddr).
- switch (--pkmap_count[nr]):
  - case 0: BUG()  /* unbalanced */.
  - case 1: need_wakeup = waitqueue_active(get_pkmap_wait_queue_head(color)).
- spin_unlock_irqrestore.
- if need_wakeup: wake_up(pkmap_map_wait).

REQ-7: kmap_high_get(page):
- /* Only defined when ARCH_NEEDS_KMAP_HIGH_GET (e.g. ARM with cache-aliasing) */
- /* Callable from atomic context */
- spin_lock_irqsave(&kmap_lock, flags).
- vaddr = page_address(page).
- if vaddr: BUG_ON(pkmap_count[nr] < 1); pkmap_count[nr]++.
- spin_unlock_irqrestore.
- return vaddr (or NULL).

REQ-8: kmap_local stack (per-task `kmap_ctrl`):
- task.kmap_ctrl.idx: current depth (push/pop discipline).
- task.kmap_ctrl.pteval[KM_MAX_IDX]: saved PTEs for sched_out/in.
- KM_INCR: 1 normally, 2 with CONFIG_DEBUG_KMAP_LOCAL (guard slots).
- BUG_ON(idx >= KM_MAX_IDX) on push; BUG_ON(idx < 0) on pop.

REQ-9: __kmap_local_pfn_prot(pfn, prot):
- /* Never sleeps; nests on per-task stack */
- migrate_disable().  /* pin to CPU so fixmap entry is stable */
- preempt_disable().
- idx = arch_kmap_local_map_idx(kmap_local_idx_push(), pfn).
- vaddr = __fix_to_virt(FIX_KMAP_BEGIN + idx).
- kmap_pte = kmap_get_pte(vaddr, idx).
- BUG_ON(!pte_none(ptep_get(kmap_pte))).  /* slot must be free */
- pteval = pfn_pte(pfn, prot).
- arch_kmap_local_set_pte(&init_mm, vaddr, kmap_pte, pteval).
- arch_kmap_local_post_map(vaddr, pteval).
- current.kmap_ctrl.pteval[kmap_local_idx()] = pteval.
- preempt_enable().  /* migrate still disabled */
- return vaddr.

REQ-10: __kmap_local_page_prot(page, prot):
- if !CONFIG_DEBUG_KMAP_LOCAL_FORCE_MAP ∧ !PageHighMem(page): return page_address(page).  /* lowmem fast path */
- /* Try existing arch pkmap (ARM kmap_high_get) */
- kmap = arch_kmap_local_high_get(page); if kmap: return kmap.
- return __kmap_local_pfn_prot(page_to_pfn(page), prot).

REQ-11: kunmap_local_indexed(vaddr):
- /* vaddr must be the topmost outstanding kmap_local on this task */
- if vaddr below FIX_KMAP_END / above FIX_KMAP_BEGIN:
  - /* May be an ARM kmap_high mapping returned via arch_kmap_local_high_get */
  - if !kmap_high_unmap_local(addr): WARN if addr < PAGE_OFFSET.
  - return.
- preempt_disable().
- idx = arch_kmap_local_unmap_idx(kmap_local_idx(), addr).
- WARN_ON_ONCE(addr != __fix_to_virt(FIX_KMAP_BEGIN + idx)).  /* stack discipline */
- pte_clear(&init_mm, addr, kmap_get_pte(addr, idx)).
- arch_kmap_local_post_unmap(addr).
- current.kmap_ctrl.pteval[kmap_local_idx()] = pte_zero.
- kmap_local_idx_pop().
- preempt_enable().
- migrate_enable().

REQ-12: __kmap_local_sched_out / sched_in:
- /* Called from __schedule() to tear down + restore per-task fixmap entries */
- sched_out: for i in 0..idx: pte_clear corresponding fixmap entry; do NOT decrement idx.
- sched_in: for i in 0..idx: re-install pteval[i] into fixmap entry.
- DEBUG_KMAP_LOCAL: skip even slots (guard pages).

REQ-13: kmap_local_fork(child_task):
- child_task.kmap_ctrl.idx = 0.  /* a forked task inherits no local kmaps */

REQ-14: HASHED_PAGE_VIRTUAL page_address oracle:
- struct page_address_map { page, virtual, list }.
- page_address_maps[LAST_PKMAP]: pool, one per pkmap slot.
- page_address_htable[1 << PA_HASH_ORDER] (PA_HASH_ORDER=7): hash buckets.
- page_slot(p) = &page_address_htable[hash_ptr(p, PA_HASH_ORDER)].
- page_address(p):
  - if !PageHighMem(p): return lowmem_page_address(p)  /* direct-map */.
  - spin_lock_irqsave(&pas.lock).
  - walk pas.lh, return pam.virtual where pam.page == p, or NULL.
- set_page_address(p, vaddr):
  - BUG_ON(!PageHighMem(p)).
  - if vaddr: pam = &page_address_maps[PKMAP_NR(vaddr)]; pam.page = p; pam.virtual = vaddr; list_add_tail(&pam.list, &pas.lh).
  - else: list_del matching entry.

REQ-15: __kmap_to_page(vaddr) reverse lookup:
- if PKMAP_ADDR(0) ≤ vaddr < PKMAP_ADDR(LAST_PKMAP): return pte_page(pkmap_page_table[PKMAP_NR(vaddr)]).
- if FIX_KMAP_END ≤ vaddr ≤ FIX_KMAP_BEGIN: return pte_page(__kmap_pte[-(idx)]).
- else: return virt_to_page(vaddr).  /* direct-map */

REQ-16: zero_user_segments(page, s1, e1, s2, e2):
- /* Zero two byte-ranges of a (possibly compound) highmem page */
- BUG_ON(end > page_size(page)).
- for sub in 0..compound_nr(page):
  - kaddr = kmap_local_page(page + sub).
  - memset within [s1,e1) ∩ PAGE_SIZE.
  - memset within [s2,e2) ∩ PAGE_SIZE.
  - kunmap_local(kaddr).
  - flush_dcache_page(page + sub).

REQ-17: nr_free_highpages / totalhigh_pages stats:
- __nr_free_highpages: iterate zones; sum NR_FREE_PAGES for ZONE_HIGHMEM and ZONE_MOVABLE.
- __totalhigh_pages: per-boot total.

REQ-18: Per-arch x86_64 disposition:
- CONFIG_HIGHMEM=n, CONFIG_KMAP_LOCAL=n on x86_64.
- kmap_high / kunmap_high / pkmap_count / page_address_htable absent.
- kmap_local_page(p) inlined to page_address(p) (= lowmem_page_address(p) = __va(PFN_PHYS(page_to_pfn(p)))).
- The `Highmem::*` Rust surface compiles to a no-op trait impl on x86_64; full impl required on hypothetical 32-bit-arch port.

## Acceptance Criteria

- [ ] AC-1: x86_64 build: PageHighMem(p) statically false; kmap_local_page(p) = page_address(p) = __va(PFN_PHYS(...)).
- [ ] AC-2: 32-bit-arch build: kmap_high under pressure with all slots full sleeps until a kunmap_high wakes it.
- [ ] AC-3: kmap_high on already-mapped page returns existing vaddr, increments pkmap_count.
- [ ] AC-4: kunmap_high decrements; transition to count==1 wakes one waiter if any.
- [ ] AC-5: flush_all_zero_pkmaps clears count==1 slots and issues flush_tlb_kernel_range.
- [ ] AC-6: kmap_local_pfn_prot: pushes idx, installs PTE, returns FIX_KMAP-region vaddr; preempt_disable held across PTE install.
- [ ] AC-7: kunmap_local_indexed pops in reverse order; mismatched order WARNs.
- [ ] AC-8: __schedule() invokes sched_out (clear all PTEs) before context-switch; sched_in (restore) after.
- [ ] AC-9: fork: child task.kmap_ctrl.idx = 0.
- [ ] AC-10: page_address(p) for non-mapped highmem page returns NULL.
- [ ] AC-11: set_page_address(p, vaddr) installs; set_page_address(p, NULL) removes from hash.
- [ ] AC-12: kmap_to_page round-trip: page_address(p) → __kmap_to_page → p.
- [ ] AC-13: zero_user_segments zeros exactly the requested byte ranges across compound pages.
- [ ] AC-14: kmap_local nesting up to KM_MAX_IDX-1 succeeds; exceeding BUG()s.

## Architecture

```
struct PageAddressMap {
  page: *Page,
  virtual: *u8,
  list: ListHead,                       // hash bucket linkage
}

struct PageAddressSlot {
  lh: ListHead,
  lock: SpinLock,                       // bucket-local, IRQ-safe
}

struct KmapCtrl {                       // embedded in TaskStruct
  idx: i32,
  pteval: [Pte; KM_MAX_IDX],
}

struct Highmem {
  // Persistent (pkmap) state — present only when CONFIG_HIGHMEM
  pkmap_count: [u32; LAST_PKMAP],
  kmap_lock: SpinLock,                  // protects pkmap_count + page_address hash
  pkmap_color_wait: [WaitQueueHead; PKMAP_COLORS],
  last_pkmap_nr: AtomicU32,             // per-color round-robin cursor

  // page_address oracle (HASHED_PAGE_VIRTUAL)
  page_address_pool: [PageAddressMap; LAST_PKMAP],
  page_address_htable: [PageAddressSlot; 1 << PA_HASH_ORDER],

  // kmap_local fixmap pte cache
  __kmap_pte: *Pte,
}
```

`Highmem::kmap_high(page) -> *u8`:
1. spin_lock_irq(&self.kmap_lock).
2. vaddr = self.page_address(page).
3. if vaddr == NULL: vaddr = self.map_new_virtual(page).
4. nr = PKMAP_NR(vaddr).
5. self.pkmap_count[nr] += 1.
6. BUG_ON(self.pkmap_count[nr] < 2).
7. spin_unlock_irq.
8. return vaddr.

`Highmem::map_new_virtual(page) -> *u8`:
1. color = get_pkmap_color(page).
2. loop:
   - count = LAST_PKMAP / PKMAP_COLORS.
   - inner_loop:
     - nr = self.next_pkmap_nr(color).
     - if no_more_pkmaps(nr, color): self.flush_all_zero_pkmaps().
     - if self.pkmap_count[nr] == 0: break inner_loop with nr.
     - count -= 1.
     - if count == 0:
       - wait = WaitQueueEntry::new(current()).
       - add_wait_queue(&self.pkmap_color_wait[color], &wait).
       - set_current_state(TASK_UNINTERRUPTIBLE).
       - spin_unlock_irq(&self.kmap_lock).
       - schedule().
       - spin_lock_irq(&self.kmap_lock).
       - remove_wait_queue(&self.pkmap_color_wait[color], &wait).
       - if self.page_address(page) != NULL: return self.page_address(page).
       - restart outer loop.
3. vaddr = PKMAP_ADDR(nr).
4. set_pte_at(&init_mm, vaddr, &pkmap_page_table[nr], mk_pte(page, kmap_prot)).
5. self.pkmap_count[nr] = 1.
6. self.set_page_address(page, vaddr).
7. return vaddr.

`Highmem::flush_all_zero_pkmaps()`:
1. /* kmap_lock held by caller */
2. for nr in 0..LAST_PKMAP:
   - if self.pkmap_count[nr] != 1: continue.
   - self.pkmap_count[nr] = 0.
   - vaddr = PKMAP_ADDR(nr).
   - page = pte_page(ptep_get(&pkmap_page_table[nr])).
   - pte_clear(&init_mm, vaddr, &pkmap_page_table[nr]).
   - self.set_page_address(page, NULL).
3. flush_tlb_kernel_range(PKMAP_ADDR(0), PKMAP_ADDR(LAST_PKMAP)).

`Highmem::kunmap_high(page)`:
1. spin_lock_irqsave(&self.kmap_lock, flags).
2. vaddr = self.page_address(page); BUG_ON(vaddr == NULL).
3. nr = PKMAP_NR(vaddr).
4. self.pkmap_count[nr] -= 1.
5. match self.pkmap_count[nr]:
   - 0 => BUG()  /* unbalanced kmap/kunmap */,
   - 1 => need_wakeup = waitqueue_active(&self.pkmap_color_wait[get_pkmap_color(page)]),
   - _ => need_wakeup = false.
6. spin_unlock_irqrestore.
7. if need_wakeup: wake_up(&self.pkmap_color_wait[color]).

`Highmem::kmap_local_pfn_prot(pfn, prot) -> *u8`:
1. migrate_disable().
2. preempt_disable().
3. idx = arch_kmap_local_map_idx(kmap_local_idx_push(), pfn).
4. vaddr = __fix_to_virt(FIX_KMAP_BEGIN + idx).
5. kmap_pte = self.kmap_get_pte(vaddr, idx).
6. BUG_ON(!pte_none(ptep_get(kmap_pte)))  /* slot stack-discipline violated */.
7. pteval = pfn_pte(pfn, prot).
8. arch_kmap_local_set_pte(&init_mm, vaddr, kmap_pte, pteval).
9. arch_kmap_local_post_map(vaddr, pteval).
10. current.kmap_ctrl.pteval[kmap_local_idx()] = pteval.
11. preempt_enable()  /* migrate still disabled — vaddr stable on this CPU */.
12. return vaddr.

`Highmem::kmap_local_page_prot(page, prot) -> *u8`:
1. if !CONFIG_DEBUG_KMAP_LOCAL_FORCE_MAP ∧ !PageHighMem(page): return page_address(page).
2. /* Reuse any existing arch persistent kmap (e.g. ARM) to avoid re-faulting */
3. kmap = arch_kmap_local_high_get(page); if kmap: return kmap.
4. return self.kmap_local_pfn_prot(page_to_pfn(page), prot).

`Highmem::kunmap_local_indexed(vaddr)`:
1. addr = vaddr & PAGE_MASK.
2. if addr < FIX_KMAP_END || addr > FIX_KMAP_BEGIN:
   - if !self.kmap_high_unmap_local(addr): WARN_ON_ONCE(addr < PAGE_OFFSET).
   - return.
3. preempt_disable().
4. idx = arch_kmap_local_unmap_idx(kmap_local_idx(), addr).
5. WARN_ON_ONCE(addr != __fix_to_virt(FIX_KMAP_BEGIN + idx))  /* push/pop order */.
6. kmap_pte = self.kmap_get_pte(addr, idx).
7. arch_kmap_local_pre_unmap(addr).
8. pte_clear(&init_mm, addr, kmap_pte).
9. arch_kmap_local_post_unmap(addr).
10. current.kmap_ctrl.pteval[kmap_local_idx()] = Pte::zero().
11. kmap_local_idx_pop().
12. preempt_enable().
13. migrate_enable().

`Highmem::sched_out() / sched_in()`:
- sched_out: for i in 0..current.kmap_ctrl.idx:
  - if CONFIG_DEBUG_KMAP_LOCAL ∧ !(i & 1): skip-guard-slot.
  - pteval = current.kmap_ctrl.pteval[i].
  - WARN_ON_ONCE(pte_none(pteval)).
  - idx = arch_kmap_local_map_idx(i, pte_pfn(pteval)).
  - pte_clear(&init_mm, __fix_to_virt(FIX_KMAP_BEGIN + idx), kmap_pte).
- sched_in: symmetric, re-install pteval via arch_kmap_local_set_pte.

`Highmem::page_address(page) -> *u8`:
1. if !PageHighMem(page): return lowmem_page_address(page)  /* __va(PFN_PHYS(page_to_pfn(page))) */.
2. pas = &self.page_address_htable[hash_ptr(page, PA_HASH_ORDER)].
3. spin_lock_irqsave(&pas.lock, flags).
4. for pam in pas.lh.iter():
   - if pam.page == page: ret = pam.virtual; break.
5. spin_unlock_irqrestore.
6. return ret (or NULL).

`Highmem::set_page_address(page, vaddr)`:
1. BUG_ON(!PageHighMem(page)).
2. pas = &self.page_address_htable[hash_ptr(page, PA_HASH_ORDER)].
3. if vaddr != NULL:
   - pam = &self.page_address_pool[PKMAP_NR(vaddr)].
   - pam.page = page; pam.virtual = vaddr.
   - spin_lock_irqsave(&pas.lock, flags).
   - list_add_tail(&pam.list, &pas.lh).
   - spin_unlock_irqrestore.
4. else:
   - spin_lock_irqsave(&pas.lock, flags).
   - for pam in pas.lh.iter():
     - if pam.page == page: list_del(&pam.list); break.
   - spin_unlock_irqrestore.

`Highmem::kmap_to_page(vaddr) -> *Page`:
1. if PKMAP_ADDR(0) ≤ vaddr < PKMAP_ADDR(LAST_PKMAP):
   - return pte_page(ptep_get(&pkmap_page_table[PKMAP_NR(vaddr)])).
2. if FIX_KMAP_END ≤ (vaddr & PAGE_MASK) ≤ FIX_KMAP_BEGIN:
   - idx = (FIX_KMAP_BEGIN - ((vaddr & PAGE_MASK) >> PAGE_SHIFT)).
   - return pte_page(ptep_get(self.kmap_get_pte(vaddr & PAGE_MASK, idx))).
3. return virt_to_page(vaddr).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pkmap_count_never_zero_without_flush` | INVARIANT | per-kunmap_high: count→1 never→0 outside flush_all_zero_pkmaps. |
| `kmap_high_balanced` | INVARIANT | per-(kmap_high, kunmap_high) pair: net pkmap_count delta = 0. |
| `map_new_virtual_holds_kmap_lock` | INVARIANT | per-map_new_virtual: kmap_lock held except across schedule(). |
| `kmap_local_idx_bounded` | INVARIANT | per-push/pop: 0 ≤ idx < KM_MAX_IDX. |
| `kmap_local_preempt_held_at_pte_install` | INVARIANT | per-kmap_local_pfn_prot: preempt_disable held during set_pte_at. |
| `kmap_local_migrate_held_across_user` | INVARIANT | per-kmap_local lifetime: migrate_disable held vaddr-issue → kunmap. |
| `kmap_local_stack_discipline` | INVARIANT | per-kunmap_local_indexed: vaddr matches top-of-stack idx. |
| `page_address_lock_pairs` | INVARIANT | per-bucket: every lock IRQ-saved/restored. |
| `set_page_address_only_highmem` | INVARIANT | per-set_page_address: BUG_ON(!PageHighMem). |
| `pkmap_count_overflow_free` | INVARIANT | per-kmap_high: pkmap_count[nr] ≤ U32::MAX always. |

### Layer 2: TLA+

`mm/highmem.tla`:
- Per-pkmap-slot lifecycle + per-kmap_local-stack + per-page_address-hash + per-sched_out/in.
- Properties:
  - `safety_pkmap_count_monotone_per_op` — per-kmap_high: count strictly +1; per-kunmap_high: strictly -1.
  - `safety_no_zero_without_flush` — per-state: ∀ slot s: s.count=0 ⟹ s was last touched by flush_all_zero_pkmaps with TLB flush.
  - `safety_pkmap_waitqueue_progress` — per-waiter: ∃ kunmap_high that drops count to 1 ⟹ waiter eventually scheduled.
  - `safety_kmap_local_stack_lifo` — per-task: kunmap_local sequence reverses kmap_local sequence (LIFO).
  - `safety_kmap_local_sched_out_in_idempotent` — per-task switch_out→switch_in: idx unchanged, pteval[] unchanged.
  - `safety_page_address_consistent` — per-page p with PKMAP slot nr: page_address(p) == PKMAP_ADDR(nr).
  - `liveness_kmap_high_eventually_succeeds` — per-kmap_high under bounded outstanding: eventually returns vaddr.
  - `liveness_kunmap_high_wakes_one` — per-count→1 transition: ≥1 waiter (if any) made runnable.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Highmem::kmap_high` post: pkmap_count[nr] ≥ 2 ∧ page_address(page) == vaddr | `Highmem::kmap_high` |
| `Highmem::kunmap_high` post: pkmap_count[nr] decreased by 1 ∧ ≥ 1 | `Highmem::kunmap_high` |
| `Highmem::map_new_virtual` post: pkmap_count[nr] == 1 ∧ pte set | `Highmem::map_new_virtual` |
| `Highmem::flush_all_zero_pkmaps` post: ∀ slots with input.count==1: count==0 ∧ pte cleared ∧ TLB flushed | `Highmem::flush_all_zero_pkmaps` |
| `Highmem::kmap_local_pfn_prot` post: idx incremented ∧ pte installed ∧ vaddr ∈ FIX_KMAP | `Highmem::kmap_local_pfn_prot` |
| `Highmem::kunmap_local_indexed` post: idx decremented ∧ pte cleared | `Highmem::kunmap_local_indexed` |
| `Highmem::page_address` post: returns NULL for highmem-not-mapped ∨ matches set_page_address record | `Highmem::page_address` |
| `Highmem::set_page_address` post: hash bucket reflects new mapping (or removal) | `Highmem::set_page_address` |
| `Highmem::kmap_to_page` post: round-trip with page_address | `Highmem::kmap_to_page` |

### Layer 4: Verus/Creusot functional

`Per-arch: x86_64 surface no-ops (PageHighMem ≡ false, kmap_local_page ≡ page_address). Per-32-bit arch: kmap_high (acquire pkmap slot, refcount, possibly sleep) + kunmap_high (release, wake) + kmap_local push/pop + page_address hash oracle, semantic equivalence per Documentation/mm/highmem.rst and include/linux/highmem.h.`

## Hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening.)

Highmem reinforcement:

- **Per-kmap_lock IRQ-safe** — defense against per-IRQ-context kmap-vs-kunmap deadlock.
- **Per-pkmap_count never-zero-without-TLB-flush invariant** — defense against per-stale-PTE access after slot reuse.
- **Per-kunmap_high BUG on unbalanced** — defense against per-double-kunmap memory-corruption.
- **Per-kmap_local KM_MAX_IDX BUG** — defense against per-stack-overflow into wrong fixmap entry.
- **Per-kmap_local stack-discipline WARN_ON_ONCE** — defense against per-out-of-order kunmap.
- **Per-kmap_local migrate_disable across mapping lifetime** — defense against per-CPU-migrate-into-stale-fixmap.
- **Per-kmap_local preempt_disable at PTE install** — defense against per-preempt-into-partial-install.
- **Per-DEBUG_KMAP_LOCAL guard slots (KM_INCR=2)** — defense against per-overflow-into-neighbor-slot.
- **Per-kmap_high BUG_ON(pkmap_count<2) after increment** — defense against per-refcount-corruption.
- **Per-set_page_address BUG_ON(!PageHighMem)** — defense against per-lowmem-page hash pollution.
- **Per-page_address bucket lock IRQ-saved** — defense against per-IRQ hash mutation race.
- **Per-flush_tlb_kernel_range covering full PKMAP window** — defense against per-partial-flush stale TLB.
- **Per-arch x86_64 compile-out** — defense against per-dead-code attack-surface; HIGHMEM code only present on 32-bit arch builds.
- **Per-fork kmap_ctrl.idx = 0 zero** — defense against per-child-inherits-parent-fixmap escape.
- **Per-WARN_ON_ONCE on hardirq + irqs_enabled at kmap_local push** — defense against per-IRQ-context-kmap_local misuse.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- include/linux/highmem.h inline wrappers (kmap_atomic backward-compat, kmap_to_page consumers — Tier-2 if exposed as ABI)
- arch/x86/include/asm/fixmap.h (fixmap allocation — arch-specific)
- arch/x86/include/asm/highmem.h (i386 PKMAP_BASE definition — 32-bit only)
- mm/page_alloc.c ZONE_HIGHMEM zone selection (covered in `page-allocator.md` Tier-3)
- mm/memory.c page-cache flushing (covered in `page-cache.md` Tier-3)
- Implementation code
