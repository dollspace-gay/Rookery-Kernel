# Tier-3: mm/page-allocator — buddy allocator, per-CPU pagesets, zones, folios

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - mm/page_alloc.c
  - mm/page_isolation.c
  - mm/page_reporting.c
  - mm/page_owner.c
  - mm/page_ext.c
  - mm/page_idle.c
  - mm/page_poison.c
  - mm/page_table_check.c
  - mm/page_vma_mapped.c
  - mm/folio-compat.c
  - include/linux/mm_types.h
  - include/linux/page-flags.h
  - include/linux/gfp.h
  - include/linux/gfp_types.h
  - include/linux/mmzone.h
-->

## Summary
Tier-3 design for the kernel's lowest-level physical memory allocator: the buddy system. Owns per-NUMA-node + per-zone free lists at power-of-2 sizes, per-CPU pagesets (lock-free fast-path), zone selection per GFP flags, page isolation (memory-hotplug + CMA), page reporting (free pages reported to hypervisor), `struct page` + `struct folio` per-page metadata, and the page-flag bit definitions visible via `/proc/<pid>/pagemap`.

This is **one of the four MANDATORY Layer-3-Kani-harness subsystems** per `00-overview.md` D4. Buddy invariants — disjoint free lists per migrate-type, no double-allocation, free-page-order matches the list it lives in — are mechanically verified.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Buddy core | `mm/page_alloc.c` |
| Per-CPU pageset | `mm/page_alloc.c` (interleaved with buddy core) |
| Page isolation (memory hotplug + CMA) | `mm/page_isolation.c` |
| Page reporting (to hypervisor for memory ballooning) | `mm/page_reporting.c` |
| `struct page` + `struct folio` definitions | `include/linux/mm_types.h` |
| Page flags | `include/linux/page-flags.h` |
| GFP flags | `include/linux/gfp.h`, `include/linux/gfp_types.h` |
| Zones + nodes (NUMA) | `include/linux/mmzone.h` |
| Page accounting | `mm/page_owner.c` (per-page allocation backtrace), `mm/page_ext.c` (per-page extension) |
| Idle-page tracking | `mm/page_idle.c` (`/sys/kernel/mm/page_idle/`) |
| Page poison (debug) | `mm/page_poison.c` |
| Page table check (debug) | `mm/page_table_check.c` |
| Reverse-mapping helper | `mm/page_vma_mapped.c` |
| Folio compat (transitional) | `mm/folio-compat.c` |

## Compatibility contract

### `/proc` surfaces

| Path | Compat level |
|---|---|
| `/proc/buddyinfo` | Format-identical: per-node + per-zone + per-order free counts |
| `/proc/pagetypeinfo` | Format-identical: per-migrate-type free lists |
| `/proc/zoneinfo` | Format-identical: per-zone watermarks, free pages, slab pages, etc. |
| `/proc/<pid>/pagemap` | Bit-identical: per-page bitfield encoding PFN, swap, file/anon, exclusive, soft-dirty, page-table-check fields |
| `/proc/meminfo` (relevant fields: MemTotal, MemFree, MemAvailable, Buffers, Cached, KReclaimable, Slab, KernelStack, PageTables, Bounce, CmaTotal, CmaFree, HardwareCorrupted, AnonHugePages, HugePages_*, DirectMap{4k,2M,1G}) | Field-identical |

### Page flag bit positions

`include/linux/page-flags.h` enum `pageflags` defines bit positions used in `struct page->flags`. The bits are NOT direct UAPI but ARE userspace-visible via `/proc/<pid>/pagemap` (which exports a per-page bitfield encoding the relevant flags) AND `/sys/kernel/page_owner/`. Bit positions MUST match upstream so existing tools (`pmap`, `tools/vm/page-types`, `bpftrace` scripts) parse identically.

### `struct page` / `struct folio` layout

Internal but observed:
- First cache line (offset 0..63) is read by inline-fastpath macros (`PageCompound`, `PageDirty`, `PageLocked`, etc.) baked into out-of-tree drivers and BPF programs
- `struct folio` first cache line ditto

Compat target: layout-equivalent (same field offsets) for the cache-line-zero fields. Deeper fields may be reorganized.

### GFP flag semantics

| Flag class | Flags | Semantics |
|---|---|---|
| Sleep policy | `GFP_KERNEL` (sleepable), `GFP_NOWAIT` (no sleep, no reclaim), `GFP_ATOMIC` (no sleep, may dip into reserves), `GFP_NOIO` / `GFP_NOFS` (sleepable but bounded I/O context) | Identical |
| Zone hints | `GFP_DMA`, `GFP_DMA32`, `GFP_HIGHUSER`, `GFP_HIGHUSER_MOVABLE` | Identical |
| Modifiers | `__GFP_ZERO` (zero on alloc), `__GFP_NOFAIL` (retry forever), `__GFP_NORETRY` (single attempt), `__GFP_RETRY_MAYFAIL`, `__GFP_NOWARN`, `__GFP_THISNODE`, `__GFP_HARDWALL`, `__GFP_ACCOUNT` (memcg-charge), `__GFP_RECLAIM`, `__GFP_DIRECT_RECLAIM`, `__GFP_KSWAPD_RECLAIM`, `__GFP_HIGH`, `__GFP_MEMALLOC`, `__GFP_NOMEMALLOC`, `__GFP_COMP` (compound page), `__GFP_NOLOCKDEP` | Identical |

## Requirements

- REQ-1: Buddy allocator preserves the upstream-defined zone hierarchy: per-node, per-zone, per-migrate-type, per-order free lists. Per-CPU pageset fast-path matches upstream's `pcp` design.
- REQ-2: GFP flag semantics (zone selection, sleep policy, retry behavior, zeroing, memcg charging, NUMA node hint) match upstream byte-for-byte. Allocations with `__GFP_ZERO` return zero-filled memory.
- REQ-3: `/proc/buddyinfo`, `/proc/pagetypeinfo`, `/proc/zoneinfo` content is format-identical for equivalent kernel state.
- REQ-4: `struct page` flag bit positions match upstream (defined in `include/linux/page-flags.h`).
- REQ-5: `struct folio` first cache line layout is identical so drivers and BPF programs accessing folio fields via inline macros work unmodified.
- REQ-6: `/proc/<pid>/pagemap` per-page bitfield is bit-identical.
- REQ-7: Per-CPU pageset fast-path is lock-free (preempt-disabled region only). Buddy slow-path uses per-zone `zone->lock` (raw spinlock).
- REQ-8: NUMA node selection per GFP flag + caller's mempolicy (cross-ref `mm/numa-mempolicy.md`) matches upstream.
- REQ-9: Page isolation for memory hotplug (`offline_pages` flow) and CMA preserves upstream semantics.
- REQ-10: Page reporting to hypervisor (`page_reporting.c`) preserves the hypercall ABI (cross-arch contract honored).
- REQ-11: page_owner debug feature (`/sys/kernel/page_owner/`) preserves output format so `tools/vm/page_owner_sort` works unmodified.
- REQ-12: page_idle tracking preserves `/sys/kernel/mm/page_idle/bitmap` ABI.
- REQ-13: Buddy invariant Layer-3 Kani harnesses are MANDATORY (per `mm/00-overview.md` REQ-14 / `00-overview.md` D4).
- REQ-14: TLA+ model `models/mm/buddy.tla` proves no double-allocation under concurrent alloc/free across multiple CPUs and zones.
- REQ-15: DELAY_FREE_ONE_PAGE (page quarantine on free) default-on per `00-security-principles.md` knob inventory; sysctl `vm.delay_free_pages = 1`.
- REQ-16: MEMORY_SANITIZE for page allocator (zero on free) default-on per `00-security-principles.md`; boot param `init_on_free=1` AND sysctl `vm.zero_on_free=1`.
- REQ-17: Hardening section per `00-security-principles.md` template.
- REQ-18: Layer-4 functional-correctness opt-in for buddy split/coalesce arithmetic via Creusot per `mm/00-overview.md` Layer-4 candidates list.

## Acceptance Criteria

- [ ] AC-1: A test allocates pages with each documented GFP flag combination and asserts: sleepability via `might_sleep` instrumentation; zone selection via PFN range; zeroing via memcmp to zero buffer; retry behavior via fault-injection. Output matches upstream within ±1 retry on `__GFP_RETRY_MAYFAIL`. (covers REQ-1, REQ-2)
- [ ] AC-2: `/proc/buddyinfo` output diff between Rookery and upstream after identical workload is empty. Same for `/proc/pagetypeinfo`, `/proc/zoneinfo`. (covers REQ-3)
- [ ] AC-3: `pahole struct page` produces byte-identical first-cache-line layout vs. upstream. `pahole struct folio` produces byte-identical first-cache-line layout. (covers REQ-4, REQ-5)
- [ ] AC-4: `tools/vm/page-types` (upstream tool) runs unmodified against Rookery and produces identical flag breakdowns. (covers REQ-4)
- [ ] AC-5: `tools/vm/page_owner_sort` runs unmodified against Rookery's `/sys/kernel/page_owner/page_owner_full` output. (covers REQ-11)
- [ ] AC-6: `/proc/<pid>/pagemap` raw bytes for a curated test program match upstream byte-for-byte. (covers REQ-6)
- [ ] AC-7: A 16-CPU concurrent-alloc benchmark (`fio + alloc-bench`) shows comparable throughput (within ±5%) and zero buddy-list corruption under stress. (covers REQ-7, REQ-13, REQ-14)
- [ ] AC-8: NUMA-aware allocation test on multi-socket hardware: allocations from CPU 0 prefer node 0 per upstream's `local_node` policy; same for CPU N. `numastat` output matches upstream. (covers REQ-8)
- [ ] AC-9: Memory-offline test (`echo 0 > /sys/devices/system/memory/memoryN/online`) succeeds; affected pages migrate per upstream. (covers REQ-9)
- [ ] AC-10: `page_reporting` test against an emulated hypervisor consumer (e.g., virtio-balloon free-page-reporting) hands off pages identically. (covers REQ-10)
- [ ] AC-11: `cat /sys/kernel/mm/page_idle/bitmap` round-trips correctly. (covers REQ-12)
- [ ] AC-12: `make verify` runs `kani::proofs::mm::page_alloc::*` and reports zero failures. (covers REQ-13)
- [ ] AC-13: `make tla` runs `models/mm/buddy.tla` (TLC) and reports invariants hold. (covers REQ-14)
- [ ] AC-14: `vm.delay_free_pages=1` default verified by `sysctl -a | grep delay_free_pages`. (covers REQ-15)
- [ ] AC-15: `init_on_free=1` boot-param + `vm.zero_on_free=1` sysctl default verified by post-boot dump. A freshly-allocated then immediately-freed page contains zero bytes when re-allocated (test). (covers REQ-16)
- [ ] AC-16: Hardening section present and follows template. (covers REQ-17)
- [ ] AC-17: Layer-4 Creusot proof of `__buddy_split` + `__buddy_coalesce` arithmetic compiles and verifies. (covers REQ-18)

## Architecture

### Rust module organization

- `kernel::mm::page_alloc::buddy` — per-zone free-list management
- `kernel::mm::page_alloc::pcp` — per-CPU pageset fast-path
- `kernel::mm::page_alloc::zone` — zone selection per GFP
- `kernel::mm::page_alloc::isolation` — memory-hotplug + CMA isolation
- `kernel::mm::page_alloc::reporting` — page reporting to hypervisor
- `kernel::mm::page_alloc::owner` — page_owner debug
- `kernel::mm::page_alloc::idle` — page_idle tracking
- `kernel::mm::page_alloc::poison` — page poison debug
- `kernel::mm::page::Page` — `struct page` Rust binding
- `kernel::mm::folio::Folio` — `struct folio` Rust binding
- `kernel::mm::gfp::Gfp` — typed GFP flag set (replaces raw `gfp_t`)

### Key data structures

- `Page` — `#[repr(C)]` mirroring upstream `struct page`. First-cache-line fields: `flags`, `mapping`, `index`, `_refcount`, `_mapcount`, `private`, optional `compound_head` for tail pages
- `Folio` — `#[repr(transparent)]` over `Page`; first cache line: same as Page; subsequent fields per `struct folio`
- `Zone` — per-NUMA-node + per-zone state; per-migrate-type free lists at orders 0..MAX_ORDER (10 by default)
- `PerCpuPageset` — small lock-free cache per CPU per zone
- `Gfp` — newtype `u32`; const builders `Gfp::KERNEL()`, `Gfp::ATOMIC()`, etc.; method `with_zone_hint(z)` etc.

### Locking and concurrency

- **Per-CPU pageset (fast path)**: lock-free, preempt-disabled region only. Reads + writes use atomic counters; refill/drain holds zone->lock.
- **Per-zone `zone->lock`**: raw_spinlock. Held during slow-path alloc/free, list manipulation, watermark checks.
- **Per-cpuset `cs->mems_allowed`**: cgroup-side; consulted under RCU read.
- **Page isolation**: `isolate_page` uses page-level atomic flag + per-zone lock. PageOffline / PageHWPoison are atomic.
- **Page reporting**: workqueue-driven; protected by `pgdat->reporting_active` flag.

TLA+ model `models/mm/buddy.tla` proves the protocol under concurrent multi-CPU alloc/free with watermark crossing.

### Error handling

- `Err(ENOMEM)` — out of memory after exhausting fallbacks (zone fallback, slowpath, OOM kill if not `__GFP_NORETRY`)
- `Err(EAGAIN)` — page-isolation retry needed (compaction in progress)
- `Err(EBUSY)` — page hotplug-offline in progress

`__GFP_NOFAIL` allocations panic via `kernel::panic!` if they cannot be satisfied (matches upstream's `BUG_ON` behavior).

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Per-CPU pageset push/pop (lock-free) | `kani::proofs::mm::page_alloc::pcp_safety` |
| Buddy split (slot-pointer arithmetic) | `kani::proofs::mm::page_alloc::buddy_split_safety` |
| Buddy coalesce (find buddy, merge) | `kani::proofs::mm::page_alloc::buddy_coalesce_safety` |
| Folio split / compound | `kani::proofs::mm::folio::split_safety` (cross-ref `mm/00-overview.md` Layer 1) |
| Page-flag atomic ops | `kani::proofs::mm::page::flag_safety` |
| Page reporting hypercall | `kani::proofs::mm::page_alloc::report_safety` |

### Layer 2: TLA+ models

- `models/mm/buddy.tla` (mandatory per `mm/00-overview.md` Layer 2) — proves buddy invariants under concurrent alloc/free across CPUs + zones. Safety: no double-allocation, no free-list corruption.

### Layer 3: Kani harnesses for data-structure invariants (MANDATORY)

Per `mm/00-overview.md` REQ-14 / `00-overview.md` D4 — buddy is one of the four mandatory Layer-3 areas:

| Data structure | Invariant | Harness |
|---|---|---|
| Buddy free lists (per zone, per migrate-type, per order) | "No physical PFN appears in two free lists simultaneously" + "Every free entry's order matches the list it lives in" | `kani::proofs::mm::page_alloc::buddy_disjoint_invariants` |
| Per-CPU pageset | "Sum of pageset sizes + zone free-list sizes ≤ zone total available memory" | `kani::proofs::mm::page_alloc::pcp_count_invariants` |
| Page isolation list | "Every isolated page has PageOffline set + appears on the isolated list exactly once" | `kani::proofs::mm::page_alloc::isolation_invariants` |

### Layer 4: Functional correctness (opt-in; declared in `mm/00-overview.md`)

- **Buddy split/coalesce arithmetic** via Creusot — proves: for any input order N + buddy at PFN P, the split produces buddies at P and P + 2^(N-1), both with order N-1. Tractable; high-leverage.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **MEMORY_SANITIZE** (zero on free) | `boot param init_on_free=1` + `sysctl vm.zero_on_free=1` zero pages on free; flipped from upstream default-off | § Default-on configurable off |
| **DELAY_FREE_ONE_PAGE** | `sysctl vm.delay_free_pages=1`; freed pages quarantined briefly via per-zone delay queue before returning to free list; defeats UAF spray races | § Default-on configurable off |
| **AUTOSLAB** (per-cache type tagging) | Cross-ref `mm/slab.md`; this component allocates the underlying pages | § Mandatory |

### Row-1 features consumed by this component

- **KERNEXEC**: page allocator's text + rodata are RX/RO; runtime self-modification forbidden
- **REFCOUNT**: `struct page->_refcount` is a `Refcount` (saturating; defeats overflow-to-zero attacks)
- **SIZE_OVERFLOW**: PFN/order arithmetic uses checked operators per the lint
- **CONSTIFY**: zone-watermark thresholds, per-order constants are static const

### Row-2 / GR-RBAC integration

The page allocator runs below LSM hooks. memcg charging (cross-ref `mm/memcg.md`) is the only LSM-relevant interaction; that lives in the memcg component, not here.

### Userspace-visible behavior changes

Per Axiom 4 of `00-security-principles.md`:
- **`vm.zero_on_free=1` default**: allocator returns zero-filled memory more often than upstream's default (where the page may contain previous user's data until first write). Performance: ~5–10% allocation overhead. Configurable off via sysctl.
- **`vm.delay_free_pages=1` default**: freed pages enter a brief quarantine queue. Memory overhead: a few percent. Configurable off via sysctl.

Both are documented in the user manual; system administrators can disable for performance-critical workloads.

### Verification

(See § Verification above.)

## Open Questions

(none — buddy allocator semantics are well-defined; no architectural ambiguities at this tier)

## Out of Scope

- 32-bit-only paths (`mm/highmem.c`)
- Slab allocator (cross-ref `mm/slab.md`)
- Per-task mm/VMA (`mm/virtual-memory.md`)
- Implementation code
