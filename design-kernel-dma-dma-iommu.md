---
title: "Tier-3: drivers/iommu/dma-iommu.c — DMA-API backend over IOMMU (per-device IOAS + IOVA alloc + iommu_map dispatch)"
tags: ["tier-3", "kernel-dma", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

The "DMA-API backend over IOMMU" — when a device is attached to an IOMMU domain (default-DMA or default-IDENTITY mode), kernel-side `dma_map_single` / `dma_map_sg` / `dma_alloc_coherent` for that device dispatches through this code. Bridges the consumer DMA-API (`<linux/dma-mapping.h>`) to the IOMMU API (`iommu_map` / `iommu_unmap` from `iommu-core.md`) + the IOVA allocator (`alloc_iova_fast` / `free_iova_fast` from `iova.md`). Strict-flush mode (`iommu.strict=1`) flushes IOMMU TLB on every unmap; deferred-flush mode (`iommu.strict=0`) batches unmaps via per-CPU flush queue.

This Tier-3 covers `drivers/iommu/dma-iommu.c` (~2300 lines) — the full DMA-IOMMU integration including dma_map_single + sg + page + coherent + scatter-gather list construction + dma_pool integration + per-device cookie state.

### Acceptance Criteria

- [ ] AC-1: NVMe IO at sustained throughput on IOMMU-enabled system uses dma-iommu path; per-poll cycle profile shows < 5% in dma_map / dma_unmap.
- [ ] AC-2: 100GbE NIC (mlx5/ice) sustained line-rate with iommu enabled doesn't bottleneck on dma-iommu.
- [ ] AC-3: GPU dma_alloc_coherent allocation works (texture upload, command buffer alloc) at expected throughput.
- [ ] AC-4: Strict-flush vs deferred-flush comparative test: deferred-flush path has measurably-lower TLB-flush count via `perf stat -e iommu:*`.
- [ ] AC-5: P2PDMA test: NVMe-to-GPU direct DMA via P2PDMA + iommu_dma_map_resource works.
- [ ] AC-6: SG-list test: 16-segment scatter-gather IO at 4K each; mapped to 16 consecutive 4K iova chunks; dma_unmap_sg cleans up correctly.
- [ ] AC-7: Reserved-region respect: per-device alloc never returns iova in MSI-X window or RMRR ranges.
- [ ] AC-8: kselftest dma-iommu-related tests pass.

### Architecture

`Cookie` lives in `kernel::dma::iommu::Cookie`:

```
struct Cookie {
  type_: CookieType,                  // DMA / DMA_FQ / MSI / NULL
  iovad: KBox<IovaDomain>,             // per-cookie IOVA pool
  msi_iova_start: u64,                 // typically 0xFEE00000
  msi_iova_end: u64,                   // typically 0xFEEFFFFF
  msi_page_list: Mutex<HashMap<u64, Arc<Page>>>,
  msi_lookup_lock: SpinLock<()>,
  fq_domain: AtomicPtr<IommuDomain>,
}
```

Per-domain DMA setup:
1. On `iommu_attach_device(domain, dev)` (cross-ref `iommu-core.md`):
   - If `domain.type == DMA OR DMA_FQ`: `iommu_get_dma_cookie(domain)` → install Cookie.
   - `iommu_dma_init_domain(domain, dma_base, dma_limit, dev)` → init cookie's iovad with appropriate aperture.
2. `iommu_setup_dma_ops(dev, dma_base, dma_limit)`:
   - `dev.dma_ops = &iommu_dma_ops`.
   - Subsequent `dma_map_single(dev, ...)` calls dispatch through iommu_dma_map_page.

`Cookie::map_page(dev, page, offset, size, dir, attrs)`:
1. Compute `phys = page_to_phys(page) + offset`.
2. `aligned_size = ALIGN(size + (offset & ~PAGE_MASK), PAGE_SIZE)`.
3. `iova = alloc_iova_fast(&cookie.iovad, aligned_size >> PAGE_SHIFT, dma_limit_pfn, true)`.
4. If iova == 0: WARN_ONCE(...); return DMA_MAPPING_ERROR.
5. `prot = dma_dir_to_prot(dir)` (READ / WRITE / RW).
6. `iommu_map(domain, iova << PAGE_SHIFT, phys & PAGE_MASK, aligned_size, prot, GFP_ATOMIC)`.
7. On iommu_map failure: `free_iova_fast(...)`; return DMA_MAPPING_ERROR.
8. If strict mode: `iommu_iotlb_sync_map(domain, ...)` to ensure new mapping visible.
9. Return `(iova << PAGE_SHIFT) + (phys & ~PAGE_MASK)`.

`Cookie::unmap_page(dev, dma_handle, size, dir, attrs)`:
1. `iova_pfn = (dma_handle & PAGE_MASK) >> PAGE_SHIFT`.
2. `aligned_size_pages = ALIGN(size, PAGE_SIZE) >> PAGE_SHIFT`.
3. `iotlb_gather_init(&gather)`.
4. `iommu_unmap_fast(domain, iova_pfn << PAGE_SHIFT, aligned_size_pages << PAGE_SHIFT, &gather)`.
5. If `cookie.type == DMA` (strict):
   - `iommu_iotlb_sync(domain, &gather)` (wait for TLB drain).
   - `free_iova_fast(&cookie.iovad, iova_pfn, aligned_size_pages)`.
6. If `cookie.type == DMA_FQ` (deferred):
   - `queue_iova(&cookie.iovad, iova_pfn, aligned_size_pages, &gather.freelist)`.
   - Per-CPU IovaFq accumulates; fq_flush_timeout will batch-flush + free IOVAs.

`Cookie::map_sg(dev, sg, nents, dir, attrs)`:
1. Walk sg-list to compute total length + per-segment offsets.
2. Single `alloc_iova_fast(&cookie.iovad, total_pages, dma_limit_pfn, true)` → contiguous IOVA range.
3. Per-segment: compute per-seg iova = base_iova + per-seg-offset; `iommu_map(domain, per-seg-iova, sg_phys(seg), seg_len, prot)`.
4. Update sg[i].dma_address + sg[i].dma_length per segment.
5. Return number of mapped segments.

`Cookie::alloc_coherent(dev, size, &handle, gfp, attrs)`:
- If DMA_ATTR_NO_KERNEL_MAPPING:
  1. Alloc array of pages via `__iommu_dma_alloc_pages(dev, count, alloc_size, gfp)`.
  2. `iova = alloc_iova_fast(...)`.
  3. `iommu_map_sg(domain, iova, sg_table, nents, prot, gfp)` → per-page map.
  4. Return iova as handle, NULL as cpu_addr.
- If DMA_ATTR_FORCE_CONTIGUOUS:
  1. `dma_alloc_from_contiguous(dev, count, get_order(size), gfp)` → CMA-allocated contiguous pages.
  2. `iova = alloc_iova_fast(...)`.
  3. `iommu_map(domain, iova, page_to_phys(page), size, prot)`.
  4. Return iova as handle, page_address(page) as cpu_addr.
- Else (default):
  1. Alloc array of pages.
  2. `iova = alloc_iova_fast(...)`.
  3. `iommu_map_sg(domain, iova, sg_table, nents, prot, gfp)`.
  4. `vmap(pages, count, VM_DMA_COHERENT, prot_pgprot(prot, false))` → vmapped kernel virtual address.
  5. Return iova as handle, vmap_addr as cpu_addr.

Deferred-flush queue integration: per-cookie iovad's fq + fq_timer (cross-ref `iova.md`); per-driver flush_cb installed at cookie init = `iommu_iotlb_sync(domain, &gather)` for batch.

P2PDMA: `iommu_dma_map_resource(dev, phys_addr, size, dir, attrs)` works for PCIe-BAR or other non-page-backed phys range; same as map_page but skips page_to_phys conversion.

### Out of Scope

- IOVA allocator internals (covered in `drivers/iommu/iova.md` Tier-3)
- iommu_map/unmap (covered in `drivers/iommu/iommu-core.md` Tier-3)
- Per-vendor IOMMU TLB-flush impl (covered in `drivers/iommu/intel-iommu.md` + `amd-iommu.md` Tier-3s)
- Generic DMA-API dispatch (covered in `kernel/dma/mapping.md` future Tier-3)
- 32-bit-only paths
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct iommu_dma_cookie` | per-domain cookie holding iova_domain + flush queue + msi_iova_lookup table | `kernel::dma::iommu::Cookie` |
| `iommu_setup_dma_ops(dev, dma_base, dma_limit)` | per-device DMA-ops install | `Cookie::setup_dma_ops` |
| `iommu_dma_init_domain(domain, base, limit, dev)` | per-domain cookie init | `Cookie::init_domain` |
| `iommu_dma_map_page(dev, page, offset, size, dir, attrs)` | per-page DMA-map | `Cookie::map_page` |
| `iommu_dma_unmap_page(dev, dma_handle, size, dir, attrs)` | per-page DMA-unmap | `Cookie::unmap_page` |
| `iommu_dma_map_sg(dev, sg, nents, dir, attrs)` | scatter-gather list map | `Cookie::map_sg` |
| `iommu_dma_unmap_sg(dev, sg, nents, dir, attrs)` | scatter-gather list unmap | `Cookie::unmap_sg` |
| `iommu_dma_alloc(dev, size, &handle, gfp, attrs)` | dma_alloc_coherent backend | `Cookie::alloc_coherent` |
| `iommu_dma_free(dev, size, cpu_addr, handle, attrs)` | dma_free_coherent backend | `Cookie::free_coherent` |
| `iommu_dma_alloc_noncontiguous(dev, size, dir, gfp, attrs)` | dma_alloc_noncontiguous backend | `Cookie::alloc_noncontiguous` |
| `iommu_dma_free_noncontiguous(dev, size, sgt, dir)` | dma_free_noncontiguous backend | `Cookie::free_noncontiguous` |
| `iommu_dma_alloc_iova(domain, size, dma_limit, dev)` | alloc IOVA from cookie's iova_domain | `Cookie::alloc_iova` |
| `iommu_dma_free_iova(cookie, iova, size, gather)` | free IOVA (strict or deferred) | `Cookie::free_iova` |
| `iommu_get_msi_cookie(domain, base)` | install MSI-only cookie (for IRQ-remap path) | `Cookie::get_msi` |
| `iommu_get_dma_cookie(domain)` | install full DMA cookie | `Cookie::get_dma` |
| `iommu_put_dma_cookie(domain)` | release | `Cookie::put` (Drop) |
| `iommu_dma_map_resource(dev, phys, size, dir, attrs)` | map non-page-backed phys range (e.g., MMIO BAR for P2PDMA) | `Cookie::map_resource` |
| `iommu_dma_unmap_resource(dev, dma_handle, size, dir, attrs)` | inverse | `Cookie::unmap_resource` |
| `iommu_dma_sync_single_for_cpu(dev, dma_handle, size, dir)` / `_for_device(...)` | per-direction sync | `Cookie::sync_single_for_cpu` / `_for_device` |
| `iommu_dma_sync_sg_for_cpu(dev, sg, nents, dir)` / `_for_device(...)` | sg sync | `Cookie::sync_sg_for_cpu` / `_for_device` |
| `iommu_dma_get_resv_regions(dev, list)` | per-device reserved-region list (from per-vendor ops) | `Cookie::get_resv_regions` |

### compatibility contract

REQ-1: Per-device DMA-API consumer transparency: in-tree drivers using `<linux/dma-mapping.h>` work unchanged regardless of whether DMA path is dma-direct, swiotlb, or dma-iommu.

REQ-2: Per-domain cookie initialized at first IOMMU attach + freed on detach; per-cookie iova_domain init'd with start_pfn = `cookie->msi_iova_start / PAGE_SIZE` (above MSI-X region).

REQ-3: `iommu_dma_map_page(dev, page, offset, size, dir, attrs)`:
1. `phys = page_to_phys(page) + offset`.
2. `iova = alloc_iova_fast(cookie->iovad, size_aligned, dma_limit, true)`.
3. If iova == 0: return DMA_MAPPING_ERROR.
4. `iommu_map(domain, iova, phys, size_aligned, prot)`.
5. Return iova + (phys & ~PAGE_MASK) (preserve sub-page offset).

REQ-4: `iommu_dma_unmap_page(dev, dma_handle, size, dir, attrs)`:
1. `iova = dma_handle & PAGE_MASK`.
2. `iommu_unmap(domain, iova, size_aligned)`.
3. Strict mode: `iommu_iotlb_sync(domain, gather)` synchronously.
4. Deferred mode: `queue_iova(cookie->iovad, iova >> PAGE_SHIFT, size_pages, &freelist)` defers IOVA free + flush.
5. Strict mode: `free_iova_fast(cookie->iovad, iova >> PAGE_SHIFT, size_pages)`.

REQ-5: `iommu_dma_map_sg(dev, sg, nents, dir, attrs)`:
1. Walk sg-list, sum total length.
2. Single iova-allocation for entire sg (concat'd contiguous IOVA range).
3. Per-segment iommu_map for segments mapped to consecutive IOVA chunks.
4. Update sg-list entries with returned dma_address + dma_length per segment.
5. Return number of mapped sg entries (`outsides`).

REQ-6: `iommu_dma_alloc(dev, size, &handle, gfp, attrs)`:
- For DMA_ATTR_NO_KERNEL_MAPPING: alloc page-array (lazy-mapping; user-mmap'able later).
- For DMA_ATTR_FORCE_CONTIGUOUS: alloc contiguous physical range via CMA.
- For default: alloc per-page from page allocator + iommu_map into IOAS + vmap for cpu-visible kernel address.
- Return cpu_addr (vmap'd) + handle (iova).

REQ-7: Per-device DMA-ops `iommu_dma_ops` installed via `set_dma_ops(dev, &iommu_dma_ops)` during `iommu_setup_dma_ops`.

REQ-8: MSI cookie: per-MSI-only domain (used by hyperv-iommu, virt-only); reserves a small region for MSI doorbell mapping; not used for general DMA.

REQ-9: P2PDMA (peer-to-peer DMA between PCIe devices): `iommu_dma_map_resource` maps a peer device's BAR phys-range as iova; subsequent DMA from the source device targets the peer's BAR directly.

REQ-10: Per-device reserved-region enumeration via `iommu_get_resv_regions(dev, &list)` (per-vendor ops); cookie reserves these IOVAs at init so subsequent allocs avoid them.

REQ-11: Strict-flush vs deferred-flush selection per `iommu.strict=1/0` cmdline + per-domain DMA_FQ type (cross-ref `iommu-core.md` § Domain types).

REQ-12: Per-CPU flush queue `iommu_dma_flush_queue` integrated with `iova.md` IovaFq; per-CPU batch up to 256 iova frees + 10ms hrtimer-driven flush.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cookie_no_uaf` | UAF | per-domain Cookie outlives all in-flight dma_map; cookie destroy waits for refcount==0. |
| `iova_alloc_no_collision` | UNIQUENESS | per-cookie iovad alloc returns unique IOVA per call; covered by `iova.md`'s rb-tree invariants. |
| `dma_handle_no_corruption` | INVARIANT | returned dma_handle correctly encodes (iova_pfn << PAGE_SHIFT) | (phys & ~PAGE_MASK); unmap correctly decodes. |
| `iommu_map_failure_rollback` | ATOMICITY | iommu_map failure path frees IOVA before returning DMA_MAPPING_ERROR; no leak. |

### Layer 2: TLA+

`models/iommu/iova_alloc.tla` (parent-declared in `drivers/iommu/00-overview.md`): proves IOVA allocator under N concurrent CPUs.
`models/dma/swiotlb_alloc.tla` is for swiotlb; dma-iommu has implicit reliance on `iova_alloc.tla`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `iommu_dma_map_page` post: returned dma_handle != DMA_MAPPING_ERROR ↔ underlying IOMMU mapping installed AND IOVA reserved | `Cookie::map_page` |
| `iommu_dma_unmap_page` post: IOVA freed (after fq drain in deferred mode) AND IOMMU mapping unmapped | `Cookie::unmap_page` |
| `iommu_dma_alloc_coherent` post: returned cpu_addr is mmappable AND handle is device-addressable AND coherent (no extra sync needed for CPU↔device) | `Cookie::alloc_coherent` |
| Per-cookie `iovad.cached_node` invariant: hint always points within iovad's aperture range | (delegated to `iova.md`) |

### Layer 4: Verus/Creusot functional

`Cookie::map_page(page, offset, size, dir) → device DMA → Cookie::unmap_page(handle, size)` round-trip equivalence: post-map, device DMA at `dma_handle` reads/writes the underlying page; post-unmap, subsequent device DMA at handle produces fault. Encoded as Verus invariant chained with `iommu-core.md`'s map↔unmap inverse.

### hardening

(Inherits row-1 features from `kernel/dma/00-overview.md` § Hardening.)

dma-iommu specific reinforcement:

- **Reserved-region enforced** — per-cookie iovad reserves MSI-X window + RMRR ranges at init; subsequent alloc never returns IOVA in those ranges. Defense against MSI-X table corruption via accidental DMA.
- **Per-cookie IOVA accounting** — per-process locked-memory cap enforced via cgroup-iommu (cross-ref `kernel/cgroup/`); defense against unprivileged process pinning all of host RAM via dma_alloc_coherent.
- **Strict-flush mode default-on** for security-sensitive devices (vfio-pci-bound, USB-storage); deferred-flush optional via `iommu.strict=0` cmdline.
- **Per-IOVA flush before reuse** — defense against stale TLB entry from prior IOVA owner servicing new device's DMA.
- **iommu_map failure rollback** — per-segment iommu_map failure in iommu_dma_map_sg unmaps prior segments before returning error; no partial mapping left in IOAS.
- **DMA-direction prot mapping correct** — DMA_TO_DEVICE → IOMMU_READ; DMA_FROM_DEVICE → IOMMU_WRITE; DMA_BIDIRECTIONAL → IOMMU_READ | IOMMU_WRITE; defense against per-direction permission downgrade attacks.
- **MSI cookie isolation** — MSI-only domains have separate cookie that doesn't allow general DMA mapping; defense against MSI-domain-confusion attacks.

