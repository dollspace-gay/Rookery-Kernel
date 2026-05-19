# Tier-3: arch/x86/mm/init.c — x86 memory init (direct map, brk pgt, initmem free)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: arch/x86/00-overview.md
upstream-paths:
  - arch/x86/mm/init.c (~1128 lines)
  - arch/x86/mm/init_64.c (64-bit kernel_physical_mapping_init)
  - arch/x86/mm/init_32.c (32-bit kernel_physical_mapping_init)
  - arch/x86/mm/ident_map.c (kernel_ident_mapping_init)
  - arch/x86/mm/mm_internal.h
  - arch/x86/kernel/setup.c (setup_arch caller)
  - arch/x86/kernel/e820.c (e820__memblock_setup)
  - include/linux/memblock.h
-->

## Summary

`arch/x86/mm/init.c` is the architecture-neutral half of x86's early memory bring-up — the code that runs after `start_kernel` reaches `setup_arch()` and before the page allocator becomes available. It owns: the **early page-table allocator** (`alloc_low_pages` backed by a BRK-reserved buffer plus memblock fallback), the **direct mapping** of all physical RAM into the kernel half of the virtual address space (`init_mem_mapping` → `init_memory_mapping` → `kernel_physical_mapping_init`), the per-range **map_range splitting** that picks 4K/2M/1G page sizes (`split_mem_range`, `adjust_range_page_size_mask`, `save_mr`), the **PAT / PSE / PGE / PCID / GBPAGES** probe (`probe_page_size_mask`, `setup_pcid`), the **real-mode trampoline alias** in the page table (`init_trampoline`), the **`pfn_mapped[]` registry** of mapped PFN ranges with `max_pfn_mapped` / `max_low_pfn_mapped`, the **`__init` section unmap and free** (`free_init_pages`, `free_kernel_image_pages`, `free_initmem`), the **`__initrd` free** (`free_initrd_mem`), the per-zone PFN limit calculation (`arch_zone_limits_init`), the **text-poking mm** init (`poking_init`), and `devmem_is_allowed` for `/dev/mem`. Critical for: every kernel virtual address being backed before the page allocator boots, the real-mode trampoline being addressable for AP bringup and S3 wakeup, and reclaiming `__init` text+data after boot.

This Tier-3 covers `arch/x86/mm/init.c` (~1128 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `alloc_low_pages()` | per-early-pgt-page allocator (BRK / memblock / buddy) | `X86Mm::alloc_low_pages` |
| `early_alloc_pgt_buf()` | per-boot BRK page-table buffer carve-out | `X86Mm::early_alloc_pgt_buf` |
| `probe_page_size_mask()` | per-boot PSE/PGE/GBPAGES/PTI probe | `X86Mm::probe_page_size_mask` |
| `setup_pcid()` | per-boot PCID enable + INVLPG-miss errata | `X86Mm::setup_pcid` |
| `cr4_set_bits_and_update_boot()` | per-boot CR4 shadow + trampoline-CR4 update | `X86Mm::cr4_set_bits_and_update_boot` |
| `save_mr()` | per-`map_range` slot append | `X86Mm::save_mr` |
| `adjust_range_page_size_mask()` | per-range 2M/1G upgrade if surroundings are RAM | `X86Mm::adjust_range_page_size_mask` |
| `split_mem_range()` | per-init_memory_mapping range split into 4K/2M/1G | `X86Mm::split_mem_range` |
| `page_size_string()` | per-range "4K"/"2M"/"1G" log helper | `X86Mm::page_size_string` |
| `add_pfn_range_mapped()` | per-mapped-range registry update | `X86Mm::add_pfn_range_mapped` |
| `pfn_range_is_mapped()` | per-PFN-range membership query | `X86Mm::pfn_range_is_mapped` |
| `init_memory_mapping()` | per-(start,end) direct-map call site | `X86Mm::init_memory_mapping` |
| `init_range_memory_mapping()` | per-init-range iteration over for_each_mem_pfn_range | `X86Mm::init_range_memory_mapping` |
| `get_new_step_size()` | per-iteration step-size escalator | `X86Mm::get_new_step_size` |
| `memory_map_top_down()` | per-top-down direct-map fill | `X86Mm::memory_map_top_down` |
| `memory_map_bottom_up()` | per-bottom-up direct-map fill | `X86Mm::memory_map_bottom_up` |
| `init_trampoline()` | per-boot real-mode trampoline alias (KASLR-aware) | `X86Mm::init_trampoline` |
| `init_mem_mapping()` | per-boot top-level orchestration | `X86Mm::init_mem_mapping` |
| `poking_init()` | per-boot text-poking mm setup | `X86Mm::poking_init` |
| `devmem_is_allowed()` | per-PFN `/dev/mem` access policy | `X86Mm::devmem_is_allowed` |
| `free_init_pages()` | per-range init-section free + set_memory_{nx,rw} | `X86Mm::free_init_pages` |
| `free_kernel_image_pages()` | per-range kernel-image free + PTI alias unmap | `X86Mm::free_kernel_image_pages` |
| `free_initmem()` | per-boot end-of-`__init` reclaim | `X86Mm::free_initmem` |
| `free_initrd_mem()` | per-boot initrd reclaim | `X86Mm::free_initrd_mem` |
| `arch_zone_limits_init()` | per-boot zone-PFN limit setup | `X86Mm::arch_zone_limits_init` |
| `update_cache_mode_entry()` | per-PAT-entry runtime update | `X86Mm::update_cache_mode_entry` |
| `arch_max_swapfile_size()` | per-arch swap-file size limit (L1TF mitigation) | `X86Mm::arch_max_swapfile_size` |
| `execmem_arch_setup()` | per-arch executable-memory range setup | `X86Mm::execmem_arch_setup` |
| `cachemode2protval()` / `pgprot2cachemode()` / `x86_has_pat_wp()` | per-PAT cache-mode <-> pgprot conversion | shared |
| `pfn_mapped[]`, `nr_pfn_mapped` | per-boot mapped-PFN registry | `X86Mm::pfn_mapped` |
| `max_pfn_mapped`, `max_low_pfn_mapped` | per-boot mapped-PFN watermarks | shared |
| `cpu_tlbstate` (per-CPU) | per-CPU TLB state shadow | shared (defined here, used by `tlb.c`) |

## Compatibility contract

REQ-1: alloc_low_pages(num) — early page-table page allocator:
- if after_bootmem: return `__get_free_pages(GFP_ATOMIC | __GFP_ZERO, get_order(num << PAGE_SHIFT))`.
- if (pgt_buf_end + num) > pgt_buf_top ∨ !can_use_brk_pgt:
  - if min_pfn_mapped < max_pfn_mapped: ret = `memblock_phys_alloc_range(PAGE_SIZE*num, PAGE_SIZE, min_pfn_mapped<<PAGE_SHIFT, max_pfn_mapped<<PAGE_SHIFT)`.
  - if !ret ∧ can_use_brk_pgt: ret = `__pa(extend_brk(PAGE_SIZE*num, PAGE_SIZE))`.
  - if !ret: panic("alloc_low_pages: can not alloc memory").
  - pfn = ret >> PAGE_SHIFT.
- else: pfn = pgt_buf_end; pgt_buf_end += num.
- for i in 0..num: clear_page(__va((pfn+i) << PAGE_SHIFT)).
- return __va(pfn << PAGE_SHIFT).

REQ-2: early_alloc_pgt_buf() — BRK-reserved page-table buffer:
- INIT_PGD_PAGE_TABLES = 4.
- INIT_PGD_PAGE_COUNT = 2*INIT_PGD_PAGE_TABLES (or 4* with CONFIG_RANDOMIZE_MEMORY).
- INIT_PGT_BUF_SIZE = INIT_PGD_PAGE_COUNT * PAGE_SIZE.
- RESERVE_BRK(early_pgt_alloc, INIT_PGT_BUF_SIZE).
- base = __pa(extend_brk(INIT_PGT_BUF_SIZE, PAGE_SIZE)).
- pgt_buf_start = base >> PAGE_SHIFT; pgt_buf_end = pgt_buf_start; pgt_buf_top = pgt_buf_start + (INIT_PGT_BUF_SIZE >> PAGE_SHIFT).

REQ-3: probe_page_size_mask() — CR4 + page_size_mask:
- if X86_FEATURE_PSE ∧ !debug_pagealloc_enabled: page_size_mask |= 1 << PG_LEVEL_2M else direct_gbpages = 0.
- if X86_FEATURE_PSE: cr4_set_bits_and_update_boot(X86_CR4_PSE).
- __supported_pte_mask &= ~_PAGE_GLOBAL.
- if X86_FEATURE_PGE: cr4_set_bits_and_update_boot(X86_CR4_PGE); __supported_pte_mask |= _PAGE_GLOBAL.
- __default_kernel_pte_mask = __supported_pte_mask.
- if X86_FEATURE_PTI: __default_kernel_pte_mask &= ~_PAGE_GLOBAL.
- if direct_gbpages ∧ X86_FEATURE_GBPAGES: page_size_mask |= 1 << PG_LEVEL_1G else direct_gbpages = 0.

REQ-4: setup_pcid() — PCID enable + INVLPG errata:
- if !CONFIG_X86_64: return.
- if !X86_FEATURE_PCID: return.
- if x86_match_cpu(invlpg_miss_ids) ∧ microcode < required: setup_clear_cpu_cap(X86_FEATURE_PCID); return.
- if X86_FEATURE_PGE: cr4_set_bits(X86_CR4_PCIDE) (not via `cr4_set_bits_and_update_boot` — trampoline cannot handle PCIDE; secondary CPUs set it manually in `start_secondary`).
- else: setup_clear_cpu_cap(X86_FEATURE_PCID).

REQ-5: split_mem_range(mr, nr_range, start, end):
- Splits [start, end) into up to NR_RANGE_MR sub-ranges (3 on 32-bit, 5 on 64-bit) with appropriate `page_size_mask` per sub-range based on alignment to PMD_SIZE / PUD_SIZE and the global `page_size_mask`.
- Calls `save_mr` for each sub-range.
- Calls `adjust_range_page_size_mask` to upgrade nearby-RAM ranges.
- Coalesces adjacent ranges with equal `page_size_mask`.

REQ-6: save_mr(mr, nr_range, start_pfn, end_pfn, page_size_mask):
- if start_pfn < end_pfn:
  - if nr_range >= NR_RANGE_MR: panic("run out of range for init_memory_mapping").
  - mr[nr_range] = { start_pfn<<PAGE_SHIFT, end_pfn<<PAGE_SHIFT, page_size_mask }.
  - nr_range++.
- return nr_range.

REQ-7: adjust_range_page_size_mask(mr, nr_range):
- For each range, if global page_size_mask has 2M but range lacks it: if `memblock_is_region_memory(round_down(start, PMD_SIZE), round_up(end, PMD_SIZE))`: set the 2M bit.
- Same for 1G with PUD_SIZE.
- On 32-bit, never upgrade ranges that cross max_low_pfn.

REQ-8: init_memory_mapping(start, end, prot) — directly map [start, end):
- nr_range = split_mem_range(mr, 0, start, end).
- for i in 0..nr_range: ret = kernel_physical_mapping_init(mr[i].start, mr[i].end, mr[i].page_size_mask, prot).
- add_pfn_range_mapped(start >> PAGE_SHIFT, ret >> PAGE_SHIFT).
- return ret >> PAGE_SHIFT.

REQ-9: add_pfn_range_mapped(start_pfn, end_pfn) — registry update:
- nr_pfn_mapped = add_range_with_merge(pfn_mapped, E820_MAX_ENTRIES, nr_pfn_mapped, start_pfn, end_pfn).
- nr_pfn_mapped = clean_sort_range(pfn_mapped, E820_MAX_ENTRIES).
- max_pfn_mapped = max(max_pfn_mapped, end_pfn).
- if start_pfn < (1UL << (32 - PAGE_SHIFT)): max_low_pfn_mapped = max(max_low_pfn_mapped, min(end_pfn, 1UL << (32 - PAGE_SHIFT))).

REQ-10: pfn_range_is_mapped(start_pfn, end_pfn):
- for each pfn_mapped[i]: if start_pfn >= pfn_mapped[i].start ∧ end_pfn <= pfn_mapped[i].end: return true.
- return false.

REQ-11: init_range_memory_mapping(r_start, r_end) — drive direct-map across memblock RAM:
- mapped_ram_size = 0.
- for_each_mem_pfn_range(i, MAX_NUMNODES, &start_pfn, &end_pfn, NULL):
  - start = clamp(PFN_PHYS(start_pfn), r_start, r_end).
  - end = clamp(PFN_PHYS(end_pfn), r_start, r_end).
  - if start >= end: continue.
  - /* Switch off brk-pgt if overlap with brk */
  - can_use_brk_pgt = max(start, pgt_buf_end<<PAGE_SHIFT) >= min(end, pgt_buf_top<<PAGE_SHIFT).
  - init_memory_mapping(start, end, PAGE_KERNEL).
  - mapped_ram_size += end - start.
  - can_use_brk_pgt = true.
- return mapped_ram_size.

REQ-12: memory_map_top_down(map_start, map_end) — direct-map fill, top-down:
- addr = memblock_phys_alloc_range(PMD_SIZE, PMD_SIZE, map_start, map_end).
- real_end = addr ? addr + PMD_SIZE : max(map_start, ALIGN_DOWN(map_end, PMD_SIZE)).
- step_size = PMD_SIZE; max_pfn_mapped = 0; min_pfn_mapped = real_end >> PAGE_SHIFT; last_start = real_end.
- while last_start > map_start:
  - start = last_start > step_size ? round_down(last_start - 1, step_size) max with map_start : map_start.
  - mapped_ram_size += init_range_memory_mapping(start, last_start).
  - last_start = start.
  - min_pfn_mapped = last_start >> PAGE_SHIFT.
  - if mapped_ram_size >= step_size: step_size = get_new_step_size(step_size) (step_size << (PMD_SHIFT - PAGE_SHIFT - 1)).
- if real_end < map_end: init_range_memory_mapping(real_end, map_end).

REQ-13: memory_map_bottom_up(map_start, map_end) — direct-map fill, bottom-up:
- start = map_start; min_pfn_mapped = start >> PAGE_SHIFT; step_size = PMD_SIZE; mapped_ram_size = 0.
- while start < map_end:
  - next = step_size ∧ (map_end - start > step_size) ? round_up(start + 1, step_size) min map_end : map_end.
  - mapped_ram_size += init_range_memory_mapping(start, next).
  - start = next.
  - if mapped_ram_size >= step_size: step_size = get_new_step_size(step_size).

REQ-14: init_trampoline() — real-mode trampoline pgd alias:
- if !CONFIG_X86_64: nop.
- if !kaslr_memory_enabled: trampoline_pgd_entry = init_top_pgt[pgd_index(__PAGE_OFFSET)] (full PGD copy).
- else: init_trampoline_kaslr (copy only the PUD covering low 1MB; randomization granularity = 1GB).

REQ-15: init_mem_mapping() — top-level orchestration:
- pti_check_boottime_disable.
- probe_page_size_mask.
- setup_pcid.
- end = CONFIG_X86_64 ? max_pfn << PAGE_SHIFT : max_low_pfn << PAGE_SHIFT.
- init_memory_mapping(0, ISA_END_ADDRESS, PAGE_KERNEL) — ISA range always mapped regardless of memory holes.
- init_trampoline.
- if memblock_bottom_up:
  - kernel_end = __pa_symbol(_end).
  - memory_map_bottom_up(kernel_end, end).
  - memory_map_bottom_up(ISA_END_ADDRESS, kernel_end).
- else: memory_map_top_down(ISA_END_ADDRESS, end).
- if CONFIG_X86_64 ∧ max_pfn > max_low_pfn: max_low_pfn = max_pfn.
- else (32-bit): early_ioremap_page_table_range_init.
- load_cr3(swapper_pg_dir); __flush_tlb_all.
- x86_init.hyper.init_mem_mapping (Xen/Hyper-V hook).
- early_memtest(0, max_pfn_mapped << PAGE_SHIFT).

REQ-16: poking_init() — text-poking mm:
- text_poke_mm = mm_alloc; BUG_ON(!text_poke_mm).
- paravirt_enter_mmap(text_poke_mm) (Xen-PV needs PGD pinned).
- set_notrack_mm(text_poke_mm).
- text_poke_mm_addr = TASK_UNMAPPED_BASE; if CONFIG_RANDOMIZE_BASE: randomize within PMD; allocate ptl, ptep.

REQ-17: free_init_pages(what, begin, end) — free `__init` range:
- Page-align begin/end; if mismatch: WARN.
- if begin >= end: return.
- if debug_pagealloc_enabled: kmemleak_free_part; set_memory_np (unmap, no free).
- else: set_memory_nx; set_memory_rw; free_reserved_area(begin, end, POISON_FREE_INITMEM, what).

REQ-18: free_kernel_image_pages(what, begin, end):
- free_init_pages(what, begin, end).
- if CONFIG_X86_64 ∧ X86_FEATURE_PTI: set_memory_np_noalias(begin, len_pages) — clear PTI high-kernel-image alias to defend Meltdown.

REQ-19: free_initmem():
- e820__reallocate_tables.
- mem_encrypt_free_decrypted_mem.
- free_kernel_image_pages("unused kernel image (initmem)", &__init_begin, &__init_end).

REQ-20: free_initrd_mem(start, end) (under CONFIG_BLK_DEV_INITRD):
- free_init_pages("initrd", start, PAGE_ALIGN(end)).

REQ-21: arch_zone_limits_init(max_zone_pfns[]):
- if CONFIG_ZONE_DMA: max_zone_pfns[ZONE_DMA] = min(MAX_DMA_PFN, max_low_pfn).
- if CONFIG_ZONE_DMA32: max_zone_pfns[ZONE_DMA32] = min(MAX_DMA32_PFN, max_low_pfn).
- max_zone_pfns[ZONE_NORMAL] = max_low_pfn.
- if CONFIG_HIGHMEM: max_zone_pfns[ZONE_HIGHMEM] = max_pfn.

REQ-22: devmem_is_allowed(pagenr) — /dev/mem access policy:
- if region_intersects(PFN_PHYS(pagenr), PAGE_SIZE, IORESOURCE_SYSTEM_RAM, IORES_DESC_NONE) != REGION_DISJOINT:
  - if pagenr < 256: return 2 (allow but show as zero-filled — BIOS low 1MB shadow).
  - return 0.
- if iomem_is_exclusive(pagenr << PAGE_SHIFT):
  - if pagenr < 256: return 1 (low 1MB bypasses iomem restriction).
  - return 0.
- return 1.

REQ-23: arch_max_swapfile_size() — L1TF swap-file limit:
- pages = generic_max_swapfile_size.
- if X86_BUG_L1TF ∧ l1tf_mitigation != OFF:
  - l1tf_limit = l1tf_pfn_limit.
  - if PGTABLE_LEVELS > 2: l1tf_limit <<= PAGE_SHIFT - SWP_OFFSET_FIRST_BIT.
  - pages = min(l1tf_limit, pages).
- return pages.

REQ-24: PAT-conversion accessors (cachemode2protval, pgprot2cachemode, x86_has_pat_wp):
- __cachemode2pte_tbl[] and __pte2cachemode_tbl[] hold the runtime-updatable PAT cache-mode mapping.
- pat_init updates entries via update_cache_mode_entry.
- entry 0 must always be _PAGE_CACHE_MODE_WB (hardwired translation invariant).

## Acceptance Criteria

- [ ] AC-1: After `init_mem_mapping()`, every byte of every E820_TYPE_RAM range is directly mapped at PAGE_OFFSET + phys (`pfn_range_is_mapped` returns true).
- [ ] AC-2: `pfn_mapped[]` is sorted and merged (no duplicate or overlapping ranges); `nr_pfn_mapped <= E820_MAX_ENTRIES`.
- [ ] AC-3: `max_pfn_mapped` is monotonically non-decreasing across all `add_pfn_range_mapped` calls.
- [ ] AC-4: `alloc_low_pages(num)` returns a virtually-mapped buffer with `num` contiguous physical pages, zero-filled.
- [ ] AC-5: BRK page-table buffer exhausted → fallback to memblock_phys_alloc_range within [min_pfn_mapped, max_pfn_mapped) succeeds.
- [ ] AC-6: ISA range (0..ISA_END_ADDRESS) is mapped even if E820 marks it as a hole.
- [ ] AC-7: Real-mode trampoline pgd alias is installed (`trampoline_pgd_entry` set, or kaslr trampoline initialized).
- [ ] AC-8: After CR3 reload + `__flush_tlb_all`, kernel runs on `swapper_pg_dir`.
- [ ] AC-9: `free_initmem()` reclaims `[__init_begin, __init_end)` and (under PTI) removes the high-kernel-image alias.
- [ ] AC-10: `free_init_pages` with `debug_pagealloc_enabled` unmaps via `set_memory_np` (no free).
- [ ] AC-11: `devmem_is_allowed`: pagenr ∈ system RAM → 0 (denied) or 2 (zero-filled for low 1MB).
- [ ] AC-12: `arch_zone_limits_init` produces `max_zone_pfns[ZONE_NORMAL] == max_low_pfn` and (CONFIG_HIGHMEM) `[ZONE_HIGHMEM] == max_pfn`.
- [ ] AC-13: With X86_FEATURE_GBPAGES + `gbpages` cmdline + tdp-not-debug: 1G pages used for direct map.
- [ ] AC-14: X86_FEATURE_PCID without PGE → PCID disabled by `setup_pcid` (combination unsupported).
- [ ] AC-15: `x86_match_cpu(invlpg_miss_ids)` with unfixed microcode → `setup_clear_cpu_cap(X86_FEATURE_PCID)`.

## Architecture

```
struct MapRange {
  start:           u64,             // phys start
  end:             u64,             // phys end (exclusive)
  page_size_mask:  u32,             // bitmask: (1 << PG_LEVEL_2M) | (1 << PG_LEVEL_1G)
}

struct PfnRange {
  start:           u64,             // start PFN
  end:             u64,             // end PFN (exclusive)
}

static PFN_MAPPED:           [PfnRange; E820_MAX_ENTRIES];
static mut NR_PFN_MAPPED:    i32;
static mut MAX_PFN_MAPPED:   u64;
static mut MAX_LOW_PFN_MAPPED: u64;

// Early page-table allocator state
static mut PGT_BUF_START:    u64;   // BRK page-table buffer (start PFN)
static mut PGT_BUF_END:      u64;
static mut PGT_BUF_TOP:      u64;
static mut MIN_PFN_MAPPED:   u64;
static mut CAN_USE_BRK_PGT:  bool;
static mut AFTER_BOOTMEM:    i32;   // set to 1 once page allocator is up

const NR_RANGE_MR:           usize = if cfg!(CONFIG_X86_64) { 5 } else { 3 };
const INIT_PGD_PAGE_TABLES:  usize = 4;
const INIT_PGD_PAGE_COUNT:   usize = if cfg!(CONFIG_RANDOMIZE_MEMORY) { 4 * INIT_PGD_PAGE_TABLES } else { 2 * INIT_PGD_PAGE_TABLES };
const INIT_PGT_BUF_SIZE:     usize = INIT_PGD_PAGE_COUNT * PAGE_SIZE;
```

`X86Mm::init_mem_mapping()`:
1. pti_check_boottime_disable.
2. `X86Mm::probe_page_size_mask()`.
3. `X86Mm::setup_pcid()`.
4. end = if cfg!(CONFIG_X86_64) { max_pfn << PAGE_SHIFT } else { max_low_pfn << PAGE_SHIFT }.
5. `X86Mm::init_memory_mapping(0, ISA_END_ADDRESS, PAGE_KERNEL)`.
6. `X86Mm::init_trampoline()`.
7. if memblock_bottom_up():
   - kernel_end = __pa_symbol(_end).
   - `X86Mm::memory_map_bottom_up(kernel_end, end)`.
   - `X86Mm::memory_map_bottom_up(ISA_END_ADDRESS, kernel_end)`.
   - else: `X86Mm::memory_map_top_down(ISA_END_ADDRESS, end)`.
8. if cfg!(CONFIG_X86_64) ∧ max_pfn > max_low_pfn: max_low_pfn = max_pfn.
9. else: early_ioremap_page_table_range_init().
10. load_cr3(swapper_pg_dir); __flush_tlb_all().
11. x86_init.hyper.init_mem_mapping().
12. early_memtest(0, max_pfn_mapped << PAGE_SHIFT).

`X86Mm::alloc_low_pages(num) -> *mut u8`:
1. if AFTER_BOOTMEM:
   - order = get_order(num * PAGE_SIZE).
   - return `__get_free_pages(GFP_ATOMIC | __GFP_ZERO, order)`.
2. if (PGT_BUF_END + num) > PGT_BUF_TOP ∨ !CAN_USE_BRK_PGT:
   - ret = 0.
   - if MIN_PFN_MAPPED < MAX_PFN_MAPPED:
     - ret = `memblock_phys_alloc_range(PAGE_SIZE*num, PAGE_SIZE, MIN_PFN_MAPPED<<PAGE_SHIFT, MAX_PFN_MAPPED<<PAGE_SHIFT)`.
   - if ret == 0 ∧ CAN_USE_BRK_PGT:
     - ret = __pa(extend_brk(PAGE_SIZE*num, PAGE_SIZE)).
   - if ret == 0: panic("alloc_low_pages: can not alloc memory").
   - pfn = ret >> PAGE_SHIFT.
3. else:
   - pfn = PGT_BUF_END; PGT_BUF_END += num.
4. for i in 0..num: clear_page(__va((pfn+i) << PAGE_SHIFT)).
5. return __va(pfn << PAGE_SHIFT).

`X86Mm::init_memory_mapping(start, end, prot) -> u64`:
1. mr = [MapRange; NR_RANGE_MR]; nr_range = `X86Mm::split_mem_range(&mut mr, 0, start, end)`.
2. ret = 0.
3. for i in 0..nr_range:
   - ret = kernel_physical_mapping_init(mr[i].start, mr[i].end, mr[i].page_size_mask, prot).
4. `X86Mm::add_pfn_range_mapped(start >> PAGE_SHIFT, ret >> PAGE_SHIFT)`.
5. return ret >> PAGE_SHIFT.

`X86Mm::add_pfn_range_mapped(start_pfn, end_pfn)`:
1. NR_PFN_MAPPED = add_range_with_merge(PFN_MAPPED, E820_MAX_ENTRIES, NR_PFN_MAPPED, start_pfn, end_pfn).
2. NR_PFN_MAPPED = clean_sort_range(PFN_MAPPED, E820_MAX_ENTRIES).
3. MAX_PFN_MAPPED = max(MAX_PFN_MAPPED, end_pfn).
4. if start_pfn < (1 << (32 - PAGE_SHIFT)):
   - MAX_LOW_PFN_MAPPED = max(MAX_LOW_PFN_MAPPED, min(end_pfn, 1 << (32 - PAGE_SHIFT))).

`X86Mm::init_trampoline()`:
1. if !cfg!(CONFIG_X86_64): return.
2. if !kaslr_memory_enabled():
   - trampoline_pgd_entry = init_top_pgt[pgd_index(__PAGE_OFFSET)].
3. else: init_trampoline_kaslr().

`X86Mm::free_init_pages(what, begin, end)`:
1. begin_aligned = PAGE_ALIGN(begin); end_aligned = end & PAGE_MASK.
2. if begin_aligned != begin ∨ end_aligned != end: WARN; begin = begin_aligned; end = end_aligned.
3. if begin >= end: return.
4. if debug_pagealloc_enabled():
   - pr_info("debug: unmapping init ...").
   - kmemleak_free_part(begin, end - begin).
   - set_memory_np(begin, (end - begin) >> PAGE_SHIFT).
5. else:
   - set_memory_nx(begin, (end - begin) >> PAGE_SHIFT).
   - set_memory_rw(begin, (end - begin) >> PAGE_SHIFT).
   - free_reserved_area(begin, end, POISON_FREE_INITMEM, what).

`X86Mm::free_kernel_image_pages(what, begin, end)`:
1. `X86Mm::free_init_pages(what, begin, end)`.
2. if cfg!(CONFIG_X86_64) ∧ X86_FEATURE_PTI: set_memory_np_noalias(begin, len_pages).

`X86Mm::free_initmem()`:
1. e820__reallocate_tables().
2. mem_encrypt_free_decrypted_mem().
3. `X86Mm::free_kernel_image_pages("unused kernel image (initmem)", &__init_begin, &__init_end)`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `alloc_low_pages_zeroed` | INVARIANT | per-alloc_low_pages: every returned page cleared via clear_page. |
| `pgt_buf_within_brk` | INVARIANT | per-alloc_low_pages: PGT_BUF_END ≤ PGT_BUF_TOP after BRK path. |
| `pfn_mapped_sorted` | INVARIANT | per-add_pfn_range_mapped: clean_sort_range invariant — pfn_mapped[i].end ≤ pfn_mapped[i+1].start. |
| `pfn_mapped_no_overlap` | INVARIANT | per-add_pfn_range_mapped: no overlapping ranges after merge. |
| `nr_pfn_mapped_bounded` | INVARIANT | per-add_pfn_range_mapped: NR_PFN_MAPPED ≤ E820_MAX_ENTRIES. |
| `max_pfn_mapped_monotone` | INVARIANT | per-add_pfn_range_mapped: MAX_PFN_MAPPED non-decreasing. |
| `save_mr_bounded` | INVARIANT | per-save_mr: nr_range < NR_RANGE_MR (else panic). |
| `free_init_pages_aligned` | INVARIANT | per-free_init_pages: begin, end are PAGE_ALIGN-corrected before free. |
| `init_memory_mapping_isa_first` | INVARIANT | per-init_mem_mapping: init_memory_mapping(0, ISA_END_ADDRESS) called before memory_map_*. |

### Layer 2: TLA+

`arch/x86/mm/init.tla`:
- Per-boot-init-mem-mapping orchestration: per-probe → per-pcid-setup → per-ISA-range → per-trampoline → per-top-down/bottom-up → per-CR3-reload → per-flush.
- Per-page-table-allocator: per-BRK → per-memblock-fallback → per-extend-brk → per-buddy (after_bootmem).
- Properties:
  - `safety_isa_range_always_mapped` — per-init_mem_mapping: pfn_range_is_mapped(0, ISA_END_ADDRESS >> PAGE_SHIFT).
  - `safety_max_pfn_mapped_monotone` — per-add_pfn_range_mapped: never decreases.
  - `safety_brk_exhaustion_panics_or_falls_back` — per-alloc_low_pages: either succeeds via memblock/brk-extend or panics, never silent corruption.
  - `safety_after_bootmem_no_brk` — once AFTER_BOOTMEM, the BRK/memblock path is never re-entered.
  - `safety_initmem_freed_once` — per-free_initmem: [__init_begin, __init_end) reclaimed exactly once per boot.
  - `safety_pti_alias_removed_on_image_free` — per-free_kernel_image_pages with X86_FEATURE_PTI: high-kernel-image alias unmapped.
  - `liveness_init_mem_mapping_terminates` — per-boot: orchestration completes within memblock-bounded iterations.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `X86Mm::init_mem_mapping` post: max_pfn_mapped ≥ (max_pfn ∨ max_low_pfn) | `X86Mm::init_mem_mapping` |
| `X86Mm::init_memory_mapping` post: returns PFN-mapped count; pfn_mapped[] updated | `X86Mm::init_memory_mapping` |
| `X86Mm::alloc_low_pages` post: returns PAGE-aligned VA covering `num` zeroed phys pages | `X86Mm::alloc_low_pages` |
| `X86Mm::probe_page_size_mask` post: CR4.PSE / CR4.PGE consistent with X86_FEATURE_* | `X86Mm::probe_page_size_mask` |
| `X86Mm::setup_pcid` post: X86_FEATURE_PCID enabled ⟹ X86_FEATURE_PGE was enabled | `X86Mm::setup_pcid` |
| `X86Mm::split_mem_range` post: ∑(mr[i].end - mr[i].start) == end - start; nr_range ≤ NR_RANGE_MR | `X86Mm::split_mem_range` |
| `X86Mm::free_init_pages` post: begin >= end ∨ free_reserved_area called ∨ set_memory_np called | `X86Mm::free_init_pages` |
| `X86Mm::pfn_range_is_mapped` post: true ⟹ ∃ i. pfn_mapped[i] covers [start_pfn, end_pfn) | `X86Mm::pfn_range_is_mapped` |
| `X86Mm::devmem_is_allowed` post: returns 0/1/2; pagenr<256 ⟹ never returns 0 for low-RAM/iomem case | `X86Mm::devmem_is_allowed` |
| `X86Mm::arch_zone_limits_init` post: max_zone_pfns[ZONE_NORMAL] == max_low_pfn | `X86Mm::arch_zone_limits_init` |

### Layer 4: Verus/Creusot functional

`per-setup_arch → per-e820_to_memblock → per-init_mem_mapping → per-paging_init → per-free_initmem` semantic equivalence: per-Documentation/arch/x86/x86_64/mm.rst (kernel virtual address-space layout), per-Documentation/admin-guide/kernel-parameters.txt (`gbpages`, `nogbpages`, `debugpat`, `noinvpcid`, `nopti`), and per-Documentation/admin-guide/mm/memory-hotplug.rst (max_pfn / max_low_pfn watermarks for hotplug compatibility).

## Hardening

(Inherits row-1 features from `arch/x86/00-overview.md` § Hardening.)

Memory-init reinforcement:

- **Per-`alloc_low_pages` panic on exhaustion** — defense against per-silent-zero-page-table corruption.
- **Per-`clear_page` of every allocated pgt page** — defense against per-stale-data-in-PTE.
- **Per-`add_range_with_merge` + `clean_sort_range` of pfn_mapped[]** — defense against per-duplicate or overlapping mapped ranges (would corrupt direct map).
- **Per-`NR_RANGE_MR` panic on exhaustion in save_mr** — defense against per-silent-truncation-of-mapping.
- **Per-ISA-range always-mapped (init_memory_mapping(0, ISA_END_ADDRESS))** — defense against per-trampoline / BIOS legacy access fault.
- **Per-INVLPG-miss errata: PCID disabled on affected Intel uarches with stale microcode** — defense against per-stale-TLB-on-global-flush.
- **Per-`can_use_brk_pgt` toggled around brk-overlap regions** — defense against per-double-allocation of BRK-resident PFNs.
- **Per-PTI high-kernel-image-alias unmap on free** — defense against per-Meltdown-residual-mapping after initmem free.
- **Per-`set_memory_nx` + `set_memory_rw` before free_reserved_area** — defense against per-W^X-violation on freed __init code.
- **Per-`debug_pagealloc_enabled` unmap-but-not-free** — defense against per-UAF-of-init-section detection.
- **Per-`set_notrack_mm` for text_poke_mm** — defense against per-MM-tracking accounting drift during text patching.
- **Per-`x86_init.hyper.init_mem_mapping` paravirt hook** — defense against per-hypervisor-mapping desync (Xen-PV pgd pinning).
- **Per-`early_memtest` over mapped range** — defense against per-bad-RAM corruption escaping into kernel.

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_KERNEXEC** — W^X enforcement for kernel rwx zones: `free_init_pages` issues `set_memory_nx` + `set_memory_rw` before `free_reserved_area`; `free_kernel_image_pages` also unmaps the PTI high-kernel-image alias under X86_FEATURE_PTI.
- **PAX_RANDMMAP / KASLR** — kernel base + direct map randomized; `init_trampoline_kaslr` aliases only the PUD covering low 1MB (1GB-granular randomization).
- **PAX_RANDKSTACK** — per-syscall kernel-stack randomization enabled once paging is up.
- **PAX_MEMORY_SANITIZE** — `clear_page` of every freshly-allocated page-table page (via `alloc_low_pages`); `free_init_pages` poisons reclaimed __init memory with `POISON_FREE_INITMEM`.
- **PAX_USERCOPY** — `/dev/mem` access policy via `devmem_is_allowed` gates page reads against system-RAM membership.
- **PAX_UDEREF (SMAP/SMEP)** — CR4 bits set early; direct map establishes the U/S separation.
- **PAX_RAP / kCFI** — indirect-call signature enforcement on `x86_init.hyper.init_mem_mapping` paravirt hook.
- **PAX_REFCOUNT** — saturating refcount on early `alloc_low_pages` BRK buffer accounting.
- **GRKERNSEC_HIDESYM** — kernel-pointer hiding in /proc and dmesg; `pfn_mapped[]` table not exposed.
- **GRKERNSEC_DMESG** — restrict syslog output to CAP_SYSLOG (including `early_memtest` failure logs).
- **GRKERNSEC_KMEM** — block `/dev/mem` access to kernel direct-map; only IORESOURCE_BUSY-marked iomem ranges accessible under CAP_SYS_RAWIO.
- **ISA-range always-mapped (REQ-15)** — defense against trampoline / BIOS legacy access fault under hostile e820.
- **INVLPG-miss errata PCID disable** — defense against stale-TLB on global flush on affected Intel uarches with stale microcode.
- **PTI alias unmap on initmem free** — defense against Meltdown-residual mapping after `free_initmem`.
- **debug_pagealloc unmap-but-don't-free** — defense against UAF-of-init-section detection.
- **L1TF swapfile cap (`arch_max_swapfile_size`)** — defense against L1TF swap-entry abuse.
- **set_notrack_mm for text_poke_mm** — defense against MM-tracking accounting drift.

Per-doc rationale: mm/init is the seam between firmware memory layout and the kernel direct map; grsec/PaX hardening here ensures W^X is established before any page is freed back to the allocator, KASLR entropy is preserved through trampoline aliasing, and Meltdown / L1TF / PTI residual-mapping defenses are layered atop the direct map.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `arch/x86/mm/init_64.c` `kernel_physical_mapping_init` page-table walker (covered in `paging.md` Tier-3).
- `arch/x86/mm/init_32.c` 32-bit `kernel_physical_mapping_init` (Rookery is 64-bit only, but the contract is documented here for parity).
- `arch/x86/mm/ident_map.c` `kernel_ident_mapping_init` (used for kexec + trampoline; covered separately or folded into `boot.md`).
- `arch/x86/mm/fault.c` page-fault handler (covered in `paging.md` Tier-3).
- `arch/x86/mm/pat/` memory-type tables update path beyond `update_cache_mode_entry` (covered in `paging.md` Tier-3).
- `arch/x86/mm/tlb.c` TLB-flush IPI machinery (covered in `paging.md` Tier-3, alongside `cpu_tlbstate`).
- `arch/x86/mm/kaslr.c` KASLR memory randomization beyond `init_trampoline_kaslr` invocation (covered in `paging.md` Tier-3 or a dedicated `kaslr.md`).
- `arch/x86/mm/mem_encrypt*.c` SME/SEV beyond `mem_encrypt_free_decrypted_mem` invocation (covered in `mem_encrypt.md` Tier-3).
- `arch/x86/kernel/setup.c` `setup_arch` orchestration (covered in `kernel-platform.md` Tier-3).
- `arch/x86/kernel/e820.c` E820 → memblock translation (covered in `kernel-platform.md` Tier-3).
- `mm/memblock.c` memblock allocator core (covered in `mm/memblock.md` Tier-3).
- `mm/sparse.c` `sparse_init` / `setup_memory` (covered in `mm/sparse.md` Tier-3).
- Implementation code.
