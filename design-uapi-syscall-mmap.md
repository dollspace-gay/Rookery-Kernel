---
title: "Tier-5: syscall 9 — mmap(2)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`mmap(2)` is **x86_64 syscall 9**, the core memory-mapping syscall. It establishes a new VMA in `current->mm` of `len` bytes (page-aligned) with protection bits `prot`, mapped from offset `offset` of `fd` (or anonymous if `MAP_ANONYMOUS`). On x86_64, the kernel exposes it as `SYSCALL_DEFINE6(mmap_pgoff, ...)` and the architecture wrapper shifts `offset >> PAGE_SHIFT` before calling `ksys_mmap_pgoff` → `vm_mmap_pgoff` → `do_mmap` → `mmap_region`. `mmap` interacts with the page allocator, VFS (`f_op->mmap`), VMA-merging, anonymous reverse-map (anon_vma), MLOCK, hugetlb, MTE/pkeys, and the OOM heuristic. It is also the syscall most exercised by grsecurity/PaX hardening: `PAX_MPROTECT`, `PAX_NOEXEC`, `READ_IMPLIES_EXEC`, `PAX_RANDMMAP`.

Critical for: every libc `malloc` (anonymous private map), every `dlopen` / dynamic linker (file-backed RX), every JIT compiler (RW → RX flip via `mprotect`), every shared-memory IPC (`MAP_SHARED`), every persistent-memory user (`MAP_SYNC`).

### Acceptance Criteria

- [ ] AC-1: `mmap(NULL, 4096, PROT_READ, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0)` returns a page-aligned address; accessing it reads zeros.
- [ ] AC-2: `mmap(NULL, 4096, PROT_NONE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0)` returns success; read/write/exec on the page raises SIGSEGV.
- [ ] AC-3: `mmap(NULL, 0, ..., MAP_ANONYMOUS, -1, 0) == -EINVAL`.
- [ ] AC-4: `mmap(NULL, 4096, PROT_READ, MAP_PRIVATE, badfd, 0) == -EBADF`.
- [ ] AC-5: `mmap(NULL, 4096, PROT_WRITE, MAP_SHARED, ro_fd, 0) == -EACCES`.
- [ ] AC-6: `mmap(NULL, 4096, PROT_EXEC, MAP_PRIVATE, noexec_fd, 0) == -EPERM`.
- [ ] AC-7: `mmap(NULL, len, ..., MAP_PRIVATE, fd, 1) == -EINVAL` (offset not page-aligned).
- [ ] AC-8: `mmap(NULL, very_large, ..., MAP_ANONYMOUS) == -ENOMEM` once `RLIMIT_AS` or `sysctl_max_map_count` exceeded.
- [ ] AC-9: `mmap(addr, len, ..., MAP_FIXED_NOREPLACE, ...)` with existing mapping ⟹ `-EEXIST`.
- [ ] AC-10: `mmap(NULL, len, PROT_WRITE|PROT_EXEC, MAP_PRIVATE|MAP_ANONYMOUS)` under PAX_MPROTECT ⟹ `-EACCES`.
- [ ] AC-11: `mmap(NULL, len, ..., MAP_POPULATE, fd, 0)` pre-faults all pages before returning.
- [ ] AC-12: `mmap` of huge-page-aligned region with `MAP_HUGETLB|MAP_HUGE_2MB` succeeds when hugepages available.
- [ ] AC-13: Two `mmap(MAP_SHARED)` calls on the same file at the same offset see each other's writes.

### Architecture

```
struct MmapArgs {
  addr: u64, len: usize, prot: u64, flags: u64, fd: i32, offset: i64,
}
```

`sys_mmap(args) -> u64`:

1. If `args.offset & !PAGE_MASK != 0` ⟹ return `-EINVAL`.
2. `let pgoff = args.offset as u64 >> PAGE_SHIFT;`
3. Return `ksys_mmap_pgoff(addr, len, prot, flags, fd, pgoff)`.

`Mm::ksys_mmap_pgoff(addr, len, prot, flags, fd, pgoff) -> u64`:

1. `let mut file: Option<File> = None;`
2. If `!(flags & MAP_ANONYMOUS)`:
   - `audit_mmap_fd(fd, flags);`
   - `file = Some(fget(fd)?);`
   - If `is_file_hugepages(&file)` ⟹ align `len` up to huge_page_size.
   - Else if `flags & MAP_HUGETLB` ⟹ goto `out_fput` with `-EINVAL`.
3. Else if `flags & MAP_HUGETLB`:
   - Look up hstate by `(flags >> MAP_HUGE_SHIFT) & MAP_HUGE_MASK`.
   - If no hstate ⟹ `-EINVAL`.
   - `len = ALIGN(len, huge_page_size(hs));`
   - `file = Some(hugetlb_file_setup(...))`.
4. `let retval = vm_mmap_pgoff(file, addr, len, prot, flags, pgoff);`
5. If `file.is_some()` ⟹ `fput(file)`.
6. Return `retval`.

`Mm::vm_mmap_pgoff(file, addr, len, prot, flags, pgoff) -> u64`:

1. `mmap_write_lock(current.mm);`
2. `let mut populate = 0;`
3. `let ret = do_mmap(file, addr, len, prot, flags, 0, pgoff, &mut populate, None);`
4. `mmap_write_unlock(current.mm);`
5. If `!IS_ERR_VALUE(ret) && populate > 0` ⟹ `mm_populate(ret, populate);`
6. Return `ret`.

`Mm::do_mmap(...)` (key invariants):

1. Assert `mmap_lock` write-held.
2. `if !len ⟹ -EINVAL`.
3. Apply `READ_IMPLIES_EXEC` transformation.
4. `if flags & MAP_FIXED_NOREPLACE ⟹ flags |= MAP_FIXED`.
5. `len = PAGE_ALIGN(len); if !len ⟹ -ENOMEM`.
6. Overflow check `pgoff + (len>>PAGE_SHIFT) < pgoff ⟹ -EOVERFLOW`.
7. `if mm.map_count > get_sysctl_max_map_count() ⟹ -ENOMEM`.
8. If `prot == PROT_EXEC` ⟹ try `execute_only_pkey`.
9. Compute `vm_flags = calc_vm_prot_bits(prot, pkey) | calc_vm_flag_bits(file, flags) | mm.def_flags | VM_MAYREAD|MAYWRITE|MAYEXEC`.
10. `addr = get_unmapped_area(file, addr, len, pgoff, flags);` (if not FIXED).
11. `addr = mmap_region(file, addr, len, vm_flags, pgoff, uf);`
12. If `MAP_LOCKED || MAP_POPULATE && !MAP_NONBLOCK` ⟹ `*populate = len`.
13. Return `addr`.

### Out of Scope

- `mmap2(2)` (32-bit legacy with `pgoff` parameter; not in x86_64 ABI).
- `munmap(2)` (separate Tier-5 if expanded).
- `mremap(2)` (separate Tier-5).
- `mprotect(2)` (separate Tier-5 — `mprotect.md`).
- `msync(2)` (separate Tier-5).
- DAX / persistent-memory pmem internals (`MAP_SYNC`).
- io_uring fixed buffers.
- Implementation code.

### signature

C (POSIX / man-pages):

```c
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
```

glibc wrapper: `__mmap` → `INLINE_SYSCALL(mmap, 6, addr, length, prot, flags, fd, offset)`.

Kernel SYSCALL_DEFINE (x86_64 arch wrapper shifts `offset` to `pgoff`):

```c
SYSCALL_DEFINE6(mmap, unsigned long, addr, unsigned long, len,
                unsigned long, prot, unsigned long, flags,
                unsigned long, fd, off_t, offset);
/* arch wrapper validates offset & ~PAGE_MASK and calls: */
SYSCALL_DEFINE6(mmap_pgoff, unsigned long, addr, unsigned long, len,
                unsigned long, prot, unsigned long, flags,
                unsigned long, fd, unsigned long, pgoff);
```

Rookery dispatch:

```rust
pub fn sys_mmap(addr: u64, len: usize, prot: u64, flags: u64,
                fd: i32, offset: i64) -> SyscallResult<u64>;
```

### parameters

| name   | type            | constraints                                                       | errno-on-bad |
|--------|-----------------|-------------------------------------------------------------------|--------------|
| addr   | `void *`        | hint (or required if `MAP_FIXED`); page-aligned                    | `EINVAL` (FIXED + misalign) |
| length | `size_t`        | `> 0`; page-aligned on round-up                                    | `EINVAL` / `ENOMEM` |
| prot   | `int`           | subset of `PROT_NONE|READ|WRITE|EXEC|SEM`, plus `PROT_GROWSDOWN`/`GROWSUP` (mprotect-side)  | `EINVAL`     |
| flags  | `int`           | exactly one of `MAP_SHARED` / `MAP_SHARED_VALIDATE` / `MAP_PRIVATE`; any combination of others | `EINVAL`     |
| fd     | `int`           | open fd unless `MAP_ANONYMOUS`; must have read for `MAP_PRIVATE`, read+write for `MAP_SHARED` write | `EBADF` / `EACCES` |
| offset | `off_t`         | page-aligned (`offset & ~PAGE_MASK == 0`)                          | `EINVAL`     |

### return value

- Success: page-aligned virtual address `>= mmap_min_addr` of the new mapping.
- Failure: `MAP_FAILED` (i.e., `(void *)-1`) and `errno` is set. Kernel returns a negative errno in `%rax`; glibc converts.

### errors

| errno      | condition                                                                                            |
|------------|------------------------------------------------------------------------------------------------------|
| `EINVAL`   | `length == 0`, `prot` invalid combination, `flags` invalid combination, `offset` not page-aligned, `addr` not page-aligned with `MAP_FIXED`, `MAP_HUGETLB` invalid huge-page size, `MAP_SHARED_VALIDATE` with unknown flag bits, `MAP_FIXED_NOREPLACE` overlaps existing mapping. |
| `EBADF`    | `fd` not open and `MAP_ANONYMOUS` not set.                                                            |
| `EACCES`   | `MAP_PRIVATE` with fd not opened for read; `MAP_SHARED|PROT_WRITE` with fd not opened for write; LSM denial; filesystem noexec with `PROT_EXEC`. |
| `EPERM`    | `PROT_EXEC` denied under `PAX_MPROTECT`; `MAP_FIXED` over a sealed range; `addr < mmap_min_addr` for unprivileged. |
| `ENOMEM`   | `current->mm->map_count > sysctl_max_map_count`; or no free VA range satisfies; or `RLIMIT_AS`/`RLIMIT_DATA` exceeded; or overcommit denied; or post-`PAGE_ALIGN` length becomes 0 (overflow). |
| `EOVERFLOW`| `pgoff + (len >> PAGE_SHIFT) < pgoff` overflow.                                                       |
| `ETXTBSY`  | `MAP_DENYWRITE` requested and writers exist (legacy; ignored on modern kernels).                      |
| `EAGAIN`   | `RLIMIT_MEMLOCK` exceeded for `MAP_LOCKED`.                                                          |
| `ENODEV`   | Filesystem does not support `mmap`.                                                                  |
| `ENFILE`   | System-wide open-file limit hit when allocating hugetlb file.                                        |
| `EEXIST`   | `MAP_FIXED_NOREPLACE` and existing mapping overlaps.                                                 |
| `EINTR`    | Signal during long blocking init (e.g., hugepage reservation).                                       |

### abi surface (constants + flags)

### `prot` (mutually composable):

- `PROT_NONE`   `0x0` — no access (signals SIGSEGV on touch).
- `PROT_READ`   `0x1`.
- `PROT_WRITE`  `0x2`.
- `PROT_EXEC`   `0x4`.
- `PROT_SEM`    `0x8` — page may be used for atomic ops (arch-specific; tile/parisc).
- `PROT_GROWSDOWN` `0x01000000` (mprotect-only; extend to stack start).
- `PROT_GROWSUP`   `0x02000000` (mprotect-only; extend to stack end).

### `flags` — mapping type (exactly one required):

- `MAP_SHARED`           `0x01` — writes visible to others sharing the same file/inode; persisted to file.
- `MAP_PRIVATE`          `0x02` — copy-on-write; writes private to this mm.
- `MAP_SHARED_VALIDATE`  `0x03` (`= MAP_SHARED | MAP_PRIVATE`) — like `MAP_SHARED` but unknown flag bits ⟹ `-EINVAL`.

### `flags` — combinable modifiers:

- `MAP_FIXED`            `0x10` — interpret `addr` exactly; unmap any overlapping mappings.
- `MAP_ANONYMOUS`        `0x20` — no backing file; pages zeroed.
- `MAP_GROWSDOWN`        `0x0100` — stack-like (extend downward).
- `MAP_DENYWRITE`        `0x0800` — legacy (`ETXTBSY` on writers); ignored.
- `MAP_EXECUTABLE`       `0x1000` — mark binary mapping (legacy hint).
- `MAP_LOCKED`           `0x2000` — `mlock` the pages.
- `MAP_NORESERVE`        `0x4000` — don't reserve swap space.
- `MAP_POPULATE`         `0x008000` — pre-fault pages.
- `MAP_NONBLOCK`         `0x010000` — only with `MAP_POPULATE`; skip prefault if I/O would block.
- `MAP_STACK`            `0x020000` — hint: use process-stack-suitable region (arch-dependent).
- `MAP_HUGETLB`          `0x040000` — back with hugetlb pages.
- `MAP_HUGE_SHIFT`       `26` (high bits hold page-size selector when `MAP_HUGETLB`).
- `MAP_HUGE_MASK`        `0x3f`.
- `MAP_SYNC`             `0x080000` — synchronous page faults (DAX persistent memory).
- `MAP_FIXED_NOREPLACE`  `0x100000` — like `MAP_FIXED` but `-EEXIST` instead of replacing.
- `MAP_UNINITIALIZED`    `0x4000000` — anonymous map may contain previous contents (no-mmu only).

### Architectural derived flags inside the kernel (not user ABI but transformed from `prot`/`flags`):

- `VM_READ`, `VM_WRITE`, `VM_EXEC`, `VM_SHARED`.
- `VM_MAYREAD`, `VM_MAYWRITE`, `VM_MAYEXEC`, `VM_MAYSHARE` (cap bits).
- `VM_LOCKED`, `VM_HUGETLB`, `VM_NORESERVE`, `VM_PFNMAP`.

### compatibility contract

- REQ-1: Argument lowering: `%rdi=addr`, `%rsi=len`, `%rdx=prot`, `%r10=flags`, `%r8=fd`, `%r9=offset`.
- REQ-2: Architecture wrapper checks `offset & ~PAGE_MASK == 0` ⟹ else `-EINVAL`, then computes `pgoff = offset >> PAGE_SHIFT`.
- REQ-3: `length` is page-aligned via `PAGE_ALIGN(len)`; if rounding overflows to `0` ⟹ `-ENOMEM`.
- REQ-4: `length == 0` ⟹ `-EINVAL`.
- REQ-5: `pgoff + (len >> PAGE_SHIFT) < pgoff` overflow ⟹ `-EOVERFLOW`.
- REQ-6: `current->mm->map_count > sysctl_max_map_count` ⟹ `-ENOMEM`.
- REQ-7: `MAP_ANONYMOUS`: `fd` ignored; `pgoff` ignored (or used as hugetlb-size selector via `MAP_HUGE_SHIFT`).
- REQ-8: `!MAP_ANONYMOUS`: `fget(fd)` must succeed; else `-EBADF`. File's `f_op->mmap` invoked inside `mmap_region`.
- REQ-9: `MAP_HUGETLB` with `!MAP_ANONYMOUS`: file must be `is_file_hugepages(file)`; else `-EINVAL`. With `MAP_ANONYMOUS`, kernel allocates anonymous hugetlb file via `hugetlb_file_setup`.
- REQ-10: `READ_IMPLIES_EXEC` personality + `prot & PROT_READ` ⟹ `prot |= PROT_EXEC` (unless backing filesystem is noexec).
- REQ-11: `MAP_FIXED_NOREPLACE` is treated as `MAP_FIXED` for VA selection, but VMA overlap ⟹ `-EEXIST`.
- REQ-12: Without `MAP_FIXED`, `addr = round_hint_to_min(addr)`; `get_unmapped_area` chooses a free range.
- REQ-13: For `prot == PROT_EXEC` exclusive (no R/W), kernel attempts to allocate an execute-only memory protection key via `execute_only_pkey(mm)`.
- REQ-14: `MAP_LOCKED` requires `RLIMIT_MEMLOCK` budget; else `-EAGAIN`. `mlock` happens post-fault.
- REQ-15: `MAP_NORESERVE` honored only if `sysctl_overcommit_memory != OVERCOMMIT_NEVER`.
- REQ-16: `MAP_POPULATE` ⟹ post-map `*populate = len` and the kernel pre-faults via `mm_populate(addr, len)`.
- REQ-17: `MAP_NONBLOCK` modifies `MAP_POPULATE` semantics: only pre-fault pages already resident.
- REQ-18: `MAP_SHARED|PROT_WRITE` on file fd not opened `O_RDWR`/`O_WRONLY` ⟹ `-EACCES`.
- REQ-19: `MAP_PRIVATE` on file fd not opened with read access ⟹ `-EACCES`.
- REQ-20: `PROT_EXEC` on a `noexec`-mounted file ⟹ `-EPERM` (after `path_noexec(&file->f_path)` check).
- REQ-21: `addr < mmap_min_addr` (default `4096`) without `CAP_SYS_RAWIO` ⟹ `-EPERM`.
- REQ-22: VMA-merging happens transparently after `mmap_region`; returned `addr` may land inside a merged VMA whose extent exceeds `[addr, addr+len)`.
- REQ-23: On success, `f_pos` of `fd` is **not** advanced; `mmap` does not consume bytes the way `read` does.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `offset_page_aligned` | INVARIANT | `offset & ~PAGE_MASK == 0`. |
| `len_nonzero_post_align` | INVARIANT | `PAGE_ALIGN(len) != 0`. |
| `pgoff_no_overflow` | INVARIANT | `pgoff + (len>>PAGE_SHIFT)` does not wrap. |
| `map_count_within_sysctl` | INVARIANT | Post-mmap, `mm.map_count <= sysctl_max_map_count`. |
| `vm_flags_consistent` | INVARIANT | `vm_flags & VM_EXEC ⟹ prot & PROT_EXEC` (or `READ_IMPLIES_EXEC`). |
| `mmap_lock_write_held` | INVARIANT | `do_mmap` runs under `mmap_write_lock`. |
| `fixed_noreplace_no_overlap` | INVARIANT | `MAP_FIXED_NOREPLACE` post-mmap ⟹ existing mappings unchanged. |

### Layer 2: TLA+

`uapi/syscalls/mmap.tla`:
- Per-call → arg validation → fget → mmap_lock → do_mmap → fput → populate.
- Properties:
  - `safety_lock_held_during_mutation`,
  - `safety_no_overflow_in_pgoff_arith`,
  - `safety_PROT_to_VM_translation`,
  - `liveness_mmap_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post: returned `addr` is page-aligned and in `[TASK_UNMAPPED_BASE, TASK_SIZE)` | `Mm::do_mmap` |
| Post: new VMA has `vm_flags` consistent with `prot|flags` | `Mm::mmap_region` |
| `MAP_FIXED_NOREPLACE` and overlap ⟹ `-EEXIST` | `Mm::mmap_region` |
| `MAP_LOCKED ⟹ populate = len` | `Mm::do_mmap` |

### Layer 4: Verus/Creusot functional

`mmap(addr, len, prot, flags, fd, offset) ≡ POSIX.1-2024 mmap` semantic equivalence (with Linux extensions per `man 2 mmap`).

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`mmap(2)` reinforcement:

- **Per-`mmap_min_addr`** — defense against NULL-pointer kernel exploit (low pages unmappable without CAP).
- **Per-`sysctl_max_map_count`** — defense against VMA-exhaustion DoS.
- **Per-`RLIMIT_AS` / `RLIMIT_DATA` / `RLIMIT_MEMLOCK`** — defense against memory-resource DoS.
- **Per-`MAP_FIXED_NOREPLACE`** — defense against silent overwrite of existing mappings.
- **Per-`path_noexec` check** — defense against PROT_EXEC on noexec-mounted filesystem (e.g., `/tmp` noexec).
- **Per-`audit_mmap_fd`** — every file-backed mmap recorded.
- **Per-`execute_only_pkey`** — defense against side-channel reads of code pages (PROT_EXEC without PROT_READ).
- **Per-`READ_IMPLIES_EXEC` only when filesystem allows** — defense against forcing PROT_EXEC on noexec-mounted file.
- **Per-`PROT_WRITE+MAP_SHARED` requires `FMODE_WRITE`** — defense against writing read-only files.
- **Per-`mmap_lock` write-held** — defense against concurrent VMA-table mutation.
- **Per-`mm_populate` post-unlock** — defense against deadlock during page fault during populate.

### grsecurity/pax-style reinforcement

- **PAX_MPROTECT** — `PROT_WRITE|PROT_EXEC` simultaneously on a mapping prohibited; flipping a non-EXEC mapping to EXEC also prohibited unless granted via `chpax`. Forces W^X.
- **PAX_NOEXEC** — explicit denial of `PROT_EXEC` on anonymous/heap/stack mappings except for those marked via ELF flag or by the dynamic loader.
- **PAX_RANDMMAP** — `get_unmapped_area` randomizes the base of unhint'd mmaps (in addition to ASLR randomization of `mmap_base`).
- **PAX_RANDEXEC** — randomizes the load address of file-backed `PROT_EXEC` mappings (PIE / shared-objects).
- **GRKERNSEC_RWXMAP_LOG** — logs any attempt to create a writable+executable mapping (denied or allowed).
- **PAX_REFCOUNT** — `mm_struct.mm_users`, `file.f_count` saturating refcounts.
- **PAX_UDEREF** — `addr` not dereferenced kernel-side; SMAP guards any user copies.
- **PAX_MEMORY_SANITIZE** — freed pages (e.g., on `MAP_FIXED` overwrite) sanitized.
- **GRKERNSEC_HIDESYM** — error printks redact kernel pointers (VMA addresses).
- **GRKERNSEC_BRUTE** — repeated mmap-failure-driven crashes (BFD/ROP brute) detected and throttled.
- **PAX_RANDKSTACK** — kstack offset randomized at mmap syscall entry.

