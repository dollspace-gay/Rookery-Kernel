---
title: "Tier-3: mm/memcontrol.c — Memory cgroup (memcg) accounting + reclaim"
tags: ["tier-3", "mm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Memory cgroup (memcg) implements per-cgroup memory accounting + limits + reclaim. Per-memcg has 4 page_counters: `memory` (anon + file + kmem + sock), `memsw` (memory + swap, v1-only), `kmem` (slab), `swap`. Per-charge: `try_charge` walks ancestor hierarchy, atomically reserves N pages; on over-limit, kicks per-memcg reclaim (`mem_cgroup_reclaim`). Per-LRU: per-memcg LRU lists for fair reclaim. Per-cgroup-v2 unified hierarchy: memory.{low, high, max, peak} as throttling knobs. Per-protection (memory.min): per-memcg unreclaimable. Critical for: container memory enforcement; OOM-isolation per cgroup.

This Tier-3 covers `memcontrol.c` (~5958 lines, includes both v1 and v2 paths).

### Acceptance Criteria

- [ ] AC-1: mem_cgroup_alloc: page_counters initialized; per-node arrays allocated.
- [ ] AC-2: cgroup-v2 mkdir: memcg created; counters at zero.
- [ ] AC-3: charge per-allocation: page_counter_try_charge succeeds within limit.
- [ ] AC-4: charge over .max: try_charge invokes reclaim; retry; -ENOMEM if still over.
- [ ] AC-5: per-memcg-OOM: kill within memcg only.
- [ ] AC-6: memory.protected = N: under reclaim pressure, N pages protected.
- [ ] AC-7: memory.high triggers PSI: tasks throttled.
- [ ] AC-8: kmem charge: kmem_cache_alloc → memcg.kmem incremented.
- [ ] AC-9: swap-out: swap_counter incremented; memory_counter decremented.
- [ ] AC-10: cgroup-v2 memcg.peak: tracks max-observed.
- [ ] AC-11: rmdir empty memcg: page_counters at zero; memcg freed.
- [ ] AC-12: orphan objcgs reparent on memcg-die.

### Architecture

Per-memcg state:

```
struct MemCgroup {
  css: CgroupSubsysState,
  id: u16,
  memory: PageCounter,
  swap: PageCounter,
  kmem: PageCounter,
  memsw: Option<PageCounter>,                    // v1-only
  oom_lock: Mutex<()>,
  oom_kill_disable: bool,
  oom_kill: AtomicU32,
  high_threshold: AtomicI64,
  swappiness: AtomicI32,
  use_hierarchy: bool,
  nodeinfo: Vec<MemCgPerNode>,                   // per-NUMA-node
  events: MemCgEvents,
  vmstats: PerCpu<MemCgVmStats>,
  protections: MemCgProtection,                  // min, low
  ...
}

struct PageCounter {
  usage: AtomicI64,                              // current (pages)
  watermark: AtomicI64,                          // peak
  failcnt: AtomicI64,
  parent: Option<*PageCounter>,
  min: AtomicI64,
  low: AtomicI64,
  max: AtomicI64,
}

struct MemCgPerNode {
  lruvec: Lruvec,                                 // per-(node, memcg) LRU
  lru_lock: SpinLock,
  ...
}
```

`PageCounter::try_charge(c, nr_pages, fail_p) -> Result<()>`:
1. Iterate from c upward via parent:
   - new = atomic_add_return(nr_pages, &cur.usage).
   - if new > cur.max: *fail_p = cur; atomic_sub(nr_pages, &cur.usage); return Err.
   - watermark = max(watermark, new).
2. Ok.

`PageCounter::uncharge(c, nr_pages)`:
1. Iterate from c upward: atomic_sub(nr_pages, &cur.usage).

`Memcg::try_charge(memcg, gfp, nr_pages) -> Result<()>`:
1. for retry in 0..MAX_RECLAIM_RETRIES:
   - err = PageCounter::try_charge(&memcg.memory, nr_pages, &fail_p).
   - if !err: return Ok.
   - reclaimed = Memcg::reclaim(fail_p_memcg, gfp, nr_pages, MEMCG_RECLAIM_MAY_SWAP).
   - if reclaimed >= nr_pages: continue.
2. /* Last resort: per-memcg OOM */
3. mem_cgroup_oom(memcg, gfp, get_order(nr_pages)).
4. if oom_killed_in_memcg: continue once more.
5. Return Err(NoMem).

`Memcg::__charge(folio, mm, gfp) -> Result<()>`:
1. memcg = Memcg::from_mm(mm).
2. nr_pages = folio_nr_pages(folio).
3. err = Memcg::try_charge(memcg, gfp, nr_pages)?
4. folio_set_memcg(folio, memcg).
5. /* Update per-memcg event-counters */

`Memcg::uncharge(folio)`:
1. memcg = folio_memcg(folio).
2. nr_pages = folio_nr_pages(folio).
3. PageCounter::uncharge(&memcg.memory, nr_pages).
4. folio_clear_memcg(folio).

`Memcg::reclaim(memcg, gfp, nr_pages, flags) -> usize`:
1. Per-memcg-LRU shrink_lruvec.
2. For each lruvec in memcg-tree:
   - shrink_lruvec(lruvec, sc).
   - reclaimed += sc.nr_reclaimed.
3. Return reclaimed.

`Memcg::oom_synchronize(memcg, wait)`:
1. /* Per-memcg OOM mutual-exclusion */
2. mutex_lock(&memcg.oom_lock).
3. select OOM victim within memcg-subtree.
4. send SIGKILL to victim.
5. mutex_unlock.
6. wait for victim-exit.

### Out of Scope

- Cgroup core (covered in `kernel/cgroup/` separately)
- Reclaim core (covered in `reclaim.md` Tier-3)
- Page counter library (folded here)
- Per-memcg slab (covered separately if needed)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct mem_cgroup` | per-memcg state | `MemCgroup` |
| `struct page_counter` | per-counter (atomic + parent + max) | `PageCounter` |
| `struct mem_cgroup_per_node` | per-node LRU + lru_lock + ... | `MemCgPerNode` |
| `mem_cgroup_alloc()` | per-memcg create | `Memcg::alloc` |
| `mem_cgroup_init()` | per-module init | `Memcg::init` |
| `try_charge()` | per-mm charge attempt | `Memcg::try_charge` |
| `cancel_charge()` | per-mm charge cancel | `Memcg::cancel_charge` |
| `__mem_cgroup_charge()` | per-folio charge | `Memcg::__charge` |
| `mem_cgroup_uncharge()` | per-folio uncharge | `Memcg::uncharge` |
| `mem_cgroup_reclaim()` | per-memcg reclaim | `Memcg::reclaim` |
| `mem_cgroup_swap_in_charge()` | per-swap-in charge | `Memcg::swap_in_charge` |
| `mem_cgroup_oom_synchronize()` | per-OOM wait+kill | `Memcg::oom_synchronize` |
| `mem_cgroup_protected()` | per-min/low protection check | `Memcg::protected` |
| `mem_cgroup_iter()` | per-hierarchy walk | `Memcg::iter` |
| `mem_cgroup_from_task()` | per-task → memcg | `Memcg::from_task` |
| `mem_cgroup_id_get()` / `_put()` | per-memcg unique ID for swap-cache | `Memcg::id_get` / `id_put` |
| `page_counter_try_charge()` | per-counter atomic charge | `PageCounter::try_charge` |
| `page_counter_uncharge()` | per-counter atomic uncharge | `PageCounter::uncharge` |
| `MEMCG_DATA_OBJCGS` | per-slab obj_cgroup-array tag | shared |
| `obj_cgroup` | per-slab-obj cgroup attribution | `ObjCgroup` |

### compatibility contract

REQ-1: cgroup-v2 unified hierarchy:
- /sys/fs/cgroup/<group>/memory.{current, peak, max, high, low, min, swap.{current, max, peak}, kmem.{...}, events, stat, ...}.

REQ-2: Per-memcg page_counters:
- memory: total memory charged (in pages).
- memsw (v1): memory + swap.
- kmem: kernel memory (slab).
- swap: swap usage.

REQ-3: try_charge(memcg, gfp_mask, nr_pages):
- For each memcg ∈ memcg-ancestor-chain:
  - page_counter_try_charge(&memcg.memory, nr_pages).
  - If over: mem_cgroup_reclaim(memcg, gfp_mask, nr_pages); retry.
  - If still over after MAX_RECLAIM_RETRIES: invoke OOM or return -ENOMEM.

REQ-4: __mem_cgroup_charge(folio, mm, gfp_mask):
- memcg = mem_cgroup_from_mm(mm).
- nr_pages = folio_nr_pages(folio).
- err = try_charge(memcg, gfp_mask, nr_pages).
- folio_set_memcg(folio, memcg).

REQ-5: mem_cgroup_uncharge(folio):
- memcg = folio_memcg(folio).
- nr_pages = folio_nr_pages(folio).
- page_counter_uncharge(&memcg.memory, nr_pages).
- folio_clear_memcg(folio).

REQ-6: Per-LRU: mem_cgroup_per_node:
- Per-node has per-memcg LRU lists (active/inactive × file/anon).
- lru_lock per-node-per-memcg.

REQ-7: mem_cgroup_reclaim(memcg, gfp, nr_pages):
- Per-memcg shrink_lruvec via vmscan.
- Walks per-memcg LRU; per-page tries to reclaim.
- Per-protected (memcg.min): not reclaimed.

REQ-8: Per-memcg.min / .low (protection):
- mem_cgroup_protected returns level:
  - PROTECTED_NONE: no protection.
  - PROTECTED_LOW: best-effort protected.
  - PROTECTED_MIN: hard-protected (only OOM).

REQ-9: Per-memcg.high (throttling):
- Per-charge above .high triggers PSI-pressure throttling.

REQ-10: Per-memcg.max (limit):
- Per-charge above .max + retry-fails → OOM in memcg or -ENOMEM.

REQ-11: Per-memcg-OOM:
- Per-OOM kill victim within memcg only (not host-OOM).
- mem_cgroup_oom_synchronize: wait for OOM-kill complete.

REQ-12: Per-objcg (kmem/slab):
- Per-slab-obj attributed to obj_cgroup (separate from memcg for ref-management).
- obj_cgroup → memcg via active-pointer.
- Per-memcg-die: obj_cgroup decoupled; orphan objs reparent to parent.

REQ-13: Per-task switch:
- mm_struct.memcg referenced via mm.owner.
- task move: cgroup_migrate_*; per-task pages reparented (cgroup-v1).
- v2: pages stay in original memcg until uncharge.

REQ-14: Per-id:
- mem_cgroup.id: u16 unique ID for swap-cache attribution.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `usage_le_max_or_charge_failed` | INVARIANT | per-PageCounter: usage ≤ max post-successful-charge. |
| `parent_chain_acyclic` | INVARIANT | per-counter parent chain finite. |
| `id_unique_per_memcg` | INVARIANT | per-memcg.id unique system-wide. |
| `folio_memcg_consistent` | INVARIANT | folio.memcg == charged-memcg post-charge. |
| `protection_min_le_low_le_max` | INVARIANT | min ≤ low ≤ max. |

### Layer 2: TLA+

`mm/memcg.tla`:
- Per-memcg charge/uncharge + per-hierarchy + reclaim + OOM.
- Properties:
  - `safety_no_double_charge` — per-page charged once.
  - `safety_uncharge_balanced` — per-charge has matching uncharge at folio-free.
  - `safety_no_orphan_objcg_after_die` — per-memcg-die: objcgs reparented.
  - `liveness_overflow_eventually_oom` — per-charge over .max with no reclaim ⟹ OOM.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `PageCounter::try_charge` post: usage += nr_pages OR Err with usage unchanged | `PageCounter::try_charge` |
| `Memcg::try_charge` post: per-ancestor chain charged | `Memcg::try_charge` |
| `Memcg::__charge` post: folio.memcg = memcg | `Memcg::__charge` |
| `Memcg::uncharge` post: usage -= nr_pages; folio.memcg cleared | `Memcg::uncharge` |
| `Memcg::reclaim` post: returned reclaimed-pages count | `Memcg::reclaim` |

### Layer 4: Verus/Creusot functional

`Per-cgroup-v2 memory accounting + limits + reclaim per memory.{min, low, high, max}; per-memcg-OOM contains kill within subtree` semantic equivalence: per-cgroup-v2 memory controller spec.

### hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening.)

Memcg-specific reinforcement:

- **Per-counter atomic-add-return** — defense against per-charge race causing overflow.
- **Per-hierarchy ancestor walk on charge** — defense against per-leaf-only counting bypass.
- **Per-memcg-OOM scoped to subtree** — defense against per-cgroup OOM killing host-tasks.
- **Per-protection (min/low)** — defense against per-cgroup unreclaimable hard-isolation.
- **Per-id uniqueness via xa_alloc** — defense against per-id-reuse swap-cache misattribution.
- **Per-RCU-protected mem_cgroup_iter** — defense against per-iter use of dying memcg.
- **Per-objcg reparent on die** — defense against per-stale-obj UAF on memcg-free.
- **Per-folio_memcg WRITE_ONCE** — defense against per-rcu torn-read.
- **Per-charge-retry bounded** — defense against per-overflow infinite retry.
- **Per-MAX_RECLAIM_RETRIES** — defense against per-charge livelock.
- **Per-task-move (v1) reparent** — defense against per-page-stale-attribution.

