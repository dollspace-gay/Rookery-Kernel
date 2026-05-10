# Tier-3: drivers/iommu/iova.c — IOVA allocator (rb-tree + per-CPU rcache magazines for fast alloc)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/iommu/00-overview.md
upstream-paths:
  - drivers/iommu/iova.c
  - include/linux/iova.h
-->

## Summary

The IOVA (IO Virtual Address) allocator — every `dma_map_single` / `dma_map_sg` on an IOMMU-attached device allocates an IOVA from a per-domain pool managed here. Per-pool global rb-tree tracks free + allocated IOVA ranges; per-CPU magazine cache (rcache) provides single-cycle alloc/free for hot path of common-size allocations. Backbone of dma-iommu performance — without rcache, every DMA op would contend on per-pool rb-tree mutex causing massive scaling problems on multi-CPU IO-heavy workloads (NVMe-of, mlx5 RDMA, ice 100GbE).

This Tier-3 covers `drivers/iommu/iova.c` (~1000 lines) — implementation of `iova_domain` + per-CPU `iova_rcache` + IOVA reservation/allocation/release/release-batch.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct iova_domain` | per-domain IOVA pool | `kernel::iommu::iova::IovaDomain` |
| `struct iova` | per-allocation rb-tree node | `kernel::iommu::iova::Iova` |
| `struct iova_rcache` | per-CPU magazine cache | `kernel::iommu::iova::IovaRcache` |
| `init_iova_domain(iovad, granule, start_pfn)` | init pool | `IovaDomain::init` |
| `put_iova_domain(iovad)` | release pool | `IovaDomain::put` (Drop) |
| `iova_cache_get()` / `_put()` | global iova kmem_cache lifecycle | `IovaCache::get_global` / `_put_global` |
| `alloc_iova(iovad, size, limit_pfn, size_aligned)` | per-pool alloc (slow path, rb-tree) | `IovaDomain::alloc_slow` |
| `__alloc_and_insert_iova_range(iovad, size, limit_pfn, &new_iova, size_aligned)` | core rb-tree alloc | `IovaDomain::alloc_and_insert_range` |
| `alloc_iova_fast(iovad, size, limit_pfn, flush_rcache)` | per-CPU rcache fast-path alloc | `IovaDomain::alloc_fast` |
| `free_iova(iovad, pfn)` | per-pool free (slow path, rb-tree) | `IovaDomain::free_slow` |
| `__free_iova(iovad, iova)` | core rb-tree free | `IovaDomain::__free` |
| `free_iova_fast(iovad, pfn, size)` | per-CPU rcache fast-path free | `IovaDomain::free_fast` |
| `iova_rcache_put(iovad, pfn, size)` | put IOVA into per-CPU magazine | `IovaRcache::put` |
| `iova_rcache_get(iovad, size, limit_pfn)` | get IOVA from per-CPU magazine | `IovaRcache::get` |
| `find_iova(iovad, pfn)` | rb-tree lookup | `IovaDomain::find` |
| `reserve_iova(iovad, pfn_lo, pfn_hi)` | reserve specific range (e.g., MSI-X window) | `IovaDomain::reserve` |
| `copy_reserved_iova(from, to)` | copy reserved-range list to another domain | `IovaDomain::copy_reserved` |
| `init_iova_flush_queue(iovad, flush_cb)` / `_free_flush_queue` | per-iovad deferred-flush queue | `IovaDomain::init_flush_queue` / `_free_flush_queue` |
| `queue_iova(iovad, pfn, pages, &freelist)` | defer iova free + flush via fq | `IovaDomain::queue_iova` |
| `fq_flush_timeout(timer)` | per-iovad fq flush timer callback | `IovaDomain::fq_flush_timeout` |
| `iova_domain_init_rcaches(iovad)` / `_free_rcaches(...)` | per-domain rcache init | `IovaDomain::init_rcaches` / `_free_rcaches` |

## Compatibility contract

REQ-1: `init_iova_domain(iovad, granule, start_pfn)` initializes per-pool rb-tree + per-CPU rcache (one rcache per CPU, with magazines per power-of-2 size class).

REQ-2: `alloc_iova_fast(iovad, size, limit_pfn, flush_rcache=true)` fast-path:
- Round size up to nearest power-of-2 size class.
- Per-CPU rcache for that size class: pop from per-CPU magazine; if hit, return immediately.
- If miss: call slow-path `alloc_iova` (rb-tree).

REQ-3: `free_iova_fast(iovad, pfn, size)` fast-path:
- Round size up; push to per-CPU rcache magazine.
- If magazine full: try push to depot of full magazines; if depot full, fall back to slow-path `free_iova` (rb-tree).

REQ-4: Per-CPU magazine size: 128 IOVAs per magazine; depot per-pool holds N full magazines (typically 32); per-CPU rcache holds 2 magazines (loaded + previous, to avoid bouncing on alloc/free interleave).

REQ-5: Per-pool rb-tree (`iovad.rbroot`): per-allocation `struct iova` node with `pfn_lo`, `pfn_hi`; allocator scans from `cached_node` hint backwards looking for hole.

REQ-6: Per-pool `cached_node` hint: tracks last-allocated IOVA; subsequent alloc starts here for locality (defense against fragmentation under sustained alloc/free).

REQ-7: Per-pool `granule` (typically 4 KB); all IOVAs aligned to granule.

REQ-8: Per-pool reserved-IOVA list: `reserve_iova(iovad, lo, hi)` reserves a specific range (used for MSI-X window 0xFEE00000-0xFEEFFFFF on x86, RMRR ranges from DMAR firmware table).

REQ-9: Deferred-flush queue (CONFIG_IOMMU_DMA_FLUSH_QUEUE): when `iommu.strict=0`, IOVA frees deferred to per-CPU fq + batched flush via `fq_flush_timeout` hrtimer (default 10ms); reduces per-free TLB-flush overhead at cost of reuse-after-flush latency.

REQ-10: Per-pool init_iova_flush_queue installs per-pool flush_cb (per-driver callback for batched TLB invalidate); fq_flush_timeout calls flush_cb with batched range list.

REQ-11: rcache flush on memory pressure: `IovaRcache::flush_all` empties all magazines back to rb-tree (called by shrinker when memory low).

REQ-12: Per-pool `iova_rcache_range`: per-power-of-2 size class up to IOVA_RANGE_CACHE_MAX_SIZE (default 4 MB); larger sizes go directly to slow-path.

## Acceptance Criteria

- [ ] AC-1: NVMe-PCIe IO at sustained 4M+ IOPS does not bottleneck on iova allocator (verified via `perf top` showing < 5% in iova path).
- [ ] AC-2: mlx5 100GbE sustained line-rate doesn't bottleneck on iova alloc.
- [ ] AC-3: rcache hit-rate > 90% for typical NVMe + NIC workloads (verified via `/sys/kernel/debug/iommu_groups/<N>/iova_rcache_stat`).
- [ ] AC-4: Reserved-IOVA test: MSI-X window correctly reserved; subsequent alloc never returns IOVA in [0xFEE00000, 0xFEEFFFFF].
- [ ] AC-5: Deferred-flush queue test: with `iommu.strict=0`, IOVA frees coalesced into per-CPU fq; flush callback invoked at 10ms intervals.
- [ ] AC-6: Memory pressure test: shrinker invocation flushes per-CPU rcaches back to rb-tree; subsequent alloc still succeeds.
- [ ] AC-7: kselftest IOVA-related tests pass.

## Architecture

`IovaDomain` lives in `kernel::iommu::iova::IovaDomain`:

```
struct IovaDomain {
  iova_rbtree_lock: SpinLock<()>,
  rbroot: RBTree<Iova>,                 // per-allocation tree
  cached_node: AtomicPtr<RBNode<Iova>>,  // last-alloc hint (rightmost free hole)
  cached32_node: AtomicPtr<RBNode<Iova>>,// hint for 32-bit-DMA-mask allocs
  granule: u64,                          // typically PAGE_SIZE = 4096
  start_pfn: u64,                        // pool start
  dma_32bit_pfn: u64,                    // 32-bit-DMA-mask boundary
  max32_alloc_size: AtomicU64,
  fq_lock: SpinLock<()>,
  fq: PerCpu<KBox<IovaFq>>,              // per-CPU flush queue
  fq_timer: HrTimer,                     // deferred-flush hrtimer
  flush_cb: Option<fn(&IovaDomain)>,     // per-driver batched TLB flush callback
  iova_alloc_dirty_callback: Option<...>,
  rcaches: VarLenArray<IovaRcache>,      // per-power-of-2 size class
  cpuhp_dead: CpuHpState,                // CPU-hotplug callback
  flush_queue_alloc_total: AtomicU64,
  flush_queue_alloc_flush: AtomicU64,
}

struct Iova {
  pfn_lo: u64,
  pfn_hi: u64,
}

struct IovaRcache {
  lock: SpinLock<()>,
  cpu_rcaches: PerCpu<IovaCpuRcache>,
  depot: KBox<IovaMagazine>,             // singly-linked list of full magazines
  depot_size: AtomicU32,
}

struct IovaCpuRcache {
  loaded: KBox<IovaMagazine>,            // current magazine (alloc/free here)
  prev: KBox<IovaMagazine>,              // previous magazine (alloc here if loaded empty; free here if loaded full)
}

struct IovaMagazine {
  size: u32,                             // # IOVAs currently in pfns[]
  pfns: [u64; IOVA_MAG_SIZE /* = 128 */],
}
```

Init `IovaDomain::init(iovad, granule, start_pfn)`:
1. Initialize empty rb-tree.
2. Insert anchor: `Iova { pfn_lo: 0, pfn_hi: 0 }` (left sentinel) and `Iova { pfn_lo: ULONG_MAX, pfn_hi: ULONG_MAX }` (right sentinel).
3. Initialize per-CPU rcaches for each size class (size = 2^k pages for k = 0..log2(IOVA_RANGE_CACHE_MAX_SIZE)).
4. Init fq + fq_timer (deferred 10ms flush).

Fast alloc `IovaDomain::alloc_fast(iovad, size, limit_pfn, flush_rcache)`:
1. `size_log = order_base_2(size)`.
2. If `size_log >= MAX_RCACHE_LOG_SIZE`: fall to `alloc_slow`.
3. `rcache = &iovad.rcaches[size_log]`.
4. `rcache.cpu_rcaches[smp_processor_id()]`:
   - If `loaded.size > 0`: pop pfn from loaded; return.
   - Else if `prev.size > 0`: swap loaded ↔ prev; pop from new loaded; return.
   - Else: try take from depot via `IovaRcache::depot_pop_locked`; if got magazine, install as new loaded; pop; return.
5. Fall to `alloc_slow(iovad, size, limit_pfn, true)`.

Slow alloc `IovaDomain::alloc_slow(iovad, size, limit_pfn, size_aligned)`:
1. Take iova_rbtree_lock.
2. Walk rb-tree from `cached_node` backwards looking for hole >= size.
3. If found: insert new Iova node; update `cached_node` hint; return pfn.
4. If not found AND flush_rcache: drop lock; flush all rcaches (return all magazines to rb-tree); retry.
5. If still not found: return DMA_MAPPING_ERROR.

Fast free `IovaDomain::free_fast(iovad, pfn, size)`:
1. `size_log = order_base_2(size)`.
2. If `size_log >= MAX_RCACHE_LOG_SIZE`: fall to `free_slow`.
3. `rcache = &iovad.rcaches[size_log]`.
4. `rcache.cpu_rcaches[smp_processor_id()]`:
   - If `loaded.size < IOVA_MAG_SIZE`: push pfn to loaded; return.
   - Else if `prev.size < IOVA_MAG_SIZE`: swap loaded ↔ prev; push to new loaded; return.
   - Else (both full): try push loaded to depot via `IovaRcache::depot_push_locked`; reset loaded; push pfn; return.
   - If depot full: fall to `free_slow`.

Slow free `IovaDomain::free_slow(iovad, pfn)`:
1. `find_iova(iovad, pfn)` → Iova node.
2. Take iova_rbtree_lock.
3. Remove node from rb-tree.
4. Release node memory.

Deferred-flush queue (`iommu.strict=0`):
- Per-CPU `IovaFq` accumulates pending IOVA frees + count.
- On every Nth free (or at hrtimer), `fq_flush_timeout`:
  - For each per-CPU fq: collect freelist.
  - Call `flush_cb(iovad)` to invalidate IOMMU TLB for batch.
  - For each pfn in freelist: `__free_iova(iovad, pfn)`.

Reserved IOVA `reserve_iova(iovad, lo, hi)`:
1. Allocate Iova node with `pfn_lo=lo, pfn_hi=hi`.
2. Insert into rb-tree as already-allocated.
3. Subsequent alloc skips this range.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `iova_no_uaf` | UAF | `Iova` node freed only after removal from rb-tree; per-magazine pfn arrays bounded. |
| `magazine_no_oob` | OOB | per-magazine pfns array index < IOVA_MAG_SIZE; size < MAG_SIZE invariant maintained. |
| `rb_tree_consistency` | INVARIANT | rb-tree binary-search invariant maintained across insert/delete; per-node pfn_lo < pfn_hi. |
| `cached_node_no_uaf` | UAF | `cached_node` ptr always points to live tree node OR set to NULL on remove. |

### Layer 2: TLA+

`models/iommu/iova_alloc.tla` (parent-declared): proves IOVA allocator + per-cpu rcache + global rb-tree concurrency: alloc + free under high contention never produce duplicate IOVA, never leak slot.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `alloc_iova_fast` post: returned pfn is in [start_pfn, limit_pfn] AND not allocated to any other concurrent caller | `IovaDomain::alloc_fast` |
| `free_iova_fast` invariant: returned pfn was previously allocated (not double-free) | `IovaDomain::free_fast` (debug-asserted) |
| `reserve_iova` post: subsequent alloc never returns pfn in [lo, hi] | `IovaDomain::reserve` |
| Per-rcache `loaded.size + prev.size + sum(depot.size)` invariant: equals total IOVAs in this size-class's per-CPU + depot caches | rcache mgmt |

### Layer 4: Verus/Creusot functional

`alloc_iova_fast(size) → use → free_iova_fast(pfn, size)` round-trip: per-pool total free + allocated count remains constant (modulo reserved + active allocations).

## Hardening

(Inherits row-1 features from `drivers/iommu/00-overview.md` § Hardening.)

iova-specific reinforcement:

- **Per-rb-tree-node refcount-style** — each Iova node is uniquely owned by either rb-tree OR per-CPU magazine OR depot magazine; never two simultaneously.
- **Per-CPU rcache cpuhp callback** — on CPU offline, that CPU's rcache magazines flushed back to depot; defense against per-CPU IOVA leak on hotplug.
- **Magazine pfn array bounded** — per-magazine cap = 128; defense against unbounded magazine growth.
- **Reserved-IOVA enforced at alloc** — every alloc validates against reserved-range list; defense against driver alloc'ing into MSI-X / RMRR window.
- **Cached-node hint bounded** — cached_node walked backwards only N steps before fallback to full tree-walk; defense against pathological cached_node placement.
- **Deferred-flush queue size cap** — per-CPU fq batch capped at 256 IOVAs; over-cap forces immediate flush; defense against fq-flood causing OOM.
- **fq_timer interval bounded** — minimum 1ms, maximum 100ms; defense against userspace-controllable flush latency tuning.
- **Per-pool size limit** — IOVA range bounded by domain.geometry.aperture_end (cross-ref `iommu-core.md`).

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-vendor IOMMU TLB-flush callback (covered in `intel-iommu.md` + `amd-iommu.md` Tier-3s)
- DMA-API integration (covered in `kernel/dma/dma-iommu.md` future Tier-3)
- iommu_domain abstraction (covered in `iommu-core.md` Tier-3)
- 32-bit-only paths
- Implementation code
