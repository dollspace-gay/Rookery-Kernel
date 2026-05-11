---
title: "Tier-3: kernel/bpf/devmap.c — BPF devmap (XDP net_device redirect)"
tags: ["tier-3", "kernel", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-11
updated: 2026-05-11
---


## Design Specification

### Summary

The `devmap` (and `devmap_hash` variant) is the backend map for `bpf_redirect_map()` + `XDP_REDIRECT` that pushes XDP frames out to **another `net_device`** via `ndo_xdp_xmit`. Two map types share most code: per-`BPF_MAP_TYPE_DEVMAP` (key = u32 index, dense linear array `netdev_map[max_entries]`) and per-`BPF_MAP_TYPE_DEVMAP_HASH` (key = u32 ifindex, sparse `hlist_head[n_buckets]` with `n_buckets = roundup_pow_of_two(max_entries)`). Per-`bpf_dtab_netdev` slot holds an `*net_device`, an optional secondary XDP `bpf_prog` (`expected_attach_type == BPF_XDP_DEVMAP`), and a back-reference index. Per-`net_device` has a per-CPU `xdp_bulkq` (`struct xdp_dev_bulk_queue`) of `DEV_MAP_BULK_SIZE` frames, allocated by the `NETDEV_REGISTER` notifier when `ndo_xdp_xmit` is set; per-CPU staging means the datapath is lockless except for `local_lock_nested_bh` (PREEMPT_RT only). On `XDP_REDIRECT-to-devmap`: `__xdp_enqueue` validates dev XDP features (`NETDEV_XDP_ACT_NDO_XMIT[_SG]`), then `bq_enqueue` stages; on `xdp_do_flush()` the per-CPU flush_list is drained — each `bq_xmit_all` optionally runs the per-entry XDP prog (`dev_map_bpf_prog_run`), then `dev->netdev_ops->ndo_xdp_xmit(... XDP_XMIT_FLUSH)`. Per-`BPF_F_BROADCAST` flag (with optional `BPF_F_EXCLUDE_INGRESS`): `dev_map_enqueue_multi` walks every slot, clones xdp_frames via `xdpf_clone` to all-but-last, enqueues last directly. Per-`NETDEV_UNREGISTER` notifier: walks `dev_map_list` and `cmpxchg`-removes matching slots; `call_rcu(__dev_map_entry_free)` drops the `dev_put` ref. Critical for: per-XDP-forwarding (bridge, router), per-multicast XDP broadcast, per-veth-pair fast-path.

This Tier-3 covers `kernel/bpf/devmap.c` (~1198 lines).

### Acceptance Criteria

- [ ] AC-1: dev_map_alloc_check rejects value_size not in {ifindex-end, bpf_prog.fd-end} with -EINVAL.
- [ ] AC-2: dev_map_alloc_check rejects DEVMAP_HASH max_entries > 1UL << 31 with -EINVAL.
- [ ] AC-3: dev_map_init_map forces map_flags |= BPF_F_RDONLY_PROG.
- [ ] AC-4: DEVMAP_HASH n_buckets = roundup_pow_of_two(max_entries).
- [ ] AC-5: __dev_map_alloc_node rejects xdp_prog with expected_attach_type != BPF_XDP_DEVMAP.
- [ ] AC-6: __dev_map_update_elem with ifindex==0 ∧ bpf_prog.fd > 0 returns -EINVAL.
- [ ] AC-7: __dev_map_update_elem xchg then call_rcu(__dev_map_entry_free) on old; atomic_inc items on fresh slot.
- [ ] AC-8: __dev_map_hash_update_elem caps live items at max_entries with -E2BIG.
- [ ] AC-9: __xdp_enqueue rejects dev lacking NETDEV_XDP_ACT_NDO_XMIT with -EOPNOTSUPP.
- [ ] AC-10: __xdp_enqueue rejects multi-frag xdpf on dev without NETDEV_XDP_ACT_NDO_XMIT_SG with -EOPNOTSUPP.
- [ ] AC-11: bq_enqueue auto-flushes when count reaches DEV_MAP_BULK_SIZE.
- [ ] AC-12: __dev_flush invokes bq_xmit_all with XDP_XMIT_FLUSH and clears dev_rx + xdp_prog.
- [ ] AC-13: bq_xmit_all frees unsent frames via xdp_return_frame_rx_napi on partial / failed ndo_xdp_xmit.
- [ ] AC-14: dev_map_enqueue_multi clones to all-but-last, enqueues original to last; empty map ⟹ xdp_return_frame_rx_napi.
- [ ] AC-15: dev_map_enqueue_multi with exclude_ingress excludes dev_rx + all upper devs (MAX_NEST_DEV).
- [ ] AC-16: dev_map_notification NETDEV_REGISTER allocates per-CPU xdp_bulkq iff ndo_xdp_xmit set and !xdp_bulkq.
- [ ] AC-17: dev_map_notification NETDEV_UNREGISTER cmpxchg-removes entries; DEVMAP_HASH path uses index_lock.
- [ ] AC-18: dev_map_free synchronize_rcu + rcu_barrier before tearing entries.
- [ ] AC-19: dev_map_redirect accepts BPF_F_BROADCAST|BPF_F_EXCLUDE_INGRESS only.
- [ ] AC-20: BUILD_BUG_ON tracepoint shadow offset matches.

### Architecture

```
struct BpfDtab {
  map: BpfMap,
  netdev_map: Option<RcuArray<Option<Arc<BpfDtabNetdev>>>>,  // DEVMAP
  list: ListNode,                                            // dev_map_list linkage
  dev_index_head: Option<Box<[HlistHead]>>,                  // DEVMAP_HASH
  index_lock: SpinLock,                                       // DEVMAP_HASH
  items: AtomicU32,
  n_buckets: u32,                                            // DEVMAP_HASH
}

struct BpfDtabNetdev {
  dev: NetDevice,                          // must be first member
  index_hlist: HlistNode,                  // DEVMAP_HASH only
  xdp_prog: Option<BpfProg>,                // expected_attach_type == BPF_XDP_DEVMAP
  rcu: RcuHead,
  idx: u32,
  val: BpfDevmapVal,                        // ifindex, bpf_prog {fd, id}
}

struct XdpDevBulkQueue {                    // net_device.xdp_bulkq per-CPU
  q: [XdpFrame; DEV_MAP_BULK_SIZE],
  flush_node: ListHead,
  dev: *NetDevice,
  dev_rx: Option<*NetDevice>,
  xdp_prog: Option<BpfProg>,
  count: u32,
  bq_lock: LocalLock,
}
```

`DevMap::alloc_check(attr) -> i32`:
1. Validate max_entries > 0, key_size == 4, value_size variant, flags ⊆ DEV_CREATE_FLAG_MASK.
2. For DEVMAP_HASH: max_entries ≤ 1<<31.

`DevMap::alloc(attr) -> BpfMap`:
1. Allocate dtab; init_map (forces BPF_F_RDONLY_PROG).
2. For DEVMAP_HASH: alloc hlist_head[n_buckets = roundup_pow_of_two], init index_lock.
3. For DEVMAP: alloc netdev_map[max_entries].
4. Under dev_map_lock: list_add_tail_rcu to dev_map_list.

`DevMap::update_elem(map, key, value, flags)` (DEVMAP):
1. Validate flags ≤ BPF_EXIST, key < max_entries, flags != BPF_NOEXIST, !ifindex ⟹ no fd.
2. dev = ifindex ? alloc_node(net, dtab, &val, i) : None.
3. old = xchg(netdev_map[i], dev).
4. If old: call_rcu(old.rcu, entry_free); else: items++.

`DevMap::hash_update_elem(map, key, value, flags)` (DEVMAP_HASH):
1. Validate flags ≤ BPF_EXIST, val.ifindex != 0.
2. Lock index_lock.
3. old = hash_lookup(map, idx).
4. If old ∧ flags == BPF_NOEXIST: -EEXIST.
5. dev = alloc_node(...); fail ⟹ unlock+err.
6. If old: hlist_del_rcu(old). Else if items ≥ max: unlock+free new+-E2BIG. Else items++.
7. hlist_add_head_rcu(&dev.index_hlist, bucket(idx)).
8. Unlock; old ⟹ call_rcu(entry_free).

`DevMap::alloc_node(net, dtab, val, idx) -> BpfDtabNetdev`:
1. Alloc node GFP_NOWAIT NUMA-local.
2. dev.dev = dev_get_by_index(net, val.ifindex) — bumps net_device ref.
3. If val.bpf_prog.fd > 0: prog = bpf_prog_get_type_dev; reject if expected_attach_type != BPF_XDP_DEVMAP or !bpf_prog_map_compatible.
4. Stash idx, val.ifindex, prog + prog.aux.id.

`DevMap::entry_free(rcu)`:
1. bpf_prog_put(xdp_prog) if set.
2. dev_put(dev.dev).
3. kfree(dev).

`DevMap::redirect(map, ifindex, flags)`:
1. return __bpf_xdp_redirect_map(map, ifindex, flags, BPF_F_BROADCAST|BPF_F_EXCLUDE_INGRESS, __dev_map_lookup_elem).

`DevMap::xdp_enqueue(dev, xdpf, dev_rx, xdp_prog)`:
1. Check NETDEV_XDP_ACT_NDO_XMIT.
2. Check NETDEV_XDP_ACT_NDO_XMIT_SG iff frame has frags.
3. xdp_ok_fwd_dev(dev, frame_len).
4. bq_enqueue(dev, xdpf, dev_rx, xdp_prog).

`DevMap::bq_enqueue(dev, xdpf, dev_rx, xdp_prog)`:
1. local_lock_nested_bh(&dev.xdp_bulkq.bq_lock).
2. bq = this_cpu_ptr(dev.xdp_bulkq).
3. If bq.count == DEV_MAP_BULK_SIZE: bq_xmit_all(bq, 0).
4. If !bq.dev_rx: bq.dev_rx = dev_rx; bq.xdp_prog = xdp_prog; list_add(&bq.flush_node, per-CPU flush_list).
5. bq.q[bq.count++] = xdpf.
6. local_unlock_nested_bh.

`DevMap::bq_xmit_all(bq, flags)`:
1. lockdep_assert_held(bq.bq_lock); skip if !count.
2. Prefetch each frame.
3. If bq.xdp_prog: to_send = bpf_prog_run(xdp_prog, q, count, dev, dev_rx); 0 ⟹ skip ndo_xdp_xmit.
4. sent = dev.netdev_ops.ndo_xdp_xmit(dev, to_send, q, flags); <0 ⟹ err = sent; sent = 0.
5. Free unsent (sent..to_send) via xdp_return_frame_rx_napi.
6. count = 0; trace_xdp_devmap_xmit(dev_rx, dev, sent, cnt - sent, err).

`DevMap::flush(flush_list)`:
1. For each bq in flush_list (per-CPU): lock; bq_xmit_all(bq, XDP_XMIT_FLUSH); clear dev_rx + xdp_prog; del from list; unlock.

`DevMap::bpf_prog_run(xdp_prog, frames, n, tx_dev, rx_dev)`:
1. txq = {tx_dev}; rxq = {rx_dev}.
2. For each frame: convert to buff, run prog. XDP_PASS retained; everything else freed via xdp_return_frame_rx_napi + traces.

`DevMap::enqueue_multi(xdpf, dev_rx, map, exclude_ingress)`:
1. If exclude_ingress: excluded = upper-ifindexes(dev_rx) ++ [dev_rx.ifindex].
2. Walk all slots (DEVMAP) or buckets (DEVMAP_HASH); skip !is_valid_dst or excluded.
3. last_dst = first valid; for subsequent: enqueue_clone(last_dst); last_dst = dst.
4. Finally: bq_enqueue(last_dst.dev, xdpf, dev_rx, last_dst.xdp_prog) — original frame consumed.
5. Empty map ⟹ xdp_return_frame_rx_napi(xdpf).

`DevMap::generic_redirect(dst, skb, xdp_prog)`:
1. xdp_ok_fwd_dev(dst.dev, skb.len).
2. If bpf_prog_run_skb(skb, dst) != XDP_PASS: return 0 (handled inside).
3. skb.dev = dst.dev; generic_xdp_tx(skb, xdp_prog).

`DevMap::notification(notifier, event, ptr)`:
1. NETDEV_REGISTER: alloc per-CPU xdp_bulkq iff ndo_xdp_xmit ∧ !already allocated.
2. NETDEV_UNREGISTER: under rcu_read_lock, walk dev_map_list; per-dtab DEVMAP ⟹ cmpxchg-clear matching slots; per-dtab DEVMAP_HASH ⟹ hash_remove_netdev under index_lock.

`DevMap::free(map)`:
1. Remove from dev_map_list under dev_map_lock.
2. synchronize_rcu — drain XDP readers + bpf_redirect_info.
3. rcu_barrier — wait for outstanding entry_free callbacks.
4. Walk all entries: bpf_prog_put + dev_put + kfree.
5. Free index_head / netdev_map; free dtab.

### Out of Scope

- include/net/xdp.h `xdp_do_redirect` / `xdp_do_flush` / `xdpf_clone` core (covered in `net/xdp.md` Tier-3)
- kernel/bpf/cpumap.c (covered in `cpumap.md` Tier-3)
- net/core/dev.c `generic_xdp_tx` + `netdev_for_each_upper_dev_rcu` (covered in `net/core/dev.md` Tier-3 if expanded)
- net/core/filter.c `__bpf_xdp_redirect_map` (covered in `net/core/filter.md` Tier-3 if expanded)
- Driver-side `ndo_xdp_xmit` implementations (out of scope — per-driver)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct bpf_dtab` | per-map (devmap + devmap_hash) | `BpfDtab` |
| `struct bpf_dtab_netdev` | per-slot entry | `BpfDtabNetdev` |
| `struct xdp_dev_bulk_queue` | per-CPU bulk staging on net_device | `XdpDevBulkQueue` |
| `dev_map_alloc_check()` | per-attr sanity | `DevMap::alloc_check` |
| `dev_map_alloc()` | per-map alloc + list_add_tail_rcu | `DevMap::alloc` |
| `dev_map_init_map()` | per-type init (array vs hash) | `DevMap::init_map` |
| `dev_map_free()` | per-map teardown | `DevMap::free` |
| `dev_map_update_elem()` / `dev_map_hash_update_elem()` | per-slot install | `DevMap::update_elem`/`hash_update_elem` |
| `__dev_map_alloc_node()` | per-entry construct (dev_get_by_index + prog) | `DevMap::alloc_node` |
| `dev_map_delete_elem()` / `dev_map_hash_delete_elem()` | per-slot remove | `DevMap::delete_elem`/`hash_delete_elem` |
| `__dev_map_entry_free()` | per-RCU-callback entry free | `DevMap::entry_free` |
| `dev_map_lookup_elem()` / `dev_map_hash_lookup_elem()` | per-syscall lookup | `DevMap::lookup_elem`/`hash_lookup_elem` |
| `__dev_map_lookup_elem()` / `__dev_map_hash_lookup_elem()` | per-XDP fast lookup | `DevMap::lookup`/`hash_lookup` |
| `dev_map_get_next_key()` / `dev_map_hash_get_next_key()` | per-iter walk | `DevMap::next_key`/`hash_next_key` |
| `dev_map_redirect()` / `dev_hash_map_redirect()` | per-XDP redirect dispatch | `DevMap::redirect`/`hash_redirect` |
| `__xdp_enqueue()` | per-frame stage + features check | `DevMap::xdp_enqueue` |
| `bq_enqueue()` | per-CPU bulkq add | `DevMap::bq_enqueue` |
| `bq_xmit_all()` | per-bulkq drain via ndo_xdp_xmit | `DevMap::bq_xmit_all` |
| `dev_map_bpf_prog_run()` | per-batch secondary XDP run | `DevMap::bpf_prog_run` |
| `dev_map_bpf_prog_run_skb()` | per-skb generic-XDP run | `DevMap::bpf_prog_run_skb` |
| `__dev_flush()` | per-NAPI flush_list drain | `DevMap::flush` |
| `dev_map_enqueue()` / `dev_xdp_enqueue()` | per-XDP-frame entry | `DevMap::enqueue` / `DevMap::xdp_enqueue_direct` |
| `dev_map_enqueue_multi()` | per-broadcast XDP-frame fanout | `DevMap::enqueue_multi` |
| `dev_map_enqueue_clone()` | per-clone XDP-frame | `DevMap::enqueue_clone` |
| `dev_map_generic_redirect()` | per-skb generic redirect | `DevMap::generic_redirect` |
| `dev_map_redirect_multi()` | per-broadcast skb fanout | `DevMap::redirect_multi` |
| `dev_map_redirect_clone()` | per-clone skb | `DevMap::redirect_clone` |
| `is_valid_dst()` | per-frame feature check | `DevMap::is_valid_dst` |
| `is_ifindex_excluded()` | per-broadcast exclude-list match | `DevMap::is_ifindex_excluded` |
| `get_upper_ifindexes()` | per-broadcast upper-dev list | `DevMap::get_upper_ifindexes` |
| `dev_map_hash_remove_netdev()` | per-NETDEV_UNREGISTER hash sweep | `DevMap::hash_remove_netdev` |
| `dev_map_notification()` | per-netdev REG/UNREG hook | `DevMap::notification` |
| `dev_map_mem_usage()` | per-fdinfo accounting | `DevMap::mem_usage` |
| `dev_map_ops` / `dev_map_hash_ops` | per-map_ops vtables | `DevMap::OPS` / `DevMap::HASH_OPS` |

### compatibility contract

REQ-1: struct bpf_dtab:
- map: embedded struct bpf_map.
- netdev_map: __rcu **bpf_dtab_netdev — used only for DEVMAP (linear, size = max_entries).
- list: linkage into global dev_map_list (under dev_map_lock).
- dev_index_head: *hlist_head[n_buckets] — used only for DEVMAP_HASH.
- index_lock: spinlock — protects dev_index_head bucket mutations.
- items: unsigned int — count of present entries (atomic_t aliased for DEVMAP).
- n_buckets: u32 — power-of-2; only for DEVMAP_HASH.

REQ-2: struct bpf_dtab_netdev:
- dev: *net_device — must be first member (tracepoint shadow `_bpf_dtab_netdev`).
- index_hlist: hlist_node — only used in DEVMAP_HASH chains.
- xdp_prog: optional *bpf_prog (BPF_PROG_TYPE_XDP, expected_attach_type BPF_XDP_DEVMAP).
- rcu: rcu_head — deferred-free vehicle.
- idx: u32 — original key (== ifindex for DEVMAP_HASH; == array index for DEVMAP).
- val: struct bpf_devmap_val (ifindex, bpf_prog.{fd,id}).

REQ-3: struct xdp_dev_bulk_queue (`net_device.xdp_bulkq` per-CPU):
- q[DEV_MAP_BULK_SIZE] — staged xdp_frames.
- flush_node: list_head — into per-CPU dev flush_list.
- dev: *net_device — egress (back-ref).
- dev_rx: *net_device — ingress (captured per-batch; same for all frames in a bulk).
- xdp_prog: optional *bpf_prog — secondary prog associated with this batch.
- count: unsigned int.
- bq_lock: local_lock_t.

REQ-4: dev_map_alloc_check(attr):
- if max_entries == 0 ∨ key_size != 4: return -EINVAL.
- /* value_size in {offsetofend(ifindex), offsetofend(bpf_prog.fd)} */
- if valsize ∉ {4-after-ifindex, 4-after-bpf_prog.fd}: return -EINVAL.
- if map_flags & ~(BPF_F_NUMA_NODE|BPF_F_RDONLY|BPF_F_WRONLY): return -EINVAL.
- if map_type == BPF_MAP_TYPE_DEVMAP_HASH ∧ max_entries > 1UL << 31: return -EINVAL.
- return 0.

REQ-5: dev_map_init_map(dtab, attr):
- attr.map_flags |= BPF_F_RDONLY_PROG — verifier blocks BPF-side writes to lookup result.
- bpf_map_init_from_attr(&dtab.map, attr).
- if BPF_MAP_TYPE_DEVMAP_HASH:
  - dtab.n_buckets = roundup_pow_of_two(max_entries).
  - dtab.dev_index_head = dev_map_create_hash(n_buckets, numa).
  - spin_lock_init(&dtab.index_lock).
- else:
  - dtab.netdev_map = bpf_map_area_alloc(max_entries * sizeof(*entry)).

REQ-6: dev_map_alloc(attr):
- dtab = bpf_map_area_alloc(sizeof(*dtab)).
- dev_map_init_map(dtab, attr) — fail ⟹ free + ERR.
- spin_lock(&dev_map_lock); list_add_tail_rcu(&dtab.list, &dev_map_list); spin_unlock.
- return &dtab.map.

REQ-7: dev_map_free(map):
- spin_lock(&dev_map_lock); list_del_rcu(&dtab.list); spin_unlock.
- synchronize_rcu — drain XDP + bpf_redirect_info readers.
- rcu_barrier — wait for prior __dev_map_entry_free callbacks.
- if DEVMAP_HASH:
  - for i in 0..n_buckets: walk hlist, hlist_del_rcu, bpf_prog_put + dev_put + kfree per entry.
  - bpf_map_area_free(dtab.dev_index_head).
- else:
  - for i in 0..max_entries: dereference, bpf_prog_put + dev_put + kfree per entry.
  - bpf_map_area_free(dtab.netdev_map).
- bpf_map_area_free(dtab).

REQ-8: __dev_map_alloc_node(net, dtab, val, idx):
- dev = bpf_map_kmalloc_node(&dtab.map, sizeof(*dev), GFP_NOWAIT, numa).
- dev.dev = dev_get_by_index(net, val.ifindex) — fail ⟹ err_out.
- if val.bpf_prog.fd > 0:
  - prog = bpf_prog_get_type_dev(fd, BPF_PROG_TYPE_XDP, false).
  - if prog.expected_attach_type != BPF_XDP_DEVMAP ∨ !bpf_prog_map_compatible: err_put_prog.
- dev.idx = idx; dev.val.ifindex = val.ifindex.
- if prog: dev.xdp_prog = prog; dev.val.bpf_prog.id = prog.aux.id.
- else: dev.xdp_prog = NULL; dev.val.bpf_prog.id = 0.
- return dev.

REQ-9: __dev_map_update_elem(net, map, key, value, flags):
- /* DEVMAP only */
- if flags > BPF_EXIST ∨ i >= max_entries: -EINVAL / -E2BIG.
- if flags == BPF_NOEXIST: -EEXIST.
- /* ifindex == 0 ⟹ delete-equivalent; fd must also be 0 */
- if !val.ifindex ∧ val.bpf_prog.fd > 0: -EINVAL.
- dev = ifindex ? __dev_map_alloc_node(...) : NULL.
- old = xchg(&dtab.netdev_map[i], RCU_INITIALIZER(dev)).
- if old: call_rcu(&old.rcu, __dev_map_entry_free).
- else: atomic_inc(&dtab.items).
- return 0.

REQ-10: __dev_map_hash_update_elem(net, map, key, value, flags):
- /* DEVMAP_HASH */
- memcpy(&val, value, value_size).
- if flags > BPF_EXIST ∨ !val.ifindex: return -EINVAL.
- spin_lock_irqsave(&dtab.index_lock).
- old = __dev_map_hash_lookup_elem(map, idx).
- if old ∧ flags == BPF_NOEXIST: -EEXIST.
- dev = __dev_map_alloc_node(...).
- if old: hlist_del_rcu(&old.index_hlist).
- else if dtab.items >= max_entries: -E2BIG (call_rcu free of new dev).
- else: dtab.items++.
- hlist_add_head_rcu(&dev.index_hlist, dev_map_index_hash(dtab, idx)).
- spin_unlock_irqrestore.
- if old: call_rcu(&old.rcu, __dev_map_entry_free).

REQ-11: __dev_map_entry_free(rcu):
- dev = container_of(rcu, bpf_dtab_netdev, rcu).
- if dev.xdp_prog: bpf_prog_put.
- dev_put(dev.dev).
- kfree(dev).

REQ-12: dev_map_delete_elem(map, key):
- old = unrcu_pointer(xchg(&dtab.netdev_map[k], NULL)).
- if old: call_rcu(&old.rcu, __dev_map_entry_free); atomic_dec(&dtab.items).

REQ-13: dev_map_hash_delete_elem(map, key):
- spin_lock_irqsave(&dtab.index_lock).
- old = __dev_map_hash_lookup_elem(map, k).
- if old: dtab.items--; hlist_del_init_rcu(&old.index_hlist); call_rcu(&old.rcu, __dev_map_entry_free).
- spin_unlock_irqrestore.

REQ-14: __dev_map_lookup_elem(map, key):
- if key >= max_entries: return NULL.
- return rcu_dereference_check(dtab.netdev_map[key], rcu_read_lock_bh_held()).

REQ-15: __dev_map_hash_lookup_elem(map, key):
- hlist_for_each_entry_rcu(dev, head, index_hlist, lockdep_is_held(&dtab.index_lock)):
  - if dev.idx == key: return dev.

REQ-16: dev_map_redirect(map, ifindex, flags):
- return __bpf_xdp_redirect_map(map, ifindex, flags, BPF_F_BROADCAST|BPF_F_EXCLUDE_INGRESS, __dev_map_lookup_elem).
- /* allowed_flags includes BROADCAST and EXCLUDE_INGRESS */

REQ-17: dev_hash_map_redirect(map, ifindex, flags):
- return __bpf_xdp_redirect_map(map, ifindex, flags, BPF_F_BROADCAST|BPF_F_EXCLUDE_INGRESS, __dev_map_hash_lookup_elem).

REQ-18: __xdp_enqueue(dev, xdpf, dev_rx, xdp_prog):
- /* Features */
- if !(dev.xdp_features & NETDEV_XDP_ACT_NDO_XMIT): return -EOPNOTSUPP.
- if !(dev.xdp_features & NETDEV_XDP_ACT_NDO_XMIT_SG) ∧ xdp_frame_has_frags(xdpf): return -EOPNOTSUPP.
- /* Size */
- if !xdp_ok_fwd_dev(dev, xdp_get_frame_len(xdpf)): return -err.
- bq_enqueue(dev, xdpf, dev_rx, xdp_prog).
- return 0.

REQ-19: bq_enqueue(dev, xdpf, dev_rx, xdp_prog):
- local_lock_nested_bh(&dev.xdp_bulkq.bq_lock).
- bq = this_cpu_ptr(dev.xdp_bulkq).
- if bq.count == DEV_MAP_BULK_SIZE: bq_xmit_all(bq, 0).
- if !bq.dev_rx:
  - bq.dev_rx = dev_rx; bq.xdp_prog = xdp_prog.
  - list_add(&bq.flush_node, bpf_net_ctx_get_dev_flush_list()).
- bq.q[bq.count++] = xdpf.
- local_unlock_nested_bh.

REQ-20: bq_xmit_all(bq, flags):
- lockdep_assert_held(&bq.bq_lock); if !bq.count: return.
- prefetch each bq.q[i].
- to_send = bq.count.
- if bq.xdp_prog: to_send = dev_map_bpf_prog_run(bq.xdp_prog, bq.q, cnt, bq.dev, bq.dev_rx); if 0: goto out.
- sent = bq.dev.netdev_ops.ndo_xdp_xmit(bq.dev, to_send, bq.q, flags).
- if sent < 0: err = sent; sent = 0.
- /* Free unsent */
- for i in sent..to_send: xdp_return_frame_rx_napi(bq.q[i]).
- out: bq.count = 0; trace_xdp_devmap_xmit(bq.dev_rx, bq.dev, sent, cnt - sent, err).

REQ-21: __dev_flush(flush_list):
- list_for_each_entry_safe(bq, tmp, flush_list, flush_node):
  - local_lock_nested_bh(&bq.dev.xdp_bulkq.bq_lock).
  - bq_xmit_all(bq, XDP_XMIT_FLUSH).
  - bq.dev_rx = NULL; bq.xdp_prog = NULL.
  - __list_del_clearprev(&bq.flush_node).
  - local_unlock_nested_bh.

REQ-22: dev_map_bpf_prog_run(xdp_prog, frames, n, tx_dev, rx_dev):
- xdp.txq = {.dev = tx_dev}; xdp.rxq = {.dev = rx_dev}.
- for i in 0..n:
  - xdp_convert_frame_to_buff(xdpf, &xdp); xdp.txq = ...; xdp.rxq = ...
  - act = bpf_prog_run_xdp(xdp_prog, &xdp).
  - XDP_PASS ⟹ xdp_update_frame_from_buff; success ⟹ frames[nframes++]; fail ⟹ xdp_return_frame_rx_napi.
  - default ⟹ bpf_warn_invalid_xdp_action.
  - XDP_ABORTED ⟹ trace_xdp_exception.
  - XDP_DROP ⟹ xdp_return_frame_rx_napi.
- return nframes.

REQ-23: dev_xdp_enqueue(dev, xdpf, dev_rx):
- return __xdp_enqueue(dev, xdpf, dev_rx, NULL) — no secondary prog.

REQ-24: dev_map_enqueue(dst, xdpf, dev_rx):
- return __xdp_enqueue(dst.dev, xdpf, dev_rx, dst.xdp_prog).

REQ-25: dev_map_enqueue_multi(xdpf, dev_rx, map, exclude_ingress):
- if exclude_ingress: excluded[0..] = upper-ifindexes(dev_rx) + dev_rx.ifindex.
- last_dst = NULL.
- if DEVMAP:
  - for i in 0..max_entries: dst = rcu_deref; skip if !is_valid_dst ∨ excluded.
  - if !last_dst: last_dst = dst; continue.
  - dev_map_enqueue_clone(last_dst, dev_rx, xdpf); last_dst = dst.
- else (DEVMAP_HASH): same logic walking hlist buckets.
- if last_dst: bq_enqueue(last_dst.dev, xdpf, dev_rx, last_dst.xdp_prog).  /* original consumed */
- else: xdp_return_frame_rx_napi(xdpf).  /* empty map */
- return 0.

REQ-26: dev_map_enqueue_clone(obj, dev_rx, xdpf):
- nxdpf = xdpf_clone(xdpf).
- if !nxdpf: return -ENOMEM.
- bq_enqueue(obj.dev, nxdpf, dev_rx, obj.xdp_prog).
- return 0.

REQ-27: is_valid_dst(obj, xdpf):
- !obj ⟹ false.
- !NETDEV_XDP_ACT_NDO_XMIT ⟹ false.
- !NETDEV_XDP_ACT_NDO_XMIT_SG ∧ xdp_frame_has_frags ⟹ false.
- !xdp_ok_fwd_dev(obj.dev, xdp_get_frame_len(xdpf)) ⟹ false.
- else true.

REQ-28: get_upper_ifindexes(dev, indexes, max):
- netdev_for_each_upper_dev_rcu(dev, upper, iter):
  - if n >= max: return -EOVERFLOW.
  - indexes[n++] = upper.ifindex.
- return n.

REQ-29: dev_map_generic_redirect(dst, skb, xdp_prog):
- xdp_ok_fwd_dev(dst.dev, skb.len); fail ⟹ -err.
- if dev_map_bpf_prog_run_skb(skb, dst) != XDP_PASS: return 0  /* dropped/redirected internally */.
- skb.dev = dst.dev; generic_xdp_tx(skb, xdp_prog).
- return 0.

REQ-30: dev_map_bpf_prog_run_skb(skb, dst):
- if !dst.xdp_prog: return XDP_PASS.
- __skb_pull(skb, mac_len); xdp.txq = {.dev = dst.dev}.
- act = bpf_prog_run_generic_xdp(skb, &xdp, dst.xdp_prog).
- XDP_PASS ⟹ __skb_push(skb, mac_len).
- default ⟹ bpf_warn_invalid_xdp_action.
- XDP_ABORTED ⟹ trace_xdp_exception.
- XDP_DROP ⟹ kfree_skb.
- return act.

REQ-31: dev_map_redirect_multi(dev, skb, xdp_prog, map, exclude_ingress):
- Mirror of enqueue_multi for SKB-path:
  - dev_map_redirect_clone for all-but-last → dev_map_generic_redirect for last.
- If map empty: consume_skb.

REQ-32: dev_map_redirect_clone(dst, skb, xdp_prog):
- nskb = skb_clone(skb, GFP_ATOMIC); fail ⟹ -ENOMEM.
- dev_map_generic_redirect(dst, nskb, xdp_prog).
- fail ⟹ consume_skb(nskb).

REQ-33: dev_map_notification(notifier, event, ptr):
- netdev = netdev_notifier_info_to_dev(ptr).
- case NETDEV_REGISTER:
  - if !netdev.netdev_ops.ndo_xdp_xmit ∨ netdev.xdp_bulkq already set: break.
  - netdev.xdp_bulkq = alloc_percpu(xdp_dev_bulk_queue).
  - for_each_possible_cpu(cpu): bq.dev = netdev; local_lock_init(&bq.bq_lock).
- case NETDEV_UNREGISTER:
  - rcu_read_lock; list_for_each_entry_rcu(dtab, &dev_map_list, list):
    - if DEVMAP_HASH: dev_map_hash_remove_netdev(dtab, netdev); continue.
    - else for i in 0..max_entries:
      - dev = rcu_dereference(dtab.netdev_map[i]).
      - if dev ∧ dev.dev == netdev:
        - odev = unrcu_pointer(cmpxchg(&dtab.netdev_map[i], RCU_INITIALIZER(dev), NULL)).
        - if odev == dev: call_rcu(&dev.rcu, __dev_map_entry_free); atomic_dec(&dtab.items).
  - rcu_read_unlock.
- return NOTIFY_OK.

REQ-34: dev_map_hash_remove_netdev(dtab, netdev):
- spin_lock_irqsave(&dtab.index_lock).
- for i in 0..n_buckets: hlist_for_each_entry_safe — if dev.dev == netdev: items--; hlist_del_rcu; call_rcu(__dev_map_entry_free).
- spin_unlock_irqrestore.

REQ-35: dev_map_mem_usage(map):
- usage = sizeof(*dtab).
- DEVMAP_HASH ⟹ + n_buckets * sizeof(hlist_head).
- DEVMAP ⟹ + max_entries * sizeof(*entry-ptr).
- + items * sizeof(bpf_dtab_netdev).

REQ-36: dev_map_init (subsys_initcall):
- BUILD_BUG_ON offsetof(bpf_dtab_netdev, dev) != offsetof(_bpf_dtab_netdev, dev) — tracepoint shadow.
- register_netdevice_notifier(&dev_map_notifier).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `max_entries_devmap_hash_bounded` | INVARIANT | per-alloc_check: DEVMAP_HASH max_entries ≤ 1<<31. |
| `n_buckets_power_of_two` | INVARIANT | per-init: n_buckets = roundup_pow_of_two(max_entries). |
| `value_size_variant` | INVARIANT | per-alloc_check: value_size ∈ {ifindex-end, prog.fd-end}. |
| `rdonly_prog_forced` | INVARIANT | per-init: map_flags |= BPF_F_RDONLY_PROG. |
| `prog_type_devmap` | INVARIANT | per-alloc_node: prog.expected_attach_type == BPF_XDP_DEVMAP. |
| `bulkq_count_bounded` | INVARIANT | per-bq: 0 ≤ count ≤ DEV_MAP_BULK_SIZE. |
| `dev_features_ndo_xdp_xmit` | INVARIANT | per-xdp_enqueue: dev.xdp_features & NETDEV_XDP_ACT_NDO_XMIT. |
| `dev_features_xmit_sg_for_frags` | INVARIANT | per-xdp_enqueue: frags ⟹ NETDEV_XDP_ACT_NDO_XMIT_SG. |
| `unsent_freed_via_xdp_return_frame_rx_napi` | INVARIANT | per-bq_xmit_all: sent..to_send freed. |
| `cmpxchg_on_unregister_protects_concurrent_update` | INVARIANT | per-NETDEV_UNREGISTER: cmpxchg(old, NULL) ensures slot still references unregistering dev. |
| `dev_get_then_dev_put_balanced` | INVARIANT | per-entry: dev_get_by_index paired with dev_put in entry_free. |
| `xdp_bulkq_freed_in_free_netdev` | INVARIANT | per-alloc_percpu in REGISTER, freed by free_netdev. |
| `tracepoint_shadow_offset_matches` | INVARIANT | per-BUILD_BUG_ON in init: bpf_dtab_netdev.dev == _bpf_dtab_netdev.dev. |

### Layer 2: TLA+

`kernel/bpf/devmap.tla`:
- Per-XDP-redirect + per-bulkq-stage + per-flush + per-broadcast-clone + per-update/delete + per-NETDEV_UNREGISTER notifier.
- Properties:
  - `safety_no_uaf_on_unregister` — per-NETDEV_UNREGISTER: removal completes before final entry_free; XDP readers under rcu_read_lock_bh.
  - `safety_no_double_xmit` — per-bq: each frame xmit at most once (either ndo_xdp_xmit or xdp_return_frame_rx_napi).
  - `safety_broadcast_n_minus_one_clones` — per-broadcast: original consumed at last_dst; clones for all other (n-1) valid slots.
  - `safety_exclude_ingress_excludes_all_uppers` — per-exclude_ingress: dev_rx + all upper-devs excluded.
  - `liveness_eventually_flush` — per-frame enqueued: drained on xdp_do_flush.
  - `liveness_old_entry_freed` — per-update: old entry freed after one RCU grace period.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `DevMap::alloc_check` post: validated key_size + value_size + flags + max_entries | `DevMap::alloc_check` |
| `DevMap::alloc_node` post: dev_get_by_index ref held; prog type validated | `DevMap::alloc_node` |
| `DevMap::update_elem` post: xchg + call_rcu on old; items++ on fresh | `DevMap::update_elem` |
| `DevMap::hash_update_elem` post: items capped at max_entries | `DevMap::hash_update_elem` |
| `DevMap::xdp_enqueue` post: feature-validated then staged in bulkq | `DevMap::xdp_enqueue` |
| `DevMap::bq_xmit_all` post: bq.count == 0 ∧ unsent frames freed | `DevMap::bq_xmit_all` |
| `DevMap::flush` post: each bq drained + cleared + delisted | `DevMap::flush` |
| `DevMap::enqueue_multi` post: original frame consumed-or-returned | `DevMap::enqueue_multi` |
| `DevMap::notification` post: REGISTER allocates per-CPU bulkq; UNREGISTER detaches via cmpxchg | `DevMap::notification` |
| `DevMap::free` post: list_del + synchronize_rcu + rcu_barrier + free all entries | `DevMap::free` |

### Layer 4: Verus/Creusot functional

`Per-NAPI XDP_REDIRECT → __bpf_xdp_redirect_map(lookup) → dev_map_enqueue → __xdp_enqueue (features+size) → bq_enqueue → xdp_do_flush → __dev_flush → bq_xmit_all (optional dev_map_bpf_prog_run) → ndo_xdp_xmit(XDP_XMIT_FLUSH) → trace_xdp_devmap_xmit` semantic equivalence: per-Documentation/networking/xdp.rst + `tools/testing/selftests/bpf/prog_tests/xdp_devmap_attach.c` + `Documentation/bpf/map_devmap.rst`.

`Per-broadcast: __bpf_xdp_redirect_map(BPF_F_BROADCAST) → dev_map_enqueue_multi → for each valid_dst except last: dev_map_enqueue_clone(xdpf_clone) → last: bq_enqueue(original)` semantic equivalence.

`Per-netdev-unregister: NETDEV_UNREGISTER notifier → walk dev_map_list → cmpxchg-clear DEVMAP slot OR hlist_del_rcu DEVMAP_HASH entry → call_rcu(__dev_map_entry_free) → bpf_prog_put + dev_put` semantic equivalence.

### hardening

(Inherits row-1 features from `kernel/bpf/00-overview.md` § Hardening.)

DevMap reinforcement:

- **Per-BPF_F_RDONLY_PROG forced on init** — defense against per-BPF-side overwrite of lookup result.
- **Per-NETDEV_XDP_ACT_NDO_XMIT feature gate** — defense against per-redirect to non-XDP-capable dev.
- **Per-NETDEV_XDP_ACT_NDO_XMIT_SG gate for multi-frag frames** — defense against per-driver-overrun on jumbo XDP.
- **Per-xdp_ok_fwd_dev MTU + features check** — defense against per-oversized-frame.
- **Per-prog expected_attach_type == BPF_XDP_DEVMAP** — defense against per-wrong-prog-type attach.
- **Per-dev_get_by_index balanced with dev_put in entry_free** — defense against per-net_device leak.
- **Per-update xchg + call_rcu(__dev_map_entry_free)** — defense against per-UAF on in-flight frames.
- **Per-DEVMAP_HASH index_lock irqsave** — defense against per-hlist-race with notifier.
- **Per-DEVMAP_HASH items capped at max_entries** — defense against per-unbounded growth.
- **Per-cmpxchg on NETDEV_UNREGISTER** — defense against per-concurrent-update-loss race.
- **Per-CPU xdp_bulkq + local_lock_nested_bh** — defense against per-bulkq corruption (PREEMPT_RT).
- **Per-unsent xdp_return_frame_rx_napi on partial ndo_xdp_xmit** — defense against per-frame leak.
- **Per-synchronize_rcu + rcu_barrier on free** — defense against per-UAF in flush + outstanding callbacks.
- **Per-BUILD_BUG_ON tracepoint shadow offset** — defense against per-tracepoint-OOB on layout drift.
- **Per-broadcast clone via xdpf_clone (n-1 clones)** — defense against per-double-xmit of original.
- **Per-exclude_ingress upper-dev walk bounded by MAX_NEST_DEV** — defense against per-upper-dev recursion.

