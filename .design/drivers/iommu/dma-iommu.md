# Tier-3: drivers/iommu/dma-iommu.c — generic DMA-IOMMU backend (default DMA ops for IOMMU-equipped systems)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/iommu/00-overview.md
upstream-paths:
  - drivers/iommu/dma-iommu.c (~2255 lines)
  - drivers/iommu/dma-iommu.h
  - include/linux/iommu-dma.h
  - include/linux/iova.h
  - include/linux/dma-iommu.h
  - Documentation/core-api/dma-api.rst
-->

## Summary

`drivers/iommu/dma-iommu.c` is the **generic DMA-API backend that uses an IOMMU domain as the translation engine**. It replaces `swiotlb-direct` on systems with an IOMMU (Intel VT-d / AMD-Vi / ARM-SMMU / virtio-iommu / Hyper-V). Every `dma_map_page` / `dma_alloc_coherent` / `dma_map_sg` call from a driver bound to an IOMMU-managed device routes here.

Architectural role:
- The **`iommu_dma_ops` vtable** (`iommu_dma_map_phys`, `iommu_dma_unmap_phys`, `iommu_dma_map_sg`, `iommu_dma_unmap_sg`, `iommu_dma_alloc`, `iommu_dma_free`, `iommu_dma_alloc_noncontiguous`, `iommu_dma_free_noncontiguous`, `iommu_dma_mmap`, `iommu_dma_get_sgtable`, `iommu_dma_sync_*`, `iommu_dma_opt_mapping_size`, `iommu_dma_max_mapping_size`) plugs into `struct dma_map_ops` consumed by `kernel/dma/`.
- The **IOMMU-domain cookie machinery** (`iommu_get_dma_cookie`, `iommu_put_dma_cookie`, `iommu_get_msi_cookie`, `iommu_put_msi_cookie`, `iommu_dma_init_domain`, `iommu_setup_dma_ops`): attaches a `struct iommu_dma_cookie` (containing the `struct iova_domain` + msi-page-list + flush-queue) to a domain at first DMA use, freed on domain destroy. Two cookie flavors: `IOMMU_COOKIE_DMA_IOVA` (full DMA-API) and `IOMMU_COOKIE_DMA_MSI` (MSI-only, for `IOMMU_DOMAIN_UNMANAGED` domains owned by VFIO/iommufd).
- The **IOVA allocation wrapper** (`iommu_dma_alloc_iova`, `iommu_dma_free_iova`): converts the iovad rb-tree (32-bit-first then 64-bit fallback to avoid SAC-vs-DAC firmware bugs) into a per-DMA-op address; consumes `alloc_iova_fast` (with per-CPU magazine cache from `drivers/iommu/iova.c`).
- The **flush-queue (FQ) lazy-unmap mechanism** (`iova_fq`, `iova_fq_entry`, `queue_iova`, `fq_ring_free_locked`, `fq_flush_iotlb`, `fq_flush_timeout`, `iommu_dma_init_fq`, `iommu_dma_free_fq`): defer IOTLB flush + IOVA-free for `IOMMU_DOMAIN_DMA_FQ` domains. Per-CPU queue (default) or single-queue (for hardware that can't afford per-CPU memory), backed by a 10ms (default) or 1000ms (single-queue) timer that empties the queue + invalidates the IOTLB. Cuts the cost of high-rate DMA-unmap by batching invalidations.
- The **swiotlb bounce-buffer integration** (`iommu_dma_map_swiotlb`, `dev_use_swiotlb`, `dev_use_sg_swiotlb`, `iommu_dma_iova_link_swiotlb`, `iommu_dma_iova_bounce_and_link`, `iommu_dma_map_sg_swiotlb`, `iommu_dma_unmap_sg_swiotlb`): bounce DMA through `swiotlb_tbl_map_single` when granule-mis-aligned or device-untrusted (CoCo / TDX guests). Preserves swiotlb safety for sub-granule mappings even on IOMMU systems.
- The **MSI doorbell remapping** (`iommu_dma_get_msi_page`, `iommu_dma_sw_msi`, `cookie_init_hw_msi_region`, `has_msi_cookie`, `cookie_msi_granule`, `cookie_msi_pages`): when an attached device sends an MSI to an APIC doorbell address (e.g., 0xFEE0_0000 on x86), the IOMMU must translate that address. dma-iommu maintains a per-cookie cached list of `iommu_dma_msi_page` entries (msi_phys → msi_iova mapping in the IOMMU) so multiple devices sharing the same doorbell share the same IOVA mapping.
- The **dma-iova streaming API** (new): `dma_iova_try_alloc`, `dma_iova_free`, `dma_iova_link`, `dma_iova_sync`, `dma_iova_unlink`, `dma_iova_destroy`, `__dma_iova_link`, `iommu_dma_iova_link_swiotlb`, `iommu_dma_iova_bounce_and_link`, `iommu_dma_iova_unlink_range_slow`, `__iommu_dma_iova_unlink`. Lets callers (notably the new io_uring zero-copy path + dmabuf) pre-reserve an IOVA range then progressively link physical segments. Decouples IOVA-alloc from IOMMU-map.
- The **PCI-host-bridge windows + reserved-region honoring** (`iova_reserve_pci_windows`, `iova_reserve_iommu_regions`, `cookie_init_hw_msi_region`, `iommu_dma_get_resv_regions`): pre-reserve IOVA ranges that overlap host-bridge MMIO windows (so generated IOVAs don't collide with peer-to-peer routing) and per-iommu-driver reserved regions (e.g., GICv3 ITS doorbells, Intel RMRR ranges).
- The **`iommu_setup_dma_ops` hook called from device-bind** chooses between this backend, `dma-direct` (no IOMMU), or swiotlb-direct (no IOMMU, bounce-only). Sets `dev->dma_iommu = true` and `dev->dma_ops = NULL` (the DMA-API core then takes the iommu path).
- The **`iommu_dma_forcedac` boot param** (`iommu.forcedac=1`): force 64-bit IOVAs even for 32-bit-PCI workaround paths.
- The **deferred-attach kdump path** (`iommu_deferred_attach_enabled` static branch): kdump kernel inherits primary kernel's IOMMU page tables, so attach is deferred to first DMA.

Critical for: every NVMe / NIC / GPU / accelerator DMA on x86_64 with intel_iommu=on, dma_iommu=force, or amd_iommu=on; CoCo guest swiotlb fallback; SVA / iommufd VFIO replacement chain; live migration via fq-flush dirty tracking; MSI remap correctness (without dma-iommu's `iommu_dma_get_msi_page`, MSIs from a passthrough device would bypass the IOMMU and break isolation).

This Tier-3 covers `drivers/iommu/dma-iommu.c` (~2255 lines). The IOVA allocator body, the per-vendor IOMMU driver attach/map/unmap ops, and the dma-API public surface (`kernel/dma/`) are covered separately.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct iommu_dma_cookie` | per-DMA-domain backing | `IommuDmaCookie` |
| `struct iommu_dma_msi_cookie` | per-MSI-only-domain backing | `IommuDmaMsiCookie` |
| `struct iommu_dma_msi_page` | per-cached-MSI-doorbell IOVA | `IommuDmaMsiPage` |
| `struct iommu_dma_options` | per-FQ-tuning options | `IommuDmaOptions` |
| `enum iommu_dma_queue_type` | per-FQ kind selector | `IommuDmaQueueType` |
| `struct iova_fq` | per-flush-queue ring | `IovaFq` |
| `struct iova_fq_entry` | per-flush-queue entry | `IovaFqEntry` |
| `iommu_get_dma_cookie()` / `iommu_put_dma_cookie()` | per-domain cookie alloc | `DmaIommu::get_dma_cookie` / `put_dma_cookie` |
| `iommu_get_msi_cookie()` / `iommu_put_msi_cookie()` | per-MSI-domain cookie | `DmaIommu::get_msi_cookie` |
| `iommu_dma_init_domain()` | per-domain late-init | `DmaIommu::init_domain` |
| `iommu_dma_init_options()` | per-domain options select | `DmaIommu::init_options` |
| `iommu_dma_init_fq()` | per-domain FQ init | `DmaIommu::init_fq` |
| `iommu_dma_init_fq_single()` / `_percpu()` | per-FQ-kind alloc | `DmaIommu::init_fq_single` / `_percpu` |
| `iommu_dma_init_one_fq()` | per-FQ ring init | `DmaIommu::init_one_fq` |
| `iommu_dma_free_fq()` / `_fq_single()` / `_fq_percpu()` | per-FQ teardown | `DmaIommu::free_fq` |
| `iommu_dma_alloc_iova()` / `_free_iova()` | per-IOVA alloc/free | `DmaIommu::alloc_iova` / `free_iova` |
| `__iommu_dma_map()` / `__iommu_dma_unmap()` | per-low-level map/unmap | `DmaIommu::map_inner` / `unmap_inner` |
| `iommu_dma_map_phys()` / `iommu_dma_unmap_phys()` | per-public map page/phys | `DmaIommu::map_phys` / `unmap_phys` |
| `iommu_dma_map_sg()` / `iommu_dma_unmap_sg()` | per-public map sg | `DmaIommu::map_sg` / `unmap_sg` |
| `iommu_dma_map_swiotlb()` | per-bounce-then-map | `DmaIommu::map_swiotlb` |
| `iommu_dma_map_sg_swiotlb()` / `iommu_dma_unmap_sg_swiotlb()` | per-bounce sg | `DmaIommu::map_sg_swiotlb` |
| `iommu_dma_alloc()` / `iommu_dma_free()` | per-coherent alloc | `DmaIommu::alloc` / `free` |
| `iommu_dma_alloc_pages()` | per-discontig-pages alloc | `DmaIommu::alloc_pages` |
| `iommu_dma_alloc_remap()` | per-vmap-alloc | `DmaIommu::alloc_remap` |
| `iommu_dma_alloc_noncontiguous()` / `_free_noncontiguous()` | per-dmabuf alloc | `DmaIommu::alloc_noncontiguous` |
| `iommu_dma_vmap_noncontiguous()` / `_mmap_noncontiguous()` | per-dmabuf vmap | `DmaIommu::vmap_noncontiguous` |
| `iommu_dma_mmap()` | per-userspace BAR mmap | `DmaIommu::mmap` |
| `iommu_dma_get_sgtable()` | per-coherent sg_table | `DmaIommu::get_sgtable` |
| `iommu_dma_sync_single_for_cpu()` / `_for_device()` | per-bounce sync (cpu/dev) | `DmaIommu::sync_single` |
| `iommu_dma_sync_sg_for_cpu()` / `_for_device()` | per-bounce-sg sync | `DmaIommu::sync_sg` |
| `iommu_dma_opt_mapping_size()` | per-best-streaming-size | `DmaIommu::opt_mapping_size` |
| `iommu_dma_max_mapping_size()` | per-max-streaming-size | `DmaIommu::max_mapping_size` |
| `iommu_setup_dma_ops()` | per-device select-backend | `DmaIommu::setup_dma_ops` |
| `dma_iova_try_alloc()` / `dma_iova_free()` | per-pre-reserved-IOVA | `DmaIommu::iova_try_alloc` |
| `dma_iova_link()` / `dma_iova_sync()` / `dma_iova_unlink()` / `dma_iova_destroy()` | per-progressive map | `DmaIommu::iova_link` |
| `__dma_iova_link()` / `iommu_dma_iova_bounce_and_link()` / `iommu_dma_iova_link_swiotlb()` | per-link impl | `DmaIommu::iova_link_inner` |
| `iommu_dma_iova_unlink_range_slow()` / `__iommu_dma_iova_unlink()` | per-unlink impl | `DmaIommu::iova_unlink_inner` |
| `iommu_dma_get_msi_page()` | per-MSI-doorbell cached IOVA | `DmaIommu::get_msi_page` |
| `iommu_dma_sw_msi()` | per-msi-desc IOVA program | `DmaIommu::sw_msi` |
| `cookie_init_hw_msi_region()` | per-static-MSI-region (ITS) | `DmaIommu::init_hw_msi_region` |
| `has_msi_cookie()` / `cookie_msi_granule()` / `cookie_msi_pages()` | per-cookie-flavor helpers | `IommuDmaCookie::msi_*` |
| `iommu_dma_get_resv_regions()` | per-device-reserved-regions | `DmaIommu::get_resv_regions` |
| `iova_reserve_pci_windows()` / `iova_reserve_iommu_regions()` | per-domain pre-reserve | `DmaIommu::reserve_*` |
| `queue_iova()` | per-fq-enqueue | `DmaIommu::queue_iova` |
| `fq_ring_free_locked()` / `fq_ring_free()` | per-fq-drain | `DmaIommu::fq_ring_free` |
| `fq_flush_iotlb()` | per-fq global iotlb-flush | `DmaIommu::fq_flush_iotlb` |
| `fq_flush_timeout()` | per-fq lazy-flush timer | `DmaIommu::fq_flush_timeout` |
| `dev_use_swiotlb()` / `dev_use_sg_swiotlb()` | per-device bounce decision | `DmaIommu::dev_use_swiotlb` |
| `dma_info_to_prot()` | per-DMA-dir → IOMMU prot bits | `DmaIommu::info_to_prot` |
| `iova_unaligned()` | per-buffer granule-check | `DmaIommu::iova_unaligned` |
| `iommu_dma_forcedac` | per-bool cmdline | shared (`iommu.forcedac=1`) |
| `iommu_deferred_attach_enabled` | per-static-key kdump | shared |
| `iommu_dma_init()` | per-arch_initcall | `DmaIommu::init` |
| `IOVA_DEFAULT_FQ_SIZE` / `_SINGLE_FQ_SIZE` | per-FQ ring size const | shared (256 / 32768) |
| `IOVA_DEFAULT_FQ_TIMEOUT` / `_SINGLE_FQ_TIMEOUT` | per-FQ timer const | shared (10 ms / 1000 ms) |

## Compatibility contract

REQ-1: Cookie data layout:
- `struct iommu_dma_cookie`:
  - `iovad: struct iova_domain` (rb-tree + per-cpu magazines from `drivers/iommu/iova.c`).
  - `msi_page_list: list_head` (cached MSI doorbell IOVA mappings).
  - union { `single_fq: *iova_fq`; `percpu_fq: __percpu *iova_fq` } selected by `options.qt`.
  - `fq_flush_start_cnt: atomic64`, `fq_flush_finish_cnt: atomic64` (acquire-release ordering for batched flushes).
  - `fq_timer: timer_list`, `fq_timer_on: atomic_t` (one-shot mod_timer arming).
  - `fq_domain: *iommu_domain` (non-NULL when FQ active; NULL falls back to strict mode).
  - `options: iommu_dma_options { qt, fq_size, fq_timeout }`.
- `struct iommu_dma_msi_cookie`:
  - `msi_iova: dma_addr_t` (bump-pointer through pre-reserved MSI IOVA region).
  - `msi_page_list: list_head`.
- `struct iommu_dma_msi_page`:
  - `list: list_head`, `iova: dma_addr_t`, `phys: phys_addr_t`.
- `domain.cookie_type ∈ { IOMMU_COOKIE_NONE, IOMMU_COOKIE_DMA_IOVA, IOMMU_COOKIE_DMA_MSI, ... }`.

REQ-2: Cookie acquire:
- `iommu_get_dma_cookie(domain)`:
  - If `domain.cookie_type != IOMMU_COOKIE_NONE`: return -EEXIST.
  - kzalloc `cookie`.
  - INIT_LIST_HEAD `msi_page_list`.
  - `domain.cookie_type = IOMMU_COOKIE_DMA_IOVA`; `domain.iova_cookie = cookie`.
  - Return 0. (Note: `iovad` is **not** initialized — that's `init_domain`'s job, deferred until first device attach.)
- `iommu_get_msi_cookie(domain, base)`:
  - Require `domain.type == IOMMU_DOMAIN_UNMANAGED`.
  - Require `cookie_type == NONE`.
  - kzalloc `cookie`; set `msi_iova = base`; INIT msi_page_list.
  - `cookie_type = IOMMU_COOKIE_DMA_MSI`; `msi_cookie = cookie`.
  - Return 0.

REQ-3: Cookie release:
- `iommu_put_dma_cookie(domain)`:
  - `cookie = domain.iova_cookie`.
  - If `cookie.iovad.granule != 0`: `iommu_dma_free_fq(cookie)`; `put_iova_domain(&cookie.iovad)`.
  - Free every entry on `msi_page_list`.
  - kfree(cookie).
- `iommu_put_msi_cookie(domain)`: free every msi_page; kfree cookie.

REQ-4: Domain late-init (`iommu_dma_init_domain(domain, dev)`):
- Validate: `cookie != NULL` ∧ `cookie_type == IOMMU_COOKIE_DMA_IOVA`.
- `order = __ffs(domain.pgsize_bitmap)` (smallest hardware page size).
- `base_pfn = max(1, domain.geometry.aperture_start >> order)`.
- If `dev.dma_range_map` extends below `aperture_start` or above `aperture_end`: warn, return -EFAULT.
- If `iovad.start_pfn != 0` (already-initialised): assert `1UL << order == iovad.granule` ∧ `base_pfn == iovad.start_pfn`; else return -EFAULT. Return 0.
- `init_iova_domain(&iovad, 1UL << order, base_pfn)`.
- `iova_domain_init_rcaches(&iovad)`.
- `iommu_dma_init_options(&options, dev)`: select FQ kind (per-CPU vs single) + size + timeout based on dev untrusted-ness / IOMMU strict-ness.
- If `domain.type == IOMMU_DOMAIN_DMA_FQ`: try `iommu_dma_init_fq(domain)`; on failure downgrade `domain.type = IOMMU_DOMAIN_DMA` (strict mode).
- `iova_reserve_iommu_regions(dev, domain)`: pre-reserve PCI host-bridge windows + iommu_resv regions.

REQ-5: Flush queue (FQ) data:
- `struct iova_fq_entry { iova_pfn: ulong; pages: ulong; freelist: iommu_pages_list; counter: u64 }`. `counter` = `fq_flush_start_cnt` at enqueue.
- `struct iova_fq { lock: spinlock_t; head: u32; tail: u32; mod_mask: u32; entries: [iova_fq_entry; .] }`.
- Sizes: `IOVA_DEFAULT_FQ_SIZE = 256` (per-CPU), `IOVA_SINGLE_FQ_SIZE = 32768` (single-queue).
- Timeout: `IOVA_DEFAULT_FQ_TIMEOUT = 10` ms (per-CPU), `IOVA_SINGLE_FQ_TIMEOUT = 1000` ms (single).
- `fq_full(fq)`: `((tail+1) & mod_mask) == head`.
- `fq_ring_add(fq)`: `idx = tail; tail = (idx+1) & mod_mask; return idx`.

REQ-6: FQ flush counters:
- `fq_flush_iotlb(cookie)`:
  - `atomic64_inc(&fq_flush_start_cnt)`.
  - `cookie.fq_domain.ops.flush_iotlb_all(cookie.fq_domain)`.
  - `atomic64_inc(&fq_flush_finish_cnt)`.
- An entry with `counter < finish_cnt` is safe to free (its IOTLB-shootdown has completed).

REQ-7: FQ enqueue (`queue_iova(cookie, pfn, pages, freelist)`):
- `smp_mb()` (order against IOMMU page-table store).
- Select `fq`: single → `cookie.single_fq`; percpu → `raw_cpu_ptr(cookie.percpu_fq)`.
- Acquire `fq.lock` (irqsave).
- `fq_ring_free_locked(cookie, fq)` (drain already-flushed entries).
- If `fq_full(fq)`: `fq_flush_iotlb(cookie)`; `fq_ring_free_locked(cookie, fq)` again.
- `idx = fq_ring_add(fq)`; populate `entries[idx]` with pfn / pages / freelist / current `fq_flush_start_cnt`.
- Splice `freelist` into `entries[idx].freelist`.
- Release lock.
- If `!atomic_read(&fq_timer_on) ∧ !atomic_xchg(&fq_timer_on, 1)`: `mod_timer(&fq_timer, jiffies + msecs_to_jiffies(options.fq_timeout))`.

REQ-8: FQ drain (`fq_ring_free_locked(cookie, fq)`):
- `counter = atomic64_read(&fq_flush_finish_cnt)`.
- Iterate from `head` to `tail`:
  - if `entries[idx].counter >= counter`: stop (shootdown not done yet).
  - `iommu_put_pages_list(&entries[idx].freelist)`.
  - `free_iova_fast(&cookie.iovad, entries[idx].iova_pfn, entries[idx].pages)`.
  - Reset freelist.
  - Advance `head`.

REQ-9: FQ timer (`fq_flush_timeout`):
- `cookie = timer_container_of(cookie, t, fq_timer)`.
- `atomic_set(&fq_timer_on, 0)`.
- `fq_flush_iotlb(cookie)`.
- For each fq (single or per-CPU): `fq_ring_free(cookie, fq)`.

REQ-10: IOVA alloc (`iommu_dma_alloc_iova(domain, size, dma_limit, dev)`):
- MSI cookie special case: bump-pointer `cookie.msi_iova += size; return cookie.msi_iova - size`.
- `iovad = &iova_cookie.iovad`; `shift = iova_shift(iovad)`; `iova_len = size >> shift`.
- `dma_limit = min_not_zero(dma_limit, dev.bus_dma_limit)`.
- If `domain.geometry.force_aperture`: clamp `dma_limit ≤ aperture_end`.
- 32-bit-first heuristic: if `dma_limit > 0xFFFFFFFF ∧ dev.iommu.pci_32bit_workaround`:
  - try `alloc_iova_fast(iovad, iova_len, 0xFFFFFFFF >> shift, false)`. If success: goto done.
  - Else: set `dev.iommu.pci_32bit_workaround = false`; `dev_notice("Using %d-bit DMA addresses", bits_per(dma_limit))`.
- `iova = alloc_iova_fast(iovad, iova_len, dma_limit >> shift, true)`.
- Return `iova << shift`.

REQ-11: IOVA free (`iommu_dma_free_iova(domain, iova, size, gather)`):
- MSI cookie: `msi_iova -= size`.
- Else if `gather && gather.queued`: `queue_iova(cookie, iova_pfn(iovad, iova), size >> iova_shift, &gather.freelist)` (FQ defer).
- Else: `free_iova_fast(iovad, iova_pfn, size >> iova_shift)` (strict).

REQ-12: Low-level map (`__iommu_dma_map(dev, phys, size, prot, dma_mask)`):
- Deferred-attach: if `static_branch_unlikely(iommu_deferred_attach_enabled) ∧ iommu_deferred_attach(dev, domain)` fails: return DMA_MAPPING_ERROR.
- `iova_off = iova_offset(iovad, phys)`.
- `size = iova_align(iovad, size + iova_off)`.
- `iova = iommu_dma_alloc_iova(domain, size, dma_mask, dev)`. If 0: DMA_MAPPING_ERROR.
- `iommu_map(domain, iova, phys - iova_off, size, prot, GFP_ATOMIC)`. On failure: free_iova; DMA_MAPPING_ERROR.
- Return `iova + iova_off`.

REQ-13: Low-level unmap (`__iommu_dma_unmap(dev, dma_addr, size)`):
- `iova_off = iova_offset(iovad, dma_addr)`.
- `dma_addr -= iova_off`; `size = iova_align(iovad, size + iova_off)`.
- `iommu_iotlb_gather_init(&iotlb_gather)`; `iotlb_gather.queued = READ_ONCE(cookie.fq_domain)`.
- `unmapped = iommu_unmap_fast(domain, dma_addr, size, &iotlb_gather)`; WARN if `unmapped != size`.
- If `!iotlb_gather.queued`: `iommu_iotlb_sync(domain, &iotlb_gather)` (strict synchronous).
- `iommu_dma_free_iova(domain, dma_addr, size, &iotlb_gather)`.

REQ-14: Public map (`iommu_dma_map_phys(dev, phys, size, dir, attrs)`):
- `prot = dma_info_to_prot(dir, coherent=dev_is_dma_coherent(dev), attrs)`.
- If `dev_use_swiotlb(dev, size, dir) ∧ iova_unaligned(iovad, phys, size)`:
  - if `attrs & (DMA_ATTR_MMIO | DMA_ATTR_REQUIRE_COHERENT)`: return DMA_MAPPING_ERROR.
  - `phys = iommu_dma_map_swiotlb(dev, phys, size, dir, attrs)` (bounce).
  - if `phys == DMA_MAPPING_ERROR`: return error.
- If !coherent ∧ !(attrs & (SKIP_CPU_SYNC | MMIO)): `arch_sync_dma_for_device(phys, size, dir)`; `arch_sync_dma_flush()`.
- `iova = __iommu_dma_map(dev, phys, size, prot, dma_get_mask(dev))`.
- On error + not MMIO/REQUIRE_COHERENT: `swiotlb_tbl_unmap_single(dev, phys, size, dir, attrs)` (rollback bounce).
- Return iova.

REQ-15: Public unmap (`iommu_dma_unmap_phys(dev, dma_handle, size, dir, attrs)`):
- If MMIO ∨ REQUIRE_COHERENT: `__iommu_dma_unmap(dev, dma_handle, size)`; return.
- `phys = iommu_iova_to_phys(domain, dma_handle)`. WARN_ON if 0.
- If !SKIP_CPU_SYNC ∧ !coherent: `arch_sync_dma_for_cpu(phys, size, dir)`; `arch_sync_dma_flush()`.
- `__iommu_dma_unmap(dev, dma_handle, size)`.
- `swiotlb_tbl_unmap_single(dev, phys, size, dir, attrs)` (idempotent if not bounced).

REQ-16: SG map (`iommu_dma_map_sg(dev, sg, nents, dir, attrs)`):
- Deferred-attach.
- If `dev_use_sg_swiotlb(dev, sg, nents, dir)`: route to `iommu_dma_map_sg_swiotlb` (per-seg map_phys).
- Sync per-device.
- Stash original segment `offset` + `length` in `dma_address` + `dma_len` fields (recoverable on error).
- For each segment:
  - Per-segment `pci_p2pdma_state` check: MAP_THRU_HOST_BRIDGE → normal IOVA; MAP_BUS_ADDR → set `s.dma_address` to bus addr, mark `sg_dma_mark_bus_address`, skip IOVA allocation; MAP_NONE → normal; default → -EREMOTEIO.
  - Align `s.offset` down + `s.length` up to iovad granule.
  - Track per-segment-boundary mask for combined-IOVA allocation.
- Allocate single IOVA for combined iova_len.
- `iommu_map_sg(domain, iova, sg, nents, prot, GFP_ATOMIC)`. On failure: free_iova, restore_sg.
- `__finalise_sg(dev, sg, nents, iova)`: merge per-segment `dma_address` back, applying segment-boundary + max-segment-size.
- Return number of merged segments.

REQ-17: SG unmap (`iommu_dma_unmap_sg`):
- If `sg_dma_is_swiotlb(sg)`: `iommu_dma_unmap_sg_swiotlb` per segment.
- Else: walk sg → find min(start) + max(end) → single `__iommu_dma_unmap` covering the combined IOVA range.

REQ-18: Coherent alloc (`iommu_dma_alloc(dev, size, handle, gfp, attrs)`):
- If gfp & GFP_DMA / DMA32: split path.
- If size > PAGE_SIZE: `iommu_dma_alloc_remap` (vmap noncontig pages + IOMMU-map).
- Else: `iommu_dma_alloc_pages(dev, size, gfp, &page, attrs)` (contiguous pages + IOMMU-map).
- On success, store iova in `*handle`.

REQ-19: Coherent free (`iommu_dma_free`):
- `__iommu_dma_unmap`.
- `__iommu_dma_free`: vunmap if remap; free pages.

REQ-20: Noncontiguous alloc (`iommu_dma_alloc_noncontiguous(dev, size, dir, gfp, attrs)`):
- Allocate `dma_sgt_handle` (sg_table + pages array).
- `__iommu_dma_alloc_noncontiguous(dev, size, sgt, gfp, attrs)`:
  - `min_size = alloc_sizes & -alloc_sizes` (smallest pgsize_bitmap entry).
  - Page count = `PAGE_ALIGN(size) >> PAGE_SHIFT`.
  - `pages = __iommu_dma_alloc_pages(dev, count, alloc_sizes >> PAGE_SHIFT, gfp)`: try high-order first (NORETRY), fall back to min-order.
  - Build sg_table from pages.
  - Allocate IOVA for total size, `iommu_map_sg`, mark sg with IOVA.
- Return sg_table for dmabuf use.

REQ-21: `iommu_dma_get_sgtable(dev, sgt, cpu_addr, dma_addr, size, attrs)`:
- For `_alloc_remap` allocations: lookup `dma_sgt_handle` from cpu_addr (`vmalloc_to_handle`); duplicate sg_table for caller.
- For contiguous coherent: `dma_common_get_sgtable` falls back.

REQ-22: MSI page (`iommu_dma_get_msi_page(dev, msi_addr, domain)`):
- `msi_page_list = cookie_msi_pages(domain)`.
- `size = cookie_msi_granule(domain)` (= iovad.granule for DMA cookie; PAGE_SIZE for MSI cookie).
- Align `msi_addr` down to granule.
- Search list for existing entry with `msi_page.phys == msi_addr`; return it.
- Else: kzalloc page; `iova = iommu_dma_alloc_iova(domain, size, dma_get_mask(dev), dev)`; `iommu_map(domain, iova, msi_addr, size, IOMMU_WRITE|IOMMU_NOEXEC|IOMMU_MMIO, GFP_KERNEL)`; insert into list.
- Return page.

REQ-23: `iommu_dma_sw_msi(domain, desc, msi_addr)`:
- If `!has_msi_cookie(domain)`: `msi_desc_set_iommu_msi_iova(desc, 0, 0)`; return 0.
- `iommu_group_mutex_assert(dev)`.
- `msi_page = iommu_dma_get_msi_page(dev, msi_addr, domain)`. If NULL: return -ENOMEM.
- `msi_desc_set_iommu_msi_iova(desc, msi_page.iova, ilog2(cookie_msi_granule(domain)))`.
- This re-writes the MSI message-address to the IOVA so the device's MSI write is translated by the IOMMU.

REQ-24: Static MSI region init (`cookie_init_hw_msi_region(cookie, start, end)`):
- For each page in `[start, end)` aligned to iovad.granule:
  - kmalloc msi_page with `phys = iova = start` (identity map within reserved region).
  - Add to msi_page_list.
- Used for GICv3 ITS region on ARM ACPI / DT.

REQ-25: Reserved-region pre-reserve (`iova_reserve_iommu_regions(dev, domain)`):
- If PCI: `iova_reserve_pci_windows(to_pci_dev(dev), iovad)`:
  - Iterate `bridge.windows` (host-bridge MMIO/IO apertures): reserve via `reserve_iova(iovad, lo, hi)`.
  - Iterate `bridge.dma_ranges` (sorted): reserve gaps between consecutive dma_ranges so the iovad excludes any address the host bridge can't translate.
- `iommu_get_resv_regions(dev, &resv_regions)` (per-iommu-driver list — Intel RMRR / AMD UPL / IORT_RMR).
- For each region:
  - `IOMMU_RESV_DIRECT_RELAXABLE`: skip (this is what RMRR-relaxable means).
  - `IOMMU_RESV_MSI`: `cookie_init_hw_msi_region(cookie, region.start, region.start + region.length)`.
  - Else: `reserve_iova(iovad, pfn_lo, pfn_hi)`.

REQ-26: `iommu_dma_get_resv_regions(dev, list)`: per-arch glue — IORT (ARM ACPI) + OF (`of_iommu_get_resv_regions`).

REQ-27: `iommu_setup_dma_ops(dev, domain)`:
- For PCI: `dev.iommu.pci_32bit_workaround = !iommu_dma_forcedac`.
- `dev.dma_iommu = iommu_is_dma_domain(domain)`.
- If `dev.dma_iommu`: `iommu_dma_init_domain(domain, dev)`. On error: warn + `dev.dma_iommu = false`.

REQ-28: `dma_info_to_prot(dir, coherent, attrs)`:
- Base: `MMIO` (if `DMA_ATTR_MMIO`); else `coherent ? IOMMU_CACHE : 0`.
- `DMA_ATTR_PRIVILEGED` → `| IOMMU_PRIV`.
- Direction: BIDIRECTIONAL → `| IOMMU_READ | IOMMU_WRITE`; TO_DEVICE → `| IOMMU_READ`; FROM_DEVICE → `| IOMMU_WRITE`; NONE → 0.

REQ-29: SG bounce decision (`dev_use_sg_swiotlb`):
- True if device is untrusted ∧ any segment has unaligned phys / length, or `swiotlb_force=on`.

REQ-30: Pre-reserved IOVA dma-iova API:
- `dma_iova_try_alloc(dev, state, phys, size)`: alloc combined IOVA `[state.addr, state.addr+size)`; record in `dma_iova_state`. Used by io_uring + dmabuf to amortize IOVA allocation across many `dma_iova_link` calls.
- `dma_iova_link(dev, state, phys, offset, size, dir, attrs)`: `__dma_iova_link(dev, state.addr + offset, phys, size, prot)` calling `iommu_map_nosync` (deferred IOTLB flush).
- `dma_iova_sync(dev, state, offset, size)`: `iommu_sync_map` (commit deferred flush).
- `dma_iova_unlink(dev, state, offset, size, dir, attrs)`: `iommu_unmap_fast` of the range.
- `dma_iova_destroy(dev, state, mapped_len, dir, attrs)`: final teardown — `iommu_unmap_fast` of full range + `iommu_dma_free_iova` of the IOVA span.

REQ-31: Boot params:
- `iommu.forcedac=1` → `iommu_dma_forcedac = true`.
- `iommu_dma_init` (`arch_initcall`): if kdump-kernel: `static_branch_enable(&iommu_deferred_attach_enabled)`. Call `iova_cache_get()` (init the slab caches used by `iova.c`).

## Acceptance Criteria

- [ ] AC-1: `iommu_get_dma_cookie(domain)` twice: second returns -EEXIST.
- [ ] AC-2: `iommu_dma_init_domain(domain, dev)`: idempotent — re-init with same iovad params returns 0; with mismatched granule returns -EFAULT.
- [ ] AC-3: `iommu_dma_alloc_iova(domain, size, dma_limit=0xFFFFFFFF, dev)`: returns IOVA below 4GB when `pci_32bit_workaround` set; falls back to full mask after first 4GB failure + clears flag with `dev_notice`.
- [ ] AC-4: `iommu_dma_map_phys(dev, phys, size, dir, 0)` followed by `iommu_dma_unmap_phys(dev, iova, size, dir, 0)`: IOVA range round-trips through `iommu_map` / `iommu_unmap_fast`; in strict mode, IOTLB sync happens before free.
- [ ] AC-5: `IOMMU_DOMAIN_DMA_FQ` mode: unmap enqueues to FQ; `fq_full` triggers immediate flush + drain; timer drains queue after `IOVA_DEFAULT_FQ_TIMEOUT` ms.
- [ ] AC-6: FQ counter monotonicity: entry inserted with `counter == start_cnt` is freed only when `finish_cnt > counter`.
- [ ] AC-7: `iommu_dma_map_sg(dev, sg, nents, BIDIR, 0)`: returns ≤ nents merged segments; each merged seg has valid `sg_dma_address` / `sg_dma_len`; original `offset` / `length` preserved on rollback.
- [ ] AC-8: SG map with one segment marked `PCI_P2PDMA_MAP_BUS_ADDR`: that segment skips IOMMU-map, uses `pci_p2pdma_bus_addr_map` output.
- [ ] AC-9: `iommu_dma_alloc(dev, size > PAGE_SIZE, ...)`: uses `_alloc_remap` (vmap noncontig); returned cpu_addr is `vmalloc` address; `dma_addr` is single IOVA.
- [ ] AC-10: `iommu_dma_alloc_noncontiguous(dev, size, dir, GFP_KERNEL, 0)`: returns sg_table with pages mapped contiguously in IOVA, possibly non-contiguous in phys.
- [ ] AC-11: `iommu_dma_sw_msi(domain, desc, msi_addr)`: first call for given `msi_addr` allocates a new `iommu_dma_msi_page` and maps `msi_addr` to a new IOVA; subsequent calls with the same `msi_addr` reuse the cached IOVA.
- [ ] AC-12: `iommu_dma_sw_msi` on a domain without dma/msi cookie: `msi_desc_set_iommu_msi_iova(desc, 0, 0)` (no remap); returns 0.
- [ ] AC-13: `iommu_setup_dma_ops(dev, domain)` with non-DMA domain: `dev.dma_iommu = false`; DMA-API falls back to dma-direct.
- [ ] AC-14: PCI host-bridge MMIO windows are pre-reserved in iovad: subsequent `alloc_iova` cannot return an IOVA overlapping those windows.
- [ ] AC-15: Intel RMRR region (advertised as `IOMMU_RESV_DIRECT`) is reserved in iovad → identity-map preserved.
- [ ] AC-16: `iommu_dma_map_phys` with unaligned phys + untrusted device: bounce through swiotlb, then map the bounce buffer's aligned phys.
- [ ] AC-17: `dma_iova_try_alloc(dev, state, phys, size)` + multiple `dma_iova_link(...)` segments: each link increments `iommu_map_nosync` calls; final `dma_iova_destroy` unmaps + frees IOVA exactly once.
- [ ] AC-18: kdump kernel: `iommu_deferred_attach_enabled` static branch on → first DMA triggers attach; non-kdump: attach happens at device-bind, branch off (no per-DMA overhead).

## Architecture

```
struct IommuDmaCookie {
  iovad: IovaDomain,                  // rb-tree + per-cpu magazines
  msi_page_list: List<IommuDmaMsiPage>,
  fq: FqStorage,                       // enum { Single(Box<IovaFq>), Percpu(PerCpu<IovaFq>) }
  fq_flush_start_cnt: AtomicU64,
  fq_flush_finish_cnt: AtomicU64,
  fq_timer: TimerList,
  fq_timer_on: AtomicI32,
  fq_domain: Option<*IommuDomain>,
  options: IommuDmaOptions,
}

struct IommuDmaOptions {
  qt: IommuDmaQueueType,               // PerCpu | Single
  fq_size: usize,                       // 256 (default) | 32768 (single)
  fq_timeout: u32,                      // 10 ms | 1000 ms
}

struct IovaFq {
  lock: SpinLock,
  head: u32,
  tail: u32,
  mod_mask: u32,
  entries: Box<[IovaFqEntry]>,
}

struct IovaFqEntry {
  iova_pfn: u64,
  pages: u64,
  freelist: IommuPagesList,
  counter: u64,
}

struct IommuDmaMsiCookie {
  msi_iova: u64,
  msi_page_list: List<IommuDmaMsiPage>,
}

struct IommuDmaMsiPage {
  iova: u64,
  phys: u64,
}
```

### `DmaIommu::init_domain(domain, dev) -> Result<()>`

1. `cookie = domain.iova_cookie`; require `cookie ∧ cookie_type == DMA_IOVA`.
2. `order = __ffs(domain.pgsize_bitmap)`.
3. Validate `dev.dma_range_map` ⊂ `domain.geometry`.
4. `base_pfn = max(1, domain.geometry.aperture_start >> order)`.
5. If `iovad.start_pfn != 0`: assert match; return Ok.
6. `init_iova_domain(&iovad, 1UL << order, base_pfn)`.
7. `iova_domain_init_rcaches(&iovad)?`.
8. `iommu_dma_init_options(&cookie.options, dev)`.
9. If `domain.type == DMA_FQ`: try `iommu_dma_init_fq(domain)`. On failure: downgrade to `DMA` (strict).
10. `iova_reserve_iommu_regions(dev, domain)?`.

### `DmaIommu::alloc_iova(domain, size, dma_limit, dev) -> u64`

1. If `cookie_type == DMA_MSI`: bump-alloc from `msi_cookie.msi_iova`; return.
2. `shift = iova_shift(iovad)`; `iova_len = size >> shift`.
3. `dma_limit = min_not_zero(dma_limit, dev.bus_dma_limit)`.
4. If `force_aperture`: clamp to `aperture_end`.
5. If `dma_limit > 0xFFFFFFFF ∧ dev.iommu.pci_32bit_workaround`:
   - try `alloc_iova_fast(iovad, iova_len, 0xFFFFFFFF >> shift, false)`.
   - on success: goto done.
   - else: clear flag + notice.
6. `iova = alloc_iova_fast(iovad, iova_len, dma_limit >> shift, true)`.
7. `done`: return `iova << shift`.

### `DmaIommu::map_phys(dev, phys, size, dir, attrs) -> dma_addr_t`

1. `coherent = dev_is_dma_coherent(dev)`; `prot = info_to_prot(dir, coherent, attrs)`.
2. `domain = iommu_get_dma_domain(dev)`; `cookie = domain.iova_cookie`; `iovad = &cookie.iovad`.
3. `dma_mask = dma_get_mask(dev)`.
4. If `dev_use_swiotlb(dev, size, dir) ∧ iova_unaligned(iovad, phys, size)`:
   - if attrs & (MMIO | REQUIRE_COHERENT): return ERROR.
   - `phys = map_swiotlb(dev, phys, size, dir, attrs)`.
   - if `phys == ERROR`: return ERROR.
5. If `!coherent ∧ !(attrs & (SKIP_CPU_SYNC | MMIO))`:
   - `arch_sync_dma_for_device(phys, size, dir)`.
   - `arch_sync_dma_flush()`.
6. `iova = __iommu_dma_map(dev, phys, size, prot, dma_mask)`.
7. If `iova == ERROR ∧ !(attrs & (MMIO | REQUIRE_COHERENT))`: `swiotlb_tbl_unmap_single(dev, phys, size, dir, attrs)`.
8. Return iova.

### `DmaIommu::queue_iova(cookie, pfn, pages, freelist)` (FQ enqueue)

1. `smp_mb()`.
2. Pick `fq`: single → `cookie.single_fq`; percpu → `raw_cpu_ptr(percpu_fq)`.
3. `spin_lock_irqsave(&fq.lock, flags)`.
4. `fq_ring_free_locked(cookie, fq)` (drain already-flushed).
5. If `fq_full(fq)`:
   - `fq_flush_iotlb(cookie)`.
   - `fq_ring_free_locked(cookie, fq)`.
6. `idx = fq_ring_add(fq)`.
7. `fq.entries[idx] = { pfn, pages, freelist=splice(freelist), counter=read(fq_flush_start_cnt) }`.
8. `spin_unlock_irqrestore(&fq.lock, flags)`.
9. If `!fq_timer_on.read() ∧ !fq_timer_on.xchg(1)`: `mod_timer(&fq_timer, jiffies + msecs_to_jiffies(options.fq_timeout))`.

### `DmaIommu::sw_msi(domain, desc, msi_addr) -> Result<()>`

1. If `!has_msi_cookie(domain)`:
   - `msi_desc_set_iommu_msi_iova(desc, 0, 0)` (no remap).
   - Return Ok.
2. `iommu_group_mutex_assert(dev)` (caller serialization).
3. `msi_page = get_msi_page(dev, msi_addr, domain)?`.
4. `msi_desc_set_iommu_msi_iova(desc, msi_page.iova, ilog2(cookie_msi_granule(domain)))`.

### FQ cookie sizing decision (`iommu_dma_init_options`)

| Condition | qt | fq_size | fq_timeout (ms) |
|---|---|---|---|
| default IOMMU_DOMAIN_DMA_FQ | PER_CPU | 256 | 10 |
| `iommu.strict=0` ∧ many CPUs | PER_CPU | 256 | 10 |
| Intel ATS / per-driver hint | SINGLE | 32768 | 1000 |

### Map-attr → IOMMU prot translation (`dma_info_to_prot`)

| DMA_ATTR | IOMMU prot bit |
|---|---|
| (none) coherent device | `IOMMU_CACHE` |
| `DMA_ATTR_MMIO` | `IOMMU_MMIO` (no CACHE) |
| `DMA_ATTR_PRIVILEGED` | `| IOMMU_PRIV` |
| `DMA_BIDIRECTIONAL` | `| IOMMU_READ | IOMMU_WRITE` |
| `DMA_TO_DEVICE` | `| IOMMU_READ` |
| `DMA_FROM_DEVICE` | `| IOMMU_WRITE` |

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cookie_type_exclusive` | INVARIANT | per-`get_dma_cookie` / `get_msi_cookie`: cookie_type transitions NONE → one-of-{DMA_IOVA, DMA_MSI}, never both. |
| `iovad_granule_immutable_after_init` | INVARIANT | per-`init_domain`: once `iovad.start_pfn != 0`, granule fixed. |
| `fq_ring_head_tail_modular` | INVARIANT | per-`fq_ring_add`: `(tail+1) & mod_mask != head` (no overrun); `(tail - head) & mod_mask < fq_size`. |
| `fq_counter_monotonic` | INVARIANT | per-`fq_flush_iotlb`: `start_cnt` ≥ `finish_cnt` always; both increment exactly once per call. |
| `fq_entry_freed_after_flush` | INVARIANT | per-`fq_ring_free_locked`: entry freed only when `entry.counter < finish_cnt`. |
| `iova_alloc_below_dma_limit` | INVARIANT | per-`alloc_iova`: returned `iova + size ≤ dma_limit + 1`. |
| `map_unmap_round_trip` | INVARIANT | per-`map_phys` + `unmap_phys`: same iova, IOVA freed exactly once. |
| `swiotlb_only_when_unaligned_or_untrusted` | INVARIANT | per-`dev_use_swiotlb`: enabled iff `is_swiotlb_force_bounce ∨ dev_is_untrusted ∨ size > swiotlb_max`. |
| `msi_page_cached` | INVARIANT | per-`get_msi_page`: at most one entry per phys-aligned `msi_addr` in `msi_page_list`. |
| `pci_32bit_fallback_one_shot` | INVARIANT | per-`alloc_iova`: `pci_32bit_workaround` cleared at most once, never re-set. |
| `cookie_freed_via_put` | INVARIANT | per-`put_dma_cookie`: msi_page_list drained ∧ iovad torn down ∧ fq freed before kfree(cookie). |
| `sg_map_finalise_count_correct` | INVARIANT | per-`__finalise_sg`: returned count ≤ nents ∧ every output seg has DMA_MAPPING_ERROR no-overwrite. |
| `dma_iova_link_within_state` | INVARIANT | per-`__dma_iova_link`: `state.addr + offset + size ≤ state.addr + state.size`. |
| `fq_timer_armed_once_per_dirty` | INVARIANT | per-`queue_iova`: `fq_timer_on` atomic-xchg ensures single armer. |

### Layer 2: TLA+

`drivers/iommu/dma-iommu.tla`:
- Per-`map_phys` + `unmap_phys` interleaved with FQ-timer firing on N producers + 1 timer-CPU.
- Per-`map_sg` + concurrent `unmap_sg` from different drivers on same domain.
- Per-cookie-lifecycle: `get` → `init_domain` → ops → `put`.
- Per-MSI-doorbell cache: many devices on same domain mapping same `msi_addr`.
- Properties:
  - `safety_no_iova_double_free` — `free_iova` never called twice for same `(iova, size)`.
  - `safety_no_unmap_before_flush` — strict mode: `iommu_iotlb_sync` always before `iommu_dma_free_iova`.
  - `safety_fq_entry_iotlb_after_unmap` — FQ mode: `iommu_unmap_fast` happens-before `fq_flush_iotlb` that gates entry's free.
  - `safety_msi_page_unique` — for any (cookie, phys), at most one `msi_page` in list.
  - `liveness_fq_eventually_drains` — every FQ entry is eventually freed (timer fires within `fq_timeout` ms).
  - `liveness_alloc_iova_eventually_returns` — `alloc_iova` either returns IOVA or 0 (no spinning).
  - `safety_p2pdma_bus_addr_skips_iommu` — sg segments with `PCI_P2PDMA_MAP_BUS_ADDR` are not handed to `iommu_map`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `get_dma_cookie` post: `cookie_type == DMA_IOVA` ∧ `iova_cookie != NULL` | `DmaIommu::get_dma_cookie` |
| `put_dma_cookie` post: msi_page_list empty ∧ iovad freed ∧ cookie freed | `DmaIommu::put_dma_cookie` |
| `init_domain` post: `iovad.granule = 1 << __ffs(domain.pgsize_bitmap)` ∧ host-bridge windows reserved | `DmaIommu::init_domain` |
| `init_fq` post (success): `cookie.fq_domain == domain` ∧ timer initialized ∧ ring(s) allocated | `DmaIommu::init_fq` |
| `init_fq` post (fail): `domain.type` downgraded to `DMA` ∧ no FQ resources allocated | `DmaIommu::init_fq` |
| `alloc_iova` post: `iova == 0` ∨ `iova ≥ aperture_start ∧ iova + size ≤ min(dma_limit, aperture_end) + 1` | `DmaIommu::alloc_iova` |
| `map_phys` post: returns `DMA_MAPPING_ERROR` ∨ `iommu_iova_to_phys(domain, iova) == phys` | `DmaIommu::map_phys` |
| `unmap_phys` post: range no longer in domain page tables; IOVA returned to allocator (strict) or queue (FQ) | `DmaIommu::unmap_phys` |
| `map_sg` post: returns ≤ nents; for each output seg, IOVA range contiguous + maps to original phys range | `DmaIommu::map_sg` |
| `queue_iova` post: entry appended; timer armed if not already | `DmaIommu::queue_iova` |
| `fq_ring_free_locked` post: entries with `counter < finish_cnt` removed; head advanced | `DmaIommu::fq_ring_free_locked` |
| `sw_msi` post (DMA cookie): msi_desc.iommu_iova set to cached msi_page.iova | `DmaIommu::sw_msi` |
| `sw_msi` post (no cookie): msi_desc.iommu_iova cleared | `DmaIommu::sw_msi` |
| `setup_dma_ops` post: `dev.dma_iommu == iommu_is_dma_domain(domain)` ∧ init_domain attempted | `DmaIommu::setup_dma_ops` |

### Layer 4: Verus/Creusot functional

`Per-streaming map: dma_map_page → iommu_dma_map_phys → __iommu_dma_map → iommu_map; device DMA → iommu translates iova → phys; dma_unmap_page → iommu_dma_unmap_phys → iommu_unmap_fast → IOTLB-sync or fq-queue` semantic equivalence: `Documentation/core-api/dma-api.rst`, VT-d spec §3 + AMD-Vi spec §1.5.

`Per-MSI remap: device generates MSI to msi_addr → IOMMU translates msi_addr (per iommu_dma_msi_page mapping) → CPU LAPIC sees the correct doorbell` semantic equivalence: VT-d spec §5.1 (Interrupt Remapping) + AMD-Vi spec §2.2.5.

`Per-FQ flush correctness: deferred IOVA free held until at least one full IOTLB-flush cycle has completed (counter > entry.counter)` semantic equivalence: lazy-unmap correctness paper (Pfaff et al. 2012) + Intel VT-d errata for ATS invalidation.

`Per-noncontig coherent alloc: iommu_dma_alloc_noncontiguous → vmap of high-order pages, single IOVA covering all` semantic equivalence: drm-noncontig dmabuf semantics.

`Per-pre-reserved iova: dma_iova_try_alloc + dma_iova_link sequence equivalent to single dma_map_sg over same physical regions` semantic equivalence: io_uring zero-copy + dmabuf streaming proposals.

## Hardening

(Inherits row-1 features from `drivers/iommu/00-overview.md` § Hardening.)

dma-iommu reinforcement:

- **Per-cookie-type exclusivity** — defense against per-double-use of a domain as both DMA-IOVA and MSI cookie (would corrupt msi_page_list).
- **Per-iovad-granule immutable after first init** — defense against per-late-attach with different pgsize_bitmap shrinking iova alignment under live mappings.
- **Per-PCI-host-bridge-window pre-reserve** — defense against per-allocator returning an IOVA that overlaps p2p-routing space (would deliver bus-translated DMA to wrong endpoint).
- **Per-IOMMU-resv-region honored** — defense against per-RMRR-stomp on Intel pre-boot firmware regions (BIOS / UEFI may still DMA there after boot).
- **Per-`iommu_dma_msi_page` cache + identity-map for static MSI regions** — defense against per-MSI-routing-bypass.
- **Per-FQ counter monotone-acquire-release** — defense against per-premature-free of an IOVA whose IOTLB invalidation hasn't completed (would let a stale device-write hit a freshly-reused page).
- **Per-`fq_full` immediate flush** — defense against per-FQ overflow silently dropping IOVAs.
- **Per-FQ timer arm-once** — defense against per-spurious-CPU-flood of timer reprogramming.
- **Per-swiotlb-fallback for untrusted devices** — defense against per-CoCo-guest DMA outside the encrypted memory region.
- **Per-`pci_32bit_workaround` one-shot drop** — defense against per-firmware/hw bug with 64-bit IOVAs (still in 2024 some MSI controllers fail on >32-bit DAC).
- **Per-`iommu_deferred_attach` static-key kdump-only** — defense against per-DMA-overhead in healthy boot.
- **Per-p2pdma bus-addr bypass** — defense against per-IOMMU-double-translation of peer-to-peer DMA (which must stay bus-routed).
- **Per-`dma_info_to_prot` direction-strict** — defense against per-driver-bug mapping write-only / read-only buffers with wrong direction (would skip cache invalidation on the wrong side).
- **Per-cookie msi_page_list mutex via iommu_group_mutex** — defense against per-concurrent msi_desc set+get racing get_msi_page.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- IOVA rb-tree + magazine allocator body (covered in `iova.md` Tier-3)
- Per-vendor IOMMU driver attach + map/unmap implementations (covered in `intel-iommu.md`, `intel-cache.md`, `intel-pasid.md`, `intel-svm.md`, `amd/iommu.md` etc.)
- iommufd UAPI (covered in `iommufd-*.md` Tier-3 set)
- swiotlb internals (covered in `kernel/dma/` Tier-2 — `swiotlb.c`)
- DMA-API public surface (`dma_map_*` etc.) — covered in `kernel/dma/` Tier-2
- mmu-notifier / SVA invalidation (covered in `iommu-sva.md` Tier-3)
- io-pgfault PRI handling (covered separately)
- ARM-SMMU / ARM-SMMU-v3 (out of v0; ARM-only)
- Implementation code
