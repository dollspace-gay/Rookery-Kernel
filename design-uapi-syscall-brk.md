---
title: "Tier-5: syscall 12 — brk(2)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`brk(2)` is **x86_64 syscall 12**, the legacy heap-grow syscall. It sets `current->mm->brk` to the requested address (rounded up to page boundary), growing or shrinking the program break. The program break is the address one past the end of the process's data segment: pages in `[mm->start_brk, mm->brk)` constitute the classic Unix heap. `brk` is the foundation that glibc's `malloc` uses for small allocations (when configured to do so via `M_MMAP_THRESHOLD`); on modern systems most allocations bypass `brk` in favor of anonymous `mmap`. The kernel implements `brk` essentially as a single growing VMA at `mm->start_brk`: growing the break is a no-overlap-checked anonymous mmap with `VM_DATA_DEFAULT_FLAGS`; shrinking the break is a partial unmap.

Critical for: glibc `malloc` arena allocator (sbrk path), legacy programs that use `sbrk(3)` directly, and the kernel's bootstrap setup of the initial heap at exec time (via `set_brk` in `binfmt_elf.c`).

### Acceptance Criteria

- [ ] AC-1: `brk(0)` returns the current `mm->brk` and is a pure query (no VMA changes observable via `/proc/self/maps`).
- [ ] AC-2: `brk(current + 4096)` grows the heap VMA by one page and returns the new (page-aligned) break.
- [ ] AC-3: `brk(current - 4096)` (when current is page-aligned and one page above `start_brk`) shrinks the heap VMA by one page.
- [ ] AC-4: `brk(addr)` with `addr < mm->start_brk` is a no-op; returned value equals pre-call `mm->brk`.
- [ ] AC-5: `brk(addr)` that would exceed `RLIMIT_DATA` ⟹ returned value equals pre-call `mm->brk` (no growth).
- [ ] AC-6: `brk(addr)` that collides with an existing mapping at `[oldbrk, newbrk+PAGE_SIZE)` ⟹ no-op.
- [ ] AC-7: Newly granted heap pages read as zero before first write.
- [ ] AC-8: `brk(addr)` interrupted by a fatal signal returns pre-call `mm->brk`.
- [ ] AC-9: Under `mlockall(MCL_FUTURE)`, newly granted heap pages are resident and locked immediately after `brk` returns.
- [ ] AC-10: `/proc/self/maps` heap line covers `[start_brk, brk)` after a successful grow.

### Architecture

```
struct BrkArgs { new_brk: u64 }
```

`sys_brk(new_brk) -> u64`:

1. `let mm = current.mm;`
2. `mmap_write_lock_killable(mm)?;` — on fatal signal return `mm->brk`.
3. `let oldbrk = page_align(mm.brk);`
4. `let newbrk = page_align(new_brk);`
5. `let origbrk = mm.brk;`
6. If `new_brk < mm.start_brk` ⟹ goto `out`.
7. `let newbrk_data = newbrk - mm.end_data;` if `newbrk_data > rlimit(RLIMIT_DATA)` and `!capable(CAP_SYS_RESOURCE)` ⟹ goto `out`.
8. If `newbrk == oldbrk`:
   - `mm.brk = new_brk;` — sub-page move.
   - `success = true;` goto `out`.
9. If `newbrk <= oldbrk`:
   - `if do_vmi_munmap(&vmi, mm, newbrk, oldbrk - newbrk, &uf, true).is_ok() { mm.brk = new_brk; success = true; }`
   - goto `out`.
10. Grow path:
    - `let next = find_vma_intersection(mm, oldbrk, newbrk + PAGE_SIZE);`
    - If `next.is_some()` ⟹ goto `out` (would overlap).
    - `if do_brk_flags(&vmi, &brkvma, oldbrk, newbrk - oldbrk, 0).is_ok() { mm.brk = new_brk; success = true; }`
11. `out:`
    - `let retval = if success { new_brk } else { origbrk };`
    - `mmap_write_unlock(mm);`
    - `if success && (mm.def_flags & VM_LOCKED) { mm_populate(oldbrk, newbrk - oldbrk); }`
    - `userfaultfd_unmap_complete(mm, &uf);`
    - Return `retval`.

`Mm::do_brk_flags(vmi, vma, addr, len, flags)`:

1. `if mm.map_count > sysctl_max_map_count ⟹ -ENOMEM`.
2. `if !vm_enough_memory(len >> PAGE_SHIFT) ⟹ -ENOMEM`.
3. Merge with `*vma` if it ends at `addr` and flags match; else allocate a new VMA `[addr, addr+len)` with `VM_DATA_DEFAULT_FLAGS | flags | mm.def_flags | VM_SOFTDIRTY`.
4. `vma_link(&vmi, vma)`.
5. `mm->total_vm += len >> PAGE_SHIFT; mm->data_vm += len >> PAGE_SHIFT;`
6. Return `0`.

### Out of Scope

- `sbrk(3)` glibc wrapper (userspace; computes `brk(cur + inc)` and returns old `cur`).
- `mmap(2)` (separate Tier-5 — `mmap.md`).
- `prctl(PR_SET_MM_BRK)` (separate Tier-5 if prctl is broken out).
- `binfmt_elf` initial-brk setup (`set_brk`) — exec-side, not syscall-side.
- glibc `malloc` arena heuristics (`M_MMAP_THRESHOLD`).
- Implementation code.

### signature

C (POSIX / man-pages):

```c
int brk(void *addr);
void *sbrk(intptr_t increment); /* glibc wrapper */
```

glibc wrapper: `__brk` / `__sbrk` → `INLINE_SYSCALL(brk, 1, addr)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE1(brk, unsigned long, brk);
```

Rookery dispatch:

```rust
pub fn sys_brk(brk: u64) -> SyscallResult<u64>;
```

### parameters

| name | type            | constraints                                                        | errno-on-bad |
|------|-----------------|--------------------------------------------------------------------|--------------|
| brk  | `unsigned long` | requested new program break; `0` is a query (returns current brk)  | none direct  |

### return value

- Success: the new value of `mm->brk` (which equals page-aligned-up `addr` on grow, or `addr` clamped down on shrink). On failure, **returns the current `mm->brk` unchanged** — `brk(2)` does *not* return a negative errno: glibc's `brk()` C wrapper compares the returned value to the requested one and synthesizes `ENOMEM`.
- The query convention is `brk(0)` ⟹ returns current `mm->brk`.

### errors

`brk(2)` itself never returns `-errno`. It returns the unchanged current break on failure. Conditions that prevent growth:

| condition (returned-current means failure)                                                                       |
|-------------------------------------------------------------------------------------------------------------------|
| Requested `brk < mm->start_brk` (would shrink below initial brk).                                                 |
| Requested `brk` collides with an existing VMA (excluding the heap VMA itself).                                    |
| `RLIMIT_DATA` would be exceeded by `(brk - mm->start_brk)`.                                                       |
| Overcommit policy denies (`sysctl_overcommit_memory == OVERCOMMIT_NEVER` and no commit slack).                    |
| `mm->map_count > sysctl_max_map_count` (cannot create new VMA segment).                                           |
| `do_brk_flags()` returns `-ENOMEM` for any reason (no free VA, allocator failure).                                |
| `mmap_write_lock_killable` interrupted (fatal signal) ⟹ returns current brk unchanged.                            |

### abi surface

No flag constants. The single argument is an absolute address. Internally the kernel uses:

- `VM_DATA_DEFAULT_FLAGS = VM_READ | VM_WRITE | VM_MAYREAD | VM_MAYWRITE | VM_MAYEXEC` — heap VMA permission flags. On x86_64 with `READ_IMPLIES_EXEC` personality, `VM_EXEC` is additionally set.
- `mm->def_flags` — process-default VMA flags (notably `VM_LOCKED` if `MCL_FUTURE` is set via `mlockall(2)`).
- `mm->start_brk` / `mm->brk` / `mm->start_data` / `mm->end_data` — process layout fields seeded by `binfmt_elf` at exec.

Underlying call chain on grow:

```
sys_brk(addr)
 └─ vm_brk_flags(oldbrk_page_aligned, len, VM_DATA_DEFAULT_FLAGS)
     └─ do_brk_flags(addr, len, flags)
         └─ vma allocation + vma_merge or vma_link
```

On shrink: `do_vmi_munmap(&vmi, mm, newbrk, oldbrk - newbrk, NULL, true)`.

### compatibility contract

- REQ-1: Argument lowering: `%rdi = brk`. Single-argument syscall via `SYSCALL_DEFINE1`.
- REQ-2: `brk(0)` is a query and returns the current `mm->brk` (after taking the mmap lock).
- REQ-3: The page-rounded-up requested new break is `newbrk = PAGE_ALIGN(brk)`. The page-rounded-up current break is `oldbrk = PAGE_ALIGN(mm->brk)`.
- REQ-4: If `brk < mm->start_brk` ⟹ no-op, return current `mm->brk`.
- REQ-5: Compute `newbrk_data_size = newbrk - mm->end_data`. If `newbrk_data_size > rlimit(RLIMIT_DATA)` and `!CAP_SYS_RESOURCE` ⟹ no-op, return current `mm->brk`. (Linux 4.7+ enforces `RLIMIT_DATA` against `brk` growth.)
- REQ-6: If `newbrk == oldbrk` ⟹ update `mm->brk = brk` (sub-page change) and return.
- REQ-7: Shrink (`brk <= mm->brk`): `do_vmi_munmap(&vmi, mm, newbrk, oldbrk - newbrk, &uf, true)` — on success update `mm->brk = brk`; on failure return current `mm->brk`.
- REQ-8: Grow: `find_vma_intersection(mm, oldbrk, newbrk + PAGE_SIZE)` — any VMA found other than the heap VMA's own continuation ⟹ no-op (would-overlap).
- REQ-9: Grow: `vm_brk_flags(oldbrk, newbrk - oldbrk, 0)` — call returns `0` on success; else no-op, return current `mm->brk`.
- REQ-10: Overcommit check via `check_data_rlimit` and `__vm_enough_memory` — denial ⟹ no-op.
- REQ-11: `vm_brk_flags` internally takes `mmap_write_lock_killable`; if fatal signal during wait ⟹ return current `mm->brk` (effectively `-EINTR`-equivalent).
- REQ-12: On successful grow, the new pages are **not** pre-faulted. They are zero-page-on-first-read, anonymous-allocated-on-first-write. (Unless `mm->def_flags & VM_LOCKED` ⟹ `mm_populate(oldbrk, newbrk - oldbrk)` post-unlock.)
- REQ-13: On success, `userfaultfd_unmap_complete` flushes any pending UFD events for the affected range.
- REQ-14: The returned value is unconditionally `mm->brk`, even on failure (it just equals the pre-call value in that case).
- REQ-15: `PR_SET_MM_BRK` via `prctl` can move `mm->brk` directly without VMA changes (CAP_SYS_RESOURCE); independent of this syscall.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `brk_monotone_on_success` | INVARIANT | On success, returned `brk >= mm.start_brk`. |
| `brk_unchanged_on_failure` | INVARIANT | On failure, returned `brk == origbrk` and `mm.brk == origbrk`. |
| `rlimit_data_honored` | INVARIANT | After call, `mm.brk - mm.end_data <= rlimit(RLIMIT_DATA) || capable(CAP_SYS_RESOURCE)`. |
| `mmap_lock_write_held` | INVARIANT | All VMA mutation occurs under `mmap_write_lock`. |
| `query_is_pure` | INVARIANT | `brk(0)` does not modify `mm->brk` or any VMA. |

### Layer 2: TLA+

`uapi/syscalls/brk.tla`:
- Per-call state: `{mm.brk, mm.start_brk, mm.end_data, rlimit_data, vmas}`.
- Actions: `Query`, `Grow`, `Shrink`, `NoOpUnderLimit`, `NoOpOverlap`, `Signal`.
- Properties:
  - `safety_brk_in_bounds` — `mm.start_brk <= mm.brk`.
  - `safety_no_partial_change_on_failure`.
  - `liveness_grow_eventually_succeeds_under_capacity`.

### Layer 3: Verus invariants

| Invariant | Component |
|---|---|
| `do_brk_flags` only adds VMAs with `VM_DATA_DEFAULT_FLAGS` | `Mm::do_brk_flags` |
| Shrink path passes `unlock=true` so `mmap_lock` is released across the unmap | `Mm::sys_brk` |
| Populate happens outside the mmap lock | `Mm::sys_brk` post-unlock block |

### Layer 4: Verus/Creusot functional

`brk(addr) ≡ POSIX.1-2024 sbrk(addr - cur)` semantic equivalence — modulo Linux extensions (`RLIMIT_DATA` enforcement, soft-dirty bit).

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`brk(2)` reinforcement:

- **Per-`RLIMIT_DATA` enforcement on `brk` growth** — Linux 4.7+ closed the loophole where `brk` bypassed `RLIMIT_DATA`. Prevents heap-as-RWX-overflow attacks bypassing data resource limits.
- **Per-overlap rejection** — `find_vma_intersection` ensures heap growth cannot stomp on adjacent mappings (defense against pointer-overlap heap-grooming).
- **Per-`__vm_enough_memory`** — overcommit accounting prevents fork-bomb-style memory exhaustion.
- **Per `mmap_write_lock_killable`** — heap growth interruptible by SIGKILL, preventing kernel-side livelock.
- **Per `mm->def_flags`** — `mlockall(MCL_FUTURE)` semantics carry into new heap pages.
- **No-pre-fault default** — defense against information leak via uninitialized heap pages (kernel maps zero-page until first write).

### grsecurity/pax-style reinforcement

- **PAX_MPROTECT** — heap VMA is created without `VM_MAYEXEC` under PaX policy; subsequent `mprotect(addr, len, PROT_EXEC)` against heap pages is denied. Defense against ret2heap / heap-spray code execution.
- **PAX_NOEXEC** — `VM_DATA_DEFAULT_FLAGS` is rewritten to omit `VM_EXEC`/`VM_MAYEXEC` for ELF binaries lacking the `PT_PAX_FLAGS` exec marker. Heap is non-executable by default.
- **PAX_RANDMMAP** — initial `mm->start_brk` is randomized at exec time (independent of `mmap_base` randomization). Defense against deterministic heap-base prediction.
- **PAX_RANDEXEC** — irrelevant for heap (no exec), but PT_LOAD data segment relocation affects `mm->end_data` (the cap below which brk cannot shrink).
- **GRKERNSEC_BRUTE** — repeated `brk` failures (e.g., from VA-exhaustion brute) trigger throttling of the process and its forked descendants.
- **PAX_UDEREF** — `addr` is a numeric argument; SMAP/SMEP enforced. No user dereference inside the kernel path.
- **PAX_MEMORY_SANITIZE** — pages freed by `brk` shrink are sanitized via `__free_pages` → zero-on-free, preventing heap-residue infoleak.
- **RLIMIT_DATA hardening** — grsec extends with `chroot.deny_chroot_caps` and `grsec_resource_logging` so unprivileged `brk` overshoot is audited.
- **PAX_REFCOUNT** — `mm->mm_users` saturating refcount; reentrant `brk` from signal handler safe.
- **GRKERNSEC_HIDESYM** — error printks (e.g., `do_brk_flags` allocator failure) redact kernel pointers.
- **PAX_RANDKSTACK** — kstack offset rerolled at brk syscall entry.

