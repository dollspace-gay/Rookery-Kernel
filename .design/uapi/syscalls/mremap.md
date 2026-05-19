# Tier-5: syscall 25 — mremap(2)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - arch/x86/entry/syscalls/syscall_64.tbl (`25  common  mremap  sys_mremap`)
  - mm/mremap.c (`SYSCALL_DEFINE5(mremap, ...)`, `mremap_to`, `move_vma`, `move_page_tables`, `vma_to_resize`)
  - include/uapi/linux/mman.h (`MREMAP_MAYMOVE`, `MREMAP_FIXED`, `MREMAP_DONTUNMAP`)
  - include/linux/mm.h (`VM_DONTEXPAND`, `VM_SHARED`, `VM_LOCKED`, `VM_HUGETLB`)
  - mm/internal.h (`vma_to_resize`)
-->

## Summary

`mremap(2)` is **x86_64 syscall 25**, the in-place VMA-resize / relocate primitive. It grows, shrinks, or moves an existing mapping at `[old_address, old_address + old_size)` to size `new_size`, optionally at a caller-supplied destination `new_address`. The kernel either expands the existing VMA in place (if there is adjacent free VA), or — if `MREMAP_MAYMOVE` is set — allocates a new VA range, walks the page tables of the old range, and atomically moves PTEs (no data copy; this is a page-table re-rooting). `MREMAP_FIXED` (which implies `MREMAP_MAYMOVE`) forces the destination address; `MREMAP_DONTUNMAP` (Linux 5.7+) leaves the *old* range mapped with the same PTEs but anonymous-faulting on next touch (used by GC-style userland concurrent compactors).

Critical for: glibc `realloc` (large allocations move via `mremap`), userspace memory compactors (`MREMAP_DONTUNMAP`), persistent-memory libraries (`pmem`), VM migration, every JIT growing its code arena.

## Signature

C (POSIX / man-pages):

```c
void *mremap(void *old_address, size_t old_size,
             size_t new_size, int flags, ... /* void *new_address */);
```

glibc wrapper: variadic — fetches the 5th arg only when `flags & MREMAP_FIXED`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE5(mremap, unsigned long, addr, unsigned long, old_len,
                unsigned long, new_len, unsigned long, flags,
                unsigned long, new_addr);
```

Rookery dispatch:

```rust
pub fn sys_mremap(addr: u64, old_len: usize, new_len: usize,
                  flags: u64, new_addr: u64) -> SyscallResult<u64>;
```

## Parameters

| name        | type             | constraints                                                       | errno-on-bad |
|-------------|------------------|-------------------------------------------------------------------|--------------|
| addr        | `unsigned long`  | page-aligned; existing mapping start                              | `EINVAL`     |
| old_len     | `size_t`         | `>= 0`; if `0` requires `MREMAP_MAYMOVE` and `[addr, addr+old]` shared | `EINVAL` |
| new_len     | `size_t`         | `> 0`; page-aligned on round-up; not overflow                     | `EINVAL` / `ENOMEM` |
| flags       | `int`            | `MREMAP_MAYMOVE | MREMAP_FIXED | MREMAP_DONTUNMAP` subset         | `EINVAL`     |
| new_address | `unsigned long`  | required iff `MREMAP_FIXED`; page-aligned; non-overlapping        | `EINVAL`     |

## Return value

- Success: page-aligned virtual address of the (possibly relocated, possibly resized) mapping.
- Failure: `(void *)-1` (i.e., `MAP_FAILED`); kernel returns negative errno in `%rax`.

## Errors

| errno    | condition                                                                                  |
|----------|--------------------------------------------------------------------------------------------|
| `EINVAL` | `addr` not page-aligned; flag bits outside the allowed set; `MREMAP_FIXED` without `MREMAP_MAYMOVE`; `MREMAP_DONTUNMAP` without `MREMAP_MAYMOVE`; `MREMAP_DONTUNMAP` and `old_len != new_len`; `MREMAP_DONTUNMAP` on non-private-anonymous VMA; `new_len == 0`; old/new ranges overlap when `MREMAP_FIXED`; `new_address & ~PAGE_MASK`; `old_len == 0` and source is not `MAP_SHARED`. |
| `ENOMEM` | Range `[addr, addr+old_len)` not entirely a single VMA; cannot expand in place without `MREMAP_MAYMOVE`; cannot find free VA; `RLIMIT_AS` / `RLIMIT_DATA` exceeded; `RLIMIT_MEMLOCK` exceeded if `VM_LOCKED`; `sysctl_max_map_count` exceeded; overcommit denial. |
| `EFAULT` | `addr` not actually mapped.                                                                |
| `EAGAIN` | `VM_LOCKED` and post-move `mlock` exceeds `RLIMIT_MEMLOCK`.                                |
| `EINTR`  | `mmap_write_lock_killable` interrupted by fatal signal.                                    |
| `EBUSY`  | LSM denial or `VM_DONTEXPAND` set on the VMA (e.g., DAX, hugetlb non-resizable).           |

## ABI surface (constants + flags)

`flags` (uapi `linux/mman.h`):

- `MREMAP_MAYMOVE`   `0x01` — permission to relocate. Without this, an in-place expansion that requires moving fails with `-ENOMEM`.
- `MREMAP_FIXED`     `0x02` — must be combined with `MREMAP_MAYMOVE`. Use `new_address` as the destination; unmap anything in the destination range.
- `MREMAP_DONTUNMAP` `0x04` — must be combined with `MREMAP_MAYMOVE`; `old_len == new_len`; only on `MAP_PRIVATE|MAP_ANONYMOUS`. Old range remains mapped but page-tables-cleared; next touch faults a fresh zero page. Useful for userspace GC barriers.

`VM_*` flags consulted on the source VMA:

- `VM_SHARED` — affects `MREMAP_DONTUNMAP` rejection.
- `VM_LOCKED` — propagated through; requires `RLIMIT_MEMLOCK` budget.
- `VM_HUGETLB` — has stricter alignment constraints (huge_page_size).
- `VM_DONTEXPAND` — denies grow (some special VMAs like DAX/UFFD).
- `VM_PFNMAP` — special-cased; `move_page_tables` must use PFN-aware walker.

## Compatibility contract

- REQ-1: Argument lowering: `%rdi=addr`, `%rsi=old_len`, `%rdx=new_len`, `%r10=flags`, `%r8=new_addr`.
- REQ-2: `addr = untagged_addr(addr); new_addr = untagged_addr(new_addr);` strip TBI/LAM tags.
- REQ-3: `flags & ~(MREMAP_MAYMOVE | MREMAP_FIXED | MREMAP_DONTUNMAP) != 0` ⟹ `-EINVAL`.
- REQ-4: `(flags & MREMAP_FIXED) && !(flags & MREMAP_MAYMOVE)` ⟹ `-EINVAL`.
- REQ-5: `(flags & MREMAP_DONTUNMAP) && (!(flags & MREMAP_MAYMOVE) || (flags & MREMAP_FIXED))` ⟹ `-EINVAL`.
- REQ-6: `addr & ~PAGE_MASK` ⟹ `-EINVAL`.
- REQ-7: `old_len = PAGE_ALIGN(old_len); new_len = PAGE_ALIGN(new_len);` — if `new_len == 0` ⟹ `-EINVAL`.
- REQ-8: `new_len > TASK_SIZE` or `addr + old_len < addr` ⟹ `-EINVAL`.
- REQ-9: `(flags & MREMAP_FIXED)` requires `mremap_to(addr, old_len, new_addr, new_len, ...)` path.
- REQ-10: `(flags & MREMAP_DONTUNMAP) && old_len != new_len` ⟹ `-EINVAL`.
- REQ-11: `(flags & MREMAP_DONTUNMAP)`: VMA must be `MAP_PRIVATE|MAP_ANONYMOUS` (i.e., `!(vma->vm_flags & VM_SHARED)`); else `-EINVAL`.
- REQ-12: `old_len == 0` (zero-length resize): only legal if VMA is `MAP_SHARED` and `MREMAP_MAYMOVE`; creates a second mapping aliasing the same shared region.
- REQ-13: `mmap_write_lock_killable(mm)` — fatal signal ⟹ `-EINTR`.
- REQ-14: `vma = vma_to_resize(addr, old_len, new_len, flags)` validates: `addr` within a single VMA; `addr + old_len <= vma->vm_end`; VMA not `VM_DONTEXPAND` or `VM_PFNMAP` (unless explicitly handled); `RLIMIT_AS` / `RLIMIT_DATA` would accommodate.
- REQ-15: Shrink (`new_len < old_len`): split VMA at `addr + new_len`; `do_vmi_munmap(&vmi, mm, addr+new_len, old_len-new_len)`; return `addr`.
- REQ-16: In-place expand (`new_len > old_len`, no `MREMAP_FIXED`, adjacent free VA): extend `vma->vm_end` to `addr + new_len`; account `mm->total_vm` and `mm->data_vm`; populate if `VM_LOCKED`.
- REQ-17: Move (`MREMAP_MAYMOVE` or `MREMAP_FIXED`): allocate destination via `get_unmapped_area` (or `new_addr` if FIXED); call `move_vma(vma, addr, old_len, new_len, new_addr, &locked, flags, &uf, &uf_unmap)`.
- REQ-18: `move_vma` ⟹ `move_page_tables(vma, addr, new_vma, new_addr, old_len, ...)` — atomic PTE relocation; locked under `mmap_write_lock` + per-VMA write lock + PTL.
- REQ-19: `MREMAP_DONTUNMAP`: source VMA's PTEs are cleared but VMA remains; refault repopulates with zero pages.
- REQ-20: On success, returned value is the new mapping's start address (== `addr` for in-place or shrink; == new VA for move).
- REQ-21: `VM_LOCKED` propagation: if old VMA was locked, new VMA is locked. Post-move `mm_populate(new_addr, new_len)` re-faults pages outside the lock.
- REQ-22: `userfaultfd_unmap_complete` flushes UFD events for the affected source/destination ranges.

## Acceptance Criteria

- [ ] AC-1: `mremap(p, 4096, 8192, 0)` succeeds and returns `p` when the next page is free; the mapping now covers `[p, p+8192)`.
- [ ] AC-2: `mremap(p, 4096, 8192, 0)` returns `-ENOMEM` when the next page is occupied.
- [ ] AC-3: `mremap(p, 4096, 8192, MREMAP_MAYMOVE)` succeeds and may return a different address; old pages now mapped at the new address.
- [ ] AC-4: `mremap(p, 8192, 4096, 0)` shrinks: returned value `== p`; second page no longer accessible (`SIGSEGV` on touch).
- [ ] AC-5: `mremap(p, 4096, 4096, MREMAP_FIXED|MREMAP_MAYMOVE, q)` relocates the mapping atomically to `q`; touching `q` returns the original contents; touching `p` segfaults.
- [ ] AC-6: `mremap(p, 4096, 4096, MREMAP_DONTUNMAP|MREMAP_MAYMOVE, ...)` returns a new address `q` with the data, but `p` remains mapped (and reads as zeros on next touch).
- [ ] AC-7: `mremap(p, 4096, 4096, MREMAP_FIXED)` (without `MREMAP_MAYMOVE`) ⟹ `-EINVAL`.
- [ ] AC-8: `mremap(p, 4096, 8192, MREMAP_DONTUNMAP|MREMAP_MAYMOVE, ...)` ⟹ `-EINVAL` (sizes differ).
- [ ] AC-9: `mremap(p, 4096, 4096, MREMAP_DONTUNMAP|MREMAP_MAYMOVE, ...)` on a `MAP_SHARED` VMA ⟹ `-EINVAL`.
- [ ] AC-10: `mremap(unmapped, 4096, 8192, MREMAP_MAYMOVE)` ⟹ `-EFAULT`.
- [ ] AC-11: `mremap(p, 4096, big_len, MREMAP_MAYMOVE)` ⟹ `-ENOMEM` when `RLIMIT_AS` exceeded.
- [ ] AC-12: `mremap(p, 0, len, MREMAP_MAYMOVE)` on `MAP_SHARED` aliases a second mapping at returned address.

## Architecture

```
struct MremapArgs {
  addr: u64, old_len: usize, new_len: usize,
  flags: u64, new_addr: u64,
}
```

`sys_mremap(args) -> u64`:

1. Strip address tags.
2. Validate `flags` (REQ-3, REQ-4, REQ-5).
3. Validate `addr` page-alignment (REQ-6).
4. Round `old_len` / `new_len` up.
5. `mmap_write_lock_killable(mm)?;`
6. If `flags & MREMAP_FIXED`:
   - `ret = mremap_to(addr, old_len, new_addr, new_len, flags, &locked, &uf, &uf_unmap_early, &uf_unmap);`
7. Else:
   - `vma = vma_to_resize(addr, old_len, new_len, flags)?;`
   - Shrink path: split + munmap, return `addr`.
   - Expand-in-place: extend VMA, return `addr`.
   - Else if `flags & MREMAP_MAYMOVE`: allocate destination via `get_unmapped_area`; `move_vma(...)`.
   - Else: `-ENOMEM`.
8. `mmap_write_unlock(mm);`
9. If success and `locked` ⟹ `mm_populate(new_addr_or_addr, new_len)`.
10. `userfaultfd_unmap_complete(mm, &uf_unmap_early); userfaultfd_unmap_complete(mm, &uf_unmap);`
11. Return.

`Mm::move_vma(vma, old_addr, old_len, new_len, new_addr, ...)`:

1. Account: `RLIMIT_AS`, `mm->total_vm`, `mm->locked_vm`.
2. Allocate new VMA via `copy_vma(&vma, new_addr, new_len, new_pgoff, &need_rmap_locks)`.
3. `move_page_tables(vma, old_addr, new_vma, new_addr, old_len, need_rmap_locks)` — bulk PTE relocation under proper locks.
4. If `MREMAP_DONTUNMAP`:
   - Clear PTEs in old VMA via `arch_unmap` / `clear_page_tables(old_vma)`.
   - Set `old_vma->vm_flags &= ~(VM_LOCKED|VM_LOCKONFAULT)` if applicable.
5. Else:
   - `do_vmi_munmap(&vmi, mm, old_addr, old_len, &uf_unmap, false)` removes the source VMA.
6. Update accounting and `VM_LOCKED` tracking.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flag_combination_legal` | INVARIANT | `MREMAP_FIXED ⟹ MREMAP_MAYMOVE`; `MREMAP_DONTUNMAP ⟹ MREMAP_MAYMOVE && old_len==new_len && !VM_SHARED`. |
| `range_disjoint` | INVARIANT | `MREMAP_FIXED` source and destination ranges do not overlap. |
| `pte_relocation_atomic` | INVARIANT | `move_page_tables` runs with both VMAs' write locks + PTL held. |
| `lock_held` | INVARIANT | All VMA mutation under `mmap_write_lock`. |
| `rlimit_honored` | INVARIANT | Post-call, `mm.total_vm * PAGE_SIZE <= rlimit(RLIMIT_AS)`. |

### Layer 2: TLA+

`uapi/syscalls/mremap.tla`:
- States: `{src_vma, dst_vma, ptes_src, ptes_dst}`.
- Actions: `Shrink`, `ExpandInPlace`, `Move`, `MoveFixed`, `DontUnmap`.
- Properties:
  - `safety_no_double_mapping_except_dontunmap_or_shared_zero`.
  - `safety_ptes_consistent_post_move`.
  - `liveness_move_terminates_under_mmap_lock`.

### Layer 3: Verus invariants

| Invariant | Component |
|---|---|
| `copy_vma` returns a VMA with `vm_flags` matching source modulo VM_LOCKED | `Mm::move_vma` |
| Source PTEs cleared exactly when `!MREMAP_DONTUNMAP` | `Mm::move_vma` |
| `RLIMIT_MEMLOCK` re-checked under VM_LOCKED propagation | `Mm::vma_to_resize` |

### Layer 4: Verus/Creusot functional

`mremap(addr, old, new, flags, ...)` ≡ Linux man-page semantics (no formal POSIX standard).

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`mremap(2)` reinforcement:

- **Per-flag-validation early-exit** — invalid flag combinations rejected before locks taken.
- **Per-`vma_to_resize` cap-check** — `VM_DONTEXPAND`, `VM_PFNMAP`, hugetlb size constraints checked up front.
- **Per-`RLIMIT_AS` / `RLIMIT_DATA` / `RLIMIT_MEMLOCK`** — DoS defense.
- **Per-`sysctl_max_map_count`** — VMA-table-exhaustion defense.
- **Per `move_page_tables` atomicity** — PTE relocation cannot be observed half-applied.
- **Per `MREMAP_DONTUNMAP` private-anon-only constraint** — prevents using DONTUNMAP to alias `MAP_SHARED` regions for TOCTOU.
- **Per `userfaultfd_unmap_complete`** — UFD events delivered post-unlock for both source and destination.
- **Per `move_vma` LSM check (`security_file_mprotect` analog)** — relocation observed by SELinux/AppArmor for executable VMAs.

## Grsecurity/PaX-style Reinforcement

- **PAX_MPROTECT** — `mremap` cannot upgrade VMA protections; if the source had `VM_EXEC` and target VA is in a non-exec region (PaX policy), the move is denied. Defense against using `mremap` to bypass W^X by relocating an RX mapping into a writable hole.
- **PAX_NOEXEC** — `VM_EXEC|VM_WRITE` simultaneous bits forbidden — re-checked on `mremap` resize/move so the resized VMA cannot acquire forbidden combination.
- **PAX_RANDMMAP** — `get_unmapped_area` selects randomized destination when `MREMAP_MAYMOVE` and no `MREMAP_FIXED`. Defense against deterministic relocation prediction (used in heap-spray escalation chains).
- **PAX_RANDEXEC** — file-backed PROT_EXEC source mappings re-randomized on `MREMAP_MAYMOVE` only if explicitly enabled (most kernels: keep base under move).
- **GRKERNSEC_RWXMAP_LOG** — any `mremap` that would yield a W+X mapping (rare; usually pre-existing) audited.
- **PAX_REFCOUNT** — VMA `vm_users`-equivalent saturating refcounts during `copy_vma`.
- **PAX_UDEREF** — `addr` / `new_addr` are numeric arguments; SMAP enforced; PaX user-deref check ensures no kernel-side dereference of these pointers.
- **PAX_MEMORY_SANITIZE** — `MREMAP_DONTUNMAP` source pages sanitized when freed via shrink; relocated PTEs sanitized between PTE-clear and PTE-set on the destination.
- **GRKERNSEC_BRUTE** — repeated `mremap`-failure crash detected as brute-force; relocation-attempt brute throttled.
- **GRKERNSEC_HIDESYM** — error printks redact kernel pointers in VMA dumps.
- **PAX_RANDKSTACK** — kstack offset rerolled at mremap syscall entry.
- **MFD_NOEXEC_SEAL interplay** — `mremap` on a `memfd_create(MFD_EXEC=0)` mapping cannot grant `VM_EXEC` even via `MREMAP_FIXED` to an executable region; seals respected.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `mmap(2)` (separate Tier-5 — `mmap.md`).
- `munmap(2)` (separate Tier-5 — `munmap.md`).
- `move_pages(2)` NUMA migration.
- `madvise(MADV_DONTNEED)` (separate Tier-5 — `madvise.md`).
- Userfaultfd register / unregister around mremap'd regions (separate Tier-5).
- `pmem` / DAX `MAP_SYNC` interplay with `mremap`.
- Implementation code.
