# Tier-3: virt/kvm/kvm_main.c (mmu-memory-cache subset) — KVM per-vCPU MMU memory cache pre-alloc

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/x86-mmu-shadow.md
upstream-paths:
  - virt/kvm/kvm_main.c (mmu-cache subset: ~348..440)
  - include/linux/kvm_host.h (struct kvm_mmu_memory_cache)
  - arch/x86/kvm/mmu/mmu.c (per-arch caches: pte_list_desc_cache, mmu_page_header_cache, etc.)
-->

## Summary

KVM MMU memory caches pre-allocate frequently-needed objects (page-table pages, shadow-page-table headers, pte_list_desc rmap chunks, etc.) **before** entering a critical section that may not sleep (e.g., MMU spinlock held during page-table mutation). Per-vCPU has multiple caches (pte_list_desc, mmu_page_header, mmu_shadow_page, mmu_shadowed_info). At fault entry: `kvm_mmu_topup_memory_caches(vcpu)` ensures min-objects available; per-fault path then `kvm_mmu_memory_cache_alloc(mc)` returns pre-allocated obj without GFP_KERNEL allocation. Critical for: page-fault paths that hold spinlocks (cannot sleep) yet must allocate.

This Tier-3 covers the cross-arch MMU-memory-cache subset of `kvm_main.c` (~100 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct kvm_mmu_memory_cache` | per-vCPU per-cache state | `KvmMmuMemoryCache` |
| `kvm_mmu_topup_memory_cache()` | per-cache pre-alloc | `MmuCache::topup` |
| `__kvm_mmu_topup_memory_cache()` | per-cache pre-alloc with capacity | `MmuCache::__topup` |
| `kvm_mmu_memory_cache_alloc()` | per-fault grab obj | `MmuCache::alloc` |
| `mmu_memory_cache_alloc_obj()` | per-cache backing alloc fn | `MmuCache::alloc_obj` |
| `kvm_mmu_free_memory_cache()` | per-cache flush at vcpu-destroy | `MmuCache::free` |
| `kvm_mmu_memory_cache_nr_free_objects()` | per-cache count | `MmuCache::nr_free_objects` |
| `KVM_ARCH_NR_OBJS_PER_MEMORY_CACHE` | default capacity (40) | shared |
| `mc->kmem_cache` | optional per-cache backing kmem_cache_t | shared |
| `mc->init_value` | per-obj zero-fill template | shared |
| `mc->objects` | array of pre-allocated ptrs | shared |
| `mc->nobjs` | current count | shared |
| `mc->capacity` | max count | shared |
| `mc->gfp_zero` | per-alloc GFP flags | shared |
| `mc->gfp_custom` | per-alloc custom GFP flags | shared |

## Compatibility contract

REQ-1: Per-cache state:
- objects: per-cache vector of pre-allocated pointers.
- nobjs: number currently held.
- capacity: max held.
- kmem_cache: optional kmem_cache_t for sized objs (NULL → page-aligned).
- gfp_zero: GFP flags for zero-fill (typically __GFP_ZERO).
- gfp_custom: caller-specified GFP override (else GFP_KERNEL_ACCOUNT).
- init_value: per-obj initial value template (else zero).

REQ-2: kvm_mmu_topup_memory_cache(mc, min):
- Default capacity = KVM_ARCH_NR_OBJS_PER_MEMORY_CACHE (40).
- If mc.nobjs ≥ min: return 0.
- Else: alloc until mc.nobjs == capacity (or alloc-fail).

REQ-3: __kvm_mmu_topup_memory_cache(mc, capacity, min):
- If mc.objects == NULL: allocate mc.objects array of capacity ptrs.
- mc.capacity = capacity.
- While mc.nobjs < mc.capacity:
  - obj = mmu_memory_cache_alloc_obj(mc, gfp).
  - If !obj: return mc.nobjs >= min ? 0 : -ENOMEM.
  - mc.objects[mc.nobjs++] = obj.

REQ-4: mmu_memory_cache_alloc_obj(mc, gfp):
- gfp |= mc.gfp_zero ∨ mc.gfp_custom.
- If mc.kmem_cache: kmem_cache_alloc(mc.kmem_cache, gfp).
- Else: free_page(__get_free_page(gfp)).

REQ-5: kvm_mmu_memory_cache_alloc(mc):
- If mc.nobjs > 0: p = mc.objects[--mc.nobjs].
- Else (cache empty in atomic ctx): WARN; alloc with GFP_ATOMIC | __GFP_ACCOUNT.
- Return p.

REQ-6: kvm_mmu_free_memory_cache(mc):
- While mc.nobjs > 0:
  - If mc.kmem_cache: kmem_cache_free(mc.kmem_cache, mc.objects[--mc.nobjs]).
  - Else: free_page((unsigned long)mc.objects[--mc.nobjs]).
- kvfree(mc.objects); mc.objects = NULL; mc.capacity = 0.

REQ-7: Per-vCPU MMU caches (x86 example):
- pte_list_desc_cache: rmap descriptors.
- mmu_page_header_cache: per-shadow-page header.
- mmu_shadow_page_cache: per-page-table page (4KB pages).
- mmu_shadowed_info_cache: per-mirror shadow-info.

REQ-8: Per-pre-fault topup:
- kvm_mmu_load_pml4 / kvm_mmu_load_pdptrs / handle_mmu_fault path:
  - r = kvm_mmu_topup_memory_caches(vcpu, false).

REQ-9: Per-vCPU teardown:
- kvm_mmu_destroy(vcpu) calls kvm_mmu_free_memory_caches.

REQ-10: Per-init_value:
- For per-arch shadow-info: not just zero, but template (e.g., shadow_init_pte_value).

## Acceptance Criteria

- [ ] AC-1: kvm_mmu_topup_memory_cache(mc, 40): mc.nobjs == 40 post-call.
- [ ] AC-2: kvm_mmu_topup with mc.nobjs already 40: no-op.
- [ ] AC-3: kvm_mmu_memory_cache_alloc post-topup: returns object; mc.nobjs--.
- [ ] AC-4: kmem_cache backed: alloc returns from per-cache slab.
- [ ] AC-5: page backed: alloc returns 4KB page-aligned pointer.
- [ ] AC-6: kvm_mmu_free_memory_cache: per-obj returned to backing; mc.nobjs == 0.
- [ ] AC-7: Per-fault path: kvm_mmu_memory_cache_alloc within mmu_lock spinlock — no sleep.
- [ ] AC-8: kvm_mmu_topup with min=1 + alloc-failure mid-way: returns -ENOMEM if mc.nobjs < 1.
- [ ] AC-9: gfp_zero respected: alloc'd page zeroed.
- [ ] AC-10: Per-vCPU destroy: all caches freed; no leaks.

## Architecture

Per-cache state:

```
struct KvmMmuMemoryCache {
  capacity: usize,
  nobjs: usize,
  gfp_zero: GfpT,                                // typically __GFP_ZERO
  gfp_custom: GfpT,                              // caller override
  init_value: u64,                               // per-obj template
  kmem_cache: Option<&KmemCache>,                // None ⟹ page-backed
  objects: Vec<*mut u8>,
}

const KVM_ARCH_NR_OBJS_PER_MEMORY_CACHE: usize = 40;
```

Per-vCPU MMU caches (x86):

```
struct KvmVcpuArch {
  ...
  mmu_pte_list_desc_cache: KvmMmuMemoryCache,    // pte_list_desc kmem_cache
  mmu_page_header_cache: KvmMmuMemoryCache,      // mmu_page_header kmem_cache
  mmu_shadow_page_cache: KvmMmuMemoryCache,      // 4KB page (kmem_cache=NULL)
  mmu_shadowed_info_cache: KvmMmuMemoryCache,    // mirror_info kmem_cache
}
```

`MmuCache::__topup(mc, capacity, min) -> Result<()>`:
1. If !mc.objects:
   - mc.objects = kvmalloc_array(capacity, sizeof(void*), GFP_KERNEL_ACCOUNT).
2. mc.capacity = capacity.
3. while mc.nobjs < mc.capacity:
   - obj = MmuCache::alloc_obj(mc, GFP_KERNEL_ACCOUNT).
   - If !obj:
     - if mc.nobjs >= min: return Ok.
     - else: return Err(NoMem).
   - mc.objects[mc.nobjs++] = obj.

`MmuCache::topup(mc, min) -> Result<()>`:
1. Return MmuCache::__topup(mc, KVM_ARCH_NR_OBJS_PER_MEMORY_CACHE, min).

`MmuCache::alloc_obj(mc, gfp) -> Option<*mut u8>`:
1. gfp |= mc.gfp_zero | mc.gfp_custom.
2. If mc.kmem_cache: return kmem_cache_alloc(mc.kmem_cache, gfp).
3. Else: return __get_free_page(gfp) as *mut u8.

`MmuCache::alloc(mc) -> *mut u8`:
1. If mc.nobjs > 0: p = mc.objects[--mc.nobjs].
2. Else:
   - WARN_ON(!mc.nobjs).
   - p = MmuCache::alloc_obj(mc, GFP_ATOMIC | __GFP_ACCOUNT).
3. If p ∧ mc.init_value != 0:
   - if mc.kmem_cache: memset_init(p, mc.init_value, mc.kmem_cache.size).
   - else: memset_init(p, mc.init_value, PAGE_SIZE).
4. Return p.

`MmuCache::free(mc)`:
1. while mc.nobjs > 0:
   - obj = mc.objects[--mc.nobjs].
   - If mc.kmem_cache: kmem_cache_free(mc.kmem_cache, obj).
   - Else: free_page(obj as USIZE).
2. kvfree(mc.objects); mc.objects = None.
3. mc.capacity = 0.

`MmuCache::nr_free_objects(mc) -> usize`:
1. Return mc.nobjs.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `nobjs_le_capacity` | INVARIANT | mc.nobjs ≤ mc.capacity. |
| `objects_array_sized` | INVARIANT | mc.objects.len() ≥ mc.capacity. |
| `topup_post_nobjs_ge_min` | INVARIANT | post-topup-success: mc.nobjs ≥ min. |
| `alloc_decrements_nobjs` | INVARIANT | post-alloc: mc.nobjs decremented (if was > 0). |
| `free_zeroes_nobjs` | INVARIANT | post-free: mc.nobjs == 0; mc.objects == None. |

### Layer 2: TLA+

`virt/kvm/mmu_memory_cache.tla`:
- Per-vCPU cache topup + per-fault alloc + per-vCPU teardown.
- Properties:
  - `safety_no_alloc_in_atomic_below_min` — per-fault: alloc only after topup ensured min.
  - `safety_no_leak` — per-vCPU destroy: all objs freed.
  - `liveness_topup_eventually_succeeds` — per-non-OOM system: topup reaches capacity.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `MmuCache::__topup` post: mc.nobjs ≥ min OR -ENOMEM | `MmuCache::__topup` |
| `MmuCache::alloc` post: returned obj non-null; nobjs decremented | `MmuCache::alloc` |
| `MmuCache::free` post: all objs returned to backing | `MmuCache::free` |
| `MmuCache::alloc_obj` post: per-mc.kmem_cache vs page backing dispatched | `MmuCache::alloc_obj` |

### Layer 4: Verus/Creusot functional

`Per-pre-fault topup ensures min objs ⟹ per-fault alloc succeeds without GFP_KERNEL within mmu_lock spinlock` semantic equivalence: per-MMU memory cache satisfies "no-sleep-while-holding-spinlock" contract.

## Hardening

(Inherits row-1 features from `virt/kvm/x86-mmu-shadow.md` § Hardening.)

MMU-cache-specific reinforcement:

- **Per-pre-fault topup ensures min** — defense against per-fault alloc-failure stalling page-fault handler.
- **Per-WARN if alloc with empty cache + atomic ctx** — defense against silent fault-path failure.
- **Per-mc.objects array sized to capacity** — defense against per-grow OOB.
- **Per-vCPU cache flush on destroy** — defense against per-VM-destroy leak.
- **Per-init_value template applied at alloc** — defense against per-obj uninit value.
- **Per-kmem_cache vs page backing dispatched correctly** — defense against per-free-mismatch corruption.
- **Per-gfp_zero ensures no info-leak** — defense against per-page leaking host kernel data.
- **Per-cache alloc atomic in fault path** — defense against per-fault-spinlock-sleep oops.
- **Per-topup honors min before capacity** — defense against per-topup over-alloc when min sufficient.
- **Per-arch caches (pte_list_desc, page_header, etc.) zero-init backing kmem** — defense against per-arch-specific stale data.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- KVM core MMU (covered in `x86-mmu-shadow.md` Tier-3)
- KVM rmap (covered in `x86-mmu-rmap.md` Tier-3)
- KVM SPTE (covered in `x86-mmu-spte.md` Tier-3)
- KVM TDP MMU (covered in `x86-mmu-tdp.md` Tier-3)
- Linux kmem_cache (covered separately)
- Implementation code
