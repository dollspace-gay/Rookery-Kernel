# Tier-3: kernel/dma/direct.c — Direct (non-IOMMU) DMA API

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/dma/00-overview.md
upstream-paths:
  - kernel/dma/direct.c (~678 lines)
  - kernel/dma/direct.h
  - include/linux/dma-direct.h
  - include/linux/dma-map-ops.h
-->

## Summary

DMA-direct provides the **passthrough** (no-IOMMU) per-device DMA-mapping path: physical-addr == DMA-addr for devices with full system-DMA-mask. Per-`dma_alloc_attrs` allocates DMA-coherent buffer (via cma_alloc or alloc_pages); returns kvirt-addr + dma_handle (== phys). Per-`dma_map_page` for streaming-DMA simply returns phys-addr + applies dma_handle = phys - offset. Per-arch dma_addr_t conversion (e.g. 32-bit-limited devices need bounce-buffers from swiotlb). Per-cache-coherency: per-arch sync_for_{cpu, device}. Critical for: per-non-IOMMU bus (legacy PCI, embedded) device DMA.

This Tier-3 covers `direct.c` (~678 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `dma_direct_map_page()` | per-page streaming-map | `DmaDirect::map_page` |
| `dma_direct_unmap_page()` | per-page streaming-unmap | `DmaDirect::unmap_page` |
| `dma_direct_map_sg()` | per-sg streaming-map | `DmaDirect::map_sg` |
| `dma_direct_unmap_sg()` | per-sg streaming-unmap | `DmaDirect::unmap_sg` |
| `dma_direct_alloc()` | per-coherent alloc | `DmaDirect::alloc` |
| `dma_direct_free()` | per-coherent free | `DmaDirect::free` |
| `dma_direct_alloc_pages()` | per-page-only alloc | `DmaDirect::alloc_pages` |
| `dma_direct_free_pages()` | per-page-only free | `DmaDirect::free_pages` |
| `dma_direct_sync_single_for_device()` / `_cpu()` | per-cache sync | `DmaDirect::sync_*` |
| `dma_direct_can_mmap()` / `_mmap()` | per-mmap | `DmaDirect::can_mmap` / `mmap` |
| `dma_direct_get_sgtable()` | per-sg-table | `DmaDirect::get_sgtable` |
| `dma_direct_supported()` | per-(dev, mask) capability | `DmaDirect::supported` |
| `phys_to_dma()` / `dma_to_phys()` | per-arch conversion | macros |
| `dma_direct_need_sync()` | per-CPU/dev sync needed | `DmaDirect::need_sync` |

## Compatibility contract

REQ-1: Per-device dma_mask:
- dev.dma_mask: u64 bitmask of addressable bits.
- Per-32-bit-device: mask == 0xFFFFFFFF.
- Per-64-bit-device: mask == DMA_BIT_MASK(64).

REQ-2: dma_direct_alloc(dev, size, dma_handle, gfp, attrs):
- Allocate per-pages via __dma_direct_alloc_pages (which uses CMA, dma-pool, or alloc_pages).
- Per-coherent: pages.
- *dma_handle = phys_to_dma(dev, page_to_phys(pages)).
- Return virt address.

REQ-3: dma_direct_alloc_pages:
- if cma: cma_alloc.
- elif dma_pool: dma_pool_alloc.
- else: alloc_pages(gfp, order).
- Per-non-coherent: arch_sync_dma_for_device.

REQ-4: dma_direct_map_page:
- phys = page_to_phys(page) + offset.
- if phys > dev.dma_mask: -ENOMEM (or swiotlb-bounce).
- /* Per-cache: pre-sync */
- if !attrs & DMA_ATTR_SKIP_CPU_SYNC ∧ need_sync:
   - arch_sync_dma_for_device(phys, size, dir).
- return phys_to_dma(dev, phys).

REQ-5: dma_direct_unmap_page:
- phys = dma_to_phys(dev, dma_addr).
- /* Per-cache: post-sync */
- if !attrs & DMA_ATTR_SKIP_CPU_SYNC ∧ need_sync:
   - arch_sync_dma_for_cpu(phys, size, dir).

REQ-6: Per-streaming sync:
- dma_direct_sync_single_for_device: arch_sync_dma_for_device.
- dma_direct_sync_single_for_cpu: arch_sync_dma_for_cpu.

REQ-7: Per-coherent attribute:
- Coherent: per-arch may use uncached map (e.g. ARM); x86 typically cache-coherent.

REQ-8: Per-swiotlb fallback:
- If phys > dev.dma_mask: swiotlb-bounce (covered in `swiotlb.md`).

REQ-9: dma_direct_supported(dev, mask):
- Return mask >= min(dev.bus.max_phys_addr).

REQ-10: Per-CMA region:
- per-dev.contiguous = per-(start, size) CMA pool.
- Per-large-coherent alloc uses CMA.

REQ-11: Per-DMA_ATTR_*:
- DMA_ATTR_SKIP_CPU_SYNC: caller handles sync.
- DMA_ATTR_NO_WARN: no warning on alloc-fail.
- DMA_ATTR_WEAK_ORDERING: per-arch may relax.
- DMA_ATTR_FORCE_CONTIGUOUS: per-CMA mandatory.

## Acceptance Criteria

- [ ] AC-1: dma_alloc_coherent(64KB): page-aligned + dma_handle == phys (assuming no IOMMU).
- [ ] AC-2: dma_map_page(page, offset, size, DMA_TO_DEVICE): returns phys-addr.
- [ ] AC-3: Per-cache-non-coherent: sync_for_device called.
- [ ] AC-4: Per-32-bit-device on 64-bit-mem: phys > mask → swiotlb-bounce.
- [ ] AC-5: dma_free_coherent: pages freed.
- [ ] AC-6: dma_direct_supported with 64-bit mask: returns true on x86_64.
- [ ] AC-7: dma_map_sg: per-sg-entry mapped via map_page.
- [ ] AC-8: CMA-backed alloc: per-large contig-region used.
- [ ] AC-9: DMA_ATTR_SKIP_CPU_SYNC: arch_sync_dma_* not called.
- [ ] AC-10: dma_direct_mmap: per-page mmap to userspace.

## Architecture

`DmaDirect::map_page(dev, page, offset, size, dir, attrs) -> DmaAddrT`:
1. phys = page_to_phys(page) + offset.
2. if phys + size > dev.dma_mask:
   - /* swiotlb fallback */
   - return swiotlb_map(dev, phys, size, dir, attrs).
3. if !(attrs & DMA_ATTR_SKIP_CPU_SYNC) ∧ DmaDirect::need_sync(dev, dir):
   - arch_sync_dma_for_device(phys, size, dir).
4. return phys_to_dma(dev, phys).

`DmaDirect::unmap_page(dev, addr, size, dir, attrs)`:
1. phys = dma_to_phys(dev, addr).
2. if is_swiotlb_buffer(phys): swiotlb_unmap(dev, phys, size, dir, attrs); return.
3. if !(attrs & DMA_ATTR_SKIP_CPU_SYNC) ∧ DmaDirect::need_sync(dev, dir):
   - arch_sync_dma_for_cpu(phys, size, dir).

`DmaDirect::alloc(dev, size, dma_handle, gfp, attrs) -> *u8`:
1. order = get_order(size).
2. /* CMA-first */
3. if dma_alloc_from_contiguous_pool: page = cma_alloc(...).
4. elif is_swiotlb_active ∧ size > swiotlb threshold: page = swiotlb_alloc(...).
5. else: page = alloc_pages_node(dev_node(dev), gfp, order).
6. if !page: return NULL.
7. phys = page_to_phys(page).
8. *dma_handle = phys_to_dma(dev, phys).
9. if !dev_is_dma_coherent(dev):
   - vaddr = arch_dma_map_coherent(dev, phys, size).
10. return vaddr.

`DmaDirect::need_sync(dev, dir) -> bool`:
1. return !dev_is_dma_coherent(dev) ∧ !(dir == DMA_TO_DEVICE ∧ dev_is_write_only_cache).

`DmaDirect::supported(dev, mask) -> bool`:
1. return mask >= zone_dma32_bw_mask.

`DmaDirect::map_sg(dev, sg, nents, dir, attrs) -> i32`:
1. mapped = 0.
2. for_each_sg(sg, s, nents, i):
   - s.dma_address = DmaDirect::map_page(dev, sg_page(s), s.offset, s.length, dir, attrs).
   - if (s.dma_address == DMA_MAPPING_ERROR): bail.
   - mapped++.
3. return mapped.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `phys_le_dma_mask` | INVARIANT | per-map: phys + size ≤ dev.dma_mask (or swiotlb-fallback). |
| `dma_handle_eq_phys_to_dma` | INVARIANT | per-alloc: *dma_handle == phys_to_dma(dev, page_to_phys(page)). |
| `sync_called_per_dir` | INVARIANT | per-map non-coherent ∧ !SKIP_CPU_SYNC: arch_sync called. |
| `unmap_balanced` | INVARIANT | per-map paired with unmap. |

### Layer 2: TLA+

`kernel/dma/dma_direct.tla`:
- Per-map/unmap + per-coherent alloc/free + cache-sync.
- Properties:
  - `safety_dma_handle_within_mask` — per-map success: dma_handle ≤ dev.dma_mask.
  - `safety_no_double_free` — per-coherent alloc balanced with free.
  - `liveness_swiotlb_fallback_when_above_mask` — per-32-bit-device phys > mask: swiotlb-bounce.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `DmaDirect::map_page` post: returned dma_addr within dev mask | `DmaDirect::map_page` |
| `DmaDirect::alloc` post: *dma_handle valid; pages allocated | `DmaDirect::alloc` |
| `DmaDirect::map_sg` post: per-sg entry mapped or error | `DmaDirect::map_sg` |
| `DmaDirect::need_sync` post: returns iff non-coherent | `DmaDirect::need_sync` |

### Layer 4: Verus/Creusot functional

`Per-device DMA-direct: phys-addr == dma-addr (modulo per-arch offset); per-non-IOMMU device DMA path` semantic equivalence: per-DMA API doc.

## Hardening

(Inherits row-1 features from `kernel/dma/00-overview.md` § Hardening.)

DMA-direct reinforcement:

- **Per-dma_mask enforced** — defense against per-32-bit-device DMA to high-mem.
- **Per-swiotlb fallback** — defense against per-out-of-mask DMA.
- **Per-CMA backing for large coherent** — defense against per-fragmentation alloc-fail.
- **Per-cache sync per-dir** — defense against per-non-coherent stale-CPU-cache data.
- **Per-DMA_ATTR_SKIP_CPU_SYNC explicit** — defense against per-driver forgetting sync.
- **Per-dev_is_dma_coherent test** — defense against per-arch confusion.
- **Per-attrs validated** — defense against per-malformed attrs.
- **Per-coherent-alloc respects gfp** — defense against per-atomic-context sleeping.
- **Per-32-bit-device on 64-bit-mem warning** — defense against per-config silent bounce-waste.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — `dma_direct_alloc` returns a kernel virtual address that drivers may eventually mmap to userspace through `dma_direct_mmap`; whitelist the allocated extent so a driver bug cannot use `remap_pfn_range` to expose adjacent kernel pages across the user boundary.
- **PAX_KERNEXEC** — `dma_direct_ops` (`dma_direct_alloc`, `_free`, `_map_page`, `_unmap_page`, `_sync_single_for_device`, `_sync_single_for_cpu`) is the fallback dma_map_ops vector; enforce W^X on the ops page and refuse rwx for the per-arch bounce trampoline.
- **PAX_RANDKSTACK** — `dma_direct_map_page` runs from network/block driver IRQ-disabled context at user-driven packet/bio depth; randomized kstack offset disrupts ROP through the `swiotlb_tbl_map_single` bounce fallback.
- **PAX_REFCOUNT** — `dev->dma_pools` and `swiotlb` slot accounting use refcount_t with saturating overflow; defense against driver-storm underflow racing dma_direct_free against in-flight map.
- **PAX_MEMORY_SANITIZE** — `dma_direct_free` and `swiotlb_tbl_unmap_single` must zero the bounce buffer (and the original page if `DMA_ATTR_NO_KERNEL_MAPPING`) before returning to the allocator; defense against post-free residual leaking DMA content (network packets, disk blocks) to next allocator user.
- **PAX_UDEREF** — `dma_direct_mmap` keeps SMAP/PAN engaged across the user vma fault → `remap_pfn_range` chain; never enable AC outside the page-table update.
- **PAX_RAP/kCFI** — `dma_map_ops.*` indirect calls (alloc/free/map_page/unmap_page/sync_for_device/sync_for_cpu) must verify kCFI tag at every dispatch from `dma_map_single` / `dma_unmap_single` / `dma_sync_*`.
- **GRKERNSEC_HIDESYM** — `swiotlb_print_info` and `/proc/swiotlb` (if exposed) must elide the bounce-buffer kernel virtual address from non-CAP_SYSLOG readers; expose only slot count and high-water mark.
- **GRKERNSEC_DMESG** — `WARN_ON` on `dma_direct_map_page` failure, 32-bit-device-on-64-bit-mem mismatch, and `swiotlb_tbl_map_single` slot exhaustion must not splat raw bounce-buffer pointers into dmesg readable by non-CAP_SYSLOG.
- **Bounce-buffer PAX_MEMORY_SANITIZE strict** — every `swiotlb_tbl_unmap_single` MUST zero the bounce slot before marking it free even when the direction is DMA_TO_DEVICE; defense against driver bug leaving stale RAM in a slot reused for DMA_FROM_DEVICE.
- **swiotlb fallback bounded** — `swiotlb_max_size` must be enforced so a single oversized map request cannot consume the entire swiotlb pool and starve sibling devices.
- **`dma_direct_alloc` GFP_ATOMIC discipline** — atomic-context allocations must use the per-device `atomic_pool`; refusing to sleep is mandatory and must not silently fall back to a sleeping path (which would deadlock under RCU read-side).
- **32-bit-device on 64-bit-mem warning** — already mitigated upstream; harden by upgrading WARN to `dev_err_once` + iommu-prefer fallback so a misconfigured driver is steered to swiotlb explicitly rather than silently bouncing.
- **DMA-attr validation** — `attrs` mask must be validated against a fixed allowlist (`DMA_ATTR_*` known set); defense against future-flag laundering granting unintended cache behaviour.
- **Rationale** — dma-direct is the universal fallback DMA path for devices without an IOMMU; a single bounce-buffer sanitize miss leaks cross-driver content (a network packet bounce slot reused for a disk read exposes packet data to the disk driver). The grsec regime forces an attacker to defeat W^X on dma_map_ops + kCFI on map/unmap dispatch + strict bounce sanitize + swiotlb pool bound + GFP_ATOMIC discipline simultaneously before reaching a cross-device data-leak primitive.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- kernel/dma/00-overview (Tier-2)
- kernel/dma/swiotlb.c (covered in `swiotlb.md` Tier-3)
- kernel/dma/iommu (covered in `dma-iommu.md` Tier-3)
- kernel/dma/contiguous.c (CMA; covered separately)
- Per-arch dma_addr_t conversion (covered in `arch/x86/00-overview.md` Tier-3)
- Implementation code
