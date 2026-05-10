# Tier-3: mm/mincore.c — mincore(2) page-residency probe

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: mm/00-overview.md
upstream-paths:
  - mm/mincore.c (~343 lines)
  - include/linux/pagewalk.h
  - include/linux/swap.h
  - include/linux/shmem_fs.h
  - include/uapi/asm-generic/mman-common.h
-->

## Summary

`mincore(2)` is the per-process **page-residency probe**: given `[start, start + len)` and a user byte-vector `vec[]`, the kernel sets `vec[i] & 1 = 1` iff page `i` of the range is currently resident in RAM (would not fault on touch), and `0` otherwise (swapped out, page-cache miss, or never-faulted hole). The walker `mincore_walk_ops` plugs into the generic page-table walker: per-PMD `mincore_pte_range` inspects PTEs (present, swap entry, or none), per-pte-hole `mincore_unmapped_range` consults page-cache for file-backed gaps, per-hugetlb `mincore_hugetlb` lock-folds the entire huge mapping to one residency bit replicated across the covered base pages. Per-VMA gate `can_do_mincore` permits residency disclosure only on (a) anonymous VMAs or (b) file VMAs where the caller owns the inode or has write permission — closing the per-pagecache side-channel for non-owners. Returns `-ENOMEM` if any page in the range lies outside a mapped VMA. Critical for: per-application page-residency-aware preloading (`mmap`+`mincore`+`readahead`), per-test of swap policy, per-memory-pressure-aware schedulers.

This Tier-3 covers `mm/mincore.c` (~343 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `SYSCALL_DEFINE3(mincore, ...)` | per-syscall entry | `Mincore::sys_mincore` |
| `do_mincore(addr, pages, vec)` | per-chunk inner loop | `Mincore::do_mincore` |
| `mincore_walk_ops` | per-pagewalk-ops table | `Mincore::WALK_OPS` |
| `mincore_pte_range(pmd, addr, end, walk)` | per-PMD-entry visitor | `Mincore::pte_range` |
| `mincore_unmapped_range(addr, end, depth, walk)` | per-pte-hole visitor | `Mincore::unmapped_range` |
| `__mincore_unmapped_range(addr, end, vma, vec)` | per-hole file lookup | `Mincore::unmapped_range_inner` |
| `mincore_hugetlb(pte, hmask, addr, end, walk)` | per-hugetlb-entry visitor | `Mincore::hugetlb` |
| `mincore_page(mapping, index)` | per-page-cache lookup | `Mincore::page` |
| `mincore_swap(entry, shmem)` | per-swap-entry residency | `Mincore::swap` |
| `can_do_mincore(vma)` | per-VMA permission gate | `Mincore::can_do_mincore` |
| `walk_page_range(mm, start, end, ops, private)` | per-range walker (generic) | `PageWalk::range` |
| `pte_present(pte)` | per-PTE present test | `PteFlags::present` |
| `pte_none(pte)` | per-PTE absent test | `PteFlags::none` |
| `pte_is_marker(pte)` | per-PTE marker test (uffd, swapin-error) | `PteFlags::is_marker` |
| `softleaf_is_swap(entry)` | per-soft-PTE swap discriminator | `Softleaf::is_swap` |
| `filemap_get_entry(mapping, index)` | per-pagecache xarray lookup | `Filemap::get_entry` |
| `folio_test_uptodate(folio)` | per-folio uptodate test | `Folio::is_uptodate` |
| `inode_owner_or_capable(idmap, inode)` | per-owner / CAP_FOWNER check | `Inode::owner_or_capable` |
| `mmap_read_lock(mm)` / `mmap_read_unlock(mm)` | per-mm read-side | `MmStruct::mmap_read_lock` |

## Compatibility contract

REQ-1: `mincore_walk_ops` populates:
- `.pmd_entry = mincore_pte_range` — per-PMD-leaf path: split THP into base-page bitmap.
- `.pte_hole = mincore_unmapped_range` — per-hole path: for file VMAs consult page-cache; for anon VMAs emit zeros.
- `.hugetlb_entry = mincore_hugetlb` — per-hugetlb leaf path.
- `.walk_lock = PGWALK_RDLOCK` — walker takes `mmap_read_lock`.
- `.test_walk` unset — walk every VMA in [start, end).

REQ-2: `SYSCALL_DEFINE3(mincore, start, len, vec)`:
- start = untagged_addr(start).
- if start & ~PAGE_MASK: return -EINVAL.
- if !access_ok((void __user *)start, len): return -ENOMEM.
- pages = (len >> PAGE_SHIFT) + (offset_in_page(len) != 0 ? 1 : 0).
- if !access_ok(vec, pages): return -EFAULT.
- tmp = (u8 *)__get_free_page(GFP_USER).
- if !tmp: return -EAGAIN.
- while pages:
  - mmap_read_lock(current->mm).
  - retval = do_mincore(start, min(pages, PAGE_SIZE), tmp).
  - mmap_read_unlock(current->mm).
  - if retval <= 0: break.
  - if copy_to_user(vec, tmp, retval): retval = -EFAULT; break.
  - pages -= retval; vec += retval; start += retval << PAGE_SHIFT; retval = 0.
- free_page(tmp).
- return retval.

REQ-3: `do_mincore(addr, pages, vec) -> long`:
- vma = vma_lookup(current->mm, addr).
- if !vma: return -ENOMEM.
- end = min(vma->vm_end, addr + (pages << PAGE_SHIFT)).
- if !can_do_mincore(vma):
  - n = DIV_ROUND_UP(end - addr, PAGE_SIZE).
  - memset(vec, 1, n).
  - return n.        /* opaque "resident" for non-owner shared file VMAs */
- err = walk_page_range(vma->vm_mm, addr, end, &mincore_walk_ops, vec).
- if err < 0: return err.
- return (end - addr) >> PAGE_SHIFT.

REQ-4: `can_do_mincore(vma) -> bool`:
- if vma_is_anonymous(vma): return true.
- if !vma->vm_file: return false.
- return inode_owner_or_capable(&nop_mnt_idmap, file_inode(vma->vm_file)) ∨
        file_permission(vma->vm_file, MAY_WRITE) == 0.
- /* Side-channel close: never reveal pagecache residency to a non-owner of a
     shared file mapping the caller cannot write. */

REQ-5: `mincore_pte_range(pmd, addr, end, walk)`:
- vma = walk->vma; vec = walk->private; nr = (end - addr) >> PAGE_SHIFT.
- /* THP fast path */
- ptl = pmd_trans_huge_lock(pmd, vma).
- if ptl: memset(vec, 1, nr); spin_unlock(ptl); goto out.
- /* Base-page walk */
- ptep = pte_offset_map_lock(walk->mm, pmd, addr, &ptl).
- if !ptep: walk->action = ACTION_AGAIN; return 0.
- for (; addr != end; ptep += step, addr += step * PAGE_SIZE):
  - pte = ptep_get(ptep). step = 1.
  - if pte_none(pte) ∨ pte_is_marker(pte):
    - __mincore_unmapped_range(addr, addr+PAGE_SIZE, vma, vec).     /* consult pagecache */
  - else if pte_present(pte):
    - batch = pte_batch_hint(ptep, pte).
    - if batch > 1: step = min(batch, (end-addr)>>PAGE_SHIFT).
    - for i in 0..step: vec[i] = 1.        /* present ⇒ resident */
  - else:    /* swap entry */
    - entry = softleaf_from_pte(pte).
    - *vec = mincore_swap(entry, false).
  - vec += step.
- pte_unmap_unlock(ptep - 1, ptl).
- out: walk->private += nr; cond_resched(); return 0.

REQ-6: `mincore_unmapped_range(addr, end, depth, walk)`:
- walk->private += __mincore_unmapped_range(addr, end, walk->vma, walk->private).
- return 0.

REQ-7: `__mincore_unmapped_range(addr, end, vma, vec) -> int`:
- nr = (end - addr) >> PAGE_SHIFT.
- if vma->vm_file:
  - pgoff = linear_page_index(vma, addr).
  - for i in 0..nr: vec[i] = mincore_page(vma->vm_file->f_mapping, pgoff++).
- else:
  - for i in 0..nr: vec[i] = 0.        /* anonymous hole ⇒ never-touched */
- return nr.

REQ-8: `mincore_page(mapping, index) -> u8`:
- folio = filemap_get_entry(mapping, index).
- if !folio: return 0.
- if xa_is_value(folio):
  - if shmem_mapping(mapping): return mincore_swap(radix_to_swp_entry(folio), true).
  - else: return 0.                    /* file-backed non-shmem shadow ⇒ absent */
- present = folio_test_uptodate(folio).
- folio_put(folio).
- return present.

REQ-9: `mincore_swap(entry, shmem) -> u8`:
- if !IS_ENABLED(CONFIG_SWAP): WARN_ON(1); return 0.
- if !softleaf_is_swap(entry): return !shmem.
- /* Migration / hwpoison / uffd PTE-marker: always uptodate from a fault POV. */
- /* shmem swap-in-error entries: absent.                                       */
- if shmem:
  - si = get_swap_device(entry).
  - if !si: return 0.
- folio = swap_cache_get_folio(entry).
- if shmem: put_swap_device(si).
- if folio ∧ !xa_is_value(folio):
  - present = folio_test_uptodate(folio).
  - folio_put(folio).
- else: present = 0.
- return present.

REQ-10: `mincore_hugetlb(pte, hmask, addr, end, walk)`:
- vec = walk->private.
- ptl = huge_pte_lock(hstate_vma(walk->vma), walk->mm, pte).
- if !pte: present = 0.
- else:
  - ptep = huge_ptep_get(walk->mm, addr, pte).
  - if huge_pte_none(ptep) ∨ pte_is_marker(ptep): present = 0.
  - else: present = 1.
- for (; addr != end; vec++, addr += PAGE_SIZE): *vec = present.
- walk->private = vec.
- spin_unlock(ptl).

REQ-11: Per-mmap-lock discipline:
- per-syscall outer-loop chunk: mmap_read_lock, do_mincore, mmap_read_unlock.
- Cannot hold across copy_to_user (may fault → re-enter mm).
- Chunk size = PAGE_SIZE entries (4096 result-bytes ⇔ 16 MiB of probed address space on x86-64 base pages).

REQ-12: Per-result encoding (uapi):
- vec[i] bit 0 = 1 ⇒ probed page i is "in core" (would not fault for read).
- Higher bits reserved (zero).
- A `1` return is **advisory**: the page can be evicted between probe and use unless mlocked.

REQ-13: Per-side-channel guard:
- For a shared, non-anonymous VMA whose underlying file the caller cannot write and does not own, `can_do_mincore` returns false and the per-chunk path memsets `vec` to all-ones. Rationale: a non-owner could otherwise enumerate which pages of a shared library / mmap'd database another process has touched, recovering covert-channel data; CVE-2019-5489 forced this gate.

REQ-14: Per-error mapping:
- -EINVAL: start not page-aligned.
- -ENOMEM: [start, start+len) crosses an unmapped hole (vma_lookup miss) OR the user range is invalid (`access_ok` reject of probed addresses).
- -EFAULT: vec[] is not writable.
- -EAGAIN: kernel could not get the bounce page.
- -EINTR: not synthesised here, but interruptible if walker schedules.

REQ-15: Per-PTE-marker handling:
- A `pte_is_marker(pte)` PTE (uffd-wp, swapin-error, etc.) is treated identically to `pte_none`: residency is determined by page-cache consultation (file VMA) or zero (anon VMA). Rationale: a marker carries no folio; whether the page would fault depends entirely on the backing store.

REQ-16: Per-THP fast path:
- A `pmd_trans_huge_lock`-protected PMD short-circuits the per-base-page loop: the whole PMD's worth of vec[] is memset to 1. Rationale: a present THP is by definition entirely resident.

REQ-17: Per-batched-PTE optimisation:
- `pte_batch_hint(ptep, pte)` may report that consecutive PTEs share the same present-resident state (contiguous mapping). The walker steps by `batch` instead of 1, writing `batch` ones at a time. Strictly an optimisation; an implementation may always use step = 1.

REQ-18: Per-`ACTION_AGAIN`:
- A failed `pte_offset_map_lock` (concurrent collapse / split) signals `walk->action = ACTION_AGAIN` so the generic walker retries the PMD. The walker must not advance `walk->private` on retry.

## Acceptance Criteria

- [ ] AC-1: mincore over an all-anon, all-resident range returns vec[i] & 1 == 1 for every i.
- [ ] AC-2: mincore over an anon VMA hole (never-touched page) returns vec[i] & 1 == 0.
- [ ] AC-3: mincore over a swapped-out anon page returns 0 unless the page is in swap-cache uptodate.
- [ ] AC-4: mincore over a shmem mapping where the page has been evicted to swap returns 0 (no swap-cache hit) or 1 (swap-cache hit + uptodate).
- [ ] AC-5: mincore over a file-backed mapping where the page is not in pagecache returns 0.
- [ ] AC-6: mincore on an address range with an unmapped hole returns -ENOMEM and writes nothing beyond the first VMA.
- [ ] AC-7: mincore with unaligned `start` returns -EINVAL.
- [ ] AC-8: mincore with unwritable `vec` returns -EFAULT.
- [ ] AC-9: mincore on a shared file mapping the caller cannot write and does not own returns vec all-ones (side-channel mitigation), not actual residency.
- [ ] AC-10: mincore over a hugetlb VMA reports a single residency bit replicated across all base pages of the huge mapping.
- [ ] AC-11: mincore over a THP-collapsed range reports all-ones in one PMD-locked memset.
- [ ] AC-12: mincore is interruptible via cond_resched between PMDs; large probes do not stall the CPU.
- [ ] AC-13: A concurrent THP split during the walk produces ACTION_AGAIN, not a torn bitmap.
- [ ] AC-14: per-chunk loop bounds: at most PAGE_SIZE (= 4096) entries per kernel→user copy_to_user.

## Architecture

```
struct MincoreWalkCtx {
  vec: *mut u8,                          // walk->private, advanced as walk proceeds
  vma: *const VmAreaStruct,              // walk->vma, set by generic walker
}

const WALK_OPS: MmWalkOps = MmWalkOps {
  pmd_entry: Some(Mincore::pte_range),
  pte_hole: Some(Mincore::unmapped_range),
  hugetlb_entry: Some(Mincore::hugetlb),
  walk_lock: PgWalkLock::RdLock,
  ..MmWalkOps::EMPTY
};
```

`Mincore::sys_mincore(start: usize, len: usize, vec: UserPtrMut<u8>) -> isize`:
1. start = untagged_addr(start).
2. if start & !PAGE_MASK != 0: return -EINVAL.
3. if !access_ok_user(start, len): return -ENOMEM.
4. pages = (len >> PAGE_SHIFT) + (offset_in_page(len) != 0) as usize.
5. if !access_ok(vec, pages): return -EFAULT.
6. tmp: KernelPage = __get_free_page(Gfp::USER).ok_or(-EAGAIN)?.
7. while pages > 0:
   - mm.mmap_read_lock().
   - retval = Mincore::do_mincore(start, min(pages, PAGE_SIZE), tmp.as_mut_slice()).
   - mm.mmap_read_unlock().
   - if retval <= 0: break.
   - if copy_to_user(vec, &tmp[..retval]).is_err(): retval = -EFAULT; break.
   - pages -= retval; vec += retval; start += retval << PAGE_SHIFT; retval = 0.
8. free_page(tmp).
9. return retval.

`Mincore::do_mincore(addr, pages, vec) -> isize`:
1. vma = mm.vma_lookup(addr).ok_or(-ENOMEM)?.
2. end = vma.vm_end.min(addr + (pages << PAGE_SHIFT)).
3. if !Mincore::can_do_mincore(vma):
   - n = (end - addr).div_ceil(PAGE_SIZE).
   - vec[..n].fill(1).
   - return n as isize.
4. ctx = MincoreWalkCtx { vec: vec.as_mut_ptr(), vma: ptr::null() }.
5. PageWalk::range(vma.mm, addr, end, &WALK_OPS, &mut ctx)?.
6. return ((end - addr) >> PAGE_SHIFT) as isize.

`Mincore::can_do_mincore(vma) -> bool`:
1. if vma.is_anonymous(): return true.
2. let file = vma.vm_file.ok_or(return false)?.
3. let inode = file.f_inode.
4. return Inode::owner_or_capable(&NOP_MNT_IDMAP, inode) ∨
          file.permission(MAY_WRITE).is_ok().

`Mincore::pte_range(pmd, addr, end, walk) -> i32`:
1. vma = walk.vma; vec = walk.private as *mut u8; nr = (end - addr) >> PAGE_SHIFT.
2. /* THP fast path */
3. if Some(ptl) = pmd_trans_huge_lock(pmd, vma):
   - core::ptr::write_bytes(vec, 1, nr).
   - ptl.unlock().
   - walk.private = unsafe { vec.add(nr) }.
   - cond_resched(); return 0.
4. let (ptep, ptl) = match pte_offset_map_lock(walk.mm, pmd, addr): Some(p) => p, None => { walk.action = ACTION_AGAIN; return 0; }.
5. for cursor in PteCursor::new(ptep, addr, end):
   - pte = cursor.read().
   - if pte.is_none() ∨ pte.is_marker():
     - Mincore::unmapped_range_inner(cursor.addr, cursor.addr + PAGE_SIZE, vma, vec_at(cursor)).
     - step = 1.
   - else if pte.is_present():
     - batch = pte_batch_hint(cursor.ptep, pte).
     - step = batch.min(((end - cursor.addr) >> PAGE_SHIFT) as u32).
     - vec_slice(cursor, step).fill(1).
   - else:    /* swap */
     - entry = softleaf_from_pte(pte).
     - *vec_at(cursor) = Mincore::swap(entry, false).
     - step = 1.
   - cursor.advance(step).
6. pte_unmap_unlock(ptep - 1, ptl).
7. walk.private = unsafe { vec.add(nr) }.
8. cond_resched(); return 0.

`Mincore::unmapped_range(addr, end, _depth, walk) -> i32`:
1. n = Mincore::unmapped_range_inner(addr, end, walk.vma, walk.private as *mut u8).
2. walk.private = unsafe { (walk.private as *mut u8).add(n) }.
3. return 0.

`Mincore::unmapped_range_inner(addr, end, vma, vec) -> usize`:
1. nr = (end - addr) >> PAGE_SHIFT.
2. if let Some(file) = vma.vm_file:
   - mut pgoff = linear_page_index(vma, addr).
   - for i in 0..nr:
     - unsafe { *vec.add(i) = Mincore::page(file.f_mapping, pgoff) }.
     - pgoff += 1.
3. else:
   - unsafe { core::ptr::write_bytes(vec, 0, nr) }.
4. return nr.

`Mincore::page(mapping, index) -> u8`:
1. let folio = Filemap::get_entry(mapping, index).
2. let folio = match folio: Some(f) => f, None => return 0.
3. if folio.is_xa_value():
   - if mapping.is_shmem(): return Mincore::swap(radix_to_swp_entry(folio), true).
   - else: return 0.
4. let present = folio.is_uptodate() as u8.
5. folio.put().
6. return present.

`Mincore::swap(entry, shmem) -> u8`:
1. if !cfg!(CONFIG_SWAP): warn_on!(true); return 0.
2. if !Softleaf::is_swap(entry): return !shmem as u8.   /* migration/hwpoison/uffd */
3. let si = if shmem { Some(get_swap_device(entry)?) } else { None }.
4. let folio = swap_cache_get_folio(entry).
5. if let Some(si) = si: put_swap_device(si).
6. match folio:
   - Some(f) if !f.is_xa_value() => { let r = f.is_uptodate() as u8; f.put(); r }.
   - _ => 0.

`Mincore::hugetlb(pte, _hmask, addr, end, walk) -> i32`:
1. let vec_base = walk.private as *mut u8.
2. let ptl = huge_pte_lock(hstate_vma(walk.vma), walk.mm, pte).
3. let present = if pte.is_null() {
     0
   } else {
     let ptep = huge_ptep_get(walk.mm, addr, pte).
     if huge_pte_none(ptep) ∨ pte_is_marker(ptep) { 0 } else { 1 }
   }.
4. let n = (end - addr) >> PAGE_SHIFT.
5. unsafe { core::ptr::write_bytes(vec_base, present, n) }.
6. walk.private = unsafe { vec_base.add(n) }.
7. ptl.unlock().
8. return 0.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `vec_writes_stay_in_bounds` | INVARIANT | per-walk: every `*vec_at()` write lies within the per-syscall PAGE_SIZE temp buffer. |
| `private_advances_match_pages` | INVARIANT | per-walk: walk.private advances by exactly nr per pmd/pte_hole/hugetlb visitor. |
| `mmap_read_lock_held_across_walk` | INVARIANT | per-chunk: mmap_read_lock held from do_mincore entry to exit. |
| `no_copy_to_user_under_mmap_lock` | INVARIANT | per-syscall: copy_to_user only after mmap_read_unlock. |
| `unaligned_start_rejected` | INVARIANT | per-syscall: start & ~PAGE_MASK ⟹ -EINVAL. |
| `unmapped_range_yields_enomem` | INVARIANT | per-do_mincore: vma_lookup miss ⟹ -ENOMEM. |
| `non_owner_shared_file_obfuscated` | INVARIANT | per-do_mincore: !can_do_mincore ⟹ all-ones output (side-channel close). |
| `pmd_trans_huge_lock_unlocked` | INVARIANT | per-pte_range: THP fast path unlocks ptl on all exits. |
| `pte_unmap_unlock_balanced` | INVARIANT | per-pte_range: pte_offset_map_lock paired with pte_unmap_unlock. |
| `huge_pte_lock_unlocked` | INVARIANT | per-hugetlb: huge_pte_lock released on all exits. |

### Layer 2: TLA+

`mm/mincore.tla`:
- Per-syscall outer loop (lock/walk/copy) + per-PMD inner loop (THP / base-page / hole / swap).
- Properties:
  - `safety_vec_writes_bounded` — per-walk: vec[] writes never exceed PAGE_SIZE per chunk.
  - `safety_no_user_access_under_mm_lock` — per-syscall: copy_to_user is outside mmap_read_lock.
  - `safety_side_channel_closed` — per-VMA: !can_do_mincore ⟹ output is constant.
  - `safety_enomem_on_unmapped_hole` — per-do_mincore: vma_lookup miss ⟹ -ENOMEM, no partial output mismatch.
  - `liveness_walk_terminates` — per-range walk: bounded by (end - start) / PAGE_SIZE.
  - `liveness_chunk_loop_makes_progress` — per-syscall: either pages decreases or retval set ⟹ loop exits.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Mincore::sys_mincore` post: ret ∈ {0, -EINVAL, -ENOMEM, -EFAULT, -EAGAIN} | `Mincore::sys_mincore` |
| `Mincore::do_mincore` post: returns nr_pages > 0 ∨ negative errno | `Mincore::do_mincore` |
| `Mincore::can_do_mincore` post: !file ∧ !anon ⟹ false | `Mincore::can_do_mincore` |
| `Mincore::pte_range` post: walk.private advanced by (end-addr)>>PAGE_SHIFT | `Mincore::pte_range` |
| `Mincore::unmapped_range_inner` post: returns (end-addr)>>PAGE_SHIFT | `Mincore::unmapped_range_inner` |
| `Mincore::page` post: returns 0 or `folio.is_uptodate()` | `Mincore::page` |
| `Mincore::swap` post: returns 0 / 1 ∧ no leaked swap_device ref | `Mincore::swap` |
| `Mincore::hugetlb` post: writes `present` to every base-page slot, releases ptl | `Mincore::hugetlb` |

### Layer 4: Verus/Creusot functional

`Per-syscall mincore(start, len, vec) ⇔ ∀ i ∈ [0, ceil(len/PAGE_SIZE)): vec[i] & 1 == residency(addr_of_page(start, i))` semantic equivalence: per-`man 2 mincore` + per-`fs/Documentation/admin-guide/mm/transhuge.rst` THP semantics + per-`Documentation/admin-guide/mm/concepts.rst` page-cache / swap residency.

## Hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening.)

mincore reinforcement:

- **Per-untagged_addr on start** — defense against per-MTE / per-PAC tagged-pointer abuse.
- **Per-access_ok on probed range** — defense against per-non-canonical address speculative fetch.
- **Per-access_ok on vec output** — defense against per-kernel write into kernel addresses.
- **Per-PAGE_SIZE temp buffer cap** — defense against per-arbitrary-stack / per-arbitrary-kalloc DoS.
- **Per-mmap_read_lock dropped before copy_to_user** — defense against per-mm_lock-inversion deadlock vs page fault.
- **Per-can_do_mincore side-channel close** — defense against per-CVE-2019-5489-class shared-page-cache inference attacks.
- **Per-pte_offset_map_lock retry (ACTION_AGAIN)** — defense against per-THP-split / per-collapse races yielding torn bitmaps.
- **Per-huge_pte_lock around hugetlb probe** — defense against per-hugetlb-fault race.
- **Per-cond_resched after each PMD** — defense against per-RT-latency starvation on large probes.
- **Per-folio_put balanced with filemap_get_entry / swap_cache_get_folio** — defense against per-folio-refcount-leak.
- **Per-put_swap_device balanced with get_swap_device** — defense against per-swap-device-pin DoS.
- **Per-PTE-marker treated as hole** — defense against per-uffd-wp / per-swapin-error misclassified as present.
- **Per-rcu-free for tmp page (free_page after final dereference)** — defense against per-UAF on the bounce buffer.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- mm/pagewalk.c generic walker mechanics (covered separately if expanded)
- mm/swap_state.c swap-cache (covered in `swap.md` Tier-3)
- mm/shmem.c shmem-specific swap entries (covered in `shmem.md` Tier-3 if expanded)
- mm/hugetlb.c hugetlb mapping (covered in `hugetlb.md` Tier-3 if expanded)
- mm/madvise.c (separate Tier-3)
- mm/userfaultfd.c PTE markers (separate Tier-3)
- Implementation code
