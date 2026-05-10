---
title: "Tier-3: net/core/page_pool.c — per-NIC per-queue page allocator (DMA-mapped recycling + per-NUMA-node cache)"
tags: ["tier-3", "net", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

`page_pool` is the high-perf per-NIC RX page allocator that NIC drivers use to source DMA-mapped pages for incoming packets — per-pool NAPI-bound LRU cache + per-CPU per-pool freelist + per-NUMA-node fallback. Per-page DMA mapping cached + reused (avoiding per-skb dma_map/_unmap costs). When skb-frag is recycled (via skb_release_data), the page returns to its pool's freelist instead of being freed; reused on next RX. Critical for: high-PPS NICs (mlx5, i40e, etc.) where slab-allocator + dma_map costs would saturate per-CPU cycles.

This Tier-3 covers `net/core/page_pool.c` (~1374 lines).

### Acceptance Criteria

- [ ] AC-1: page_pool_create with PP_FLAG_DMA_MAP: per-pool DMA-mapping pre-allocated.
- [ ] AC-2: page_pool_alloc_pages: returns DMA-mapped page; fast-path < 100ns.
- [ ] AC-3: page_pool_put_page after RX: page recycled to cache; subsequent alloc returns same.
- [ ] AC-4: NAPI-binding: cross-CPU put → ring; same-CPU put → cache.
- [ ] AC-5: PP_FLAG_PAGE_FRAG: 4KB page split into 4× 1KB frags; all 4 used before alloc new.
- [ ] AC-6: XDP redirect: pool registered; XDP_REDIRECT delivers via pool.
- [ ] AC-7: Per-NUMA-node fallback: per-NUMA-node cache miss → same-node alloc preferred.
- [ ] AC-8: Pool destroy: in-flight pages tracked; destroy waits for completion.
- [ ] AC-9: 100M alloc+recycle cycles: no leak; throughput ≥ 90% of mempool baseline.
- [ ] AC-10: pktgen at line-rate on 100Gbps NIC: per-CPU page_pool fast-path achieves wire-speed.

### Architecture

`PagePool`:

```
struct PagePool {
  params: PagePoolParams,
  cpu: AtomicI32,
  napi_id: u32,
  pool_size: u32,
  cache: PerCpu<KVec<KArc<Page>>>,
  ring: KArc<PtrRing>,
  alloc_cache: KArc<PtrRing>,                  // lockless SPMC
  dma_map: bool,
  dma_sync: bool,
  pages_state_release_cnt: AtomicU32,
  pages_state_hold_cnt: AtomicU32,
  cur_page: KAtomicPtr<Page>,
  frag_offset: u32,
  frag_users: AtomicI32,
  user: KArc<dyn PagePoolUser>,
  ...
}

struct PagePoolParams {
  flags: u32,
  order: u32,
  pool_size: u32,
  nid: i32,
  dev: KArc<Device>,
  napi: Option<KArc<Napi>>,
  dma_dir: DmaDirection,
  max_len: u32,
  offset: u32,
}
```

`PagePool::alloc_pages(pool, gfp)`:
1. Try alloc_cache pop (lockless).
2. If empty: try ring pop.
3. If empty: __alloc_pages_slow:
   - alloc_pages_node(pool.params.nid, gfp, pool.params.order).
   - If pool.dma_map: dma_addr := dma_map_page(pool.params.dev, page, 0, page_size, pool.params.dma_dir).
   - page->dma_addr = dma_addr.
   - page->pp = pool.
   - page->pp_magic = MAGIC.
4. Increment pages_state_hold_cnt.
5. Return page.

`PagePool::put_page(pool, page, dma_sync_size, allow_direct)`:
1. If page->pp != pool: free directly (orphan).
2. If pool.dma_sync: dma_sync_single_for_device(pool.params.dev, dma_addr, dma_sync_size, pool.params.dma_dir).
3. If allow_direct + on-NAPI-CPU:
   - alloc_cache push.
4. Else:
   - ring push.
5. If push fails (full): __page_pool_release_page → dma_unmap + free.

`PagePool::recycle_direct(pool, page)`:
1. cpu := smp_processor_id().
2. cache := per_cpu(pool.cache, cpu).
3. If cache.len < pool_size: cache.push(page); return.
4. Else: defer to ring.

`PagePool::destroy(pool)`:
1. Drain alloc_cache + ring + per-CPU caches.
2. Per-page: dma_unmap_page; __free_pages.
3. Wait pages_state_hold_cnt == pages_state_release_cnt.
4. Free pool struct.

### Out of Scope

- net/00-overview (covered)
- struct sock (covered in `net/struct-sock.md` Tier-2)
- skbuff (covered in `net/core/skbuff-alloc.md` Tier-3)
- net_device + NAPI (covered in `net/core/dev.md` Tier-3)
- XDP (covered in `kernel/bpf/bpf-core.md` Tier-3)
- DMA-API (covered in `kernel/dma/dma-iommu.md` Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct page_pool` | per-pool config + state | `net::core::page_pool::PagePool` |
| `struct page_pool_params` | per-pool ctor params | `PagePoolParams` |
| `page_pool_create(params)` | per-driver create | `PagePool::create` |
| `page_pool_destroy(pool)` | per-driver destroy | `PagePool::destroy` |
| `page_pool_alloc_pages(pool, gfp)` | per-page alloc | `PagePool::alloc_pages` |
| `page_pool_alloc_frag(pool, &offset, size, gfp)` | per-frag alloc (sub-page) | `PagePool::alloc_frag` |
| `page_pool_recycle_direct(pool, page)` | per-page direct-recycle (NAPI fast-path) | `PagePool::recycle_direct` |
| `page_pool_put_page(pool, page, dma_sync_size, allow_direct)` | per-page put with DMA-sync | `PagePool::put_page` |
| `page_pool_release_page(pool, page)` | per-page release without recycle | `PagePool::release_page` |
| `page_pool_dma_sync_for_cpu(pool, page, offset, dma_sync_size)` | per-page CPU-side DMA sync | `PagePool::dma_sync_for_cpu` |
| `page_pool_dma_sync_for_device(pool, page, offset, dma_sync_size)` | per-page device-side DMA sync | `PagePool::dma_sync_for_device` |
| `page_pool_use_xdp_mem(pool, ...)` | per-pool XDP redirect-target | `PagePool::use_xdp_mem` |
| `__page_pool_alloc_pages_slow(pool, gfp)` | per-pool slow-path alloc (refill) | `PagePool::alloc_pages_slow` |
| `page_pool_recycle_in_cache(pool, page)` | per-pool LRU cache push | `PagePool::recycle_in_cache` |
| `page_pool_recycle_in_ring(pool, page)` | per-pool ring-buffer push | `PagePool::recycle_in_ring` |
| `page_pool_get_dma_dir(pool)` | per-pool DMA direction | `PagePool::get_dma_dir` |
| `page_pool_get_dma_addr(page)` | per-page DMA addr | `Page::get_dma_addr` |
| `page_pool_set_dma_addr(page, addr)` | per-page DMA addr cache | `Page::set_dma_addr` |

### compatibility contract

REQ-1: Per-pool `page_pool`:
- `params` (PagePoolParams snapshot).
- `cpu` (NAPI-bound CPU; -1 if unbound).
- `napi_id` (NAPI ID; for safety check).
- `pool_size` (max ring entries; typically 1024).
- `slow_alloc_count` (slow-path counter).
- `recycle_count` (recycle counter).
- `frag_users` (per-frag refcount on current frag-page).
- `frag_offset` (current frag offset).
- `cur_page` (current page being fragmented).
- `frag_size`.
- `cache` (per-CPU LRU cache; CONFIG_NUMA-aware).
- `ring` (per-pool ringbuf for cross-CPU recycle).
- `pp_alloc_cache` (alloc cache; lockless single-producer-multi-consumer).
- `dma_map` (bool: use dma_map_page).
- `dma_sync` (bool: use dma_sync_*).

REQ-2: `PagePoolParams`:
- `flags` (PP_FLAG_DMA_MAP / _DMA_SYNC_DEV / _PAGE_FRAG / _SYSTEM_POOL / _ALLOW_UNREADABLE_NETMEM).
- `order` (page-order; 0 = single page).
- `pool_size`.
- `nid` (NUMA node).
- `dev` (per-NIC device for DMA-API).
- `napi` (KArc<Napi> — NAPI binding).
- `dma_dir` (DMA_FROM_DEVICE / _TO_DEVICE / _BIDIRECTIONAL).
- `max_len` (per-page max bytes; for sync_size).
- `offset` (per-page reserved headroom).

REQ-3: Per-pool alloc flow:
1. `page_pool_alloc_pages(pool, gfp)`:
   - Try alloc_cache (lockless): if entry: pop + return.
   - Try ring (per-pool cross-CPU): if entry: pop + return.
   - Slow path: alloc page from buddy via __page_pool_alloc_pages_slow.
   - DMA-map if PP_FLAG_DMA_MAP.
   - Cache DMA addr in page-private.

REQ-4: Per-pool recycle flow:
1. `page_pool_put_page(pool, page, dma_sync_size, allow_direct)`:
   - If !page_pool_recycle: directly free.
   - DMA-sync if PP_FLAG_DMA_SYNC_DEV.
   - If allow_direct + on-NAPI-CPU: page_pool_recycle_direct (push to alloc_cache).
   - Else: page_pool_recycle_in_ring (push to ring).
   - Else: free via __page_pool_release_page.

REQ-5: NAPI-binding safety:
- Per-pool napi_id snapshot at create.
- Per-recycle: if !in_napi_softirq(pool.napi_id): defer to ring (cross-CPU).
- Else: direct cache (avoid lock).
- Ensures lockless cache only accessed from binding-NAPI.

REQ-6: Per-pool ring buffer:
- Lock-free ptr_ring (multi-producer-single-consumer).
- Cross-CPU recycle pushes to ring; NAPI-CPU pops to cache.

REQ-7: Per-frag allocation (PP_FLAG_PAGE_FRAG):
- Pool maintains current page + offset.
- Each alloc returns sub-page region; offset advances.
- Per-page reused until exhausted; then alloc new page.

REQ-8: DMA semantics:
- Per-pool dma_map_page once at slow-path alloc.
- Per-recycle: dma_sync_for_device before reuse.
- Per-page-release: dma_unmap_page.
- Avoids per-skb dma_map/unmap costs.

REQ-9: XDP integration:
- page_pool_use_xdp_mem registers pool as XDP-redirect-target.
- per-frame XDP redirect via pool memory.

REQ-10: Per-page accounting:
- `page->pp_magic` (validity check).
- `page->pp` (pool back-ref).
- `page->dma_addr` (cached DMA address).
- `page->pp_frag_count` (per-frag refcount).

REQ-11: Per-NUMA-node cache:
- pool.cache.list (per-NUMA-node).
- Slow-path prefers same-node alloc.

REQ-12: Per-pool destroy:
- Wait for all in-flight pages via release-counter.
- Drain cache + ring.
- Per-page DMA-unmap before final-free.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `page_pool_magic_validated` | INVARIANT | per-page->pp_magic checked before recycle; defense against orphan-page recycle. |
| `dma_map_paired_with_unmap` | INVARIANT | every dma_map_page paired with dma_unmap on release. |
| `pages_state_balance` | INVARIANT | hold_cnt - release_cnt == in-flight pages. |
| `napi_safety_per_recycle` | INVARIANT | direct-recycle only on NAPI-CPU; cross-CPU goes to ring. |
| `pool_size_cap_enforced` | INVARIANT | cache.len ≤ pool_size; defense against unbounded growth. |

### Layer 2: TLA+

`net/core/page_pool_lifecycle.tla`:
- Per-page state ∈ {Free, Allocated(pool), Recycled(cache_or_ring), Released}.
- Properties:
  - `safety_dma_mapped_when_allocated` — Allocated implies dma_addr cached.
  - `safety_recycle_only_owned_pages` — Recycled implies page->pp == pool.
  - `liveness_pool_destroy_eventually_drains` — destroy → all pages eventually Released.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `PagePool::alloc_pages` post: page returned with valid pp + pp_magic + dma_addr | `PagePool::alloc_pages` |
| `PagePool::put_page` post: page in cache OR ring OR released | `PagePool::put_page` |
| `PagePool::destroy` post: all in-flight pages released; pool freed | `PagePool::destroy` |
| Per-pool dma_map invariant: page.dma_addr valid iff PP_FLAG_DMA_MAP active | invariants on alloc/release |

### Layer 4: Verus/Creusot functional

`Per-page round-trip: alloc → use → put → recycle → reuse` semantic equivalence: per-page DMA-map preserved across recycle (no remap); subsequent alloc returns DMA-coherent page.

### hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

page_pool-specific reinforcement:

- **Per-page pp_magic check** — defense against orphan-page recycle (cross-pool leak).
- **Per-pool napi_id binding** — defense against direct-cache access from non-NAPI CPU causing race.
- **Lockless ptr_ring** — defense against single-lock contention on hot RX path.
- **Per-CPU NUMA-aware cache** — defense against cross-NUMA cacheline ping-pong.
- **PP_FLAG_DMA_SYNC_DEV explicit** — defense against missing sync causing dirty-cache visibility issue.
- **Per-pool destroy waits in-flight** — defense against UAF on per-driver-shutdown.
- **Per-pool pages_state_hold/release counters** — defense against page-leak detection.
- **DMA-direction enforced** — defense against bidirectional sync on read-only buffer.
- **Per-frag refcount atomic** — defense against torn frag-allocation.
- **PP_FLAG_PAGE_FRAG bounded** — defense against unbounded page-fragmentation.
- **alloc_cache size capped** — defense against unbounded cache growth.

