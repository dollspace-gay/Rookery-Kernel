# Tier-3: kernel/dma/swiotlb.c — SoftWare IOTLB bounce-buffer (restricted-DMA-mask + SEV/TDX confidential-guest forced path)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/dma/00-overview.md
upstream-paths:
  - kernel/dma/swiotlb.c
  - include/linux/swiotlb.h
-->

## Summary

The software bounce-buffer used when a device's DMA mask cannot reach a buffer's physical address (legacy 32-bit DMA NIC trying to DMA above 4GB) OR when the host RAM is encrypted (SEV/SEV-ES/SEV-SNP/TDX confidential guest forces every DMA through a shared unencrypted buffer pool). Allocates a contiguous reserved region at boot (default 64MB on x86_64, configurable via `swiotlb=` cmdline) split into per-NUMA areas; per-area slab allocator hands out aligned slot ranges; map+sync copies between the original buffer and the slab. Critical security boundary: the only DMA path on confidential guests.

This Tier-3 covers `kernel/dma/swiotlb.c` (~1900 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct io_tlb_mem` | per-pool control block (default + per-device restricted) | `kernel::dma::swiotlb::IoTlbMem` |
| `struct io_tlb_area` | per-NUMA-area sub-pool | `IoTlbArea` |
| `struct io_tlb_pool` | per-pool slot-tracking | `IoTlbPool` |
| `struct io_tlb_slot` | per-slot orig_addr + alloc_size | `IoTlbSlot` |
| `swiotlb_init(addressing_limit, flags)` | early-boot init: alloc default pool from memblock | `swiotlb::init` |
| `swiotlb_init_late(size, gfp, remap)` | late-boot init for fallback | `swiotlb::init_late` |
| `swiotlb_init_remap(addressing_limit, flags, remap)` | with arch-remap callback for SEV/TDX (memblock_alloc + force-shared-pages) | `swiotlb::init_remap` |
| `swiotlb_size_or_default()` | resolve `swiotlb=N` cmdline | `swiotlb::size_or_default` |
| `swiotlb_init_io_tlb_pool(pool, start, nslabs, late_alloc, nareas)` | initialize a pool's slot table | `IoTlbPool::init` |
| `swiotlb_tbl_map_single(dev, orig_addr, size, alloc_size, mapping_size, dir, attrs)` | allocate slot + bounce | `IoTlbMem::map_single` |
| `swiotlb_tbl_unmap_single(dev, tlb_addr, mapping_size, dir, attrs)` | sync + free slot | `IoTlbMem::unmap_single` |
| `swiotlb_sync_single_for_cpu(dev, tlb_addr, size, dir)` / `_for_device` | per-direction sync | `IoTlbMem::sync_for_cpu` / `_for_device` |
| `is_swiotlb_buffer(dev, paddr)` | is this paddr in a swiotlb pool? | `IoTlbMem::contains_buffer` |
| `is_swiotlb_force_bounce(dev)` | forced bounce mode (SEV/TDX confidential) | `swiotlb::is_force_bounce` |
| `swiotlb_alloc(dev, size)` / `_free(dev, vaddr, size)` | direct allocator from pool (for dma_alloc_attrs DIRECT path) | `IoTlbMem::alloc` / `_free` |
| `swiotlb_max_mapping_size(dev)` | per-device max single mapping | `IoTlbMem::max_mapping_size` |
| `default_swiotlb_base()` / `_limit()` | default pool address range | `swiotlb::default_base` / `_limit` |
| `swiotlb_dev_init(dev)` | per-device restricted-DMA-pool init from DT | `Device::swiotlb_init` |
| `swiotlb_print_info()` | boot-time print of pool size | `swiotlb::print_info` |
| `dma_swiotlb_active(dev)` | is dev using swiotlb? | `Device::swiotlb_active` |

## Compatibility contract

REQ-1: `swiotlb=N[,force][,noforce]` cmdline parsed identically; default size 64MB on x86_64, 0 on most ARM (cross-ref per-arch defaults).

REQ-2: Per-pool slot table allocates aligned regions: per-slot 2KB minimum (IO_TLB_SHIFT=11), aligned to power-of-two via per-mapping `alloc_size` and `iommu_offset`.

REQ-3: Map+unmap semantics: `map_single(addr, size)` copies `[addr, addr+size)` into a slab slot if direction is TO_DEVICE / BIDIRECTIONAL; returns slab paddr. `unmap_single(slab_paddr, size)` copies back if FROM_DEVICE / BIDIRECTIONAL, frees slot.

REQ-4: Sync semantics: `sync_for_device` copies orig→slab for TO_DEVICE / BIDIRECTIONAL; `sync_for_cpu` copies slab→orig for FROM_DEVICE / BIDIRECTIONAL.

REQ-5: Per-NUMA area split for scaling: each NUMA node gets its own area; per-area lock reduces contention vs single global lock.

REQ-6: SEV/SEV-ES/SEV-SNP/TDX confidential-guest detection: per-arch `cc_platform_has(CC_ATTR_GUEST_MEM_ENCRYPT)` returns true → swiotlb forced (`is_swiotlb_force_bounce` returns true regardless of dma_mask); arch-remap callback ensures pool pages are unencrypted shared.

REQ-7: Per-device restricted-DMA-pool: DT `restricted-dma-pool` reserved-memory entry + `memory-region = <&pool>` ref → per-device pool used instead of global default.

REQ-8: Debug surface: `/sys/kernel/debug/swiotlb/io_tlb_used` (current in-use slot count) + `/sys/kernel/debug/swiotlb/io_tlb_used_hiwater` (high-water; writable to reset).

REQ-9: Tracepoint `events/swiotlb/swiotlb_bounced` byte-identical (consumed by perf for diagnostic).

REQ-10: Out-of-slots behavior: `map_single` returns DMA_MAPPING_ERROR; per-arch fallback (e.g., DMA-coherent allocator failure → -ENOMEM); WARN_ONCE logged on first occurrence per-pool.

## Acceptance Criteria

- [ ] AC-1: Boot with `swiotlb=8192,force` cmdline → `dmesg | grep "software IO TLB"` shows 16MB pool size + force-bounce mode.
- [ ] AC-2: 32-bit-DMA-mask test driver allocates above-4GB RAM + DMA via swiotlb bounce; payload integrity preserved.
- [ ] AC-3: SEV-SNP guest with `mem=8G`: every device DMA goes via swiotlb (verify `cat /sys/kernel/debug/swiotlb/io_tlb_used` non-zero under load).
- [ ] AC-4: NUMA-area scaling test: 2-socket system, per-area lock contention < 1% under sustained DMA from both sockets.
- [ ] AC-5: Out-of-slots stress: pool sized 64KB, sustained DMA from 100 concurrent threads → some get DMA_MAPPING_ERROR but no UAF/corruption.
- [ ] AC-6: Tracepoint `swiotlb:swiotlb_bounced` fires on every bounce; perf record decodes correctly.
- [ ] AC-7: Per-device restricted-pool test: DT-described per-device pool used instead of global default.

## Architecture

`IoTlbMem` lives in `kernel::dma::swiotlb::IoTlbMem`:

```
struct IoTlbMem {
  default_pool: KBox<IoTlbPool>,    // primary pool
  pools: AtomicPtr<KBox<IoTlbPool>>,// linked-list of additional pools (dynamic-grow)
  capacity: AtomicUsize,             // total slot count
  busy: AtomicUsize,                 // in-use slot count
  hiwater: AtomicUsize,              // high-water
  for_alloc: bool,                   // direct-allocator backing
  force_bounce: bool,                // SEV/TDX forced
  defer_init: bool,
  total_used: AtomicU64,             // tracepoint accounting
}

struct IoTlbPool {
  start: u64,                        // physical start
  end: u64,                          // physical end
  vaddr: NonNull<u8>,                // mapped virtual addr (for memcpy)
  nslabs: u32,
  used: AtomicUsize,
  areas: VarLenArray<IoTlbArea>,
  late_alloc: bool,                  // alloc'd via vmalloc vs memblock
  next_pool: AtomicPtr<KBox<IoTlbPool>>,
}

struct IoTlbArea {
  index_lock: SpinLock<()>,           // per-area lock
  list_index: AtomicU32,              // bitmap rotation index
  used: AtomicUsize,
  slots: VarLenArray<IoTlbSlot>,
}

struct IoTlbSlot {
  orig_addr: u64,                     // original buffer paddr (or 0 if free)
  alloc_size: u32,                    // slot count for this allocation
  list: u32,                          // free-list link / slot count to next free
}
```

`swiotlb::init(addressing_limit, flags)`:
1. Read `swiotlb=` cmdline via `swiotlb_size_or_default`.
2. `memblock_alloc(size)` → contiguous physical region.
3. If `cc_platform_has(CC_ATTR_GUEST_MEM_ENCRYPT)` AND remap callback supplied: `arch_remap(start, end)` → mark pages as shared (SEV C-bit clear OR TDX SHARED-bit set).
4. `IoTlbPool::init(pool, start, nslabs=size/IO_TLB_SHIFT, false, nareas)`.
5. Set `mem->default_pool`, `mem->force_bounce` from cmdline `force`/`noforce`.
6. Print info to dmesg.

`IoTlbMem::map_single(dev, orig_addr, mapping_size, alloc_size, dir, attrs)`:
1. `area_idx = current_cpu % nareas` (per-CPU area selection for cache locality).
2. Take `area.index_lock`.
3. `find_free_slot(area, alloc_size, dev_required_alignment)`:
   - Iterate slots starting at `area.list_index` (rotating allocation for fairness).
   - Look for run of `alloc_size` free slots with required alignment.
   - On hit: mark slots as used (`slot.list = 0`, `slot.orig_addr = orig_addr`, `slot.alloc_size = alloc_size`).
   - On miss (no free run of required size): release lock, try next area, after all-area exhaustion → grow pool (if dynamic) or return DMA_MAPPING_ERROR.
4. `area.used += alloc_size`; `mem->busy += alloc_size`; update `mem->hiwater` if exceeded.
5. Drop `area.index_lock`.
6. If `dir == TO_DEVICE || BIDIRECTIONAL`: `memcpy(slot_vaddr, orig_addr, mapping_size)` (via vaddr stored in pool — pool was vmap'd at init).
7. Tracepoint: `trace_swiotlb_bounced(dev, dev_addr, mapping_size, force_bounce ? FORCE : NORMAL)`.
8. Return slot's physical addr.

`IoTlbMem::unmap_single(dev, tlb_addr, mapping_size, dir, attrs)`:
1. Find pool + area + slot index from tlb_addr.
2. If `dir == FROM_DEVICE || BIDIRECTIONAL` AND not `attrs & DMA_ATTR_SKIP_CPU_SYNC`: `memcpy(orig_addr, slot_vaddr, mapping_size)`.
3. Take `area.index_lock`.
4. Mark slots free; rebuild free-list pointer; clear orig_addr.
5. `area.used -= alloc_size`; `mem->busy -= alloc_size`.
6. Drop lock.

Sync ops (`sync_for_device`, `sync_for_cpu`): direction-conditional memcpy without changing slot allocation.

SEV/TDX confidential-guest detection: at boot, `cc_platform_has(CC_ATTR_GUEST_MEM_ENCRYPT)` returns true; arch-remap callback (`x86_swiotlb_remap = set_memory_decrypted`) marks pool pages as unencrypted (clears C-bit on SEV, sets SHARED-bit on TDX). Subsequent `dma_map_single(dev, addr, sz)`: even if `dev->dma_mask` reaches the entire address space, `is_swiotlb_force_bounce` returns true, so map goes through bounce path.

Per-device restricted-DMA-pool: parse DT `memory-region` ref → `swiotlb_init_io_tlb_mem` with per-device IoTlbMem; `dev->dma_io_tlb_mem` points at this instead of global default.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `slot_no_uaf` | UAF | per-slot orig_addr cleared on unmap; subsequent map of same slot writes new orig_addr; concurrent map+unmap on disjoint slots safe via per-area lock. |
| `area_idx_no_oob` | OOB | `area_idx = cpu % nareas` always within bounds; per-area slot-iteration bounded by area.nslabs. |
| `slot_alloc_no_overflow` | OVERFLOW | alloc_size + offset arithmetic checked against area capacity. |
| `bounce_no_oob` | OOB | memcpy bounded by `mapping_size`; `mapping_size <= alloc_size * IO_TLB_SIZE`. |

### Layer 2: TLA+

`models/dma/swiotlb_alloc.tla` (parent-declared): proves per-area slab allocator under N concurrent CPUs + alloc + free + sync — no double-allocation of the same slab range, no leaked slab on driver-unbind path; per-cpu padding + per-area lock contention bounded.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `IoTlbMem::map_single` post: returned slot is owned by exactly this allocation; no other allocation owns same slot range; orig_addr correctly set | `IoTlbMem::map_single` |
| `IoTlbMem::unmap_single` post: slot range freed; orig_addr cleared; data sync'd back if FROM_DEVICE | `IoTlbMem::unmap_single` |
| `area.used` invariant: equals sum over (slot.alloc_size for slot in area where slot.orig_addr != 0) | `IoTlbArea` |

### Layer 4: Verus/Creusot functional

`IoTlbMem::map_single(dev, addr, sz)` ↔ `IoTlbMem::unmap_single(dev, slab, sz)` round-trip equivalence: post-`unmap`, `[addr, addr+sz)` contains same bytes as pre-`map` if dir ∈ {NONE, TO_DEVICE}; contains device-written bytes if dir ∈ {FROM_DEVICE, BIDIRECTIONAL}. Encoded as Verus invariant on the buffer + slot abstract state.

## Hardening

(Inherits row-1 features from `kernel/dma/00-overview.md` § Hardening.)

swiotlb-specific reinforcement:

- **Confidential-guest forced-bounce default-on** — when SEV/TDX detected at boot, every DMA bounced regardless of dma_mask (defense against host-RAM leakage via direct DMA bypassing encryption).
- **Per-area lock contention bounded** — per-NUMA-node area split; per-area lock; per-cpu start-area selection for cache locality.
- **Tracepoint accounting** — `swiotlb_bounced` for every bounce; per-pool used + hiwater visible in debugfs (operator monitoring).
- **Out-of-slots detection** — WARN_ONCE per-pool on first DMA_MAPPING_ERROR; subsequent errors logged at rate-limited level.
- **Slot-padding for IOMMU IRQ-remap window** — per-pool reserved-region exclusion ensures swiotlb pool doesn't overlap with MSI-X window or IOMMU reserved-regions.
- **Per-mapping size cap** — `swiotlb_max_mapping_size(dev)` returns per-pool single-mapping cap; defense against single-DMA-of-the-entire-pool starvation.
- **Pool size cap** — `swiotlb_max_pool_size = 1GB` default; defense against `swiotlb=` cmdline asking for absurd pool size.
- **memcpy via vmapped pool addr** — pool mapped once at init via `memremap`; map/unmap operations don't repeatedly map/unmap (fast path).

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- DMA-API dispatch (covered in `kernel/dma/mapping.md` future Tier-3)
- DMA-direct path (covered in `kernel/dma/direct.md` future Tier-3)
- DMA-IOMMU path (covered in `kernel/dma/dma-iommu.md` future Tier-3)
- CMA allocator (covered in `kernel/dma/contiguous.md` future Tier-3)
- Per-arch SEV/TDX C-bit / SHARED-bit handling (covered in `arch/x86/sev.md` future Tier-3)
- 32-bit-only paths
- Implementation code
