# Tier-3: mm/percpu.c — Per-CPU memory allocator

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: mm/00-overview.md
upstream-paths:
  - mm/percpu.c (~3388 lines)
  - mm/percpu-vm.c
  - mm/percpu-km.c
  - include/linux/percpu.h
  - include/asm-generic/percpu.h
-->

## Summary

Per-CPU allocator provides **per-CPU dynamic memory** via small-object packing across N replicas (one per online CPU). Per-`alloc_percpu(size)` returns offset; per-CPU-pointer dereference via per-arch macro (e.g. `this_cpu_ptr(ptr)`) yields current-CPU's replica. Per-chunk = N pages × NR_CPUS contiguous virtual addr; chunk's free-bitmap tracks per-byte allocation. Per-NUMA-aware via `pcpu_alloc_fc_*`. Fast-path uses RCU + per-chunk-cache; slow-path takes `pcpu_alloc_mutex`. Critical for: per-CPU statistics, per-CPU caches, RCU primitives (rcu_data), tracepoints, percpu_counter, kernel stats.

This Tier-3 covers `percpu.c` (~3388 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct pcpu_chunk` | per-(N-pages × NR_CPUS) chunk | `PcpuChunk` |
| `struct pcpu_block_md` | per-block metadata bitmap | `PcpuBlockMd` |
| `pcpu_alloc()` | per-(size, align) alloc | `Pcpu::alloc` |
| `__alloc_percpu()` / `alloc_percpu()` | wrapper | `Pcpu::alloc_percpu` |
| `__alloc_percpu_gfp()` | with GFP flags | `Pcpu::alloc_percpu_gfp` |
| `__alloc_reserved_percpu()` | from reserved-chunk | `Pcpu::alloc_reserved` |
| `free_percpu()` | per-ptr free | `Pcpu::free_percpu` |
| `is_kernel_percpu_address()` | per-addr check | `Pcpu::is_kernel_percpu_address` |
| `pcpu_setup_first_chunk()` | per-arch boot init | `Pcpu::setup_first_chunk` |
| `pcpu_create_chunk()` | per-(GFP) chunk-alloc | `Pcpu::create_chunk` |
| `pcpu_destroy_chunk()` | per-chunk free | `Pcpu::destroy_chunk` |
| `pcpu_first_chunk` | per-system root chunk | shared |
| `pcpu_reserved_chunk` | per-system reserved-chunk for early-boot | shared |
| `pcpu_alloc_mutex` | per-system slow-path mutex | shared |
| `pcpu_chunk_lists[]` | per-slot free-list of chunks | shared |
| `PCPU_BITMAP_BLOCK_SIZE` (4KB) | per-block size | shared |

## Compatibility contract

REQ-1: Per-chunk:
- pages_per_unit: # pages per replica.
- nr_units: NR_CPUS.
- Total: pages_per_unit × nr_units × PAGE_SIZE.
- alloc_map: bitmap (PCPU_MIN_ALLOC_SIZE-granularity).
- bound_map: per-allocated-region boundary marker (1 = boundary).
- md_blocks[]: per-PCPU_BITMAP_BLOCK_SIZE metadata.

REQ-2: Per-PCPU_MIN_ALLOC_SIZE:
- Default 4 bytes.
- Per-alloc rounded up.

REQ-3: pcpu_alloc(size, align, reserved, gfp):
- size <= PCPU_MIN_UNIT_SIZE.
- spin_lock_irq(pcpu_lock) for fast-path.
- per-chunk (in chunk_lists) walk: find free run.
- If no chunk has space: pcpu_create_chunk in slow-path.
- offset returned (relative to chunk-base).
- per-replica advances: __pcpu_unit_offset[cpu] + offset.

REQ-4: free_percpu(ptr):
- per-chunk lookup (ptr → chunk + offset).
- bitmap clear range.
- coalesce free regions; possibly enqueue chunk for free.

REQ-5: Per-arch boot init (pcpu_setup_first_chunk):
- arch supplies group-info: per-NUMA node group of CPUs.
- pcpu_first_chunk: covers static-percpu (kernel BSS region).
- pcpu_reserved_chunk: for module-percpu before first-real-chunk available.

REQ-6: Per-NUMA awareness:
- Per-NUMA-node group of CPUs gets contiguous unit-region.
- alloc tries node-local first.

REQ-7: Per-population state:
- chunk.populated bitmap: per-page populated (RAM-backed).
- non-populated pages allocated on-demand via populate_chunk.

REQ-8: Per-reserved-chunk:
- Module-load uses reserved-chunk to avoid creating new chunk.
- Returns PCPU_DUMMY_DECL flag.

REQ-9: Per-sysfs:
- /proc/sys/vm/percpu_enabled (legacy, mostly defunct).
- /proc/vmallocinfo includes percpu chunks.

REQ-10: Per-this_cpu_*() macros:
- this_cpu_ptr(ptr): current-CPU replica.
- per_cpu_ptr(ptr, cpu): explicit CPU's replica.
- this_cpu_inc(var): atomic-relax inc.
- per_cpu(var, cpu): static-percpu access.

REQ-11: Per-percpu_counter:
- Built atop alloc_percpu.
- Per-CPU local count + global atomic; flush threshold.

REQ-12: Per-shrink (in-kernel):
- pcpu_balance_workfn: periodic; reclaims excess chunks.
- Per-chunk fully-free → eligible for reclaim.

## Acceptance Criteria

- [ ] AC-1: alloc_percpu(struct foo): returns valid ptr; this_cpu_ptr derives current-CPU replica.
- [ ] AC-2: free_percpu(ptr): bitmap cleared.
- [ ] AC-3: alloc_percpu_gfp(size, GFP_ATOMIC): no-sleep allocation.
- [ ] AC-4: chunk full: pcpu_create_chunk creates new chunk.
- [ ] AC-5: NUMA: alloc on node-X CPU: prefers node-X chunk.
- [ ] AC-6: Module load: __alloc_reserved_percpu uses reserved chunk.
- [ ] AC-7: alloc with align=8: aligned ptr.
- [ ] AC-8: pcpu_balance_workfn: idle chunks reclaimed.
- [ ] AC-9: this_cpu_inc atomically: per-CPU stat.
- [ ] AC-10: Boot: pcpu_setup_first_chunk: static-percpu init.

## Architecture

Per-chunk:

```
struct PcpuChunk {
  list: ListLink,                                 // pcpu_chunk_lists[slot]
  free_bytes: u32,
  contig_bits: u32,                               // largest contiguous-free
  contig_bits_start: i32,
  alloc_map: BitMap,                              // per-byte bitmap
  bound_map: BitMap,                              // per-region boundary
  md_blocks: Vec<PcpuBlockMd>,
  base_addr: *mut u8,
  populated: BitMap,                              // per-page populated
  nr_pages: u32,
  nr_populated: u32,
  start_offset: u32,
  end_offset: u32,
  obj_cgroups: Option<Vec<&ObjCgroup>>,            // per-obj memcg
  ...
}

struct PcpuBlockMd {
  scan_hint: u32,
  scan_hint_start: u32,
  contig_hint: u32,
  contig_hint_start: u32,
  left_free: u32,
  right_free: u32,
  first_free: u32,
  nr_bits: u32,
}

const PCPU_BITMAP_BLOCK_SIZE: usize = 4096;
const PCPU_MIN_ALLOC_SIZE: usize = 4;
```

Globals:

```
static PCPU_LOCK: SpinLock;
static PCPU_ALLOC_MUTEX: Mutex<()>;
static PCPU_CHUNK_LISTS: [ListHead<PcpuChunk>; PCPU_NR_SLOTS];
static PCPU_FIRST_CHUNK: *PcpuChunk;
static PCPU_RESERVED_CHUNK: *PcpuChunk;
static __PCPU_UNIT_OFFSET: [u64; NR_CPUS];          // per-CPU base offset
```

`Pcpu::alloc(size, align, reserved, gfp) -> *u8`:
1. /* Lock + fast path */
2. spin_lock_irqsave(&pcpu_lock).
3. /* Walk pcpu_chunk_lists[chunk-slot] looking for chunk with enough contig-bits */
4. for chunk in lists[best_slot]:
   - off = pcpu_find_block_fit(chunk, size_bits, align_bits, ...).
   - if off != -1:
     - /* Pull from chunk */
     - pcpu_block_update_alloc_offset(chunk, off, size_bits).
     - chunk.free_bytes -= size_bytes.
     - return chunk.base_addr + off.
5. spin_unlock_irqrestore.
6. /* Slow-path: create new chunk */
7. mutex_lock(&pcpu_alloc_mutex).
8. chunk = Pcpu::create_chunk(gfp).
9. /* Re-attempt with new chunk; or retry on failure */
10. mutex_unlock.

`Pcpu::create_chunk(gfp) -> *PcpuChunk`:
1. /* Allocate metadata */
2. chunk = pcpu_alloc_chunk(gfp).
3. /* Per-NUMA group: alloc pages */
4. for nid in groups:
   - pages = alloc_pages_node(nid, gfp, get_order(pages_per_unit × PAGE_SIZE)).
   - vmap to chunk.base_addr.
5. populate per-page populated-bitmap.
6. list_add(&chunk.list, &pcpu_chunk_lists[empty-slot]).

`Pcpu::free_percpu(ptr)`:
1. chunk = Pcpu::ptr_to_chunk(ptr).
2. off = (uintptr_t)ptr - chunk.base_addr.
3. spin_lock(&pcpu_lock).
4. size_bits = read-bound_map(chunk, off).
5. clear-alloc-map(chunk, off, size_bits).
6. coalesce per-block-md.
7. chunk.free_bytes += size_bytes.
8. /* Possibly relink chunk to better slot */
9. spin_unlock.

`Pcpu::ptr_to_chunk(ptr)`:
1. /* Per-page lookup via page->private */
2. page = virt_to_page(ptr).
3. Return page.private as *PcpuChunk.

`Pcpu::alloc_percpu_gfp(size, align, gfp)`:
1. /* Always per-CPU-aligned */
2. if size > PCPU_MIN_UNIT_SIZE: return NULL.
3. Return Pcpu::alloc(size, align, false, gfp).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `chunk_in_chunk_list` | INVARIANT | per-chunk in exactly one pcpu_chunk_lists[slot]. |
| `alloc_map_bound_consistent` | INVARIANT | bound_map[off]=1 ⟺ region starts at off. |
| `free_bytes_eq_unallocated` | INVARIANT | chunk.free_bytes == sum-of-zero-bits in alloc_map × MIN_ALLOC. |
| `populated_le_nr_pages` | INVARIANT | chunk.nr_populated ≤ chunk.nr_pages. |
| `lock_held_during_mutate` | INVARIANT | per-alloc_map/bound_map mutate: pcpu_lock held. |

### Layer 2: TLA+

`mm/percpu.tla`:
- Per-chunk alloc/free + per-NUMA group + per-balance.
- Properties:
  - `safety_no_double_alloc` — per-bit set ⟹ at-most-one allocator owns.
  - `safety_replica_offset_consistent` — per-CPU access uses correct __pcpu_unit_offset.
  - `liveness_balance_eventually_reclaims` — per-fully-empty-chunk eventually destroyed.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Pcpu::alloc` post: ptr ∈ chunk's range; alloc_map updated | `Pcpu::alloc` |
| `Pcpu::free_percpu` post: alloc_map cleared; chunk.free_bytes incremented | `Pcpu::free_percpu` |
| `Pcpu::create_chunk` post: chunk in list; pages allocated per-group | `Pcpu::create_chunk` |
| `Pcpu::ptr_to_chunk` post: returned-chunk contains ptr | `Pcpu::ptr_to_chunk` |

### Layer 4: Verus/Creusot functional

`Per-CPU dynamic mem alloc → per-CPU replica access via __pcpu_unit_offset[cpu]; per-NUMA local-alloc preferred` semantic equivalence: per-Linux per-cpu-allocator design.

## Hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening.)

Per-CPU allocator reinforcement:

- **Per-pcpu_lock for alloc_map mutation** — defense against per-bit race.
- **Per-bound_map maintains region-start** — defense against per-malformed-free OOB.
- **Per-PCPU_MIN_UNIT_SIZE upper-bound** — defense against per-huge-alloc unbounded.
- **Per-chunk vmap separate from kmap** — defense against per-page wrong-mapping.
- **Per-NUMA group aware** — defense against per-cross-node thrash.
- **Per-balance work periodic** — defense against per-leaked-chunk OOM.
- **Per-reserved-chunk only for module-load** — defense against per-runtime-alloc using up reserve.
- **Per-this_cpu_*-macros use preempt-safe atomics** — defense against per-preempted access wrong-CPU.
- **Per-page->private = chunk** — defense against per-page misattribution.
- **Per-free-balance re-link to slot** — defense against per-fragmented chunk lookup-fail.

## Grsecurity/PaX-style Reinforcement

Baseline hardening that applies to the per-CPU allocator (`alloc_percpu` / `free_percpu`):

- **PAX_USERCOPY** — per-CPU areas hold kernel-only state; no user-copy path.
- **PAX_KERNEXEC** — `pcpu_alloc`, `pcpu_free`, `pcpu_balance_work` reside in `.rodata`.
- **PAX_RANDKSTACK** — percpu allocator entered with randomized kernel stack offset on syscall paths.
- **PAX_REFCOUNT** — chunk refcounts and `pcpu_nr_populated` use saturating refcount_t; an unbalanced alloc/free traps before chunk-list corruption.
- **PAX_MEMORY_SANITIZE** — freed per-CPU slots zeroed before they re-enter the slot free-list, preventing leak of one subsystem's per-CPU state into the next consumer.
- **PAX_UDEREF** — `this_cpu_*` operations are kernel-internal; never indexed by user-supplied values.
- **PAX_RAP / kCFI** — `pcpu_alloc` callbacks (page-allocator backing, vmalloc backing) dispatched through type-checked vtables.
- **GRKERNSEC_HIDESYM** — per-CPU chunk and offset addresses redacted from non-root WARN output.
- **GRKERNSEC_DMESG** — chunk-balance / OOM-on-percpu warnings restricted from unprivileged dmesg.

percpu-specific reinforcement:

- **Per-CPU allocator PAX_REFCOUNT** — `chunk->nr_populated` and chunk slot occupancy use saturating semantics so a double-`free_percpu` cannot wrap a chunk into a "fully free" state while still pinned.
- **Alloc/free integrity** — `pcpu_chunk_addr_search` validates that a freed pointer lies inside an existing chunk; out-of-range frees are rejected before they corrupt the chunk free-list.
- **Reserved chunk only for module load** — the reserved percpu chunk is consumed only during early module init; runtime allocations cannot drain the reserve and starve module loading.
- **`this_cpu_*` macros preempt-safe** — wrap-around to the right CPU's offset is enforced; a preempted access cannot land on a different CPU's slot.
- **`page->private = chunk`** — back-pointer from page to its owning chunk validated on free; cross-chunk free attempts are refused.
- **Balance work periodic** — leaked chunks are reclaimed by the balance worker so a slow leak cannot turn into an unbounded percpu OOM.

Rationale: the percpu allocator backs critical kernel state (sched, cgroup, network); a refcount underflow or cross-chunk free becomes a kernel-wide write primitive. The above keep percpu chunks attestable, refcount-safe, and immune to cross-CPU slot confusion.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- mm/00-overview (Tier-2)
- arch-percpu (covered in `arch/x86/00-overview.md` Tier-3)
- percpu_counter / percpu_ref (covered separately)
- Tracepoints (covered separately)
- Implementation code
