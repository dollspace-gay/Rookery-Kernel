---
title: "Tier-3: mm/page_owner.c — Per-page allocation-tracking debugger"
tags: ["tier-3", "mm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-11
updated: 2026-05-11
---


## Design Specification

### Summary

`page_owner` is a per-page allocation-tracking debugger: for every `struct page` the kernel stores a `struct page_owner` record (via `page_ext`) capturing the allocation's order, gfp_mask, last-migrate-reason, allocation-task pid/tgid/comm, allocation-timestamp, free-task pid/tgid, free-timestamp, and a `depot_stack_handle_t` pointing into stackdepot for the captured call-stack. Per-allocation, `__set_page_owner()` saves the caller stack into stackdepot via `save_stack()`, then `__update_page_owner_handle()` writes the record across the `1 << order` pages. Per-free, `__reset_page_owner()` saves a free-time stack and decrements the stackdepot refcount. Per-debug, `__dump_page_owner()` is called from page-poisoning / use-after-free panics. Per-userspace, `/sys/kernel/debug/page_owner` exposes a PFN-walking dump of all currently-allocated pages with their stacks. Per-userspace, `/sys/kernel/debug/page_owner_stacks/{show_stacks,show_handles,show_stacks_handles,count_threshold}` exposes per-stack page-count aggregation. Per-`/proc/pagetypeinfo`, `pagetypeinfo_showmixedcount_print()` enriches the migratetype dump with mixed-block counts. Critical for: leak-hunting, fragmentation analysis, migrate-type debugging, allocation-attribution.

This Tier-3 covers `mm/page_owner.c` (~1001 lines).

### Acceptance Criteria

- [ ] AC-1: Boot with `page_owner=on`: static_branch enabled, page_ext_ops registered.
- [ ] AC-2: __set_page_owner: PAGE_EXT_OWNER + PAGE_EXT_OWNER_ALLOCATED set on all `1 << order` pages.
- [ ] AC-3: __set_page_owner: stackdepot handle non-zero (or `failure_handle` on stack-depot exhaustion).
- [ ] AC-4: __reset_page_owner: PAGE_EXT_OWNER_ALLOCATED cleared; free_handle / free_ts / free_pid recorded.
- [ ] AC-5: __reset_page_owner: stackdepot refcount decremented for alloc_handle (except early_handle).
- [ ] AC-6: __split_page_owner: every sub-page's order field rewritten to new_order; no stackdepot ref churn.
- [ ] AC-7: __folio_copy_owner: new folio inherits old's handle/order/gfp/pid/tgid/comm/ts; old folio retains PAGE_EXT_OWNER.
- [ ] AC-8: __dump_page_owner: emits allocated/freed state, alloc & free stacks, migrate reason.
- [ ] AC-9: /sys/kernel/debug/page_owner read: emits PFN-ordered alloc records, advances *ppos.
- [ ] AC-10: /sys/kernel/debug/page_owner_stacks/show_stacks: emits stack-trace + nr_base_pages above threshold.
- [ ] AC-11: pagetypeinfo_showmixedcount_print: emits per-migratetype mixed-block counts.
- [ ] AC-12: in_page_owner recursion guard: nested allocation from save_stack returns dummy_handle, no infinite recurse.
- [ ] AC-13: early_handle pages: enumerated in init_pages_in_zone; never dec'd on free.
- [ ] AC-14: count_threshold (debugfs u64): filters show_stacks output by nr_base_pages >= threshold.
- [ ] AC-15: dec_stack_record_count: refcount-to-zero emits pr_warn (imbalance detection).

### Architecture

```
struct PageOwner {                       // stored via page_ext
  order: u16,
  last_migrate_reason: i16,
  gfp_mask: u32,                         // gfp_t
  handle: u32,                           // depot_stack_handle_t
  free_handle: u32,
  ts_nsec: u64,
  free_ts_nsec: u64,
  comm: [u8; TASK_COMM_LEN],
  pid: i32,
  tgid: i32,
  free_pid: i32,
  free_tgid: i32,
}

struct StackNode {                       // per-distinct stackdepot record
  stack_record: *StackRecord,
  next: *StackNode,
}

struct StackPrintCtx {                   // per-seq-file private
  stack: *StackNode,
  flags: u8,                             // STACK_PRINT_FLAG_*
}
```

`PageOwner::set(page, order, gfp_mask)`:
1. ts_nsec = local_clock().
2. handle = PageOwner::save_stack(gfp_mask).
3. PageOwner::update_alloc(page, handle, order, gfp_mask, -1 /* no migrate reason */, ts_nsec, current.pid, current.tgid, current.comm).
4. PageOwner::stack_inc(handle, gfp_mask, 1 << order).

`PageOwner::reset(page, order)`:
1. free_ts_nsec = local_clock().
2. page_ext = page_ext_get(page); bail if None.
3. alloc_handle = get_page_owner(page_ext).handle.
4. page_ext_put(page_ext).
5. handle = PageOwner::save_stack(__GFP_NOWARN) /* non-spinning */.
6. PageOwner::update_free(page, handle, order, current.pid, current.tgid, free_ts_nsec).
7. if alloc_handle != EARLY_HANDLE: PageOwner::stack_dec(alloc_handle, 1 << order).

`PageOwner::save_stack(flags) -> u32`:
1. if current.in_page_owner: return DUMMY_HANDLE.
2. current.in_page_owner = true.
3. nr = stack_trace_save(entries, PAGE_OWNER_STACK_DEPTH, skip=2).
4. handle = stack_depot_save(entries[..nr], flags).
5. if handle == 0: handle = FAILURE_HANDLE.
6. current.in_page_owner = false.
7. return handle.

`PageOwner::stack_inc(handle, gfp_mask, nr_base_pages)`:
1. sr = stack_depot_get_stack_record(handle); bail if None.
2. if refcount_read(sr.count) == REFCOUNT_SATURATED:
   - if cmpxchg(sr.count.refs, SATURATED → 1):
     - PageOwner::stack_list_add(sr, gfp_mask).
3. refcount_add(nr_base_pages, &sr.count).

`PageOwner::stack_dec(handle, nr_base_pages)`:
1. sr = stack_depot_get_stack_record(handle); bail if None.
2. if refcount_sub_and_test(nr_base_pages, &sr.count): WARN(refcount → 0).

`PageOwner::stack_list_add(sr, gfp_mask)`:
1. if !gfpflags_allow_spinning(gfp_mask): return.
2. current.in_page_owner = true.
3. node = kmalloc(StackNode, gfp_nested_mask(gfp_mask)).
4. current.in_page_owner = false.
5. if !node: return.
6. node.stack_record = sr; node.next = NULL.
7. spin_lock_irqsave(&STACK_LIST_LOCK).
8. node.next = STACK_LIST.
9. smp_store_release(&STACK_LIST, node) /* pairs with stack_start's smp_load_acquire */.
10. spin_unlock_irqrestore.

`PageOwner::update_alloc(page, handle, order, gfp_mask, mr, ts, pid, tgid, comm)`:
1. rcu_read_lock.
2. for_each_page_ext(page, 1 << order, page_ext, iter):
   - po = get_page_owner(page_ext).
   - Write handle, order, gfp_mask, last_migrate_reason=mr, pid, tgid, ts_nsec=ts.
   - strscpy(po.comm, comm).
   - set_bit(PAGE_EXT_OWNER, &page_ext.flags).
   - set_bit(PAGE_EXT_OWNER_ALLOCATED, &page_ext.flags).
3. rcu_read_unlock.

`PageOwner::update_free(page, handle, order, pid, tgid, free_ts)`:
1. rcu_read_lock.
2. for_each_page_ext(page, 1 << order, page_ext, iter):
   - po = get_page_owner(page_ext).
   - if handle != 0:
     - clear_bit(PAGE_EXT_OWNER_ALLOCATED, &page_ext.flags).
     - po.free_handle = handle.
   - po.free_ts_nsec = free_ts.
   - po.free_pid = current.pid; po.free_tgid = current.tgid.
3. rcu_read_unlock.

`PageOwner::split(page, old_order, new_order)`:
1. rcu_read_lock.
2. for_each_page_ext(page, 1 << old_order, page_ext, iter):
   - get_page_owner(page_ext).order = new_order.
3. rcu_read_unlock.

`PageOwner::copy_folio(newfolio, old)`:
1. old_po = page_owner of &old.page (snapshot).
2. new_po = page_owner of &newfolio.page (snapshot).
3. migrate_handle = new_po.handle.
4. PageOwner::update_alloc(&newfolio.page, old_po.handle, old_po.order, old_po.gfp_mask, old_po.last_migrate_reason, old_po.ts_nsec, old_po.pid, old_po.tgid, old_po.comm).
5. PageOwner::update_free(&newfolio.page, 0 /* preserve ALLOCATED bit */, old_po.order, old_po.free_pid, old_po.free_tgid, old_po.free_ts_nsec).
6. /* Re-attribute new_po's prior stack to old folio for refcount balance */
7. rcu_read_lock.
8. for_each_page_ext(&old.page, 1 << new_po.order, page_ext, iter):
   - get_page_owner(page_ext).handle = migrate_handle.
9. rcu_read_unlock.

`PageOwner::dump(page)`:
1. page_ext = page_ext_get(page).
2. if !page_ext: pr_alert("There is not page extension available"); return.
3. po = get_page_owner(page_ext); mt = gfp_migratetype(po.gfp_mask).
4. if !test_bit(PAGE_EXT_OWNER, &page_ext.flags): pr_alert("never set"); page_ext_put; return.
5. pr_alert allocated-or-freed state per PAGE_EXT_OWNER_ALLOCATED.
6. pr_alert order, migratetype_names[mt], gfp_mask, pid, tgid, comm, ts_nsec, free_ts_nsec.
7. handle = READ_ONCE(po.handle); if !handle: pr_alert "missing", else stack_depot_print(handle).
8. free_handle = READ_ONCE(po.free_handle); if !free_handle: pr_alert "missing", else pr_alert free_pid/free_tgid + stack_depot_print(free_handle).
9. if last_migrate_reason != -1: pr_alert migrate_reason_names[last_migrate_reason].
10. page_ext_put.

`PageOwner::debugfs_read(buf, count, ppos) -> isize`:
1. if !PAGE_OWNER_INITED: return -EINVAL.
2. pfn = (*ppos == 0) ? min_low_pfn : *ppos.
3. While !pfn_valid(pfn) ∧ (pfn & (MAX_ORDER_NR_PAGES - 1)) != 0: pfn++.
4. for pfn in pfn..max_pfn:
   - if (pfn & (MAX_ORDER_NR_PAGES - 1)) == 0 ∧ !pfn_valid(pfn): pfn += MAX_ORDER_NR_PAGES - 1; continue.
   - page = pfn_to_page(pfn).
   - if PageBuddy: pfn += (1 << buddy_order_unsafe) - 1; continue.
   - page_ext = page_ext_get(page); skip if None / !PAGE_EXT_OWNER / !PAGE_EXT_OWNER_ALLOCATED.
   - po = get_page_owner(page_ext).
   - if !IS_ALIGNED(pfn, 1 << po.order): page_ext_put; continue.
   - handle = READ_ONCE(po.handle); if 0: page_ext_put; continue.
   - *ppos = pfn + 1.
   - po_snap = *po.
   - page_ext_put.
   - return PageOwner::print_one(buf, count, pfn, page, &po_snap, handle).
5. return 0.

`PageOwner::pagetypeinfo_mixed(seq, pgdat, zone)`:
1. For pfn in zone.zone_start_pfn..zone_end_pfn (pageblock-stepped):
   - page = pfn_to_online_page; if None: pfn = ALIGN(pfn + 1, MAX_ORDER_NR_PAGES); continue.
   - block_end = min(pageblock_end_pfn(pfn), end_pfn).
   - pageblock_mt = get_pageblock_migratetype(page).
   - for pfn in pfn..block_end:
     - page = pfn_to_page(pfn).
     - skip cross-zone pages.
     - if PageBuddy: pfn += (1 << buddy_order_unsafe) - 1; continue.
     - if PageReserved: continue.
     - page_ext = page_ext_get; skip if None / !PAGE_EXT_OWNER_ALLOCATED.
     - po = get_page_owner(page_ext); page_mt = gfp_migratetype(po.gfp_mask).
     - if page_mt != pageblock_mt:
       - count[is_migrate_cma(pageblock_mt) ? MIGRATE_MOVABLE : pageblock_mt]++.
       - pfn = block_end; page_ext_put; break.
     - pfn += (1 << po.order) - 1.
2. Emit "Node %d, zone %s" + count[0..MIGRATE_TYPES-1].

`PageOwner::init_early(zone)`:
1. For pfn in zone.zone_start_pfn..zone_end_pfn:
   - Skip !pfn_valid blocks at MAX_ORDER_NR_PAGES granule.
   - For each page in pageblock:
     - Skip cross-zone, Buddy, Reserved.
     - page_ext = page_ext_get; skip if PAGE_EXT_OWNER already set.
     - PageOwner::update_alloc(page, EARLY_HANDLE, 0, 0, -1, local_clock(), current.pid, current.tgid, current.comm).
     - count++.
   - cond_resched per pageblock.
2. pr_info per zone with count.

### Out of Scope

- mm/page_ext.c (covered separately if expanded)
- lib/stackdepot.c (covered separately if expanded)
- mm/page_alloc.c set/reset call-sites (covered in `page-allocator.md` Tier-3)
- mm/migrate.c folio_copy_owner call-sites (covered in `migration-compaction.md` Tier-3)
- mm/vmstat.c /proc/pagetypeinfo body (covered in `page-allocator.md` Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct page_owner` | per-page record | `PageOwner` |
| `struct stack` | per-stack list-node | `StackNode` |
| `struct stack_print_ctx` | per-seq-file ctx | `StackPrintCtx` |
| `page_owner_ops` | `page_ext_operations` registration | `PageOwner::OPS` |
| `early_page_owner_param()` | per-`page_owner=on` boot-param | `PageOwner::early_param` |
| `need_page_owner()` | per-page_ext need-callback | `PageOwner::need` |
| `init_page_owner()` | per-page_ext init-callback | `PageOwner::init` |
| `register_dummy_stack()` / `register_failure_stack()` / `register_early_stack()` | per-sentinel handles | `PageOwner::register_sentinels` |
| `save_stack()` | per-capture stack into stackdepot | `PageOwner::save_stack` |
| `inc_stack_record_count()` | per-alloc stack-refcount inc | `PageOwner::stack_inc` |
| `dec_stack_record_count()` | per-free stack-refcount dec | `PageOwner::stack_dec` |
| `add_stack_record_to_list()` | per-stack list-link | `PageOwner::stack_list_add` |
| `__update_page_owner_handle()` | per-alloc record-write | `PageOwner::update_alloc` |
| `__update_page_owner_free_handle()` | per-free record-write | `PageOwner::update_free` |
| `__set_page_owner()` | per-alloc entrypoint | `PageOwner::set` |
| `__reset_page_owner()` | per-free entrypoint | `PageOwner::reset` |
| `__split_page_owner()` | per-split order-rewrite | `PageOwner::split` |
| `__folio_copy_owner()` | per-migrate copy | `PageOwner::copy_folio` |
| `__folio_set_owner_migrate_reason()` | per-migrate reason | `PageOwner::set_migrate_reason` |
| `__dump_page_owner()` | per-panic dump | `PageOwner::dump` |
| `pagetypeinfo_showmixedcount_print()` | per-/proc/pagetypeinfo | `PageOwner::pagetypeinfo_mixed` |
| `read_page_owner()` | per-debugfs read-handler | `PageOwner::debugfs_read` |
| `lseek_page_owner()` | per-debugfs lseek-handler | `PageOwner::debugfs_lseek` |
| `stack_start` / `stack_next` / `stack_print` / `stack_stop` | per-seq_operations | `PageOwner::stack_seq_*` |
| `page_owner_threshold_get` / `_set` | per-threshold sysfs | `PageOwner::threshold_*` |
| `init_pages_in_zone()` / `init_early_allocated_pages()` | per-early-page seed | `PageOwner::init_early` |
| `pageowner_init()` | per-late_initcall debugfs setup | `PageOwner::debugfs_init` |
| `page_owner_inited` (static_branch) | per-feature switch | `PAGE_OWNER_INITED` |
| `PAGE_OWNER_STACK_DEPTH` (= 16) | per-stack-depth | `PAGE_OWNER_STACK_DEPTH` |

### compatibility contract

REQ-1: struct page_owner (stored via page_ext):
- order: u16, per-allocation order at alloc time.
- last_migrate_reason: i16, per-MR_* (MR_COMPACTION, MR_MEMORY_FAILURE, ...); -1 = none.
- gfp_mask: gfp_t, per-allocation gfp.
- handle: depot_stack_handle_t, per-alloc call-stack.
- free_handle: depot_stack_handle_t, per-free call-stack.
- ts_nsec: u64, per-alloc local_clock() ns.
- free_ts_nsec: u64, per-free local_clock() ns.
- comm: [u8; TASK_COMM_LEN], per-alloc task comm.
- pid / tgid: pid_t, per-alloc task pid / tgid.
- free_pid / free_tgid: pid_t, per-free task pid / tgid.

REQ-2: page_owner_ops registration:
- size = sizeof(struct page_owner).
- need = need_page_owner (true iff page_owner_enabled).
- init = init_page_owner.
- need_shared_flags = true (PAGE_EXT_OWNER / PAGE_EXT_OWNER_ALLOCATED reside in shared page_ext.flags).

REQ-3: early_page_owner_param("page_owner=on|off"):
- kstrtobool(buf, &page_owner_enabled).
- If enabled: stack_depot_request_early_init().

REQ-4: init_page_owner() (early-boot, after stackdepot ready):
- Bail if !page_owner_enabled.
- register_dummy_stack(), register_failure_stack(), register_early_stack().
- init_early_allocated_pages() — seed every already-allocated page with early_handle.
- Pull stack_record for dummy + failure, refcount_set(=, 1).
- Link dummy_stack.next = &failure_stack; stack_list = &dummy_stack.
- static_branch_enable(&page_owner_inited).

REQ-5: save_stack(flags) -> depot_stack_handle_t:
- /* Avoid recursion: page_owner may allocate while saving */
- if current.in_page_owner: return dummy_handle.
- set_current_in_page_owner.
- stack_trace_save(entries, PAGE_OWNER_STACK_DEPTH, skip=2).
- handle = stack_depot_save(entries, nr_entries, flags).
- if !handle: handle = failure_handle.
- unset_current_in_page_owner.
- return handle.

REQ-6: inc_stack_record_count(handle, gfp_mask, nr_base_pages):
- stack_record = __stack_depot_get_stack_record(handle).
- if !stack_record: return.
- /* First-use transition from REFCOUNT_SATURATED → 1, link into stack_list */
- if refcount_read(count) == REFCOUNT_SATURATED:
  - if atomic_try_cmpxchg_relaxed(refs, &REFCOUNT_SATURATED, 1):
    - add_stack_record_to_list(stack_record, gfp_mask).
- refcount_add(nr_base_pages, count).

REQ-7: dec_stack_record_count(handle, nr_base_pages):
- stack_record = __stack_depot_get_stack_record(handle).
- if !stack_record: return.
- if refcount_sub_and_test(nr_base_pages, count):
  - pr_warn("refcount went to 0 for handle %u").

REQ-8: add_stack_record_to_list(stack_record, gfp_mask):
- if !gfpflags_allow_spinning(gfp_mask): return — non-blocking caller, skip linkage.
- set_current_in_page_owner.
- stack = kmalloc(sizeof(struct stack), gfp_nested_mask(gfp_mask)).
- unset_current_in_page_owner.
- if !stack: return.
- stack.stack_record = stack_record; stack.next = NULL.
- spin_lock_irqsave(&stack_list_lock).
- stack.next = stack_list.
- smp_store_release(&stack_list, stack) — pairs with smp_load_acquire in stack_start.
- spin_unlock_irqrestore.

REQ-9: __update_page_owner_handle(page, handle, order, gfp_mask, migrate_reason, ts_nsec, pid, tgid, comm):
- rcu_read_lock.
- for_each_page_ext(page, 1 << order, page_ext, iter):
  - po = get_page_owner(page_ext).
  - Write handle, order, gfp_mask, last_migrate_reason, pid, tgid, ts_nsec.
  - strscpy(po.comm, comm).
  - __set_bit(PAGE_EXT_OWNER, &page_ext.flags).
  - __set_bit(PAGE_EXT_OWNER_ALLOCATED, &page_ext.flags).
- rcu_read_unlock.

REQ-10: __update_page_owner_free_handle(page, handle, order, pid, tgid, free_ts_nsec):
- rcu_read_lock.
- for_each_page_ext(page, 1 << order, page_ext, iter):
  - po = get_page_owner(page_ext).
  - if handle != 0 (i.e. real reset, not migrate-copy):
    - __clear_bit(PAGE_EXT_OWNER_ALLOCATED, &page_ext.flags).
    - po.free_handle = handle.
  - po.free_ts_nsec = free_ts_nsec.
  - po.free_pid = current.pid; po.free_tgid = current.tgid.
- rcu_read_unlock.

REQ-11: __set_page_owner(page, order, gfp_mask):
- ts_nsec = local_clock().
- handle = save_stack(gfp_mask).
- __update_page_owner_handle(page, handle, order, gfp_mask, -1, ts_nsec, current.pid, current.tgid, current.comm).
- inc_stack_record_count(handle, gfp_mask, 1 << order).

REQ-12: __reset_page_owner(page, order):
- free_ts_nsec = local_clock().
- page_ext = page_ext_get(page); bail if !page_ext.
- alloc_handle = get_page_owner(page_ext).handle.
- page_ext_put(page_ext).
- /* save_stack with __GFP_NOWARN so gfpflags_allow_spinning() == false — avoid spinlocks in stackdepot */
- handle = save_stack(__GFP_NOWARN).
- __update_page_owner_free_handle(page, handle, order, current.pid, current.tgid, free_ts_nsec).
- /* early_handle pages were not inc'd, do not dec */
- if alloc_handle != early_handle:
  - dec_stack_record_count(alloc_handle, 1 << order).

REQ-13: __split_page_owner(page, old_order, new_order):
- rcu_read_lock.
- for_each_page_ext(page, 1 << old_order, page_ext, iter):
  - get_page_owner(page_ext).order = new_order.
- rcu_read_unlock.
- /* Stackdepot refcount untouched: same alloc, just smaller chunks */

REQ-14: __folio_copy_owner(newfolio, old):
- /* Migration copy: preserve old stack into new folio */
- old_po = page_owner of &old.page.
- new_po = page_owner of &newfolio.page.
- migrate_handle = new_po.handle.
- __update_page_owner_handle(&newfolio.page, old_po.handle, old_po.order, old_po.gfp_mask, old_po.last_migrate_reason, old_po.ts_nsec, old_po.pid, old_po.tgid, old_po.comm).
- /* Old folio: do NOT clear bits (it will free soon), but copy free metadata across */
- __update_page_owner_free_handle(&newfolio.page, 0, old_po.order, old_po.free_pid, old_po.free_tgid, old_po.free_ts_nsec).
- /* Re-attribute the new folio's prior stack to the old folio for refcount balance */
- for_each_page_ext(&old.page, 1 << new_po.order, ...): old_po.handle = migrate_handle.

REQ-15: __folio_set_owner_migrate_reason(folio, reason):
- page_ext = page_ext_get(&folio.page); bail if !page_ext.
- get_page_owner(page_ext).last_migrate_reason = reason.
- page_ext_put.

REQ-16: pagetypeinfo_showmixedcount_print(seq, pgdat, zone):
- Walks zone in pageblock_nr_pages steps.
- For each page in block: if PageBuddy → skip 1 << freepage_order; if PageReserved → skip; else look up page_ext, require PAGE_EXT_OWNER_ALLOCATED.
- page_mt = gfp_migratetype(po.gfp_mask); pageblock_mt = get_pageblock_migratetype(page).
- If page_mt != pageblock_mt: count[is_migrate_cma(pageblock_mt) ? MIGRATE_MOVABLE : pageblock_mt]++; break out of block.
- Emit "Node %d, zone %s" + count[0..MIGRATE_TYPES-1].

REQ-17: __dump_page_owner(page) — called from page_alloc bug-paths:
- Resolve page_ext; if !page_ext: pr_alert("There is not page extension available") and return.
- Check PAGE_EXT_OWNER: if !set, pr_alert("page_owner info is not present (never set?)") and return.
- pr_alert allocated-or-freed-state per PAGE_EXT_OWNER_ALLOCATED.
- pr_alert order, migratetype-name, gfp_mask, pid, tgid, comm, ts_nsec, free_ts_nsec.
- stack_depot_print(handle) for alloc-stack; pr_alert "missing" if 0.
- stack_depot_print(free_handle) for free-stack with free_pid/free_tgid prelude; pr_alert "missing" if 0.
- If last_migrate_reason != -1: print "page has been migrated, last migrate reason: %s".

REQ-18: read_page_owner(file, buf, count, ppos) — /sys/kernel/debug/page_owner:
- Bail with -EINVAL if !page_owner_inited.
- pfn = (*ppos == 0) ? min_low_pfn : *ppos.
- Advance pfn to next valid pfn within MAX_ORDER_NR_PAGES granule.
- For pfn < max_pfn:
  - Skip MAX_ORDER_NR_PAGES regions that are not pfn_valid.
  - page = pfn_to_page; if PageBuddy: skip 1 << buddy_order_unsafe.
  - page_ext = page_ext_get; skip if !PAGE_EXT_OWNER or !PAGE_EXT_OWNER_ALLOCATED.
  - Skip non-head pages: !IS_ALIGNED(pfn, 1 << po.order).
  - handle = READ_ONCE(po.handle); skip if 0.
  - *ppos = pfn + 1.
  - Snapshot page_owner into stack-local (page_owner_tmp), drop page_ext_put, then print_page_owner to user.
- print_page_owner: kmalloc PAGE_SIZE buffer, scnprintf header, scnprintf pageblock-mt + page-mt + flags, stack_depot_snprint, optional last_migrate_reason, optional memcg info, copy_to_user.

REQ-19: /sys/kernel/debug/page_owner_stacks/* seq-file:
- show_stacks → flags = STACK_PRINT_FLAG_STACK | STACK_PRINT_FLAG_PAGES.
- show_handles → flags = STACK_PRINT_FLAG_HANDLE | STACK_PRINT_FLAG_PAGES.
- show_stacks_handles → flags = STACK_PRINT_FLAG_STACK | STACK_PRINT_FLAG_HANDLE.
- count_threshold → DEFINE_SIMPLE_ATTRIBUTE(page_owner_threshold_fops, &threshold_get, &threshold_set, "%llu").
- stack_start uses smp_load_acquire(&stack_list).
- stack_print computes nr_base_pages = refcount - 1; gated by STACK_PRINT_FLAG_PAGES threshold.

REQ-20: init_pages_in_zone(zone):
- For each pfn in zone, skip Buddy / Reserved / already-set-PAGE_EXT_OWNER.
- For un-tagged pages: __update_page_owner_handle(page, early_handle, order=0, gfp=0, mr=-1, local_clock(), current.pid/tgid/comm).
- cond_resched() per pageblock.
- pr_info per zone with discovered count.

REQ-21: PAGE_OWNER_STACK_DEPTH = 16 — fixed compile-time stack-trace depth.

REQ-22: Per-recursion-guard via task.in_page_owner — set/unset around any allocation done by page_owner.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `recursion_guard_invariant` | INVARIANT | per-save_stack: nested call returns DUMMY_HANDLE without re-entering stackdepot. |
| `stack_inc_dec_refcount_balanced` | INVARIANT | per-set+reset pair on same handle: refcount net delta == 0 (except early_handle). |
| `early_handle_never_decremented` | INVARIANT | per-reset: alloc_handle == early_handle ⟹ no stack_dec. |
| `page_ext_owner_bit_set_on_alloc` | INVARIANT | per-update_alloc: PAGE_EXT_OWNER + PAGE_EXT_OWNER_ALLOCATED both set. |
| `page_ext_owner_allocated_cleared_on_reset` | INVARIANT | per-update_free with handle != 0: PAGE_EXT_OWNER_ALLOCATED cleared. |
| `stack_list_smp_release_acquire` | INVARIANT | per-stack_list_add: smp_store_release(&stack_list, …) pairs with smp_load_acquire in stack_start. |
| `seq_file_no_use_after_put` | INVARIANT | per-debugfs_read: page_ext_put after po snapshot, before print to user. |
| `bounded_stack_depth` | INVARIANT | per-save_stack: nr_entries ≤ PAGE_OWNER_STACK_DEPTH (= 16). |
| `migrate_handle_swap_atomic` | INVARIANT | per-copy_folio: migrate_handle captured before update_alloc overwrites new_po.handle. |

### Layer 2: TLA+

`mm/page-owner.tla`:
- Per-alloc + per-free + per-split + per-migrate + per-recursive-save + per-debugfs-walk.
- Properties:
  - `safety_refcount_nonnegative` — per-handle: refcount never < 0.
  - `safety_no_dec_without_prior_inc` — per-handle: every stack_dec preceded by stack_inc (or early_handle).
  - `safety_no_double_clear` — per-page: PAGE_EXT_OWNER_ALLOCATED cleared at most once per alloc/free cycle.
  - `safety_recursion_bounded` — per-task: in_page_owner re-entry returns dummy_handle, depth ≤ 1.
  - `safety_split_preserves_alloc_bit` — per-split: PAGE_EXT_OWNER_ALLOCATED unchanged on all sub-pages.
  - `liveness_per_alloc_eventually_dumpable` — per-set: __dump_page_owner subsequently reads consistent record.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `PageOwner::save_stack` post: handle ∈ {valid, DUMMY_HANDLE, FAILURE_HANDLE} | `PageOwner::save_stack` |
| `PageOwner::set` post: ∀ p in span: PAGE_EXT_OWNER ∧ PAGE_EXT_OWNER_ALLOCATED ∧ handle written | `PageOwner::set` |
| `PageOwner::reset` post: ∀ p in span: PAGE_EXT_OWNER_ALLOCATED cleared ∧ free_handle written | `PageOwner::reset` |
| `PageOwner::stack_inc` post: refcount += nr_base_pages | `PageOwner::stack_inc` |
| `PageOwner::stack_dec` post: refcount -= nr_base_pages; warn if hits 0 | `PageOwner::stack_dec` |
| `PageOwner::copy_folio` post: new_po inherits old; migrate_handle re-attributed to old | `PageOwner::copy_folio` |
| `PageOwner::dump` post: pr_alert lines emitted in fixed order (state, header, alloc-stack, free-stack, migrate) | `PageOwner::dump` |
| `PageOwner::debugfs_read` post: returns one record per call; *ppos monotone non-decreasing | `PageOwner::debugfs_read` |

### Layer 4: Verus/Creusot functional

`Per-allocation → save_stack → stack_inc → update_alloc (set PAGE_EXT_OWNER + PAGE_EXT_OWNER_ALLOCATED) → … → reset (clear PAGE_EXT_OWNER_ALLOCATED + save free stack + stack_dec)` semantic equivalence: per-Documentation/mm/page_owner.rst.

### hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening.)

page_owner reinforcement:

- **Per-`current.in_page_owner` recursion-guard** — defense against per-save_stack-allocates-which-saves-stack infinite recurse.
- **Per-FAILURE_HANDLE fallback** — defense against per-stackdepot-exhaustion silent loss; tagged record remains traceable.
- **Per-DUMMY_HANDLE for recursion** — defense against per-skipped-record being indistinguishable from missing-record.
- **Per-EARLY_HANDLE special-case (no decrement)** — defense against per-init_pages_in_zone-seeded pages being underflow'd by later free.
- **Per-`refcount_sub_and_test` pr_warn on zero** — defense against per-imbalanced inc/dec (lost-alloc / double-free attribution).
- **Per-`page_ext_get`/`page_ext_put` discipline** — defense against per-page_ext UAF; snapshot to stack-local before put.
- **Per-`smp_store_release`/`smp_load_acquire` on stack_list** — defense against per-reader-sees-partial-link torn-list.
- **Per-`gfpflags_allow_spinning` gate for stack_list_add** — defense against per-non-blocking-context spin-lock acquisition.
- **Per-`gfp_nested_mask` for nested kmalloc** — defense against per-reclaim-recursion via page_owner's own allocations.
- **Per-static_branch_unlikely(page_owner_inited)** — defense against per-hot-path overhead when feature disabled.
- **Per-`READ_ONCE(po.handle)` in debugfs / dump** — defense against per-torn-handle read during concurrent alloc/free.
- **Per-`copy_to_user` after `page_ext_put`** — defense against per-fault-while-holding-page_ext-ref deadlock.
- **Per-`cond_resched()` in init_pages_in_zone** — defense against per-early-boot soft-lockup on huge zones.

