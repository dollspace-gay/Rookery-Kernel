# Tier-3: lib/xarray.c — XArray (radix-tree successor)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: lib/00-overview.md
upstream-paths:
  - lib/xarray.c (~2481 lines)
  - include/linux/xarray.h
  - Documentation/core-api/xarray.rst
-->

## Summary

The **XArray** is the modern successor to the radix tree: an indexed array of `void *` values keyed by an `unsigned long`. Per-instance state lives in `struct xarray` (root pointer, per-flags, per-spinlock); per-walk state lives in `struct xa_state` (cursor `xa_index`, current `xa_node`, slot offset, per-allocation cache). Per-`xa_load` returns the entry at an index; per-`xa_store` writes an entry; per-`xa_erase` removes one. Iteration uses `xa_for_each` / `xas_for_each` driven by `xa_find` / `xas_find`. Per-entry "marks" (XA_MARK_0..2) provide multi-bit tagging propagated up the radix tree so per-marked-iteration is O(log N). Per-xa_lock serializes mutations; readers use RCU. Backs IDR, IDA, page cache (`address_space.i_pages`), `/proc/<pid>/fd`, mqueue, devpts, IRQ-domain mapping, and many more.

This Tier-3 covers `lib/xarray.c` (~2481 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct xarray` | per-array anchor (root + flags + lock) | `XArray` |
| `struct xa_state` | per-walk cursor | `XaState` |
| `struct xa_node` | per-node (slots + marks + parent) | `XaNode` |
| `xa_load()` | per-index lookup | `XArray::load` |
| `xa_store()` | per-index store (returns old) | `XArray::store` |
| `__xa_store()` | per-locked store | `XArray::store_locked` |
| `xa_erase()` | per-index erase | `XArray::erase` |
| `__xa_erase()` | per-locked erase | `XArray::erase_locked` |
| `xa_cmpxchg()` / `__xa_cmpxchg()` | per-index CAS | `XArray::cmpxchg` |
| `xa_insert()` / `__xa_insert()` | per-index insert (EEXIST if used) | `XArray::insert` |
| `xa_store_range()` | per-range multi-index store | `XArray::store_range` |
| `xa_find()` / `xa_find_after()` | per-iter next-present | `XArray::find` / `find_after` |
| `xa_extract()` | per-bulk gather present entries | `XArray::extract` |
| `xa_get_mark()` / `xa_set_mark()` / `xa_clear_mark()` | per-mark read/set/clear | `XArray::get_mark` / `set_mark` / `clear_mark` |
| `__xa_set_mark()` / `__xa_clear_mark()` | per-locked mark | `XArray::set_mark_locked` / `clear_mark_locked` |
| `xa_destroy()` | per-array free-all | `XArray::destroy` |
| `__xa_alloc()` / `__xa_alloc_cyclic()` | per-ID allocator (IDR/IDA backing) | `XArray::alloc` / `alloc_cyclic` |
| `xa_get_order()` / `xas_get_order()` | per-index entry order | `XArray::get_order` |
| `xas_load()` | per-state lookup | `XaState::load` |
| `xas_store()` | per-state store | `XaState::store` |
| `xas_find()` / `xas_find_marked()` / `xas_find_conflict()` | per-state iter | `XaState::find` / `find_marked` / `find_conflict` |
| `xas_next()` / `__xas_next()` / `xas_prev()` / `__xas_prev()` | per-state ±1 step | `XaState::next` / `prev` |
| `xas_pause()` | per-state checkpoint for drop-lock | `XaState::pause` |
| `xas_nomem()` | per-state allocation-retry | `XaState::nomem` |
| `xas_create()` / `xas_create_range()` | per-state path-construct | `XaState::create` / `create_range` |
| `xas_split()` / `xas_split_alloc()` / `xas_try_split()` | per-multi-index entry split | `XaState::split` |
| `xas_get_mark()` / `xas_set_mark()` / `xas_clear_mark()` / `xas_init_marks()` | per-state mark ops | `XaState::*_mark` |
| `xas_destroy()` | per-state release-alloc | `XaState::destroy` |
| `XA_MARK_0..2` | per-mark indices | `XaMark` enum |
| `XA_FLAGS_LOCK_IRQ` / `_LOCK_BH` / `_TRACK_FREE` / `_ZERO_BUSY` / `_ALLOC` / `_ALLOC1` / `_ACCOUNT` | per-flags | `XaFlags` bitset |

## Compatibility contract

REQ-1: struct xarray:
- xa_lock: per-spinlock_t (mutator serialization).
- xa_flags: per-gfp_t bitset (XA_FLAGS_LOCK_* + XA_FLAGS_TRACK_FREE + XA_FLAGS_ZERO_BUSY + XA_FLAGS_MARK(n) + XA_FLAGS_ALLOC{,1} + XA_FLAGS_ACCOUNT + XA_FLAGS_ALLOC_WRAPPED).
- xa_head: per-`void __rcu *` root (NULL / value-entry / xa_node pointer).
- Per-statically-init via `DEFINE_XARRAY(name)` / `DEFINE_XARRAY_FLAGS(name, f)` / `DEFINE_XARRAY_ALLOC(name)` / `DEFINE_XARRAY_ALLOC1(name)`.

REQ-2: struct xa_state:
- xa_xa: per-array pointer.
- xa_index: per-current index.
- xa_shift: per-current node shift (0 = leaf).
- xa_sibs: per-multi-index entry sibling count (log2).
- xa_offset: per-slot offset in xa_node.
- xa_pad: alignment.
- xa_node: per-current xa_node (XAS_RESTART / XAS_BOUNDS / XAS_ERROR sentinels possible).
- xa_alloc: per-stashed pre-allocated xa_node (for xas_nomem).
- xa_update: per-update_node callback (used by IDR).
- xa_lru: per-LRU-list head pointer for tracked-free allocation.

REQ-3: struct xa_node:
- shift: per-this-level radix shift (multiples of XA_CHUNK_SHIFT=6, ⟹ 64-way fanout).
- offset: per-slot in parent.
- count: per-non-NULL-slot count.
- nr_values: per-internal value-entry count.
- parent: per-parent xa_node.
- array: per-parent xarray.
- private_list / private_area: per-IDR free-list link.
- slots[XA_CHUNK_SIZE]: per-64-slot array of `void __rcu *`.
- marks[XA_MAX_MARKS][XA_MARK_LONGS]: per-3 marks × bitmap.

REQ-4: xa_load(xa, index):
- rcu_read_lock.
- XA_STATE(xas, xa, index).
- entry = xas_load(&xas) (descend root → leaf; deref RCU).
- if xa_is_internal(entry): entry = NULL (retry / sibling-marker not exposed).
- rcu_read_unlock.
- return entry.

REQ-5: xa_store(xa, index, entry, gfp):
- xa_lock(xa).
- curr = __xa_store(xa, index, entry, gfp).
- xa_unlock(xa).
- return curr.

REQ-6: __xa_store(xa, index, entry, gfp):
- XA_STATE(xas, xa, index).
- if xa_is_advanced(entry): return XA_ERROR(-EINVAL).
- if entry NULL ∧ XA_FREE_MARK present: mark set (free).
- do:
  - curr = xas_store(&xas, entry).
  - if xa_track_free(xa): xas mark-update against XA_FREE_MARK.
- while xas_nomem(&xas, gfp).
- return xas_result(&xas, curr).

REQ-7: xa_erase(xa, index):
- xa_lock(xa).
- entry = __xa_erase(xa, index)   (= __xa_store(xa, index, NULL, 0)).
- xa_unlock(xa).
- return entry.

REQ-8: xa_cmpxchg(xa, index, old, entry, gfp):
- xa_lock(xa).
- curr = __xa_cmpxchg(xa, index, old, entry, gfp).
- xa_unlock(xa).
- return curr.
- /* CAS semantics: store only if current == old; returns old current */

REQ-9: xa_insert(xa, index, entry, gfp):
- xa_lock(xa).
- err = __xa_insert(xa, index, entry, gfp).
- xa_unlock(xa).
- return err.
- /* Returns -EBUSY if a non-NULL entry already exists. */

REQ-10: xa_for_each(xa, index, entry):
- index = 0.
- while (entry = xa_find(xa, &index, ULONG_MAX, XA_PRESENT)) != NULL:
  - body(entry, index).
  - index++.

REQ-11: xa_for_each_marked(xa, index, entry, mark):
- index = 0.
- while (entry = xa_find(xa, &index, ULONG_MAX, mark)) != NULL:
  - body(entry).
  - index++.

REQ-12: xa_set_mark / xa_clear_mark / xa_get_mark:
- Per-mark ∈ {XA_MARK_0, XA_MARK_1, XA_MARK_2}.
- xa_set_mark(xa, index, mark): lock-then-`__xa_set_mark`; per-leaf-set then propagate-up if first.
- xa_clear_mark(xa, index, mark): lock-then-`__xa_clear_mark`; clear leaf bit; if last bit of node cleared, propagate-up.
- xa_get_mark(xa, index, mark): rcu-read; descend; bit-test slot.

REQ-13: xa_destroy(xa):
- Per-walk root; free every xa_node (recursive); free any value-entries via xa_zero / xa_is_value.

REQ-14: xa_find(xa, &index, max, filter):
- XA_STATE(xas, xa, *index).
- rcu_read_lock.
- if filter == XA_PRESENT: entry = xas_find(&xas, max).
- else (mark): entry = xas_find_marked(&xas, max, mark).
- *index = xas.xa_index.
- rcu_read_unlock.
- return entry.

REQ-15: xa_extract(xa, dst, start, max, n, filter):
- Per-bulk-gather up to n entries with index ∈ [start, max] passing filter; returns count placed.

REQ-16: xa_alloc / xa_alloc_cyclic (XA_FLAGS_ALLOC):
- Per-XA_FREE_MARK tracks free indices.
- xa_alloc finds lowest free index in [min, max], stores entry, returns id.
- xa_alloc_cyclic continues from per-array cursor; wraps with XA_FLAGS_ALLOC_WRAPPED.

REQ-17: xa_get_order(xa, index):
- Per-multi-index entry order (in radix-tree bits). 0 = single slot.

REQ-18: xas_pause(&xas):
- Marks xas safe for drop-then-reacquire of xa_lock / rcu_read_lock.
- xa_index updated to next-target; xa_node reset to XAS_RESTART.
- Next xas_find / xas_next will re-descend.

## Acceptance Criteria

- [ ] AC-1: `xa_store` then `xa_load` returns the stored entry; second `xa_store(NULL)` returns the stored entry; subsequent `xa_load` returns NULL.
- [ ] AC-2: `xa_erase(unknown index)` returns NULL; tree empty after erase of last entry.
- [ ] AC-3: `xa_cmpxchg(xa, i, old, new, gfp)` performs CAS atomically under xa_lock; returns previous value.
- [ ] AC-4: `xa_insert` on occupied index returns -EBUSY; on free index returns 0.
- [ ] AC-5: `xa_set_mark(xa, i, m)` then `xa_get_mark(xa, i, m)` returns true; cleared sibling indices unaffected.
- [ ] AC-6: `xa_for_each_marked` yields only entries with the given mark set.
- [ ] AC-7: `xa_find(xa, &i, max, XA_PRESENT)` from i=0 enumerates indices in ascending order until > max.
- [ ] AC-8: `xa_alloc` on a XA_FLAGS_ALLOC xarray returns the lowest free id; allocates monotonically until erased.
- [ ] AC-9: `xa_alloc_cyclic` continues from last id; wraps past max → XA_FLAGS_ALLOC_WRAPPED set.
- [ ] AC-10: `xa_destroy` frees all nodes and value-entries; no leaks under KMEMLEAK.
- [ ] AC-11: Per-concurrent `xa_load` under RCU during `xa_store` never observes torn / freed memory.
- [ ] AC-12: `xas_pause` then re-iterate produces the same remaining sequence as uninterrupted iteration.
- [ ] AC-13: `xa_store_range(xa, first, last, entry, gfp)` returns same entry for every load in [first, last].
- [ ] AC-14: Mark propagation: clearing the last marked leaf clears the path-marks up to the root.
- [ ] AC-15: XA_FLAGS_LOCK_IRQ: xa_lock_irqsave / xa_unlock_irqrestore variants used; no IRQ context violation.

## Architecture

```
struct XArray {
    xa_lock: Spinlock,
    xa_flags: XaFlags,                 // bitset: LOCK_IRQ | LOCK_BH | TRACK_FREE | ZERO_BUSY | ALLOC_WRAPPED | ACCOUNT | MARK(n)
    xa_head: AtomicPtr<()>,            // RCU-protected root (Null / Value / *XaNode)
}

struct XaNode {
    shift: u8,                         // 0 = leaf; otherwise k * XA_CHUNK_SHIFT
    offset: u8,                        // slot in parent
    count: u8,                         // non-NULL slot count
    nr_values: u8,                     // value-entry count
    parent: *XaNode,
    array: *XArray,
    private_list: ListHead,
    slots: [AtomicPtr<()>; XA_CHUNK_SIZE], // 64
    marks: [[usize; XA_MARK_LONGS]; XA_MAX_MARKS], // 3 marks
}

struct XaState {
    xa: *XArray,
    xa_index: u64,
    xa_shift: u8,
    xa_sibs: u8,
    xa_offset: u8,
    xa_pad: u8,
    xa_node: *XaNode,                  // or XAS_RESTART / XAS_BOUNDS / XAS_ERROR
    xa_alloc: *XaNode,                 // stashed by xas_nomem
    xa_update: Option<fn(*XaNode)>,
    xa_lru: *ListHead,
}
```

`XArray::load(xa, index) -> *void`:
1. rcu_read_lock().
2. xas = XaState::new(xa, index).
3. entry = xas.load().
4. if XaState::is_internal(entry): entry = null.
5. rcu_read_unlock().
6. return entry.

`XaState::load(&mut self) -> *void`:
1. entry = xas_start(self): set self.xa_node from self.xa.xa_head; if entry is value or null-leaf, return.
2. while XaNode::is(entry) ∧ entry.shift > self.xa_shift:
   - entry = xas_descend(self, node): self.xa_offset = (self.xa_index >> node.shift) & XA_CHUNK_MASK; self.xa_node = node; entry = node.slots[xa_offset].
3. return entry.

`XArray::store(xa, index, entry, gfp) -> *void`:
1. xa_lock(xa).
2. curr = XArray::store_locked(xa, index, entry, gfp).
3. xa_unlock(xa).
4. return curr.

`XArray::store_locked(xa, index, entry, gfp) -> *void`:
1. xas = XaState::new(xa, index).
2. if XaState::is_advanced(entry): return XaState::error(-EINVAL).
3. loop:
   - curr = xas.store(entry).
   - if XArray::tracks_free(xa): xas.update_mark(XA_FREE_MARK, entry.is_null()).
   - if xas.nomem(gfp): continue.
   - break.
4. return xas.result(curr).

`XaState::store(&mut self, entry) -> *void`:
1. first = xas_create(self, true): grow tree as needed; return current entry at index.
2. node = self.xa_node.
3. if entry.is_null():
   - rcu_assign_pointer(node.slots[self.xa_offset], null).
   - if first non-null: node.count -= 1; xas_delete_node-up if count=0.
4. else:
   - rcu_assign_pointer(node.slots[self.xa_offset], entry).
   - if first.is_null(): node.count += 1.
5. xas_squash_marks(self) if multi-index.
6. return first.

`XArray::erase(xa, index) -> *void`:
1. xa_lock(xa).
2. curr = XArray::store_locked(xa, index, null, 0).
3. xa_unlock(xa).
4. return curr.

`XArray::cmpxchg(xa, index, old, entry, gfp) -> *void`:
1. xa_lock(xa).
2. xas = XaState::new(xa, index).
3. loop:
   - curr = xas.load().
   - if curr == old: xas.store(entry).
   - if !xas.nomem(gfp): break.
4. xa_unlock(xa).
5. return curr.

`XArray::set_mark(xa, index, mark)`:
1. xa_lock(xa).
2. xas = XaState::new(xa, index).
3. entry = xas.load().
4. if !entry.is_null():
   - xas.set_mark(mark): set bit in node.marks[mark]; walk to parent and set bit while propagating up; set xa.xa_flags |= XA_FLAGS_MARK(mark).
5. xa_unlock(xa).

`XArray::clear_mark(xa, index, mark)`:
1. xa_lock(xa).
2. xas = XaState::new(xa, index).
3. xas.clear_mark(mark): clear bit; if node.marks[mark] now all-zero, propagate-up clear; if root all-zero, clear XA_FLAGS_MARK(mark).
4. xa_unlock(xa).

`XaState::find(&mut self, max) -> *void`:
1. if self.xa_node == XAS_RESTART: xas_start(self).
2. else: self.xa_index += 1; xas_next_offset(self).
3. while self.xa_index ≤ max:
   - entry = xas_load-or-step.
   - if !entry.is_null() ∧ !XaState::is_internal(entry): return entry.
   - advance.
4. return null.

`XaState::find_marked(&mut self, max, mark) -> *void`:
1. Walk tree skipping subtrees with marks[mark] all-zero (O(log N)).
2. Return first entry with mark set in [self.xa_index, max].

`XaState::pause(&mut self)`:
1. if self.xa_node ∉ {RESTART, BOUNDS, ERROR}:
   - self.xa_index = (self.xa_index | mask(self.xa_node.shift)) + 1.
2. self.xa_node = XAS_RESTART.

`XArray::alloc(xa, id_out, entry, range, gfp) -> int`:
1. requires xa.xa_flags & XA_FLAGS_TRACK_FREE.
2. xa_lock(xa).
3. xas = XaState::new(xa, range.min).
4. xas.find_marked(range.max, XA_FREE_MARK) → first free index.
5. if found: xas.store(entry); *id_out = xas.xa_index; clear free-mark at that index.
6. else: return -EBUSY.
7. xa_unlock(xa).
8. return 0.

`XArray::store_range(xa, first, last, entry, gfp) -> *void`:
1. xa_lock(xa).
2. xas = XaState::new(xa, first).
3. xas_set_range(&xas, first, last): set xa_shift to cover [first,last]; xa_sibs accordingly.
4. xas_create_range(&xas): allocate intermediate nodes spanning range.
5. xas.store(entry) — single multi-index store + sibling-marker propagation.
6. xa_unlock(xa).
7. return prior entry at first.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `xas_load_holds_rcu` | INVARIANT | per-xas_load: rcu_read_lock held (or xa_lock). |
| `xas_store_holds_xa_lock` | INVARIANT | per-xas_store: xa_lock held. |
| `node_count_matches_slots` | INVARIANT | per-XaNode: count == popcount(slots != NULL). |
| `marks_imply_present_leaf_or_path` | INVARIANT | per-mark-bit: any path-bit ⟹ ∃ leaf with mark set under it. |
| `xa_internal_never_returned` | INVARIANT | per-xa_load: never returns xa_is_internal(entry). |
| `xa_state_offset_in_range` | INVARIANT | per-XaState: xa_offset < XA_CHUNK_SIZE. |
| `xa_alloc_returns_within_range` | INVARIANT | per-alloc: id ∈ [min, max] on success. |
| `xa_destroy_no_leak` | INVARIANT | per-destroy: post-destroy, no XaNode allocations live. |

### Layer 2: TLA+

`lib/xarray.tla`:
- Per-store / per-load / per-erase / per-cmpxchg / per-mark / per-iterate.
- Properties:
  - `safety_load_after_store_returns_value` — per-store(i,v) then load(i) = v (no intervening erase).
  - `safety_load_after_erase_returns_null` — per-erase(i) then load(i) = null.
  - `safety_mark_propagation` — per-set_mark(i): every ancestor node has bit set; per-clear_mark final: every ancestor cleared.
  - `safety_concurrent_load_during_store_consistent` — RCU readers observe pre or post but never torn.
  - `safety_xa_alloc_unique_ids` — per-allocator: no two concurrent allocs return same id.
  - `liveness_eventual_completion` — per-mutator: xas_nomem retry terminates on success / OOM.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `XArray::load` post: result == None ∨ rcu_dereference of slot at index | `XArray::load` |
| `XArray::store` post: load(index) returns entry under xa_lock | `XArray::store` |
| `XArray::cmpxchg` post: atomic-CAS under xa_lock | `XArray::cmpxchg` |
| `XArray::set_mark` post: get_mark(index, m) = true | `XArray::set_mark` |
| `XArray::clear_mark` post: get_mark(index, m) = false | `XArray::clear_mark` |
| `XaState::find` post: returns lowest index ≥ start with entry present | `XaState::find` |
| `XaState::find_marked` post: returns lowest index ≥ start with mark set | `XaState::find_marked` |
| `XArray::alloc` post: id unused at entry; id in range | `XArray::alloc` |
| `XaState::pause` post: subsequent find resumes correctly post-drop-lock | `XaState::pause` |

### Layer 4: Verus/Creusot functional

`Per-XArray load / store / erase / mark / iterate / alloc` semantic equivalence: per-`Documentation/core-api/xarray.rst` and `lib/test_xarray.c` test corpus. RCU-reader / xa_lock-writer separation modeled per-rcu protocol.

## Hardening

(Inherits row-1 features from `lib/00-overview.md` § Hardening.)

XArray reinforcement:

- **Per-RCU readers / xa_lock writers strict separation** — defense against per-tear / per-UAF on concurrent load+store.
- **Per-xa_lock_irq / _bh variants honored** — defense against per-deadlock from soft / hard-IRQ acquisition.
- **Per-xa_node count == popcount(slots) invariant** — defense against per-leak / per-double-free of subtree.
- **Per-mark propagation atomic under xa_lock** — defense against per-marked-iteration missing entries.
- **Per-XA_ERROR / XA_ZERO / sibling-marker entries never leak to caller** — defense against per-internal-pointer dereference.
- **Per-xas_nomem retry bounded** — defense against per-infinite-retry on perpetual OOM.
- **Per-xa_destroy walks RCU-quiescent** — defense against per-reader-stuck-on-freed-node.
- **Per-XA_FLAGS_ALLOC tracks XA_FREE_MARK invariantly** — defense against per-duplicate-id allocation.
- **Per-xas_pause re-descend on resume** — defense against per-stale-node deref after lock drop.
- **Per-xa_store_range sibling-marker integrity** — defense against per-partial-multi-index store inconsistency.
- **Per-XA_CHUNK_SIZE=64 fanout bounded** — defense against per-stack-overflow on deep trees (max depth ~ 11 for 64-bit indices).
- **Per-xa_head NULL singleton for empty xarray** — defense against per-spurious-allocation on read-only use.
- **Per-XA_BUG_ON debug asserts compile-out in production** — defense against per-debug-bloat in hot path.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- IDR / IDA (covered in `lib/idr.md` Tier-3)
- Page cache i_pages users (covered in `mm/filemap.md` Tier-3)
- /proc/<pid>/fd (covered in `fs/proc/fd.md` if expanded)
- mqueue uses (covered in `ipc/mqueue.md` if expanded)
- Implementation code
