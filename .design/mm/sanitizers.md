# Tier-3: mm/kasan/ + mm/kmsan/ + mm/kfence/ — Memory sanitizers (KASAN, KMSAN, KFENCE)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: mm/00-overview.md
upstream-paths:
  - mm/kasan/{common, generic, hw_tags, init, quarantine, report}.c
  - mm/kmsan/{init, instrumentation, kmsan, kmsan_test, ...}.c
  - mm/kfence/core.c, kfence_test.c
  - include/linux/kasan.h
  - include/linux/kmsan.h
  - include/linux/kfence.h
  - Documentation/dev-tools/{kasan, kmsan, kfence}.rst
-->

## Summary

Linux supports three runtime memory sanitizers: **KASAN** (KernelAddressSanitizer; bug-detector for use-after-free, out-of-bounds, double-free), **KMSAN** (KernelMemorySanitizer; uninitialized-read detector), **KFENCE** (Kernel Electric-Fence; low-overhead sampling-based UAF/OOB detector for prod). KASAN has 3 modes: generic (compiler-instrumented per-load/store), software-tags (4-bit tag in pointer high-bits), hw-tags (ARM64-only MTE). KMSAN: per-byte shadow + origins; runtime instrumentation per-load/store. KFENCE: sampling fence-pages around tiny pool of allocations; per-fault-on-touch detects bugs at near-zero cost. Critical for: kernel-bug detection during development (KASAN/KMSAN) + production (KFENCE).

This Tier-3 covers the three sanitizer subsystems.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `kasan_init()` | per-arch KASAN init | `Kasan::init` |
| `kasan_check_byte()` / `kasan_check_*` | per-access shadow check | `Kasan::check_*` |
| `kasan_alloc_pages()` / `kasan_free_pages()` | per-buddy alloc/free | `Kasan::alloc_pages` / `free_pages` |
| `kasan_kmalloc()` / `kasan_kfree_large()` | per-slab alloc/free | `Kasan::kmalloc` / `kfree_large` |
| `kasan_save_alloc_info()` | per-track stack | `Kasan::save_alloc_info` |
| `kasan_record_aux_stack()` | per-track aux stack | `Kasan::record_aux_stack` |
| `kasan_report()` | per-bug report | `Kasan::report` |
| `quarantine_*` | per-freed-slab quarantine | `Kasan::quarantine` |
| `kmsan_init_runtime()` | per-arch KMSAN init | `Kmsan::init_runtime` |
| `kmsan_alloc_page()` / `kmsan_free_page()` | per-buddy track | `Kmsan::alloc_page` / `free_page` |
| `kmsan_init_alloc()` / `kmsan_handle_dma()` | per-track init | `Kmsan::init_alloc` |
| `__msan_*` (compiler instrumentation) | per-load/store callbacks | `Kmsan::__msan_*` |
| `kmsan_get_metadata()` | per-page → shadow / origin | `Kmsan::get_metadata` |
| `kfence_init()` | per-arch KFENCE init | `Kfence::init` |
| `kfence_alloc()` | per-slab sample candidate | `Kfence::alloc` |
| `kfence_handle_page_fault()` | per-fence-fault | `Kfence::handle_page_fault` |
| `__kfence_pool` | per-pool of N (~256) protected slots | shared |
| `KFENCE_SAMPLE_INTERVAL` | per-sample-rate ms | shared |

## Compatibility contract

REQ-1: KASAN modes:
- generic (CONFIG_KASAN_GENERIC): per-byte shadow, 1:8 ratio.
- sw-tags (CONFIG_KASAN_SW_TAGS): 4-bit tag in high-bits of ptr (top-byte-ignore on ARM64).
- hw-tags (CONFIG_KASAN_HW_TAGS): ARM64 MTE; HW tag-bits in ptr + page metadata.

REQ-2: KASAN shadow:
- Per-8-bytes-of-virt-addr → 1-byte shadow.
- Shadow-byte values:
  - 0x00: addressable (all 8 bytes valid).
  - 0x01..0x07: partially-addressable (first N bytes valid).
  - 0xFA: alloca redzone.
  - 0xFB: stack-after-return.
  - 0xFC: stack-redzone.
  - 0xF7: heap-redzone.
  - 0xFA..0xFE: per-bug-class poison.
- Per-load/store: compiler emits `__asan_load1` / `__asan_store4` / etc. → kasan_check_*.

REQ-3: KASAN per-allocation tracking:
- kasan_save_alloc_info: capture stack trace at alloc.
- kasan_record_aux_stack: capture stack at second access (e.g. wq-callback).
- Per-slab: track via meta_data in object header.
- Per-bug: report contains alloc-stack + free-stack + access-stack.

REQ-4: KASAN quarantine:
- Per-freed slab object: held in quarantine FIFO (~16MB default).
- Prevents immediate reuse → catches use-after-free.

REQ-5: KMSAN shadow + origins:
- Per-byte shadow: bit = 1 if uninitialized.
- Per-4-bytes origin: 32-bit pointer to origin-stack-trace.
- Compiler emits __msan_load_*/__msan_store_* per access.
- Per-uninit-load + branch-on: report.

REQ-6: KMSAN integration:
- kmsan_init_alloc: alloc-result marked initialized iff GFP_ZERO.
- kmsan_init_kernel_obj: per-static-data initialized.
- kmsan_handle_dma: per-DMA tx initialized post-completion.
- kmsan_check_memory: per-call site forced check.

REQ-7: KFENCE pool:
- Pre-allocated pool of N slots (default 255).
- Each slot: 1 page allocation + 1 unmapped fence-page after.
- Per-kfence_alloc: sample at KFENCE_SAMPLE_INTERVAL (ms).
- Per-touch-fence-page: page-fault → kfence_handle_page_fault → bug-report.

REQ-8: KFENCE alloc-decision:
- Per-slab-alloc (kmem_cache_alloc): occasionally diverted.
- Per-non-slab pages: not covered.

REQ-9: Per-bug-report format:
- ==== KASAN: <bug-class> in <fn> ====
- Or ==== KMSAN: uninit-value in <fn> ====
- Or ==== KFENCE: out-of-bounds in <fn> ====
- + per-stack-trace: alloc-by, freed-by, accessed-by.
- + per-shadow-dump: bytes around access.

REQ-10: Per-config build:
- CONFIG_KASAN: heavy compile-time + runtime instrumentation; ~3x slowdown.
- CONFIG_KMSAN: ~3-4x slowdown + ~2x mem.
- CONFIG_KFENCE: ~few% overhead at default sample rate.

REQ-11: Per-userspace ABI:
- /sys/kernel/kfence_sample_interval: tuneable.
- /proc/<pid>/kasan_enabled: per-task toggle (rare).

## Acceptance Criteria

- [ ] AC-1: KASAN-generic build: kasan_init populates shadow.
- [ ] AC-2: kmalloc + free + access: KASAN reports use-after-free.
- [ ] AC-3: kmalloc(8) + access[8]: KASAN reports out-of-bounds.
- [ ] AC-4: KASAN quarantine: freed obj not immediately reused.
- [ ] AC-5: KMSAN: read-from-uninitialized-stack: report uninit-value.
- [ ] AC-6: KMSAN: kzalloc'd buf: no false-positive.
- [ ] AC-7: KMSAN: per-DMA RX completes: bytes initialized.
- [ ] AC-8: KFENCE alloc sampled: kmem_cache_alloc returns kfence-page.
- [ ] AC-9: KFENCE OOB-write to fence-page: handle_page_fault → report.
- [ ] AC-10: KFENCE freed + access: report use-after-free.
- [ ] AC-11: Per-build no-sanitizer: zero-overhead.

## Architecture

KASAN per-shadow:

```
const KASAN_SHADOW_SCALE_SHIFT: u8 = 3;          // 1:8 ratio
const KASAN_FREE_PAGE: u8 = 0xFF;
const KASAN_KMALLOC_REDZONE: u8 = 0xFC;
const KASAN_KMALLOC_FREE: u8 = 0xFB;
const KASAN_KMALLOC_FREETRACK: u8 = 0xFA;

fn kasan_check_byte(addr: u64) -> Result<()>:
  shadow = mem_to_shadow(addr).
  byte = *shadow.
  offset = addr & 7.
  if byte == 0: return Ok.
  if (offset >= byte): return Err(OutOfBounds).
  Ok.
```

KMSAN per-shadow + origin:

```
fn kmsan_metadata_for(addr: u64) -> (shadow: u64, origin: u64):
  /* per-page meta lookup */
  page = virt_to_page(addr).
  shadow = page.shadow + (addr & ~PAGE_MASK).
  origin = page.origin + (addr & ~PAGE_MASK).

fn __msan_load4(ptr: *u32) -> u32:
  shadow = kmsan_metadata_for(ptr).shadow.
  if shadow != 0: kmsan_report(uninit-load).
  return *ptr.
```

KFENCE per-pool:

```
struct KfenceMetadata {
  state: KfenceObjectState,                      // FREED / ALLOCATED
  addr: u64,
  size: u32,
  alloc_track: AllocTrack,
  free_track: AllocTrack,
}

const KFENCE_NUM_OBJECTS: usize = 255;
static KFENCE_METADATA: [KfenceMetadata; KFENCE_NUM_OBJECTS];
static __KFENCE_POOL: *u8;                        // virt-region of 255 × 2 pages

fn kfence_alloc(s: *KmemCache, size: usize, flags: GfpT) -> Option<*u8>:
  if !sample_now: return None.
  /* pick free slot */
  meta = pool_find_free_slot().
  if !meta: return None.
  meta.state = ALLOCATED; meta.size = size.
  return meta.addr.
```

`Kasan::init()`:
1. /* arch supplies shadow region */
2. kasan_populate_shadow(start, end).
3. For per-page: kasan_unpoison_range(addr, size).

`Kasan::kmalloc(s, object, size, flags)`:
1. kasan_save_alloc_info(object).
2. /* Mark redzone */
3. kasan_unpoison_object_data(object, size).
4. kasan_poison_redzone(object + size, redzone_size).

`Kasan::quarantine_put(object)`:
1. queue object on per-CPU quarantine.
2. If quarantine over-cap: dequeue oldest → actually-free.

`Kmsan::init_alloc(folio, order)`:
1. /* Mark page initialized iff GFP_ZERO */
2. shadow = kmsan_get_metadata(page_addr(folio), shadow).
3. memset(shadow, 0, size).  // 0 = initialized.

`Kfence::handle_page_fault(addr, esr, regs)`:
1. /* Check addr ∈ __kfence_pool fence region */
2. meta = pool_find_meta_by_addr(addr).
3. /* Determine bug-class */
4. if meta.state == ALLOCATED ∧ addr beyond meta.size: KFENCE_OOB.
5. if meta.state == FREED: KFENCE_USE_AFTER_FREE.
6. report_bug(KFENCE_BUG_CLASS, regs).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `kasan_shadow_addr_in_range` | INVARIANT | per-mem_to_shadow: result in shadow-region. |
| `kmsan_shadow_origin_aligned` | INVARIANT | per-shadow/origin lookup aligned. |
| `kfence_meta_lt_num_objects` | INVARIANT | per-meta-idx < KFENCE_NUM_OBJECTS. |
| `quarantine_size_le_max` | INVARIANT | per-quarantine: total bytes ≤ max. |
| `sanitizer_disabled_no_overhead` | INVARIANT | per-non-CONFIG_KASAN: no shadow check. |

### Layer 2: TLA+

`mm/sanitizers.tla`:
- Per-alloc/free shadow-poison + per-access shadow-check.
- Properties:
  - `safety_uaf_detected_post_free` — per-kfree + access ⟹ KASAN reports.
  - `safety_oob_detected_redzone` — per-access beyond size ⟹ KASAN reports.
  - `safety_uninit_load_detected` — per-uninit-load + branch ⟹ KMSAN reports.
  - `liveness_kfence_eventually_samples` — per-N-allocs: at least one kfence-sampled.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Kasan::kmalloc` post: object unpoisoned [0, size); redzone poisoned | `Kasan::kmalloc` |
| `Kasan::quarantine_put` post: object NOT immediately re-allocatable | `Kasan::quarantine_put` |
| `Kmsan::init_alloc` post: per-shadow zeroed iff GFP_ZERO | `Kmsan::init_alloc` |
| `Kfence::handle_page_fault` post: per-fence-fault: bug reported | `Kfence::handle_page_fault` |

### Layer 4: Verus/Creusot functional

`Per-bug class detected: UAF / OOB / uninit-load / double-free` semantic equivalence: per-Documentation/dev-tools/{kasan,kmsan,kfence}.rst.

## Hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening.)

Sanitizer-specific reinforcement:

- **Per-shadow region read-write only by sanitizer-runtime** — defense against per-userspace shadow-tamper.
- **Per-quarantine per-CPU local** — defense against per-CPU contention.
- **Per-KASAN-mode strict** — defense against per-mode-mix corruption.
- **Per-KMSAN runtime-checks before report** — defense against per-false-positive flood.
- **Per-KFENCE pool isolated** — defense against per-pool corruption affecting kernel.
- **Per-arch shadow region pre-populated** — defense against per-shadow-fault.
- **Per-stackdepot dedup per-stack-trace** — defense against per-stack-trace memory-balloon.
- **Per-non-sanitizer-build no-instrumentation** — defense against per-prod-overhead.
- **Per-KFENCE_SAMPLE_INTERVAL bounded** — defense against per-sample storm.
- **Per-bug-report rate-limited** — defense against per-bug log-flood.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- mm/00-overview (Tier-2)
- KCSAN (Kernel Concurrency Sanitizer; covered separately if needed)
- UBSAN (Undefined-Behavior Sanitizer; covered separately if needed)
- Per-arch sanitizer hooks (covered in `arch/x86/00-overview.md` if added)
- Implementation code
