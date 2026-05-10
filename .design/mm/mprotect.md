# Tier-3: mm/mprotect.c — Changing page protection

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: mm/00-overview.md
upstream-paths:
  - mm/mprotect.c (~1044 lines)
  - include/linux/mm.h (MM_CP_*, VM_READ/WRITE/EXEC, calc_vm_prot_bits)
  - include/linux/pkeys.h (mm_pkey_alloc/free, arch_*_pkey_*)
  - include/uapi/asm-generic/mman-common.h (PROT_*, PROT_NONE, PKEY_*)
-->

## Summary

`mprotect(2)` and `pkey_mprotect(2)` change the per-page protection of a virtual
address range without unmapping it: per-VMA the requested PROT_R/W/X bits are
translated to `VM_READ`, `VM_WRITE`, `VM_EXEC` and stamped via `mprotect_fixup`
(which splits/merges VMAs as needed and updates `vma->vm_page_prot`), and
per-PTE the kernel rewalks `[start, end)` through `change_protection_range →
change_p4d_range → change_pud_range → change_pmd_range → change_pte_range`,
modifying each present PTE via `modify_prot_start_ptes` / `pte_modify`, batching
contiguous identical PTEs with `mprotect_folio_pte_batch` for large folios, and
flushing through `struct mmu_gather *tlb`. NUMA-balancing piggybacks the same
walker by passing `MM_CP_PROT_NUMA` so PTEs are rewritten with `PAGE_NONE` to
trigger fault-time hinting; `MM_CP_TRY_CHANGE_WRITABLE` eagerly upgrades a PTE
to writable (instead of taking a write-fault) when the VMA's COW / writenotify
preconditions are met; `MM_CP_UFFD_WP` / `MM_CP_UFFD_WP_RESOLVE` toggle the
userfaultfd-WP marker. Per-mm protection-keys are allocated via
`pkey_alloc(2)` / freed via `pkey_free(2)` and bound to VMAs via
`arch_override_mprotect_pkey` (x86 PKU, arm64 MTE/PKEY). Critical for:
JIT codegen W^X transitions, sandbox unlock/relock, GC write-barriers,
NUMA-balancing, userfaultfd-WP, MPK isolation.

This Tier-3 covers `mm/mprotect.c` (~1044 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `maybe_change_pte_writable()` | per-PTE writability gate | `Mprotect::maybe_writable` |
| `can_change_private_pte_writable()` | per-private writability gate | `Mprotect::can_write_private` |
| `can_change_shared_pte_writable()` | per-shared writability gate | `Mprotect::can_write_shared` |
| `can_change_pte_writable()` | per-PTE dispatch | `Mprotect::can_write` |
| `mprotect_folio_pte_batch()` | per-large-folio PTE batch | `Mprotect::folio_batch` |
| `prot_commit_flush_ptes()` | per-batch PTE commit + TLB | `Mprotect::commit_flush` |
| `page_anon_exclusive_sub_batch()` | per-batch anon-exclusive split | `Mprotect::anon_excl_sub_batch` |
| `commit_anon_folio_batch()` | per-anon-batch commit | `Mprotect::commit_anon_batch` |
| `set_write_prot_commit_flush_ptes()` | per-write-upgrade commit | `Mprotect::set_write_commit` |
| `change_softleaf_pte()` | per-non-present (swap / migration) entry | `Mprotect::change_softleaf` |
| `change_present_ptes()` | per-present PTE batch driver | `Mprotect::change_present` |
| `change_pte_range()` | per-PMD PTE-loop walker | `Mprotect::pte_range` |
| `pgtable_split_needed()` | per-VMA huge-split predicate | `Mprotect::split_needed` |
| `pgtable_populate_needed()` | per-VMA pgtable-populate predicate | `Mprotect::populate_needed` |
| `change_pmd_prepare` (macro) | per-PMD prepare populate | `Mprotect::pmd_prepare` |
| `change_prepare` (macro) | per-PUD/P4D/PGD prepare | `Mprotect::prepare` |
| `change_pmd_range()` | per-PUD PMD loop | `Mprotect::pmd_range` |
| `change_pud_range()` | per-P4D PUD loop | `Mprotect::pud_range` |
| `change_p4d_range()` | per-PGD P4D loop | `Mprotect::p4d_range` |
| `change_protection_range()` | per-VMA PGD top-level | `Mprotect::range` |
| `change_protection()` | per-VMA dispatch (incl. hugetlb) | `Mprotect::change_protection` |
| `prot_none_pte_entry()` | per-PFN PROT_NONE allow check | `Mprotect::prot_none_pte` |
| `prot_none_hugetlb_entry()` | per-hugetlb PFN allow check | `Mprotect::prot_none_hugetlb` |
| `prot_none_test()` | per-walk gate | `Mprotect::prot_none_test` |
| `prot_none_walk_ops` | per-walker ops table | `PROT_NONE_WALK_OPS` |
| `mprotect_fixup()` | per-VMA split/merge + apply | `Mprotect::fixup` |
| `do_mprotect_pkey()` | per-syscall body | `Mprotect::do_mprotect_pkey` |
| `SYSCALL_DEFINE3(mprotect, ...)` | mprotect(2) entry | `Mprotect::sys_mprotect` |
| `SYSCALL_DEFINE4(pkey_mprotect, ...)` | pkey_mprotect(2) entry | `Mprotect::sys_pkey_mprotect` |
| `SYSCALL_DEFINE2(pkey_alloc, ...)` | pkey_alloc(2) entry | `Mprotect::sys_pkey_alloc` |
| `SYSCALL_DEFINE1(pkey_free, ...)` | pkey_free(2) entry | `Mprotect::sys_pkey_free` |
| `MM_CP_PROT_NUMA` | per-walker flag: NUMA hinting | shared |
| `MM_CP_TRY_CHANGE_WRITABLE` | per-walker flag: lazy COW | shared |
| `MM_CP_UFFD_WP` / `_RESOLVE` / `_ALL` | per-walker flag: uffd-wp | shared |
| `MM_CP_DIRTY_ACCT` | per-walker flag: dirty acct (writenotify) | shared |
| `MMU_NOTIFY_PROTECTION_VMA` | per-VMA mmu-notifier reason | shared |

## Compatibility contract

REQ-1: PROT_* → VM_* mapping (`calc_vm_prot_bits`):
- `PROT_READ`  → `VM_READ`.
- `PROT_WRITE` → `VM_WRITE`.
- `PROT_EXEC`  → `VM_EXEC`.
- `PROT_NONE`  → none (and PTE prot becomes `PAGE_NONE`).
- `PROT_GROWSDOWN` / `PROT_GROWSUP`: extend the range to `vma->vm_start` /
  `vma->vm_end` for stack-growth semantics.
- `personality & READ_IMPLIES_EXEC` ∧ `PROT_READ` ∧ `vma->vm_flags & VM_MAYEXEC`:
  add `PROT_EXEC`.
- `calc_vm_prot_bits(prot, pkey)` also folds the pkey into the high bits of the
  VM flag word.

REQ-2: per-VMA cap (`VM_MAY*`):
- `(newflags & ~(newflags >> 4)) & VM_ACCESS_FLAGS` must be 0; otherwise
  `-EACCES` (asking for a permission the VMA never permitted at `mmap`).

REQ-3: `do_mprotect_pkey(start, len, prot, pkey)`:
- `start = untagged_addr(start)`.
- Strip `PROT_GROWSDOWN | PROT_GROWSUP` into `grows`; both-set is `-EINVAL`.
- Page-align: `start & ~PAGE_MASK` → `-EINVAL`; `len == 0` → 0;
  `len = PAGE_ALIGN(len); end = start + len`; `end <= start` → `-ENOMEM`.
- `!arch_validate_prot(prot, start)` → `-EINVAL` (x86 rejects unknown bits,
  arm64 validates BTI / MTE compatibility).
- `mmap_write_lock_killable(current->mm)`.
- if `pkey != -1` ∧ `!mm_pkey_is_allocated(current->mm, pkey)`: `-EINVAL`.
- Initialize `vma_iter`; `vma_find(end)` → `-ENOMEM` if none.
- Handle `PROT_GROWSDOWN` / `PROT_GROWSUP` extensions per `VM_GROWSDOWN` /
  `VM_GROWSUP`.
- `tlb_gather_mmu(&tlb, current->mm)`.
- iterate `for_each_vma_range(vmi, vma, end)`:
  - Reject discontinuity (`vma->vm_start != tmp` → `-ENOMEM`).
  - `mask_off_old_flags = VM_ACCESS_FLAGS | VM_FLAGS_CLEAR`.
  - `new_vma_pkey = arch_override_mprotect_pkey(vma, prot, pkey)`.
  - `newflags = calc_vm_prot_bits(prot, new_vma_pkey) | (vma->vm_flags & ~mask_off_old_flags)`.
  - VM_MAY* cap check (REQ-2).
  - `map_deny_write_exec(old, new)` → `-EACCES` (W^X policy).
  - `arch_validate_flags(newflags)` → `-EINVAL` if rejected.
  - `security_file_mprotect(vma, reqprot, prot)` LSM hook.
  - `tmp = min(vma->vm_end, end)`.
  - if `vma->vm_ops->mprotect`: invoke it (e.g. drivers).
  - `mprotect_fixup(&vmi, &tlb, vma, &prev, nstart, tmp, newflags)`.
  - `tmp = vma_iter_end(&vmi); nstart = tmp; prot = reqprot`.
- `tlb_finish_mmu(&tlb)`.
- if `!error ∧ tmp < end`: `-ENOMEM`.
- `mmap_write_unlock`.

REQ-4: `sys_mprotect`: `do_mprotect_pkey(start, len, prot, -1)`.
REQ-5: `sys_pkey_mprotect`: `do_mprotect_pkey(start, len, prot, pkey)`.
REQ-6: `sys_pkey_alloc(flags, init_val)`:
- `flags != 0` → `-EINVAL`.
- `init_val & ~PKEY_ACCESS_MASK` → `-EINVAL`.
- `mmap_write_lock(current->mm)`.
- `pkey = mm_pkey_alloc(current->mm)`; `-1` → `-ENOSPC`.
- `arch_set_user_pkey_access(pkey, init_val)`; on failure `mm_pkey_free` and
  propagate.
- `mmap_write_unlock`; return `pkey`.

REQ-7: `sys_pkey_free(pkey)`:
- `mmap_write_lock; mm_pkey_free(mm, pkey); mmap_write_unlock`.

REQ-8: `mprotect_fixup(vmi, tlb, vma, pprev, start, end, newflags)`:
- `vma_is_sealed(vma)` → `-EPERM` (memfd-sealed regions).
- if `vma_flags_same_pair(old, new)`: `*pprev = vma`; return 0.
- PROT_NONE PFN permission walk (`prot_none_walk_ops`) when
  `arch_has_pfn_modify_check()` and VMA is `VMA_PFNMAP_BIT` or
  `VMA_MIXEDMAP_BIT` and the new flags have no access bits.
- Accounting:
  - new VM_WRITE ∧ space cap exceeded ∧ !old VM_WRITE: `-ENOMEM`.
  - new VM_WRITE ∧ !any-already-charged: `security_vm_enough_memory_mm`;
    `charged = nrpages`; set `VMA_ACCOUNT_BIT`.
  - !new VM_WRITE ∧ old VMA_ACCOUNT_BIT ∧ `vma_is_anonymous ∧ !vma->anon_vma`:
    drop `VMA_ACCOUNT_BIT`.
- `vma = vma_modify_flags(vmi, *pprev, vma, start, end, &new_vma_flags)` (may
  split / merge).
- `vma_start_write`; `vma_flags_reset_once`; if
  `vma_wants_manual_pte_write_upgrade(vma)`: `mm_cp_flags |= MM_CP_TRY_CHANGE_WRITABLE`.
- `vma_set_page_prot(vma)` recomputes `vma->vm_page_prot` from
  `vma->vm_flags` + arch defaults.
- `change_protection(tlb, vma, start, end, mm_cp_flags)`.
- If old was VMA_ACCOUNT and new is not: `vm_unacct_memory(nrpages)`.
- If new is writable VMA_LOCKED that was readonly: `populate_vma_page_range`
  to trigger COW immediately (avoids major fault on later write).
- `vm_stat_account(mm, old, -nrpages); vm_stat_account(mm, new, +nrpages)`.
- `perf_event_mmap(vma)` so perf sees the new prot.
- On error: `vm_unacct_memory(charged)`.

REQ-9: `change_protection`:
- `BUG_ON((cp_flags & MM_CP_UFFD_WP_ALL) == MM_CP_UFFD_WP_ALL)` — set/resolve
  cannot both be set.
- `newprot = vma->vm_page_prot`; if `MM_CP_PROT_NUMA`: `newprot = PAGE_NONE`.
- if `is_vm_hugetlb_page(vma)`: `hugetlb_change_protection`.
- else: `change_protection_range`.

REQ-10: `change_protection_range`:
- `tlb_start_vma(tlb, vma)`.
- iterate pgd: `change_prepare(vma, pgd, p4d, addr, cp_flags)` → populate p4d if
  uffd-wp requires markers; `pgd_none_or_clear_bad` skip; otherwise
  `change_p4d_range`.
- `tlb_end_vma(tlb, vma)`.

REQ-11: `change_pud_range`:
- Lazily wraps `mmu_notifier_invalidate_range_{start,end}` with
  `MMU_NOTIFY_PROTECTION_VMA` reason.
- `change_prepare(vma, pudp, pmd, addr, cp_flags)`.
- If `pud_leaf(pud)` (1 GiB huge): either split via `__split_huge_pud`
  (when range doesn't cover full PUD or markers needed) or `change_huge_pud`
  (whole leaf rewritten); otherwise recurse into `change_pmd_range`.

REQ-12: `change_pmd_range`:
- `change_pmd_prepare(vma, pmd, cp_flags)` allocates the PTE table if uffd-wp
  needs markers.
- If `pmd_is_huge(pmd)`: either `__split_huge_pmd` (range mismatch /
  file-backed THP + uffd-wp) or `change_huge_pmd` (whole leaf — counts
  `NUMA_HUGE_PTE_UPDATES`); otherwise fall through to `change_pte_range`.

REQ-13: `change_pte_range`:
- `tlb_change_page_size(tlb, PAGE_SIZE)`.
- `pte = pte_offset_map_lock(pmd, addr, &ptl)`; NULL → `-EAGAIN` (retry at
  caller).
- `flush_tlb_batched_pending(mm); lazy_mmu_mode_enable()`.
- Loop:
  - `oldpte = ptep_get(pte)`.
  - PRESENT branch:
    - if `prot_numa ∧ pte_protnone(oldpte)`: skip (already hinted).
    - `page = vm_normal_page(vma, addr, oldpte); folio = page_folio(page)`.
    - if `prot_numa` and folio not eligible (zero / KSM / NUMA-pinned):
      `nr_ptes = mprotect_folio_pte_batch(folio, pte, oldpte, max_nr_ptes, 0)`
      and skip the whole batch.
    - else: `nr_ptes = mprotect_folio_pte_batch(folio, pte, oldpte,
      max_nr_ptes, FPB_RESPECT_SOFT_DIRTY | FPB_RESPECT_WRITE)`.
    - `change_present_ptes(tlb, vma, addr, pte, nr_ptes, end, newprot, folio,
      page, cp_flags)`.
    - `pages += nr_ptes`.
  - NONE branch (`pte_none(oldpte)`):
    - if `!uffd_wp`: skip.
    - else if `userfaultfd_wp_use_markers(vma)`: install `PTE_MARKER_UFFD_WP`
      via `make_pte_marker` to wr-protect even an absent page.
  - else (swap / migration / hwpoison entry):
    - `pages += change_softleaf_pte(vma, addr, pte, oldpte, cp_flags)`.
  - advance by `nr_ptes`.
- `lazy_mmu_mode_disable(); pte_unmap_unlock`.

REQ-14: `change_present_ptes`:
- `oldpte = modify_prot_start_ptes(vma, addr, ptep, nr_ptes)` (arch hook;
  clears writable + collects dirty before retiring the entry).
- `ptent = pte_modify(oldpte, newprot)`.
- if `MM_CP_UFFD_WP`: `pte_mkuffd_wp(ptent)`.
- else if `MM_CP_UFFD_WP_RESOLVE`: `pte_clear_uffd_wp(ptent)`.
- if `MM_CP_TRY_CHANGE_WRITABLE ∧ !pte_write(ptent)`:
  `set_write_prot_commit_flush_ptes(...)` (write-upgrade path; calls
  `can_change_pte_writable` per page).
- else: `prot_commit_flush_ptes(...)` (read-only commit).

REQ-15: `can_change_pte_writable(vma, addr, pte)`:
- Dispatch on `VM_SHARED`:
  - private: `vm_normal_page(vma, addr, pte)` must be `PageAnon` ∧
    `PageAnonExclusive` (otherwise COW could be needed).
  - shared: `pte_dirty(pte)` (clean shared still wants writenotify trap).
- Both branches first gate on `maybe_change_pte_writable(vma, pte)`:
  - `vma->vm_flags & VM_WRITE` (warn if not).
  - `!pte_protnone(pte)` (NUMA-hinted PTEs are not writable).
  - `!pte_needs_soft_dirty_wp(vma, pte)` (softdirty tracking still wants the trap).
  - `!userfaultfd_pte_wp(vma, pte)` (uffd-wp still wants the trap).

REQ-16: `mprotect_folio_pte_batch`:
- Batch contiguous PTEs of the same folio (large folios only), respecting the
  caller's `fpb_t` flags (soft-dirty / write); return 1 for small folios.

REQ-17: `prot_none_pte_entry` / `prot_none_hugetlb_entry`:
- Used by `prot_none_walk_ops` to refuse PROT_NONE → R/W/X transitions on
  PFNMAP / hugetlb VMAs where the underlying PFN can't legally take the new
  permission (`pfn_modify_allowed(pfn, new_pgprot) == false` → `-EACCES`).

REQ-18: NUMA-hinting (`MM_CP_PROT_NUMA`):
- Caller (`task_numa_work`) supplies the flag; `change_protection` rewrites
  `newprot = PAGE_NONE`; PTEs get `pte_protnone` so the next access traps and
  enters `do_numa_page`.
- KSM-folios and zero-page are skipped (cannot migrate).
- Huge-PMD path increments `NUMA_HUGE_PTE_UPDATES`.

REQ-19: `MM_CP_DIRTY_ACCT` / `vma_wants_manual_pte_write_upgrade`:
- VMA is writable, shared and wants writenotify ∨ shared FS-backed:
  `mprotect_fixup` sets `MM_CP_TRY_CHANGE_WRITABLE` so dirty PTEs upgrade
  in-place; clean ones still trap so the FS can be notified.

REQ-20: `MM_CP_UFFD_WP_ALL = MM_CP_UFFD_WP | MM_CP_UFFD_WP_RESOLVE`:
- Cannot both be set (BUG_ON). UFFDIO_WRITEPROTECT enters with exactly one of
  the two depending on the requested transition.

REQ-21: `anon_vma` reuse:
- `mprotect_fixup` does NOT allocate `anon_vma`; that is `mmap_region` /
  `vma_modify_flags` territory. A previously anonymous, never-faulted VMA
  becoming non-writable can drop `VMA_ACCOUNT_BIT` because `anon_vma == NULL`
  guarantees no charged pages exist yet (REQ-8).

REQ-22: MPK / pkey integration:
- `pkey == -1` means legacy `mprotect`: `arch_override_mprotect_pkey` returns
  the current pkey of the VMA (or default 0) on x86.
- `pkey != -1`: `mm_pkey_is_allocated` mandatory; the pkey is folded into the
  per-VMA flags by `calc_vm_prot_bits`, then realized in PTE bits by
  `pte_modify` via `vma->vm_page_prot` recompute.
- `pkey_alloc` reserves a pkey out of `mm->context.pkey_allocation_map` and
  programs `PKRU` (x86) / equivalent register; `pkey_free` clears the bit.

REQ-23: VMA-split / merge:
- `vma_modify_flags` (mm/mmap.c) decides whether to merge with `prev` / `next`
  or split. `mprotect_fixup` only stamps flags and triggers PTE rewrite; it
  must re-fetch `vma` from the return.

REQ-24: TLB batching:
- One `mmu_gather *tlb` per `do_mprotect_pkey` call covering all touched VMAs.
- `tlb_start_vma` / `tlb_end_vma` bookend each VMA inside
  `change_protection_range`.
- `tlb_finish_mmu` at the end flushes once.

## Acceptance Criteria

- [ ] AC-1: `mprotect(p, len, PROT_READ)` strips `VM_WRITE` from every covered
  VMA and rewrites every present PTE read-only.
- [ ] AC-2: `mprotect(p, len, PROT_NONE)` rewrites PTEs to `PAGE_NONE`;
  subsequent access SIGSEGVs at user level.
- [ ] AC-3: `mprotect` returning `0` guarantees the entire range was
  re-permissioned, not a prefix (else `-ENOMEM` propagates).
- [ ] AC-4: Mid-range protect splits the originating VMA (head, middle, tail)
  with no gap; merges into neighbors if flags match.
- [ ] AC-5: Asking for a permission absent from `VM_MAY*` yields `-EACCES`.
- [ ] AC-6: `map_deny_write_exec` rejects W^X violations with `-EACCES`.
- [ ] AC-7: NUMA-hinting (`MM_CP_PROT_NUMA`) leaves zero / KSM folios alone
  and rewrites the rest to `PAGE_NONE`.
- [ ] AC-8: `MM_CP_TRY_CHANGE_WRITABLE` upgrades a clean+dirty shared dirty PTE
  to writable without taking a write-fault.
- [ ] AC-9: `MM_CP_UFFD_WP` on a `pte_none` slot installs a uffd-wp marker
  when the VMA uses markers.
- [ ] AC-10: `pkey_alloc` → `pkey_mprotect(p, len, prot, pkey)` →
  `pkey_free(pkey)` round-trip leaves no dangling pkey reference; reusing the
  pkey post-free starts from a clean access mask.
- [ ] AC-11: `pkey_mprotect(..., pkey)` with an unallocated pkey returns
  `-EINVAL`.
- [ ] AC-12: `mprotect_fixup` recomputes `vma->vm_page_prot` so a later
  fault uses the new protections without re-running the walker.
- [ ] AC-13: `perf_event_mmap(vma)` fires once per touched VMA.
- [ ] AC-14: Sealed VMA (`vma_is_sealed`) refuses `mprotect` with `-EPERM`.
- [ ] AC-15: `PROT_GROWSDOWN` only succeeds on a `VM_GROWSDOWN` VMA;
  `PROT_GROWSUP` only on `VM_GROWSUP`.

## Architecture

```
struct CpFlags(u64);              // MM_CP_* bitset
const MM_CP_PROT_NUMA: u64           = 1 << 0;
const MM_CP_TRY_CHANGE_WRITABLE: u64 = 1 << 1;
const MM_CP_DIRTY_ACCT: u64          = 1 << 2;
const MM_CP_UFFD_WP: u64             = 1 << 3;
const MM_CP_UFFD_WP_RESOLVE: u64     = 1 << 4;
const MM_CP_UFFD_WP_ALL: u64         = MM_CP_UFFD_WP | MM_CP_UFFD_WP_RESOLVE;

// PROT_* → VM_*
fn calc_vm_prot_bits(prot: u32, pkey: i32) -> VmFlags { ... }
```

`Mprotect::sys_mprotect(start, len, prot)`:
1. return `do_mprotect_pkey(start, len, prot, -1)`.

`Mprotect::sys_pkey_mprotect(start, len, prot, pkey)`:
1. return `do_mprotect_pkey(start, len, prot, pkey)`.

`Mprotect::do_mprotect_pkey(start, len, prot, pkey) -> i32`:
1. `start = untagged_addr(start)`.
2. `grows = prot & (PROT_GROWSDOWN | PROT_GROWSUP); prot &= ~grows`.
3. `if grows == both: return -EINVAL`.
4. align/check: `start & ~PAGE_MASK` → -EINVAL; `len == 0` → 0;
   `len = PAGE_ALIGN(len); end = start + len; end <= start` → -ENOMEM.
5. `!arch_validate_prot(prot, start)` → -EINVAL.
6. `reqprot = prot`.
7. `mmap_write_lock_killable(current.mm)?`.
8. err = -EINVAL; if `pkey != -1 ∧ !mm_pkey_is_allocated(current.mm, pkey)`:
   goto out.
9. `vma_iter_init(&vmi, current.mm, start)`; `vma = vma_find(&vmi, end)`;
   if !vma: err = -ENOMEM; goto out.
10. Handle GROWSDOWN/GROWSUP extension (REQ-3).
11. `prev = vma_prev(&vmi); if start > vma.vm_start: prev = vma`.
12. `tlb_gather_mmu(&tlb, current.mm)`.
13. `nstart = start; tmp = vma.vm_start`.
14. for_each_vma_range(vmi, vma, end):
    - if `vma.vm_start != tmp`: err = -ENOMEM; break.
    - if `rier ∧ (vma.vm_flags & VM_MAYEXEC)`: `prot |= PROT_EXEC`.
    - `mask_off = VM_ACCESS_FLAGS | VM_FLAGS_CLEAR`.
    - `new_vma_pkey = arch_override_mprotect_pkey(vma, prot, pkey)`.
    - `newflags = calc_vm_prot_bits(prot, new_vma_pkey) | (vma.vm_flags & ~mask_off)`.
    - `new_vma_flags = legacy_to_vma_flags(newflags)`.
    - if `(newflags & ~(newflags >> 4)) & VM_ACCESS_FLAGS`: err = -EACCES; break.
    - if `map_deny_write_exec(&vma.flags, &new_vma_flags)`: err = -EACCES; break.
    - if `!arch_validate_flags(newflags)`: err = -EINVAL; break.
    - `security_file_mprotect(vma, reqprot, prot)?`.
    - `tmp = min(vma.vm_end, end)`.
    - if `vma.vm_ops ∧ vma.vm_ops->mprotect`:
      `vma.vm_ops.mprotect(vma, nstart, tmp, newflags)?`.
    - `mprotect_fixup(&vmi, &tlb, vma, &prev, nstart, tmp, newflags)?`.
    - `tmp = vma_iter_end(&vmi); nstart = tmp; prot = reqprot`.
15. `tlb_finish_mmu(&tlb)`.
16. if `!err ∧ tmp < end`: err = -ENOMEM.
17. out: `mmap_write_unlock`; return err.

`Mprotect::fixup(vmi, tlb, vma, pprev, start, end, newflags) -> i32`:
1. if `vma_is_sealed(vma)`: return -EPERM.
2. `old = READ_ONCE(vma.flags); new = legacy_to_vma_flags(newflags)`.
3. if `vma_flags_same_pair(old, new)`: `*pprev = vma`; return 0.
4. PROT_NONE PFN walk (REQ-8) when arch_has_pfn_modify_check and
   PFNMAP/MIXEDMAP and no access bits in new.
5. Accounting (REQ-8): may set/clear `VMA_ACCOUNT_BIT`, may charge or refuse
   via `security_vm_enough_memory_mm`.
6. `vma = vma_modify_flags(vmi, *pprev, vma, start, end, &new)?`.
7. `*pprev = vma`.
8. `vma_start_write(vma); vma_flags_reset_once(vma, &new)`.
9. `mm_cp_flags = 0`.
10. if `vma_wants_manual_pte_write_upgrade(vma)`:
    `mm_cp_flags |= MM_CP_TRY_CHANGE_WRITABLE`.
11. `vma_set_page_prot(vma)`.
12. `change_protection(tlb, vma, start, end, mm_cp_flags)`.
13. if `old.VMA_ACCOUNT_BIT ∧ !new.VMA_ACCOUNT_BIT`: `vm_unacct_memory(nrpages)`.
14. if `new.VMA_WRITE_BIT ∧ old.VMA_LOCKED_BIT ∧
    !(old.VMA_WRITE_BIT | old.VMA_SHARED_BIT)`:
    `populate_vma_page_range(vma, start, end, NULL)` (force COW for VM_LOCKED).
15. `vm_stat_account(mm, legacy(old), -nrpages); vm_stat_account(mm, newflags, +nrpages)`.
16. `perf_event_mmap(vma)`.
17. return 0.

`Mprotect::change_protection(tlb, vma, start, end, cp_flags) -> i64`:
1. `assert (cp_flags & MM_CP_UFFD_WP_ALL) != MM_CP_UFFD_WP_ALL`.
2. `newprot = vma.vm_page_prot`.
3. if `MM_CP_PROT_NUMA`: `newprot = PAGE_NONE`.
4. if `is_vm_hugetlb_page(vma)`:
   `pages = hugetlb_change_protection(vma, start, end, newprot, cp_flags)`.
5. else:
   `pages = change_protection_range(tlb, vma, start, end, newprot, cp_flags)`.
6. return pages.

`Mprotect::range(tlb, vma, addr, end, newprot, cp_flags) -> i64`:
1. `pgd = pgd_offset(mm, addr); pages = 0`.
2. `tlb_start_vma(tlb, vma)`.
3. do:
   - `next = pgd_addr_end(addr, end)`.
   - `ret = change_prepare(vma, pgd, p4d, addr, cp_flags)`.
   - if `ret`: pages = ret; break.
   - if `pgd_none_or_clear_bad(pgd)`: continue.
   - `pages += change_p4d_range(tlb, vma, pgd, addr, next, newprot, cp_flags)`.
   - advance pgd; addr = next.
   while `addr != end`.
4. `tlb_end_vma(tlb, vma)`.

`Mprotect::pud_range(tlb, vma, p4d, addr, end, newprot, cp_flags) -> i64`:
1. init `range`, lazily set up `mmu_notifier_range_init(MMU_NOTIFY_PROTECTION_VMA,...)`
   on first non-empty PUD.
2. for each pud:
   - `ret = change_prepare(vma, pudp, pmd, addr, cp_flags)`; on err break.
   - if `pud_none(pud)`: continue.
   - if `pud_leaf(pud)`:
     - if `next - addr != PUD_SIZE ∨ pgtable_split_needed(vma, cp_flags)`:
       `__split_huge_pud(vma, pudp, addr); goto again`.
     - else: `ret = change_huge_pud(tlb, vma, pudp, addr, newprot, cp_flags)`;
       on `ret == 0` again; if `HPAGE_PUD_NR` count it; continue.
   - else: `pages += change_pmd_range(tlb, vma, pudp, addr, next, newprot, cp_flags)`.
3. if `range.start`: `mmu_notifier_invalidate_range_end(&range)`.

`Mprotect::pmd_range(tlb, vma, pud, addr, end, newprot, cp_flags) -> i64`:
1. iterate pmd:
   - `change_pmd_prepare(vma, pmd, cp_flags)`.
   - if `pmd_none(*pmd)`: next.
   - `_pmd = pmdp_get_lockless(pmd)`.
   - if `pmd_is_huge(_pmd)`:
     - if `next - addr != HPAGE_PMD_SIZE ∨ pgtable_split_needed`:
       `__split_huge_pmd(vma, pmd, addr, false); change_pmd_prepare again;` fall through.
     - else: `ret = change_huge_pmd(tlb, vma, pmd, addr, newprot, cp_flags)`;
       if `ret == HPAGE_PMD_NR`: `pages += HPAGE_PMD_NR; nr_huge_updates++`; goto next.
   - `ret = change_pte_range(tlb, vma, pmd, addr, next, newprot, cp_flags)`.
   - on `ret < 0`: goto again (race on `pte_offset_map_lock`).
   - `pages += ret`.
2. if `nr_huge_updates`: `count_vm_numa_events(NUMA_HUGE_PTE_UPDATES, nr_huge_updates)`.

`Mprotect::pte_range(tlb, vma, pmd, addr, end, newprot, cp_flags) -> i64`:
1. `tlb_change_page_size(tlb, PAGE_SIZE)`.
2. `(pte, ptl) = pte_offset_map_lock(mm, pmd, addr)`; if !pte: return -EAGAIN.
3. `if MM_CP_PROT_NUMA: is_priv_single = vma_is_single_threaded_private(vma)`.
4. `flush_tlb_batched_pending(mm); lazy_mmu_mode_enable()`.
5. loop:
   - `oldpte = ptep_get(pte)`.
   - PRESENT:
     - if `prot_numa ∧ pte_protnone(oldpte)`: continue.
     - `page = vm_normal_page; folio = page_folio(page)`.
     - `flags = FPB_RESPECT_SOFT_DIRTY | FPB_RESPECT_WRITE`.
     - `max_nr = (end - addr) >> PAGE_SHIFT`.
     - if `prot_numa ∧ !folio_can_map_prot_numa(folio, vma, is_priv_single)`:
       `nr_ptes = mprotect_folio_pte_batch(folio, pte, oldpte, max_nr, 0)`;
       continue.
     - `nr_ptes = mprotect_folio_pte_batch(folio, pte, oldpte, max_nr, flags)`.
     - if `nr_ptes == 1`: `change_present_ptes(tlb, vma, addr, pte, 1, end,
       newprot, folio, page, cp_flags)`.
     - else: `change_present_ptes(... nr_ptes ...)`.
     - `pages += nr_ptes`.
   - NONE:
     - if `!uffd_wp`: continue.
     - if `userfaultfd_wp_use_markers(vma)`:
       `set_pte_at(mm, addr, pte, make_pte_marker(PTE_MARKER_UFFD_WP)); pages++`.
   - else (swap-leaf etc):
     - `pages += change_softleaf_pte(vma, addr, pte, oldpte, cp_flags)`.
   - advance.
6. `lazy_mmu_mode_disable(); pte_unmap_unlock(pte - 1, ptl)`.

`Mprotect::change_present(tlb, vma, addr, ptep, nr_ptes, end, newprot, folio, page, cp_flags)`:
1. `oldpte = modify_prot_start_ptes(vma, addr, ptep, nr_ptes)`.
2. `ptent = pte_modify(oldpte, newprot)`.
3. if `MM_CP_UFFD_WP`: `ptent = pte_mkuffd_wp(ptent)`.
4. elif `MM_CP_UFFD_WP_RESOLVE`: `ptent = pte_clear_uffd_wp(ptent)`.
5. if `MM_CP_TRY_CHANGE_WRITABLE ∧ !pte_write(ptent)`:
   `set_write_prot_commit_flush_ptes(vma, folio, page, addr, ptep, oldpte,
   ptent, nr_ptes, tlb)`.
6. else: `prot_commit_flush_ptes(vma, addr, ptep, oldpte, ptent, nr_ptes, 0,
   false, tlb)`.

`Mprotect::can_write(vma, addr, pte) -> bool`:
1. if `!(vma.vm_flags & VM_SHARED)`: return `can_write_private(vma, addr, pte)`.
2. else: return `can_write_shared(vma, pte)`.

`Mprotect::sys_pkey_alloc(flags, init_val) -> i32`:
1. if `flags != 0`: return -EINVAL.
2. if `init_val & ~PKEY_ACCESS_MASK`: return -EINVAL.
3. `mmap_write_lock(current.mm)`.
4. `pkey = mm_pkey_alloc(current.mm)`; if `pkey == -1`: ret = -ENOSPC; goto out.
5. `ret = arch_set_user_pkey_access(pkey, init_val)`; if `ret`:
   `mm_pkey_free(current.mm, pkey)`; goto out.
6. ret = pkey.
7. out: `mmap_write_unlock`; return ret.

`Mprotect::sys_pkey_free(pkey) -> i32`:
1. `mmap_write_lock(current.mm)`.
2. `ret = mm_pkey_free(current.mm, pkey)`.
3. `mmap_write_unlock`; return ret.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pte_modify_preserves_pfn` | INVARIANT | per-PTE: `pte_modify` keeps the PFN intact while replacing prot bits. |
| `vm_may_cap_enforced` | INVARIANT | per-VMA: `(newflags & ~(newflags >> 4)) & VM_ACCESS_FLAGS == 0`. |
| `uffd_wp_set_resolve_exclusive` | INVARIANT | per-call: `(cp_flags & MM_CP_UFFD_WP_ALL) != MM_CP_UFFD_WP_ALL`. |
| `mprotect_locks_mmap_write` | INVARIANT | per-syscall: `mmap_write_lock` held across the whole VMA iteration. |
| `tlb_gather_finish_paired` | INVARIANT | per-syscall: exactly one `tlb_gather_mmu` matched by one `tlb_finish_mmu`. |
| `pkey_alloc_free_balanced` | INVARIANT | per-mm: `pkey_alloc` returns a pkey; `pkey_free` clears the same bit. |
| `protnone_skipped_under_prot_numa` | INVARIANT | per-PTE: PROT_NUMA walk skips already-protnone PTEs. |

### Layer 2: TLA+

`mm/mprotect.tla`:
- Models: VMA flag transitions, per-PTE prot state, pkey allocation map, TLB
  gather lifecycle, mmu-notifier `start/end` bracket.
- Properties:
  - `safety_no_partial_grant` — `mprotect` returning 0 implies every PTE in
    `[start, end)` has the new prot (or is a softleaf / none that the caller
    semantics permit).
  - `safety_protnone_blocks_write` — after PROT_NONE, no successful write to
    any PTE in the range until another `mprotect` upgrades it.
  - `safety_pkey_isolated` — a pkey freed by one mm does not affect VMAs of
    another mm holding the same pkey number.
  - `safety_writeupgrade_only_when_safe` — `MM_CP_TRY_CHANGE_WRITABLE` upgrades
    a PTE only if `can_change_pte_writable` holds.
  - `liveness_per_mprotect_terminates` — `do_mprotect_pkey` either succeeds or
    returns a single error code.
  - `safety_mmu_notifier_paired` — every `_start` is matched by exactly one `_end`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_mprotect_pkey` post: success ⟹ all VMAs in `[start, end)` updated | `Mprotect::do_mprotect_pkey` |
| `fixup` post: `vma->vm_page_prot` recomputed; PTEs walked | `Mprotect::fixup` |
| `change_protection` post: pages-returned == count of PTEs whose prot changed | `Mprotect::change_protection` |
| `change_pte_range` post: lazy_mmu_mode disabled, pte_unmap_unlock called | `Mprotect::pte_range` |
| `change_present_ptes` post: `pte_modify` applied; uffd-wp bit set/cleared per cp_flags | `Mprotect::change_present` |
| `can_change_pte_writable` post: returns true ⟹ writable upgrade is observationally equivalent to write-fault | `Mprotect::can_write` |
| `prot_none_walk` post: returns 0 only if every PFN passed `pfn_modify_allowed` | `PROT_NONE_WALK_OPS` |
| `pkey_alloc` post: pkey ∈ `mm.context.pkey_allocation_map` | `Mprotect::sys_pkey_alloc` |
| `pkey_free` post: pkey ∉ `mm.context.pkey_allocation_map` | `Mprotect::sys_pkey_free` |

### Layer 4: Verus/Creusot functional

`Per-mprotect(2) / pkey_mprotect(2) → do_mprotect_pkey → (for_each VMA) {
arch_override_mprotect_pkey → calc_vm_prot_bits → map_deny_write_exec →
security_file_mprotect → vma_ops->mprotect? → mprotect_fixup ( → PROT_NONE PFN
walk → vma_modify_flags → vma_set_page_prot → change_protection (hugetlb |
range → p4d → pud → pmd → pte) → pte_modify per-PTE → tlb_gather/finish ) }`
semantic equivalence: per-`Documentation/admin-guide/mm/` and POSIX `mprotect`.
NUMA-balancing: per-`Documentation/admin-guide/mm/numa_memory_policy.rst` and
`task_numa_work` invocation with `MM_CP_PROT_NUMA`. uffd-wp: per-userfaultfd
`UFFDIO_WRITEPROTECT`. MPK: per-`Documentation/core-api/protection-keys.rst`.

## Hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening.)

mprotect reinforcement:

- **Per-VMA `VM_MAY*` cap** — defense against per-elevation of protection
  beyond what `mmap` originally allowed.
- **Per-syscall `map_deny_write_exec`** — defense against per-W^X violation.
- **Per-syscall `security_file_mprotect` LSM hook** — defense against
  per-policy bypass (SELinux EXECMOD, AppArmor mprotect rules).
- **Per-arch `arch_validate_prot` / `arch_validate_flags`** — defense against
  per-arch invalid bit combinations (BTI / MTE / PKEY / SEALED).
- **Per-VMA sealed-region check (`vma_is_sealed`)** — defense against
  per-memfd-seal bypass.
- **Per-pkey `mm_pkey_is_allocated` gate** — defense against
  per-stale-pkey grant after another thread's `pkey_free`.
- **Per-PFN `prot_none_walk_ops`** — defense against per-driver PFN
  inadvertently granted PROT_NONE → R/W transitions on hot-plug holes.
- **Per-`MM_CP_UFFD_WP_ALL` BUG_ON** — defense against per-caller passing both
  set and resolve.
- **Per-`mmap_write_lock_killable`** — defense against per-OOM-stall hang.
- **Per-`tlb_gather_mmu` / `tlb_finish_mmu` paired** — defense against
  per-stale-TLB exposing the old prot.
- **Per-VMA `mmu_notifier_invalidate_range_{start,end}` paired** — defense
  against per-secondary-MMU (IOMMU / KVM EPT) racing the prot change.
- **Per-PTE `modify_prot_start_ptes` collects dirty/young** — defense against
  per-lost-dirty causing FS data loss after read-only → writable downgrade.
- **Per-`can_change_pte_writable` (dirty + anon-exclusive only)** — defense
  against per-COW bypass / writenotify bypass on write-upgrade.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `mm/huge_memory.c` `__split_huge_pmd`, `__split_huge_pud`, `change_huge_pmd`,
  `change_huge_pud` (covered in `transparent-hugepage.md` Tier-3).
- `mm/hugetlb.c` `hugetlb_change_protection` (covered in `hugetlb.md`).
- `mm/mmap.c` `vma_modify_flags` split/merge mechanics (covered in `mmap.md`).
- `mm/userfaultfd.c` uffd-wp marker plumbing (covered in `userfaultfd.md` if
  expanded).
- `arch/x86/mm/pkeys.c` / `arch/arm64/mm/pkeys.c` MPK realization (covered in
  per-arch Tier-3 if expanded).
- `mm/migrate.c` NUMA-balancing migration after fault (covered in
  `migration-compaction.md`).
- Implementation code.
