# Tier-3: mm/mremap.c — mremap(2): resize / move VMA

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: mm/00-overview.md
upstream-paths:
  - mm/mremap.c (~2056 lines)
  - include/uapi/linux/mman.h (MREMAP_MAYMOVE, MREMAP_FIXED, MREMAP_DONTUNMAP)
  - include/linux/mm.h (move_page_tables, struct pagetable_move_control)
  - Documentation/admin-guide/mm/index.rst
-->

## Summary

`mremap(2)` resizes — and optionally moves — an existing VMA. Per-call inputs: `addr`, `old_len`, `new_len`, `flags ∈ { 0, MREMAP_MAYMOVE, MREMAP_FIXED, MREMAP_DONTUNMAP }`, and (for FIXED/DONTUNMAP) `new_addr`. Per-`MREMAP_NO_RESIZE` shrink/expand-in-place: when the VMA can grow into the next gap or shrink into a smaller suffix, no copy is performed; only `vma_merge_extend()` / `do_vmi_munmap()` runs. Per-`MREMAP_MAYMOVE` move: `copy_vma()` allocates a new VMA at `get_unmapped_area()`'s chosen address, `move_page_tables()` walks PUD/PMD/PTE entries and uses `move_normal_pud` / `move_normal_pmd` to swap entire page-table levels by pointer when the alignment permits (fast path), falling back to per-PTE batched `move_ptes` (slow path); on success, `unmap_source_vma()` removes the original. Per-`MREMAP_FIXED` user-supplied `new_addr`: behaves like `MAP_FIXED` (overlap unmapped first via `do_munmap`). Per-`MREMAP_DONTUNMAP` (`MREMAP_FIXED|MAYMOVE`-implying): page tables are *moved*, but the source VMA is *kept* with VM_LOCKED cleared, used by userfaultfd post-copy and CRIU. Per-hugetlb VMA: `move_hugetlb_page_tables` dispatched instead of the generic walker. Per-anon-vma: `copy_vma()` re-uses the source's `anon_vma` when address-compatible, avoiding a fresh `anon_vma_chain` allocation. Per-`vma_start_write` / mmu_notifier bracketing keeps page-table moves atomic with respect to GUP-fast / KVM secondary MMU.

This Tier-3 covers `mm/mremap.c` (~2056 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `SYSCALL_DEFINE5(mremap, ...)` | per-syscall entry | `Mremap::sys_mremap` |
| `do_mremap()` | per-call dispatcher | `Mremap::do_mremap` |
| `check_mremap_params()` | per-call validation | `Mremap::check_params` |
| `vrm_remap_type()` | per-call NO_RESIZE/SHRINK/EXPAND | `Mremap::remap_type` |
| `vrm_implies_new_addr()` | per-call FIXED/DONTUNMAP test | `Mremap::implies_new_addr` |
| `vrm_overlaps()` | per-call src/dst overlap | `Mremap::overlaps` |
| `vrm_set_new_addr()` | per-call get_unmapped_area | `Mremap::set_new_addr` |
| `vrm_calc_charge()` / `vrm_uncharge()` | per-call accounting | `Mremap::calc_charge` / `uncharge` |
| `vrm_stat_account()` | per-call vm_stat update | `Mremap::stat_account` |
| `check_map_count_against_split()` / `_early()` | per-call map_count guard | `Mremap::check_map_count` |
| `mremap_to()` | per-call new-address path | `Mremap::mremap_to` |
| `mremap_at()` | per-call in-place path | `Mremap::mremap_at` |
| `remap_move()` | per-call multi-VMA move | `Mremap::remap_move` |
| `shrink_vma()` | per-call shrink | `Mremap::shrink_vma` |
| `expand_vma_in_place()` | per-call in-place expand | `Mremap::expand_in_place` |
| `vma_expandable()` / `vrm_can_expand_in_place()` | per-call expand-feasibility | `Mremap::expandable` / `can_expand_in_place` |
| `prep_move_vma()` | per-call pre-move (map_count, may_split, ksm_madvise) | `Mremap::prep_move_vma` |
| `move_vma()` | per-call move-driver | `Mremap::move_vma` |
| `copy_vma_and_data()` | per-call VMA-copy + page-table-copy | `Mremap::copy_vma_and_data` |
| `unmap_source_vma()` | per-call src VMA unmap | `Mremap::unmap_source_vma` |
| `dontunmap_complete()` | per-call DONTUNMAP post-fixup | `Mremap::dontunmap_complete` |
| `move_page_tables()` | per-call PT walker | `Mremap::move_page_tables` |
| `move_pgt_entry()` | per-call dispatch (PMD/PUD, normal/huge) | `Mremap::move_pgt_entry` |
| `move_normal_pmd()` / `move_normal_pud()` | per-PMD/PUD pointer-swap fast | `Mremap::move_normal_pmd/pud` |
| `move_huge_pmd()` / `move_huge_pud()` | per-THP/PUD-huge | `Mremap::move_huge_pmd/pud` |
| `move_ptes()` | per-PTE batched move | `Mremap::move_ptes` |
| `mremap_folio_pte_batch()` | per-batch contiguous-folio detect | `Mremap::folio_pte_batch` |
| `move_hugetlb_page_tables()` (mm/hugetlb.c) | per-hugetlb walker | external |
| `try_realign_addr()` / `can_realign_addr()` / `can_align_down()` | per-PMD-boundary alignment | `Mremap::try_realign_addr` |
| `pmc_done` / `pmc_next` / `pmc_progress` | per-PT-iter state | `PageTableMoveControl::done/next/progress` |
| `should_take_rmap_locks()` / `take_rmap_locks` / `drop_rmap_locks` | per-PMD/PUD rmap-lock | `Mremap::rmap_locks` |
| `notify_uffd()` | per-call userfaultfd-uffd_remove | `Mremap::notify_uffd` |
| `mremap_userfaultfd_prep()` | per-call uffd new-vma register | `Mremap::uffd_prep` |
| `vrm_move_only()` | per-call (DONTUNMAP|FIXED ∧ no resize) | `Mremap::move_only` |
| `MREMAP_MAYMOVE` / `MREMAP_FIXED` / `MREMAP_DONTUNMAP` | per-call flags | `MremapFlags` bitflags |
| `struct vma_remap_struct` | per-call context | `VmaRemap` |
| `struct pagetable_move_control` | per-call PT-iter context | `PageTableMoveControl` |

## Compatibility contract

REQ-1: struct vma_remap_struct (per-call context):
- addr / old_len / new_len: user inputs.
- flags: MREMAP_MAYMOVE | MREMAP_FIXED | MREMAP_DONTUNMAP.
- new_addr: user-supplied (FIXED/DONTUNMAP) or get_unmapped_area-chosen.
- vma: current source VMA pointer.
- delta: abs(old_len - new_len).
- remap_type: MREMAP_NO_RESIZE | MREMAP_SHRINK | MREMAP_EXPAND | MREMAP_INVALID.
- charged: pages charged via security_vm_enough_memory_mm.
- mmap_locked: bool — is mmap_lock currently held.
- populate_expand: bool — must mm_populate the new tail (VM_LOCKED expand).
- vmi_needs_invalidate: bool — VMA-iterator now stale.
- uf / uf_unmap / uf_unmap_early: userfaultfd contexts.

REQ-2: `sys_mremap(addr, old_len, new_len, flags, new_addr)`:
- vrm = { .addr = untagged_addr(addr), .old_len, .new_len, .flags, .new_addr, .uf*, .remap_type = MREMAP_INVALID }.
- return do_mremap(&vrm).
- Note: addr is untagged (ARM64 MTE); new_addr is *not* untagged (kept for symmetry with mmap()).

REQ-3: `do_mremap(vrm)`:
- vrm.old_len = PAGE_ALIGN(vrm.old_len).
- vrm.new_len = PAGE_ALIGN(vrm.new_len).
- res = check_mremap_params(vrm); if res: return res.
- mmap_write_lock_killable(mm) ∨ return -EINTR.
- vrm.mmap_locked = true.
- if !check_map_count_against_split_early(): mmap_write_unlock; return -ENOMEM.
- if vrm_move_only(vrm): res = remap_move(vrm).
- else:
  - vrm.vma = vma_lookup(mm, vrm.addr).
  - res = check_prep_vma(vrm); if res: goto out.
  - res = vrm_implies_new_addr(vrm) ? mremap_to(vrm) : mremap_at(vrm).
- out: failed = IS_ERR_VALUE(res).
- if vrm.mmap_locked: mmap_write_unlock(mm).
- if !failed ∧ vrm.populate_expand: mm_populate(vrm.new_addr + vrm.old_len, vrm.delta).
- notify_uffd(vrm, failed).
- return res.

REQ-4: `check_mremap_params(vrm)`:
- if flags & ~(MREMAP_FIXED|MREMAP_MAYMOVE|MREMAP_DONTUNMAP): return -EINVAL.
- if offset_in_page(addr): return -EINVAL.
- if new_len == 0: return -EINVAL.
- if new_len > TASK_SIZE: return -EINVAL.
- if !vrm_implies_new_addr(vrm): return 0.
- if new_addr > TASK_SIZE - new_len: return -EINVAL.
- if offset_in_page(new_addr): return -EINVAL.
- if !(flags & MREMAP_MAYMOVE): return -EINVAL.
- if (flags & MREMAP_DONTUNMAP) ∧ old_len != new_len: return -EINVAL.
- if vrm_overlaps(vrm): return -EINVAL.
- return 0.

REQ-5: `mremap_at(vrm)` (no new_addr; in-place expand/shrink):
- remap_type = vrm_remap_type(vrm).
- case MREMAP_NO_RESIZE: return vrm.addr (no-op).
- case MREMAP_SHRINK: return shrink_vma(vrm, /*drop_lock=*/true).
- case MREMAP_EXPAND:
  - if vrm_can_expand_in_place(vrm): return expand_vma_in_place(vrm).
  - if !(vrm.flags & MREMAP_MAYMOVE): return -ENOMEM.
  - // Otherwise fall through to move.
  - res = vrm_set_new_addr(vrm); if res: return res.
  - return move_vma(vrm).

REQ-6: `mremap_to(vrm)` (explicit new_addr; FIXED ∨ DONTUNMAP):
- if MREMAP_FIXED:
  - res = do_munmap(mm, new_addr, new_len, uf_unmap_early).
  - vrm.vma = NULL (invalidated).
  - if res: return res.
  - vrm.vma = vma_lookup(mm, vrm.addr).
  - if !vrm.vma: return -EFAULT.
- if remap_type == MREMAP_SHRINK:
  - res = shrink_vma(vrm, /*drop_lock=*/false); if res: return res.
  - vrm.old_len = vrm.new_len.
- if MREMAP_DONTUNMAP:
  - if !may_expand_vm(mm, &vma_flags, old_len >> PAGE_SHIFT): return -ENOMEM.
- res = vrm_set_new_addr(vrm); if res: return res.
- return move_vma(vrm).

REQ-7: `move_vma(vrm)`:
- err = prep_move_vma(vrm); if err: return err.
- if !vrm_calc_charge(vrm): return -ENOMEM.
- vma_start_write(vrm.vma) — block racing faults.
- err = copy_vma_and_data(vrm, &new_vma).
- if err ∧ !new_vma: return err.
- hiwater_vm = mm.hiwater_vm.
- vrm_stat_account(vrm, vrm.new_len).
- if !err ∧ (vrm.flags & MREMAP_DONTUNMAP): dontunmap_complete(vrm, new_vma).
- else: unmap_source_vma(vrm).
- mm.hiwater_vm = hiwater_vm.
- return err ?: vrm.new_addr.

REQ-8: `copy_vma_and_data(vrm, &new_vma)`:
- internal_offset = vrm.addr - vrm.vma.vm_start; new_pgoff = vma.vm_pgoff + (offset >> PAGE_SHIFT).
- PAGETABLE_MOVE(pmc, NULL, NULL, vrm.addr, vrm.new_addr, vrm.old_len).
- new_vma = copy_vma(&vma, vrm.new_addr, vrm.new_len, new_pgoff, &pmc.need_rmap_locks).
- if !new_vma: vrm_uncharge; return -ENOMEM.
- if vma != vrm.vma: vrm.vmi_needs_invalidate = true.
- vrm.vma = vma; pmc.old = vma; pmc.new = new_vma.
- moved_len = move_page_tables(&pmc).
- if moved_len < vrm.old_len: err = -ENOMEM.
- else if vma.vm_ops.mremap: err = vma.vm_ops.mremap(new_vma).
- if err:
  - PAGETABLE_MOVE(pmc_revert, new_vma, vma, vrm.new_addr, vrm.addr, moved_len).
  - pmc_revert.need_rmap_locks = true; move_page_tables(&pmc_revert).
  - vrm.vma = new_vma; vrm.old_len = vrm.new_len; vrm.addr = vrm.new_addr.
- else: mremap_userfaultfd_prep(new_vma, vrm.uf).
- fixup_hugetlb_reservations(vma).
- return err.

REQ-9: `move_page_tables(pmc)`:
- if !pmc.len_in: return 0.
- if is_vm_hugetlb_page(pmc.old): return move_hugetlb_page_tables(pmc.old, pmc.new, pmc.old_addr, pmc.new_addr, pmc.len_in).
- try_realign_addr(pmc, PMD_MASK).
- flush_cache_range(pmc.old, pmc.old_addr, pmc.old_end).
- mmu_notifier_invalidate_range_start(MMU_NOTIFY_UNMAP, pmc.old_addr..pmc.old_end).
- while !pmc_done(pmc):
  - cond_resched.
  - try PUD-sized: if HPAGE_PUD ∧ aligned: move_pgt_entry(HPAGE_PUD); pmc_next(PUD_SIZE).
  - else if PMD-sized: if HPAGE_PMD ∧ aligned: move_pgt_entry(HPAGE_PMD); pmc_next(PMD_SIZE).
  - else if NORMAL_PUD candidate (aligned, idle target): move_pgt_entry(NORMAL_PUD); pmc_next(PUD_SIZE).
  - else if NORMAL_PMD candidate: move_pgt_entry(NORMAL_PMD); pmc_next(PMD_SIZE).
  - else: extent = PMD_SIZE - (pmc.old_addr & ~PMD_MASK); extent = min(extent, pmc.old_end - pmc.old_addr); move_ptes(pmc, old_pmd, new_pmd, extent); pmc_next(extent).
- mmu_notifier_invalidate_range_end.
- return pmc_progress(pmc).

REQ-10: `move_normal_pmd(pmc, old_pmd, new_pmd)`:
- /* Fast path: swap entire PMD pointer */
- if old_pmd_none(*old_pmd): return false (nothing to move).
- pmd = pmdp_get_and_clear(mm, old_addr, old_pmd).
- set_pmd_at(mm, new_addr, new_pmd, pmd).
- flush_tlb_range(old_vma, old_addr, old_addr + PMD_SIZE).
- return true.

REQ-11: `move_ptes(pmc, old_pmd, new_pmd, extent)`:
- /* Slow path: per-PTE batched */
- old_ptep = pte_offset_map_lock; new_ptep = pte_offset_map_nested_lock.
- for each PTE in extent:
  - batch = mremap_folio_pte_batch(vma, old_addr, old_ptep, max=extent).
  - pte_clear_flush (old) → set_pte_at (new).
  - flush_tlb_batched_pending.
- pte_unmap_unlock.

REQ-12: `unmap_source_vma(vrm)`:
- accountable_move = (vma.vm_flags & VM_ACCOUNT) ∧ !(vrm.flags & MREMAP_DONTUNMAP).
- if accountable_move: vm_flags_clear(vma, VM_ACCOUNT); store vm_start/vm_end.
- err = do_vmi_munmap(&vmi, mm, addr, len, vrm.uf_unmap, /*unlock=*/false).
- vrm.vma = NULL; vrm.vmi_needs_invalidate = true.
- if err: vm_acct_memory(len >> PAGE_SHIFT) (best-effort account fix); return.
- if accountable_move: re-set VM_ACCOUNT on remaining prefix/suffix VMAs.

REQ-13: `dontunmap_complete(vrm, new_vma)`:
- /* MREMAP_DONTUNMAP keeps source VMA */
- vm_flags_clear(vrm.vma, VM_LOCKED_MASK) — old VMA is logically un-mlock'd.
- if new_vma != vrm.vma ∧ start==old_start ∧ end==old_end: unlink_anon_vmas(vrm.vma).
- // total_vm / locked_vm untouched (no munmap).

REQ-14: `prep_move_vma(vrm)`:
- if !check_map_count_against_split(): return -ENOMEM.
- if vma.vm_ops.may_split:
  - if vm_start != old_addr: vma.vm_ops.may_split(vma, old_addr).
  - if vm_end != old_addr + old_len: vma.vm_ops.may_split(vma, old_addr + old_len).
  - propagate err.
- err = ksm_madvise(vma, old_addr, old_addr+old_len, MADV_UNMERGEABLE, &dummy).
- return err.

REQ-15: `remap_move(vrm)` (DONTUNMAP-only multi-VMA move):
- iter [addr, addr+old_len):
  - per-VMA: addr = max(vma.vm_start, start); len = min(end, vma.vm_end) - addr.
  - first VMA: no gap permitted at start ⟹ -EFAULT.
  - offset = seen_vma ? vma.vm_start - last_end : 0.
  - new_addr = target_addr + offset (preserve inter-VMA gaps).
  - multi_allowed = vma_multi_allowed(vma); else single-VMA-only or abort.
  - check_prep_vma(vrm); mremap_to(vrm); propagate err.
  - VM_WARN_ON_ONCE !vrm.mmap_locked; VM_WARN_ON_ONCE vrm.populate_expand.
  - if vrm.vmi_needs_invalidate: vma_iter_invalidate; reset flag.
  - target_addr = res_vma + vrm.new_len.
- return first-VMA result.

REQ-16: Per-MREMAP_DONTUNMAP constraints:
- Requires (MREMAP_FIXED ∨ MREMAP_MAYMOVE).
- old_len == new_len enforced.
- Source VMA retained with VM_LOCKED cleared; page tables transplanted.
- Used by userfaultfd post-copy / process migration (CRIU).

REQ-17: Per-anon_vma reuse:
- `copy_vma()` will set new_vma.anon_vma = vma.anon_vma when address ranges are compatible (anon_vma_compatible).
- Otherwise a fresh anon_vma_chain is allocated.
- DONTUNMAP fully-copies the range: anon_vmas of source are unlinked via unlink_anon_vmas after the copy.

REQ-18: Per-hugetlb:
- check_prep_vma checks alignment: addr / new_addr / old_len / new_len all must be huge-page-aligned, else -EINVAL.
- move_hugetlb_page_tables walks huge_pte_offset, transfers huge PTEs.
- fixup_hugetlb_reservations after copy_vma_and_data.

REQ-19: Per-mmu_notifier:
- move_page_tables brackets the entire move with MMU_NOTIFY_UNMAP range_start/range_end so KVM / IOMMUv2 / amdgpu shadow translations are invalidated.
- vma_start_write blocks GUP-fast and page-fault concurrency for the source VMA.

## Acceptance Criteria

- [ ] AC-1: mremap(addr, old_len, new_len, 0): expand-in-place when next gap allows; -ENOMEM if not.
- [ ] AC-2: mremap(addr, old_len, new_len, MREMAP_MAYMOVE): on no-in-place, allocates new VMA via get_unmapped_area and moves page tables; returns new address.
- [ ] AC-3: mremap with new_len < old_len: shrinks (do_vmi_munmap of tail); returns addr.
- [ ] AC-4: mremap with MREMAP_FIXED, new_addr: do_munmap(new_addr, new_len) first, then move.
- [ ] AC-5: mremap with MREMAP_DONTUNMAP: source VMA preserved with VM_LOCKED cleared; page tables empty in source range; new VMA holds them.
- [ ] AC-6: mremap with MREMAP_DONTUNMAP ∧ old_len != new_len: -EINVAL.
- [ ] AC-7: mremap with MREMAP_FIXED but no MREMAP_MAYMOVE: -EINVAL.
- [ ] AC-8: mremap over VM_IO / VM_PFNMAP / VM_DONTEXPAND: -EINVAL via check_prep_vma.
- [ ] AC-9: mremap on hugetlb VMA: src/dst must be hugepage-aligned; otherwise -EINVAL.
- [ ] AC-10: move_page_tables fast path: PMD-aligned move copies a single PMD entry by pointer, then flush_tlb_range.
- [ ] AC-11: move_page_tables slow path: PTE-by-PTE move with batching of contiguous large-folio subpages.
- [ ] AC-12: Error during move: pmc_revert moves entries back; vrm targets new_vma for unmap.
- [ ] AC-13: vm_ops.mremap callback returns err: copy reverted; new_vma unmapped.
- [ ] AC-14: VM_ACCOUNT correctness across move: vm_committed_as not net-changed by a successful move.
- [ ] AC-15: VMA count never exceeds sysctl_max_map_count (check_map_count_against_split{,_early}).

## Architecture

```
bitflags MremapFlags : u32 {
  MAYMOVE   = 1 << 0,
  FIXED     = 1 << 1,
  DONTUNMAP = 1 << 2,
}

enum MremapType { NoResize, Shrink, Expand, Invalid }

struct VmaRemap {
  addr: u64, old_len: u64, new_len: u64,
  flags: MremapFlags, new_addr: u64,
  vma: Option<*Vma>,
  delta: u64,
  remap_type: MremapType,
  charged: u64,
  mmap_locked: bool,
  populate_expand: bool,
  vmi_needs_invalidate: bool,
  uf: *VmUserfaultfdCtx,
  uf_unmap: *List<...>,
  uf_unmap_early: *List<...>,
}

struct PageTableMoveControl {
  old: *Vma, new: *Vma,
  old_addr: u64, new_addr: u64,
  old_end: u64, len_in: u64,
  need_rmap_locks: bool,
  for_stack: bool,
}
```

`Mremap::sys_mremap(addr, old_len, new_len, flags, new_addr) -> u64`:
1. let vrm = VmaRemap {
   - addr: untagged_addr(addr),
   - old_len, new_len, flags: MremapFlags::from_bits(flags), new_addr,
   - uf, uf_unmap, uf_unmap_early,
   - remap_type: MremapType::Invalid,
   - ..Default::default()
   }.
2. return Mremap::do_mremap(&mut vrm).

`Mremap::do_mremap(vrm) -> u64`:
1. vrm.old_len = page_align(vrm.old_len).
2. vrm.new_len = page_align(vrm.new_len).
3. res = Mremap::check_params(vrm); if res != 0: return res.
4. mmap_write_lock_killable(current().mm) ?? return -EINTR.
5. vrm.mmap_locked = true.
6. if !Mremap::check_map_count_early(): mmap_write_unlock; return -ENOMEM.
7. if Mremap::move_only(vrm):
   - res = Mremap::remap_move(vrm).
8. else:
   - vrm.vma = vma_lookup(current().mm, vrm.addr).
   - res = Mremap::check_prep_vma(vrm); if res: goto out.
   - res = if Mremap::implies_new_addr(vrm) { Mremap::mremap_to(vrm) } else { Mremap::mremap_at(vrm) }.
9. out: failed = is_err_value(res).
10. if vrm.mmap_locked: mmap_write_unlock(current().mm).
11. if !failed ∧ vrm.populate_expand: mm_populate(vrm.new_addr + vrm.old_len, vrm.delta).
12. Mremap::notify_uffd(vrm, failed).
13. return res.

`Mremap::check_params(vrm) -> i64`:
1. if vrm.flags & !(MAYMOVE|FIXED|DONTUNMAP): return -EINVAL.
2. if offset_in_page(vrm.addr): return -EINVAL.
3. if vrm.new_len == 0: return -EINVAL.
4. if vrm.new_len > TASK_SIZE: return -EINVAL.
5. if !Mremap::implies_new_addr(vrm): return 0.
6. if vrm.new_addr > TASK_SIZE - vrm.new_len: return -EINVAL.
7. if offset_in_page(vrm.new_addr): return -EINVAL.
8. if !vrm.flags.contains(MAYMOVE): return -EINVAL.
9. if vrm.flags.contains(DONTUNMAP) ∧ vrm.old_len != vrm.new_len: return -EINVAL.
10. if Mremap::overlaps(vrm): return -EINVAL.
11. return 0.

`Mremap::mremap_at(vrm) -> u64`:
1. let kind = Mremap::remap_type(vrm).
2. match kind:
   - NoResize: return vrm.addr.
   - Shrink: return Mremap::shrink_vma(vrm, /*drop_lock=*/true).
   - Expand:
     - if Mremap::can_expand_in_place(vrm): return Mremap::expand_in_place(vrm).
     - if !vrm.flags.contains(MAYMOVE): return -ENOMEM.
     - res = Mremap::set_new_addr(vrm); if res: return res.
     - return Mremap::move_vma(vrm).

`Mremap::mremap_to(vrm) -> u64`:
1. if vrm.flags.contains(FIXED):
   - res = do_munmap(mm, vrm.new_addr, vrm.new_len, vrm.uf_unmap_early).
   - vrm.vma = None; vrm.vmi_needs_invalidate = true.
   - if res: return res.
   - vrm.vma = vma_lookup(mm, vrm.addr).
   - if vrm.vma.is_none(): return -EFAULT.
2. if vrm.remap_type == Shrink:
   - res = Mremap::shrink_vma(vrm, /*drop_lock=*/false); if res: return res.
   - vrm.old_len = vrm.new_len.
3. if vrm.flags.contains(DONTUNMAP):
   - let pages = vrm.old_len >> PAGE_SHIFT.
   - if !may_expand_vm(mm, &vrm.vma.flags, pages): return -ENOMEM.
4. res = Mremap::set_new_addr(vrm); if res: return res.
5. return Mremap::move_vma(vrm).

`Mremap::move_vma(vrm) -> u64`:
1. err = Mremap::prep_move_vma(vrm); if err: return err.
2. if !Mremap::calc_charge(vrm): return -ENOMEM.
3. vma_start_write(vrm.vma) /* block racing faults */.
4. err = Mremap::copy_vma_and_data(vrm, &mut new_vma).
5. if err ∧ new_vma.is_none(): return err.
6. let hiwater_vm = mm.hiwater_vm.
7. Mremap::stat_account(vrm, vrm.new_len).
8. if err == 0 ∧ vrm.flags.contains(DONTUNMAP):
   - Mremap::dontunmap_complete(vrm, new_vma).
9. else:
   - Mremap::unmap_source_vma(vrm).
10. mm.hiwater_vm = hiwater_vm.
11. return if err != 0 { err as u64 } else { vrm.new_addr }.

`Mremap::copy_vma_and_data(vrm, new_vma_out) -> i32`:
1. internal_offset = vrm.addr - vrm.vma.vm_start.
2. new_pgoff = vrm.vma.vm_pgoff + (internal_offset >> PAGE_SHIFT).
3. let pmc = PageTableMoveControl::new(None, None, vrm.addr, vrm.new_addr, vrm.old_len).
4. new_vma = copy_vma(&mut vrm.vma, vrm.new_addr, vrm.new_len, new_pgoff, &mut pmc.need_rmap_locks).
5. if new_vma.is_none(): Mremap::uncharge(vrm); *new_vma_out = None; return -ENOMEM.
6. if vma != vrm.vma: vrm.vmi_needs_invalidate = true.
7. pmc.old = vrm.vma; pmc.new = new_vma.
8. moved_len = Mremap::move_page_tables(&mut pmc).
9. err = 0.
10. if moved_len < vrm.old_len: err = -ENOMEM.
11. else if let Some(op) = vrm.vma.vm_ops.mremap: err = op(new_vma).
12. if err != 0:
    - let pmc_rev = PageTableMoveControl::new(Some(new_vma), Some(vrm.vma), vrm.new_addr, vrm.addr, moved_len).
    - pmc_rev.need_rmap_locks = true.
    - Mremap::move_page_tables(&mut pmc_rev) /* revert */.
    - vrm.vma = new_vma; vrm.old_len = vrm.new_len; vrm.addr = vrm.new_addr.
13. else: mremap_userfaultfd_prep(new_vma, vrm.uf).
14. fixup_hugetlb_reservations(vrm.vma).
15. *new_vma_out = new_vma.
16. return err.

`Mremap::move_page_tables(pmc) -> u64`:
1. if pmc.len_in == 0: return 0.
2. if is_vm_hugetlb_page(pmc.old):
   - return move_hugetlb_page_tables(pmc.old, pmc.new, pmc.old_addr, pmc.new_addr, pmc.len_in).
3. Mremap::try_realign_addr(pmc, PMD_MASK).
4. flush_cache_range(pmc.old, pmc.old_addr, pmc.old_end).
5. mmu_notifier_invalidate_range_start(MMU_NOTIFY_UNMAP, pmc.old_addr..pmc.old_end).
6. while !PageTableMoveControl::done(pmc):
   - cond_resched.
   - let extent: u64.
   - PUD-leaf path (CONFIG_HAVE_MOVE_PUD ∧ HPAGE_PUD-aligned): move_pgt_entry(HPAGE_PUD); extent = PUD_SIZE.
   - else NORMAL_PUD pointer-swap candidate: move_pgt_entry(NORMAL_PUD); extent = PUD_SIZE.
   - else PMD-leaf path (THP, HPAGE_PMD-aligned): move_pgt_entry(HPAGE_PMD); extent = PMD_SIZE.
   - else NORMAL_PMD pointer-swap candidate: move_pgt_entry(NORMAL_PMD); extent = PMD_SIZE.
   - else (slow path):
     - extent = min(PMD_SIZE - (pmc.old_addr & ~PMD_MASK), pmc.old_end - pmc.old_addr).
     - Mremap::move_ptes(pmc, old_pmd, new_pmd, extent).
   - PageTableMoveControl::next(pmc, extent).
7. mmu_notifier_invalidate_range_end.
8. return PageTableMoveControl::progress(pmc).

`Mremap::move_pgt_entry(pmc, entry, old_ent, new_ent) -> bool`:
1. need_rmap_locks = should_take_rmap_locks(pmc, entry).
2. if need_rmap_locks: take_rmap_locks(pmc.old).
3. moved = match entry:
   - NORMAL_PMD: Mremap::move_normal_pmd(pmc, old_ent, new_ent).
   - NORMAL_PUD: Mremap::move_normal_pud(pmc, old_ent, new_ent).
   - HPAGE_PMD: if CONFIG_TRANSPARENT_HUGEPAGE { move_huge_pmd(pmc.old, pmc.old_addr, pmc.new_addr, old_ent, new_ent) } else { false }.
   - HPAGE_PUD: if CONFIG_TRANSPARENT_HUGEPAGE { Mremap::move_huge_pud(pmc, old_ent, new_ent) } else { false }.
4. if need_rmap_locks: drop_rmap_locks(pmc.old).
5. return moved.

`Mremap::move_normal_pmd(pmc, old_pmd, new_pmd) -> bool`:
1. if pmd_none(*old_pmd): return false.
2. let pmd_val = pmdp_get_and_clear(pmc.old.vm_mm, pmc.old_addr, old_pmd).
3. set_pmd_at(pmc.new.vm_mm, pmc.new_addr, new_pmd, pmd_val).
4. flush_tlb_range(pmc.old, pmc.old_addr, pmc.old_addr + PMD_SIZE).
5. return true.

`Mremap::move_ptes(pmc, old_pmd, new_pmd, extent)`:
1. old_ptep = pte_offset_map_lock(mm, old_pmd, pmc.old_addr, &old_ptl).
2. new_ptep = pte_offset_map_nested_lock(mm, new_pmd, pmc.new_addr, &new_ptl).
3. flush_tlb_batched_pending(pmc.old.vm_mm).
4. addr = pmc.old_addr; new_addr = pmc.new_addr.
5. while addr < pmc.old_addr + extent:
   - batch = Mremap::folio_pte_batch(pmc.old, addr, old_ptep, max_in_extent).
   - for k in 0..batch:
     - pte = ptep_get_and_clear(pmc.old.vm_mm, addr+k*PAGE_SIZE, old_ptep+k).
     - if !pte_none(pte):
       - pte = pte_modify_for_move(pte, pmc.old, pmc.new).
       - set_pte_at(pmc.new.vm_mm, new_addr+k*PAGE_SIZE, new_ptep+k, pte).
   - addr += batch * PAGE_SIZE; new_addr += batch * PAGE_SIZE.
6. flush_tlb_range(pmc.old, pmc.old_addr, pmc.old_addr + extent).
7. pte_unmap_unlock(old_ptep, old_ptl); pte_unmap_unlock(new_ptep, new_ptl).

`Mremap::unmap_source_vma(vrm)`:
1. let accountable_move = vrm.vma.vm_flags.contains(VM_ACCOUNT) ∧ !vrm.flags.contains(DONTUNMAP).
2. if accountable_move: vm_flags_clear(vrm.vma, VM_ACCOUNT); store vm_start, vm_end.
3. err = do_vmi_munmap(&mut vmi, mm, vrm.addr, vrm.old_len, vrm.uf_unmap, /*unlock=*/false).
4. vrm.vma = None; vrm.vmi_needs_invalidate = true.
5. if err: vm_acct_memory(vrm.old_len >> PAGE_SHIFT) /* best-effort fix */; return.
6. if accountable_move:
   - if vm_start < vrm.addr: vm_flags_set(vma_prev(&vmi), VM_ACCOUNT).
   - if vm_end > vrm.addr + vrm.old_len: vm_flags_set(vma_next(&vmi), VM_ACCOUNT).

`Mremap::dontunmap_complete(vrm, new_vma)`:
1. vm_flags_clear(vrm.vma, VM_LOCKED_MASK).
2. if new_vma != vrm.vma ∧ vrm.addr == vrm.vma.vm_start ∧ vrm.addr+vrm.old_len == vrm.vma.vm_end:
   - unlink_anon_vmas(vrm.vma).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `dontunmap_implies_fixed_or_maymove_and_equal_len` | INVARIANT | per-check_params: DONTUNMAP ⟹ (FIXED ∨ MAYMOVE) ∧ old_len == new_len. |
| `fixed_implies_maymove` | INVARIANT | per-check_params: FIXED ⟹ MAYMOVE. |
| `src_dst_no_overlap` | INVARIANT | per-check_params + vrm_overlaps: src and dst ranges disjoint. |
| `mmap_write_locked_during_pt_move` | INVARIANT | per-move_page_tables: mmap_write_lock held throughout. |
| `vma_start_write_blocks_faults` | INVARIANT | per-move_vma: vma_start_write held before copy_vma_and_data. |
| `pt_move_revert_on_error_balances` | INVARIANT | per-copy_vma_and_data err: pmc_revert restores entries; vrm targets new_vma for unmap. |
| `map_count_split_headroom_enforced` | INVARIANT | per-check_map_count_against_split: map_count + 2 ≤ sysctl_max_map_count. |

### Layer 2: TLA+

`mm/mremap.tla`:
- Per-syscall + per-check_params + per-in-place expand/shrink + per-move (copy_vma + move_page_tables + unmap_source_vma) + per-DONTUNMAP variant + per-error-revert.
- Properties:
  - `safety_no_user_address_overlap` — per-call: dst-range never overlaps src-range when call returns success.
  - `safety_vma_count_bounded` — per-call: mm.map_count never exceeds sysctl_max_map_count.
  - `safety_dontunmap_keeps_source_vma` — per-DONTUNMAP success: source VMA still mapped (but locked_vm cleared).
  - `safety_vm_committed_balanced` — per-success: vm_committed_as net-unchanged for accounted move.
  - `safety_pt_move_atomic_w_r_t_uffd` — per-move: vma_start_write held; userfaultfd readers serialize.
  - `liveness_per_mremap_terminates` — per-do_mremap: returns within bounded iterations of move_page_tables.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Mremap::check_params` post: all invalid combos ⟹ -EINVAL; valid ⟹ 0 | `Mremap::check_params` |
| `Mremap::move_vma` post: returns new_addr ∨ -errno; on success, src VMA gone (unless DONTUNMAP) ∧ dst VMA at vrm.new_addr | `Mremap::move_vma` |
| `Mremap::move_page_tables` post: returned moved_len ≤ pmc.len_in; moved_len bytes of page-table entries transferred old→new | `Mremap::move_page_tables` |
| `Mremap::move_normal_pmd` post: old_pmd_none(*old_pmd) ∧ pmd_val(*new_pmd) == prior(*old_pmd) | `Mremap::move_normal_pmd` |
| `Mremap::unmap_source_vma` post: src-range unmapped ∧ VM_ACCOUNT correctly redistributed | `Mremap::unmap_source_vma` |
| `Mremap::dontunmap_complete` post: VM_LOCKED cleared on src; anon_vmas unlinked iff entire-VMA copy | `Mremap::dontunmap_complete` |
| `Mremap::copy_vma_and_data` post: err ⟹ revert leaves entries in src; ok ⟹ entries in dst | `Mremap::copy_vma_and_data` |

### Layer 4: Verus/Creusot functional

`Per-syscall → check_params → mmap_write_lock → check_map_count_early → (move_only: remap_move) | (in-place: mremap_at(NoResize|Shrink|Expand)) | (new-addr: mremap_to → munmap(new_addr)? → set_new_addr → move_vma → copy_vma → move_page_tables(PMD/PUD-swap or PTE-batch) → unmap_source_vma | dontunmap_complete) → mmap_write_unlock → mm_populate? → notify_uffd → return new-or-old addr` semantic equivalence: per-man-page mremap(2) + per-Linux mm/mremap.c + per-Documentation/admin-guide/mm/userfaultfd.rst (DONTUNMAP semantics).

## Hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening.)

mremap reinforcement:

- **Per-DONTUNMAP-must-equal-len enforced** — defense against per-userspace partial-move + size-change confusion.
- **Per-FIXED-requires-MAYMOVE enforced** — defense against per-FIXED-implicit-overwrite without explicit move intent.
- **Per-src/dst overlap rejected** — defense against per-self-overlapping page-table aliasing.
- **Per-VM_IO / VM_PFNMAP / VM_DONTEXPAND rejected** — defense against per-driver-MMIO mremap corruption.
- **Per-map_count split-headroom check pre-write-lock** — defense against per-mid-move ENOMEM with VMA tree in a torn state.
- **Per-vma_start_write before move_page_tables** — defense against per-GUP-fast / per-fault race on moving entries.
- **Per-mmu_notifier UNMAP bracket around move** — defense against per-secondary-MMU (KVM/IOMMU) stale translation.
- **Per-move_page_tables error: pmc_revert + new_vma unmap** — defense against per-partial-move leaving entries in both src and dst.
- **Per-VM_ACCOUNT clear/restore around src munmap** — defense against per-vm_committed_as accounting drift.
- **Per-vma_multi_allowed gate in remap_move** — defense against per-batched-move on hugetlb / special-VMAs that can't tolerate multi-VMA semantics.
- **Per-hugetlb alignment strictly enforced in check_prep_vma** — defense against per-hugetlb misaligned mremap corruption.
- **Per-ksm_madvise(MADV_UNMERGEABLE) before move** — defense against per-KSM-page address-confusion on the new location.
- **Per-untagged_addr(addr) but not new_addr** — defense against per-ARM64-MTE aliasing of the destination range (per ABI docs).

## Grsecurity/PaX-style Reinforcement

Baseline hardening that applies to `mremap(2)`:

- **PAX_USERCOPY** — `mremap` itself carries no payload; only `old_addr`/`old_len`/`new_len`/`flags`/`new_addr` flow through the syscall ABI.
- **PAX_KERNEXEC** — `move_vma`, `move_page_tables`, `move_ptes` reside in `.rodata`.
- **PAX_RANDKSTACK** — `sys_mremap` entered with randomized kernel stack offset.
- **PAX_REFCOUNT** — folio / VMA refcounts manipulated during move use saturating refcount_t; an unbalanced move cannot overflow into a free folio.
- **PAX_MEMORY_SANITIZE** — the unmapped source range is wiped before its pages re-enter the allocator if MREMAP_DONTUNMAP is not in effect.
- **PAX_UDEREF** — `old_addr`/`new_addr` validated via `find_vma`; page-table move dereferences kernel PTE pointers exclusively.
- **PAX_RAP / kCFI** — `vm_ops->mremap` dispatched through type-checked vtables.
- **GRKERNSEC_HIDESYM** — VMA / folio addresses in mremap WARNs redacted for non-root.
- **GRKERNSEC_DMESG** — accounting-drift warnings restricted from unprivileged dmesg.

mremap-specific reinforcement:

- **MREMAP_DONTUNMAP CAP_SYS_RESOURCE** — leaving a "shadow" mapping of the moved region after the move is restricted; without the capability, the source range is unmapped on success so an attacker cannot mass-pin memory through repeated DONTUNMAP cycles.
- **MREMAP_FIXED bounded** — `new_addr` must satisfy mmap_min_addr, address-space-randomization placement constraints, and PAX_MPROTECT W^X carryover from the source VMA; cross-class moves are refused.
- **`move_page_tables` error revert** — on partial failure the source/destination are unwound (`pmc_revert`, unmap of `new_vma`) so a failed move cannot leave entries installed in both ranges.
- **`vma_multi_allowed` gate** — batched multi-VMA moves refused for hugetlb / special VMAs that cannot tolerate the semantics.
- **`ksm_madvise(MADV_UNMERGEABLE)` before move** — KSM-merged pages are split before move so the new address cannot inherit a stale KSM identity.
- **`untagged_addr` enforced on source, not destination** — prevents ARM64 MTE tag-aliasing of the destination range.

Rationale: mremap is the only syscall that moves live page-table state across virtual addresses; the above ensure the move is atomic-with-revert, refcount-safe, and not a DONTUNMAP-based memory-pinning primitive for unprivileged callers.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- mm/mmap.c VMA allocation / merge / split (covered in `mmap.md` Tier-3)
- mm/hugetlb.c hugetlb-specific page-table move (covered in `hugetlb.md` Tier-3)
- mm/huge_memory.c THP move_huge_pmd internals (covered in `thp.md` Tier-3)
- mm/userfaultfd.c uffd-remove / uffd-prep registration (covered in `userfaultfd.md` Tier-3)
- mm/ksm.c MADV_UNMERGEABLE machinery (covered in `ksm.md` Tier-3)
- arch-specific pmdp_get_and_clear / set_pmd_at primitives (per-arch Tier-3)
- Implementation code
