# Tier-3: lib/idr.c — IDR (integer-to-pointer) + IDA (integer-id allocator)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: lib/00-overview.md
upstream-paths:
  - lib/idr.c (~668 lines)
  - lib/radix-tree.c (idr_get_free, radix_tree_iter_* used by IDR)
  - lib/xarray.c (xas_* used by IDA)
  - include/linux/idr.h
  - include/linux/xarray.h (XA_STATE, xa_mk_value, xa_to_value, BITS_PER_XA_VALUE, IDA_BITMAP_BITS)
  - Documentation/core-api/idr.rst
-->

## Summary

IDR maps small non-negative integers to pointers; IDA allocates the same kind of IDs without storing pointers (one bit per ID). Both sit on top of the kernel's XArray/radix-tree machinery. IDR: `struct idr { struct radix_tree_root idr_rt; unsigned idr_base; unsigned idr_next; }` — the radix_tree_root carries `ROOT_IS_IDR` and `IDR_RT_MARKER` so the radix tree's free-tag (`IDR_FREE`) is maintained per slot; `idr_alloc` finds a slot via `idr_get_free` and stores `ptr` via `radix_tree_iter_replace` (with the memory-barrier that hides `*nextid = id` from racing lookups); `idr_alloc_cyclic` resumes from `idr_next` and wraps; `idr_remove` is `radix_tree_delete_item(... id - idr_base, NULL)`; `idr_find` is `radix_tree_lookup`; `idr_for_each` / `idr_get_next` iterate via `radix_tree_iter`. IDA: `struct ida { struct xarray xa; }` indexed by `id / IDA_BITMAP_BITS` storing either an `xa_mk_value(bits)` (for ≤BITS_PER_XA_VALUE low bits) or a heap-allocated `struct ida_bitmap` (128-byte chunk of `IDA_BITMAP_BITS` bits, default 1024). IDA is self-locking via `xas_lock_irqsave`; the `XA_FREE_MARK` tag identifies index nodes that still have free bits, which makes `ida_alloc_range` an O(log N) `xas_find_marked` scan. Used by: per-net device IDs (`net/core/dev.c`), PID allocation (`kernel/pid.c`), inode numbers (`fs/inode.c get_next_ino_full`), block/loop device minors, tty index, anon_inode minor, kobject_uevent seqnum, drm minor, posix-timer ids, cgroup ids, and pretty much every "give me a unique small number" pattern in the kernel.

This Tier-3 covers `lib/idr.c` (~668 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct idr` (`include/linux/idr.h`) | per-IDR descriptor (idr_rt, idr_base, idr_next) | `Idr` |
| `struct ida` (`include/linux/idr.h`) | per-IDA descriptor (xa) | `Ida` |
| `struct ida_bitmap` (`include/linux/idr.h`) | per-IDA leaf bitmap (IDA_BITMAP_LONGS longs) | `IdaBitmap` |
| `idr_alloc_u32()` | per-u32-range alloc | `Idr::alloc_u32` |
| `idr_alloc()` | per-int-range alloc | `Idr::alloc` |
| `idr_alloc_cyclic()` | per-cyclic alloc | `Idr::alloc_cyclic` |
| `idr_remove()` | per-id remove | `Idr::remove` |
| `idr_find()` | per-id lookup | `Idr::find` |
| `idr_replace()` | per-id pointer swap | `Idr::replace` |
| `idr_for_each()` | per-iter callback | `Idr::for_each` |
| `idr_get_next_ul()` / `idr_get_next()` | per-iter next-populated | `Idr::get_next_ul` / `get_next` |
| `idr_get_free()` (radix-tree.c) | per-find-free-slot | `radix_tree::iter_get_free` |
| `radix_tree_iter_init()` | per-iter cursor | shared radix-tree helper |
| `radix_tree_iter_replace()` | per-store with barrier | shared radix-tree helper |
| `radix_tree_iter_tag_clear()` | clear IDR_FREE | shared radix-tree helper |
| `radix_tree_delete_item()` | per-remove | shared radix-tree helper |
| `radix_tree_lookup()` | per-find | shared radix-tree helper |
| `radix_tree_for_each_slot()` | per-iter | shared radix-tree helper |
| `radix_tree_iter_retry()` | per-iter restart | shared radix-tree helper |
| `__radix_tree_lookup()` | per-iter lookup w/ node | shared radix-tree helper |
| `__radix_tree_replace()` | per-replace w/ barrier | shared radix-tree helper |
| `radix_tree_tag_get()` | per-tag check (IDR_FREE) | shared radix-tree helper |
| `ROOT_IS_IDR` / `IDR_RT_MARKER` / `IDR_FREE` | per-tag bits | shared |
| `xa_is_internal()` / `xa_is_retry()` | per-iter sentinel check | shared XArray helpers |
| `rcu_dereference_raw()` | per-RCU read | shared RCU |
| `ida_alloc_range()` | per-range alloc | `Ida::alloc_range` |
| `ida_alloc()` / `ida_alloc_min()` / `ida_alloc_max()` (header inlines) | per-shortcuts | `Ida::alloc` / `alloc_min` / `alloc_max` |
| `ida_find_first_range()` | per-find-lowest-used | `Ida::find_first_range` |
| `ida_free()` | per-id free | `Ida::free` |
| `ida_destroy()` | per-IDA teardown | `Ida::destroy` |
| `ida_is_empty()` (header inline) | per-empty check | `Ida::is_empty` |
| `IDA_BITMAP_BITS` / `IDA_BITMAP_LONGS` / `BITS_PER_XA_VALUE` | per-tunables | shared |
| `XA_STATE`, `xas_lock_irqsave`, `xas_unlock_irqrestore` | per-XArray cursor | shared XArray helpers |
| `xas_find_marked()` | per-XArray marked scan | shared XArray helper |
| `xas_load()` / `xas_store()` | per-XArray load/store | shared XArray helper |
| `xas_set_mark()` / `xas_clear_mark()` | XA_FREE_MARK toggle | shared XArray helper |
| `xas_nomem()` | per-retry-with-alloc | shared XArray helper |
| `xa_mk_value()` / `xa_to_value()` / `xa_is_value()` | tagged value encoding | shared XArray helper |
| `XA_FREE_MARK` / `XA_PRESENT` | per-mark bits | shared |
| `DEFINE_IDR(name)` / `DEFINE_IDA(name)` (macros) | per-static init | `idr::DEFINE_*` macros |
| `idr_init(idr)` / `ida_init(ida)` (header inlines) | per-dynamic init | `Idr::init` / `Ida::init` |
| `idr_preload` / `idr_preload_end` (header inlines) | per-preload pool | `Idr::preload` |
| `kfree()` / `kzalloc_obj()` | per-bitmap alloc/free | shared slab |

## Compatibility contract

REQ-1: struct idr layout (per include/linux/idr.h):
- idr_rt: struct radix_tree_root — radix-tree backing store. Must be initialised with `IDR_INIT_BASE(name, base)` or via idr_init(); xa_flags includes ROOT_IS_IDR.
- idr_base: unsigned int — id-space origin (subtracted from external id before radix-tree indexing).
- idr_next: unsigned int — next-cyclic cursor (only consumed by idr_alloc_cyclic).
- ROOT_IS_IDR tag enforces the IDR_FREE bookkeeping in the radix tree so unused slots stay marked free.

REQ-2: idr_alloc_u32(idr, ptr, *nextid, max, gfp):
- WARN_ON_ONCE(!(idr.idr_rt.xa_flags & ROOT_IS_IDR)) and force IDR_RT_MARKER.
- base = idr.idr_base; id = *nextid.
- if max < base: return -ENOSPC.
- id = (id < base) ? 0 : id - base.
- radix_tree_iter_init(&iter, id).
- slot = idr_get_free(&idr.idr_rt, &iter, gfp, max - base).
- if IS_ERR(slot): return PTR_ERR(slot) (-ENOMEM / -ENOSPC).
- *nextid = iter.index + base — published before store so concurrent RCU lookups see initialised metadata.
- radix_tree_iter_replace(&idr.idr_rt, &iter, slot, ptr) — embeds smp_wmb so readers see populated entry only after *nextid is set in the caller's struct (but this barrier is for the radix slot itself).
- radix_tree_iter_tag_clear(&idr.idr_rt, &iter, IDR_FREE).
- return 0.

REQ-3: idr_alloc(idr, ptr, start, end, gfp):
- WARN_ON_ONCE(start < 0) ⟹ -EINVAL.
- max = (end > 0 ? end - 1 : INT_MAX).
- id = start. ret = idr_alloc_u32(idr, ptr, &id, max, gfp). On error: return ret.
- return (int)id.

REQ-4: idr_alloc_cyclic(idr, ptr, start, end, gfp):
- id = idr.idr_next; if (int)id < start: id = start.
- max = (end > 0 ? end - 1 : INT_MAX).
- err = idr_alloc_u32(idr, ptr, &id, max, gfp).
- if err == -ENOSPC ∧ id > start: id = start; err = idr_alloc_u32(idr, ptr, &id, max, gfp) (one wrap retry).
- if err: return err.
- idr.idr_next = id + 1.
- return id.

REQ-5: idr_remove(idr, id):
- return radix_tree_delete_item(&idr.idr_rt, id - idr.idr_base, NULL).
- NULL when id was not present.

REQ-6: idr_find(idr, id):
- return radix_tree_lookup(&idr.idr_rt, id - idr.idr_base).
- May be called under rcu_read_lock(); RCU-safe.
- NULL distinguishes "absent" from "present and equal to NULL" only if caller never stored NULL pointers.

REQ-7: idr_replace(idr, ptr, id):
- id -= idr.idr_base.
- entry = __radix_tree_lookup(&idr.idr_rt, id, &node, &slot).
- if !slot ∨ radix_tree_tag_get(&idr.idr_rt, id, IDR_FREE): return ERR_PTR(-ENOENT).
- __radix_tree_replace(&idr.idr_rt, node, slot, ptr) (embeds barrier).
- return entry (old pointer).

REQ-8: idr_for_each(idr, fn, data):
- base = idr.idr_base.
- radix_tree_for_each_slot(slot, &idr.idr_rt, &iter, 0):
  - id = iter.index + base.
  - WARN_ON_ONCE(id > INT_MAX) ⟹ break.
  - ret = fn(id, rcu_dereference_raw(*slot), data). If ret != 0: return ret.
- return 0.
- May run concurrently with idr_alloc / idr_remove under RCU; new entries may not be seen, removed entries may be seen, but never spurious skips.

REQ-9: idr_get_next_ul(idr, *nextid):
- base = idr.idr_base; id = *nextid; id = (id < base) ? 0 : id - base.
- radix_tree_for_each_slot(slot, &idr.idr_rt, &iter, id):
  - entry = rcu_dereference_raw(*slot).
  - if !entry: continue.
  - if !xa_is_internal(entry): break.
  - if slot != &idr.idr_rt.xa_head ∧ !xa_is_retry(entry): break.
  - slot = radix_tree_iter_retry(&iter).
- if !slot: return NULL.
- *nextid = iter.index + base.
- return entry.

REQ-10: idr_get_next(idr, *nextid):
- id_ul = *nextid; entry = idr_get_next_ul(idr, &id_ul).
- WARN_ON_ONCE(id_ul > INT_MAX) ⟹ return NULL.
- *nextid = id_ul; return entry.

REQ-11: IDR locking model:
- Writers (idr_alloc / _remove / _replace): caller-supplied exclusion (e.g. spinlock around the idr).
- Readers (idr_find / idr_for_each / idr_get_next): may run under rcu_read_lock or under writer-exclusion.
- All allocation paths inherit gfp; with GFP_NOWAIT they may return -ENOMEM under contention.

REQ-12: struct ida layout:
- xa: struct xarray initialised with IDA_INIT_FLAGS (XA_FLAGS_LOCK_IRQ + XA_FLAGS_ALLOC).
- Tag XA_FREE_MARK is set on indices whose bitmap still has at least one free bit, and cleared once the bitmap is fully populated. ida_alloc_range uses xas_find_marked over XA_FREE_MARK to find the first usable index in O(log N).

REQ-13: struct ida_bitmap:
- unsigned long bitmap[IDA_BITMAP_LONGS] — IDA_BITMAP_BITS (default 1024) bits.
- Allocated lazily when the index outgrows the inline xa-value (BITS_PER_XA_VALUE bits).

REQ-14: ida_alloc_range(ida, min, max, gfp):
- if (int)min < 0: return -ENOSPC. if (int)max < 0: max = INT_MAX.
- XA_STATE(xas, &ida.xa, min / IDA_BITMAP_BITS); bit = min % IDA_BITMAP_BITS.
- retry: xas_lock_irqsave(&xas, flags).
- next: bitmap = xas_find_marked(&xas, max / IDA_BITMAP_BITS, XA_FREE_MARK).
- if xas.xa_index > min/IDA_BITMAP_BITS: bit = 0 (we advanced past min's chunk).
- if xas.xa_index*IDA_BITMAP_BITS + bit > max: goto nospc.
- Case A (xa_is_value(bitmap)):
  - tmp = xa_to_value(bitmap).
  - if bit < BITS_PER_XA_VALUE: bit = find_next_zero_bit(&tmp, BITS_PER_XA_VALUE, bit); over-max ⟹ nospc; if bit < BITS_PER_XA_VALUE: tmp |= 1<<bit; xas_store(&xas, xa_mk_value(tmp)); goto out.
  - Otherwise upgrade to a heap bitmap: bitmap = alloc or kzalloc_obj(*bitmap, GFP_NOWAIT); on NULL: goto alloc (drop lock, retry with caller gfp).
  - bitmap.bitmap[0] = tmp; xas_store(&xas, bitmap). On xas_error: revert bitmap.bitmap[0]=0; goto out.
- Case B (bitmap heap):
  - bit = find_next_zero_bit(bitmap.bitmap, IDA_BITMAP_BITS, bit). Over-max ⟹ nospc. If bit == IDA_BITMAP_BITS: goto next (advance to next chunk).
  - __set_bit(bit, bitmap.bitmap). If bitmap_full(bitmap.bitmap, IDA_BITMAP_BITS): xas_clear_mark(&xas, XA_FREE_MARK).
- Case C (bitmap == NULL):
  - if bit < BITS_PER_XA_VALUE: bitmap = xa_mk_value(1UL<<bit).
  - else: bitmap = alloc or kzalloc_obj(*bitmap, GFP_NOWAIT); on NULL: goto alloc; __set_bit(bit, bitmap.bitmap).
  - xas_store(&xas, bitmap).
- out: xas_unlock_irqrestore. xas_nomem(&xas, gfp) — if rescheduled with allocation, reset xas to min/IDA_BITMAP_BITS, bit=min%IDA_BITMAP_BITS; goto retry.
- if bitmap != alloc: kfree(alloc). xas_error ⟹ return xas_error. Return xas.xa_index*IDA_BITMAP_BITS + bit.
- alloc: xas_unlock_irqrestore. alloc = kzalloc_obj(*bitmap, gfp). On NULL: return -ENOMEM. xas_set(&xas, min/IDA_BITMAP_BITS); bit = min%IDA_BITMAP_BITS; goto retry.
- nospc: xas_unlock_irqrestore. kfree(alloc). return -ENOSPC.

REQ-15: ida_find_first_range(ida, min, max):
- if (int)min < 0: -EINVAL. if (int)max < 0: max = INT_MAX.
- xa_lock_irqsave(&ida.xa, flags).
- entry = xa_find(&ida.xa, &index, max/IDA_BITMAP_BITS, XA_PRESENT). if !entry: ret = -ENOENT; goto err_unlock.
- if index > min/IDA_BITMAP_BITS: offset = 0.
- if index*IDA_BITMAP_BITS + offset > max: -ENOENT.
- if xa_is_value(entry): tmp = xa_to_value(entry); addr = &tmp; size = BITS_PER_XA_VALUE. Else: addr = entry.bitmap; size = IDA_BITMAP_BITS.
- bit = find_next_bit(addr, size, offset). xa_unlock_irqrestore.
- if bit == size ∨ index*IDA_BITMAP_BITS + bit > max: return -ENOENT.
- return index*IDA_BITMAP_BITS + bit.

REQ-16: ida_free(ida, id):
- if (int)id < 0: return.
- XA_STATE(xas, &ida.xa, id / IDA_BITMAP_BITS); bit = id % IDA_BITMAP_BITS.
- xas_lock_irqsave; bitmap = xas_load(&xas).
- if xa_is_value(bitmap):
  - v = xa_to_value(bitmap).
  - if bit ≥ BITS_PER_XA_VALUE ∨ !(v & 1<<bit): goto err.
  - v &= ~(1<<bit). if !v: goto delete (store NULL). Else xas_store(&xas, xa_mk_value(v)).
- else (heap):
  - if !bitmap ∨ !test_bit(bit, bitmap.bitmap): goto err.
  - __clear_bit(bit, bitmap.bitmap); xas_set_mark(&xas, XA_FREE_MARK).
  - if bitmap_empty: kfree(bitmap); delete: xas_store(&xas, NULL).
- xas_unlock_irqrestore. return.
- err: xas_unlock_irqrestore; WARN(1, "ida_free called for id=%d which is not allocated.").

REQ-17: ida_destroy(ida):
- XA_STATE(xas, &ida.xa, 0). xas_lock_irqsave.
- xas_for_each(&xas, bitmap, ULONG_MAX):
  - if !xa_is_value(bitmap): kfree(bitmap).
  - xas_store(&xas, NULL).
- xas_unlock_irqrestore.

REQ-18: IDA locking model:
- Fully self-locking via xas_lock_irqsave / xa_lock_irqsave.
- Any-context (IRQ-safe).

REQ-19: ID-space management:
- IDR id-space is [idr_base, INT_MAX]; external id = internal index + idr_base.
- idr_alloc / _alloc_cyclic enforce start ≥ 0 and clamp end ≤ INT_MAX + 1; idr_alloc_u32 caps max at u32 but callers normally bound max ≤ INT_MAX.
- IDA id-space is [0, INT_MAX]. Negative min/max are rejected with -ENOSPC/-EINVAL.

REQ-20: RCU semantics:
- idr_find / idr_for_each / idr_get_next: safe under rcu_read_lock once writers serialise via their own lock.
- IDA has no RCU lookup path (locking required by ida_alloc_range comment).

REQ-21: Memory allocation behaviour:
- idr_alloc_u32: -ENOMEM only when idr_get_free needs to grow the radix tree and gfp does not permit reclaim.
- ida_alloc_range: may drop lock and retry (xas_nomem) when the XArray needs internal nodes; also explicitly kzalloc_obj's an ida_bitmap when promoting from value to bitmap. Both retries surface -ENOMEM on failure.
- ida_alloc_range never sleeps under the xa_lock; allocation happens via xas_nomem after unlock.

REQ-22: Static initialisers (include/linux/idr.h):
- DEFINE_IDR(name) = { .idr_rt = RADIX_TREE_INIT(name, IDR_RT_MARKER), .idr_base = 0, .idr_next = 0 }.
- DEFINE_IDA(name) = { .xa = XARRAY_INIT(name, IDA_INIT_FLAGS) }.

REQ-23: Preload API (idr_preload / idr_preload_end, header inlines):
- Wraps radix_tree_preload(gfp) so a subsequent idr_alloc under a spinlock can use GFP_NOWAIT without -ENOMEM.

## Acceptance Criteria

- [ ] AC-1: idr_alloc returns smallest free id in [start, end); WARN_ON_ONCE(start<0); -EINVAL on bad start; -ENOSPC if no free; -ENOMEM if radix-tree alloc fails.
- [ ] AC-2: idr_alloc_u32 publishes *nextid before radix_tree_iter_replace so concurrent rcu readers never see an uninitialised id.
- [ ] AC-3: idr_alloc_cyclic resumes from idr_next, wraps once on -ENOSPC, advances idr_next on success.
- [ ] AC-4: idr_remove returns the pointer formerly registered; NULL when absent.
- [ ] AC-5: idr_find returns the pointer or NULL; safe under rcu_read_lock with the standard publish-then-store discipline.
- [ ] AC-6: idr_replace returns ERR_PTR(-ENOENT) for unallocated ids; returns the prior pointer on success.
- [ ] AC-7: idr_for_each visits every populated id ≤ INT_MAX; bails on first non-zero callback return.
- [ ] AC-8: idr_get_next_ul / idr_get_next yield (id, ptr) pairs in ascending id order; WARN_ON_ONCE when id exceeds INT_MAX.
- [ ] AC-9: ida_alloc_range returns lowest free id in [min, max]; -ENOSPC otherwise; -ENOMEM only when allocation fails.
- [ ] AC-10: ida_alloc_range promotes xa_value → heap ida_bitmap when bit ≥ BITS_PER_XA_VALUE; clears XA_FREE_MARK when bitmap becomes full.
- [ ] AC-11: ida_free WARNs and is a no-op when freeing an id that is not set; frees the heap bitmap (storing NULL) when empty.
- [ ] AC-12: ida_destroy releases every heap ida_bitmap and clears every xa entry.
- [ ] AC-13: ida_find_first_range returns the lowest set bit in [min, max]; -ENOENT when none.
- [ ] AC-14: All IDA operations are IRQ-safe (xas_lock_irqsave / xa_lock_irqsave) and may be called from any context.

## Architecture

```
struct Idr {
    idr_rt: RadixTreeRoot,            // ROOT_IS_IDR + IDR_RT_MARKER tags
    idr_base: u32,                    // external id - idr_base = internal index
    idr_next: u32,                    // cursor for alloc_cyclic
}

struct Ida {
    xa: XArray,                       // XA_FLAGS_LOCK_IRQ | XA_FLAGS_ALLOC
}

struct IdaBitmap {
    bitmap: [usize; IDA_BITMAP_LONGS], // IDA_BITMAP_BITS bits (default 1024)
}
```

`Idr::alloc_u32(idr, ptr, *nextid, max, gfp) -> i32`:
1. WARN_ON_ONCE(!(idr.idr_rt.xa_flags & ROOT_IS_IDR)); set IDR_RT_MARKER.
2. base = idr.idr_base; id = *nextid.
3. if max < base: return -ENOSPC.
4. id = if id < base { 0 } else { id - base }.
5. radix_tree_iter_init(&iter, id).
6. slot = idr_get_free(&idr.idr_rt, &iter, gfp, max - base).
7. if IS_ERR(slot): return PTR_ERR(slot).
8. *nextid = iter.index + base.
9. radix_tree_iter_replace(&idr.idr_rt, &iter, slot, ptr).
10. radix_tree_iter_tag_clear(&idr.idr_rt, &iter, IDR_FREE).
11. return 0.

`Idr::alloc(idr, ptr, start, end, gfp) -> i32`:
1. WARN_ON_ONCE(start < 0) ⟹ -EINVAL.
2. id = start; max = if end > 0 { end - 1 } else { INT_MAX }.
3. ret = Idr::alloc_u32(idr, ptr, &id, max, gfp); if ret: return ret.
4. return id as i32.

`Idr::alloc_cyclic(idr, ptr, start, end, gfp) -> i32`:
1. id = idr.idr_next; if (int)id < start: id = start.
2. max = if end > 0 { end - 1 } else { INT_MAX }.
3. err = Idr::alloc_u32(idr, ptr, &id, max, gfp).
4. if err == -ENOSPC ∧ id > start: id = start; err = Idr::alloc_u32(idr, ptr, &id, max, gfp).
5. if err: return err.
6. idr.idr_next = id + 1.
7. return id.

`Idr::remove(idr, id) -> *mut c_void`:
1. radix_tree_delete_item(&idr.idr_rt, id - idr.idr_base, NULL).

`Idr::find(idr, id) -> *mut c_void`:
1. radix_tree_lookup(&idr.idr_rt, id - idr.idr_base).

`Idr::replace(idr, ptr, id) -> *mut c_void`:
1. id -= idr.idr_base.
2. entry = __radix_tree_lookup(&idr.idr_rt, id, &node, &slot).
3. if !slot ∨ radix_tree_tag_get(&idr.idr_rt, id, IDR_FREE): return ERR_PTR(-ENOENT).
4. __radix_tree_replace(&idr.idr_rt, node, slot, ptr).
5. return entry.

`Idr::for_each(idr, fn, data) -> i32`:
1. base = idr.idr_base.
2. radix_tree_for_each_slot(slot, &idr.idr_rt, &iter, 0):
   - id = iter.index + base.
   - WARN_ON_ONCE(id > INT_MAX) ⟹ break.
   - ret = fn(id, rcu_dereference_raw(*slot), data).
   - if ret != 0: return ret.
3. return 0.

`Idr::get_next_ul(idr, *nextid) -> *mut c_void`:
1. base = idr.idr_base; id = *nextid; id = if id < base { 0 } else { id - base }.
2. radix_tree_for_each_slot(slot, &idr.idr_rt, &iter, id):
   - entry = rcu_dereference_raw(*slot).
   - if !entry: continue.
   - if !xa_is_internal(entry): break.
   - if slot != &idr.idr_rt.xa_head ∧ !xa_is_retry(entry): break.
   - slot = radix_tree_iter_retry(&iter).
3. if !slot: return NULL.
4. *nextid = iter.index + base.
5. return entry.

`Ida::alloc_range(ida, min, max, gfp) -> i32`:
1. Validate min/max (negative ⟹ -ENOSPC/-EINVAL).
2. XA_STATE(xas, &ida.xa, min / IDA_BITMAP_BITS); bit = min % IDA_BITMAP_BITS.
3. retry: xas_lock_irqsave.
4. next: bitmap = xas_find_marked(&xas, max/IDA_BITMAP_BITS, XA_FREE_MARK).
5. Recompute bit if advanced past min's chunk; check max bound.
6. Switch on bitmap state (xa_value / heap / NULL) — see REQ-14 cases.
7. xas_store + (xas_clear_mark on full) or (kzalloc + retry) as required.
8. out: unlock; xas_nomem retry; free overshoot alloc; return id or xas_error.
9. alloc: unlock; kzalloc_obj(*bitmap, gfp); on NULL return -ENOMEM; reset xas and retry.
10. nospc: unlock; kfree(alloc); return -ENOSPC.

`Ida::free(ida, id)`:
1. if (int)id < 0: return.
2. XA_STATE(xas, &ida.xa, id/IDA_BITMAP_BITS); bit = id%IDA_BITMAP_BITS.
3. xas_lock_irqsave; bitmap = xas_load.
4. xa_value branch: v = xa_to_value(bitmap); bounds + presence check; clear bit; store value or NULL.
5. heap branch: presence check; __clear_bit; xas_set_mark(XA_FREE_MARK); if empty: kfree + xas_store(NULL).
6. err branch: WARN(1, "ida_free called for id=%d which is not allocated.").
7. xas_unlock_irqrestore.

`Ida::destroy(ida)`:
1. XA_STATE(xas, &ida.xa, 0).
2. xas_lock_irqsave; xas_for_each(&xas, bitmap, ULONG_MAX): if !xa_is_value(bitmap): kfree(bitmap); xas_store(&xas, NULL).
3. xas_unlock_irqrestore.

`Ida::find_first_range(ida, min, max) -> i32`:
1. Validate min/max.
2. xa_lock_irqsave.
3. entry = xa_find(&ida.xa, &index, max/IDA_BITMAP_BITS, XA_PRESENT); !entry ⟹ -ENOENT.
4. Adjust offset (0 if advanced); check max bound.
5. value vs heap: select addr+size; bit = find_next_bit(addr, size, offset).
6. unlock; bound check; return id or -ENOENT.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `idr_root_is_idr` | INVARIANT | per-idr_alloc_u32: ROOT_IS_IDR + IDR_RT_MARKER set in xa_flags. |
| `idr_nextid_published_pre_store` | INVARIANT | per-idr_alloc_u32: *nextid written before radix_tree_iter_replace. |
| `idr_alloc_start_nonneg` | INVARIANT | per-idr_alloc: WARN_ON_ONCE(start < 0) ⟹ -EINVAL. |
| `idr_cyclic_single_wrap` | INVARIANT | per-idr_alloc_cyclic: at most one wrap (start>0 ⟹ second attempt with id=start). |
| `idr_id_minus_base_bounds` | INVARIANT | per-idr_remove/_find/_replace: index = id - idr_base ≥ 0. |
| `idr_replace_rejects_free` | INVARIANT | per-idr_replace: IDR_FREE tag ⟹ ERR_PTR(-ENOENT). |
| `idr_for_each_int_max_guard` | INVARIANT | per-idr_for_each: WARN_ON_ONCE(id > INT_MAX) ⟹ break. |
| `ida_xa_lock_around_state` | INVARIANT | per-ida_alloc_range / _free / _find / _destroy: xa lock held across xas_load/store/find_marked. |
| `ida_value_upgrade_capacity` | INVARIANT | per-ida_alloc_range: bit ≥ BITS_PER_XA_VALUE ⟹ heap ida_bitmap allocated. |
| `ida_free_mark_consistency` | INVARIANT | per-ida_alloc_range: XA_FREE_MARK cleared exactly when bitmap_full; per-ida_free: set on any clear. |
| `ida_free_warns_on_unallocated` | INVARIANT | per-ida_free: WARN(1, ...) when bit not set. |
| `ida_destroy_frees_all_heap_bitmaps` | INVARIANT | per-ida_destroy: every !xa_is_value entry kfree'd. |

### Layer 2: TLA+

`lib/idr.tla`:
- Per-idr_alloc + per-idr_remove + per-idr_replace + per-idr_find + per-idr_for_each + per-cyclic.
- Per-ida_alloc_range + per-ida_free + per-ida_find_first_range + per-ida_destroy + per-value/bitmap promotion.
- Properties:
  - `safety_idr_no_duplicate_ids` — per-idr_alloc: never returns an id already populated.
  - `safety_idr_returned_id_in_range` — per-idr_alloc(_u32|_cyclic): start ≤ id ≤ max.
  - `safety_idr_remove_uninstalls_only_match` — per-idr_remove: state changes ⟺ id was set.
  - `safety_idr_replace_preserves_id` — per-idr_replace: id allocated before and after.
  - `safety_idr_cyclic_monotone_or_wrap` — per-alloc_cyclic: idr_next advances monotonically modulo one wrap.
  - `safety_ida_no_duplicate_bits` — per-ida_alloc_range: returned id corresponds to a previously-clear bit.
  - `safety_ida_free_clears_bit` — per-ida_free: bit cleared (and bitmap freed if empty).
  - `safety_ida_free_mark_invariant` — XA_FREE_MARK set ⟺ chunk has ≥ 1 free bit.
  - `liveness_idr_alloc_finite` — per-idr_alloc: terminates with success / -ENOMEM / -ENOSPC.
  - `liveness_ida_alloc_xas_nomem_retry_bounded` — per-ida_alloc_range: at most one allocation-retry loop.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Idr::alloc_u32` post: ret=0 ⟹ slot @ (*nextid - idr_base) holds ptr ∧ IDR_FREE cleared | `Idr::alloc_u32` |
| `Idr::alloc` post: ret ≥ 0 ⟹ start ≤ ret ≤ max | `Idr::alloc` |
| `Idr::alloc_cyclic` post: ret ≥ start ∧ idr_next == ret + 1 | `Idr::alloc_cyclic` |
| `Idr::remove` post: id-slot empty ∧ IDR_FREE tag set | `Idr::remove` |
| `Idr::replace` post: slot holds new ptr ∧ returns old ptr | `Idr::replace` |
| `Idr::for_each` post: visits every populated id ≤ INT_MAX exactly once | `Idr::for_each` |
| `Ida::alloc_range` post: ret ≥ min ∧ ret ≤ max ∧ bit at ret was 0, is now 1 | `Ida::alloc_range` |
| `Ida::free` post: bit at id is 0 ∧ (bitmap_empty ⟹ slot is NULL) | `Ida::free` |
| `Ida::destroy` post: xa empty ∧ no heap ida_bitmap leaked | `Ida::destroy` |
| `Ida::find_first_range` post: ret is smallest set bit ≥ min and ≤ max | `Ida::find_first_range` |

### Layer 4: Verus/Creusot functional

`Per-idr_alloc → idr_get_free → radix_tree_iter_replace + tag_clear`, `Per-idr_remove → radix_tree_delete_item`, `Per-idr_replace → __radix_tree_replace`, `Per-idr_for_each / get_next → radix_tree_for_each_slot`, `Per-ida_alloc_range → xas_find_marked + value/bitmap promotion + xas_clear_mark on full`, `Per-ida_free → xas_load + bit-clear + free-on-empty`, `Per-ida_destroy → xas_for_each + kfree`: semantic equivalence with per-Documentation/core-api/idr.rst and per-Documentation/core-api/xarray.rst.

## Hardening

(Inherits row-1 features from `lib/00-overview.md` § Hardening.)

IDR / IDA reinforcement:

- **Per-WARN_ON_ONCE(!ROOT_IS_IDR)** — defense against per-misuse of a raw radix-tree as an idr.
- **Per-publish-id-before-store memory order** — defense against per-RCU-reader observing uninitialised id while the slot is already populated.
- **Per-idr_alloc_cyclic single-wrap bound** — defense against per-livelock when the id-space is full.
- **Per-IDR_FREE tag check in idr_replace** — defense against per-replace-on-deleted-id silently resurrecting an id.
- **Per-WARN_ON_ONCE(id > INT_MAX) in iterators** — defense against per-overflow when callers stuff ids beyond signed-int range.
- **Per-ida_free WARN on unallocated id** — defense against per-double-free / per-stray-free corrupting the bitmap.
- **Per-XA_FREE_MARK / bitmap_full coupling** — defense against per-O(N) scans on saturated chunks.
- **Per-IRQ-safe xa lock on every IDA op** — defense against per-IRQ-during-ida_free race.
- **Per-xas_nomem bounded retry** — defense against per-allocation-thrash livelock when GFP allows reclaim.
- **Per-ida_destroy frees every heap ida_bitmap** — defense against per-128-byte leak per saturated chunk.
- **Per-idr_base subtract before radix-tree index** — defense against per-INT_MAX-overflow when callers rebase to non-zero start.
- **Per-rcu_dereference_raw in for_each / get_next** — defense against per-stale-pointer load on weakly-ordered architectures.
- **Per-kzalloc_obj(*bitmap) sized allocation** — defense against per-undersized bitmap allocation.

## Grsecurity/PaX-style Reinforcement

Baseline (apply to every Tier-3 surface):

- **PAX_USERCOPY** — IDR-mapped pointers exposed to user (e.g. `fd -> file *`) traverse owner's usercopy whitelist; IDR itself never copies to user.
- **PAX_KERNEXEC** — IDR/IDA helpers are pure data manipulation; no writable text.
- **PAX_RANDKSTACK** — caller-side stack base re-randomized; `idr_alloc`/`idr_remove` inherit fresh base.
- **PAX_REFCOUNT** — radix-tree slot refcounts (`__radix_tree_lookup` parent refs) saturate.
- **PAX_MEMORY_SANITIZE** — `radix_tree_node` slab zeroed on free; IDA bitmap slabs sanitized.
- **PAX_UDEREF** — no user-pointer deref in IDR core.
- **PAX_RAP / kCFI** — `idr_for_each` callback type-tagged.
- **GRKERNSEC_HIDESYM** — `idr_get_free`, `__radix_tree_*` hidden from non-CAP_SYSLOG kallsyms.
- **GRKERNSEC_DMESG** — IDA `WARN_ON_ONCE(ida_id < 0)` gated behind CAP_SYSLOG.

Subsystem-specific reinforcement:

- **IDR integer alloc bounds** — `idr_alloc(idr, ptr, start, end, gfp)` enforces `0 <= start < end <= INT_MAX`; PaX additionally rejects `end > id_max_per_namespace` for namespace-scoped IDRs (e.g. per-task fd IDR clamped by RLIMIT_NOFILE).
- **OOR validation on lookup** — `idr_find(idr, id)` returns NULL on `id < 0` or `id > INT_MAX`; PaX_REFCOUNT-style overflow check on the internal `unsigned long` cast prevents sign-extension confusion that has historically produced OOB reads.
- **Per-namespace ID ceiling** — IDA bitmaps backing pid/userns/ipcns allocators bounded by per-namespace `max_id`; exhaustion returns `-ENOSPC` cleanly with audit log entry, denying ID-exhaustion DoS.
- **RCU dereference discipline** — `idr_for_each` / `idr_get_next` use `rcu_dereference_raw` only inside RCU read-side critical section; PaX lockdep variant asserts `rcu_read_lock_held()` at every call.
- **Rationale** — IDR is the substrate for fd, posix-timer-id, pid, sysv-ipc-id and many other ID spaces an attacker can pump; bounded ceilings + OOR validation + RCU discipline collapse the integer-overflow and TOCTOU vectors that have produced multiple historical CVEs.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- lib/radix-tree.c radix-tree core (covered separately if expanded; idr_get_free, radix_tree_iter_* live there)
- lib/xarray.c XArray core (covered separately if expanded; XA_STATE, xas_*, xa_mk_value live there)
- lib/test_ida.c / lib/test_xarray.c kunit suites (covered separately if expanded)
- kernel/pid.c PID allocator built on IDR (covered in `kernel/pid.md` if expanded)
- net/core/dev.c per-net device id allocation (covered in `net/core.md` if expanded)
- fs/inode.c inode-number allocation via IDA (covered in `fs/inode.md` if expanded)
- include/linux/idr.h header inlines (DEFINE_IDR, DEFINE_IDA, idr_init, ida_init, idr_preload, ida_alloc, ida_alloc_min, ida_alloc_max, ida_is_empty)
- Implementation code
