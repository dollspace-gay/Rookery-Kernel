# Tier-3: mm/mmap — mmap/mprotect/mremap/munmap/madvise/mlock/mincore syscalls

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - mm/mmap.c
  - mm/mprotect.c
  - mm/mremap.c
  - mm/madvise.c
  - mm/mlock.c
  - mm/mincore.c
  - mm/util.c
  - arch/x86/mm/mmap.c
  - include/uapi/asm-generic/mman.h
  - include/uapi/asm-generic/mman-common.h
  - arch/x86/include/uapi/asm/mman.h
  - include/linux/mm.h
-->

## Summary
Tier-3 design for the mmap syscall family — the userspace-visible interface to virtual memory. Owns `mmap`, `mmap2`, `munmap`, `mremap`, `mprotect`, `pkey_mprotect`, `madvise`, `process_madvise`, `mlock`, `mlock2`, `munlock`, `mlockall`, `munlockall`, `mincore`, `msync`, `pkey_alloc`, `pkey_free`, `arch_prctl(ARCH_GET/SET_FS/GS)`. **Owns mm-side enforcement of MPROTECT-W→X-block and NOEXEC-strict** per `00-security-principles.md`'s default-on system-wide policy with per-process exemption.

This Tier-3 is the gatekeeper for every userspace memory operation. Drop-in compat (REQ-1 of `00-overview.md`) hinges almost entirely on these syscalls behaving identically to upstream.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| mmap / munmap / brk | `mm/mmap.c` |
| mprotect / pkey_mprotect | `mm/mprotect.c` |
| mremap | `mm/mremap.c` |
| madvise / process_madvise | `mm/madvise.c` |
| mlock / mlock2 / munlock / mlockall / munlockall | `mm/mlock.c` |
| mincore | `mm/mincore.c` |
| Generic mm helpers | `mm/util.c` |
| x86 arch-specific mmap policy | `arch/x86/mm/mmap.c` |
| UAPI mman.h | `include/uapi/asm-generic/mman.h`, `include/uapi/asm-generic/mman-common.h`, `include/uapi/asm/mman.h` |

## Compatibility contract

### Syscall set (each gets a Tier-5 `uapi/syscalls/<name>.md` in Phase D)

`mmap`, `mmap2` (legacy 32-bit), `munmap`, `mremap`, `mprotect`, `pkey_mprotect`, `madvise`, `process_madvise`, `mlock`, `mlock2`, `munlock`, `mlockall`, `munlockall`, `mincore`, `msync`, `pkey_alloc`, `pkey_free`, `arch_prctl(ARCH_SET_FS/GET_FS/SET_GS/GET_GS)`.

### MAP_* flags (mmap)

Bit-identical numeric values + semantics (per `include/uapi/asm-generic/mman*.h`):

`MAP_SHARED=1`, `MAP_PRIVATE=2`, `MAP_SHARED_VALIDATE=3`, `MAP_TYPE=0xf` (mask), `MAP_FIXED=0x10`, `MAP_ANONYMOUS=0x20`, `MAP_GROWSDOWN=0x100`, `MAP_DENYWRITE=0x800`, `MAP_EXECUTABLE=0x1000`, `MAP_LOCKED=0x2000`, `MAP_NORESERVE=0x4000`, `MAP_POPULATE=0x8000`, `MAP_NONBLOCK=0x10000`, `MAP_STACK=0x20000`, `MAP_HUGETLB=0x40000`, `MAP_SYNC=0x80000`, `MAP_FIXED_NOREPLACE=0x100000`, `MAP_UNINITIALIZED=0x4000000`, plus huge-page size hint bits (`MAP_HUGE_2MB`, `MAP_HUGE_1GB`).

### PROT_* flags

`PROT_READ=0x1`, `PROT_WRITE=0x2`, `PROT_EXEC=0x4`, `PROT_SEM=0x8`, `PROT_NONE=0x0`, `PROT_GROWSDOWN=0x01000000`, `PROT_GROWSUP=0x02000000`. Memory Protection Key bits PROT_PKEY{0..15} encoded in the high bits.

### MADV_* advice values

`MADV_NORMAL=0`, `MADV_RANDOM=1`, `MADV_SEQUENTIAL=2`, `MADV_WILLNEED=3`, `MADV_DONTNEED=4`, `MADV_FREE=8`, `MADV_REMOVE=9`, `MADV_DONTFORK=10`, `MADV_DOFORK=11`, `MADV_HWPOISON=100`, `MADV_SOFT_OFFLINE=101`, `MADV_MERGEABLE=12`, `MADV_UNMERGEABLE=13`, `MADV_HUGEPAGE=14`, `MADV_NOHUGEPAGE=15`, `MADV_DONTDUMP=16`, `MADV_DODUMP=17`, `MADV_WIPEONFORK=18`, `MADV_KEEPONFORK=19`, `MADV_COLD=20`, `MADV_PAGEOUT=21`, `MADV_POPULATE_READ=22`, `MADV_POPULATE_WRITE=23`, `MADV_DONTNEED_LOCKED=24`, `MADV_COLLAPSE=25`, `MADV_GUARD_INSTALL=102`, `MADV_GUARD_REMOVE=103`. Identical numeric values + semantics.

### MREMAP_* flags

`MREMAP_MAYMOVE=1`, `MREMAP_FIXED=2`, `MREMAP_DONTUNMAP=4`. Identical.

### Errno semantics

| Syscall | Errno | Condition |
|---|---|---|
| `mmap` | EINVAL | bad flags / bad alignment |
| `mmap` | ENOMEM | no available address range |
| `mmap` | EAGAIN | RLIMIT_AS exceeded |
| `mmap` | EACCES | file mode incompatible with prot |
| `mmap` | ENODEV | filesystem doesn't support mmap |
| `mprotect` | EACCES | **(Rookery-new on default-on path)** W→X transition denied without exec_gain_state |
| `mprotect` | EINVAL | bad alignment / unmapped range |
| `mremap` | EFAULT | region not mapped |
| `mremap` | EINVAL | new_size 0 / bad flags |
| `madvise` | EINVAL | bad advice / bad alignment |

The Rookery-NEW EACCES from MPROTECT-W→X-block is a userspace-visible behavior change documented in `00-security-principles.md` Axiom 4.

### Address-space layout policy

`arch/x86/mm/mmap.c` defines the per-process mmap base address randomization, stack base, and `TASK_SIZE` consumption. Identical to upstream:
- `mmap_base = TASK_UNMAPPED_BASE` adjusted by `arch_mmap_rnd()` (RANDMMAP)
- Stack base `STACK_TOP` per-mode (32-bit-on-64-bit vs. native)
- `mmap_min_addr` sysctl enforced

## Requirements

- REQ-1: Every documented syscall in scope has a byte-identical entry/exit ABI per upstream + the linked Tier-5 doc.
- REQ-2: All `MAP_*`, `PROT_*`, `MADV_*`, `MREMAP_*`, `MLOCK_*`, `MS_*`, `MCL_*` flag values are bit-identical.
- REQ-3: All errno emissions are identical (per the table above + per-syscall doc), including order of preference when multiple errors apply.
- REQ-4: VMA tree updates honor the `mmap_lock` write-side discipline; `mm/virtual-memory.md` mandates the lock contract.
- REQ-5: VMA flag propagation: `vma->vm_flags` set per syscall args; `vma->vm_pgoff` set per file-backed mmap; PKEY bits set per `pkey_mprotect`.
- REQ-6: VMA merging: adjacent VMAs with compatible flags merge per upstream. VMA splitting: range edits split a VMA into ≤ 3 pieces.
- REQ-7: `MAP_FIXED` semantics: caller-specified address is honored; previously-mapped overlap is unmapped before remap.
- REQ-8: `MAP_FIXED_NOREPLACE` semantics: if any byte of the requested range is already mapped, return EEXIST.
- REQ-9: `mremap` with `MREMAP_FIXED|MREMAP_MAYMOVE` semantics: source region freed, target installed; `MREMAP_DONTUNMAP` keeps source mapped read-only.
- REQ-10: Memory Protection Keys: `pkey_alloc(0, perm)` returns key 0..15; `pkey_free` releases; `pkey_mprotect` writes PKEY bits in PTE; `wrpkru` updates the per-CPU access mask. Hardware-supported on Intel (CPUID 0x7 ECX bit 3) + AMD (Zen 3+).
- REQ-11: madvise extensions for memcg / NUMA semantics match upstream.
- REQ-12: `mlock` consumes `RLIMIT_MEMLOCK`; `MCL_ONFAULT` defers locking to first fault per upstream.
- REQ-13: `mincore` returns per-page residency bitmap; `arch_prctl(ARCH_*_FS/GS)` sets/gets per-task FS_BASE/GS_BASE MSRs.
- REQ-14: **MPROTECT-W→X-block enforcement (mm-side)**: `mprotect` rejects W→X transitions on a VMA when `task->exec_gain_state` lacks NEEDS_EXEC_GAIN. Returns EACCES per `00-security-principles.md`.
- REQ-15: **NOEXEC-strict enforcement (mm-side)**: `mmap` with `PROT_EXEC | PROT_WRITE` on anon mapping rejected when `task->exec_gain_state` lacks NEEDS_RWX_ANON. Returns EACCES.
- REQ-16: VMA non-overlap invariant honored by every syscall in scope (Layer-3 mandate per `mm/virtual-memory.md`).
- REQ-17: LSM hook dispatch: `security_mmap_addr`, `security_mmap_file`, `security_file_mprotect` consulted on every mmap/mprotect; deny → EACCES.
- REQ-18: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: A `strace` golden trace of each syscall in scope on a curated test program produces byte-identical output vs. upstream. (covers REQ-1)
- [ ] AC-2: A grep over UAPI headers confirms all flag values match upstream. (covers REQ-2)
- [ ] AC-3: An errno-injection test: each documented (syscall, error-condition) pair returns the same errno on Rookery and upstream. (covers REQ-3)
- [ ] AC-4: A 16-CPU concurrent mmap+munmap test exhibits zero VMA-tree corruption under stress; mmap_lock + per-VMA-lock fast path operate per the TLA+ model. (covers REQ-4)
- [ ] AC-5: A test creating a VMA with various flag combos asserts the flags via `/proc/<pid>/maps` permission column. (covers REQ-5)
- [ ] AC-6: Three adjacent compatible VMAs are merged into one after a pre-fragment'd unmap; conversely, a midpoint mprotect produces 3 VMAs. Verifiable via `/proc/<pid>/maps`. (covers REQ-6)
- [ ] AC-7: `MAP_FIXED` over an existing mapping unmaps it; `MAP_FIXED_NOREPLACE` returns EEXIST. (covers REQ-7, REQ-8)
- [ ] AC-8: `mremap` test exercises MOVE, MAYMOVE, FIXED, DONTUNMAP combinations; outcomes match upstream. (covers REQ-9)
- [ ] AC-9: Pkey selftest (`tools/testing/selftests/x86/pkey-tester`) passes. (covers REQ-10)
- [ ] AC-10: madvise selftest exercises every MADV_* value; behavior matches upstream. (covers REQ-11)
- [ ] AC-11: `mlock` test under low `RLIMIT_MEMLOCK` returns EAGAIN at the same threshold; `MCL_ONFAULT` semantics verified. (covers REQ-12)
- [ ] AC-12: `mincore` test on a mapping with a faulted-in subset returns the same residency bitmap. (covers REQ-13)
- [ ] AC-13: A test that lacks the exec-gain ELF note attempts `mprotect(PROT_READ|PROT_WRITE|PROT_EXEC, ...)` to add EXEC to a W mapping; returns EACCES. The same test with `prctl(PR_REQUEST_EXEC_GAIN, 1, ...)` first succeeds. (covers REQ-14, MPROTECT-W→X-block)
- [ ] AC-14: A test that lacks the exec-gain note attempts `mmap(NULL, len, PROT_READ|PROT_WRITE|PROT_EXEC, MAP_ANON|MAP_PRIVATE, ...)`; returns EACCES. With prctl, succeeds. (covers REQ-15, NOEXEC-strict)
- [ ] AC-15: An LSM (e.g., yama, landlock) policy that denies an mmap operation translates to EACCES from the syscall. (covers REQ-17)
- [ ] AC-16: Hardening section present and follows template. (covers REQ-18)

## Architecture

### Rust module organization

- `kernel::mm::syscalls::mmap` — mmap + mmap2 + brk
- `kernel::mm::syscalls::munmap` — munmap
- `kernel::mm::syscalls::mremap` — mremap
- `kernel::mm::syscalls::mprotect` — mprotect + pkey_mprotect (MPROTECT enforcement here)
- `kernel::mm::syscalls::madvise` — madvise + process_madvise
- `kernel::mm::syscalls::mlock` — mlock + mlock2 + munlock + mlockall + munlockall
- `kernel::mm::syscalls::mincore` — mincore
- `kernel::mm::syscalls::msync` — msync
- `kernel::mm::syscalls::pkey` — pkey_alloc + pkey_free
- `kernel::mm::vma::merge` — VMA merge logic
- `kernel::mm::vma::split` — VMA split logic
- `kernel::mm::layout::arch::x86` — x86-specific mmap_base + stack policy

### Locking and concurrency

- **mmap_lock write-side**: held during mmap, mprotect, mremap, munmap, brk, madvise (when range modification needed), mlock (state change). Single writer at a time.
- **mmap_lock read-side**: never held by these syscalls (they're all writers); fault path holds read-side, released to per-VMA lock.
- **per-VMA lock**: not held by these syscalls (they're writers; per-VMA lock is for reader fast path).
- **anon_vma->root->rwsem**: held during anon-VMA chain mutation (e.g., during mremap or mprotect that splits an anon-VMA chain).
- **address_space->i_mmap_rwsem**: held during file-backed VMA changes that affect the file's reverse map.
- **i_mutex / inode lock**: held in some madvise paths that touch file data.

### Error handling

`Result<i64, Error>` for syscall returns. Common errors per the Compatibility contract table.

For LSM denials, the dispatch is:
1. `security_mmap_addr(addr)` consulted first
2. `security_mmap_file(file, prot, flags)` next
3. Returns EACCES on deny

GR-RBAC participates per `00-security-principles.md` Axiom 3.

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| VMA insert/remove under mmap_lock write-side | `kani::proofs::mm::syscalls::vma_insert_safety` |
| VMA merge (compatible-flags check + adjacent-range check) | `kani::proofs::mm::syscalls::merge_safety` |
| VMA split (≤ 3 pieces) | `kani::proofs::mm::syscalls::split_safety` |
| MPROTECT W→X-block rejection path | `kani::proofs::mm::syscalls::mprotect_wx_safety` |
| NOEXEC-strict rejection path | `kani::proofs::mm::syscalls::mmap_rwx_safety` |

### Layer 2: TLA+ models

- `models/mm/mmap_lock.tla` (inherited from `mm/00-overview.md` Layer-2; co-owned with `mm/virtual-memory.md`).
- `models/mm/exec_gain_propagation.tla` (inherited from `arch/x86/entry.md` Layer-2; co-owned). Proves that the per-task exec_gain_state read in mmap/mprotect is consistent with the value set by the ELF note + prctl on this task.

### Layer 3: Kani harnesses for data-structure invariants

- VMA non-overlap invariant (mandatory per `mm/00-overview.md` REQ-14 / `00-overview.md` D4) — co-owned with `mm/virtual-memory.md`. Verified for every VMA-modifying syscall in this Tier-3.

### Layer 4: Functional correctness (opt-in)

- **VMA merge / split arithmetic** via Creusot (declared in `mm/virtual-memory.md`) — proves correctness of split/merge in this Tier-3's syscall handlers.
- **MPROTECT W→X-block decision logic** via Verus — proves: given (vma->vm_flags, new prot, task->exec_gain_state), the function returns EACCES iff the upstream rule + Rookery's hardening rule both deny.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **MPROTECT-W→X-block** (mm-side enforcement) | `mprotect` rejects W→X transitions; reads `task->exec_gain_state` (set by ELF note recognition in `fs/exec-binfmt.md` + prctl in `kernel/task-lifecycle.md`) | § Default-on system-wide with per-process exemption |
| **NOEXEC-strict** (mm-side enforcement) | `mmap` rejects PROT_EXEC|PROT_WRITE on anon mapping unless task has NEEDS_RWX_ANON | § Default-on system-wide with per-process exemption |
| **NOEXEC** (default mode) | Default for stack/heap/anon-mmap unless ELF binary's PT_GNU_STACK has PF_X | § Default-on configurable off |
| **MPROTECT** (PaX classic — mm policy enforcement) | Implemented as the mm-side of REQ-14; defaults follow Rookery's locked-in policy | § Default-on system-wide with exemption |
| **RANDMMAP / ASLR** | mmap_base randomized via `arch_mmap_rnd()` per CONFIG_RANDOMIZE_BASE; sysctl `kernel.randomize_va_space=2` | § Default-on configurable off |

### Row-1 features consumed by this component

- **UDEREF**: every syscall reads `UserPtr<...>` for caller-supplied addresses; raw user-VA dereferencing forbidden
- **REFCOUNT**: VMA refcount uses `Refcount` (saturating)
- **AUTOSLAB**: VMAs allocated via `KmemCache::<Vma>::new()` per-type cache
- **MEMORY_SANITIZE**: VMA cache freed objects zeroed

### Row-2 / GR-RBAC integration

This component is the LSM hook dispatcher for memory operations:

- `security_mmap_addr(addr)` — called before address selection
- `security_mmap_file(file, prot, flags)` — called for file-backed mmap
- `security_file_mprotect(vma, reqprot, prot)` — called for mprotect
- `security_perm(...)` — generic
- GR-RBAC's policy can deny any of these per loaded gradm policy (default empty, drop-in compat preserved)

### Userspace-visible behavior changes

Per Axiom 4 of `00-security-principles.md`:
- **MPROTECT-W→X-block default-on**: documented in this Tier-3's Compatibility contract errno table; userspace JIT runtimes need the exec-gain ELF note OR `prctl(PR_REQUEST_EXEC_GAIN)` to opt out.
- **NOEXEC-strict default-on**: same.
- **All other behaviors match upstream**.

### Verification

(See § Verification above.)

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — `pkey_alloc`/`pkey_free`/`mincore` return buffers validated against slab usercopy zones.
- **PAX_KERNEXEC** — syscall .text + rodata are RX/RO; mmap handler dispatch tables are CONSTIFY'd.
- **PAX_RANDKSTACK** — per-syscall kernel-stack randomization at every entry-point this subsystem owns (mmap, mprotect, mremap, munmap, madvise, mlock, mincore, pkey_*).
- **PAX_REFCOUNT** — VMA refcount, anon_vma refcount, file refcount on file-backed mmap are saturating `Refcount` types.
- **PAX_MEMORY_SANITIZE** — VMA cache freed objects zeroed; anon mappings zero-on-free.
- **PAX_MEMORY_STACKLEAK** — kernel-stack residual zeroing on syscall exit from every mmap-family syscall.
- **PAX_RAP / kCFI** — indirect-call type-signature enforcement on file-backed `vm_operations_struct` vtables and LSM hooks (`security_mmap_addr`, `security_mmap_file`, `security_file_mprotect`).
- **GRKERNSEC_HIDESYM** — `/proc/<pid>/maps` permission column visible but pointers hidden from non-CAP_SYSLOG; kASLR-leak via maps-scraping defeated.
- **GRKERNSEC_DMESG** — mprotect-deny / mmap-deny dmesg lines restricted to CAP_SYSLOG; defeats fingerprinting exec-gain policy by an attacker.
- **PAX_ASLR** — full ASLR: text, data, brk, mmap base, stack base each independently randomized; `kernel.randomize_va_space=2` default-on.
- **PAX_RANDMMAP** — mmap base + each new VMA placement randomized via `arch_mmap_rnd()`; `mmap(NULL, ...)` placement non-deterministic.
- **PAX_MPROTECT** — **owned by this subsystem**: mprotect rejects W→X transitions on a VMA unless `task->exec_gain_state` lacks NEEDS_EXEC_GAIN; returns EACCES per `00-security-principles.md`.
- **PAX_PAGEEXEC** — PROT_EXEC bit propagates to PTE NX-bit clear; absence yields NX set; no page is executable without explicit PROT_EXEC.
- **PAX_NOEXEC** — **owned**: mmap with PROT_EXEC|PROT_WRITE on anon mapping rejected unless task has NEEDS_RWX_ANON; returns EACCES (NOEXEC-strict).
- **PAX_SEGMEXEC** — x86-32 segmented protection; x86_64-irrelevant (cross-ref `arch/x86/00-overview.md` D1).
- **PAX_UDEREF** — every syscall reads `UserPtr<...>` for caller-supplied addresses; raw user-VA dereferencing forbidden.
- **GRKERNSEC_KSTACKOVERFLOW** — kernel stack overflow detection at every recursive mmap/mprotect path (e.g., LSM-hook recursion).
- **mmap_min_addr** — sysctl-enforced lower bound on mmap; defeats NULL-pointer-dereference exploits that rely on mapping at low addresses.

Per-doc rationale: the mmap family is the userspace's primary handle on virtual memory; an attacker who can mprotect W→X, mmap RWX-anon, mremap a sensitive mapping, or mmap at NULL has won every memory-corruption exploit primitive. This Tier-3 **owns** PaX's two flagship features — PAX_MPROTECT (W→X-block) and the NOEXEC-strict mmap policy — and enforces them at the syscall entry, returning EACCES rather than installing the dangerous mapping. PAX_ASLR + PAX_RANDMMAP randomize the address space so no exploit can rely on predictable VA placement; mmap_min_addr blocks the NULL-deref classic; PAX_USERCOPY + PAX_UDEREF deny direct user-pointer abuse from the syscall handlers. GRKERNSEC_HIDESYM ensures that even a debugger-grade `/proc/<pid>/maps` scrape cannot leak kernel pointers to break randomization.

## Open Questions

(none — mmap-family syscalls are exhaustively specified by POSIX + Linux extensions; no architectural ambiguities at this tier)

## Out of Scope

- Per-fault page-fault handling (cross-ref `mm/virtual-memory.md`)
- Page allocator / slab (cross-ref `mm/page-allocator.md` + `mm/slab.md`)
- 32-bit-only paths
- Implementation code
