# Tier-3: mm/dmapool.c — DMA-coherent slab-style pool

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: mm/00-overview.md
upstream-paths:
  - mm/dmapool.c (~527 lines)
  - include/linux/dmapool.h
  - include/linux/poison.h (POOL_POISON_FREED / POOL_POISON_ALLOCATED)
-->

## Summary

The **DMA pool allocator** carves small, DMA-coherent, device-mappable blocks out of larger `dma_alloc_coherent` regions. It is a slab-style allocator specialized for DMA: callers (network drivers, USB host controllers, SCSI HBAs, NVMe, AHCI, audio, GPU command-buffer fragments) need many small (a few bytes to a few hundred bytes) coherent blocks at known alignments and not-crossing power-of-two boundaries; raw `dma_alloc_coherent` per-block would waste at least one PAGE_SIZE per allocation and would be slow. Per-pool data: `struct dma_pool` holds the per-device list of allocated pages (`page_list`), a single LIFO `next_block` free-list across all pages, the immutable parameters (`size`, `align`, `boundary`, `allocation`, `node`), counters (`nr_blocks`, `nr_active`, `nr_pages`), and registration linkage (`pools` on `dev->dma_pools`). Per-page data: `struct dma_page` is a cacheable header for one `allocation`-byte coherent region: it holds the kernel virtual address (`vaddr`), the DMA-mapped address (`dma`), and the `page_list` linkage. Per-block, the first `sizeof(struct dma_block)` bytes serve double duty: when free, they hold `next_block` and the `dma` address; when allocated, the block is returned to the caller as opaque storage. Per-`dma_pool_create_node()` validates parameters (size > 0, align is power of two, size ≥ sizeof(dma_block), boundary is power of two and ≥ size and ≤ allocation), rounds size up to align, picks `allocation = max(size, PAGE_SIZE)`, attaches the pool to `dev->dma_pools`, and on first pool for a device creates the `pools` sysfs file (`/sys/devices/.../pools`). Per-`dma_pool_alloc()` is the kfree-style fast path: take pool->lock, pop `next_block`; if NULL, drop the lock, `dma_alloc_coherent` a new page, re-take the lock, slice the page into blocks honoring `boundary`, and link them into `next_block`. Per-`dma_pool_free()` returns one block via push-onto-`next_block`. Per-managed wrapper: `dmam_pool_create()` / `dmam_pool_destroy()` use devres so the pool is auto-destroyed on driver detach. Critical for: usb/host-controller transfer descriptors (UHCI/EHCI/XHCI TDs/QHs), scsi mid-layer scatterlists, nvme command-queue ring fragments, ahci command tables, sound-card buffer descriptors.

This Tier-3 covers `mm/dmapool.c` (~527 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct dma_pool` | per-pool state | `DmaPool` |
| `struct dma_page` | per-coherent-region header | `DmaPoolPage` |
| `struct dma_block` | per-free-list-link (in-place in block) | `DmaPoolBlock` |
| `dma_pool_create_node()` | per-pool create | `DmaPool::create_node` |
| `dma_pool_create()` | per-default-node wrapper | `DmaPool::create` |
| `dma_pool_destroy()` | per-pool destroy | `DmaPool::destroy` |
| `dma_pool_alloc()` | per-block alloc | `DmaPool::alloc` |
| `dma_pool_free()` | per-block free | `DmaPool::free` |
| `dma_pool_zalloc()` (inline header) | per-block alloc + zero | `DmaPool::zalloc` |
| `pool_alloc_page()` | per-coherent-page-alloc | `DmaPool::alloc_page` |
| `pool_initialise_page()` | per-page slice into blocks | `DmaPool::initialise_page` |
| `pool_init_page()` | per-page poison (DEBUG) | `DmaPool::init_page` |
| `pool_block_pop()` | per-LIFO pop | `DmaPool::block_pop` |
| `pool_block_push()` | per-LIFO push | `DmaPool::block_push` |
| `pool_find_page()` | per-vaddr→page linear scan | `DmaPool::find_page` |
| `pool_block_err()` | per-double-free / bad-dma check | `DmaPool::block_err` |
| `pool_check_block()` | per-poison-corruption check | `DmaPool::check_block` |
| `pools_show()` | per-/sys/devices sysfs | `DmaPool::sysfs_show` |
| `dev_attr_pools` | per-device sysfs attribute | `DmaPool::DEV_ATTR_POOLS` |
| `dmam_pool_create()` | per-managed-create (devres) | `DmaPool::managed_create` |
| `dmam_pool_destroy()` | per-managed-destroy (devres) | `DmaPool::managed_destroy` |
| `dmam_pool_release()` | per-devres release callback | `DmaPool::managed_release` |
| `dmam_pool_match()` | per-devres match | `DmaPool::managed_match` |
| `pools_lock` / `pools_reg_lock` | per-globals | shared |
| `POOL_POISON_FREED` / `POOL_POISON_ALLOCATED` | per-debug poison | shared |

## Compatibility contract

REQ-1: struct dma_pool:
- page_list: per-`struct list_head` of `dma_page` headers (doubly-linked, head in pool).
- lock: per-spinlock; held in alloc/free fast paths.
- next_block: per-`struct dma_block *` LIFO head of free blocks across all pages.
- nr_blocks: per-total blocks (free + allocated) carved across all pages.
- nr_active: per-currently-allocated block count.
- nr_pages: per-coherent-page count = `len(page_list)`.
- dev: per-`struct device *` owner (drives dma_alloc_coherent + sysfs registration).
- size: per-block size in bytes (post align-up).
- allocation: per-coherent-region size = max(size, PAGE_SIZE).
- boundary: per-power-of-two byte boundary blocks must not cross (≤ allocation; defaults to allocation).
- node: per-NUMA node for `struct dma_pool` + `struct dma_page` slab allocations.
- name[32]: per-diagnostic name (truncated via strscpy).
- pools: per-`struct list_head` linkage into `dev->dma_pools`.

REQ-2: struct dma_page:
- page_list: per-linkage into pool.page_list.
- vaddr: per-kernel virtual address of the coherent region (`dma_alloc_coherent`).
- dma: per-`dma_addr_t` of the coherent region.

REQ-3: struct dma_block (in-place in free blocks):
- next_block: per-`struct dma_block *` LIFO link.
- dma: per-`dma_addr_t` of this block (precomputed at page-init to avoid re-derivation in free).
- /* layout requires sizeof(dma_block) ≤ size; create rejects size < sizeof(dma_block) by clamping */

REQ-4: dma_pool_create_node(name, dev, size, align, boundary, node):
- /* Caller context: !in_interrupt() (may sleep) */
- if !dev: return NULL.
- /* align: power-of-two; 0 → 1 */
- if align == 0: align = 1.
- elif align & (align - 1): return NULL.
- /* size: > 0 ∧ ≤ INT_MAX */
- if size == 0 ∨ size > INT_MAX: return NULL.
- if size < sizeof(struct dma_block): size = sizeof(struct dma_block).
- /* Round up size to align */
- size = ALIGN(size, align).
- /* allocation: max(size, PAGE_SIZE) — single coherent region per page */
- allocation = max_t(size_t, size, PAGE_SIZE).
- /* boundary: power-of-two ∧ ≥ size ∧ ≤ allocation; 0 → allocation */
- if !boundary: boundary = allocation.
- elif (boundary < size) ∨ (boundary & (boundary - 1)): return NULL.
- boundary = min(boundary, allocation).
- /* Allocate pool struct */
- retval = kzalloc_node(sizeof(*retval), GFP_KERNEL, node).
- if !retval: return NULL.
- strscpy(retval.name, name, sizeof(retval.name)).
- retval.dev = dev.
- INIT_LIST_HEAD(&retval.page_list).
- spin_lock_init(&retval.lock).
- retval.size = size; retval.boundary = boundary; retval.allocation = allocation; retval.node = node.
- INIT_LIST_HEAD(&retval.pools).
- /* Register on dev->dma_pools under pools_reg_lock + pools_lock */
- mutex_lock(&pools_reg_lock).
- mutex_lock(&pools_lock).
- empty = list_empty(&dev->dma_pools).
- list_add(&retval.pools, &dev->dma_pools).
- mutex_unlock(&pools_lock).
- /* On first pool for this device, create sysfs `pools` attr */
- if empty:
  - err = device_create_file(dev, &dev_attr_pools).
  - if err: list_del under pools_lock, kfree(retval), return NULL.
- mutex_unlock(&pools_reg_lock).
- return retval.

REQ-5: dma_pool_destroy(pool):
- /* Caller context: !in_interrupt(); promises no in-flight users */
- if !pool: return.
- /* Deregister from dev->dma_pools */
- mutex_lock(&pools_reg_lock); mutex_lock(&pools_lock).
- list_del(&pool->pools).
- empty = list_empty(&pool.dev->dma_pools).
- mutex_unlock(&pools_lock).
- if empty: device_remove_file(pool.dev, &dev_attr_pools).
- mutex_unlock(&pools_reg_lock).
- /* Leak-check + WARN */
- busy = (pool.nr_active != 0).
- if busy: dev_err("%s busy"); /* Do NOT free coherent pages — they may still be DMA'd to */
- /* Free all dma_page headers; free coherent pages only if !busy */
- list_for_each_entry_safe(page, tmp, &pool.page_list, page_list):
  - if !busy: dma_free_coherent(pool.dev, pool.allocation, page.vaddr, page.dma).
  - list_del(&page.page_list).
  - kfree(page).
- kfree(pool).

REQ-6: dma_pool_alloc(pool, mem_flags, *handle):
- might_alloc(mem_flags).  /* sleep-able context check */
- spin_lock_irqsave(&pool.lock, flags).
- block = pool_block_pop(pool).
- if !block:
  - /* Drop lock, allocate page (may sleep / GFP_*), re-take lock */
  - spin_unlock_irqrestore(&pool.lock, flags).
  - page = pool_alloc_page(pool, mem_flags & ~__GFP_ZERO).
  - if !page: return NULL.
  - spin_lock_irqsave(&pool.lock, flags).
  - pool_initialise_page(pool, page).
  - block = pool_block_pop(pool).
- spin_unlock_irqrestore(&pool.lock, flags).
- /* Return DMA address via handle */
- *handle = block.dma.
- pool_check_block(pool, block, mem_flags).  /* DEBUG only: poison check + re-poison ALLOCATED */
- if want_init_on_alloc(mem_flags): memset(block, 0, pool.size).
- return block.

REQ-7: dma_pool_free(pool, vaddr, dma):
- block = vaddr.
- spin_lock_irqsave(&pool.lock, flags).
- if !pool_block_err(pool, vaddr, dma):
  - pool_block_push(pool, block, dma).
  - pool.nr_active--.
- spin_unlock_irqrestore(&pool.lock, flags).
- /* pool_block_err returns true on bad-dma or double-free; bad blocks are dropped (leaked) intentionally */

REQ-8: pool_alloc_page(pool, mem_flags):
- page = kmalloc_node(sizeof(*page), mem_flags, pool.node).
- if !page: return NULL.
- page.vaddr = dma_alloc_coherent(pool.dev, pool.allocation, &page.dma, mem_flags).
- if !page.vaddr: kfree(page); return NULL.
- return page.

REQ-9: pool_initialise_page(pool, page):
- /* Slice page into blocks honoring boundary; chain into first..last → push to pool.next_block */
- next_boundary = pool.boundary; offset = 0.
- first = NULL; last = NULL.
- pool_init_page(pool, page).  /* DEBUG: memset POOL_POISON_FREED across allocation; no-op otherwise */
- while offset + pool.size ≤ pool.allocation:
  - /* Skip past boundary to keep block within one boundary-aligned region */
  - if offset + pool.size > next_boundary:
    - offset = next_boundary.
    - next_boundary += pool.boundary.
    - continue.
  - block = page.vaddr + offset.
  - block.dma = page.dma + offset.
  - block.next_block = NULL.
  - if last: last.next_block = block.
  - else: first = block.
  - last = block.
  - offset += pool.size.
  - pool.nr_blocks++.
- /* Splice carved list onto pool.next_block (LIFO) */
- last.next_block = pool.next_block.
- pool.next_block = first.
- list_add(&page.page_list, &pool.page_list).
- pool.nr_pages++.

REQ-10: pool_block_pop(pool):
- block = pool.next_block.
- if block:
  - pool.next_block = block.next_block.
  - pool.nr_active++.
- return block.

REQ-11: pool_block_push(pool, block, dma):
- block.dma = dma.
- block.next_block = pool.next_block.
- pool.next_block = block.

REQ-12: pool_find_page(pool, dma):
- /* Linear scan of pool.page_list; coherent regions don't overlap */
- list_for_each_entry(page, &pool.page_list, page_list):
  - if dma < page.dma: continue.
  - if (dma - page.dma) < pool.allocation: return page.
- return NULL.

REQ-13: pool_block_err(pool, vaddr, dma) (DEBUG):
- /* 1. dma must fall inside some page */
- page = pool_find_page(pool, dma).
- if !page: dev_err("bad dma"); return true.
- /* 2. block must not already be on the free list (double-free check) */
- for block in pool.next_block list:
  - if block == vaddr: dev_err("already free"); return true.
- /* 3. Poison block for next allocation */
- memset(vaddr, POOL_POISON_FREED, pool.size).
- return false.

REQ-13b: pool_block_err(pool, vaddr, dma) (non-DEBUG):
- if want_init_on_free(): memset(vaddr, 0, pool.size).
- return false.  /* no validation in production build */

REQ-14: pool_check_block(pool, block, mem_flags) (DEBUG):
- /* Verify block content is POOL_POISON_FREED before re-use */
- data = block.
- for i in sizeof(dma_block)..pool.size:
  - if data[i] == POOL_POISON_FREED: continue.
  - dev_err("corrupted"); print_hex_dump; break.
- /* Re-poison ALLOCATED unless caller asks for zero */
- if !want_init_on_alloc(mem_flags): memset(block, POOL_POISON_ALLOCATED, pool.size).

REQ-14b: pool_check_block(pool, block, mem_flags) (non-DEBUG): no-op.

REQ-15: pools_show(dev, attr, buf):
- size = sysfs_emit(buf, "poolinfo - 0.1\n").
- mutex_lock(&pools_lock).
- list_for_each_entry(pool, &dev->dma_pools, pools):
  - size += sysfs_emit_at(buf, size, "%-16s %4zu %4zu %4u %2zu\n",
                          pool.name, pool.nr_active, pool.nr_blocks, pool.size, pool.nr_pages).
- mutex_unlock(&pools_lock).
- return size.

REQ-16: dmam_pool_create(name, dev, size, align, allocation):
- /* Managed: devres auto-destroys on driver detach */
- ptr = devres_alloc(dmam_pool_release, sizeof(*ptr), GFP_KERNEL).
- if !ptr: return NULL.
- pool = *ptr = dma_pool_create(name, dev, size, align, allocation).
- if pool: devres_add(dev, ptr).
- else: devres_free(ptr).
- return pool.

REQ-17: dmam_pool_destroy(pool):
- dev = pool.dev.
- WARN_ON(devres_release(dev, dmam_pool_release, dmam_pool_match, pool)).
- /* devres_release invokes dmam_pool_release → dma_pool_destroy(pool); WARN signals "not registered" */

REQ-18: dmam_pool_release(dev, res): dma_pool_destroy(*(struct dma_pool **)res).

REQ-19: dmam_pool_match(dev, res, match_data): return *(struct dma_pool **)res == match_data.

REQ-20: Locking and context:
- pools_reg_lock (mutex): serializes registration + sysfs file create/remove.
- pools_lock (mutex): protects `dev->dma_pools` list and sysfs read.
- pool.lock (spin_lock_irqsave): protects pool.next_block / nr_active / nr_pages / nr_blocks during alloc/free.
- dma_pool_create_node / _destroy must run in process context (mutex use; may sleep).
- dma_pool_alloc may sleep iff mem_flags allows (GFP_KERNEL) and slow path is taken; with GFP_ATOMIC it returns NULL on slow-path failure.
- dma_pool_free may be called from interrupt context (spin_lock_irqsave).

REQ-21: Boundary semantics:
- Blocks never straddle a `boundary`-aligned region of `allocation`.
- Achieved by `pool_initialise_page` skipping `offset` forward to next_boundary when the next block would cross.
- Used by hardware with addressing restrictions (e.g. PCI 4 KiB-boundary).
- If `boundary == allocation` (default), no skipping happens.

REQ-22: Sizing:
- `allocation = max(size, PAGE_SIZE)`: one `dma_alloc_coherent` per page, never smaller than PAGE_SIZE.
- Blocks per page (without boundary): `floor(allocation / size)`.
- With boundary: each boundary-aligned region holds `floor(boundary / size)` blocks; full page holds `(allocation / boundary) * floor(boundary / size)` blocks.

REQ-23: Diagnostic / non-functional state:
- DMAPOOL_DEBUG enabled iff CONFIG_SLUB_DEBUG_ON; otherwise debug variants are no-ops.
- POOL_POISON_FREED applied at page-init and on free; POOL_POISON_ALLOCATED applied at alloc (DEBUG).
- want_init_on_alloc(gfp) / want_init_on_free(): respect init_on_alloc / init_on_free kernel options regardless of DEBUG.

## Acceptance Criteria

- [ ] AC-1: dma_pool_create(name, dev, size=64, align=8, allocation=0) returns non-NULL; size = 64 (already aligned); allocation = PAGE_SIZE; boundary = PAGE_SIZE.
- [ ] AC-2: dma_pool_create with align=3 (not power-of-two): returns NULL.
- [ ] AC-3: dma_pool_create with size=0: returns NULL.
- [ ] AC-4: dma_pool_create with boundary < size: returns NULL.
- [ ] AC-5: dma_pool_create with size < sizeof(dma_block): size is clamped up to sizeof(dma_block); pool succeeds.
- [ ] AC-6: First dma_pool_create for a device creates /sys/devices/.../pools sysfs file; second pool on same device does not re-create.
- [ ] AC-7: dma_pool_alloc(pool, GFP_KERNEL, &handle): returns vaddr; *handle is DMA address; nr_active+=1; on first call nr_pages becomes 1.
- [ ] AC-8: dma_pool_alloc on empty pool with GFP_ATOMIC: may return NULL if dma_alloc_coherent fails.
- [ ] AC-9: dma_pool_alloc with __GFP_ZERO: returned block is zeroed; coherent page itself is not (alloc_page strips __GFP_ZERO).
- [ ] AC-10: dma_pool_free(pool, vaddr, dma): nr_active-=1; block pushed onto next_block LIFO; subsequent alloc returns the same vaddr.
- [ ] AC-11: dma_pool_free with bogus DMA address (DEBUG): pool_block_err logs "bad dma" and leaks the block (not pushed).
- [ ] AC-12: dma_pool_free called twice on same block (DEBUG): pool_block_err logs "already free" and leaves nr_active unchanged on second call.
- [ ] AC-13: dma_pool_destroy on pool with nr_active > 0: logs busy; frees pool struct + page headers but NOT coherent regions (DMA may still be live).
- [ ] AC-14: dma_pool_destroy as last pool on device: removes sysfs file.
- [ ] AC-15: dmam_pool_create + driver detach: devres release auto-destroys the pool.
- [ ] AC-16: Boundary semantics: pool with allocation=4096, size=300, boundary=1024: blocks do not cross 1024-byte regions; ~3 blocks per 1024-byte region (300*3=900 ≤ 1024).
- [ ] AC-17: pools_show via sysfs: emits one row per pool with name, nr_active, nr_blocks, size, nr_pages.
- [ ] AC-18: Concurrent dma_pool_alloc / dma_pool_free from interrupt + process context: pool.lock taken irq-safe; no list corruption.

## Architecture

```
struct DmaPool {
  page_list: ListHead,                  // dma_page headers
  lock: SpinLockIrq,
  next_block: AtomicPtr<DmaPoolBlock>,  // LIFO free-list head
  nr_blocks: usize,
  nr_active: usize,
  nr_pages: usize,
  dev: *Device,
  size: u32,                            // post-align block size
  allocation: u32,                      // coherent region size = max(size, PAGE_SIZE)
  boundary: u32,                        // power-of-two; blocks won't cross
  node: i32,                            // NUMA node
  name: [u8; 32],                       // strscpy'd
  pools: ListNode,                      // on dev->dma_pools
}

struct DmaPoolPage {
  page_list: ListNode,                  // on DmaPool.page_list
  vaddr: *u8,                           // kernel virtual address
  dma: DmaAddr,                         // device DMA address
}

#[repr(C)]
struct DmaPoolBlock {                   // overlays the start of every free block
  next_block: *mut DmaPoolBlock,        // LIFO link
  dma: DmaAddr,                         // precomputed at page-init
}
```

`DmaPool::create_node(name, dev, size, align, boundary, node) -> Option<KBox<DmaPool>>`:
1. /* Caller context: process; may sleep */
2. if dev.is_null(): return None.
3. let align = if align == 0 { 1 } else { align }.
4. if !align.is_power_of_two(): return None.
5. if size == 0 || size > i32::MAX as usize: return None.
6. let size = size.max(size_of::<DmaPoolBlock>()).
7. let size = align_up(size, align) as u32.
8. let allocation = max(size, PAGE_SIZE as u32).
9. let boundary = if boundary == 0 { allocation } else {
     if boundary < size || !boundary.is_power_of_two(): return None;
     boundary.min(allocation)
   }.
10. let mut pool = KBox::<DmaPool>::zalloc_node(GFP_KERNEL, node)?.
11. strscpy(&mut pool.name, name).
12. pool.dev = dev; pool.size = size; pool.allocation = allocation; pool.boundary = boundary; pool.node = node.
13. ListHead::init(&mut pool.page_list).
14. SpinLockIrq::init(&mut pool.lock).
15. ListNode::init(&mut pool.pools).
16. /* Register on dev */
17. let _reg_guard = pools_reg_lock.lock().
18. let _list_guard = pools_lock.lock().
19. let empty = (*dev).dma_pools.is_empty().
20. list_add(&pool.pools, &(*dev).dma_pools).
21. drop(_list_guard).
22. if empty:
    - if device_create_file(dev, &DEV_ATTR_POOLS).is_err():
      - let _g = pools_lock.lock(); list_del(&pool.pools); drop(_g).
      - drop(_reg_guard).
      - return None.
23. drop(_reg_guard).
24. Some(pool).

`DmaPool::destroy(pool: KBox<DmaPool>)`:
1. /* Caller context: process; promises no in-flight users */
2. let _reg_guard = pools_reg_lock.lock().
3. let _list_guard = pools_lock.lock().
4. list_del(&pool.pools).
5. let empty = (*pool.dev).dma_pools.is_empty().
6. drop(_list_guard).
7. if empty: device_remove_file(pool.dev, &DEV_ATTR_POOLS).
8. drop(_reg_guard).
9. let busy = pool.nr_active != 0.
10. if busy: dev_err!(pool.dev, "{} {} busy", "dma_pool_destroy", pool.name).
11. for (page, tmp) in list_for_each_entry_safe(&pool.page_list):
    - if !busy: dma_free_coherent(pool.dev, pool.allocation, page.vaddr, page.dma).
    - list_del(&page.page_list).
    - drop(KBox::from_raw(page)).
12. drop(pool).  /* KBox dropped */

`DmaPool::alloc(pool: &DmaPool, mem_flags: Gfp, handle: &mut DmaAddr) -> Option<NonNull<u8>>`:
1. might_alloc(mem_flags).
2. let mut guard = pool.lock.lock_irqsave().
3. let block = DmaPool::block_pop(pool).
4. let block = if block.is_some() { block.unwrap() } else {
5.   drop(guard).
6.   let page = DmaPool::alloc_page(pool, mem_flags & !__GFP_ZERO)?.
7.   guard = pool.lock.lock_irqsave().
8.   DmaPool::initialise_page(pool, page).
9.   DmaPool::block_pop(pool).expect("just sliced page → at least one block")
10. }.
11. drop(guard).
12. *handle = block.dma.
13. DmaPool::check_block(pool, block, mem_flags).  /* DEBUG only */
14. if want_init_on_alloc(mem_flags): write_bytes(block as *mut u8, 0, pool.size).
15. Some(NonNull::new_unchecked(block as *mut u8)).

`DmaPool::free(pool: &DmaPool, vaddr: NonNull<u8>, dma: DmaAddr)`:
1. let block = vaddr.cast::<DmaPoolBlock>().
2. let _guard = pool.lock.lock_irqsave().
3. if !DmaPool::block_err(pool, vaddr.as_ptr(), dma):
   - DmaPool::block_push(pool, block, dma).
   - pool.nr_active -= 1.

`DmaPool::alloc_page(pool: &DmaPool, mem_flags: Gfp) -> Option<&mut DmaPoolPage>`:
1. let page = KBox::<DmaPoolPage>::alloc_node(mem_flags, pool.node)?.
2. page.vaddr = dma_alloc_coherent(pool.dev, pool.allocation, &mut page.dma, mem_flags).
3. if page.vaddr.is_null(): drop(page); return None.
4. Some(KBox::leak(page)).  /* ownership lives on pool.page_list */

`DmaPool::initialise_page(pool: &DmaPool, page: &mut DmaPoolPage)`:
1. let mut next_boundary = pool.boundary; let mut offset = 0u32.
2. let (mut first, mut last) = (ptr::null_mut(), ptr::null_mut()).
3. DmaPool::init_page(pool, page).  /* DEBUG memset */
4. while offset + pool.size ≤ pool.allocation:
   - if offset + pool.size > next_boundary:
     - offset = next_boundary.
     - next_boundary += pool.boundary.
     - continue.
   - let block = (page.vaddr.add(offset as usize)) as *mut DmaPoolBlock.
   - (*block).dma = page.dma + offset as u64.
   - (*block).next_block = ptr::null_mut().
   - if !last.is_null(): (*last).next_block = block.
   - else: first = block.
   - last = block.
   - offset += pool.size.
   - pool.nr_blocks += 1.
5. (*last).next_block = pool.next_block.
6. pool.next_block = first.
7. list_add(&page.page_list, &pool.page_list).
8. pool.nr_pages += 1.

`DmaPool::block_pop(pool: &DmaPool) -> Option<&mut DmaPoolBlock>`:
1. let block = pool.next_block.
2. if !block.is_null():
   - pool.next_block = (*block).next_block.
   - pool.nr_active += 1.
3. if block.is_null() { None } else { Some(&mut *block) }.

`DmaPool::block_push(pool: &DmaPool, block: &mut DmaPoolBlock, dma: DmaAddr)`:
1. block.dma = dma.
2. block.next_block = pool.next_block.
3. pool.next_block = block.

`DmaPool::find_page(pool: &DmaPool, dma: DmaAddr) -> Option<&DmaPoolPage>`:
1. for page in list_for_each_entry(&pool.page_list):
   - if dma < page.dma: continue.
   - if (dma - page.dma) < pool.allocation as u64: return Some(page).
2. None.

`DmaPool::block_err(pool: &DmaPool, vaddr: *mut u8, dma: DmaAddr) -> bool` (DEBUG):
1. let page = DmaPool::find_page(pool, dma)?  /* return true on None */.
2. for block in walk(pool.next_block):
   - if block as *mut u8 == vaddr: dev_err!(pool.dev, "dma {} already free", dma); return true.
3. write_bytes(vaddr, POOL_POISON_FREED, pool.size).
4. false.

`DmaPool::managed_create(name, dev, size, align, allocation) -> Option<*DmaPool>`:
1. let ptr = devres_alloc(DmaPool::managed_release, size_of::<*DmaPool>(), GFP_KERNEL)?.
2. let pool = DmaPool::create(name, dev, size, align, allocation)?.
3. unsafe { *(ptr as *mut *mut DmaPool) = pool; }
4. devres_add(dev, ptr).
5. Some(pool).

`DmaPool::managed_destroy(pool: &DmaPool)`:
1. let dev = pool.dev.
2. WARN_ON(devres_release(dev, DmaPool::managed_release, DmaPool::managed_match, pool).is_err()).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `size_clamp_to_dma_block` | INVARIANT | per-create_node: size after adjustment ≥ size_of::<DmaPoolBlock>(). |
| `align_power_of_two` | INVARIANT | per-create_node: returns Some ⟹ align.is_power_of_two(). |
| `boundary_invariants` | INVARIANT | per-create_node: returns Some ⟹ boundary ≤ allocation ∧ boundary ≥ size ∧ boundary.is_power_of_two(). |
| `allocation_min_page` | INVARIANT | per-create_node: pool.allocation ≥ PAGE_SIZE. |
| `freelist_LIFO_order` | INVARIANT | per-block_pop / block_push: stack semantics; head = last-pushed. |
| `nr_active_balanced` | INVARIANT | per-alloc/free pair: nr_active +1 on alloc, -1 on free; never < 0. |
| `block_inside_owned_page` | INVARIANT | per-pool_initialise_page: every block.dma falls inside [page.dma, page.dma + allocation). |
| `block_no_cross_boundary` | INVARIANT | per-pool_initialise_page: floor(block.dma / boundary) == floor((block.dma + size - 1) / boundary). |
| `find_page_disjoint` | INVARIANT | per-find_page: coherent regions are disjoint; at most one page matches. |
| `lock_held_in_block_pop_push` | INVARIANT | per-block_pop / block_push: pool.lock held by caller. |
| `pool_destroy_after_no_active` | SAFETY | per-destroy: nr_active > 0 ⟹ coherent regions NOT freed (DMA-live safety). |
| `sysfs_file_lifecycle` | INVARIANT | per-create: file created iff empty(dev.dma_pools) at insert; per-destroy: removed iff empty(dev.dma_pools) at remove. |

### Layer 2: TLA+

`mm/dmapool.tla`:
- Per-create + per-alloc + per-free + per-destroy lifecycle.
- Per-block_pop / push interleavings.
- Properties:
  - `safety_alloc_returns_owned_block` — per-alloc: returned block belongs to some page in pool.page_list.
  - `safety_no_double_free` — per-pool: a block in pool.next_block is never popped twice without a push in between.
  - `safety_no_use_after_free` — per-pool: a block returned to caller is not on pool.next_block.
  - `safety_destroy_after_quiesce` — per-destroy: nr_active > 0 ⟹ logs busy ∧ retains coherent pages.
  - `safety_sysfs_consistent` — per-dev: pools sysfs file present iff dev.dma_pools is non-empty.
  - `liveness_alloc_either_succeeds_or_gfp_atomic_fails` — per-alloc: with sleeping GFP, eventually returns block; with GFP_ATOMIC, may return None.
  - `liveness_destroy_terminates` — per-destroy: returns in finite steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `DmaPool::create_node` post: Some ⟹ pool linked in dev.dma_pools | `DmaPool::create_node` |
| `DmaPool::create_node` post: None ⟹ no partial state (no sysfs file, no list entry, no leak) | `DmaPool::create_node` |
| `DmaPool::destroy` post: pool unlinked; coherent pages freed iff !busy | `DmaPool::destroy` |
| `DmaPool::alloc` post: Some(block) ⟹ *handle = block.dma; block ∉ pool.next_block | `DmaPool::alloc` |
| `DmaPool::alloc` post: None ⟹ pool.nr_active unchanged | `DmaPool::alloc` |
| `DmaPool::free` post: block on pool.next_block (LIFO head); nr_active-=1 | `DmaPool::free` |
| `DmaPool::initialise_page` post: every carved block.dma is page.dma + k*size for some k; no block crosses boundary | `DmaPool::initialise_page` |
| `DmaPool::find_page` post: Some(page) ⟹ page.dma ≤ dma < page.dma + pool.allocation | `DmaPool::find_page` |
| `DmaPool::block_err` post: true ⟹ block was already free or dma outside any page | `DmaPool::block_err` |
| `DmaPool::managed_create` post: pool added to dev's devres; auto-released on detach | `DmaPool::managed_create` |

### Layer 4: Verus/Creusot functional

`Per-pool lifecycle (create_node → alloc N → free N → destroy) refines a `Multiset<Block>` model with operations { allocate, deallocate }: blocks unique per `dma_addr_t`, alloc returns a free block (or grows by one page-worth of new blocks), free returns block to the multiset, destroy collapses pool. Per-Documentation/core-api/dma-api.rst contract: returned blocks have device-coherent mappings; no cache-flush primitive needed by driver.

## Hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening.)

DMA-pool reinforcement:

- **Per-align power-of-two reject** — defense against per-misaligned-DMA bus error.
- **Per-size ≤ INT_MAX reject** — defense against per-allocation-size overflow.
- **Per-boundary < size reject** — defense against per-impossible-fit infinite loop in initialise_page.
- **Per-strscpy(name)** — defense against per-name-overrun in dev_err / sysfs.
- **Per-pool.lock irq-save** — defense against per-IRQ-context dma_pool_free corrupting list.
- **Per-pool_block_err leak on bad-free** — defense against per-double-free re-publish.
- **Per-POOL_POISON_FREED/_ALLOCATED (DEBUG)** — defense against per-use-after-free / write-after-poison.
- **Per-busy pool keeps coherent regions on destroy** — defense against per-live-DMA-after-free.
- **Per-sysfs file under pools_reg_lock + pools_lock** — defense against per-concurrent-create vs sysfs race.
- **Per-dma_alloc_coherent failure path frees dma_page header** — defense against per-OOM-leak.
- **Per-managed-pool devres auto-destroy** — defense against per-driver-detach pool leak.
- **Per-mem_flags & ~__GFP_ZERO for page alloc** — defense against per-double-zero / per-coherent-zero-uselessness.
- **Per-zone-aware kmalloc_node + dma_alloc_coherent on pool.node** — defense against per-cross-NUMA cacheline pingpong.

## Grsecurity/PaX-style Reinforcement

Baseline grsec/PaX features inherited project-wide:

- **PAX_USERCOPY** — DMA-coherent buffers never copied across user boundary directly; whitelist enforced on adjacent metadata slabs.
- **PAX_KERNEXEC** — dma_pool descriptor list head and per-page metadata in W^X kernel data.
- **PAX_RANDKSTACK** — dma_pool_alloc / dma_pool_free entry randomized; predictable stack layout for DMA-completion paths prevented.
- **PAX_REFCOUNT** — pool refcount and per-page in_use counter saturate; destroy-of-busy-pool refuses rather than free-while-used.
- **PAX_MEMORY_SANITIZE** — POOL_POISON_FREED/_ALLOCATED extended to production builds (lightweight); released buffers zero-poisoned before reuse.
- **PAX_UDEREF** — pool name and size args from drivers boundary-checked before strscpy / list insertion.
- **PAX_RAP / kCFI** — dma_pool_create / dma_pool_destroy callbacks type-tagged; mismatched managed-pool devres callback refused.
- **GRKERNSEC_HIDESYM** — `dma_pool_alloc`, `dma_pool_free`, `pools_lock` absent from `/proc/kallsyms` for unpriv.
- **GRKERNSEC_DMESG** — pool_block_err / busy-on-destroy WARN traces redacted from dmesg for non-CAP_SYSLOG.

DMA-pool-specific reinforcements:

- **Per-page in_use saturating refcount** — block alloc/free arithmetic uses cmpxchg trap; under/overflow on free path blocks corruption.
- **Free-list integrity (next_block pointer)** — every free-list link verified within page bounds before dereference; OOB next-block traps under PAX_USERCOPY.
- **DMA-coherent region pinned across pool lifetime** — busy-pool destroy refused; coherent regions never freed while peripheral may still DMA.
- **Poison-marker check on alloc-from-freelist** — POOL_POISON_FREED pattern verified before handoff; write-after-free detected at allocation.
- **align/size/boundary validated pre-create** — non-power-of-two align, size > INT_MAX, boundary < size all rejected before any allocation.

Rationale: DMA pools hand bus-addressable coherent memory to peripherals; a refcount underflow lets a free'd buffer be reallocated while a NIC/disk continues DMA writes — a classic DMA-after-free escalation. Grsec emphasis: saturate refcounts, validate free-list links so a corrupted next_block cannot redirect a future alloc onto arbitrary kernel memory, and refuse busy-pool destruction outright.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- kernel/dma/* core DMA-mapping (covered in DMA layer Tier-3)
- kernel/dma/coherent.c `dma_alloc_coherent` itself (covered separately)
- kernel/dma/direct.c / iommu paths (covered in IOMMU Tier-3)
- drivers/base/devres.c devres infrastructure (covered in driver-base)
- Per-driver use sites (usb/host/, drivers/scsi/, drivers/nvme/, drivers/ata/) — out of mm scope
- Implementation code
