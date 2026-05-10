---
title: "Tier-3: mm/workingset.c — Refault distance & LRU activation"
tags: ["tier-3", "mm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

**Workingset detection** distinguishes pages whose refault distance fits within the inactive list (likely-needed) from pages whose distance exceeds active+inactive sums (genuine miss, not promote-worthy). On eviction, `workingset_eviction(folio, target_memcg)` packs `{memcgid, pgdat->node_id, eviction_timestamp >> bucket_order[file], workingset-bit}` into an `xa_value` and returns it; the page-cache `xa_store` then places that shadow entry where the folio used to live. The eviction timestamp is `lruvec->nonresident_age` (per-lruvec atomic_long), advanced by `workingset_age_nonresident` on every reclaim/activation. On refault, `workingset_refault(folio, shadow)` calls `workingset_test_recent(shadow, file, &workingset, true)`: it unpacks `{memcgid, pgdat, eviction}`, takes the live `refault = atomic_long_read(&eviction_lruvec->nonresident_age)`, computes `refault_distance = (refault - eviction) & EVICTION_MASK`, and compares against `workingset_size = NR_ACTIVE_FILE + (if anon-or-swap-considered) NR_*_ANON + NR_INACTIVE_FILE`. If `refault_distance <= workingset_size`, the folio is activated (`folio_set_active`) and, if the shadow's workingset-bit was set, `folio_set_workingset` + `WORKINGSET_RESTORE_BASE` accounted. Per-`shadow_nodes` `struct list_lru` tracks xa_nodes that hold only shadow entries; a shrinker (`count_shadow_nodes`/`scan_shadow_nodes`) caps shadow-node count at `present_pages >> (XA_CHUNK_SHIFT - 3)` (≈ 1/8 density). Per-LRU_GEN (`CONFIG_LRU_GEN`): alternate `lru_gen_eviction` / `lru_gen_refault` encode `{min_seq << LRU_REFS_WIDTH | (refs-1)}` token, recency tested via `max_seq` distance < MAX_NR_GENS. Critical for: avoiding inactive-list thrashing, distinguishing transitioning workingsets from genuinely cold cache, page-cache promotion correctness.

This Tier-3 covers `mm/workingset.c` (~845 lines).

### Acceptance Criteria

- [ ] AC-1: pack_shadow / unpack_shadow round-trip preserves {memcgid, nid, eviction, workingset}.
- [ ] AC-2: workingset_eviction returns xa_value-tagged shadow (xa_is_value(s) == true).
- [ ] AC-3: workingset_age_nonresident increments lruvec.nonresident_age and propagates up parent_lruvec chain.
- [ ] AC-4: workingset_refault on recent shadow (refault_distance <= workingset_size) ⟹ folio_set_active + WORKINGSET_ACTIVATE incremented.
- [ ] AC-5: workingset_refault on stale shadow (refault_distance > workingset_size) ⟹ folio not activated; WORKINGSET_REFAULT still incremented.
- [ ] AC-6: Shadow with workingset-bit set ∧ recent ⟹ folio_set_workingset + WORKINGSET_RESTORE incremented.
- [ ] AC-7: workingset_test_recent with mem_cgroup_tryget failure ⟹ false (unless memcg disabled).
- [ ] AC-8: workingset_update_node: all-shadow node → adds to shadow_nodes + NR_WORKINGSET_NODES++.
- [ ] AC-9: workingset_update_node: any-page-or-empty node → removes from shadow_nodes + NR_WORKINGSET_NODES--.
- [ ] AC-10: count_shadow_nodes caps at present_pages >> (XA_CHUNK_SHIFT - 3).
- [ ] AC-11: shadow_lru_isolate xa_trylock failure ⟹ LRU_RETRY without state change.
- [ ] AC-12: scan_shadow_nodes reclaims node ⟹ WORKINGSET_NODERECLAIM++.
- [ ] AC-13: bucket_order[file] = max(0, fls_long(totalram_pages-1) - timestamp_bits[file]).
- [ ] AC-14: CONFIG_LRU_GEN: lru_gen_eviction encodes (min_seq << LRU_REFS_WIDTH | refs-1) token.
- [ ] AC-15: CONFIG_LRU_GEN: lru_gen_test_recent true ⟺ abs_diff(max_seq, token>>LRU_REFS_WIDTH) < MAX_NR_GENS.
- [ ] AC-16: workingset_init registers shrinker with SHRINKER_NUMA_AWARE | SHRINKER_MEMCG_AWARE and seeks=0.

### Architecture

```
const WORKINGSET_SHIFT: u32 = 1;
const EVICTION_SHIFT_FILE: u32 =
    (BITS_PER_LONG - BITS_PER_XA_VALUE) + WORKINGSET_SHIFT + NODES_SHIFT + MEM_CGROUP_ID_SHIFT;
const EVICTION_SHIFT_ANON: u32 = EVICTION_SHIFT_FILE + SWAP_COUNT_SHIFT;
const EVICTION_MASK_FILE: usize = !0 >> EVICTION_SHIFT_FILE;
const EVICTION_MASK_ANON: usize = !0 >> EVICTION_SHIFT_ANON;

static BUCKET_ORDER: [u32; ANON_AND_FILE] = [0; ANON_AND_FILE];   // set in init
static SHADOW_NODES: ListLru = ListLru::uninit();
```

`Workingset::pack_shadow(memcgid, pgdat, eviction, workingset, file) -> *mut c_void`:
1. let mask = if file { EVICTION_MASK_FILE } else { EVICTION_MASK_ANON }.
2. let mut e = eviction & mask.
3. e = (e << MEM_CGROUP_ID_SHIFT) | memcgid as usize.
4. e = (e << NODES_SHIFT) | pgdat.node_id as usize.
5. e = (e << WORKINGSET_SHIFT) | (workingset as usize & 1).
6. return XArray::mk_value(e).

`Workingset::unpack_shadow(shadow) -> UnpackedShadow`:
1. let mut entry = XArray::to_value(shadow).
2. let workingset = (entry & ((1 << WORKINGSET_SHIFT) - 1)) != 0.
3. entry >>= WORKINGSET_SHIFT.
4. let nid = entry & ((1 << NODES_SHIFT) - 1).
5. entry >>= NODES_SHIFT.
6. let memcgid = (entry & ((1 << MEM_CGROUP_ID_SHIFT) - 1)) as i32.
7. entry >>= MEM_CGROUP_ID_SHIFT.
8. return UnpackedShadow { memcgid, pgdat: NodeData::of(nid), eviction: entry, workingset }.

`Workingset::age_nonresident(lruvec, nr_pages)`:
1. loop:
   - lruvec.nonresident_age.fetch_add(nr_pages, Ordering::Relaxed).
   - if let Some(p) = lruvec.parent_lruvec(): lruvec = p; continue.
   - break.

`Workingset::eviction(folio, target_memcg) -> *mut c_void`:
1. assert(!folio.test_lru()).
2. assert(folio.ref_count() == 0).
3. assert(folio.test_locked()).
4. if cfg(LRU_GEN) ∧ Workingset::lru_gen_enabled(): return Workingset::lru_gen_eviction(folio).
5. let pgdat = folio.pgdat().
6. let file = folio.is_file_lru() as usize.
7. let lruvec = mem_cgroup_lruvec(target_memcg, pgdat).
8. let memcgid = mem_cgroup_private_id(lruvec.memcg()).
9. let eviction = lruvec.nonresident_age.load(Ordering::Relaxed) >> BUCKET_ORDER[file].
10. Workingset::age_nonresident(lruvec, folio.nr_pages()).
11. return Workingset::pack_shadow(memcgid, pgdat, eviction, folio.test_workingset(), file != 0).

`Workingset::test_recent(shadow, file, out_workingset, flush) -> bool`:
1. if cfg(LRU_GEN) ∧ Workingset::lru_gen_enabled():
   - rcu_read_lock.
   - let recent = Workingset::lru_gen_test_recent(shadow, &lruvec, &eviction, out_workingset, file).
   - rcu_read_unlock.
   - return recent.
2. rcu_read_lock.
3. let UnpackedShadow { memcgid, pgdat, eviction: stored, workingset: ws } = Workingset::unpack_shadow(shadow).
4. *out_workingset = ws.
5. let stored = stored << BUCKET_ORDER[file as usize].
6. let mut eviction_memcg = mem_cgroup_from_private_id(memcgid).
7. if !mem_cgroup_tryget(eviction_memcg): eviction_memcg = None.
8. rcu_read_unlock.
9. if !mem_cgroup_disabled() ∧ eviction_memcg.is_none(): return false.
10. if flush: mem_cgroup_flush_stats_ratelimited(eviction_memcg).
11. let lruvec = mem_cgroup_lruvec(eviction_memcg, pgdat).
12. let refault = lruvec.nonresident_age.load(Ordering::Relaxed).
13. let mask = if file { EVICTION_MASK_FILE } else { EVICTION_MASK_ANON }.
14. let refault_distance = refault.wrapping_sub(stored) & mask.
15. let mut size = lruvec_page_state(lruvec, NR_ACTIVE_FILE).
16. if !file: size += lruvec_page_state(lruvec, NR_INACTIVE_FILE).
17. if mem_cgroup_get_nr_swap_pages(eviction_memcg) > 0:
    - size += lruvec_page_state(lruvec, NR_ACTIVE_ANON).
    - if file: size += lruvec_page_state(lruvec, NR_INACTIVE_ANON).
18. mem_cgroup_put(eviction_memcg).
19. return refault_distance <= size.

`Workingset::refault(folio, shadow)`:
1. assert(folio.test_locked()).
2. let file = folio.is_file_lru().
3. if cfg(LRU_GEN) ∧ Workingset::lru_gen_enabled(): Workingset::lru_gen_refault(folio, shadow); return.
4. let nr = folio.nr_pages().
5. let memcg = get_mem_cgroup_from_folio(folio).
6. let lruvec = mem_cgroup_lruvec(memcg, folio.pgdat()).
7. mod_lruvec_state(lruvec, WORKINGSET_REFAULT_BASE + file as i32, nr as i64).
8. let mut workingset = false.
9. if !Workingset::test_recent(shadow, file, &mut workingset, true):
   - mem_cgroup_put(memcg); return.
10. folio.set_active().
11. Workingset::age_nonresident(lruvec, nr).
12. mod_lruvec_state(lruvec, WORKINGSET_ACTIVATE_BASE + file as i32, nr as i64).
13. if workingset:
    - folio.set_workingset().
    - lru_note_cost_refault(folio).
    - mod_lruvec_state(lruvec, WORKINGSET_RESTORE_BASE + file as i32, nr as i64).
14. mem_cgroup_put(memcg).

`Workingset::activation(folio)`:
1. if mem_cgroup_disabled() ∨ folio.memcg_charged():
   - rcu_read_lock.
   - Workingset::age_nonresident(folio.lruvec(), folio.nr_pages()).
   - rcu_read_unlock.

`Workingset::update_node(node)` — caller holds `node.array.xa_lock`:
1. let page = virt_to_page(node).
2. let all_shadow = node.count != 0 ∧ node.count == node.nr_values.
3. if all_shadow ∧ node.private_list.is_empty():
   - SHADOW_NODES.add(&node.private_list).
   - __inc_node_page_state(page, WORKINGSET_NODES).
4. else if !all_shadow ∧ !node.private_list.is_empty():
   - SHADOW_NODES.del(&node.private_list).
   - __dec_node_page_state(page, WORKINGSET_NODES).

`Workingset::count_shadow_nodes(shrinker, sc) -> u64`:
1. let nodes = SHADOW_NODES.shrink_count(sc).
2. if nodes == 0: return SHRINK_EMPTY.
3. let pages = if cfg(MEMCG) ∧ sc.memcg.is_some():
   - mem_cgroup_flush_stats_ratelimited(sc.memcg).
   - let lruvec = mem_cgroup_lruvec(sc.memcg, NodeData::of(sc.nid)).
   - Σ lruvec_lru_size(lruvec, i, MAX_NR_ZONES-1) for i in 0..NR_LRU_LISTS
     + (NR_SLAB_RECLAIMABLE_B + NR_SLAB_UNRECLAIMABLE_B) >> PAGE_SHIFT.
   else: node_present_pages(sc.nid).
4. let max_nodes = pages >> (XA_CHUNK_SHIFT - 3).
5. if nodes <= max_nodes: return 0.
6. return nodes - max_nodes.

`Workingset::shadow_lru_isolate(item, lru, _arg) -> LruStatus`:
1. let node = container_of(item, xa_node, private_list).
2. let mapping = container_of(node.array, address_space, i_pages).
3. if !mapping.i_pages.try_lock(): lru.unlock_irq(); return LRU_RETRY.
4. if mapping.host.is_some():
   - if !mapping.host.i_lock.try_lock(): mapping.i_pages.unlock; lru.unlock_irq; return LRU_RETRY.
5. lru.isolate(item).
6. __dec_node_page_state(virt_to_page(node), WORKINGSET_NODES).
7. lru.unlock.
8. assert(node.nr_values > 0).
9. assert(node.count == node.nr_values).
10. xa_delete_node(node, Workingset::update_node).
11. mod_lruvec_kmem_state(node, WORKINGSET_NODERECLAIM, 1).
12. mapping.i_pages.unlock_irq.
13. if mapping.host: if mapping_shrinkable(mapping): inode_lru_list_add(mapping.host); host.i_lock.unlock.
14. cond_resched.
15. return LRU_REMOVED_RETRY.

`Workingset::init()` — module_init:
1. const_assert(BITS_PER_LONG >= EVICTION_SHIFT_FILE).
2. let timestamp_bits = BITS_PER_LONG - EVICTION_SHIFT_FILE.
3. let timestamp_bits_anon = BITS_PER_LONG - EVICTION_SHIFT_ANON.
4. let max_order = fls_long(totalram_pages() - 1).
5. BUCKET_ORDER[WORKINGSET_FILE] = max_order.saturating_sub(timestamp_bits).
6. BUCKET_ORDER[WORKINGSET_ANON] = max_order.saturating_sub(timestamp_bits_anon).
7. pr_info!("workingset: timestamp_bits=… max_order=… bucket_order=…").
8. let s = shrinker_alloc(SHRINKER_NUMA_AWARE | SHRINKER_MEMCG_AWARE, "mm-shadow").
9. list_lru_init_memcg_key(&SHADOW_NODES, s, &SHADOW_NODES_KEY).
10. s.count_objects = Workingset::count_shadow_nodes.
11. s.scan_objects = Workingset::scan_shadow_nodes.
12. s.seeks = 0.
13. shrinker_register(s).

### Out of Scope

- mm/vmscan.c reclaim core (covered in `reclaim.md` Tier-3)
- mm/swap.c LRU list-management primitives (covered separately if expanded)
- mm/filemap.c page-cache insertion/deletion (covered in `page-cache.md` Tier-3)
- mm/memcontrol.c memcg / lruvec management (covered in `memcg.md` Tier-3)
- lib/xarray.c xa_node and shadow-entry encoding primitives (covered separately if expanded)
- mm/mglru / CONFIG_LRU_GEN core (multi-gen LRU framework — covered separately if expanded)
- mm/page_io.c readahead refault triggers (covered in `page-io.md` Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `pack_shadow()` | per-evict encode shadow | `Workingset::pack_shadow` |
| `unpack_shadow()` | per-refault decode shadow | `Workingset::unpack_shadow` |
| `EVICTION_SHIFT` / `EVICTION_MASK` | per-shadow bit-layout (file) | `EVICTION_SHIFT_FILE` / `EVICTION_MASK_FILE` |
| `EVICTION_SHIFT_ANON` / `EVICTION_MASK_ANON` | per-shadow bit-layout (anon) | `EVICTION_SHIFT_ANON` / `EVICTION_MASK_ANON` |
| `bucket_order[ANON_AND_FILE]` | per-timestamp-granularity | `Workingset::BUCKET_ORDER` |
| `workingset_age_nonresident()` | per-lruvec age tick | `Workingset::age_nonresident` |
| `workingset_eviction()` | per-evict encode + age | `Workingset::eviction` |
| `workingset_test_recent()` | per-refault distance check | `Workingset::test_recent` |
| `workingset_refault()` | per-refault evaluate + activate | `Workingset::refault` |
| `workingset_activation()` | per-activation age tick | `Workingset::activation` |
| `workingset_update_node()` | per-xa-node track add/remove | `Workingset::update_node` |
| `count_shadow_nodes()` | per-shrinker count | `Workingset::count_shadow_nodes` |
| `scan_shadow_nodes()` | per-shrinker scan | `Workingset::scan_shadow_nodes` |
| `shadow_lru_isolate()` | per-shrinker isolate + reclaim | `Workingset::shadow_lru_isolate` |
| `shadow_nodes` (`struct list_lru`) | per-node-shadow LRU | `Workingset::SHADOW_NODES` |
| `lru_gen_eviction()` | per-MGLRU evict (CONFIG_LRU_GEN) | `Workingset::lru_gen_eviction` |
| `lru_gen_test_recent()` | per-MGLRU recency | `Workingset::lru_gen_test_recent` |
| `lru_gen_refault()` | per-MGLRU refault | `Workingset::lru_gen_refault` |
| `workingset_init()` | per-init shrinker + list_lru | `Workingset::init` |
| `lruvec->nonresident_age` | per-lruvec eviction-age counter | `Lruvec.nonresident_age` |
| `NR_WORKINGSET_NODES` | per-node shadow-node count stat | shared |
| `WORKINGSET_REFAULT_BASE` + file | per-{file,anon} refault counter | shared |
| `WORKINGSET_ACTIVATE_BASE` + file | per-{file,anon} activate counter | shared |
| `WORKINGSET_RESTORE_BASE` + file | per-{file,anon} restore counter | shared |
| `WORKINGSET_NODERECLAIM` | per-shadow-node-reclaim counter | shared |

### compatibility contract

REQ-1: Shadow encoding (file LRU):
- EVICTION_SHIFT = (BITS_PER_LONG - BITS_PER_XA_VALUE) + WORKINGSET_SHIFT + NODES_SHIFT + MEM_CGROUP_ID_SHIFT.
- EVICTION_MASK = (~0UL) >> EVICTION_SHIFT.
- Layout (low→high after xa_mk_value strip): [workingset:1] [nid:NODES_SHIFT] [memcgid:MEM_CGROUP_ID_SHIFT] [timestamp:timestamp_bits].
- WORKINGSET_SHIFT == 1 (one bit for workingset-flag).

REQ-2: Shadow encoding (anon LRU):
- EVICTION_SHIFT_ANON = EVICTION_SHIFT + SWAP_COUNT_SHIFT.
- EVICTION_MASK_ANON = (~0UL) >> EVICTION_SHIFT_ANON.

REQ-3: bucket_order[file]:
- Per file ∈ {WORKINGSET_FILE, WORKINGSET_ANON}.
- At init: timestamp_bits = BITS_PER_LONG - EVICTION_SHIFT (or _ANON).
- max_order = fls_long(totalram_pages() - 1).
- if max_order > timestamp_bits[file]: bucket_order[file] = max_order - timestamp_bits[file].
- else: bucket_order[file] = 0.

REQ-4: pack_shadow(memcgid, pgdat, eviction, workingset, file):
- eviction &= file ? EVICTION_MASK : EVICTION_MASK_ANON.
- eviction = (eviction << MEM_CGROUP_ID_SHIFT) | memcgid.
- eviction = (eviction << NODES_SHIFT) | pgdat.node_id.
- eviction = (eviction << WORKINGSET_SHIFT) | workingset.
- return xa_mk_value(eviction).

REQ-5: unpack_shadow(shadow, &memcgidp, &pgdat, &evictionp, &workingsetp):
- entry = xa_to_value(shadow).
- workingset = entry & ((1 << WORKINGSET_SHIFT) - 1).
- entry >>= WORKINGSET_SHIFT.
- nid = entry & ((1 << NODES_SHIFT) - 1).
- entry >>= NODES_SHIFT.
- memcgid = entry & ((1 << MEM_CGROUP_ID_SHIFT) - 1).
- entry >>= MEM_CGROUP_ID_SHIFT.
- *pgdat = NODE_DATA(nid).
- *evictionp = entry.
- *workingsetp = workingset.

REQ-6: workingset_age_nonresident(lruvec, nr_pages):
- /* Hierarchical aging: leaf age propagates to all ancestor cgroup-lruvecs */
- do:
  - atomic_long_add(nr_pages, &lruvec.nonresident_age).
- while (lruvec = parent_lruvec(lruvec)).

REQ-7: workingset_eviction(folio, target_memcg) -> shadow:
- VM_BUG_ON_FOLIO(folio_test_lru(folio)).
- VM_BUG_ON_FOLIO(folio_ref_count(folio)).
- VM_BUG_ON_FOLIO(!folio_test_locked(folio)).
- if lru_gen_enabled(): return lru_gen_eviction(folio).
- pgdat = folio_pgdat(folio).
- file = folio_is_file_lru(folio).
- lruvec = mem_cgroup_lruvec(target_memcg, pgdat).
- memcgid = mem_cgroup_private_id(lruvec_memcg(lruvec)).
- eviction = atomic_long_read(&lruvec.nonresident_age) >> bucket_order[file].
- workingset_age_nonresident(lruvec, folio_nr_pages(folio)).
- return pack_shadow(memcgid, pgdat, eviction, folio_test_workingset(folio), file).

REQ-8: workingset_test_recent(shadow, file, &workingset, flush) -> bool:
- if lru_gen_enabled():
  - rcu_read_lock.
  - recent = lru_gen_test_recent(shadow, ...).
  - rcu_read_unlock.
  - return recent.
- rcu_read_lock.
- unpack_shadow(shadow, &memcgid, &pgdat, &eviction, &workingset).
- eviction <<= bucket_order[file].
- eviction_memcg = mem_cgroup_from_private_id(memcgid).
- if !mem_cgroup_tryget(eviction_memcg): eviction_memcg = NULL.
- rcu_read_unlock.
- if !mem_cgroup_disabled() ∧ !eviction_memcg: return false.
- if flush: mem_cgroup_flush_stats_ratelimited(eviction_memcg).
- eviction_lruvec = mem_cgroup_lruvec(eviction_memcg, pgdat).
- refault = atomic_long_read(&eviction_lruvec.nonresident_age).
- refault_distance = ((refault - eviction) & (file ? EVICTION_MASK : EVICTION_MASK_ANON)).
- workingset_size = lruvec_page_state(eviction_lruvec, NR_ACTIVE_FILE).
- if !file: workingset_size += NR_INACTIVE_FILE.
- if mem_cgroup_get_nr_swap_pages(eviction_memcg) > 0:
  - workingset_size += NR_ACTIVE_ANON.
  - if file: workingset_size += NR_INACTIVE_ANON.
- mem_cgroup_put(eviction_memcg).
- return refault_distance <= workingset_size.

REQ-9: workingset_refault(folio, shadow):
- VM_BUG_ON_FOLIO(!folio_test_locked(folio)).
- file = folio_is_file_lru(folio).
- if lru_gen_enabled(): lru_gen_refault(folio, shadow); return.
- nr = folio_nr_pages(folio).
- memcg = get_mem_cgroup_from_folio(folio).
- lruvec = mem_cgroup_lruvec(memcg, folio_pgdat(folio)).
- mod_lruvec_state(lruvec, WORKINGSET_REFAULT_BASE + file, nr).
- if !workingset_test_recent(shadow, file, &workingset, true): goto out.
- folio_set_active(folio).
- workingset_age_nonresident(lruvec, nr).
- mod_lruvec_state(lruvec, WORKINGSET_ACTIVATE_BASE + file, nr).
- if workingset:
  - folio_set_workingset(folio).
  - lru_note_cost_refault(folio).
  - mod_lruvec_state(lruvec, WORKINGSET_RESTORE_BASE + file, nr).
- out: mem_cgroup_put(memcg).

REQ-10: workingset_activation(folio):
- if mem_cgroup_disabled() ∨ folio_memcg_charged(folio):
  - rcu_read_lock.
  - workingset_age_nonresident(folio_lruvec(folio), folio_nr_pages(folio)).
  - rcu_read_unlock.

REQ-11: workingset_update_node(node) — called under node.array.xa_lock:
- page = virt_to_page(node).
- if node.count ∧ node.count == node.nr_values:
  - /* All-shadow node */
  - if list_empty(&node.private_list):
    - list_lru_add_obj(&shadow_nodes, &node.private_list).
    - __inc_node_page_state(page, WORKINGSET_NODES).
- else:
  - if !list_empty(&node.private_list):
    - list_lru_del_obj(&shadow_nodes, &node.private_list).
    - __dec_node_page_state(page, WORKINGSET_NODES).

REQ-12: Shadow-node shrinker — count_shadow_nodes(shrinker, sc):
- nodes = list_lru_shrink_count(&shadow_nodes, sc).
- if !nodes: return SHRINK_EMPTY.
- if sc.memcg (CONFIG_MEMCG):
  - mem_cgroup_flush_stats_ratelimited(sc.memcg).
  - lruvec = mem_cgroup_lruvec(sc.memcg, NODE_DATA(sc.nid)).
  - pages = Σ lruvec_lru_size(lruvec, i, MAX_NR_ZONES-1) for i in [0..NR_LRU_LISTS).
  - pages += (NR_SLAB_RECLAIMABLE_B + NR_SLAB_UNRECLAIMABLE_B) >> PAGE_SHIFT.
- else: pages = node_present_pages(sc.nid).
- max_nodes = pages >> (XA_CHUNK_SHIFT - 3) /* ≈ 1/8 density */
- if nodes <= max_nodes: return 0.
- return nodes - max_nodes.

REQ-13: shadow_lru_isolate(item, lru, arg):
- /* lock order inversion: list_lru → i_pages */
- node = container_of(item, xa_node, private_list).
- mapping = container_of(node.array, address_space, i_pages).
- if !xa_trylock(&mapping.i_pages): spin_unlock_irq(&lru.lock) + LRU_RETRY.
- if mapping.host: if !spin_trylock(&mapping.host.i_lock): xa_unlock + spin_unlock_irq + LRU_RETRY.
- list_lru_isolate(lru, item).
- __dec_node_page_state(virt_to_page(node), WORKINGSET_NODES).
- spin_unlock(&lru.lock).
- WARN_ON_ONCE(!node.nr_values) → out_invalid.
- WARN_ON_ONCE(node.count != node.nr_values) → out_invalid.
- xa_delete_node(node, workingset_update_node).
- mod_lruvec_kmem_state(node, WORKINGSET_NODERECLAIM, 1).
- xa_unlock_irq + (mapping.host: inode_lru_list_add if mapping_shrinkable, spin_unlock).
- cond_resched.
- return LRU_REMOVED_RETRY.

REQ-14: workingset_init() (module_init):
- BUILD_BUG_ON(BITS_PER_LONG < EVICTION_SHIFT).
- compute timestamp_bits, timestamp_bits_anon, max_order = fls_long(totalram_pages()-1).
- bucket_order[WORKINGSET_FILE] = max(0, max_order - timestamp_bits).
- bucket_order[WORKINGSET_ANON] = max(0, max_order - timestamp_bits_anon).
- pr_info shadow geometry.
- workingset_shadow_shrinker = shrinker_alloc(SHRINKER_NUMA_AWARE | SHRINKER_MEMCG_AWARE, "mm-shadow").
- list_lru_init_memcg_key(&shadow_nodes, workingset_shadow_shrinker, &shadow_nodes_key).
- shrinker.count_objects = count_shadow_nodes.
- shrinker.scan_objects = scan_shadow_nodes.
- shrinker.seeks = 0 /* count reports only fully expendable */
- shrinker_register.

REQ-15: CONFIG_LRU_GEN — lru_gen_eviction(folio):
- BUILD_BUG_ON(LRU_GEN_WIDTH + LRU_REFS_WIDTH > BITS_PER_LONG - max(EVICTION_SHIFT, EVICTION_SHIFT_ANON)).
- rcu_read_lock.
- memcg = folio_memcg(folio).
- lruvec = mem_cgroup_lruvec(memcg, pgdat).
- lrugen = &lruvec.lrugen.
- min_seq = READ_ONCE(lrugen.min_seq[type]).
- token = (min_seq << LRU_REFS_WIDTH) | max(refs - 1, 0).
- atomic_long_add(delta, &lrugen.evicted[hist][type][tier]).
- memcg_id = mem_cgroup_private_id(memcg).
- rcu_read_unlock.
- return pack_shadow(memcg_id, pgdat, token, workingset, type).

REQ-16: CONFIG_LRU_GEN — lru_gen_test_recent(shadow, ..., file):
- unpack_shadow → memcg_id, pgdat, token, workingset.
- memcg = mem_cgroup_from_private_id(memcg_id).
- *lruvec = mem_cgroup_lruvec(memcg, pgdat).
- max_seq = READ_ONCE((*lruvec).lrugen.max_seq).
- max_seq &= (file ? EVICTION_MASK : EVICTION_MASK_ANON) >> LRU_REFS_WIDTH.
- return abs_diff(max_seq, *token >> LRU_REFS_WIDTH) < MAX_NR_GENS.

REQ-17: CONFIG_LRU_GEN — lru_gen_refault(folio, shadow):
- recent = lru_gen_test_recent under rcu.
- if lruvec != folio_lruvec(folio): unlock.
- mod_lruvec_state(lruvec, WORKINGSET_REFAULT_BASE + type, delta).
- if !recent: unlock.
- hist = lru_hist_from_seq(min_seq).
- refs = (token & ((1 << LRU_REFS_WIDTH) - 1)) + 1.
- tier = lru_tier_from_refs(refs, workingset).
- atomic_long_add(delta, &lrugen.refaulted[hist][type][tier]).
- if lru_gen_in_fault(): mod_lruvec_state(WORKINGSET_ACTIVATE_BASE + type, delta).
- if workingset: folio_set_workingset + mod_lruvec_state(WORKINGSET_RESTORE_BASE + type, delta).
- else: set_mask_bits(&folio.flags.f, LRU_REFS_MASK, (refs - 1) << LRU_REFS_PGOFF).

REQ-18: workingset_size formula:
- Base: NR_ACTIVE_FILE.
- If anon page (!file): + NR_INACTIVE_FILE.
- If swap available (mem_cgroup_get_nr_swap_pages > 0): + NR_ACTIVE_ANON.
  - If file page: + NR_INACTIVE_ANON.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pack_unpack_roundtrip` | INVARIANT | per-pack_shadow ∘ unpack_shadow == identity on {memcgid, nid, ws bit}. |
| `eviction_mask_bounded` | INVARIANT | per-pack_shadow: eviction & MASK fits below memcg/nid/ws fields. |
| `nonresident_age_only_inc` | INVARIANT | per-age_nonresident: monotonically non-decreasing. |
| `bucket_order_nonneg` | INVARIANT | per-init: bucket_order ∈ [0, max_order]. |
| `workingset_eviction_preconds` | INVARIANT | per-eviction: !LRU, ref==0, locked. |
| `refault_distance_bounded` | INVARIANT | per-test_recent: distance & mask ⟹ ≤ mask. |
| `workingset_update_node_lock` | INVARIANT | per-update_node: xa_lock held. |
| `shadow_lru_isolate_lock_inversion` | INVARIANT | per-isolate: lru.lock dropped before i_pages.lock acquired (try_lock or retry). |
| `lru_gen_token_width_fits` | INVARIANT | per-lru_gen_eviction: LRU_GEN_WIDTH + LRU_REFS_WIDTH ≤ BITS_PER_LONG - max(EVICTION_SHIFT, EVICTION_SHIFT_ANON). |

### Layer 2: TLA+

`mm/workingset.tla`:
- Per-eviction + per-refault + per-shrinker.
- Properties:
  - `safety_shadow_well_formed` — per-eviction: shadow encodes a valid {nid, memcgid, ws} triple decodable by unpack.
  - `safety_refault_activates_iff_recent` — per-refault: folio_set_active iff refault_distance ≤ workingset_size.
  - `safety_workingset_bit_preserved` — per-(eviction, refault): if folio had workingset-bit at eviction and refault is recent, folio_set_workingset on refault.
  - `safety_shadow_node_lru_membership` — per-update_node: list-LRU membership reflects "all-shadow" predicate.
  - `liveness_shrinker_caps` — per-count_shadow_nodes: shadow_node count bounded by present_pages >> (XA_CHUNK_SHIFT - 3).
  - `liveness_refault_terminates` — per-workingset_refault: every call returns (no unbounded retry).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Workingset::pack_shadow` post: xa_is_value(ret) == true | `Workingset::pack_shadow` |
| `Workingset::unpack_shadow` post: pgdat = NODE_DATA(nid), nid < NR_NODES, memcgid < (1 << MEM_CGROUP_ID_SHIFT) | `Workingset::unpack_shadow` |
| `Workingset::eviction` post: lruvec.nonresident_age incremented by folio_nr_pages | `Workingset::eviction` |
| `Workingset::test_recent` post: ret ⟹ refault_distance ≤ workingset_size | `Workingset::test_recent` |
| `Workingset::refault` post: !test_recent ⟹ !folio_test_active(folio) | `Workingset::refault` |
| `Workingset::refault` post: test_recent ⟹ folio_test_active(folio) ∧ WORKINGSET_ACTIVATE++ | `Workingset::refault` |
| `Workingset::refault` post: test_recent ∧ shadow.workingset ⟹ folio_test_workingset(folio) ∧ WORKINGSET_RESTORE++ | `Workingset::refault` |
| `Workingset::update_node` post: all-shadow ⟺ on shadow_nodes list ⟺ WORKINGSET_NODES counted | `Workingset::update_node` |
| `Workingset::count_shadow_nodes` post: ret ≤ max(0, nodes - max_nodes) | `Workingset::count_shadow_nodes` |
| `Workingset::shadow_lru_isolate` post: LRU_REMOVED_RETRY ⟹ node freed ∧ WORKINGSET_NODERECLAIM++ | `Workingset::shadow_lru_isolate` |

### Layer 4: Verus/Creusot functional

`Per-cache-page lifecycle: cold-fault → inactive-LRU → reclaim (workingset_eviction stores shadow) → refault (workingset_refault unpacks shadow + tests recency) → activated-LRU if refault_distance ≤ workingset_size` semantic equivalence: per-Documentation/admin-guide/mm/workingset.rst and the commentary at the top of mm/workingset.c.

### hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening for page-cache + LRU.)

Workingset-detection reinforcement:

- **Per-shadow xa_value tag (`xa_mk_value`/`xa_is_value`)** — defense against per-shadow-confused-with-folio-pointer dereference.
- **Per-pack_shadow mask before shift** — defense against per-timestamp overflow corrupting memcgid/nid bits.
- **Per-eviction VM_BUG_ON_FOLIO preconditions** — defense against per-aliased-LRU corruption (must be !LRU, ref==0, locked).
- **Per-mem_cgroup_tryget on refault memcg** — defense against per-deleted-memcg-id race (returns false → no activation).
- **Per-refault_distance EVICTION_MASK truncation** — defense against per-wrapping-age false-recent.
- **Per-workingset_size includes anon ⟺ swap free** — defense against per-overcount when swap unavailable.
- **Per-bucket_order saturating_sub** — defense against per-tiny-memory negative shift.
- **Per-shrinker cap pages >> (XA_CHUNK_SHIFT - 3)** — defense against per-malicious-streamer shadow-node DoS.
- **Per-shadow_lru_isolate xa_trylock + try_lock pair with LRU_RETRY** — defense against per-lock-inversion deadlock.
- **Per-update_node lockdep_assert_held(xa_lock)** — defense against per-unsynchronized list_lru mutation.
- **Per-WARN_ON_ONCE in shadow_lru_isolate (node.nr_values, count)** — defense against per-corrupt-node reclaim.
- **Per-shrinker seeks=0 + SHRINKER_NUMA_AWARE | SHRINKER_MEMCG_AWARE** — defense against per-cross-memcg reclaim and per-priority skew.
- **Per-CONFIG_LRU_GEN BUILD_BUG_ON token-width** — defense against per-MGLRU shadow overflow at compile time.

