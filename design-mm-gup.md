---
title: "Tier-3: mm/gup.c — get_user_pages() family"
tags: ["tier-3", "mm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

`get_user_pages()` and its siblings pin user-virtual pages so that kernel code (typically a DMA-issuing driver, RDMA stack, vfio, io_uring, ptrace, or process_vm_readv/writev) can safely access them without the user mm pulling them out from underneath. Two distinct families exist: the `_get` family (FOLL_GET — refcount +1 via `folio_ref_add`) and the `_pin` family (FOLL_PIN — refcount +GUP_PIN_COUNTING_BIAS for normal folios, or refcount+1 plus `folio->_pincount` for large/longterm-tracked folios). Per-call fast-path `gup_fast()` walks the page tables lockless under `local_irq_save` using `raw_seqcount` against `mm->write_protect_seq` (to catch a racing fork() COW-protect); slow-path `__get_user_pages_locked()` takes `mmap_lock` for read, calls `follow_page_mask()` → `follow_page_pte()` → `try_grab_folio()`, and on absent / write-on-COW pages invokes `faultin_page()` → `handle_mm_fault()`. Per-`FOLL_LONGTERM` callers (RDMA, vfio) are routed through `__gup_longterm_locked()` which rejects pages from MOVABLE/CMA zones and from fs-DAX VMAs. Per-`FOLL_FORCE` allows ptrace-style read-of-PROT_NONE / COW-write of read-only-private. Per-COW: a write-pin of a read-only PTE returns `-EMLINK` to trigger `gup_must_unshare`, forcing the COW to break before the page is pinned. Critical for: GPU/RDMA DMA correctness, fork-safety of pinned buffers, hugetlb compatibility, long-term-pin accounting (`mm->pinned_vm`, `MMF_HAS_PINNED`).

This Tier-3 covers `mm/gup.c` (~3557 lines).

### Acceptance Criteria

- [ ] AC-1: get_user_pages on a present writable page: returns 1 with refcount + 1.
- [ ] AC-2: pin_user_pages on a present writable page: returns 1 with pincount + 1 ∧ MMF_HAS_PINNED set on mm.
- [ ] AC-3: get_user_pages on absent page: faultin_page → handle_mm_fault → retry succeeds.
- [ ] AC-4: pin_user_pages with FOLL_WRITE on read-only COW anon: gup_must_unshare triggers — COW broken — pin on the new exclusive page.
- [ ] AC-5: FOLL_LONGTERM on MOVABLE-zone folio: folio migrated to UNMOVABLE before pinning.
- [ ] AC-6: FOLL_LONGTERM on fsdax VMA: -EOPNOTSUPP.
- [ ] AC-7: get_user_pages_fast: fork() racing with FOLL_PIN: write_protect_seq retry → 0 nr_pinned → fallback to slow.
- [ ] AC-8: FOLL_FAST_ONLY + miss: returns partial nr_pinned ∨ 0; no slow-path call.
- [ ] AC-9: FOLL_WRITE on VM_READ-only VMA without FOLL_FORCE: -EFAULT.
- [ ] AC-10: FOLL_FORCE + write + VM_MAYWRITE COW-mapping: succeeds.
- [ ] AC-11: unpin_user_pages_dirty_lock(make_dirty=true): set_page_dirty observed on filesystem.
- [ ] AC-12: FOLL_GET + FOLL_PIN in same call: VM_WARN_ON_ONCE fires.
- [ ] AC-13: Zero-page pin: refcount unchanged; unpin no-op.
- [ ] AC-14: Hugetlb VMA + get_user_pages: returns subpages with head refcount incremented per page.
- [ ] AC-15: Signal pending mid-loop: -EINTR returned; mmap_lock dropped.

### Architecture

```
bitflags FollFlags : u32 {
  WRITE = 1<<0, GET = 1<<1, DUMP = 1<<2, FORCE = 1<<3,
  NOWAIT = 1<<4, NOFAULT = 1<<5, HWPOISON = 1<<6, ANON = 1<<7,
  LONGTERM = 1<<8, SPLIT_PMD = 1<<9, PCI_P2PDMA = 1<<10,
  INTERRUPTIBLE = 1<<11, HONOR_NUMA_FAULT = 1<<12,
  TOUCH = 1<<16, TRIED = 1<<17, REMOTE = 1<<18,
  PIN = 1<<19, FAST_ONLY = 1<<20, UNLOCKABLE = 1<<21,
  MADV_POPULATE = 1<<22,
}

const GUP_PIN_COUNTING_BIAS: u32 = 1024;
```

`Gup::get_user_pages(start, nr, flags, pages) -> i64`:
1. let mut locked = 1.
2. if !Gup::validate_args(pages, None, &mut flags, FollFlags::TOUCH): return -EINVAL.
3. return Gup::walk_locked(current().mm, start, nr, pages, &mut locked, flags).

`Gup::get_user_pages_fast(start, nr, flags, pages) -> i32`:
1. if !Gup::validate_args(pages, None, &mut flags, FollFlags::GET): return -EINVAL.
2. return Gup::fast_fallback(start, nr, flags, pages).

`Gup::pin_user_pages(start, nr, flags, pages) -> i64`:
1. let mut locked = 1.
2. if !Gup::validate_args(pages, None, &mut flags, FollFlags::PIN): return 0.
3. return Gup::longterm_locked(current().mm, start, nr, pages, &mut locked, flags).

`Gup::walk_locked(mm, start, nr, pages, locked, flags) -> i64`:
1. if nr == 0: return 0.
2. let must_unlock = false.
3. if !*locked: mmap_read_lock_killable(mm)?; must_unlock = true; *locked = 1.
4. else: mmap_assert_locked(mm).
5. if flags.contains(PIN): Gup::mm_set_has_pinned(mm).
6. if pages.is_some() ∧ !flags.contains(PIN): flags |= GET.
7. let pages_done: i64 = 0.
8. loop:
   - ret = Gup::walk_inner(mm, start, nr, flags, pages, locked).
   - if !flags.contains(UNLOCKABLE): pages_done = ret; break.
   - if ret > 0: nr -= ret; pages_done += ret; if nr == 0: break.
   - if *locked != 0: if pages_done == 0: pages_done = ret; break.
   - // VM_FAULT_RETRY path: lock dropped; retry 1 page with FOLL_TRIED.
   - pages += ret; start += (ret as usize) << PAGE_SHIFT; must_unlock = true.
   - retry: if Gup::signal_pending(flags): if pages_done == 0: pages_done = -EINTR; break.
   - mmap_read_lock_killable(mm)?.
   - *locked = 1.
   - ret = Gup::walk_inner(mm, start, 1, flags | TRIED, pages, locked).
   - if *locked == 0: VM_WARN_ON_ONCE(ret != 0); goto retry.
   - if ret != 1: if pages_done == 0: pages_done = ret; break.
   - nr -= 1; pages_done += 1; if nr == 0: break.
   - pages += 1; start += PAGE_SIZE.
9. if must_unlock ∧ *locked != 0: mmap_read_unlock(mm); *locked = 0.
10. if pages_done == 0 ∧ !flags.contains(NOWAIT): WARN_ON_ONCE; return -EFAULT.
11. return pages_done.

`Gup::walk_inner(mm, start, nr, flags, pages, locked) -> i64`:
1. start = untagged_addr_remote(mm, start).
2. let mut vma: Option<&Vma> = None.
3. let mut page_mask: u64 = 0.
4. let mut i: i64 = 0.
5. loop while nr > 0:
   - if vma.is_none() ∨ start >= vma.unwrap().vm_end:
     - vma = Gup::vma_lookup(mm, start).
     - if vma.is_none():
       - if Gup::in_gate_area(mm, start): handle gate-area path → next_page.
       - else: return -EFAULT.
     - Gup::check_vma_flags(vma.unwrap(), flags)?.
   - retry: if fatal_signal_pending(current()): return -EINTR.
   - cond_resched().
   - page = Gup::follow_page(vma.unwrap(), start, flags, &mut page_mask).
   - match page:
     - Ok(None) | Err(-EMLINK):
       - r = Gup::faultin_page(vma.unwrap(), start, flags, page == Err(-EMLINK), locked).
       - 0 ⟹ goto retry.
       - -EBUSY|-EAGAIN ⟹ r = 0; fallthrough.
       - -EFAULT|-ENOMEM|-EHWPOISON ⟹ return r.
       - else ⟹ BUG.
     - Err(-EEXIST) ∧ pages.is_some(): return -EEXIST.
     - Err(e): return e.
     - Ok(Some(p)): proceed.
   - next_page:
     - page_increm = 1 + (!(start >> PAGE_SHIFT) & page_mask) min(nr).
     - if pages.is_some(): fill pages[i..i+page_increm] with subpages (large folio).
     - i += page_increm; start += page_increm * PAGE_SIZE; nr -= page_increm.
6. return i.

`Gup::follow_page(vma, addr, flags, page_mask) -> Result<Option<*Page>, i32>`:
1. pgd, p4d, pud descent.
2. pmd path:
   - pmdval = pmdp_get_lockless(pmd).
   - if !pmd_present: no_page_table.
   - if !pmd_leaf: follow_page_pte(vma, addr, pmd, flags).
   - leaf (THP/PUD-huge): leaf-spinlock; try_grab_folio of compound head; set page_mask = HPAGE_PMD_NR-1.
3. follow_page_pte:
   - pte_offset_map_lock; pte = ptep_get.
   - !pte_present ⟹ no_page.
   - pte_protnone ∧ !gup_can_follow_protnone ⟹ no_page.
   - page = vm_normal_page.
   - FOLL_WRITE ∧ !can_follow_write_pte ⟹ Ok(None).
   - !page: zero-pfn → page = pte_page; else follow_pfn_pte.
   - !pte_write ∧ gup_must_unshare ⟹ Err(-EMLINK).
   - try_grab_folio(folio, 1, flags)?.
   - FOLL_PIN ⟹ arch_make_folio_accessible.
   - FOLL_TOUCH ⟹ mark dirty / accessed.
   - return Ok(Some(page)).

`Gup::faultin_page(vma, addr, flags, unshare, locked) -> i32`:
1. fault_flags = 0.
2. if FOLL_WRITE: fault_flags |= FAULT_FLAG_WRITE.
3. if FOLL_REMOTE: fault_flags |= FAULT_FLAG_REMOTE.
4. if unshare: fault_flags |= FAULT_FLAG_UNSHARE.
5. if FOLL_NOWAIT: fault_flags |= FAULT_FLAG_RETRY_NOWAIT.
6. if FOLL_TRIED: fault_flags |= FAULT_FLAG_TRIED.
7. if FOLL_INTERRUPTIBLE: fault_flags |= FAULT_FLAG_INTERRUPTIBLE.
8. if FOLL_UNLOCKABLE: fault_flags |= FAULT_FLAG_ALLOW_RETRY.
9. ret = handle_mm_fault(vma, addr, fault_flags, current()).
10. if ret & VM_FAULT_RETRY: *locked = 0; return 0.
11. if ret & VM_FAULT_ERROR: map VM_FAULT_OOM→-ENOMEM, VM_FAULT_HWPOISON→ -EHWPOISON ∨ -EFAULT, VM_FAULT_SIGSEGV→-EFAULT, etc.
12. return 0.

`Gup::try_grab_folio(folio, refs, flags) -> i32`:
1. if folio_ref_count(folio) <= 0: return -ENOMEM.
2. if !flags.PCI_P2PDMA ∧ folio_is_pci_p2pdma(folio): return -EREMOTEIO.
3. if flags.GET: folio_ref_add(folio, refs).
4. else if flags.PIN:
   - if is_zero_folio(folio): return 0.
   - if folio_has_pincount(folio):
     - folio_ref_add(folio, refs).
     - atomic_add(refs, &folio._pincount).
   - else:
     - folio_ref_add(folio, refs * GUP_PIN_COUNTING_BIAS).
   - node_stat_mod_folio(folio, NR_FOLL_PIN_ACQUIRED, refs).
5. return 0.

`Gup::fast_walk(start, end, flags, pages) -> u64 (nr_pinned)`:
1. if !CONFIG_HAVE_GUP_FAST ∨ !gup_fast_permitted(start, end): return 0.
2. if flags.PIN:
   - if !raw_seqcount_try_begin(&current().mm.write_protect_seq, seq): return 0.
3. local_irq_save(saved).
4. gup_fast_pgd_range(start, end, flags, pages, &mut nr_pinned).
5. local_irq_restore(saved).
6. if flags.PIN:
   - if read_seqcount_retry(&current().mm.write_protect_seq, seq):
     - Gup::fast_unpin(pages, nr_pinned); return 0.
   - else: Gup::sanity_check_pinned(pages, nr_pinned).
7. return nr_pinned.

`Gup::fast_fallback(start, nr, flags, pages) -> i32`:
1. validate flags subset: {WRITE|LONGTERM|FORCE|PIN|GET|FAST_ONLY|NOFAULT|PCI_P2PDMA|HONOR_NUMA_FAULT}.
2. if flags.PIN: Gup::mm_set_has_pinned(current().mm).
3. if !flags.FAST_ONLY: might_lock_read(&current().mm.mmap_lock).
4. start = untagged_addr(start) & PAGE_MASK; len = (nr as u64) << PAGE_SHIFT.
5. check_add_overflow(start, len, &end) → -EOVERFLOW.
6. if end > TASK_SIZE_MAX: -EFAULT.
7. nr_pinned = Gup::fast_walk(start, end, flags, pages).
8. if nr_pinned == nr ∨ flags.FAST_ONLY: return nr_pinned.
9. start += nr_pinned << PAGE_SHIFT; pages += nr_pinned.
10. ret = Gup::longterm_locked(current().mm, start, nr - nr_pinned, pages, &mut locked, flags | TOUCH | UNLOCKABLE).
11. if ret < 0: return nr_pinned ?: ret.
12. return ret + nr_pinned.

`Gup::longterm_locked(mm, start, nr, pages, locked, flags) -> i64`:
1. if !flags.LONGTERM: return Gup::walk_locked(mm, start, nr, pages, locked, flags).
2. loop:
   - ret = Gup::walk_locked(mm, start, nr, pages, locked, flags).
   - if ret <= 0: return ret.
   - if Gup::check_and_migrate_movable_folios(pages, ret) == 0: return ret.
   - // migrated; retry the whole range (pages are now in non-movable zones).

`Gup::unpin_user_page(page)`:
1. Gup::sanity_check_pinned(&[page]).
2. Gup::put_folio(page_folio(page), 1, FollFlags::PIN).

`Gup::put_folio(folio, refs, flags)`:
1. if flags.PIN:
   - if is_zero_folio(folio): return.
   - node_stat_mod_folio(folio, NR_FOLL_PIN_RELEASED, refs).
   - if folio_has_pincount(folio): atomic_sub(refs, &folio._pincount).
   - else: refs *= GUP_PIN_COUNTING_BIAS.
2. folio_put_refs(folio, refs).

### Out of Scope

- mm/hugetlb.c follow_hugetlb_page slow-path internals (covered in `hugetlb.md` Tier-3)
- mm/huge_memory.c THP follow / split details (covered in `thp.md` Tier-3)
- mm/memory.c handle_mm_fault (covered in `virtual-memory.md` Tier-3)
- mm/migrate.c folio_migrate machinery for FOLL_LONGTERM (covered in `migration-compaction.md` Tier-3)
- mm/mmap.c VMA tree (covered in `mmap.md` Tier-3)
- arch-specific gup_fast page-table-walk variants (per-arch Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `get_user_pages()` | per-call entry (current mm) | `Gup::get_user_pages` |
| `get_user_pages_remote()` | per-call entry (foreign mm) | `Gup::get_user_pages_remote` |
| `get_user_pages_unlocked()` | per-call entry (mmap_lock internal) | `Gup::get_user_pages_unlocked` |
| `get_user_pages_fast()` | per-call lockless+fallback | `Gup::get_user_pages_fast` |
| `get_user_pages_fast_only()` | per-call lockless-only | `Gup::get_user_pages_fast_only` |
| `pin_user_pages()` | per-call FOLL_PIN | `Gup::pin_user_pages` |
| `pin_user_pages_remote()` | per-call FOLL_PIN foreign mm | `Gup::pin_user_pages_remote` |
| `pin_user_pages_fast()` | per-call FOLL_PIN lockless+fallback | `Gup::pin_user_pages_fast` |
| `pin_user_pages_unlocked()` | per-call FOLL_PIN auto-lock | `Gup::pin_user_pages_unlocked` |
| `__get_user_pages()` | per-walk slow loop | `Gup::walk_inner` |
| `__get_user_pages_locked()` | per-call retry+lock-mgmt | `Gup::walk_locked` |
| `__gup_longterm_locked()` | per-FOLL_LONGTERM wrap | `Gup::longterm_locked` |
| `gup_fast()` / `gup_fast_pgd_range()` | per-fast-path PGD walk | `Gup::fast_walk` |
| `gup_fast_fallback()` | per-fast→slow fallback | `Gup::fast_fallback` |
| `follow_page_mask()` / `follow_page_pte()` | per-PTE follow | `Gup::follow_page` |
| `try_grab_folio()` | per-slow-path refcount/pincount inc | `Gup::try_grab_folio` |
| `try_grab_folio_fast()` | per-fast-path refcount/pincount inc | `Gup::try_grab_folio_fast` |
| `gup_put_folio()` | per-release | `Gup::put_folio` |
| `faultin_page()` | per-fault-in via handle_mm_fault | `Gup::faultin_page` |
| `check_vma_flags()` | per-VMA permission check | `Gup::check_vma_flags` |
| `gup_must_unshare()` | per-COW trigger | `Gup::must_unshare` |
| `sanity_check_pinned_pages()` | per-DEBUG verify | `Gup::sanity_check_pinned` |
| `unpin_user_page()` / `unpin_user_pages()` | per-release pin | `Gup::unpin_user_page(s)` |
| `unpin_user_pages_dirty_lock()` | per-release+dirty | `Gup::unpin_user_pages_dirty_lock` |
| `folio_add_pin()` | per-extra-pin on already-pinned folio | `Gup::folio_add_pin` |
| `mm_set_has_pinned_flag()` | per-mm MMF_HAS_PINNED set | `Gup::mm_set_has_pinned` |
| `is_valid_gup_args()` | per-call arg validation | `Gup::validate_args` |
| `gup_fast_folio_allowed()` | per-fast-path file-backed/secretmem reject | `Gup::fast_folio_allowed` |
| `FOLL_WRITE / FOLL_GET / FOLL_PIN / FOLL_LONGTERM / FOLL_FORCE / FOLL_TOUCH / FOLL_REMOTE / FOLL_UNLOCKABLE / FOLL_FAST_ONLY / FOLL_NOFAULT / FOLL_NOWAIT / FOLL_HWPOISON / FOLL_HONOR_NUMA_FAULT / FOLL_SPLIT_PMD / FOLL_PCI_P2PDMA / FOLL_INTERRUPTIBLE / FOLL_TRIED / FOLL_DUMP / FOLL_ANON / FOLL_MADV_POPULATE` | per-call flag set | `FollFlags` bitflags |
| `GUP_PIN_COUNTING_BIAS` | per-folio pin bias | constant |
| `MMF_HAS_PINNED` | per-mm pinned-flag | mm_struct flag |
| `mm->write_protect_seq` | per-mm seqcount for fork-vs-pin | `MmStruct::write_protect_seq` |

### compatibility contract

REQ-1: Per-call flag semantics:
- FOLL_GET: refcount +1 via `folio_ref_add`; release via `put_page` / `unpin_user_pages` is wrong for these.
- FOLL_PIN: refcount += GUP_PIN_COUNTING_BIAS (small folios) or refcount+1 ∧ `_pincount`+1 (large/folio_has_pincount); release ONLY via `unpin_user_page(s)`.
- FOLL_GET ∧ FOLL_PIN are mutually exclusive per call (VM_WARN_ON_ONCE enforced).
- FOLL_WRITE: write-intent; requires VM_WRITE ∨ FOLL_FORCE with VM_MAYWRITE.
- FOLL_FORCE: relax permission: read-of-PROT_NONE allowed; write-of-COW allowed if `is_cow_mapping(vm_flags)`.
- FOLL_LONGTERM: caller pins indefinitely; routed through `__gup_longterm_locked`; rejects fsdax VMAs, MOVABLE/CMA folios (migrated first), file-backed long-term writes (writeback-races).
- FOLL_TOUCH: mark folio accessed / dirty as if a user access happened.
- FOLL_REMOTE: target mm != current->mm; arch_vma_access_permitted called with foreign=true.
- FOLL_UNLOCKABLE: `mmap_lock` may be released on VM_FAULT_RETRY; caller observes `*locked == 0`.
- FOLL_FAST_ONLY: no fallback to slow path; abort with partial-count.
- FOLL_NOFAULT: never invoke `handle_mm_fault`.
- FOLL_NOWAIT: never wait for IO during fault.
- FOLL_HWPOISON: return -EHWPOISON instead of -EFAULT for hwpoisoned pages.
- FOLL_HONOR_NUMA_FAULT: PROT_NONE NUMA-hinting page treated as not-present.
- FOLL_SPLIT_PMD: split THP PMDs to PTEs before follow.
- FOLL_PCI_P2PDMA: allow p2pdma folios (else -EREMOTEIO).
- FOLL_INTERRUPTIBLE: signal-interruptible during fault.
- FOLL_TRIED: pass FAULT_FLAG_TRIED on fault retry.
- FOLL_DUMP: core-dump caller; skip special / zero pages.
- FOLL_ANON: reject non-anonymous VMAs.

REQ-2: `get_user_pages(start, nr, gup_flags, pages) -> long`:
- locked = 1.
- if !is_valid_gup_args(pages, NULL, &gup_flags, FOLL_TOUCH): return -EINVAL.
- return __get_user_pages_locked(current.mm, start, nr, pages, &locked, gup_flags).

REQ-3: `get_user_pages_remote(mm, start, nr, gup_flags, pages, locked) -> long`:
- if !is_valid_gup_args(pages, locked, &gup_flags, FOLL_TOUCH | FOLL_REMOTE): return -EINVAL.
- local_locked = 1.
- return __get_user_pages_locked(mm, start, nr, pages, locked ?: &local_locked, gup_flags).

REQ-4: `__get_user_pages_locked(mm, start, nr, pages, locked, flags) -> long`:
- if nr == 0: return 0.
- if !*locked: mmap_read_lock_killable(mm); must_unlock = true; *locked = 1.
- else: mmap_assert_locked(mm).
- if flags & FOLL_PIN: mm_set_has_pinned_flag(mm).
- if pages ∧ !(flags & FOLL_PIN): flags |= FOLL_GET.
- pages_done = 0.
- for-ever:
  - ret = __get_user_pages(mm, start, nr, flags, pages, locked).
  - if !(flags & FOLL_UNLOCKABLE): pages_done = ret; break.
  - if ret > 0: nr -= ret; pages_done += ret; if !nr: break.
  - if *locked: if !pages_done: pages_done = ret; break.
  - // VM_FAULT_RETRY case: lock dropped; retry one page.
  - pages += ret; start += ret << PAGE_SHIFT; must_unlock = true.
  - retry: if gup_signal_pending(flags): pages_done = -EINTR; break.
  - mmap_read_lock_killable; *locked = 1.
  - ret = __get_user_pages(mm, start, 1, flags | FOLL_TRIED, pages, locked).
  - if !*locked: VM_WARN_ON_ONCE(ret != 0); goto retry.
  - if ret != 1: if !pages_done: pages_done = ret; break.
  - nr -= 1; pages_done += 1; if !nr: break.
  - pages += 1; start += PAGE_SIZE.
- if must_unlock ∧ *locked: mmap_read_unlock(mm); *locked = 0.
- return pages_done.

REQ-5: `__get_user_pages(mm, start, nr, gup_flags, pages, locked) -> long`:
- start = untagged_addr_remote(mm, start).
- VM_WARN_ON_ONCE !!pages != !!(gup_flags & (FOLL_GET | FOLL_PIN)).
- do:
  - if first iter ∨ start >= vma.vm_end: vma = gup_vma_lookup(mm, start).
  - if !vma ∧ in_gate_area: get_gate_page; goto next_page.
  - if !vma: return -EFAULT.
  - if check_vma_flags(vma, gup_flags) ≠ 0: return -EINVAL.
  - retry: if fatal_signal_pending(current): return -EINTR.
  - cond_resched.
  - page = follow_page_mask(vma, start, gup_flags, &page_mask).
  - if !page ∨ PTR_ERR(page) == -EMLINK:
    - ret = faultin_page(vma, start, gup_flags, /*unshare=*/ -EMLINK, locked).
    - 0 → goto retry; -EBUSY/-EAGAIN → ret=0,fallthrough; -EFAULT/-ENOMEM/-EHWPOISON → return.
  - else if PTR_ERR(page) == -EEXIST ∧ pages: return -EEXIST.
  - else if IS_ERR(page): return PTR_ERR(page).
  - next_page: page_increm = 1 + (~(start >> PAGE_SHIFT) & page_mask).
  - if pages: fill pages[i..i+page_increm] (subpages of large folio).
  - i += page_increm; start += page_increm * PAGE_SIZE; nr -= page_increm.
- while nr > 0.
- return i.

REQ-6: `check_vma_flags(vma, gup_flags)`:
- if vm_flags & (VM_IO | VM_PFNMAP): return -EFAULT.
- if FOLL_ANON ∧ !vma_is_anonymous(vma): return -EFAULT.
- if FOLL_LONGTERM ∧ vma_is_fsdax(vma): return -EOPNOTSUPP.
- if FOLL_SPLIT_PMD ∧ is_vm_hugetlb_page(vma): return -EOPNOTSUPP.
- if vma_is_secretmem(vma): return -EFAULT.
- write case:
  - if !vma_anon ∧ !writable_file_mapping_allowed(vma, gup_flags): return -EFAULT.
  - if !VM_WRITE ∨ VM_SHADOW_STACK:
    - if !FOLL_FORCE: return -EFAULT.
    - if !is_cow_mapping(vm_flags): return -EFAULT.
- read case:
  - if !VM_READ ∧ !FOLL_FORCE: return -EFAULT.
  - if !VM_MAYREAD: return -EFAULT.
- if !arch_vma_access_permitted(vma, write, /*execute=*/false, foreign=FOLL_REMOTE): return -EFAULT.
- return 0.

REQ-7: `follow_page_pte(vma, address, pmd, flags)`:
- lock pte; if !pte_present: no_page.
- if pte_protnone ∧ !gup_can_follow_protnone: no_page.
- page = vm_normal_page(vma, address, pte).
- if FOLL_WRITE ∧ !can_follow_write_pte: page = NULL; out.
- if !page ∧ FOLL_DUMP: -EFAULT.
- if !page ∧ is_zero_pfn: page = pte_page(pte) else: follow_pfn_pte path.
- if !pte_write(pte) ∧ gup_must_unshare(vma, flags, page): return -EMLINK (trigger COW unshare).
- try_grab_folio(folio, 1, flags) — applies FOLL_GET or FOLL_PIN bias.
- if FOLL_PIN: arch_make_folio_accessible (s390 SE pages).
- if FOLL_TOUCH: folio_mark_dirty (if WRITE ∧ !pte_dirty); folio_mark_accessed.
- return page.

REQ-8: `gup_must_unshare(vma, flags, page)`:
- Only triggers when FOLL_PIN ∧ vma is COW-mapping ∧ page is anon ∧ !PageAnonExclusive.
- Forces COW break so the pin is never on a shared-with-fork anon page; required for fork-vs-pin correctness (CVE-2020-29368 family).

REQ-9: `try_grab_folio(folio, refs, flags) -> 0 | -ENOMEM | -EREMOTEIO`:
- if folio_ref_count(folio) <= 0: -ENOMEM.
- if !FOLL_PCI_P2PDMA ∧ folio_is_pci_p2pdma(folio): -EREMOTEIO.
- if FOLL_GET: folio_ref_add(folio, refs).
- if FOLL_PIN:
  - if is_zero_folio: return 0 (zero page never pinned).
  - if folio_has_pincount: folio_ref_add(refs); atomic_add(refs, &_pincount).
  - else: folio_ref_add(refs * GUP_PIN_COUNTING_BIAS).
  - node_stat_mod_folio(NR_FOLL_PIN_ACQUIRED, refs).

REQ-10: `gup_put_folio(folio, refs, flags)`:
- if FOLL_PIN:
  - if is_zero_folio: return.
  - node_stat_mod_folio(NR_FOLL_PIN_RELEASED, refs).
  - if folio_has_pincount: atomic_sub(refs, &_pincount).
  - else: refs *= GUP_PIN_COUNTING_BIAS.
- folio_put_refs(folio, refs).

REQ-11: `gup_fast(start, end, gup_flags, pages) -> nr_pinned`:
- if !CONFIG_HAVE_GUP_FAST ∨ !gup_fast_permitted: return 0.
- if FOLL_PIN: raw_seqcount_try_begin(&mm->write_protect_seq, seq) ∨ return 0.
- local_irq_save(flags).
- gup_fast_pgd_range(start, end, gup_flags, pages, &nr_pinned).
- local_irq_restore(flags).
- if FOLL_PIN ∧ read_seqcount_retry(&mm->write_protect_seq, seq):
  - gup_fast_unpin_user_pages(pages, nr_pinned); return 0.
- else: sanity_check_pinned_pages.
- return nr_pinned.

REQ-12: `gup_fast_fallback(start, nr, gup_flags, pages)`:
- WARN_ON_ONCE if flags ∉ {WRITE|LONGTERM|FORCE|PIN|GET|FAST_ONLY|NOFAULT|PCI_P2PDMA|HONOR_NUMA_FAULT}.
- if FOLL_PIN: mm_set_has_pinned_flag(current.mm).
- if !FOLL_FAST_ONLY: might_lock_read(&mm->mmap_lock).
- start = untagged_addr(start) & PAGE_MASK; len = nr << PAGE_SHIFT.
- check_add_overflow(start, len, &end) ⟹ -EOVERFLOW; end > TASK_SIZE_MAX ⟹ -EFAULT.
- nr_pinned = gup_fast(start, end, gup_flags, pages).
- if nr_pinned == nr ∨ FOLL_FAST_ONLY: return nr_pinned.
- /* Slow fallback */
- ret = __gup_longterm_locked(mm, start+nr_pinned*PAGE_SIZE, nr-nr_pinned, pages+nr_pinned, &locked, gup_flags | FOLL_TOUCH | FOLL_UNLOCKABLE).
- if ret < 0: return nr_pinned ?: ret.
- return ret + nr_pinned.

REQ-13: `__gup_longterm_locked(mm, start, nr, pages, locked, flags)`:
- if !(flags & FOLL_LONGTERM): plain __get_user_pages_locked.
- Otherwise: loop:
  - ret = __get_user_pages_locked.
  - if ret < 0: return.
  - migrated = check-and-migrate-movable-folios(pages, ret).
  - if !migrated: return ret.
  - // some pages were in MOVABLE/CMA; migrated; retry the same range.

REQ-14: `unpin_user_page(page)`:
- sanity_check_pinned_pages(&page, 1).
- gup_put_folio(page_folio(page), 1, FOLL_PIN).

REQ-15: `unpin_user_pages_dirty_lock(pages, npages, make_dirty)`:
- if !make_dirty: unpin_user_pages(pages, npages); return.
- for i in 0..npages step nr:
  - folio = gup_folio_next(pages, npages, i, &nr).
  - if !folio_test_dirty(folio): folio_lock; set_page_dirty(folio); folio_unlock.
  - gup_put_folio(folio, nr, FOLL_PIN).

REQ-16: Per-mm pinned tracking:
- First FOLL_PIN call on mm sets MMF_HAS_PINNED (never cleared).
- fork() consults MMF_HAS_PINNED → must page-copy-on-fork (not COW-share) anon pages with a pin, to prevent the GUP-fork race.

REQ-17: Per-fast-path concurrency:
- `local_irq_save` blocks RCU-table-free / IPI-driven TLB flush for x86 / archs that broadcast IPIs.
- For archs with MMU_GATHER_RCU_TABLE_FREE: free of page-table pages is rcu_sched-deferred; irq-disable inhibits the callback.
- Per-FOLL_PIN: write_protect_seq retry catches fork()'s `copy_page_range` write-protect happening mid-walk.

REQ-18: Per-hugetlb / THP:
- `follow_pmd_mask` / `follow_pud_mask` handle leaf hugepages: under pmd_lock / pud_lock, try_grab_folio of compound head.
- `gup_fast_pmd_leaf` / `gup_fast_pud_leaf` handle leaf in fast path.
- hugetlb: follow_hugetlb_page or follow_huge_pgd/p4d/pud/pmd dispatch.
- FOLL_SPLIT_PMD on THP: split-and-retry.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `foll_get_pin_exclusive` | INVARIANT | per-call: GET ∧ PIN never both set after validate_args. |
| `pages_array_consistent_with_flag` | INVARIANT | per-call: pages.is_some() ⟺ (GET ∨ PIN). |
| `refcount_balanced_on_partial_failure` | INVARIANT | per-walk: nr_pinned pages get'd/pin'd ⟹ caller responsible; on -errno with nr_pinned > 0, return nr_pinned not -errno. |
| `gup_must_unshare_triggers_on_write_cow` | INVARIANT | per-follow_page_pte: FOLL_PIN ∧ !pte_write ∧ anon-shared ⟹ -EMLINK. |
| `fast_walk_irq_disabled` | INVARIANT | per-fast_walk: local_irq_save held across pgd-walk. |
| `fast_pin_seqcount_retry` | INVARIANT | per-FOLL_PIN fast: write_protect_seq retry ⟹ unpin + return 0. |
| `vm_io_pfnmap_rejected` | INVARIANT | per-check_vma_flags: VM_IO ∨ VM_PFNMAP ⟹ -EFAULT. |

### Layer 2: TLA+

`mm/gup.tla`:
- Per-call entry + per-walk + per-fault + per-fast-walk + per-fork-write-protect race.
- Properties:
  - `safety_no_pin_on_zero_page` — per-pin: zero folio refcount never modified.
  - `safety_pin_implies_mmf_has_pinned` — per-FOLL_PIN: MMF_HAS_PINNED set on mm before refcount touched.
  - `safety_fast_pin_seq_consistent` — per-fast FOLL_PIN: ¬(race-with-fork ∧ kept-pin); either unpinned-on-retry or fork-saw-pin.
  - `safety_check_vma_flags_precedes_follow` — per-walk: check_vma_flags called before any follow_page_pte on a new vma.
  - `liveness_per_walk_terminates` — per-walk_locked: nr decreases or fatal signal pending.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Gup::walk_inner` post: ret > 0 ⟹ ret ≤ nr ∧ pages[0..ret] populated | `Gup::walk_inner` |
| `Gup::try_grab_folio` post: GET ⟹ folio.refcount += refs; PIN small ⟹ refcount += refs*BIAS; PIN large ⟹ refcount += refs ∧ pincount += refs | `Gup::try_grab_folio` |
| `Gup::put_folio` post: inverse of try_grab_folio (per FOLL_*) | `Gup::put_folio` |
| `Gup::check_vma_flags` post: returns 0 ⟹ vm_flags permit the requested access mode | `Gup::check_vma_flags` |
| `Gup::fast_walk` post: nr_pinned ≤ (end-start)/PAGE_SIZE; FOLL_PIN ⟹ seq-stable | `Gup::fast_walk` |
| `Gup::longterm_locked` post: returned pages.zone ∉ {MOVABLE, CMA} ∧ vma.is_fsdax == false | `Gup::longterm_locked` |
| `Gup::faultin_page` post: VM_FAULT_RETRY ⟹ *locked == 0 | `Gup::faultin_page` |

### Layer 4: Verus/Creusot functional

`Per-call → validate_args → (fast-path: seqcount-protected pgd walk under irq-disabled) → (slow-path: mmap_lock_read; for each addr: vma_lookup; check_vma_flags; follow_page_mask; on miss: handle_mm_fault retry; on EMLINK: unshare-fault; try_grab_folio (GET-add OR PIN-bias OR PIN-pincount)) → on FOLL_LONGTERM: migrate-out-of-MOVABLE/CMA → return nr_pinned` semantic equivalence: per-Documentation/core-api/pin_user_pages.rst + per-Linux mm/gup.c.

### hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening.)

GUP reinforcement:

- **Per-FOLL_PIN gup_must_unshare on COW** — defense against per-fork-vs-pin (CVE-2020-29368) anon-share-then-pin bug.
- **Per-mm write_protect_seq retry in fast-path** — defense against per-fork-window pinning a page that fork() then COW-protects.
- **Per-MMF_HAS_PINNED sticky** — defense against per-fork COW-merging anon page that has an outstanding pin.
- **Per-fsdax FOLL_LONGTERM rejected** — defense against per-DAX writeback-vs-pin filesystem corruption.
- **Per-MOVABLE/CMA migration before LONGTERM pin** — defense against per-memory-hotplug failure due to immovable pin.
- **Per-file-backed long-term-write rejected** — defense against per-page-cache writeback-vs-pin (writable_file_mapping_allowed gate).
- **Per-VM_IO / VM_PFNMAP rejection** — defense against per-driver-MMIO page-as-page misuse.
- **Per-secretmem VMA rejection** — defense against per-secretmem disclosure via GUP.
- **Per-PCI-P2PDMA opt-in only** — defense against per-p2pdma host-DMA pinning misuse.
- **Per-FOLL_GET ∧ FOLL_PIN mutual exclusion (VM_WARN_ON_ONCE)** — defense against per-caller refcount/pincount mismatch.
- **Per-zero-folio never pinned** — defense against per-zero-page refcount inflation.
- **Per-fast-walk irq-disabled** — defense against per-RCU-table-free / per-TLB-IPI race on page-table pages.
- **Per-FOLL_HONOR_NUMA_FAULT respect** — defense against per-NUMA-hinting-fault-while-pinning correctness drift.

