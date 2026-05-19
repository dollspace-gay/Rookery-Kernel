# Tier-3: kernel/cgroup/dmem.c — dmem cgroup (DRM/GPU device-memory limits)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/cgroup/00-overview.md
upstream-paths:
  - kernel/cgroup/dmem.c (~885 lines)
  - include/linux/cgroup_dmem.h
  - include/linux/page_counter.h
  - Documentation/admin-guide/cgroup-v2.rst (dmem section)
-->

## Summary

The **dmem** cgroup limits per-cgroup usage of device-local memory exposed by DRM / GPU drivers (VRAM, TT, stolen, etc.) — analogous to the memcg controller but indexed by **named regions** registered at driver bind time, not by NUMA nodes. DRM drivers call `dmem_cgroup_register_region(size, fmt, ...)` once per memory region (e.g. `"drm/card0/vram0"`); each region carries a refcounted, RCU-protected name + capacity + a list of per-cgroup `dmem_cgroup_pool_state` page_counters. On GEM/TTM allocation a driver calls `dmem_cgroup_try_charge(region, size, &pool, &limit_pool)`; on success the returned pool must be released via `dmem_cgroup_uncharge(pool, size)`. On hitting a limit (-EAGAIN) the limit_pool is returned for the driver's eviction policy to consult via `dmem_cgroup_state_evict_valuable(limit_pool, test_pool, ignore_low, &hit_low)`, which evaluates min/low watermarks (`page_counter_calculate_protection`) hierarchically to decide whether the test_pool is fair game for eviction. Per-cgroup file surface: `dmem.capacity` (root, per-region totals), `dmem.current`, `dmem.min`, `dmem.low`, `dmem.max` — using `region_name <value|max>` newline-separated entries. Hierarchy: pools chain through `cnt.parent` so child charges propagate to ancestors via the `page_counter` machinery; pools are created lazily on first reference and initialized top-down via `get_cg_pool_locked` walking up the css tree, then fixing parent links + marking inited. Critical for: per-tenant VRAM fairness in DRM scheduling, multi-user GPU sharing, container-isolated VRAM quotas, eviction policy fairness in TTM/GEM pools.

This Tier-3 covers `kernel/cgroup/dmem.c` (~885 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct dmem_cgroup_region` | per-DRM-region descriptor (size + name + pools) | `Dmem::Region` |
| `struct dmemcg_state` | per-cgroup state (css + pool list) | `Dmem::State` |
| `struct dmem_cgroup_pool_state` | per-(cgroup, region) page_counter | `Dmem::PoolState` |
| `dmem_cgroup_register_region()` | per-driver region register | `Dmem::register_region` |
| `dmem_cgroup_unregister_region()` | per-driver region unregister | `Dmem::unregister_region` |
| `dmem_cgroup_try_charge()` | per-allocation charge | `Dmem::try_charge` |
| `dmem_cgroup_uncharge()` | per-free uncharge | `Dmem::uncharge` |
| `dmem_cgroup_pool_state_put()` | per-limit-pool ref drop | `Dmem::pool_state_put` |
| `dmem_cgroup_state_evict_valuable()` | per-eviction decision | `Dmem::state_evict_valuable` |
| `dmem_cgroup_calculate_protection()` | per-hierarchical min/low calc | `Dmem::calculate_protection` |
| `dmemcs_alloc/free/offline` | per-css lifecycle | `Dmem::css_*` |
| `alloc_pool_single()` | per-(cgroup, region) pool alloc | `Dmem::alloc_pool_single` |
| `get_cg_pool_locked()` | per-recursive pool create + init | `Dmem::get_cg_pool_locked` |
| `get_cg_pool_unlocked()` | per-rcu-fast-path + slowpath | `Dmem::get_cg_pool_unlocked` |
| `find_cg_pool_locked()` | per-rcu-list lookup | `Dmem::find_cg_pool_locked` |
| `pool_parent()` | per-page_counter parent unwrap | `Dmem::pool_parent` |
| `dmemcg_pool_get/tryget/put` | per-pool refcount | `Dmem::PoolState::ref_*` |
| `dmemcg_free_region()` | per-region kref release | `Dmem::free_region` |
| `dmemcg_pool_free_rcu()` | per-pool RCU free | `Dmem::pool_free_rcu` |
| `dmemcg_get_region_by_name()` | per-cfg-file region lookup | `Dmem::get_region_by_name` |
| `dmemcg_limit_write()` / `_show()` | per-cfg file r/w | `Dmem::limit_write` / `limit_show` |
| `dmemcg_parse_limit()` | per-value parser ("max" or memparse) | `Dmem::parse_limit` |
| `set_resource_min/low/max` | per-page_counter setters | `Dmem::set_resource_*` |
| `get_resource_min/low/max/current` | per-page_counter getters | `Dmem::get_resource_*` |
| `reset_all_resource_limits()` | per-offline reset | `Dmem::reset_all_resource_limits` |
| `dmem_cgrp_subsys` | per-cgroup_subsys vtable | `Dmem::SUBSYS` |
| `dmem_cgroup_regions` (list_head) | per-global region list | `Dmem::REGIONS` |
| `dmemcg_lock` (spinlock) | per-region-list + per-pool-list lock | `Dmem::LOCK` |

## Compatibility contract

REQ-1: struct dmem_cgroup_region:
- ref: kref keeping region alive across RCU lookups.
- rcu: rcu_head for deferred free.
- region_node: list_head into global `dmem_cgroup_regions` list (RCU + dmemcg_lock).
- pools: list_head of all per-cgroup pools binding to this region (dmemcg_lock only).
- size: region capacity in bytes.
- name: kvasprintf-allocated identifier (e.g. `"drm/card0/vram"`).
- unregistered: bool latched true by `unregister_region`; blocks new pool creation.

REQ-2: struct dmemcg_state:
- css: standard cgroup_subsys_state.
- pools: list_head of pools owned by this cgroup (RCU on read, dmemcg_lock on mutate).

REQ-3: struct dmem_cgroup_pool_state:
- region: back-pointer to dmem_cgroup_region (with kref).
- cs: back-pointer to dmemcg_state.
- css_node: list_node into dmemcs.pools (RCU + dmemcg_lock).
- region_node: list_node into region.pools (dmemcg_lock only).
- rcu: rcu_head.
- cnt: struct page_counter (current, max, low, min, emin, elow, parent).
- parent: parent pool (refcount-held).
- ref: refcount_t lifecycle.
- inited: bool — false until parent chain wired up, gates fast-path use.

REQ-4: dmem_cgroup_register_region(size, fmt, ...):
- if !size: return NULL.
- region_name = kvasprintf(GFP_KERNEL, fmt, ap); on alloc fail -ENOMEM.
- region = kzalloc; on fail kfree(region_name) + -ENOMEM.
- INIT_LIST_HEAD(&region->pools); region->name = region_name; region->size = size; kref_init(&region->ref).
- Under dmemcg_lock: list_add_tail_rcu(&region->region_node, &dmem_cgroup_regions).
- Returns region pointer (NULL allowed iff size == 0).

REQ-5: dmem_cgroup_unregister_region(region):
- Tolerate NULL.
- Under dmemcg_lock:
  - list_del_rcu(&region->region_node).
  - For each pool in region->pools: list_del_rcu(&pool->css_node); list_del(&pool->region_node); dmemcg_pool_put(pool).
  - region->unregistered = true.
- kref_put(&region->ref, dmemcg_free_region) — drops driver's initial reference.

REQ-6: dmem_cgroup_try_charge(region, size, ret_pool, ret_limit_pool):
- *ret_pool = NULL; if ret_limit_pool: *ret_limit_pool = NULL.
- cg = get_current_dmemcs() (task_get_css holds css ref).
- pool = get_cg_pool_unlocked(cg, region); IS_ERR ⟹ ret = PTR_ERR; goto err.
- If !page_counter_try_charge(&pool->cnt, size, &fail):
  - if ret_limit_pool: *ret_limit_pool = container_of(fail, pool_state, cnt); css_get(*ret_limit_pool->cs->css); dmemcg_pool_get(*ret_limit_pool).
  - dmemcg_pool_put(pool); ret = -EAGAIN; goto err.
- *ret_pool = pool; return 0 (css ref transferred to *ret_pool).
- err: css_put(&cg->css); return ret.

REQ-7: dmem_cgroup_uncharge(pool, size):
- if !pool: return.
- page_counter_uncharge(&pool->cnt, size).
- css_put(&pool->cs->css).
- dmemcg_pool_put(pool).

REQ-8: dmem_cgroup_pool_state_put(pool):
- if !pool: return.
- css_put(&pool->cs->css); dmemcg_pool_put(pool).
- Used to release *ret_limit_pool from try_charge after eviction.

REQ-9: dmem_cgroup_state_evict_valuable(limit_pool, test_pool, ignore_low, ret_hit_low):
- if limit_pool == test_pool: return true (always evict from current).
- if limit_pool:
  - if !parent_dmemcs(limit_pool->cs): return true (root-cgroup limit ⟹ no protection).
  - Walk test_pool ancestors searching for limit_pool; not found ⟹ return false (test_pool not in subtree).
- else:
  - limit_pool = root pool of test_pool's hierarchy.
- dmem_cgroup_calculate_protection(limit_pool, test_pool).
- used = page_counter_read(&test_pool->cnt); min = READ_ONCE(test_pool->cnt.emin).
- if used <= min: return false.
- if !ignore_low:
  - low = READ_ONCE(test_pool->cnt.elow).
  - if used > low: return true.
  - *ret_hit_low = true; return false.
- return true.

REQ-10: dmem_cgroup_calculate_protection(limit_pool, test_pool):
- climit = &limit_pool->cnt.
- rcu_read_lock.
- css_for_each_descendant_pre(css, &limit_pool->cs->css):
  - dmemcg_iter = container_of(css, dmemcg_state, css).
  - Find pool in dmemcg_iter->pools where pool->region == limit_pool->region.
  - If found: page_counter_calculate_protection(climit, &found_pool->cnt, true).
  - If found_pool == test_pool: break.
- rcu_read_unlock.

REQ-11: get_cg_pool_unlocked(cg, region):
- /* Fastpath */
- rcu_read_lock; pool = find_cg_pool_locked(cg, region); if pool ∧ !READ_ONCE(pool->inited): pool = NULL; if pool ∧ !dmemcg_pool_tryget(pool): pool = NULL. rcu_read_unlock.
- /* Slowpath loop */
- while !pool:
  - spin_lock(&dmemcg_lock).
  - if !region->unregistered: pool = get_cg_pool_locked(cg, region, &allocpool); else pool = -ENODEV.
  - if !IS_ERR(pool): dmemcg_pool_get(pool).
  - spin_unlock.
  - if pool == -ENOMEM:
    - WARN_ON(allocpool); allocpool = kzalloc_obj(GFP_NOWAIT-ish); if allocpool: pool = NULL; continue.
- kfree(allocpool); return pool.

REQ-12: get_cg_pool_locked(dmemcs, region, allocpool):
- Recursive pool creation walking up cgroup tree:
  - For p in dmemcs, parent, grandparent, ...:
    - pool = find_cg_pool_locked(p, region); else alloc_pool_single(p, region, allocpool).
    - if IS_ERR: return.
    - if p == dmemcs ∧ pool.inited: return pool.
    - if pool.inited: break.
- Second walk: fix up parent links + mark inited bottom-up from p down to dmemcs:
  - retpool = find_cg_pool_locked(dmemcs, region).
  - For p = dmemcs, pp = parent(dmemcs); pp; p = pp, pp = parent(p):
    - if pool.inited: break.
    - ppool = find_cg_pool_locked(pp, region).
    - pool.cnt.parent = &ppool.cnt.
    - if ppool ∧ !pool.parent: pool.parent = ppool; dmemcg_pool_get(ppool).
    - pool.inited = true.
    - pool = ppool.
- return retpool.

REQ-13: alloc_pool_single(dmemcs, region, allocpool):
- if !*allocpool: pool = kzalloc(GFP_NOWAIT) else pool = *allocpool; *allocpool = NULL.
- pool.region = region; pool.cs = dmemcs.
- parent = parent_dmemcs(dmemcs); if parent: ppool = find_cg_pool_locked(parent, region).
- page_counter_init(&pool.cnt, ppool ? &ppool.cnt : NULL, /*protection-tracking=*/true).
- reset_all_resource_limits(pool) — min=0, low=0, max=PAGE_COUNTER_MAX.
- refcount_set(&pool.ref, 1); kref_get(&region->ref).
- if ppool ∧ !pool.parent: pool.parent = ppool; dmemcg_pool_get(ppool).
- list_add_tail_rcu(&pool.css_node, &dmemcs.pools); list_add_tail(&pool.region_node, &region.pools).
- if !parent: pool.inited = true (root); else pool.inited = ppool.inited.

REQ-14: dmemcg_pool_get / tryget / put:
- get: refcount_inc(&pool->ref).
- tryget: refcount_inc_not_zero(&pool->ref).
- put: refcount_dec_and_test ⟹ call_rcu(&pool->rcu, dmemcg_pool_free_rcu).

REQ-15: dmemcg_pool_free_rcu(rcu):
- pool = container_of.
- if pool.parent: dmemcg_pool_put(pool.parent) — release parent chain.
- kref_put(&pool->region->ref, dmemcg_free_region).
- kfree(pool).

REQ-16: dmemcg_free_region(ref) → call_rcu(&region->rcu, dmemcg_free_rcu); dmemcg_free_rcu:
- For each pool in region.pools: free_cg_pool(pool) = list_del(region_node) + dmemcg_pool_put.
- kfree(region->name); kfree(region).

REQ-17: dmemcs_offline(css):
- Under rcu_read_lock, list_for_each_entry_rcu(pool, &dmemcs.pools, css_node): reset_all_resource_limits(pool).
- Drops accounting limits but preserves pool refcounts for in-flight charges.

REQ-18: dmemcs_free(css):
- Under dmemcg_lock: list_for_each_entry_safe(pool, &dmemcs.pools, css_node): list_del(&pool.css_node); free_cg_pool(pool).
- kfree(dmemcs).

REQ-19: cftype files[]:
- "capacity": ONLY_ON_ROOT; seq_show iterates `dmem_cgroup_regions` and prints `name size` per region.
- "current": seq_show via dmemcg_limit_show + get_resource_current — `name <bytes>` per region.
- "min": NOT_ON_ROOT; write via set_resource_min, show via get_resource_min.
- "low": NOT_ON_ROOT; write via set_resource_low, show via get_resource_low.
- "max": NOT_ON_ROOT; write via set_resource_max, show via get_resource_max ("max" if PAGE_COUNTER_MAX).

REQ-20: dmemcg_limit_write(of, buf, nbytes, off, apply):
- For each newline-delimited line (modifies buf in place):
  - Strip whitespace; skip empty.
  - region_name = strsep(&options, " \t").
  - if !options || !*options: return -EINVAL.
  - region = dmemcg_get_region_by_name(region_name) under rcu_read_lock; !region ⟹ -EINVAL.
  - parse_limit(options, &new_limit) — "max" → PAGE_COUNTER_MAX; else memparse + end-check.
  - pool = get_cg_pool_unlocked(dmemcs, region); IS_ERR ⟹ -PTR_ERR.
  - apply(pool, new_limit); dmemcg_pool_put(pool).
  - kref_put(&region->ref, dmemcg_free_region).
- Return err ?: nbytes.

REQ-21: dmem_cgrp_subsys vtable:
- css_alloc = dmemcs_alloc; css_free = dmemcs_free; css_offline = dmemcs_offline.
- legacy_cftypes = files; dfl_cftypes = files (both v1 + v2 surface).

## Acceptance Criteria

- [ ] AC-1: DRM driver calls `dmem_cgroup_register_region(SZ_16G, "drm/card0/vram")`: returns non-NULL; "capacity" file in root shows `drm/card0/vram 17179869184`.
- [ ] AC-2: `echo "drm/card0/vram 1G" > /sys/fs/cgroup/X/dmem.max`: max = 1 << 30 for pool(X, region).
- [ ] AC-3: `echo "drm/card0/vram max" > .../dmem.max`: max = PAGE_COUNTER_MAX.
- [ ] AC-4: `dmem_cgroup_try_charge(region, 512M)` after max=1G: succeeds for first two charges; third returns -EAGAIN with limit_pool == X-pool.
- [ ] AC-5: `dmem_cgroup_uncharge(pool, 512M)`: page_counter decremented; css_put + pool_put applied.
- [ ] AC-6: Charges in child cgroup propagate to ancestor pools via cnt.parent chain.
- [ ] AC-7: Setting min on parent + low on child: state_evict_valuable correctly returns false until child usage > emin and respects low watermark.
- [ ] AC-8: `dmemcs_offline` resets min/low/max to (0,0,PAGE_COUNTER_MAX) for all pools.
- [ ] AC-9: `dmem_cgroup_unregister_region`: marks region->unregistered; subsequent try_charge for that region returns -ENODEV.
- [ ] AC-10: Concurrent register + unregister + charge under RCU: no use-after-free; lockdep clean.
- [ ] AC-11: Writing limit for non-existent region name: -EINVAL.
- [ ] AC-12: `dmem.capacity` is ONLY_ON_ROOT; `dmem.min/low/max` are NOT_ON_ROOT.
- [ ] AC-13: Root cgroup `dmem.current` shows aggregate across hierarchy.

## Architecture

```
struct DmemRegion {
  ref: KRef,
  rcu: RcuHead,
  region_node: ListNode,                 // into REGIONS
  pools: ListHead,                       // PoolState.region_node
  size: u64,
  name: KBox<str>,                       // kvasprintf
  unregistered: bool,
}

struct DmemState {
  css: CgroupSubsysState,
  pools: ListHead,                       // PoolState.css_node
}

struct DmemPoolState {
  region: *DmemRegion,
  cs: *DmemState,
  css_node: ListNode,
  region_node: ListNode,
  rcu: RcuHead,
  cnt: PageCounter,
  parent: Option<*DmemPoolState>,
  ref: RefCount,
  inited: bool,
}

static LOCK: SpinLock<()> = ...;
static REGIONS: ListHead = ...;
```

`Dmem::register_region(size, fmt, ...) -> *Region or ErrPtr`:
1. if size == 0: return NULL.
2. name = kvasprintf(GFP_KERNEL, fmt, args); !name ⟹ -ENOMEM.
3. region = kzalloc; !region ⟹ kfree(name) + -ENOMEM.
4. INIT_LIST_HEAD(&region.pools); region.name = name; region.size = size; kref_init(&region.ref).
5. Under LOCK: list_add_tail_rcu(&region.region_node, &REGIONS).
6. Return region.

`Dmem::unregister_region(region)`:
1. !region ⟹ return.
2. Under LOCK:
   - list_del_rcu(&region.region_node).
   - For each pool in region.pools: list_del_rcu(&pool.css_node); list_del(&pool.region_node); pool_put(pool).
   - region.unregistered = true.
3. kref_put(&region.ref, free_region) — defer-free via call_rcu.

`Dmem::try_charge(region, size, ret_pool, ret_limit_pool) -> i32`:
1. *ret_pool = NULL; if ret_limit_pool: *ret_limit_pool = NULL.
2. cg = get_current_dmemcs() — task_get_css holds css ref.
3. pool = Self::get_cg_pool_unlocked(cg, region); IS_ERR ⟹ ret = PTR_ERR; goto err.
4. If !page_counter_try_charge(&pool.cnt, size, &fail):
   - If ret_limit_pool: *ret_limit_pool = container_of(fail, PoolState, cnt); css_get(*ret_limit_pool.cs.css); pool_get(*ret_limit_pool).
   - pool_put(pool); ret = -EAGAIN; goto err.
5. *ret_pool = pool; return 0.
6. err: css_put(&cg.css); return ret.

`Dmem::uncharge(pool, size)`:
1. !pool ⟹ return.
2. page_counter_uncharge(&pool.cnt, size).
3. css_put(&pool.cs.css).
4. pool_put(pool).

`Dmem::state_evict_valuable(limit_pool, test_pool, ignore_low, ret_hit_low) -> bool`:
1. limit_pool == test_pool ⟹ return true.
2. If limit_pool:
   - parent_dmemcs(limit_pool.cs) is None ⟹ return true.
   - Walk test_pool ancestors via pool_parent; if limit_pool not found ⟹ return false.
3. Else: limit_pool = root pool of test_pool's chain via pool_parent.
4. Self::calculate_protection(limit_pool, test_pool).
5. used = page_counter_read(&test_pool.cnt); min = test_pool.cnt.emin (READ_ONCE).
6. used <= min ⟹ return false.
7. !ignore_low:
   - low = test_pool.cnt.elow (READ_ONCE).
   - used > low ⟹ return true.
   - *ret_hit_low = true; return false.
8. return true.

`Dmem::calculate_protection(limit_pool, test_pool)`:
1. climit = &limit_pool.cnt.
2. rcu_read_lock.
3. css_for_each_descendant_pre(css, &limit_pool.cs.css):
   - dmemcg_iter = container_of(css, DmemState, css).
   - found_pool = scan dmemcg_iter.pools for pool.region == limit_pool.region.
   - If found_pool: page_counter_calculate_protection(climit, &found_pool.cnt, true).
   - found_pool == test_pool ⟹ break.
4. rcu_read_unlock.

`Dmem::get_cg_pool_unlocked(cg, region) -> *PoolState`:
1. Fastpath: rcu_read_lock; pool = find_cg_pool_locked(cg, region); !pool.inited ⟹ pool = NULL; !pool_tryget(pool) ⟹ pool = NULL. rcu_read_unlock.
2. Slowpath loop:
   - Under LOCK: !region.unregistered ⟹ pool = get_cg_pool_locked(cg, region, &allocpool); else pool = -ENODEV.
   - !IS_ERR(pool) ⟹ pool_get(pool).
   - Drop LOCK.
   - pool == -ENOMEM ⟹ WARN_ON(allocpool); allocpool = kzalloc; if alloc'd: pool = NULL; continue.
3. kfree(allocpool); return pool.

`Dmem::get_cg_pool_locked(dmemcs, region, allocpool) -> *PoolState`:
1. /* Top-down ensure-exists walk */
2. for p = dmemcs; p; p = parent_dmemcs(p):
   - pool = find_cg_pool_locked(p, region); else pool = Self::alloc_pool_single(p, region, allocpool).
   - IS_ERR(pool) ⟹ return.
   - p == dmemcs ∧ pool.inited ⟹ return pool.
   - pool.inited ⟹ break.
3. /* Bottom-up fix parent links + mark inited */
4. retpool = pool = find_cg_pool_locked(dmemcs, region).
5. for p = dmemcs, pp = parent_dmemcs(dmemcs); pp; p = pp, pp = parent_dmemcs(p):
   - pool.inited ⟹ break.
   - ppool = find_cg_pool_locked(pp, region).
   - pool.cnt.parent = &ppool.cnt.
   - ppool ∧ !pool.parent ⟹ pool.parent = ppool; pool_get(ppool).
   - pool.inited = true.
   - pool = ppool.
6. return retpool.

`Dmem::alloc_pool_single(dmemcs, region, allocpool) -> *PoolState`:
1. pool = *allocpool ?: kzalloc(GFP_NOWAIT); err -ENOMEM.
2. pool.region = region; pool.cs = dmemcs.
3. parent = parent_dmemcs(dmemcs); ppool = parent ? find_cg_pool_locked(parent, region) : NULL.
4. page_counter_init(&pool.cnt, ppool ? &ppool.cnt : NULL, true).
5. reset_all_resource_limits(pool).
6. refcount_set(&pool.ref, 1); kref_get(&region.ref).
7. ppool ∧ !pool.parent ⟹ pool.parent = ppool; pool_get(ppool).
8. list_add_tail_rcu(&pool.css_node, &dmemcs.pools); list_add_tail(&pool.region_node, &region.pools).
9. !parent ⟹ pool.inited = true; else pool.inited = ppool.inited.

`Dmem::limit_write(of, buf, nbytes, off, apply) -> isize`:
1. While buf non-empty + no err:
   - options = buf; buf = strchr(buf, '\n'); if buf: *buf++ = 0.
   - options = strstrip(options); skip if empty.
   - region_name = strsep(&options, " \t").
   - !options || !*options ⟹ -EINVAL.
   - Under rcu_read_lock: region = Self::get_region_by_name(region_name).
   - !region ⟹ -EINVAL.
   - Self::parse_limit(options, &new_limit); err ⟹ goto out_put.
   - pool = Self::get_cg_pool_unlocked(dmemcs, region); IS_ERR ⟹ err.
   - apply(pool, new_limit); pool_put(pool).
   - out_put: kref_put(&region.ref, free_region).
2. Return err ?: nbytes.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `region_kref_balanced` | INVARIANT | per-register/get_by_name/unregister: kref get == kref put. |
| `pool_refcount_balanced` | INVARIANT | per-try_charge success: pool ref taken; per-uncharge: dropped. |
| `pool_init_before_use` | INVARIANT | per-fastpath: returned pool.inited == true. |
| `region_unregistered_blocks_new_pools` | INVARIANT | per-get_cg_pool_unlocked: region.unregistered ⟹ -ENODEV. |
| `pool_parent_chain_acyclic` | INVARIANT | per-pool: parent chain terminates at root, no cycles. |
| `evict_current_pool_always_true` | INVARIANT | per-state_evict_valuable: limit_pool == test_pool ⟹ true. |
| `evict_below_emin_false` | INVARIANT | per-state_evict_valuable: used <= emin ⟹ false. |
| `try_charge_failure_pool_put` | INVARIANT | per-try_charge -EAGAIN: pool ref released; css_put on err. |
| `limit_write_atomicity` | INVARIANT | per-limit_write: failed line does not partially apply. |
| `dmemcg_lock_protects_region_list_mutations` | INVARIANT | per-list_add_tail_rcu/list_del_rcu on REGIONS: LOCK held. |

### Layer 2: TLA+

`kernel/cgroup/dmem.tla`:
- Per-region-lifecycle: register → in-use → unregister → rcu-free.
- Per-pool-lifecycle: alloc → init → in-use → unregister-or-cgroup-offline → rcu-free.
- Per-charge: try_charge → page_counter_try_charge → success-or-EAGAIN(limit_pool).
- Per-evict: state_evict_valuable evaluating (emin, elow) hierarchy.
- Properties:
  - `safety_region_not_freed_with_live_pool` — per-region: free only after all pools dropped.
  - `safety_pool_not_freed_with_live_charge` — per-pool: free only after charge == 0.
  - `safety_inited_implies_parent_inited_or_no_parent` — per-pool: inited ⟹ (parent == NULL ∨ parent.inited).
  - `safety_unregister_blocks_new_pools` — per-region: unregistered ⟹ no new pool entries on its list.
  - `liveness_eviction_when_max_hit` — per-overcommit: state_evict_valuable eventually true for some pool in subtree.
  - `liveness_rcu_free_progresses` — per-call_rcu: eventually freed.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Dmem::register_region` post: region in REGIONS, kref==1 | `Dmem::register_region` |
| `Dmem::unregister_region` post: region.unregistered ∧ no pools on its list | `Dmem::unregister_region` |
| `Dmem::try_charge` post: ret==0 ⟹ *ret_pool != NULL ∧ pool.cnt incremented by size | `Dmem::try_charge` |
| `Dmem::try_charge` post: ret==-EAGAIN ⟹ *ret_limit_pool != NULL ∨ ret_limit_pool == NULL | `Dmem::try_charge` |
| `Dmem::uncharge` post: pool.cnt decremented by size; pool ref dropped | `Dmem::uncharge` |
| `Dmem::state_evict_valuable` post: result reflects emin/elow per REQ-9 | `Dmem::state_evict_valuable` |
| `Dmem::get_cg_pool_locked` post: returned pool.inited == true | `Dmem::get_cg_pool_locked` |
| `Dmem::alloc_pool_single` post: pool linked into both css_node + region_node lists | `Dmem::alloc_pool_single` |
| `Dmem::limit_write` post: every successful line applied; failure ⟹ remainder untouched | `Dmem::limit_write` |
| `Dmem::reset_all_resource_limits` post: min=0, low=0, max=PAGE_COUNTER_MAX | `Dmem::reset_all_resource_limits` |

### Layer 4: Verus/Creusot functional

`Per-DRM-driver register_region → driver-allocates(size) → try_charge(region, size) → page_counter_try_charge → success-or-EAGAIN(limit_pool) → state_evict_valuable(limit_pool, candidate) → driver-eviction-frees-buffer → uncharge` semantic equivalence: per-Documentation/admin-guide/cgroup-v2.rst (dmem section) + per-include/linux/cgroup_dmem.h API contract + per-page_counter min/low/max protection semantics.

## Hardening

(Inherits row-1 features from `kernel/cgroup/00-overview.md` § Hardening.)

dmem reinforcement:

- **Per-region kref + RCU free** — defense against per-region UAF when driver unregisters while a charge is in flight.
- **Per-pool refcount + call_rcu free** — defense against per-pool UAF during concurrent eviction lookup.
- **Per-region->unregistered latch** — defense against per-new-pool-on-dead-region.
- **Per-pool inited flag before fastpath** — defense against per-uninited-parent dangling parent pointer.
- **Per-css_put on try_charge failure** — defense against per-css refcount leak.
- **Per-page_counter_try_charge atomic** — defense against per-double-spend on concurrent charges.
- **Per-page_counter protection (min/low/elow/emin)** — defense against per-tenant-starvation eviction.
- **Per-RCU read of REGIONS** — defense against per-list mutation during enumeration.
- **Per-dmemcg_lock global covering region-list + pool-list mutations** — defense against per-lock-ordering bugs (single lock).
- **Per-pool parent ref held until pool_put** — defense against per-parent free-while-child-charging.
- **Per-name kvasprintf bounded by GFP_KERNEL** — defense against per-OOM during region register.
- **Per-config-file write-atomic per line** — defense against per-partial-config corruption.
- **Per-strscpy/memparse + end-pointer check** — defense against per-malformed user input parse confusion.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — `dmem.max` / `dmem.min` / `dmem.low` write handlers copy attacker-supplied size strings; whitelist `PAGE_SIZE - 1` and trap on any oversized write before `memparse` runs.
- **PAX_KERNEXEC** — DRM/GPU memory controller cftypes indirect-call vectors live in `.rodata`; refuse rwx so a kernel-write primitive cannot rewrite `dmem_charge` to skip the per-region quota check.
- **PAX_RANDKSTACK** — `dmem_try_charge` / `dmem_uncharge` fire from GPU driver allocation paths (TTM, GEM) at user-driven depth via ioctl; randomized kstack offset disrupts ROP through the page_counter chain.
- **PAX_REFCOUNT** — `dmemcg_state->page_counter` family and per-region `dmem_region->ref` are refcount_t with saturating overflow; defense against ioctl-storm underflow racing `dmemcs_offline` against in-flight charge.
- **PAX_MEMORY_SANITIZE** — `dmemcs_free` must zero per-region counter array and `dmem.events` buffer before kfree; defense against post-free residual exposing GPU-allocation telemetry to a fresh cgroup.
- **PAX_UDEREF** — `memparse` of size strings keeps SMAP/PAN engaged across the whole `cftype->write` → `dmem_max_write` → `page_counter_set_max` chain.
- **PAX_RAP/kCFI** — `dmem_cgrp_subsys.*` ops vector (`css_alloc`, `css_offline`, `css_free`) must verify kCFI tag at indirect-call sites in cgroup core.
- **GRKERNSEC_HIDESYM** — `dmem.current` / `dmem.max` readers expose counter values, not kernel pointers; refuse to leak the `struct dmem_region *` address through any error path or fdinfo.
- **GRKERNSEC_DMESG** — `WARN_ON` paths in `dmem_region_register` (kvasprintf failure, end-pointer check) must not splat raw pointers into dmesg readable by non-CAP_SYSLOG.
- **DRM/GPU memory CAP_SYS_ADMIN gate** — `dmem.max` / `dmem.min` write must enforce `ns_capable(user_ns, CAP_SYS_ADMIN)` against the userns owning the cgroup; defense against per-userns dmem cgroup granting GPU-memory limit manipulation outside operator policy.
- **Per-cgroup limit ceiling** — Rookery imposes a host-wide upper bound on the sum of dmem.max across cgroups (configured via sysctl) so a CAP_SYS_ADMIN-in-container cannot reserve >100% of GPU VRAM and starve sibling containers.
- **Per-region register CAP_SYS_RAWIO** — `dmem_region_register` (called from DRM driver bind) must require CAP_SYS_RAWIO in init userns; defense against a userns-confined driver registering a fake "GPU memory" region to side-channel charge other cgroups.
- **`dmem.events` rate-limit** — write-storm on max-hit must be rate-limited per-cgroup so OOM-like notifications cannot saturate `cgroup_file_notify` and starve unrelated controllers.
- **Atomic write-line config** — config-file write must be atomic per line; partial writes leaving the page_counter in a half-updated state are forbidden.
- **Rationale** — dmem mediates GPU memory which is increasingly attacker-accessible via Vulkan/CUDA from unprivileged containers; mis-accounting yields cross-tenant DoS (VRAM starvation) or even leakage of GPU page contents. The grsec regime forces an attacker to defeat W^X on cftype + kCFI on subsys ops + CAP_SYS_ADMIN strict-userns + CAP_SYS_RAWIO on register + refcount saturation simultaneously before reaching a host-GPU manipulation primitive.

## Open Questions

- Should `dmem.events` (max-hit counter, OOM-like) be added in Rookery to surface eviction pressure to userspace, mirroring `pids.events` / `memory.events`? Upstream baseline does not expose it.
- Does `dmemcs_offline` need to also `page_counter_uncharge` outstanding charges to prevent stuck eviction in the released cgroup, or is the current "reset limits only, keep pools alive for in-flight" semantics correct under all eviction policies?

## Out of Scope

- `struct page_counter` semantics (covered in `mm/memcontrol.md` Tier-3 alongside memcg)
- DRM/TTM GEM pool eviction policy (driver-side; covered in `drivers/gpu/drm/*` Tier-2/3 as expanded)
- Misc cgroup controller (separate file `kernel/cgroup/misc.c`)
- RDMA cgroup controller (separate file `kernel/cgroup/rdma.c`)
- cgroup core lifecycle (covered in `cgroup-core.md` Tier-3)
- Implementation code
