---
title: "Tier-5: syscall 11 — munmap(2)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`munmap(2)` is **x86_64 syscall 11**, the VMA-removal primitive. It removes any mappings that intersect `[addr, addr+length)`, splitting VMAs at the endpoints if necessary. Affected pages are zapped (PTEs cleared, TLBs flushed, page references dropped). The kernel does not require `addr` and `addr+length` to coincide with VMA boundaries: any partial overlap is split. After return, accesses to the unmapped region generate `SIGSEGV`. `munmap` is the deallocator counterpart to `mmap`; glibc's `free` calls `munmap` for `mmap`-backed allocations larger than `M_MMAP_THRESHOLD`. The kernel implements `munmap` via a single VMA-iterator walk (`vma_iter`) that collects all intersecting VMAs into a detached chain, then does the heavy work (TLB flush, page-free, file releases) with the mmap lock dropped.

Critical for: glibc `free` of large allocations, `dlclose` (unmapping the shared object's text/data/bss segments), every dynamic memory pool releasing arenas, every JIT freeing code arenas.

### Acceptance Criteria

- [ ] AC-1: `munmap(p, 4096)` after `p = mmap(NULL, 4096, ...)` succeeds; subsequent read of `p` raises SIGSEGV.
- [ ] AC-2: `munmap(unmapped, 4096)` returns `0` (no-op on already-unmapped range).
- [ ] AC-3: `munmap(p, 0) == -EINVAL`.
- [ ] AC-4: `munmap(p + 1, 4096) == -EINVAL` (addr not page-aligned).
- [ ] AC-5: `munmap(p, 4096)` against the middle of a 12KiB mapping creates two surrounding 4KiB VMAs and unmaps the middle 4KiB.
- [ ] AC-6: `munmap(p, 4096)` against an `mlock`'d range decrements `RLIMIT_MEMLOCK` accounting.
- [ ] AC-7: `munmap(p, 0xFFFFFFFFFFFFF000) == -EINVAL` (overflow / exceeds `TASK_SIZE`).
- [ ] AC-8: Heavy parallel unmaps under contention complete without TLB-flush race (verified via `perf trace`/stress test).
- [ ] AC-9: `munmap` on a file-backed `MAP_SHARED` VMA does not unlink the underlying file; only the mapping is removed; `fput` decrements `f_count`.
- [ ] AC-10: Userfaultfd registered region: `munmap` delivers `UFFD_EVENT_UNMAP` to the userfaultfd reader after the lock is dropped.

### Architecture

```
struct MunmapArgs { addr: u64, len: usize }
```

`sys_munmap(args) -> i32`:

1. Strip address tag.
2. Validate `addr` page alignment, `len > 0`, no overflow, `addr+len <= TASK_SIZE`.
3. `mmap_write_lock_killable(mm)?;`
4. `do_vmi_munmap(&vmi, mm, addr, len, &uf, unlock=true)?;`
   - Returns `0` on success, `-ENOMEM` on split-count overflow, etc.
5. `userfaultfd_unmap_complete(mm, &uf);`
6. Return `0`.

`Mm::do_vmi_munmap(&vmi, mm, start, len, uf, unlock)`:

1. `let end = start + len;`
2. `vma = vma_find(&vmi, end);` — first VMA in range.
3. If `vma.is_none() || vma.vm_start >= end` ⟹ return `0` (no-op).
4. `do_vmi_align_munmap(&vmi, vma, mm, start, end, uf, unlock)`.

`Mm::do_vmi_align_munmap(...)`:

1. Pre-check split count: if range partially overlaps either end ⟹ +1 each end. If post-split `mm.map_count > sysctl_max_map_count` ⟹ `-ENOMEM`.
2. Split front if needed: `__split_vma(&vmi, vma, start, 1)`.
3. Iterate VMAs in `[start, end)`, splitting back if last one extends past `end`: `__split_vma(&vmi, last, end, 0)`.
4. Collect detached VMA list (`mas_detach`).
5. Lock each VMA (write barrier).
6. Update accounting: `mm.total_vm -= ...; mm.locked_vm -= ...;` etc.
7. If `unlock` ⟹ `mmap_write_downgrade(mm)` — downgrade to read for the heavy work.
8. `unmap_region(mm, &mas_detach, vma, prev, next, start, end, mm_wr_seq, !unlock)` — TLB zap.
9. `mmap_read_unlock(mm)` or `mmap_write_unlock(mm)` depending on path.
10. Free detached VMAs (`vm_area_free` on each); call `vm_ops->close`.
11. Return `0`.

### Out of Scope

- `mmap(2)` (separate Tier-5 — `mmap.md`).
- `mprotect(2)` (separate Tier-5 — `mprotect.md`).
- `mremap(2)` (separate Tier-5 — `mremap.md`).
- `madvise(MADV_FREE)` (separate Tier-5 — `madvise.md`).
- Page-table batched flushing internals (`tlb_gather_mmu`).
- IOMMU / KVM / GPU MMU notifier protocol details.
- Implementation code.

### signature

C (POSIX / man-pages):

```c
int munmap(void *addr, size_t length);
```

glibc wrapper: `__munmap` → `INLINE_SYSCALL(munmap, 2, addr, length)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE2(munmap, unsigned long, addr, size_t, len);
```

Rookery dispatch:

```rust
pub fn sys_munmap(addr: u64, len: usize) -> SyscallResult<i32>;
```

### parameters

| name | type             | constraints                                                        | errno-on-bad |
|------|------------------|--------------------------------------------------------------------|--------------|
| addr | `unsigned long`  | page-aligned (`addr & ~PAGE_MASK == 0`)                            | `EINVAL`     |
| len  | `size_t`         | `> 0`; rounded up via `PAGE_ALIGN(len)`; sum must not overflow      | `EINVAL` / `ENOMEM` |

### return value

- Success: `0`.
- Failure: `< 0` — negated errno. **No partial application is observable**: either the call atomically succeeds and the entire range is unmapped, or it fails and leaves all VMAs intact.

### errors

| errno    | condition                                                                                            |
|----------|------------------------------------------------------------------------------------------------------|
| `EINVAL` | `addr` not page-aligned; `len == 0`; `addr + len` overflows or exceeds `TASK_SIZE`.                  |
| `ENOMEM` | Range intersects a `VM_SPECIAL` / `VM_PFNMAP` VMA that does not permit partial unmap; would require splitting a non-splittable VMA (e.g., locked hugetlb that disallows); `mm->map_count` would exceed `sysctl_max_map_count` due to splits. |
| `EINTR`  | `mmap_write_lock_killable` interrupted by fatal signal.                                              |
| `EPERM`  | LSM denial (rare; `security_munmap` analog) or sealed range (`memfd` seal-on-shrink prevents unmap-shrink for shared memfd). |

### abi surface

No flag constants. Internally the kernel honors:

- `VM_LOCKED` — pages must be `munlock`'d as part of the unmap.
- `VM_HUGETLB` — `vma->vm_ops->close` (`hugetlb_vm_op_close`) called; reservations decremented.
- `VM_PFNMAP` — special PTE walker required; `unmap_region` calls `untrack_pfn`.
- `VM_DONTEXPAND` — does not prevent unmap (only prevents grow).
- `VM_UFFD_*` (userfaultfd) — unmap triggers `userfaultfd_unmap_prep` event delivery post-unlock.

### compatibility contract

- REQ-1: Argument lowering: `%rdi=addr`, `%rsi=len`.
- REQ-2: `addr = untagged_addr(addr)`.
- REQ-3: `addr & ~PAGE_MASK ⟹ -EINVAL`.
- REQ-4: `len = PAGE_ALIGN(len);` if rounding overflows to `0` (i.e., original `len != 0` but rounded to `0` due to wrap) ⟹ `-EINVAL`.
- REQ-5: `len == 0` ⟹ `-EINVAL`. (Distinct from `mprotect(len=0)` which is success no-op.)
- REQ-6: `addr + len < addr` overflow check ⟹ `-EINVAL`.
- REQ-7: `addr + len > TASK_SIZE` ⟹ `-EINVAL`.
- REQ-8: `mmap_write_lock_killable(mm)` — fatal signal ⟹ `-EINTR`.
- REQ-9: `vma_iter_init(&vmi, mm, addr); vma = vma_find(&vmi, addr+len);` — if no VMA in range, return `0` (success no-op; man page documents that unmapping an unmapped range is **not** an error).
- REQ-10: For each intersecting VMA, if `VM_LOCKED` ⟹ `munlock_vma_pages_range` (decrement `mm->locked_vm`, clear PG_mlocked).
- REQ-11: Splits at endpoints if VMA extends beyond `[addr, addr+len)`: `__split_vma(&vmi, vma, addr, 1)` and `__split_vma(&vmi, vma, addr+len, 0)`. Each split increases `mm->map_count` by 1; if `mm->map_count > sysctl_max_map_count` post-split ⟹ `-ENOMEM`, rollback.
- REQ-12: Detach intersecting VMAs from `mm->mm_mt` (Maple Tree) atomically via `vma_iter_clear_gfp`.
- REQ-13: `unmap_region(mm, &mas_detach, vma, prev, next, start, end, mm_wr_seq, mm_wr_locked=true)` — zaps PTEs, flushes TLBs (`tlb_gather_mmu` / `tlb_finish_mmu`), drops page references.
- REQ-14: Free detached VMAs (`vm_area_free`); call `vm_ops->close` for each (e.g., hugetlb release, file-backed `fput`).
- REQ-15: Userfaultfd: `userfaultfd_unmap_prep` collects events into `&uf`; delivered after `mmap_write_unlock`.
- REQ-16: `mm->total_vm` / `mm->data_vm` / `mm->exec_vm` / `mm->stack_vm` / `mm->locked_vm` decremented per VMA.
- REQ-17: On success, `mmap_write_unlock` (or `mmap_write_downgrade` if heavy work remains) and TLB invalidation visible to all CPUs by return.
- REQ-18: `arch_unmap(mm, start, end)` arch hook invoked (x86_64: clears `mm->context.vdso_base` if VDSO unmapped).
- REQ-19: `MMU_NOTIFIER` `invalidate_range_start/end` brackets the page-table walk, allowing KVM / IOMMU / GPU drivers to invalidate their secondary mappings.
- REQ-20: After return, any `read(2)` / `write(2)` / explicit access to `[addr, addr+len)` ⟹ `SIGSEGV` (for direct access) or `-EFAULT` (for syscall copy).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `addr_page_aligned` | INVARIANT | `addr & ~PAGE_MASK == 0`. |
| `len_nonzero` | INVARIANT | `PAGE_ALIGN(len) != 0`. |
| `range_within_task_size` | INVARIANT | `addr + len <= TASK_SIZE` and no overflow. |
| `atomic_unmap` | INVARIANT | Either every page in `[addr, addr+len)` is unmapped or none is. |
| `map_count_post_split` | INVARIANT | Post-split `mm.map_count <= sysctl_max_map_count`. |
| `tlb_flushed_before_return` | INVARIANT | All affected PTE entries flushed before user resumes execution. |

### Layer 2: TLA+

`uapi/syscalls/munmap.tla`:
- States: `{vmas, ptes, locked_vm, total_vm}`.
- Actions: `Split`, `Detach`, `ZapPTEs`, `FlushTLB`, `Free`.
- Properties:
  - `safety_no_dangling_ptes`,
  - `safety_lock_drop_after_zap`,
  - `safety_accounting_consistent`,
  - `liveness_unmap_terminates`.

### Layer 3: Verus invariants

| Invariant | Component |
|---|---|
| Detached VMAs reachable only via `mas_detach` between `vma_iter_clear` and free | `Mm::do_vmi_align_munmap` |
| `unmap_region` runs under either `mmap_write_lock` or downgraded read lock + per-VMA write | `Mm::unmap_region` |
| `userfaultfd_unmap_complete` runs after lock drop | `Mm::sys_munmap` |

### Layer 4: Verus/Creusot functional

`munmap(addr, len) ≡ POSIX.1-2024 munmap` semantic equivalence (with Linux unmap-of-unmapped extension).

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`munmap(2)` reinforcement:

- **Per-`mmap_write_lock_killable`** — interruptible by fatal signal; defense against unkillable processes blocked in munmap.
- **Per-split-count cap** — `sysctl_max_map_count` consulted post-split projection so a malicious partial unmap cannot blow up the VMA table.
- **Per-TLB-flush completion** — TLB invalidated before the mmap lock is dropped, guaranteeing no stale cross-CPU translation post-unmap.
- **Per-MMU-notifier brackets** — KVM / IOMMU / GPU mappings invalidated atomically with the kernel page tables.
- **Per `userfaultfd_unmap_complete`** — UFD observers notified after lock drop so they observe a coherent post-unmap state.
- **Per-`vm_ops->close`** — file-backed releases (`fput`, hugetlb reservation accounting) happen exactly once per detached VMA.
- **Per memfd-seal check** — `F_SEAL_SHRINK` on a `memfd_create`'d region prevents shrinking via munmap when the VMA is a `MAP_SHARED` view.

### grsecurity/pax-style reinforcement

- **PAX_MPROTECT** — `munmap` cannot upgrade protections; not directly relevant, but PaX tracks unmapped ranges so subsequent re-mmap with mismatched protections (W^X bypass attempt) is denied.
- **PAX_NOEXEC** — unmap of an exec mapping records the range; a re-mmap of the same VA with `PROT_EXEC` requires identical security context. Defense against `munmap`+`mmap(PROT_EXEC)` to bypass JIT-exec policies.
- **PAX_RANDMMAP** — irrelevant for unmap (no address selection), but freed VA holes are not reused predictably (PaX biases the next `get_unmapped_area` to avoid recently freed VA).
- **PAX_RANDEXEC** — irrelevant for unmap itself.
- **GRKERNSEC_RWXMAP_LOG** — `munmap` of a logged WX mapping audited so the security record is complete.
- **PAX_REFCOUNT** — `mm->mm_users` and `file->f_count` saturating refcounts during the unmap.
- **PAX_UDEREF** — `addr` is a numeric argument; SMAP enforced; PaX user-deref check ensures no kernel-side dereference of these pointers.
- **PAX_MEMORY_SANITIZE** — pages freed by unmap are sanitized via `__free_pages` (zero-on-free) preventing post-free infoleak via residue.
- **GRKERNSEC_BRUTE** — repeated munmap-driven crashes (e.g., munmap during signal handler) detected as brute and the process is throttled.
- **GRKERNSEC_HIDESYM** — VMA-related printks redact kernel pointers.
- **PAX_RANDKSTACK** — kstack offset rerolled at munmap syscall entry.
- **MFD_SEAL_* enforcement** — `F_SEAL_SHRINK`/`F_SEAL_GROW` interplay with munmap-shrink prevents seal-bypass via partial unmap of `MAP_SHARED` memfd views.

