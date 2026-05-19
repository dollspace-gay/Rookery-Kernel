# Tier-3: mm/memory_hotplug.c — Memory hotplug (per-block add/remove + online/offline)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: mm/00-overview.md
upstream-paths:
  - mm/memory_hotplug.c (~2432 lines)
  - mm/sparse.c
  - drivers/base/memory.c (sysfs glue)
  - include/linux/memory_hotplug.h
-->

## Summary

Memory hotplug supports per-section add/remove + online/offline of memory regions: per-VM-balloon (KVM/Xen), per-NUMA-node hotswap, per-physical DIMM. Per-block (default 128MB on x86_64) corresponds to a sysfs entry `/sys/devices/system/memory/memoryN/state`. Per-add: `add_memory_resource` registers per-section structures (sparsemem) + creates struct page array (vmemmap). Per-online: `online_pages` per-block transitions to ONLINE; per-pages rejoin buddy + freelists. Per-offline: `offline_pages` migrates movable + isolates per-pageblock. Per-remove: free vmemmap; release resource. Critical for: cloud-VM resize, hot-DIMM, memory-sparing.

This Tier-3 covers `memory_hotplug.c` (~2432 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `add_memory()` | per-(start, size) add | `MemHotplug::add_memory` |
| `add_memory_resource()` | per-resource add backend | `MemHotplug::add_memory_resource` |
| `try_remove_memory()` | per-(start, size) remove | `MemHotplug::try_remove_memory` |
| `online_pages()` | per-block online | `MemHotplug::online_pages` |
| `offline_pages()` | per-block offline | `MemHotplug::offline_pages` |
| `arch_add_memory()` | per-arch hook | `MemHotplug::arch_add_memory` |
| `arch_remove_memory()` | per-arch hook | `MemHotplug::arch_remove_memory` |
| `__add_section()` | per-section sparsemem populate | `MemHotplug::__add_section` |
| `online_pages_range()` | per-pfn-range online | `MemHotplug::online_pages_range` |
| `init_currently_empty_zone()` | per-zone init | `MemHotplug::init_currently_empty_zone` |
| `memory_block_online()` / `_offline()` | per-block sysfs callback | `MemHotplug::memory_block_*` |
| `memmap_init_zone_device()` | per-vmemmap init | `MemHotplug::memmap_init_zone_device` |
| `register_memory_block_under_node()` | per-NUMA tagging | `MemHotplug::register_under_node` |
| `MHP_*` flags | per-add config | UAPI |
| `MEM_GOING_ONLINE` / `MEM_ONLINE` / `MEM_GOING_OFFLINE` / `MEM_OFFLINE` / `MEM_CANCEL_*` | per-event | shared |

## Compatibility contract

REQ-1: Per-block size:
- memory_block_size_bytes(): per-arch (typically 128MB on x86_64 small / 2GB large).
- Block-size aligned add/remove.

REQ-2: add_memory(nid, start, size, mhp_flags):
- Allocate struct resource, register kernel-resource.
- arch_add_memory: per-arch page-table population.
- __add_section: per-section sparsemem populate (struct page allocation).
- For each block: register via memory_block.

REQ-3: arch_add_memory:
- Map physical-RAM into kernel-direct-map (linear-mapping).
- Allocate vmemmap (struct page metadata).
- per-x86: pud_present + pmd_present + populate.

REQ-4: online_pages(pfn, nr_pages, group):
- Per-pfn-range: validate online-state.
- arch_zone-init populates zone if first online.
- For each pageblock: set_pageblock_migratetype(MIGRATE_MOVABLE or MIGRATE_UNMOVABLE).
- online_pages_range adds pages to buddy.
- mem_hotplug_notifier(MEM_GOING_ONLINE → MEM_ONLINE).

REQ-5: online_pages_range:
- Per-PFN: free_unref_page; freed-pages += 1.
- zone counters updated; node_present_pages += nr_pages.

REQ-6: offline_pages(pfn, nr_pages, group):
- isolate-pageblocks (MIGRATE_ISOLATE).
- migrate any non-movable pages.
- if any pages remain unmigrate-able: -EBUSY.
- mem_hotplug_notifier(MEM_GOING_OFFLINE → MEM_OFFLINE).

REQ-7: try_remove_memory:
- Validate all blocks in range OFFLINE.
- arch_remove_memory: free vmemmap; remove direct-map.
- __remove_section per-sparsemem.
- release_resource.

REQ-8: Per-userspace ABI:
- /sys/devices/system/memory/memoryN/state: read "online"/"offline".
- write "online"/"offline" to trigger transition.
- /sys/devices/system/memory/memoryN/removable: 1 if all-pages movable.
- /sys/devices/system/memory/memoryN/phys_device: phys-addr.
- /sys/devices/system/memory/probe: write phys-addr to add manually.

REQ-9: Per-NUMA-node:
- register_memory_block_under_node: per-block bound to node.
- Node-Y add: hotadd_node(Y) creates pgdat.

REQ-10: Per-balloon-driver (xen-balloon, virtio-balloon):
- pre-allocates "movable" memory for guest-balloon.
- Per-page MIGRATE_BALLOON migrate-type.

REQ-11: Per-kernel-cmdline:
- `movable_node`: per-NUMA node configurable as ZONE_MOVABLE.
- `memmap=size@start`: reserve physical-region.

REQ-12: Per-MHP_* flags:
- MHP_NONE: default.
- MHP_NID_IS_MGID: nid is memory-group ID.
- MHP_MEMMAP_ON_MEMORY: vmemmap stored in hot-added memory itself.

## Acceptance Criteria

- [ ] AC-1: add_memory(0, 0x100000000, 128MB): per-section populated; struct page allocated.
- [ ] AC-2: write "online" to memoryN/state: pages join buddy.
- [ ] AC-3: write "offline" to memoryN/state: pages migrated; per-block off.
- [ ] AC-4: try_remove_memory after offline: arch_remove_memory cleans up.
- [ ] AC-5: Per-non-movable allocation in offline-target: -EBUSY.
- [ ] AC-6: Per-NUMA hotadd: new pgdat created.
- [ ] AC-7: MHP_MEMMAP_ON_MEMORY: vmemmap consumed from hot-added range.
- [ ] AC-8: Per-MEM_ONLINE notifier callback: subscribers run.
- [ ] AC-9: /sys/devices/system/memory/probe write: triggers add_memory.
- [ ] AC-10: Hot-remove during heavy alloc: -EBUSY; user retries.

## Architecture

Per-block state (in drivers/base/memory.c):

```
struct MemoryBlock {
  start_section_nr: u64,
  end_section_nr: u64,
  state: i32,                                    // MEM_ONLINE / OFFLINE / GOING_*
  online_type: i32,                              // MMOP_OFFLINE / ONLINE / ONLINE_KERNEL / ONLINE_MOVABLE
  nid: i32,
  group: *MemoryGroup,
  altmap: *VmemAltmap,
}
```

`MemHotplug::add_memory(nid, start, size, mhp_flags) -> Result<()>`:
1. res = register_memory_resource(start, size, "System RAM").
2. err = MemHotplug::add_memory_resource(nid, res, mhp_flags).
3. /* Per-section populate; per-block create */.

`MemHotplug::add_memory_resource(nid, res, mhp_flags) -> Result<()>`:
1. /* Validate aligned to memory_block_size. */
2. start_pfn = PHYS_PFN(res.start).
3. nr_pages = PHYS_PFN(resource_size(res)).
4. arch_add_memory(nid, res.start, resource_size(res), &params).
5. for_each_section in [start_pfn, end_pfn):
   - __add_section(nid, pfn).
6. for_each_block: create_memory_block(start_section_nr, end_section_nr, nid).

`MemHotplug::online_pages(pfn, nr_pages, group) -> Result<()>`:
1. mem_hotplug_lock.
2. zone = zone_for_pfn_range(MMOP_ONLINE, nid, group, pfn, nr_pages).
3. /* Init zone if first online */
4. init_currently_empty_zone(zone, pfn, nr_pages).
5. mem_hotplug_notifier(MEM_GOING_ONLINE).
6. /* Set per-pageblock migrate-type */
7. for pageblock_pfn in [pfn, end):
   - set_pageblock_migratetype(MIGRATE_MOVABLE).
8. online_pages_range(pfn, nr_pages).
9. mem_hotplug_notifier(MEM_ONLINE).
10. mem_hotplug_unlock.

`MemHotplug::online_pages_range(start_pfn, nr_pages)`:
1. for pfn in [start_pfn, start_pfn+nr_pages):
   - free_unref_page(pfn_to_page(pfn), 0).
2. zone.present_pages += nr_pages.
3. node.present_pages += nr_pages.

`MemHotplug::offline_pages(pfn, nr_pages, group) -> Result<()>`:
1. mem_hotplug_lock.
2. mem_hotplug_notifier(MEM_GOING_OFFLINE).
3. /* Isolate pageblocks */
4. for pageblock_pfn in [pfn, end):
   - start_isolate_page_range(pageblock_pfn, pageblock_pfn+pageblock_nr_pages, MIGRATE_MOVABLE).
5. /* Migrate movable pages */
6. for pfn in range:
   - if page_movable(pfn): migrate_pages_to_target(pfn, ...).
7. /* Verify all-non-movable migrated */
8. if pages_remain: rollback isolation; return -EBUSY.
9. /* Deduct from zone counters */
10. zone.present_pages -= nr_pages.
11. mem_hotplug_notifier(MEM_OFFLINE).

`MemHotplug::try_remove_memory(start, size) -> Result<()>`:
1. /* All blocks in range MUST be OFFLINE. */
2. for block in range: assert(block.state == MEM_OFFLINE).
3. arch_remove_memory(start, size, ...).
4. for_each_section: __remove_section.
5. release_mem_region.

`MemHotplug::memory_block_online(mem) -> Result<()>` (per-block sysfs callback):
1. nr_pages = PAGES_PER_SECTION × (mem.end_section_nr - mem.start_section_nr + 1).
2. err = online_pages(start_pfn, nr_pages, mem.group).
3. mem.state = MEM_ONLINE.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `block_size_aligned` | INVARIANT | per-add/remove: range aligned to memory_block_size. |
| `online_after_add` | INVARIANT | online_pages requires sparse-section populated. |
| `offline_requires_movable_or_isolate` | INVARIANT | per-offline: all pages migrated or isolated. |
| `state_transitions_valid` | INVARIANT | mem.state transitions: OFFLINE → GOING_ONLINE → ONLINE → GOING_OFFLINE → OFFLINE. |
| `notifier_chain_per_event` | INVARIANT | per-MEM_GOING_*/MEM_*: notifier chain invoked. |

### Layer 2: TLA+

`mm/memory_hotplug.tla`:
- Per-block state-machine + per-online buddy-add + per-offline migrate.
- Properties:
  - `safety_no_double_online` — per-block ONLINE ⟹ no second ONLINE.
  - `safety_no_remove_when_in_use` — per-pages allocated → cannot offline.
  - `liveness_offline_eventually_completes` — per-all-movable blocks: offline succeeds.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `MemHotplug::add_memory` post: per-section populated; arch hooks invoked | `MemHotplug::add_memory` |
| `MemHotplug::online_pages` post: per-pageblock MIGRATE_MOVABLE; zone counters updated | `MemHotplug::online_pages` |
| `MemHotplug::offline_pages` post: per-pageblock MIGRATE_ISOLATE; pages migrated | `MemHotplug::offline_pages` |
| `MemHotplug::try_remove_memory` post: arch_remove_memory invoked; resource released | `MemHotplug::try_remove_memory` |

### Layer 4: Verus/Creusot functional

`Per-block add → online → use → offline → remove lifecycle; per-pages buddy-managed when ONLINE` semantic equivalence: per-Documentation/admin-guide/mm/memory-hotplug.rst.

## Hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening.)

Memory-hotplug reinforcement:

- **Per-block-size alignment enforced** — defense against per-config invalid range.
- **Per-mem_hotplug_lock for state transitions** — defense against per-state torn-update.
- **Per-pageblock isolation pre-offline** — defense against per-alloc race-with-offline.
- **Per-non-movable migration verified** — defense against per-offline silent data-loss.
- **Per-arch_add/remove hooks atomic** — defense against per-direct-map torn.
- **Per-vmemmap allocation tracked** — defense against per-vmemmap leak.
- **Per-NUMA-node creation gated** — defense against per-pgdat resource exhaustion.
- **Per-MHP_MEMMAP_ON_MEMORY size-bounded** — defense against per-vmemmap consuming all hot-added memory.
- **Per-MEM_*-event notifier failure handling** — defense against per-subscriber-fail-leaving-inconsistent-state.
- **Per-/sysfs-write CAP_SYS_ADMIN** — defense against per-unprivileged hotplug-trigger.

## Grsecurity/PaX-style Reinforcement

Baseline grsec/PaX features inherited project-wide:

- **PAX_USERCOPY** — hot-added page descriptors (struct page) live in vmemmap; whitelist invariants of slabs unchanged by hotplug.
- **PAX_KERNEXEC** — arch_add_memory direct-map updates honor W^X; new linear-map PTEs default to NX, kernel text remains read-only.
- **PAX_RANDKSTACK** — hotplug notifier and add_memory_resource entry stack randomized.
- **PAX_REFCOUNT** — zone present/spanned/managed page counters saturate; under/overflow during online/offline traps before zone metadata corruption.
- **PAX_MEMORY_SANITIZE** — vmemmap pages associated with offlined ranges zero-poisoned on release; no stale struct-page metadata leaks across hotplug cycles.
- **PAX_UDEREF** — sysfs hot-add inputs (start, size, online_type) range-validated against installable resource limits.
- **PAX_RAP / kCFI** — memory_notifier callbacks type-tagged; subsystem-registered hotplug listeners verified at registration.
- **GRKERNSEC_HIDESYM** — `add_memory`, `offline_pages`, `online_pages`, `arch_add_memory` absent from `/proc/kallsyms` for unpriv.
- **GRKERNSEC_DMESG** — hotplug add/online/offline trace lines (with physical addresses and node IDs) redacted from dmesg for non-CAP_SYSLOG.

Memory-hotplug-specific reinforcements:

- **CAP_SYS_ADMIN strict on every entry point** — sysfs writes to `/sys/devices/system/memory/memoryN/state`, ACPI hotplug events surfaced to userspace, and add_memory_driver_managed paths all require CAP_SYS_ADMIN; no per-namespace bypass.
- **Zone-online validation** — online_pages refuses ZONE_MOVABLE designation if the range overlaps unmovable allocations; pageblock migrate-type set deterministically before zone span update.
- **Pageblock isolation pre-offline** — every pageblock in the offline range placed in MIGRATE_ISOLATE before scan; concurrent allocators cannot grab pages from the offlining range.
- **Non-movable migration verified** — offline path refuses to complete if any unmovable allocation remains; no silent data-loss via forced-free of pinned pages.
- **MHP_MEMMAP_ON_MEMORY size-bounded** — vmemmap-on-memory mode caps the vmemmap fraction so a hotplug-add cannot consume itself with vmemmap overhead, preventing self-DoS.

Rationale: memory hotplug rewrites the kernel's view of physical memory at runtime — direct map, vmemmap, zone spans, and per-node pgdat all mutate. A refcount underflow or a pageblock isolation gap here can either (a) leak a freed range back into buddy with stale struct-page state, or (b) let a malicious userspace trigger zone-bookkeeping corruption via repeated online/offline. Grsec emphasis: CAP_SYS_ADMIN strict gating on every entry, saturating zone counters under PAX_REFCOUNT, and HIDESYM/DMESG redaction on physical-address disclosure so an unprivileged observer cannot use hotplug traces to map kernel layout.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- mm/00-overview (Tier-2)
- Sparsemem (covered in `mm-sparsemem.md` if added)
- vmemmap optimization (covered separately if expanded)
- Per-arch hooks (covered in `arch/x86/00-overview.md` Tier-3)
- xen-balloon / virtio-balloon (covered separately)
- Implementation code
