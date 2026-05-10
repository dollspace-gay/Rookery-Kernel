# Tier-3: mm/madvise.c — madvise(2) and process_madvise(2)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: mm/00-overview.md
upstream-paths:
  - mm/madvise.c (~2248 lines)
  - include/linux/mm.h (vm_flags VM_DONTCOPY, VM_DONTDUMP, VM_WIPEONFORK, VM_LOCKED, VM_RAND_READ, VM_SEQ_READ)
  - include/uapi/asm-generic/mman-common.h (MADV_* constants)
  - Documentation/admin-guide/mm/madvise.rst
-->

## Summary

The **madvise(2)** syscall lets userspace give the kernel hints (or destructive requests) about page-cache and anonymous-memory usage over a range `[start, start+len_in)`. Per-call: validated by `is_valid_madvise` (page-aligned start, non-zero rounded len, no wrap, behavior in the known set), locked via `madvise_lock` (NO_LOCK for memory-failure injection, MMAP_READ for read-only walks like WILLNEED/COLD/PAGEOUT/COLLAPSE, VMA_READ for DONTNEED/FREE/GUARD_*, MMAP_WRITE for VMA-mutating advice like DONTFORK/HUGEPAGE/MERGEABLE/anon-vma-name), then dispatched per-VMA through `madvise_walk_vmas` to `madvise_vma_behavior`. Per-behavior the kernel either (i) updates `vma->vm_flags` and re-splits/merges VMAs via `madvise_update_vma`, (ii) walks PTEs to install/clear page state (cold/pageout/free/guard markers, swap-in), or (iii) destructively unmaps via `madvise_remove` / `madvise_dontneed_free`. The dual syscall **process_madvise(2)** lets a privileged peer (CAP_SYS_NICE + PTRACE_MODE_READ) push hints into another process's address space over an iovec, used by Android-style memory managers. Critical for: garbage-collected runtimes (MADV_FREE), THP coalescing (MADV_COLLAPSE), userland page-out (Android lmkd, oomd), KSM dedup opt-in/out, fork copy-on-write avoidance (DONTFORK), and guard-page allocators (GUARD_INSTALL).

This Tier-3 covers `mm/madvise.c` (~2248 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct madvise_behavior` | per-call context | `MadviseBehavior` |
| `struct madvise_behavior_range` | per-VMA range slice | `MadviseRange` |
| `struct madvise_walk_private` | per-walk private (pageout tlb) | `MadviseWalkPrivate` |
| `enum madvise_lock_mode` | per-call lock mode | `MadviseLockMode` |
| `do_madvise()` | per-mm dispatch | `Madvise::do_madvise` |
| `SYSCALL_DEFINE3(madvise, ...)` | per-syscall entry | `sys::madvise` |
| `SYSCALL_DEFINE5(process_madvise, ...)` | per-remote-syscall entry | `sys::process_madvise` |
| `vector_madvise()` | per-iovec loop | `Madvise::vector_madvise` |
| `madvise_lock()` / `madvise_unlock()` | per-lock acquire/release | `Madvise::lock` / `unlock` |
| `get_lock_mode()` | per-behavior lock policy | `Madvise::get_lock_mode` |
| `madvise_walk_vmas()` | per-VMA traversal | `Madvise::walk_vmas` |
| `madvise_vma_behavior()` | per-VMA dispatch | `Madvise::vma_behavior` |
| `madvise_update_vma()` | per-VMA flag-update + split/merge | `Madvise::update_vma` |
| `madvise_willneed()` | per-prefault / readahead | `Madvise::willneed` |
| `madvise_cold()` | per-deactivate to inactive-LRU | `Madvise::cold` |
| `madvise_pageout()` | per-force-pageout | `Madvise::pageout` |
| `madvise_cold_or_pageout_pte_range()` | per-PTE cold/pageout walker | `Madvise::cold_or_pageout_pte_range` |
| `madvise_dontneed_free()` | per-DONTNEED / FREE | `Madvise::dontneed_free` |
| `madvise_dontneed_single_vma()` | per-DONTNEED inner | `Madvise::dontneed_single_vma` |
| `madvise_free_single_vma()` | per-FREE inner | `Madvise::free_single_vma` |
| `madvise_free_pte_range()` | per-PTE FREE walker | `Madvise::free_pte_range` |
| `madvise_remove()` | per-REMOVE (file hole punch) | `Madvise::remove` |
| `madvise_populate()` | per-POPULATE_READ/WRITE | `Madvise::populate` |
| `madvise_collapse()` | per-COLLAPSE (THP coalesce) | `Madvise::collapse` |
| `madvise_guard_install()` | per-GUARD_INSTALL (poison PTE marker) | `Madvise::guard_install` |
| `madvise_guard_remove()` | per-GUARD_REMOVE (clear marker) | `Madvise::guard_remove` |
| `madvise_inject_error()` | per-HWPOISON / SOFT_OFFLINE | `Madvise::inject_error` |
| `madvise_set_anon_name()` / `set_anon_vma_name()` | per-PR_SET_VMA_ANON_NAME | `Madvise::set_anon_name` |
| `anon_vma_name_alloc()` / `anon_vma_name_free()` | per-named-anon-VMA refcount | `Madvise::anon_name_alloc` / `anon_name_free` |
| `swapin_walk_pmd_entry()` / `shmem_swapin_range()` | per-WILLNEED swap-in | `Madvise::swapin_pmd` / `shmem_swapin_range` |
| `is_valid_madvise()` / `madvise_should_skip()` | per-arg-validation | `Madvise::is_valid` / `should_skip` |
| `can_madvise_modify()` | per-VMA-modification permission (sealing) | `Madvise::can_modify` |
| `madvise_batch_tlb_flush()` | per-batch-TLB policy | `Madvise::batch_tlb_flush` |
| `madvise_init_tlb()` / `madvise_finish_tlb()` | per-batched-mmu-gather | `Madvise::init_tlb` / `finish_tlb` |
| `is_guard_pte_marker()` / `is_valid_guard_vma()` | per-GUARD predicate | `Madvise::is_guard_pte_marker` / `is_valid_guard_vma` |
| `process_madvise_remote_valid()` | per-remote-only behavior allow-list | `Madvise::remote_valid` |

## Compatibility contract

REQ-1: struct madvise_behavior (per-call context):
- mm: *mm_struct — target address-space (current->mm or remote via process_madvise).
- behavior: i32 — MADV_*.
- tlb: *mmu_gather — batched TLB / page-table teardown.
- lock_mode: MadviseLockMode — NO_LOCK / MMAP_READ_LOCK / MMAP_WRITE_LOCK / VMA_READ_LOCK.
- anon_name: *anon_vma_name (for __MADV_SET_ANON_VMA_NAME).
- range: MadviseRange { start, end } — current sub-range being applied.
- prev: *vm_area_struct (previous VMA, for merging).
- vma: *vm_area_struct (current target).
- lock_dropped: bool (set when COLLAPSE temporarily drops mmap_lock).

REQ-2: do_madvise(mm, start, len_in, behavior):
- madv_behavior = { mm, behavior, tlb: &tlb }.
- if madvise_should_skip(start, len_in, behavior, &error): return error /* may be 0 for no-op zero-length */.
- if madvise_lock(&madv_behavior) ≠ 0: return error /* -EINTR if killable */.
- madvise_init_tlb(&madv_behavior).
- error = madvise_do_behavior(start, len_in, &madv_behavior).
- madvise_finish_tlb(&madv_behavior).
- madvise_unlock(&madv_behavior).
- return error.

REQ-3: is_valid_madvise / madvise_should_skip:
- valid ⟺ madvise_behavior_valid(behavior) ∧ PAGE_ALIGNED(start) ∧ (len_in == 0 ∨ PAGE_ALIGN(len_in) ≠ 0) ∧ ¬overflow(start + len).
- skip true ⟹ invalid (err = -EINVAL) OR zero-length (err = 0).

REQ-4: get_lock_mode(behavior):
- is_memory_failure ⟹ NO_LOCK.
- {REMOVE, WILLNEED, COLD, PAGEOUT, POPULATE_READ, POPULATE_WRITE, COLLAPSE} ⟹ MMAP_READ_LOCK.
- {GUARD_INSTALL, GUARD_REMOVE, DONTNEED, DONTNEED_LOCKED, FREE} ⟹ VMA_READ_LOCK (per-VMA read lock, acquired in walk).
- All others (NORMAL, RANDOM, SEQUENTIAL, DONTFORK, DOFORK, WIPEONFORK, KEEPONFORK, DONTDUMP, DODUMP, MERGEABLE, UNMERGEABLE, HUGEPAGE, NOHUGEPAGE, __MADV_SET_ANON_VMA_NAME) ⟹ MMAP_WRITE_LOCK.

REQ-5: madvise_lock(behavior) -> error:
- NO_LOCK: nothing.
- MMAP_WRITE_LOCK: mmap_write_lock_killable(mm); -EINTR on signal.
- MMAP_READ_LOCK: mmap_read_lock(mm).
- VMA_READ_LOCK: per-VMA read lock inside walk.

REQ-6: madvise_walk_vmas:
- Iterates VMAs intersecting [range.start, range.end).
- For VMA_READ_LOCK mode: try_vma_read_lock per-VMA; if fails, drop to mmap_read_lock fallback.
- For each VMA: clamp range to VMA bounds, set madv_behavior->vma + range, call madvise_vma_behavior; on success advance to next VMA, on -ENOMEM (no mapping in gap) return -ENOMEM.

REQ-7: madvise_vma_behavior dispatch:
- !can_madvise_modify ⟹ -EPERM (mseal protection).
- MADV_REMOVE → madvise_remove.
- MADV_WILLNEED → madvise_willneed.
- MADV_COLD → madvise_cold.
- MADV_PAGEOUT → madvise_pageout.
- MADV_FREE / MADV_DONTNEED / MADV_DONTNEED_LOCKED → madvise_dontneed_free.
- MADV_COLLAPSE → madvise_collapse(vma, start, end, &lock_dropped).
- MADV_GUARD_INSTALL → madvise_guard_install.
- MADV_GUARD_REMOVE → madvise_guard_remove.
- VM_flags-update group → set new_flags then madvise_update_vma:
  - MADV_NORMAL: clear VM_RAND_READ | VM_SEQ_READ.
  - MADV_SEQUENTIAL: set VM_SEQ_READ, clear VM_RAND_READ.
  - MADV_RANDOM: set VM_RAND_READ, clear VM_SEQ_READ.
  - MADV_DONTFORK: set VM_DONTCOPY.
  - MADV_DOFORK: clear VM_DONTCOPY (reject VM_SPECIAL with -EINVAL).
  - MADV_WIPEONFORK: set VM_WIPEONFORK (anon-only; reject VM_SHARED or file-backed with -EINVAL).
  - MADV_KEEPONFORK: clear VM_WIPEONFORK (reject VM_DROPPABLE with -EINVAL).
  - MADV_DONTDUMP: set VM_DONTDUMP.
  - MADV_DODUMP: clear VM_DONTDUMP (reject VM_SPECIAL non-hugetlb / VM_DROPPABLE with -EINVAL).
  - MADV_MERGEABLE / MADV_UNMERGEABLE: delegate to ksm_madvise to compute new_flags.
  - MADV_HUGEPAGE / MADV_NOHUGEPAGE: delegate to hugepage_madvise.
  - __MADV_SET_ANON_VMA_NAME: anon or anon-shmem only (reject other file-backed with -EBADF).
- On -ENOMEM remap to -EAGAIN.

REQ-8: madvise_update_vma(new_flags, madv_behavior):
- vma_modify(prev, vma, range.start, range.end, new_flags, anon_name) — splits / merges VMAs as needed.
- Updates madv_behavior->prev to point past the modified range for next iteration.

REQ-9: madvise_willneed:
- File-backed: vfs readahead force_page_cache_readahead(file, mapping, start_off, end_off).
- Anonymous: walk PMDs (swapin_walk_ops) and read swap entries into pages via swapin_readahead.
- Shmem: shmem_swapin_range loops over xa_for_each_range and shmem_swapin to in-core.
- Sets mark_mmap_lock_dropped if mmap_lock was dropped during I/O.

REQ-10: madvise_cold:
- Walks PTEs via cold_walk_ops / madvise_cold_or_pageout_pte_range with pageout = false.
- For each present, mapped, non-pinned, non-VM_LOCKED, non-VM_SPECIAL folio: deactivate_folio (moves from active LRU to inactive LRU).
- No I/O issued; reclaim happens later under memory pressure.

REQ-11: madvise_pageout:
- can_do_file_pageout(vma) ⟹ file-backed must be writable+regular file; else skip.
- Walks PTEs identically to cold; but issues reclaim_pages on the gathered folio list (synchronous force-reclaim).
- Uses madv_behavior->tlb for batched TLB flush.

REQ-12: madvise_dontneed_free (DONTNEED / FREE):
- madvise_dontneed_free_valid_vma rejects VM_LOCKED, VM_PFNMAP, VM_HUGETLB (DONTNEED-only).
- DONTNEED: unmap pages via zap_page_range_single using madv_behavior->tlb; subsequent fault repopulates from backing store (or zero-fill for anon).
- FREE: walk PTEs via madvise_free_pte_range — for each present anonymous folio, mark folio_lazyfree and clear PTE dirty bit so allocator may reclaim without write-out; rejects non-anonymous VMAs with -EINVAL.
- mmu_gather batched across all VMAs in the call (madvise_batch_tlb_flush).

REQ-13: madvise_remove:
- File-backed only (anon ⟹ -EINVAL).
- Requires (vma->vm_file && shmem || filesystem supports fallocate-punch).
- Calls vfs_fallocate(file, FALLOC_FL_PUNCH_HOLE | FALLOC_FL_KEEP_SIZE, offset, end-start) under mmap_read_lock, releasing it during I/O.

REQ-14: madvise_populate:
- POPULATE_READ / POPULATE_WRITE: prefault pages via __get_user_pages with FOLL_TOUCH | (FOLL_WRITE for WRITE).
- Returns the count of failed pages as -ENOMEM/-EHWPOISON/-EFAULT propagation.
- Outside madvise_walk_vmas (special-cased in madvise_do_behavior).

REQ-15: madvise_collapse:
- Per-THP: synchronously coalesce 4KiB PTEs into a PMD-level huge page.
- May drop mmap_lock during khugepaged-style page allocation / migration; lock_dropped flag signals walker to re-acquire and re-lookup VMA.
- Delegates to khugepaged's collapse_pte_mapped_thp / hpage_collapse_scan_file.

REQ-16: madvise_guard_install / madvise_guard_remove:
- is_valid_guard_vma: anon, !VM_SHARED, !VM_HUGETLB, !VM_PFNMAP, !VM_IO; locked-VMAs rejected unless allow_locked.
- guard_install: walks page tables, allocates PMD/PUD as needed, replaces empty PTE with guard PTE marker (PTE_MARKER_GUARD) so future faults at that address synthesize -EFAULT (SEGV) rather than allocating a zero page.
- guard_remove: walks and clears guard markers; non-marker entries left alone.
- is_guard_pte_marker(ptent): pte_marker test + PTE_MARKER_GUARD bit.

REQ-17: madvise_inject_error (CONFIG_MEMORY_FAILURE):
- MADV_HWPOISON: requires CAP_SYS_ADMIN; per-page: get_user_pages, then memory_failure(pfn, MF_ACTION_REQUIRED).
- MADV_SOFT_OFFLINE: requires CAP_SYS_ADMIN; soft_offline_page(pfn, 0).
- NO_LOCK mode; operates outside mmap_lock.

REQ-18: process_madvise(2) (SYSCALL_DEFINE5):
- flags must be 0 (-EINVAL otherwise).
- import_iovec into iter (DEST direction, UIO_FASTIOV fastpath).
- pidfd_get_task(pidfd); mm_access(task, PTRACE_MODE_READ_FSCREDS) — leaks ASLR otherwise.
- if mm ≠ current->mm: require process_madvise_remote_valid(behavior) (∈ {COLD, PAGEOUT, WILLNEED, COLLAPSE, GUARD_INSTALL/REMOVE, DONTNEED variants per policy}) ∧ capable(CAP_SYS_NICE).
- vector_madvise(mm, &iter, behavior) — per-iov call do_madvise.
- mmput; put_task_struct; kfree(iov).

REQ-19: vector_madvise:
- Loops iov_iter; for each (start, len) pair: do_madvise(mm, start, len, behavior).
- Returns total bytes successfully processed or first error if no progress made; partial-progress semantics per Linux conventions.

REQ-20: can_madvise_modify (mseal):
- If VM is sealed (VM_SEALED) and the behavior is destructive (REMOVE / DONTNEED-on-shared / FREE on shared / GUARD_INSTALL / GUARD_REMOVE per is_discard policy): return false ⟹ -EPERM.
- Non-destructive hints (WILLNEED, RANDOM, HUGEPAGE, etc.) allowed even when sealed.

REQ-21: madvise_batch_tlb_flush:
- Only DONTNEED, DONTNEED_LOCKED, FREE participate in mmu_gather batching.
- madvise_init_tlb: tlb_gather_mmu(tlb, mm); madvise_finish_tlb: tlb_finish_mmu(tlb).
- All other behaviors leave tlb un-gathered (callee-local TLB if needed).

REQ-22: anon_vma_name lifecycle:
- struct anon_vma_name: { kref, name[] }.
- anon_vma_name_alloc(name): kmalloc_flex; kref_init.
- anon_vma_name_free(kref): kfree.
- replace_anon_vma_name(vma, anon_name): drops old ref, reuses new (anon_vma_name_reuse increments).
- set_anon_vma_name: strndup_user(uname, ANON_VMA_NAME_MAX_LEN); reject chars outside printable ASCII ∪ {\\, `, $, [, ]}.

REQ-23: Errno mapping:
- -EINVAL: invalid behavior / start not page-aligned / len wraps / VMA wrong type for op.
- -ENOMEM: address-range has unmapped gaps; remapped to -EAGAIN at madvise_vma_behavior exit per upstream.
- -EBADF: REMOVE on non-file-backed; SET_ANON_VMA_NAME on file-backed.
- -EPERM: sealed VMA; MADV_HWPOISON without CAP_SYS_ADMIN; remote without CAP_SYS_NICE.
- -EIO: I/O error during WILLNEED.
- -EAGAIN: transient resource shortage (post-remap from ENOMEM).
- -EINTR: signal during mmap_write_lock_killable.

## Acceptance Criteria

- [ ] AC-1: madvise(addr, 0, MADV_NORMAL) returns 0 (zero-length no-op via madvise_should_skip).
- [ ] AC-2: madvise(non-page-aligned start, _, _) returns -EINVAL.
- [ ] AC-3: madvise(addr, len causing wrap, _) returns -EINVAL.
- [ ] AC-4: MADV_DONTNEED on anonymous private range: subsequent read returns zero (pages dropped).
- [ ] AC-5: MADV_FREE on anonymous range: pages reclaimable under pressure; before pressure, read returns original data; write clears lazyfree.
- [ ] AC-6: MADV_WILLNEED on file-backed: file pages prefetched into page cache (readahead invoked).
- [ ] AC-7: MADV_COLD: target folios moved to inactive LRU (no immediate reclaim).
- [ ] AC-8: MADV_PAGEOUT: target anonymous folios paged out (verifiable via /proc/self/smaps or PG_reclaim).
- [ ] AC-9: MADV_REMOVE on file-backed: hole punched in underlying file; -EINVAL on anonymous.
- [ ] AC-10: MADV_DONTFORK then fork: child sees no mapping in that range (gap).
- [ ] AC-11: MADV_DOFORK: clears VM_DONTCOPY; reject when VMA has VM_SPECIAL (-EINVAL).
- [ ] AC-12: MADV_WIPEONFORK: child sees zero-filled range; -EINVAL on file-backed or shared.
- [ ] AC-13: MADV_KEEPONFORK undoes WIPEONFORK; -EINVAL on VM_DROPPABLE.
- [ ] AC-14: MADV_DONTDUMP: range excluded from core dump.
- [ ] AC-15: MADV_DODUMP: range included; -EINVAL on VM_SPECIAL non-hugetlb / VM_DROPPABLE.
- [ ] AC-16: MADV_MERGEABLE: VMA participates in KSM dedup (verifiable via /proc/self/ksm_stat).
- [ ] AC-17: MADV_UNMERGEABLE: VMA opted out of KSM.
- [ ] AC-18: MADV_HUGEPAGE: VM_HUGEPAGE flag set; khugepaged scans range.
- [ ] AC-19: MADV_NOHUGEPAGE: VM_NOHUGEPAGE set; khugepaged skips range.
- [ ] AC-20: MADV_RANDOM: VM_RAND_READ set; readahead disabled for range.
- [ ] AC-21: MADV_SEQUENTIAL: VM_SEQ_READ set; readahead window expanded.
- [ ] AC-22: MADV_NORMAL: clears both VM_RAND_READ and VM_SEQ_READ.
- [ ] AC-23: MADV_COLLAPSE on suitable PTE-mapped range: forms PMD-level THP.
- [ ] AC-24: MADV_GUARD_INSTALL: faulting page in range raises SIGSEGV (no zero-fill).
- [ ] AC-25: MADV_GUARD_REMOVE clears markers; range reverts to normal demand-fault.
- [ ] AC-26: MADV_GUARD_INSTALL on file-backed / VM_SPECIAL / VM_HUGETLB: -EINVAL.
- [ ] AC-27: process_madvise on remote pid with CAP_SYS_NICE + valid behavior: hint delivered.
- [ ] AC-28: process_madvise without CAP_SYS_NICE on remote: -EPERM.
- [ ] AC-29: process_madvise with destructive (DONTNEED on policy-disallowed): -EINVAL.
- [ ] AC-30: madvise on sealed VMA with destructive behavior: -EPERM.
- [ ] AC-31: madvise on unmapped range (gap): -ENOMEM remapped to -EAGAIN.
- [ ] AC-32: PR_SET_VMA_ANON_NAME with name containing $ or \\: -EINVAL.
- [ ] AC-33: PR_SET_VMA_ANON_NAME on file-backed non-anon-shmem: -EBADF.
- [ ] AC-34: madvise_lock with MMAP_WRITE_LOCK + pending fatal signal: returns -EINTR.
- [ ] AC-35: Batched mmu_gather: DONTNEED across N VMAs issues ≤ 1 TLB flush at finish_tlb.

## Architecture

```
enum MadviseLockMode {
  NoLock,
  MmapWriteLock,
  MmapReadLock,
  VmaReadLock,
}

struct MadviseRange { start: usize, end: usize }

struct MadviseBehavior<'a> {
  mm: &'a MmStruct,
  behavior: i32,                    // MADV_*
  tlb: &'a mut MmuGather,
  lock_mode: MadviseLockMode,
  anon_name: Option<&'a AnonVmaName>,
  range: MadviseRange,
  prev: Option<&'a VmaArea>,
  vma:  Option<&'a VmaArea>,
  lock_dropped: bool,
}

struct MadviseWalkPrivate {
  tlb: *mut MmuGather,
  pageout: bool,
}

struct AnonVmaName {
  kref: KRef,
  name: [u8; ANON_VMA_NAME_MAX_LEN + 1],  // NUL-terminated, ≤ 80 chars
}
```

`Madvise::do_madvise(mm, start, len_in, behavior) -> Result<isize, Errno>`:
1. let mut tlb = MmuGather::default().
2. let mut madv = MadviseBehavior { mm, behavior, tlb: &mut tlb, ..Default::default() }.
3. if let Some(err) = Madvise::should_skip(start, len_in, behavior): return err.
4. Madvise::lock(&mut madv)?.
5. Madvise::init_tlb(&mut madv).
6. let error = Madvise::do_behavior(start, len_in, &mut madv).
7. Madvise::finish_tlb(&mut madv).
8. Madvise::unlock(&mut madv).
9. error.

`Madvise::do_behavior(start, len_in, madv) -> Result<isize, Errno>`:
1. if Madvise::is_memory_failure(madv):
   - madv.range.start = start.
   - madv.range.end = start + len_in.
   - return Madvise::inject_error(madv).
2. madv.range.start = Madvise::get_untagged_addr(madv.mm, start).
3. madv.range.end = madv.range.start + PAGE_ALIGN(len_in).
4. blk_start_plug(&plug).
5. let error = if Madvise::is_populate(madv) { Madvise::populate(madv) } else { Madvise::walk_vmas(madv) }.
6. blk_finish_plug(&plug).
7. error.

`Madvise::vma_behavior(madv) -> Result<(), Errno>`:
1. if !Madvise::can_modify(madv): return Err(EPERM).
2. let mut new_flags = madv.vma.unwrap().vm_flags.
3. match madv.behavior:
   - MADV_REMOVE => return Madvise::remove(madv).
   - MADV_WILLNEED => return Madvise::willneed(madv).
   - MADV_COLD => return Madvise::cold(madv).
   - MADV_PAGEOUT => return Madvise::pageout(madv).
   - MADV_FREE | MADV_DONTNEED | MADV_DONTNEED_LOCKED => return Madvise::dontneed_free(madv).
   - MADV_COLLAPSE => return Madvise::collapse(madv.vma, madv.range.start, madv.range.end, &mut madv.lock_dropped).
   - MADV_GUARD_INSTALL => return Madvise::guard_install(madv).
   - MADV_GUARD_REMOVE => return Madvise::guard_remove(madv).
   - MADV_NORMAL => new_flags &= !(VM_RAND_READ | VM_SEQ_READ).
   - MADV_SEQUENTIAL => new_flags = (new_flags & !VM_RAND_READ) | VM_SEQ_READ.
   - MADV_RANDOM => new_flags = (new_flags & !VM_SEQ_READ) | VM_RAND_READ.
   - MADV_DONTFORK => new_flags |= VM_DONTCOPY.
   - MADV_DOFORK => if new_flags & VM_SPECIAL { return Err(EINVAL) }; new_flags &= !VM_DONTCOPY.
   - MADV_WIPEONFORK => if vma.vm_file.is_some() ∨ (new_flags & VM_SHARED) { return Err(EINVAL) }; new_flags |= VM_WIPEONFORK.
   - MADV_KEEPONFORK => if new_flags & VM_DROPPABLE { return Err(EINVAL) }; new_flags &= !VM_WIPEONFORK.
   - MADV_DONTDUMP => new_flags |= VM_DONTDUMP.
   - MADV_DODUMP => if (¬is_vm_hugetlb_page(vma) ∧ (new_flags & VM_SPECIAL)) ∨ (new_flags & VM_DROPPABLE) { return Err(EINVAL) }; new_flags &= !VM_DONTDUMP.
   - MADV_MERGEABLE | MADV_UNMERGEABLE => Madvise::ksm_madvise(vma, madv.range.start, madv.range.end, madv.behavior, &mut new_flags)?.
   - MADV_HUGEPAGE | MADV_NOHUGEPAGE => Madvise::hugepage_madvise(vma, &mut new_flags, madv.behavior)?.
   - __MADV_SET_ANON_VMA_NAME => if vma.vm_file.is_some() ∧ !vma_is_anon_shmem(vma) { return Err(EBADF) }.
4. VM_WARN_ON_ONCE(madv.lock_mode ≠ MmapWriteLock).
5. let error = Madvise::update_vma(new_flags, madv).
6. if error == Err(ENOMEM): return Err(EAGAIN).
7. error.

`Madvise::lock(madv) -> Result<(), Errno>`:
1. let mode = Madvise::get_lock_mode(madv).
2. match mode:
   - NoLock => {}.
   - MmapWriteLock => mmap_write_lock_killable(madv.mm)? /* -EINTR */.
   - MmapReadLock => mmap_read_lock(madv.mm).
   - VmaReadLock => /* per-VMA lock in walker */.
3. madv.lock_mode = mode.
4. Ok(()).

`Madvise::walk_vmas(madv) -> Result<(), Errno>`:
1. let mut addr = madv.range.start.
2. let mut prev = None.
3. while addr < madv.range.end:
   - let vma = find_vma(madv.mm, addr).
   - if vma.is_none() ∨ vma.vm_start > madv.range.end: return Err(ENOMEM).
   - if vma.vm_start > addr: addr = vma.vm_start /* skip gap is not allowed — ENOMEM above handles gap */.
   - if madv.lock_mode == VmaReadLock:
     - if !Madvise::try_vma_read_lock(madv): /* fall back: drop to mmap_read_lock; retry walk */.
   - madv.vma = Some(vma).
   - madv.prev = prev.
   - madv.range.start = max(addr, vma.vm_start).
   - madv.range.end_for_vma = min(madv.range.end, vma.vm_end).
   - Madvise::vma_behavior(madv)?.
   - if madv.lock_dropped: /* re-lookup; loop with updated lock state */.
   - addr = vma.vm_end.
   - prev = Some(vma).
4. Ok(()).

`Madvise::dontneed_free(madv) -> Result<(), Errno>`:
1. if !madvise_dontneed_free_valid_vma(madv): return Err(EINVAL).
2. match madv.behavior:
   - MADV_FREE => Madvise::free_single_vma(madv).
   - MADV_DONTNEED | MADV_DONTNEED_LOCKED => Madvise::dontneed_single_vma(madv).
3. (Returns 0 on success.)

`Madvise::guard_install(madv) -> Result<(), Errno>`:
1. if !is_valid_guard_vma(madv.vma, /*allow_locked=*/false): return Err(EINVAL).
2. walk_page_range_vma(vma, range.start, range.end, &guard_install_ops, NULL).
3. /* guard_install_pmd/pud_entry allocate; guard_install_set_pte writes PTE_MARKER_GUARD where pte_none */.
4. Ok(()).

`Madvise::guard_remove(madv) -> Result<(), Errno>`:
1. if !is_valid_guard_vma(madv.vma, /*allow_locked=*/true): return Err(EINVAL).
2. walk_page_range_vma(vma, range.start, range.end, &guard_remove_ops, NULL).
3. /* clear PTE_MARKER_GUARD where present; leave normal PTEs */.
4. Ok(()).

`Madvise::vector_madvise(mm, iter, behavior) -> Result<isize, Errno>`:
1. let mut total = 0.
2. while iov_iter_count(iter) > 0:
   - let (start, len) = iter_iov_addr_len(iter).
   - let r = Madvise::do_madvise(mm, start, len, behavior).
   - if r.is_err() ∧ total == 0: return r.
   - if r.is_err(): break.
   - total += len.
   - iov_iter_advance(iter, len).
3. Ok(total).

`sys::process_madvise(pidfd, vec, vlen, behavior, flags) -> Result<isize, Errno>`:
1. if flags != 0: return Err(EINVAL).
2. import_iovec(ITER_DEST, vec, vlen, UIO_FASTIOV, &iov, &iter)?.
3. let task = pidfd_get_task(pidfd, &mut f_flags)?.
4. let mm = mm_access(task, PTRACE_MODE_READ_FSCREDS)?.
5. if mm != current.mm ∧ !process_madvise_remote_valid(behavior): return Err(EINVAL).
6. if mm != current.mm ∧ !capable(CAP_SYS_NICE): return Err(EPERM).
7. let ret = Madvise::vector_madvise(mm, &mut iter, behavior).
8. mmput(mm); put_task_struct(task); kfree(iov).
9. ret.

`Madvise::set_anon_name(mm, addr, size, anon_name) -> Result<(), Errno>`:
1. if addr & ~PAGE_MASK: return Err(EINVAL).
2. let len = (size + ~PAGE_MASK) & PAGE_MASK.
3. if size != 0 ∧ len == 0: return Err(EINVAL).
4. let end = addr + len.
5. if end < addr: return Err(EINVAL).
6. if end == addr: return Ok(()).
7. let mut madv = MadviseBehavior { mm, behavior: __MADV_SET_ANON_VMA_NAME, anon_name, range: { addr, end }, .. }.
8. Madvise::lock(&mut madv)?.
9. let error = Madvise::walk_vmas(&mut madv).
10. Madvise::unlock(&mut madv).
11. error.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `is_valid_madvise_strict` | INVARIANT | per-call: PAGE_ALIGNED(start) ∧ ¬overflow(start+len) ∧ behavior ∈ valid-set. |
| `lock_mode_per_behavior` | INVARIANT | per-behavior: get_lock_mode dispatch matches REQ-4 table exactly. |
| `mmap_lock_released_on_error` | INVARIANT | per-do_madvise: madvise_unlock invoked on every exit path (success + error). |
| `vm_flags_mutation_requires_write_lock` | INVARIANT | per-update_vma: lock_mode == MmapWriteLock. |
| `vma_destructive_requires_can_modify` | INVARIANT | per-REMOVE/DONTNEED/FREE/GUARD on sealed VMA: returns -EPERM before touching pages. |
| `guard_install_only_anon_non_special` | INVARIANT | per-GUARD_INSTALL: is_valid_guard_vma true ⟹ anon ∧ ¬VM_SHARED ∧ ¬VM_HUGETLB ∧ ¬VM_PFNMAP ∧ ¬VM_IO. |
| `process_madvise_remote_capable` | INVARIANT | per-remote-syscall: mm ≠ current.mm ⟹ behavior ∈ remote_valid ∧ capable(CAP_SYS_NICE). |

### Layer 2: TLA+

`mm/madvise.tla`:
- Per-call entry + per-lock + per-walk + per-VMA-dispatch + per-update + per-unlock.
- Properties:
  - `safety_lock_acquire_release_balanced` — per-do_madvise: every lock acquire eventually paired with release.
  - `safety_no_partial_mutation_on_einval` — per-call: -EINVAL ⟹ no VMA flag changed.
  - `safety_destructive_under_seal_rejected` — per-sealed-VMA: REMOVE/DONTNEED/FREE/GUARD return -EPERM without state change.
  - `safety_remote_requires_cap_sys_nice` — per-process_madvise: remote mm without CAP_SYS_NICE returns -EPERM.
  - `liveness_walk_terminates` — per-walk_vmas: ∀ range. eventually walker visits all VMAs in range or returns -ENOMEM.
  - `safety_tlb_gather_finished` — per-DONTNEED/FREE: madvise_finish_tlb always invoked when batched.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Madvise::is_valid` post: ret == false ⟹ args malformed (page-align / len-wrap / behavior unknown) | `Madvise::is_valid` |
| `Madvise::should_skip` post: ret == true ⟹ caller may return *err immediately without lock | `Madvise::should_skip` |
| `Madvise::get_lock_mode` post: behavior == MEMORY_FAILURE ⟹ NoLock; else per REQ-4 | `Madvise::get_lock_mode` |
| `Madvise::lock` post: madv.lock_mode == get_lock_mode(madv) ∧ corresponding lock acquired | `Madvise::lock` |
| `Madvise::vma_behavior` post: error in {0, -EPERM, -EINVAL, -EBADF, -EAGAIN, -ENOMEM (remapped), -EIO} | `Madvise::vma_behavior` |
| `Madvise::update_vma` post: vma flags reflect new_flags exactly; split/merge maintains coverage | `Madvise::update_vma` |
| `Madvise::dontneed_free` post: DONTNEED ⟹ PTEs cleared (zap); FREE ⟹ folios lazyfree | `Madvise::dontneed_free` |
| `Madvise::guard_install` post: all empty PTEs in range carry PTE_MARKER_GUARD; existing PTEs unchanged | `Madvise::guard_install` |
| `Madvise::guard_remove` post: all guard markers cleared; non-marker entries unchanged | `Madvise::guard_remove` |
| `Madvise::populate` post: range pages present (or err count) | `Madvise::populate` |
| `sys::process_madvise` post: mm-ref + task-ref released on all exit paths | `sys::process_madvise` |
| `Madvise::set_anon_name` post: vma.anon_name == anon_name for VMAs in range | `Madvise::set_anon_name` |

### Layer 4: Verus/Creusot functional

`Per-call: madvise(start, len, behavior) = sequence over VMAs[start..start+len] of vma_behavior(behavior) under the lock_mode determined by get_lock_mode(behavior)` semantic equivalence: per-Documentation/admin-guide/mm/madvise.rst, per-POSIX 2024 advisory + Linux-specific MADV_* extensions, per-man madvise(2). Combined with the seal-check (can_madvise_modify) and the mmu_gather batching (madvise_batch_tlb_flush), the doc-level pre/post conditions for every MADV_* constant match the field-tested behavior of upstream `mm/madvise.c`.

## Hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening.)

madvise reinforcement:

- **Per-arg-validation up front (is_valid_madvise)** — defense against per-overflow / per-unaligned-start kernel-pointer leak.
- **Per-lock_mode-table strict (get_lock_mode)** — defense against per-wrong-lock-class (e.g., page-walker without mmap_read_lock).
- **Per-mmap_write_lock_killable** — defense against per-unkillable-mmap-write-lock deadlock under fatal signal.
- **Per-can_madvise_modify (mseal)** — defense against per-destructive-op-on-sealed-mapping (e.g., a JIT runtime sealing its code pages and a buggy advice path zapping them).
- **Per-VMA-read-lock fallback to mmap_read_lock** — defense against per-VMA-lock starvation racing with concurrent VMA mutations.
- **Per-is_valid_guard_vma strict (anon, !SHARED, !HUGETLB, !PFNMAP, !IO)** — defense against per-guard-marker on un-suitable VMA confusing fault handler.
- **Per-batched mmu_gather for DONTNEED/FREE only** — defense against per-stale-TLB-entry after PTE clear.
- **Per-blk_start_plug around walk** — defense against per-readahead-fragmentation under WILLNEED.
- **Per-ANON_VMA_NAME_INVALID_CHARS reject** — defense against per-format-injection through procfs anon-name display (`\\` `\`` `$` `[` `]`).
- **Per-process_madvise CAP_SYS_NICE + PTRACE_MODE_READ_FSCREDS** — defense against per-cross-process ASLR-metadata leak and per-unauthorized-remote-paging.
- **Per-process_madvise remote allow-list (process_madvise_remote_valid)** — defense against per-destructive-remote-advice (only non-destructive hints permitted remote-side).
- **Per-MADV_WIPEONFORK anon-only enforcement** — defense against per-file-backed-corruption on fork (would otherwise corrupt shared file pages).
- **Per-MADV_DOFORK reject on VM_SPECIAL** — defense against per-VM_PFNMAP-cow that has no backing page.
- **Per-ENOMEM-to-EAGAIN remap** — defense against per-confusing-errno on transient resource shortage during walk.
- **Per-anon_vma_name kref refcount** — defense against per-name-UAF on concurrent set/clear.
- **Per-MADV_HWPOISON CAP_SYS_ADMIN** — defense against per-unprivileged-page-corruption injection.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- mm/khugepaged.c (MADV_COLLAPSE backend / scan; covered in `thp.md` Tier-3)
- mm/ksm.c (MADV_MERGEABLE/UNMERGEABLE backend; covered in `ksm.md` Tier-3)
- mm/swap_state.c (WILLNEED swap-in helpers; covered in `swap.md` Tier-3)
- mm/memory-failure.c (MADV_HWPOISON / MADV_SOFT_OFFLINE handlers; covered in `memory-failure.md` Tier-3)
- mm/mmap.c (vma_modify, find_vma; covered in `mmap.md` Tier-3)
- mm/userfaultfd.c (separate VMA-mutation path; covered in `userfaultfd.md` Tier-3)
- fs/* fallocate FALLOC_FL_PUNCH_HOLE (covered in `fs/00-overview.md`)
- kernel/sys.c PR_SET_VMA_ANON_NAME prctl wrapper
- Documentation/admin-guide/mm/madvise.rst prose
- Implementation code
