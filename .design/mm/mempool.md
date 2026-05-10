# Tier-3: mm/mempool.c — Guaranteed-allocation memory pools

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: mm/00-overview.md
upstream-paths:
  - mm/mempool.c (~766 lines)
  - include/linux/mempool.h
  - Documentation/core-api/mm-api.rst (mempool section)
-->

## Summary

A **mempool** is a fixed-size reserved pool of pre-allocated elements that guarantees forward progress for the critical I/O paths even when the underlying slab/page allocator fails. Per-pool: `struct mempool` carries `min_nr` (reserve count), `curr_nr` (currently reserved), an `elements[]` array of opaque void pointers, plus user-supplied `alloc_fn` / `free_fn` callbacks and `pool_data`. Per-`mempool_alloc(pool, gfp_mask)`: first attempts `alloc_fn(adjusted_gfp, pool_data)` with `__GFP_NOMEMALLOC | __GFP_NORETRY | __GFP_NOWARN` and reclaim/IO masked off; on failure dips into the reserved `elements[]` array under `pool->lock`; on reserve exhaustion either returns NULL (NOWAIT) or `io_schedule_timeout(5*HZ)` waits on `pool->wait` until a `mempool_free` wakes it. Per-`mempool_free`: refills the reserve when `curr_nr < min_nr`, otherwise hands the element to `free_fn`. Critical for: swap-out (bio submission cannot fail under memory pressure), md/dm/bio-set, fs journal writeback — paths that would deadlock if a memory failure during reclaim could cancel the very allocation that is about to free memory.

This Tier-3 covers `mm/mempool.c` (~766 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct mempool` (mempool_s) | per-pool state | `Mempool` |
| `mempool_init_node()` | per-init (in-place, NUMA) | `Mempool::init_node` |
| `mempool_init()` / `mempool_init_noprof` | per-init (kmalloc-embedded) | `Mempool::init` |
| `mempool_create_node_noprof()` | per-create (heap-allocated) | `Mempool::create_node` |
| `mempool_create()` | per-create (default node) | `Mempool::create` |
| `mempool_exit()` | per-teardown (in-place) | `Mempool::exit` |
| `mempool_destroy()` | per-teardown + free pool | `Mempool::destroy` |
| `mempool_resize()` | per-grow/shrink `min_nr` | `Mempool::resize` |
| `mempool_alloc_noprof()` | per-alloc (fall back to reserve) | `Mempool::alloc` |
| `mempool_alloc_preallocated()` | per-alloc-reserve-only | `Mempool::alloc_preallocated` |
| `mempool_alloc_bulk_noprof()` | per-bulk-alloc | `Mempool::alloc_bulk` |
| `mempool_alloc_from_pool()` | per-internal-dip-reserve | `Mempool::alloc_from_pool` |
| `mempool_adjust_gfp()` | per-mask GFP_DIRECT_RECLAIM/IO | `Mempool::adjust_gfp` |
| `mempool_free()` | per-free (refill-or-release) | `Mempool::free` |
| `mempool_free_bulk()` | per-bulk-free | `Mempool::free_bulk` |
| `add_element()` / `remove_element()` | per-reserve LIFO | `Mempool::push_reserve` / `pop_reserve` |
| `mempool_alloc_slab()` / `mempool_free_slab()` | per-kmem-cache callbacks | `Mempool::slab_alloc` / `slab_free` |
| `mempool_kmalloc()` / `mempool_kfree()` | per-kmalloc callbacks | `Mempool::kmalloc_alloc` / `kmalloc_free` |
| `mempool_alloc_pages()` / `mempool_free_pages()` | per-page callbacks | `Mempool::pages_alloc` / `pages_free` |
| `poison_element()` / `check_element()` | per-debug-poison reserve | `Mempool::poison` / `check` |
| `kasan_poison_element()` / `kasan_unpoison_element()` | per-KASAN reserve | `Mempool::kasan_poison` / `kasan_unpoison` |

## Compatibility contract

REQ-1: struct mempool layout:
- lock: spinlock_t (irq-safe).
- min_nr: i32 — guaranteed reserve count.
- curr_nr: i32 — currently reserved (∈ [0, max(1, min_nr)]).
- elements: *mut *mut c_void — LIFO array, length = max(1, min_nr).
- pool_data: *mut c_void — opaque user-pointer passed to callbacks.
- alloc: mempool_alloc_t — fn(gfp_t, *mut c_void) -> *mut c_void.
- free: mempool_free_t — fn(*mut c_void, *mut c_void).
- wait: wait_queue_head_t — sleepers blocked on reserve.

REQ-2: mempool_init_node(pool, min_nr, alloc_fn, free_fn, pool_data, gfp_mask, node_id):
- spin_lock_init(&pool.lock).
- pool.min_nr = min_nr; pool.pool_data = pool_data.
- pool.alloc = alloc_fn; pool.free = free_fn.
- init_waitqueue_head(&pool.wait).
- /* Per-zero-min-pool: still allocate 1-slot array for the rare add path */
- pool.elements = kmalloc_array_node(max(1, min_nr), sizeof(void *), gfp_mask, node_id).
- if !pool.elements: return -ENOMEM.
- /* Pre-fill min_nr (or 1, for zero-min-pool) */
- while pool.curr_nr < max(1, pool.min_nr):
  - element = pool.alloc(gfp_mask, pool.pool_data).
  - if !element: mempool_exit(pool); return -ENOMEM.
  - add_element(pool, element).
- return 0.

REQ-3: mempool_create_node_noprof:
- pool = kmalloc_node(sizeof(*pool), gfp_mask | __GFP_ZERO, node_id).
- if !pool: return NULL.
- if mempool_init_node(pool, ...) != 0: kfree(pool); return NULL.
- return pool.

REQ-4: mempool_exit / mempool_destroy:
- /* Drain reserve via free_fn */
- while pool.curr_nr: free(remove_element(pool), pool.pool_data).
- kfree(pool.elements); pool.elements = NULL.
- mempool_destroy: NULL-tolerant; mempool_exit then kfree(pool).
- /* Idempotent for zeroed-but-uninit pool */

REQ-5: mempool_resize(pool, new_min_nr):
- BUG_ON(new_min_nr <= 0); might_sleep().
- spin_lock_irqsave.
- if new_min_nr <= pool.min_nr: shrink in place — drop curr_nr elements via free_fn until curr_nr == new_min_nr; set pool.min_nr = new_min_nr; return 0.
- spin_unlock_irqrestore.
- /* Grow path: alloc new_elements outside lock */
- new_elements = kmalloc_array(new_min_nr, sizeof(void *)).
- spin_lock_irqsave.
- if raced (new_min_nr <= pool.min_nr): unlock; kfree(new_elements); retry.
- memcpy old elements; kfree(pool.elements); pool.elements = new_elements; pool.min_nr = new_min_nr.
- /* Refill outside lock; on failure roll back min_nr to curr_nr and return -ENOMEM */
- while pool.curr_nr < pool.min_nr: unlock; element = alloc(GFP_KERNEL, pool_data); relock; add_element or rollback.
- return 0 or -ENOMEM.

REQ-6: mempool_alloc(pool, gfp_mask):
- VM_WARN_ON_ONCE(gfp_mask & __GFP_ZERO) — caller responsibility to zero.
- might_alloc(gfp_mask).
- gfp_temp = mempool_adjust_gfp(&gfp_mask) — strips __GFP_DIRECT_RECLAIM | __GFP_IO; adds __GFP_NOMEMALLOC | __GFP_NORETRY | __GFP_NOWARN.
- repeat_alloc:
  - if fault-injection forces failure: element = NULL.
  - else: element = pool.alloc(gfp_temp, pool.pool_data).
  - if element: return element.
  - /* alloc_fn failed — dip into reserve */
  - if mempool_alloc_from_pool(pool, &element, 1, 0, gfp_temp) returned 0:
    - if gfp_temp != gfp_mask: gfp_temp = gfp_mask; goto repeat_alloc /* second pass with full reclaim */.
    - if gfp_mask & __GFP_DIRECT_RECLAIM: goto repeat_alloc /* slept on pool.wait; retry */.
  - return element (may be NULL only when ¬(gfp_mask & __GFP_DIRECT_RECLAIM)).

REQ-7: mempool_adjust_gfp(&gfp_mask):
- *gfp_mask |= __GFP_NOMEMALLOC | __GFP_NORETRY | __GFP_NOWARN.
- return *gfp_mask & ~(__GFP_DIRECT_RECLAIM | __GFP_IO).
- Rationale: the mempool reserve replaces emergency reserves; we must never dip into PF_MEMALLOC reserves nor invoke reclaim recursively on the first pass.

REQ-8: mempool_alloc_from_pool(pool, elems, count, allocated, gfp_mask) -> u32:
- spin_lock_irqsave.
- if pool.curr_nr < count - allocated: goto fail.
- for i in 0..count: if !elems[i]: elems[i] = remove_element(pool); allocated++.
- spin_unlock_irqrestore.
- smp_wmb() /* pair with rmb in mempool_free_bulk */.
- for i in 0..count: kmemleak_update_trace(elems[i]).
- return allocated.
- fail:
  - if gfp_mask & __GFP_DIRECT_RECLAIM:
    - DEFINE_WAIT(wait); prepare_to_wait(&pool.wait, &wait, TASK_UNINTERRUPTIBLE).
    - spin_unlock_irqrestore.
    - io_schedule_timeout(5 * HZ) /* periodic wakeup in case alloc_fn newly succeeds */.
    - finish_wait(&pool.wait, &wait).
  - else: spin_unlock_irqrestore.
  - return allocated.

REQ-9: mempool_alloc_preallocated(pool):
- element = NULL.
- mempool_alloc_from_pool(pool, &element, 1, 0, GFP_NOWAIT) /* never sleeps */.
- return element.

REQ-10: mempool_free(element, pool):
- if !element: return.
- /* smp_rmb pairs with mempool_alloc_from_pool wmb */
- smp_rmb().
- if curr_nr < min_nr OR (min_nr == 0 ∧ curr_nr == 0):
  - spin_lock_irqsave; add_element(pool, element); spin_unlock_irqrestore.
  - if wq_has_sleeper(&pool.wait): wake_up(&pool.wait).
  - return.
- /* Reserve full — hand to free_fn */
- pool.free(element, pool.pool_data).

REQ-11: mempool_free_bulk(pool, elems[], count) -> u32:
- smp_rmb(); freed = 0; added = false.
- /* Fast path: refill while reserve below water-mark */
- if curr_nr < min_nr: lock; while curr_nr < min_nr ∧ freed < count: add_element(pool, elems[freed++]); added = true; unlock.
- /* Zero-min edge: at least one waiter cannot be woken by curr_nr < 0 < 0 — push one element */
- else if min_nr == 0 ∧ curr_nr == 0: lock; if curr_nr == 0: add_element(pool, elems[freed++]); added = true; unlock.
- if added ∧ wq_has_sleeper(&pool.wait): wake_up(&pool.wait).
- return freed /* caller frees remaining elems[freed..count] */.

REQ-12: mempool_alloc_bulk(pool, elems[], count, allocated):
- VM_WARN_ON_ONCE(count > pool.min_nr).
- might_alloc(GFP_KERNEL).
- gfp_temp = mempool_adjust_gfp(&gfp_mask).
- repeat_alloc:
  - for i in 0..count: if elems[i]: continue; elems[i] = alloc(gfp_temp, pool_data); if !elems[i]: goto use_pool; allocated++.
  - return 0.
- use_pool: allocated = mempool_alloc_from_pool(...); gfp_temp = gfp_mask; goto repeat_alloc.

REQ-13: add_element / remove_element invariants:
- add_element: BUG_ON(min_nr != 0 ∧ curr_nr >= min_nr); poison; KASAN-poison; elements[curr_nr++] = element.
- remove_element: element = elements[--curr_nr]; BUG_ON(curr_nr < 0); KASAN-unpoison; check_element() validates poison pattern.

REQ-14: Default callback pairs (`include/linux/mempool.h`):
- mempool_alloc_slab(gfp, pool_data) ⇒ kmem_cache_alloc((struct kmem_cache *)pool_data, gfp).
- mempool_free_slab(element, pool_data) ⇒ kmem_cache_free(pool_data, element).
- mempool_kmalloc(gfp, pool_data) ⇒ kmalloc((size_t)pool_data, gfp).
- mempool_kfree(element, pool_data) ⇒ kfree(element).
- mempool_alloc_pages(gfp, pool_data) ⇒ alloc_pages(gfp, (int)pool_data).
- mempool_free_pages(element, pool_data) ⇒ __free_pages(element, (int)pool_data).

REQ-15: Memory barriers:
- mempool_alloc_from_pool: smp_wmb() after curr_nr decrement; mempool_free / free_bulk: smp_rmb() before reading curr_nr.
- Pair documented: an element passed via shared variable without explicit barrier (e.g., one task allocs, another frees) is still observed correctly because the wmb ensures the freer sees the post-decrement curr_nr.

REQ-16: Reserve-exhaustion liveness:
- A NOIO/NOFS allocator stuck on the reserve must be eventually unblocked by a mempool_free that crosses the curr_nr<min_nr threshold; io_schedule_timeout(5*HZ) is the safety net that periodically retries alloc_fn even without a free, in case reclaim has independently freed memory.

REQ-17: Fault injection:
- DECLARE_FAULT_ATTR(fail_mempool_alloc) and DECLARE_FAULT_ATTR(fail_mempool_alloc_bulk) registered under /sys/kernel/debug/fail_mempool_alloc[ _bulk]/ when CONFIG_FAULT_INJECTION_DEBUG_FS.
- should_fail_ex(... FAULT_NOWARN) forces alloc_fn to return NULL to exercise reserve-fallback code paths.

REQ-18: Poison & KASAN contract:
- When CONFIG_DEBUG_SLAB / CONFIG_SLUB_DEBUG_ON: __poison_element fills element with POISON_INUSE pattern; check_element verifies on remove; mismatch ⇒ poison_error WARN + dump.
- When CONFIG_KASAN: kasan_poison_element marks element as freed in shadow memory while in reserve; kasan_unpoison_element on remove.

## Acceptance Criteria

- [ ] AC-1: mempool_init with min_nr = N pre-allocates exactly N elements (curr_nr == N on return).
- [ ] AC-2: mempool_alloc with full reserve uses alloc_fn first; reserve untouched if alloc_fn succeeds.
- [ ] AC-3: mempool_alloc with failing alloc_fn (forced fault) returns a reserved element; curr_nr decrements.
- [ ] AC-4: mempool_alloc with empty reserve and GFP_NOWAIT returns NULL immediately (no schedule).
- [ ] AC-5: mempool_alloc with empty reserve and __GFP_DIRECT_RECLAIM sleeps on pool.wait; mempool_free wakes it.
- [ ] AC-6: mempool_free with curr_nr < min_nr refills reserve; with curr_nr == min_nr calls free_fn.
- [ ] AC-7: mempool_resize grow from N → M (M > N): min_nr == M and curr_nr eventually replenishes to M.
- [ ] AC-8: mempool_resize shrink from M → N drains M-N elements via free_fn and sets min_nr = N.
- [ ] AC-9: mempool_exit on populated pool drains all elements via free_fn and frees elements array.
- [ ] AC-10: mempool_destroy(NULL) is a no-op (does not panic).
- [ ] AC-11: __GFP_NOMEMALLOC always observed in pool->alloc invocation (verified via GFP-mask capture wrapper).
- [ ] AC-12: __GFP_ZERO passed by caller triggers VM_WARN_ON_ONCE.
- [ ] AC-13: Zero-min-pool (min_nr == 0): mempool_alloc-after-mempool_free always succeeds; waiters woken on push.
- [ ] AC-14: mempool_alloc_bulk(count > min_nr) emits VM_WARN_ON_ONCE.
- [ ] AC-15: Fault injection (fail_mempool_alloc) exercises reserve-fallback path on every alloc.

## Architecture

```
struct Mempool {
  lock: SpinLockIrq,                  // protects curr_nr, elements[]
  min_nr: i32,
  curr_nr: AtomicI32,                 // READ_ONCE-eligible fast-path read
  elements: AtomicPtr<*mut c_void>,   // length = max(1, min_nr)
  pool_data: *mut c_void,
  alloc: MempoolAllocFn,              // fn(Gfp, *mut c_void) -> *mut c_void
  free: MempoolFreeFn,                // fn(*mut c_void, *mut c_void)
  wait: WaitQueueHead,
}

type MempoolAllocFn = unsafe extern "C" fn(gfp: Gfp, pool_data: *mut c_void) -> *mut c_void;
type MempoolFreeFn  = unsafe extern "C" fn(element: *mut c_void, pool_data: *mut c_void);
```

`Mempool::init_node(pool, min_nr, alloc_fn, free_fn, pool_data, gfp, node) -> Result<(), Errno>`:
1. spin_lock_init(&pool.lock).
2. pool.min_nr = min_nr; pool.alloc = alloc_fn; pool.free = free_fn; pool.pool_data = pool_data.
3. init_waitqueue_head(&pool.wait).
4. pool.elements = kmalloc_array_node(max(1, min_nr), size_of::<*mut c_void>(), gfp, node).
5. if pool.elements.is_null(): return Err(ENOMEM).
6. while pool.curr_nr < max(1, pool.min_nr):
   - element = (pool.alloc)(gfp, pool.pool_data).
   - if element.is_null(): Mempool::exit(pool); return Err(ENOMEM).
   - Mempool::push_reserve(pool, element).
7. return Ok(()).

`Mempool::adjust_gfp(gfp_mask) -> Gfp`:
1. *gfp_mask |= __GFP_NOMEMALLOC | __GFP_NORETRY | __GFP_NOWARN.
2. return *gfp_mask & !(__GFP_DIRECT_RECLAIM | __GFP_IO).

`Mempool::alloc_from_pool(pool, elems, count, mut allocated, gfp_mask) -> u32`:
1. flags = spin_lock_irqsave(&pool.lock).
2. if pool.curr_nr < count - allocated: goto fail.
3. for i in 0..count: if elems[i].is_null(): elems[i] = Mempool::pop_reserve(pool); allocated += 1.
4. spin_unlock_irqrestore(&pool.lock, flags).
5. smp_wmb() /* pair with smp_rmb in mempool_free* */.
6. for i in 0..count: kmemleak_update_trace(elems[i]).
7. return allocated.
8. fail:
   - if gfp_mask & __GFP_DIRECT_RECLAIM:
     - wait = DEFINE_WAIT().
     - prepare_to_wait(&pool.wait, &wait, TASK_UNINTERRUPTIBLE).
     - spin_unlock_irqrestore(&pool.lock, flags).
     - io_schedule_timeout(5 * HZ).
     - finish_wait(&pool.wait, &wait).
   - else: spin_unlock_irqrestore.
   - return allocated.

`Mempool::alloc(pool, gfp_mask) -> *mut c_void`:
1. VM_WARN_ON_ONCE(gfp_mask & __GFP_ZERO).
2. might_alloc(gfp_mask).
3. gfp_temp = Mempool::adjust_gfp(&mut gfp_mask).
4. loop /* repeat_alloc */:
   - if should_fail_ex(&fail_mempool_alloc, 1, FAULT_NOWARN): element = ptr::null_mut().
   - else: element = (pool.alloc)(gfp_temp, pool.pool_data).
   - if !element.is_null(): return element.
   - if Mempool::alloc_from_pool(pool, &mut [element], 1, 0, gfp_temp) == 0:
     - if gfp_temp != gfp_mask: gfp_temp = gfp_mask; continue.
     - if gfp_mask & __GFP_DIRECT_RECLAIM: continue.
   - return element /* possibly NULL when no direct reclaim */.

`Mempool::alloc_preallocated(pool) -> *mut c_void`:
1. element = ptr::null_mut().
2. Mempool::alloc_from_pool(pool, &mut [element], 1, 0, GFP_NOWAIT).
3. return element.

`Mempool::free(element, pool)`:
1. if element.is_null(): return.
2. smp_rmb().
3. let curr = READ_ONCE(pool.curr_nr).
4. if curr < pool.min_nr OR (pool.min_nr == 0 ∧ curr == 0):
   - flags = spin_lock_irqsave(&pool.lock).
   - if pool.curr_nr < pool.min_nr OR (pool.min_nr == 0 ∧ pool.curr_nr == 0):
     - Mempool::push_reserve(pool, element); added = true.
   - else: added = false.
   - spin_unlock_irqrestore(&pool.lock, flags).
   - if added:
     - if wq_has_sleeper(&pool.wait): wake_up(&pool.wait).
     - return.
5. (pool.free)(element, pool.pool_data).

`Mempool::resize(pool, new_min_nr) -> Result<(), Errno>`:
1. BUG_ON(new_min_nr <= 0); might_sleep().
2. flags = spin_lock_irqsave(&pool.lock).
3. if new_min_nr <= pool.min_nr:
   - while new_min_nr < pool.curr_nr:
     - element = Mempool::pop_reserve(pool).
     - spin_unlock_irqrestore(&pool.lock, flags).
     - (pool.free)(element, pool.pool_data).
     - flags = spin_lock_irqsave(&pool.lock).
   - pool.min_nr = new_min_nr.
   - spin_unlock_irqrestore; return Ok(()).
4. spin_unlock_irqrestore(&pool.lock, flags).
5. new_elements = kmalloc_array(new_min_nr, size_of::<*mut c_void>(), GFP_KERNEL).
6. if new_elements.is_null(): return Err(ENOMEM).
7. flags = spin_lock_irqsave(&pool.lock).
8. if new_min_nr <= pool.min_nr /* raced */: spin_unlock_irqrestore; kfree(new_elements); goto step 2 (retry from top via tail call).
9. memcpy(new_elements, pool.elements, pool.curr_nr * size_of::<*mut c_void>()).
10. kfree(pool.elements); pool.elements = new_elements; pool.min_nr = new_min_nr.
11. while pool.curr_nr < pool.min_nr:
    - spin_unlock_irqrestore(&pool.lock, flags).
    - element = (pool.alloc)(GFP_KERNEL, pool.pool_data).
    - if element.is_null(): /* rollback min_nr to curr_nr; return Err(ENOMEM) */.
    - flags = spin_lock_irqsave(&pool.lock).
    - if pool.curr_nr < pool.min_nr: Mempool::push_reserve(pool, element); else: spin_unlock; (pool.free)(element, pool.pool_data); flags = spin_lock_irqsave.
12. spin_unlock_irqrestore; return Ok(()).

`Mempool::exit(pool)`:
1. while pool.curr_nr > 0:
   - element = Mempool::pop_reserve(pool).
   - (pool.free)(element, pool.pool_data).
2. kfree(pool.elements); pool.elements = ptr::null_mut().

`Mempool::destroy(pool)`:
1. if pool.is_null(): return.
2. Mempool::exit(pool).
3. kfree(pool).

`Mempool::push_reserve(pool, element)`:
1. BUG_ON(pool.min_nr != 0 ∧ pool.curr_nr >= pool.min_nr).
2. Mempool::poison(pool, element).
3. if Mempool::kasan_poison(pool, element): pool.elements[pool.curr_nr] = element; pool.curr_nr += 1.

`Mempool::pop_reserve(pool) -> *mut c_void`:
1. pool.curr_nr -= 1.
2. element = pool.elements[pool.curr_nr].
3. BUG_ON(pool.curr_nr < 0).
4. Mempool::kasan_unpoison(pool, element).
5. Mempool::check(pool, element).
6. return element.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `reserve_lifo_bounded` | INVARIANT | per-push/pop: 0 ≤ curr_nr ≤ max(1, min_nr). |
| `lock_held_during_reserve_mutation` | INVARIANT | per-add_element/remove_element: pool.lock held. |
| `adjust_gfp_strips_reclaim_io` | INVARIANT | per-alloc: gfp_temp & (__GFP_DIRECT_RECLAIM | __GFP_IO) == 0 on first pass. |
| `nomemalloc_always_set` | INVARIANT | per-alloc: callback always observes __GFP_NOMEMALLOC. |
| `nowait_never_sleeps` | INVARIANT | per-alloc with ¬__GFP_DIRECT_RECLAIM: io_schedule_timeout not called. |
| `smp_wmb_after_pop_smp_rmb_before_curr_nr_read` | INVARIANT | per-alloc_from_pool / per-free: barrier pairing intact. |
| `min_nr_invariant_under_resize` | INVARIANT | per-resize: on success pool.min_nr == new_min_nr; on failure rollback to prior min_nr. |

### Layer 2: TLA+

`mm/mempool.tla`:
- Per-init + per-alloc + per-free + per-resize + per-wait/wake.
- Properties:
  - `safety_reserve_never_exceeds_min_nr` — per-pool: curr_nr ≤ max(1, min_nr) always.
  - `safety_alloc_never_returns_null_when_reclaim_set` — per-alloc with __GFP_DIRECT_RECLAIM: returns non-NULL or sleeps; never NULL.
  - `safety_free_into_reserve_below_min_nr` — per-free: curr_nr < min_nr ⟹ element added to reserve, not free_fn.
  - `liveness_waiters_eventually_woken` — per-wait: any waiter eventually woken by a free that crosses the curr_nr threshold OR by io_schedule_timeout expiry.
  - `safety_resize_atomic` — per-resize: observers see either old (min_nr, elements) or new (min_nr, elements) — never a torn pair.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Mempool::init_node` post: curr_nr == max(1, min_nr) ∧ ∀ i. elements[i] ≠ NULL | `Mempool::init_node` |
| `Mempool::alloc` post: ret ≠ NULL ∨ ¬(gfp_mask & __GFP_DIRECT_RECLAIM) | `Mempool::alloc` |
| `Mempool::alloc_from_pool` post: lock released; smp_wmb issued after pop sequence | `Mempool::alloc_from_pool` |
| `Mempool::free` post: curr_nr < min_nr ⟹ element pushed; else free_fn invoked | `Mempool::free` |
| `Mempool::free_bulk` post: returned freed ≤ count; for i < freed: elems[i] set NULL semantically (caller skips them) | `Mempool::free_bulk` |
| `Mempool::resize` post: success ⟹ pool.min_nr == new_min_nr; failure ⟹ pool.min_nr unchanged | `Mempool::resize` |
| `Mempool::exit` post: curr_nr == 0 ∧ elements == NULL | `Mempool::exit` |
| `Mempool::adjust_gfp` post: ret == (*gfp & !(__GFP_DIRECT_RECLAIM | __GFP_IO)) ∧ (*gfp & __GFP_NOMEMALLOC) ≠ 0 | `Mempool::adjust_gfp` |

### Layer 4: Verus/Creusot functional

`Per-pool guaranteed-allocation contract: ∀ N. mempool_create(N) ⟹ ∀ N concurrent mempool_alloc(GFP_NOIO)-then-mempool_free pairs eventually complete in bounded steps — independent of global allocator state` semantic equivalence: per-Documentation/core-api/mm-api.rst § mempool, per-Pearce-Bonwick-style slab-pool semantics. Reserve LIFO discipline + adjust_gfp first-pass + reserve dip + io_schedule_timeout(5*HZ) wakeup loop together prove the bio-submission-under-OOM forward-progress property.

## Hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening.)

Mempool reinforcement:

- **Per-`__GFP_NOMEMALLOC` always set** — defense against per-PF_MEMALLOC-recursion (reclaim path dipping into emergency reserves on behalf of the very allocation that is supposed to free those reserves).
- **Per-`__GFP_NORETRY` first pass** — defense against per-OOM-spin (alloc_fn should fail fast and let the reserve take over rather than invoking the OOM killer under the bio path).
- **Per-first-pass-no-direct-reclaim** — defense against per-reclaim-recursion in bio submission code paths.
- **Per-LIFO reserve discipline** — defense against per-cold-cache element churn (recently-freed elements are reused first, preserving cache footprint).
- **Per-spin_lock_irqsave** — defense against per-IRQ-context corruption (mempool_free legally callable from soft-IRQ / hard-IRQ on the I/O completion side).
- **Per-`io_schedule_timeout(5*HZ)`** — defense against per-livelock when reclaim has independently freed memory but no mempool_free has occurred (periodic alloc_fn retry).
- **Per-`smp_wmb` after pop / `smp_rmb` before curr_nr read** — defense against per-cross-CPU stale-curr_nr observation.
- **Per-`add_element` BUG_ON overshoot** — defense against per-double-free that would overflow elements[] beyond min_nr.
- **Per-`remove_element` BUG_ON underflow** — defense against per-corrupted curr_nr.
- **Per-poison-on-push / check-on-pop** — defense against per-stale-write into reserved element while it sits in elements[].
- **Per-KASAN poison-on-push / unpoison-on-pop** — defense against per-use-after-pool-free.
- **Per-fault-injection `fail_mempool_alloc[ _bulk]`** — defense against per-untested reserve-fallback code path.
- **Per-`mempool_destroy(NULL)` no-op** — defense against per-cleanup-double-call on init failure unwind.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- mm/slab.c / mm/slub.c (kmem_cache_alloc/free — backing for mempool_alloc_slab; covered in `slab.md` Tier-3)
- mm/page_alloc.c (alloc_pages — backing for mempool_alloc_pages; covered in `page-allocator.md` Tier-3)
- block/bio.c bio-set mempools (bio.md Tier-3 in `block/` design tree)
- include/linux/sbitmap.h (separate reservation primitive, unrelated)
- Documentation/core-api/mm-api.rst prose (separate doc-system task)
- Implementation code
