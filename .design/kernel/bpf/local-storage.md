# Tier-3: kernel/bpf/bpf_local_storage.c — BPF local-storage common engine

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/bpf/00-overview.md
upstream-paths:
  - kernel/bpf/bpf_local_storage.c (~877 lines)
  - include/linux/bpf_local_storage.h
-->

## Summary

The **BPF local-storage** subsystem is the shared engine behind `BPF_MAP_TYPE_TASK_STORAGE`, `BPF_MAP_TYPE_INODE_STORAGE`, `BPF_MAP_TYPE_SK_STORAGE`, and `BPF_MAP_TYPE_CGRP_STORAGE`. Unlike conventional hash/array maps keyed by a user-chosen key, local-storage maps are keyed by an **owner kernel object** (a `task_struct`, `inode`, `sock`, or `cgroup`) whose lifetime determines the lifetime of the stored value. The map holds no entries directly; instead each owner gains a per-owner `struct bpf_local_storage` linked list of `struct bpf_local_storage_elem` (selem), one selem per (owner, map) pair. The selem is double-linked into both (a) `owner->storage->list` and (b) the per-map bucket's `b->list`, enabling iteration from either direction. Per-`bpf_local_storage_lookup` is RCU-walked with a one-slot per-storage cache (`storage->cache[map->cache_idx]`) for hot maps; per-`bpf_local_storage_update` resolves BPF_NOEXIST/BPF_EXIST/BPF_F_LOCK semantics; per-`bpf_selem_unlink` performs the locked unlink-from-map-then-unlink-from-storage two-step; per-`bpf_local_storage_destroy` is the owner-going-away path. Critical for: lsm/tracing per-task data, per-socket scratch in sched_cls, per-inode security context, per-cgroup BPF state.

This Tier-3 covers `kernel/bpf/bpf_local_storage.c` (~877 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct bpf_local_storage` | per-owner anchor (list of selems + cache) | `BpfLocalStorage` |
| `struct bpf_local_storage_elem` | per-(owner, map) selem | `BpfLocalStorageElem` |
| `struct bpf_local_storage_data` | per-selem value-with-map-ptr | `BpfLocalStorageData` |
| `struct bpf_local_storage_map` | per-map: buckets + cache_idx + elem_size | `BpfLocalStorageMap` |
| `struct bpf_local_storage_map_bucket` | per-bucket: hlist + rqspinlock | `BpfLocalStorageBucket` |
| `struct bpf_local_storage_cache` | per-type one-slot cache idx allocator | `BpfLocalStorageCache` |
| `bpf_selem_alloc()` | per-selem alloc (charged) | `BpfLocalStorage::selem_alloc` |
| `bpf_selem_free()` / `_list()` / `_trace_rcu` | per-selem free (RCU/RCU-trace) | `BpfLocalStorage::selem_free*` |
| `bpf_selem_link_storage_nolock()` | per-selem link to owner storage | `BpfLocalStorage::link_storage_nolock` |
| `bpf_selem_unlink_storage_nolock()` | per-selem unlink from owner storage | `BpfLocalStorage::unlink_storage_nolock` |
| `bpf_selem_link_map()` / `_nolock()` | per-selem link to map bucket | `BpfLocalStorage::link_map*` |
| `bpf_selem_unlink_map()` / `_nolock()` | per-selem unlink from map bucket | `BpfLocalStorage::unlink_map*` |
| `bpf_selem_unlink()` | per-locked two-step unlink | `BpfLocalStorage::selem_unlink` |
| `bpf_selem_unlink_nofail()` | per-best-effort destroy-path unlink | `BpfLocalStorage::selem_unlink_nofail` |
| `bpf_local_storage_alloc()` | per-first-elem owner-storage publish | `BpfLocalStorage::alloc` |
| `bpf_local_storage_lookup()` | per-RCU lookup (with cache) | `BpfLocalStorage::lookup` |
| `bpf_local_storage_update()` | per-insert/replace under flags | `BpfLocalStorage::update` |
| `bpf_local_storage_destroy()` | per-owner-dies path | `BpfLocalStorage::destroy` |
| `__bpf_local_storage_insert_cache()` | per-lookup cache publish | `BpfLocalStorage::insert_cache` |
| `bpf_local_storage_map_alloc()` / `_free()` / `_alloc_check()` / `_check_btf()` / `_mem_usage()` | per-map lifecycle | `BpfLocalStorage::map_*` |
| `bpf_local_storage_cache_idx_get()` / `_free()` | per-cache-idx slot allocator | `BpfLocalStorage::cache_idx_get` / `_free` |
| `select_bucket()` | per-storage-ptr → bucket hash | `BpfLocalStorage::select_bucket` |
| `check_flags()` | per-update BPF_NOEXIST/BPF_EXIST gate | `BpfLocalStorage::check_flags` |
| `SDATA(selem)` / `SELEM(sdata)` | per-selem ↔ sdata container_of | `sdata_of` / `selem_of` |
| `SELEM_MAP_UNLINKED` / `_STORAGE_UNLINKED` / `_UNLINKED` / `_TOFREE` | per-selem state bits | `SelemState` |
| `BPF_LOCAL_STORAGE_CREATE_FLAG_MASK` (= F_NO_PREALLOC | F_CLONE) | per-map_flags mask | const |
| `BPF_LOCAL_STORAGE_CACHE_SIZE` | per-cache slot count (16) | const |

## Compatibility contract

REQ-1: struct bpf_local_storage (per-owner anchor):
- list: hlist of selems (one per (owner, map) pair).
- owner: void * — the owning task_struct / inode / sock / cgroup.
- cache[BPF_LOCAL_STORAGE_CACHE_SIZE]: per-cache-idx RCU pointer to last-touched sdata.
- lock: raw_res_spinlock (rqspinlock).
- mem_charge: aggregate bytes charged for this storage + its selems.
- owner_refcnt: refcount for racing destroy/map_free.
- rcu: rcu_head for kfree_rcu / call_rcu_tasks_trace.

REQ-2: struct bpf_local_storage_elem (per-(owner, map) selem):
- snode: hlist_node in storage->list.
- map_node: hlist_node in map-bucket->list.
- local_storage: __rcu pointer back to owner anchor.
- state: atomic{0, SELEM_MAP_UNLINKED, SELEM_STORAGE_UNLINKED, SELEM_UNLINKED, SELEM_TOFREE}.
- sdata: bpf_local_storage_data { smap: __rcu *bpf_local_storage_map; data[] }.
- rcu / free_node: union — used as rcu_head for free, or hlist_node in free-list during batch free.

REQ-3: struct bpf_local_storage_map (per-map):
- map: struct bpf_map base.
- buckets: pow-of-2 array of bpf_local_storage_map_bucket.
- bucket_log: ilog2(nbuckets).
- cache_idx: u16 — slot assigned in shared bpf_local_storage_cache.
- elem_size: offsetof(struct bpf_local_storage_elem, sdata.data[attr.value_size]).

REQ-4: select_bucket(smap, local_storage):
- return &smap.buckets[hash_ptr(local_storage, smap.bucket_log)].
- Bucket is keyed by the storage pointer (owner-anchored), NOT by user key — there is no user key.
- nbuckets = roundup_pow_of_two(num_possible_cpus()); ≥ 2.

REQ-5: bpf_selem_alloc(smap, owner, value, swap_uptrs):
- if mem_charge(smap, owner, smap.elem_size): return NULL.
- selem = bpf_map_kmalloc_nolock(&smap.map, smap.elem_size, __GFP_ZERO, NUMA_NO_NODE).
- if !selem: mem_uncharge(...); return NULL.
- RCU_INIT_POINTER(SDATA(selem).smap, smap).
- atomic_set(&selem.state, 0).
- if value:
  - copy_map_value(&smap.map, SDATA(selem).data, value).
  - if swap_uptrs: bpf_obj_swap_uptrs(smap.map.record, SDATA(selem).data, value).
- return selem.

REQ-6: bpf_selem_link_storage_nolock(storage, selem):
- smap = rcu_dereference(SDATA(selem).smap).
- storage.mem_charge += smap.elem_size.
- RCU_INIT_POINTER(selem.local_storage, storage).
- hlist_add_head_rcu(&selem.snode, &storage.list).

REQ-7: bpf_selem_unlink_storage_nolock(storage, selem, &free_list):
- smap = rcu_dereference(SDATA(selem).smap).
- free_local_storage = hlist_is_singular_node(&selem.snode, &storage.list).
- bpf_selem_unlink_storage_nolock_misc(selem, smap, storage, free_local_storage, false):
  - if storage.cache[smap.cache_idx] == SDATA(selem): RCU_INIT_POINTER(cache slot, NULL).
  - uncharge = smap.elem_size + (free_local_storage ? sizeof(*storage) : 0).
  - mem_uncharge(smap, storage.owner, uncharge); storage.mem_charge -= uncharge.
  - if free_local_storage: storage.owner = NULL; RCU_INIT_POINTER(*owner_storage(smap, owner), NULL).
- hlist_del_init_rcu(&selem.snode).
- hlist_add_head(&selem.free_node, &free_list).
- return free_local_storage.

REQ-8: bpf_selem_link_map(smap, storage, selem):
- b = select_bucket(smap, storage).
- raw_res_spin_lock_irqsave(&b.lock, flags); on error return err.
- hlist_add_head_rcu(&selem.map_node, &b.list).
- raw_res_spin_unlock_irqrestore.
- return 0.

REQ-9: bpf_selem_unlink_map(selem):
- storage = rcu_dereference(selem.local_storage).
- smap = rcu_dereference(SDATA(selem).smap).
- b = select_bucket(smap, storage).
- raw_res_spin_lock_irqsave(&b.lock, flags).
- hlist_del_init_rcu(&selem.map_node).
- raw_res_spin_unlock_irqrestore.

REQ-10: bpf_selem_unlink(selem) — primary delete path:
- if in_nmi(): return -EOPNOTSUPP.
- if !selem_linked_to_storage_lockless(selem): return 0 (already gone).
- storage = rcu_dereference(selem.local_storage).
- raw_res_spin_lock_irqsave(&storage.lock, flags).
- if selem_linked_to_storage(selem):
  - bpf_selem_unlink_map(selem) /* map first */.
  - free_local_storage = bpf_selem_unlink_storage_nolock(storage, selem, &selem_free_list).
- raw_res_spin_unlock_irqrestore.
- bpf_selem_free_list(&selem_free_list, false) /* call_rcu_tasks_trace */.
- if free_local_storage: bpf_local_storage_free(storage, false) /* call_rcu_tasks_trace */.

REQ-11: bpf_selem_unlink_nofail(selem, b) — destroy/map_free path:
- in_map_free = (b != NULL).
- Try lock bucket → unlink map_node + bpf_obj_free_fields under bucket lock (exactly-once guarantee).
- Try lock storage → unlink snode (and clear owner_storage if free_local_storage ∧ in_map_free).
- Set RCU_INIT_POINTER(SDATA(selem).smap, NULL) and selem.local_storage = NULL when respective lock succeeded or in_map_free.
- If either lock failed (rqspinlock returned -ETIMEDOUT): WARN_ON_ONCE (resource leak), do NOT block.
- Two-sided cooperative free via selem.state atomic transitions:
  - unlink == 2: free immediately.
  - else: cmpxchg(state, SELEM_UNLINKED, SELEM_TOFREE) == SELEM_UNLINKED ⟹ free.
  - else: peer side will set the bit, transition to TOFREE, and free.
- bpf_selem_free(selem, true) /* kfree_rcu */.
- if free_storage: bpf_local_storage_free(storage, true) /* kfree_rcu */.

REQ-12: bpf_local_storage_alloc(owner, smap, first_selem):
- mem_charge(smap, owner, sizeof *storage).
- storage = bpf_map_kmalloc_nolock(&smap.map, sizeof *storage, __GFP_ZERO, NUMA_NO_NODE).
- INIT_HLIST_HEAD(&storage.list); raw_res_spin_lock_init(&storage.lock).
- storage.owner = owner; storage.mem_charge = sizeof *storage.
- refcount_set(&storage.owner_refcnt, 1).
- bpf_selem_link_storage_nolock(storage, first_selem).
- b = select_bucket(smap, storage); raw_res_spin_lock_irqsave(&b.lock, flags).
- bpf_selem_link_map_nolock(b, first_selem).
- /* Publish via cmpxchg */
- prev = cmpxchg(owner_storage(smap, owner), NULL, storage).
- if prev: unlink, unlock, return -EAGAIN (another CPU won).
- raw_res_spin_unlock_irqrestore.
- return 0.

REQ-13: bpf_local_storage_update(owner, smap, value, map_flags, swap_uptrs):
- Validate map_flags: (raw & ~BPF_F_LOCK) > BPF_EXIST ⟹ -EINVAL.
- Validate BPF_F_LOCK requires map value to have BPF_SPIN_LOCK field ⟹ else -EINVAL.
- storage = rcu_dereference(*owner_storage(smap, owner)).
- /* First-elem fast path */
- if !storage ∨ hlist_empty(&storage.list):
  - check_flags(NULL, map_flags) /* BPF_EXIST ⟹ -ENOENT */.
  - selem = bpf_selem_alloc(smap, owner, value, swap_uptrs).
  - bpf_local_storage_alloc(owner, smap, selem).
  - return SDATA(selem).
- /* In-place fast path (BPF_F_LOCK ∧ !BPF_NOEXIST) */
- if (map_flags & BPF_F_LOCK) ∧ !(map_flags & BPF_NOEXIST):
  - old_sdata = bpf_local_storage_lookup(storage, smap, false).
  - check_flags(old_sdata, map_flags).
  - if old_sdata ∧ selem_linked_to_storage_lockless(SELEM(old_sdata)):
    - copy_map_value_locked(&smap.map, old_sdata.data, value, false).
    - return old_sdata.
- /* General path: alloc new, replace old */
- alloc_selem = selem = bpf_selem_alloc(smap, owner, value, swap_uptrs).
- raw_res_spin_lock_irqsave(&storage.lock, flags).
- if hlist_empty(&storage.list): err = -EAGAIN; goto unlock.
- old_sdata = bpf_local_storage_lookup(storage, smap, false).
- check_flags(old_sdata, map_flags); if err: goto unlock.
- if old_sdata ∧ (map_flags & BPF_F_LOCK):
  - copy_map_value_locked(...); selem = SELEM(old_sdata); goto unlock.
- b = select_bucket(smap, storage).
- raw_res_spin_lock_irqsave(&b.lock, b_flags).
- alloc_selem = NULL /* ownership transferred */.
- bpf_selem_link_map_nolock(b, selem) /* first link map */.
- bpf_selem_link_storage_nolock(storage, selem) /* then link storage (publish) */.
- if old_sdata: bpf_selem_unlink_map_nolock(SELEM(old_sdata)); bpf_selem_unlink_storage_nolock(storage, SELEM(old_sdata), &old_free_list).
- raw_res_spin_unlock_irqrestore(&b.lock).
- raw_res_spin_unlock_irqrestore(&storage.lock).
- bpf_selem_free_list(&old_free_list, false).
- if alloc_selem still set: mem_uncharge(...); bpf_selem_free(alloc_selem, true).
- return ERR_PTR(err) ∨ SDATA(selem).

REQ-14: check_flags(old_sdata, map_flags):
- if old_sdata ∧ (raw & ~BPF_F_LOCK) == BPF_NOEXIST: return -EEXIST.
- if !old_sdata ∧ (raw & ~BPF_F_LOCK) == BPF_EXIST: return -ENOENT.
- return 0.

REQ-15: __bpf_local_storage_insert_cache(storage, smap, selem):
- raw_res_spin_lock_irqsave(&storage.lock, flags).
- if selem_linked_to_storage(selem): rcu_assign_pointer(storage.cache[smap.cache_idx], SDATA(selem)).
- raw_res_spin_unlock_irqrestore.
- Called by bpf_local_storage_lookup() to publish a freshly-found selem into the cache slot.

REQ-16: bpf_local_storage_cache_idx_get(cache):
- spin_lock(&cache.idx_lock).
- Scan cache.idx_usage_counts[0..16]; pick lowest-usage index (break on 0).
- cache.idx_usage_counts[res]++.
- spin_unlock.
- return res.

REQ-17: bpf_local_storage_cache_idx_free(cache, idx):
- spin_lock; cache.idx_usage_counts[idx]--; spin_unlock.

REQ-18: bpf_local_storage_destroy(local_storage) — owner-going-away:
- hlist_for_each_entry_rcu(selem, &storage.list, snode): bpf_selem_unlink_nofail(selem, NULL).
- if !refcount_dec_and_test(&storage.owner_refcnt):
  - busy-wait until owner_refcnt reaches 0 (cpu_relax loop).
  - smp_mb.
- return storage.mem_charge.

REQ-19: bpf_local_storage_map_alloc_check(attr):
- attr.map_flags & ~BPF_LOCAL_STORAGE_CREATE_FLAG_MASK ⟹ -EINVAL.
- !(map_flags & BPF_F_NO_PREALLOC) ⟹ -EINVAL (no-prealloc mandatory).
- attr.max_entries != 0 ⟹ -EINVAL.
- attr.key_size != sizeof(int) ⟹ -EINVAL.
- !attr.value_size ⟹ -EINVAL.
- BTF key+value mandatory ⟹ -EINVAL.
- value_size > BPF_LOCAL_STORAGE_MAX_VALUE_SIZE ⟹ -E2BIG.

REQ-20: bpf_local_storage_map_check_btf(map, btf, key_type, value_type):
- !btf_type_is_i32(key_type) ⟹ -EINVAL  /* key is always int placeholder */.
- return 0.

REQ-21: bpf_local_storage_map_alloc(attr, cache):
- smap = bpf_map_area_alloc(sizeof *smap, NUMA_NO_NODE).
- bpf_map_init_from_attr(&smap.map, attr).
- nbuckets = max(2, roundup_pow_of_two(num_possible_cpus())).
- smap.bucket_log = ilog2(nbuckets).
- smap.buckets = bpf_map_kvcalloc(...).
- for i in 0..nbuckets: INIT_HLIST_HEAD(&buckets[i].list); raw_res_spin_lock_init(&buckets[i].lock).
- smap.elem_size = offsetof(struct bpf_local_storage_elem, sdata.data[attr.value_size]).
- smap.cache_idx = bpf_local_storage_cache_idx_get(cache).
- return &smap.map.

REQ-22: bpf_local_storage_map_free(map, cache):
- bpf_local_storage_cache_idx_free(cache, smap.cache_idx).
- synchronize_rcu() /* wait for in-flight bpf_sk_storage_clone etc. */.
- for each bucket b:
  - rcu_read_lock.
  - hlist_for_each_entry_rcu(selem, &b.list, map_node): bpf_selem_unlink_nofail(selem, b).
    - if need_resched(): cond_resched_rcu(); goto restart.
  - rcu_read_unlock.
- synchronize_rcu() /* storage may still reference smap.elem_size for uncharging */.
- rcu_barrier_tasks_trace(); rcu_barrier().
- kvfree(smap.buckets); bpf_map_area_free(smap).

REQ-23: SELEM_UNLINKED state machine:
- 0 (alive).
- SELEM_MAP_UNLINKED — destroy() or map_free() removed from map but not storage.
- SELEM_STORAGE_UNLINKED — peer removed from storage but not map.
- SELEM_UNLINKED = MAP | STORAGE.
- SELEM_TOFREE — cmpxchg winner takes ownership of free.

## Acceptance Criteria

- [ ] AC-1: bpf_local_storage_map_alloc: attr without BPF_F_NO_PREALLOC ⟹ -EINVAL.
- [ ] AC-2: bpf_local_storage_map_alloc: attr.max_entries != 0 ⟹ -EINVAL.
- [ ] AC-3: First lookup-miss + update: bpf_local_storage_alloc publishes storage via cmpxchg; concurrent loser gets -EAGAIN.
- [ ] AC-4: bpf_local_storage_update(BPF_NOEXIST) on existing ⟹ -EEXIST.
- [ ] AC-5: bpf_local_storage_update(BPF_EXIST) on absent ⟹ -ENOENT.
- [ ] AC-6: bpf_local_storage_update(BPF_F_LOCK) on existing-with-spin-lock ⟹ in-place copy_map_value_locked (no alloc).
- [ ] AC-7: bpf_local_storage_update(BPF_F_LOCK) on map without BPF_SPIN_LOCK field ⟹ -EINVAL.
- [ ] AC-8: bpf_selem_unlink: in NMI ⟹ -EOPNOTSUPP.
- [ ] AC-9: bpf_selem_unlink: under storage lock, map_node unlinked before snode (map-first ordering).
- [ ] AC-10: When last selem unlinked from storage: owner_storage pointer cleared, storage freed after RCU tasks trace grace period.
- [ ] AC-11: bpf_local_storage_destroy: unlinks all selems best-effort; busy-waits owner_refcnt to 0; returns mem_charge.
- [ ] AC-12: bpf_local_storage_map_free: drains all buckets, two synchronize_rcu + rcu_barrier_tasks_trace + rcu_barrier before kvfree.
- [ ] AC-13: Concurrent destroy(selem) ∥ map_free(selem): each unlinks one side; SELEM_UNLINKED cmpxchg→SELEM_TOFREE winner frees exactly once.
- [ ] AC-14: bpf_local_storage_lookup hits cache when storage.cache[map.cache_idx] == matching sdata.
- [ ] AC-15: nbuckets ≥ 2 (select_bucket UB on 1).
- [ ] AC-16: bpf_obj_free_fields invoked under bucket lock (exactly-once for special-field cleanup).

## Architecture

```
struct BpfLocalStorage {
  list: HlistHead<BpfLocalStorageElem, snode>,
  owner: *mut c_void,                              // task/inode/sock/cgrp
  cache: [RcuPtr<BpfLocalStorageData>; BPF_LOCAL_STORAGE_CACHE_SIZE],
  lock: RawResSpinLock,
  mem_charge: u32,
  owner_refcnt: Refcount,
  rcu: RcuHead,
}

struct BpfLocalStorageElem {
  snode: HlistNode,                                // in storage.list
  map_node: HlistNode,                             // in bucket.list
  local_storage: RcuPtr<BpfLocalStorage>,
  state: AtomicU32,                                // SELEM_* bits
  sdata: BpfLocalStorageData,
  // union { rcu: RcuHead, free_node: HlistNode }
}

struct BpfLocalStorageData {
  smap: RcuPtr<BpfLocalStorageMap>,
  data: [u8; ...],                                 // value
}

struct BpfLocalStorageMap {
  map: BpfMap,
  buckets: Box<[BpfLocalStorageBucket]>,
  bucket_log: u32,
  cache_idx: u16,
  elem_size: u32,
}

struct BpfLocalStorageBucket {
  list: HlistHead<BpfLocalStorageElem, map_node>,
  lock: RawResSpinLock,
}

struct BpfLocalStorageCache {
  idx_lock: SpinLock,
  idx_usage_counts: [u64; BPF_LOCAL_STORAGE_CACHE_SIZE],
}
```

`BpfLocalStorage::select_bucket(smap, storage) -> &Bucket`:
1. &smap.buckets[hash_ptr(storage, smap.bucket_log)].

`BpfLocalStorage::selem_alloc(smap, owner, value, swap_uptrs) -> Option<*Selem>`:
1. mem_charge(smap, owner, smap.elem_size)?; on failure return None.
2. selem = bpf_map_kmalloc_nolock(&smap.map, smap.elem_size, __GFP_ZERO, NUMA_NO_NODE).
3. if selem.is_null(): mem_uncharge(smap, owner, smap.elem_size); return None.
4. RCU_INIT_POINTER(SDATA(selem).smap, smap).
5. selem.state.store(0, Relaxed).
6. if !value.is_null():
   - copy_map_value(&smap.map, SDATA(selem).data, value).
   - if swap_uptrs: bpf_obj_swap_uptrs(smap.map.record, SDATA(selem).data, value).
7. Some(selem).

`BpfLocalStorage::link_storage_nolock(storage, selem)`:
1. smap = rcu_dereference_check(SDATA(selem).smap, bpf_rcu_lock_held()).
2. storage.mem_charge += smap.elem_size.
3. RCU_INIT_POINTER(selem.local_storage, storage).
4. hlist_add_head_rcu(&selem.snode, &storage.list).

`BpfLocalStorage::unlink_storage_nolock(storage, selem, &free_list) -> bool`:
1. smap = rcu_dereference_check(SDATA(selem).smap, bpf_rcu_lock_held()).
2. free_storage = hlist_is_singular_node(&selem.snode, &storage.list).
3. /* clear cache slot if it points at us */
4. if rcu_access_pointer(storage.cache[smap.cache_idx]) == SDATA(selem):
   - RCU_INIT_POINTER(storage.cache[smap.cache_idx], NULL).
5. uncharge = smap.elem_size + (if free_storage { sizeof *storage } else { 0 }).
6. mem_uncharge(smap, storage.owner, uncharge).
7. storage.mem_charge -= uncharge.
8. if free_storage:
   - storage.owner = NULL.
   - RCU_INIT_POINTER(*owner_storage(smap, owner_was), NULL).
9. hlist_del_init_rcu(&selem.snode).
10. hlist_add_head(&selem.free_node, &free_list).
11. return free_storage.

`BpfLocalStorage::selem_unlink(selem) -> Result<()>`:
1. if in_nmi(): return Err(EOPNOTSUPP).
2. if !selem_linked_to_storage_lockless(selem): return Ok.
3. storage = rcu_dereference_check(selem.local_storage, bpf_rcu_lock_held()).
4. let mut free_selem_list = HlistHead::default().
5. raw_res_spin_lock_irqsave(&storage.lock, flags)?.
6. if likely(selem_linked_to_storage(selem)):
   - BpfLocalStorage::unlink_map(selem)?     /* map FIRST */.
   - free_storage = BpfLocalStorage::unlink_storage_nolock(storage, selem, &mut free_selem_list).
7. raw_res_spin_unlock_irqrestore.
8. BpfLocalStorage::selem_free_list(&free_selem_list, false) /* call_rcu_tasks_trace */.
9. if free_storage: BpfLocalStorage::local_storage_free(storage, false).
10. Ok.

`BpfLocalStorage::alloc(owner, smap, first_selem) -> Result<()>`:
1. mem_charge(smap, owner, sizeof *Storage)?.
2. storage = bpf_map_kmalloc_nolock(&smap.map, sizeof *Storage, __GFP_ZERO, NUMA_NO_NODE).ok_or(ENOMEM)?.
3. INIT_HLIST_HEAD(&storage.list); raw_res_spin_lock_init(&storage.lock).
4. storage.owner = owner; storage.mem_charge = sizeof *Storage.
5. refcount_set(&storage.owner_refcnt, 1).
6. BpfLocalStorage::link_storage_nolock(storage, first_selem).
7. b = BpfLocalStorage::select_bucket(smap, storage).
8. raw_res_spin_lock_irqsave(&b.lock, flags)?.
9. BpfLocalStorage::link_map_nolock(b, first_selem).
10. owner_ptr = owner_storage(smap, owner) /* &mut *RcuPtr */.
11. prev = cmpxchg(owner_ptr, NULL, storage).
12. if !prev.is_null():
    - BpfLocalStorage::unlink_map_nolock(first_selem).
    - raw_res_spin_unlock_irqrestore(&b.lock).
    - goto uncharge; err = -EAGAIN.
13. raw_res_spin_unlock_irqrestore(&b.lock).
14. Ok.

`BpfLocalStorage::update(owner, smap, value, map_flags, swap_uptrs) -> Result<*Sdata>`:
1. validate map_flags.
2. storage = rcu_dereference_check(*owner_storage(smap, owner), bpf_rcu_lock_held()).
3. if storage.is_null() ∨ hlist_empty(&storage.list):
   - check_flags(None, map_flags)?.
   - selem = BpfLocalStorage::selem_alloc(smap, owner, value, swap_uptrs).ok_or(ENOMEM)?.
   - BpfLocalStorage::alloc(owner, smap, selem)?  /* on err: free selem, uncharge */.
   - return Ok(SDATA(selem)).
4. if (map_flags & BPF_F_LOCK) ∧ !(map_flags & BPF_NOEXIST):
   - old_sdata = BpfLocalStorage::lookup(storage, smap, false).
   - check_flags(old_sdata, map_flags)?.
   - if old_sdata.is_some() ∧ selem_linked_to_storage_lockless(SELEM(old_sdata)):
     - copy_map_value_locked(&smap.map, old_sdata.data, value, false).
     - return Ok(old_sdata).
5. alloc_selem = selem = BpfLocalStorage::selem_alloc(smap, owner, value, swap_uptrs).ok_or(ENOMEM)?.
6. raw_res_spin_lock_irqsave(&storage.lock, flags)?.
7. if hlist_empty(&storage.list): err = -EAGAIN; goto unlock.
8. old_sdata = BpfLocalStorage::lookup(storage, smap, false).
9. check_flags(old_sdata, map_flags); if err: goto unlock.
10. if old_sdata.is_some() ∧ (map_flags & BPF_F_LOCK):
    - copy_map_value_locked(..., false); selem = SELEM(old_sdata); goto unlock.
11. b = BpfLocalStorage::select_bucket(smap, storage).
12. raw_res_spin_lock_irqsave(&b.lock, b_flags)?.
13. alloc_selem = None /* ownership transferred */.
14. BpfLocalStorage::link_map_nolock(b, selem).
15. BpfLocalStorage::link_storage_nolock(storage, selem).
16. if let Some(o) = old_sdata:
    - BpfLocalStorage::unlink_map_nolock(SELEM(o)).
    - BpfLocalStorage::unlink_storage_nolock(storage, SELEM(o), &mut old_free_list).
17. raw_res_spin_unlock_irqrestore(&b.lock).
18. unlock: raw_res_spin_unlock_irqrestore(&storage.lock).
19. BpfLocalStorage::selem_free_list(&old_free_list, false).
20. if alloc_selem.is_some(): mem_uncharge(...); BpfLocalStorage::selem_free(alloc_selem, true).
21. Result::from(err, SDATA(selem)).

`BpfLocalStorage::destroy(storage) -> u32`:
1. hlist_for_each_entry_rcu(selem, &storage.list, snode):
   - BpfLocalStorage::selem_unlink_nofail(selem, NULL).
2. if !refcount_dec_and_test(&storage.owner_refcnt):
   - while refcount_read(&storage.owner_refcnt) != 0: cpu_relax().
   - smp_mb().
3. return storage.mem_charge.

`BpfLocalStorage::map_free(map, cache)`:
1. cache_idx_free(cache, smap.cache_idx).
2. synchronize_rcu() /* wait for clone-in-flight readers */.
3. for b in smap.buckets:
   - rcu_read_lock.
   - 'restart: loop { for selem in b.list (rcu): selem_unlink_nofail(selem, b);
     if need_resched(): cond_resched_rcu(); continue 'restart; }
   - rcu_read_unlock.
4. synchronize_rcu().
5. rcu_barrier_tasks_trace(); rcu_barrier().
6. kvfree(smap.buckets); bpf_map_area_free(smap).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `nmi_forbidden_in_unlink` | INVARIANT | per-selem_unlink: in_nmi ⟹ -EOPNOTSUPP early-return. |
| `map_unlink_before_storage_unlink` | INVARIANT | per-selem_unlink under storage.lock: hlist_del map_node happens-before hlist_del snode. |
| `cmpxchg_owner_storage_publish` | INVARIANT | per-alloc: prev != NULL ⟹ caller-side rollback (unlink_map, free). |
| `cache_idx_in_bounds` | INVARIANT | per-storage: smap.cache_idx < BPF_LOCAL_STORAGE_CACHE_SIZE. |
| `bucket_log_ge_1` | INVARIANT | per-map_alloc: bucket_log ≥ 1 (nbuckets ≥ 2). |
| `selem_free_once` | INVARIANT | per-selem_unlink_nofail: at most one path satisfies (unlink==2 ∨ cmpxchg-winner) → bpf_selem_free. |
| `lock_owner_storage_lock_irqsave` | INVARIANT | per-update / unlink mutations: storage.lock held + IRQs off. |
| `lock_bucket_lock_irqsave` | INVARIANT | per-map mutations: b.lock held + IRQs off. |
| `bpf_obj_free_fields_exactly_once` | INVARIANT | per-selem: bpf_obj_free_fields called once under bucket lock at unlink_nofail map side. |
| `no_prealloc_required` | INVARIANT | per-map_alloc_check: BPF_F_NO_PREALLOC absent ⟹ -EINVAL. |
| `value_size_bounded` | INVARIANT | per-map_alloc_check: value_size > BPF_LOCAL_STORAGE_MAX_VALUE_SIZE ⟹ -E2BIG. |
| `BPF_F_LOCK_requires_BPF_SPIN_LOCK_field` | INVARIANT | per-update: BPF_F_LOCK without spin_lock field in record ⟹ -EINVAL. |

### Layer 2: TLA+

`kernel/bpf/local-storage.tla`:
- Models: owner, storage (None or {list, cache, refcnt}), map.buckets[*], selems with state ∈ {0, MAP_UNLINKED, STORAGE_UNLINKED, UNLINKED, TOFREE}.
- Properties:
  - `safety_owner_storage_publish_unique` — at most one storage ever installed at *owner_storage(smap, owner); racing alloc returns -EAGAIN.
  - `safety_selem_freed_exactly_once` — for each selem, exactly one Free action occurs across destroy ∥ map_free.
  - `safety_map_unlink_precedes_storage_unlink_in_selem_unlink` — selem_unlink emits map-unlink-event before storage-unlink-event.
  - `safety_owner_storage_ptr_null_iff_no_storage` — *owner_storage == NULL ⟺ no storage record for owner.
  - `safety_in_nmi_no_unlink` — selem_unlink in NMI never mutates state.
  - `liveness_destroy_drains_storage` — bpf_local_storage_destroy: eventually all selems unlinked ∧ refcount 0 ∧ free.
  - `liveness_map_free_drains_buckets` — bpf_local_storage_map_free: every bucket eventually emptied; selems eventually freed (after rcu_barrier_tasks_trace + rcu_barrier).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `BpfLocalStorage::alloc` post: Ok ⟹ *owner_storage(smap, owner) == storage ∧ storage.list contains first_selem | `BpfLocalStorage::alloc` |
| `BpfLocalStorage::update` post: Ok(sdata) ⟹ sdata.smap == smap ∧ SELEM(sdata) linked into storage.list ∧ bucket.list | `BpfLocalStorage::update` |
| `BpfLocalStorage::selem_unlink` post: Ok ⟹ !selem_linked_to_storage ∧ !selem_linked_to_map | `BpfLocalStorage::selem_unlink` |
| `BpfLocalStorage::destroy` post: returns storage.mem_charge ∧ list empty ∧ refcount 0 | `BpfLocalStorage::destroy` |
| `BpfLocalStorage::map_free` post: smap.buckets freed ∧ smap freed ∧ no selem leaks | `BpfLocalStorage::map_free` |
| `check_flags` post: (old, BPF_NOEXIST) ⟹ -EEXIST; (∅, BPF_EXIST) ⟹ -ENOENT; else 0 | `check_flags` |
| `cache_idx_get` post: returns idx with cache.idx_usage_counts[idx] incremented atomically | `cache_idx_get` |

### Layer 4: Verus/Creusot functional

`Per-(owner,map) selem lifecycle: alloc → publish via cmpxchg(owner_storage) → linked into bucket+storage → updated/replaced under check_flags → unlinked map-first then storage → state.cmpxchg(SELEM_UNLINKED, SELEM_TOFREE) winner frees via call_rcu_tasks_trace` semantic equivalence: per-Documentation/bpf/map_*_storage.rst.

## Hardening

(Inherits row-1 features from `kernel/bpf/00-overview.md` § Hardening.)

local-storage reinforcement:

- **Per-rqspinlock raw_res_spin_lock_irqsave** — defense against per-NMI/IRQ deadlock; -ETIMEDOUT escape valve.
- **Per-storage.lock + per-bucket.lock acquire ordering (storage outer, bucket inner)** — defense against per-AB-BA deadlock.
- **Per-map-first-then-storage unlink ordering** — defense against per-selem-freed-while-on-bucket-list iteration UAF.
- **Per-cmpxchg owner-storage publish** — defense against per-double-storage-allocate races without owner-side locks.
- **Per-SELEM_UNLINKED → SELEM_TOFREE cmpxchg** — defense against per-double-free across destroy ∥ map_free.
- **Per-call_rcu_tasks_trace for default free** — defense against per-bpf-trampoline-still-reading selem after unlink.
- **Per-kfree_rcu for reuse_now free** — defense against per-RCU-reader UAF when caller asserts no trampoline reference.
- **Per-bucket-lock-held bpf_obj_free_fields** — defense against per-special-field-double-free (kptrs, timers, rb_root).
- **Per-cond_resched_rcu in map_free drain** — defense against per-RCU-CPU-stall on huge maps.
- **Per-synchronize_rcu + rcu_barrier_tasks_trace + rcu_barrier in map_free** — defense against per-trampoline-freed-callback UAF.
- **Per-BPF_F_NO_PREALLOC mandatory** — defense against per-prealloc-allocation-per-owner blow-up.
- **Per-value_size ≤ BPF_LOCAL_STORAGE_MAX_VALUE_SIZE** — defense against per-attacker-driven huge selem.
- **Per-mem_charge / _uncharge symmetric** — defense against per-map memcg accounting drift.
- **Per-WARN_ON_ONCE on rqspinlock timeout in destroy path** — defense against per-silent leak.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — `bpf_*_storage_get` / `_delete` from BPF programs return a user-readable `value` pointer of `value_size` bytes; whitelist the per-map value window so a verifier-side size-confusion between map type (TASK/INODE/SK/CGRP) cannot drag adjacent `bpf_local_storage_elem` metadata across the kernel boundary.
- **PAX_KERNEXEC** — `bpf_local_storage_map_ops` dispatch (per-type alloc/free/destroy) is indirect-called from BPF helpers; enforce W^X on the ops table page and refuse rwx for any JIT trampoline emitting the helper call.
- **PAX_RANDKSTACK** — storage helpers run from arbitrary BPF prog context (LSM, tracing, sched_cls) at user-driven stack depth; randomized kstack offset disrupts ROP through the `bpf_local_storage_update` rqspinlock critical section.
- **PAX_REFCOUNT** — `bpf_local_storage->refcnt` and per-selem `bpf_selem->snode` lifetime counters are hot fast-path; refcount_t with saturating overflow trap prevents underflow during concurrent owner-free (task_exit / inode_evict / sk_destruct / css_offline) racing storage update.
- **PAX_MEMORY_SANITIZE** — `bpf_selem_free_rcu` callback must zero the value region (including any embedded kptr / spin_lock / timer special-fields) before returning the selem to bpf_mem_cache; defense against post-RCU stale-kptr resurrection.
- **PAX_UDEREF** — `bpf_*_storage_get` from syscall side (BPF_MAP_*_ELEM on a local_storage map) reads `key` (an fd-or-pointer to the owner) from user `bpf_attr`; SMAP/PAN must engage across `bpf_fd_*_storage_lookup_elem` translation.
- **PAX_RAP/kCFI** — per-type `bpf_local_storage_cache` `local_storage_update` indirect calls (task/inode/sk/cgrp) must verify kCFI tag; defense against confused-deputy where a TASK_STORAGE selem could be threaded through SK_STORAGE update path post-corruption.
- **GRKERNSEC_HIDESYM** — `bpf_map_show_fdinfo` for local_storage maps must hide the per-cache pointer and the selem head address from non-CAP_SYSLOG readers; expose only id + value_size.
- **GRKERNSEC_DMESG** — `WARN_ON_ONCE(rqspinlock timeout)` in the destroy path must not splat raw selem / owner pointers into dmesg readable by non-CAP_SYSLOG.
- **Per-object lifetime binding** — TASK_STORAGE selems must die with `release_task`; INODE_STORAGE with `__destroy_inode`; SK_STORAGE with `sk_destruct`; CGRP_STORAGE with `css_free_rwork_fn`. Any decoupling (e.g., delayed teardown via workqueue) widens a UAF window and is forbidden under grsec mode.
- **BPF_F_NO_PREALLOC mandatory + memcg charge** — local_storage maps must reject `!(map_flags & BPF_F_NO_PREALLOC)`; per-selem mem_charge must go through memcg with the owner cgroup so a CAP_BPF user cannot pin host slab via a flood of task-storage allocations.
- **Per-owner storage ceiling** — Rookery should impose `bpf_local_storage_max_per_owner` (default 64) so a single task/inode/sk/cgroup cannot host an unbounded selem chain that defeats RCU-grace destroy bounds.
- **Rationale** — local_storage is uniquely hazardous: its lifetime is bound to a kernel object the BPF user does not own (task, inode, sk, cgroup), so an attacker can use storage maps to extend BPF-controlled state across cred boundaries (post-`exec`, post-`fork`, post-`fchown`). The grsec regime forces multi-layer escalation (defeat refcount saturation + RCU sanitize + kCFI dispatch + memcg quota) before storage UAF becomes a control-flow primitive.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- kernel/bpf/bpf_task_storage.c (per-task storage glue) — separate Tier-3
- kernel/bpf/bpf_inode_storage.c (per-inode storage glue) — separate Tier-3
- net/core/bpf_sk_storage.c (per-sock storage glue) — separate Tier-3
- kernel/bpf/bpf_cgrp_storage.c (per-cgroup storage glue) — separate Tier-3
- kernel/bpf/syscall.c map-type registration (covered in `bpf-core.md` Tier-3)
- kernel/bpf/verifier.c map-type semantics (covered in `verifier.md` Tier-3)
- kernel/bpf/memalloc.c bpf_map_kmalloc_nolock backing (covered separately if expanded)
- kernel/rcu/* (covered in `kernel/rcu/*.md`)
- Implementation code
