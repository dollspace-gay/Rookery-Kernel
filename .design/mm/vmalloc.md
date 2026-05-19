# Tier-3: mm/vmalloc — vmalloc / vfree, virtually-contiguous kernel allocator

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - mm/vmalloc.c
  - include/linux/vmalloc.h
-->

## Summary
Tier-3 design for the kernel's virtually-contiguous, physically-non-contiguous allocator. Used when:
- A subsystem needs large contiguous virtual memory but doesn't care about physical contiguity (kernel module .text/.bss; kvmalloc fallback for very-large slab requests; per-CPU areas above the percpu-area threshold; loadable firmware blob staging)
- Per-CPU memory needs to be virtually contiguous across CPUs (cross-ref `mm/percpu.md`)

vmalloc is consumed by every subsystem that needs more than a few pages of contiguous kernel memory. Also owns the `kvmalloc` family (try-kmalloc-then-vmalloc fallback), `vmap`/`vunmap` (manual page → virtual mapping), and the lazy-free list that defers TLB invalidation.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| vmalloc core | `mm/vmalloc.c` |
| Public API | `include/linux/vmalloc.h` |

## Compatibility contract

### `/proc/vmallocinfo`

Format-identical to upstream: per-allocation: `<addr>-<addr> <size> caller=<caller> pages=<n> phys=<addr>` line per active vmalloc area. Existing tools (`vmstat`-style monitoring, `crash`, `drgn`) parse identically.

### `/proc/meminfo` vmalloc fields

- `VmallocTotal` — vmalloc address space total
- `VmallocUsed` — currently used
- `VmallocChunk` — largest free contiguous range

Format-identical.

### Sysctls

- `vm.vmalloc_huge_threshold` (when CONFIG_VMAP_HUGE_PAGES=y) — minimum size for huge-page-backed vmalloc
- `vm.lazy_mb` — lazy-free list size before forced sync

### vmap address-space layout

The vmalloc address range (per `arch/x86/include/asm/pgtable_64.h`):
- 4-level paging: `0xffffc90000000000 .. 0xffffe8ffffffffff` (32 TiB)
- 5-level paging: `0xffa0000000000000 .. 0xffd1ffffffffffff` (12 PiB)

Identical layout so KASAN-shadow + KASLR randomization work unchanged.

## Requirements

- REQ-1: `vmalloc(size)` returns virtually-contiguous, physically-non-contiguous memory; `vfree(addr)` releases. Identical semantics + return-value conventions.
- REQ-2: `vmalloc_node(size, node)` allocates from the specified NUMA node. `vmalloc_user(size)` adds usermap-friendly flags.
- REQ-3: `kvmalloc(size, gfp)` family: try kmalloc first; on failure, fall back to vmalloc. Identical fallback threshold.
- REQ-4: `vmap(pages, count, flags, prot)` maps an array of pages into a virtually-contiguous range with specified prot/cacheability.
- REQ-5: `vmap_pfn(pfns, count, prot)` maps PFN array (used by drivers for MMIO regions).
- REQ-6: Lazy-free list: when `vfree` is called, the area is added to a per-CPU lazy-free queue; TLB invalidation deferred until queue threshold or sync. Reduces IPI traffic for high-rate vfree.
- REQ-7: Huge-page-backed vmalloc (CONFIG_VMAP_HUGE_PAGES=y): when `size >= vm.vmalloc_huge_threshold` and alignment permits, allocate using PMD-level (2 MiB) pages. Reduces TLB pressure.
- REQ-8: `/proc/vmallocinfo` content format-identical to upstream for equivalent kernel state.
- REQ-9: KASAN integration: vmalloc'd memory is KASAN-instrumented per upstream; shadow memory mapped on vmalloc, unmapped on vfree.
- REQ-10: Page-table modification: vmalloc updates kernel page tables; coordinates with all CPUs via `sync_kernel_mappings` for TLB consistency.
- REQ-11: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: A test allocates `vmalloc(N MiB)` for N in {1, 16, 256}; reads/writes; vfrees. Resulting `/proc/vmallocinfo` shows the area during use, not after. (covers REQ-1)
- [ ] AC-2: NUMA-aware test: `vmalloc_node(size, 1)` on multi-socket allocates from node 1; verifiable via PFN range. (covers REQ-2)
- [ ] AC-3: `kvmalloc(small)` returns kmalloc-backed memory; `kvmalloc(huge)` falls back to vmalloc. Verifiable via address range (kmalloc returns linear-map address; vmalloc returns vmalloc-range address). (covers REQ-3)
- [ ] AC-4: A driver `vmap(pages, count, ...)` test maps pages, accesses them, vunmaps; the underlying physical pages are reachable correctly. (covers REQ-4)
- [ ] AC-5: A driver `vmap_pfn` test on simulated MMIO range succeeds. (covers REQ-5)
- [ ] AC-6: A high-rate vfree benchmark (1M vfree/s) shows lazy-free TLB-invalidation amortization (visible via PMU counter for IPI count). (covers REQ-6)
- [ ] AC-7: With CONFIG_VMAP_HUGE_PAGES=y and `vm.vmalloc_huge_threshold=2MiB`, a `vmalloc(2 MiB)` call results in PMD-level mapping (verifiable via `pgt_dump`). (covers REQ-7)
- [ ] AC-8: `/proc/vmallocinfo` after a curated workload byte-identical (modulo addresses) to upstream. (covers REQ-8)
- [ ] AC-9: A KASAN-built kernel + a deliberate UAF on a vmalloc'd buffer is detected; report format-identical. (covers REQ-9)
- [ ] AC-10: Concurrent vmalloc + vfree from 16 CPUs produces consistent kernel page tables; no stale-TLB-induced kernel oops. (covers REQ-10)
- [ ] AC-11: Hardening section present and follows template. (covers REQ-11)

## Architecture

### Rust module organization

- `kernel::mm::vmalloc::Vmalloc` — `vmalloc` / `vfree` family
- `kernel::mm::vmalloc::Kvmalloc` — `kvmalloc` fallback wrapper
- `kernel::mm::vmalloc::Vmap` — `vmap` / `vmap_pfn` / `vunmap`
- `kernel::mm::vmalloc::lazy_free` — per-CPU lazy-free queue
- `kernel::mm::vmalloc::tree::VmallocTree` — vmalloc-area allocator (rbtree of free areas)
- `kernel::mm::vmalloc::huge` — huge-page-backed vmalloc when supported

### Locking and concurrency

- **`vmap_area_lock`** (global spinlock) — serializes vmalloc-area allocation/free
- **`free_vmap_area_lock`** — serializes lazy-free list manipulation
- **Per-CPU `vmap_block` arrays** (when CONFIG_VMAP_PERCPU=y) — fast-path lock-free push/pop for small vmaps
- **`vmap_purge_lock`** — held during lazy-free flush

### Error handling

- `Err(ENOMEM)` — page allocation failed
- `Err(EAGAIN)` — vmalloc-area allocator out of fragments; caller may retry
- `Err(EINVAL)` — bad alignment / size

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| vmalloc-area rbtree manipulation | `kani::proofs::mm::vmalloc::tree_safety` |
| Page-table modification (kernel-side mapping) | `kani::proofs::mm::vmalloc::pgt_modify_safety` |
| Lazy-free queue push/pop | `kani::proofs::mm::vmalloc::lazy_free_safety` |
| vmap with prot/cache flags | `kani::proofs::mm::vmalloc::vmap_safety` |

### Layer 2: TLA+ models

- `models/mm/vmalloc_lazy_free.tla` (NEW) — proves: lazy-free queue eventually drains; no orphaned allocations remain after sync.

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| vmalloc-area free-list rbtree | "Free areas are non-overlapping, address-ordered, and cover the entire vmalloc range" | `kani::proofs::mm::vmalloc::tree_invariants` |
| Lazy-free queue | "Sum of queued areas + freed areas == initial allocations - current live allocations" | `kani::proofs::mm::vmalloc::lazy_free_invariants` |

### Layer 4: Functional correctness (opt-in; declared in `mm/00-overview.md` Layer-4 candidates)

- **Vmalloc fragmentation analysis** via Creusot — proves: under any sequence of alloc/free, the largest free contiguous range remains bounded by a documented function of total fragmentation. Tractable; high-leverage.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **KERNEXEC** | vmalloc'd memory's protection enforced via page-table flags; default RW (no X); allocator uses `vmalloc_exec` for code-allocating callers (e.g., BPF JIT) — these go through additional hardening (cross-ref `arch/x86/00-overview.md` § bpf-jit.md) | § Mandatory |
| **MEMORY_SANITIZE** | Freed vmalloc pages routed through page-allocator's free path; zeroed if `vm.zero_on_free=1` | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT**: vm-area refcount uses `Refcount`
- **SIZE_OVERFLOW**: alignment + page-count arithmetic uses checked operators
- **AUTOSLAB**: vm_struct + vmap_area allocated via per-type slab caches
- **CONSTIFY**: per-arch vmalloc-range constants are `static const`

### Row-2 / GR-RBAC integration

vmalloc is below the LSM-hook layer; no LSM hooks fire. Higher-tier callers (BPF JIT alloc, module load) invoke their own LSM hooks.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — vmalloc regions are not part of any usercopy whitelist; copy_to_user/copy_from_user against vmalloc addresses must go through explicit `check_object_size()` validation per caller.
- **PAX_KERNEXEC** — default vmalloc allocations are RW only; executable vmalloc (BPF JIT, modules) is routed through `vmalloc_exec` / `module_alloc` with separate W^X enforcement and per-region permission transitions.
- **PAX_RANDKSTACK** — kernel stacks allocated via vmalloc (CONFIG_VMAP_STACK) cooperate with randomized kernel-stack offset; guard pages bracket every vmapped stack to trap overflow into adjacent vmalloc regions.
- **PAX_REFCOUNT** — vm_struct and vmap_area refcounts use the hardened refcount type; vm_area refcount saturation traps instead of wrapping into a UAF.
- **PAX_MEMORY_SANITIZE** — freed vmalloc pages route through the page-allocator free path and are zeroed when `vm.zero_on_free=1` so a subsequent vmalloc never returns stale kernel data.
- **PAX_UDEREF** — vmalloc range is firmly in kernel half; UDEREF prevents a user pointer being mistakenly dereferenced through a vmalloc-style helper.
- **PAX_RAP / kCFI** — vmalloc-allocated trampolines/BPF programs land in CFI-typed regions; indirect dispatch into vmalloc executable memory is type-checked.
- **GRKERNSEC_HIDESYM** — /proc/vmallocinfo redacts kernel addresses and caller PCs for unprivileged readers, exposing only sizes and flag bits to non-CAP_SYSLOG holders.
- **GRKERNSEC_DMESG** — vmalloc-failure warnings (allocation failed, kasan-tag exhausted) gate behind dmesg_restrict so unprivileged tasks cannot map the address-space layout via probing.
- **Per-vmap_area RB-tree guarded by hardened spinlock** — defense against per-tree race producing overlapping mappings.
- **Per-vmap kernel-stack guard page mandatory** — defense against per-stack-overflow corrupting neighboring vmalloc allocations.
- **Per-vmalloc_huge path KERNEXEC-aware** — defense against per-PMD-execute granting accidental X over MB-sized regions.
- **Per-purge_vmap_area_lazy bounded** — defense against per-tlb-flush amplification DoS.
- **Per-VM_FLUSH_RESET_PERMS on free** — defense against per-stale-X permission persisting after vfree of executable vmalloc.

Rationale: vmalloc is the kernel's primary non-contiguous virtual allocator and the home of every JIT, every vmap stack, and most module text — exactly the kind of region an attacker targets for code injection or layout discovery. Grsec compounding adds W^X discipline (KERNEXEC + RAP), stack-guard mandatoriness for VMAP_STACK, free-time sanitization (MEMORY_SANITIZE), refcount hardening on vm_struct/vmap_area, and layout secrecy via HIDESYM/DMESG.

## Open Questions

(none — vmalloc semantics are exhaustively specified by Linux internal conventions)

## Out of Scope

- BPF JIT-specific vmalloc_exec usage (cross-ref `arch/x86/00-overview.md` § bpf-jit.md)
- Per-CPU memory allocator (cross-ref `mm/percpu.md`)
- 32-bit-only highmem paths
- Implementation code
