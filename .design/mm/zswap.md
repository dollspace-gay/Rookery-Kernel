# Tier-3: mm/zswap.c — Compressed-RAM swap cache

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: mm/00-overview.md
upstream-paths:
  - mm/zswap.c (~1837 lines)
  - include/linux/zswap.h
  - mm/zsmalloc.c (allocation backend; see zsmalloc.md Tier-3)
  - include/crypto/acompress.h (acomp_request / crypto_acomp)
  - Documentation/admin-guide/mm/zswap.rst
-->

## Summary

zswap is a **transparent, compressed, RAM-resident swap cache** sitting in front of disk-backed swap. When the swap subsystem would `swap_writeout()` an anonymous folio to disk, zswap intercepts, compresses the page into a pool-managed buffer (via `crypto_acomp` — lzo/lz4/zstd/etc.), and parks a pointer in an xarray keyed by `swp_entry_t`. Reads via `zswap_load()` decompress in-place and invalidate the compressed copy. Per-(swap-type, address-space-chunk) xarray: one `xarray` per 64 MiB swap region (`ZSWAP_ADDRESS_SPACE_PAGES = 1 << 14`). Per-`struct zswap_entry`: `{swpentry, length, referenced, pool, handle, objcg, lru}`. Per-`struct zswap_pool`: `{zs_pool (zsmalloc backing), acomp_ctx per-CPU, percpu_ref, list, release_work}`. Per-memcg LRU + shrinker: a global `list_lru zswap_list_lru` partitioned per-node-per-memcg, walked by a registered shrinker that calls `zswap_writeback_entry()` to push compressed pages out to the underlying swap device under pressure (the "writeback to backing swap on pressure" path). Per-`zswap_max_pool_percent` (default 20 % of RAM) bounds the compressed pool; `zswap_accept_thr_percent` (90 % of max) is the high-watermark that triggers async `shrink_worker` reclaim. Per-memcg accounting via `obj_cgroup_charge_zswap`. Critical for: swap-amplification on memory-constrained systems, low-latency anonymous-page re-fetch, container/memcg-aware compressed-swap policy.

This Tier-3 covers `mm/zswap.c` (~1837 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct zswap_pool` | per-pool: zsmalloc + acomp + percpu-ref | `ZswapPool` |
| `struct zswap_entry` | per-compressed-page metadata | `ZswapEntry` |
| `struct crypto_acomp_ctx` | per-CPU compressor context | `AcompCtx` |
| `zswap_trees[MAX_SWAPFILES]` | per-swap-type xarray-array | `Zswap::trees` |
| `swap_zswap_tree()` | per-swp_entry → xarray | `Zswap::tree_for_swp` |
| `zswap_pools` (LIST_HEAD) | per-active-pool list | `Zswap::pools` |
| `zswap_pools_lock` | per-pool-list lock | `Zswap::pools_lock` |
| `zswap_list_lru` | per-memcg-per-node LRU of entries | `Zswap::list_lru` |
| `zswap_shrinker` | per-shrinker-callback | `Zswap::shrinker` |
| `zswap_shrink_work` | per-WQ work for async reclaim | `Zswap::shrink_work` |
| `zswap_next_shrink` | per-shrinker memcg cursor | `Zswap::next_shrink` |
| `zswap_entry_cache` | per-kmem_cache for zswap_entry | `Zswap::entry_cache` |
| `zswap_pool_create()` | per-compressor pool ctor | `ZswapPool::create` |
| `zswap_pool_destroy()` | per-pool dtor | `ZswapPool::destroy` |
| `__zswap_pool_empty()` | per-percpu_ref-zero hook | `ZswapPool::on_ref_zero` |
| `__zswap_pool_release()` | per-RCU-grace release | `ZswapPool::release_work` |
| `zswap_pool_tryget` / `_get` / `_put` | per-refcount API | `ZswapPool::tryget` / `get` / `put` |
| `zswap_pool_current_get()` | per-current-pool ref | `Zswap::current_pool_get` |
| `zswap_pool_find_get()` | per-named lookup | `Zswap::pool_find_get` |
| `zswap_max_pages()` | per-stat: max compressed pages | `Zswap::max_pages` |
| `zswap_accept_thr_pages()` | per-stat: accept-threshold | `Zswap::accept_thr_pages` |
| `zswap_total_pages()` | per-stat: current pool size | `Zswap::total_pages` |
| `zswap_check_limits()` | per-store admission test | `Zswap::check_limits` |
| `zswap_compressor_param_set()` | per-sysfs compressor swap | `Zswap::set_compressor_param` |
| `zswap_enabled_param_set()` | per-sysfs enable | `Zswap::set_enabled_param` |
| `zswap_cpu_comp_prepare()` | per-CPU acomp alloc hotplug | `Zswap::cpu_comp_prepare` |
| `zswap_compress()` | per-page compress | `Zswap::compress` |
| `zswap_decompress()` | per-entry decompress | `Zswap::decompress` |
| `zswap_writeback_entry()` | per-entry writeback to disk swap | `Zswap::writeback_entry` |
| `shrink_memcg_cb()` | per-LRU walk cb (second-chance) | `Zswap::shrink_memcg_cb` |
| `zswap_shrinker_scan` / `_count` | per-shrinker interface | `Zswap::shrinker_scan` / `_count` |
| `zswap_alloc_shrinker()` | per-shrinker ctor | `Zswap::alloc_shrinker` |
| `shrink_memcg()` | per-memcg LRU drain | `Zswap::shrink_memcg` |
| `shrink_worker()` | per-WQ shrink loop | `Zswap::shrink_worker` |
| `zswap_store_page()` | per-page store | `Zswap::store_page` |
| `zswap_store()` | per-folio store entry-point | `Zswap::store` |
| `zswap_load()` | per-folio load entry-point | `Zswap::load` |
| `zswap_invalidate()` | per-swp_entry invalidate | `Zswap::invalidate` |
| `zswap_swapon()` / `swapoff()` | per-swap-type setup / teardown | `Zswap::swapon` / `swapoff` |
| `zswap_lru_add()` / `_del()` | per-entry LRU add/remove | `Zswap::lru_add` / `lru_del` |
| `zswap_lruvec_state_init()` | per-lruvec stat init | `Zswap::lruvec_state_init` |
| `zswap_folio_swapin()` | per-swapin LRU touch | `Zswap::folio_swapin` |
| `zswap_memcg_offline_cleanup()` | per-memcg-offline cursor cleanup | `Zswap::memcg_offline_cleanup` |
| `zswap_entry_cache_alloc` / `_free` / `zswap_entry_free` | per-entry alloc/free | `ZswapEntry::alloc` / `free` |
| `zswap_setup()` / `zswap_init()` | per-boot init | `Zswap::setup` / `init` |

## Compatibility contract

REQ-1: struct zswap_pool:
- zs_pool: backing zsmalloc pool (the only supported backend in 7.1; legacy zpool indirection collapsed).
- acomp_ctx: percpu `crypto_acomp_ctx` { acomp, req, wait, buffer (PAGE_SIZE), mutex }.
- ref: `percpu_ref` with PERCPU_REF_ALLOW_REINIT.
- list: zswap_pools linkage.
- release_work: per-zero-ref RCU-deferred destroy.
- node: cpuhp hlist node for CPUHP_MM_ZSWP_POOL_PREPARE.
- tfm_name[CRYPTO_MAX_ALG_NAME]: compressor name.

REQ-2: struct zswap_entry:
- swpentry: `swp_entry_t` — both the xarray key (`swp_offset`) and a back-reference.
- length: compressed length in bytes; PAGE_SIZE means "incompressible — stored as-is".
- referenced: second-chance bit; set on store / swapin, cleared by shrinker.
- pool: owning zswap_pool (refcounted via percpu_ref while entry alive).
- handle: zsmalloc allocation handle into pool.zs_pool.
- objcg: per-objcg charging anchor; NULL for unaccounted entries.
- lru: linkage into zswap_list_lru (per-node-per-memcg).

REQ-3: Per-swap-type tree fan-out:
- zswap_trees[type] = array of `nr_zswap_trees[type]` xarrays.
- ZSWAP_ADDRESS_SPACE_SHIFT = 14; ZSWAP_ADDRESS_SPACE_PAGES = 1<<14 (16 384 pages = 64 MiB per tree).
- swap_zswap_tree(swp) = &zswap_trees[swp_type(swp)][swp_offset(swp) >> 14].

REQ-4: Sysfs/module parameters:
- enabled (bool, 0644): runtime on/off (`zswap_enabled_param_set` validates pool exists).
- compressor (charp, 0644): live-swap compressor (creates new pool, makes it current, drops old).
- zpool (charp, 0644): backend selector; in 7.1 only zsmalloc remains.
- max_pool_percent (uint, 0644, default 20): pool size as % of totalram.
- accept_threshold_percent (uint, 0644, default 90): high-watermark for accepting new stores after `zswap_pool_reached_full`.
- shrinker_enabled (bool, 0644): registered shrinker on/off.

REQ-5: zswap_compress(page, entry, pool):
- acomp_ctx = raw_cpu_ptr(pool.acomp_ctx); mutex_lock(&acomp_ctx.mutex).
- sg_set_page(input, page, PAGE_SIZE, 0); sg_init_one(output, acomp_ctx.buffer, PAGE_SIZE).
- crypto_wait_req(crypto_acomp_compress(acomp_ctx.req), &acomp_ctx.wait).
- dlen = acomp_ctx.req.dlen.
- if comp_ret ∨ !dlen ∨ dlen ≥ PAGE_SIZE:
  - if !mem_cgroup_zswap_writeback_enabled(folio_memcg(page_folio(page))): reject.
  - else: dlen = PAGE_SIZE; dst = kmap_local_page(page)  /* store uncompressed */.
- handle = zs_malloc(pool.zs_pool, dlen, GFP_NOWAIT|__GFP_NORETRY|__GFP_HIGHMEM|__GFP_MOVABLE, page_to_nid(page)).
- zs_obj_write(pool.zs_pool, handle, dst, dlen).
- entry.handle = handle; entry.length = dlen.
- mutex_unlock.

REQ-6: zswap_decompress(entry, folio):
- acomp_ctx = raw_cpu_ptr(entry.pool.acomp_ctx); mutex_lock(&acomp_ctx.mutex).
- zs_obj_read_sg_begin(pool.zs_pool, entry.handle, input, entry.length).
- if entry.length == PAGE_SIZE:
  - dst = kmap_local_folio(folio, 0).
  - memcpy_from_sglist(dst, input, 0, PAGE_SIZE).
  - kunmap_local(dst); flush_dcache_folio(folio).
- else:
  - sg_set_folio(output, folio, PAGE_SIZE, 0).
  - crypto_wait_req(crypto_acomp_decompress(req), &wait); dlen = req.dlen.
- zs_obj_read_sg_end(pool.zs_pool, entry.handle).
- mutex_unlock.
- return ret == 0 ∧ dlen == PAGE_SIZE.

REQ-7: zswap_store_page(page, objcg, pool) → bool:
- entry = zswap_entry_cache_alloc(GFP_KERNEL, page_to_nid(page)).
- if !zswap_compress(page, entry, pool): goto compress_failed.
- old = xa_store(swap_zswap_tree(swp), swp_offset(swp), entry, GFP_KERNEL).
- if xa_is_err(old): goto store_failed.
- if old: zswap_entry_free(old)  /* replaced stale */.
- zswap_pool_get(pool); if objcg: obj_cgroup_get(objcg); obj_cgroup_charge_zswap(objcg, entry.length).
- atomic_long_inc(&zswap_stored_pages); if length==PAGE_SIZE: zswap_stored_incompressible_pages++.
- entry.pool = pool; entry.swpentry = swp; entry.objcg = objcg; entry.referenced = true.
- if entry.length: zswap_lru_add(&zswap_list_lru, entry)  /* incompressibles excluded from LRU */.
- return true.

REQ-8: zswap_store(folio) → bool:
- VM_WARN_ON_ONCE(!folio_test_locked(folio)).
- VM_WARN_ON_ONCE(!folio_test_swapcache(folio)).
- if !zswap_enabled: goto check_old (invalidate stale, return false).
- objcg = get_obj_cgroup_from_folio(folio).
- if objcg ∧ !obj_cgroup_may_zswap(objcg):
  - memcg = get_mem_cgroup_from_objcg(objcg).
  - if shrink_memcg(memcg): goto put_objcg.
- if zswap_check_limits(): goto put_objcg.
- pool = zswap_pool_current_get(); if !pool: goto put_objcg.
- for page in folio: if !zswap_store_page(page, objcg, pool): goto put_pool.
- count_vm_events(ZSWPOUT, nr_pages).
- ret = true.
- on failure or !ret: invalidate stale entries at all offsets in this folio so writeback doesn't overwrite new data.
- if !ret ∧ zswap_pool_reached_full: queue_work(shrink_wq, &zswap_shrink_work).

REQ-9: zswap_load(folio) → int:
- VM_WARN_ON_ONCE(!folio_test_locked); VM_WARN_ON_ONCE(!folio_test_swapcache).
- if zswap_never_enabled(): return -ENOENT.
- if WARN_ON_ONCE(folio_test_large(folio)): folio_unlock; return -EINVAL  /* large folios not supported */.
- entry = xa_load(swap_zswap_tree(swp), swp_offset(swp)).
- if !entry: return -ENOENT  /* not in zswap; caller must do real swapin */.
- if !zswap_decompress(entry, folio): folio_unlock; return -EIO.
- folio_mark_uptodate(folio).
- count_vm_event(ZSWPIN); if entry.objcg: count_objcg_events(entry.objcg, ZSWPIN, 1).
- /* Eager invalidate: swapcache now authoritative */
- folio_mark_dirty(folio); xa_erase(tree, offset); zswap_entry_free(entry).
- folio_unlock(folio); return 0.

REQ-10: zswap_invalidate(swp):
- tree = swap_zswap_tree(swp).
- if xa_empty(tree): return.
- entry = xa_erase(tree, swp_offset(swp)).
- if entry: zswap_entry_free(entry).

REQ-11: zswap_entry_free(entry):
- if entry.objcg: obj_cgroup_uncharge_zswap(entry.objcg, entry.length); obj_cgroup_put(entry.objcg).
- if entry.length: zswap_lru_del(&zswap_list_lru, entry).
- zs_free(entry.pool.zs_pool, entry.handle).
- atomic_long_dec(&zswap_stored_pages); if length==PAGE_SIZE: stored_incompressible_pages--.
- zswap_pool_put(entry.pool).
- zswap_entry_cache_free(entry).

REQ-12: zswap_writeback_entry(entry, swpentry):
- si = get_swap_device(swpentry); if !si: return -EEXIST.
- mpol = get_task_policy(current).
- folio = swap_cache_alloc_folio(swpentry, GFP_KERNEL, mpol, NO_INTERLEAVE_INDEX, &allocated).
- if !folio: return -ENOMEM.
- if !allocated: ret = -EEXIST; goto out  /* raced with swapin or shrinker */.
- /* Re-validate under tree */
- tree = swap_zswap_tree(swpentry).
- if entry != xa_load(tree, swp_offset(swpentry)): ret = -ENOMEM; goto out.
- if !zswap_decompress(entry, folio): ret = -EIO; goto out.
- /* Hand to swap_writeout(); folio remains locked + dirty */
- xa_erase(tree, swp_offset(swpentry)).
- zswap_entry_free(entry).
- count_vm_event(ZSWPWB).
- swap_writeout(folio, ...).

REQ-13: shrink_memcg_cb (LRU walker, second-chance):
- if entry.referenced: entry.referenced = false; return LRU_ROTATE  /* second-chance */.
- list_move_tail(item, &l.list)  /* rotate before drop */.
- swpentry = entry.swpentry  /* copy to stack: entry may be freed after lru_unlock */.
- spin_unlock(&l.lock).
- writeback_result = zswap_writeback_entry(entry, swpentry).
- if writeback_result: zswap_reject_reclaim_fail++; return LRU_RETRY (or LRU_STOP on -EEXIST + warm-region hint).
- else: zswap_written_back_pages++; return LRU_REMOVED_RETRY.

REQ-14: zswap_shrinker_scan / _count:
- scan: if !zswap_shrinker_enabled ∨ !mem_cgroup_zswap_writeback_enabled(sc.memcg): return SHRINK_STOP.
- list_lru_shrink_walk(&zswap_list_lru, sc, &shrink_memcg_cb, &encountered_in_swapcache).
- if encountered_in_swapcache: SHRINK_STOP  /* shrinking into warm region */.
- count: returns a heuristic of evictable compressed pages adjusted by recent disk-swapin pressure on the memcg-lruvec (`zswap_lruvec_state.nr_disk_swapins`).

REQ-15: shrink_worker (async, queued from store-path when pool reached full):
- thr = zswap_accept_thr_pages().
- iter memcgs from zswap_next_shrink cursor (mem_cgroup_iter, online only).
- ret = shrink_memcg(memcg); mem_cgroup_put(memcg).
- on -ENOENT: continue (no candidates in this memcg).
- on other err: ++attempts; if failures hit MAX_RECLAIM_RETRIES: break.
- cond_resched.
- continue while zswap_total_pages() > thr.

REQ-16: shrink_memcg(memcg):
- if !mem_cgroup_zswap_writeback_enabled(memcg): return -ENOENT.
- if memcg ∧ !mem_cgroup_online(memcg): return -ENOENT  /* zombie skip */.
- for each N_NORMAL_MEMORY node:
  - list_lru_walk_one(&zswap_list_lru, nid, memcg, &shrink_memcg_cb, NULL, &nr_to_walk=1).
- return shrunk ? 0 : -EAGAIN (or -ENOENT if nothing scanned).

REQ-17: zswap_pool_create(compressor):
- pool = kzalloc.
- pool.zs_pool = zs_create_pool("zswap%x" % atomic_inc_return(&zswap_pools_count)).
- pool.acomp_ctx = alloc_percpu_gfp(*pool.acomp_ctx, GFP_KERNEL|__GFP_ZERO).
- cpuhp_state_add_instance(CPUHP_MM_ZSWP_POOL_PREPARE, &pool.node)  /* per-CPU acomp_ctx setup */.
- percpu_ref_init(&pool.ref, __zswap_pool_empty, PERCPU_REF_ALLOW_REINIT, GFP_KERNEL).

REQ-18: Pool ref lifecycle:
- zswap_pool_tryget: percpu_ref_tryget; returns 0 if pool already dying.
- zswap_pool_get: percpu_ref_get.
- zswap_pool_put: percpu_ref_put; on zero → __zswap_pool_empty queues release_work.
- __zswap_pool_release (worker): synchronize_rcu(); percpu_ref_exit; zswap_pool_destroy.
- zswap_pool_destroy: cpuhp_state_remove_instance; free per-CPU acomp_ctx; zs_destroy_pool; kfree.

REQ-19: zswap_check_limits:
- cur = zswap_total_pages(); max = zswap_max_pages().
- if cur ≥ max: zswap_pool_limit_hit++; zswap_pool_reached_full = true; return true (reject store).
- if zswap_pool_reached_full ∧ cur > zswap_accept_thr_pages(): return true (still over high-watermark).
- if cur ≤ zswap_accept_thr_pages(): zswap_pool_reached_full = false.
- return false.

REQ-20: zswap_swapon(type, nr_pages):
- nr = DIV_ROUND_UP(nr_pages, ZSWAP_ADDRESS_SPACE_PAGES).
- trees = kvzalloc(nr * sizeof(struct xarray)); if !trees: return -ENOMEM.
- for i in 0..nr: xa_init(&trees[i]).
- nr_zswap_trees[type] = nr; zswap_trees[type] = trees.

REQ-21: zswap_swapoff(type):
- for i in 0..nr_zswap_trees[type]: WARN_ON_ONCE(!xa_empty(&trees[i])).  /* try_to_unuse() already invalidated */
- kvfree(trees); nr_zswap_trees[type] = 0; zswap_trees[type] = NULL.

REQ-22: Per-memcg LRU + offline cleanup:
- zswap_lru_add / zswap_lru_del use memcg → list_lru_one mapping.
- zswap_memcg_offline_cleanup: advance zswap_next_shrink past offlining memcg under zswap_shrink_lock; release reference; ensures cursor never holds an offline-only ref.

REQ-23: debugfs:
- /sys/kernel/debug/zswap/{pool_limit_hit, reject_reclaim_fail, reject_alloc_fail, reject_kmemcache_fail, reject_compress_fail, reject_compress_poor, decompress_fail, written_back_pages, pool_total_size, stored_pages, stored_incompressible_pages}.

## Acceptance Criteria

- [ ] AC-1: zswap_enabled=false: zswap_store returns false; existing entries at same swp invalidated.
- [ ] AC-2: zswap_store: small page (compressible) → entry.length < PAGE_SIZE; xarray populated; LRU populated.
- [ ] AC-3: zswap_store: incompressible page (dlen ≥ PAGE_SIZE) with writeback-enabled memcg → stored as-is (length == PAGE_SIZE); NOT on LRU.
- [ ] AC-4: zswap_store: incompressible page with writeback-disabled memcg → rejected; zswap_reject_compress_poor++.
- [ ] AC-5: zswap_load: hit → folio uptodate + dirty; xarray entry erased; ZSWPIN counted.
- [ ] AC-6: zswap_load: miss → -ENOENT; folio stays locked for caller to do real swapin.
- [ ] AC-7: zswap_load: large folio → -EINVAL; folio unlocked.
- [ ] AC-8: zswap_invalidate(swp): xa_erase + zswap_entry_free; idempotent on empty tree.
- [ ] AC-9: zswap_check_limits: ≥ max_pool_percent → rejects; queues shrink_work when full.
- [ ] AC-10: shrink_memcg_cb: entry.referenced → LRU_ROTATE + clear bit (second-chance honored).
- [ ] AC-11: shrink_memcg_cb: writeback path eagerly removes entry from tree + frees.
- [ ] AC-12: zswap_swapon(type, n) creates ceil(n / 16384) xarrays; swapoff WARNs if non-empty.
- [ ] AC-13: zswap_pool ref drop to zero → release_work → synchronize_rcu → destroy (no use-after-free).
- [ ] AC-14: live compressor swap: writing /sys/.../compressor with new name → new pool, becomes current, old retained until refs drop.
- [ ] AC-15: memcg offline → zswap_next_shrink cursor advanced past dying memcg before drop.

## Architecture

```
struct ZswapEntry {
  swpentry: SwpEntryT,
  length: u32,
  referenced: bool,
  pool: NonNull<ZswapPool>,
  handle: usize,                        // zsmalloc handle
  objcg: Option<NonNull<ObjCgroup>>,
  lru: ListHead,                        // zswap_list_lru linkage
}

struct AcompCtx {
  acomp: NonNull<CryptoAcomp>,
  req: NonNull<AcompReq>,
  wait: CryptoWait,
  buffer: NonNull<u8>,                  // PAGE_SIZE staging
  mutex: Mutex,
}

struct ZswapPool {
  zs_pool: NonNull<ZsPool>,
  acomp_ctx: PerCpu<AcompCtx>,
  ref: PercpuRef,                       // ALLOW_REINIT
  list: ListHead,                       // zswap_pools linkage
  release_work: WorkStruct,
  node: HlistNode,                      // CPUHP_MM_ZSWP_POOL_PREPARE
  tfm_name: [u8; CRYPTO_MAX_ALG_NAME],
}

struct Zswap {
  trees: [Option<Box<[XArray<ZswapEntry>]>>; MAX_SWAPFILES],
  nr_zswap_trees: [u32; MAX_SWAPFILES],
  pools: ListHead,
  pools_lock: SpinLock,
  pools_count: AtomicI32,
  has_pool: bool,
  init_state: ZswapInitType,
  init_lock: Mutex,
  list_lru: ListLru,
  shrink_lock: SpinLock,
  next_shrink: Option<NonNull<MemCgroup>>,
  shrink_work: WorkStruct,
  shrink_wq: NonNull<Workqueue>,
  shrinker: NonNull<Shrinker>,
  entry_cache: NonNull<KmemCache>,
  enabled: bool,
  pool_reached_full: bool,
  // Module params
  max_pool_percent: u32,                // default 20
  accept_thr_percent: u32,              // default 90
  shrinker_enabled: bool,
  compressor: String,
}
```

`Zswap::store(folio) -> bool`:
1. VM_WARN_ON_ONCE(!folio_test_locked(folio)).
2. VM_WARN_ON_ONCE(!folio_test_swapcache(folio)).
3. if !self.enabled: goto check_old.
4. objcg = get_obj_cgroup_from_folio(folio).
5. if objcg ∧ !obj_cgroup_may_zswap(objcg):
   - memcg = get_mem_cgroup_from_objcg(objcg).
   - if self.shrink_memcg(memcg): mem_cgroup_put(memcg); goto put_objcg.
   - mem_cgroup_put(memcg).
6. if self.check_limits(): goto put_objcg.
7. pool = self.current_pool_get().
8. if pool.is_none(): goto put_objcg.
9. if objcg.is_some(): memcg_list_lru_alloc(memcg, &self.list_lru, GFP_KERNEL).
10. for page in folio.pages():
    - if !self.store_page(page, objcg, pool): goto put_pool.
11. count_vm_events(ZSWPOUT, nr_pages); ret = true.
12. on label put_pool: self.pool_put(pool).
13. on label put_objcg: obj_cgroup_put(objcg).
14. if !ret ∧ self.pool_reached_full: queue_work(&self.shrink_wq, &self.shrink_work).
15. on label check_old (always, if !ret): for offset in folio: erase stale entry.
16. return ret.

`Zswap::store_page(page, objcg, pool) -> bool`:
1. swp = page_swap_entry(page).
2. entry = ZswapEntry::alloc(GFP_KERNEL, page_to_nid(page))?; on fail: reject_kmemcache_fail++.
3. if !self.compress(page, entry, pool): goto compress_failed.
4. tree = self.tree_for_swp(swp).
5. old = xa_store(tree, swp_offset(swp), entry, GFP_KERNEL).
6. if xa_is_err(old): reject_alloc_fail++; goto store_failed.
7. if old.is_some(): self.entry_free(old).
8. self.pool_get(pool).
9. if objcg.is_some(): obj_cgroup_get(objcg); obj_cgroup_charge_zswap(objcg, entry.length).
10. atomic_long_inc(&zswap_stored_pages); if length==PAGE_SIZE: stored_incompressible_pages++.
11. /* Publish-after-store: writeback is excluded until lru_add */
12. entry.pool = pool; entry.swpentry = swp; entry.objcg = objcg; entry.referenced = true.
13. if entry.length != 0: self.lru_add(&self.list_lru, entry).
14. return true.

`Zswap::compress(page, entry, pool) -> bool`:
1. acomp_ctx = raw_cpu_ptr(pool.acomp_ctx).
2. mutex_lock(&acomp_ctx.mutex).
3. dst = acomp_ctx.buffer.
4. sg_init_table(&input, 1); sg_set_page(&input, page, PAGE_SIZE, 0).
5. sg_init_one(&output, dst, PAGE_SIZE).
6. acomp_request_set_params(acomp_ctx.req, &input, &output, PAGE_SIZE, PAGE_SIZE).
7. comp_ret = crypto_wait_req(crypto_acomp_compress(acomp_ctx.req), &acomp_ctx.wait).
8. dlen = acomp_ctx.req.dlen.
9. if comp_ret != 0 ∨ dlen == 0 ∨ dlen ≥ PAGE_SIZE:
   - rcu_read_lock; wb_enabled = mem_cgroup_zswap_writeback_enabled(folio_memcg(page_folio(page))); rcu_read_unlock.
   - if !wb_enabled: zswap_reject_compress_poor++ / fail++; goto unlock.
   - comp_ret = 0; dlen = PAGE_SIZE; dst = kmap_local_page(page); mapped = true.
10. handle = zs_malloc(pool.zs_pool, dlen, GFP_NOWAIT|__GFP_NORETRY|__GFP_HIGHMEM|__GFP_MOVABLE, page_to_nid(page)).
11. if IS_ERR_VALUE(handle): alloc_ret = err; goto unlock.
12. zs_obj_write(pool.zs_pool, handle, dst, dlen).
13. entry.handle = handle; entry.length = dlen.
14. on label unlock: if mapped: kunmap_local(dst); mutex_unlock(&acomp_ctx.mutex).
15. return comp_ret == 0 ∧ alloc_ret == 0.

`Zswap::load(folio) -> i32`:
1. VM_WARN_ON_ONCE locked/swapcache.
2. if self.never_enabled(): return -ENOENT.
3. if WARN_ON_ONCE(folio_test_large(folio)): folio_unlock; return -EINVAL.
4. tree = self.tree_for_swp(folio.swap).
5. entry = xa_load(tree, swp_offset(folio.swap)); if entry.is_none(): return -ENOENT.
6. if !self.decompress(entry, folio): folio_unlock; return -EIO.
7. folio_mark_uptodate(folio).
8. count_vm_event(ZSWPIN); if entry.objcg: count_objcg_events(entry.objcg, ZSWPIN, 1).
9. folio_mark_dirty(folio).
10. xa_erase(tree, swp_offset(folio.swap)).
11. self.entry_free(entry).
12. folio_unlock(folio); return 0.

`Zswap::writeback_entry(entry, swpentry) -> i32`:
1. si = get_swap_device(swpentry); if si.is_none(): return -EEXIST.
2. mpol = get_task_policy(current()).
3. folio = swap_cache_alloc_folio(swpentry, GFP_KERNEL, mpol, NO_INTERLEAVE_INDEX, &allocated).
4. put_swap_device(si); if folio.is_none(): return -ENOMEM.
5. if !allocated: ret = -EEXIST; goto out.
6. tree = self.tree_for_swp(swpentry).
7. if entry != xa_load(tree, swp_offset(swpentry)): ret = -ENOMEM; goto out.
8. if !self.decompress(entry, folio): ret = -EIO; goto out.
9. xa_erase(tree, swp_offset(swpentry)); self.entry_free(entry).
10. count_vm_event(ZSWPWB).
11. /* Folio remains locked + dirty; the swap_writeout path bio-writes it */
12. swap_writeout(folio).
13. on label out: folio_unlock(folio); folio_put(folio); return ret.

`Zswap::shrink_memcg_cb(item, lru_one, arg) -> LruStatus`:
1. entry = container_of(item, ZswapEntry, lru).
2. if entry.referenced: entry.referenced = false; return LRU_ROTATE.
3. list_move_tail(item, &lru_one.list)  /* rotate before dropping lock */.
4. swpentry = entry.swpentry  /* copy to stack — entry may be freed after unlock */.
5. spin_unlock(&lru_one.lock).
6. wb_result = self.writeback_entry(entry, swpentry).
7. if wb_result != 0:
   - zswap_reject_reclaim_fail++.
   - if wb_result == -EEXIST ∧ encountered_swapcache: *encountered = true; return LRU_STOP.
   - return LRU_RETRY.
8. zswap_written_back_pages++; return LRU_REMOVED_RETRY.

`Zswap::shrink_worker()`:
1. thr = self.accept_thr_pages().
2. loop:
   - spin_lock(&self.shrink_lock).
   - do: memcg = mem_cgroup_iter(NULL, self.next_shrink, NULL); self.next_shrink = memcg.
     while memcg ∧ !mem_cgroup_tryget_online(memcg).
   - spin_unlock(&self.shrink_lock).
   - if !memcg: if !attempts ∧ ++failures == MAX_RECLAIM_RETRIES: break; goto resched.
   - ret = self.shrink_memcg(memcg); mem_cgroup_put(memcg).
   - if ret == -ENOENT: continue.
   - ++attempts; if ret ∧ ++failures == MAX_RECLAIM_RETRIES: break.
   - on label resched: cond_resched.
   - while self.total_pages() > thr.

`Zswap::pool_create(compressor) -> Option<NonNull<ZswapPool>>`:
1. if !self.has_pool ∧ compressor == ZSWAP_PARAM_UNSET: return None.
2. pool = kzalloc(*pool).
3. snprintf(name, "zswap%x", atomic_inc_return(&self.pools_count)).
4. pool.zs_pool = zs_create_pool(name).
5. strscpy(pool.tfm_name, compressor).
6. pool.acomp_ctx = alloc_percpu_gfp(*acomp_ctx, GFP_KERNEL|__GFP_ZERO).
7. cpuhp_state_add_instance(CPUHP_MM_ZSWP_POOL_PREPARE, &pool.node).
8. percpu_ref_init(&pool.ref, Self::__pool_empty, PERCPU_REF_ALLOW_REINIT, GFP_KERNEL).
9. INIT_LIST_HEAD(&pool.list).
10. return Some(pool).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `entry_xarray_pool_ref_paired` | INVARIANT | per-store_page success: pool_get matched by entry_free's pool_put. |
| `entry_objcg_charge_paired` | INVARIANT | per-store_page success: objcg charge matched by entry_free uncharge. |
| `lru_membership_iff_length_nonzero` | INVARIANT | per-entry: on-LRU ⟺ length > 0 ∧ length < PAGE_SIZE. |
| `incompressible_excluded_from_lru` | INVARIANT | per-entry length==PAGE_SIZE: never on LRU. |
| `writeback_disabled_rejects_incompressible` | INVARIANT | per-compress with !writeback-enabled memcg + comp-fail: store rejected. |
| `large_folio_load_rejected` | INVARIANT | per-load: folio_test_large ⟹ -EINVAL. |
| `pool_ref_zero_then_rcu_grace` | INVARIANT | per-pool_release: synchronize_rcu before destroy. |
| `shrink_cb_second_chance_then_clear` | INVARIANT | per-cb: referenced ⟹ rotate ∧ clear. |
| `cb_uses_stack_swpentry_after_unlock` | INVARIANT | per-cb: entry not dereferenced after lru_unlock; swpentry copied to stack. |
| `acomp_ctx_mutex_paired` | INVARIANT | per-(compress, decompress): mutex_lock/unlock paired. |
| `next_shrink_cursor_no_offline_ref` | INVARIANT | per-memcg_offline_cleanup: zswap_next_shrink never holds offline-only ref. |
| `tree_idx_in_range` | INVARIANT | per-tree_for_swp: result tree idx < nr_zswap_trees[type]. |
| `swapoff_trees_empty` | INVARIANT | per-swapoff: WARN_ON_ONCE on non-empty xarray. |

### Layer 2: TLA+

`mm/zswap.tla`:
- Per-store + per-load + per-invalidate + per-writeback + per-shrinker concurrent model.
- Properties:
  - `safety_load_after_store_returns_zero` — per-(store, load) of same swp with no intervening invalidate: load returns 0 ∧ folio uptodate.
  - `safety_load_after_invalidate_returns_enoent` — per-(store, invalidate, load): load returns -ENOENT.
  - `safety_load_after_writeback_returns_enoent` — per-(store, shrink-writeback, load): load returns -ENOENT.
  - `safety_no_concurrent_dual_store_same_swp` — per-swp: at most one zswap_entry in tree at any time.
  - `safety_pool_use_after_free_free` — per-pool ref-zero: no thread holds pointer past release_work.
  - `safety_writeback_validates_tree_match` — per-writeback: entry != xa_load ⟹ -ENOMEM; no overwrite of newer entry.
  - `safety_check_limits_enforced` — per-store: total_pages > max ⟹ store rejected.
  - `liveness_pool_full_eventually_shrinks` — per-store rejection with pool_reached_full: shrink_work eventually runs.
  - `liveness_shrink_worker_terminates` — per-shrink_worker: bounded by failures ≤ MAX_RECLAIM_RETRIES.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Zswap::store` post: ret ⟹ ∀ page: tree.contains(swp_offset(swp)) ∨ stale erased | `Zswap::store` |
| `Zswap::store_page` post: success ⟹ pool refcount +1 ∧ objcg refcount +1 (if objcg) ∧ entry published in tree | `Zswap::store_page` |
| `Zswap::compress` post: success ⟹ entry.length ∈ [1, PAGE_SIZE] ∧ handle valid in zs_pool | `Zswap::compress` |
| `Zswap::decompress` post: success ⟹ folio[0..PAGE_SIZE] = decompressed data | `Zswap::decompress` |
| `Zswap::load` post: 0 ⟹ folio uptodate ∧ tree erased ∧ entry freed | `Zswap::load` |
| `Zswap::load` post: -ENOENT ⟹ folio still locked | `Zswap::load` |
| `Zswap::invalidate` post: tree.get(swp_offset(swp)) == None | `Zswap::invalidate` |
| `Zswap::writeback_entry` post: 0 ⟹ tree erased ∧ entry freed ∧ folio queued for swap_writeout | `Zswap::writeback_entry` |
| `Zswap::shrink_memcg_cb` post: referenced ⟹ LRU_ROTATE ∧ entry.referenced = false | `Zswap::shrink_memcg_cb` |
| `ZswapEntry::free` post: pool.refcount -1 ∧ objcg.refcount -1 ∧ zs_free called ∧ stored_pages -1 | `ZswapEntry::free` |
| `Zswap::check_limits` post: returns true ⟺ total_pages ≥ max_pages ∨ (pool_reached_full ∧ total > accept_thr) | `Zswap::check_limits` |

### Layer 4: Verus/Creusot functional

`Per-folio swap_writeout intercept: compress (acomp) → zsmalloc → xa_store(tree, offset, entry) → list_lru_add. Per-folio swap_readahead intercept: xa_load → decompress → folio_mark_uptodate → xa_erase + entry_free. Per-shrinker tick: list_lru_walk_one (second-chance) → writeback_entry (decompress + swap_cache + swap_writeout) → entry_free. Semantic equivalence per Documentation/admin-guide/mm/zswap.rst and include/linux/zswap.h.`

## Hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening.)

zswap reinforcement:

- **Per-folio-locked + swapcache-asserted at store/load** — defense against per-store-on-uncached folio race.
- **Per-large-folio load rejected with -EINVAL** — defense against per-partial-load corruption (large folios may be only partially in zswap).
- **Per-entry copied swpentry on stack before lru_unlock** — defense against per-UAF in shrink_memcg_cb when entry freed concurrently.
- **Per-writeback xa_load == entry re-check** — defense against per-stale-data overwrite (entry swapped under us).
- **Per-pool percpu_ref + RCU grace before destroy** — defense against per-pool UAF on hot pool swap.
- **Per-stale-entry invalidation on store failure (check_old)** — defense against per-writeback-overwrites-new-data.
- **Per-load eager invalidate after decompress** — defense against per-double-copy (compressed + uncompressed) memory waste + incoherence.
- **Per-incompressible writeback-disabled-memcg rejection** — defense against per-pool-fill-with-zero-savings.
- **Per-acomp_ctx per-CPU + mutex** — defense against per-cross-CPU acomp_req state corruption.
- **Per-shrinker second-chance bit cleared atomically** — defense against per-immediate-reclaim-after-touch thrashing.
- **Per-zswap_pool unique zsmalloc name** — defense against per-pool-name collision.
- **Per-memcg offline cursor cleanup** — defense against per-shrinker-holds-dead-memcg ref leak.
- **Per-zswap_next_shrink under shrink_lock + tryget_online** — defense against per-memcg-offline-during-iteration UAF.
- **Per-zs_malloc GFP_NOWAIT|__GFP_NORETRY** — defense against per-allocator deadlock under direct reclaim.
- **Per-swapoff WARN on non-empty tree** — defense against per-swapoff-loses-entries leak.
- **Per-shrinker MAX_RECLAIM_RETRIES bound** — defense against per-shrinker-infinite-loop livelock.
- **Per-compress reject counters surfaced via debugfs** — defense against per-silent-degradation; admin observability.
- **Per-mem_cgroup_zswap_writeback_enabled gating** — defense against per-cgroup-policy-violation forced writeback.
- **Per-objcg charge_zswap paired with uncharge** — defense against per-memcg accounting leak.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- mm/zsmalloc.c allocator internals (covered in `zsmalloc.md` Tier-3)
- mm/swapfile.c / mm/swap_state.c swap-device + swap-cache (covered in `swap.md` Tier-3)
- mm/memcontrol.c memcg charging primitives (covered in `memcg.md` Tier-3)
- crypto/acompress.c crypto_acomp framework (out of mm Tier-3 scope)
- Documentation/admin-guide/mm/zswap.rst (admin manual; not behavioral)
- Implementation code
