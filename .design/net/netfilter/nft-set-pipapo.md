# Tier-3: net/netfilter/nft_set_pipapo.c — pipapo concatenated-range set backend

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/netfilter/00-overview.md
upstream-paths:
  - net/netfilter/nft_set_pipapo.c (~2442 lines)
  - net/netfilter/nft_set_pipapo.h
  - net/netfilter/nft_set_pipapo_avx2.c
  - net/netfilter/nft_set_pipapo_avx2.h
-->

## Summary

**Pipapo** ("PIle PAcket POlicies") is the nftables set backend that classifies packets against entries composed of **arbitrary concatenations of ranged or non-ranged fields**. It supports `NFT_SET_INTERVAL | NFT_SET_MAP | NFT_SET_OBJECT | NFT_SET_TIMEOUT`. The algorithm (loosely from Ligatti 2010 / Rottenstreich 2010 / Kogan 2014) decomposes each field's b bits into ⌈b/t⌉ groups, builds per-field lookup tables (`lt`) with one column per possible group value ("bucket") and one row per group, and represents matching rules as bitmaps. Ranges are expanded into ≤ 2b composing netmasks (`pipapo_expand`). Each field's mapping table (`mt`) chains matched rules into the next field's rule range, and the last field's mt maps to `nft_pipapo_elem` references.

Per-lookup (`pipapo_get_slow`) maintains two per-CPU bitmaps in `scratch` (`res_map`, `fill_map`), ANDs the buckets for each group, calls `pipapo_refill` to project to the next field's rule range, swaps res/fill, and on the last field returns the matched element. The per-CPU `bh_lock` and `local_bh_disable` enforce no-reentrancy. On x86_64 with AVX2 the data-path dispatcher (`pipapo_get` / `nft_pipapo_avx2_lookup`) uses `pipapo_get_avx2` when `irq_fpu_usable()`. Per-write path uses a deferred-commit clone (`pipapo_clone`): all inserts and removes operate on `priv->clone`, then `nft_pipapo_commit` `rcu_replace_pointer`s `priv->match`. Per-`pipapo_lt_bits_adjust` rebalances lookup tables between 4-bit and 8-bit grouping based on the current rule count.

Critical for: high-cardinality concatenated-key sets (e.g., `{ ip saddr . ip daddr . tcp dport }`), interval sets, NAT-map sets — the standard `nft` data structure for everything that doesn't fit in `nft_set_rbtree` (single key, ranged) or `nft_set_hash` (single key, exact).

This Tier-3 covers `net/netfilter/nft_set_pipapo.c` (~2442 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct nft_pipapo` (priv) | per-set context: match, clone, gc_head, last_gc, width | `Pipapo` |
| `struct nft_pipapo_match` | per-version match data: field_count, bsize_max, scratch, rcu, f[] | `PipapoMatch` |
| `struct nft_pipapo_field` | per-field tables: groups, bb, bsize, rules, rules_alloc, lt, mt | `PipapoField` |
| `struct nft_pipapo_elem` | per-element: priv, ext | `PipapoElem` |
| `struct nft_pipapo_scratch` | per-CPU bitmaps: map_index, bh_lock, __map[] | `PipapoScratch` |
| `union nft_pipapo_map_bucket` | (to, n) rule-range pair or (e) elem pointer (last field) | `PipapoMapBucket` |
| `pipapo_refill()` | per-bitmap → next-field bitmap projection | `Pipapo::refill` |
| `pipapo_get_slow()` | per-key C lookup | `Pipapo::get_slow` |
| `pipapo_get()` | per-key dispatcher (AVX2 / slow) | `Pipapo::get` |
| `nft_pipapo_lookup()` | datapath set lookup | `Pipapo::lookup` |
| `nft_pipapo_get()` | control-plane set lookup | `Pipapo::get_cp` |
| `pipapo_realloc_mt()` | per-field mt resize | `Pipapo::realloc_mt` |
| `lt_calculate_size()` | per-field lt size calc | `Pipapo::lt_calculate_size` |
| `pipapo_resize()` | per-field lt + mt resize | `Pipapo::resize` |
| `pipapo_bucket_set()` | per-(group, bucket, rule) bit set | `Pipapo::bucket_set` |
| `pipapo_lt_4b_to_8b()` | per-field 4-bit → 8-bit lt rewrite | `Pipapo::lt_4b_to_8b` |
| `pipapo_lt_8b_to_4b()` | per-field 8-bit → 4-bit lt rewrite | `Pipapo::lt_8b_to_4b` |
| `pipapo_lt_bits_adjust()` | per-field choose 4b or 8b | `Pipapo::lt_bits_adjust` |
| `pipapo_insert()` | per-field insert single rule | `Pipapo::insert_rule` |
| `pipapo_step_diff()` / `_step_after_end()` / `_base_sum()` | per-range expansion arith | `Pipapo::step_*` |
| `pipapo_expand()` | per-range → composing netmasks | `Pipapo::expand` |
| `pipapo_map()` | per-mt cross-field chaining | `Pipapo::map` |
| `pipapo_free_scratch()` / `_realloc_scratch()` | per-CPU scratch | `Pipapo::*_scratch` |
| `nft_pipapo_transaction_mutex_held()` | per-lockdep assertion | `Pipapo::tx_mutex_held` |
| `pipapo_clone()` | per-write working-copy clone | `Pipapo::clone` |
| `pipapo_maybe_clone()` | per-write lazy clone | `Pipapo::maybe_clone` |
| `nft_pipapo_insert()` | per-API insert | `Pipapo::insert` |
| `pipapo_rules_same_key()` | per-mt run-length | `Pipapo::rules_same_key` |
| `pipapo_unmap()` | per-mt unmap rules | `Pipapo::unmap` |
| `pipapo_drop()` | per-rulemap drop from all fields | `Pipapo::drop` |
| `nft_pipapo_gc_deactivate()` | per-gc transaction | `Pipapo::gc_deactivate` |
| `pipapo_gc_scan()` | per-set expired-walk | `Pipapo::gc_scan` |
| `pipapo_gc_queue()` | per-gc reclaim | `Pipapo::gc_queue` |
| `pipapo_free_fields()` / `_free_match()` | per-match free | `Pipapo::free_match` |
| `pipapo_reclaim_match()` | per-RCU callback | `Pipapo::reclaim_match` |
| `nft_pipapo_commit()` | per-tx swap clone→match | `Pipapo::commit` |
| `nft_pipapo_abort()` | per-tx discard clone | `Pipapo::abort` |
| `nft_pipapo_activate()` | per-elem activate | `Pipapo::activate` |
| `nft_pipapo_deactivate()` | per-elem deactivate | `Pipapo::deactivate` |
| `nft_pipapo_flush()` | per-elem flush | `Pipapo::flush` |
| `pipapo_get_boundaries()` | per-field rule → byte interval | `Pipapo::get_boundaries` |
| `pipapo_match_field()` | per-field byte-interval check | `Pipapo::match_field` |
| `nft_pipapo_remove()` | per-API remove | `Pipapo::remove` |
| `nft_pipapo_do_walk()` / `_walk()` | per-set iteration | `Pipapo::walk` |
| `nft_pipapo_privsize()` | per-set priv size | `Pipapo::privsize` |
| `nft_pipapo_estimate()` | per-desc accept + complexity | `Pipapo::estimate` |
| `nft_pipapo_init()` | per-set construct | `Pipapo::init` |
| `nft_set_pipapo_match_destroy()` | per-set elem destroy | `Pipapo::match_destroy` |
| `nft_pipapo_destroy()` | per-set destruct | `Pipapo::destroy` |
| `nft_pipapo_gc_init()` | per-set gc bookkeeping | `Pipapo::gc_init` |
| `nft_set_pipapo_type` / `_avx2_type` | nft_set_type ops table | shared |
| `pipapo_get_avx2()` / `nft_pipapo_avx2_lookup()` | per-x86_64 AVX2 fast path | `PipapoAvx2::*` |

## Compatibility contract

REQ-1: struct nft_pipapo (per-set private data):
- match: RCU pointer to current `nft_pipapo_match` (datapath / control-plane reads).
- clone: working `nft_pipapo_match` for in-progress transaction (held under nft_pernet().commit_mutex).
- last_gc: jiffies of last gc scan.
- width: sum of per-field round_up(len, sizeof(u32)).
- gc_head: list of pending `nft_trans_gc` reclaim batches.

REQ-2: struct nft_pipapo_match (versioned match-data):
- field_count: # concatenated fields.
- bsize_max: max per-field bitmap size in longs.
- scratch: per-CPU `*nft_pipapo_scratch`.
- rcu: rcu_head for reclaim.
- f[]: trailing flex-array of `nft_pipapo_field`.

REQ-3: struct nft_pipapo_field:
- groups: # bit-groups in this field (b / t).
- bb: bits-per-group (4 or 8).
- bsize: bitmap longs needed to hold `rules` bits.
- rules: # rules (each = one netmask after `pipapo_expand`).
- rules_alloc: rule slots allocated in mt[].
- lt: lookup table, `groups * NFT_PIPAPO_BUCKETS(bb) * bsize` longs, NFT_PIPAPO_LT_ALIGN-aligned.
- mt: mapping table, `rules_alloc` `nft_pipapo_map_bucket`s (per-rule (to, n) for non-last field; per-rule (e) for last field).

REQ-4: union nft_pipapo_map_bucket:
- (to: u32, n: u32) — first rule index + run length in next field.
- (e: *nft_pipapo_elem) — for the last field.

REQ-5: struct nft_pipapo_scratch (per-CPU):
- map_index: 0 or 1, selects which half of __map[] is res vs fill.
- bh_lock: per-CPU local-bh nested lock (lockdep).
- __map[]: 2 * bsize_max longs (double-buffered bitmaps).

REQ-6: pipapo_refill(map, len, rules, dst, mt, match_only) -> i32:
- For k in 0..len:
  - bitset = map[k].
  - while bitset:
    - r = __builtin_ctzl(bitset). i = k * BITS_PER_LONG + r.
    - if i ≥ rules: map[k] = 0; return -1.
    - if match_only: bitmap_clear(map, i, 1); return i.
    - bitmap_set(dst, mt[i].to, mt[i].n).
    - bitset ^= (bitset & -bitset).
  - map[k] = 0.
- return 0 if any bits processed, else -1.

REQ-7: pipapo_get_slow(m, data, genmask, tstamp) -> *nft_pipapo_elem:
- local_bh_disable.
- scratch = *raw_cpu_ptr(m.scratch). If NULL → return NULL.
- __local_lock_nested_bh(scratch.bh_lock).
- map_index = scratch.map_index.
- res_map = NFT_PIPAPO_LT_ALIGN(&scratch.__map[0]) + (map_index ? bsize_max : 0).
- fill_map = NFT_PIPAPO_LT_ALIGN(&scratch.__map[0]) + (map_index ? 0 : bsize_max).
- pipapo_resmap_init(m, res_map). /* all-ones to start */
- foreach (f, i) in m.f:
  - last = (i == m.field_count - 1).
  - if f.bb == 8: pipapo_and_field_buckets_8bit(f, res_map, data).
  - else: pipapo_and_field_buckets_4bit(f, res_map, data).
  - next_match:
  - b = pipapo_refill(res_map, f.bsize, f.rules, fill_map, f.mt, last).
  - if b < 0: scratch.map_index = map_index; unlock_bh; local_bh_enable; return NULL.
  - if last:
    - e = f.mt[b].e.
    - if __nft_set_elem_expired(&e.ext, tstamp) OR !nft_set_elem_active(&e.ext, genmask): goto next_match.
    - scratch.map_index = map_index. unlock_bh. local_bh_enable. return e.
  - swap(res_map, fill_map). map_index = !map_index.
  - data += NFT_PIPAPO_GROUPS_PADDED_SIZE(f).
- unlock_bh; local_bh_enable; return NULL.

REQ-8: pipapo_get(m, data, genmask, tstamp):
- local_bh_disable.
- if CONFIG_X86_64 ∧ !CONFIG_UML ∧ boot_cpu_has(X86_FEATURE_AVX2) ∧ irq_fpu_usable():
  - e = pipapo_get_avx2(m, data, genmask, tstamp). local_bh_enable; return e.
- e = pipapo_get_slow(m, data, genmask, tstamp). local_bh_enable; return e.

REQ-9: nft_pipapo_lookup(net, set, key):
- priv = nft_set_priv(set).
- m = rcu_dereference(priv.match).
- e = pipapo_get_slow(m, (const u8*)key, 0, get_jiffies_64()).
- return e ? &e.ext : NULL.
- Note: uses genmask = 0 (not nft_genmask_cur()) — see REQ-29.

REQ-10: nft_pipapo_get(net, set, elem, flags):
- priv = nft_set_priv(set).
- m = rcu_dereference(priv.match).
- e = pipapo_get(m, elem.key.val.data, nft_genmask_cur(net), get_jiffies_64()).
- if !e: return ERR_PTR(-ENOENT).
- return &e.priv.

REQ-11: pipapo_realloc_mt(f, old_rules, rules) -> int:
- might_sleep.
- if rules == 0: kvfree(f.mt); f.mt = NULL; f.rules_alloc = 0; return 0.
- if rules > old_rules ∧ f.rules_alloc > rules: return 0 (room).
- if rules < old_rules ∧ (f.rules_alloc − rules) < 2 * PAGE_SIZE/sizeof(*mt): return 0 (slack ok).
- rules_alloc = max(rules, f.rules_alloc rule allocation target).
- new_mt = kvmalloc_objs(*new_mt, rules_alloc, GFP_KERNEL_ACCOUNT).
- copy old.
- kvfree(f.mt); f.mt = new_mt; f.rules_alloc = rules_alloc.

REQ-12: lt_calculate_size(groups, bb, bsize) -> ssize_t:
- Bytes for f.lt: groups * NFT_PIPAPO_BUCKETS(bb) * bsize * sizeof(long), plus NFT_PIPAPO_LT_ALIGN padding.
- Return -1 on overflow.

REQ-13: pipapo_resize(f, old_rules, rules) -> int:
- Compute new bsize = BITS_TO_LONGS(rules).
- If bsize == f.bsize ∧ f.rules_alloc ≥ rules: return 0.
- Allocate new lt with lt_calculate_size(f.groups, f.bb, new_bsize); copy/transcribe per-bucket bits.
- kvfree old f.lt; swap in new.
- pipapo_realloc_mt(f, old_rules, rules).

REQ-14: pipapo_bucket_set(f, rule, group, value):
- pos = NFT_PIPAPO_LT_ALIGN(f.lt) + (group * NFT_PIPAPO_BUCKETS(f.bb) + value) * f.bsize.
- set_bit(rule, pos).

REQ-15: pipapo_lt_4b_to_8b(old_groups, bsize, old_lt, new_lt):
- Translate per-(group, bucket) bitmaps from 16 buckets × 2*groups to 256 buckets × groups by OR-merging pairs of nibble-buckets that match each byte value.

REQ-16: pipapo_lt_8b_to_4b(old_groups, bsize, old_lt, new_lt):
- Inverse: split 256-bucket bytes into pairs of 16-bucket nibbles.

REQ-17: pipapo_lt_bits_adjust(f):
- Heuristic switch between bb=4 and bb=8 based on f.rules and bsize: choose whichever uses less memory for current rule count.
- Reallocate f.lt under new bb.

REQ-18: pipapo_insert(f, k, len_bits):
- For each group g in 0..f.groups:
  - value = group bits of k.
  - pipapo_bucket_set(f, f.rules, g, value).
- f.rules += 1.
- return 1 (number of rules added).

REQ-19: pipapo_step_diff / _step_after_end / _base_sum (file-static):
- Internal helpers used by pipapo_expand for incrementing the rolling base and detecting termination of the netmask sequence.

REQ-20: pipapo_expand(f, start, end, len_bits) -> int:
- Per Theorem 3 (Rottenstreich 2010): a contiguous [start, end] range in b bits is the union of ≤ 2b composing netmasks.
- Loop:
  - Emit one netmask covering current base; pipapo_insert(f, base, len_bits).
  - Compute step (next netmask increment).
  - pipapo_base_sum(base, step, len).
  - if pipapo_step_after_end(base, end, step, len): break.
- Return number of netmasks inserted.

REQ-21: pipapo_map(m, rulemap, e):
- For each non-last field i: f.mt[rulemap[i].to .. +rulemap[i].n].to = rulemap[i+1].to; .n = rulemap[i+1].n.
- For the last field i_last: f.mt[rulemap[i_last].to .. +rulemap[i_last].n].e = e.

REQ-22: pipapo_free_scratch(m, cpu):
- s = *per_cpu_ptr(m.scratch, cpu).
- if s: free_pages_exact(s, ...).

REQ-23: pipapo_realloc_scratch(clone, bsize_max) -> int:
- For each possible cpu:
  - bytes = 2 * bsize_max * sizeof(long) + offsetof(nft_pipapo_scratch, __map) + NFT_PIPAPO_ALIGN_HEADROOM.
  - new = alloc_pages_exact_nid(cpu_to_node(cpu), bytes, GFP_KERNEL_ACCOUNT).
  - if !new: return -ENOMEM.
  - new.map_index = 0; init bh_lock.
  - pipapo_free_scratch(clone, cpu).
  - *per_cpu_ptr(clone.scratch, cpu) = new.

REQ-24: nft_pipapo_transaction_mutex_held(set):
- CONFIG_PROVE_LOCKING: lockdep_is_held(&nft_pernet(net).commit_mutex). Else: true.

REQ-25: pipapo_clone(old) -> *nft_pipapo_match:
- new = kmalloc_flex(*new, f, old.field_count).
- new.field_count = old.field_count; new.bsize_max = old.bsize_max.
- new.scratch = alloc_percpu(*new.scratch). Per-cpu *= NULL.
- pipapo_realloc_scratch(new, old.bsize_max).
- rcu_head_init(&new.rcu).
- For each src, dst in old.f, new.f:
  - memcpy(dst, src, offsetof(struct nft_pipapo_field, lt)).
  - lt_size = lt_calculate_size; new_lt = kvzalloc(lt_size, GFP_KERNEL_ACCOUNT).
  - memcpy(NFT_PIPAPO_LT_ALIGN(new_lt), NFT_PIPAPO_LT_ALIGN(src.lt), src.bsize * sizeof(*dst.lt) * src.groups * NFT_PIPAPO_BUCKETS(src.bb)).
  - if src.rules > 0: dst.mt = kvmalloc_objs(*src.mt, src.rules_alloc, GFP_KERNEL_ACCOUNT); memcpy(dst.mt, src.mt, src.rules * sizeof(*src.mt)).
  - else: dst.mt = NULL; dst.rules_alloc = 0.
- Failure path frees partial allocations and per-cpu scratch.

REQ-26: pipapo_maybe_clone(set):
- if priv.clone: return priv.clone.
- m = rcu_dereference_protected(priv.match, tx_mutex_held).
- priv.clone = pipapo_clone(m).

REQ-27: nft_pipapo_insert(net, set, elem, *elem_priv):
- ext = nft_set_elem_ext.
- m = pipapo_maybe_clone(set); if !m: return -ENOMEM.
- start = elem.key.val.data; end = NFT_SET_EXT_KEY_END ? ext.key_end.data : start.
- dup = pipapo_get(m, start, nft_genmask_next(net), nft_net_tstamp(net)).
- if dup:
  - if memcmp matches on both start and end: *elem_priv = &dup.priv; return -EEXIST.
  - return -ENOTEMPTY.
- dup = pipapo_get(m, end, ...); if dup: return -ENOTEMPTY.
- /* Validate */ for each field: start_p ≤ end_p in memcmp on f.groups bytes.
- BUILD_BUG_ON(NFT_PIPAPO_RULE0_MAX > INT_MAX).
- /* Insert */ rulemap[NFT_PIPAPO_MAX_FIELDS].
- for each field f:
  - if f.rules ≥ NFT_PIPAPO_RULE0_MAX: return -ENOSPC.
  - rulemap[i].to = f.rules.
  - if memcmp(start, end) on f.groups == 0: ret = pipapo_insert(f, start, f.groups * f.bb).
  - else: ret = pipapo_expand(f, start, end, f.groups * f.bb).
  - if ret < 0: return ret.
  - if f.bsize > bsize_max: bsize_max = f.bsize.
  - rulemap[i].n = ret.
  - start, end += NFT_PIPAPO_GROUPS_PADDED_SIZE(f).
- If !*get_cpu_ptr(m.scratch) OR bsize_max > m.bsize_max:
  - put_cpu_ptr; pipapo_realloc_scratch(m, bsize_max); m.bsize_max = bsize_max.
- e = nft_elem_priv_cast(elem.priv).
- *elem_priv = &e.priv.
- pipapo_map(m, rulemap, e).
- return 0.

REQ-28: pipapo_rules_same_key(f, first):
- For r = first..f.rules: if mt[r].e != mt[first].e: return r − first. Else: continue.
- Last group: return f.rules − first.

REQ-29: nft_pipapo_commit(set):
- if !priv.clone: return.
- if time_after_eq(jiffies, priv.last_gc + nft_set_gc_interval(set)): pipapo_gc_scan(set, priv.clone).
- old = rcu_replace_pointer(priv.match, priv.clone, tx_mutex_held).
- priv.clone = NULL.
- if old: call_rcu(&old.rcu, pipapo_reclaim_match).
- pipapo_gc_queue(set).
- (Reason `nft_pipapo_lookup` passes genmask=0: when commit flips priv.match, old entries are still active in genmask_cur but unreachable; new entries are reachable in priv.match but already past genmask_next. genmask=0 means "match by structure, defer activation to nft core" — see the kerneldoc on nft_pipapo_lookup.)

REQ-30: nft_pipapo_abort(set):
- if !priv.clone: return.
- pipapo_free_match(priv.clone).
- priv.clone = NULL.

REQ-31: nft_pipapo_activate(net, set, elem_priv):
- e = nft_elem_priv_cast(elem_priv).
- nft_clear(net, &e.ext) /* mark active in genmask_next */.

REQ-32: nft_pipapo_deactivate(net, set, elem) -> *nft_elem_priv:
- m = pipapo_maybe_clone(set); if !m: return NULL.
- e = pipapo_get(m, elem.key.val.data, nft_genmask_next(net), nft_net_tstamp(net)).
- if !e: return NULL.
- nft_set_elem_change_active(net, set, &e.ext) /* mark inactive in genmask_next */.
- return &e.priv.

REQ-33: nft_pipapo_flush(net, set, elem_priv):
- e = nft_elem_priv_cast(elem_priv).
- nft_set_elem_change_active(net, set, &e.ext).

REQ-34: pipapo_get_boundaries(f, first_rule, rule_count, left, right) -> mask_len:
- For each group g: scan all NFT_PIPAPO_BUCKETS(f.bb) buckets; x0 = first bucket with first_rule bit set; x1 = last bucket with (first_rule + rule_count - 1) bit set.
- Pack x0, x1 into left, right with f.bb-bit nibbles.
- Accumulate mask_len from per-group (x1 − x0): 0⇒+4, 1⇒+3, 3⇒+2, 7⇒+1 (per bit-pair narrowing).

REQ-35: pipapo_match_field(f, first_rule, rule_count, start, end) -> bool:
- pipapo_get_boundaries(f, first_rule, rule_count, left, right).
- return memcmp(start, left, f.groups / NFT_PIPAPO_GROUPS_PER_BYTE(f)) == 0 ∧ memcmp(end, right, ...) == 0.

REQ-36: nft_pipapo_remove(net, set, elem_priv):
- m = priv.clone.
- e = nft_elem_priv_cast(elem_priv). data = ext.key.
- first_rule = 0.
- while (rules_f0 = pipapo_rules_same_key(m.f, first_rule)):
  - start = first_rule. rules_fx = rules_f0. match_start = data; match_end = ext.key_end or data.
  - for each field f, i:
    - if !pipapo_match_field(f, start, rules_fx, match_start, match_end): break.
    - rulemap[i] = {to: start, n: rules_fx}.
    - rules_fx = f.mt[start].n. start = f.mt[start].to.
    - match_start, match_end += NFT_PIPAPO_GROUPS_PADDED_SIZE(f).
    - if last ∧ f.mt[rulemap[i].to].e == e: pipapo_drop(m, rulemap); return.
  - first_rule += rules_f0.
- WARN_ON_ONCE(1).

REQ-37: pipapo_drop(m, rulemap[]):
- For each field f:
  - For each group g in 0..f.groups:
    - For each bucket b in 0..NFT_PIPAPO_BUCKETS(f.bb): bitmap_cut(pos, pos, rulemap[i].to, rulemap[i].n, f.bsize * BITS_PER_LONG).
  - pipapo_unmap(f.mt, f.rules, rulemap[i].to, rulemap[i].n, last ? 0 : rulemap[i+1].n, last).
  - pipapo_resize(f, f.rules, f.rules − rulemap[i].n) /* failure ignored: shrink is best-effort */.
  - f.rules −= rulemap[i].n.
  - pipapo_lt_bits_adjust(f).

REQ-38: pipapo_gc_scan(set, m):
- gc = nft_trans_gc_alloc(set, 0, GFP_KERNEL); if !gc: return. list_add to priv.gc_head.
- first_rule = 0. tstamp = nft_net_tstamp(net).
- while rules_f0 = pipapo_rules_same_key(m.f, first_rule):
  - For each field walk rulemap (to, n) chain to last field's e = f.mt[..].e.
  - if __nft_set_elem_expired(&e.ext, tstamp):
    - if !nft_trans_gc_space(gc): allocate new gc; list_add to gc_head.
    - nft_pipapo_gc_deactivate(net, set, e).
    - pipapo_drop(m, rulemap).
    - nft_trans_gc_elem_add(gc, e).
  - else: first_rule += rules_f0.
- priv.last_gc = jiffies.

REQ-39: pipapo_gc_queue(set):
- gc = nft_trans_gc_alloc; nft_trans_gc_catchall_sync; nft_trans_gc_queue_sync_done.
- For each gc in priv.gc_head: list_del; nft_trans_gc_queue_sync_done.

REQ-40: pipapo_free_match(m):
- For each cpu: pipapo_free_scratch(m, cpu).
- free_percpu(m.scratch).
- For each f: kvfree(f.lt); kvfree(f.mt).
- kfree(m).

REQ-41: pipapo_reclaim_match(rcu):
- m = container_of(rcu, struct nft_pipapo_match, rcu).
- pipapo_free_match(m).

REQ-42: nft_pipapo_privsize(nla, desc) -> u64:
- return sizeof(struct nft_pipapo).

REQ-43: nft_pipapo_estimate(desc, features, est) -> bool:
- Require features & NFT_SET_INTERVAL.
- Require desc.field_count ≥ NFT_PIPAPO_MIN_FIELDS.
- est.size = pipapo_estimate_size(desc); 0 ⟹ false.
- est.lookup = NFT_SET_CLASS_O_LOG_N.
- est.space  = NFT_SET_CLASS_O_N.

REQ-44: nft_pipapo_init(set, desc, nla):
- BUILD_BUG_ON(offsetof(nft_pipapo_elem, priv) != 0).
- field_count = desc.field_count ?: 1.
- BUILD_BUG_ON(NFT_PIPAPO_MAX_FIELDS > 255).
- BUILD_BUG_ON(NFT_PIPAPO_MAX_FIELDS != NFT_REG32_COUNT).
- if field_count > NFT_PIPAPO_MAX_FIELDS: return -EINVAL.
- m = kmalloc_flex(*m, f, field_count); m.field_count = field_count; m.bsize_max = 0.
- m.scratch = alloc_percpu(*); per-cpu = NULL.
- rcu_head_init(&m.rcu).
- For each f in m.f:
  - len = desc.field_len[i] ?: set.klen.
  - f.bb = NFT_PIPAPO_GROUP_BITS_INIT.
  - f.groups = len * NFT_PIPAPO_GROUPS_PER_BYTE(f).
  - priv.width += round_up(len, sizeof(u32)).
  - f.bsize = 0; f.rules = 0; f.rules_alloc = 0; f.lt = NULL; f.mt = NULL.
- INIT_LIST_HEAD(priv.gc_head).
- rcu_assign_pointer(priv.match, m).

REQ-45: nft_pipapo_destroy(ctx, set):
- WARN_ON_ONCE(!list_empty(priv.gc_head)).
- m = rcu_dereference_protected(priv.match, true).
- if priv.clone: nft_set_pipapo_match_destroy(ctx, set, priv.clone); pipapo_free_match(priv.clone); priv.clone = NULL. else: nft_set_pipapo_match_destroy(ctx, set, m).
- pipapo_free_match(m).

REQ-46: nft_pipapo_gc_init(set):
- priv.last_gc = jiffies.

REQ-47: nft_pipapo_walk(ctx, set, iter):
- Walk priv.clone first if present (transaction view), else priv.match.
- For each unique element e in the last field's mt[]: iter.fn(ctx, set, iter, &e.priv).

REQ-48: AVX2 fast path (CONFIG_X86_64 ∧ !CONFIG_UML):
- `pipapo_get_avx2` gated by `boot_cpu_has(X86_FEATURE_AVX2)` ∧ `irq_fpu_usable()`.
- `kernel_fpu_begin(); /* ymm operations */; kernel_fpu_end()` (implementation in `nft_set_pipapo_avx2.c`).
- Same semantic as `pipapo_get_slow`; preserves res_map / fill_map invariants.

## Acceptance Criteria

- [ ] AC-1: `nft_pipapo_estimate` rejects descriptions without `NFT_SET_INTERVAL` or with `field_count < NFT_PIPAPO_MIN_FIELDS`.
- [ ] AC-2: `nft_pipapo_init` rejects `field_count > NFT_PIPAPO_MAX_FIELDS`.
- [ ] AC-3: `pipapo_refill` projects res_map to dst via mt[i].(to, n) for every set bit, clears src as it goes, and returns -1 if no rule index < rules survives.
- [ ] AC-4: `pipapo_get_slow` returns first non-expired, active-in-genmask element when concatenated bytes match; NULL otherwise.
- [ ] AC-5: `pipapo_get` falls back to `pipapo_get_slow` when AVX2 not available or `irq_fpu_usable()` false.
- [ ] AC-6: `nft_pipapo_lookup` uses `genmask = 0` (per REQ-29 rationale) — old generation lookups are filtered by clone visibility, not by `nft_genmask_cur()`.
- [ ] AC-7: `pipapo_clone` produces a deep copy: independent `scratch`, independent per-field `lt` and `mt`; original `lt`/`mt` unchanged.
- [ ] AC-8: `pipapo_maybe_clone` is idempotent within a single transaction: repeated calls return the same `priv.clone`.
- [ ] AC-9: `nft_pipapo_insert` rejects exact duplicate with `-EEXIST` (and returns existing `elem_priv`), partial overlap with `-ENOTEMPTY`.
- [ ] AC-10: `nft_pipapo_insert` rejects when any field reaches `NFT_PIPAPO_RULE0_MAX` with `-ENOSPC`.
- [ ] AC-11: `nft_pipapo_insert` expands ranges via `pipapo_expand` into ≤ 2*b composing netmasks.
- [ ] AC-12: `nft_pipapo_commit` swaps `priv.match` via `rcu_replace_pointer` and schedules `pipapo_reclaim_match` RCU callback.
- [ ] AC-13: `nft_pipapo_abort` frees `priv.clone` without touching `priv.match`.
- [ ] AC-14: `nft_pipapo_remove` calls `pipapo_drop` on the rulemap chain that resolves to the target element; `WARN_ON_ONCE` if not found.
- [ ] AC-15: `pipapo_gc_scan` deactivates and queues every expired element via `nft_trans_gc_*`; subsequent `pipapo_gc_queue` drains.
- [ ] AC-16: `pipapo_lt_bits_adjust` switches between bb=4 and bb=8 based on memory heuristic; lookup semantics preserved across the switch.

## Architecture

```
struct Pipapo {                     // nft_set_priv
  match: RcuPtr<PipapoMatch>,
  clone: Option<Box<PipapoMatch>>,  // tx-mutex protected
  last_gc: u64,
  width: u32,
  gc_head: ListHead<NftTransGc>,
}

struct PipapoMatch {
  field_count: u8,
  bsize_max: u32,
  scratch: PerCpu<*PipapoScratch>,
  rcu: RcuHead,
  f: [PipapoField; field_count],    // flex-array
}

struct PipapoField {
  groups: u8,
  bb: u8,                           // 4 or 8
  bsize: u32,
  rules: u32,
  rules_alloc: u32,
  lt: *u64,                         // groups * BUCKETS(bb) * bsize longs, LT_ALIGN'd
  mt: *PipapoMapBucket,             // rules_alloc entries
}

union PipapoMapBucket {
  Range { to: u32, n: u32 },        // non-last field
  Elem  { e: *PipapoElem },         // last field
}

struct PipapoScratch {
  map_index: u8,
  bh_lock: LocalLockNestedBh,
  __map: [u64; 2 * bsize_max + headroom],  // double-buffered
}
```

`Pipapo::get_slow(m, data, genmask, tstamp) -> Option<*PipapoElem>`:
1. local_bh_disable.
2. scratch = raw_cpu_ptr(m.scratch). if NULL → drop bh; return None.
3. lock scratch.bh_lock.
4. map_index = scratch.map_index. res_map, fill_map = halves of __map per map_index.
5. pipapo_resmap_init(m, res_map) /* all-ones */.
6. for (i, f) in m.f.enumerate():
   - last = (i == m.field_count - 1).
   - if f.bb == 8: pipapo_and_field_buckets_8bit(f, res_map, data).
   - else: pipapo_and_field_buckets_4bit(f, res_map, data).
   - loop {
     - b = pipapo_refill(res_map, f.bsize, f.rules, fill_map, f.mt, last).
     - if b < 0: scratch.map_index = map_index; unlock; bh_enable; return None.
     - if last:
       - e = f.mt[b].e.
       - if expired(e, tstamp) OR !active(e, genmask): continue.
       - scratch.map_index = map_index; unlock; bh_enable; return Some(e).
     - else: break.
   }
   - swap(res_map, fill_map). map_index = !map_index. data += NFT_PIPAPO_GROUPS_PADDED_SIZE(f).
7. unlock; bh_enable; return None.

`Pipapo::lookup(net, set, key) -> Option<*NftSetExt>`:
1. priv = set.priv.
2. m = rcu_dereference(priv.match).
3. e = Pipapo::get_slow(m, key as *u8, 0, get_jiffies_64()).
4. return e.map(|e| &e.ext).

`Pipapo::clone(old) -> Option<*PipapoMatch>`:
1. new = kmalloc_flex(*new, f, old.field_count)?.
2. new.field_count = old.field_count; new.bsize_max = old.bsize_max.
3. new.scratch = alloc_percpu()?; per-cpu init NULL.
4. pipapo_realloc_scratch(new, old.bsize_max)?.
5. rcu_head_init(&new.rcu).
6. for (src, dst) in old.f, new.f:
   - memcpy header (everything up to lt).
   - lt_size = lt_calculate_size(src.groups, src.bb, src.bsize); abort on overflow.
   - dst.lt = kvzalloc(lt_size)?; memcpy LT_ALIGN'd payload.
   - if src.rules > 0: dst.mt = kvmalloc_objs(src.rules_alloc)?; memcpy mt[0..src.rules].
   - else: dst.mt = NULL; dst.rules_alloc = 0.
7. Failure unwinds: kvfree partial lt/mt, free per-cpu scratch, free new.

`Pipapo::insert(net, set, elem, *elem_priv) -> Result<(), Errno>`:
1. m = Pipapo::maybe_clone(set)?.
2. start = elem.key; end = ext.key_end OR start.
3. dup = pipapo_get(m, start, nft_genmask_next(net), tstamp).
   - exact match (start && end equal) ⟹ Err(EEXIST), set *elem_priv.
   - else ⟹ Err(ENOTEMPTY).
4. dup = pipapo_get(m, end, ...); if Some(_) ⟹ Err(ENOTEMPTY).
5. /* per-field validate start ≤ end */
6. rulemap[NFT_PIPAPO_MAX_FIELDS].
7. for each f:
   - if f.rules ≥ NFT_PIPAPO_RULE0_MAX: return Err(ENOSPC).
   - rulemap[i].to = f.rules.
   - if memcmp(start, end, f.groups bytes) == 0: ret = pipapo_insert(f, start, f.groups * f.bb).
   - else: ret = pipapo_expand(f, start, end, f.groups * f.bb).
   - if ret < 0: return Err(ret).
   - bsize_max = max(bsize_max, f.bsize).
   - rulemap[i].n = ret.
   - start, end += NFT_PIPAPO_GROUPS_PADDED_SIZE(f).
8. if !*get_cpu_ptr(m.scratch) OR bsize_max > m.bsize_max:
   - pipapo_realloc_scratch(m, bsize_max)?; m.bsize_max = bsize_max.
9. e = nft_elem_priv_cast(elem.priv); *elem_priv = &e.priv.
10. pipapo_map(m, rulemap, e).

`Pipapo::commit(set)`:
1. if priv.clone.is_none(): return.
2. if jiffies ≥ priv.last_gc + nft_set_gc_interval(set): Pipapo::gc_scan(set, priv.clone).
3. old = rcu_replace_pointer(priv.match, priv.clone, tx_mutex_held).
4. priv.clone = None.
5. if let Some(old) = old: call_rcu(&old.rcu, Pipapo::reclaim_match).
6. Pipapo::gc_queue(set).

`Pipapo::remove(net, set, elem_priv)`:
1. m = priv.clone.
2. e = nft_elem_priv_cast(elem_priv); data = e.ext.key.
3. first_rule = 0.
4. loop {
   - rules_f0 = pipapo_rules_same_key(m.f, first_rule); if rules_f0 == 0: break.
   - start = first_rule; rules_fx = rules_f0.
   - match_start = data; match_end = e.ext.key_end OR data.
   - for each f, i:
     - if !pipapo_match_field(f, start, rules_fx, match_start, match_end): break.
     - rulemap[i] = (start, rules_fx).
     - rules_fx = f.mt[start].n; start = f.mt[start].to.
     - match_start, match_end += NFT_PIPAPO_GROUPS_PADDED_SIZE(f).
     - if last ∧ f.mt[rulemap[i].to].e == e: pipapo_drop(m, rulemap); return.
   - first_rule += rules_f0.
}
5. WARN_ON_ONCE(1).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `refill_no_oob` | INVARIANT | per-refill: i = k * BITS_PER_LONG + r is checked against `rules`; otherwise map[k] = 0 and return -1. |
| `get_slow_bh_disabled` | INVARIANT | per-get_slow: local_bh disabled across full traversal; bh_lock held. |
| `lookup_uses_genmask_zero` | INVARIANT | per-lookup: passes 0 to pipapo_get_slow (per REQ-29). |
| `clone_independent_storage` | INVARIANT | per-clone: clone.lt and clone.mt distinct from old.lt and old.mt for every field. |
| `maybe_clone_idempotent` | INVARIANT | per-maybe_clone: second call returns same priv.clone pointer. |
| `insert_rules_bounded` | INVARIANT | per-insert: refuses when f.rules ≥ NFT_PIPAPO_RULE0_MAX. |
| `commit_atomic_rcu_replace` | INVARIANT | per-commit: priv.match swapped atomically via rcu_replace_pointer under tx mutex. |
| `commit_reclaim_via_rcu` | INVARIANT | per-commit: old not freed synchronously; call_rcu(pipapo_reclaim_match). |
| `abort_does_not_touch_match` | INVARIANT | per-abort: only priv.clone freed. |
| `remove_pipapo_drop_on_match` | INVARIANT | per-remove: returns only after successful pipapo_drop OR WARN_ON_ONCE. |
| `avx2_fpu_guarded` | INVARIANT | per-get: AVX2 path only when boot_cpu_has(X86_FEATURE_AVX2) ∧ irq_fpu_usable(). |
| `scratch_double_buffered` | INVARIANT | per-scratch: __map[] holds 2 * bsize_max longs; map_index toggles between fields. |

### Layer 2: TLA+

`net/netfilter/nft-set-pipapo.tla`:
- States: priv.match generation, priv.clone presence, per-field rules count, per-CPU scratch.map_index, expired-element set.
- Operations: lookup, insert, deactivate, remove, commit, abort, gc_scan, gc_queue, init, destroy.
- Properties:
  - `safety_lookup_consistent_during_swap` — per-commit: an in-flight datapath lookup observes either old or new priv.match (never both halves).
  - `safety_clone_only_under_tx_mutex` — per-write path: priv.clone mutated only with commit_mutex held.
  - `safety_rcu_reclaim_one_grace_period` — per-commit: old not freed before grace period.
  - `safety_no_double_drop` — per-remove: pipapo_drop on a rulemap with rule_count > 0 reduces f.rules by exactly rulemap[i].n.
  - `safety_gc_drains` — per-gc: gc_head emptied by gc_queue at end of commit.
  - `liveness_insert_or_error` — per-insert: returns in finite steps with one of {0, EEXIST, ENOTEMPTY, ENOSPC, ENOMEM, EINVAL}.
  - `liveness_lookup_terminates` — per-lookup: at most field_count iterations; each refill processes < bsize * BITS_PER_LONG bits.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `refill` post: dst contains union of mt[i].(to, n) ranges for every set bit in map | `Pipapo::refill` |
| `get_slow` post: returned e satisfies !expired ∧ active(genmask) | `Pipapo::get_slow` |
| `lookup` post: returns ext iff some active element matches concatenated key | `Pipapo::lookup` |
| `clone` post: deep copy; old.scratch, old.f[i].lt, old.f[i].mt unchanged | `Pipapo::clone` |
| `insert` post: success ⟹ pipapo_map applied; failure ⟹ no mutation of m | `Pipapo::insert` |
| `commit` post: priv.match == prior priv.clone; priv.clone == None | `Pipapo::commit` |
| `abort` post: priv.clone freed; priv.match unchanged | `Pipapo::abort` |
| `remove` post: f.rules reduced by rulemap[i].n in every field | `Pipapo::remove` |
| `gc_scan` post: every expired element queued under priv.gc_head | `Pipapo::gc_scan` |
| `init` post: priv.match has field_count fields with bb = NFT_PIPAPO_GROUP_BITS_INIT and rules = 0 | `Pipapo::init` |
| `destroy` post: priv.clone freed if non-NULL; priv.match freed; all elements destroyed | `Pipapo::destroy` |
| `lt_bits_adjust` post: lookup semantics unchanged across bb=4 ↔ bb=8 transition | `Pipapo::lt_bits_adjust` |

### Layer 4: Verus/Creusot functional

`Per-set transaction: pipapo_maybe_clone → pipapo_insert / pipapo_remove → nft_pipapo_commit (rcu_replace_pointer + call_rcu reclaim + gc_queue)` semantic equivalence: per-Stefano Brivio "PIle PAcket POlicies" reference and per-`Documentation/networking/nf_tables.rst`. Per-lookup-vs-commit RCU race-freedom: datapath `nft_pipapo_lookup` sees a single coherent `priv.match` snapshot. Per-AVX2 path: bit-identical lookup result to slow path under same scratch state. Per-range expansion (`pipapo_expand`): ≤ 2b composing netmasks cover the contiguous interval (Rottenstreich 2010 Theorem 3).

## Hardening

(Inherits row-1 features from `net/netfilter/00-overview.md` § Hardening.)

Pipapo reinforcement:

- **Per-RCU priv.match swap** — defense against per-mid-commit datapath UAF; rcu_replace_pointer + call_rcu enforce one full grace period.
- **Per-tx commit_mutex serialization** — defense against per-concurrent-write corruption of priv.clone; lockdep-checked via nft_pipapo_transaction_mutex_held.
- **Per-CPU scratch.bh_lock + local_bh_disable** — defense against per-softirq-reentrancy on shared res_map / fill_map.
- **Per-pipapo_refill bounds-check `i < rules`** — defense against per-stale-bit OOB read in mt[i].
- **Per-NFT_PIPAPO_RULE0_MAX cap** — defense against per-unbounded rule growth (INT_MAX hard ceiling).
- **Per-field_count ≤ NFT_PIPAPO_MAX_FIELDS** — defense against per-flex-array OOB.
- **Per-NFT_PIPAPO_LT_ALIGN aligned lt** — defense against per-misaligned vector load in AVX2 path.
- **Per-AVX2 gated by irq_fpu_usable()** — defense against per-FPU-state corruption during interrupt context.
- **Per-pipapo_clone deep copy** — defense against per-shared-table aliasing between live and pending.
- **Per-commit RCU reclaim (call_rcu pipapo_reclaim_match)** — defense against per-immediate-free UAF in concurrent readers.
- **Per-gc deactivation under transaction** — defense against per-races where a freed element is still reachable by lookup.
- **Per-pipapo_get duplicate check on insert** — defense against per-set-element double-insert (EEXIST/ENOTEMPTY).
- **Per-pipapo_drop best-effort resize (`pipapo_resize` failure ignored)** — defense against per-OOM mid-remove; tables remain valid, just slack.
- **Per-WARN_ON_ONCE in nft_pipapo_remove** — defense against per-silent-not-found which would corrupt counts.

## Grsecurity/PaX-style Reinforcement

- PAX_USERCOPY: pipapo field/element copy paths bound by `set->klen` and `set->dlen`; whitelisted slabs for `nft_pipapo_field`/`nft_pipapo_match`.
- PAX_KERNEXEC: pipapo AVX2 lookup runs in .text under kernel_fpu_begin/end; no JIT codegen at this layer; lookup tables are data-only RO pages where possible.
- PAX_RANDKSTACK: stack-base randomization on `setsockopt`/netlink ingress that triggers pipapo set rebuild.
- PAX_REFCOUNT: `nft_pipapo_match->use` and parent `nft_set->use` saturating refcount_t across RCU swap of match tables.
- PAX_MEMORY_SANITIZE: old `nft_pipapo_match` zeroed in `pipapo_reclaim_match()` before kvfree; bitmap pages scrubbed.
- PAX_UDEREF: NFTA_SET_ELEM_KEY / NFTA_SET_ELEM_KEY_END pointers validated against `set->klen` before deref.
- PAX_RAP / kCFI: pipapo `nft_set_ops` (insert/remove/lookup/walk/gc) kCFI-typed.
- GRKERNSEC_HIDESYM: `nft list set` output redacts kernel pointers; match-table addresses hidden.
- GRKERNSEC_DMESG: pipapo rebuild OOM / size-overflow warnings rate-limited; CAP_SYSLOG gates detail.
- nftables CAP_NET_ADMIN: NFT_MSG_NEWSET/NEWSETELEM with `NFT_SET_OBJECT|NFT_SET_INTERVAL|NFT_SET_CONCAT` gated by CAP_NET_ADMIN.
- Transaction commit-abort: pipapo match-table swap is RCU-published only on commit; abort kvfrees the staged table without ever exposing it to lookup.
- nft_chain PAX_REFCOUNT: chains bound via `nft_lookup` to a pipapo set use `nft_use_inc()` saturating.
- JIT W^X (PAX_KERNEXEC): no executable JIT in pipapo; SIMD lookup uses signed kernel-mode AVX2 with `irq_fpu_usable()` checks.

Rationale: pipapo is a high-fanout match engine that handled CVE-2023-0179-style OOB-read regressions; bounded field copies and RCU-published table swaps under saturating refcount prevent the documented torn-update class.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- AVX2 implementation details (`net/netfilter/nft_set_pipapo_avx2.c`, ymm shuffles, byte-aligned bucket masks — covered separately if expanded)
- nft_set_rbtree / nft_set_hash / nft_set_bitmap backends (covered in `nft-sets.md` sibling)
- nft core (covered in `nft-core.md`)
- nft transaction infrastructure: `struct nft_trans`, `struct nft_trans_gc`, commit_mutex semantics (covered in `nf-tables.md`)
- nftables expression evaluation (covered in `nft-expressions.md`)
- iptables / x_tables legacy path (covered in `x_tables.md` and `iptables.md`)
- Implementation code
