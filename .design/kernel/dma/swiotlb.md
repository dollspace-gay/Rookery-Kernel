# Tier-3: kernel/dma/swiotlb.c — Software IOTLB bounce buffer

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/dma/00-overview.md
upstream-paths:
  - kernel/dma/swiotlb.c (~1901 lines)
  - include/linux/swiotlb.h
  - include/trace/events/swiotlb.h
  - Documentation/core-api/swiotlb.rst
-->

## Summary

The **Software IOTLB** ("swiotlb") is the kernel's last-resort DMA bounce buffer used whenever a buffer's physical address is unreachable by the device, or whenever the system policy forbids exposing arbitrary physical addresses to the device. Per-pool data: `struct io_tlb_mem` (top-level handle) + `struct io_tlb_pool` (one contiguous slab range) + per-NUMA `struct io_tlb_area` (per-area lock + rotating index) + per-slot `struct io_tlb_slot` (orig_addr, alloc_size, free-list link, padding-slot count). Per-`swiotlb_init` / `_init_remap` / `_init_late` allocate a default pool from `memblock` at boot (default 64 MiB on x86_64 via `IO_TLB_DEFAULT_SIZE >> IO_TLB_SHIFT`); per-`swiotlb_tbl_map_single` carves a slot range out of the pool, copies the original buffer in if direction is TO_DEVICE/BIDIRECTIONAL, and returns the bounce-buffer physical address; per-`swiotlb_tbl_unmap_single` copies back if direction is FROM_DEVICE/BIDIRECTIONAL and releases the slots. Per-`CONFIG_SWIOTLB_DYNAMIC` allows on-demand growth via `swiotlb_dyn_alloc` workqueue and transient pools attached to a single mapping. Per-`CONFIG_DMA_RESTRICTED_POOL` allows a per-device pool described in DT (`restricted-dma-pool` reserved-memory entry) used instead of the global default. Per-`cc_platform_has(CC_ATTR_MEM_ENCRYPT)` confidential-guest path: swiotlb is forced for every DMA (`force_bounce = true`) and the pool pages are `set_memory_decrypted` so the device sees plaintext. Critical for: 32-bit-DMA legacy devices, SEV/SEV-ES/SEV-SNP/TDX confidential guests, restricted-DMA platforms (ARM/RISC-V SoCs), IOMMU fallback when no IOMMU page is available.

This Tier-3 covers `kernel/dma/swiotlb.c` (~1901 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct io_tlb_mem` | per-pool control block (default + per-device restricted) | `Swiotlb::IoTlbMem` |
| `struct io_tlb_pool` | per-contiguous-pool slot table | `Swiotlb::IoTlbPool` |
| `struct io_tlb_area` | per-NUMA-area sub-pool with `used`, `index`, `lock` | `Swiotlb::IoTlbArea` |
| `struct io_tlb_slot` | per-slot orig_addr + alloc_size + list + pad_slots | `Swiotlb::IoTlbSlot` |
| `io_tlb_default_mem` | per-boot default pool | `Swiotlb::DEFAULT_MEM` |
| `swiotlb_init(addressing_limit, flags)` | per-boot memblock init | `Swiotlb::init` |
| `swiotlb_init_remap(limit, flags, remap)` | per-boot with arch-remap callback (SEV/TDX) | `Swiotlb::init_remap` |
| `swiotlb_init_late(size, gfp, remap)` | per-runtime buddy-allocator init | `Swiotlb::init_late` |
| `swiotlb_exit()` | per-tear-down for kdump/kexec | `Swiotlb::exit` |
| `swiotlb_init_io_tlb_pool(pool, start, nslabs, late, nareas)` | per-pool slot-table init | `IoTlbPool::init` |
| `swiotlb_memblock_alloc(nslabs, flags, remap)` | per-boot memblock alloc helper | `Swiotlb::memblock_alloc` |
| `swiotlb_adjust_size(size)` / `_adjust_nareas(n)` | per-arch tuning hook | `Swiotlb::adjust_size` / `adjust_nareas` |
| `swiotlb_size_or_default()` | per-`swiotlb=` cmdline resolver | `Swiotlb::size_or_default` |
| `swiotlb_print_info()` | per-boot dmesg banner | `Swiotlb::print_info` |
| `swiotlb_dev_init(dev)` | per-device init (default pool pointer) | `Device::swiotlb_init` |
| `swiotlb_tbl_map_single(dev, orig, size, align, dir, attrs)` | per-map: alloc slot + bounce-in | `IoTlbMem::map_single` |
| `__swiotlb_tbl_unmap_single(dev, tlb, size, dir, attrs, pool)` | per-unmap: bounce-out + release | `IoTlbMem::unmap_single` |
| `__swiotlb_sync_single_for_device(dev, tlb, size, dir, pool)` | per-direction sync TO device | `IoTlbMem::sync_for_device` |
| `__swiotlb_sync_single_for_cpu(dev, tlb, size, dir, pool)` | per-direction sync FROM device | `IoTlbMem::sync_for_cpu` |
| `swiotlb_bounce(dev, tlb, size, dir, pool)` | per-direction memcpy primitive | `IoTlbMem::bounce` |
| `swiotlb_align_offset(dev, mask, addr)` | per-alignment offset within slot | `IoTlbMem::align_offset` |
| `swiotlb_find_slots(dev, orig, size, align, &pool)` | per-allocation slot search | `IoTlbMem::find_slots` |
| `swiotlb_search_pool_area(dev, pool, area_idx, orig, size, align)` | per-area search | `IoTlbPool::search_area` |
| `swiotlb_search_area(dev, start_cpu, cpu_offset, orig, size, align, &pool)` | per-CPU area-rotation search | `IoTlbMem::search_area` |
| `swiotlb_release_slots(dev, tlb_addr, pool)` | per-free + free-list merge | `IoTlbPool::release_slots` |
| `swiotlb_map(dev, paddr, size, dir, attrs)` | per-direct-DMA fallback wrapper | `Swiotlb::map` |
| `swiotlb_max_mapping_size(dev)` | per-device single-mapping cap | `IoTlbMem::max_mapping_size` |
| `swiotlb_alloc(dev, size)` / `_free(dev, page, size)` | per-`for_alloc` direct-allocator | `IoTlbMem::alloc_page` / `_free_page` |
| `__swiotlb_find_pool(dev, paddr)` | per-paddr → pool lookup (RCU) | `IoTlbMem::find_pool` |
| `swiotlb_alloc_pool(dev, minslabs, nslabs, nareas, phys_limit, gfp)` | per-dynamic-growth pool alloc | `IoTlbMem::alloc_pool` |
| `swiotlb_dyn_alloc(work)` | per-workqueue background pool grow | `IoTlbMem::dyn_alloc` |
| `swiotlb_dyn_free(rcu)` | per-RCU pool free | `IoTlbMem::dyn_free` |
| `swiotlb_del_pool(dev, pool)` | per-RCU pool unlink | `IoTlbMem::del_pool` |
| `swiotlb_del_transient(dev, tlb, pool)` | per-unmap transient-pool teardown | `IoTlbMem::del_transient` |
| `is_swiotlb_allocated()` / `is_swiotlb_active(dev)` | per-state probe | `Swiotlb::allocated` / `active` |
| `default_swiotlb_base()` / `_limit()` | per-default-pool extent | `Swiotlb::default_base` / `default_limit` |
| `rmem_swiotlb_device_init(rmem, dev)` | per-DT restricted-pool device attach | `Swiotlb::rmem_device_init` |
| `rmem_swiotlb_setup(node, rmem)` | per-DT restricted-pool node setup | `Swiotlb::rmem_setup` |
| `trace_swiotlb_bounced` | per-bounce tracepoint | shared |

## Compatibility contract

REQ-1: struct io_tlb_slot:
- orig_addr: phys_addr_t — original buffer paddr (`INVALID_PHYS_ADDR` if free).
- alloc_size: size_t — mapping size carved in this slot range (recorded only in head slot).
- list: u16 — free-list count of contiguous free slots from this index (0 = used).
- pad_slots: u16 — number of leading padding slots due to `min_align_mask` (head slot only).

REQ-2: struct io_tlb_area:
- used: unsigned long — number of in-use slots in this area.
- index: unsigned int — rotating search start within this area.
- lock: spinlock_t — protects `used` + `index` + the area's slot free-list edits.

REQ-3: struct io_tlb_pool:
- start, end: phys_addr_t — pool physical extent.
- vaddr: void * — `memremap`/`phys_to_virt` linear mapping used by `swiotlb_bounce` memcpy.
- nslabs: unsigned long — total slot count in pool.
- area_nslabs: unsigned long — slots per area (`nslabs / nareas`).
- nareas: unsigned int — per-NUMA area count.
- areas: struct io_tlb_area * — area array (`nareas` entries).
- slots: struct io_tlb_slot * — slot array (`nslabs` entries).
- late_alloc: bool — true if allocated via buddy (`__get_free_pages`) rather than memblock.
- transient: bool — per-CONFIG_SWIOTLB_DYNAMIC transient single-mapping pool.
- node: struct list_head — linked into `io_tlb_mem.pools` (RCU-linked).
- rcu: struct rcu_head — for `call_rcu` free.

REQ-4: struct io_tlb_mem:
- defpool: struct io_tlb_pool — primary embedded pool.
- pools: struct list_head — RCU list of additional pools (dynamic growth + restricted).
- nslabs: unsigned long — total across all pools.
- can_grow: bool — per-CONFIG_SWIOTLB_DYNAMIC dynamic-growth allowed.
- phys_limit: u64 — upper paddr bound for dynamic alloc (per zone).
- force_bounce: bool — per-`swiotlb=force` or per-CC-guest forced path.
- for_alloc: bool — per-restricted-pool direct-allocator backing.
- lock: spinlock_t — per-CONFIG_SWIOTLB_DYNAMIC pool-list mutation lock.
- dyn_alloc: struct work_struct — per-CONFIG_SWIOTLB_DYNAMIC background grow worker.
- total_used: atomic_long_t — debugfs accounting (under CONFIG_DEBUG_FS).
- used_hiwater: atomic_long_t — debugfs high-water mark.
- transient_nslabs: atomic_long_t — per-CONFIG_SWIOTLB_DYNAMIC transient-pool accounting.

REQ-5: swiotlb_init_remap(addressing_limit, flags, remap):
- /* Skip if no need + not forced */
- if !addressing_limit ∧ !swiotlb_force_bounce: return.
- if swiotlb_force_disable: return.
- /* Forced-bounce policy */
- io_tlb_default_mem.force_bounce = swiotlb_force_bounce ∨ (flags & SWIOTLB_FORCE).
- /* Per-area count from possible CPUs */
- if !default_nareas: swiotlb_adjust_nareas(num_possible_cpus()).
- nslabs = default_nslabs; nareas = limit_nareas(default_nareas, nslabs).
- /* memblock alloc + arch remap (SEV/TDX) — retry halving on failure */
- while !(tlb = swiotlb_memblock_alloc(nslabs, flags, remap)):
  - if nslabs ≤ IO_TLB_MIN_SLABS: return (give-up).
  - nslabs = ALIGN(nslabs >> 1, IO_TLB_SEGSIZE); nareas = limit_nareas(nareas, nslabs).
- /* Per-slot table from memblock */
- mem->slots = memblock_alloc(nslabs * sizeof(io_tlb_slot), PAGE_SIZE).
- /* Per-area table from memblock */
- mem->areas = memblock_alloc(nareas * sizeof(io_tlb_area), SMP_CACHE_BYTES).
- swiotlb_init_io_tlb_pool(mem, __pa(tlb), nslabs, late=false, nareas).
- add_mem_pool(&io_tlb_default_mem, mem).
- if flags & SWIOTLB_VERBOSE: swiotlb_print_info().

REQ-6: swiotlb_init_late(size, gfp_mask, remap):
- /* Runtime fallback: only if not already initialized */
- if io_tlb_default_mem.nslabs: return 0.
- if swiotlb_force_disable: return 0.
- /* Determine phys_limit per gfp zone */
- nslabs = ALIGN(size >> IO_TLB_SHIFT, IO_TLB_SEGSIZE).
- /* Buddy-allocator pool */
- order = get_order(nslabs << IO_TLB_SHIFT).
- while order > min: vstart = __get_free_pages(gfp | __GFP_NOWARN, order); if vstart: break.
- if !vstart: return -ENOMEM.
- /* Arch remap (SEV/TDX) — retry halving on failure */
- if remap: rc = remap(vstart, nslabs); if rc: free + halve + retry.
- /* areas + slots from buddy */
- mem->areas = __get_free_pages(GFP_KERNEL | __GFP_ZERO, area_order).
- mem->slots = __get_free_pages(GFP_KERNEL | __GFP_ZERO, slot_order).
- set_memory_decrypted(vstart, …) — unencrypt pool pages (SEV C-bit clear / TDX SHARED-bit set).
- swiotlb_init_io_tlb_pool(mem, virt_to_phys(vstart), nslabs, late=true, nareas).
- add_mem_pool(&io_tlb_default_mem, mem).
- swiotlb_print_info(); return 0.

REQ-7: swiotlb_exit():
- /* No-op if forced or never initialized */
- if swiotlb_force_bounce ∨ !mem->nslabs: return.
- /* Restore encryption + free */
- set_memory_encrypted(tbl_vaddr, …) — re-encrypt pool pages.
- if mem->late_alloc: free_pages(areas); free_pages(tlb); free_pages(slots).
- else: memblock_free(areas); memblock_phys_free(start); memblock_free(slots).
- memset(mem, 0, sizeof(*mem)) — wipe.

REQ-8: swiotlb_tbl_map_single(dev, orig_addr, mapping_size, alloc_align_mask, dir, attrs):
- mem = dev->dma_io_tlb_mem.
- /* No pool → DMA_MAPPING_ERROR */
- if !mem ∨ !mem->nslabs: return DMA_MAPPING_ERROR (warn-ratelimited).
- /* CC-guest banner once */
- if cc_platform_has(CC_ATTR_MEM_ENCRYPT): pr_warn_once.
- /* Alignment offset = orig_addr & dma_get_min_align_mask(dev) & (align_mask | (IO_TLB_SIZE - 1)) */
- offset = swiotlb_align_offset(dev, alloc_align_mask, orig_addr).
- size = ALIGN(mapping_size + offset, alloc_align_mask + 1).
- index = swiotlb_find_slots(dev, orig_addr, size, alloc_align_mask, &pool).
- if index == -1: return DMA_MAPPING_ERROR (warn-ratelimited "swiotlb buffer is full").
- /* Reset dma_skip_sync so subsequent map/unmap honor sync */
- dma_reset_need_sync(dev).
- /* Record padding + per-slot orig_addr */
- pad_slots = offset >> IO_TLB_SHIFT; offset &= (IO_TLB_SIZE - 1); index += pad_slots.
- pool->slots[index].pad_slots = pad_slots.
- for i in 0..(nr_slots(size) - pad_slots): pool->slots[index+i].orig_addr = slot_addr(orig_addr, i).
- tlb_addr = slot_addr(pool->start, index) + offset.
- /* Always bounce-in (TO_DEVICE) — even FROM_DEVICE preserves caller bytes if partial write */
- swiotlb_bounce(dev, tlb_addr, mapping_size, DMA_TO_DEVICE, pool).
- return tlb_addr.

REQ-9: __swiotlb_tbl_unmap_single(dev, tlb_addr, mapping_size, dir, attrs, pool):
- /* Sync back unless skipped */
- if !(attrs & DMA_ATTR_SKIP_CPU_SYNC) ∧ (dir == FROM_DEVICE ∨ dir == BIDIRECTIONAL):
  - swiotlb_bounce(dev, tlb_addr, mapping_size, DMA_FROM_DEVICE, pool).
- /* Per-CONFIG_SWIOTLB_DYNAMIC transient-pool teardown */
- if swiotlb_del_transient(dev, tlb_addr, pool): return.
- swiotlb_release_slots(dev, tlb_addr, pool).

REQ-10: __swiotlb_sync_single_for_device(dev, tlb_addr, size, dir, pool):
- if dir == TO_DEVICE ∨ dir == BIDIRECTIONAL: swiotlb_bounce(…, DMA_TO_DEVICE, …).
- else: BUG_ON(dir != FROM_DEVICE).

REQ-11: __swiotlb_sync_single_for_cpu(dev, tlb_addr, size, dir, pool):
- if dir == FROM_DEVICE ∨ dir == BIDIRECTIONAL: swiotlb_bounce(…, DMA_FROM_DEVICE, …).
- else: BUG_ON(dir != TO_DEVICE).

REQ-12: swiotlb_bounce(dev, tlb_addr, size, dir, pool):
- index = (tlb_addr - pool->start) >> IO_TLB_SHIFT.
- orig_addr = pool->slots[index].orig_addr; alloc_size = pool->slots[index].alloc_size.
- vaddr = pool->vaddr + (tlb_addr - pool->start).
- if orig_addr == INVALID_PHYS_ADDR: return.
- /* DMA flush on FROM_DEVICE for non-coherent arches */
- if dir == FROM_DEVICE ∧ !dev_is_dma_coherent(dev): arch_sync_dma_flush().
- /* Compute tlb_offset relative to first-slot alignment */
- tlb_offset = (tlb_addr & (IO_TLB_SIZE - 1)) - swiotlb_align_offset(dev, 0, orig_addr).
- orig_addr += tlb_offset; alloc_size -= tlb_offset.
- if size > alloc_size: WARN_ONCE "Buffer overflow"; size = alloc_size.
- /* Highmem path: kmap each page; lowmem path: memcpy(orig_addr, vaddr, size) */
- if PageHighMem(pfn_to_page(PFN_DOWN(orig_addr))): per-page kmap_local + memcpy.
- else: per-direction memcpy between phys_to_virt(orig_addr) and vaddr.

REQ-13: swiotlb_release_slots(dev, tlb_addr, pool):
- offset = swiotlb_align_offset(dev, 0, tlb_addr).
- index = (tlb_addr - offset - pool->start) >> IO_TLB_SHIFT.
- index -= pool->slots[index].pad_slots.
- nslots = nr_slots(pool->slots[index].alloc_size + offset).
- aindex = index / pool->area_nslabs; area = &pool->areas[aindex].
- BUG_ON(aindex >= pool->nareas).
- /* Take area lock; rebuild free-list with merge */
- spin_lock_irqsave(&area->lock).
- /* Count run to succeeding free slot, capped at next segment boundary */
- count = (slots[index + nslots] within same IO_TLB_SEGSIZE ∧ list != 0) ? slots[…].list : 0.
- /* Step 1: mark `nslots` free, descending, accumulating count */
- for i in (index + nslots - 1) … index: slots[i].list = ++count; slots[i].orig_addr = INVALID_PHYS_ADDR; slots[i].alloc_size = 0; slots[i].pad_slots = 0.
- /* Step 2: merge with predecessor in same segment */
- for i in (index - 1) … while io_tlb_offset(i) != IO_TLB_SEGSIZE - 1 ∧ slots[i].list: slots[i].list = ++count.
- area->used -= nslots.
- spin_unlock_irqrestore.
- dec_used(dev->dma_io_tlb_mem, nslots).

REQ-14: Per-CONFIG_SWIOTLB_DYNAMIC dynamic growth:
- io_tlb_mem.can_grow = true (unless arch remap callback supplied).
- io_tlb_mem.dyn_alloc = INIT_WORK(swiotlb_dyn_alloc) — background worker.
- swiotlb_alloc_pool(dev, minslabs, nslabs, nareas, phys_limit, gfp):
  - alloc_dma_pages(gfp, bytes, phys_limit) — buddy alloc with phys_limit retry.
  - swiotlb_init_io_tlb_pool(pool, paddr, …); set pool->transient if attached to single mapping.
- __swiotlb_find_pool(dev, paddr): rcu_read_lock; list_for_each_entry_rcu over mem->pools + dev->dma_io_tlb_pools; return matching pool by paddr range.
- swiotlb_del_pool(dev, pool): list_del_rcu under dma_io_tlb_lock; call_rcu(swiotlb_dyn_free).
- swiotlb_dyn_free(rcu): swiotlb_free_tlb(vaddr, bytes) (set_memory_encrypted + free_pages).

REQ-15: Per-CONFIG_DMA_RESTRICTED_POOL per-device pool:
- DT `restricted-dma-pool` reserved-memory entry registered via `RESERVEDMEM_OF_DECLARE(dma, "restricted-dma-pool", &rmem_swiotlb_ops)`.
- rmem_swiotlb_setup(node, rmem): reject `reusable` / `linux,cma-default` / `linux,dma-default` / `no-map`.
- rmem_swiotlb_device_init(rmem, dev): on first attach allocate `io_tlb_mem` + pool->slots + pool->areas via slab; set_memory_decrypted; swiotlb_init_io_tlb_pool(false-late, nareas=1); mem->force_bounce = true; mem->for_alloc = true; subsequent attaches reuse rmem->priv.
- dev->dma_io_tlb_mem = mem (overrides default).
- rmem_swiotlb_device_release(rmem, dev): dev->dma_io_tlb_mem = &io_tlb_default_mem.

REQ-16: Per-CC-guest forced bounce (SEV/SEV-ES/SEV-SNP/TDX):
- cc_platform_has(CC_ATTR_GUEST_MEM_ENCRYPT) at boot → swiotlb_force_bounce = true.
- swiotlb_init_remap honors SWIOTLB_FORCE flag; passes arch `remap` callback (e.g., `set_memory_decrypted`) which clears SEV C-bit / sets TDX SHARED-bit on pool pages.
- After init, every `dma_map_*` for a device on this guest funnels into `swiotlb_map` regardless of `dev->dma_mask`.

REQ-17: Out-of-slots policy:
- swiotlb_tbl_map_single returns `DMA_MAPPING_ERROR` (== `~(dma_addr_t)0`).
- dev_warn_ratelimited "swiotlb buffer is full (sz: %zd, total %lu slots, used %lu)" unless `DMA_ATTR_NO_WARN`.
- Per-CONFIG_SWIOTLB_DYNAMIC: scheduler hits `swiotlb_dyn_alloc` to add a new pool in the background.
- Per-`swiotlb_max_mapping_size(dev)` caps single mapping at `(IO_TLB_SEGSIZE << IO_TLB_SHIFT) - dma_get_min_align_mask(dev) - 1` to keep one mapping confined to a segment.

REQ-18: Debug surface (CONFIG_DEBUG_FS):
- `/sys/kernel/debug/swiotlb/io_tlb_nslabs` — total slot count.
- `/sys/kernel/debug/swiotlb/io_tlb_used` — current in-use slot count (via `io_tlb_used_get`).
- `/sys/kernel/debug/swiotlb/io_tlb_used_hiwater` — high-water (read via `io_tlb_hiwater_get`; writable via `io_tlb_hiwater_set` to reset).
- `/sys/kernel/debug/swiotlb/io_tlb_transient_nslabs` — per-CONFIG_SWIOTLB_DYNAMIC transient-pool slot count.

## Acceptance Criteria

- [ ] AC-1: Boot with `swiotlb=8192,force` → dmesg "software IO TLB: mapped [mem …] (64MB)" + `cat /sys/kernel/debug/swiotlb/io_tlb_nslabs` == 8192.
- [ ] AC-2: 32-bit-DMA-mask test driver allocates buffer above 4 GiB and `dma_map_single` → returned dma_addr is within swiotlb pool extent; payload preserved after `dma_unmap_single`.
- [ ] AC-3: SEV-SNP guest with `mem=8G` and PCI net device → every `dma_map_*` returns a pool address; `cat /sys/kernel/debug/swiotlb/io_tlb_used` non-zero under sustained traffic.
- [ ] AC-4: NUMA-area scaling: 2-socket system, 64 concurrent DMA threads → per-area lock contention as observed in `perf lock` remains bounded; no single area takes > 70% of allocations.
- [ ] AC-5: Out-of-slots stress: pool sized to IO_TLB_MIN_SLABS, 100 concurrent threads → some `dma_map_single` return `DMA_MAPPING_ERROR`; no UAF, no double-free, no memory corruption.
- [ ] AC-6: Tracepoint `swiotlb:swiotlb_bounced` fires on every map; `perf record -e swiotlb:swiotlb_bounced` decodes per-`force` flag.
- [ ] AC-7: Per-device restricted-pool via DT `restricted-dma-pool` → `dev->dma_io_tlb_mem` points at per-device pool, not default; debugfs entry created under pool name.
- [ ] AC-8: CONFIG_SWIOTLB_DYNAMIC: default pool exhausted → background `swiotlb_dyn_alloc` extends pool list within bounded time; `/sys/kernel/debug/swiotlb/io_tlb_transient_nslabs` updates.
- [ ] AC-9: `swiotlb_max_mapping_size(dev)` returns ≤ `(IO_TLB_SEGSIZE << IO_TLB_SHIFT) - 1` for default device.
- [ ] AC-10: BIDIRECTIONAL map: write to original buffer, dma_sync_for_device, device writes pool, dma_sync_for_cpu → original buffer contains device-written bytes.
- [ ] AC-11: `swiotlb_exit` on kdump path → pool memory re-encrypted (set_memory_encrypted) and released; subsequent `is_swiotlb_active(dev)` returns false.
- [ ] AC-12: `min_align_mask` honored: device with `dma_get_min_align_mask(dev) = 0xFF`, original buffer at `paddr = X` → bounce tlb_addr satisfies `(tlb_addr & 0xFF) == (X & 0xFF)`.

## Architecture

```
struct IoTlbSlot {
  orig_addr: u64,                      // INVALID_PHYS_ADDR (!0u64) when free
  alloc_size: usize,
  list: u16,                            // free-run count from this index
  pad_slots: u16,
}

struct IoTlbArea {
  used: AtomicUsize,
  index: AtomicU32,                     // rotating search start
  lock: SpinLock<()>,
}

struct IoTlbPool {
  start: u64,
  end: u64,
  vaddr: NonNull<u8>,                   // memremap / phys_to_virt
  nslabs: u32,
  area_nslabs: u32,
  nareas: u32,
  areas: KBox<[IoTlbArea]>,
  slots: KBox<[IoTlbSlot]>,
  late_alloc: bool,
  transient: bool,
  node: ListLink,                       // RCU list link
  rcu: RcuHead,
}

struct IoTlbMem {
  defpool: IoTlbPool,
  pools: RcuList<IoTlbPool>,            // additional dynamic/restricted pools
  nslabs: AtomicUsize,
  can_grow: bool,
  phys_limit: u64,
  force_bounce: bool,                   // SEV/TDX/swiotlb=force
  for_alloc: bool,                      // restricted-pool direct allocator
  lock: SpinLock<()>,                   // pool-list mutation
  dyn_alloc: WorkStruct,                // background grow worker
  total_used: AtomicU64,                // debugfs
  used_hiwater: AtomicU64,              // debugfs
  transient_nslabs: AtomicU64,
}
```

`Swiotlb::init_remap(addressing_limit, flags, remap) -> ()`:
1. if !addressing_limit ∧ !swiotlb_force_bounce: return.
2. if swiotlb_force_disable: return.
3. DEFAULT_MEM.force_bounce = swiotlb_force_bounce ∨ (flags & SWIOTLB_FORCE).
4. /* Per-CONFIG_SWIOTLB_DYNAMIC */
5. if !remap: DEFAULT_MEM.can_grow = true.
6. DEFAULT_MEM.phys_limit = (flags & SWIOTLB_ANY) ? high_memory_phys : ARCH_LOW_ADDRESS_LIMIT.
7. if !default_nareas: Swiotlb::adjust_nareas(num_possible_cpus()).
8. nslabs = default_nslabs; nareas = limit_nareas(default_nareas, nslabs).
9. /* Retry halving on memblock failure */
10. loop {
    - tlb = Swiotlb::memblock_alloc(nslabs, flags, remap)?;
    - if tlb.is_some(): break.
    - if nslabs ≤ IO_TLB_MIN_SLABS: return.
    - nslabs = align(nslabs >> 1, IO_TLB_SEGSIZE); nareas = limit_nareas(nareas, nslabs).
   }
11. /* slots + areas arrays from memblock */
12. defpool.slots = memblock_alloc(nslabs * sizeof(IoTlbSlot), PAGE_SIZE).
13. defpool.areas = memblock_alloc(nareas * sizeof(IoTlbArea), SMP_CACHE_BYTES).
14. IoTlbPool::init(&mut defpool, __pa(tlb), nslabs, late=false, nareas).
15. add_mem_pool(&mut DEFAULT_MEM, &defpool).
16. if flags & SWIOTLB_VERBOSE: Swiotlb::print_info().

`IoTlbMem::map_single(dev, orig_addr, mapping_size, alloc_align_mask, dir, attrs) -> phys_addr_t`:
1. mem = dev.dma_io_tlb_mem.
2. if !mem ∨ mem.nslabs == 0:
   - dev_warn_ratelimited "Cannot allocate SWIOTLB buffer".
   - return DMA_MAPPING_ERROR.
3. if cc_platform_has(CC_ATTR_MEM_ENCRYPT): pr_warn_once "Memory encryption active; using DMA bounce".
4. dev_WARN_ONCE(dev, alloc_align_mask > ~PAGE_MASK, "Alloc alignment may prevent max mapping_size").
5. offset = IoTlbMem::align_offset(dev, alloc_align_mask, orig_addr).
6. size = align_up(mapping_size + offset, alloc_align_mask + 1).
7. (pool, index) = IoTlbMem::find_slots(dev, orig_addr, size, alloc_align_mask)?.
8. if index == -1:
   - if !(attrs & DMA_ATTR_NO_WARN): dev_warn_ratelimited "swiotlb buffer is full".
   - return DMA_MAPPING_ERROR.
9. dma_reset_need_sync(dev).
10. pad_slots = offset >> IO_TLB_SHIFT.
11. offset &= IO_TLB_SIZE - 1.
12. index += pad_slots.
13. pool.slots[index].pad_slots = pad_slots.
14. for i in 0..(nr_slots(size) - pad_slots):
    - pool.slots[index + i].orig_addr = slot_addr(orig_addr, i).
15. tlb_addr = slot_addr(pool.start, index) + offset.
16. /* Always bounce in (TO_DEVICE) — preserves caller bytes for partial-write devices */
17. IoTlbMem::bounce(dev, tlb_addr, mapping_size, DMA_TO_DEVICE, &pool).
18. trace_swiotlb_bounced(dev, dma_addr, mapping_size, force_bounce).
19. return tlb_addr.

`IoTlbMem::unmap_single(dev, tlb_addr, mapping_size, dir, attrs, pool)`:
1. if !(attrs & DMA_ATTR_SKIP_CPU_SYNC) ∧ (dir == FROM_DEVICE ∨ dir == BIDIRECTIONAL):
   - IoTlbMem::bounce(dev, tlb_addr, mapping_size, DMA_FROM_DEVICE, pool).
2. if IoTlbMem::del_transient(dev, tlb_addr, pool): return.
3. IoTlbPool::release_slots(dev, tlb_addr, pool).

`IoTlbPool::release_slots(dev, tlb_addr, pool)`:
1. offset = IoTlbMem::align_offset(dev, 0, tlb_addr).
2. index = ((tlb_addr - offset - pool.start) >> IO_TLB_SHIFT) - pool.slots[…].pad_slots.
3. nslots = nr_slots(pool.slots[index].alloc_size + offset).
4. aindex = index / pool.area_nslabs.
5. assert!(aindex < pool.nareas).
6. let _g = pool.areas[aindex].lock.lock_irqsave();
7. /* Determine merge-with-successor count */
8. count = if (index + nslots) within same IO_TLB_SEGSIZE && pool.slots[index + nslots].list != 0 {
     pool.slots[index + nslots].list } else { 0 };
9. /* Step 1: mark released slots free, descending, accumulating run */
10. for i in (index + nslots - 1)..=index reversed:
    - pool.slots[i].list = (count += 1; count).
    - pool.slots[i].orig_addr = INVALID_PHYS_ADDR.
    - pool.slots[i].alloc_size = 0.
    - pool.slots[i].pad_slots = 0.
11. /* Step 2: merge backwards with predecessors until segment boundary */
12. let mut i = index - 1;
13. while io_tlb_offset(i) != IO_TLB_SEGSIZE - 1 ∧ pool.slots[i].list != 0:
    - pool.slots[i].list = (count += 1; count).
    - i -= 1.
14. pool.areas[aindex].used -= nslots.
15. drop(_g).
16. dec_used(dev.dma_io_tlb_mem, nslots).

`IoTlbMem::bounce(dev, tlb_addr, size, dir, pool)`:
1. index = (tlb_addr - pool.start) >> IO_TLB_SHIFT.
2. orig_addr = pool.slots[index].orig_addr; alloc_size = pool.slots[index].alloc_size.
3. if orig_addr == INVALID_PHYS_ADDR: return.
4. vaddr_slot = pool.vaddr.add(tlb_addr - pool.start).
5. if dir == FROM_DEVICE ∧ !dev_is_dma_coherent(dev): arch_sync_dma_flush().
6. tlb_offset = (tlb_addr & (IO_TLB_SIZE - 1)) - IoTlbMem::align_offset(dev, 0, orig_addr).
7. orig_addr += tlb_offset; alloc_size -= tlb_offset.
8. if size > alloc_size: WARN_ONCE "Buffer overflow"; size = alloc_size.
9. if PageHighMem(pfn_to_page(PFN_DOWN(orig_addr))):
   - per-page: kmap_local; memcpy(direction-selected); kunmap_local.
10. else: memcpy(direction-selected) between phys_to_virt(orig_addr) and vaddr_slot.

`IoTlbMem::find_slots(dev, orig_addr, size, alloc_align_mask) -> (pool, index)`:
1. start_cpu = raw_smp_processor_id().
2. /* Search defpool first, then mem->pools, then dev->dma_io_tlb_pools */
3. for pool in [defpool] + mem.pools.rcu_iter() + dev.dma_io_tlb_pools.rcu_iter():
   - index = IoTlbMem::search_area(dev, start_cpu, 0, pool, orig_addr, size, alloc_align_mask).
   - if index != -1: return (pool, index).
4. /* Per-CONFIG_SWIOTLB_DYNAMIC + can_grow: schedule grow + try transient pool */
5. if mem.can_grow:
   - schedule_work(&mem.dyn_alloc).
   - pool = IoTlbMem::alloc_pool(dev, nr_slots(size), nslabs, 1, phys_limit, GFP_NOWAIT).
   - if pool: list_add_rcu(&pool.node, &dev.dma_io_tlb_pools); return (pool, 0).
6. return (NULL, -1).

`IoTlbPool::search_area(dev, start_cpu, cpu_offset, pool, orig_addr, size, align_mask) -> index`:
1. area_idx = (start_cpu + cpu_offset) % pool.nareas.
2. let _g = pool.areas[area_idx].lock.lock_irqsave();
3. start_index = pool.areas[area_idx].index.
4. /* Walk slot indices from start_index forward, with alignment + boundary + min_align checks */
5. for i in start_index..(start_index + area_nslabs):
   - wrapped = wrap_area_index(pool, i % area_nslabs + area_idx * area_nslabs).
   - if pool.slots[wrapped].list >= nr_slots(size) ∧ aligned(wrapped, align_mask) ∧ within_boundary_mask(dev, orig_addr, wrapped):
     - reserve run; advance pool.areas[area_idx].index.
     - inc_used_and_hiwater(mem, nr_slots(size)).
     - return wrapped.
6. drop(_g); return -1.

`IoTlbMem::alloc_pool(dev, minslabs, nslabs, nareas, phys_limit, gfp) -> Option<KBox<IoTlbPool>>` (CONFIG_SWIOTLB_DYNAMIC):
1. while nslabs ≥ minslabs:
   - tlb = swiotlb_alloc_tlb(dev, nslabs << IO_TLB_SHIFT, phys_limit, gfp).
   - if tlb: break.
   - nslabs >>= 1.
2. if !tlb: return None.
3. pool = KBox::new_zeroed_in(KernelAlloc, gfp)?.
4. pool.slots = KBox::new_zero_slice(nslabs, gfp)?.
5. pool.areas = KBox::new_zero_slice(nareas, gfp)?.
6. IoTlbPool::init(&mut pool, virt_to_phys(tlb), nslabs, late=true, nareas).
7. pool.transient = true.
8. Some(pool).

`Swiotlb::rmem_device_init(rmem, dev) -> i32` (CONFIG_DMA_RESTRICTED_POOL):
1. nslabs = rmem.size >> IO_TLB_SHIFT; nareas = 1.
2. if PageHighMem(pfn_to_page(PHYS_PFN(rmem.base))): return -EINVAL.
3. if !rmem.priv:
   - mem = kzalloc(sizeof(IoTlbMem))?; pool = &mem.defpool.
   - pool.slots = kzalloc(nslabs * sizeof(IoTlbSlot))?; pool.areas = kzalloc(nareas * sizeof(IoTlbArea))?.
   - set_memory_decrypted(phys_to_virt(rmem.base), rmem.size >> PAGE_SHIFT).
   - IoTlbPool::init(pool, rmem.base, nslabs, late=false, nareas).
   - mem.force_bounce = true; mem.for_alloc = true.
   - add_mem_pool(mem, pool).
   - rmem.priv = mem; create debugfs files.
4. dev.dma_io_tlb_mem = rmem.priv.
5. return 0.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `slot_idx_no_oob` | OOB | per-`find_slots`: returned index satisfies `index + nr_slots(size) ≤ pool.nslabs`. |
| `aindex_no_oob` | OOB | per-`release_slots`: `aindex < pool.nareas` enforced via assert. |
| `slot_no_uaf` | UAF | per-slot: `orig_addr = INVALID_PHYS_ADDR` after release; reused slot writes fresh `orig_addr` under `area.lock`. |
| `bounce_size_bounded` | OOB | per-`bounce`: `size ≤ alloc_size` after `tlb_offset` adjustment via WARN_ONCE + clamp. |
| `area_lock_held_for_mutation` | INVARIANT | per-`search_area` / `release_slots`: `area.lock` held across slot list edits. |
| `pool_rcu_list_iter_safe` | INVARIANT | per-`find_pool` / `find_slots`: iterations over `mem.pools` / `dev.dma_io_tlb_pools` under `rcu_read_lock`. |
| `mem_encryption_force_bounce_obeyed` | INVARIANT | per-CC-guest: `force_bounce = true` implies every map goes through swiotlb. |
| `pad_slot_alignment_preserved` | INVARIANT | per-`map_single`: `tlb_addr & dma_get_min_align_mask(dev) == orig_addr & dma_get_min_align_mask(dev)`. |

### Layer 2: TLA+

`models/dma/swiotlb.tla`:
- Per-N CPU concurrent map + unmap + sync + dynamic-grow + restricted-pool attach + CC-guest force.
- Properties:
  - `safety_no_double_allocate` — no two concurrent maps return overlapping slot ranges.
  - `safety_no_double_free` — `release_slots` on a free slot is forbidden (`list != 0` precondition).
  - `safety_free_list_well_formed` — `slots[i].list` always counts contiguous free slots up to the next used slot or segment boundary.
  - `safety_pool_rcu_release` — `dyn_free` reachable only after `del_pool` + grace period.
  - `safety_force_bounce_funnels_cc_guest` — CC-guest detection implies `dma_map_*` reaches `swiotlb_map`.
  - `safety_restricted_pool_isolation` — restricted-pool device DMA addresses never escape `pool.start..pool.end`.
  - `liveness_per_map_completes_or_returns_error` — `map_single` returns ≠ `DMA_MAPPING_ERROR` within bounded retries, or returns `DMA_MAPPING_ERROR`.
  - `liveness_dyn_alloc_makes_progress` — exhausted pool + `can_grow` ⟹ `dyn_alloc` schedules a new pool within finite time.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `IoTlbMem::map_single` post: returned slot range owned exclusively; orig_addr recorded in each slot of the run | `IoTlbMem::map_single` |
| `IoTlbMem::unmap_single` post: slot range freed; orig_addr = INVALID_PHYS_ADDR; FROM_DEVICE bytes copied back if !SKIP_CPU_SYNC | `IoTlbMem::unmap_single` |
| `IoTlbPool::release_slots` post: free-list count = #contiguous free slots from index until segment boundary or next used slot | `IoTlbPool::release_slots` |
| `IoTlbMem::bounce` post: `size ≤ alloc_size - tlb_offset` after clamp | `IoTlbMem::bounce` |
| `area.used` invariant: equals count of slots with `orig_addr != INVALID_PHYS_ADDR` in `[aindex * area_nslabs, (aindex+1) * area_nslabs)` | `IoTlbArea` |
| `mem.nslabs` invariant: equals Σ pool.nslabs over all pools in `mem.pools` + defpool | `IoTlbMem` |
| `IoTlbMem::find_slots` post: returned (pool, index) satisfies `pool.slots[index].list ≥ nr_slots(size)` and alignment | `IoTlbMem::find_slots` |

### Layer 4: Verus/Creusot functional

`Per-map / per-sync / per-unmap round-trip`:
- `map_single(dev, addr, sz, TO_DEVICE)` → `unmap_single(dev, tlb, sz, TO_DEVICE)`:
  - device sees identical bytes as `addr[0..sz]` at start of map.
  - original `addr[0..sz]` unchanged through entire round-trip.
- `map_single(dev, addr, sz, FROM_DEVICE)` → device writes bytes B → `unmap_single(dev, tlb, sz, FROM_DEVICE)`:
  - post-unmap, `addr[0..sz] == B` (assuming `!SKIP_CPU_SYNC`).
- `sync_for_device` / `sync_for_cpu` between map and unmap update the appropriate side per direction.
- CC-guest invariant: pool pages are decrypted from boot; bytes visible to device are plaintext.

Encoded as Verus invariants on the abstract `(buffer, slot, direction)` triple per-`Documentation/core-api/swiotlb.rst`.

## Hardening

(Inherits row-1 features from `kernel/dma/00-overview.md` § Hardening.)

swiotlb-specific reinforcement:

- **Per-CC-guest forced-bounce default-on** — when SEV/SEV-ES/SEV-SNP/TDX detected at boot, every DMA bounced regardless of `dev->dma_mask`; defense against host-RAM leakage via direct DMA bypassing memory encryption.
- **Per-pool pages decrypted at init / re-encrypted at exit** — `set_memory_decrypted` on pool create, `set_memory_encrypted` on `swiotlb_exit`; defense against post-kdump plaintext-page reuse.
- **Per-NUMA-area lock + per-CPU start-area** — per-area `spinlock_t` + `area.index` rotating start + `start_cpu = raw_smp_processor_id()`; defense against single-global-lock contention on multi-socket workloads.
- **Per-mapping size cap** — `swiotlb_max_mapping_size(dev) = (IO_TLB_SEGSIZE << IO_TLB_SHIFT) - dma_get_min_align_mask(dev) - 1`; defense against single mapping starving the pool.
- **Per-`alloc_align_mask` upper bound** — `dev_WARN_ONCE(dev, alloc_align_mask > ~PAGE_MASK, …)`; defense against driver requesting alignment that makes max mapping infeasible.
- **Per-segment-boundary free-list merge** — `release_slots` merges only within `IO_TLB_SEGSIZE` boundary; defense against cross-segment alloc that would corrupt segment-relative offset math.
- **Per-slot `pad_slots` tracked head-only** — only head slot of an allocation carries `pad_slots`, preventing double-count on free.
- **Per-`DMA_ATTR_SKIP_CPU_SYNC` honored on unmap** — caller-opt-out of sync-back when caller has done its own; defense against double-memcpy.
- **Per-`DMA_ATTR_NO_WARN` honored on out-of-slots** — caller-opt-out of dmesg flood when caller fallback-retries; rate-limited otherwise.
- **Per-`swiotlb=force` / `noforce` / `=N` cmdline parsed strictly** — defense against silently ignoring operator override.
- **Per-restricted-pool `force_bounce + for_alloc` set** — DT `restricted-dma-pool` always uses bounce + direct-allocator path; defense against pool-bypass via direct mapping.
- **Per-CONFIG_SWIOTLB_DYNAMIC transient-pool RCU-free** — `swiotlb_dyn_free` via `call_rcu` after `del_pool`; defense against in-flight `find_pool` racing pool free.
- **Tracepoint `swiotlb_bounced` for every bounce** — operator can correlate bounce frequency to workload; defense against silent perf cliff.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- DMA API top-level dispatch (covered in `kernel/dma/mapping.md` Tier-3).
- DMA-direct path when bounce is not needed (covered in `kernel/dma/dma-direct.md` Tier-3).
- IOMMU-backed DMA path that may fall back to swiotlb (covered in `kernel/dma/dma-iommu.md` Tier-3).
- Per-device coherent SRAM scratchpad allocator (covered in `kernel/dma/coherent.md` Tier-3).
- CMA / DMA-contiguous allocator (covered separately if expanded).
- Per-arch `set_memory_decrypted` / `set_memory_encrypted` SEV-C-bit / TDX-SHARED-bit handling (covered in arch-specific Tier-3).
- 32-bit-only legacy paths.
- Implementation code.
