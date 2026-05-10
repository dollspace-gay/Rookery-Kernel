# Tier-3: kernel/bpf/hashtab.c — BPF hashmap (HASH / LRU_HASH / PERCPU_*)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/bpf/00-overview.md
upstream-paths:
  - kernel/bpf/hashtab.c (~2741 lines)
  - kernel/bpf/bpf_lru_list.c
  - kernel/bpf/percpu_freelist.c
  - kernel/bpf/map_in_map.c (for HASH_OF_MAPS)
  - include/linux/bpf.h (struct bpf_map, struct bpf_map_ops)
  - include/uapi/linux/bpf.h (BPF_F_NO_PREALLOC, BPF_F_NO_COMMON_LRU, BPF_F_ZERO_SEED, BPF_F_LOCK, BPF_F_CPU, BPF_F_ALL_CPUS)
-->

## Summary

The **BPF hashmap** is the workhorse keyed-store for eBPF: every program type that needs key-value lookup (per-flow stats, per-pid counters, conntrack-mirror, sk_storage backing, syscall classifiers) uses one of its five flavors — `BPF_MAP_TYPE_HASH`, `_PERCPU_HASH`, `_LRU_HASH`, `_LRU_PERCPU_HASH`, `_HASH_OF_MAPS`. Per-map: `struct bpf_htab` wraps a base `struct bpf_map`, a power-of-two array of `struct bucket` (each holding an `hlist_nulls_head` + `rqspinlock_t`), a `bpf_mem_alloc` for dynamic mode or a `pcpu_freelist`/`bpf_lru` for preallocated mode, an `extra_elems` percpu-pointer for in-place update of preallocated maps, plus an element counter (`atomic_t` or `percpu_counter` per `use_percpu_counter`) and a `hashrnd`. Per-element: `struct htab_elem` carries an `hlist_nulls_node` (union'd with `pcpu_freelist_node` for preallocated maps), an optional `lru_node` or `ptr_to_pptr` for percpu mode, a precomputed `hash`, and a flexible `key[]` array followed (after `round_up(key_size, 8)` padding) by either the value bytes inline or a percpu pointer. Per-lookup: `htab_map_hash(key, key_len, hashrnd)` (jhash2 if 4-byte aligned, jhash otherwise) → `__select_bucket` (mask with `n_buckets-1`) → `lookup_nulls_elem_raw` (RCU-walk `hlist_nulls_for_each_entry_rcu`, retry-on-nulls-mismatch). Per-update: bucket-locked path under `htab_lock_bucket` (rqspinlock) with `lookup_elem_raw`, `alloc_htab_elem` (from prealloc freelist + per-cpu extra_elems, or from `bpf_mem_cache_alloc`), `hlist_nulls_add_head_rcu`, then `hlist_nulls_del_rcu` of the old element and deferred free. Per-LRU: `prealloc_lru_pop` evicts the oldest unreferenced element via `bpf_lru_pop_free`; ref-marking on lookup via `bpf_lru_node_set_ref`. Per-percpu: a percpu-allocated value buffer per element; copy-in via `pcpu_copy_value` honoring `BPF_F_CPU` (write to one CPU) and `BPF_F_ALL_CPUS` (broadcast to all). Per-create flags: `BPF_F_NO_PREALLOC` (dynamic mem-alloc), `BPF_F_NO_COMMON_LRU` (per-CPU LRU lists), `BPF_F_ZERO_SEED` (require CAP_SYS_ADMIN — disables anti-collision), `BPF_F_NUMA_NODE` (numa-pinned), `BPF_F_ACCESS_MASK`. Critical for: Cilium/Calico/Katran dataplane keyed maps, bpftrace per-pid aggregation, conntrack-bpf, sk_storage, the verifier's `map_gen_lookup` inlining path, and the resilience of all of these against DoS via hash-collision flooding.

This Tier-3 covers `kernel/bpf/hashtab.c` (~2741 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct bpf_htab` | per-map state | `BpfHtab` |
| `struct bucket` | per-bucket lock + hlist head | `Bucket` |
| `struct htab_elem` | per-element node + key + value-or-pptr | `HtabElem` |
| `struct htab_btf_record` | per-mem-alloc dtor BTF descriptor | `HtabBtfRecord` |
| `htab_map_alloc()` | per-create main | `Htab::alloc` |
| `htab_map_alloc_check()` | per-create attr validation | `Htab::alloc_check` |
| `htab_map_free()` | per-destroy | `Htab::free` |
| `htab_map_free_internal_structs()` / `htab_free_malloced_internal_structs()` / `htab_free_prealloced_internal_structs()` / `htab_free_prealloced_fields()` | per-uref-drop cleanup | `Htab::free_internal_structs` |
| `htab_init_buckets()` | per-create bucket init | `Htab::init_buckets` |
| `htab_lock_bucket()` / `htab_unlock_bucket()` | per-bucket rqspinlock | `Htab::lock_bucket` / `unlock_bucket` |
| `htab_map_hash()` | per-key hash (jhash2/jhash) | `Htab::map_hash` |
| `__select_bucket()` / `select_bucket()` | per-hash → bucket | `Htab::select_bucket` |
| `lookup_elem_raw()` / `lookup_nulls_elem_raw()` | per-bucket walk (locked / RCU) | `Htab::lookup_elem_raw` / `lookup_nulls_elem_raw` |
| `__htab_map_lookup_elem()` / `htab_map_lookup_elem()` | per-bpf-prog lookup | `Htab::map_lookup_elem_inner` / `map_lookup_elem` |
| `__htab_lru_map_lookup_elem()` / `htab_lru_map_lookup_elem()` / `htab_lru_map_lookup_elem_sys()` | per-LRU lookup (mark / no-mark) | `Htab::lru_map_lookup_elem` |
| `htab_percpu_map_lookup_elem()` / `htab_lru_percpu_map_lookup_elem()` | per-percpu lookup (this_cpu_ptr) | `Htab::percpu_map_lookup_elem` |
| `htab_percpu_map_lookup_percpu_elem()` / `htab_lru_percpu_map_lookup_percpu_elem()` | per-arbitrary-CPU percpu lookup | `Htab::percpu_map_lookup_percpu_elem` |
| `htab_map_gen_lookup()` / `htab_lru_map_gen_lookup()` / `htab_percpu_map_gen_lookup()` | per-verifier inlining bytecode | `Htab::map_gen_lookup` |
| `htab_map_update_elem()` | per-bpf-prog update | `Htab::map_update_elem` |
| `htab_lru_map_update_elem()` | per-LRU update | `Htab::lru_map_update_elem` |
| `htab_map_update_elem_in_place()` | per-syscall in-place update | `Htab::map_update_elem_in_place` |
| `__htab_lru_percpu_map_update_elem()` | per-LRU percpu update | `Htab::lru_percpu_map_update_elem_inner` |
| `htab_percpu_map_update_elem()` / `htab_lru_percpu_map_update_elem()` | per-percpu update entry | `Htab::percpu_map_update_elem` |
| `htab_map_delete_elem()` / `htab_lru_map_delete_elem()` | per-delete | `Htab::map_delete_elem` / `lru_map_delete_elem` |
| `htab_map_get_next_key()` | per-iter | `Htab::map_get_next_key` |
| `htab_map_lookup_and_delete_elem()` / `htab_percpu_map_lookup_and_delete_elem()` / `htab_lru_map_lookup_and_delete_elem()` / `htab_lru_percpu_map_lookup_and_delete_elem()` | per-lookup-and-pop | `Htab::map_lookup_and_delete_elem` |
| `__htab_map_lookup_and_delete_elem()` | per-shared lookup-and-delete | `Htab::map_lookup_and_delete_elem_inner` |
| `alloc_htab_elem()` | per-element alloc (prealloc-pop / mem-cache) | `Htab::alloc_elem` |
| `htab_elem_free()` / `free_htab_elem()` / `htab_put_fd_value()` | per-element free + fd-map put | `Htab::elem_free` / `free_elem` / `put_fd_value` |
| `htab_lru_push_free()` | per-LRU push-free | `Htab::lru_push_free` |
| `prealloc_init()` / `prealloc_destroy()` / `prealloc_lru_pop()` | per-prealloc lifecycle | `Htab::prealloc_init` / `destroy` / `lru_pop` |
| `alloc_extra_elems()` | per-CPU spare elem for in-place update | `Htab::alloc_extra_elems` |
| `htab_free_elems()` | per-prealloc bulk free | `Htab::free_elems` |
| `check_flags()` | per-{BPF_NOEXIST, BPF_EXIST, BPF_F_LOCK} | `Htab::check_flags` |
| `pcpu_copy_value()` / `pcpu_init_value()` | per-percpu copy/init | `Htab::pcpu_copy_value` / `init_value` |
| `is_map_full()` / `inc_elem_count()` / `dec_elem_count()` | per-counter accounting | `Htab::is_map_full` / `inc_elem_count` / `dec_elem_count` |
| `check_and_free_fields()` | per-bpf-obj-free special fields | `Htab::check_and_free_fields` |
| `htab_lru_map_delete_node()` | per-LRU eviction callback | `Htab::lru_map_delete_node` |
| `delete_all_elements()` | per-map-free bulk delete | `Htab::delete_all_elements` |
| `htab_map_seq_show_elem()` / `htab_percpu_map_seq_show_elem()` | per-bpffs seq | `Htab::seq_show_elem` |
| `htab_map_check_btf()` / `htab_set_dtor()` / `htab_mem_dtor()` / `htab_pcpu_mem_dtor()` / `htab_dtor_ctx_free()` | per-BTF dtor wiring | `Htab::map_check_btf` / `set_dtor` |
| `htab_map_mem_usage()` | per-show-fdinfo | `Htab::map_mem_usage` |
| `bpf_for_each_hash_elem()` | per-bpf-callback iteration | `Htab::bpf_for_each_hash_elem` |
| `bpf_iter_init_hash_map()` / `_fini` / `bpf_hash_map_seq_*` | per-bpf_iter | `Htab::bpf_iter_*` |
| `bpf_percpu_hash_copy()` / `bpf_percpu_hash_update()` | per-syscall percpu lookup/update | `Htab::bpf_percpu_hash_copy` / `update` |
| `bpf_fd_htab_map_lookup_elem()` / `bpf_fd_htab_map_update_elem()` / `fd_htab_map_alloc_check()` / `fd_htab_map_free()` | per-HASH_OF_MAPS support | `Htab::fd_htab_*` |
| `htab_of_map_alloc()` / `htab_of_map_free()` / `htab_of_map_lookup_elem()` / `htab_of_map_gen_lookup()` | per-HASH_OF_MAPS map_ops | `Htab::htab_of_map_*` |
| `htab_map_ops` / `htab_lru_map_ops` / `htab_percpu_map_ops` / `htab_lru_percpu_map_ops` / `htab_of_maps_map_ops` | per-flavor map_ops vtable | shared |

## Compatibility contract

REQ-1: struct bpf_htab (per-map):
- map: `struct bpf_map` (base; map_type, key_size, value_size, max_entries, map_flags, numa_node, record, ...).
- ma: `struct bpf_mem_alloc` for element backing (non-prealloc).
- pcpu_ma: `struct bpf_mem_alloc` for percpu value backing (non-prealloc percpu maps).
- buckets: `*Bucket` array of n_buckets entries.
- elems: prealloc'd elem block (NULL in non-prealloc mode).
- freelist (union): `pcpu_freelist` (non-LRU prealloc) or `bpf_lru` (LRU prealloc).
- extra_elems: `percpu htab_elem *` — per-CPU spare slot to avoid freelist push/pop during in-place update of a prealloc map.
- pcount: `percpu_counter` for element count when `use_percpu_counter`.
- count: `atomic_t` for element count otherwise.
- use_percpu_counter: chosen at create if `max_entries / 2 > num_online_cpus() * PERCPU_COUNTER_BATCH` (32).
- n_buckets: roundup_pow_of_two(max_entries).
- elem_size: sizeof(htab_elem) + round_up(key_size, 8) + (percpu ? sizeof(void*) : round_up(value_size, 8)).
- hashrnd: get_random_u32() (or 0 if BPF_F_ZERO_SEED).

REQ-2: struct bucket (per-bucket):
- head: `struct hlist_nulls_head`.
- raw_lock: `rqspinlock_t` (resilient queued spinlock — may fail and return ETIMEDOUT on contention loop).

REQ-3: struct htab_elem (per-element):
- union { hash_node: hlist_nulls_node | { padding: void*; fnode: pcpu_freelist_node | batch_flink: htab_elem* } } — first 16 bytes overlap so prealloc'd elements can be on freelist before being on a bucket.
- union { ptr_to_pptr: void* | lru_node: bpf_lru_node }.
- hash: u32 — precomputed.
- key: char[] __aligned(8) — flexible array; followed by value or pptr after round_up(key_size, 8).
- BUILD_BUG_ON: `offsetof(htab_elem, fnode.next) == offsetof(htab_elem, hash_node.pprev)` — guarantees freelist→hlist transition safe.

REQ-4: htab_map_alloc_check(attr):
- /* Per-flag derive */
- percpu = (map_type ∈ {PERCPU_HASH, LRU_PERCPU_HASH}).
- lru = (map_type ∈ {LRU_HASH, LRU_PERCPU_HASH}).
- percpu_lru = (map_flags & BPF_F_NO_COMMON_LRU).
- prealloc = !(map_flags & BPF_F_NO_PREALLOC).
- zero_seed = (map_flags & BPF_F_ZERO_SEED).
- /* BUILD_BUG_ON */
- BUILD_BUG_ON(offsetof(htab_elem, fnode.next) != offsetof(htab_elem, hash_node.pprev)).
- /* Per-cap */
- if zero_seed ∧ !capable(CAP_SYS_ADMIN): return -EPERM /* anti-DoS */.
- /* Per-flag-mask */
- if (map_flags & ~HTAB_CREATE_FLAG_MASK) ∨ !bpf_map_flags_access_ok(map_flags): return -EINVAL.
- /* Per-LRU constraints */
- if !lru ∧ percpu_lru: return -EINVAL.
- if lru ∧ !prealloc: return -ENOTSUPP /* LRU requires prealloc */.
- if numa_node != NUMA_NO_NODE ∧ (percpu ∨ percpu_lru): return -EINVAL.
- /* Per-attr-sanity */
- if max_entries == 0 ∨ key_size == 0 ∨ value_size == 0: return -EINVAL.
- if (u64)key_size + value_size ≥ KMALLOC_MAX_SIZE - sizeof(htab_elem): return -E2BIG.
- if percpu ∧ round_up(value_size, 8) > PCPU_MIN_UNIT_SIZE: return -E2BIG.
- return 0.

REQ-5: htab_map_alloc(attr):
- htab = bpf_map_area_alloc(sizeof(*htab), NUMA_NO_NODE).
- bpf_map_init_from_attr(&htab.map, attr).
- if percpu_lru: max_entries rounded up to num_possible_cpus().
- if max_entries > 1<<31: -E2BIG.
- n_buckets = roundup_pow_of_two(max_entries).
- elem_size = sizeof(htab_elem) + round_up(key_size, 8) + (percpu ? sizeof(void*) : round_up(value_size, 8)).
- bpf_map_init_elem_count(&htab.map).
- buckets = bpf_map_area_alloc(n_buckets * sizeof(Bucket), numa_node).
- hashrnd = (BPF_F_ZERO_SEED ? 0 : get_random_u32()).
- htab_init_buckets — per-bucket INIT_HLIST_NULLS_HEAD(i) + raw_res_spin_lock_init.
- use_percpu_counter = (max_entries/2 > num_online_cpus() * 32).
- if use_percpu_counter: percpu_counter_init(&pcount, 0, GFP_KERNEL).
- if prealloc: prealloc_init; if has_extra_elems: alloc_extra_elems.
- else: bpf_mem_alloc_init(&ma, elem_size, false); if percpu: bpf_mem_alloc_init(&pcpu_ma, round_up(value_size, 8), true).
- return &htab.map.

REQ-6: prealloc_init(htab):
- num_entries = max_entries + (has_extra_elems ? num_possible_cpus() : 0).
- elems = bpf_map_area_alloc(elem_size * num_entries, numa_node).
- if percpu: for-each-elem alloc per-cpu value pptr (bpf_map_alloc_percpu); htab_elem_set_ptr.
- if lru: bpf_lru_init(&lru, NO_COMMON_LRU?, offset, htab_lru_map_delete_node, htab).
- else: pcpu_freelist_init(&freelist).
- if lru: bpf_lru_populate; else: pcpu_freelist_populate.
- return 0.

REQ-7: alloc_extra_elems(htab):
- pptr = bpf_map_alloc_percpu(htab, sizeof(htab_elem *), 8, GFP_USER|__GFP_NOWARN).
- for_each_possible_cpu(cpu): l = pcpu_freelist_pop(&freelist); *per_cpu_ptr(pptr, cpu) = l (extra elem reserved per CPU).
- htab.extra_elems = pptr.
- return 0.

REQ-8: htab_map_hash(key, key_len, hashrnd):
- if likely(key_len % 4 == 0): return jhash2(key, key_len / 4, hashrnd).
- return jhash(key, key_len, hashrnd).

REQ-9: __select_bucket(htab, hash) / select_bucket:
- return &htab.buckets[hash & (n_buckets - 1)].

REQ-10: lookup_elem_raw(head, hash, key, key_size) — bucket-lock held:
- hlist_nulls_for_each_entry_rcu(l, n, head, hash_node):
  - if l.hash == hash ∧ !memcmp(&l.key, key, key_size): return l.
- return NULL.

REQ-11: lookup_nulls_elem_raw(head, hash, key, key_size, n_buckets) — RCU only:
- again:
- hlist_nulls_for_each_entry_rcu(...): match → return l.
- if get_nulls_value(n) != (hash & (n_buckets - 1)): goto again /* element moved bucket */.
- return NULL.

REQ-12: htab_map_lookup_elem(map, key) (BPF_MAP_TYPE_HASH):
- WARN_ON_ONCE(!bpf_rcu_lock_held()).
- hash = htab_map_hash(key, key_size, hashrnd).
- head = select_bucket(htab, hash).
- l = lookup_nulls_elem_raw(head, hash, key, key_size, n_buckets).
- return l ? htab_elem_value(l, key_size) : NULL.

REQ-13: htab_map_gen_lookup(map, insn_buf):
- *insn++ = BPF_EMIT_CALL(__htab_map_lookup_elem).
- *insn++ = BPF_JMP_IMM(BPF_JEQ, BPF_REG_0, 0, 1) /* skip if NULL */.
- *insn++ = BPF_ALU64_IMM(BPF_ADD, BPF_REG_0, offsetof(htab_elem, key) + round_up(key_size, 8)) /* advance to value */.
- return insn - insn_buf (= 3).
- (percpu variant additionally emits BPF_LDX_MEM + BPF_MOV64_PERCPU_REG; returns 5.)

REQ-14: htab_map_update_elem(map, key, value, map_flags) (BPF_MAP_TYPE_HASH):
- if map_flags & ~BPF_F_LOCK > BPF_EXIST: return -EINVAL.
- /* Lock-flag fast path: in-place update via element spinlock without bucket lock */
- if map_flags & BPF_F_LOCK:
  - if !btf_record_has_field(record, BPF_SPIN_LOCK): return -EINVAL.
  - l_old = lookup_nulls_elem_raw(head, hash, key, key_size, n_buckets).
  - check_flags(l_old, map_flags).
  - if l_old: copy_map_value_locked(map, htab_elem_value(l_old, key_size), value, false); return 0.
  - /* fall through */
- htab_lock_bucket(b, &flags); /* may return -EBUSY on rqspinlock timeout */
- l_old = lookup_elem_raw(head, hash, key, key_size).
- check_flags(l_old, map_flags) — BPF_NOEXIST and l_old → -EEXIST; BPF_EXIST and !l_old → -ENOENT.
- if l_old ∧ (map_flags & BPF_F_LOCK): copy_map_value_locked + unlock + return 0 (rare race).
- l_new = alloc_htab_elem(htab, key, value, key_size, hash, false, false, l_old, map_flags).
- hlist_nulls_add_head_rcu(&l_new.hash_node, head).
- if l_old:
  - hlist_nulls_del_rcu(&l_old.hash_node).
  - if htab_is_prealloc(htab): check_and_free_fields(htab, l_old).
- htab_unlock_bucket(b, flags).
- if l_old ∧ !htab_is_prealloc(htab): free_htab_elem(htab, l_old).
- return 0.

REQ-15: alloc_htab_elem(htab, key, value, key_size, hash, percpu, onallcpus, old_elem, map_flags):
- /* Prealloc path */
- if prealloc:
  - if old_elem: l_new = *this_cpu_ptr(extra_elems); *this_cpu_ptr(extra_elems) = old_elem /* swap */.
  - else: l = __pcpu_freelist_pop(&freelist); if !l: return -E2BIG; l_new = container_of(l, htab_elem, fnode); bpf_map_inc_elem_count.
- /* Non-prealloc path */
- else:
  - if is_map_full(htab) ∧ !old_elem: return -E2BIG.
  - inc_elem_count(htab).
  - l_new = bpf_mem_cache_alloc(&ma); if !l_new: dec_count; return -ENOMEM.
- memcpy(l_new.key, key, key_size).
- if percpu:
  - if prealloc: pptr = htab_elem_get_ptr(l_new, key_size).
  - else: ptr = bpf_mem_cache_alloc(&pcpu_ma); l_new.ptr_to_pptr = ptr; pptr = *(void __percpu **)ptr.
  - pcpu_init_value(htab, pptr, value, onallcpus, map_flags).
  - if !prealloc: htab_elem_set_ptr(l_new, key_size, pptr).
- else if fd_htab_map_needs_adjust(htab): memcpy 8-rounded value (HASH_OF_MAPS).
- else if map_flags & BPF_F_LOCK: copy_map_value_locked(map, elem_value, value, false).
- else: copy_map_value(map, elem_value, value).
- l_new.hash = hash.
- return l_new.

REQ-16: htab_lru_map_update_elem(map, key, value, map_flags):
- /* Alloc before lock (LRU pop may need bucket lock for eviction) */
- l_new = prealloc_lru_pop(htab, key, hash).
- if !l_new: return -ENOMEM.
- copy_map_value(map, htab_elem_value(l_new, key_size), value).
- htab_lock_bucket(b, &flags).
- l_old = lookup_elem_raw(head, hash, key, key_size).
- check_flags(htab, l_old, map_flags).
- hlist_nulls_add_head_rcu(&l_new.hash_node, head).
- if l_old: bpf_lru_node_set_ref(&l_new.lru_node); hlist_nulls_del_rcu(&l_old.hash_node).
- htab_unlock_bucket(b, flags).
- if err: htab_lru_push_free(htab, l_new); else if l_old: htab_lru_push_free(htab, l_old).
- return ret.

REQ-17: htab_map_delete_elem(map, key):
- htab_lock_bucket(b, &flags).
- l = lookup_elem_raw(head, hash, key, key_size).
- if l: hlist_nulls_del_rcu(&l.hash_node) else ret = -ENOENT.
- htab_unlock_bucket(b, flags).
- if l: free_htab_elem(htab, l).
- return ret.

REQ-18: free_htab_elem(htab, l):
- htab_put_fd_value(htab, l) — for HASH_OF_MAPS: map_fd_put_ptr.
- if htab_is_prealloc(htab): bpf_map_dec_elem_count; check_and_free_fields; pcpu_freelist_push(&freelist, &l.fnode).
- else: dec_elem_count; htab_elem_free(htab, l) — bpf_mem_cache_free(&ma); for percpu also bpf_mem_cache_free(&pcpu_ma, l.ptr_to_pptr).

REQ-19: htab_lru_push_free(htab, elem):
- check_and_free_fields(htab, elem).
- bpf_map_dec_elem_count(&htab.map).
- bpf_lru_push_free(&htab.lru, &elem.lru_node).

REQ-20: htab_lru_map_delete_node(arg, node) — LRU eviction callback:
- elem = container_of(node, htab_elem, lru_node).
- /* Re-derive bucket; lock; unlink-if-still-present */
- htab_lock_bucket(b, &flags).
- hlist_nulls_for_each_entry walking matching head/hash/key found-l == elem → hlist_nulls_del_rcu; ret = true.
- htab_unlock_bucket(b, flags).
- return ret /* tells LRU framework to reclaim slot */.

REQ-21: htab_map_get_next_key(map, key, next_key):
- if !key: next_key = first non-empty bucket's first element's key.
- else: find element matching key → advance within bucket; if end → walk forward through buckets.
- return 0 or -ENOENT.

REQ-22: __htab_map_lookup_and_delete_elem(map, key, value, is_lru_map, is_percpu, flags):
- htab_lock_bucket; l = lookup_elem_raw; if !l: -ENOENT.
- if is_percpu: per-CPU copy out (round_up(value_size, 8) per CPU) into value buffer; check_and_init_map_value.
- else: copy_map_value(_locked) into value; check_and_init_map_value.
- hlist_nulls_del_rcu(&l.hash_node).
- htab_unlock_bucket.
- if l: is_lru_map ? htab_lru_push_free : free_htab_elem.
- return 0.

REQ-23: pcpu_copy_value / pcpu_init_value (percpu maps):
- pcpu_copy_value: if !onallcpus → write only to this_cpu_ptr(pptr); if onallcpus ∧ BPF_F_CPU → write to per_cpu_ptr(pptr, map_flags >> 32); if onallcpus ∧ BPF_F_ALL_CPUS → broadcast; otherwise copy in stride `round_up(value_size, 8) * cpu`.
- pcpu_init_value: zero-fill non-current CPUs (since elem was preallocated and special fields cannot be re-initialized).

REQ-24: bpf_percpu_hash_copy / bpf_percpu_hash_update (syscall path):
- copy: rcu_read_lock; l = __htab_map_lookup_elem; per-cpu copy_map_value into value (BPF_F_CPU honored); rcu_read_unlock.
- update: thin wrapper to htab_map_update_elem_in_place(map, key, value, map_flags, percpu=true, onallcpus=true).

REQ-25: htab_map_free(map):
- if !prealloc: delete_all_elements(htab) /* hlist walk + bpf_mem_cache_free under no-rcu (called from workqueue) */.
- else: htab_free_prealloced_fields; prealloc_destroy.
- bpf_map_free_elem_count.
- free_percpu(extra_elems).
- bpf_map_area_free(buckets).
- bpf_mem_alloc_destroy(&pcpu_ma); bpf_mem_alloc_destroy(&ma).
- if use_percpu_counter: percpu_counter_destroy(&pcount).
- bpf_map_area_free(htab).

REQ-26: check_flags(htab, l_old, map_flags):
- BPF_NOEXIST ∧ l_old: return -EEXIST.
- BPF_EXIST ∧ !l_old: return -ENOENT.
- else: return 0.

REQ-27: bpf_lru push/pop (LRU flavors):
- prealloc_lru_pop: bpf_lru_pop_free(&htab.lru, hash) — may evict the LRU-tail via htab_lru_map_delete_node which re-locks bucket.
- bpf_lru_node_set_ref(&l.lru_node) — sets ref bit so element survives one more LRU pass.
- htab_lru_map_lookup_elem_sys: lookup without ref-set (used by syscall map-walk to avoid skewing eviction).

REQ-28: is_map_full / use_percpu_counter accounting:
- is_map_full: use_percpu_counter ? __percpu_counter_compare(pcount, max_entries, PERCPU_COUNTER_BATCH) ≥ 0 : atomic_read(count) ≥ max_entries.
- inc_elem_count / dec_elem_count: bpf_map_inc/dec_elem_count + percpu_counter_add_batch (or atomic_inc/dec).

REQ-29: HASH_OF_MAPS support:
- htab_of_map_alloc → fd_htab_map_alloc_check + htab_map_alloc with adjusted value_size (= sizeof(void *) for inner-map fd-tracking).
- htab_of_map_free → fd_htab_map_free walks elements calling map_fd_put_ptr on each inner-map ref.
- htab_of_map_lookup_elem returns inner-map pointer; htab_of_map_gen_lookup inlines the deref to inner-map.

REQ-30: Verifier inlining via map_gen_lookup:
- htab_map_ops.map_gen_lookup = htab_map_gen_lookup → 3 BPF insns inline the lookup.
- htab_percpu_map_ops.map_gen_lookup = htab_percpu_map_gen_lookup → 5 insns (calls __htab_map_lookup_elem; advances; loads percpu ptr).
- htab_lru_map_ops.map_gen_lookup = htab_lru_map_gen_lookup (with ref-set).

## Acceptance Criteria

- [ ] AC-1: htab_map_alloc(max_entries=128, key=4, value=8) succeeds; n_buckets == 128; elem_size = sizeof(htab_elem)+8+8.
- [ ] AC-2: htab_map_alloc with key_size=0 ∨ value_size=0 ∨ max_entries=0 → -EINVAL.
- [ ] AC-3: BPF_F_ZERO_SEED without CAP_SYS_ADMIN → -EPERM.
- [ ] AC-4: BPF_F_NO_PREALLOC + BPF_MAP_TYPE_LRU_HASH → -ENOTSUPP (LRU requires prealloc).
- [ ] AC-5: htab_map_update_elem with BPF_NOEXIST on existing key → -EEXIST.
- [ ] AC-6: htab_map_update_elem with BPF_EXIST on missing key → -ENOENT.
- [ ] AC-7: htab_map_lookup_elem after update finds the value (RCU-walk).
- [ ] AC-8: Non-prealloc map at max_entries with no replacement → alloc_htab_elem returns -E2BIG.
- [ ] AC-9: htab_map_delete_elem removes element; subsequent lookup → NULL.
- [ ] AC-10: LRU map under pressure evicts oldest non-ref'd element via htab_lru_map_delete_node.
- [ ] AC-11: Percpu map update with BPF_F_CPU writes only the target CPU's slot.
- [ ] AC-12: Percpu map update with BPF_F_ALL_CPUS broadcasts to all CPUs.
- [ ] AC-13: htab_map_gen_lookup emits 3 insns; percpu variant emits 5.
- [ ] AC-14: htab_lock_bucket returns -EBUSY on rqspinlock timeout; caller propagates error.
- [ ] AC-15: Lookup-and-delete returns value and removes element atomically.

## Architecture

```
struct BpfHtab {
  map: BpfMap,
  ma: BpfMemAlloc,
  pcpu_ma: BpfMemAlloc,
  buckets: *Bucket,
  elems: *u8,                              // prealloc block, None if non-prealloc
  // union: freelist (non-LRU prealloc) or lru (LRU prealloc)
  freelist_or_lru: FreelistOrLru,
  extra_elems: PerCpu<*HtabElem>,
  pcount: PerCpuCounter,
  count: AtomicI32,
  use_percpu_counter: bool,
  n_buckets: u32,
  elem_size: u32,
  hashrnd: u32,
}

struct Bucket {
  head: HlistNullsHead,
  raw_lock: RqSpinlock,
}

struct HtabElem {
  // first 16B: union of hash_node | (padding + fnode/batch_flink)
  hash_node_or_fnode: HtabElemUnion,
  // next 16B: union of ptr_to_pptr | lru_node
  ptr_or_lru: HtabElemSecondUnion,
  hash: u32,
  key: [u8; 0],                            // flexible, aligned to 8
  // implicit: after round_up(key_size, 8): value bytes or pptr
}
```

`Htab::alloc(attr) -> Result<*BpfMap>`:
1. /* Validate flags */
2. Htab::alloc_check(attr)?.
3. htab = bpf_map_area_alloc(sizeof(BpfHtab), NUMA_NO_NODE).
4. bpf_map_init_from_attr(&htab.map, attr).
5. /* Per-LRU rounding */
6. if percpu_lru: max_entries = round_up(max_entries, num_possible_cpus).
7. if max_entries > 1<<31: -E2BIG.
8. n_buckets = roundup_pow_of_two(max_entries).
9. elem_size = sizeof(HtabElem) + round_up(key_size, 8) + (percpu ? sizeof(void*) : round_up(value_size, 8)).
10. /* Allocate buckets */
11. buckets = bpf_map_area_alloc(n_buckets * sizeof(Bucket), numa_node).
12. hashrnd = (BPF_F_ZERO_SEED ? 0 : get_random_u32()).
13. Htab::init_buckets(htab).
14. /* Counter strategy */
15. use_percpu_counter = (max_entries / 2 > num_online_cpus * 32).
16. if use_percpu_counter: percpu_counter_init(&pcount, 0, GFP_KERNEL).
17. /* Prealloc vs dynamic */
18. if prealloc:
    - Htab::prealloc_init(htab)?.
    - if htab_has_extra_elems: Htab::alloc_extra_elems(htab)?.
19. else:
    - bpf_mem_alloc_init(&ma, elem_size, false)?.
    - if percpu: bpf_mem_alloc_init(&pcpu_ma, round_up(value_size, 8), true)?.
20. return Ok(&htab.map).

`Htab::map_lookup_elem(map, key) -> Option<*u8>`:
1. WARN_ON_ONCE(!bpf_rcu_lock_held()).
2. hash = Htab::map_hash(key, key_size, htab.hashrnd).
3. head = Htab::select_bucket(htab, hash).
4. l = Htab::lookup_nulls_elem_raw(head, hash, key, key_size, n_buckets).
5. return l.map(|l| htab_elem_value(l, key_size)).

`Htab::lookup_nulls_elem_raw(head, hash, key, key_size, n_buckets) -> Option<*HtabElem>`:
1. again:
2. hlist_nulls_for_each_entry_rcu(l, n, head, hash_node):
   - if l.hash == hash ∧ memcmp(&l.key, key, key_size) == 0: return Some(l).
3. /* Hit nulls marker — verify we ended at our bucket's marker */
4. if get_nulls_value(n) != (hash & (n_buckets - 1)): goto again.
5. return None.

`Htab::map_update_elem(map, key, value, map_flags) -> Result<()>`:
1. if (map_flags & ~BPF_F_LOCK) > BPF_EXIST: return -EINVAL.
2. WARN_ON_ONCE(!bpf_rcu_lock_held()).
3. hash = Htab::map_hash(key, key_size, hashrnd); b = Htab::__select_bucket(htab, hash); head = &b.head.
4. /* BPF_F_LOCK fast path: locate via RCU + per-elem spinlock */
5. if map_flags & BPF_F_LOCK:
   - if !btf_record_has_field(record, BPF_SPIN_LOCK): return -EINVAL.
   - l_old = Htab::lookup_nulls_elem_raw(head, hash, key, key_size, n_buckets).
   - Htab::check_flags(htab, l_old, map_flags)?.
   - if l_old: copy_map_value_locked(map, htab_elem_value(l_old, key_size), value, false); return Ok(()).
6. /* Bucket-locked slow path */
7. Htab::lock_bucket(b, &flags)?.
8. l_old = Htab::lookup_elem_raw(head, hash, key, key_size).
9. let ret = Htab::check_flags(htab, l_old, map_flags); if Err(e) = ret { unlock; return Err(e); }.
10. /* Tail-race: lock-flag lost race to find existing */
11. if l_old ∧ (map_flags & BPF_F_LOCK):
    - copy_map_value_locked + unlock + return Ok(()).
12. l_new = Htab::alloc_elem(htab, key, value, key_size, hash, false, false, l_old, map_flags)?.
13. hlist_nulls_add_head_rcu(&l_new.hash_node, head).
14. if l_old:
    - hlist_nulls_del_rcu(&l_old.hash_node).
    - if htab_is_prealloc(htab): Htab::check_and_free_fields(htab, l_old).
15. Htab::unlock_bucket(b, flags).
16. if l_old ∧ !htab_is_prealloc(htab): Htab::free_elem(htab, l_old).
17. return Ok(()).

`Htab::alloc_elem(htab, key, value, key_size, hash, percpu, onallcpus, old_elem, map_flags) -> Result<*HtabElem>`:
1. /* Prealloc path */
2. if htab_is_prealloc(htab):
   - if old_elem:
     - pl_new = this_cpu_ptr(htab.extra_elems).
     - l_new = *pl_new.
     - *pl_new = old_elem /* swap: extra_elems now holds the previous element until next update */.
   - else:
     - l = __pcpu_freelist_pop(&htab.freelist).
     - if !l: return Err(-E2BIG).
     - l_new = container_of(l, HtabElem, fnode).
     - bpf_map_inc_elem_count(&htab.map).
3. /* Non-prealloc path */
4. else:
   - if Htab::is_map_full(htab) ∧ !old_elem: return Err(-E2BIG).
   - Htab::inc_elem_count(htab).
   - l_new = bpf_mem_cache_alloc(&htab.ma).
   - if !l_new: dec_count; return Err(-ENOMEM).
5. /* Copy key + value */
6. memcpy(l_new.key, key, key_size).
7. if percpu:
   - pptr = (prealloc ? htab_elem_get_ptr(l_new, key_size) : bpf_mem_cache_alloc(&htab.pcpu_ma)).
   - Htab::pcpu_init_value(htab, pptr, value, onallcpus, map_flags).
   - if !prealloc: htab_elem_set_ptr(l_new, key_size, pptr).
8. else if fd_htab_map_needs_adjust(htab): memcpy(htab_elem_value(l_new, key_size), value, round_up(value_size, 8)).
9. else if map_flags & BPF_F_LOCK: copy_map_value_locked(...).
10. else: copy_map_value(map, htab_elem_value(l_new, key_size), value).
11. l_new.hash = hash.
12. return Ok(l_new).

`Htab::map_delete_elem(map, key) -> Result<()>`:
1. WARN_ON_ONCE(!bpf_rcu_lock_held()).
2. hash = Htab::map_hash(...); b = __select_bucket; head = &b.head.
3. Htab::lock_bucket(b, &flags)?.
4. l = Htab::lookup_elem_raw(head, hash, key, key_size).
5. if l: hlist_nulls_del_rcu(&l.hash_node) else ret = Err(-ENOENT).
6. Htab::unlock_bucket(b, flags).
7. if l: Htab::free_elem(htab, l).
8. return ret.

`Htab::free_elem(htab, l)`:
1. Htab::put_fd_value(htab, l) /* HASH_OF_MAPS: drop inner-map ref */.
2. if htab_is_prealloc(htab):
   - bpf_map_dec_elem_count(&htab.map).
   - Htab::check_and_free_fields(htab, l).
   - pcpu_freelist_push(&htab.freelist, &l.fnode).
3. else:
   - Htab::dec_elem_count(htab).
   - Htab::elem_free(htab, l) → bpf_mem_cache_free(&htab.ma); if percpu_map: bpf_mem_cache_free(&htab.pcpu_ma, l.ptr_to_pptr).

`Htab::lru_map_update_elem(map, key, value, map_flags) -> Result<()>`:
1. /* Alloc before lock — LRU eviction may need a bucket lock */
2. l_new = Htab::prealloc_lru_pop(htab, key, hash).
3. if !l_new: return Err(-ENOMEM).
4. copy_map_value(&htab.map, htab_elem_value(l_new, key_size), value).
5. Htab::lock_bucket(b, &flags)?.
6. l_old = Htab::lookup_elem_raw(head, hash, key, key_size).
7. Htab::check_flags(htab, l_old, map_flags)?.
8. hlist_nulls_add_head_rcu(&l_new.hash_node, head).
9. if l_old: bpf_lru_node_set_ref(&l_new.lru_node); hlist_nulls_del_rcu(&l_old.hash_node).
10. Htab::unlock_bucket(b, flags).
11. on-err: Htab::lru_push_free(htab, l_new); else if l_old: Htab::lru_push_free(htab, l_old).
12. return ret.

`Htab::free(map)`:
1. if !htab_is_prealloc(htab): Htab::delete_all_elements(htab).
2. else: Htab::free_prealloced_fields(htab); Htab::prealloc_destroy(htab).
3. bpf_map_free_elem_count(map).
4. free_percpu(htab.extra_elems).
5. bpf_map_area_free(htab.buckets).
6. bpf_mem_alloc_destroy(&htab.pcpu_ma); bpf_mem_alloc_destroy(&htab.ma).
7. if htab.use_percpu_counter: percpu_counter_destroy(&htab.pcount).
8. bpf_map_area_free(htab).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `bucket_idx_in_range` | INVARIANT | per-select_bucket: hash & (n_buckets-1) < n_buckets. |
| `n_buckets_power_of_two` | INVARIANT | per-alloc: n_buckets is power of 2. |
| `elem_size_bounded` | INVARIANT | per-alloc: key_size + value_size + sizeof(htab_elem) < KMALLOC_MAX_SIZE. |
| `bucket_lock_held_for_modify` | INVARIANT | per-update_elem / delete_elem: bucket lock held over hlist mutation. |
| `rcu_held_for_lookup` | INVARIANT | per-map_lookup_elem: bpf_rcu_lock_held(). |
| `elem_count_bounded` | INVARIANT | per-alloc_htab_elem: count ≤ max_entries (non-prealloc + !old_elem). |
| `zero_seed_capable` | INVARIANT | per-alloc_check: BPF_F_ZERO_SEED ⟹ CAP_SYS_ADMIN. |
| `lru_requires_prealloc` | INVARIANT | per-alloc_check: LRU ∧ NO_PREALLOC ⟹ -ENOTSUPP. |
| `fnode_overlay_layout` | INVARIANT | per-BUILD_BUG_ON: offsetof(fnode.next) == offsetof(hash_node.pprev). |
| `nulls_check_consistent` | INVARIANT | per-lookup_nulls_elem_raw: retry iff get_nulls_value mismatch. |

### Layer 2: TLA+

`kernel/bpf/hashmap.tla`:
- Per-create + per-lookup + per-update + per-delete + per-iter + per-LRU-evict + per-prealloc-init/destroy.
- Properties:
  - `safety_no_dup_keys` — per-map: at most one element per (hash, key) pair on bucket list.
  - `safety_no_use_after_free` — per-element: freed only after bucket unlinks AND RCU grace period (or pcpu_freelist_push in prealloc).
  - `safety_lru_evicts_oldest` — per-LRU: htab_lru_map_delete_node selects head of LRU tail.
  - `safety_bucket_lock_exclusive` — per-bucket: at most one writer at a time under rqspinlock.
  - `safety_extra_elems_balance` — per-update with old_elem: extra_elems[cpu] reused symmetrically.
  - `liveness_per_lookup_terminates` — per-lookup: lookup_nulls retry bounded by element moves.
  - `liveness_per_update_progresses` — per-update: htab_lock_bucket returns 0 or -EBUSY in bounded time.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Htab::alloc` post: n_buckets is power of 2 ∧ elem_size valid | `Htab::alloc` |
| `Htab::map_lookup_elem` post: returns NULL ∨ element with matching (hash, key) | `Htab::map_lookup_elem` |
| `Htab::map_update_elem` post: bucket contains l_new; l_old (if any) unlinked | `Htab::map_update_elem` |
| `Htab::alloc_elem` post: l_new.hash == hash ∧ l_new.key[..key_size] == key | `Htab::alloc_elem` |
| `Htab::map_delete_elem` post: bucket no longer contains element matching key | `Htab::map_delete_elem` |
| `Htab::lru_map_update_elem` post: l_old (if any) on lru-free list; l_new linked | `Htab::lru_map_update_elem` |
| `Htab::free` post: all elements freed; ma + pcpu_ma destroyed; buckets freed | `Htab::free` |
| `Htab::map_get_next_key` post: next_key is lexicographic successor or -ENOENT | `Htab::map_get_next_key` |

### Layer 4: Verus/Creusot functional

`Per-create: htab_map_alloc → htab_init_buckets → prealloc_init/bpf_mem_alloc_init.` `Per-update: htab_map_hash → __select_bucket → htab_lock_bucket → lookup_elem_raw → alloc_htab_elem → hlist_nulls_add_head_rcu → optional hlist_nulls_del_rcu → htab_unlock_bucket → free_htab_elem.` `Per-lookup (RCU-only): htab_map_hash → select_bucket → lookup_nulls_elem_raw → htab_elem_value.` `Per-LRU: prealloc_lru_pop (may evict via htab_lru_map_delete_node) → copy_value → lock_bucket → swap → htab_lru_push_free(old).` Semantic equivalence: per-`Documentation/bpf/map_hash.rst` + `Documentation/bpf/map_lru_hash_update.rst` + `tools/testing/selftests/bpf/test_maps.c` + `tools/testing/selftests/bpf/progs/test_map_in_map.c`.

## Hardening

(Inherits row-1 features from `kernel/bpf/00-overview.md` § Hardening.)

BPF-hashmap reinforcement:

- **Per-hashrnd random seed** — defense against per-hash-collision flooding DoS: unprivileged callers cannot force BPF_F_ZERO_SEED.
- **Per-CAP_SYS_ADMIN required for BPF_F_ZERO_SEED** — defense against per-deterministic-collision-attack from non-root.
- **Per-bucket rqspinlock** — defense against per-priority-inversion + per-RT-lockup: resilient queued spinlock returns -EBUSY on timeout.
- **Per-WARN_ON_ONCE(!bpf_rcu_lock_held)** — defense against per-lookup-without-RCU UAF.
- **Per-hlist_nulls retry** — defense against per-lockless-walk-into-wrong-bucket aliasing.
- **Per-max_entries cap** — defense against per-unbounded-memory: max_entries > 1<<31 rejected; elem_size + count clamped to KMALLOC_MAX_SIZE.
- **Per-PCPU_MIN_UNIT_SIZE percpu value cap** — defense against per-percpu-OOM.
- **Per-LRU requires prealloc** — defense against per-eviction-during-allocation deadlock.
- **Per-NO_COMMON_LRU requires LRU type** — defense against per-misconfigured-LRU layouts.
- **Per-numa_node forbidden for percpu** — defense against per-cross-node percpu allocation mismatch.
- **Per-bpf_mem_alloc backing** — defense against per-bpf-prog GFP_ATOMIC failure during update: bpf_mem_alloc maintains percpu cache.
- **Per-extra_elems per-CPU spare** — defense against per-freelist-empty-during-replace: in-place update never blocks on global freelist.
- **Per-rcu-grace before bpf_mem_cache_free** — defense against per-concurrent-reader UAF: bpf_mem_alloc handles call_rcu internally.
- **Per-check_and_free_fields on prealloc reuse** — defense against per-stale-special-fields (spin_lock, kptr, timer) reuse.
- **Per-BUILD_BUG_ON fnode/hash_node overlap** — defense against per-layout-drift breaking transition prealloc→bucket.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- bpf_mem_alloc internals (covered separately if expanded)
- bpf_lru_list LRU policy + percpu_freelist (covered separately if expanded; `bpf_lru_list.c`)
- Verifier-side map-pointer tracking (covered in `verifier.md` Tier-3)
- map_gen_lookup inlining decisions in verifier (covered in `verifier.md` Tier-3)
- SOCKMAP / SOCKHASH (separate Tier-3 doc)
- LPM_TRIE / DEVMAP / CPUMAP (separate Tier-3 docs)
- ARENA / RINGBUF (separate Tier-3 docs)
- Map batch operations (`generic_map_*_batch`) (covered in `bpf-syscall.md` if expanded)
- bpf_iter for hashmaps (`bpf_iter_init_hash_map`, etc.) (covered in `bpf-iter.md` if expanded)
- rqspinlock semantics (covered in `kernel-platform.md` / `locking.md`)
- Implementation code
