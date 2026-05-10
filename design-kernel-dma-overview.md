---
title: "Tier-2: kernel/dma — DMA-API (mapping + direct + swiotlb + coherent + contiguous + pool + remap + dma-buf consumer)"
tags: ["tier-2", "kernel", "dma", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 overview for the kernel DMA-mapping subsystem — every device driver that touches DMA (NVMe, network, storage, GPU, sound, USB-host, I2C/SPI bus controllers, …) consumes through `<linux/dma-mapping.h>`. The framework bridges three completely different DMA models behind one API:

- **dma-direct**: host RAM directly addressable by the device — no IOMMU, no bounce; just `phys_to_dma()` / `dma_to_phys()` with optional sync ops on non-coherent arches
- **swiotlb**: bounce-buffer fallback for devices with restricted DMA mask (32-bit-only NIC, ISA DMA channels, restricted-DMA SoC peripherals); also the SEV/TDX confidential-guest path (host RAM is encrypted, bounce-buffer is the unencrypted shared page)
- **dma-iommu**: IOMMU-translated DMA — addresses returned are IOVAs, not physical (cross-ref `drivers/iommu/00-overview.md`)

Plus orthogonal facilities consumed by drivers across all three backends:

- **dma-coherent**: per-device coherent (uncached on non-coherent arches; cache-coherent on x86) memory allocator with optional per-device dedicated mempool from device-tree `memory-region` or `dma-coherent` pools
- **contiguous (CMA)**: large physically-contiguous allocations (default global CMA pool + per-device CMA areas); used by camera/video/HDMI capture/GPU backing-store needing physically-contiguous swaths; cross-ref `mm/00-overview.md` migrate machinery
- **pool**: tiny-coherent allocator (`dma_pool_*` family) backed by a slab-like sub-allocator over `dma_alloc_coherent`
- **remap**: per-page virtual remapping for DMA buffers needing cache-attribute changes (write-combine, uncached) on architectures lacking PAT-style attribute control
- **map_benchmark**: in-tree microbenchmark module for DMA-API throughput regression testing
- **debug**: DMA-API debug layer (CONFIG_DMA_API_DEBUG=y) — tracks every map/unmap to detect lost-unmap, double-unmap, sync-without-map, alloc/map size mismatch, leak on driver unbind
- **dma-buf**: cross-driver buffer-sharing fd (V4L2 capture → DRM display, vfio-pci BAR → DRM, NVDEC → V4L2 encoder, etc.); the kernel-wide buffer-handle abstraction that makes pipelines like `gst-launch v4l2src ! kmssink` work zero-copy

Heavily cross-referenced from every device-driver Tier-2 (NVMe, NIC, GPU, storage, USB, sound, video capture, DRM, V4L2, sound, …), `drivers/iommu/00-overview.md` (`dma-iommu.c` is the IOMMU backend; cross-ref'd in IOMMU Tier-2), `arch/x86/cpu-init.md` (PAT + cache-attribute modes), `arch/x86/mm-pgtable.md` (per-page memory-type mapping for non-coherent uncached allocations), `mm/00-overview.md` (CMA migration + coherent_pool from `dma_pool` PageBlock claims), `virt/kvm/00-overview.md` (SEV/TDX confidential guest forced-swiotlb path).

### Out of Scope

- Per-Tier-3 (Phase D)
- `drivers/dma/` DMA-engine controller framework (separate Tier-2 wrapper future) — this is the controller-driver framework for DMA-engine peripherals (Intel-IOAT, OMAP, Synopsys-PL330, …), distinct from the DMA-mapping API
- ARM-specific non-coherent cache-maintenance ops (out of v0)
- s390 / PPC zPCI DMA quirks
- 32-bit-only paths
- Implementation code

### components

- **mapping** (`mapping.c`): top-level DMA-API dispatch — `dma_map_single` / `dma_unmap_single` / `dma_map_page` / `dma_map_sg` / `dma_alloc_coherent` / `dma_free_coherent` / `dma_alloc_attrs` / `dma_alloc_pages` / `dma_alloc_noncoherent` / `dma_get_required_mask` / `dma_set_mask` / `dma_set_coherent_mask` / `dma_set_min_align_mask` / `dma_get_merge_boundary` / `dma_can_mmap` / `dma_mmap_attrs` / `dma_get_sgtable_attrs` / `dma_need_sync` / `dma_max_mapping_size` / `dma_opt_mapping_size` / `dma_addressing_limited`. Per-device `dma_map_ops` selection (direct vs iommu vs per-bus override).
- **direct** (`direct.c` + `direct.h`): dma-direct ops — phys_to_dma / dma_to_phys translation; `dma_direct_alloc` / `dma_direct_map_page` / `dma_direct_map_sg` / `dma_direct_sync_*`; bounce-via-swiotlb fallback when device DMA-mask doesn't reach allocation
- **swiotlb** (`swiotlb.c`): software bounce buffer — global (default 64MB at boot, x86_64) + optional per-device "restricted DMA pool" + per-numa-node area split for scaling (CONFIG_SWIOTLB); the only DMA path on SEV/TDX confidential guests since host RAM is encrypted
- **coherent** (`coherent.c`): per-device dedicated coherent pool registered via `dma_declare_coherent_memory` (used by some embedded SoCs to back coherent allocs from on-chip SRAM)
- **contiguous** (`contiguous.c`): CMA — Contiguous Memory Allocator; reserves a large boot-time region (default 0; configurable via `cma=` cmdline + `CMA_SIZE_MBYTES` Kconfig); per-device CMA areas via DT `memory-region`
- **pool** (`pool.c`): atomic-coherent-pool for `dma_alloc_coherent(.., GFP_ATOMIC)` from interrupt context — pre-allocated since allocation in atomic context can't sleep
- **remap** (`remap.c`): `dma_common_pages_remap` / `dma_common_contiguous_remap` — page-array → vmalloc-region with cache attributes (UC/WC); used by GFP_DMA32 + non-coherent allocations
- **debug** (`debug.c` + `debug.h`): DMA-API tracking layer; per-cpu shadow of every active mapping; reports lost-unmap on driver unbind; CONFIG_DMA_API_DEBUG=y
- **map_benchmark** (`map_benchmark.c`): debugfs-driven microbenchmark for `dma_map_*` throughput
- **ops_helpers** (`ops_helpers.c`): per-arch `dma_map_ops` plumbing helpers
- **dummy** (`dummy.c`): no-DMA-supported stub for devices with no DMA capability
- **dma-buf** (separate top-level subdir at `drivers/dma-buf/` but conceptually part of DMA stack — keep listed here): exporter API (`dma_buf_export` / `dma_buf_attach` / `dma_buf_map_attachment` / `dma_buf_unmap_attachment` / `dma_buf_begin_cpu_access` / `dma_buf_end_cpu_access` / `dma_buf_mmap`), reservation-object (dma_resv) for fence-based synchronization, sync-file (per-fd waiter), heaps (system + cma + secure-cma DMA-buf heap allocators), udmabuf (userspace-allocated dmabuf for cross-process buffer sharing), debug surface

### scope

This Tier-2 governs:
- `/home/doll/linux-src/kernel/dma/` (~14 source files)
- Public headers `include/linux/{dma-mapping, dma-direct, dma-map-ops, swiotlb, dma-fence*, dma-resv, dma-buf}.h` + UAPI `include/uapi/linux/{dma-buf, dma-heap, udmabuf, sync_file}.h`
- `/home/doll/linux-src/drivers/dma-buf/` (~10 source files — dma-buf core + heaps + udmabuf + sync_file) — conceptually grouped here at Tier-2 since dma-buf is the kernel's cross-driver DMA buffer-handle abstraction; future may carve into a separate Tier-2 if drivers/dma-buf/ grows much

`drivers/dma/` (DMA-engine subsystem — DMA-controller drivers like Intel-IOAT, OMAP, Synopsys-PL330; consumed by SPI/I2C/sound) is a separate Tier-2 wrapper future (it's the DMA-engine controller framework, distinct from the DMA-mapping API documented here).

### compatibility contract — outline

### `<linux/dma-mapping.h>` API source compat

In-tree drivers consume via this header. Source-compatible — every documented function preserved with same signature + same semantics. Ones that move (e.g., the deprecated `pci_alloc_consistent` → `dma_alloc_coherent` migration is already done upstream) stay deprecated.

### Per-device DMA mask

`dma_set_mask(dev, DMA_BIT_MASK(N))` + `dma_set_coherent_mask` + `dma_set_min_align_mask`. In-tree drivers depend on consistent semantics: mask too narrow → fall back to swiotlb on x86_64 (or fail on systems without swiotlb).

### swiotlb cmdline + sysfs

- `swiotlb=N[,force][,noforce]` — bounce-buffer slab count + force/noforce
- `/sys/kernel/debug/swiotlb/io_tlb_used` — in-use slabs (high-water tracking)
- `/sys/kernel/debug/swiotlb/io_tlb_used_hiwater` (writable: reset)

Wire format byte-identical (sosreport + sar consume).

### CMA cmdline + sysfs

- `cma=N[MG]` — global CMA size
- `cma_pernuma=N[MG]` — per-node CMA size
- `/sys/kernel/debug/cma/<name>/{count, used, alloc_pages_success, alloc_pages_fail, ...}` — per-CMA-area counters

### `/proc/iomem` DMA reservations

CMA + per-device coherent pools + swiotlb regions appear in `/proc/iomem` with byte-identical labels (`Reserved` / `cma area` / `dma-coherent`).

### `dma-buf` UAPI

- `DMA_BUF_IOCTL_SYNC` — `dma_buf_sync.flags` (DMA_BUF_SYNC_READ / _WRITE / _START / _END / _RW)
- `DMA_BUF_SET_NAME_A` (deprecated 32B) / `DMA_BUF_SET_NAME_B` (32B-ASCIIZ — current)
- `DMA_BUF_IOCTL_EXPORT_SYNC_FILE` / `_IMPORT_SYNC_FILE` (sync-file integration)

Wire-format byte-identical so V4L2 / DRM / GStreamer / Android-graphics consume unchanged.

### `dma-heap` UAPI (`/dev/dma_heap/<name>`)

- `DMA_HEAP_IOCTL_ALLOC` — request a dma-buf from a heap (system / cma / system-uncached)

Per-heap chardev byte-identical.

### `udmabuf` UAPI (`/dev/udmabuf`)

- `UDMABUF_CREATE` / `UDMABUF_CREATE_LIST` — wrap memfd pages as dma-buf (used by qemu virtio-gpu + virtio-video for cross-process zero-copy)

Wire format byte-identical.

### `sync_file` UAPI

- `SYNC_IOC_MERGE`, `SYNC_IOC_FILE_INFO`, `SYNC_IOC_FENCE_INFO`

Wire format byte-identical so Android-graphics fence-pipeline works.

### dma-fence kernel API

`<linux/dma-fence.h>` — fence + reservation-object (dma_resv) + per-fence callback chain + signaling. Source-compatible for in-tree GPU/V4L2 drivers.

### `dma_alloc_*` GFP-flag handling

`dma_alloc_coherent(dev, size, &handle, gfp)` — on x86_64 with iommu-passthrough, falls through to allocate from kernel page allocator + apply DMA mask; with iommu-default, allocates from iommu IOAS. Behavior identical to upstream — drivers don't observe the difference.

### Per-arch `dma_map_ops`

For x86_64: chooses dma-direct if no IOMMU, dma-iommu otherwise. Per-device override possible via `set_dma_ops(dev, ops)` (Hyper-V uses this). Source-compat preserved.

### CONFIG_DMA_API_DEBUG

Optional but commonly enabled. When on, every map/unmap is shadowed; warnings on:
- unmap of unmapped region
- double-unmap
- sync of unmapped region
- alloc/map size mismatch
- map of address overlapping kernel text
- driver unbind with mappings still active

Behavior + warning text identical (consumed by automated regression testing + perf-debug profiles).

### tier-3 docs governed by this tier-2

(Phase D will add these incrementally.)

| Tier-3 doc | Scope |
|---|---|
| `kernel/dma/mapping.md` | `mapping.c`: top-level DMA-API dispatch + per-device dma_map_ops selection |
| `kernel/dma/direct.md` | `direct.c` + `direct.h`: dma-direct ops + phys↔dma translation + bounce fallback |
| `kernel/dma/swiotlb.md` | `swiotlb.c`: software bounce-buffer (incl. SEV/TDX confidential-guest path) |
| `kernel/dma/coherent.md` | `coherent.c`: per-device dedicated coherent pools |
| `kernel/dma/contiguous.md` | `contiguous.c`: CMA — Contiguous Memory Allocator |
| `kernel/dma/pool.md` | `pool.c`: atomic-coherent-pool for GFP_ATOMIC dma_alloc_coherent |
| `kernel/dma/remap.md` | `remap.c`: per-page virtual remap for cache-attribute-changed DMA buffers |
| `kernel/dma/debug.md` | `debug.c` + `debug.h`: DMA-API debug tracking layer |
| `kernel/dma/map-benchmark.md` | `map_benchmark.c`: in-tree DMA-API microbenchmark |
| `kernel/dma/ops-helpers.md` | `ops_helpers.c`: per-arch dma_map_ops plumbing |
| `kernel/dma/dummy.md` | `dummy.c`: no-DMA stub |
| `kernel/dma/dmabuf-core.md` | `drivers/dma-buf/dma-buf.c` + `dma-resv.c` + `dma-fence*.c`: dma-buf exporter/importer + reservation-object + fence chain |
| `kernel/dma/dmabuf-heaps.md` | `drivers/dma-buf/heaps/`: dma-heap framework + system / cma / system-uncached heaps |
| `kernel/dma/dmabuf-udmabuf.md` | `drivers/dma-buf/udmabuf.c`: userspace-allocated dma-buf wrapper |
| `kernel/dma/dmabuf-sync-file.md` | `drivers/dma-buf/sync_file.c`: sync-file fd for fence pipelines |

### compatibility outline (top-level)

- REQ-O1: `<linux/dma-mapping.h>` API source-compat for in-tree drivers (every documented function preserved).
- REQ-O2: dma-direct + swiotlb + dma-iommu backends select transparently per-device based on dma_mask + IOMMU presence; behavior matches upstream.
- REQ-O3: `swiotlb=` + `cma=` + `cma_pernuma=` cmdline options parsed identically; default values match upstream.
- REQ-O4: `/sys/kernel/debug/swiotlb/` + `/sys/kernel/debug/cma/` debugfs counters byte-identical.
- REQ-O5: `/proc/iomem` DMA reservation labels byte-identical.
- REQ-O6: SEV/SEV-ES/SEV-SNP/TDX confidential-guest forced-swiotlb path works (bounce buffer is unencrypted shared page).
- REQ-O7: dma-buf UAPI byte-identical (V4L2 / DRM / GStreamer / Android-graphics consume unchanged).
- REQ-O8: dma-heap UAPI + udmabuf UAPI + sync_file UAPI byte-identical.
- REQ-O9: dma-fence + dma-resv kernel API source-compat for in-tree GPU/V4L2 drivers.
- REQ-O10: CONFIG_DMA_API_DEBUG warnings byte-identical text + same trigger conditions.
- REQ-O11: Per-device CMA (DT `memory-region`) + per-device coherent pool (`dma_declare_coherent_memory`) source-compat.
- REQ-O12: TLA+ models declared at this Tier-2 (swiotlb slab allocator concurrency, CMA migration race, dma-buf attach/detach + reservation-object fence chaining).
- REQ-O13: Verus/Creusot Layer-4 functional contracts on dma_alloc/free pairing + map/unmap pairing + dma-buf refcount.
- REQ-O14: Hardening: row-1 features applied per `00-security-principles.md`, with DMA-specific reinforcement (forced-swiotlb default-on for confidential-guest, dma-buf import LSM mediation, CMA per-device pool capacity limit).

### acceptance criteria (top-level)

- [ ] AC-O1: NVMe driver `dma_map_single` + `dma_unmap_single` round-trip works at full PCIe Gen4 throughput on reference HW. (covers REQ-O1, REQ-O2)
- [ ] AC-O2: 32-bit-DMA-mask test driver allocates above-4GB RAM + DMA via swiotlb bounce; payload integrity preserved. (covers REQ-O2)
- [ ] AC-O3: SEV-SNP guest with `mem=8G`: every device DMA goes via swiotlb (verify `cat /sys/kernel/debug/swiotlb/io_tlb_used` non-zero under load). (covers REQ-O6)
- [ ] AC-O4: `cma=128M` cmdline reserves 128MB CMA; `cat /sys/kernel/debug/cma/<name>/count` shows 32768 pages. (covers REQ-O3)
- [ ] AC-O5: GStreamer pipeline `v4l2src ! kmssink` (V4L2 capture into DRM display via dma-buf) achieves zero-copy. (covers REQ-O7)
- [ ] AC-O6: `dma-heap-test` userspace tool allocates from `/dev/dma_heap/system`; subsequently mmap'd + read/written. (covers REQ-O8)
- [ ] AC-O7: `udmabuf` test: memfd-create + UDMABUF_CREATE + import into qemu virtio-gpu; cross-process buffer share works. (covers REQ-O8)
- [ ] AC-O8: CONFIG_DMA_API_DEBUG=y test: a buggy driver missing dma_unmap on remove triggers WARN with byte-identical text vs upstream. (covers REQ-O10)
- [ ] AC-O9: kselftest `tools/testing/selftests/dma-buf/` passes. (covers REQ-O7, REQ-O12)
- [ ] AC-O10: kselftest `tools/testing/selftests/dmabuf-heaps/` passes. (covers REQ-O8, REQ-O12)
- [ ] AC-O11: `map_benchmark` debugfs throughput within 5% of upstream baseline on dma-direct + dma-iommu paths. (covers REQ-O1, REQ-O12)
- [ ] AC-O12: dma-fence chain test: 3-fence chain with merged sync-file resolves in correct order; signal callbacks invoked exactly once each. (covers REQ-O9)

### verification (top-level)

### Layer 2: TLA+ models — mandatory list

| Model | Owned by |
|---|---|
| `models/dma/swiotlb_alloc.tla` | `kernel/dma/swiotlb.md` (proves: per-area swiotlb slab allocator under N concurrent CPUs + alloc + free + sync — no double-allocation of the same slab range, no leaked slab on driver-unbind path; per-cpu padding + per-area lock contention bounded) |
| `models/dma/cma_migrate.tla` | `kernel/dma/contiguous.md` (proves: CMA region migration — when CMA pages are temporarily holding non-CMA allocations and a CMA-allocator request arrives, migrate-and-isolate handshake completes; concurrent isolate + alloc never double-claims a page) |
| `models/dma/dmabuf_attach.tla` | `kernel/dma/dmabuf-core.md` (proves: dma-buf attach + map_attachment + unmap + detach + close lifecycle: refcount integrity across N concurrent importers; reservation-object fence chain + signal observe acquire-release) |
| `models/dma/dma_resv_lock.tla` | `kernel/dma/dmabuf-core.md` (proves: dma-resv ww-mutex chain locking with deadlock-avoidance via wound-wait — N consumers acquiring a chain of M dmabufs converge without deadlock) |
| `models/dma/dma_pool_atomic.tla` | `kernel/dma/pool.md` (proves: atomic-coherent-pool refill from process context + concurrent atomic-context alloc never depletes pool past floor before refill; high-water + refill-trigger correctly maintained) |

### Layer 4: Functional contracts (Verus / Creusot)

| Component | Contract topic |
|---|---|
| `kernel/dma/mapping.md` | `dma_map_single(dev, ptr, sz, dir)` post-condition: returned dma_addr is device-addressable per dev->dma_mask; `dma_unmap_single(dev, addr, sz, dir)` invariant: addr was previously returned by matching dma_map_single (debug layer enforces) |
| `kernel/dma/direct.md` | `dma_direct_alloc(dev, sz, &handle, gfp, attrs)` post: returned ptr is page-aligned, sized at least `sz`, addressable per dev->dma_mask; `dma_direct_free` accepts only previously-allocated handles |
| `kernel/dma/dmabuf-core.md` | `dma_buf_get(fd)` invariant: returned dma_buf has refcount >= 1; `dma_buf_put` decrements; close-during-attach safe |

### hardening (top-level)

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this Tier-2

| Feature | Default inherited from |
|---|---|
| **REFCOUNT** | per-dma_buf + per-dma_attachment + per-dma_fence + per-cma-area refcounts use `Refcount` (saturating) | § Mandatory |
| **CONSTIFY** | per-arch `dma_map_ops`, per-driver dma_buf `dma_buf_ops`, dma-heap `dma_heap_ops` `static const` | § Mandatory |
| **SIZE_OVERFLOW** | DMA-mapping size + offset arithmetic + CMA page-count + swiotlb slab arithmetic uses checked operators | § Mandatory |
| **RAP / FORTIFY_RW** | per-arch dma_map_ops vtables + per-driver dma_buf_ops vtables read-only-after-init | § Mandatory |
| **MEMORY_SANITIZE** | freed CMA pages cleared (cross-driver buffer reuse may leak prior driver's data); freed dma-buf state cleared | § Default-on configurable off |
| **STRICT_KERNEL_RWX** | DMA-API has no JIT — N/A | § Mandatory |

### Row-2 (LSM-stackable) features for DMA

LSM hooks called: `security_dma_buf_*` (proposed for Rookery; upstream coverage minimal). File-LSM hooks on `/dev/dma_heap/*` open + ioctl, `/dev/udmabuf` open + ioctl, sync-file fd open.

GR-RBAC adds:
- Per-role disallow `/dev/dma_heap/*` open (denies heap-from-userspace allocation).
- Per-role disallow `/dev/udmabuf` open (denies userspace-allocated dma-buf wrapping).
- Per-role audit of every dma-buf import-by-fd (cross-process sharing), with source pid + fd inode.
- Per-role disallow `cma_pernuma=` cmdline override (root-only at boot).

### DMA-specific reinforcement

- **Forced-swiotlb default-on for confidential-guest**: in SEV/TDX detected mode, every dma_map path forced through swiotlb regardless of device dma_mask (host RAM is encrypted; bounce-buffer is the unencrypted shared page).
- **Per-device CMA capacity limit**: per-device CMA pool size capped (default 1G) to prevent device-tree-supplied or userspace-supplied huge per-device pools from exhausting CMA reserve.
- **swiotlb force-bounce for vfio devices**: when device is bound to vfio-pci with iommu-passthrough disabled, force swiotlb to prevent userspace driver from DMA-ing into kernel memory regions outside its allowed range. (Compose with iommufd IOAS bounds; this is defense-in-depth.)
- **dma-buf import LSM mediation**: every `dma_buf_get(fd)` from a different process namespace mediated; cross-namespace buffer share requires explicit role grant.
- **dma-fence callback bounded**: per-fence signal callback list bounded (default 64 callbacks); attempt to add 65th returns -ENOSPC with WARN (defense against fence-callback DoS).
- **udmabuf inode-list size cap**: `UDMABUF_CREATE_LIST` accepts at most 1024 page-list entries (defense against memory exhaustion via huge list).
- **CMA page-attribute restoration**: on CMA page release, restore default cache attributes (defense against pages being returned to global pool with prior driver's WC/UC attrs leaking).
- **DMA-API debug WARN escalates to BUG when CONFIG_DMA_API_DEBUG_PARANOID=y** (Rookery-specific Kconfig — defense against silently ignored unmap mismatches in production builds; off by default but available for hardened deployments).

