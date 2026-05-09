---
title: "Subsystem: mm/ — memory management"
tags: ["design-doc", "subsystem", "mm"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-2 overview for `mm/` — the kernel's memory-management subsystem. Owns physical-page allocation (buddy + per-CPU pagesets), slab allocation (SLUB), virtual memory (mm_struct + VMA tree + page tables + page-fault handling), the page cache (folio-based), reclaim and swap, transparent + explicit huge pages, NUMA balancing and mempolicy, cgroup memory accounting, userfaultfd, KSM, memory hotplug, memory failure (hwpoison), and the KASAN/KMSAN/KFENCE/DAMON instrumentation frameworks.

mm/ is the heart of any kernel: ~50K LoC at baseline with the densest concurrency surface (RCU, seqlock, multiple lock orderings, lock-free fast paths) and the most safety-critical invariants (page double-free, page UAF, page-table corruption, page-flag tearing). Per `00-overview.md` D4, mm/ is one of four subsystems where Layer-3 Kani harnesses for data-structure invariants are MANDATORY (alongside kernel/sched, fs/dcache, and the `kernel::arch::x86::paging` page-table walker).

### Requirements

- REQ-1: All mm-related syscalls listed above are implemented with byte-identical entry/exit conventions and identical observable side effects.
- REQ-2: All `/proc` paths listed above produce format-identical content for equivalent kernel state.
- REQ-3: All `/sys` paths listed above exist and respond identically to reads and writes.
- REQ-4: `struct page` flag bit positions (per `include/linux/page-flags.h` enum `pageflags`) match upstream so that `/proc/<pid>/pagemap` and `/sys/kernel/page_owner` parse identically.
- REQ-5: GFP flag semantics (`GFP_KERNEL`, `GFP_ATOMIC`, `GFP_NOWAIT`, `GFP_USER`, `GFP_DMA32`, `__GFP_NOFAIL`, `__GFP_NORETRY`, `__GFP_RETRY_MAYFAIL`, `__GFP_ZERO`, `__GFP_NOWARN`, etc.) match upstream — sleepability, zeroing, retry behavior. Tracked in detail in `page-allocator.md`.
- REQ-6: SLUB cache hardening (CONFIG_SLAB_FREELIST_HARDENED, freelist randomization) is on by default, matching upstream defaults.
- REQ-7: Per-page sanitizers (KASAN, KMSAN, KFENCE) are implemented; their userspace-visible reports (printk format on detection, `/proc/<pid>/kasan_*`, `/sys/kernel/kasan/*`) match upstream.
- REQ-8: DAMON's user interface (`/sys/kernel/mm/damon/`) and userspace-test harness behavior matches upstream.
- REQ-9: Cgroup v2 memory controller (`memory.high`, `memory.max`, `memory.current`, `memory.events`, `memory.swap.*`, `memory.peak`) is implemented identically; cgroup v1 controller (`memory.limit_in_bytes`, `memory.usage_in_bytes`, `memory.stat`) preserved.
- REQ-10: NUMA mempolicy syscalls and `MPOL_*` modes (`MPOL_DEFAULT`, `MPOL_PREFERRED`, `MPOL_BIND`, `MPOL_INTERLEAVE`, `MPOL_LOCAL`, `MPOL_PREFERRED_MANY`, `MPOL_WEIGHTED_INTERLEAVE`) match upstream behavior.
- REQ-11: Memory hotplug events (`udev` `memory<N>` notifications, `online`/`offline` semantics) are byte-identical so distros' udev rules continue working.
- REQ-12: userfaultfd's UFFD_FEATURE_* feature bits and ioctl semantics are identical (CRIU and other consumers depend on this).
- REQ-13: Page-table walker invariants (no stale TLB after invalidation, no torn entries on multi-CPU updates) hold and are mechanically verified per Layer-3 Kani harnesses.
- REQ-14: All Tier-3 docs spawned by this overview each declare their unsafe-block clusters, TLA+ models (where concurrency primitives are introduced), and Kani invariant harnesses. Layer-3 invariant harnesses are MANDATORY for `page-allocator.md` (buddy invariants), `slab.md` (freelist integrity), `virtual-memory.md` (VMA tree + page-table walker), `reclaim.md` (LRU list invariants).
- REQ-15: The implementation reuses upstream rust-for-linux's existing mm abstractions (`rust/kernel/mm.rs`, `rust/kernel/mm/`) and extends rather than parallel-implements.
- REQ-16: Out-of-memory behavior under pressure (oom_kill scoring, oom_score_adj effect, panic_on_oom) matches upstream so that real-world systems' OOM-handler tuning carries over.

### Acceptance Criteria

- [ ] AC-1: `strace` of every mm syscall (`mmap`, `mprotect`, `mremap`, `madvise`, `mlock`, `mincore`, `msync`, `mbind`, `migrate_pages`, `move_pages`, `userfaultfd`, `memfd_create`, `pkey_alloc`, `pkey_mprotect`, `swapon`, `swapoff`, `cachestat`, `process_mrelease`, etc.) on Rookery and upstream produces byte-identical traces given equivalent inputs. (covers REQ-1)
- [ ] AC-2: A diff between Rookery's and upstream's `/proc/<pid>/maps`, `/proc/<pid>/smaps`, `/proc/<pid>/pagemap` for an identical reference binary boot is empty (modulo addresses subject to KASLR). (covers REQ-2)
- [ ] AC-3: Every `/sys/kernel/mm/transparent_hugepage/*`, `/sys/kernel/mm/ksm/*`, `/sys/kernel/mm/hugepages/*`, `/sys/kernel/mm/damon/*` knob exists on Rookery; writing each documented value succeeds with identical observable behavior. (covers REQ-3)
- [ ] AC-4: A `/proc/<pid>/pagemap` parser (e.g., the `tools/vm/page-types` utility from upstream) runs unmodified against a Rookery kernel and reports the same flag set. (covers REQ-4)
- [ ] AC-5: A reference test allocates with each documented GFP flag combination and asserts: sleepability via `might_sleep` instrumentation; zeroing via memcmp to zero buffer; retry behavior via fault-injection. (covers REQ-5)
- [ ] AC-6: KFENCE detection injection produces a printk-format report byte-identical (modulo timestamps) to upstream's. (covers REQ-7)
- [ ] AC-7: cgroup v2 selftests under `tools/testing/selftests/cgroup/` pass with the same pass/fail set as upstream. (covers REQ-9)
- [ ] AC-8: NUMA selftests (`tools/testing/selftests/mm/numa-related`) pass identically. (covers REQ-10)
- [ ] AC-9: An udev rule listening on `/sys/devices/system/memory/memory<N>` `online`/`offline` events fires identically. (covers REQ-11)
- [ ] AC-10: CRIU's userfaultfd-based snapshotting works against Rookery without modification. (covers REQ-12)
- [ ] AC-11: `make verify` passes all `kernel/mm/proofs/` Kani harnesses; `make tla` passes all `kernel/mm/models/` TLA+ models. (covers REQ-13, REQ-14)
- [ ] AC-12: A grep over Rookery for `pub use kernel::mm::*` finds usages but not parallel `pub use kernel::memory::*` or `pub use kernel::vm::*` namespaces. (covers REQ-15)
- [ ] AC-13: Reference workloads (kernel build, postgres, redis) running under memory-pressure exhibit identical OOM-kill victim selection on Rookery and upstream. (covers REQ-16)

### Architecture

### Layout map (Tier-3 docs spawned from this overview)

```
.design/mm/
  00-overview.md             ← this document
  page-allocator.md          ← buddy + per-CPU pageset + zones + folios + GFP semantics
  slab.md                    ← SLUB + freelist hardening + kmem_cache + mempool/dmapool helpers
  virtual-memory.md          ← mm_struct, VMA tree (maple), page faults, COW, GUP, rmap, page-writeback
  mmap.md                    ← mmap/mprotect/mremap/munmap/madvise/mlock/mincore/msync/process_madvise syscalls
  page-cache.md              ← filemap, folio operations, readahead, truncate
  reclaim.md                 ← vmscan, LRU lists, MGLRU, shrinker, kswapd, oom_kill, list_lru, page-writeback orchestration
  swap.md                    ← swapfile, swap_state, swap_cgroup, page_io for swap, zswap (compressed swap cache)
  zsmalloc.md                ← compressed page-pool allocator (used by zswap, zram)
  thp.md                     ← transparent huge pages, hugetlb_vmemmap, khugepaged
  hugetlb.md                 ← explicit huge pages (hugetlbfs cross-references fs/00-overview.md)
  migration-compaction.md    ← page migration, compaction (anti-fragmentation)
  ksm.md                     ← Kernel Samepage Merging
  userfaultfd.md             ← userfaultfd syscall + UFFD_* feature bits + ioctls
  memcg.md                   ← memory cgroup v1 + v2 (cross-references kernel/cgroup/)
  numa-mempolicy.md          ← NUMA balancing, mbind/get_mempolicy/set_mempolicy/migrate_pages/move_pages syscalls
  memory-hotplug.md          ← add/remove memory at runtime + udev events
  memory-failure.md          ← hwpoison, mce-driven memory-failure, soft-offline
  vmalloc.md                 ← vmalloc/vfree, virtual address arena, lazy free
  bootmem.md                 ← memblock early allocator, sparse memory model, init-mm setup, memtest
  percpu.md                  ← per-CPU kernel allocator
  hmm.md                     ← Heterogeneous Memory Management (drivers / GPU helpers)
  maccess.md                 ← safe-failable memory access (probe_kernel_read / probe_user_read variants)
  sanitizers.md              ← KASAN, KMSAN, KFENCE setup and runtime
  damon.md                   ← Data Access MONitor — autonomous reclaim policy framework
  mm-debug.md                ← page_owner, page_ext, page_idle, page_poison, page_table_check, debug_page_alloc, debug_vm_pgtable, rodata_test
```

### Cross-references

- `arch/x86/00-overview.md` § paging.md — x86-specific page table mechanics, KASLR, KASAN init, AMD SEV/SME memory encryption. The arch tier owns *how* the hardware MMU is programmed; this mm tier owns *what* virtual memory means abstractly.
- `lib/00-overview.md` — `iov_iter` (used by GUP), maple_tree (the VMA tree), data structures used by mm.
- `kernel/00-overview.md` (Phase B) — `kernel/cgroup/` substrate; cross-references from `memcg.md`.
- `fs/00-overview.md` (Phase B) — `fs/hugetlbfs/` is the userspace-mounted FS for explicit huge pages; cross-referenced from `hugetlb.md`. `fs/proc/` exposes mm's `/proc` files; cross-referenced from this overview.
- `block/00-overview.md` (Phase B) — `mm/page_io.c` integrates with the block layer for swap I/O; cross-ref from `swap.md`.
- `00-glossary.md` — `page`, `folio`, `pgd/p4d/pud/pmd/pte`, `vma`, `mm_struct`, `slab`, `GFP flags`, `buddy allocator`, `zone`, `node`, `vmalloc`, `swap`, `THP`.

### Rust module organization (informative)

- `kernel::mm` — exists upstream (`rust/kernel/mm.rs`, `rust/kernel/mm/`) — extend
- `kernel::mm::page_alloc` — Rookery to author
- `kernel::mm::slab` — Rookery to author (extends upstream `kernel::alloc::{KBox, KVec}` with cache-creation)
- `kernel::mm::vma` — Rookery to author
- `kernel::mm::page_cache` — Rookery to author
- `kernel::mm::reclaim` — Rookery to author
- `kernel::mm::swap` — Rookery to author
- `kernel::mm::userfaultfd` — Rookery to author
- `kernel::mm::memcg` — Rookery to author
- `kernel::mm::numa` — Rookery to author
- `kernel::mm::vmalloc` — Rookery to author
- `kernel::mm::percpu` — Rookery to author (overlaps issue #4 PerCpu<T> abstraction)
- `kernel::mm::sanitizers::{kasan, kmsan, kfence}` — Rookery to author; KASAN already has a Rust-side test fixture upstream (`mm/kasan/kasan_test_rust.rs`)

### Locking and concurrency

mm/ is the densest concurrency surface in the kernel. The locking landscape:

- `mm_struct.mmap_lock` (rwsem) — protects the VMA tree; held for read in page-fault, for write in mmap/mprotect/mremap. Per-mm.
- Per-zone `zone->lock` (spinlock) — protects buddy free lists per migrate-type per order.
- Per-CPU pageset locks (preempt-disabled regions) — fast-path page allocation.
- `lru_lock` (spinlock per memcg per node per LRU type) — protects LRU lists.
- `i_pages` (xarray lock) — page-cache lookup.
- `swap_lock`, `swap_avail_lock` — swap subsystem.
- `pte_lockptr` (spinlock per page-table page on x86) — protects PTEs during update.
- RCU for many readers (`rcu_dereference` over xarray pages, mempolicy lookups, etc.).
- Seqlock for mempolicy reads.
- `mm_struct.page_table_lock` (mm-wide spinlock) — fallback for some ops.

Tier-3 docs each declare which locks they own. TLA+ models live with the primitives that have novel concurrency contracts (vmscan's reclaim ordering, memcg's hierarchical accounting, KSM's stable-tree mutex, userfaultfd's wait-queue + uffd_msg_lock).

### Error handling

mm/ allocations are fundamentally fallible. All allocation paths return `Result<T, KernelError>` per `00-rust-conventions.md` REQ-6. Common returns:
- `Err(ENOMEM)` — allocation failure
- `Err(EFAULT)` — bad userspace pointer (in usercopy + GUP paths)
- `Err(EINVAL)` — bad flags or alignment
- `Err(EAGAIN)` — temporary failure (e.g., backoff during compaction, NUMA-balancing)
- `Err(ENOSPC)` — swap full, hugepage pool exhausted

Page-fault handler is the special case: it does NOT return `Result` to the page-fault caller (CPU exception). Instead, mm encodes the outcome as a `PageFaultOutcome` enum (Continue, RetryFault, DeliverSignal) for the arch caller to act on. Defined in `kernel::mm::vma::page_fault_outcome` (Tier 3 in `virtual-memory.md`).

### Out of Scope

- 32-bit-only kernel mm paths (`mm/highmem.c`, `mm/highmem-internal.h`) — out of scope per `arch/x86/00-overview.md` D1 (no `X86_32=y` in v0).
- frontswap (deprecated upstream) — not reverse-engineered.
- CXL memory hot-add path beyond what `memory-hotplug.md` already covers — Cross-references DAX storage and CXL devices land in `drivers/00-overview.md` Phase E.
- DAX (Direct Access for persistent memory) — covered in `fs/00-overview.md` (DAX-aware filesystems) + `drivers/00-overview.md` (DAX block driver). The `mm/memremap.c` part is in scope here under `bootmem.md`.
- Non-x86 sanitizers (HWASAN ARM64, etc.) — out of v0.
- BPF page-cache APIs — covered in `kernel/00-overview.md` (kernel/bpf/).
- DAMOS (DAMON-based Operations Schemes) advanced policies — covered in `damon.md` Tier 3 but Rookery v0 does not extend beyond upstream parity.
- Implementation code — `.design/` contains specs only.

### upstream references in scope

`mm/` (128 top-level files + 5 subdirs at baseline). Categorized into the Tier-3 docs spawned from this overview:

| Category | Upstream paths (selection) | Planned Tier-3 doc |
|---|---|---|
| Page allocator (buddy + per-CPU pagesets + zones + folios) | `mm/page_alloc.c`, `mm/page_isolation.c`, `mm/page_reporting.c`, `mm/folio-compat.c`, `mm/page_owner.c`, `mm/page_ext.c`, `mm/page_idle.c`, `mm/page_poison.c`, `mm/page_table_check.c`, `mm/page_vma_mapped.c`, `include/linux/{mmzone,page-flags,gfp,gfp_types}.h`, `include/linux/mm_types.h` (struct page, struct folio) | `page-allocator.md` |
| Slab allocator | `mm/slub.c`, `mm/slab_common.c`, `mm/slab.h`, `include/linux/slab.h` | `slab.md` |
| Virtual memory core (mm_struct, page faults, page tables) | `mm/memory.c`, `mm/init-mm.c`, `mm/rmap.c`, `mm/gup.c`, `mm/maccess.c`, `mm/page-writeback.c`, `include/linux/{mm,mm_types}.h` | `virtual-memory.md` |
| mmap and friends (mmap/mprotect/mremap/madvise/mlock/mincore syscalls) | `mm/mmap.c`, `mm/mprotect.c`, `mm/mremap.c`, `mm/madvise.c`, `mm/mlock.c`, `mm/mincore.c`, `mm/util.c` (mmap helpers) | `mmap.md` |
| Page cache (folio-based) | `mm/filemap.c`, `mm/readahead.c`, `mm/truncate.c`, `include/linux/pagemap.h` | `page-cache.md` |
| Reclaim, LRU, OOM | `mm/vmscan.c`, `mm/swap.c` (LRU mgmt), `mm/shrinker.c`, `mm/oom_kill.c`, `mm/show_mem.c`, `mm/list_lru.c` | `reclaim.md` |
| Swap | `mm/swapfile.c`, `mm/swap_state.c`, `mm/swap_cgroup.c`, `mm/page_io.c`, `mm/zswap.c`, `include/linux/swap.h` | `swap.md` |
| Compressed page pool | `mm/zsmalloc.c`, `include/linux/zsmalloc.h` | `zsmalloc.md` |
| Transparent huge pages | `mm/huge_memory.c`, `mm/hugetlb_vmemmap.c`, `include/linux/huge_mm.h` | `thp.md` |
| Explicit huge pages (hugetlbfs) | `mm/hugetlb.c`, `include/linux/hugetlb.h`, `fs/hugetlbfs/` (cross-ref to `fs/00-overview.md`) | `hugetlb.md` |
| Migration + compaction | `mm/migrate.c`, `mm/compaction.c` | `migration-compaction.md` |
| KSM | `mm/ksm.c`, `include/linux/ksm.h` | `ksm.md` |
| userfaultfd | `mm/userfaultfd.c`, `include/linux/userfaultfd_k.h`, `include/uapi/linux/userfaultfd.h` | `userfaultfd.md` |
| Memory cgroup | `mm/memcontrol.c`, `include/linux/memcontrol.h` | `memcg.md` (cross-ref to `kernel/cgroup/`) |
| NUMA + mempolicy | `mm/mempolicy.c`, `mm/numa.c`, `mm/numa_emulation.c`, `include/uapi/linux/mempolicy.h` | `numa-mempolicy.md` |
| Memory hotplug | `mm/memory_hotplug.c`, `mm/sparse.c`, `include/linux/memory_hotplug.h` | `memory-hotplug.md` |
| Memory failure (hwpoison) | `mm/memory-failure.c`, `mm/hwpoison-inject.c`, `include/linux/mm.h` (poison flags) | `memory-failure.md` |
| vmalloc | `mm/vmalloc.c`, `include/linux/vmalloc.h` | `vmalloc.md` |
| Boot-time memory (memblock, sparse, early init) | `mm/memblock.c`, `mm/bootmem_info.c`, `mm/sparse.c`, `mm/init-mm.c`, `mm/memtest.c`, `include/linux/memblock.h` | `bootmem.md` |
| Per-CPU kernel memory | `mm/percpu.c`, `mm/percpu-vm.c` | `percpu.md` |
| HMM (Heterogeneous Memory Management — driver/GPU helpers) | `mm/hmm.c`, `include/linux/hmm.h` | `hmm.md` |
| Memory pools | `mm/mempool.c`, `mm/dmapool.c`, `include/linux/mempool.h`, `include/linux/dmapool.h` | folded into `slab.md` |
| Sanitizers | `mm/kasan/`, `mm/kmsan/`, `mm/kfence/` | `sanitizers.md` |
| DAMON (Data Access MONitor) | `mm/damon/` | `damon.md` |
| Debug + introspection | `mm/debug.c`, `mm/debug_page_alloc.c`, `mm/debug_page_ref.c`, `mm/debug_vm_pgtable.c`, `mm/rodata_test.c` | `mm-debug.md` |

Cross-arch interaction: `arch/x86/mm/` (paging implementation, page-fault handler, ioremap, KASLR-paging, memory encryption) is owned by `arch/x86/00-overview.md` § paging.md. The split: `arch/x86/mm/` owns x86-specific page-table mechanics; `mm/` owns the architecture-agnostic VM model. The two cooperate through `arch::*` traits.

### compatibility contract

mm/ owns a large slice of the userspace-visible compat surface:

### Syscall surface

| Syscall | UAPI header | Compat level |
|---|---|---|
| `mmap`, `mmap2`, `munmap`, `mremap`, `mprotect`, `pkey_mprotect`, `madvise`, `process_madvise`, `mlock`, `mlock2`, `munlock`, `mlockall`, `munlockall`, `mincore`, `msync` | `include/uapi/asm-generic/mman*.h`, `include/uapi/asm/mman.h` | Identical (each gets a Tier-5 `uapi/syscalls/<name>.md`) |
| `swapon`, `swapoff` | `include/uapi/linux/swap.h` | Identical |
| `mbind`, `get_mempolicy`, `set_mempolicy`, `migrate_pages`, `move_pages` | `include/uapi/linux/mempolicy.h` | Identical |
| `userfaultfd` | `include/uapi/linux/userfaultfd.h` | Identical (incl. all `UFFD_*` ioctls) |
| `pkey_alloc`, `pkey_free` | `include/uapi/asm-generic/mman.h` (PROT_*) + arch headers | Identical |
| `memfd_create`, `memfd_secret` | `include/uapi/linux/memfd.h` | Identical |
| `cachestat` (5.18+) | `include/uapi/linux/cachestat.h` | Identical |
| `map_shadow_stack` (CET) | arch-specific | Identical |
| `process_mrelease` (5.15+) | (no uapi header beyond `linux/syscalls.h`) | Identical |

### `/proc` surfaces

| Path | Content | Owner doc | Compat level |
|---|---|---|---|
| `/proc/<pid>/maps` | Per-VMA listing: addr range, perms, offset, dev, inode, path | `mmap.md` | Format-identical (compat-critical for gdb, valgrind, strace, perf, libunwind, /lib/x86_64-linux-gnu/libc.so.6 mapping detection) |
| `/proc/<pid>/smaps`, `/proc/<pid>/smaps_rollup` | Per-VMA detailed memory accounting | `mmap.md` | Format-identical (per `Documentation/filesystems/proc.rst`) |
| `/proc/<pid>/pagemap` | Per-page bitfield exposing PFN, swap, soft-dirty, file/anon, exclusive | `virtual-memory.md` | Bit-identical (pmap and other tools binary-parse it) |
| `/proc/<pid>/numa_maps` | Per-VMA NUMA stats | `numa-mempolicy.md` | Format-identical |
| `/proc/<pid>/status` (mm-related fields: VmPeak, VmSize, VmLck, VmPin, VmHWM, VmRSS, VmData, VmStk, VmExe, VmLib, VmPTE, VmSwap) | Process resource use | `virtual-memory.md` | Field-identical |
| `/proc/<pid>/stat` (RSS field, vsize) | | `virtual-memory.md` | Field-identical |
| `/proc/<pid>/oom_score`, `/proc/<pid>/oom_score_adj`, `/proc/<pid>/oom_adj` | OOM-killer accounting | `reclaim.md` | Numeric-identical |
| `/proc/meminfo` | System-wide memory stats (MemTotal, MemFree, MemAvailable, Buffers, Cached, SwapCached, Active, Inactive, AnonPages, Mapped, Shmem, KReclaimable, Slab, SReclaimable, SUnreclaim, KernelStack, PageTables, NFS_Unstable, Bounce, WritebackTmp, CommitLimit, Committed_AS, VmallocTotal, VmallocUsed, VmallocChunk, Percpu, HardwareCorrupted, AnonHugePages, ShmemHugePages, ShmemPmdMapped, FileHugePages, FilePmdMapped, CmaTotal, CmaFree, HugePages_Total, HugePages_Free, HugePages_Rsvd, HugePages_Surp, Hugepagesize, Hugetlb, DirectMap4k, DirectMap2M, DirectMap1G) | `reclaim.md` (umbrella) | Field name + units identical; values match given equivalent kernel state |
| `/proc/buddyinfo`, `/proc/pagetypeinfo`, `/proc/zoneinfo` | Buddy-allocator statistics per node/zone | `page-allocator.md` | Format-identical |
| `/proc/slabinfo` | Per-cache slab statistics | `slab.md` | Format-identical |
| `/proc/vmstat` | Counters per-zone and per-node | `reclaim.md` | Field-identical |
| `/proc/vmallocinfo` | vmalloc allocations | `vmalloc.md` | Format-identical |
| `/proc/swaps` | Active swap devices | `swap.md` | Format-identical |
| `/proc/sys/vm/*` | Sysctl knobs (swappiness, overcommit_memory, overcommit_ratio, dirty_*, vfs_cache_pressure, etc.) | various Tier-3 docs per knob | Knob-name + semantics identical |

### `/sys` surfaces

| Path | Owner doc | Compat level |
|---|---|---|
| `/sys/kernel/mm/transparent_hugepage/{enabled,defrag,khugepaged/*}` | `thp.md` | File-name + value-set identical |
| `/sys/kernel/mm/ksm/{run,sleep_millisecs,pages_to_scan,advisor,*}` | `ksm.md` | File-name + value-set identical |
| `/sys/kernel/mm/hugepages/hugepages-<size>kB/{nr_hugepages,nr_overcommit_hugepages,free_hugepages,resv_hugepages,surplus_hugepages,demote,demote_size}` | `hugetlb.md` | File-name + value-set identical |
| `/sys/kernel/mm/numa/demotion_enabled` | `numa-mempolicy.md` | Identical |
| `/sys/kernel/mm/lru_gen/*` (MGLRU) | `reclaim.md` | Identical |
| `/sys/kernel/mm/damon/*` | `damon.md` | Identical |
| `/sys/devices/system/memory/memory<N>/state`, `online`, etc. | `memory-hotplug.md` | Identical (used by drivers and udev rules) |
| `/sys/devices/system/node/node<N>/meminfo`, `numastat`, `vmstat`, `hugepages/` | `numa-mempolicy.md` | Identical |
| `/sys/kernel/slab/<cache>/*` (when CONFIG_SLUB_DEBUG=y) | `slab.md` | Identical |

### Page-flag bit definitions (visible via `/proc/<pid>/pagemap` and `/sys/kernel/page_owner`)

`include/linux/page-flags.h` defines flags whose bit positions are user-visible (ABI). Bit positions MUST match upstream so that pagemap-parsing tools (e.g., `pmap`, custom monitoring) work identically.

### Folio/struct-page layout (pmap, perf, drivers)

`struct folio` and `struct page` layouts are kernel-internal but their *first cache line* is observed by drivers and out-of-tree code via inline-fastpath macros (PageCompound, PageDirty, etc.). The expected level here is layout-equivalent (same field offsets) for the cache-line-zero fields; deeper fields may be reorganized.

### verification

Per `00-overview.md` D4, mm/ is one of the four mandatory Layer-3 subsystems. This section sets the floor; each Tier-3 doc fleshes out specifics.

### Layer 1: Kani SAFETY proofs (mandatory all `unsafe` blocks)

Anticipated `unsafe` clusters:
- Page-table entry read/write (`kernel::mm::page_table::*`) — buddy-with-arch tier proofs in `arch/x86/paging.md`
- Folio splitting + compounding (`kernel::mm::folio::*`)
- Slab object freelist pointer dereference (`kernel::mm::slab::*`)
- VMA list manipulation (`kernel::mm::vma::*`)
- LRU list manipulation (`kernel::mm::reclaim::*`)
- vmalloc page-table modifications (`kernel::mm::vmalloc::*`)
- Per-CPU pageset access (`kernel::mm::page_alloc::*`)
- GUP raw pin-page operations (`kernel::mm::gup::*`)
- userfaultfd page-fault hand-off (`kernel::mm::userfaultfd::*`)

### Layer 2: TLA+ models (mandatory for novel concurrency)

Models required at this tier:
- `models/mm/buddy.tla` — proves buddy allocator invariants under concurrent allocation/free across multiple CPUs and zones. Safety: no double-allocation, no free-list corruption.
- `models/mm/slub_per_cpu.tla` — proves SLUB per-CPU partial-list + tail-pointer invariants under concurrent CPU-local allocation and IPI-driven cache drain.
- `models/mm/lru.tla` — proves LRU list ordering under concurrent reclaim + add + touch operations, including memcg hierarchical accounting.
- `models/mm/mmap_lock.tla` — proves the rwsem-based mmap_lock plus per-VMA-lock fast path correctly serializes readers vs. writers without livelock under page-fault retry.
- `models/mm/userfaultfd.tla` — proves UFFD ioctl/wait/wake correctness — specifically that no fault is lost across `UFFDIO_COPY` + `UFFDIO_WAKE` on multiple CPUs.
- `models/mm/swap_atomic.tla` — proves `swap_info_struct.bdev_max_pages` + slot allocation never produces double-allocation under concurrent `get_swap_pages`/`put_swap_pages`.
- `models/mm/memcg_hierarchy.tla` — proves cgroup v2 memcg counter-roll-up correctness under concurrent allocation across descendant cgroups.

### Layer 3: Kani harnesses for data-structure invariants (MANDATORY)

mm/ data-structure invariants enumerated for Kani harnesses:

| Data structure | Invariant | Harness |
|---|---|---|
| Buddy allocator free lists (per zone, per migrate-type, per order) | "No physical page address appears in two free lists simultaneously" + "Every free entry's order matches the list it lives in" | `kani::proofs::mm::page_alloc::buddy_invariants` |
| SLUB freelist | "Every cached object's `freelist_next` pointer points within the same slab page" + "No object appears twice in the freelist" | `kani::proofs::mm::slab::freelist_invariants` |
| VMA maple tree | (delegated to `lib/data-structures.md` `kani::proofs::lib::maple_tree::invariants`); plus mm-specific: "No two VMAs in one mm overlap" | `kani::proofs::mm::vma::no_overlap_invariant` |
| Page tables | "Every PTE's PFN resolves to a valid page-table page or to a populated `struct page`" + "Page-table walker preserves access/dirty bits semantics" | `kani::proofs::mm::page_table::invariants` |
| LRU lists | "Every page on an LRU list has the corresponding LRU page-flag set" + "Memcg accounting is consistent: sum-of-charges equals sum-of-page-references" | `kani::proofs::mm::reclaim::lru_invariants` |
| Refcount on `struct folio` | "Refcount never reaches zero with pinned references outstanding" | `kani::proofs::mm::folio::refcount_invariants` |
| Swap slot allocator | "Every allocated slot is in exactly one of: in-flight read, in-flight write, mapped to a swap-cache folio, free" | `kani::proofs::mm::swap::slot_invariants` |
| Userfaultfd wait queue | "Every fault that hits a UFFD region is enqueued exactly once before being dispatched" | `kani::proofs::mm::userfaultfd::queue_invariants` |

### Layer 4: Functional correctness via Creusot / Verus / Prusti (opt-in)

Strong opt-in candidates (declared in their Tier-3 docs):
- `page-allocator.md` — buddy splitting/coalescing arithmetic via Creusot. Algorithm is small and well-defined; provides a high-leverage proof of "the buddy allocator returns a properly-aligned region of the requested size".
- `slab.md` — slab cache color/offset arithmetic via Creusot.
- `vmalloc.md` — virtual-address-arena fragmentation analysis (size/alignment correctness) via Creusot.
- `reclaim.md` — MGLRU generation accounting; harder, lower priority.

### hardening

Placeholder per `00-overview.md` D6. mm/ owns implementation of:

- **PAX_USERCOPY-equivalent**: enforced via type-system in `00-rust-conventions.md` (`UserPtr<T>`); slab caches type-tagged for participation in `copy_to/from_user`.
- **PAX_AUTOSLAB-equivalent**: per-callsite slab-cache typing (Rust's `core::any::type_name` is already discriminated by type).
- **PAX_RANDSTRUCT-equivalent**: Rust's default `repr(Rust)` provides layout freedom; opt-in `#[repr(C)]` only where layout stability matters.
- **PAX_MEMORY_SANITIZE-equivalent**: zero on free for slab caches marked `SENSITIVE`.
- **PAX_DELAY_FREE_ONE_PAGE-equivalent**: page-quarantine on free (covered in `page-allocator.md`).
- **CONFIG_SLAB_FREELIST_HARDENED + CONFIG_SLAB_FREELIST_RANDOM**: SLUB hardening is on by default, matching upstream.
- **KASAN, KMSAN, KFENCE**: covered in `sanitizers.md`.

Each Tier-3 doc whose content touches a hardening feature gains a "Hardening" section.

Per D6, the binding security policy lands in `00-security-principles.md` after this `mm/00-overview.md` is reviewed (issue #2 unblocks once mm/ + arch/x86/ overviews are reviewed).

