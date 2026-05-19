# Tier-3: mm/kasan/common.c — KASAN common (Generic / SW-Tags / HW-Tags)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: mm/00-overview.md
upstream-paths:
  - mm/kasan/common.c (~630 lines)
  - mm/kasan/kasan.h
  - mm/kasan/generic.c
  - mm/kasan/sw_tags.c
  - mm/kasan/hw_tags.c
  - mm/kasan/quarantine.c
  - mm/kasan/report.c
  - include/linux/kasan.h
  - include/linux/kasan-enabled.h
  - Documentation/dev-tools/kasan.rst
-->

## Summary

**KASAN (Kernel Address Sanitizer)** detects out-of-bounds (OOB) and use-after-free (UAF) bugs in kernel heap, slab, page, vmalloc, and stack memory. Three modes share the **common** plumbing in `mm/kasan/common.c`: (1) **Generic** uses *shadow memory* — one shadow byte per 8-byte granule encodes how many bytes of the granule are addressable (0 = all valid, 1–7 = partial, negative = poison reason); checks happen on every load/store via compiler-inserted `__asan_load*`/`__asan_store*` calls. (2) **SW-Tags** stores a 4-bit tag in the top byte of every pointer; the matching 4-bit tag is stored in the shadow per 16-byte granule; pointer dereference verifies the match. (3) **HW-Tags** (arm64 MTE) uses hardware tag bits in pointers and per-16-byte memory tags, with the CPU enforcing the match. Per-object metadata: `struct kasan_alloc_meta` (alloc stack), `struct kasan_free_meta` (free stack), reached via stackdepot handles. Per-`__kasan_unpoison_pages` opens an order-N page run, per-`__kasan_poison_pages` closes it with `KASAN_PAGE_FREE`. Per-`__kasan_poison_slab` paints the whole slab `KASAN_SLAB_REDZONE` at slab boot. Per-`__kasan_slab_alloc` unpoisons one object; per-`__kasan_kmalloc` additionally poisons the tail redzone with `KASAN_SLAB_REDZONE`. Per-`__kasan_slab_free` / `__kasan_slab_pre_free` verifies the free is legal (in-cache + accessible) and enqueues the object into quarantine via `kasan_quarantine_put`. Per-`__kasan_kmalloc_large` poisons large-kmalloc tail redzone with `KASAN_PAGE_REDZONE`. Mempool, vmalloc, vrealloc, and krealloc helpers piggyback on the same poison/unpoison primitives. On any violation: `kasan_report(addr, size, write, ip)` dispatches to per-mode reporter with the alloc/free stacks. Critical for: catching slab OOB, slab UAF, kmalloc redzone overruns, page-alloc UAF, vmalloc UAF, and double-free.

This Tier-3 covers `mm/kasan/common.c` (~630 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `kasan_flag_enabled` (static key) | per-runtime KASAN gate (deferred / HW-Tags) | `KasanCommon::FLAG_ENABLED` |
| `struct kasan_track` | per-stack pid/cpu/ts/handle | `KasanTrack` |
| `struct kasan_alloc_meta` / `kasan_free_meta` | per-object metadata | `KasanAllocMeta` / `KasanFreeMeta` |
| `kasan_addr_to_slab()` | per-addr → slab lookup | `KasanCommon::addr_to_slab` |
| `kasan_save_stack()` | per-stackdepot save | `KasanCommon::save_stack` |
| `kasan_set_track()` | per-track set | `KasanCommon::set_track` |
| `kasan_save_track()` | per-track save (stack + pid) | `KasanCommon::save_track` |
| `kasan_enable_current()` / `_disable_current()` | per-task kasan_depth toggle | `KasanCommon::enable_current` / `disable_current` |
| `__kasan_unpoison_range()` | per-range unpoison | `KasanCommon::unpoison_range` |
| `kasan_unpoison_task_stack()` | per-task-stack unpoison | `KasanCommon::unpoison_task_stack` |
| `kasan_unpoison_task_stack_below()` | per-watermark stack unpoison | `KasanCommon::unpoison_task_stack_below` |
| `__kasan_unpoison_pages()` | per-page-alloc unpoison | `KasanCommon::unpoison_pages` |
| `__kasan_poison_pages()` | per-page-free poison | `KasanCommon::poison_pages` |
| `__kasan_poison_slab()` | per-new-slab poison | `KasanCommon::poison_slab` |
| `__kasan_unpoison_new_object()` | per-ctor unpoison | `KasanCommon::unpoison_new_object` |
| `__kasan_poison_new_object()` | per-ctor poison | `KasanCommon::poison_new_object` |
| `assign_tag()` | per-tag assignment (generic = 0xff) | `KasanCommon::assign_tag` |
| `__kasan_init_slab_obj()` | per-slab-create object init | `KasanCommon::init_slab_obj` |
| `check_slab_allocation()` | per-free sanity (in-cache + byte_accessible) | `KasanCommon::check_slab_allocation` |
| `poison_slab_object()` | per-free poison `KASAN_SLAB_FREE` | `KasanCommon::poison_slab_object` |
| `__kasan_slab_pre_free()` | per-RCU-pre-free check | `KasanCommon::slab_pre_free` |
| `__kasan_slab_free()` | per-free poison + quarantine | `KasanCommon::slab_free` |
| `check_page_allocation()` | per-page-free sanity | `KasanCommon::check_page_allocation` |
| `__kasan_kfree_large()` | per-large-kfree check | `KasanCommon::kfree_large` |
| `unpoison_slab_object()` | per-alloc unpoison | `KasanCommon::unpoison_slab_object` |
| `__kasan_slab_alloc()` | per-alloc unpoison + tag | `KasanCommon::slab_alloc` |
| `poison_kmalloc_redzone()` | per-kmalloc tail poison `KASAN_SLAB_REDZONE` | `KasanCommon::poison_kmalloc_redzone` |
| `__kasan_kmalloc()` | per-kmalloc tail poison | `KasanCommon::kmalloc` |
| `poison_kmalloc_large_redzone()` | per-large-kmalloc tail poison `KASAN_PAGE_REDZONE` | `KasanCommon::poison_kmalloc_large_redzone` |
| `__kasan_kmalloc_large()` | per-large-kmalloc tail poison | `KasanCommon::kmalloc_large` |
| `__kasan_krealloc()` | per-krealloc unpoison + redzone | `KasanCommon::krealloc` |
| `__kasan_mempool_poison_pages()` | per-mempool page-poison | `KasanCommon::mempool_poison_pages` |
| `__kasan_mempool_unpoison_pages()` | per-mempool page-unpoison | `KasanCommon::mempool_unpoison_pages` |
| `__kasan_mempool_poison_object()` | per-mempool obj poison | `KasanCommon::mempool_poison_object` |
| `__kasan_mempool_unpoison_object()` | per-mempool obj unpoison | `KasanCommon::mempool_unpoison_object` |
| `__kasan_check_byte()` | per-byte accessible probe | `KasanCommon::check_byte` |
| `__kasan_unpoison_vmap_areas()` | per-vmap unpoison | `KasanCommon::unpoison_vmap_areas` |
| `__kasan_vrealloc()` | per-vrealloc range adjust | `KasanCommon::vrealloc` |
| `kasan_report()` / `kasan_report_invalid_free()` | per-report dispatch | external (`mm/kasan/report.c`) |
| `kasan_unpoison()` / `kasan_poison()` | per-mode low-level shadow paint | per-mode (`generic.c` / `sw_tags.c` / `hw_tags.c`) |
| `kasan_quarantine_put()` / `kasan_quarantine_reduce()` | per-quarantine enqueue / drain | external (`mm/kasan/quarantine.c`) |
| `kasan_byte_accessible()` | per-byte probe (mode-specific) | per-mode |

## Compatibility contract

REQ-1: KASAN-runtime gate:
- `DEFINE_STATIC_KEY_FALSE(kasan_flag_enabled)`: present iff `CONFIG_ARCH_DEFER_KASAN ∨ CONFIG_KASAN_HW_TAGS`.
- Generic and SW-Tags are always-on once compiled in; HW-Tags / deferred toggles at boot.
- `EXPORT_SYMBOL_GPL(kasan_flag_enabled)` for modules and call-sites in fast paths.

REQ-2: struct kasan_track:
- pid: per-task pid at save.
- stack: depot_stack_handle_t.
- cpu (if CONFIG_KASAN_EXTRA_INFO): raw_smp_processor_id().
- timestamp (if CONFIG_KASAN_EXTRA_INFO): local_clock() >> 9.

REQ-3: kasan_addr_to_slab(addr):
- if !virt_addr_valid(addr): return NULL.
- return virt_to_slab(addr).

REQ-4: kasan_save_stack(flags, depot_flags):
- entries[KASAN_STACK_DEPTH]; nr = stack_trace_save(entries, ARRAY_SIZE, 0).
- return stack_depot_save_flags(entries, nr, flags, depot_flags).

REQ-5: kasan_set_track(track, stack):
- track.pid = current.pid; track.stack = stack.
- under CONFIG_KASAN_EXTRA_INFO: track.cpu = raw_smp_processor_id; track.timestamp = local_clock >> 9.

REQ-6: kasan_save_track(track, flags):
- stack = kasan_save_stack(flags, STACK_DEPOT_FLAG_CAN_ALLOC).
- kasan_set_track(track, stack).

REQ-7: kasan_enable_current / disable_current (Generic ∨ SW-Tags only):
- current.kasan_depth++ / --.
- Used by sites that must elide KASAN checks (e.g., early boot, KASAN itself).

REQ-8: __kasan_unpoison_range(address, size):
- if is_kfence_address(address): return /* KFENCE owns these pages */.
- kasan_unpoison(address, size, init=false) /* mode-specific shadow paint */.

REQ-9: kasan_unpoison_task_stack(task) (under CONFIG_KASAN_STACK):
- base = task_stack_page(task).
- kasan_unpoison(base, THREAD_SIZE, false).

REQ-10: kasan_unpoison_task_stack_below(watermark):
- base = (void *)((unsigned long)watermark & ~(THREAD_SIZE - 1)).
- kasan_unpoison(base, watermark - base, false).

REQ-11: __kasan_unpoison_pages(page, order, init) -> bool:
- if PageHighMem(page): return false.
- if !kasan_sample_page_alloc(order): return false /* sampled-out */.
- tag = kasan_random_tag().
- kasan_unpoison(set_tag(page_address(page), tag), PAGE_SIZE << order, init).
- for i in 0..(1 << order): page_kasan_tag_set(page + i, tag).
- return true.

REQ-12: __kasan_poison_pages(page, order, init):
- if !PageHighMem(page): kasan_poison(page_address(page), PAGE_SIZE << order, KASAN_PAGE_FREE, init).

REQ-13: __kasan_poison_slab(slab):
- page = slab_page(slab).
- for i in 0..compound_nr(page): page_kasan_tag_reset(page + i).
- kasan_poison(page_address(page), page_size(page), KASAN_SLAB_REDZONE, false).

REQ-14: __kasan_unpoison_new_object(cache, object):
- kasan_unpoison(object, cache.object_size, false).

REQ-15: __kasan_poison_new_object(cache, object):
- kasan_poison(object, round_up(cache.object_size, KASAN_GRANULE_SIZE), KASAN_SLAB_REDZONE, false).

REQ-16: assign_tag(cache, object, init) -> u8:
- IS_ENABLED(CONFIG_KASAN_GENERIC) ⇒ return 0xff /* tag ignored */.
- if !cache.ctor ∧ !(cache.flags & SLAB_TYPESAFE_BY_RCU):
  - return init ? KASAN_TAG_KERNEL : kasan_random_tag().
- else /* ctor or RCU */: return init ? kasan_random_tag() : get_tag(object).

REQ-17: __kasan_init_slab_obj(cache, object) -> void *:
- if kasan_requires_meta(): kasan_init_object_meta(cache, object).
- return set_tag(object, assign_tag(cache, object, true)).

REQ-18: check_slab_allocation(cache, object, ip) -> bool /* true = invalid */:
- tagged_object = object; object = kasan_reset_tag(object).
- if nearest_obj(cache, virt_to_slab(object), object) != object:
  - kasan_report_invalid_free(tagged_object, ip, KASAN_REPORT_INVALID_FREE); return true.
- if !kasan_byte_accessible(tagged_object):
  - kasan_report_invalid_free(tagged_object, ip, KASAN_REPORT_DOUBLE_FREE); return true.
- return false.

REQ-19: poison_slab_object(cache, object, init):
- tagged = object; object = kasan_reset_tag.
- kasan_poison(object, round_up(cache.object_size, KASAN_GRANULE_SIZE), KASAN_SLAB_FREE, init).
- if kasan_stack_collection_enabled(): kasan_save_free_info(cache, tagged).

REQ-20: __kasan_slab_pre_free(cache, object, ip):
- if is_kfence_address(object): return false.
- return check_slab_allocation(cache, object, ip).

REQ-21: __kasan_slab_free(cache, object, init, still_accessible, no_quarantine) -> bool:
- if is_kfence_address(object): return false.
- if still_accessible: return false /* RCU-grace required */.
- poison_slab_object(cache, object, init).
- if no_quarantine: return false.
- if kasan_quarantine_put(cache, object): return true /* slab does not free */.
- return false.

REQ-22: check_page_allocation(ptr, ip):
- if ptr != page_address(virt_to_head_page(ptr)): kasan_report_invalid_free; return true.
- if !kasan_byte_accessible(ptr): kasan_report_invalid_free(... DOUBLE_FREE); return true.
- return false.

REQ-23: __kasan_kfree_large(ptr, ip):
- check_page_allocation(ptr, ip) /* poison done later by kasan_poison_pages */.

REQ-24: unpoison_slab_object(cache, object, flags, init):
- kasan_unpoison(object, cache.object_size, init).
- if kasan_stack_collection_enabled() ∧ !is_kmalloc_cache(cache): kasan_save_alloc_info(cache, object, flags).

REQ-25: __kasan_slab_alloc(cache, object, flags, init):
- if gfpflags_allow_blocking(flags): kasan_quarantine_reduce().
- if object == NULL: return NULL.
- if is_kfence_address(object): return object.
- tag = assign_tag(cache, object, false); tagged = set_tag(object, tag).
- unpoison_slab_object(cache, tagged, flags, init).
- return tagged.

REQ-26: poison_kmalloc_redzone(cache, object, size, flags):
- if CONFIG_KASAN_GENERIC: kasan_poison_last_granule(object, size).
- redzone_start = round_up(object + size, KASAN_GRANULE_SIZE).
- redzone_end = round_up(object + cache.object_size, KASAN_GRANULE_SIZE).
- kasan_poison(redzone_start, end - start, KASAN_SLAB_REDZONE, false).
- if kasan_stack_collection_enabled() ∧ is_kmalloc_cache(cache): kasan_save_alloc_info(cache, object, flags).

REQ-27: __kasan_kmalloc(cache, object, size, flags):
- if gfpflags_allow_blocking(flags): kasan_quarantine_reduce.
- if object == NULL: return NULL.
- if is_kfence_address(object): return object.
- poison_kmalloc_redzone(cache, object, size, flags).
- return object /* tag preserved from kasan_slab_alloc */.
- EXPORT_SYMBOL.

REQ-28: poison_kmalloc_large_redzone(ptr, size, flags):
- if CONFIG_KASAN_GENERIC: kasan_poison_last_granule(ptr, size).
- redzone_start = round_up(ptr + size, KASAN_GRANULE_SIZE).
- redzone_end = ptr + page_size(virt_to_page(ptr)).
- kasan_poison(redzone_start, end - start, KASAN_PAGE_REDZONE, false).

REQ-29: __kasan_kmalloc_large(ptr, size, flags):
- if gfpflags_allow_blocking(flags): kasan_quarantine_reduce.
- if ptr == NULL: return NULL.
- poison_kmalloc_large_redzone(ptr, size, flags).
- return ptr /* tag from alloc_pages */.

REQ-30: __kasan_krealloc(object, size, flags):
- if gfpflags_allow_blocking(flags): kasan_quarantine_reduce.
- if object == ZERO_SIZE_PTR ∨ is_kfence_address(object): return object.
- kasan_unpoison(object, size, false) /* data part */.
- slab = virt_to_slab(object).
- if !slab: poison_kmalloc_large_redzone(object, size, flags).
- else: poison_kmalloc_redzone(slab.slab_cache, object, size, flags).

REQ-31: __kasan_mempool_poison_pages(page, order, ip) -> bool:
- if PageHighMem(page): return true.
- if !CONFIG_KASAN_GENERIC ∧ page_kasan_tag(page) == KASAN_TAG_KERNEL: return true /* sampled-out */.
- ptr = page_address(page).
- if check_page_allocation(ptr, ip): return false.
- kasan_poison(ptr, PAGE_SIZE << order, KASAN_PAGE_FREE, false).
- return true.

REQ-32: __kasan_mempool_unpoison_pages(page, order, ip):
- __kasan_unpoison_pages(page, order, false).

REQ-33: __kasan_mempool_poison_object(ptr, ip):
- page = virt_to_page(ptr).
- if PageLargeKmalloc(page):
  - if check_page_allocation(ptr, ip): return false.
  - kasan_poison(ptr, page_size(page), KASAN_PAGE_FREE, false); return true.
- if is_kfence_address(ptr): return true.
- slab = page_slab(page).
- if check_slab_allocation(slab.slab_cache, ptr, ip): return false.
- poison_slab_object(slab.slab_cache, ptr, false).
- return true.

REQ-34: __kasan_mempool_unpoison_object(ptr, size, ip):
- slab = virt_to_slab(ptr); flags = 0.
- if !slab: kasan_unpoison(ptr, size, false); poison_kmalloc_large_redzone(ptr, size, flags); return.
- if is_kfence_address(ptr): return.
- unpoison_slab_object(slab.slab_cache, ptr, flags, false).
- if is_kmalloc_cache(slab.slab_cache): poison_kmalloc_redzone(slab.slab_cache, ptr, size, flags).

REQ-35: __kasan_check_byte(address, ip):
- if !kasan_byte_accessible(address): kasan_report(address, 1, write=false, ip); return false.
- return true.

REQ-36: __kasan_unpoison_vmap_areas(vms[], nr_vms, flags) (under CONFIG_KASAN_VMALLOC):
- WARN_ON_ONCE(flags & KASAN_VMALLOC_KEEP_TAG); return on warn.
- vms[0].addr = __kasan_unpoison_vmalloc(vms[0].addr, vms[0].size, flags); tag = get_tag(vms[0].addr).
- for area in 1..nr_vms: vms[area].addr = __kasan_unpoison_vmalloc(set_tag(vms[area].addr, tag), vms[area].size, flags | KEEP_TAG).

REQ-37: __kasan_vrealloc(addr, old_size, new_size):
- if new_size < old_size:
  - kasan_poison_last_granule(addr, new_size).
  - round both sizes up; if new_round < old_round: __kasan_poison_vmalloc(addr + new_size, old_size - new_size).
- else if new_size > old_size:
  - old_size = round_down(old_size, KASAN_GRANULE_SIZE).
  - __kasan_unpoison_vmalloc(addr + old_size, new_size - old_size, NORMAL | VM_ALLOC | KEEP_TAG).

REQ-38: Shadow-byte semantics (Generic):
- 1 shadow byte covers 8 user bytes (KASAN_SHADOW_SCALE_SHIFT == 3).
- 0 ⇒ all 8 bytes addressable; 1..7 ⇒ first N bytes addressable; negative codes ⇒ KASAN_*_REDZONE / _FREE / _PAGE_FREE.

REQ-39: Tag granule (SW-Tags / HW-Tags):
- 1 tag covers 16 bytes (KASAN_GRANULE_SIZE == 16 for tag modes).
- Pointer top-byte tag must match memory tag on every access.

REQ-40: Per-quarantine lifecycle:
- On kfree: kasan_quarantine_put puts object on per-CPU quarantine list; slab keeps object reserved.
- kasan_quarantine_reduce drains old entries back to slab under memory pressure / blocking-alloc.

## Acceptance Criteria

- [ ] AC-1: __kasan_unpoison_range with kfence address: no shadow paint.
- [ ] AC-2: __kasan_unpoison_pages on HighMem page: returns false, no shadow paint.
- [ ] AC-3: __kasan_poison_slab paints whole slab with KASAN_SLAB_REDZONE.
- [ ] AC-4: __kasan_slab_alloc returns tagged pointer; non-Generic tag varies per alloc unless ctor/RCU.
- [ ] AC-5: __kasan_kmalloc poisons [object+size, object+cache.object_size) tail redzone with KASAN_SLAB_REDZONE.
- [ ] AC-6: __kasan_kmalloc_large poisons [ptr+size, ptr+page_size) with KASAN_PAGE_REDZONE.
- [ ] AC-7: check_slab_allocation on wrong-object pointer: kasan_report_invalid_free, returns true.
- [ ] AC-8: check_slab_allocation on already-poisoned object: kasan_report_invalid_free(DOUBLE_FREE).
- [ ] AC-9: __kasan_slab_free with still_accessible=true: returns false, skips quarantine.
- [ ] AC-10: __kasan_slab_free with no_quarantine=true: returns false (slab frees immediately).
- [ ] AC-11: __kasan_slab_free in default path: poisons + enqueues; returns true ⟺ kasan_quarantine_put accepted.
- [ ] AC-12: __kasan_krealloc on ZERO_SIZE_PTR: returns object unmodified.
- [ ] AC-13: __kasan_mempool_poison_object: PageLargeKmalloc ⇒ KASAN_PAGE_FREE; slab obj ⇒ poison_slab_object.
- [ ] AC-14: __kasan_check_byte returns false + kasan_report on inaccessible byte.
- [ ] AC-15: kasan_save_track captures pid + stackdepot handle (and cpu/ts when CONFIG_KASAN_EXTRA_INFO).

## Architecture

```
struct KasanTrack {
  pid: i32,
  stack: depot_stack_handle_t,
  cpu: u32,                 // CONFIG_KASAN_EXTRA_INFO
  timestamp: u64,           // CONFIG_KASAN_EXTRA_INFO
}

struct KasanAllocMeta { alloc_track: KasanTrack, .. }
struct KasanFreeMeta { free_track: KasanTrack, .. }

enum KasanMode { Generic, SwTags, HwTags }

const KASAN_GRANULE_SIZE_GENERIC: usize = 8;
const KASAN_GRANULE_SIZE_TAGS: usize = 16;

// Shadow byte codes (Generic):
const KASAN_FREE_PAGE: i8           = -0xFF;
const KASAN_PAGE_REDZONE: i8        = -0xFD;
const KASAN_KMALLOC_REDZONE: i8     = -0xFC;
const KASAN_SLAB_REDZONE: i8        = -0xFB;
const KASAN_SLAB_FREE: i8           = -0xFA;
// ... per mm/kasan/kasan.h
```

`KasanCommon::save_track(track, flags)`:
1. stack = save_stack(flags, STACK_DEPOT_FLAG_CAN_ALLOC).
2. track.pid = current.pid; track.stack = stack.
3. If CONFIG_KASAN_EXTRA_INFO: track.cpu = raw_smp_processor_id(); track.timestamp = local_clock() >> 9.

`KasanCommon::unpoison_pages(page, order, init) -> bool`:
1. If PageHighMem(page): return false.
2. If !sample_page_alloc(order): return false.
3. tag = random_tag().
4. unpoison(set_tag(page_address(page), tag), PAGE_SIZE << order, init).
5. For i in 0..(1<<order): page_kasan_tag_set(page + i, tag).
6. Return true.

`KasanCommon::poison_pages(page, order, init)`:
1. If !PageHighMem(page): poison(page_address(page), PAGE_SIZE << order, KASAN_PAGE_FREE, init).

`KasanCommon::poison_slab(slab)`:
1. page = slab_page(slab).
2. For i in 0..compound_nr(page): page_kasan_tag_reset(page + i).
3. poison(page_address(page), page_size(page), KASAN_SLAB_REDZONE, false).

`KasanCommon::init_slab_obj(cache, object) -> *void`:
1. If kasan_requires_meta(): kasan_init_object_meta(cache, object).
2. Return set_tag(object, assign_tag(cache, object, init=true)).

`KasanCommon::assign_tag(cache, object, init) -> u8`:
1. If CONFIG_KASAN_GENERIC: return 0xff.
2. If !cache.ctor ∧ !(cache.flags & SLAB_TYPESAFE_BY_RCU):
   - Return if init { KASAN_TAG_KERNEL } else { random_tag() }.
3. Else: return if init { random_tag() } else { get_tag(object) }.

`KasanCommon::check_slab_allocation(cache, object, ip) -> bool`:
1. tagged = object; object = reset_tag(object).
2. If nearest_obj(cache, virt_to_slab(object), object) != object:
   - report_invalid_free(tagged, ip, INVALID_FREE); return true.
3. If !byte_accessible(tagged):
   - report_invalid_free(tagged, ip, DOUBLE_FREE); return true.
4. Return false.

`KasanCommon::poison_slab_object(cache, object, init)`:
1. tagged = object; object = reset_tag(object).
2. poison(object, round_up(cache.object_size, KASAN_GRANULE_SIZE), KASAN_SLAB_FREE, init).
3. If stack_collection_enabled(): save_free_info(cache, tagged).

`KasanCommon::slab_pre_free(cache, object, ip) -> bool`:
1. If is_kfence_address(object): return false.
2. Return check_slab_allocation(cache, object, ip).

`KasanCommon::slab_free(cache, object, init, still_accessible, no_quarantine) -> bool`:
1. If is_kfence_address(object): return false.
2. If still_accessible: return false.
3. poison_slab_object(cache, object, init).
4. If no_quarantine: return false.
5. If quarantine_put(cache, object): return true.
6. Return false.

`KasanCommon::slab_alloc(cache, object, flags, init) -> *void`:
1. If gfpflags_allow_blocking(flags): quarantine_reduce().
2. If object == NULL: return NULL.
3. If is_kfence_address(object): return object.
4. tag = assign_tag(cache, object, init=false).
5. tagged = set_tag(object, tag).
6. unpoison_slab_object(cache, tagged, flags, init).
7. Return tagged.

`KasanCommon::poison_kmalloc_redzone(cache, object, size, flags)`:
1. If CONFIG_KASAN_GENERIC: poison_last_granule(object, size).
2. redzone_start = round_up(object + size, KASAN_GRANULE_SIZE).
3. redzone_end = round_up(object + cache.object_size, KASAN_GRANULE_SIZE).
4. poison(redzone_start, end - start, KASAN_SLAB_REDZONE, false).
5. If stack_collection_enabled() ∧ is_kmalloc_cache(cache): save_alloc_info.

`KasanCommon::kmalloc(cache, object, size, flags) -> *void`:
1. If gfpflags_allow_blocking(flags): quarantine_reduce.
2. If object == NULL: return NULL.
3. If is_kfence_address(object): return object.
4. poison_kmalloc_redzone(cache, object, size, flags).
5. Return object.

`KasanCommon::poison_kmalloc_large_redzone(ptr, size, flags)`:
1. If CONFIG_KASAN_GENERIC: poison_last_granule(ptr, size).
2. redzone_start = round_up(ptr + size, KASAN_GRANULE_SIZE).
3. redzone_end = ptr + page_size(virt_to_page(ptr)).
4. poison(redzone_start, end - start, KASAN_PAGE_REDZONE, false).

`KasanCommon::kmalloc_large(ptr, size, flags) -> *void`:
1. If gfpflags_allow_blocking(flags): quarantine_reduce.
2. If ptr == NULL: return NULL.
3. poison_kmalloc_large_redzone(ptr, size, flags).
4. Return ptr.

`KasanCommon::krealloc(object, size, flags) -> *void`:
1. If gfpflags_allow_blocking(flags): quarantine_reduce.
2. If object == ZERO_SIZE_PTR ∨ is_kfence_address(object): return object.
3. unpoison(object, size, false).
4. slab = virt_to_slab(object).
5. If !slab: poison_kmalloc_large_redzone(object, size, flags).
6. Else: poison_kmalloc_redzone(slab.slab_cache, object, size, flags).
7. Return object.

`KasanCommon::mempool_poison_pages(page, order, ip) -> bool`:
1. If PageHighMem(page): return true.
2. If !CONFIG_KASAN_GENERIC ∧ page_kasan_tag(page) == KASAN_TAG_KERNEL: return true.
3. ptr = page_address(page).
4. If check_page_allocation(ptr, ip): return false.
5. poison(ptr, PAGE_SIZE << order, KASAN_PAGE_FREE, false).
6. Return true.

`KasanCommon::mempool_poison_object(ptr, ip) -> bool`:
1. page = virt_to_page(ptr).
2. If PageLargeKmalloc(page):
   - If check_page_allocation(ptr, ip): return false.
   - poison(ptr, page_size(page), KASAN_PAGE_FREE, false); return true.
3. If is_kfence_address(ptr): return true.
4. slab = page_slab(page).
5. If check_slab_allocation(slab.slab_cache, ptr, ip): return false.
6. poison_slab_object(slab.slab_cache, ptr, false); return true.

`KasanCommon::mempool_unpoison_object(ptr, size, ip)`:
1. slab = virt_to_slab(ptr); flags = 0.
2. If !slab: unpoison(ptr, size, false); poison_kmalloc_large_redzone(ptr, size, flags); return.
3. If is_kfence_address(ptr): return.
4. unpoison_slab_object(slab.slab_cache, ptr, flags, false).
5. If is_kmalloc_cache(slab.slab_cache): poison_kmalloc_redzone(slab.slab_cache, ptr, size, flags).

`KasanCommon::check_byte(address, ip) -> bool`:
1. If !byte_accessible(address): report(address, 1, write=false, ip); return false.
2. Return true.

`KasanCommon::unpoison_vmap_areas(vms[], nr_vms, flags)`:
1. WARN_ON_ONCE(flags & KASAN_VMALLOC_KEEP_TAG) ⇒ return.
2. vms[0].addr = unpoison_vmalloc(vms[0].addr, vms[0].size, flags); tag = get_tag(vms[0].addr).
3. For area in 1..nr_vms: vms[area].addr = unpoison_vmalloc(set_tag(vms[area].addr, tag), vms[area].size, flags | KEEP_TAG).

`KasanCommon::vrealloc(addr, old_size, new_size)`:
1. If new_size < old_size:
   - poison_last_granule(addr, new_size).
   - new_round = round_up(new_size, GRANULE); old_round = round_up(old_size, GRANULE).
   - If new_round < old_round: poison_vmalloc(addr + new_size, old_size - new_size).
2. Else if new_size > old_size:
   - old_round = round_down(old_size, GRANULE).
   - unpoison_vmalloc(addr + old_round, new_size - old_round, NORMAL | VM_ALLOC | KEEP_TAG).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `kfence_skips_shadow` | INVARIANT | per-{unpoison,poison}_* : is_kfence_address(p) ⟹ no shadow paint. |
| `highmem_skipped` | INVARIANT | per-{poison,unpoison}_pages : PageHighMem(page) ⟹ no shadow paint. |
| `granule_aligned_poison` | INVARIANT | per-poison_kmalloc_redzone : redzone_start, redzone_end are KASAN_GRANULE_SIZE-aligned. |
| `assign_tag_generic_constant` | INVARIANT | per-assign_tag : CONFIG_KASAN_GENERIC ⟹ returns 0xff. |
| `invalid_free_reports_once` | INVARIANT | per-check_slab_allocation : wrong-object pointer ⟹ kasan_report_invalid_free exactly once. |
| `double_free_detected` | INVARIANT | per-check_slab_allocation : already-poisoned object ⟹ DOUBLE_FREE report. |
| `quarantine_only_on_success` | INVARIANT | per-slab_free : quarantine_put true ⟺ return true. |
| `krealloc_zero_size_passthrough` | INVARIANT | per-krealloc : object == ZERO_SIZE_PTR ⟹ return object unmodified. |
| `mempool_large_kmalloc_poison_page_free` | INVARIANT | per-mempool_poison_object : PageLargeKmalloc ⟹ KASAN_PAGE_FREE. |
| `track_pid_set` | INVARIANT | per-save_track : track.pid == current.pid post. |

### Layer 2: TLA+

`mm/kasan-common.tla`:
- Per-object lifecycle: NewSlab → InitSlabObj → SlabAlloc → Kmalloc → Use → SlabPreFree → SlabFree → Quarantine → Reduce → Free.
- Properties:
  - `safety_no_use_after_free` — Free ⟹ subsequent Use raises kasan_report.
  - `safety_no_oob_via_redzone` — kmalloc(size) leaves [size, cache.object_size) poisoned with SLAB_REDZONE.
  - `safety_no_double_free` — SlabFree on already-poisoned object ⟹ report(DOUBLE_FREE).
  - `safety_invalid_free_caught` — Free on non-object pointer ⟹ report(INVALID_FREE).
  - `safety_quarantine_holds_until_reduce` — object enqueued via quarantine_put cannot be allocated until quarantine_reduce drains it.
  - `liveness_quarantine_drained_under_pressure` — eventual quarantine_reduce after gfpflags_allow_blocking allocs.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `KasanCommon::slab_alloc` post: returned pointer has assign_tag tag | `KasanCommon::slab_alloc` |
| `KasanCommon::kmalloc` post: object’s [size..object_size) shadow == KASAN_SLAB_REDZONE | `KasanCommon::kmalloc` |
| `KasanCommon::kmalloc_large` post: [ptr+size..ptr+page_size(virt_to_page(ptr))) == KASAN_PAGE_REDZONE | `KasanCommon::kmalloc_large` |
| `KasanCommon::slab_free` post: object_size shadow == KASAN_SLAB_FREE iff !still_accessible | `KasanCommon::slab_free` |
| `KasanCommon::krealloc` post: data [object..object+size) unpoisoned; tail re-poisoned | `KasanCommon::krealloc` |
| `KasanCommon::mempool_poison_object` post: large-kmalloc ⟹ KASAN_PAGE_FREE; slab ⟹ KASAN_SLAB_FREE | `KasanCommon::mempool_poison_object` |
| `KasanCommon::vrealloc` post: shrink ⟹ tail poisoned; grow ⟹ new tail unpoisoned with KEEP_TAG | `KasanCommon::vrealloc` |
| `KasanCommon::check_byte` post: returns false ⟺ kasan_report fired | `KasanCommon::check_byte` |

### Layer 4: Verus/Creusot functional

`Per-object: init_slab_obj → slab_alloc → kmalloc (redzone) → access → slab_pre_free → slab_free (poison + quarantine) → eventual quarantine_reduce → object reusable` semantic equivalence: per-Documentation/dev-tools/kasan.rst § "Implementation details" + per-mm/kasan/kasan.h shadow encoding table.

`Per-mempool: mempool_unpoison_object → use → mempool_poison_object` semantic equivalence: per-mm/mempool.c interface.

`Per-vmalloc: unpoison_vmap_areas (with shared tag across areas) → vrealloc (grow/shrink) → __kasan_poison_vmalloc on teardown` semantic equivalence: per-mm/vmalloc.c interface.

## Hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening.)

KASAN reinforcement:

- **Per-kfence_address bypass guards** — defense against per-double-bookkeeping when KFENCE owns the page.
- **Per-byte_accessible double-free check** — defense against per-quarantine-bypassing free-then-free.
- **Per-nearest_obj invalid-free check** — defense against per-arbitrary-pointer kfree (e.g., interior pointer).
- **Per-redzone strict GRANULE-aligned poison** — defense against per-OOB miss caused by misaligned redzone start/end.
- **Per-quarantine_put delay** — defense against per-immediate-reuse UAF on hot caches.
- **Per-quarantine_reduce only under gfpflags_allow_blocking** — defense against per-atomic-context deadlock.
- **Per-still_accessible bypass for SLAB_TYPESAFE_BY_RCU** — defense against per-RCU-grace-period violations.
- **Per-no_quarantine for SLUB_RCU_DEBUG path** — defense against per-bogus quarantine when slab insists on direct free.
- **Per-PageHighMem skip** — defense against per-mapping into shadow that isn’t direct-mapped.
- **Per-kasan_sample_page_alloc sampling** — defense against per-shadow-paint cost for hot page-alloc paths.
- **Per-set_tag/get_tag pointer-tag round-trip** — defense against per-tag drop in error paths.
- **Per-stackdepot stack save with STACK_DEPOT_FLAG_CAN_ALLOC** — defense against per-stack-loss on OOM (graceful degrade).
- **Per-WARN_ON_ONCE(flags & KASAN_VMALLOC_KEEP_TAG) at vmap entry** — defense against per-caller misuse forcing tag-keep at first area.
- **Per-static-key kasan_flag_enabled gating** — defense against per-cold-path overhead when HW-Tags off at boot.
- **Per-kfence + KASAN cooperation in mempool path** — defense against per-conflicting-ownership UAF reports.

## Grsecurity/PaX-style Reinforcement

Baseline grsec/PaX features inherited project-wide:

- **PAX_USERCOPY** — KASAN shadow memory acts as a runtime PAX_USERCOPY-equivalent: every kernel access (including copy_to_user/copy_from_user paths under instrumentation) is range-validated against shadow.
- **PAX_KERNEXEC** — KASAN shadow region itself mapped W^X; shadow accesses are reads-only from instrumented code, writes only by allocator hooks.
- **PAX_RANDKSTACK** — kasan_report entry-stack randomized so OOB report itself is not predictable.
- **PAX_REFCOUNT** — quarantine_batch counters and stackdepot refcounts saturate; overflow on attacker-induced quarantine churn refuses rather than wraps.
- **PAX_MEMORY_SANITIZE** — quarantine free path zero-poisons + KASAN_FREE_PAGE shadow byte; double-sanitize harmonized.
- **PAX_UDEREF** — KASAN-instrumented copy_*_user variant validates user pointer before shadow load on the kernel-side scratch.
- **PAX_RAP / kCFI** — KASAN runtime callbacks (`__kasan_check_read`, `__asan_loadN`) type-tagged; instrumentation hot-patch verified.
- **GRKERNSEC_HIDESYM** — `kasan_report`, `__kasan_check_*`, shadow base addresses absent from `/proc/kallsyms` for unpriv.
- **GRKERNSEC_DMESG** — KASAN OOB/UAF reports redacted from dmesg for non-CAP_SYSLOG; full report only visible to root.

KASAN-specific reinforcements:

- **Shadow-memory bounded kasan_report** — report path uses a fixed-size scratch buffer; OOB during report itself prevented by KASAN_REPORT_MAX_FRAMES cap.
- **Quarantine timing** — quarantine_put delay (per-cache batch) prevents immediate-reuse UAF on hot caches without exposing quarantine-bypass primitives.
- **Pointer-tag round-trip integrity** — set_tag / get_tag on HW-Tags KASAN verifies tag was preserved across allocator/caller boundary; tag-drop refuses to free.
- **stackdepot graceful degrade** — STACK_DEPOT_FLAG_CAN_ALLOC handles depot OOM without losing the report; KASAN never silently drops a UAF discovery.
- **Static-key gating** — kasan_flag_enabled keeps cold-path overhead near zero when KASAN compiled-in but boot-disabled, preventing perf-pressure to remove instrumentation in prod.

Rationale: KASAN is the kernel's runtime tripwire for memory-safety violations — under grsec doctrine it complements PAX_USERCOPY (compile-time slab whitelist) with runtime byte-granular detection. The hardening focus is making KASAN itself unbypassable and unforgeable: shadow memory writable only by allocator hooks, reports redacted to root via GRKERNSEC_DMESG so a UAF discovery doesn't leak addresses to the attacker, and refcount/quarantine arithmetic saturated to refuse rather than corrupt under adversarial pressure.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- mm/kasan/generic.c shadow-byte encoding + `__asan_*` callback bodies (covered by Generic Tier-3 if expanded).
- mm/kasan/sw_tags.c / hw_tags.c per-mode tag implementations (covered by per-mode Tier-3 if expanded).
- mm/kasan/quarantine.c quarantine FIFO + per-CPU shrink heuristics (covered by `sanitizers.md` Tier-3 extension).
- mm/kasan/report.c report formatting + per-mode dispatch (covered separately).
- mm/kasan/init.c shadow allocation at boot / hot-add (covered separately).
- mm/kfence/* KFENCE (sampled UAF detector) — orthogonal mechanism.
- Implementation code.
