# Tier-3: lib/rhashtable.c — Resizable hash table

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: lib/00-overview.md
upstream-paths:
  - lib/rhashtable.c (~1282 lines)
  - include/linux/rhashtable.h
  - include/linux/rhashtable-types.h
  - Documentation/core-api/rhashtable.rst
-->

## Summary

The **resizable hashtable** is an RCU-safe, lockless-read, lock-bucket-write hash table that grows / shrinks online via a **two-table rehash**. Per-instance state lives in `struct rhashtable` (params, two `bucket_table` pointers, deferred-rehash work). Per-`rhashtable_lookup_fast` walks the bucket chain under RCU. Per-`rhashtable_insert_fast` inserts under a per-bucket bit-spinlock. Resize occurs when load factor crosses 75% (grow) or 30% (shrink): a `future_tbl` is allocated and chains are migrated one-bucket-at-a-time by `rhashtable_rehash_chain`; readers transparently follow `future_tbl`. Per-`struct rhash_head` is the embedded link. Per-`rhashtable_params` defines key length, head offset, hashfn, obj_hashfn, obj_cmpfn, automatic-shrinking, nelem_hint, min/max-size. Used by netfilter conntrack, mptcp tokens, nft sets, ipvs-conn, RDS, network-namespace tags, and many more.

This Tier-3 covers `lib/rhashtable.c` (~1282 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct rhashtable` | per-table top | `RHashTable` |
| `struct rhashtable_params` | per-table config | `RHashParams` |
| `struct bucket_table` | per-table bucket array | `BucketTable` |
| `struct rhash_head` | per-object embedded link | `RHashHead` |
| `struct rhash_lock_head` | per-bucket head + lock-bit | `RHashLockHead` |
| `struct rhashtable_iter` | per-walk cursor | `RHashIter` |
| `struct rhashtable_walker` | per-walker (registered for rehash) | `RHashWalker` |
| `struct rhltable` | per-list-variant (chained per-key list) | `RHashList` |
| `rhashtable_init()` / `rhashtable_init_noprof()` | per-table init | `RHashTable::init` |
| `rhltable_init()` / `rhltable_init_noprof()` | per-list-variant init | `RHashList::init` |
| `rhashtable_destroy()` | per-table teardown (assumes empty) | `RHashTable::destroy` |
| `rhashtable_free_and_destroy()` | per-table free-each + teardown | `RHashTable::free_and_destroy` |
| `rhashtable_lookup_fast()` (inline) | per-RCU lookup | `RHashTable::lookup_fast` |
| `rhashtable_insert_fast()` (inline) | per-bucket-locked insert (fast path) | `RHashTable::insert_fast` |
| `rhashtable_remove_fast()` (inline) | per-bucket-locked remove | `RHashTable::remove_fast` |
| `rhashtable_replace_fast()` (inline) | per-bucket-locked atomic-replace | `RHashTable::replace_fast` |
| `rhashtable_lookup_insert_fast()` (inline) | per-lookup-or-insert | `RHashTable::lookup_insert_fast` |
| `rhashtable_lookup_get_insert_fast()` (inline) | per-lookup-get-or-insert | `RHashTable::lookup_get_insert_fast` |
| `rhashtable_insert_slow()` | per-slow-path insert (rehash / nested-tbl) | `RHashTable::insert_slow` |
| `rhashtable_insert_rehash()` (static) | per-on-insert-trigger rehash | `RHashTable::insert_rehash` |
| `rhashtable_walk_enter()` | per-iterator register | `RHashIter::enter` |
| `rhashtable_walk_exit()` | per-iterator unregister | `RHashIter::exit` |
| `rhashtable_walk_start()` / `rhashtable_walk_start_check()` | per-iter begin (RCU) | `RHashIter::start` |
| `rhashtable_walk_next()` | per-iter advance | `RHashIter::next` |
| `rhashtable_walk_peek()` | per-iter peek | `RHashIter::peek` |
| `rhashtable_walk_stop()` | per-iter pause (releases RCU) | `RHashIter::stop` |
| `bucket_table_alloc()` / `_free()` (static) | per-bucket-table lifecycle | `BucketTable::alloc` / `free` |
| `nested_bucket_table_alloc()` (static) | per-large-table 2-level alloc | `BucketTable::alloc_nested` |
| `rhashtable_rehash_one()` / `_chain()` / `_table()` (static) | per-bucket / per-chain / per-table rehash | `RHashTable::rehash_*` |
| `rhashtable_rehash_alloc()` / `_attach()` (static) | per-future-tbl alloc + publish | `RHashTable::rehash_alloc` / `attach` |
| `rhashtable_shrink()` (static) | per-shrink path | `RHashTable::shrink` |
| `rht_deferred_worker()` (static) | per-workqueue rehash work | `RHashTable::deferred_worker` |
| `rht_deferred_irq_work()` (static) | per-irq-work kick to wq | `RHashTable::deferred_irq_work` |
| `__rht_bucket_nested()` / `rht_bucket_nested()` / `rht_bucket_nested_insert()` | per-nested-tbl bucket access | `BucketTable::nested_*` |
| `rht_grow_above_75()` / `rht_grow_above_100()` / `rht_shrink_below_30()` / `rht_grow_above_max()` | per-load-factor predicates | `BucketTable::should_*` |
| `lockdep_rht_mutex_is_held()` / `lockdep_rht_bucket_is_held()` | per-lockdep helper | `RHashTable::lockdep_*` |
| `RHT_NULLS_MARKER` / `INIT_RHT_NULLS_HEAD` / `rht_is_a_nulls` | per-end-of-chain marker | `RHashHead::nulls` |

## Compatibility contract

REQ-1: struct rhashtable:
- tbl: per-`struct bucket_table __rcu *` current table.
- key_len: per-fixed-key-length (0 = obj_hashfn variable).
- max_elems: per-cap (or 0 = unlimited).
- p: per-`struct rhashtable_params` (copy of init params).
- rhlist: per-bool: this is an rhltable variant.
- run_work: per-`struct work_struct` deferred rehash.
- defer_irq_work: per-`struct irq_work` to kick rehash from atomic.
- mutex: per-`struct mutex` serializing rehash-driver.
- lock: per-spinlock_t for insert-slow / walker-list.
- nelems: per-`atomic_t` element count.
- walkers: per-walker list (linked under lock).

REQ-2: struct rhashtable_params:
- nelem_hint: per-init-size hint.
- key_len: per-fixed-key-length.
- key_offset: per-offset of key in object.
- head_offset: per-offset of `struct rhash_head` in object.
- max_size: per-table-cap (rounded up to power-of-two).
- min_size: per-table-floor.
- automatic_shrinking: per-allow auto-shrink-below-30%.
- hashfn: per-key hash callback (default jhash2).
- obj_hashfn: per-object hash callback (preferred over hashfn if key_len=0).
- obj_cmpfn: per-object compare callback (default memcmp on key).

REQ-3: struct bucket_table:
- size: per-bucket count (power-of-two).
- nest: per-2-level-nest exponent (0 = flat).
- rehash: per-current rehash-progress index.
- hash_rnd: per-table jhash seed.
- walkers: per-walker list head (cloned on rehash-attach).
- rcu: per-rcu_head for table free.
- future_tbl: per-`struct bucket_table __rcu *` rehash target.
- buckets[]: per-flexible `struct rhash_lock_head __rcu *` array (size entries).

REQ-4: rhashtable_init(ht, params):
- Validate params (size, alignment, callbacks).
- Copy params into ht.p.
- nbuckets = rounded_hashtable_size(&params).
- ht.tbl = bucket_table_alloc(ht, nbuckets, GFP_KERNEL).
- if !ht.tbl: return -ENOMEM.
- INIT_WORK(&ht.run_work, rht_deferred_worker).
- init_irq_work(&ht.defer_irq_work, rht_deferred_irq_work).
- mutex_init / spin_lock_init.
- atomic_set(&ht.nelems, 0).
- return 0.

REQ-5: rhashtable_destroy(ht):
- Assumes ht empty.
- cancel_work_sync(&ht.run_work).
- bucket_table_free(ht.tbl).

REQ-6: rhashtable_free_and_destroy(ht, free_fn, arg):
- cancel_work_sync(&ht.run_work).
- mutex_lock(&ht.mutex).
- /* Walk both tables (current + future) */
- for tbl in {ht.tbl, ht.tbl.future_tbl}:
  - for i in 0..tbl.size:
    - for obj in tbl.buckets[i] chain:
      - rhashtable_free_one(ht, obj, free_fn, arg).
- mutex_unlock.
- bucket_table_free(ht.tbl) (recursive on future_tbl).

REQ-7: rhashtable_lookup_fast (inline, in rhashtable.h):
- rcu_read_lock.
- tbl = rht_dereference_rcu(ht.tbl).
- hash = rht_key_hashfn(ht, tbl, key, params).
- bucket = rht_bucket(tbl, hash).
- for he in rht_for_each_rcu(bucket):
  - if obj_cmpfn(he, key) == 0:
    - rcu_read_unlock.
    - return rht_obj(ht, he).
- /* If rehash in progress: follow future_tbl */
- tbl = rht_dereference_rcu(tbl.future_tbl).
- if tbl: retry-hash-lookup-on-future.
- rcu_read_unlock.
- return NULL.

REQ-8: rhashtable_insert_fast (inline → slow on contention):
- rcu_read_lock.
- tbl = rht_dereference_rcu(ht.tbl).
- hash = rht_head_hashfn(ht, tbl, obj, params).
- lock-bucket: bit-spin on rht_bucket(tbl, hash).
- /* Optimistic */
- if !duplicate ∧ !rehashing(tbl) ∧ !grow-needed:
  - cmpxchg head pointer (insert at head; preserves nulls marker).
  - unlock; atomic_inc(nelems); rcu_read_unlock; return 0.
- /* Else slow path */
- unlock; rcu_read_unlock.
- return rhashtable_insert_slow(ht, key, obj).

REQ-9: rhashtable_insert_slow(ht, key, obj):
- Loop:
  - rcu_read_lock.
  - tbl = rhashtable_last_table(ht, rht_dereference_rcu(ht.tbl)): walk future_tbl chain to end.
  - bkt = rht_bucket_var-or-insert(tbl, hash).
  - rht_lock(tbl, bkt).
  - data = rhashtable_lookup_one(ht, &bkt, tbl, hash, key, obj).
  - if IS_ERR(data) ∧ PTR_ERR == -EAGAIN: rht_unlock; rcu_read_unlock; cpu_relax; continue.
  - new_tbl = rhashtable_insert_one(ht, &bkt, tbl, hash, obj, data).
  - rht_unlock.
  - rcu_read_unlock.
  - if !new_tbl: return data (or NULL on success).
  - tbl = new_tbl; continue.   /* moved to future_tbl mid-insert */

REQ-10: rhashtable_remove_fast (inline):
- rcu_read_lock.
- tbl = rht_dereference_rcu(ht.tbl).
- hash = rht_head_hashfn(ht, tbl, obj, params).
- rht_lock_bucket.
- find ∧ unlink (RCU-safe, preserve nulls).
- rht_unlock.
- if !found ∧ ht.tbl.future_tbl: retry on future_tbl.
- atomic_dec(nelems).
- /* Auto-shrink trigger */
- if params.automatic_shrinking ∧ rht_shrink_below_30(ht, tbl):
  - schedule rht_deferred_worker.
- rcu_read_unlock.
- return 0 / -ENOENT.

REQ-11: Rehash protocol (two-table):
- Trigger: rht_grow_above_75 ∨ rht_grow_above_100 ∨ explicit shrink.
- rht_deferred_worker:
  - mutex_lock(&ht.mutex).
  - if rht_grow_above_75: new_tbl = bucket_table_alloc(2 * tbl.size). rhashtable_rehash_alloc + attach.
  - else if rht_shrink_below_30 ∧ automatic_shrinking: new_tbl = alloc(tbl.size / 2). attach.
  - rhashtable_rehash_table(ht): for each bucket, rhashtable_rehash_chain(ht, old_hash):
    - rht_lock(old_tbl, old_bucket).
    - for each entry: rhashtable_rehash_one: compute new_hash; rht_lock(new_tbl, new_bucket); link to new; mark old slot with RHT_NULLS_MARKER pointing at new_tbl.
    - rht_unlock both.
  - rcu_assign_pointer(ht.tbl, new_tbl).
  - synchronize_rcu.
  - bucket_table_free(old_tbl) via call_rcu.
  - mutex_unlock.

REQ-12: rht_is_a_nulls / RHT_NULLS_MARKER:
- End-of-chain marker encodes hash bucket index in low bits with NULLS_MARKER tag.
- Readers crossing a NULLS marker detect "moved to future_tbl" and retry on future_tbl.

REQ-13: Nested bucket-table (large):
- For sizes > BUCKET_TABLE_SIZE_FOR_NEST_BUCKETS: 2-level radix with per-level pointer arrays to amortize contiguous allocation.
- nested_table_alloc / __rht_bucket_nested / rht_bucket_nested_insert handle lazy 2nd-level allocation.

REQ-14: Walker (iterator) registration:
- rhashtable_walk_enter(ht, iter):
  - spin_lock(&ht.lock).
  - iter.walker.tbl = rht_dereference_protected(ht.tbl).
  - list_add(&iter.walker.list, &iter.walker.tbl.walkers).
  - spin_unlock.
- During rehash: rehash_alloc clones walkers list to future_tbl; iterator transparently transitions.
- rhashtable_walk_exit: list_del under lock.

REQ-15: rhashtable_walk_start_check / _next:
- start_check: rcu_read_lock; restore iter.p / iter.slot / iter.skip; return -EAGAIN if walker fell off-rehash.
- next: walk chain; advance bucket; transition to future_tbl on end.
- stop: list-add walker back to current tbl.walkers; rcu_read_unlock.

REQ-16: rhashtable_jhash2:
- Default per-`hashfn` when key_len % 4 == 0 ∧ key_offset aligned.

REQ-17: Insert-time auto-grow:
- If insert-slow detects rht_grow_above_75 ∨ chain-elongation: rhashtable_insert_rehash schedules work + may grow inline via irq_work.

REQ-18: max_size enforcement:
- rht_grow_above_max: ENOSPC if exceeded — insert_slow returns ERR_PTR(-E2BIG).

## Acceptance Criteria

- [ ] AC-1: `rhashtable_init` allocates round-up-pow2(nelem_hint) buckets; respects min_size / max_size.
- [ ] AC-2: `rhashtable_lookup_fast` returns inserted object; NULL for absent key.
- [ ] AC-3: `rhashtable_insert_fast` returns 0 on success; -EEXIST on duplicate (when uniqueness required).
- [ ] AC-4: `rhashtable_remove_fast` returns 0 on success; -ENOENT if not present.
- [ ] AC-5: `rhashtable_replace_fast` atomically swaps the object under bucket-lock.
- [ ] AC-6: Concurrent insert / lookup under RCU: lookup observes nulls-marker correctly and never returns torn pointer.
- [ ] AC-7: Inserting > 75% of size triggers rehash; new tbl size = 2 × old.
- [ ] AC-8: Removing below 30% with automatic_shrinking triggers shrink; new tbl size = old / 2 (≥ min_size).
- [ ] AC-9: max_size cap: further insert past cap returns -E2BIG.
- [ ] AC-10: `rhashtable_walk_enter` + iteration covers every element present at start; new inserts during walk may or may not appear (documented).
- [ ] AC-11: During rehash, walker followed via cloned walkers list; no infinite loop, no skip of stable elements.
- [ ] AC-12: `rhashtable_free_and_destroy` calls free_fn on every element of both current and future tables; final nelems = 0.
- [ ] AC-13: Nested bucket-table: large tables (> nest threshold) allocate 2-level; access via rht_bucket_nested.
- [ ] AC-14: NULLS marker preserved across insert / remove / rehash; reader retrying on future_tbl is bounded.
- [ ] AC-15: Lockdep: per-bucket bit-spin asserts mutual-exclusion with rehash worker holding mutex on that bucket.

## Architecture

```
struct RHashTable {
    tbl: AtomicPtr<BucketTable>,           // RCU
    key_len: u32,
    max_elems: u32,
    p: RHashParams,
    rhlist: bool,
    run_work: WorkStruct,
    defer_irq_work: IrqWork,
    mutex: Mutex,
    lock: Spinlock,                         // walker list + insert-slow
    nelems: AtomicI64,
    walkers: ListHead,
}

struct BucketTable {
    size: u32,                              // power-of-two
    nest: u8,                               // 0 = flat
    rehash: u32,                            // rehash progress (bucket index)
    hash_rnd: u32,
    walkers: ListHead,
    rcu: RcuHead,
    future_tbl: AtomicPtr<BucketTable>,     // RCU
    buckets: [AtomicPtr<RHashLockHead>; size], // flexible array
}

struct RHashHead {
    next: AtomicPtr<RHashHead>,             // RCU (low bit = bit-spin lock on head)
}

struct RHashParams {
    nelem_hint: u16,
    key_len: u16,
    key_offset: u16,
    head_offset: u16,
    max_size: u32,
    min_size: u32,
    automatic_shrinking: bool,
    hashfn: Option<fn(key: *const u8, len: u32, seed: u32) -> u32>,
    obj_hashfn: Option<fn(data: *const c_void, len: u32, seed: u32) -> u32>,
    obj_cmpfn: Option<fn(arg: *const RhashtableCompareArg, obj: *const c_void) -> i32>,
}
```

`RHashTable::init(ht, params) -> Result<()>`:
1. Validate: params.head_offset aligned; params.hashfn ∨ params.obj_hashfn ∨ params.key_len > 0.
2. ht.p = params.
3. nbuckets = rounded_hashtable_size(&params).
4. ht.tbl = BucketTable::alloc(ht, nbuckets, GFP_KERNEL)?
5. INIT_WORK(&ht.run_work, RHashTable::deferred_worker).
6. init_irq_work(&ht.defer_irq_work, RHashTable::deferred_irq_work).
7. Mutex / Spinlock init.
8. AtomicI64::store(0, &ht.nelems).
9. Ok(()).

`RHashTable::lookup_fast(ht, key) -> Option<*c_void>`:
1. rcu_read_lock.
2. tbl = RCU::deref(ht.tbl).
3. loop:
   - hash = RHashTable::key_hashfn(ht, tbl, key).
   - bkt = BucketTable::bucket(tbl, hash).
   - for he in BucketTable::for_each_rcu(bkt):
     - if RHashHead::is_nulls(he): break (end-of-chain).
     - if RHashTable::obj_cmpfn(he, key) == 0:
       - obj = RHashHead::obj(ht, he).
       - rcu_read_unlock; return Some(obj).
   - next_tbl = RCU::deref(tbl.future_tbl).
   - if next_tbl.is_null(): break.
   - tbl = next_tbl.
4. rcu_read_unlock; None.

`RHashTable::insert_fast(ht, obj) -> Result<()>`:
1. rcu_read_lock.
2. tbl = RCU::deref(ht.tbl).
3. hash = RHashTable::head_hashfn(ht, tbl, obj).
4. bkt = BucketTable::bucket(tbl, hash).
5. rht_lock(tbl, bkt) (bit-spin on head pointer).
6. /* Fast path: no rehash + no duplicate + no grow */
7. if !BucketTable::rehashing(tbl) ∧ chain_len(bkt) < ELASTICITY ∧ duplicates-not-found:
   - obj.next = read(bkt) (or NULLS marker if empty).
   - publish(bkt, obj) (RCU).
   - rht_unlock.
   - AtomicI64::add(1, &ht.nelems).
   - rcu_read_unlock.
   - Ok(()).
8. else:
   - rht_unlock; rcu_read_unlock.
   - RHashTable::insert_slow(ht, key, obj).

`RHashTable::insert_slow(ht, key, obj) -> Result<()>`:
1. loop:
   - rcu_read_lock.
   - tbl = BucketTable::last(ht.tbl).
   - hash = RHashTable::head_hashfn(ht, tbl, obj).
   - bkt = BucketTable::bucket_or_alloc(tbl, hash)? // nested-tbl 2nd-level alloc
   - rht_lock(tbl, bkt).
   - existing = RHashTable::lookup_one(ht, bkt, tbl, hash, key, obj).
   - if matches!(existing, Err(EAGAIN)): rht_unlock; rcu_read_unlock; cpu_relax; continue.
   - new_tbl = RHashTable::insert_one(ht, bkt, tbl, hash, obj, existing)?.
   - rht_unlock.
   - rcu_read_unlock.
   - if new_tbl.is_null(): return existing.
   - tbl = new_tbl; continue.

`RHashTable::remove_fast(ht, obj) -> Result<()>`:
1. rcu_read_lock.
2. tbl = RCU::deref(ht.tbl).
3. loop:
   - hash = RHashTable::head_hashfn(ht, tbl, obj).
   - bkt = BucketTable::bucket(tbl, hash).
   - rht_lock(tbl, bkt).
   - find obj in chain; unlink RCU-safely (preserve NULLS).
   - rht_unlock.
   - if found:
     - AtomicI64::sub(1, &ht.nelems).
     - if ht.p.automatic_shrinking ∧ BucketTable::shrink_below_30(ht, tbl):
       - schedule_work(&ht.run_work).
     - rcu_read_unlock.
     - return Ok(()).
   - tbl = RCU::deref(tbl.future_tbl).
   - if tbl.is_null(): break.
4. rcu_read_unlock; Err(-ENOENT).

`RHashTable::deferred_worker(work)`:
1. ht = container_of(work, RHashTable, run_work).
2. mutex_lock(&ht.mutex).
3. tbl = RCU::deref_protected(ht.tbl, &ht.mutex).
4. err = 0.
5. if BucketTable::grow_above_75(ht, tbl):
   - err = RHashTable::rehash_alloc(ht, tbl, tbl.size * 2).
6. else if BucketTable::shrink_below_30(ht, tbl) ∧ ht.p.automatic_shrinking:
   - err = RHashTable::shrink(ht).
7. if err == 0:
   - err = RHashTable::rehash_table(ht).
8. mutex_unlock.
9. if err is EAGAIN: schedule_work again.

`RHashTable::rehash_table(ht) -> int`:
1. old = RCU::deref_protected(ht.tbl, &ht.mutex).
2. new = old.future_tbl.
3. for h in 0..old.size:
   - RHashTable::rehash_chain(ht, h):
     - rht_lock(old, &old.buckets[h]).
     - for entry in chain:
       - RHashTable::rehash_one(ht, &old.buckets[h], h):
         - new_hash = RHashTable::head_hashfn(ht, new, entry).
         - rht_lock(new, &new.buckets[new_hash]).
         - link entry into new chain head.
         - rht_unlock(new).
     - mark old.buckets[h] = RHT_NULLS_MARKER (signals "moved to new").
     - rht_unlock(old).
4. /* Publish + drain readers */
5. rcu_assign_pointer(ht.tbl, new).
6. /* Move walkers over */
7. spin_lock(&ht.lock).
8. list_splice_init(&old.walkers, &new.walkers).
9. for w in new.walkers: w.tbl = new.
10. spin_unlock.
11. call_rcu(&old.rcu, BucketTable::free_rcu).
12. return 0.

`RHashTable::walk_next(iter) -> Option<*c_void>`:
1. /* Returns next element under RCU; if rehash moved table, walker.tbl re-bound */
2. while iter.slot < iter.walker.tbl.size:
   - bkt = BucketTable::bucket(iter.walker.tbl, iter.slot).
   - if iter.p.is_null(): iter.p = head(bkt).
   - while !RHashHead::is_nulls(iter.p):
     - if iter.skip-- > 0: iter.p = next(iter.p); continue.
     - obj = RHashHead::obj(ht, iter.p).
     - iter.p = next(iter.p).
     - return Some(obj).
   - iter.slot += 1; iter.p = null.
3. /* Walk done on this tbl; try future_tbl */
4. next = RCU::deref(iter.walker.tbl.future_tbl).
5. if next: iter.walker.tbl = next; iter.slot = 0; iter.p = null; goto 2.
6. None.

`BucketTable::alloc(ht, nbuckets, gfp) -> Result<*BucketTable>`:
1. if nbuckets > BUCKET_TABLE_SIZE_FOR_NEST_BUCKETS:
   - return BucketTable::alloc_nested(ht, nbuckets, gfp).
2. tbl = kvzalloc(sizeof(BucketTable) + nbuckets * sizeof(*bkt), gfp)?.
3. tbl.size = nbuckets.
4. tbl.hash_rnd = get_random_u32_nonzero().
5. INIT_LIST_HEAD(&tbl.walkers).
6. for i in 0..nbuckets: tbl.buckets[i] = INIT_RHT_NULLS_HEAD(i).
7. Ok(tbl).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `lookup_fast_holds_rcu` | INVARIANT | per-lookup_fast: rcu_read_lock held throughout chain walk. |
| `insert_fast_holds_bucket_lock` | INVARIANT | per-insert_fast publish: bucket bit-spin held. |
| `nelems_matches_chain_total` | INVARIANT | per-table: atomic_read(nelems) == ∑ over buckets of chain_len. |
| `nulls_marker_terminates_chain` | INVARIANT | per-chain: ends in RHT_NULLS marker; no NULL pointer. |
| `rehash_holds_ht_mutex` | INVARIANT | per-rehash_table: ht.mutex held. |
| `walker_in_walkers_list_xor_active` | INVARIANT | per-walker: on tbl.walkers iff !active (stop / pre-start). |
| `max_size_respected` | INVARIANT | per-insert: tbl.size ≤ rounded_pow2(max_size). |

### Layer 2: TLA+

`lib/rhashtable.tla`:
- Per-insert / per-lookup / per-remove / per-rehash (grow + shrink) / per-walker.
- Properties:
  - `safety_insert_then_lookup_returns_obj` — per-insert(k,o) ⟹ lookup(k) ∈ {o, NULL-during-window} (consistent).
  - `safety_remove_then_lookup_returns_null` — per-remove(k) post-quiescence: lookup(k) = NULL.
  - `safety_concurrent_lookup_observes_nulls_marker` — per-rehash mid-flight: reader following NULLS marker retries on future_tbl ∧ finds entry.
  - `safety_walker_observes_stable_entries` — per-walker: every entry stably present in both old + new tbl visited exactly once.
  - `safety_nelems_invariant` — per-state: atomic nelems agrees with table chain census.
  - `liveness_rehash_eventually_completes` — per-deferred-worker: scheduled rehash terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `RHashTable::lookup_fast` post: ∃ obj in tbl chain with cmpfn==0 ⟺ result is Some(obj) | `RHashTable::lookup_fast` |
| `RHashTable::insert_fast` post: obj observable via lookup; nelems += 1 | `RHashTable::insert_fast` |
| `RHashTable::remove_fast` post: obj absent; nelems -= 1 | `RHashTable::remove_fast` |
| `RHashTable::rehash_table` post: ht.tbl points to new_tbl; old_tbl freed via call_rcu | `RHashTable::rehash_table` |
| `BucketTable::alloc` post: all buckets initialized to NULLS markers | `BucketTable::alloc` |
| `RHashTable::walk_next` post: returns each element ≤ once across the walk lifetime | `RHashTable::walk_next` |
| `RHashTable::free_and_destroy` post: nelems == 0; both tbl + future_tbl freed | `RHashTable::free_and_destroy` |

### Layer 4: Verus/Creusot functional

`Per-RHashTable lookup / insert / remove / rehash / walk` semantic equivalence: per-`Documentation/core-api/rhashtable.rst` and `lib/test_rhashtable.c` test corpus, including 50%-grow / 50%-shrink stress, deferred-worker invariants, walker-resume-across-rehash.

## Hardening

(Inherits row-1 features from `lib/00-overview.md` § Hardening.)

rhashtable reinforcement:

- **Per-RCU readers / per-bucket bit-spin writers strict** — defense against per-tear on concurrent insert+lookup.
- **Per-NULLS marker encodes bucket index** — defense against per-cross-bucket pointer following on rehash.
- **Per-future_tbl chain bounded** — defense against per-unbounded retries during cascaded rehashes (insert_slow uses last-table walk).
- **Per-ht.mutex serializes rehash driver** — defense against per-concurrent grow+shrink races.
- **Per-walkers list cloned on rehash** — defense against per-iterator-stuck-on-freed-table.
- **Per-hash_rnd nonzero, per-table** — defense against per-hash-flooding DoS (per-table salt).
- **Per-max_size enforced (E2BIG)** — defense against per-unbounded table growth on attacker-key flood.
- **Per-automatic_shrinking optional** — defense against per-thrash on insert/delete churn.
- **Per-chain elasticity threshold triggers rehash** — defense against per-long-chain O(n) lookup degradation.
- **Per-call_rcu free of old table** — defense against per-reader UAF on freed bucket array.
- **Per-nested 2-level alloc for large tables** — defense against per-kvzalloc large-contig-alloc failure.
- **Per-lockdep_rht_mutex_is_held / per-bucket lockdep** — defense against per-incorrect-lock-class deadlock.
- **Per-rht_grow_above_max ENOSPC** — defense against per-OOM via attacker insertion flood.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Netfilter conntrack rhashtable user (covered in `net/netfilter/nf_conntrack.md` if expanded)
- mptcp tokens rhashtable user (covered in `net/mptcp/token.md` if expanded)
- nft sets rhashtable user (covered in `net/netfilter/nf_tables.md` if expanded)
- jhash2 algorithm details (covered in `include/linux/jhash.h` reference)
- Implementation code
