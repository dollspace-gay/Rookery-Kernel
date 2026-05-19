# Tier-3: kernel/dma/mapping.c — DMA mapping facade

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/dma/00-overview.md
upstream-paths:
  - kernel/dma/mapping.c (~1025 lines)
  - include/linux/dma-mapping.h
  - include/linux/dma-map-ops.h
  - include/linux/dma-direct.h
  - include/trace/events/dma.h
-->

## Summary

The **DMA mapping facade** is the arch-independent dispatch layer between driver-visible `dma_*` API entry points and one of three backends: `dma-direct` (identity / offset map), `iommu-dma` (IOMMU-mediated map, via `use_dma_iommu(dev)`), or per-bus `dma_map_ops` (per-device override; `get_dma_ops(dev)`). Per-`struct dma_map_ops`: `alloc`, `free`, `map_phys`, `unmap_phys`, `map_sg`, `unmap_sg`, `sync_*`, `get_sgtable`, `mmap`, `alloc_pages_op`, `free_pages`, `get_required_mask`, `dma_supported`, `max_mapping_size`, `opt_mapping_size`, `get_merge_boundary`. Per-`dma_map_phys` / `dma_unmap_phys` / `dma_map_sg_attrs` / `dma_unmap_sg_attrs`: streaming-DMA buffer handoff. Per-`dma_alloc_attrs` / `dma_free_attrs`: coherent-buffer alloc. Per-`dma_alloc_from_dev_coherent`: per-device coherent pool (struct `device.dma_coherent_mem`) checked first; CMA fallback inside direct backend. Per-`dma_sync_*`: cache-flush bracket around DMA. Per-`dma_addressing_limited`: detect device mask < RAM-required mask (bounce buffering trigger). Critical for: every storage / NIC / GPU / display driver that hands a CPU buffer to a DMA-capable device.

This Tier-3 covers `kernel/dma/mapping.c` (~1025 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct dma_map_ops` | per-device-ops vtable | `DmaMapOps` |
| `get_dma_ops(dev)` | per-device dispatch lookup | `Dma::ops_for` |
| `use_dma_iommu(dev)` | per-device IOMMU active | `Dma::uses_iommu` |
| `dma_go_direct()` | per-dev direct-path predicate | `Dma::go_direct` |
| `dma_alloc_direct()` / `dma_map_direct()` | per-coherent / per-stream direct | `Dma::alloc_direct` / `Dma::map_direct` |
| `dma_map_phys()` | per-phys-addr streaming map | `Dma::map_phys` |
| `dma_unmap_phys()` | per-phys-addr streaming unmap | `Dma::unmap_phys` |
| `dma_map_page_attrs()` | per-page streaming map | `Dma::map_page_attrs` |
| `dma_unmap_page_attrs()` | per-page streaming unmap | `Dma::unmap_page_attrs` |
| `dma_map_sg_attrs()` / `dma_map_sgtable()` | per-sg streaming map | `Dma::map_sg_attrs` / `Dma::map_sgtable` |
| `dma_unmap_sg_attrs()` | per-sg streaming unmap | `Dma::unmap_sg_attrs` |
| `dma_map_resource()` / `dma_unmap_resource()` | per-MMIO map | `Dma::map_resource` / `unmap_resource` |
| `__dma_sync_single_for_cpu()` / `_for_device()` | per-cache-flush bracket | `Dma::sync_single_for_cpu` / `_for_device` |
| `__dma_sync_sg_for_cpu()` / `_for_device()` | per-sg cache-flush bracket | `Dma::sync_sg_for_cpu` / `_for_device` |
| `__dma_need_sync()` | per-addr need-sync probe | `Dma::need_sync` |
| `dma_need_unmap()` | per-dev need-unmap probe | `Dma::need_unmap` |
| `dma_setup_need_sync()` | per-dev sync-skip setup | `Dma::setup_need_sync` |
| `dma_alloc_attrs()` / `dma_free_attrs()` | per-coherent alloc / free | `Dma::alloc_attrs` / `free_attrs` |
| `dma_alloc_pages()` / `dma_free_pages()` | per-page non-coherent | `Dma::alloc_pages` / `free_pages` |
| `dma_alloc_noncontiguous()` / `dma_free_noncontiguous()` | per-noncontig sgt | `Dma::alloc_noncontiguous` / `free_noncontiguous` |
| `dma_vmap_noncontiguous()` / `dma_vunmap_noncontiguous()` | per-vmap | `Dma::vmap_noncontiguous` / `vunmap_noncontiguous` |
| `dma_mmap_attrs()` / `dma_can_mmap()` | per-userspace mmap | `Dma::mmap_attrs` / `can_mmap` |
| `dma_get_sgtable_attrs()` | per-coherent → sgt export | `Dma::get_sgtable_attrs` |
| `dma_get_required_mask()` | per-dev RAM-coverage mask | `Dma::get_required_mask` |
| `dma_set_mask()` / `dma_set_coherent_mask()` | per-dev mask install | `Dma::set_mask` / `set_coherent_mask` |
| `dma_supported()` | per-mask supportability | `Dma::supported` |
| `dma_addressing_limited()` | per-dev limited probe | `Dma::addressing_limited` |
| `dma_max_mapping_size()` / `dma_opt_mapping_size()` | per-dev size caps | `Dma::max_mapping_size` / `opt_mapping_size` |
| `dma_get_merge_boundary()` | per-dev SG merge boundary | `Dma::get_merge_boundary` |
| `dma_pci_p2pdma_supported()` | per-dev P2PDMA capability | `Dma::pci_p2pdma_supported` |
| `dmam_alloc_attrs()` / `dmam_free_coherent()` | per-devres managed | `Dma::devres_alloc_attrs` / `devres_free_coherent` |
| `dma_alloc_from_dev_coherent()` | per-dev coherent-pool alloc | `Dma::alloc_from_dev_coherent` |
| `dma_release_from_dev_coherent()` | per-dev coherent-pool free | `Dma::release_from_dev_coherent` |
| `dma_pgprot()` | per-arch coherent pgprot | `Dma::pgprot` |
| `arch_dma_map_phys_direct()` / `arch_dma_alloc_direct()` | per-arch override | `ArchDma::*_direct` |

## Compatibility contract

REQ-1: struct dma_map_ops:
- alloc: per-coherent alloc.
- free: per-coherent free.
- alloc_pages_op: per-page-alloc.
- free_pages: per-page-free.
- map_phys: per-phys-addr stream map.
- unmap_phys: per-phys-addr stream unmap.
- map_sg: per-sg stream map.
- unmap_sg: per-sg stream unmap.
- sync_single_for_cpu / _for_device: per-cache-flush single.
- sync_sg_for_cpu / _for_device: per-cache-flush sg.
- get_sgtable: per-coherent → sgt.
- mmap: per-userspace mmap.
- get_required_mask: per-RAM-coverage probe.
- dma_supported: per-mask probe.
- max_mapping_size: per-cap (default SIZE_MAX).
- opt_mapping_size: per-pref-cap (default SIZE_MAX).
- get_merge_boundary: per-SG merge step.

REQ-2: Backend dispatch order (per facade call):
- Coherent path: dma_alloc_from_dev_coherent (per-device pool) → if dma_alloc_direct ∨ arch_dma_alloc_direct: dma_direct_alloc (with CMA fallback inside) → elif use_dma_iommu: iommu_dma_alloc → elif ops->alloc: ops->alloc → else NULL.
- Streaming path: if dma_map_direct ∨ arch_dma_map_phys_direct: dma_direct_map_phys → elif is_cc_shared: DMA_MAPPING_ERROR → elif use_dma_iommu: iommu_dma_map_phys → elif ops->map_phys: ops->map_phys.
- Sync path: if dma_map_direct: dma_direct_sync_* → elif use_dma_iommu: iommu_dma_sync_* → elif ops->sync_*: ops->sync_*.

REQ-3: dma_go_direct(dev, mask, ops):
- if use_dma_iommu(dev): return false.
- if !ops: return true.
- if CONFIG_DMA_OPS_BYPASS ∧ dev.dma_ops_bypass:
  - return min_not_zero(mask, dev.bus_dma_limit) >= dma_direct_get_required_mask(dev).
- return false.

REQ-4: dma_map_phys(dev, phys, size, dir, attrs) -> dma_addr_t:
- BUG_ON(!valid_dma_direction(dir)).
- if !dev.dma_mask: WARN_ON_ONCE → DMA_MAPPING_ERROR.
- if !dev_is_dma_coherent ∧ (attrs & DMA_ATTR_REQUIRE_COHERENT): DMA_MAPPING_ERROR.
- /* Direct path */
- if dma_map_direct(dev, ops) ∨ (!is_mmio ∧ !is_cc_shared ∧ arch_dma_map_phys_direct(dev, phys + size)):
  - addr = dma_direct_map_phys(dev, phys, size, dir, attrs, true).
- elif is_cc_shared: return DMA_MAPPING_ERROR.
- elif use_dma_iommu(dev): addr = iommu_dma_map_phys(dev, phys, size, dir, attrs).
- elif ops.map_phys: addr = ops.map_phys(dev, phys, size, dir, attrs).
- if !is_mmio: kmsan_handle_dma(phys, size, dir).
- trace_dma_map_phys + debug_dma_map_phys.
- return addr.

REQ-5: dma_map_page_attrs(dev, page, offset, size, dir, attrs):
- if attrs & DMA_ATTR_MMIO: return DMA_MAPPING_ERROR.
- if CONFIG_DMA_API_DEBUG ∧ is_zone_device_page(page): WARN → DMA_MAPPING_ERROR.
- phys = page_to_phys(page) + offset.
- return dma_map_phys(dev, phys, size, dir, attrs).

REQ-6: dma_unmap_page_attrs:
- if attrs & DMA_ATTR_MMIO: return.
- dma_unmap_phys(dev, addr, size, dir, attrs).

REQ-7: dma_map_sg_attrs / dma_map_sgtable:
- /* Returns mapped-nents ≥ 0; errors mapped to 0 for legacy callers */
- if !dev_is_dma_coherent ∧ (attrs & DMA_ATTR_REQUIRE_COHERENT): return -EOPNOTSUPP.
- if !dev.dma_mask: WARN → return 0.
- if dma_map_direct ∨ arch_dma_map_sg_direct: ents = dma_direct_map_sg.
- elif use_dma_iommu: ents = iommu_dma_map_sg.
- else: ents = ops.map_sg.
- if ents > 0: kmsan_handle_dma_sg + trace + debug.
- else if ents ∉ {-EINVAL, -ENOMEM, -EIO, -EREMOTEIO}: WARN → return -EIO.
- return ents.

REQ-8: dma_alloc_attrs(dev, size, dma_handle, flag, attrs):
- WARN_ON_ONCE(!dev.coherent_dma_mask).
- if flag & __GFP_COMP: WARN → return NULL.
- /* Per-device coherent pool short-circuit */
- if dma_alloc_from_dev_coherent(dev, size, dma_handle, &cpu_addr): trace → return cpu_addr.
- /* Zone hints stripped */
- flag &= ~(__GFP_DMA | __GFP_DMA32 | __GFP_HIGHMEM).
- if dma_alloc_direct ∨ arch_dma_alloc_direct: cpu_addr = dma_direct_alloc.
- elif use_dma_iommu: cpu_addr = iommu_dma_alloc.
- elif ops.alloc: cpu_addr = ops.alloc.
- else: trace → return NULL.
- trace + debug.
- return cpu_addr.

REQ-9: dma_free_attrs(dev, size, cpu_addr, dma_handle, attrs):
- if dma_release_from_dev_coherent: return.
- WARN_ON(irqs_disabled()) /* vunmap may sleep on noncoherent */
- trace_dma_free.
- if !cpu_addr: return.
- debug_dma_free_coherent.
- if dma_alloc_direct ∨ arch_dma_free_direct: dma_direct_free.
- elif use_dma_iommu: iommu_dma_free.
- elif ops.free: ops.free.

REQ-10: dma_alloc_pages / dma_free_pages:
- WARN_ON_ONCE(!dev.coherent_dma_mask).
- WARN_ON_ONCE gfp & (__GFP_DMA | __GFP_DMA32 | __GFP_HIGHMEM | __GFP_COMP).
- size = PAGE_ALIGN(size).
- direct: dma_direct_alloc_pages.
- iommu: dma_common_alloc_pages.
- ops: ops.alloc_pages_op.
- free: symmetric dispatch.

REQ-11: dma_alloc_noncontiguous / dma_free_noncontiguous:
- attrs only allow DMA_ATTR_ALLOC_SINGLE_PAGES.
- gfp & __GFP_COMP: NULL.
- iommu: iommu_dma_alloc_noncontiguous (returns sg_table).
- direct: alloc_single_sgt (single PAGE_ALIGN(size) page wrapped in sgt).
- sgt.nents = 1; trace; debug_dma_map_sg.
- free: iommu_dma_free_noncontiguous or free_single_sgt.

REQ-12: dma_vmap_noncontiguous / dma_vunmap_noncontiguous / dma_mmap_noncontiguous:
- iommu: iommu_dma_vmap_noncontiguous (real vmap).
- direct: page_address(sg_page(sgt.sgl)) (single-page → already mapped).
- mmap iommu: iommu_dma_mmap_noncontiguous.
- mmap direct: dma_mmap_pages.

REQ-13: dma_get_required_mask:
- direct: dma_direct_get_required_mask.
- iommu: DMA_BIT_MASK(32).
- ops.get_required_mask: ops.get_required_mask.
- else: DMA_BIT_MASK(32).

REQ-14: dma_set_mask / dma_set_coherent_mask:
- mask = (dma_addr_t)mask /* truncate */.
- if !dma_supported(dev, mask): -EIO.
- arch_dma_set_mask + write dev.dma_mask / dev.coherent_dma_mask.
- dma_setup_need_sync(dev) [for dma_set_mask].

REQ-15: dma_supported:
- if use_dma_iommu: WARN_ON(ops) → return true.
- if ops: if !ops.dma_supported: true; else ops.dma_supported(dev, mask).
- else: dma_direct_supported(dev, mask).

REQ-16: dma_addressing_limited:
- internal __dma_addressing_limited:
  - if min_not_zero(dma_get_mask(dev), dev.bus_dma_limit) < dma_get_required_mask(dev): true.
  - if ops ∨ use_dma_iommu: false.
  - else: !dma_direct_all_ram_mapped(dev).
- if limited: dev_dbg + true; else false.

REQ-17: dma_max_mapping_size / dma_opt_mapping_size / dma_get_merge_boundary:
- max: direct → dma_direct_max_mapping_size; iommu → iommu_dma_max_mapping_size; ops.max_mapping_size; default SIZE_MAX.
- opt: iommu → iommu_dma_opt_mapping_size; ops.opt_mapping_size; min(max, opt).
- merge: iommu → iommu_dma_get_merge_boundary; ops.get_merge_boundary; default 0 (no merge).

REQ-18: dma_setup_need_sync (CONFIG_DMA_NEED_SYNC):
- direct ∨ iommu: dev.dma_skip_sync = dev_is_dma_coherent(dev).
- ops with no sync ops: dev.dma_skip_sync = true.
- else: dev.dma_skip_sync = false.

REQ-19: Managed DMA (devres) — dmam_alloc_attrs / dmam_free_coherent:
- dmam_alloc_attrs: devres_alloc(dmam_release) + dma_alloc_attrs; on driver detach dmam_release runs dma_free_attrs.
- dmam_free_coherent: devres_destroy(dmam_release, dmam_match) + dma_free_coherent.
- struct dma_devres: {size, vaddr, dma_handle, attrs}.

REQ-20: dma_get_sgtable_attrs / dma_can_mmap / dma_mmap_attrs:
- get_sgtable: direct → dma_direct_get_sgtable; iommu → iommu_dma_get_sgtable; ops.get_sgtable; -ENXIO if none.
- can_mmap: direct → dma_direct_can_mmap; iommu → true; ops.mmap != NULL.
- mmap: direct → dma_direct_mmap; iommu → iommu_dma_mmap; ops.mmap; -ENXIO if none.

REQ-21: dma_pgprot (CONFIG_MMU):
- if dev_is_dma_coherent: return prot.
- if attrs & DMA_ATTR_WRITE_COMBINE: pgprot_writecombine(prot).
- else: pgprot_dmacoherent(prot).

REQ-22: dma_pci_p2pdma_supported:
- /* P2PDMA only on direct + default IOMMU paths, regardless of dma_ops_bypass */
- return !ops.

## Acceptance Criteria

- [ ] AC-1: get_dma_ops returns dev.dma_ops or NULL; NULL ⟹ direct.
- [ ] AC-2: use_dma_iommu(dev) ⟹ all map / unmap / sync / alloc route to iommu_dma_*.
- [ ] AC-3: ops & !use_dma_iommu & !dma_ops_bypass-direct ⟹ all calls route to ops.*.
- [ ] AC-4: !ops & !iommu ⟹ all calls route to dma_direct_*.
- [ ] AC-5: dma_alloc_from_dev_coherent succeeds ⟹ dma_alloc_attrs short-circuits to per-dev pool.
- [ ] AC-6: dma_alloc_attrs with __GFP_COMP returns NULL and WARNs.
- [ ] AC-7: dma_free_attrs in IRQ context WARNs (vunmap-sleep risk).
- [ ] AC-8: dma_map_phys with DMA_ATTR_REQUIRE_COHERENT on non-coherent dev returns DMA_MAPPING_ERROR.
- [ ] AC-9: dma_map_page_attrs with DMA_ATTR_MMIO returns DMA_MAPPING_ERROR (use dma_map_resource instead).
- [ ] AC-10: dma_map_resource passes DMA_ATTR_MMIO through and rejects pfn_valid phys when DEBUG.
- [ ] AC-11: dma_map_sg_attrs returns 0 on error (legacy contract); dma_map_sgtable returns negative.
- [ ] AC-12: dma_set_mask validates via dma_supported; failure ⟹ -EIO, no write.
- [ ] AC-13: dma_addressing_limited true ⇔ dev mask < required, and not iommu/ops-managed-with-full-RAM.
- [ ] AC-14: dma_max_mapping_size default SIZE_MAX (no cap); iommu caps; direct caps via SWIOTLB.
- [ ] AC-15: dma_alloc_noncontiguous rejects attrs ∉ {0, DMA_ATTR_ALLOC_SINGLE_PAGES}.
- [ ] AC-16: dma_pci_p2pdma_supported true ⇔ ops == NULL.
- [ ] AC-17: dmam_alloc_attrs survives driver-detach: backing freed via devres callback.

## Architecture

```
trait DmaMapOps {
  fn alloc(dev: &Device, size: usize, dma_handle: &mut dma_addr_t, gfp: Gfp, attrs: u64) -> Option<*mut u8>;
  fn free(dev: &Device, size: usize, vaddr: *mut u8, dma_handle: dma_addr_t, attrs: u64);
  fn alloc_pages_op(dev: &Device, size: usize, dma_handle: &mut dma_addr_t, dir: DmaDir, gfp: Gfp) -> Option<*mut Page>;
  fn free_pages(dev: &Device, size: usize, page: *mut Page, dma_handle: dma_addr_t, dir: DmaDir);
  fn map_phys(dev: &Device, phys: phys_addr_t, size: usize, dir: DmaDir, attrs: u64) -> dma_addr_t;
  fn unmap_phys(dev: &Device, addr: dma_addr_t, size: usize, dir: DmaDir, attrs: u64);
  fn map_sg(dev: &Device, sg: *mut Scatterlist, nents: i32, dir: DmaDir, attrs: u64) -> i32;
  fn unmap_sg(dev: &Device, sg: *mut Scatterlist, nents: i32, dir: DmaDir, attrs: u64);
  fn sync_single_for_cpu(dev: &Device, addr: dma_addr_t, size: usize, dir: DmaDir);
  fn sync_single_for_device(dev: &Device, addr: dma_addr_t, size: usize, dir: DmaDir);
  fn sync_sg_for_cpu(dev: &Device, sg: *mut Scatterlist, nents: i32, dir: DmaDir);
  fn sync_sg_for_device(dev: &Device, sg: *mut Scatterlist, nents: i32, dir: DmaDir);
  fn get_sgtable(dev: &Device, sgt: *mut SgTable, cpu_addr: *mut u8, dma_addr: dma_addr_t, size: usize, attrs: u64) -> i32;
  fn mmap(dev: &Device, vma: *mut VmAreaStruct, cpu_addr: *mut u8, dma_addr: dma_addr_t, size: usize, attrs: u64) -> i32;
  fn get_required_mask(dev: &Device) -> u64;
  fn dma_supported(dev: &Device, mask: u64) -> bool;
  fn max_mapping_size(dev: &Device) -> usize;
  fn opt_mapping_size() -> usize;
  fn get_merge_boundary(dev: &Device) -> u64;
}
```

```
struct DmaDevres {
  size:       usize,
  vaddr:      *mut u8,
  dma_handle: dma_addr_t,
  attrs:      u64,
}
```

`Dma::ops_for(dev) -> Option<&'static dyn DmaMapOps>`:
1. Returns dev.dma_ops if set, else None (= direct).

`Dma::uses_iommu(dev) -> bool`:
1. CONFIG_IOMMU_DMA-gated; returns true iff dev has iommu-attached domain (non-passthrough).

`Dma::go_direct(dev, mask, ops) -> bool`:
1. if Dma::uses_iommu(dev): return false.
2. if ops.is_none(): return true.
3. if CONFIG_DMA_OPS_BYPASS ∧ dev.dma_ops_bypass:
   - return min_not_zero(mask, dev.bus_dma_limit) >= DmaDirect::get_required_mask(dev).
4. return false.

`Dma::map_phys(dev, phys, size, dir, attrs) -> dma_addr_t`:
1. ops = Dma::ops_for(dev).
2. is_mmio = attrs & DMA_ATTR_MMIO.
3. is_cc_shared = attrs & DMA_ATTR_CC_SHARED.
4. assert(valid_dma_direction(dir)).
5. if !dev.dma_mask: warn_once → return DMA_MAPPING_ERROR.
6. if !dev_is_dma_coherent(dev) ∧ (attrs & DMA_ATTR_REQUIRE_COHERENT): return DMA_MAPPING_ERROR.
7. let addr;
8. if Dma::go_direct(dev, *dev.dma_mask, ops) ∨ (!is_mmio ∧ !is_cc_shared ∧ ArchDma::map_phys_direct(dev, phys + size)):
   - addr = DmaDirect::map_phys(dev, phys, size, dir, attrs, true).
9. else if is_cc_shared: return DMA_MAPPING_ERROR.
10. else if Dma::uses_iommu(dev): addr = IommuDma::map_phys(dev, phys, size, dir, attrs).
11. else if ops.map_phys: addr = ops.map_phys(dev, phys, size, dir, attrs).
12. else: addr = DMA_MAPPING_ERROR.
13. if !is_mmio: kmsan_handle_dma(phys, size, dir).
14. trace + debug.
15. return addr.

`Dma::map_page_attrs(dev, page, offset, size, dir, attrs)`:
1. if attrs & DMA_ATTR_MMIO: return DMA_MAPPING_ERROR.
2. if cfg(DMA_API_DEBUG) ∧ is_zone_device_page(page): warn_once → return DMA_MAPPING_ERROR.
3. phys = page_to_phys(page) + offset.
4. return Dma::map_phys(dev, phys, size, dir, attrs).

`Dma::map_sg_inner(dev, sg, nents, dir, attrs) -> i32`:
1. ops = Dma::ops_for(dev).
2. assert(valid_dma_direction(dir)).
3. if !dev_is_dma_coherent(dev) ∧ (attrs & DMA_ATTR_REQUIRE_COHERENT): return -EOPNOTSUPP.
4. if !dev.dma_mask: warn_once → return 0.
5. let ents;
6. if Dma::go_direct(dev, *dev.dma_mask, ops) ∨ ArchDma::map_sg_direct(dev, sg, nents):
   - ents = DmaDirect::map_sg(dev, sg, nents, dir, attrs).
7. else if Dma::uses_iommu(dev): ents = IommuDma::map_sg(dev, sg, nents, dir, attrs).
8. else: ents = ops.map_sg(dev, sg, nents, dir, attrs).
9. if ents > 0: kmsan_handle_dma_sg + trace + debug.
10. else if ents ∉ {-EINVAL, -ENOMEM, -EIO, -EREMOTEIO}: warn_once + trace_err → return -EIO.
11. return ents.

`Dma::map_sg_attrs(dev, sg, nents, dir, attrs) -> u32`:
1. let r = Dma::map_sg_inner(dev, sg, nents, dir, attrs).
2. if r < 0: return 0.
3. return r as u32.

`Dma::map_sgtable(dev, sgt, dir, attrs) -> i32`:
1. let n = Dma::map_sg_inner(dev, sgt.sgl, sgt.orig_nents, dir, attrs).
2. if n < 0: return n.
3. sgt.nents = n.
4. return 0.

`Dma::alloc_attrs(dev, size, dma_handle, gfp, attrs) -> Option<*mut u8>`:
1. ops = Dma::ops_for(dev).
2. warn_once_if(!dev.coherent_dma_mask).
3. if gfp & __GFP_COMP: warn_once → return None.
4. /* Per-device coherent pool first */
5. if Dma::alloc_from_dev_coherent(dev, size, dma_handle, &mut cpu_addr): trace → return Some(cpu_addr).
6. gfp &= !(__GFP_DMA | __GFP_DMA32 | __GFP_HIGHMEM).
7. if Dma::go_direct(dev, dev.coherent_dma_mask, ops) ∨ ArchDma::alloc_direct(dev):
   - cpu_addr = DmaDirect::alloc(dev, size, dma_handle, gfp, attrs).
8. else if Dma::uses_iommu(dev): cpu_addr = IommuDma::alloc(dev, size, dma_handle, gfp, attrs).
9. else if ops.alloc: cpu_addr = ops.alloc(dev, size, dma_handle, gfp, attrs).
10. else: trace → return None.
11. trace + debug.
12. return Some(cpu_addr).

`Dma::free_attrs(dev, size, cpu_addr, dma_handle, attrs)`:
1. ops = Dma::ops_for(dev).
2. if Dma::release_from_dev_coherent(dev, get_order(size), cpu_addr): return.
3. warn_if(irqs_disabled()).
4. trace_dma_free.
5. if cpu_addr.is_null(): return.
6. debug_dma_free_coherent.
7. if Dma::go_direct(dev, dev.coherent_dma_mask, ops) ∨ ArchDma::free_direct(dev, dma_handle): DmaDirect::free(...).
8. else if Dma::uses_iommu(dev): IommuDma::free(...).
9. else if ops.free: ops.free(...).

`Dma::sync_single_for_cpu(dev, addr, size, dir)`:
1. ops = Dma::ops_for(dev).
2. assert(valid_dma_direction(dir)).
3. if Dma::go_direct(dev, *dev.dma_mask, ops): DmaDirect::sync_single_for_cpu(dev, addr, size, dir, true).
4. else if Dma::uses_iommu(dev): IommuDma::sync_single_for_cpu(dev, addr, size, dir).
5. else if ops.sync_single_for_cpu: ops.sync_single_for_cpu(dev, addr, size, dir).
6. trace + debug.

(Symmetric for `sync_single_for_device`, `sync_sg_for_cpu`, `sync_sg_for_device`.)

`Dma::need_sync(dev, dma_addr) -> bool`:
1. ops = Dma::ops_for(dev).
2. if Dma::go_direct(dev, *dev.dma_mask, ops): return DmaDirect::need_sync(dev, dma_addr).
3. return true.

`Dma::need_unmap(dev) -> bool`:
1. if !Dma::go_direct(dev, *dev.dma_mask, ops): return true.
2. if !dev.dma_skip_sync: return true.
3. return cfg(DMA_API_DEBUG).

`Dma::setup_need_sync(dev)` (CONFIG_DMA_NEED_SYNC):
1. ops = Dma::ops_for(dev).
2. if Dma::go_direct(dev, *dev.dma_mask, ops) ∨ Dma::uses_iommu(dev): dev.dma_skip_sync = dev_is_dma_coherent(dev).
3. else if !ops.sync_single_for_device ∧ !ops.sync_single_for_cpu ∧ !ops.sync_sg_for_device ∧ !ops.sync_sg_for_cpu: dev.dma_skip_sync = true.
4. else: dev.dma_skip_sync = false.

`Dma::set_mask(dev, mask) -> i32`:
1. mask = mask as dma_addr_t.
2. if !dev.dma_mask ∨ !Dma::supported(dev, mask): return -EIO.
3. ArchDma::set_mask(dev, mask).
4. *dev.dma_mask = mask.
5. Dma::setup_need_sync(dev).
6. return 0.

`Dma::set_coherent_mask(dev, mask) -> i32`:
1. mask = mask as dma_addr_t.
2. if !Dma::supported(dev, mask): return -EIO.
3. dev.coherent_dma_mask = mask.
4. return 0.

`Dma::supported(dev, mask) -> bool`:
1. ops = Dma::ops_for(dev).
2. if Dma::uses_iommu(dev): warn_if(ops.is_some()) → return true.
3. if ops.is_some(): if !ops.dma_supported: return true; return ops.dma_supported(dev, mask).
4. return DmaDirect::supported(dev, mask).

`Dma::addressing_limited(dev) -> bool`:
1. /* internal */
2. if min_not_zero(dma_get_mask(dev), dev.bus_dma_limit) < Dma::get_required_mask(dev): return true.
3. if Dma::ops_for(dev).is_some() ∨ Dma::uses_iommu(dev): return false.
4. return !DmaDirect::all_ram_mapped(dev).
5. /* public wrapper */
6. if internal: dev_dbg("DMA addressing limited") → return true.
7. return false.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `dispatch_exhaustive` | INVARIANT | per-facade call: exactly one of {direct, iommu, ops, error} taken. |
| `direct_implies_no_iommu` | INVARIANT | per-go_direct: direct ⟹ !use_dma_iommu. |
| `iommu_implies_no_ops` | INVARIANT | per-supported: use_dma_iommu ⟹ WARN if ops set. |
| `valid_dma_direction_checked` | INVARIANT | per-map/unmap/sync: BUG_ON invalid dir. |
| `dma_mask_validated_before_map` | INVARIANT | per-map_phys: !dma_mask ⟹ DMA_MAPPING_ERROR. |
| `mmio_attr_routed_to_resource` | INVARIANT | per-map_page_attrs: DMA_ATTR_MMIO ⟹ MAPPING_ERROR. |
| `gfp_comp_rejected` | INVARIANT | per-alloc_attrs / alloc_pages: __GFP_COMP ⟹ NULL. |
| `free_no_irq` | INVARIANT | per-free_attrs: irqs_disabled ⟹ WARN. |
| `set_mask_atomic` | INVARIANT | per-set_mask: !supported ⟹ no write; supported ⟹ write + setup_need_sync. |
| `dev_coherent_pool_first` | INVARIANT | per-alloc_attrs: dma_alloc_from_dev_coherent tried before backend. |

### Layer 2: TLA+

`kernel/dma/mapping.tla`:
- Per-map (direct / iommu / ops) + per-unmap + per-sync + per-alloc + per-free.
- Properties:
  - `safety_map_unmap_pair` — per-mapped-addr: unmap path matches map path (direct ⟷ direct, etc.).
  - `safety_alloc_free_pair` — per-cpu_addr: free path matches alloc backend.
  - `safety_dev_pool_lifetime` — per-dev pool addrs: alloc_from_dev_coherent ⟹ release_from_dev_coherent.
  - `safety_no_sync_for_skip` — per-dma_skip_sync coherent direct: sync is no-op.
  - `liveness_set_mask_terminates` — per-set_mask: returns 0 or -EIO.
  - `liveness_alloc_attrs_terminates` — per-alloc_attrs: returns ptr or NULL.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Dma::map_phys` post: ret == ERROR ∨ ret in dev-addressable range | `Dma::map_phys` |
| `Dma::map_sg_inner` post: ret > 0 ⟹ traced; ret ∈ valid-errno-set otherwise | `Dma::map_sg_inner` |
| `Dma::alloc_attrs` post: ret None ⟹ no debug_alloc; Some ⟹ debug + trace done | `Dma::alloc_attrs` |
| `Dma::free_attrs` post: cpu_addr null ⟹ no backend free | `Dma::free_attrs` |
| `Dma::set_mask` post: ret 0 ⟹ *dev.dma_mask == mask ∧ setup_need_sync called | `Dma::set_mask` |
| `Dma::supported` post: use_dma_iommu ⟹ true ∧ ops == NULL invariant | `Dma::supported` |
| `Dma::addressing_limited` post: true ⟹ dev mask < required ∨ !all_ram_mapped | `Dma::addressing_limited` |
| `Dma::can_mmap` post: direct ⟹ direct_can_mmap; iommu ⟹ true; ops ⟹ mmap != NULL | `Dma::can_mmap` |

### Layer 4: Verus/Creusot functional

`Per-driver flow: dma_set_mask → dma_alloc_coherent → dma_map_single (per-IO) → device DMA → dma_sync_single_for_cpu → dma_unmap_single → dma_free_coherent` semantic equivalence: per-Documentation/core-api/dma-api.rst and Documentation/core-api/dma-api-howto.rst.

## Hardening

(Inherits row-1 features from `kernel/dma/00-overview.md` § Hardening and `mm/00-overview.md` § Hardening for coherent-page accounting.)

DMA-mapping facade reinforcement:

- **Per-call valid_dma_direction BUG_ON** — defense against per-stale-dir corruption.
- **Per-dev.dma_mask presence WARN_ON_ONCE → DMA_MAPPING_ERROR** — defense against per-unmasked-bus access.
- **Per-DMA_ATTR_MMIO ↔ dma_map_resource lane separation** — defense against per-RAM-mapped-as-MMIO TLP misroute.
- **Per-DMA_ATTR_REQUIRE_COHERENT honored** — defense against per-non-coherent-fooled driver.
- **Per-is_zone_device_page DEBUG WARN** — defense against per-ZONE_DEVICE-as-coherent confusion.
- **Per-__GFP_COMP rejected in alloc** — defense against per-coherent-as-compound page-pointer corruption.
- **Per-IRQ-context dma_free WARN** — defense against per-vunmap-in-IRQ scheduling-while-atomic.
- **Per-dev coherent-pool checked before fallback** — defense against per-violation of bounded-IO-MMU pool.
- **Per-dma_set_mask validated by dma_supported before write** — defense against per-write-then-fail inconsistent state.
- **Per-iommu vs ops mutual exclusion (WARN_ON in dma_supported)** — defense against per-double-backend race.
- **Per-kmsan_handle_dma instrumentation** — defense against per-DMA-uninit-leak.
- **Per-debug_dma_*** — defense against per-double-map / leak via DMA API debug.
- **Per-dma_addressing_limited dev_dbg** — defense against silent bounce-buffer overhead.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — bounds-check any copy that crosses DMA-staged kernel buffers into user space (e.g. /proc/dma debug dumps).
- **PAX_KERNEXEC** — DMA facade text is read-only; struct dma_map_ops is const and W^X enforced.
- **PAX_RANDKSTACK** — randomize per-syscall stack offset for callers entering dma_map_single / dma_alloc_coherent.
- **PAX_REFCOUNT** — saturating atomics on device-attached map counters and dma_pool refcounts (overflow traps, not wraps).
- **PAX_MEMORY_SANITIZE** — scrub freed coherent buffers on dma_free_coherent to deny residual peripheral payload reuse.
- **PAX_UDEREF** — explicit user/kernel split when faulting on DMA-mapped page metadata.
- **PAX_RAP / kCFI** — type-signed indirect calls through struct dma_map_ops (->map_page, ->alloc, ->sync_*) so a forged ops vector cannot be installed by a writable kernel pointer.
- **GRKERNSEC_HIDESYM** — hide dma_ops, default_dma_ops, dma_direct_ops symbols from /proc/kallsyms for non-root.
- **GRKERNSEC_DMESG** — restrict DMA failure / bounce-fallback diagnostics to CAP_SYSLOG so attackers cannot probe layout.
- **DMA facade PAX_RAP signature on dma_map_ops** — every backend (direct, iommu, swiotlb, virtio) registers with a typed call signature; mismatch panics rather than dispatches.
- **dma_alloc_coherent bounded allocations** — enforce per-device DMA quota and refuse oversized GFP_DMA32 requests from unprivileged ioctl paths.
- **dma_map_sg sg_table sanity** — validate nents and per-sg length under PAX_USERCOPY-class checks before any HW-visible mapping.
- **CONFIG_DMA_API_DEBUG hardened mode** — keep debug entries refcounted (PAX_REFCOUNT) so debug-allocator UAF cannot escalate.
- **Restricted-DMA child pools** — require CAP_SYS_ADMIN to install device-tree restricted-dma-pool overrides at runtime.
- **Rationale**: the DMA facade is a high-value pivot — a single forged ops pointer or unbounded coherent allocation gives the attacker DMA-capable kernel memory. PAX_RAP on the vtable plus saturating refcounts on map state turn the entire surface into a typed, bounded API rather than a raw indirect-call sink.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- kernel/dma/direct.c (covered in `dma-direct.md` Tier-3)
- drivers/iommu/dma-iommu.c (covered in `dma-iommu.md` Tier-3)
- kernel/dma/swiotlb.c (covered in `swiotlb.md` Tier-3)
- kernel/dma/coherent.c (per-device coherent-pool implementation — covered separately if expanded)
- kernel/dma/contiguous.c (CMA — covered separately if expanded)
- kernel/dma/debug.c (covered separately if expanded)
- kernel/dma/pool.c (atomic pool — covered separately if expanded)
- include/trace/events/dma.h (tracepoint definitions)
- Implementation code
