# Tier-3: mm/slab — SLUB allocator + kmem_cache + freelist hardening

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - mm/slub.c
  - mm/slab_common.c
  - mm/slab.h
  - mm/mempool.c
  - mm/dmapool.c
  - include/linux/slab.h
  - include/linux/mempool.h
  - include/linux/dmapool.h
-->

## Summary
Tier-3 design for the kernel's object-cache allocator: SLUB. Owns `kmem_cache` lifecycle, per-CPU + per-NUMA-node slab caches, freelist hardening (random freelist + freelist pointer obfuscation), the `kmalloc` size-class hierarchy, slab merging, debug + sanitizer integration, mempool fallback for atomic-context allocs, and dmapool for coherent DMA buffers. Consumes `mm/page-allocator.md` for the underlying physical pages.

This is **one of the four MANDATORY Layer-3-Kani-harness subsystems** per `00-overview.md` D4. SLUB freelist invariants (every cached object's `freelist_next` points within the same slab page; no object appears twice in the freelist) are mechanically verified.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| SLUB allocator core | `mm/slub.c` |
| Cross-allocator API | `mm/slab_common.c`, `mm/slab.h` |
| Mempool (allocation backstop for atomic context) | `mm/mempool.c`, `include/linux/mempool.h` |
| DMA pool (coherent DMA buffers) | `mm/dmapool.c`, `include/linux/dmapool.h` |
| Public API headers | `include/linux/slab.h` |

(Upstream removed SLAB and SLOB allocators — only SLUB remains. CONFIG_SLUB_TINY is the small variant for embedded; CONFIG_SLUB is the default.)

## Compatibility contract

### `/proc/slabinfo`

Format-identical to upstream. Per-cache stats: `<active_objs> <num_objs> <objsize> <objperslab> <pagesperslab> : tunables 0 0 0 : slabdata <active_slabs> <num_slabs> <sharedavail>`. Existing tools (`slabtop`, `slabinfo`) work unmodified.

### `/sys/kernel/slab/<cache>/*`

When CONFIG_SLUB_DEBUG=y or CONFIG_SLUB_DEBUG_ON=y, per-cache sysfs attributes: `align`, `cache_dma`, `cpu_slabs`, `ctor`, `destroy_by_rcu`, `failslab`, `hwcache_align`, `min_partial`, `objects`, `objects_partial`, `object_size`, `objs_per_slab`, `order`, `partial`, `poison`, `reclaim_account`, `red_zone`, `remote_node_defrag_ratio`, `sanity_checks`, `shrink`, `skip_kfence`, `slab_size`, `slabs`, `slabs_cpu_partial`, `store_user`, `total_objects`, `trace`, `usersize`, `validate`. Identical.

### `kmalloc` size classes

`include/linux/slab.h` enumerates the size classes: `kmalloc-8`, `kmalloc-16`, `kmalloc-32`, `kmalloc-64`, `kmalloc-96`, `kmalloc-128`, `kmalloc-192`, `kmalloc-256`, `kmalloc-512`, `kmalloc-1k`, `kmalloc-2k`, `kmalloc-4k`, `kmalloc-8k`, ... up to `kmalloc-8192k`. Plus `dma-kmalloc-*`, `kmalloc-cg-*` (cgroup-accounted), `kmalloc-rcl-*` (reclaimable). Identical name set so `/proc/slabinfo` listing matches.

### Hardening features

- **CONFIG_SLAB_FREELIST_HARDENED**: freelist pointers obfuscated (XOR with a per-cache random + the address itself); detects naive overwrites
- **CONFIG_SLAB_FREELIST_RANDOM**: freelist order randomized per slab page; defeats deterministic spray
- **CONFIG_SLAB_BUCKETS**: per-callsite bucketing (newer upstream)
- **CONFIG_SLUB_DEBUG**: full SLUB debug (poison, redzone, user-track)

Default-on per `00-security-principles.md` knob inventory.

### kmem_cache_create API

```c
struct kmem_cache *kmem_cache_create(
    const char *name,
    unsigned int size,
    unsigned int align,
    slab_flags_t flags,
    void (*ctor)(void *)
);
```

Rust counterpart per upstream rust-for-linux abstractions, extended:

```rust
pub struct KmemCache<T> {
    inner: Opaque<bindings::kmem_cache>,
    _phantom: PhantomData<T>,
}

impl<T> KmemCache<T> {
    pub fn new(name: &CStr, flags: SlabFlags) -> Result<Self>;
    pub fn alloc(&self, gfp: Gfp) -> Result<KBox<T>>;
    pub fn alloc_node(&self, gfp: Gfp, node: NodeId) -> Result<KBox<T>>;
    // Reuses kernel::alloc::KBox<T> for ownership semantics
}
```

Per-type slab caches (AUTOSLAB-equivalent — `00-security-principles.md` mandate) are mandatory for AUTOSLAB hardening: every distinct type allocated has its own `KmemCache<T>`.

## Requirements

- REQ-1: SLUB is the sole upstream-supported in-tree allocator; Rookery does not implement SLAB or SLOB.
- REQ-2: Per-CPU slab pool fast-path is lock-free (preempt-disabled region only). Slow-path uses per-node `n->list_lock` (spinlock).
- REQ-3: `kmalloc` size classes match upstream exactly (same names, same sizes, same merge candidates).
- REQ-4: Slab merging behavior matches upstream: caches with compatible flags + sizes merge (unless `SLAB_NO_MERGE`); CONFIG_SLAB_MERGE_DEFAULT=y default-on per upstream. **Rookery flips this default per `00-security-principles.md` knob inventory: `slab_nomerge=y` boot param default-on; mergeable caches stay separate** (defeats cross-cache UAF spray).
- REQ-5: `/proc/slabinfo` content format-identical for equivalent kernel state.
- REQ-6: `/sys/kernel/slab/<cache>/*` attribute set + value semantics identical when CONFIG_SLUB_DEBUG=y.
- REQ-7: AUTOSLAB-equivalent: every Rookery `kmem_cache_create::<T>(...)` produces a per-type slab cache; type-tagging via `core::any::type_name::<T>()` for `/proc/slabinfo` listing + tracing.
- REQ-8: CONFIG_SLAB_FREELIST_HARDENED=y default-on; CONFIG_SLAB_FREELIST_RANDOM=y default-on (per `00-security-principles.md` knob inventory).
- REQ-9: SLUB debug: poison + redzone + user-track functional when CONFIG_SLUB_DEBUG_ON=y at runtime via `slub_debug=PFZU` boot param. Output format-identical to upstream's "BUG: kmalloc-X object @ Y: poison overwritten" reports.
- REQ-10: Mempool fallback: `mempool_alloc` semantics match upstream (size-fixed pool returns from reserve when GFP-failable alloc fails).
- REQ-11: DMA pool: `dma_pool_create` / `dma_pool_alloc` semantics match upstream; coherent DMA-API integration (cross-ref `kernel/00-overview.md` § dma-mapping.md).
- REQ-12: KASAN integration (cross-ref `mm/sanitizers.md`): every slab alloc/free is reported to KASAN; quarantine-on-free for KASAN-tracked caches.
- REQ-13: KFENCE integration: configurable percentage of allocations diverted to KFENCE for guard-page UAF/OOB detection.
- REQ-14: SLUB freelist invariant Layer-3 Kani harness MANDATORY (per `mm/00-overview.md` REQ-14 / `00-overview.md` D4).
- REQ-15: TLA+ model `models/mm/slub_per_cpu.tla` proves per-CPU partial-list + tail-pointer invariants under concurrent CPU-local alloc + IPI-driven cache drain.
- REQ-16: MEMORY_SANITIZE for slab (zero on free for sensitive caches, default-on for all others) per `00-security-principles.md`. `SLAB_NO_ZERO_ON_FREE` flag for explicit opt-out.
- REQ-17: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: A grep over Rookery for `mm/slab.c` or `mm/slob.c` finds zero references; only SLUB. (covers REQ-1)
- [ ] AC-2: A 32-CPU concurrent kmem_cache_alloc benchmark shows ≥ 95% upstream throughput; zero freelist corruption under stress. (covers REQ-2, REQ-14)
- [ ] AC-3: A diff of `cat /proc/slabinfo | head` between Rookery and upstream after equivalent boot is empty (modulo dynamic counts). Same names + object sizes. (covers REQ-3, REQ-5)
- [ ] AC-4: With `slab_nomerge=y` (default), `cat /proc/slabinfo` shows zero merged caches (each has unique name). With `slab_nomerge=n`, mergeable caches consolidate per upstream. (covers REQ-4)
- [ ] AC-5: When CONFIG_SLUB_DEBUG=y, every documented attribute under `/sys/kernel/slab/<cache>/` exists and round-trips correctly. (covers REQ-6)
- [ ] AC-6: A test allocates `Box::new::<MyType>()`-equivalent; `cat /proc/slabinfo` shows a `MyType`-named cache. (covers REQ-7)
- [ ] AC-7: A heap-overflow exploit attempting freelist-pointer manipulation against a `KmemCache<MyType>` is detected (with CONFIG_SLAB_FREELIST_HARDENED=y) and produces an upstream-compatible "Freepointer corrupted" error. (covers REQ-8)
- [ ] AC-8: With `slub_debug=PFZU`, a deliberate UAF + redzone overflow + use-uninitialized are all detected; report format matches upstream. (covers REQ-9)
- [ ] AC-9: A driver using mempool to fall back to a reserved pool succeeds in atomic context after main alloc fails (fault-injection test). (covers REQ-10)
- [ ] AC-10: A DMA-coherent driver allocates from a `DmaPool` and the resulting buffer's physical address is DMA-addressable. (covers REQ-11)
- [ ] AC-11: KASAN-built kernel detects a UAF on a slab object; output matches upstream's "BUG: KASAN: use-after-free in ..." format. (covers REQ-12)
- [ ] AC-12: KFENCE-built kernel diverts the configured percentage to guard pages; a test triggering OOB on a KFENCE'd page is caught. (covers REQ-13)
- [ ] AC-13: `make verify` passes `kani::proofs::mm::slab::*` Kani harnesses. (covers REQ-14)
- [ ] AC-14: `make tla` passes `models/mm/slub_per_cpu.tla` (TLC). (covers REQ-15)
- [ ] AC-15: A "freed-then-allocated" object in a non-`SLAB_NO_ZERO_ON_FREE` cache contains zeroes when re-allocated (test). (covers REQ-16)
- [ ] AC-16: Hardening section present and follows template. (covers REQ-17)

## Architecture

### Rust module organization

- `kernel::mm::slab::cache` — `KmemCache<T>` lifecycle
- `kernel::mm::slab::alloc` — kmalloc / kfree fast-path + slow-path
- `kernel::mm::slab::pcp` — per-CPU slab pool fast-path
- `kernel::mm::slab::node` — per-NUMA-node slab pool
- `kernel::mm::slab::merge` — slab merging dispatch
- `kernel::mm::slab::hardening` — CONFIG_SLAB_FREELIST_HARDENED + RANDOM helpers
- `kernel::mm::slab::debug` — SLUB debug (poison, redzone, track)
- `kernel::mm::slab::sysfs` — `/sys/kernel/slab/<cache>/*`
- `kernel::mm::slab::proc` — `/proc/slabinfo`
- `kernel::mm::mempool::Mempool<T>` — mempool wrapper
- `kernel::mm::dmapool::DmaPool` — DMA pool wrapper

### Key data structures

- `KmemCache<T>` — newtype wrapping upstream `struct kmem_cache *`; generic-typed for AUTOSLAB
- `SlabPage` — per-slab-page metadata (overlays `struct page` flags)
- `Freelist` — typed pointer; with hardening, XOR-obfuscated
- `PerCpuSlab<T>` — per-CPU partial-slab list + freelist
- `NodeSlab<T>` — per-node partial / full / free lists

### Locking and concurrency

- **Per-CPU partial-list (fast path)**: lock-free; preempt-disabled region. Atomic CAS on freelist head + version counter to detect ABA.
- **Per-node `n->list_lock`**: raw_spinlock. Held during partial-list rebalance + slab-page promotion.
- **Cache create / destroy**: global `slab_mutex`. Slow path; only at module load / unload.
- **CPU hotplug**: `cpuhp_setup_state` registers a callback; per-CPU pool drained on offline.

TLA+ model `models/mm/slub_per_cpu.tla` (mandatory per `mm/00-overview.md`) proves the lock-free fast path is correct under all interleavings of CPU-local alloc + remote drain.

### Error handling

- `Err(ENOMEM)` — slab page allocation failed (consumed page-allocator returned ENOMEM)
- `Err(EINVAL)` — bad arguments to `kmem_cache_create` (size 0, align not power of 2, etc.)
- `Err(EBUSY)` — `kmem_cache_destroy` called with allocated objects still outstanding (panic on debug; ignored on production matches upstream behavior)

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Per-CPU pool freelist push/pop (atomic CAS) | `kani::proofs::mm::slab::pcp_safety` |
| Slab page split (when allocating fresh slab) | `kani::proofs::mm::slab::split_safety` |
| Freelist pointer write (with hardening XOR) | `kani::proofs::mm::slab::freelist_xor_safety` |
| Slab merge candidate match | `kani::proofs::mm::slab::merge_safety` |
| Mempool refill | `kani::proofs::mm::mempool::refill_safety` |
| DMA pool block-bitmap update | `kani::proofs::mm::dmapool::bitmap_safety` |

### Layer 2: TLA+ models

- `models/mm/slub_per_cpu.tla` (mandatory per `mm/00-overview.md` Layer 2) — proves per-CPU partial-list + tail-pointer invariants under concurrent CPU-local allocation + IPI-driven cache drain. Safety + (where meaningful) liveness.
- `models/mm/mempool_blocking.tla` (NEW) — proves mempool wait-queue: an alloc that blocks waiting for a free element gets exactly one wakeup when one is freed.

### Layer 3: Kani harnesses for data-structure invariants (MANDATORY)

Per `mm/00-overview.md` REQ-14 / `00-overview.md` D4 — slab freelist is one of the four mandatory Layer-3 areas:

| Data structure | Invariant | Harness |
|---|---|---|
| SLUB freelist (per slab page) | "Every cached object's `freelist_next` points within the same slab page" + "No object appears twice in the freelist" | `kani::proofs::mm::slab::freelist_invariants` |
| Per-CPU pool | "Pool size ≤ cache->cpu_partial_slabs setting"; "Active slab page (`c->page`) is fully owned by this CPU" | `kani::proofs::mm::slab::pcp_invariants` |
| Per-node lists | "Each slab page is on exactly one of: cpu, partial, full" | `kani::proofs::mm::slab::node_lists_invariants` |
| Cache merge table | "Merged caches have compatible flags + sizes; their `kmem_cache` pointers alias" | `kani::proofs::mm::slab::merge_invariants` |

### Layer 4: Functional correctness (opt-in; declared in `mm/00-overview.md`)

- **Slab cache color/offset arithmetic** via Creusot — proves: for any slab page of objects of size S, the i-th object lives at offset `color_offset + i * S` where `color_offset` is the per-page color rotation. Tractable.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **AUTOSLAB** (per-cache type tagging) | Every `KmemCache<T>` has a unique slab cache; `core::any::type_name::<T>()` provides the per-type name in `/proc/slabinfo` | § Mandatory |
| **MEMORY_SANITIZE** (zero on free) | Default-on per `vm.zero_on_free=1`; `SLAB_NO_ZERO_ON_FREE` per-cache flag for explicit opt-out (perf-critical caches that immediately overwrite the object) | § Default-on configurable off |
| **PAX_USERCOPY-equivalent** (slab cache type-tagging for copy_*_user) | `kmem_cache_create::<T>(SLAB_USERCOPY_OK)` flags type-safe-for-copy types; raw `copy_to/from_user` against an unflagged cache rejected at compile time via the `UserCopyOk` trait | (cross-ref `lib/usercopy.md`) |
| **Slab merge denial** (`slab_nomerge` boot param default-on) | Defeats cross-cache UAF spray by ensuring each cache is unique even if size-compatible | § Default-on configurable off (kernel cmdline `slab_nomerge=y`) |
| **CONFIG_SLAB_FREELIST_HARDENED** | Freelist pointer obfuscated via XOR with random + address; detects naive overwrites | § Default-on (Kconfig=y) |
| **CONFIG_SLAB_FREELIST_RANDOM** | Per-slab-page freelist order randomized; defeats deterministic spray | § Default-on (Kconfig=y) |

### Row-1 features consumed by this component

- **REFCOUNT**: `kmem_cache->refcount` is a `Refcount` (saturating)
- **SIZE_OVERFLOW**: object-size + alignment arithmetic uses checked operators
- **CONSTIFY**: per-cache `cache_size_classes[]` table is `static const`
- **KERNEXEC**: this component's text is RX/RO

### Row-2 / GR-RBAC integration

The slab allocator runs below LSM hooks. memcg-charging interaction (cross-ref `mm/memcg.md`) is the only LSM-relevant interaction; that lives in memcg, not here.

### Userspace-visible behavior changes

Per Axiom 4 of `00-security-principles.md`:
- **`slab_nomerge=y` boot-param default**: caches that would have merged on upstream are separate on Rookery. Effect: marginally higher slab overhead (~few percent); userspace-invisible except via `/proc/slabinfo` distinct-cache count. Configurable off via boot param.
- **`vm.zero_on_free=1` default**: per allocator default; documented in `mm/page-allocator.md` Hardening section too.

### Verification

(See § Verification above.)

## Open Questions

(none — slab semantics are fully specified by SLUB design + upstream conventions; no architectural ambiguities at this tier)

## Out of Scope

- SLAB / SLOB allocators (removed upstream)
- 32-bit-only paths
- Implementation code
