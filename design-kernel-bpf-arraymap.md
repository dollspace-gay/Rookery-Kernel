---
title: "Tier-3: kernel/bpf/arraymap.c — BPF arraymap (ARRAY / PERCPU_ARRAY / PROG_ARRAY / PERF_EVENT_ARRAY / CGROUP_ARRAY / ARRAY_OF_MAPS)"
tags: ["tier-3", "kernel", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

The **BPF arraymap** is the fixed-size, u32-indexed, prealloc-on-create map family. Six flavors share a single backing struct (`struct bpf_array`) but specialize their `bpf_map_ops` to cover: `BPF_MAP_TYPE_ARRAY` (flat value array, optionally mmap-able for `.bss/.data/.rodata` and BPF-light-skeleton globals), `BPF_MAP_TYPE_PERCPU_ARRAY` (one value per online CPU per slot, alignment 8), `BPF_MAP_TYPE_PROG_ARRAY` (slots hold `struct bpf_prog *`, target of `bpf_tail_call`), `BPF_MAP_TYPE_PERF_EVENT_ARRAY` (slots hold per-CPU `struct bpf_event_entry` wrapping a perf `struct file *`), `BPF_MAP_TYPE_CGROUP_ARRAY` (slots hold `struct cgroup *` for skb-cgroup membership tests), and `BPF_MAP_TYPE_ARRAY_OF_MAPS` (slots hold `struct bpf_map *` for nested-map dispatch). All flavors require `key_size == 4` (u32 slot index), all are full-size from creation (`max_entries * elem_size` bytes — or `max_entries * sizeof(void __percpu *)` plus the per-CPU value backing — allocated up front in `array_map_alloc`), all reject deletion via `array_map_delete_elem` returning `-EINVAL` for ARRAY/PERCPU_ARRAY (fd-array flavors instead reset the slot to NULL through `__fd_array_map_delete_elem`), and all clamp the index with `array->index_mask` to defeat Spectre-v1 unless `bypass_spec_v1` is set. Per-`struct bpf_array`: `map` base, `aux` (PROG_ARRAY only — `bpf_array_aux` containing `poke_progs` list and `poke_mutex` for tail-call JIT patching), `index_mask` (`roundup_pow_of_two(max_entries) - 1` unless bypass), `elem_size` (`round_up(value_size, 8)`), and the trailing flexible union `{ value[]; pptrs[]; ptrs[]; }`. Per-PROG_ARRAY: `xchg(array->ptrs + index, new_ptr)` swaps slot atomically; with `map_poke_run` (PROG_ARRAY-with-JIT), the swap is bracketed by `mutex_lock(&aux->poke_mutex)` so that `bpf_arch_poke_desc_update` rewrites each registered caller's text under poke_mutex while readers in `bpf_tail_call` see RCU-stable bytecode. Per-PERF_EVENT_ARRAY: `perf_event_get(fd)` → `perf_event_read_local` precondition check → `bpf_event_entry` allocation; `map_release` (`perf_event_fd_array_release`) walks the array on per-fd close to drop entries belonging to that map-file (unless `BPF_F_PRESERVE_ELEMS` is set). Per-CGROUP_ARRAY: `cgroup_get_from_fd(fd)` resolves and pins; `cgroup_put` after RCU. Per-ARRAY_OF_MAPS: `array_of_map_alloc` allocates a meta-template via `bpf_map_meta_alloc` and verifies inner-map compatibility via `array_map_meta_equal` (which honors `BPF_F_INNER_MAP` to allow varying `max_entries` across inner maps). Per-`BPF_F_MMAPABLE`: `array_map_alloc` uses `bpf_map_area_mmapable_alloc` (page-aligned vmalloc), `array_map_mmap` exposes the value region via `remap_vmalloc_range` with a `pgoff` offset that hides the `bpf_array` header page. Per-`BPF_F_LOCK` update: `copy_map_value_locked` provides per-element spin_lock-coordinated copy. Critical for: BPF-CO-RE global data (`.bss/.data/.rodata`), tail calls in libbpf programs (`tail_calls`), perf-event ringbuffer routing (`bpf_perf_event_output`), Cilium cgroup-membership filtering, and Cilium/Calico nested-map dispatch.

This Tier-3 covers `kernel/bpf/arraymap.c` (~1462 lines).

### Acceptance Criteria

- [ ] AC-1: array_map_alloc_check rejects key_size != 4, value_size == 0, max_entries == 0 → -EINVAL.
- [ ] AC-2: array_map_alloc_check rejects BPF_F_MMAPABLE on non-ARRAY → -EINVAL; rejects BPF_F_PRESERVE_ELEMS on non-PERF_EVENT_ARRAY → -EINVAL; rejects BPF_F_INNER_MAP on non-ARRAY → -EINVAL.
- [ ] AC-3: PERCPU_ARRAY with numa_node != NUMA_NO_NODE → -EINVAL.
- [ ] AC-4: PERCPU_ARRAY with round_up(value_size, 8) > PCPU_MIN_UNIT_SIZE → -E2BIG.
- [ ] AC-5: array_map_lookup_elem with index >= max_entries → NULL.
- [ ] AC-6: array_map_delete_elem → -EINVAL (ARRAY/PERCPU_ARRAY slots are permanent).
- [ ] AC-7: array_map_update_elem with BPF_NOEXIST → -EEXIST (all slots always exist).
- [ ] AC-8: array_map_update_elem with index >= max_entries → -E2BIG.
- [ ] AC-9: array_map_update_elem with BPF_F_LOCK + map without BPF_SPIN_LOCK field → -EINVAL.
- [ ] AC-10: bpf_array_get_next_key(NULL) → 0; with index == max_entries-1 → -ENOENT.
- [ ] AC-11: array_map_mmap on non-MMAPABLE map → -EINVAL; on MMAPABLE map maps value region (header skipped via pgoff).
- [ ] AC-12: array_map_direct_value_addr returns -ENOTSUPP for max_entries != 1; succeeds for single-slot globals.
- [ ] AC-13: PROG_ARRAY update with BPF_PROG_TYPE_EXT prog → -EINVAL; with `prog.aux.is_extended` already set → -EBUSY.
- [ ] AC-14: PROG_ARRAY update under poke_mutex: subsequent map_poke_run patches all tracked callers' direct-call sites.
- [ ] AC-15: PERF_EVENT_ARRAY release with map_file matching slot's map_file (and !PRESERVE_ELEMS) drops the slot.
- [ ] AC-16: ARRAY_OF_MAPS lookup returns READ_ONCE(*inner_map_slot); free walks slots and bpf_map_meta_free's the template.
- [ ] AC-17: array_map_meta_equal honors BPF_F_INNER_MAP — different max_entries on inner arrays accepted.
- [ ] AC-18: array_map_get_hash computes sha256 over value region and stores in map.sha (mmap-able snapshot).

### Architecture

```
struct BpfArray {
  map: BpfMap,                              // base; bypass_spec_v1 stored here
  aux: Option<*BpfArrayAux>,                // PROG_ARRAY only
  index_mask: u32,                          // roundup_pow_of_two(max_entries) - 1
  elem_size: u32,                           // round_up(value_size, 8)
  // Trailing flexible union — exactly one variant per flavor:
  //   value:  [u8; 0]                      // ARRAY (aligned 8; mmap-able skips header via pgoff)
  //   pptrs:  [*PerCpu<u8>; 0]             // PERCPU_ARRAY (one percpu unit per slot)
  //   ptrs:   [AtomicPtr<u8>; 0]           // PROG_ARRAY / PERF_EVENT_ARRAY / CGROUP_ARRAY / ARRAY_OF_MAPS
}

struct BpfArrayAux {
  poke_progs: list_head<ProgPokeElem>,      // tracked tail-call callers
  poke_mutex: Mutex,                        // serializes xchg + bpf_arch_poke_desc_update
  work: WorkStruct,                         // prog_array_map_clear_deferred
  map: *BpfMap,                             // back-pointer
}

struct BpfEventEntry {
  event: *PerfEvent,
  perf_file: *File,                         // perf-event fd (fput on RCU free)
  map_file: *File,                          // bpf-map fd (scoped per-fd release)
  rcu: RcuHead,
}

struct ProgPokeElem {
  list: ListHead,
  aux: *BpfProgAux,                         // tracked program's aux (poke_tab walked at runtime)
}
```

`Array::alloc_check(attr) -> Result<()>`:
1. /* Universal validation */
2. if max_entries == 0 ∨ key_size != 4 ∨ value_size == 0 ∨ (map_flags & ~ARRAY_CREATE_FLAG_MASK) ∨ !bpf_map_flags_access_ok(map_flags): return -EINVAL.
3. if percpu ∧ numa_node != NUMA_NO_NODE: return -EINVAL.
4. /* Flag/type compatibility */
5. if map_type != BPF_MAP_TYPE_ARRAY ∧ (map_flags & (BPF_F_MMAPABLE | BPF_F_INNER_MAP)): return -EINVAL.
6. if map_type != BPF_MAP_TYPE_PERF_EVENT_ARRAY ∧ (map_flags & BPF_F_PRESERVE_ELEMS): return -EINVAL.
7. /* Overflow caps */
8. if value_size > INT_MAX: return -E2BIG.
9. if percpu ∧ round_up(value_size, 8) > PCPU_MIN_UNIT_SIZE: return -E2BIG.

`Array::alloc(attr) -> Result<*BpfMap>`:
1. elem_size = round_up(value_size, 8).
2. mask64 = (1ULL << fls_long(max_entries - 1)) - 1; index_mask = (u32)mask64.
3. bypass_spec_v1 = bpf_bypass_spec_v1(NULL).
4. if !bypass_spec_v1: max_entries = index_mask + 1; if overflow: return -E2BIG.
5. /* Compute total size */
6. array_size = sizeof(BpfArray).
7. if percpu: array_size += max_entries * sizeof(void *).
8. else if BPF_F_MMAPABLE: array_size = PAGE_ALIGN(array_size) + PAGE_ALIGN(max_entries * elem_size).
9. else: array_size += max_entries * elem_size.
10. /* Backing alloc */
11. if BPF_F_MMAPABLE: data = bpf_map_area_mmapable_alloc(array_size, numa_node); array = data + PAGE_ALIGN(sizeof(BpfArray)) - offsetof(BpfArray, value).
12. else: array = bpf_map_area_alloc(array_size, numa_node).
13. if !array: return -ENOMEM.
14. array.index_mask = index_mask; array.map.bypass_spec_v1 = bypass_spec_v1.
15. bpf_map_init_from_attr(&array.map, attr); array.elem_size = elem_size.
16. if percpu: Array::alloc_percpu(array) or bpf_map_area_free; return -ENOMEM.
17. return Ok(&array.map).

`Array::map_lookup_elem(map, key) -> Option<*u8>`:
1. index = *(u32 *)key.
2. if index >= max_entries: return None.
3. return Some(array.value + (u64)elem_size * (index & index_mask)).

`Array::percpu_map_lookup_elem(map, key) -> Option<*u8>`:
1. index = *(u32 *)key; if index >= max_entries: return None.
2. return Some(this_cpu_ptr(array.pptrs[index & index_mask])).

`Array::map_update_elem(map, key, value, map_flags) -> Result<()>`:
1. if (map_flags & ~BPF_F_LOCK) > BPF_EXIST: return -EINVAL.
2. index = *(u32 *)key.
3. if index >= max_entries: return -E2BIG.
4. if map_flags & BPF_NOEXIST: return -EEXIST.
5. if (map_flags & BPF_F_LOCK) ∧ !btf_record_has_field(record, BPF_SPIN_LOCK): return -EINVAL.
6. if map_type == BPF_MAP_TYPE_PERCPU_ARRAY:
   - val = this_cpu_ptr(array.pptrs[index & index_mask]).
   - copy_map_value(map, val, value); bpf_obj_free_fields(record, val).
7. else:
   - val = array.value + elem_size * (index & index_mask).
   - if map_flags & BPF_F_LOCK: copy_map_value_locked(map, val, value, false) else: copy_map_value(map, val, value).
   - bpf_obj_free_fields(record, val).
8. return Ok(()).

`Array::map_delete_elem(map, key) -> Result<()>`:
1. return Err(-EINVAL) /* ARRAY/PERCPU_ARRAY slots permanent */.

`Array::bpf_fd_map_update_elem(map, map_file, key, value, map_flags) -> Result<()>` (PROG_ARRAY / PERF_EVENT_ARRAY / CGROUP_ARRAY / ARRAY_OF_MAPS):
1. if map_flags != BPF_ANY: return -EINVAL.
2. index = *(u32 *)key; if index >= max_entries: return -E2BIG.
3. ufd = *(u32 *)value.
4. new_ptr = map.ops.map_fd_get_ptr(map, map_file, ufd)?; /* PROG_ARRAY rejects BPF_PROG_TYPE_EXT, increments prog_array_member_cnt or returns -EBUSY on freplace target */.
5. if map.ops.map_poke_run:
   - mutex_lock(&array.aux.poke_mutex).
   - old_ptr = xchg(&array.ptrs[index], new_ptr).
   - map.ops.map_poke_run(map, index, old_ptr, new_ptr) /* arch JIT rewrites direct-call sites */.
   - mutex_unlock.
6. else: old_ptr = xchg(&array.ptrs[index], new_ptr).
7. if old_ptr: map.ops.map_fd_put_ptr(map, old_ptr, /* need_defer */ true).
8. return Ok(()).

`Array::fd_map_delete_elem(map, key, need_defer) -> Result<()>`:
1. index = *(u32 *)key; if index >= max_entries: return -E2BIG.
2. (mutex_lock(&aux.poke_mutex) when map_poke_run present.)
3. old_ptr = xchg(&array.ptrs[index], NULL).
4. (map_poke_run(map, index, old_ptr, NULL); mutex_unlock.)
5. if old_ptr: map.ops.map_fd_put_ptr(map, old_ptr, need_defer); return Ok(()).
6. else: return Err(-ENOENT).

`Array::prog_poke_run(map, key, old, new)` — must hold `aux.poke_mutex`:
1. WARN_ON_ONCE(!mutex_is_locked(&aux.poke_mutex)).
2. for elem in aux.poke_progs:
   - for i in 0..elem.aux.size_poke_tab:
     - poke = &elem.aux.poke_tab[i].
     - if !READ_ONCE(poke.tailcall_target_stable): continue /* JIT in progress */.
     - if poke.reason != BPF_POKE_REASON_TAIL_CALL: continue.
     - if poke.tail_call.map != map ∨ poke.tail_call.key != key: continue.
     - Array::arch_poke_desc_update(poke, new, old) /* arch JIT — text patching */.

`Array::perf_event_release(map, map_file)` — `map_release`:
1. if map_flags & BPF_F_PRESERVE_ELEMS: return.
2. rcu_read_lock.
3. for i in 0..max_entries:
   - ee = READ_ONCE(array.ptrs[i]).
   - if ee ∧ ee.map_file == map_file: Array::fd_map_delete_elem(map, &i, /* need_defer */ true).
4. rcu_read_unlock.

`Array::free(map)`:
1. if !IS_ERR_OR_NULL(map.record):
   - if percpu: for i, cpu: bpf_obj_free_fields(record, per_cpu_ptr(pptr_i, cpu)); cond_resched.
   - else: for i: bpf_obj_free_fields(record, Array::elem_ptr(array, i)).
2. if percpu: Array::free_percpu(array).
3. if BPF_F_MMAPABLE: bpf_map_area_free(Array::vmalloc_addr(array)).
4. else: bpf_map_area_free(array).

### Out of Scope

- map_in_map.c shared helpers (`bpf_map_meta_alloc`, `bpf_map_meta_free`, `bpf_map_fd_get_ptr`, `bpf_map_fd_put_ptr`) (covered in `map-in-map.md` if expanded)
- bpf_tail_call dispatch (covered in `bpf-core.md` Tier-3)
- Verifier inner-map type checking (covered in `verifier.md` Tier-3)
- BPF-CO-RE light-skeleton globals layout (`.bss/.data/.rodata` DATASEC) (covered separately if expanded)
- perf_event_get / perf_event_read_local semantics (covered in `kernel/events/core.md` Tier-3)
- cgroup_get_from_fd semantics (covered in `kernel/cgroup/cgroup.md` Tier-3)
- generic_map_lookup_batch / generic_map_update_batch (covered in `bpf-syscall.md` if expanded)
- bpf_iter machinery for arraymaps (covered in `bpf-iter.md` if expanded)
- Architecture-specific bpf_arch_poke_desc_update (e.g. arch/x86/net/bpf_jit_comp.c) (covered in arch JIT Tier-3 if expanded)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct bpf_array` | per-map state (base + index_mask + elem_size + value/pptrs/ptrs union + aux) | `BpfArray` |
| `struct bpf_array_aux` | per-PROG_ARRAY aux (poke_progs list, poke_mutex, deferred work, map back-pointer) | `BpfArrayAux` |
| `struct bpf_event_entry` | per-PERF_EVENT_ARRAY slot wrapper (event, perf_file, map_file, rcu) | `BpfEventEntry` |
| `struct prog_poke_elem` | per-tracked-prog JIT poke entry | `ProgPokeElem` |
| `array_map_alloc_check()` | per-create attr validation | `Array::alloc_check` |
| `array_map_alloc()` | per-create main (vmalloc / mmapable / percpu pptrs) | `Array::alloc` |
| `array_map_free()` | per-destroy (bpf_obj_free_fields + percpu free + mmap-or-area free) | `Array::free` |
| `bpf_array_alloc_percpu()` / `bpf_array_free_percpu()` | per-CPU value backing | `Array::alloc_percpu` / `free_percpu` |
| `array_map_elem_ptr()` | per-slot value pointer (flat) | `Array::elem_ptr` |
| `array_map_lookup_elem()` | per-bpf-prog lookup (flat) | `Array::map_lookup_elem` |
| `percpu_array_map_lookup_elem()` | per-bpf-prog lookup (this_cpu_ptr) | `Array::percpu_map_lookup_elem` |
| `percpu_array_map_lookup_percpu_elem()` | per-arbitrary-CPU percpu lookup | `Array::percpu_map_lookup_percpu_elem` |
| `array_map_gen_lookup()` | per-verifier inlining (flat — ~8 insns) | `Array::map_gen_lookup` |
| `percpu_array_map_gen_lookup()` | per-verifier inlining (percpu — ~10 insns) | `Array::percpu_map_gen_lookup` |
| `array_of_map_gen_lookup()` | per-verifier inlining (ARRAY_OF_MAPS — ~11 insns) | `Array::array_of_map_gen_lookup` |
| `array_map_update_elem()` | per-bpf-prog / syscall update (flat + percpu) | `Array::map_update_elem` |
| `bpf_percpu_array_copy()` | per-syscall percpu copy-out (BPF_F_CPU honored) | `Array::bpf_percpu_array_copy` |
| `bpf_percpu_array_update()` | per-syscall percpu update (BPF_F_CPU / BPF_F_ALL_CPUS) | `Array::bpf_percpu_array_update` |
| `array_map_delete_elem()` | per-bpf-prog delete (returns -EINVAL for ARRAY) | `Array::map_delete_elem` |
| `bpf_array_get_next_key()` | per-iter (next u32 index) | `Array::map_get_next_key` |
| `array_map_get_hash()` | per-sha256 mmap-able snapshot (BPF-CO-RE freeze) | `Array::map_get_hash` |
| `array_map_direct_value_addr()` / `array_map_direct_value_meta()` | per-direct-value-load (BPF-light-skeleton globals) | `Array::map_direct_value_addr` / `map_direct_value_meta` |
| `array_map_mmap()` | per-userspace mmap (BPF_F_MMAPABLE) | `Array::map_mmap` |
| `array_map_vmalloc_addr()` | per-vmalloc-start helper | `Array::vmalloc_addr` |
| `array_map_meta_equal()` | per-inner-map meta-compatibility (BPF_F_INNER_MAP relaxes max_entries) | `Array::map_meta_equal` |
| `array_map_seq_show_elem()` / `percpu_array_map_seq_show_elem()` / `prog_array_map_seq_show_elem()` | per-bpffs seq | `Array::seq_show_elem` |
| `array_map_check_btf()` | per-create BTF key=u32 + DATASEC special case | `Array::map_check_btf` |
| `bpf_for_each_array_elem()` | per-bpf-callback iteration | `Array::bpf_for_each_array_elem` |
| `array_map_mem_usage()` | per-show-fdinfo | `Array::map_mem_usage` |
| `array_map_free_internal_structs()` | per-uref-drop (special-field cleanup) | `Array::free_internal_structs` |
| `fd_array_map_alloc_check()` | per-fd-flavor attr validation | `Array::fd_alloc_check` |
| `fd_array_map_free()` | per-fd-flavor destroy (BUG_ON non-NULL slot) | `Array::fd_free` |
| `fd_array_map_lookup_elem()` | per-fd-flavor bpf-prog lookup (always -EOPNOTSUPP) | `Array::fd_map_lookup_elem` |
| `bpf_fd_array_map_lookup_elem()` | per-syscall fd-array lookup (resolve to id) | `Array::bpf_fd_map_lookup_elem` |
| `bpf_fd_array_map_update_elem()` | per-syscall fd-array update (xchg + map_poke_run) | `Array::bpf_fd_map_update_elem` |
| `__fd_array_map_delete_elem()` / `fd_array_map_delete_elem()` | per-fd-flavor delete (xchg NULL + put with defer) | `Array::fd_map_delete_elem` |
| `bpf_fd_array_map_clear()` | per-uref-zero bulk reset | `Array::fd_map_clear` |
| `prog_fd_array_get_ptr()` / `prog_fd_array_put_ptr()` / `prog_fd_array_sys_lookup_elem()` | per-PROG_ARRAY slot ops (rejects PROG_TYPE_EXT, tracks `prog_array_member_cnt`) | `Array::prog_fd_get_ptr` / `put_ptr` / `sys_lookup_elem` |
| `prog_array_map_alloc()` / `prog_array_map_free()` | per-PROG_ARRAY lifecycle (allocates `bpf_array_aux`) | `Array::prog_array_alloc` / `free` |
| `prog_array_map_poke_track()` / `prog_array_map_poke_untrack()` / `prog_array_map_poke_run()` | per-tail-call JIT poke registration + dispatch | `Array::prog_poke_track` / `untrack` / `run` |
| `bpf_arch_poke_desc_update()` | per-arch text patching (weak — overridden by arch JIT) | `Array::arch_poke_desc_update` |
| `prog_array_map_clear()` / `prog_array_map_clear_deferred()` | per-uref-zero deferred clear (workqueue) | `Array::prog_array_clear` |
| `perf_event_fd_array_get_ptr()` / `perf_event_fd_array_put_ptr()` | per-PERF_EVENT_ARRAY slot ops (perf_event_get + read_local probe + entry alloc) | `Array::perf_event_get_ptr` / `put_ptr` |
| `bpf_event_entry_gen()` / `__bpf_event_entry_free()` / `bpf_event_entry_free_rcu()` | per-event-entry lifecycle | `Array::event_entry_gen` / `free_rcu` |
| `perf_event_fd_array_release()` | per-map-fd close (drop slots whose map_file matches; honors PRESERVE_ELEMS) | `Array::perf_event_release` |
| `perf_event_fd_array_map_free()` | per-PERF_EVENT_ARRAY destroy | `Array::perf_event_free` |
| `cgroup_fd_array_get_ptr()` / `cgroup_fd_array_put_ptr()` / `cgroup_fd_array_free()` | per-CGROUP_ARRAY slot ops (cgroup_get_from_fd + cgroup_put) | `Array::cgroup_fd_get_ptr` / `put_ptr` / `free` |
| `array_of_map_alloc()` / `array_of_map_free()` / `array_of_map_lookup_elem()` | per-ARRAY_OF_MAPS lifecycle + lookup | `Array::array_of_map_*` |
| `bpf_map_fd_get_ptr()` / `bpf_map_fd_put_ptr()` / `bpf_map_fd_sys_lookup_elem()` | per-ARRAY_OF_MAPS slot ops (in `map_in_map.c`) | shared via map_in_map |
| `bpf_iter_init_array_map()` / `bpf_iter_fini_array_map()` / `bpf_array_map_seq_*` | per-bpf_iter seq | `Array::bpf_iter_*` |
| `array_map_ops` / `percpu_array_map_ops` / `prog_array_map_ops` / `perf_event_array_map_ops` / `cgroup_array_map_ops` / `array_of_maps_map_ops` | per-flavor map_ops vtable | shared |

### compatibility contract

REQ-1: struct bpf_array (per-map):
- map: `struct bpf_map` (base; map_type, key_size=4, value_size, max_entries, map_flags, numa_node, record, bypass_spec_v1, sha[32], ...).
- aux: `struct bpf_array_aux *` — PROG_ARRAY only (NULL for others).
- index_mask: u32 — `roundup_pow_of_two(max_entries) - 1`; equals `max_entries-1` (or stricter rounded mask) — used to AND the user-provided index in the data path to bound speculation.
- elem_size: u32 — `round_up(value_size, 8)` (always 8-byte aligned for `copy_map_value_long` and percpu unit alignment).
- /* Trailing flexible union — exactly one variant in use per flavor */
- union {
  - value: `char[]` __aligned(8) — flat ARRAY value storage (`max_entries * elem_size` bytes; first cacheline is `bpf_array` header so consumers index via `(u8 *)&array->value`).
  - pptrs: `void __percpu *[0]` — PERCPU_ARRAY (`max_entries * sizeof(void *)`; each slot points to a `bpf_map_alloc_percpu`-allocated unit of `elem_size` bytes per CPU).
  - ptrs: `void *[0]` — PROG_ARRAY / PERF_EVENT_ARRAY / CGROUP_ARRAY / ARRAY_OF_MAPS (`max_entries * sizeof(void *)`; each slot atomically swapped via `xchg`).
- }
- BUILD_BUG_ON: `offsetof(struct bpf_array, map) == 0` (enforced in `percpu_array_map_gen_lookup` — JIT relies on offset-0 of `bpf_array` being the embedded `bpf_map`).

REQ-2: struct bpf_array_aux (per-PROG_ARRAY only):
- poke_progs: `struct list_head` of `prog_poke_elem` entries — each entry holds a `bpf_prog_aux *` for a JIT'd program that has a tail-call into this map.
- poke_mutex: `struct mutex` — serializes `map_poke_run` text-patching against concurrent `xchg` updates to `array->ptrs[i]`.
- work: `struct work_struct` — for deferred `prog_array_map_clear` via `cgroup_bpf_destroy_wq`-style separate workqueue (here `schedule_work` system_wq).
- map: `struct bpf_map *` — back-pointer (used by `prog_array_map_clear_deferred`).

REQ-3: struct bpf_event_entry (per-PERF_EVENT_ARRAY slot):
- event: `struct perf_event *` — `= perf_file->private_data`.
- perf_file: `struct file *` — the perf-event fd's file (held to anchor the event lifetime).
- map_file: `struct file *` — the bpf-map's owning file (used by `perf_event_fd_array_release` to scope per-fd cleanup).
- rcu: `struct rcu_head` — deferred free via `call_rcu(&ee->rcu, __bpf_event_entry_free)` which `fput(perf_file)` then `kfree(ee)`.

REQ-4: array_map_alloc_check(attr):
- percpu = (map_type == BPF_MAP_TYPE_PERCPU_ARRAY).
- numa_node = bpf_map_attr_numa_node(attr).
- /* Universal attr checks */
- if max_entries == 0 ∨ key_size != 4 ∨ value_size == 0: return -EINVAL.
- if map_flags & ~ARRAY_CREATE_FLAG_MASK: return -EINVAL.
  - ARRAY_CREATE_FLAG_MASK = BPF_F_NUMA_NODE | BPF_F_MMAPABLE | BPF_F_ACCESS_MASK | BPF_F_PRESERVE_ELEMS | BPF_F_INNER_MAP.
- if !bpf_map_flags_access_ok(map_flags): return -EINVAL.
- if percpu ∧ numa_node != NUMA_NO_NODE: return -EINVAL /* percpu pins are per-CPU; numa pinning meaningless */.
- /* Flag/type compatibility */
- if map_type != BPF_MAP_TYPE_ARRAY ∧ (map_flags & (BPF_F_MMAPABLE | BPF_F_INNER_MAP)): return -EINVAL /* MMAPABLE + INNER_MAP only for flat ARRAY */.
- if map_type != BPF_MAP_TYPE_PERF_EVENT_ARRAY ∧ (map_flags & BPF_F_PRESERVE_ELEMS): return -EINVAL.
- /* Overflow / size caps */
- if value_size > INT_MAX: return -E2BIG /* round_up(value_size, 8) would overflow */.
- if percpu ∧ round_up(value_size, 8) > PCPU_MIN_UNIT_SIZE: return -E2BIG /* per-CPU allocator unit cap */.
- return 0.

REQ-5: array_map_alloc(attr) (called from syscall):
- elem_size = round_up(value_size, 8).
- max_entries = attr.max_entries.
- /* Build index_mask in u64 to avoid 32-bit `1U << 32` UB */
- mask64 = fls_long(max_entries - 1); mask64 = (1ULL << mask64) - 1.
- index_mask = (u32)mask64.
- bypass_spec_v1 = bpf_bypass_spec_v1(NULL).
- if !bypass_spec_v1: max_entries = index_mask + 1 /* round up to power-of-2 so AND with index_mask is sufficient */; if max_entries < attr.max_entries: return -E2BIG.
- /* Compute total size */
- array_size = sizeof(*array).
- if percpu: array_size += (u64)max_entries * sizeof(void *) /* pptrs slot array */.
- else if (map_flags & BPF_F_MMAPABLE): array_size = PAGE_ALIGN(array_size) + PAGE_ALIGN((u64)max_entries * elem_size).
- else: array_size += (u64)max_entries * elem_size.
- /* Allocate backing storage */
- if BPF_F_MMAPABLE: data = bpf_map_area_mmapable_alloc(array_size, numa_node); array = data + PAGE_ALIGN(sizeof(*array)) - offsetof(bpf_array, value) /* leaves value[] page-aligned for remap_vmalloc_range */.
- else: array = bpf_map_area_alloc(array_size, numa_node).
- if !array: return -ENOMEM.
- array.index_mask = index_mask.
- array.map.bypass_spec_v1 = bypass_spec_v1.
- bpf_map_init_from_attr(&array.map, attr).
- array.elem_size = elem_size.
- /* Per-CPU value backing */
- if percpu ∧ bpf_array_alloc_percpu(array) < 0: bpf_map_area_free(array); return -ENOMEM.
- return &array.map.

REQ-6: bpf_array_alloc_percpu(array):
- for i in 0..max_entries:
  - ptr = bpf_map_alloc_percpu(&array.map, elem_size, /* align */ 8, GFP_USER | __GFP_NOWARN).
  - if !ptr: bpf_array_free_percpu(array); return -ENOMEM.
  - array.pptrs[i] = ptr.
  - cond_resched() /* large allocations are cooperative */.
- return 0.

REQ-7: array_map_lookup_elem(map, key):
- index = *(u32 *)key.
- if unlikely(index >= max_entries): return NULL.
- /* Spec-v1 mitigation via AND-mask */
- return array.value + (u64)elem_size * (index & index_mask).

REQ-8: percpu_array_map_lookup_elem(map, key):
- index = *(u32 *)key.
- if unlikely(index >= max_entries): return NULL.
- return this_cpu_ptr(array.pptrs[index & index_mask]).

REQ-9: percpu_array_map_lookup_percpu_elem(map, key, cpu):
- if cpu >= nr_cpu_ids: return NULL.
- index = *(u32 *)key.
- if unlikely(index >= max_entries): return NULL.
- return per_cpu_ptr(array.pptrs[index & index_mask], cpu).

REQ-10: array_map_gen_lookup(map, insn_buf) — verifier inlining (~8 insns):
- if (map_flags & BPF_F_INNER_MAP): return -EOPNOTSUPP /* inner-map array — different layout */.
- emit:
  - BPF_ALU64_IMM(BPF_ADD, R1, offsetof(bpf_array, value)) /* R1 = &array->value */.
  - BPF_LDX_MEM(BPF_W, R0, R2, 0) /* R0 = *key (u32 index) */.
  - if !bypass_spec_v1: BPF_JMP_IMM(BPF_JGE, R0, max_entries, 4); BPF_ALU32_IMM(BPF_AND, R0, index_mask).
  - else: BPF_JMP_IMM(BPF_JGE, R0, max_entries, 3).
  - if is_power_of_2(elem_size): BPF_ALU64_IMM(BPF_LSH, R0, ilog2(elem_size)).
  - else: BPF_ALU64_IMM(BPF_MUL, R0, elem_size).
  - BPF_ALU64_REG(BPF_ADD, R0, R1) /* R0 = &value[index*elem_size] */.
  - BPF_JMP_IMM(BPF_JA, 0, 0, 1); BPF_MOV64_IMM(R0, 0) /* OOB → NULL */.
- return insn_count.

REQ-11: percpu_array_map_gen_lookup(map, insn_buf) — verifier inlining (~10 insns):
- if !bpf_jit_supports_percpu_insn(): return -EOPNOTSUPP.
- if (map_flags & BPF_F_INNER_MAP): return -EOPNOTSUPP.
- BUILD_BUG_ON(offsetof(struct bpf_array, map) != 0) — used to confirm R1 starts at `&array->map` so `R1 + offsetof(pptrs)` is correct.
- emit:
  - BPF_ALU64_IMM(BPF_ADD, R1, offsetof(bpf_array, pptrs)).
  - BPF_LDX_MEM(BPF_W, R0, R2, 0) /* R0 = index */.
  - if !bypass_spec_v1: JGE bound; BPF_AND R0, index_mask.
  - BPF_ALU64_IMM(BPF_LSH, R0, 3) /* * sizeof(void *) */.
  - BPF_ALU64_REG(BPF_ADD, R0, R1).
  - BPF_LDX_MEM(BPF_DW, R0, R0, 0) /* R0 = pptr */.
  - BPF_MOV64_PERCPU_REG(R0, R0) /* per-CPU translate */.
  - BPF_JMP_IMM(BPF_JA, 0, 0, 1); BPF_MOV64_IMM(R0, 0).
- return insn_count.

REQ-12: array_map_update_elem(map, key, value, map_flags):
- if (map_flags & ~BPF_F_LOCK) > BPF_EXIST: return -EINVAL.
- index = *(u32 *)key.
- if unlikely(index >= max_entries): return -E2BIG /* prealloc — cannot grow */.
- if unlikely(map_flags & BPF_NOEXIST): return -EEXIST /* all slots always exist */.
- if (map_flags & BPF_F_LOCK) ∧ !btf_record_has_field(map.record, BPF_SPIN_LOCK): return -EINVAL.
- if map_type == BPF_MAP_TYPE_PERCPU_ARRAY:
  - val = this_cpu_ptr(array.pptrs[index & index_mask]).
  - copy_map_value(map, val, value).
  - bpf_obj_free_fields(map.record, val) /* free old kptr/timer/spin_lock fields */.
- else:
  - val = array.value + (u64)elem_size * (index & index_mask).
  - if (map_flags & BPF_F_LOCK): copy_map_value_locked(map, val, value, false).
  - else: copy_map_value(map, val, value).
  - bpf_obj_free_fields(map.record, val).
- return 0.

REQ-13: bpf_percpu_array_copy(map, key, value, map_flags) — syscall percpu copy-out:
- index = *(u32 *)key.
- if unlikely(index >= max_entries): return -ENOENT.
- size = elem_size.
- rcu_read_lock.
- pptr = array.pptrs[index & index_mask].
- if (map_flags & BPF_F_CPU):
  - cpu = map_flags >> 32.
  - copy_map_value(map, value, per_cpu_ptr(pptr, cpu)).
  - check_and_init_map_value(map, value) /* zero special fields in user copy */.
- else:
  - off = 0.
  - for_each_possible_cpu(cpu):
    - copy_map_value_long(map, value + off, per_cpu_ptr(pptr, cpu)).
    - check_and_init_map_value(map, value + off).
    - off += size.
- rcu_read_unlock.
- return 0.

REQ-14: bpf_percpu_array_update(map, key, value, map_flags) — syscall percpu update:
- if (map_flags & BPF_F_LOCK) ∨ (u32)map_flags > BPF_F_ALL_CPUS: return -EINVAL.
- index = *(u32 *)key; if index >= max_entries: return -E2BIG.
- if map_flags == BPF_NOEXIST: return -EEXIST.
- size = elem_size.
- rcu_read_lock.
- pptr = array.pptrs[index & index_mask].
- if (map_flags & BPF_F_CPU):
  - cpu = map_flags >> 32.
  - ptr = per_cpu_ptr(pptr, cpu).
  - copy_map_value(map, ptr, value).
  - bpf_obj_free_fields(map.record, ptr).
- else:
  - for_each_possible_cpu(cpu):
    - ptr = per_cpu_ptr(pptr, cpu).
    - val = (map_flags & BPF_F_ALL_CPUS) ? value : value + size * cpu.
    - copy_map_value(map, ptr, val).
    - bpf_obj_free_fields(map.record, ptr).
- rcu_read_unlock.
- return 0.

REQ-15: array_map_delete_elem(map, key): return -EINVAL /* ARRAY / PERCPU_ARRAY slots are permanent */.

REQ-16: bpf_array_get_next_key(map, key, next_key):
- index = key ? *(u32 *)key : U32_MAX.
- if index >= max_entries: *next = 0; return 0 /* start of iter */.
- if index == max_entries - 1: return -ENOENT /* end */.
- *next = index + 1.

REQ-17: array_map_get_hash(map, hash_buf_size, hash_buf):
- sha256(array.value, (u64)elem_size * max_entries, hash_buf).
- memcpy(map.sha, hash_buf, sizeof(map.sha)).
- return 0.
- (Used by `bpftool prog freeze` / BPF-CO-RE mmap-able snapshot.)

REQ-18: array_map_direct_value_addr(map, *imm, off) / array_map_direct_value_meta(map, imm, *off):
- /* Only valid for single-slot maps (BPF-light-skeleton globals) */
- if map.max_entries != 1: return -ENOTSUPP.
- _addr: if off >= value_size: return -EINVAL; *imm = (unsigned long)&array.value; return 0.
- _meta: base = (u64)&array.value; range = elem_size; if imm < base ∨ imm >= base + range: return -ENOENT; *off = imm - base.

REQ-19: array_map_mmap(map, vma):
- if !(map_flags & BPF_F_MMAPABLE): return -EINVAL.
- /* Validate range against value region (not full bpf_array allocation) */
- if vma.vm_pgoff * PAGE_SIZE + (vma.vm_end - vma.vm_start) > PAGE_ALIGN((u64)max_entries * elem_size): return -EINVAL.
- /* Skip the header pages */
- pgoff = PAGE_ALIGN(sizeof(*array)) >> PAGE_SHIFT.
- return remap_vmalloc_range(vma, array_map_vmalloc_addr(array), vma.vm_pgoff + pgoff).

REQ-20: array_map_meta_equal(meta0, meta1):
- if !bpf_map_meta_equal(meta0, meta1): return false /* base map-type / key / value / flags match */.
- /* BPF_F_INNER_MAP relaxes max_entries: different sized inner arrays allowed */
- return (meta0.map_flags & BPF_F_INNER_MAP) ? true : meta0.max_entries == meta1.max_entries.

REQ-21: array_map_check_btf(map, btf, key_type, value_type):
- /* DATASEC exception: keyless .bss/.data/.rodata global maps */
- if btf_type_is_void(key_type):
  - if map_type != BPF_MAP_TYPE_ARRAY ∨ max_entries != 1: return -EINVAL.
  - if BTF_INFO_KIND(value_type.info) != BTF_KIND_DATASEC: return -EINVAL.
  - return 0.
- /* Default: u32 key required */
- if !btf_type_is_i32(key_type): return -EINVAL.

REQ-22: fd_array_map_alloc_check(attr):
- if attr.value_size != sizeof(u32): return -EINVAL /* fd-arrays hold u32 fds, not raw values */.
- if (map_flags & (BPF_F_RDONLY_PROG | BPF_F_WRONLY_PROG)): return -EINVAL.
- return array_map_alloc_check(attr).

REQ-23: bpf_fd_array_map_update_elem(map, map_file, key, value, map_flags) — syscall fd-array update (PROG_ARRAY / PERF_EVENT_ARRAY / CGROUP_ARRAY / ARRAY_OF_MAPS):
- if map_flags != BPF_ANY: return -EINVAL.
- index = *(u32 *)key; if index >= max_entries: return -E2BIG.
- ufd = *(u32 *)value.
- new_ptr = map.ops.map_fd_get_ptr(map, map_file, ufd) /* resolves fd to prog / event-entry / cgroup / inner-map; may return ERR_PTR */.
- if IS_ERR(new_ptr): return PTR_ERR.
- /* PROG_ARRAY: tail-call JIT poke synchronization */
- if map.ops.map_poke_run:
  - mutex_lock(&array.aux.poke_mutex).
  - old_ptr = xchg(array.ptrs + index, new_ptr).
  - map.ops.map_poke_run(map, index, old_ptr, new_ptr).
  - mutex_unlock(&array.aux.poke_mutex).
- else: old_ptr = xchg(array.ptrs + index, new_ptr).
- if old_ptr: map.ops.map_fd_put_ptr(map, old_ptr, /* need_defer */ true).

REQ-24: __fd_array_map_delete_elem(map, key, need_defer) / fd_array_map_delete_elem:
- index = *(u32 *)key; if index >= max_entries: return -E2BIG.
- if map_poke_run: mutex_lock; old_ptr = xchg(ptrs+index, NULL); map_poke_run(map, index, old_ptr, NULL); mutex_unlock.
- else: old_ptr = xchg(ptrs + index, NULL).
- if old_ptr: map.ops.map_fd_put_ptr(map, old_ptr, need_defer); return 0.
- else: return -ENOENT.
- /* Public fd_array_map_delete_elem always passes need_defer=true; uref-clear may pass false */

REQ-25: bpf_fd_array_map_clear(map, need_defer):
- for i in 0..max_entries: __fd_array_map_delete_elem(map, &i, need_defer); cond_resched.

REQ-26: prog_fd_array_get_ptr(map, map_file, fd) — PROG_ARRAY slot resolution:
- prog = bpf_prog_get(fd) /* increments prog refcnt */.
- if prog.type == BPF_PROG_TYPE_EXT ∨ !bpf_prog_map_compatible(map, prog): bpf_prog_put; return ERR_PTR(-EINVAL).
- mutex_lock(&prog.aux.ext_mutex).
- is_extended = prog.aux.is_extended.
- if !is_extended: prog.aux.prog_array_member_cnt++.
- mutex_unlock(&prog.aux.ext_mutex).
- if is_extended: bpf_prog_put; return ERR_PTR(-EBUSY) /* freplace target cannot be tail callee — prevents infinite loop */.
- return prog.

REQ-27: prog_fd_array_put_ptr(map, ptr, need_defer):
- prog = ptr.
- mutex_lock(&prog.aux.ext_mutex); prog.aux.prog_array_member_cnt--; mutex_unlock.
- bpf_prog_put(prog) /* freed after one RCU tasks-trace grace period — safe for tail-call in-flight */.

REQ-28: prog_array_map_alloc(attr):
- aux = kzalloc(sizeof(*aux), GFP_KERNEL_ACCOUNT); if !aux: -ENOMEM.
- INIT_WORK(&aux.work, prog_array_map_clear_deferred).
- INIT_LIST_HEAD(&aux.poke_progs).
- mutex_init(&aux.poke_mutex).
- map = array_map_alloc(attr); if IS_ERR(map): kfree(aux); return map.
- container_of(map, bpf_array, map).aux = aux.
- aux.map = map.
- return map.

REQ-29: prog_array_map_free(map):
- aux = container_of(map, bpf_array, map).aux.
- list_for_each_entry_safe(elem, tmp, &aux.poke_progs, list): list_del_init; kfree(elem).
- kfree(aux).
- fd_array_map_free(map) /* BUG_ON any non-NULL slot */.

REQ-30: prog_array_map_poke_track(map, prog_aux):
- elem = kmalloc(prog_poke_elem); elem.aux = prog_aux.
- mutex_lock(&aux.poke_mutex).
- if already-tracked: kfree(elem); return 0.
- list_add_tail(&elem.list, &aux.poke_progs).
- mutex_unlock.
- /* Called by verifier when a JIT-compiled program contains a tail-call into this PROG_ARRAY */.

REQ-31: prog_array_map_poke_run(map, key, old, new) — must hold poke_mutex:
- WARN_ON_ONCE(!mutex_is_locked(&aux.poke_mutex)).
- list_for_each_entry(elem, &aux.poke_progs):
  - for i in 0..elem.aux.size_poke_tab:
    - poke = &elem.aux.poke_tab[i].
    - /* Lifecycle guards */
    - if !READ_ONCE(poke.tailcall_target_stable): continue /* JIT still finishing */.
    - if poke.reason != BPF_POKE_REASON_TAIL_CALL: continue.
    - if poke.tail_call.map != map ∨ poke.tail_call.key != key: continue.
    - bpf_arch_poke_desc_update(poke, new, old) /* arch JIT rewrites direct call site */.

REQ-32: prog_array_map_clear(map) (`map_release_uref`):
- aux = container_of(map, bpf_array, map).aux.
- bpf_map_inc(map) /* prevent map from being freed before deferred clear runs */.
- schedule_work(&aux.work) → prog_array_map_clear_deferred:
  - bpf_fd_array_map_clear(map, /* need_defer */ true).
  - bpf_map_put(map).

REQ-33: perf_event_fd_array_get_ptr(map, map_file, fd):
- perf_file = perf_event_get(fd); if IS_ERR: return.
- event = perf_file.private_data.
- /* Preflight: perf_event_read_local must be supported for in-kernel reads */
- if perf_event_read_local(event, &value, NULL, NULL) == -EOPNOTSUPP: fput(perf_file); return ERR_PTR(-EOPNOTSUPP).
- ee = bpf_event_entry_gen(perf_file, map_file).
- if !ee: fput(perf_file); return ERR_PTR(-ENOMEM).
- return ee.

REQ-34: perf_event_fd_array_put_ptr(map, ptr, need_defer):
- bpf_event_entry_free_rcu(ptr) → call_rcu(&ee.rcu, __bpf_event_entry_free) /* deferred fput(perf_file) + kfree */.

REQ-35: perf_event_fd_array_release(map, map_file) (`map_release`):
- if (map_flags & BPF_F_PRESERVE_ELEMS): return /* slots persist past fd-close */.
- rcu_read_lock.
- for i in 0..max_entries:
  - ee = READ_ONCE(array.ptrs[i]).
  - if ee ∧ ee.map_file == map_file: __fd_array_map_delete_elem(map, &i, /* need_defer */ true).
- rcu_read_unlock.

REQ-36: perf_event_fd_array_map_free(map):
- if (map_flags & BPF_F_PRESERVE_ELEMS): bpf_fd_array_map_clear(map, /* need_defer */ false) /* PRESERVE_ELEMS skipped release-time clear */.
- fd_array_map_free(map).

REQ-37: cgroup_fd_array_get_ptr(map, map_file, fd): return cgroup_get_from_fd(fd) /* pins cgroup */.

REQ-38: cgroup_fd_array_put_ptr(map, ptr, need_defer): cgroup_put(ptr) /* freed after RCU grace */.

REQ-39: cgroup_fd_array_free(map): bpf_fd_array_map_clear(map, /* need_defer */ false); fd_array_map_free(map).

REQ-40: array_of_map_alloc(attr) — ARRAY_OF_MAPS:
- inner_map_meta = bpf_map_meta_alloc(attr.inner_map_fd); if IS_ERR: return.
- map = array_map_alloc(attr); if IS_ERR: bpf_map_meta_free(inner_map_meta); return.
- map.inner_map_meta = inner_map_meta /* template used by verifier to type-check inner-map lookups */.

REQ-41: array_of_map_free(map):
- bpf_map_meta_free(map.inner_map_meta) /* syscall-only field, protected by fdget/fdput */.
- bpf_fd_array_map_clear(map, /* need_defer */ false).
- fd_array_map_free(map).

REQ-42: array_of_map_lookup_elem(map, key):
- inner_map = array_map_lookup_elem(map, key) /* returns &array.value[index * 8] holding inner map pointer */.
- if !inner_map: return NULL.
- return READ_ONCE(*inner_map).

REQ-43: array_of_map_gen_lookup(map, insn_buf) — verifier inlining (~11 insns):
- /* Same shape as array_map_gen_lookup but with an extra BPF_LDX_MEM(BPF_DW) to dereference the inner-map pointer slot */
- emit ALU64_ADD R1 → value; LDX_W R0, R2, 0 (index); bounds + mask; multiply elem_size; ADD R1; LDX_DW R0, R0, 0 (deref inner_map); JEQ R0, 0, 1; JA 1; MOV64 R0, 0.

REQ-44: bpf_for_each_array_elem(map, callback_fn, callback_ctx, flags):
- cant_migrate.
- if flags != 0: return -EINVAL.
- is_percpu = (map_type == BPF_MAP_TYPE_PERCPU_ARRAY).
- for i in 0..max_entries:
  - val = is_percpu ? this_cpu_ptr(array.pptrs[i]) : array_map_elem_ptr(array, i).
  - key = i.
  - ret = callback_fn(map, &key, val, callback_ctx, 0).
  - if ret: break /* 1 = stop */.
- return num_elems.

REQ-45: array_map_free(map):
- /* Drop special fields per slot */
- if !IS_ERR_OR_NULL(map.record):
  - if percpu: for i, cpu: bpf_obj_free_fields(map.record, per_cpu_ptr(pptr_i, cpu)); cond_resched.
  - else: for i: bpf_obj_free_fields(map.record, array_map_elem_ptr(array, i)).
- if percpu: bpf_array_free_percpu(array).
- if (map_flags & BPF_F_MMAPABLE): bpf_map_area_free(array_map_vmalloc_addr(array)).
- else: bpf_map_area_free(array).

REQ-46: array_map_free_internal_structs(map) (`map_release_uref`):
- if !bpf_map_has_internal_structs(map): return.
- for i in 0..max_entries: bpf_map_free_internal_structs(map, array_map_elem_ptr(array, i)).

REQ-47: array_map_mem_usage(map):
- usage = sizeof(*array).
- if percpu: usage += max_entries * sizeof(void *) + max_entries * elem_size * num_possible_cpus.
- else: if BPF_F_MMAPABLE: usage = PAGE_ALIGN(usage) + PAGE_ALIGN(max_entries * elem_size); else: usage += max_entries * elem_size.

REQ-48: bpf_iter for arraymap:
- bpf_iter_init_array_map: if percpu, kmalloc percpu_value_buf of elem_size * num_possible_cpus; bpf_map_inc_with_uref(map).
- seq_start/next: index = info.index & index_mask; if percpu, return (void *)(uintptr_t)pptrs[index]; else return array_map_elem_ptr.
- seq_show: bpf_iter_run_prog with ctx = { meta, map, key, value } — percpu copies all CPUs into percpu_value_buf in stride.
- bpf_iter_fini: bpf_map_put_with_uref; kfree percpu_value_buf.

REQ-49: Map-ops dispatch (per-flavor):
- `array_map_ops` (BPF_MAP_TYPE_ARRAY): map_meta_equal=array_map_meta_equal, map_alloc/free=array_*, map_lookup_elem=array_map_lookup_elem, map_update_elem=array_map_update_elem, map_delete_elem=array_map_delete_elem (-EINVAL), map_gen_lookup=array_map_gen_lookup, map_direct_value_addr/meta, map_mmap=array_map_mmap, map_release_uref=array_map_free_internal_structs, map_lookup_batch=generic, map_get_hash=array_map_get_hash.
- `percpu_array_map_ops` (BPF_MAP_TYPE_PERCPU_ARRAY): map_meta_equal=bpf_map_meta_equal (no BPF_F_INNER_MAP relaxation), map_lookup_elem=percpu_*, map_gen_lookup=percpu_*, map_lookup_percpu_elem=percpu_array_map_lookup_percpu_elem.
- `prog_array_map_ops` (BPF_MAP_TYPE_PROG_ARRAY): map_alloc=prog_array_map_alloc, map_free=prog_array_map_free, map_poke_track/untrack/run, map_fd_get_ptr=prog_fd_array_get_ptr, map_fd_put_ptr=prog_fd_array_put_ptr, map_fd_sys_lookup_elem=prog_fd_array_sys_lookup_elem (returns prog.aux.id), map_release_uref=prog_array_map_clear, map_lookup_elem=fd_array_map_lookup_elem (always -EOPNOTSUPP). No map_meta_equal: PROG_ARRAY cannot be inner_map (runtime binding of prog type/jited).
- `perf_event_array_map_ops` (BPF_MAP_TYPE_PERF_EVENT_ARRAY): map_release=perf_event_fd_array_release, map_check_btf=map_check_no_btf, map_fd_get_ptr=perf_event_fd_array_get_ptr.
- `cgroup_array_map_ops` (BPF_MAP_TYPE_CGROUP_ARRAY): map_fd_get_ptr=cgroup_fd_array_get_ptr, gated by `#ifdef CONFIG_CGROUPS`.
- `array_of_maps_map_ops` (BPF_MAP_TYPE_ARRAY_OF_MAPS): map_alloc=array_of_map_alloc, map_free=array_of_map_free, map_lookup_elem=array_of_map_lookup_elem, map_gen_lookup=array_of_map_gen_lookup, map_fd_get_ptr=bpf_map_fd_get_ptr (from map_in_map.c).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `key_size_is_4` | INVARIANT | per-alloc_check: key_size == 4 required. |
| `index_mask_bounds_index` | INVARIANT | per-lookup / update: (index & index_mask) < max_entries (when !bypass_spec_v1, max_entries == index_mask + 1). |
| `index_mask_power_of_two_minus_one` | INVARIANT | per-alloc: index_mask + 1 is power-of-2 (== roundup_pow_of_two(max_entries)). |
| `elem_size_aligned_8` | INVARIANT | per-alloc: elem_size == round_up(value_size, 8). |
| `percpu_value_size_bounded` | INVARIANT | per-alloc_check: percpu ∧ round_up(value_size, 8) ≤ PCPU_MIN_UNIT_SIZE. |
| `mmapable_only_for_array` | INVARIANT | per-alloc_check: BPF_F_MMAPABLE ⟹ map_type == BPF_MAP_TYPE_ARRAY. |
| `inner_map_only_for_array` | INVARIANT | per-alloc_check: BPF_F_INNER_MAP ⟹ map_type == BPF_MAP_TYPE_ARRAY. |
| `preserve_elems_only_for_perf` | INVARIANT | per-alloc_check: BPF_F_PRESERVE_ELEMS ⟹ map_type == BPF_MAP_TYPE_PERF_EVENT_ARRAY. |
| `percpu_no_numa_node` | INVARIANT | per-alloc_check: percpu ⟹ numa_node == NUMA_NO_NODE. |
| `prog_array_offset_zero` | INVARIANT | per-percpu_map_gen_lookup BUILD_BUG_ON: offsetof(bpf_array, map) == 0. |
| `delete_returns_einval` | INVARIANT | per-array_map_delete_elem: returns -EINVAL (no per-slot deletion). |
| `prog_ext_rejected` | INVARIANT | per-prog_fd_array_get_ptr: BPF_PROG_TYPE_EXT or is_extended → ERR. |
| `poke_run_under_mutex` | INVARIANT | per-prog_array_map_poke_run: mutex_is_locked(&aux.poke_mutex). |
| `mmap_skips_header` | INVARIANT | per-array_map_mmap: pgoff offset == PAGE_ALIGN(sizeof(bpf_array)) >> PAGE_SHIFT. |

### Layer 2: TLA+

`kernel/bpf/arraymap.tla`:
- Per-create + per-lookup + per-update + per-fd-array-update (PROG_ARRAY xchg + poke_run) + per-PERF_EVENT_ARRAY release-on-fdput + per-bpf_iter + per-free.
- Properties:
  - `safety_index_in_range` — per-lookup / update: post-mask index < max_entries.
  - `safety_no_grow` — per-update: max_entries fixed at create; OOB index → -E2BIG (update) or NULL (lookup).
  - `safety_no_per_slot_delete` — per-ARRAY / PERCPU_ARRAY: map_delete_elem returns -EINVAL.
  - `safety_prog_array_no_self_loop` — per-PROG_ARRAY update: BPF_PROG_TYPE_EXT and is_extended programs rejected — no tail-call into freplace.
  - `safety_poke_under_mutex` — per-PROG_ARRAY update with poke_run: xchg and bpf_arch_poke_desc_update both under poke_mutex.
  - `safety_perf_release_drops_only_owned` — per-PERF_EVENT_ARRAY release: only slots with matching map_file dropped (unless PRESERVE_ELEMS).
  - `safety_inner_map_meta_consistent` — per-ARRAY_OF_MAPS: inner_map_meta released only at outer-map free.
  - `liveness_per_update_terminates` — per-array_map_update_elem / bpf_fd_array_map_update_elem: bounded by single bucket; returns 0 / -EINVAL / -E2BIG / -EEXIST in O(1).
  - `liveness_per_clear_completes` — per-prog_array_map_clear: schedule_work eventually runs bpf_fd_array_map_clear; bpf_map_put paired with bpf_map_inc.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Array::alloc` post: index_mask + 1 ≥ max_entries ∧ elem_size == round_up(value_size, 8) | `Array::alloc` |
| `Array::map_lookup_elem` post: returns NULL ∨ pointer within array.value range | `Array::map_lookup_elem` |
| `Array::percpu_map_lookup_elem` post: returns NULL ∨ this_cpu_ptr(pptrs[masked_index]) | `Array::percpu_map_lookup_elem` |
| `Array::map_update_elem` post: bytes [val..val+value_size) updated; bpf_obj_free_fields called | `Array::map_update_elem` |
| `Array::map_delete_elem` post: returns -EINVAL | `Array::map_delete_elem` |
| `Array::bpf_fd_map_update_elem` post: array.ptrs[index] == new_ptr; old_ptr put via map_fd_put_ptr | `Array::bpf_fd_map_update_elem` |
| `Array::prog_poke_run` post: every tracked stable BPF_POKE_REASON_TAIL_CALL poke matching (map, key) updated | `Array::prog_poke_run` |
| `Array::perf_event_release` post: all slots with matching map_file cleared (unless PRESERVE_ELEMS) | `Array::perf_event_release` |
| `Array::array_of_map_lookup_elem` post: returns NULL ∨ READ_ONCE(*inner_map_slot) | `Array::array_of_map_lookup_elem` |
| `Array::free` post: all slots freed (special fields + percpu); backing area freed | `Array::free` |

### Layer 4: Verus/Creusot functional

`Per-create: array_map_alloc_check → array_map_alloc → bpf_array_alloc_percpu (if percpu).` `Per-lookup (RCU-only): index bound + AND index_mask → array.value+elem_size*idx (or this_cpu_ptr(pptrs[idx])).` `Per-update (ARRAY/PERCPU_ARRAY): bound + AND index_mask → copy_map_value(_locked) + bpf_obj_free_fields.` `Per-update (fd-array): xchg(&ptrs[idx], new_ptr) bracketed by poke_mutex for PROG_ARRAY-with-JIT; old_ptr deferred-put via map_fd_put_ptr.` `Per-PERF_EVENT_ARRAY release: scan slots by map_file; drop unless PRESERVE_ELEMS.` Semantic equivalence: per-`Documentation/bpf/map_array.rst` + `Documentation/bpf/prog_cgroup.rst` (for cgroup-array semantics) + `tools/testing/selftests/bpf/test_maps.c` + `tools/testing/selftests/bpf/progs/test_tailcall*.c` + `tools/testing/selftests/bpf/progs/test_perf_buffer.c` + `tools/testing/selftests/bpf/progs/test_global_data.c`.

### hardening

(Inherits row-1 features from `kernel/bpf/00-overview.md` § Hardening.)

BPF-arraymap reinforcement:

- **Per-AND-mask Spectre-v1 mitigation** — defense against per-OOB-speculation: `index & index_mask` is the canonical sanitizer; only `bpf_bypass_spec_v1`-privileged callers see raw `max_entries-1` mask (still bounded).
- **Per-roundup_pow_of_two index_mask** — defense against per-mask-aliasing: when !bypass_spec_v1 the effective `max_entries` is itself rounded up so AND-mask is sufficient (no separate JGE branch can be bypassed).
- **Per-key_size == 4 enforced** — defense against per-out-of-band-key access: u32-only.
- **Per-max_entries fixed at create** — defense against per-grow-during-prog-run; updates never allocate or grow.
- **Per-delete returns -EINVAL** — defense against per-prog-induced-NULL-slot races (PROG_ARRAY / PERF_EVENT_ARRAY / CGROUP_ARRAY / ARRAY_OF_MAPS use the explicit fd-flavor delete that atomic-`xchg`'s NULL with deferred put).
- **Per-PROG_ARRAY rejects BPF_PROG_TYPE_EXT** — defense against per-freplace-tail-call infinite loop.
- **Per-PROG_ARRAY tracks prog_array_member_cnt under ext_mutex** — defense against per-attach freplace-on-tail-callee race.
- **Per-poke_mutex around xchg + arch_poke_desc_update** — defense against per-JIT-text-race: tail-call call-site rewrite serialized with slot swap; readers see RCU-stable program until poke commits.
- **Per-tailcall_target_stable guard** — defense against per-partial-JIT-patching: only stable pokes rewritten.
- **Per-perf_event_get + perf_event_read_local probe** — defense against per-mismatched-perf-event-type insertion (event must support in-kernel read).
- **Per-map_file-scoped release for PERF_EVENT_ARRAY** — defense against per-cross-fd slot leak: closing the BPF map fd only drops slots whose `map_file` matches.
- **Per-BPF_F_PRESERVE_ELEMS opt-in** — defense against per-required-multi-fd PERF_EVENT_ARRAY producer flows; release-time clear suppressed; explicit `bpf_fd_array_map_clear(need_defer=false)` runs at map_free.
- **Per-cgroup_get_from_fd + cgroup_put** — defense against per-stale-cgroup-pointer: cgroup ref pinned for slot lifetime; cgroup_put release deferred through RCU grace.
- **Per-ARRAY_OF_MAPS inner_map_meta fdget/fdput-protected** — defense against per-meta-template-race with verifier inner-map type-check.
- **Per-BPF_F_MMAPABLE limited to ARRAY** — defense against per-mmap-of-non-flat-storage (percpu pointers, fd slots) which would expose kernel pointers to userspace.
- **Per-array_map_get_hash sha256 over value region** — defense against per-mmap-tamper detection (BPF-CO-RE freeze): user-visible snapshot integrity hash recorded in `map.sha`.
- **Per-BPF_F_LOCK requires BTF spin_lock field** — defense against per-locked-update on a map without a `struct bpf_spin_lock` element.
- **Per-RCU bpf_event_entry_free** — defense against per-concurrent-walk UAF in `perf_event_fd_array_release` (call_rcu deferred fput + kfree).
- **Per-deferred prog_array_map_clear** — defense against per-cgroup-bpf-destroy-deadlock: `schedule_work` decouples release from `cgroup_bpf_destroy_wq`-style synchronous paths.

