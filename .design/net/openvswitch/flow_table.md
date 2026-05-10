# Tier-3: net/openvswitch/flow_table.c — OVS flow table (per-mask rhashtable)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/openvswitch/datapath.md
upstream-paths:
  - net/openvswitch/flow_table.c (~1217 lines)
  - net/openvswitch/flow_table.h
  - net/openvswitch/flow.c
-->

## Summary

OVS flow table maps `sw_flow_key` → `sw_flow` for per-skb dispatch. Per-mask `sw_flow_mask` defines per-field wildcards. Per-mask has its own per-mask rhashtable keyed by `(key & mask)`. Lookup walks mask-array; per-mask masked-key lookup; first hit returns flow. Per-mask-cache (`mask_cache_index`) accelerates re-lookup of same skb-class. Per-flow has `key` (canonical), `mask`-pointer, `sf_acts` (action-list). Per-UFID (Userspace Flow IDentifier) optional 128-bit UUID for userspace tracking. Critical for: OVS data-path performance; flow-rule installation/lookup.

This Tier-3 covers `flow_table.c` (~1217 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct flow_table` | per-DP flow-table | `FlowTable` |
| `struct table_instance` | per-rhashtable + bucket-array | `TableInstance` |
| `struct sw_flow_mask` | per-mask (key-mask) | `SwFlowMask` |
| `struct mask_array` | per-DP per-mask list | `MaskArray` |
| `struct mask_cache` | per-CPU cache | `MaskCache` |
| `ovs_flow_tbl_init()` | per-DP init | `FlowTable::init` |
| `ovs_flow_tbl_destroy()` | per-DP destroy | `FlowTable::destroy` |
| `ovs_flow_tbl_insert()` | per-flow insert | `FlowTable::insert` |
| `ovs_flow_tbl_remove()` | per-flow remove | `FlowTable::remove` |
| `ovs_flow_tbl_lookup()` | per-key lookup | `FlowTable::lookup` |
| `ovs_flow_tbl_lookup_stats()` | per-skb lookup with stats | `FlowTable::lookup_stats` |
| `ovs_flow_tbl_lookup_exact()` | per-(key, mask) exact | `FlowTable::lookup_exact` |
| `ovs_flow_tbl_lookup_ufid()` | per-UFID lookup | `FlowTable::lookup_ufid` |
| `flow_mask_find()` | per-mask cache lookup | `FlowTable::mask_find` |
| `tbl_mask_array_alloc()` / `_realloc()` | per-mask-array grow | `MaskArray::alloc` |
| `flow_table_copy_flows()` | per-rehash | `FlowTable::copy_flows` |
| `table_instance_grow()` | per-instance grow | `TableInstance::grow` |
| `tbl_mask_cache_insert()` | per-CPU cache insert | `MaskCache::insert` |
| `rht_replace_fast()` | per-hash update | helper |

## Compatibility contract

REQ-1: Per-flow_table:
- ti: TableInstance — main rhashtable.
- ufid_ti: TableInstance — per-UFID hashtable.
- mask_cache: PerCpu<MaskCache>.
- mask_array: *MaskArray — sorted by usage.

REQ-2: Per-TableInstance:
- buckets: rhashtable_buckets (resizable).
- size: 2^N.
- hash_seed: per-init random.
- node_ver: per-rehash version.

REQ-3: Per-sw_flow_mask:
- key: SwFlowKey bytes (all bytes set per masked field).
- range: { start, end }: byte range where bits set.
- ref_count: AtomicI64.

REQ-4: ovs_flow_tbl_insert(tbl, flow, mask_orig):
- /* Find or insert sw_flow_mask matching mask_orig */
- mask = flow_mask_find(tbl, mask_orig) ∨ alloc_new.
- flow.mask = mask; ref_count++.
- /* Insert into rhashtable per-mask key */
- masked_key = flow.key & mask.key (over mask.range).
- err = rhashtable_insert_fast(tbl.ti, &flow.flow_node, ovs_rht_params).
- /* Insert into UFID hashtable if UFID present */
- if flow.ufid: rhashtable_insert_fast(tbl.ufid_ti, &flow.ufid_node).

REQ-5: ovs_flow_tbl_lookup_stats(tbl, key, skb_hash, n_mask_hit, cache_hit):
- /* Per-CPU mask-cache fast-path */
- mc = this_cpu_ptr(tbl.mask_cache).
- idx = mc_hash(skb_hash).
- entry = mc.cache[idx].
- if entry.skb_hash == skb_hash: mask_idx = entry.mask_index; try-lookup; if hit: *cache_hit=true; return.
- /* Slow-path walk mask-array */
- for mask_idx in 0..mask_array.count:
   - mask = mask_array.masks[mask_idx].
   - masked_key = key & mask.
   - flow = rhashtable_lookup_fast(tbl.ti, &masked_key, ...).
   - if flow: cache-update; *n_mask_hit += 1; return flow.
- Return NULL.

REQ-6: Per-mask sorted by usage:
- mask_array sorted by usage_count descending.
- Per-N-lookups: re-sort to keep hot masks near front.

REQ-7: ovs_flow_tbl_remove:
- rhashtable_remove_fast for flow_node + ufid_node.
- mask.ref_count--; if 0: free mask + remove from mask_array.

REQ-8: Per-mask_cache (per-CPU):
- mask_cache_size: power-of-2 size.
- Per-entry: { skb_hash, mask_index }.
- Per-lookup: hash skb_hash into cache; check + use mask_index hint.

REQ-9: Per-rehash:
- Per-rhashtable_grow on load-factor exceed.
- ovs_rht_params: ovs-specific shrink/grow thresholds.

REQ-10: Per-UFID:
- Per-flow optional 128-bit UUID.
- Per-userspace tracking; per-lookup by UFID directly.

REQ-11: Per-RCU-protected reads:
- ovs_dp_process_packet calls lookup under rcu_read_lock.
- Per-update: rhashtable_replace_fast with rcu_assign_pointer.

## Acceptance Criteria

- [ ] AC-1: ovs_flow_tbl_init: empty tbl + ufid_tbl + mask_cache + mask_array.
- [ ] AC-2: ovs_flow_tbl_insert per-flow: rhashtable populated; mask refcount++.
- [ ] AC-3: ovs_flow_tbl_lookup_stats per-skb-key: returns matching flow.
- [ ] AC-4: Lookup miss + retry with same skb-hash: mask_cache hit; faster.
- [ ] AC-5: 1000-flow insert: rhashtable auto-grows.
- [ ] AC-6: ovs_flow_tbl_remove: flow + ufid node removed.
- [ ] AC-7: ovs_flow_tbl_lookup_ufid: per-UFID returns flow.
- [ ] AC-8: Per-mask refcount 0: mask removed from mask_array.
- [ ] AC-9: Per-RCU-protected lookup: no UAF during concurrent insert.
- [ ] AC-10: Per-CPU mask_cache: per-CPU hot mask remembered.

## Architecture

Per-FlowTable:

```
struct FlowTable {
  ti: AtomicPtr<TableInstance>,
  ufid_ti: AtomicPtr<TableInstance>,
  mask_cache: PerCpu<MaskCache>,
  mask_array: AtomicPtr<MaskArray>,
  count: u32,
  ufid_count: u32,
  last_rehash: u64,
  rh_seed: u32,
}

struct TableInstance {
  buckets: Vec<HListHead>,
  n_buckets: u32,
  node_ver: u8,
  rht: RhashTable,
}

struct SwFlowMask {
  ref_count: AtomicI64,
  rcu: RcuHead,
  range: SwFlowKeyRange,                          // {start, end}
  key: SwFlowKey,
}

struct MaskArray {
  count: u32,
  max: u32,
  rcu: RcuHead,
  masks_usage_zero_cntr: AtomicU64,
  masks_usage_lock: SpinLock,
  masks: Vec<&SwFlowMask>,
  masks_usage_cntrs: Vec<AtomicU64>,
}

struct MaskCache {
  cache: Vec<MaskCacheEntry>,
  cache_size: u32,
}

struct MaskCacheEntry {
  skb_hash: u32,
  mask_index: u32,
}
```

`FlowTable::init(tbl) -> Result<()>`:
1. tbl.ti = table_instance_alloc(TBL_MIN_BUCKETS).
2. tbl.ufid_ti = table_instance_alloc(TBL_MIN_BUCKETS).
3. mask_array = tbl_mask_array_alloc(MASK_ARRAY_SIZE_MIN).
4. mask_cache_size = MC_DEFAULT_SIZE.
5. tbl.mask_cache = alloc_percpu(MaskCache).

`FlowTable::insert(tbl, flow, mask_orig) -> Result<()>`:
1. /* find-or-insert mask */
2. mask = FlowTable::mask_find(tbl, mask_orig).
3. if !mask:
   - mask = kmem_cache_alloc(sw_flow_mask_cache, GFP_KERNEL).
   - mask.range = mask_orig.range; mask.key = mask_orig.key.
   - tbl_mask_array_add_mask(mask_array, mask).
4. flow.mask = mask; atomic_inc(&mask.ref_count).
5. /* Compute masked-key for rhashtable */
6. ti = rcu_dereference(tbl.ti).
7. err = rhashtable_lookup_insert_fast(&ti.rht, &flow.flow_node, ovs_rht_params).
8. if flow.ufid: rhashtable_lookup_insert_fast(&ufid_ti.rht, &flow.ufid_node, ovs_ufid_rht_params).
9. tbl.count++.

`FlowTable::lookup_stats(tbl, key, skb_hash, n_mask_hit, cache_hit) -> *SwFlow`:
1. mc = this_cpu_ptr(tbl.mask_cache).
2. ce_idx = mc_hash(skb_hash, mc.cache_size).
3. ce = &mc.cache[ce_idx].
4. if ce.skb_hash == skb_hash:
   - mask = mask_array.masks[ce.mask_index].
   - flow = FlowTable::masked_lookup(tbl, key, mask).
   - if flow: *cache_hit = true; return flow.
5. /* Slow-path */
6. for mi in 0..mask_array.count:
   - mask = mask_array.masks[mi].
   - flow = FlowTable::masked_lookup(tbl, key, mask).
   - if flow:
     - *n_mask_hit += 1.
     - /* Update cache */
     - ce.skb_hash = skb_hash; ce.mask_index = mi.
     - return flow.
7. return NULL.

`FlowTable::masked_lookup(tbl, key, mask) -> *SwFlow`:
1. masked_key = key & mask.key (over mask.range bytes).
2. ti = rcu_dereference(tbl.ti).
3. return rhashtable_lookup_fast(&ti.rht, &masked_key, ovs_rht_params).

`FlowTable::remove(tbl, flow) -> Result<()>`:
1. ti = rcu_dereference(tbl.ti).
2. rhashtable_remove_fast(&ti.rht, &flow.flow_node, ovs_rht_params).
3. if flow.ufid: rhashtable_remove_fast(&ufid_ti.rht, &flow.ufid_node, ...).
4. atomic_dec(&flow.mask.ref_count).
5. if flow.mask.ref_count == 0:
   - tbl_mask_array_remove(tbl.mask_array, flow.mask).
   - kfree_rcu(flow.mask, rcu).
6. tbl.count--.

`FlowTable::mask_find(tbl, mask_orig) -> *SwFlowMask`:
1. for mi in 0..mask_array.count:
   - mask = mask_array.masks[mi].
   - if mask.range == mask_orig.range ∧ memcmp(&mask.key, &mask_orig.key, mask.range.bytes) == 0:
     - return mask.
2. return NULL.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `mask_array_count_le_max` | INVARIANT | mask_array.count ≤ mask_array.max. |
| `flow_count_eq_rhashtable` | INVARIANT | tbl.count == # nodes in tbl.ti.rht. |
| `mask_refcount_eq_flows_using` | INVARIANT | per-mask.ref_count == #flows with this mask. |
| `lookup_under_rcu_read` | INVARIANT | per-lookup_stats: rcu_read_lock held. |
| `mask_cache_idx_lt_size` | INVARIANT | per-ce_idx < mc.cache_size. |

### Layer 2: TLA+

`net/openvswitch/flow_table.tla`:
- Per-flow insert + per-key lookup + per-mask refcount.
- Properties:
  - `safety_lookup_returns_inserted` — per-(key matching mask) inserted flow ⟹ lookup returns it.
  - `safety_no_lookup_after_remove` — per-removed flow not returned by lookup.
  - `safety_mask_freed_when_zero_refs` — per-mask.ref_count == 0 ⟹ removed from array.
  - `liveness_mask_cache_eventually_hits` — per-repeated-skb-class lookup: cache hits.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `FlowTable::insert` post: flow in ti.rht; mask refcount++ | `FlowTable::insert` |
| `FlowTable::lookup_stats` post: returned-flow matches mask × key | `FlowTable::lookup_stats` |
| `FlowTable::remove` post: flow gone from ti.rht; mask refcount-- | `FlowTable::remove` |
| `FlowTable::mask_find` post: returns matching mask or NULL | `FlowTable::mask_find` |

### Layer 4: Verus/Creusot functional

`Per-flow tuple-space search: walk per-mask → per-(masked-key) rhashtable lookup → first match wins; per-CPU mask-cache for hot-skb-class` semantic equivalence: per-OVS megaflow lookup algorithm.

## Hardening

(Inherits row-1 features from `net/openvswitch/datapath.md` § Hardening.)

Flow-table-specific reinforcement:

- **Per-mask refcount atomic** — defense against per-shared-mask UAF.
- **Per-rhashtable RCU-read protect** — defense against per-lookup mid-rehash UAF.
- **Per-mask_array bounded by max** — defense against per-mask-bomb.
- **Per-CPU mask_cache local** — defense against per-CPU contention.
- **Per-table_instance grow under update-lock** — defense against per-rehash-race.
- **Per-UFID rhashtable separate** — defense against per-UFID-collision affecting flow-lookup.
- **Per-ovs_rht_params (custom rhash-params)** — defense against per-default-params imbalance.
- **Per-replace via rcu_assign_pointer** — defense against per-read torn-replace.
- **Per-init mask_cache_size power-of-2** — defense against per-cache-index UB.
- **Per-lookup_exact distinct from lookup_stats** — defense against per-control-vs-data-plane confusion.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- OVS datapath (covered in `datapath.md` Tier-3)
- net/openvswitch/flow.c (covered separately if expanded)
- rhashtable (covered in lib/ separately)
- Implementation code
