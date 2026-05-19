# Tier-3: kernel/dma/coherent.c — Per-device coherent DMA pool

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/dma/00-overview.md
upstream-paths:
  - kernel/dma/coherent.c (~410 lines)
  - include/linux/dma-map-ops.h (dma_declare_coherent_memory, dma_release_coherent_memory)
  - Documentation/core-api/dma-api.rst (§ Part Ie - Optional: device-managed coherent memory)
-->

## Summary

A **per-device coherent DMA pool** is dedicated memory (usually on-die SRAM, a reserved DDR window, or device-local scratchpad) that the device can DMA into without snooping the system bus and without the CPU needing cache invalidate/flush. The driver declares the pool once via `dma_declare_coherent_memory(dev, phys_addr, device_addr, size)`; subsequently every `dma_alloc_coherent(dev, ...)` first tries `dma_alloc_from_dev_coherent` against this pool before falling back to the generic allocator. Per-pool data: `struct dma_coherent_mem` (virt_base, device_base, pfn_base, size in pages, bitmap, spinlock, use_dev_dma_pfn_offset). Per-`dma_init_coherent_memory` ioremaps the physical window (`memremap(..., MEMREMAP_WC)`) and allocates a `bitmap` over pages. Per-`__dma_alloc_from_coherent` finds a free contiguous run via `bitmap_find_free_region(order)`; per-`__dma_release_from_coherent` releases via `bitmap_release_region`. Per-`__dma_mmap_from_coherent` maps the pool into a userspace VMA via `remap_pfn_range`. Per-CONFIG_DMA_GLOBAL_POOL adds a system-wide `dma_coherent_default_memory` used when a device has no per-device pool. Per-CONFIG_OF_RESERVED_MEM + `shared-dma-pool` DT binding attaches the pool to devices at probe. Critical for: embedded SoCs with on-die SRAM, audio DSPs, GPU framebuffer scratchpads, devices with `coherent_dma_mask` narrower than system DRAM, and drivers that must guarantee bounce-free DMA.

This Tier-3 covers `kernel/dma/coherent.c` (~410 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct dma_coherent_mem` | per-pool control block | `Coherent::DmaCoherentMem` |
| `dev_get_coherent_memory(dev)` | per-device pool accessor | `Coherent::dev_get_mem` |
| `dma_get_device_base(dev, mem)` | per-`use_dev_dma_pfn_offset` device addr resolver | `Coherent::device_base` |
| `dma_init_coherent_memory(phys, dev_addr, size, use_dma_pfn_offset)` | per-pool ioremap + bitmap alloc | `Coherent::init_memory` |
| `_dma_release_coherent_memory(mem)` | per-pool memunmap + bitmap free + kfree | `Coherent::release_memory_inner` |
| `dma_assign_coherent_memory(dev, mem)` | per-device pool attach | `Coherent::assign_memory` |
| `dma_declare_coherent_memory(dev, phys, dev_addr, size)` | per-platform declare | `Coherent::declare_memory` |
| `dma_release_coherent_memory(dev)` | per-device pool detach | `Coherent::release_memory` |
| `__dma_alloc_from_coherent(dev, mem, size, dma_handle)` | per-pool alloc | `Coherent::alloc_from_mem` |
| `dma_alloc_from_dev_coherent(dev, size, dma_handle, ret)` | per-device alloc wrapper | `Coherent::alloc_from_dev` |
| `__dma_release_from_coherent(mem, order, vaddr)` | per-pool free | `Coherent::release_from_mem` |
| `dma_release_from_dev_coherent(dev, order, vaddr)` | per-device free wrapper | `Coherent::release_from_dev` |
| `__dma_mmap_from_coherent(mem, vma, vaddr, size, ret)` | per-pool mmap | `Coherent::mmap_from_mem` |
| `dma_mmap_from_dev_coherent(dev, vma, vaddr, size, ret)` | per-device mmap wrapper | `Coherent::mmap_from_dev` |
| `dma_coherent_default_memory` | per-CONFIG_DMA_GLOBAL_POOL system-wide pool | `Coherent::DEFAULT_MEMORY` |
| `dma_alloc_from_global_coherent(dev, size, dma_handle)` | per-global alloc | `Coherent::alloc_from_global` |
| `dma_release_from_global_coherent(order, vaddr)` | per-global free | `Coherent::release_from_global` |
| `dma_mmap_from_global_coherent(vma, vaddr, size, ret)` | per-global mmap | `Coherent::mmap_from_global` |
| `dma_init_global_coherent(phys, size)` | per-CONFIG_DMA_GLOBAL_POOL init | `Coherent::init_global` |
| `rmem_dma_device_init(rmem, dev)` | per-DT shared-dma-pool device attach | `Coherent::rmem_device_init` |
| `rmem_dma_device_release(rmem, dev)` | per-DT shared-dma-pool device detach | `Coherent::rmem_device_release` |
| `rmem_dma_setup(node, rmem)` | per-DT shared-dma-pool node setup | `Coherent::rmem_setup` |
| `dma_init_reserved_memory()` | per-core-initcall global-pool init | `Coherent::init_reserved_memory` |

## Compatibility contract

REQ-1: struct dma_coherent_mem:
- virt_base: void * — `memremap` kernel-virtual base for CPU access.
- device_base: dma_addr_t — DMA address the device must be programmed with.
- pfn_base: unsigned long — `PFN_DOWN(phys_addr)` for `remap_pfn_range`.
- size: int — pool size in pages (`size_bytes >> PAGE_SHIFT`).
- bitmap: unsigned long * — `bitmap_zalloc(pages)` per-page allocation state.
- spinlock: spinlock_t — per-pool alloc/free serialization.
- use_dev_dma_pfn_offset: bool — when true, `device_base` is computed from `phys_to_dma(dev, PFN_PHYS(pfn_base))` at alloc time (DT shared-pool case); when false, `device_base` is the literal value passed to `dma_declare_coherent_memory` (platform-code static case).

REQ-2: dma_init_coherent_memory(phys_addr, device_addr, size, use_dma_pfn_offset):
- /* Reject zero size */
- if !size: return ERR_PTR(-EINVAL).
- /* Map pool via memremap WC (write-combine, no caching) */
- mem_base = memremap(phys_addr, size, MEMREMAP_WC).
- if !mem_base: return ERR_PTR(-EINVAL).
- /* Allocate control struct + bitmap */
- dma_mem = kzalloc_obj(struct dma_coherent_mem).
- if !dma_mem: goto out_unmap_membase.
- dma_mem->bitmap = bitmap_zalloc(pages, GFP_KERNEL).
- if !dma_mem->bitmap: goto out_free_dma_mem.
- /* Populate fields */
- dma_mem->virt_base = mem_base.
- dma_mem->device_base = device_addr.
- dma_mem->pfn_base = PFN_DOWN(phys_addr).
- dma_mem->size = pages.
- dma_mem->use_dev_dma_pfn_offset = use_dma_pfn_offset.
- spin_lock_init(&dma_mem->spinlock).
- return dma_mem.
- /* Error paths log "Reserved memory: failed to init DMA memory pool at %pa" */

REQ-3: _dma_release_coherent_memory(mem):
- if !mem: return.
- memunmap(mem->virt_base).
- bitmap_free(mem->bitmap).
- kfree(mem).

REQ-4: dma_assign_coherent_memory(dev, mem):
- if !dev: return -ENODEV.
- if dev->dma_mem: return -EBUSY (one pool per device).
- dev->dma_mem = mem.
- return 0.

REQ-5: dma_declare_coherent_memory(dev, phys_addr, device_addr, size):
- mem = dma_init_coherent_memory(phys_addr, device_addr, size, use_dma_pfn_offset=false).
- if IS_ERR(mem): return PTR_ERR(mem).
- ret = dma_assign_coherent_memory(dev, mem).
- if ret: _dma_release_coherent_memory(mem) /* rollback */.
- return ret.

REQ-6: dma_release_coherent_memory(dev):
- if !dev: return.
- _dma_release_coherent_memory(dev->dma_mem).
- dev->dma_mem = NULL.

REQ-7: __dma_alloc_from_coherent(dev, mem, size, dma_handle):
- order = get_order(size).
- spin_lock_irqsave(&mem->spinlock).
- /* Reject if too big */
- if size > (mem->size << PAGE_SHIFT): goto err.
- /* Find free run of 2^order pages via bitmap */
- pageno = bitmap_find_free_region(mem->bitmap, mem->size, order).
- if pageno < 0: goto err.
- /* Resolve device address */
- *dma_handle = dma_get_device_base(dev, mem) + (pageno << PAGE_SHIFT).
- ret = mem->virt_base + (pageno << PAGE_SHIFT).
- spin_unlock_irqrestore.
- memset(ret, 0, size).
- return ret.
- err: spin_unlock_irqrestore; return NULL.

REQ-8: dma_alloc_from_dev_coherent(dev, size, dma_handle, ret):
- mem = dev_get_coherent_memory(dev) /* dev->dma_mem */.
- if !mem: return 0 /* caller proceeds with generic alloc */.
- *ret = __dma_alloc_from_coherent(dev, mem, size, dma_handle).
- return 1 /* caller returns *ret to user */.

REQ-9: __dma_release_from_coherent(mem, order, vaddr):
- /* Confirm vaddr lies within pool */
- if !mem ∨ vaddr < mem->virt_base ∨ vaddr ≥ mem->virt_base + (mem->size << PAGE_SHIFT): return 0.
- page = (vaddr - mem->virt_base) >> PAGE_SHIFT.
- spin_lock_irqsave(&mem->spinlock).
- bitmap_release_region(mem->bitmap, page, order).
- spin_unlock_irqrestore.
- return 1.

REQ-10: dma_release_from_dev_coherent(dev, order, vaddr):
- mem = dev_get_coherent_memory(dev).
- return __dma_release_from_coherent(mem, order, vaddr) /* 0 = caller uses generic free */.

REQ-11: __dma_mmap_from_coherent(mem, vma, vaddr, size, ret):
- /* Confirm vaddr+size lies within pool */
- if !mem ∨ vaddr < mem->virt_base ∨ vaddr + size > mem->virt_base + (mem->size << PAGE_SHIFT): return 0.
- off = vma->vm_pgoff.
- start = (vaddr - mem->virt_base) >> PAGE_SHIFT.
- user_count = vma_pages(vma).
- count = PAGE_ALIGN(size) >> PAGE_SHIFT.
- *ret = -ENXIO.
- if off < count ∧ user_count ≤ count - off:
  - pfn = mem->pfn_base + start + off.
  - *ret = remap_pfn_range(vma, vma->vm_start, pfn, user_count << PAGE_SHIFT, vma->vm_page_prot).
- return 1 /* caller returns *ret */.

REQ-12: dma_mmap_from_dev_coherent(dev, vma, vaddr, size, ret):
- mem = dev_get_coherent_memory(dev).
- return __dma_mmap_from_coherent(mem, vma, vaddr, size, ret).

REQ-13: Per-CONFIG_DMA_GLOBAL_POOL global pool:
- dma_coherent_default_memory: `__ro_after_init` static, set by `dma_init_global_coherent`.
- dma_alloc_from_global_coherent(dev, size, dma_handle):
  - if !dma_coherent_default_memory: return NULL.
  - return __dma_alloc_from_coherent(dev, …, size, dma_handle).
- dma_release_from_global_coherent(order, vaddr):
  - if !dma_coherent_default_memory: return 0.
  - return __dma_release_from_coherent(dma_coherent_default_memory, order, vaddr).
- dma_mmap_from_global_coherent(vma, vaddr, size, ret):
  - if !dma_coherent_default_memory: return 0.
  - return __dma_mmap_from_coherent(dma_coherent_default_memory, vma, vaddr, size, ret).
- dma_init_global_coherent(phys, size):
  - mem = dma_init_coherent_memory(phys, phys, size, use_dma_pfn_offset=true) /* device_base = phys */.
  - if IS_ERR(mem): return PTR_ERR(mem).
  - dma_coherent_default_memory = mem.
  - pr_info "DMA: default coherent area is set".
  - return 0.

REQ-14: Per-CONFIG_OF_RESERVED_MEM `shared-dma-pool` DT binding:
- rmem_dma_setup(node, rmem):
  - if reusable: return -ENODEV (cma path).
  - per-CONFIG_ARM: require `no-map` else "regions without no-map are not yet supported" + -EINVAL.
  - per-CONFIG_DMA_GLOBAL_POOL: if `linux,dma-default` set, capture `rmem->base` + `rmem->size` into `dma_reserved_default_memory_base/size`; warn if redefined.
  - pr_info "Reserved memory: created DMA memory pool".
  - return 0.
- rmem_dma_device_init(rmem, dev):
  - if !rmem->priv: mem = dma_init_coherent_memory(rmem->base, rmem->base, rmem->size, use_dma_pfn_offset=true).
  - if IS_ERR(mem): return PTR_ERR.
  - rmem->priv = mem.
  - /* Range-warn if device's coherent_dma_mask/bus_dma_limit can't reach */
  - if mem->device_base + rmem->size - 1 > min_not_zero(dev->coherent_dma_mask, dev->bus_dma_limit): dev_warn "reserved memory is beyond device's set DMA address range".
  - dma_assign_coherent_memory(dev, mem).
  - return 0.
- rmem_dma_device_release(rmem, dev):
  - if dev: dev->dma_mem = NULL.
- RESERVEDMEM_OF_DECLARE(dma, "shared-dma-pool", &rmem_dma_ops).
- dma_init_reserved_memory() core_initcall: if `dma_reserved_default_memory_size`, call dma_init_global_coherent.

REQ-15: Per-pool semantics:
- One pool per device (`dma_assign_coherent_memory` enforces `!dev->dma_mem` precondition).
- `memremap(..., MEMREMAP_WC)` — write-combine attributes; no CPU cache snooping required.
- Allocation granularity: PAGE_SIZE × 2^order via bitmap (power-of-two run only).
- Allocator zeros buffer (`memset(ret, 0, size)`) post-alloc.
- Alloc + release fully under `mem->spinlock` (irq-saved).

REQ-16: Return convention:
- `dma_alloc_from_dev_coherent` returns 1 to mean "we handled it; use `*ret`"; 0 means "fall through to generic allocator".
- `dma_release_from_dev_coherent` returns 1 to mean "we freed it"; 0 means "fall through to generic free".
- `dma_mmap_from_dev_coherent` returns 1 to mean "we handled it; *ret is the remap result"; 0 means "fall through".

## Acceptance Criteria

- [ ] AC-1: Platform code calls `dma_declare_coherent_memory(dev, phys=0x4000_0000, dev_addr=0x4000_0000, size=64K)` → returns 0; `dev->dma_mem != NULL`.
- [ ] AC-2: Second `dma_declare_coherent_memory` on same `dev` → returns `-EBUSY`; `dev->dma_mem` unchanged.
- [ ] AC-3: `dma_alloc_coherent(dev, 4K, &handle, GFP_KERNEL)` against a 64K declared pool → `handle == dev_addr + page_offset`, returned vaddr is zero-initialized.
- [ ] AC-4: `dma_alloc_coherent(dev, 128K, …)` against 64K pool → fall-back path: `dma_alloc_from_dev_coherent` returns 1 with `*ret == NULL`; per-arch `dma_alloc_coherent` returns NULL to caller (not generic-fallback).
- [ ] AC-5: `dma_free_coherent(dev, 4K, vaddr, handle)` → bitmap bit cleared; subsequent `dma_alloc_coherent(dev, 4K, …)` may reuse same `handle`.
- [ ] AC-6: `dma_mmap_coherent(dev, vma, vaddr, size)` → userspace VMA mapped via `remap_pfn_range` against `mem->pfn_base + offset`; userspace reads of zeroed allocation see zeros.
- [ ] AC-7: `dma_release_coherent_memory(dev)` → `dev->dma_mem == NULL`; pool memory unmapped (`memunmap`) and bitmap freed; subsequent `dma_alloc_coherent` falls back to generic allocator.
- [ ] AC-8: DT `shared-dma-pool` reserved-memory entry with `memory-region = <&pool>` referenced by device → at probe, `dev->dma_mem` points at per-pool `dma_coherent_mem`; `device_base = phys_to_dma(dev, PFN_PHYS(pfn_base))`.
- [ ] AC-9: DT pool with `device_base + size - 1 > dev->coherent_dma_mask` → dev_warn "reserved memory is beyond device's set DMA address range" but attach still succeeds.
- [ ] AC-10: CONFIG_DMA_GLOBAL_POOL: DT `linux,dma-default` entry → `dma_coherent_default_memory != NULL` after `core_initcall`; `dma_alloc_from_global_coherent` returns from this pool when device has no per-device pool.
- [ ] AC-11: Concurrent `dma_alloc_coherent` + `dma_free_coherent` from two threads on same device → bitmap consistent (no double-allocation, no leaked bit); serialized by `mem->spinlock`.
- [ ] AC-12: Pool fragmentation: 64K pool, alloc 4×8K then free middle two, then alloc 16K → 16K alloc succeeds via `bitmap_find_free_region(order=2)`.

## Architecture

```
struct DmaCoherentMem {
  virt_base: NonNull<u8>,               // memremap WC
  device_base: u64,                      // dma_addr_t programmed into device
  pfn_base: u64,                         // PFN_DOWN(phys_addr)
  size: u32,                             // pages
  bitmap: KBox<BitMap>,                  // size bits
  spinlock: SpinLock<()>,
  use_dev_dma_pfn_offset: bool,          // true: device_base = phys_to_dma(dev, PFN_PHYS(pfn_base))
}
```

`Coherent::init_memory(phys_addr, device_addr, size, use_dma_pfn_offset) -> Result<KBox<DmaCoherentMem>>`:
1. if size == 0: return Err(EINVAL).
2. pages = size >> PAGE_SHIFT.
3. virt_base = memremap(phys_addr, size, MEMREMAP_WC).ok_or(EINVAL)?.
4. mem = KBox::new_zeroed_in(KernelAlloc)?.
5. mem.bitmap = BitMap::new_zero(pages).map_err(|_| /* unmap virt_base */ ENOMEM)?.
6. mem.virt_base = virt_base.
7. mem.device_base = device_addr.
8. mem.pfn_base = phys_addr >> PAGE_SHIFT.
9. mem.size = pages.
10. mem.use_dev_dma_pfn_offset = use_dma_pfn_offset.
11. mem.spinlock = SpinLock::new(()).
12. Ok(mem).

`Coherent::declare_memory(dev, phys, dev_addr, size) -> i32`:
1. mem = Coherent::init_memory(phys, dev_addr, size, false)?.
2. ret = Coherent::assign_memory(dev, mem).
3. if ret != 0: Coherent::release_memory_inner(mem) /* rollback */.
4. return ret.

`Coherent::assign_memory(dev, mem) -> i32`:
1. if !dev: return -ENODEV.
2. if dev.dma_mem.is_some(): return -EBUSY.
3. dev.dma_mem = Some(mem).
4. return 0.

`Coherent::release_memory(dev)`:
1. if !dev: return.
2. Coherent::release_memory_inner(dev.dma_mem.take()).
3. dev.dma_mem = None.

`Coherent::release_memory_inner(mem)`:
1. if mem.is_none(): return.
2. memunmap(mem.virt_base).
3. drop(mem.bitmap).
4. drop(mem) /* kfree */.

`Coherent::alloc_from_dev(dev, size, dma_handle) -> Option<NonNull<u8>>`:
1. mem = Coherent::dev_get_mem(dev)?.
2. order = get_order(size).
3. let _g = mem.spinlock.lock_irqsave().
4. if size > (mem.size << PAGE_SHIFT): drop(_g); return None.
5. pageno = bitmap_find_free_region(&mut mem.bitmap, mem.size, order).
6. if pageno < 0: drop(_g); return None.
7. *dma_handle = Coherent::device_base(dev, mem) + (pageno << PAGE_SHIFT).
8. ret = mem.virt_base.add(pageno << PAGE_SHIFT).
9. drop(_g).
10. memset(ret, 0, size).
11. Some(ret).

`Coherent::device_base(dev, mem) -> dma_addr_t`:
1. if mem.use_dev_dma_pfn_offset:
   - return phys_to_dma(dev, mem.pfn_base << PAGE_SHIFT).
2. else: return mem.device_base.

`Coherent::release_from_dev(dev, order, vaddr) -> i32`:
1. mem = Coherent::dev_get_mem(dev).
2. if mem.is_none() ∨ vaddr < mem.virt_base ∨ vaddr ≥ mem.virt_base + (mem.size << PAGE_SHIFT):
   - return 0 /* caller uses generic free */.
3. page = (vaddr - mem.virt_base) >> PAGE_SHIFT.
4. let _g = mem.spinlock.lock_irqsave().
5. bitmap_release_region(&mut mem.bitmap, page, order).
6. drop(_g).
7. return 1.

`Coherent::mmap_from_dev(dev, vma, vaddr, size, ret) -> i32`:
1. mem = Coherent::dev_get_mem(dev).
2. if mem.is_none() ∨ vaddr < mem.virt_base ∨ vaddr + size > mem.virt_base + (mem.size << PAGE_SHIFT):
   - return 0.
3. off = vma.vm_pgoff.
4. start = (vaddr - mem.virt_base) >> PAGE_SHIFT.
5. user_count = vma_pages(vma).
6. count = PAGE_ALIGN(size) >> PAGE_SHIFT.
7. *ret = -ENXIO.
8. if off < count ∧ user_count ≤ (count - off):
   - pfn = mem.pfn_base + start + off.
   - *ret = remap_pfn_range(vma, vma.vm_start, pfn, user_count << PAGE_SHIFT, vma.vm_page_prot).
9. return 1.

`Coherent::rmem_device_init(rmem, dev) -> i32` (CONFIG_OF_RESERVED_MEM):
1. mem = rmem.priv.
2. if mem.is_none():
   - mem = Coherent::init_memory(rmem.base, rmem.base, rmem.size, use_dma_pfn_offset=true)?.
   - rmem.priv = Some(mem).
3. /* Range warn */
4. if mem.device_base + rmem.size - 1 > min_not_zero(dev.coherent_dma_mask, dev.bus_dma_limit):
   - dev_warn(dev, "reserved memory is beyond device's set DMA address range").
5. Coherent::assign_memory(dev, mem).
6. return 0.

`Coherent::init_global(phys, size) -> i32` (CONFIG_DMA_GLOBAL_POOL):
1. mem = Coherent::init_memory(phys, phys, size, use_dma_pfn_offset=true)?.
2. DEFAULT_MEMORY = Some(mem).
3. pr_info("DMA: default coherent area is set").
4. return 0.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `bitmap_no_oob` | OOB | per-`alloc_from_dev`: `pageno + (1 << order) ≤ mem.size`. |
| `vaddr_range_check_strict` | OOB | per-`release_from_dev` / `mmap_from_dev`: vaddr lies fully within `[virt_base, virt_base + size << PAGE_SHIFT)` before use. |
| `mmap_pgoff_within_pool` | OOB | per-`mmap_from_dev`: `vma.vm_pgoff < count ∧ user_count ≤ count - off`. |
| `assign_no_double_attach` | INVARIANT | per-`assign_memory`: `dev.dma_mem` must be None before assign; second attach returns -EBUSY. |
| `spinlock_held_for_bitmap_mutation` | INVARIANT | per-`alloc_from_dev` / `release_from_dev`: `mem.spinlock` held across `bitmap_find_free_region` / `bitmap_release_region`. |
| `init_failure_no_leak` | INVARIANT | per-`init_memory`: bitmap-alloc failure → `memunmap(virt_base)` + `kfree(mem)` on error path. |
| `declare_failure_rolls_back` | INVARIANT | per-`declare_memory`: `assign_memory` failure → `_release_memory(mem)` rollback. |

### Layer 2: TLA+

`models/dma/coherent.tla`:
- Per-N concurrent alloc + free + mmap + declare + release-device on a single device pool.
- Properties:
  - `safety_no_double_allocate` — same bitmap bit never simultaneously owned by two allocations.
  - `safety_no_double_free` — `bitmap_release_region` on an already-free run is forbidden by allocator preconditions.
  - `safety_one_pool_per_device` — `dma_assign_coherent_memory` rejected when `dev.dma_mem` non-NULL.
  - `safety_pool_release_clears_dev` — `dma_release_coherent_memory` always sets `dev.dma_mem = NULL` before freeing pool.
  - `safety_size_cap_enforced` — alloc with `size > mem.size << PAGE_SHIFT` returns NULL without touching bitmap.
  - `safety_mmap_within_pool` — `remap_pfn_range` PFN range fully within `[pfn_base, pfn_base + size)`.
  - `liveness_alloc_returns_or_null` — every `alloc_from_dev` returns either a valid pointer or NULL within bounded steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Coherent::alloc_from_dev` post: `*dma_handle = device_base + pageno * PAGE_SIZE`; returned vaddr = `virt_base + pageno * PAGE_SIZE`; first `size` bytes zeroed | `Coherent::alloc_from_dev` |
| `Coherent::release_from_dev` post: bitmap bits `[pageno, pageno + 2^order)` cleared | `Coherent::release_from_dev` |
| `Coherent::mmap_from_dev` post: ret == 1 ⟹ remap_pfn_range called with pfn ∈ `[pfn_base, pfn_base + size)` | `Coherent::mmap_from_dev` |
| `Coherent::assign_memory` post: ret == 0 ⟹ `dev.dma_mem == mem`; ret == -EBUSY ⟹ `dev.dma_mem` unchanged | `Coherent::assign_memory` |
| `Coherent::declare_memory` post: ret == 0 ⟹ pool attached; ret != 0 ⟹ pool released | `Coherent::declare_memory` |
| `Coherent::init_memory` post: returns valid pool with `bitmap` zeroed of `pages` bits, or rolls back all partial allocations on error | `Coherent::init_memory` |
| Pool invariant: `bitmap[i] == 1` ⟺ page `i` is allocated; pool capacity = `mem.size` pages | `DmaCoherentMem` |

### Layer 4: Verus/Creusot functional

`Per-pool alloc/free round-trip + per-pool mmap`:
- `alloc_from_dev(dev, size, &mut handle) = Some(va)` followed by `release_from_dev(dev, order, va) = 1`:
  - after release, the same page run can be returned by a subsequent `alloc_from_dev`.
  - bitmap state after release equals bitmap state before the original alloc.
- `mmap_from_dev(dev, vma, va, size, &mut ret) = 1` with `ret = 0`:
  - userspace VMA covers `[vma.vm_start, vma.vm_start + user_count << PAGE_SHIFT)`.
  - userspace reads of `va[0..size]` observe the kernel-side bytes (write-combine attributes).
- Two independent allocations on disjoint page runs preserve isolation: the bytes written through allocation A's vaddr do not appear in allocation B's vaddr.
- Per-DMA-API contract: `dma_get_device_base(dev, mem) + offset` is a valid device-programmable DMA address.

Encoded as Verus invariants on the `(bitmap, virt_base, device_base)` triple per-`Documentation/core-api/dma-api.rst`.

## Hardening

(Inherits row-1 features from `kernel/dma/00-overview.md` § Hardening.)

coherent-pool-specific reinforcement:

- **Per-pool `memremap(MEMREMAP_WC)`** — write-combine mapping; defense against stale CPU-cache aliasing between CPU access and device DMA (no snooping required).
- **Per-pool spinlock irq-saved** — `spin_lock_irqsave(&mem->spinlock)` around bitmap mutation; defense against alloc-from-IRQ vs alloc-from-process-context races (`dma_alloc_coherent` is callable from any context with the right gfp).
- **One pool per device enforced** — `dma_assign_coherent_memory` returns `-EBUSY` if `dev->dma_mem != NULL`; defense against pool-double-attach overwriting state.
- **Pool release zeroes `dev->dma_mem`** — `dma_release_coherent_memory` sets `dev->dma_mem = NULL` *before* freeing pool memory; defense against use-after-free on outstanding allocations (caller-driver responsibility).
- **Pool-base range warn vs coherent_dma_mask** — DT shared-pool with `device_base + size - 1 > min_not_zero(dev->coherent_dma_mask, dev->bus_dma_limit)` warns at probe; defense against silent over-mask DMA programming.
- **Pool-size cap on alloc** — `size > mem->size << PAGE_SHIFT` rejected before bitmap touch; defense against bitmap-out-of-range.
- **bitmap-based contiguous-run allocator** — `bitmap_find_free_region(order)` requires power-of-two run; defense against fragmented partial allocations that DMA controllers can't address.
- **Allocator zeroes returned buffer** — `memset(ret, 0, size)` post-alloc; defense against information leak from previously-freed pool occupant.
- **mmap pfn-range bounded** — `__dma_mmap_from_coherent` enforces `off < count ∧ user_count ≤ count - off` before `remap_pfn_range`; defense against userspace mapping beyond pool extent.
- **DT `shared-dma-pool` rejects `reusable`** — `rmem_dma_setup` returns -ENODEV on `reusable` (that's the CMA path); defense against pool-vs-CMA confusion.
- **Per-CONFIG_ARM strict `no-map` requirement** — `rmem_dma_setup` rejects pools without `no-map` on ARM; defense against CPU linear-map aliasing with device-DMA window.
- **Per-CONFIG_DMA_GLOBAL_POOL `__ro_after_init`** — `dma_coherent_default_memory` is read-only after init; defense against post-init pointer overwrite.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — coherent DMA buffers are typically mmaped to userspace via `dma_mmap_coherent`; whitelist the per-device pool extent so a driver bug cannot use `remap_pfn_range` to expose adjacent pool metadata (bitmap, virt_base) across the user boundary.
- **PAX_KERNEXEC** — coherent pool ops (`dma_alloc_from_dev_coherent`, `dma_release_from_dev_coherent`, `dma_mmap_from_dev_coherent`) are dispatched indirectly via `dma_map_ops`; enforce W^X on the ops vector and refuse rwx for any per-device trampoline.
- **PAX_RANDKSTACK** — `dma_alloc_from_dev_coherent` runs from arbitrary driver context (ioctl, probe) at user-driven depth; randomized kstack offset disrupts ROP through the bitmap-search + virt-translate chain.
- **PAX_REFCOUNT** — `struct dma_coherent_mem->use_dev_dma_pfn_offset` and per-device pool `nr_pages` bitmap accounting use refcount_t with saturating overflow; defense against ioctl-storm underflow racing pool teardown against in-flight alloc.
- **PAX_MEMORY_SANITIZE** — `dma_release_from_dev_coherent` and `dma_init_coherent_memory` rollback paths must zero the released pool extent before bitmap-mark-free; defense against post-free residual DMA-mapped data exposure to next allocator user.
- **PAX_UDEREF** — `dma_mmap_from_dev_coherent` user vma fault path keeps SMAP/PAN engaged across `remap_pfn_range`; never enable AC for more than the page-table update.
- **PAX_RAP/kCFI** — `dma_map_ops.alloc` / `.free` / `.mmap` indirect calls must verify kCFI tag at every device-coherent dispatch site.
- **GRKERNSEC_HIDESYM** — `/proc/iomem` and DT-pool `seq_show` paths must elide the kernel `virt_base` / `device_base` pointers from non-CAP_SYSLOG readers; expose only the physical pool range.
- **GRKERNSEC_DMESG** — coherent pool rollback paths (`rmem_dma_setup` failure, bitmap-alloc failure) must not splat raw pool addresses into dmesg readable by non-CAP_SYSLOG.
- **Per-device pool PAX_REFCOUNT** — `dma_coherent_mem->bitmap` extent must be tracked by refcount_t so simultaneous `dma_release_from_dev_coherent` + driver unbind cannot double-free the pool.
- **`dma_release` CAP_SYS_RAWIO** — only CAP_SYS_RAWIO holders (in the device-owning userns) can trigger `dma_release_declared_memory`; defense against an unprivileged ioctl path inadvertently tearing down a pool another driver still relies on.
- **DT `shared-dma-pool` `reusable` rejection** — already enforced; harden by requiring DT pool node to be parsed only during early boot (`__init`) and refusing dynamic re-registration of the same `<base, size>` window.
- **CONFIG_ARM `no-map` strict** — already enforced; harden by extending to CONFIG_ARM64 with `mark_rodata_ro` to lock the linear-map alias before any device coherent alloc.
- **`__ro_after_init` on `dma_coherent_default_memory`** — already enforced; harden by additionally locking the per-device `dev->dma_mem` pointer under `device_lock` so an attacker with a writable device pointer cannot swap the pool mid-allocation.
- **Rationale** — coherent DMA pools alias physical RAM through both a CPU virtual mapping and a device DMA window; a single mis-mapping yields either a kernel-RAM read primitive for the device or a device-RAM read primitive for userspace. The grsec regime forces an attacker to defeat W^X on dma_map_ops + kCFI on alloc/free dispatch + CAP_SYS_RAWIO on release + `__ro_after_init` on default pool + DT-parse early-boot lockdown before reaching either primitive.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- DMA API top-level dispatch (covered in `kernel/dma/mapping.md` Tier-3).
- DMA-direct path for devices without dedicated coherent pool (covered in `kernel/dma/dma-direct.md` Tier-3).
- swiotlb bounce buffer (covered in `kernel/dma/swiotlb.md` Tier-3).
- CMA contiguous-memory allocator (covered separately if expanded; `shared-dma-pool` with `reusable` falls through to CMA).
- IOMMU-backed DMA (covered in `kernel/dma/dma-iommu.md` Tier-3).
- Per-arch `phys_to_dma` / `dma_to_phys` translation (covered in arch-specific Tier-3).
- Per-arch `arch_sync_dma_*` cache management (covered in arch-specific Tier-3).
- Implementation code.
