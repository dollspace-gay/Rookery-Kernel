# Tier-3: mm/virtual-memory — mm_struct, VMAs, page faults, GUP, rmap

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - mm/memory.c
  - mm/init-mm.c
  - mm/rmap.c
  - mm/gup.c
  - mm/maccess.c
  - mm/page-writeback.c
  - mm/util.c
  - mm/mlock.c
  - include/linux/mm.h
  - include/linux/mm_types.h
  - include/linux/rmap.h
  - include/linux/maple_tree.h
-->

## Summary
Tier-3 design for the per-process virtual-memory abstraction: `mm_struct` lifecycle, the VMA tree (maple-tree-keyed by address range), page-fault handling (anonymous fault, file-backed fault, COW, swap-in), GUP (get_user_pages), reverse-mapping (page → VMAs that map it), page-writeback orchestration, and `init_mm` (the kernel's own mm_struct).

This is **one of the four MANDATORY Layer-3-Kani-harness subsystems** per `00-overview.md` D4. VMA non-overlap, page-table-walker correctness, and folio refcount-with-pins invariants are mechanically verified.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Page-fault handler + COW + anon-fault + swap-in | `mm/memory.c` |
| `init_mm` (kernel's mm_struct) | `mm/init-mm.c` |
| Reverse-mapping (page → VMAs) | `mm/rmap.c`, `include/linux/rmap.h` |
| GUP (get_user_pages) — pinning userspace pages | `mm/gup.c` |
| Safe failable memory access | `mm/maccess.c` |
| Page writeback (dirty-page bookkeeping) | `mm/page-writeback.c` |
| Generic mm helpers | `mm/util.c` |
| `mlock` / `munlock` | `mm/mlock.c` |
| Public types | `include/linux/mm.h`, `include/linux/mm_types.h` |
| VMA tree backbone | `include/linux/maple_tree.h` (cross-ref `lib/data-structures.md`) |

## Compatibility contract

### `mm_struct` layout

`include/linux/mm_types.h` defines `struct mm_struct`. First-cache-line fields used by macro-accessing inline code in many subsystems; layout-equivalent to upstream for those fields. Deeper fields may reorganize.

Key fields (preserved):
- `mmap_lock` (rw_semaphore)
- `pgd` (page table root pointer)
- `mm_count`, `mm_users` (refcounts; saturating per `Refcount`)
- `start_code`, `end_code`, `start_data`, `end_data`, `start_brk`, `brk`, `start_stack` (visible via `/proc/<pid>/stat`, `/proc/<pid>/status`)
- `arg_start`, `arg_end`, `env_start`, `env_end` (visible via `/proc/<pid>/cmdline`, `/proc/<pid>/environ`)
- `mm_rb` (deprecated; replaced by maple tree `mm_mt`)
- `mm_mt` (maple tree of VMAs)
- `pinned_vm`, `data_vm`, `exec_vm`, `stack_vm` (page-class counters)
- `total_vm`, `locked_vm`, `pinned_vm`, `shared_vm` (visible via `/proc/<pid>/status` VmTotal, VmLck, VmPin, VmShr)
- `flags` (mm_flags, including PR_SET_MDWE bits + Rookery's `exec_gain_state` bits)

### `vm_area_struct` layout

`include/linux/mm_types.h` defines `struct vm_area_struct`. Visible via `/proc/<pid>/maps` formatted output. Field-equivalent to upstream:
- `vm_start`, `vm_end`
- `vm_flags` (VM_READ, VM_WRITE, VM_EXEC, VM_SHARED, VM_MAYREAD, VM_MAYWRITE, VM_MAYEXEC, VM_MAYSHARE, VM_GROWSDOWN, VM_PFNMAP, VM_LOCKED, VM_IO, VM_HUGETLB, VM_DONTEXPAND, VM_DONTDUMP, VM_MIXEDMAP, VM_HUGEPAGE, VM_NOHUGEPAGE, VM_MERGEABLE, VM_UFFD_*, VM_SOFTDIRTY, VM_SAO, VM_PKEY_*)
- `vm_file`, `vm_pgoff`
- `vm_ops` (vtable: `fault`, `map_pages`, `huge_fault`, `pmd_split_unmap`, `mremap`, `mprotect`, `name`, `set_policy`, `get_policy`, `find_special_page`)
- `anon_vma`, `anon_vma_chain` (anonymous-mapping reverse chain)
- `vm_private_data`

### `/proc/<pid>/maps` line format

Upstream: `<start>-<end> <perms> <offset> <dev>:<inode> <pathname>` per line, address-ordered. Format-identical to upstream. (Cross-ref `mm/00-overview.md` for the format declaration; this Tier-3 owns the underlying VMA iteration.)

### `/proc/<pid>/smaps` extended format

Per-VMA detailed accounting per `Documentation/filesystems/proc.rst`. This Tier-3 produces the underlying counters; format owned by `fs/00-overview.md` § proc.md.

### Page-fault decode

Page-fault `pf_handler(pt_regs, error_code, cr2)` (called from `arch/x86/paging.md`) dispatches to:
- VMA lookup at `cr2` via maple tree
- VMA-found, permission-OK → handle_pte_fault → page-in / COW / anon-fault path
- VMA-not-found → `__bad_area_nosemaphore` → SIGSEGV with SEGV_MAPERR
- VMA-found, permission-deny → `__bad_area` → SIGSEGV with SEGV_ACCERR
- Hardware-poisoned page → SIGBUS with BUS_MCEERR_AR / BUS_MCEERR_AO

Identical to upstream signal/si_code emission per `arch/x86/00-overview.md` § entry.md.

### GUP API

`get_user_pages` family pins userspace pages for kernel access. Compat:
- `get_user_pages` (now `get_user_pages_remote`), `get_user_pages_fast`, `pin_user_pages`, `pin_user_pages_fast`
- `FOLL_*` flags: WRITE, FORCE, GET, PIN, NUMA, LONGTERM, REMOTE, ANON, MM, SPLIT_PMD, PCI_P2PDMA, FAST_ONLY, INTERRUPTIBLE, NOWAIT, COMPOUND
- Per upstream's pin-vs-get distinction (PIN promises long-term ownership; GET is short-term)

### `mlock(2)` / `munlock(2)` / `mlockall(2)` semantics

`mm/mlock.c` implementation. Userspace-visible:
- VMA flag VM_LOCKED set / cleared
- `RLIMIT_MEMLOCK` enforced (matches upstream)
- `MCL_CURRENT` / `MCL_FUTURE` / `MCL_ONFAULT` semantics

Identical.

## Requirements

- REQ-1: `mm_struct` and `vm_area_struct` first-cache-line layout matches upstream so out-of-tree drivers + BPF programs accessing fields via inline macros work unmodified.
- REQ-2: VMA tree implementation uses maple tree (`lib/data-structures.md`); insertion preserves address order; non-overlap is invariant-checked.
- REQ-3: Page-fault handler dispatches per the Compatibility contract decode table; SIGSEGV / SIGBUS si_code emission byte-identical.
- REQ-4: Per-VMA flags set (VM_*) byte-identical bit positions; `/proc/<pid>/maps` permission column byte-identical (`r/w/x/p/s` per upstream's encoding).
- REQ-5: `mm_struct->mmap_lock` is an `rwsem` (sleepable rw); read-side: page faults, /proc readers; write-side: mmap, mprotect, mremap, munmap, brk. Lock-ordering matches upstream (mmap_lock outermost; pte_lockptr innermost).
- REQ-6: Per-VMA fast-path lock (per-VMA spinlock, Linux 6.x+) is honored; page-fault fast path acquires per-VMA lock without holding mmap_lock.
- REQ-7: GUP API set + FOLL_* flags match upstream byte-for-byte.
- REQ-8: `mlock` / `munlock` / `mlockall` semantics + RLIMIT_MEMLOCK enforcement byte-identical.
- REQ-9: COW (copy-on-write) handling preserves upstream's COW-break behavior; `_mapcount` / `_refcount` interaction byte-identical.
- REQ-10: Anonymous-page reverse-mapping (`anon_vma_chain`) preserves the per-anon-vma chain → per-page reverse-walk algorithm.
- REQ-11: File-page reverse-mapping (via `vm_ops->find_special_page` and the file's address_space) preserves upstream's walking order.
- REQ-12: VMA non-overlap Layer-3 Kani harness MANDATORY (per `mm/00-overview.md` REQ-14 / `00-overview.md` D4).
- REQ-13: Page-table walker invariants (per `arch/x86/00-overview.md` Layer-3) cross-reference: this component consumes the walker for fault resolution.
- REQ-14: TLA+ model `models/mm/mmap_lock.tla` proves rwsem-based mmap_lock + per-VMA-lock fast path correctly serializes readers vs. writers without livelock under page-fault retry.
- REQ-15: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `pahole struct mm_struct` and `pahole struct vm_area_struct` produce byte-identical first-cache-line layout vs. upstream. (covers REQ-1)
- [ ] AC-2: A test creates 1000 VMAs in arbitrary order; iterating returns them in address order; a `find(addr)` returns the correct VMA. Maple-tree non-overlap invariant harness passes. (covers REQ-2, REQ-12)
- [ ] AC-3: A page-fault test program triggers each of: in-VMA-resolved, out-of-VMA → SIGSEGV(MAPERR), permission-deny → SIGSEGV(ACCERR), hwpoison → SIGBUS(MCEERR_AR). Each produces upstream-identical output. (covers REQ-3)
- [ ] AC-4: Diff of `/proc/<pid>/maps` permission column for a curated test program (specific mmap pattern) is empty between Rookery and upstream. (covers REQ-4)
- [ ] AC-5: An `mmap` from one CPU concurrent with a page fault from another CPU on a different VMA address completes correctly (per-VMA lock fast path doesn't block on the mmap_lock writer). (covers REQ-5, REQ-6)
- [ ] AC-6: GUP selftest (`tools/testing/selftests/mm/gup_test.c`) passes with the same set as upstream. (covers REQ-7)
- [ ] AC-7: mlock selftest passes (`tools/testing/selftests/mm/mlock-*`). (covers REQ-8)
- [ ] AC-8: A fork → child writes shared page COW test produces the same observable behavior on Rookery and upstream. `_mapcount` + `_refcount` after the COW break are identical. (covers REQ-9)
- [ ] AC-9: A test that walks an anon page's reverse-map chain produces the same VMA list on Rookery and upstream. (covers REQ-10)
- [ ] AC-10: A test that walks a file page's reverse-map (via `address_space->i_mmap`) produces the same VMA list. (covers REQ-11)
- [ ] AC-11: `make verify` passes `kani::proofs::mm::vma::*` Kani harnesses (mandatory L3). (covers REQ-12)
- [ ] AC-12: `make tla` passes `models/mm/mmap_lock.tla`. (covers REQ-14)
- [ ] AC-13: Hardening section present and follows template. (covers REQ-15)

## Architecture

### Rust module organization

- `kernel::mm::mm_struct::MmStruct` — per-process mm
- `kernel::mm::vma::Vma` — VMA wrapper
- `kernel::mm::vma::VmaTree` — maple-tree-backed VMA collection
- `kernel::mm::fault` — page-fault handler dispatcher
- `kernel::mm::fault::anon` — anonymous-page-fault handling
- `kernel::mm::fault::file` — file-backed-page fault
- `kernel::mm::fault::cow` — COW break
- `kernel::mm::fault::swap` — swap-in (cross-ref `mm/swap.md`)
- `kernel::mm::gup` — GUP family
- `kernel::mm::rmap::anon` — anon reverse-map
- `kernel::mm::rmap::file` — file reverse-map
- `kernel::mm::mlock` — mlock/munlock/mlockall
- `kernel::mm::page_writeback` — dirty-page bookkeeping
- `kernel::mm::init_mm::INIT_MM` — kernel's own mm_struct (static)

### Locking and concurrency

- **mm_struct->mmap_lock**: rwsem. Read-side acquired in fault path (then dropped to per-VMA lock); write-side in mmap/mprotect/mremap/munmap/brk. Lock-ordering: mmap_lock outermost.
- **per-VMA lock**: spinlock. Fast-path fault acquires this without mmap_lock; if VMA changed (writer in-flight), retry under mmap_lock.
- **anon_vma->root->rwsem**: rwsem. Held during anon-VMA chain traversal.
- **address_space->i_mmap_rwsem**: rwsem. File-backed mapping's reverse-map lock.
- **pte_lockptr** (per-PMD spinlock, x86): innermost. Held during PTE updates.
- **page_table_check** (debug): per-PFN bitmap; atomic.

### Error handling

- `Err(EFAULT)` — userspace addr out of range / not mapped
- `Err(ENOMEM)` — page-table allocation failed
- `Err(EAGAIN)` — fault retry (per-VMA fast path detected concurrent writer)
- `Err(ESRCH)` — `/proc/<pid>/maps` reader's target task gone
- `Err(EINTR)` — sleepable wait interrupted

Page-fault outcomes are NOT `Result`; they're `PageFaultOutcome` enum (`Continue | Retry | DeliverSignal { sig, info }`) per `mm/00-overview.md` Error handling section.

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Maple-tree slot pointer (range walk) | `kani::proofs::mm::vma::maple_walk_safety` (cross-ref `lib/data-structures.md`) |
| VMA list manipulation under per-VMA lock | `kani::proofs::mm::vma::list_safety` |
| Page-table walker callback (consumed by fault) | `kani::proofs::mm::fault::pgwalk_safety` (cross-ref `arch/x86/paging.md`) |
| GUP raw pin-page operations | `kani::proofs::mm::gup::pin_safety` |
| Anon-VMA chain pointer chase | `kani::proofs::mm::rmap::anon_walk_safety` |
| Folio refcount with pins | `kani::proofs::mm::folio::refcount_with_pins_safety` (inherited) |

### Layer 2: TLA+ models

- `models/mm/mmap_lock.tla` (mandatory per `mm/00-overview.md` Layer 2) — proves rwsem-based mmap_lock + per-VMA-lock fast path correctly serializes without livelock.
- `models/mm/anon_vma_chain.tla` (NEW) — proves anon-VMA reverse-map chain integrity under fork+exec interleaving.
- `models/mm/cow_break.tla` (NEW) — proves COW break preserves _mapcount + _refcount invariants under racing writers.

### Layer 3: Kani harnesses for data-structure invariants (MANDATORY per `00-overview.md` D4)

| Data structure | Invariant | Harness |
|---|---|---|
| VMA maple tree | "No two VMAs in one mm overlap"; "Tree iterates in address order" | `kani::proofs::mm::vma::no_overlap_invariant` |
| Anon-VMA chain | "Every page in an anon-VMA is reachable from at least one mm via the chain" | `kani::proofs::mm::rmap::anon_chain_invariants` |
| File-backed reverse map | "Every page in `address_space->i_pages` (xarray) has every VMA mapping it on its `i_mmap` interval tree" | `kani::proofs::mm::rmap::file_map_invariants` |
| Folio refcount + pin count | "0 ≤ pin_count ≤ refcount; refcount=0 implies folio is freeable" | `kani::proofs::mm::folio::refcount_pins_invariants` |

### Layer 4: Functional correctness (opt-in)

- **VMA merge / split arithmetic** via Creusot — proves: merging adjacent VMAs with compatible flags produces a single VMA covering the union; splitting at a midpoint produces two VMAs covering the original range.
- **COW break determinism** via Verus — proves: after COW break, the writer task observes its own writes; the reader task observes the original.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **MPROTECT-W→X-block** (mm-side enforcement) | `mprotect` rejects W→X transitions for tasks lacking exec_gain_state; cross-ref `arch/x86/entry.md` for the per-task state read; cross-ref `mm/mmap.md` for the mprotect syscall handler | § Default-on system-wide with per-process exemption |
| **NOEXEC-strict** (mm-side enforcement) | mmap of anon mapping with PROT_EXEC + PROT_WRITE rejected unless task has exec_gain_state | § Default-on system-wide with per-process exemption |
| **NOEXEC** (default mode for stack/heap/anon) | Default for ELF without explicit executable-stack note (matches upstream PT_GNU_STACK default) | § Default-on configurable off |

### Row-1 features consumed by this component

- **UDEREF**: every userspace-pointer access goes through `UserPtr<T>`; raw user-VA dereferencing forbidden
- **REFCOUNT**: `mm_struct->mm_count` + `mm_struct->mm_users` are `Refcount` (saturating)
- **AUTOSLAB**: `vm_area_struct` allocated via `KmemCache::<Vma>::new()`; per-type slab cache
- **MEMORY_SANITIZE**: VMA freed objects zeroed unless cache marked SLAB_NO_ZERO_ON_FREE
- **SIZE_OVERFLOW**: address arithmetic in faults uses checked operators
- **CONSTIFY**: `vm_operations_struct` vtables are `static const`

### Row-2 / GR-RBAC integration

`mmap` / `mprotect` / `mremap` / `madvise` / `mlock` syscalls are LSM hook points (`security_mmap_addr`, `security_mmap_file`, `security_file_mprotect`). GR-RBAC policy can deny memory operations beyond what other LSMs enforce. The hooks themselves are dispatched in `mm/mmap.md`'s syscall handlers; this Tier-3 honors the deny decisions.

### Userspace-visible behavior changes

Per Axiom 4 of `00-security-principles.md`:
- **MPROTECT-W→X-block default-on**: a JIT runtime without the `NT_ROOKERY_SECURITY_FLAGS` ELF note OR `prctl(PR_REQUEST_EXEC_GAIN)` cannot mprotect W→X. Distros must patch their JIT runtime packages.
- All other behaviors match upstream.

### Verification

(See § Verification above.)

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — every GUP-pinned page used as a user-copy buffer is validated against slab usercopy zones; raw bytes never bypass the check.
- **PAX_KERNEXEC** — fault handler .text + rodata are RX/RO; vtable dispatch through `vm_operations_struct` is CONSTIFY'd.
- **PAX_RANDKSTACK** — per-page-fault kernel-stack randomization on every entry from `arch/x86/entry.md`'s `do_page_fault`.
- **PAX_REFCOUNT** — `mm_struct->mm_count`, `mm_struct->mm_users`, `anon_vma->refcount`, and folio refcounts are saturating `Refcount` types.
- **PAX_MEMORY_SANITIZE** — VMA cache freed objects zeroed (unless cache marked `SLAB_NO_ZERO_ON_FREE`); anon page contents zeroed on free.
- **PAX_MEMORY_STACKLEAK** — kernel-stack residual zeroing on every fault-return path.
- **PAX_RAP / kCFI** — indirect-call type-signature enforcement on `vm_operations_struct->fault`, `->map_pages`, `->huge_fault`, `->mremap`, `->mprotect`, `->find_special_page`.
- **GRKERNSEC_HIDESYM** — kernel pointers hidden from `/proc/<pid>/maps`, `/proc/<pid>/smaps`, `/proc/<pid>/pagemap`, `/proc/<pid>/numa_maps` for non-CAP_SYSLOG readers; defeats kASLR-bypass via maps leak.
- **GRKERNSEC_DMESG** — page-fault dmesg lines (segfault details, RIP/RSP/CR2) restricted to CAP_SYSLOG; defeats fingerprinting of executable layout via dmesg-scraping.
- **PAX_ASLR** — collaborates with `mm/mmap.md` for mmap-base, stack-base, heap-base randomization; per-mm randomization seed.
- **PAX_RANDMMAP** — mmap base + each VMA placement randomized (cross-ref `mm/mmap.md`); GUP-acquired pinned pages never expose deterministic mappings.
- **PAX_MPROTECT** — fault handler honors W^X policy per-task `exec_gain_state`; W→X transitions denied at the VMA level by `mm/mmap.md`, enforced here on subsequent fault.
- **PAX_PAGEEXEC** — non-executable pages enforced via hardware NX; PROT_EXEC absence translates to NX bit set in PTE installed by fault handler.
- **PAX_NOEXEC** — anon pages default to non-executable unless ELF PT_GNU_STACK or per-task exec_gain explicitly grants exec.
- **PAX_UDEREF** — every user-pointer access (faulthandler, GUP, /proc/<pid>/maps readers) goes through `UserPtr<T>`; raw user-VA dereferencing forbidden.
- **GRKERNSEC_KSTACKOVERFLOW** — kernel stack overflow detection at every recursive fault path (e.g., reclaim-during-fault).
- **GRKERNSEC_OOM_DENY** — fault path's invocation of OOM consults grsec OOM-deny policy.

Per-doc rationale: virtual memory is the seam between userspace addresses and kernel data; an attacker who can confuse a VMA lookup, race a GUP pin against a truncate, or smuggle an executable mapping through `mprotect` has won by definition. PAX_UDEREF prevents the kernel from ever blindly dereferencing a user VA; PAX_REFCOUNT saturates `mm_count`/`mm_users` so an attacker cannot wrap-to-zero a forked-mm refcount; PAX_RAP/kCFI enforces type signatures on every fault-vtable dispatch to defeat function-pointer hijacks; GRKERNSEC_HIDESYM denies the `/proc/<pid>/maps` leak that has been a perennial source of kASLR breaks. The PAX_MPROTECT + PAX_PAGEEXEC + PAX_NOEXEC trio enforces W^X at every level of the fault-resolution pipeline.

## Open Questions

(none — virtual memory semantics are exhaustively specified by POSIX + Linux extensions; no architectural ambiguities at this tier)

## Out of Scope

- mmap / mprotect / mremap / munmap syscall implementations (cross-ref `mm/mmap.md`)
- Page-cache / file backing (cross-ref `mm/page-cache.md`)
- Memcg charging (cross-ref `mm/memcg.md`)
- 32-bit-only paths
- Implementation code
