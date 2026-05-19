# Tier-3: fs/dax.c — Direct Access (DAX) for persistent memory

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: fs/00-overview.md
upstream-paths:
  - fs/dax.c (~2278 lines)
  - include/linux/dax.h
  - include/trace/events/fs_dax.h
-->

## Summary

**DAX** (Direct Access) bypasses the page cache for files backed by persistent memory (`dax_device`-backed storage: pmem, fsdax, devdax). Reads / writes copy directly between user buffers and the pmem PFN; faults install pmem PFNs into the user PTE/PMD with no struct-page-backed cache. The `address_space->i_pages` xarray stores **DAX entries** (xa-value entries, not struct-page pointers) tagged with per-entry flags: `DAX_LOCKED` (entry-lock bit), `DAX_PMD` (PMD-size entry), `DAX_ZERO_PAGE` (shared zero-fill mapping for hole reads), `DAX_EMPTY` (placeholder for in-progress fault). Per-i_pages spinlock (`xa_lock`) serializes DAX entry mutation. Per-entry wait-queue (`wait_table[DAX_WAIT_TABLE_ENTRIES]`, hash-indexed on `(xa, pgoff)`) parks faulters when an entry is locked. Per-`dax_iomap_rw`: iomap-iterator-driven direct copy via `dax_direct_access(dax_dev, pgoff, ...)` returning a kaddr/pfn. Per-`dax_iomap_fault` (PTE order=0 or PMD order=PMD_ORDER): grab entry, query iomap, install pfn. Per-`dax_writeback_mapping_range`: walks `PAGECACHE_TAG_TOWRITE`-marked entries, `pfn_mkclean_range` write-protects each mapping, `dax_flush(dax_dev, vaddr, size)` flushes to persistence-domain (CLWB/CLFLUSHOPT). Per-`dax_break_layout`: drains `get_user_pages`-elevated pmem pinning before truncate/punch-hole (filesystem callbacks invoked by ftruncate). Critical for: pmem-aware file I/O, MAP_SYNC durability, MMU-notifier invalidation, CoW (reflink) on shared pmem extents.

This Tier-3 covers `fs/dax.c` (~2278 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct dax_device` | per-pmem-driver handle | `DaxDevice` |
| `dax_iomap_rw()` | per-read/write entry | `Dax::iomap_rw` |
| `dax_iomap_iter()` | per-segment iomap I/O | `Dax::iomap_iter` |
| `dax_iomap_fault()` | per-fault dispatch (order=0 vs PMD) | `Dax::iomap_fault` |
| `dax_iomap_pte_fault()` | per-PTE fault | `Dax::iomap_pte_fault` |
| `dax_iomap_pmd_fault()` | per-PMD fault | `Dax::iomap_pmd_fault` |
| `dax_fault_iter()` | per-fault PFN insert | `Dax::fault_iter` |
| `dax_load_hole()` | per-hole PTE zero-page | `Dax::load_hole` |
| `dax_pmd_load_hole()` | per-hole PMD huge-zero | `Dax::pmd_load_hole` |
| `dax_writeback_mapping_range()` | per-writeback range | `Dax::writeback_mapping_range` |
| `dax_writeback_one()` | per-entry write-protect+flush | `Dax::writeback_one` |
| `grab_mapping_entry()` | per-fault entry acquire / install | `Dax::grab_mapping_entry` |
| `dax_insert_entry()` | per-fault entry install | `Dax::insert_entry` |
| `dax_lock_entry()` / `dax_unlock_entry()` | per-entry bit-lock | `Dax::lock_entry` / `unlock_entry` |
| `get_next_unlocked_entry()` | per-xas lookup + wait | `Dax::get_next_unlocked_entry` |
| `wait_entry_unlocked()` / `_exclusive` | per-locked-entry sleep | `Dax::wait_entry_unlocked` / `_exclusive` |
| `dax_wake_entry()` | per-entry wakeup | `Dax::wake_entry` |
| `dax_lock_folio()` / `dax_unlock_folio()` | per-folio reverse lock | `Dax::lock_folio` / `unlock_folio` |
| `dax_lock_mapping_entry()` | per-mapping reverse lock | `Dax::lock_mapping_entry` |
| `dax_layout_busy_page_range()` | per-pin scan | `Dax::layout_busy_page_range` |
| `dax_break_layout()` | per-ftruncate drain | `Dax::break_layout` |
| `dax_break_layout_final()` | per-evict-inode drain | `Dax::break_layout_final` |
| `dax_delete_mapping_entry()` | per-truncate evict-1 | `Dax::delete_mapping_entry` |
| `dax_delete_mapping_range()` | per-truncate evict-range | `Dax::delete_mapping_range` |
| `__dax_invalidate_entry()` | per-invalidate (sync) | `Dax::invalidate_entry_inner` |
| `dax_invalidate_mapping_entry_sync()` | per-clean-only invalidate | `Dax::invalidate_mapping_entry_sync` |
| `dax_finish_sync_fault()` | per-MAP_SYNC fsync + insert | `Dax::finish_sync_fault` |
| `dax_insert_pfn_mkwrite()` | per-MAP_SYNC PFN install | `Dax::insert_pfn_mkwrite` |
| `dax_iomap_pgoff()` | per-iomap → pmem pgoff | `Dax::iomap_pgoff` |
| `dax_iomap_direct_access()` | per-iomap direct-access query | `Dax::iomap_direct_access` |
| `dax_iomap_copy_around()` | per-CoW edge copy | `Dax::iomap_copy_around` |
| `dax_zero_range()` / `dax_truncate_page()` | per-zero | `Dax::zero_range` / `truncate_page` |
| `dax_zero_iter()` / `dax_memzero()` | per-zero-iter | `Dax::zero_iter` / `memzero` |
| `dax_file_unshare()` / `dax_unshare_iter()` | per-reflink-unshare | `Dax::file_unshare` / `unshare_iter` |
| `dax_dedupe_file_range_compare()` | per-dedupe equality | `Dax::dedupe_compare` |
| `dax_remap_file_range_prep()` | per-reflink prep | `Dax::remap_file_range_prep` |
| `dax_folio_reset_order()` | per-shared-folio split | `Dax::folio_reset_order` |
| `dax_associate_entry()` / `dax_disassociate_entry()` | per-folio backref | `Dax::associate_entry` / `disassociate_entry` |
| `dax_folio_make_shared()` / `dax_folio_put()` | per-share refcount | `Dax::folio_make_shared` / `folio_put` |
| `dax_to_pfn()` / `dax_make_entry()` | per-encoding | `Dax::to_pfn` / `make_entry` |
| `dax_is_pmd_entry()` / `_pte` / `_zero` / `_empty` / `_locked` | per-flag-test | `DaxEntry::is_*` |
| `dax_entry_order()` / `dax_entry_size()` | per-order / size | `DaxEntry::order` / `size` |
| `dax_fault_is_synchronous()` | per-MAP_SYNC + IOMAP_F_DIRTY | `Dax::fault_is_synchronous` |
| `dax_fault_check_fallback()` | per-PMD fallback gate | `Dax::fault_check_fallback` |
| `dax_fault_synchronous_pfnp()` | per-NEEDDSYNC pfn export | `Dax::fault_synchronous_pfnp` |
| `dax_fault_cow_page()` | per-private-CoW page | `Dax::fault_cow_page` |
| `copy_cow_page_dax()` | per-CoW-from-pmem | `Dax::copy_cow_page` |
| `wait_page_idle()` / `_uninterruptible` | per-DMA-drain wait | `Dax::wait_page_idle` / `_uninterruptible` |
| `dax_busy_page()` | per-pin detect | `Dax::busy_page` |
| `dax_read_lock()` / `dax_read_unlock()` | per-srcu | shared with `drivers/dax` |
| `dax_direct_access()` / `dax_recovery_write()` | per-driver kaddr/pfn | shared `DaxOps` trait |
| `dax_copy_from_iter()` / `dax_copy_to_iter()` / `dax_zero_page_range()` / `dax_flush()` | per-driver bytes | shared `DaxOps` trait |
| `wait_table[DAX_WAIT_TABLE_ENTRIES]` | per-bit-lock wait | `Dax::wait_table` (4096) |
| `wake_exceptional_entry_func()` | per-wake match-pred | `Dax::wake_exceptional_entry` |

## Compatibility contract

REQ-1: DAX xarray entry encoding (`include/linux/dax.h` + `fs/dax.c`):
- Entries are `xa_value` (low-bit-tagged) ⟹ never confused with struct-page pointers.
- Low 4 bits ≡ flags: `DAX_LOCKED (1<<0)`, `DAX_PMD (1<<1)`, `DAX_ZERO_PAGE (1<<2)`, `DAX_EMPTY (1<<3)`.
- Bits above `DAX_SHIFT=4` ≡ PFN.
- `dax_make_entry(pfn, flags)` = `xa_mk_value(flags | (pfn << 4))`.
- `dax_to_pfn(entry)` = `xa_to_value(entry) >> 4`.
- `dax_entry_order(entry)` = `PMD_ORDER` if `DAX_PMD` else `0`.
- `dax_entry_size(entry)` = `PMD_SIZE` (PMD) / `PAGE_SIZE` (PTE) / `0` (zero/empty).

REQ-2: DAX entry types:
- **Normal PTE** (`!DAX_PMD ∧ !DAX_ZERO ∧ !DAX_EMPTY`): real pmem PFN, order-0.
- **Normal PMD** (`DAX_PMD ∧ !DAX_ZERO ∧ !DAX_EMPTY`): real pmem PFN, order PMD_ORDER.
- **PTE zero** (`DAX_ZERO_PAGE`): shared read-only zero-PFN (`zero_pfn`) for hole reads.
- **PMD zero** (`DAX_PMD | DAX_ZERO_PAGE`): huge-zero folio for hole reads.
- **PTE/PMD empty** (`DAX_EMPTY` [+ `DAX_PMD`]): placeholder reserved during in-progress fault (no PFN bits valid).
- All four may additionally bear `DAX_LOCKED`.

REQ-3: Per-entry bit-lock + wait-queue:
- `dax_lock_entry(xas, entry)` ⟹ `xas_store(xas, dax_make_entry(pfn, flags | DAX_LOCKED))` (atomic OR under `xa_lock`).
- `dax_unlock_entry(xas, entry)` ⟹ `xas_store(xas, entry)` (without LOCKED) then `dax_wake_entry(WAKE_NEXT)`.
- `wait_table[]` ≡ array of 4096 (`DAX_WAIT_TABLE_ENTRIES = 1 << 12`) `wait_queue_head_t`.
- `dax_entry_waitqueue(xas, entry, key)`: hash = `hash_long((xas.xa ^ index), 12)`; for PMD entry `index &= ~PG_PMD_COLOUR` so all PMD-internal offsets hash to same wq.
- `wake_exceptional_entry_func` matches `(xa, entry_start)` before consuming wakeup.

REQ-4: `get_next_unlocked_entry(xas, order)` (caller holds `xas_lock_irq`):
- Loop:
  - `entry = xas_find_conflict(xas)`.
  - if `!entry ∨ !xa_is_value(entry)`: return entry.
  - if `dax_entry_order(entry) < order`: return `XA_RETRY_ENTRY` (PMD requested, PTE present).
  - if `!dax_is_locked(entry)`: return entry.
  - Park on wait-queue, `xas_unlock_irq`, schedule, re-`xas_lock_irq`, retry.

REQ-5: `grab_mapping_entry(xas, mapping, order)` (per-fault entry acquire/install):
- Loop (`retry`):
  - `xas_lock_irq`.
  - `entry = get_next_unlocked_entry(xas, order)`.
  - if `entry ∧ dax_is_conflict(entry)`: → fallback (return `xa_mk_internal(VM_FAULT_FALLBACK)`).
  - if `entry ∧ !xa_is_value(entry)`: set `xas_err -EIO`; → out_unlock.
  - if `order == 0 ∧ dax_is_pmd_entry(entry) ∧ (dax_is_zero ∨ dax_is_empty)`: `pmd_downgrade=true`.
  - if `pmd_downgrade`:
    - `dax_lock_entry(xas, entry)` (pin while we drop lock).
    - if `dax_is_zero_entry(entry)`: `xas_unlock_irq`, `unmap_mapping_pages(mapping, idx & ~PG_PMD_COLOUR, PG_PMD_NR, false)`, `xas_reset`, `xas_lock_irq`.
    - `dax_disassociate_entry(entry, mapping, false)`, `xas_store(xas, NULL)`, `dax_wake_entry(WAKE_ALL)`, `mapping->nrpages -= PG_PMD_NR`, entry = NULL, `xas_set(xas, index)`.
  - if entry: `dax_lock_entry(xas, entry)`.
  - else: install `dax_make_entry(0, DAX_EMPTY | (DAX_PMD if order>0 else 0))`; `dax_lock_entry`; `mapping->nrpages += (1 << order)`.
  - `xas_unlock_irq`.
  - if `xas_nomem(xas, mapping_gfp & ~__GFP_HIGHMEM)`: retry.
  - if `xas->xa_node == XA_ERROR(-ENOMEM)`: return `xa_mk_internal(VM_FAULT_OOM)`.
  - if `xas_error(xas)`: return `xa_mk_internal(VM_FAULT_SIGBUS)`.
  - return entry.
- fallback: `xas_unlock_irq`; return `xa_mk_internal(VM_FAULT_FALLBACK)`.

REQ-6: `dax_insert_entry(xas, vmf, iter, entry, pfn, flags)` — install final entry post-fault:
- new_entry = `dax_make_entry(pfn, flags)`; write = `IOMAP_WRITE`; dirty = write ∧ `!dax_fault_is_synchronous(iter, vma)`; shared = `IOMAP_F_SHARED`.
- if dirty: `__mark_inode_dirty(I_DIRTY_PAGES)`.
- if shared ∨ (zero ∧ !DAX_ZERO_PAGE):
  - if PMD: `unmap_mapping_pages(mapping, idx & ~PG_PMD_COLOUR, PG_PMD_NR, false)`.
  - else: `unmap_mapping_pages(mapping, idx, 1, false)`.
- `xas_reset`; `xas_lock_irq`.
- if shared ∨ zero ∨ empty:
  - `dax_disassociate_entry(entry, mapping, false)`.
  - `dax_associate_entry(new_entry, mapping, vma, addr, shared)`.
  - `old = dax_lock_entry(xas, new_entry)`; WARN if `old != xa_mk_value(xa_to_value(entry) | DAX_LOCKED)`.
  - entry = new_entry.
- else: `xas_load(xas)` (walk xa_state only; leave existing entry).
- if dirty: `xas_set_mark(PAGECACHE_TAG_DIRTY)`.
- if write ∧ shared: `xas_set_mark(PAGECACHE_TAG_TOWRITE)`.
- `xas_unlock_irq`; return entry.

REQ-7: `dax_fault_iter(vmf, iter, pfnp, xas, entry, pmd)` (the common PTE/PMD fault actor):
- size = `pmd ? PMD_SIZE : PAGE_SIZE`; pos = `xa_index << PAGE_SHIFT`; write = `IOMAP_WRITE`; entry_flags = `pmd ? DAX_PMD : 0`.
- if `!pmd ∧ vmf->cow_page`: `dax_fault_cow_page(vmf, iter)` (private CoW path).
- if `!write ∧ (UNWRITTEN ∨ HOLE)`:
  - if `!pmd`: `dax_load_hole(xas, vmf, iter, entry)` (PTE zero-PFN).
  - else: `dax_pmd_load_hole(xas, vmf, iter, entry)` (PMD huge-zero).
- if `iomap.type != MAPPED ∧ !IOMAP_F_SHARED`: WARN; return `pmd ? FALLBACK : SIGBUS`.
- `dax_iomap_direct_access(iomap, pos, size, &kaddr, &pfn)`.
- `*entry = dax_insert_entry(xas, vmf, iter, *entry, pfn, entry_flags)`.
- if write ∧ shared: `dax_iomap_copy_around(pos, size, size, srcmap, kaddr)`.
- if `dax_fault_is_synchronous(iter, vma)`: return `dax_fault_synchronous_pfnp(pfnp, pfn)` ⟹ stores pfn, returns `VM_FAULT_NEEDDSYNC` so caller fsyncs then calls `dax_finish_sync_fault`.
- folio = `dax_to_folio(*entry)`; `folio_ref_inc(folio)`.
- if pmd: `ret = vmf_insert_folio_pmd(vmf, pfn_folio(pfn), write)`.
- else: `ret = vmf_insert_page_mkwrite(vmf, pfn_to_page(pfn), write)`.
- `folio_put(folio)`; return ret.

REQ-8: `dax_iomap_pte_fault(vmf, pfnp, iomap_errp, ops)` (order==0):
- `XA_STATE(xas, mapping->i_pages, vmf->pgoff)`.
- iter = `{ inode, pos = pgoff<<PAGE_SHIFT, len = PAGE_SIZE, flags = IOMAP_DAX | IOMAP_FAULT }`.
- if `iter.pos >= i_size_read(inode)`: ret = SIGBUS; goto out.
- if `(vmf->flags & FAULT_FLAG_WRITE) ∧ !cow_page`: `iter.flags |= IOMAP_WRITE`.
- `entry = grab_mapping_entry(xas, mapping, 0)`.
- if `xa_is_internal(entry)`: ret = `xa_to_internal(entry)`; goto out.
- if `pmd_trans_huge(*vmf->pmd)`: ret = `VM_FAULT_NOPAGE` (racing PMD fault); goto unlock_entry.
- Loop `iomap_iter(&iter, ops) > 0`:
  - if `iomap_length(iter) < PAGE_SIZE`: status = -EIO; continue (corruption).
  - `ret = dax_fault_iter(vmf, &iter, pfnp, &xas, &entry, false)`.
  - if `ret != SIGBUS ∧ (iomap.flags & IOMAP_F_NEW)`: `count_vm_event(PGMAJFAULT)`; `ret |= VM_FAULT_MAJOR`.
  - if `!(ret & VM_FAULT_ERROR)`: `iter.status = iomap_iter_advance(&iter, PAGE_SIZE)`.
- if iomap_errp: `*iomap_errp = error`.
- if `!ret ∧ error`: ret = `dax_fault_return(error)`.
- unlock_entry: `dax_unlock_entry(xas, entry)`.

REQ-9: `dax_iomap_pmd_fault(vmf, pfnp, ops)` (order==PMD_ORDER) [under `CONFIG_FS_DAX_PMD`]:
- `XA_STATE_ORDER(xas, mapping->i_pages, vmf->pgoff, PMD_ORDER)`.
- max_pgoff = `DIV_ROUND_UP(i_size, PAGE_SIZE)`.
- if `xas.xa_index >= max_pgoff`: ret = SIGBUS; goto out.
- if `dax_fault_check_fallback(vmf, &xas, max_pgoff)`: goto fallback.
- `entry = grab_mapping_entry(xas, mapping, PMD_ORDER)`.
- if `xa_is_internal(entry)`: ret = `xa_to_internal(entry)`; goto fallback.
- if `!pmd_none(*vmf->pmd) ∧ !pmd_trans_huge(*vmf->pmd)`: ret = 0 (racing PTE fault); goto unlock_entry.
- Loop `iomap_iter(&iter, ops) > 0`:
  - if `iomap_length(iter) < PMD_SIZE`: continue (breaks loop next).
  - `ret = dax_fault_iter(vmf, &iter, pfnp, &xas, &entry, true)`.
  - if `ret != FALLBACK`: `iomap_iter_advance(&iter, PMD_SIZE)`.
- unlock_entry: `dax_unlock_entry(xas, entry)`.
- fallback: if `ret == FALLBACK`: `split_huge_pmd(vma, pmd, addr)`; `count_vm_event(THP_FAULT_FALLBACK)`.

REQ-10: `dax_iomap_fault(vmf, order, pfnp, iomap_errp, ops)` dispatcher:
- if `order == 0`: `dax_iomap_pte_fault(...)`.
- else if `order == PMD_ORDER`: `dax_iomap_pmd_fault(...)`.
- else: `VM_FAULT_FALLBACK`.

REQ-11: `dax_fault_check_fallback(vmf, xas, max_pgoff)` (PMD-only gate):
- if `(pgoff & PG_PMD_COLOUR) != ((address>>PAGE_SHIFT) & PG_PMD_COLOUR)`: true (color mismatch).
- if `write ∧ !VM_SHARED`: true (private CoW unsupported at PMD).
- if `pmd_addr < vma->vm_start`: true.
- if `pmd_addr + PMD_SIZE > vma->vm_end`: true.
- if `(xa_index | PG_PMD_COLOUR) >= max_pgoff`: true (PMD extends past EOF).
- else false.

REQ-12: `dax_load_hole` / `dax_pmd_load_hole` (hole = read of unmapped region):
- PTE: `pfn = zero_pfn(vaddr)`; `*entry = dax_insert_entry(..., DAX_ZERO_PAGE)`; `vmf_insert_page_mkwrite(vmf, pfn_to_page(pfn), false)`.
- PMD: `zero_folio = mm_get_huge_zero_folio(vma->vm_mm)`; if `!zero_folio`: return `VM_FAULT_FALLBACK`. `*entry = dax_insert_entry(..., DAX_PMD | DAX_ZERO_PAGE)`; `vmf_insert_folio_pmd(vmf, zero_folio, false)`.

REQ-13: `dax_iomap_rw(iocb, iter, ops)` (read/write entry):
- iomi = `{ inode, pos = ki_pos, len = iov_iter_count(iter), flags = IOMAP_DAX }`.
- WARN+EIO if `iocb->ki_flags & IOCB_ATOMIC`.
- if `iov_iter_rw(iter) == WRITE`: `lockdep_assert_held_write(&inode->i_rwsem)`; `flags |= IOMAP_WRITE`.
- else if `!sb_rdonly`: `lockdep_assert_held(&inode->i_rwsem)`.
- if `IOCB_NOWAIT`: `flags |= IOMAP_NOWAIT`.
- Loop `iomap_iter(&iomi, ops) > 0`: `iomi.status = dax_iomap_iter(&iomi, iter)`.
- done = `iomi.pos - iocb->ki_pos`; `iocb->ki_pos = iomi.pos`; return `done ?: ret`.

REQ-14: `dax_iomap_iter(iomi, iter)` (per-segment copy):
- write = `iov_iter_rw == WRITE`; cow = write ∧ `IOMAP_F_SHARED`.
- if `!write`:
  - end = `min(end, i_size_read)`.
  - if `pos >= end`: return 0.
  - if `iomap.type == HOLE ∨ UNWRITTEN`: `iov_iter_zero(min(length, end-pos), iter)`; `iomap_iter_advance(iomi, done)`.
- WARN+EIO if `iomap.type != MAPPED ∧ !IOMAP_F_SHARED`.
- if `IOMAP_F_NEW ∨ cow`:
  - if cow: `__dax_clear_dirty_range(mapping, start_idx, end_idx)`.
  - `invalidate_inode_pages2_range(mapping, start_idx, end_idx)`.
- `id = dax_read_lock()`.
- while `(pos = iomi->pos) < end`:
  - offset = `pos & ~PAGE_MASK`; size = `ALIGN(length+offset, PAGE_SIZE)`; pgoff = `dax_iomap_pgoff(iomap, pos)`.
  - if `fatal_signal_pending(current)`: ret = -EINTR; break.
  - `map_len = dax_direct_access(dax_dev, pgoff, PHYS_PFN(size), DAX_ACCESS, &kaddr, NULL)`.
  - if `map_len == -EHWPOISON ∧ WRITE`: retry with `DAX_RECOVERY_WRITE`; recovery = true.
  - if cow: `dax_iomap_copy_around(pos, length, PAGE_SIZE, srcmap, kaddr)`.
  - map_len → bytes; kaddr += offset; map_len -= offset; clamp to end-pos.
  - if recovery: `xfer = dax_recovery_write(dax_dev, pgoff, kaddr, map_len, iter)`.
  - else if write: `xfer = dax_copy_from_iter(dax_dev, pgoff, kaddr, map_len, iter)`.
  - else: `xfer = dax_copy_to_iter(dax_dev, pgoff, kaddr, map_len, iter)`.
  - `iomap_iter_advance(iomi, xfer)`; if `!ret ∧ xfer == 0`: ret = -EFAULT; if `xfer < map_len`: break.
- `dax_read_unlock(id)`.

REQ-15: `dax_writeback_mapping_range(mapping, dax_dev, wbc)`:
- precondition WARN+EIO: `inode->i_blkbits == PAGE_SHIFT`.
- if `mapping_empty(mapping) ∨ wbc->sync_mode != WB_SYNC_ALL`: return 0.
- `tag_pages_for_writeback(mapping, xas.xa_index, end_index)` (DIRTY→TOWRITE).
- `xas_lock_irq`.
- `xas_for_each_marked(&xas, entry, end_index, PAGECACHE_TAG_TOWRITE)`:
  - `ret = dax_writeback_one(xas, dax_dev, mapping, entry)`.
  - if `ret < 0`: `mapping_set_error(mapping, ret)`; break.
  - if `++scanned % XA_CHECK_SCHED == 0`: `xas_pause`, unlock, `cond_resched`, relock.
- `xas_unlock_irq`.

REQ-16: `dax_writeback_one(xas, dax_dev, mapping, entry)`:
- WARN if `!xa_is_value(entry)`.
- if `dax_is_locked(entry)`:
  - `entry = get_next_unlocked_entry(xas, 0)`.
  - if `!entry ∨ !xa_is_value(entry)`: goto put_unlocked.
  - if `dax_to_pfn(old) != dax_to_pfn(entry)`: goto put_unlocked (reallocated).
  - WARN+EIO if empty/zero (no real PFN to flush).
  - if `!xas_get_mark(TOWRITE)`: goto put_unlocked (raced).
- `dax_lock_entry(xas, entry)`.
- `xas_clear_mark(TOWRITE)`; `xas_unlock_irq`.
- pfn = `dax_to_pfn(entry)`; count = `1 << dax_entry_order(entry)`; index = `xa_index & ~(count-1)`; end = `index+count-1`.
- `i_mmap_lock_read(mapping)`; `vma_interval_tree_foreach(vma, &i_mmap, index, end)`: `pfn_mkclean_range(pfn, count, index, vma)`; `cond_resched`. `i_mmap_unlock_read`.
- `dax_flush(dax_dev, page_address(pfn_to_page(pfn)), count * PAGE_SIZE)`.
- `xas_reset`; `xas_lock_irq`; `xas_store(xas, entry)`; `xas_clear_mark(DIRTY)`; `dax_wake_entry(WAKE_NEXT)`.

REQ-17: MMU-notifier integration: write-protection of DAX mappings during writeback uses `pfn_mkclean_range(pfn, count, index, vma)` which, in turn, invokes `mmu_notifier_invalidate_range_start/_end` brackets around the rmap walk (`mm/rmap.c`). Per-fault path: `vmf_insert_page_mkwrite` / `vmf_insert_folio_pmd` also issue MMU-notifier events for the newly-installed PFN. Per-`dax_layout_busy_page_range` issues `unmap_mapping_pages(mapping, start_idx, end_idx-start_idx+1, 0)` before walking entries, which is bracketed by MMU-notifier `invalidate_range`. Rookery must propagate the same brackets so KVM EPT shadow PTEs / IOMMU SVA TLBs see invalidations before pmem rewrite/truncate completes.

REQ-18: `dax_layout_busy_page_range(mapping, start, end)`:
- if `!dax_mapping(mapping)`: return NULL.
- if `end == LLONG_MAX`: end_idx = ULONG_MAX. else: `end_idx = end >> PAGE_SHIFT`.
- `unmap_mapping_pages(mapping, start_idx, end_idx-start_idx+1, 0)`.
- `xas_lock_irq`.
- `xas_for_each(&xas, entry, end_idx)`:
  - WARN if `!xa_is_value(entry)`; continue.
  - `entry = wait_entry_unlocked_exclusive(xas, entry)`.
  - if entry: `page = dax_busy_page(entry)` (returns folio's `page` iff `ref_count - mapcount > 0`, i.e. there is a `get_user_pages` pin).
  - `put_unlocked_entry(xas, entry, WAKE_NEXT)`.
  - if page: break.
  - if `++scanned % XA_CHECK_SCHED == 0`: `xas_pause`, unlock, `cond_resched`, relock.
- return page (or NULL when no pinned page).

REQ-19: `dax_break_layout(inode, start, end, cb)` (ftruncate / punch-hole drain):
- if `!dax_mapping(inode->i_mapping)`: return 0.
- do:
  - `page = dax_layout_busy_page_range(mapping, start, end)`.
  - if `!page`: break.
  - if `!cb` (NOWAIT mode): error = -ERESTARTSYS; break.
  - `error = wait_page_idle(page, cb, inode)` — sleeps interruptibly on `dax_page_is_idle(page)`, invoking `cb(inode)` (filesystem-supplied unlock-relock helper).
- while error == 0.
- if `!page`: `dax_delete_mapping_range(mapping, start, end)`.
- return error.

REQ-20: `dax_break_layout_final(inode)` (evict_inode path, uninterruptible):
- Same as above but `wait_page_idle_uninterruptible` (no cb, uses `schedule()`).
- Loop until `!page`; then `dax_delete_mapping_range(0, LLONG_MAX)`.

REQ-21: `dax_finish_sync_fault(vmf, order, pfn)` (MAP_SYNC):
- start = `pgoff << PAGE_SHIFT`; len = `PAGE_SIZE << order`.
- `err = vfs_fsync_range(vma->vm_file, start, start+len-1, 1)`.
- if err: return SIGBUS.
- return `dax_insert_pfn_mkwrite(vmf, pfn, order)` which:
  - `XA_STATE_ORDER(xas, mapping->i_pages, vmf->pgoff, order)`.
  - `xas_lock_irq`; `entry = get_next_unlocked_entry(xas, order)`.
  - if `!entry ∨ dax_is_conflict ∨ (order==0 ∧ !dax_is_pte_entry)`: NOPAGE.
  - `xas_set_mark(DIRTY)`; `dax_lock_entry(xas, entry)`; `xas_unlock_irq`.
  - folio = `pfn_folio(pfn)`; `folio_ref_inc`.
  - if order==0: `vmf_insert_page_mkwrite(vmf, &folio->page, true)`.
  - else if `order == PMD_ORDER`: `vmf_insert_folio_pmd(vmf, folio, FAULT_FLAG_WRITE)`.
  - else: FALLBACK.
  - `folio_put`; `dax_unlock_entry(xas, entry)`.

REQ-22: `dax_zero_range(inode, pos, len, did_zero, ops)`:
- iter = `{ inode, pos, len, flags = IOMAP_DAX | IOMAP_ZERO }`.
- Loop `iomap_iter > 0`: `iter.status = dax_zero_iter(&iter, did_zero)`.

REQ-23: `dax_zero_iter` (per-iter zero):
- if `srcmap.type == HOLE ∨ UNWRITTEN`: already zeroed; advance.
- if `iomap.flags & IOMAP_F_SHARED`: `invalidate_inode_pages2_range(mapping, ...)`.
- do { offset = pos & (PAGE_SIZE-1); pgoff = `dax_iomap_pgoff`; if PAGE-aligned ∧ length == PAGE_SIZE: `dax_zero_page_range(dax_dev, pgoff, 1)`. else `dax_memzero(iter, pos, length)`. advance. } while length > 0.
- if did_zero: `*did_zero = true`.

REQ-24: `dax_file_unshare(inode, pos, len, ops)` (reflink unshare path):
- iter = `{ inode, pos, flags = IOMAP_WRITE | IOMAP_UNSHARE | IOMAP_DAX }`.
- size = `i_size_read(inode)`; if `pos < 0 ∨ pos >= size`: return 0.
- iter.len = `min(len, size - pos)`.
- Loop `iomap_iter > 0`: `iter.status = dax_unshare_iter(&iter)` which:
  - if `!iomap_want_unshare_iter(iter)`: `iomap_iter_advance_full(iter)`.
  - pad copy_pos / copy_len to PAGE boundaries.
  - `invalidate_inode_pages2_range`.
  - `dax_read_lock`; `dax_iomap_direct_access(iomap, &daddr)`, `dax_iomap_direct_access(srcmap, &saddr)`; `copy_mc_to_kernel(daddr, saddr, copy_len)` (MC = machine-check-safe).
  - `dax_read_unlock`; advance.

REQ-25: `dax_lock_folio(folio)` / `dax_unlock_folio(folio, cookie)` (memory-failure path, reverse mapping):
- `rcu_read_lock`.
- Loop:
  - mapping = `READ_ONCE(folio->mapping)`.
  - if `!mapping ∨ !dax_mapping(mapping)`: entry = NULL; break.
  - entry = `(void *)~0UL`.
  - if `S_ISCHR(mapping->host->i_mode)`: break (device-dax: pgmap pin suffices).
  - `xas.xa = &mapping->i_pages`; `xas_lock_irq`.
  - if `mapping != folio->mapping`: unlock; continue.
  - `xas_set(xas, folio->index)`; `entry = xas_load(xas)`.
  - if locked: `rcu_read_unlock`; `wait_entry_unlocked(xas, entry)`; `rcu_read_lock`; continue.
  - `dax_lock_entry(xas, entry)`; `xas_unlock_irq`; break.
- `rcu_read_unlock`; return `(dax_entry_t)entry`.

REQ-26: `dax_associate_entry(entry, mapping, vma, addr, shared)`:
- if zero/empty: return.
- size = `dax_entry_size(entry)`; folio = `dax_to_folio(entry)`; index = `linear_page_index(vma, addr & ~(size-1))`.
- if shared ∧ (folio->mapping ∨ `dax_folio_is_shared(folio)`):
  - if folio->mapping: `dax_folio_make_shared(folio)` (clear mapping, set share=1).
  - `folio->share++`.
- else: WARN if folio->mapping; `dax_folio_init(entry)` (`prep_compound_page` if order>0); folio->mapping = mapping; folio->index = index.

REQ-27: `dax_dedupe_file_range_compare`: iomap-iterates src and dst; `dax_range_compare_iter` → `memcmp(saddr, daddr, len)`; sets *same; advance.

REQ-28: `dax_remap_file_range_prep(file_in, in_off, file_out, out_off, len, flags, ops)` ⟹ `__generic_remap_file_range_prep(...)` (per-FS).

REQ-29: Static configuration:
- `DAX_WAIT_TABLE_BITS = 12`; `DAX_WAIT_TABLE_ENTRIES = 4096`.
- `PG_PMD_COLOUR = (PMD_SIZE >> PAGE_SHIFT) - 1`.
- `PG_PMD_NR = PMD_SIZE >> PAGE_SHIFT`.
- `DAX_SHIFT = 4`.

## Acceptance Criteria

- [ ] AC-1: `dax_iomap_rw` write covers `[pos, pos+len)` byte-exact in pmem (`dax_copy_from_iter`) and updates `iocb->ki_pos`.
- [ ] AC-2: `dax_iomap_rw` read of an unmapped/UNWRITTEN region zero-fills via `iov_iter_zero` (no DAX entry created).
- [ ] AC-3: PTE fault on a hole installs a `DAX_ZERO_PAGE` entry whose PFN equals `zero_pfn(addr)` and PTE is read-only.
- [ ] AC-4: Write fault on the same hole upgrades to `IOMAP_F_NEW` allocation; the prior zero PTE is unmapped via `unmap_mapping_pages(..., 1, false)` before the real PFN is installed.
- [ ] AC-5: PMD fault where `(pgoff & PG_PMD_COLOUR) != ((addr>>PAGE_SHIFT) & PG_PMD_COLOUR)` returns `VM_FAULT_FALLBACK`.
- [ ] AC-6: PMD fault on hole ⟹ `DAX_PMD | DAX_ZERO_PAGE` entry installed; `vmf_insert_folio_pmd` returns `VM_FAULT_NOPAGE`.
- [ ] AC-7: PMD entry encountered during PTE write fault on zero/empty PMD is downgraded: zero PMD ⟹ `unmap_mapping_pages(..., PG_PMD_NR, false)` issued; xa entry cleared.
- [ ] AC-8: Concurrent PTE faulters on the same pgoff serialize via `DAX_LOCKED` ⟹ the second sleeps in `get_next_unlocked_entry` on the hash-bucket wait-queue.
- [ ] AC-9: `dax_writeback_mapping_range` walks `TOWRITE` entries, calls `pfn_mkclean_range` on each VMA mapping the PFN, `dax_flush` issued, then `DIRTY` cleared.
- [ ] AC-10: `MAP_SYNC` write fault returns `VM_FAULT_NEEDDSYNC` + `*pfnp` set; caller invokes `dax_finish_sync_fault` which fsyncs then inserts the PFN-mkwrite PTE/PMD.
- [ ] AC-11: `dax_break_layout` sleeps interruptibly in `wait_page_idle` while a `get_user_pages` pin exists; returns 0 once pin drained then deletes the entries.
- [ ] AC-12: `dax_break_layout(_, _, _, NULL)` (NOWAIT) returns `-ERESTARTSYS` immediately on first busy page.
- [ ] AC-13: `dax_delete_mapping_entry` on an absent entry triggers `WARN_ON_ONCE` (caller invariant) but does not corrupt state.
- [ ] AC-14: `dax_layout_busy_page_range` issues `unmap_mapping_pages` first; returns the first folio whose `folio_ref_count - folio_mapcount > 0`.
- [ ] AC-15: `IOMAP_F_SHARED` write fault triggers `dax_iomap_copy_around` to copy unaligned head/tail of the page (or zero if srcmap is HOLE/UNWRITTEN), then installs the new entry.
- [ ] AC-16: `dax_file_unshare` over a fully-shared range issues machine-check-safe copies (`copy_mc_to_kernel`); failure returns `-EIO`.
- [ ] AC-17: `dax_lock_folio` against a device-dax inode (`S_ISCHR`) returns `~0UL` without acquiring the xa lock.
- [ ] AC-18: PMD entry's wait-bucket hash matches PTE-aligned-to-PMD-start wait-bucket (PMD colour masking) ⟹ all faulters within one PMD serialize on the same waitqueue.

## Architecture

```
struct DaxEntry(u64);                // xa-value-encoded
  flags = bits[0..4]                 // DAX_LOCKED | DAX_PMD | DAX_ZERO_PAGE | DAX_EMPTY
  pfn   = bits[4..]                  // shifted pmem PFN

const DAX_LOCKED:     u64 = 1 << 0;
const DAX_PMD:        u64 = 1 << 1;
const DAX_ZERO_PAGE:  u64 = 1 << 2;
const DAX_EMPTY:      u64 = 1 << 3;
const DAX_SHIFT:      u32 = 4;

const DAX_WAIT_TABLE_BITS:    u32 = 12;
const DAX_WAIT_TABLE_ENTRIES: usize = 1 << 12;

struct Dax {
  wait_table: [WaitQueueHead; DAX_WAIT_TABLE_ENTRIES],
}

trait DaxOps {                       // implemented by pmem driver
  fn direct_access(&self, pgoff: Pgoff, nr: usize, mode: DaxAccessMode,
                   kaddr: *mut *mut u8, pfn: *mut Pfn) -> isize;
  fn copy_from_iter(&self, pgoff: Pgoff, addr: &mut [u8], iter: &mut IovIter) -> usize;
  fn copy_to_iter(&self, pgoff: Pgoff, addr: &[u8], iter: &mut IovIter) -> usize;
  fn recovery_write(&self, pgoff: Pgoff, addr: &mut [u8], iter: &mut IovIter) -> usize;
  fn zero_page_range(&self, pgoff: Pgoff, nr: usize) -> i32;
  fn flush(&self, addr: *const u8, size: usize);
}
```

`Dax::iomap_rw(iocb, iter, ops) -> isize`:
1. iomi = `IomapIter::new(iocb.file.mapping.host, iocb.pos, iov_iter_count(iter), IOMAP_DAX)`.
2. WARN if `iocb.flags & IOCB_ATOMIC`; return -EIO.
3. if write: assert i_rwsem held write; flags |= IOMAP_WRITE.
4. else if !sb_rdonly: assert i_rwsem held.
5. if NOWAIT: flags |= IOMAP_NOWAIT.
6. while `iomap_iter(&iomi, ops) > 0`: `iomi.status = Dax::iomap_iter(&iomi, iter)`.
7. done = iomi.pos - iocb.pos; iocb.pos = iomi.pos; return done ?: ret.

`Dax::iomap_iter(iomi, iter) -> i32`:
1. write = iter is WRITE; cow = write ∧ (iomap.flags & IOMAP_F_SHARED).
2. if !write:
   - end = min(end, i_size_read).
   - if pos >= end: return 0.
   - if iomap.type ∈ {HOLE, UNWRITTEN}: `iov_iter_zero(min(length, end-pos), iter)`; advance.
3. WARN+EIO if iomap.type != MAPPED ∧ !IOMAP_F_SHARED.
4. if IOMAP_F_NEW ∨ cow:
   - if cow: `__dax_clear_dirty_range`.
   - `invalidate_inode_pages2_range`.
5. id = `dax_read_lock`.
6. while pos < end:
   - pgoff = `dax_iomap_pgoff(iomap, pos)`.
   - if fatal_signal_pending: -EINTR.
   - map_len = `ops.direct_access(pgoff, PHYS_PFN(size), DAX_ACCESS, &kaddr, NULL)`.
   - if -EHWPOISON ∧ write: retry with DAX_RECOVERY_WRITE; recovery = true.
   - if cow: `Dax::iomap_copy_around(pos, length, PAGE_SIZE, srcmap, kaddr)`.
   - xfer = recovery ? `ops.recovery_write` : write ? `ops.copy_from_iter` : `ops.copy_to_iter`.
   - advance; break if xfer < map_len.
7. `dax_read_unlock(id)`.

`Dax::iomap_fault(vmf, order, pfnp, errp, ops) -> VmFault`:
1. match order:
   - 0 ⟹ `Dax::iomap_pte_fault(vmf, pfnp, errp, ops)`.
   - PMD_ORDER ⟹ `Dax::iomap_pmd_fault(vmf, pfnp, ops)`.
   - else ⟹ `VM_FAULT_FALLBACK`.

`Dax::iomap_pte_fault(vmf, pfnp, errp, ops) -> VmFault`:
1. xas = `XaState(mapping.i_pages, vmf.pgoff)`.
2. iter = `{ inode, pos = pgoff<<PAGE_SHIFT, len = PAGE_SIZE, flags = IOMAP_DAX|IOMAP_FAULT }`.
3. if iter.pos >= i_size_read: ret = SIGBUS; goto out.
4. if (vmf.flags & WRITE) ∧ !cow_page: iter.flags |= IOMAP_WRITE.
5. entry = `Dax::grab_mapping_entry(&xas, mapping, 0)`.
6. if `xa_is_internal(entry)`: ret = xa_to_internal(entry); goto out.
7. if `pmd_trans_huge(*pmd)`: ret = NOPAGE; goto unlock_entry (racing PMD).
8. while `iomap_iter > 0`:
   - if iomap_length < PAGE_SIZE: status = -EIO; continue.
   - ret = `Dax::fault_iter(vmf, &iter, pfnp, &xas, &entry, false)`.
   - if ret != SIGBUS ∧ IOMAP_F_NEW: vm-event PGMAJFAULT; ret |= VM_FAULT_MAJOR.
   - if !(ret & VM_FAULT_ERROR): advance PAGE_SIZE.
9. if errp: *errp = error.
10. if !ret ∧ error: ret = `dax_fault_return(error)`.
11. unlock_entry: `dax_unlock_entry(&xas, entry)`.

`Dax::iomap_pmd_fault(vmf, pfnp, ops) -> VmFault`:  (under CONFIG_FS_DAX_PMD)
1. xas = `XaStateOrder(mapping.i_pages, vmf.pgoff, PMD_ORDER)`.
2. max_pgoff = `div_round_up(i_size, PAGE_SIZE)`.
3. if xas.xa_index >= max_pgoff: SIGBUS; goto out.
4. if `Dax::fault_check_fallback(vmf, &xas, max_pgoff)`: goto fallback.
5. entry = `Dax::grab_mapping_entry(&xas, mapping, PMD_ORDER)`.
6. if `xa_is_internal(entry)`: ret = xa_to_internal; goto fallback.
7. if !pmd_none ∧ !pmd_trans_huge: ret = 0; goto unlock_entry (racing PTE).
8. iter.pos = xas.xa_index << PAGE_SHIFT.
9. while `iomap_iter > 0`:
   - if iomap_length < PMD_SIZE: continue (loop break).
   - ret = `Dax::fault_iter(vmf, &iter, pfnp, &xas, &entry, true)`.
   - if ret != FALLBACK: advance PMD_SIZE.
10. unlock_entry: `dax_unlock_entry(&xas, entry)`.
11. fallback: if ret == FALLBACK: `split_huge_pmd(vma, pmd, addr)`; vm-event THP_FAULT_FALLBACK.

`Dax::fault_iter(vmf, iter, pfnp, xas, entry, pmd) -> VmFault`:
1. size = pmd ? PMD_SIZE : PAGE_SIZE; pos = xas.xa_index << PAGE_SHIFT; write = IOMAP_WRITE.
2. entry_flags = pmd ? DAX_PMD : 0.
3. if !pmd ∧ vmf.cow_page: return `Dax::fault_cow_page(vmf, iter)`.
4. if !write ∧ (UNWRITTEN ∨ HOLE):
   - if !pmd: `Dax::load_hole(xas, vmf, iter, entry)`.
   - else: `Dax::pmd_load_hole(xas, vmf, iter, entry)`.
5. if iomap.type != MAPPED ∧ !IOMAP_F_SHARED: WARN; return pmd ? FALLBACK : SIGBUS.
6. err = `Dax::iomap_direct_access(iomap, pos, size, &kaddr, &pfn)`; if err: pmd ? FALLBACK : `dax_fault_return(err)`.
7. *entry = `Dax::insert_entry(xas, vmf, iter, *entry, pfn, entry_flags)`.
8. if write ∧ IOMAP_F_SHARED: `Dax::iomap_copy_around(pos, size, size, srcmap, kaddr)`.
9. folio = `dax_to_folio(*entry)`.
10. if `Dax::fault_is_synchronous(iter, vma)`: return `Dax::fault_synchronous_pfnp(pfnp, pfn)` = stores pfn; returns NEEDDSYNC.
11. folio_ref_inc(folio).
12. if pmd: ret = `vmf_insert_folio_pmd(vmf, pfn_folio(pfn), write)`.
13. else: ret = `vmf_insert_page_mkwrite(vmf, pfn_to_page(pfn), write)`.
14. folio_put(folio); return ret.

`Dax::grab_mapping_entry(xas, mapping, order) -> *Entry`:
1. retry:
2. pmd_downgrade = false; `xas_lock_irq`.
3. entry = `Dax::get_next_unlocked_entry(xas, order)`.
4. if entry:
   - if `dax_is_conflict(entry)`: goto fallback.
   - if !xa_is_value(entry): `xas_set_err -EIO`; goto out_unlock.
   - if order == 0 ∧ pmd ∧ (zero ∨ empty): pmd_downgrade = true.
5. if pmd_downgrade:
   - `dax_lock_entry(xas, entry)` (pin while we drop lock).
   - if zero: unlock; `unmap_mapping_pages(mapping, idx & ~PG_PMD_COLOUR, PG_PMD_NR, false)`; reset; relock.
   - `dax_disassociate_entry`; `xas_store(NULL)`; `dax_wake_entry(WAKE_ALL)`; mapping.nrpages -= PG_PMD_NR; entry = NULL; xas_set(idx).
6. if entry: `dax_lock_entry(xas, entry)`.
7. else: flags = DAX_EMPTY | (DAX_PMD if order>0); entry = `dax_make_entry(0, flags)`; `dax_lock_entry`; mapping.nrpages += (1<<order).
8. out_unlock: `xas_unlock_irq`.
9. if `xas_nomem(xas, gfp & ~__GFP_HIGHMEM)`: goto retry.
10. if `xas.xa_node == XA_ERROR(-ENOMEM)`: return `xa_mk_internal(VM_FAULT_OOM)`.
11. if `xas_error(xas)`: return `xa_mk_internal(VM_FAULT_SIGBUS)`.
12. return entry.
13. fallback: `xas_unlock_irq`; return `xa_mk_internal(VM_FAULT_FALLBACK)`.

`Dax::writeback_mapping_range(mapping, dax_dev, wbc) -> i32`:
1. WARN+EIO if `inode.i_blkbits != PAGE_SHIFT`.
2. if `mapping_empty ∨ wbc.sync_mode != WB_SYNC_ALL`: return 0.
3. `tag_pages_for_writeback(mapping, start, end)`.
4. `xas_lock_irq`.
5. `xas_for_each_marked(&xas, entry, end_index, PAGECACHE_TAG_TOWRITE)`:
   - `Dax::writeback_one(&xas, dax_dev, mapping, entry)`.
   - if rc < 0: `mapping_set_error(mapping, rc)`; break.
   - if `++scanned % XA_CHECK_SCHED == 0`: pause / unlock / cond_resched / relock.
6. `xas_unlock_irq`.

`Dax::writeback_one(xas, dax_dev, mapping, entry) -> i32`:
1. WARN+EIO if !xa_is_value.
2. if `dax_is_locked(entry)`:
   - entry = `Dax::get_next_unlocked_entry(xas, 0)`.
   - if !entry ∨ !xa_is_value: goto put_unlocked.
   - if pfn(old) != pfn(new): goto put_unlocked.
   - WARN+EIO if empty/zero.
   - if !xas_get_mark(TOWRITE): goto put_unlocked.
3. `dax_lock_entry(xas, entry)`; `xas_clear_mark(TOWRITE)`; `xas_unlock_irq`.
4. pfn / count / index / end.
5. `i_mmap_lock_read`; `vma_interval_tree_foreach`: `pfn_mkclean_range(pfn, count, index, vma)`; `cond_resched`. unlock.
6. `ops.flush(dax_dev, page_address(pfn_to_page(pfn)), count * PAGE_SIZE)`.
7. `xas_reset`; `xas_lock_irq`; `xas_store(entry)`; `xas_clear_mark(DIRTY)`; `dax_wake_entry(WAKE_NEXT)`.

`Dax::break_layout(inode, start, end, cb) -> i32`:
1. if !dax_mapping: return 0.
2. loop:
   - page = `Dax::layout_busy_page_range(mapping, start, end)`.
   - if !page: break.
   - if !cb: return -ERESTARTSYS.
   - error = `wait_page_idle(page, cb, inode)`; if error != 0: break.
3. if !page: `Dax::delete_mapping_range(mapping, start, end)`.

`Dax::layout_busy_page_range(mapping, start, end) -> Option<*Page>`:
1. if !dax_mapping: None.
2. end_idx = if end == LLONG_MAX { ULONG_MAX } else { end >> PAGE_SHIFT }.
3. `unmap_mapping_pages(mapping, start_idx, end_idx - start_idx + 1, 0)`.
4. `xas_lock_irq`; `xas_for_each(&xas, entry, end_idx)`:
   - WARN if !xa_is_value.
   - entry = `wait_entry_unlocked_exclusive(xas, entry)`.
   - if entry: page = `Dax::busy_page(entry)`.
   - `put_unlocked_entry(xas, entry, WAKE_NEXT)`.
   - if page: break.
   - cond_resched periodically.
5. return page.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `entry_flags_disjoint_with_pfn` | INVARIANT | per-DaxEntry: bits[0..4] never overlap PFN bits (shift = 4). |
| `lock_held_means_xa_value` | INVARIANT | per-DaxEntry: DAX_LOCKED ⟹ xa_is_value (never internal/page). |
| `pmd_color_masked_in_waitqueue` | INVARIANT | per-dax_entry_waitqueue: PMD entry index &= ~PG_PMD_COLOUR before hash. |
| `wait_match_pred` | INVARIANT | per-wake_exceptional_entry: (xa, entry_start) match required. |
| `grab_pmd_downgrade_only_zero_or_empty` | INVARIANT | per-grab_mapping_entry: pmd_downgrade only when order==0 ∧ PMD ∧ (zero ∨ empty). |
| `writeback_skips_empty_and_zero` | INVARIANT | per-writeback_one: WARN+EIO if empty/zero entry encountered post-lock. |
| `pmd_color_match_required` | INVARIANT | per-fault_check_fallback: PG_PMD_COLOUR identity required for PMD path. |
| `dax_iomap_rw_atomic_rejected` | INVARIANT | per-iomap_rw: IOCB_ATOMIC ⟹ -EIO. |
| `nowait_break_layout` | INVARIANT | per-break_layout: cb == NULL ⟹ first busy page returns -ERESTARTSYS. |
| `device_dax_no_xa_lock` | INVARIANT | per-lock_folio: S_ISCHR ⟹ returns (void*)~0UL without xa_lock acquisition. |
| `iomap_iter_advance_monotonic` | INVARIANT | per-iomap_iter: pos only advances; never decreases. |
| `mapping_nrpages_balanced` | INVARIANT | per-grab/delete: nrpages += (1<<order) on insert, -= on remove. |
| `mmu_notifier_brackets_writeback` | INVARIANT | per-pfn_mkclean_range: mmu_notifier_invalidate_range_start/_end pair across the rmap walk. |

### Layer 2: TLA+

`fs/dax-entry-lock.tla`:
- Vars: per-entry-state ∈ {Absent, Empty-Unlocked, Empty-Locked, PTE-Unlocked, PTE-Locked, PMD-Unlocked, PMD-Locked, Zero-PTE, Zero-PMD}; per-faulter ∈ {Sleeping(idx), Running, Done}.
- Actions: grab_mapping_entry, dax_lock_entry, dax_unlock_entry, dax_wake_entry, get_next_unlocked_entry-wait, pmd_downgrade.
- Properties:
  - `mutual_exclusion_per_index` — at most one faulter holds DAX_LOCKED at any pgoff.
  - `pmd_downgrade_only_on_zero_or_empty` — pmd_downgrade transitions guarded.
  - `every_waiter_eventually_runs` — bounded wait given finite faulters.
  - `wakeup_matches_key` — wake_exceptional_entry_func only consumes wake on (xa, entry_start) match.

`fs/dax-fault-rw.tla`:
- Actions: dax_iomap_rw / dax_iomap_fault interleavings.
- Properties:
  - `read_of_hole_returns_zeros` — read over HOLE ⟹ user buf zero-filled.
  - `write_after_invalidate_visible` — write followed by read sees new bytes.
  - `map_sync_durable` — MAP_SYNC write fault ⟹ fsync precedes user PTE install.
  - `cow_iomap_f_shared_correctness` — cow path copies before mutate; srcmap unchanged.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Dax::iomap_rw` post: `iocb.pos = iomi.pos`; return = done | `Dax::iomap_rw` |
| `Dax::iomap_iter` post: pos ≤ end; ret ∈ {0, -EFAULT, -EIO, -EINTR, ...} | `Dax::iomap_iter` |
| `Dax::iomap_fault` post: order ∈ {0, PMD_ORDER} ∨ FALLBACK | `Dax::iomap_fault` |
| `Dax::grab_mapping_entry` post: returned entry locked ∨ internal (FALLBACK / SIGBUS / OOM) | `Dax::grab_mapping_entry` |
| `Dax::fault_iter` post: pfn installed ⟹ entry has DAX_LOCKED ∧ pfn matches | `Dax::fault_iter` |
| `Dax::writeback_one` post: pfn_mkclean_range called for every VMA before `dax_flush` | `Dax::writeback_one` |
| `Dax::break_layout` post: `!page` ⟹ delete_mapping_range called | `Dax::break_layout` |
| `Dax::layout_busy_page_range` pre: unmap_mapping_pages issued ≥ once before scan | `Dax::layout_busy_page_range` |
| `Dax::insert_entry` post: `xas_set_mark(DIRTY)` iff dirty ∧ !MAP_SYNC | `Dax::insert_entry` |

### Layer 4: Verus/Creusot functional

`Per-pmem-block` semantic equivalence relation between upstream Linux DAX entries and Rookery `Dax` per-`Documentation/filesystems/dax.rst` + `Documentation/admin-guide/mm/transhuge.rst`:
- `dax_iomap_rw(iocb, iter, ops)` ↔ user buf observation/state matches pmem byte semantics under iomap.
- `dax_iomap_fault(vmf, order, ...)` ↔ resulting PTE/PMD references the correct pmem PFN with correct write-protection.
- `dax_writeback_mapping_range(mapping, dax_dev, wbc)` ↔ all PTEs/PMDs of dirty entries write-protected and dax_flush issued before clearing DIRTY.
- `dax_break_layout(inode, start, end, cb)` ↔ no pinned pmem page remains in the range when the function returns 0.

## Hardening

(Inherits row-1 features from `fs/00-overview.md` § Hardening.)

DAX reinforcement:

- **Per-entry bit-lock (DAX_LOCKED) under xa_lock** — defense against per-fault-fault entry-corruption.
- **Per-PMD color masking in wait-bucket hash** — defense against per-PMD-stale-wait-bucket / lost-wakeup.
- **Per-wake match-pred (xa, entry_start)** — defense against per-cross-mapping spurious wake.
- **Per-WB_SYNC_ALL-only writeback** — defense against per-spurious-flush-storm.
- **Per-pfn_mkclean_range mmu-notifier brackets** — defense against per-stale-IOMMU / -KVM-EPT under writeback.
- **Per-`dax_break_layout` GUP-pin drain before truncate** — defense against per-DMA-corrupting-truncate.
- **Per-NOWAIT break_layout (cb == NULL) ⟹ -ERESTARTSYS** — defense against per-blocking on AIO truncate.
- **Per-WARN+EIO on writeback of empty/zero entry** — defense against per-corruption (no real PFN to flush).
- **Per-IOCB_ATOMIC rejection (-EIO)** — defense against per-non-atomic-pmem-store-tearing.
- **Per-EHWPOISON retry with DAX_RECOVERY_WRITE** — defense against per-MCE-on-pmem read-poison loop.
- **Per-`copy_mc_to_kernel` for CoW / unshare** — defense against per-MCE during page-copy.
- **Per-`unmap_mapping_pages` before downgrading PMD-zero to PTE** — defense against per-stale-PMD in user pgtables.
- **Per-PMD fault color-match + VMA-bounds + EOF check** — defense against per-PMD-misalign / per-PMD-past-EOF.
- **Per-`dax_layout_busy_page_range` `unmap_mapping_pages` precondition** — defense against per-races against new GUP after scan.
- **Per-device-dax (`S_ISCHR`) short-circuit in `dax_lock_folio`** — defense against per-spurious-xa_lock on a dax_dev without page-cache.
- **Per-`folio.share` refcount + `dax_folio_make_shared`** — defense against per-CoW-reflink-leak / per-double-free in reverse-map.

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — bounded user-buffer copy on `dax_iomap_iter` / `copy_mc_to_user` paths so pmem MCE-injected bytes cannot overrun a slab object boundary.
- **PAX_KERNEXEC** — W^X for any executable mapping; DAX-backed text pages are mapped `PROT_EXEC` only when the underlying FS_VERITY descriptor matches.
- **PAX_RANDKSTACK** — per-syscall kernel-stack randomization across `mmap(MAP_SYNC)` / `fsync` so DAX fault paths leak no stack layout.
- **PAX_REFCOUNT** — saturating refcount on `struct dax_device`, on each DAX `xa_entry` type tag (`DAX_ZERO_PAGE`/`DAX_PMD`/`DAX_LOCKED`), and on the per-folio `folio.share` counter so reflink-CoW cannot wrap.
- **PAX_MEMORY_SANITIZE** — zero-on-free for any DAX page returning to the freelist via `dax_zero_iter`, including PMD-sized teardown.
- **PAX_UDEREF** — SMAP/SMEP strict user-pointer access; `dax_iomap_fault` rejects faults whose `vmf->address` is not user-space.
- **PAX_RAP / kCFI** — indirect-call signature enforcement on `struct dax_operations` (`direct_access`, `zero_page_range`, `recovery_write`) and on `iomap_ops` vtables so pmem driver tables cannot be hijacked.
- **GRKERNSEC_HIDESYM** — kernel pointer hiding in `/proc/<pid>/maps` and `numa_maps` for DAX-backed VMAs.
- **GRKERNSEC_DMESG** — syslog restriction on pmem-MCE recovery diagnostics, which otherwise expose PFN layout.
- **pmem-MCE poison-list isolation** — `dax_recovery_write` validates that the target PFN is on the poison list before clearing, refusing arbitrary clears.
- **MAP_SYNC validation** — `dax_iomap_fault` rejects `MAP_SYNC|MAP_SHARED_VALIDATE` on filesystems lacking `FS_DAX_SYNC` so a stale FS cannot silently downgrade to non-sync semantics.
- **DAX entry-type tag invariants** — every `xa_load` of the DAX radix is checked against `dax_is_value()` / `dax_is_locked()` / `dax_is_pmd()` mutual exclusion before deref.
- **`dax_lock_folio` device-dax short-circuit** — character-device DAX never enters `xa_lock` paths shared with FS-DAX, preventing a confused-deputy on shared lock state.
- **CAP_SYS_RAWIO gate on `/dev/daxN.M`** — character-device DAX open requires CAP_SYS_RAWIO so unprivileged code cannot directly map raw pmem.

Per-doc rationale: DAX bypasses the page cache and hands user mappings directly to persistent memory PFNs, so every classical "page-cache ate the bad bit" defense is absent; PaX/grsec reinforcement matters more here than in any other FS because a single mis-tagged radix entry, a single missed `folio.share` increment, or a single unvalidated `MAP_SYNC` flag becomes a permanent corruption written to media.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- drivers/dax/*  (covered separately if pmem-backend Tier-3 expanded)
- drivers/nvdimm/* (covered separately)
- `dax_direct_access` / `dax_copy_*` / `dax_flush` driver vtable implementation (covered in `drivers/dax/super.md` Tier-3 if expanded)
- ext4 / xfs / fuse DAX wiring (covered in respective FS Tier-3s)
- iomap framework (`fs/iomap/*`) (covered separately if expanded)
- huge-zero-folio mechanism (`mm/huge_memory.c`) (covered in mm Tier-3)
- struct-page `ZONE_DEVICE` + `dev_pagemap` lifetimes (covered in `mm/memory.c` Tier-3)
- mm-rmap `pfn_mkclean_range` internals (covered in `mm/rmap.md` Tier-3)
- Implementation code
