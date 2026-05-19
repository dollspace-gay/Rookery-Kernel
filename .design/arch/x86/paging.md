# Tier-3: arch/x86/paging — x86_64 page tables, fault handling, ioremap

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - arch/x86/mm/
  - arch/x86/mm/init_64.c
  - arch/x86/mm/init.c
  - arch/x86/mm/fault.c
  - arch/x86/mm/ioremap.c
  - arch/x86/mm/iomap_32.c
  - arch/x86/mm/pat/
  - arch/x86/mm/kaslr.c
  - arch/x86/mm/kasan_init_64.c
  - arch/x86/mm/kmsan_shadow.c
  - arch/x86/mm/mem_encrypt.c
  - arch/x86/mm/mem_encrypt_amd.c
  - arch/x86/mm/mem_encrypt_boot.S
  - arch/x86/mm/numa.c
  - arch/x86/mm/amdtopology.c
  - arch/x86/mm/dump_pagetables.c
  - arch/x86/mm/debug_pagetables.c
  - arch/x86/mm/ident_map.c
  - arch/x86/mm/maccess.c
  - arch/x86/mm/mmap.c
  - arch/x86/mm/pgprot.c
  - arch/x86/mm/pti.c
  - arch/x86/mm/tlb.c
  - arch/x86/include/asm/pgtable.h
  - arch/x86/include/asm/pgtable_types.h
  - arch/x86/include/asm/tlbflush.h
  - arch/x86/include/asm/io.h
-->

## Summary
Tier-3 design for x86_64 paging: 4-level and 5-level page table layout, page-fault handling, ioremap/iomap (MMIO mapping), Page Attribute Table (PAT, memory-type configuration), Kernel Page-Table Isolation (KPTI / Meltdown mitigation), KASLR-paging integration, KASAN/KMSAN shadow setup, AMD memory encryption (SME / SEV host integration), and TLB-flush coordination across CPUs.

This Tier-3 is mandated as a Layer-3-Kani-harness subsystem per `arch/x86/00-overview.md` (Tier 2 references this as part of the four mandatory Layer-3 subsystems). It is one of the highest-stakes pieces of the kernel: a bug here corrupts every memory access.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Page table init + tear-down | `arch/x86/mm/init_64.c`, `arch/x86/mm/init.c`, `arch/x86/mm/ident_map.c` |
| Page-fault handler | `arch/x86/mm/fault.c` |
| ioremap / iomap (MMIO) | `arch/x86/mm/ioremap.c`, `arch/x86/mm/iomap_32.c` |
| Page Attribute Table (PAT) | `arch/x86/mm/pat/` (set_memory.c, memtype.c, memtype_interval.c) |
| KASLR (paging side; boot side in `arch/x86/boot.md`) | `arch/x86/mm/kaslr.c` |
| Sanitizer shadow init | `arch/x86/mm/kasan_init_64.c`, `arch/x86/mm/kmsan_shadow.c` |
| Memory encryption (host-side AMD SME / SEV) | `arch/x86/mm/mem_encrypt.c`, `arch/x86/mm/mem_encrypt_amd.c`, `arch/x86/mm/mem_encrypt_boot.S` |
| NUMA topology | `arch/x86/mm/numa.c`, `arch/x86/mm/amdtopology.c` |
| Page-table dump / debug | `arch/x86/mm/dump_pagetables.c`, `arch/x86/mm/debug_pagetables.c` |
| Safe access to user/IO memory | `arch/x86/mm/maccess.c` |
| User-mmap policy | `arch/x86/mm/mmap.c` (cross-ref `mm/mmap.md` for the cross-arch mmap behavior) |
| KPTI (Meltdown mitigation) | `arch/x86/mm/pti.c` |
| TLB flush + coordination | `arch/x86/mm/tlb.c` |
| Page-table type definitions | `arch/x86/include/asm/{pgtable,pgtable_types,tlbflush,io}.h` |

## Compatibility contract

### Page table layout (userspace-visible)

x86_64 supports **4-level paging** (default) and **5-level paging** (CR4.LA57; CONFIG_X86_5LEVEL).

**4-level paging** (47-bit user virtual address; 47-bit kernel):
- PGD (Page Global Directory): 9 bits, 512 entries × 8 bytes = 4 KiB
- PUD (Page Upper Directory): 9 bits, 512 entries
- PMD (Page Middle Directory): 9 bits, 512 entries (can be 2 MiB huge page)
- PTE (Page Table Entry): 9 bits, 512 entries (4 KiB pages); 12-bit byte offset
- Total: 9 + 9 + 9 + 9 + 12 = 48 bits (47-bit user split, sign-extended above)

**5-level paging** (56-bit user virtual address; 56-bit kernel):
- P5D / PML5: extra 9-bit level above PGD
- 9 × 5 + 12 = 57 bits

`TASK_SIZE` (max user VA) is `0x00007fff_ffffffff` on 4-level, `0x00ffffff_ffffffff` on 5-level. Identical to upstream.

### Page sizes

- **4 KiB** — base page (PTE level)
- **2 MiB** — large page (PMD level, PSE bit)
- **1 GiB** — huge page (PUD level, PSE bit)

Userspace observes via `/proc/<pid>/smaps` `KernelPageSize` / `MMUPageSize` fields. Identical to upstream.

### `/proc/<pid>/maps` ordering

Maps come from VMA tree iteration; the maple tree (`mm/00-overview.md` cross-ref) iterates in address order. This Tier-3 owns the page-table side; address-order is preserved.

### `/proc/<pid>/pagemap` bit layout

`include/uapi/linux/pagemap.h` (cross-arch) defines the per-page bitfield. This Tier-3 produces the `present`, `swap`, `file/anon`, `exclusive`, `soft-dirty`, `pte-uffd-wp`, `mmap_exclusive` bits per upstream semantics. Bit-identical.

### PAT (memory type) ABI

`arch/x86/mm/pat/` configures cacheability per page. Userspace exposure:
- `mmap` with various cache flags via the file's underlying ops (typically `pgprot_writecombine`)
- `/sys/kernel/debug/x86/pat_memtype_list`
- `/sys/devices/.../resource*` for PCI BARs

PAT entries IA32_PAT MSR layout matches upstream: `WB, WT, UC-, UC, WB, WT, UC-, WC` (the upstream-canonical PAT layout).

### Userspace-visible page faults

Page fault → SIGSEGV / SIGBUS path is owned here:
- VMA-mapped: kernel resolves silently (file-backed page-in, anon page allocation, COW); userspace sees nothing
- Outside any VMA: SIGSEGV with `si_code = SEGV_MAPERR`
- Inside a VMA but lacking permission: SIGSEGV with `si_code = SEGV_ACCERR`
- File-backed but past EOF: SIGBUS with `si_code = BUS_ADRERR`
- Hardware-poisoned page: SIGBUS with `si_code = BUS_MCEERR_AR / BUS_MCEERR_AO`

Identical to upstream (cross-ref `arch/x86/00-overview.md` § entry.md exception → signal table).

## Requirements

- REQ-1: 4-level paging is the default; 5-level paging is supported when CR4.LA57 is set per CONFIG_X86_5LEVEL.
- REQ-2: Page table layout (PGD/PUD/PMD/PTE entries; PSE huge pages; NX bit) is byte-identical to upstream.
- REQ-3: `TASK_SIZE` constant matches upstream per paging mode.
- REQ-4: Page-fault handler dispatches to mm-side fault handler (cross-ref `mm/00-overview.md` § virtual-memory.md) with `pt_regs` + `error_code` + `cr2` per upstream calling convention.
- REQ-5: ioremap / iomap maps MMIO regions with correct cache attributes per PAT; `__iomem` typed pointer enforced.
- REQ-6: PAT entries match upstream's IA32_PAT MSR layout; `set_memory_*` functions (`set_memory_uc`, `set_memory_wc`, `set_memory_wb`, `set_memory_x`, `set_memory_nx`, `set_memory_ro`, `set_memory_rw`, `set_memory_4k`) preserve their semantics.
- REQ-7: KASLR (paging side) ASLR-shifts the kernel base + direct map + vmalloc + kasan-shadow areas per upstream; default-on per `00-security-principles.md` knob inventory.
- REQ-8: KASAN shadow region init produces a valid shadow mapping; KMSAN shadow + origin regions similarly. Per-arch glue talks to cross-arch `mm/sanitizers.md`.
- REQ-9: AMD SME (host-side) sets the C-bit per upstream conventions; SEV memory-encryption tagging is correct on memory marked `SHARED` vs `PRIVATE`.
- REQ-10: KPTI (Meltdown mitigation) maintains a separate user PGD with only entry-trampoline mapped; auto-on for affected CPUs (Intel pre-Whiskey-Lake), off otherwise. CR3 swap on entry/exit per upstream.
- REQ-11: TLB flush coordination: `flush_tlb_one_user`, `flush_tlb_mm`, `flush_tlb_kernel_range`, `flush_tlb_all` semantics match upstream; cross-CPU IPI delivery for global flushes; PCID handling matches upstream when CR4.PCIDE.
- REQ-12: Page-table walker (the cross-arch primitive consumed by `mm/`) preserves access/dirty bits, walks 4-level/5-level correctly, and respects the W^X invariants documented in `00-security-principles.md`.
- REQ-13: NUMA topology detection (BIOS/ACPI SRAT, AMD HyperTransport scan) produces identical `nodemask_t` and `numa_node_id()` results vs. upstream.
- REQ-14: `/proc/<pid>/pagemap` produces bit-identical content for any given VMA + page state.
- REQ-15: Layer-3 Kani invariant harnesses for the page-table walker are MANDATORY per `arch/x86/00-overview.md` (one of the four mandatory Layer-3 subsystems).
- REQ-16: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: Boot under qemu with `-cpu max,la57=off` + `-cpu max,la57=on`; both succeed; `/sys/kernel/debug/x86/pgt_levels` reports 4 / 5. (covers REQ-1)
- [ ] AC-2: A reference test maps a file with `mmap`, accesses pages, then dumps via `/sys/kernel/debug/x86/dump_pagetables`. The output is byte-identical to upstream (modulo addresses subject to KASLR). (covers REQ-2)
- [ ] AC-3: `cat /proc/<pid>/maps` for an unprivileged ELF reports addresses ≤ TASK_SIZE; an `mmap(addr=TASK_SIZE+1)` returns ENOMEM. (covers REQ-3)
- [ ] AC-4: A test program SIGSEGV-faults on a NULL deref; `dmesg` shows the same fault decode as upstream (RIP, error_code, cr2). (covers REQ-4)
- [ ] AC-5: An `ioremap`'d PCI BAR via a Rust driver returns a `Volatile<u8>`-typed accessor; reads/writes map to MMIO. (covers REQ-5)
- [ ] AC-6: PAT memtype interaction: a video-buffer mmap with WC succeeds; PAT MSR contents (visible via `/dev/cpu/0/msr`) match upstream's. (covers REQ-6)
- [ ] AC-7: KASLR-enabled boots produce a non-deterministic kernel base; the direct-map + vmalloc + kasan-shadow regions are also randomized within their allotted slots. (covers REQ-7)
- [ ] AC-8: A KASAN-built kernel boots and detects a deliberate UAF; the report is byte-identical (modulo timestamps) to upstream. (covers REQ-8)
- [ ] AC-9: An SME-enabled boot on AMD hardware shows the C-bit set on direct-map pages; SEV-host validation against an SEV-SNP guest succeeds. (covers REQ-9)
- [ ] AC-10: KPTI selftest: on a Meltdown-affected CPU (or simulated), the user PGD lacks kernel direct-map entries; on a Meltdown-immune CPU, KPTI is off. (covers REQ-10)
- [ ] AC-11: TLB-flush selftest: an `mprotect` cross-CPU change invalidates the TLB on all CPUs that had the mapping cached. (covers REQ-11)
- [ ] AC-12: Page-table-walker harness (Kani; mandatory Layer 3) passes. (covers REQ-12, REQ-15)
- [ ] AC-13: A NUMA-enabled boot on multi-socket hardware reports the same `nodemask_t` as upstream; `numactl --hardware` output is identical. (covers REQ-13)
- [ ] AC-14: `/proc/<pid>/pagemap` bytes for a curated test program (specific mmap pattern) are byte-identical to upstream. (covers REQ-14)
- [ ] AC-15: Hardening section present and follows template. (covers REQ-16)

## Architecture

### Rust module organization

- `kernel::arch::x86::paging::pgd` — PGD-level operations
- `kernel::arch::x86::paging::pud` — PUD-level
- `kernel::arch::x86::paging::pmd` — PMD-level
- `kernel::arch::x86::paging::pte` — PTE-level (the leaf)
- `kernel::arch::x86::paging::walker` — generic page-table walker (per-level dispatch)
- `kernel::arch::x86::paging::fault` — `__do_page_fault` analog; calls into mm-side handler
- `kernel::arch::x86::paging::ioremap` — MMIO region mapping
- `kernel::arch::x86::paging::pat` — Page Attribute Table management
- `kernel::arch::x86::paging::kaslr` — paging-side KASLR
- `kernel::arch::x86::paging::kasan` / `kmsan` — sanitizer shadow init (cross-ref `mm/sanitizers.md`)
- `kernel::arch::x86::paging::mem_encrypt` — AMD SME C-bit; SEV host-side
- `kernel::arch::x86::paging::pti` — KPTI per-task PGD swap
- `kernel::arch::x86::paging::tlb` — TLB flush + IPI coordination
- `kernel::arch::x86::paging::numa` — NUMA topology detection

### Key data structures

- `Pgd`, `Pud`, `Pmd`, `Pte` — newtype wrappers around `u64`. Methods: `is_present()`, `pfn()`, `flags()`, `set_pfn()`, `clear()`, etc.
- `PageTableLevel` enum — `{Pgd, Pud, Pmd, Pte}` for generic walker
- `MemoryType` enum — `{Uc, UcMinus, Wt, Wb, Wc}` for PAT
- `TlbFlushOp` enum — `{One{vaddr}, Mm{mm}, KernelRange{start, end}, All}`

### Locking and concurrency

- Per-process `mm->pgd` lock: `mmap_lock` (rwsem); held by `mm/mmap.md` callers
- Per-page-table-page `pte_lockptr`: spinlock per PMD page; held during PTE updates
- Cross-CPU TLB invalidation: IPI-driven; `flush_tlb_func` runs on each target CPU
- KPTI CR3 swap: per-CPU; uses per-CPU `cpu_tlbstate.user_cr3`

### Error handling

`Result<T, kernel::error::Error>` for fallible ops:
- `Err(ENOMEM)` — page-table allocation failed
- `Err(EFAULT)` — bad userspace pointer in usercopy fast path (cross-ref `lib/usercopy.md`)
- `Err(EINVAL)` — bad address range / page size combination
- `Err(EBUSY)` — `set_memory_*` change blocked by concurrent operation

Page faults do NOT return Result; they hand back `EntryOutcome` per `arch/x86/entry.md`'s convention.

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| PGD/PUD/PMD/PTE entry reads (assumes valid table address) | `kani::proofs::arch::x86::paging::pgd_read_safety` (and parallel for each level) |
| Page-table walker on arbitrary VA | `kani::proofs::arch::x86::paging::walker_safety` |
| ioremap region establish/unmap | `kani::proofs::arch::x86::paging::ioremap_safety` |
| PTE update with TLB flush | `kani::proofs::arch::x86::paging::pte_update_safety` |
| KPTI CR3 swap | `kani::proofs::arch::x86::paging::pti_swap_safety` |
| AMD C-bit application/removal | `kani::proofs::arch::x86::paging::mem_encrypt_safety` |

### Layer 2: TLA+ models

- `models/arch/x86/tlb_flush.tla` (NEW) — proves cross-CPU TLB-flush coordination: every CPU that had the mapping cached observes the flush before any subsequent access.
- `models/arch/x86/pti_swap.tla` (NEW) — proves KPTI's user-PGD swap is correctly ordered with respect to entry/exit + IRET.
- `models/arch/x86/mem_encrypt.tla` (NEW) — proves SME C-bit ordering: a page marked PRIVATE is never observed without C-bit set, and vice versa.

### Layer 3: Kani harnesses for data-structure invariants (MANDATORY)

Per `arch/x86/00-overview.md` D4 mandate:

| Data structure | Invariant | Harness |
|---|---|---|
| Page-table walker | Every PTE's PFN resolves to a valid page-table page or `struct page`; access/dirty bits never decrease except through explicit clear paths | `kani::proofs::arch::x86::paging::walker_invariants` |
| PGD-PUD-PMD-PTE chain | A present-bit-clear entry implies all child pages are unreachable | `kani::proofs::arch::x86::paging::reachability_invariants` |
| ioremap region table | Two ioremap'd regions never overlap; each has a valid PAT memtype | `kani::proofs::arch::x86::paging::ioremap_disjoint_invariants` |
| KPTI user PGD | User PGD shares only the entry-trampoline range with kernel PGD; no kernel direct-map leakage | `kani::proofs::arch::x86::paging::pti_isolation_invariants` |

### Layer 4: Functional correctness (opt-in)

- **Address-translation correctness**: Verus proof that for any VA, the page-table walker returns the same PFN that the CPU's MMU would. Tractable; high-value (the most security-sensitive function in the kernel).
- **PAT memtype merging**: `arch/x86/mm/pat/memtype.c`'s interval-tree merging logic is a strong Creusot target.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **KERNEXEC** | Page-table init sets `.text` + `.rodata` as RX/RO; `set_memory_x`/`set_memory_nx` enforce W^X for kernel sections | `00-security-principles.md` § Mandatory |
| **PAGEEXEC** (NX bit) | Default for all data/heap/stack pages | § Mandatory |
| **CLOSE_KERNEL/CLOSE_USERLAND** | SMEP+SMAP enforced via CR4 (set in boot); page-table walker honors U/S bit per upstream | § Mandatory |
| **KASLR** | `arch/x86/mm/kaslr.c` analog randomizes kernel base + direct map + vmalloc + kasan-shadow per CONFIG_RANDOMIZE_BASE | § Default-on configurable off |
| **KPTI** | `arch/x86/mm/pti.c` analog; auto-on for Meltdown-affected CPUs | Auto-on per CPU; matches upstream behavior |

### Row-1 features consumed by this component

- **UDEREF**: every user-pointer access goes through `kernel::user::UserPtr<T>`; raw user-VA dereferencing forbidden in this component
- **SIZE_OVERFLOW**: page-table-index arithmetic uses checked operators per the lint
- **CONSTIFY**: PAT MSR layout, IDT, GDT etc. are static const tables

### Row-2 / GR-RBAC integration

This component runs below the LSM hook layer. Page-fault → signal-delivery has no LSM hook (signal delivery itself does, but in `kernel/signal.md`). No direct LSM-hook surface in this Tier-3.

### Userspace-visible behavior changes

None beyond what `arch/x86/00-overview.md` already documented. KASLR strict-mode + KPTI default-on are upstream defaults; Rookery does not change them in this Tier-3.

### Verification

(See § Verification above.)

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_KERNEXEC** — W^X enforcement for kernel rwx zones; `.text` / `.rodata` are RX/RO; `set_memory_x`/`set_memory_nx`/`set_memory_ro`/`set_memory_rw` preserve W^X invariants for kernel sections.
- **PAX_PAGEEXEC (NX bit)** — default for all data/heap/stack pages; enforced at PTE level.
- **PAX_RANDMMAP / KASLR** — `arch/x86/mm/kaslr.c` analog randomizes kernel base + direct map + vmalloc + kasan-shadow regions per CONFIG_RANDOMIZE_BASE.
- **PAX_RANDKSTACK** — per-syscall kernel-stack randomization (cross-ref `entry.md`).
- **KPTI / Meltdown mitigation** — `arch/x86/mm/pti.c` maintains separate user PGD with only entry-trampoline mapped; auto-on for Meltdown-affected CPUs; CR3 swap on entry/exit.
- **PAX_UDEREF (SMAP/SMEP)** — CR4 bits set in boot and verified preserved; page-table walker honors U/S bit; `kernel::user::UserPtr<T>` newtype enforces typed user-pointer access.
- **PAX_USERCOPY** — bounded copy on `/proc/<pid>/pagemap`, `/sys/kernel/debug/x86/dump_pagetables`, `pat_memtype_list`.
- **PAX_MEMORY_SANITIZE** — zero-on-free for page-table pages; sanitizer (KASAN/KMSAN) shadow regions detect post-free use.
- **PAX_RAP / kCFI** — indirect-call signature enforcement on TLB-flush op vtable, ioremap region ops, paravirt PV-OPS dispatch.
- **PAX_REFCOUNT** — saturating refcount on ioremap region table entries and per-mm PGD.
- **GRKERNSEC_HIDESYM** — kernel-pointer hiding in /proc and dmesg; `dump_pagetables` access gated.
- **GRKERNSEC_DMESG** — restrict syslog output to CAP_SYSLOG (including page-fault traces).
- **GRKERNSEC_KMEM** — block `/dev/mem` access to kernel direct-map and page-table pages.
- **AMD SME C-bit + SEV memory-encryption tagging** — defense against memory-bus snooping on AMD platforms; C-bit ordering verified via TLA+ model.
- **TLB flush coordination via IPI** — defense against cross-CPU stale-TLB reads after PTE update.
- **Sanitizer shadow init (KASAN/KMSAN)** — defense against use-after-free / use-of-uninit-memory in kernel-space.

Per-doc rationale: paging IS the kernel's primary memory-security primitive; every grsec/PaX feature above operates at the page-table layer; a single failure (e.g., a missed PTI swap, a stale TLB after mprotect, a misplaced C-bit) is a complete privilege boundary collapse, so hardening here is overlapping and mechanical (set_memory_* + PTI + KASLR + KASAN + SMAP/SMEP + TLB-flush IPI).

## Open Questions

(none — paging is contractually defined by Intel/AMD architecture + upstream Linux semantics; no architectural ambiguities at this tier)

## Out of Scope

- 32-bit paging (`arch/x86/mm/init_32.c`) — out per `arch/x86/00-overview.md` D1
- Implementation code
