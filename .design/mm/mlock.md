# Tier-3: mm/mlock.c — Locking pages into RAM

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: mm/00-overview.md
upstream-paths:
  - mm/mlock.c (~831 lines)
  - include/linux/mm.h (VM_LOCKED, VM_LOCKONFAULT, VM_LOCKED_MASK)
  - include/linux/mman.h (MCL_CURRENT, MCL_FUTURE, MCL_ONFAULT, MLOCK_ONFAULT)
  - include/uapi/asm-generic/resource.h (RLIMIT_MEMLOCK)
-->

## Summary

`mlock(2)` / `mlock2(2)` / `munlock(2)` / `mlockall(2)` / `munlockall(2)` pin user
pages into resident memory so they cannot be paged out, reclaimed, or migrated
under VM pressure. Per-VMA the kernel records the intent with the `VM_LOCKED`
flag (and optionally `VM_LOCKONFAULT` for the deferred `MLOCK_ONFAULT` /
`MCL_ONFAULT` variant). Per-folio the population state is tracked on the LRU:
locked folios live on the (synthetic) unevictable list, carry `PG_mlocked` plus
`PG_unevictable`, and use `folio->mlock_count` as a reference for pages mapped
into multiple `VM_LOCKED` VMAs. Per-CPU a `folio_batch` (`mlock_fbatch`)
coalesces lock/unlock work so the LRU lock is taken in batches via
`mlock_folio_batch`; the page walker `mlock_pte_range` feeds the batch one PMD
at a time. Per-process the totals are charged against `mm->locked_vm` and
checked against `RLIMIT_MEMLOCK` (overridden by `CAP_IPC_LOCK`). Critical for:
real-time / low-jitter workloads, cryptographic key residency, swap-free
deployments, container `MEMLOCK` enforcement, secretmem permanence.

This Tier-3 covers `mm/mlock.c` (~831 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct mlock_fbatch` | per-CPU batch container | `MlockFbatch` |
| `mlock_fbatch` (DEFINE_PER_CPU) | per-CPU batch slot | `MLOCK_FBATCH` |
| `can_do_mlock()` | per-task capability gate | `Mlock::can_do_mlock` |
| `__mlock_folio()` | per-folio LRU cull to unevictable | `Mlock::folio_lru` |
| `__mlock_new_folio()` | per-fresh-folio cull | `Mlock::folio_new` |
| `__munlock_folio()` | per-folio LRU restore | `Mlock::folio_munlock` |
| `mlock_lru()` / `mlock_new()` | per-batch tag (low-bit) | `Mlock::tag_lru` / `tag_new` |
| `mlock_folio_batch()` | per-batch drain (fold_batch_to_lru) | `Mlock::drain_batch` |
| `mlock_drain_local()` | per-CPU drain | `Mlock::drain_local` |
| `mlock_drain_remote(cpu)` | per-remote-CPU drain | `Mlock::drain_remote` |
| `need_mlock_drain(cpu)` | per-CPU pending probe | `Mlock::need_drain` |
| `mlock_folio()` | per-folio lock enqueue | `Mlock::lock_folio` |
| `mlock_new_folio()` | per-new-folio lock enqueue | `Mlock::lock_new_folio` |
| `munlock_folio()` | per-folio unlock enqueue | `Mlock::unlock_folio` |
| `folio_mlock_step()` | per-large-folio PTE batch step | `Mlock::folio_step` |
| `allow_mlock_munlock()` | per-folio range filter | `Mlock::allow` |
| `mlock_pte_range()` | per-PMD page walker | `Mlock::pte_range_walker` |
| `mlock_vma_pages_range()` | per-VMA walker driver | `Mlock::vma_pages_range` |
| `mlock_fixup()` | per-VMA flag-set + split/merge | `Mlock::fixup` |
| `apply_vma_lock_flags()` | per-range VMA-iter loop | `Mlock::apply_range` |
| `count_mm_mlocked_page_nr()` | per-mm already-locked sum | `Mlock::count_locked` |
| `__mlock_posix_error_return()` | per-error remap | `Mlock::posix_err` |
| `do_mlock()` | per-syscall body | `Mlock::do_mlock` |
| `SYSCALL_DEFINE2(mlock, ...)` | mlock(2) entry | `Mlock::sys_mlock` |
| `SYSCALL_DEFINE3(mlock2, ...)` | mlock2(2) entry | `Mlock::sys_mlock2` |
| `SYSCALL_DEFINE2(munlock, ...)` | munlock(2) entry | `Mlock::sys_munlock` |
| `apply_mlockall_flags()` | per-mm-wide apply | `Mlock::apply_mlockall` |
| `SYSCALL_DEFINE1(mlockall, ...)` | mlockall(2) entry | `Mlock::sys_mlockall` |
| `SYSCALL_DEFINE0(munlockall)` | munlockall(2) entry | `Mlock::sys_munlockall` |
| `shmlock_user_lock` | per-shm-lock global spinlock | `SHMLOCK_USER_LOCK` |
| `user_shm_lock()` | per-shm rlimit accounting | `Mlock::shm_lock` |
| `user_shm_unlock()` | per-shm rlimit release | `Mlock::shm_unlock` |
| `populate_vma_page_range()` | per-range GUP-prefault (mm/gup.c) | shared with `gup.md` |
| `__mm_populate()` | per-mm populate driver (mm/gup.c) | shared with `gup.md` |

## Compatibility contract

REQ-1: VMA flag taxonomy:
- `VM_LOCKED`: pages mlock'd now (residence enforced).
- `VM_LOCKONFAULT`: pages lock at fault time (lazy variant, `MLOCK_ONFAULT` /
  `MCL_ONFAULT`).
- `VM_LOCKED_MASK = VM_LOCKED | VM_LOCKONFAULT`: union mask for fixup.
- VMAs failing `vma_supports_mlock` (e.g. `VM_PFNMAP`, hugetlb in some forms,
  `VM_IO`, special drivers) silently bypass — `VM_LOCKED` is never set.
- Per-secretmem VMA (`vma_is_secretmem`): always considered locked; munlock
  is a no-op (memory cannot be unlocked once secret).

REQ-2: `struct mlock_fbatch`:
- `local_lock_t lock`: per-CPU local-lock guarding the batch.
- `struct folio_batch fbatch`: up to `PAGEVEC_SIZE` (15) folio pointers.
- Folio pointers carry low-bit tags `LRU_FOLIO (0x1)` and `NEW_FOLIO (0x2)`
  (set via `mlock_lru` / `mlock_new`); untagged pointers are munlocks.

REQ-3: `can_do_mlock`:
- Returns true if `rlimit(RLIMIT_MEMLOCK) != 0` ∨ `capable(CAP_IPC_LOCK)`.
- Used as the syscall gate before any state mutation.

REQ-4: `mlock_folio(folio)`:
- `local_lock(&mlock_fbatch.lock)`.
- if !`folio_test_set_mlocked(folio)`:
  - `zone_stat_mod_folio(folio, NR_MLOCK, +nr_pages)`.
  - `__count_vm_events(UNEVICTABLE_PGMLOCKED, nr_pages)`.
- `folio_get(folio)` (ref pin while in batch).
- if `!folio_batch_add(fbatch, mlock_lru(folio)) ∨ !folio_may_be_lru_cached ∨
  lru_cache_disabled()`: `mlock_folio_batch(fbatch)` (immediate fold).
- `local_unlock`.

REQ-5: `mlock_new_folio(folio)`:
- For freshly allocated folios not yet on any LRU.
- Unconditionally sets `PG_mlocked` and bumps `NR_MLOCK` / `UNEVICTABLE_PGMLOCKED`.
- Tags the batch entry via `mlock_new(folio)` (NEW_FOLIO bit).

REQ-6: `munlock_folio(folio)`:
- `folio_get` then enqueue untagged into the same batch.
- `folio_test_clear_mlocked` deferred to `__munlock_folio` so the
  `mlock_count` multiply-mapped check sees a consistent state.

REQ-7: `mlock_folio_batch` (fold_batch_to_lru transition):
- For each tagged folio in the batch:
  - LRU_FOLIO → `__mlock_folio(folio, lruvec)`
  - NEW_FOLIO → `__mlock_new_folio(folio, lruvec)`
  - untagged  → `__munlock_folio(folio, lruvec)`
- Holds `lruvec->lru_lock` across the batch (relocked per folio via
  `folio_lruvec_relock_irq`).
- Resets `fbatch` (`folio_batch_reinit`) and drops folio refs at the end.

REQ-8: `__mlock_folio` invariants:
- If folio is now evictable but was unevictable: move from unevictable list to
  active/inactive, count `UNEVICTABLE_PGRESCUED`.
- If folio is unevictable + still mlocked: increment `folio->mlock_count`.
- Otherwise (was evictable, becoming unevictable):
  - `lruvec_del_folio`, `folio_clear_active`, `folio_set_unevictable`,
    `folio->mlock_count = !!folio_test_mlocked(folio)`, `lruvec_add_folio`,
    bump `UNEVICTABLE_PGCULLED`.

REQ-9: `__munlock_folio` invariants:
- If `folio_test_unevictable`: decrement `mlock_count`; if still > 0, leave
  unevictable and return (multiply mapped under another `VM_LOCKED` VMA).
- Otherwise: `folio_test_clear_mlocked`, decrement `NR_MLOCK`, bump
  `UNEVICTABLE_PGMUNLOCKED` (or `UNEVICTABLE_PGSTRANDED` if isolated but still
  unevictable).
- If isolated ∧ unevictable ∧ now `folio_evictable`: move back to active LRU,
  bump `UNEVICTABLE_PGRESCUED`.

REQ-10: `mlock_pte_range` (page-walker callback):
- Acquires `pmd_trans_huge_lock`; if PMD-leaf and not zero-page nor
  zone-device: `mlock_folio` / `munlock_folio` for the whole huge folio.
- Otherwise `pte_offset_map_lock` and per-PTE loop:
  - skip `!pte_present`.
  - skip `!vm_normal_folio` / zone-device.
  - `step = folio_mlock_step(folio, pte, addr, end)` (PTE batch).
  - if `!allow_mlock_munlock(folio, vma, start, end, step)`: skip step.
  - else: `mlock_folio` if `VM_LOCKED` else `munlock_folio`.
- `cond_resched` after each PMD.

REQ-11: `mlock_vma_pages_range`:
- Temporarily sets `VM_IO` on the VMA while walking (signals rmap walkers to
  back off on the now-`VM_LOCKED` VMA so they cannot race and double-count
  `mlock_count`). `VM_IO` is cleared after the walk.
- `lru_add_drain` before and after the walk to flush any pending non-mlock LRU
  caches that might interfere with `folio_test_clear_lru` in `__mlock_folio`.
- `vma_start_write(vma)` + `vma_flags_reset_once(vma, new_vma_flags)`.

REQ-12: `mlock_fixup`:
- Skip if `vma_flags_same_pair(old, new)`, `vma_is_secretmem(vma)`, or
  `!vma_supports_mlock(vma)`.
- `vma_modify_flags(vmi, *prev, vma, start, end, new_vma_flags)` may split
  and/or merge VMAs (returns `ERR_PTR` on failure).
- Update `mm->locked_vm`:
  - new `VM_LOCKED` ∧ !old: `+nr_pages`.
  - !new ∧ old `VM_LOCKED`: `-nr_pages`.
  - both: 0 (already counted).
- If transitioning to / from `VM_LOCKED`: `mlock_vma_pages_range(...)`.
- Otherwise just stamp flags under `vma_start_write`.

REQ-13: `do_mlock`:
- `untagged_addr(start)`; `can_do_mlock()` ∨ `-EPERM`.
- Align: `len = PAGE_ALIGN(len + offset_in_page(start)); start &= PAGE_MASK`.
- `lock_limit = rlimit(RLIMIT_MEMLOCK) >> PAGE_SHIFT`.
- `mmap_write_lock_killable`; on `-EINTR` propagate.
- `locked = (len >> PAGE_SHIFT) + mm->locked_vm`.
- If `locked > lock_limit` ∧ `!CAP_IPC_LOCK`:
  - subtract already-locked overlap via `count_mm_mlocked_page_nr(mm, start, len)`.
- If `locked <= lock_limit` ∨ `CAP_IPC_LOCK`: `apply_vma_lock_flags(start, len, flags)`.
- `mmap_write_unlock`.
- On success: `__mm_populate(start, len, 0)` to pre-fault (skipped for the
  `VM_LOCKONFAULT` case because the flag stays cleared on the populate path).
- `__mlock_posix_error_return(err)` for the GUP return code remap (EFAULT→ENOMEM,
  ENOMEM→EAGAIN).

REQ-14: `sys_mlock`: `do_mlock(start, len, VM_LOCKED)`.
REQ-15: `sys_mlock2`:
- Reject `flags & ~MLOCK_ONFAULT` with `-EINVAL`.
- `vm_flags = VM_LOCKED | (flags & MLOCK_ONFAULT ? VM_LOCKONFAULT : 0)`.
- `do_mlock(start, len, vm_flags)`.

REQ-16: `sys_munlock`: `apply_vma_lock_flags(start, len, 0)` under
`mmap_write_lock_killable`. No populate step.

REQ-17: `apply_mlockall_flags`:
- Validate `flags` ⊆ `{MCL_CURRENT, MCL_FUTURE, MCL_ONFAULT}`; `MCL_ONFAULT` alone
  is `-EINVAL`.
- Clear `mm->def_flags & VM_LOCKED_MASK` and reset.
- If `MCL_FUTURE`: `mm->def_flags |= VM_LOCKED` (and `VM_LOCKONFAULT` if
  `MCL_ONFAULT`). When `!MCL_CURRENT`: return without walking VMAs.
- If `MCL_CURRENT`: `to_add = VM_LOCKED [| VM_LOCKONFAULT]`; iterate every VMA,
  rebuild `newflags = (vma->vm_flags & ~VM_LOCKED_MASK) | to_add`, call
  `mlock_fixup`. Errors are absorbed (only `prev` is fixed up) so partial
  failures still walk all VMAs.

REQ-18: `sys_mlockall`:
- Validate flags (`-EINVAL` if 0, unknown bits, or `MCL_ONFAULT` alone).
- `can_do_mlock()` ∨ `-EPERM`.
- `MCL_CURRENT` check: only allowed if `total_vm <= lock_limit` ∨ `CAP_IPC_LOCK`.
- `apply_mlockall_flags(flags)`; on success ∧ `MCL_CURRENT`: `mm_populate(0, TASK_SIZE)`.

REQ-19: `sys_munlockall`: `apply_mlockall_flags(0)` (clears every VMA and
`mm->def_flags`).

REQ-20: `user_shm_lock` / `user_shm_unlock`:
- For SHM_LOCK / SHM_HUGETLB segments whose lifetime is independent of any
  single process; accounting routed via `ucounts` (`UCOUNT_RLIMIT_MEMLOCK`)
  under `shmlock_user_lock` spinlock.
- `RLIM_INFINITY` lock_limit short-circuits the rlimit comparison.

## Acceptance Criteria

- [ ] AC-1: `mlock(addr, len)` sets `VM_LOCKED` on the covered VMAs and
  populates all pages: post-call no further page-fault occurs for reads/writes
  in `[start, start+len)` until munlock/munmap.
- [ ] AC-2: `mlock2(..., MLOCK_ONFAULT)` sets `VM_LOCKONFAULT` and does NOT
  pre-fault; on first fault the page is locked.
- [ ] AC-3: `munlock` clears `VM_LOCKED` / `VM_LOCKONFAULT` and any
  `folio->mlock_count` cleanly transitions back to evictable.
- [ ] AC-4: Per-folio `mlock_count` correctly reflects the number of
  `VM_LOCKED` VMAs mapping the folio (no leak after fork/munlock).
- [ ] AC-5: `RLIMIT_MEMLOCK` is enforced: `mlock` returns `-ENOMEM` (mapped to
  `-EAGAIN` via posix remap from GUP) when `locked > lock_limit` and the task
  lacks `CAP_IPC_LOCK`.
- [ ] AC-6: `CAP_IPC_LOCK` bypasses the rlimit check.
- [ ] AC-7: `mlockall(MCL_FUTURE)` causes subsequent `mmap` regions to inherit
  `VM_LOCKED` via `mm->def_flags`.
- [ ] AC-8: `mlockall(MCL_FUTURE|MCL_ONFAULT)` inherits `VM_LOCKONFAULT`.
- [ ] AC-9: `munlockall` clears `mm->def_flags & VM_LOCKED_MASK` and unlocks
  every VMA.
- [ ] AC-10: Per-CPU `mlock_fbatch` is flushed lazily via `mlock_folio_batch`
  when full, or eagerly via `mlock_drain_local` / `mlock_drain_remote` on
  CPU-hotplug / drain-all paths.
- [ ] AC-11: `mlock_pte_range` skips zone-device, zero, and non-normal folios.
- [ ] AC-12: `mlock_vma_pages_range` toggles `VM_IO` during the walk so rmap
  cannot race-bump `mlock_count`.
- [ ] AC-13: `secretmem` VMAs cannot be unlocked.
- [ ] AC-14: `shmctl(SHM_LOCK)` accounting goes via `user_shm_lock` (ucounts),
  independent of any single mm's `locked_vm`.
- [ ] AC-15: `dump_tasks`-style `/proc/<pid>/status` `VmLck` reflects
  `mm->locked_vm * PAGE_SIZE`.

## Architecture

```
struct MlockFbatch {
  lock: LocalLock,
  fbatch: FolioBatch,                 // [Folio*; 15]
}
PER_CPU MLOCK_FBATCH: MlockFbatch;

// Tag bits packed into low bits of folio pointers in the batch.
const LRU_FOLIO: usize = 0x1;
const NEW_FOLIO: usize = 0x2;

// VMA flag union covering both mlock variants.
const VM_LOCKED_MASK: VmFlags = VM_LOCKED | VM_LOCKONFAULT;
```

`Mlock::can_do_mlock() -> bool`:
1. if `rlimit(RLIMIT_MEMLOCK) != 0`: return true.
2. if `capable(CAP_IPC_LOCK)`: return true.
3. return false.

`Mlock::lock_folio(folio)`:
1. `local_lock(&MLOCK_FBATCH.lock)`.
2. `fbatch = this_cpu_ptr(&MLOCK_FBATCH.fbatch)`.
3. if `!folio_test_set_mlocked(folio)`:
   - `zone_stat_mod_folio(folio, NR_MLOCK, +nr_pages)`.
   - `__count_vm_events(UNEVICTABLE_PGMLOCKED, nr_pages)`.
4. `folio_get(folio)`.
5. `slot = folio_batch_add(fbatch, mlock_lru(folio))`.
6. if `!slot ∨ !folio_may_be_lru_cached(folio) ∨ lru_cache_disabled()`:
   - `Mlock::drain_batch(fbatch)`.
7. `local_unlock`.

`Mlock::lock_new_folio(folio)`:
1. `local_lock(&MLOCK_FBATCH.lock)`.
2. `folio_set_mlocked(folio)` (unconditional — fresh folio).
3. `zone_stat_mod_folio(folio, NR_MLOCK, +nr_pages)`; bump `UNEVICTABLE_PGMLOCKED`.
4. `folio_get(folio)`.
5. enqueue `mlock_new(folio)` (NEW_FOLIO tag); drain if full.
6. `local_unlock`.

`Mlock::unlock_folio(folio)`:
1. `local_lock(&MLOCK_FBATCH.lock)`.
2. `folio_get(folio)`.
3. enqueue *untagged* `folio`; drain if full.
4. `local_unlock`.
5. /* `folio_test_clear_mlocked` happens inside `__munlock_folio` so the
   `mlock_count` decrement and the `PG_mlocked` clear stay consistent. */

`Mlock::drain_batch(fbatch)` (fold_batch_to_lru):
1. `lruvec = NULL`.
2. for tagged in fbatch:
   - if tag == LRU_FOLIO: `lruvec = Mlock::folio_lru(tagged, lruvec)`.
   - elif tag == NEW_FOLIO: `lruvec = Mlock::folio_new(tagged, lruvec)`.
   - else: `lruvec = Mlock::folio_munlock(tagged, lruvec)`.
3. if `lruvec`: `unlock_page_lruvec_irq(lruvec)`.
4. `folios_put(fbatch.folios, fbatch.nr)` (drop refs).
5. `folio_batch_reinit(fbatch)`.

`Mlock::folio_lru(folio, lruvec) -> lruvec`:
1. if `!folio_test_clear_lru(folio)`: return lruvec (off LRU; nothing to do).
2. `lruvec = folio_lruvec_relock_irq(folio, lruvec)`.
3. if `folio_evictable(folio)`:
   - if `folio_test_unevictable(folio)`:
     - `lruvec_del_folio`, `folio_clear_unevictable`, `lruvec_add_folio`.
     - bump `UNEVICTABLE_PGRESCUED`.
   - goto out (don't bump `mlock_count` — folio is no longer locked).
4. if `folio_test_unevictable(folio)`:
   - if `folio_test_mlocked(folio)`: `folio->mlock_count += 1`.
   - goto out.
5. /* Was evictable, now becoming unevictable. */
6. `lruvec_del_folio`; `folio_clear_active`; `folio_set_unevictable`.
7. `folio->mlock_count = if folio_test_mlocked(folio) { 1 } else { 0 }`.
8. `lruvec_add_folio`; bump `UNEVICTABLE_PGCULLED`.
9. out: `folio_set_lru(folio)`; return lruvec.

`Mlock::folio_new(folio, lruvec) -> lruvec`:
1. assert `!folio_test_lru(folio)` (fresh folio).
2. `lruvec = folio_lruvec_relock_irq(folio, lruvec)`.
3. if `folio_evictable(folio)`: skip cull (caller raced).
4. else: `folio_set_unevictable`; `mlock_count = bool(PG_mlocked)`; bump
   `UNEVICTABLE_PGCULLED`.
5. `lruvec_add_folio(lruvec, folio)`; `folio_set_lru(folio)`.

`Mlock::folio_munlock(folio, lruvec) -> lruvec`:
1. `isolated = false`.
2. if `folio_test_clear_lru(folio)`:
   - `isolated = true`; `lruvec = folio_lruvec_relock_irq(folio, lruvec)`.
   - if `folio_test_unevictable(folio)`:
     - if `folio->mlock_count`: `folio->mlock_count -= 1`.
     - if `folio->mlock_count > 0`: goto out (still locked by another VMA).
3. munlock label: if `folio_test_clear_mlocked(folio)`:
   - `__zone_stat_mod_folio(folio, NR_MLOCK, -nr_pages)`.
   - bump `UNEVICTABLE_PGMUNLOCKED` if isolated or now evictable;
     else bump `UNEVICTABLE_PGSTRANDED`.
4. if isolated ∧ unevictable ∧ `folio_evictable(folio)`:
   - `lruvec_del_folio`; `folio_clear_unevictable`; `lruvec_add_folio`.
   - bump `UNEVICTABLE_PGRESCUED`.
5. out: if isolated: `folio_set_lru(folio)`; return lruvec.

`Mlock::pte_range_walker(pmd, addr, end, walk) -> i32`:
1. `vma = walk.vma`; `start = addr`; `step = 1`.
2. `ptl = pmd_trans_huge_lock(pmd, vma)`.
3. if `ptl` (huge PMD leaf):
   - if `!pmd_present(*pmd) ∨ is_huge_zero_pmd(*pmd)`: goto out.
   - `folio = pmd_folio(*pmd)`; if `folio_is_zone_device(folio)`: goto out.
   - if `vma.vm_flags & VM_LOCKED`: `Mlock::lock_folio(folio)`.
   - else: `Mlock::unlock_folio(folio)`.
   - goto out.
4. `(start_pte, ptl) = pte_offset_map_lock(vma.vm_mm, pmd, addr)`; on NULL set
   `walk.action = ACTION_AGAIN` and return 0.
5. for `pte` in `[start_pte, end)`:
   - `ptent = ptep_get(pte)`; if `!pte_present`: continue.
   - `folio = vm_normal_folio(vma, addr, ptent)`; if `!folio ∨ zone_device`: continue.
   - `step = Mlock::folio_step(folio, pte, addr, end)`.
   - if `!Mlock::allow(folio, vma, start, end, step)`: goto next_entry.
   - if `VM_LOCKED`: `Mlock::lock_folio(folio)` else `Mlock::unlock_folio(folio)`.
   - next_entry: advance by `step` PTEs.
6. `pte_unmap(start_pte)`.
7. out: `spin_unlock(ptl)`; `cond_resched()`; return 0.

`Mlock::vma_pages_range(vma, start, end, new_vma_flags)`:
1. const ops `{ .pmd_entry = pte_range_walker, .walk_lock = PGWALK_WRLOCK_VERIFY }`.
2. if `new_vma_flags & VMA_LOCKED_BIT`: temporarily set `VMA_IO_BIT` in
   `new_vma_flags` (poison rmap during walk).
3. `vma_start_write(vma)`; `vma_flags_reset_once(vma, new_vma_flags)`.
4. `lru_add_drain()`.
5. `walk_page_range(vma.vm_mm, start, end, &ops, NULL)`.
6. `lru_add_drain()`.
7. if `VMA_IO_BIT` was injected: clear it; `vma_flags_reset_once` again.

`Mlock::fixup(vmi, vma, prev, start, end, newflags)`:
1. `new_vma_flags = legacy_to_vma_flags(newflags)`; `old = vma.flags`.
2. if `vma_flags_same_pair(old, new) ∨ secretmem ∨ !vma_supports_mlock`:
   - `*prev = vma`; return 0.
3. `vma = vma_modify_flags(vmi, *prev, vma, start, end, &new)` (may split/merge).
4. on `ERR_PTR`: propagate.
5. `nr = (end - start) >> PAGE_SHIFT`.
6. if `!new.VMA_LOCKED_BIT`: nr = -nr.
7. else if `old.VMA_LOCKED_BIT`: nr = 0.
8. `mm.locked_vm += nr`.
9. if `new.LOCKED ∧ old.LOCKED`: just rewrite flags under `vma_start_write`.
10. else: `Mlock::vma_pages_range(vma, start, end, &new)`.
11. `*prev = vma`; return 0.

`Mlock::apply_range(start, len, flags) -> i32`:
1. `VMA_ITERATOR(vmi, current.mm, start)`; sanity-check page-alignment.
2. `end = start + len`; reject overflow; trivially 0 if `end == start`.
3. `vma = vma_iter_load(&vmi)`; if NULL: `-ENOMEM`.
4. `prev = vma_prev(&vmi)`; if `start > vma.vm_start`: `prev = vma`.
5. `nstart = start; tmp = vma.vm_start`.
6. `for_each_vma_range(vmi, vma, end)`:
   - if `vma.vm_start != tmp`: gap — `-ENOMEM`.
   - `newflags = (vma.vm_flags & ~VM_LOCKED_MASK) | flags`.
   - `tmp = min(vma.vm_end, end)`.
   - `Mlock::fixup(&vmi, vma, &prev, nstart, tmp, newflags)?`.
   - `tmp = vma_iter_end(&vmi); nstart = tmp`.
7. if `tmp < end`: `-ENOMEM`.

`Mlock::do_mlock(start, len, vm_flags) -> i32`:
1. `start = untagged_addr(start)`.
2. `can_do_mlock() ∨ -EPERM`.
3. `len = PAGE_ALIGN(len + offset_in_page(start)); start &= PAGE_MASK`.
4. `lock_limit = rlimit(RLIMIT_MEMLOCK) >> PAGE_SHIFT`.
5. `mmap_write_lock_killable(current.mm)?`.
6. `locked = (len >> PAGE_SHIFT) + current.mm.locked_vm`.
7. if `locked > lock_limit ∧ !CAP_IPC_LOCK`:
   - `locked -= Mlock::count_locked(current.mm, start, len)`.
8. err = `-ENOMEM`.
9. if `locked <= lock_limit ∨ CAP_IPC_LOCK`:
   - err = `Mlock::apply_range(start, len, vm_flags)`.
10. `mmap_write_unlock`.
11. err? -> return err.
12. err = `__mm_populate(start, len, 0)`.
13. return `Mlock::posix_err(err)`.

`Mlock::sys_mlock(start, len)`: `do_mlock(start, len, VM_LOCKED)`.
`Mlock::sys_mlock2(start, len, flags)`:
- if `flags & ~MLOCK_ONFAULT`: return `-EINVAL`.
- `vm_flags = VM_LOCKED | (flags & MLOCK_ONFAULT ? VM_LOCKONFAULT : 0)`.
- `do_mlock(start, len, vm_flags)`.
`Mlock::sys_munlock(start, len)`:
- normalize start/len; `mmap_write_lock_killable`; `apply_range(start, len, 0)`;
  `mmap_write_unlock`.

`Mlock::apply_mlockall(flags) -> i32`:
1. `current.mm.def_flags &= ~VM_LOCKED_MASK`.
2. if `MCL_FUTURE`:
   - `def_flags |= VM_LOCKED`.
   - if `MCL_ONFAULT`: `def_flags |= VM_LOCKONFAULT`.
   - if `!MCL_CURRENT`: return 0.
3. if `MCL_CURRENT`:
   - `to_add = VM_LOCKED | (MCL_ONFAULT ? VM_LOCKONFAULT : 0)`.
4. iter every VMA in `current.mm`:
   - `newflags = (vma.vm_flags & ~VM_LOCKED_MASK) | to_add`.
   - `Mlock::fixup(...)`; errors fix `prev` but loop continues.
   - `cond_resched()`.
5. return 0.

`Mlock::sys_mlockall(flags)`:
1. validate flags (REQ-18).
2. `can_do_mlock() ∨ -EPERM`.
3. `lock_limit = rlimit(RLIMIT_MEMLOCK) >> PAGE_SHIFT`.
4. `mmap_write_lock_killable`.
5. err = `-ENOMEM` unless `!MCL_CURRENT ∨ total_vm <= lock_limit ∨ CAP_IPC_LOCK`.
6. err = `apply_mlockall(flags)`.
7. unlock; if `!err ∧ MCL_CURRENT`: `mm_populate(0, TASK_SIZE)`.

`Mlock::sys_munlockall`: `mmap_write_lock_killable; apply_mlockall(0); unlock`.

`Mlock::shm_lock(size, ucounts) -> i32`:
1. `locked = (size + PAGE_SIZE - 1) >> PAGE_SHIFT`.
2. `lock_limit = rlimit(RLIMIT_MEMLOCK)`; if `!RLIM_INFINITY`: `>>= PAGE_SHIFT`.
3. `spin_lock(&SHMLOCK_USER_LOCK)`.
4. `memlock = inc_rlimit_ucounts(ucounts, UCOUNT_RLIMIT_MEMLOCK, locked)`.
5. if `memlock == LONG_MAX ∨ memlock > lock_limit ∧ !CAP_IPC_LOCK`:
   - `dec_rlimit_ucounts`; allowed = 0.
6. else if `!get_ucounts(ucounts)`: `dec_rlimit_ucounts`; allowed = 0.
7. else: allowed = 1.
8. `spin_unlock`; return allowed.

`Mlock::shm_unlock(size, ucounts)`:
1. `spin_lock(&SHMLOCK_USER_LOCK)`.
2. `dec_rlimit_ucounts(ucounts, UCOUNT_RLIMIT_MEMLOCK, ceil_pages(size))`.
3. `spin_unlock`; `put_ucounts(ucounts)`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `mlock_count_nonneg` | INVARIANT | per-folio: after any lock/munlock the count never wraps below zero. |
| `mlock_count_matches_vma_locks` | INVARIANT | per-folio: `mlock_count > 0` ⟺ at least one mapping VMA carries `VM_LOCKED`. |
| `nr_mlock_balanced` | INVARIANT | per-zone: `NR_MLOCK` returns to its pre-mlock value after a complete `munlock` of the same range. |
| `locked_vm_balanced` | INVARIANT | per-mm: `locked_vm` returns to baseline after a paired mlock/munlock or `apply_mlockall(0)`. |
| `rlimit_enforced` | INVARIANT | per-task: `do_mlock` returns `-EAGAIN` (POSIX-remapped) when `locked > lock_limit` ∧ `!CAP_IPC_LOCK`. |
| `fbatch_drained_on_overflow` | INVARIANT | per-CPU: enqueue into a full `mlock_fbatch` triggers `mlock_folio_batch` before returning. |
| `vm_io_toggled_during_walk` | INVARIANT | per-VMA: `VM_IO` is set strictly across `walk_page_range` and cleared on exit. |

### Layer 2: TLA+

`mm/mlock.tla`:
- Models: per-VMA flag set, per-folio `mlock_count`, per-mm `locked_vm`, per-CPU batch state.
- Properties:
  - `safety_locked_count_matches_vmas` — at every quiescent state, each folio's
    `mlock_count` equals the number of mapping VMAs with `VM_LOCKED`.
  - `safety_unevictable_implies_locked_or_other` — `PG_unevictable` set ⟹
    `PG_mlocked ∨ other-unevictable-reason`.
  - `safety_rlimit_invariant` — no non-`CAP_IPC_LOCK` task ever increases
    `locked_vm` past `rlimit(RLIMIT_MEMLOCK)`.
  - `liveness_per_mlock_terminates` — `mlock` either returns or is killed by
    `mmap_write_lock_killable`.
  - `liveness_fbatch_eventually_folded` — every batched folio eventually
    reaches `__mlock_folio` / `__mlock_new_folio` / `__munlock_folio`.
  - `safety_munlock_eventually_evictable` — after `munlockall`, no `VM_LOCKED`
    VMA exists and every previously locked folio is either back on
    active/inactive LRU or marked `UNEVICTABLE_PGSTRANDED`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `lock_folio` pre: `local_lock` held; post: folio enqueued or batch drained | `Mlock::lock_folio` |
| `drain_batch` post: batch empty, all folios fold-applied, refs dropped | `Mlock::drain_batch` |
| `folio_lru` post: if pre-mlocked + evictable then RESCUED; if !evictable then `mlock_count >= 1` | `Mlock::folio_lru` |
| `folio_munlock` post: `mlock_count == 0` ⟹ `PG_mlocked` cleared | `Mlock::folio_munlock` |
| `pte_range_walker` post: every present non-special folio in range is enqueued | `Mlock::pte_range_walker` |
| `vma_pages_range` post: `VM_IO` not set on return | `Mlock::vma_pages_range` |
| `fixup` post: `locked_vm` delta == ±nr_pages or 0 per transition | `Mlock::fixup` |
| `do_mlock` post: success ⟹ all pages populated ∨ `VM_LOCKONFAULT` deferred | `Mlock::do_mlock` |
| `apply_mlockall` post: `def_flags` mirrors `(MCL_FUTURE → VM_LOCKED, MCL_ONFAULT → VM_LOCKONFAULT)` | `Mlock::apply_mlockall` |
| `shm_lock` post: `allowed == 1` ⟹ ucounts incremented and held | `Mlock::shm_lock` |

### Layer 4: Verus/Creusot functional

`Per-mlock(2) → do_mlock → apply_vma_lock_flags → mlock_fixup → mlock_vma_pages_range →
walk_page_range / mlock_pte_range → mlock_folio / munlock_folio → mlock_folio_batch →
__mlock_folio / __munlock_folio → __mm_populate` semantic equivalence: per-
`Documentation/admin-guide/mm/concepts.rst` Unevictable LRU + Documentation/vm
"Mlocked Pages". Per-`mlockall(MCL_FUTURE)` ⟹ `mm->def_flags |= VM_LOCKED` such
that every later `mmap` inherits the bit. Per-`MLOCK_ONFAULT` ⟹ no populate step,
faults install pages already marked. Per-`shmctl(SHM_LOCK)` accounting equivalent
to ucount-based `RLIMIT_MEMLOCK` per `Documentation/admin-guide/sysctl/kernel.rst`.

## Hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening.)

mlock reinforcement:

- **Per-task `can_do_mlock` gate** — defense against per-unprivileged-DoS
  pin-all-RAM.
- **Per-mm `RLIMIT_MEMLOCK` enforcement with overlap subtraction** — defense
  against per-double-counting on re-mlock of already-locked overlap.
- **Per-VMA `vma_supports_mlock` filter** — defense against per-driver
  PFNMAP / IO region accidental pin.
- **Per-secretmem permanent lock** — defense against per-secret leak via
  munlock + swap.
- **Per-walk `VM_IO` injection** — defense against per-rmap race
  double-incrementing `mlock_count`.
- **Per-folio `mlock_count` saturated decrement** — defense against per-double
  munlock underflow / stranded unevictable folio.
- **Per-CPU `mlock_fbatch` bounded length** — defense against per-batch unbounded
  growth → IRQ-latency spike.
- **Per-folio `folio_get` while in batch** — defense against per-batch UAF if
  the folio is freed before fold.
- **Per-`lru_add_drain` before/after walk** — defense against per-stale
  pre-mlock LRU cache hiding the folio from `__mlock_folio`.
- **Per-`mmap_write_lock_killable`** — defense against per-OOM-kill-while-holding
  mmap_lock deadlock.
- **Per-`apply_mlockall` errors absorbed but `prev` fixed** — defense against
  per-stale `prev` causing VMA-merge corruption mid-iteration.
- **Per-CAP_IPC_LOCK bypass scoped to RLIMIT only** — defense against
  per-non-root mlockall(MCL_FUTURE) silent rlimit escape.
- **Per-ucounts namespace for shm-lock** — defense against per-user-namespace
  cross-tenant `RLIMIT_MEMLOCK` confusion.

## Grsecurity/PaX-style Reinforcement

Baseline hardening that applies to the mlock / mlockall / munlock family:

- **PAX_USERCOPY** — mlock copies no payload to userspace; only `start`/`len`/`flags` flow in via syscall args, all bounded by `access_ok`.
- **PAX_KERNEXEC** — `mlock_folio` / `munlock_folio` paths and the `lru_add_drain` callbacks live in `.rodata`.
- **PAX_RANDKSTACK** — `do_mlock` / `apply_mlockall` entered with randomized kernel stack offset.
- **PAX_REFCOUNT** — `folio->_mlock_count` uses saturating refcount semantics; double-mlock cannot wrap to zero and bypass the unevictable LRU placement.
- **PAX_MEMORY_SANITIZE** — folios released back from `unevictable` LRU after `munlock` are sanitized before re-entering buddy allocator.
- **PAX_UDEREF** — page-walker portion of `populate_vma_page_range` dereferences kernel PTE pointers only.
- **PAX_RAP / kCFI** — `pagewalk`-based mlock callbacks dispatched through type-checked vtables.
- **GRKERNSEC_HIDESYM** — folio / VMA kernel addresses never leak into `mlockall`-induced WARN_ONs visible to non-root.
- **GRKERNSEC_DMESG** — `mlock`-related RLIMIT-violation warnings restricted from unprivileged dmesg.

mlock-specific reinforcement:

- **RLIMIT_MEMLOCK enforcement** — `user->locked_vm + new_pages > rlimit(RLIMIT_MEMLOCK)` is checked atomically under `mm->mmap_lock` write; no TOCTOU between check and account.
- **CAP_IPC_LOCK** — bypass of RLIMIT_MEMLOCK requires `CAP_IPC_LOCK`; the bypass is scoped *only* to the rlimit check, not to side-effects like writeability, so a non-root caller cannot escalate mappings.
- **mlock_count refcount** — `folio_mlock_count` is `refcount_t`; saturation traps before a stale-counter folio can be returned to evictable LRU while still page-table-pinned.
- **ucounts namespace** — locked-pages tally is per-`ucounts` so a user-namespace tenant cannot drain the host's locked-pages budget.

Rationale: mlock pins kernel memory by user request; the above ensure pinning stays bounded by rlimit, traceably refcounted, and unforgeable across namespaces.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `mm/gup.c` `populate_vma_page_range` / `__mm_populate` internals (covered in
  `gup.md` Tier-3).
- `mm/vmscan.c` unevictable LRU reclaim path (covered in `reclaim.md`).
- `mm/rmap.c` `try_to_unmap_one` interaction with `VM_LOCKED` (covered in
  `rmap.md`).
- `mm/hugetlb.c` hugetlb mlock specifics (covered in `hugetlb.md`).
- `mm/secretmem.c` secret memory (covered separately if expanded).
- `ipc/shm.c` `SHM_LOCK` / `SHM_UNLOCK` syscall plumbing (covered separately if
  expanded).
- Implementation code.
