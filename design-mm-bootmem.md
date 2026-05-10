---
title: "Tier-3: mm/memblock.c — Boot-time memory allocator (memblock + sparsemem init)"
tags: ["tier-3", "mm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

memblock is the **boot-time** memory allocator and physical-RAM map: at early boot (before buddy/SLUB are up), `memblock` arrays track {memory regions (RAM ranges), reserved regions (kernel image, initrd, fdt, etc.)}. Per-arch boot fills via `memblock_add(start, size)` from BIOS-e820/DT-`memory` nodes; per-`memblock_reserve` carves reservations. Per-`memblock_alloc`/-`alloc_low_*` returns aligned virtual addrs (with the side effect of marking reserved). At `mem_init`, memblock-RAM minus memblock-reserved is given to buddy via `__free_pages_memory`. Critical for: early-boot init, NUMA-node-detection, hot-plugging-aware boot.

This Tier-3 covers `memblock.c` (~2908 lines).

### Acceptance Criteria

- [ ] AC-1: Boot: e820/DT parsed; memblock.memory populated.
- [ ] AC-2: memblock_reserve(kernel-text-range): in memblock.reserved.
- [ ] AC-3: memblock_alloc(4KB): returns aligned virt-addr; range reserved.
- [ ] AC-4: memblock_alloc with NUMA nid: alloc from nid-region.
- [ ] AC-5: mem_init: per-free-range freed to buddy.
- [ ] AC-6: ZONE_MOVABLE: per-MEMBLOCK_HOTPLUG regions placed.
- [ ] AC-7: /proc/meminfo MemTotal: from memblock.memory total minus reserved.
- [ ] AC-8: memblock_for_each_region: walks all memory regions.
- [ ] AC-9: memtest: per-pattern read/write success.
- [ ] AC-10: Per-CONFIG_ARCH_KEEP_MEMBLOCK off: memblock discarded post-boot.

### Architecture

Per-system memblock:

```
struct Memblock {
  bottom_up: bool,                                // alloc-direction
  current_limit: PhysAddrT,                       // limit
  memory: MemblockType,
  reserved: MemblockType,
  #[cfg(CONFIG_ARCH_KEEP_MEMBLOCK)]
  physmem: MemblockType,
}

struct MemblockType {
  cnt: u64,                                       // # regions
  max: u64,                                       // alloc'd capacity
  total_size: u64,
  regions: *MemblockRegion,                       // sorted by base
  name: &'static str,
}

struct MemblockRegion {
  base: PhysAddrT,
  size: PhysAddrT,
  flags: MemblockFlags,                           // MEMBLOCK_*
  nid: i32,
}

const INIT_MEMBLOCK_REGIONS: usize = 128;
const INIT_PHYSMEM_REGIONS: usize = 4;
const MEMBLOCK_ALLOC_ANYWHERE: PhysAddrT = !0;
```

`Memblock::add(base, size) -> Result<()>`:
1. nid = MAX_NUMNODES; flags = MEMBLOCK_NONE.
2. err = Memblock::add_range(&memblock.memory, base, size, nid, flags).

`Memblock::add_range(type, base, size, nid, flags) -> Result<()>`:
1. /* Find overlapping + adjacent regions */
2. for existing in type.regions:
   - if overlap(existing, base, size): merge.
3. /* If no merge: insert in sorted-order */
4. memblock_insert_region(type, idx, base, size, nid, flags).
5. memblock_merge_regions(type).

`Memblock::reserve(base, size) -> Result<()>`:
1. err = Memblock::add_range(&memblock.reserved, base, size, MAX_NUMNODES, MEMBLOCK_NONE).

`Memblock::alloc(size, align) -> *mut u8`:
1. /* Per-default high-mem alloc */
2. ret = Memblock::__alloc_range_nid(size, align, 0, MEMBLOCK_ALLOC_ANYWHERE, NUMA_NO_NODE, false).
3. return ret.

`Memblock::__alloc_range_nid(size, align, low, high, nid, exact_nid) -> *mut u8`:
1. /* Iterate memory ∖ reserved within [low, high) */
2. for free_region in for_each_free_mem_range(low, high, nid, MEMBLOCK_NONE):
   - if !exact_nid ∨ free_region.nid == nid:
     - candidate = ALIGN(free_region.start, align).
     - if candidate + size ≤ free_region.end:
       - Memblock::reserve(candidate, size).
       - return phys_to_virt(candidate).
3. return NULL.

`Memblock::__free_pages_memory(start_pfn, end_pfn)`:
1. /* Per-page: free to buddy */
2. for pfn in start_pfn..end_pfn:
   - __free_pages_core(pfn_to_page(pfn), 0).

`Memblock::set_node(base, size, type, nid) -> Result<()>`:
1. /* Walk type.regions; assign nid to overlapping */
2. for region in type.regions:
   - if region in [base, base+size): region.nid = nid.

### Out of Scope

- mm/00-overview (Tier-2)
- mm/sparse.c (covered separately if expanded)
- mm/bootmem_info.c (covered separately)
- Per-arch e820 / DT parsing (covered in `arch/*` separately)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct memblock` | per-system boot-mem state | `Memblock` |
| `struct memblock_type` | per-(memory or reserved) array | `MemblockType` |
| `struct memblock_region` | per-(base, size, flags, nid) | `MemblockRegion` |
| `memblock_add()` | per-(start, size) add to memory | `Memblock::add` |
| `memblock_remove()` | per-(start, size) remove | `Memblock::remove` |
| `memblock_reserve()` | per-(start, size) reserve | `Memblock::reserve` |
| `memblock_free()` | per-(start, size) unreserve | `Memblock::free` |
| `memblock_alloc()` / `memblock_alloc_node()` | per-(size, align, nid) alloc | `Memblock::alloc` |
| `memblock_alloc_low()` / `memblock_alloc_try_nid()` | per-(low-mem / try-node) alloc | `Memblock::alloc_low` / `try_nid` |
| `memblock_phys_alloc()` | per-phys-addr alloc | `Memblock::phys_alloc` |
| `__memblock_alloc_range_nid()` | per-arch alloc helper | `Memblock::__alloc_range_nid` |
| `memblock_set_node()` | per-region NUMA-node assign | `Memblock::set_node` |
| `memblock_for_each_region` | per-region iter | macro |
| `for_each_free_mem_range()` | per-free-range iter | macro |
| `memblock_dump_all()` | per-debug-output | `Memblock::dump_all` |
| `memblock_phys_free()` | per-discard reserved range | `Memblock::phys_free` |
| `memblock_clear_hotplug()` | per-flag clear | `Memblock::clear_hotplug` |
| `__free_pages_memory()` | per-buddy-handoff | `Memblock::__free_pages_memory` |

### compatibility contract

REQ-1: Per-system memblock:
- memory: array of available RAM ranges.
- reserved: array of carved-out ranges (kernel, initrd, etc.).
- physmem (optional): per-CONFIG_ARCH_KEEP_MEMBLOCK: per-system physical RAM map.

REQ-2: Per-region:
- base: u64 phys-addr.
- size: u64 bytes.
- flags: MEMBLOCK_* (NONE / HOTPLUG / MIRROR / NOMAP / DRIVER_MANAGED).
- nid: per-NUMA node ID.

REQ-3: memblock_add(start, size):
- /* Lock */ memblock_lock.
- /* Merge adjacent + overlapping into memory array */
- For each existing-region intersecting: merge.
- Else: insert sorted.

REQ-4: memblock_reserve(start, size):
- Symmetric for reserved array.
- Per-overlap: split + merge.

REQ-5: memblock_alloc(size, align):
- Walk memory array; find free range (memory ∖ reserved) with sufficient size.
- Per-found: memblock_reserve(addr, size); return phys_to_virt(addr).

REQ-6: memblock_alloc_node(size, align, nid):
- Same as above, but restrict to nid-flagged regions.

REQ-7: Per-arch boot fill:
- e820_to_memblock (x86): per-e820-entry → memblock_add.
- DT-fdt_scan_memory (ARM): per-/memory/reg → memblock_add.

REQ-8: Per-buddy-handoff (mem_init):
- For each free-range (memory ∖ reserved):
  - __free_pages_memory(start_pfn, end_pfn).
  - per-page enters buddy free-list.

REQ-9: Per-NUMA:
- ARM/RISC-V DT: per-/numa-node-id assigned.
- x86 NUMA: per-ACPI-SRAT memblock_set_node.

REQ-10: Per-MEMBLOCK_HOTPLUG flag:
- Per-hotpluggable-RAM marked.
- Per-buddy excludes from ZONE_NORMAL; placed in ZONE_MOVABLE.

REQ-11: Per-MEMBLOCK_MIRROR:
- Per-mirrored (RAS) RAM.
- Per-buddy: critical pages go into mirrored; non-critical into non-mirrored.

REQ-12: Per-memtest:
- /init/main.c → mem_test if CONFIG_MEMTEST.
- Per-pattern read/write test before mem_init.

REQ-13: Per-discard:
- Post-boot: CONFIG_ARCH_KEEP_MEMBLOCK off: memblock arrays freed.
- Per-discard: memblock_discard.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `regions_sorted_by_base` | INVARIANT | per-type.regions[i].base < type.regions[i+1].base. |
| `no_overlap_in_type` | INVARIANT | per-type: no two regions overlap (post-merge). |
| `cnt_le_max` | INVARIANT | type.cnt ≤ type.max. |
| `nid_in_range` | INVARIANT | per-region.nid ∈ {NUMA_NO_NODE, 0..MAX_NUMNODES}. |
| `alloc_within_memory_minus_reserved` | INVARIANT | per-alloc-result ∈ memory ∖ reserved. |

### Layer 2: TLA+

`mm/memblock.tla`:
- Per-boot add/reserve/alloc + per-handoff to buddy.
- Properties:
  - `safety_no_double_reserve` — per-(addr, size) reserved at most once.
  - `safety_alloc_within_free` — per-alloc: chosen addr in memory minus reserved.
  - `liveness_buddy_eventually_receives` — per-mem_init: free-range → __free_pages_memory.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Memblock::add_range` post: regions sorted; merged adjacents | `Memblock::add_range` |
| `Memblock::alloc` post: returned addr aligned + reserved | `Memblock::alloc` |
| `Memblock::reserve` post: range in reserved type | `Memblock::reserve` |
| `Memblock::__free_pages_memory` post: per-page in buddy free-list | `Memblock::__free_pages_memory` |

### Layer 4: Verus/Creusot functional

`Per-boot physical-RAM map → memblock arrays → per-arch alloc → handoff free pages to buddy at mem_init` semantic equivalence: per-Linux mm bootmem-to-buddy handoff model.

### hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening.)

memblock-specific reinforcement:

- **Per-region merge prevents fragmentation** — defense against per-add unbounded count.
- **Per-cnt ≤ max + array grow** — defense against per-overflow.
- **Per-alloc bounded by current_limit** — defense against per-alloc beyond defined RAM.
- **Per-flag MEMBLOCK_NOMAP for reserved-but-uncached** — defense against per-driver mapping.
- **Per-arch keep memblock optional** — defense against per-runtime-mem-overhead.
- **Per-NUMA assignment validated** — defense against per-region misattribution.
- **Per-buddy handoff iterates free-only** — defense against per-handoff double-free.
- **Per-discard early-mem after boot** — defense against per-overhead leaving early-mem reserved.
- **Per-physmem (optional) tracked separately** — defense against per-mem-survey corruption.
- **Per-memtest gated by CONFIG_MEMTEST** — defense against per-prod runtime cost.

