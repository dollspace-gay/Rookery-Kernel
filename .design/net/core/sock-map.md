# Tier-3: net/core/sock_map.c — BPF sockmap / sockhash

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/core/00-overview.md
upstream-paths:
  - net/core/sock_map.c (~1959 lines)
  - include/linux/skmsg.h (struct sk_psock, struct sk_psock_progs, struct sk_psock_link)
  - include/uapi/linux/bpf.h (BPF_MAP_TYPE_SOCKMAP, BPF_MAP_TYPE_SOCKHASH, BPF_SK_*, BPF_LINK_TYPE_SOCKMAP)
  - net/core/skmsg.c (sk_psock helpers used by sock_map)
-->

## Summary

`BPF_MAP_TYPE_SOCKMAP` and `BPF_MAP_TYPE_SOCKHASH` are BPF maps whose values are `struct sock *`. Per-sockmap (`struct bpf_stab`): u32-indexed array of socket pointers. Per-sockhash (`struct bpf_shtab`): variable-key hash table with `struct bpf_shtab_bucket` (hlist + spinlock) buckets, `struct bpf_shtab_elem { rcu_head; u32 hash; struct sock *sk; hlist_node node; u8 key[]; }`. Per-attached socket: `struct sk_psock` (in `include/linux/skmsg.h`) holds the per-socket BPF program state and a back-link list of map-membership records (`struct sk_psock_link`). Per-map: `struct sk_psock_progs { msg_parser; stream_parser; stream_verdict; skb_verdict; *_link; }` — BPF programs attached to the map and inherited by inserted sockets. Per-attach types: `BPF_SK_SKB_STREAM_PARSER` (TCP framing), `BPF_SK_SKB_STREAM_VERDICT` / `BPF_SK_SKB_VERDICT` (per-skb verdict, redirect), `BPF_SK_MSG_VERDICT` (sendmsg verdict). Per-redirect helpers: `bpf_sk_redirect_map` / `_hash` (skb redirect) and `bpf_msg_redirect_map` / `_hash` (sk_msg redirect). Per-program installation: on insert, `sock_map_link()` allocates psock if absent, copies prog refs from `progs`, replaces `sk->sk_prot` via `psock_update_sk_prot` and starts strparser / verdict callback. Per-data path: incoming data triggers `data_ready` → strparser frames → BPF verdict → `__sock_map_lookup_elem` → redirect to another socket's ingress/egress. Critical for: zero-copy socket splicing, kTLS interposition, load balancing, sockmap-based proxies.

This Tier-3 covers `net/core/sock_map.c` (~1959 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct bpf_stab` | sockmap array | `BpfStab` |
| `struct bpf_shtab` | sockhash table | `BpfShtab` |
| `struct bpf_shtab_bucket` | per-hash bucket | `BpfShtabBucket` |
| `struct bpf_shtab_elem` | per-hash element | `BpfShtabElem` |
| `struct sockmap_link` | per-bpf_link wrapper | `SockmapLink` |
| `struct sock_map_seq_info` / `sock_hash_seq_info` | per-iterator | `SockMapSeqInfo` / `SockHashSeqInfo` |
| `sock_map_alloc()` | `.map_alloc` for SOCKMAP | `BpfStab::map_alloc` |
| `sock_hash_alloc()` | `.map_alloc` for SOCKHASH | `BpfShtab::map_alloc` |
| `sock_map_free()` / `sock_hash_free()` | `.map_free` | `BpfStab::map_free` / `BpfShtab::map_free` |
| `sock_map_lookup()` / `sock_hash_lookup()` | `.map_lookup_elem` (refcounted) | `BpfStab::lookup` / `BpfShtab::lookup` |
| `sock_map_lookup_sys()` / `sock_hash_lookup_sys()` | `.map_lookup_elem_sys_only` returns sk_cookie | `BpfStab::lookup_sys` / `BpfShtab::lookup_sys` |
| `__sock_map_lookup_elem()` / `__sock_hash_lookup_elem()` | per-redirect raw lookup | `BpfStab::lookup_raw` / `BpfShtab::lookup_raw` |
| `sock_map_update_elem_sys()` | per-syscall update (sockfd → sock) | `SockMap::update_elem_sys` |
| `sock_map_update_elem()` | `.map_update_elem` (BPF-side; sk = value) | `SockMap::update_elem` |
| `sock_map_update_common()` | per-SOCKMAP insert/replace | `BpfStab::update_common` |
| `sock_hash_update_common()` | per-SOCKHASH insert/replace | `BpfShtab::update_common` |
| `sock_map_delete_elem()` / `sock_hash_delete_elem()` | `.map_delete_elem` | `BpfStab::delete_elem` / `BpfShtab::delete_elem` |
| `sock_map_link()` | per-socket psock setup + prog install | `SockMap::link_psock` |
| `sock_map_init_proto()` | per-socket sk_prot swap | `SockMap::init_proto` |
| `sock_map_psock_get_checked()` | per-existing-psock guard | `SockMap::psock_get_checked` |
| `sock_map_add_link()` / `sock_map_del_link()` | per-psock back-link mgmt | `SockMap::add_link` / `del_link` |
| `sock_map_unref()` | per-link drop on map removal | `SockMap::unref` |
| `sock_map_progs()` | per-map prog struct dispatch | `SockMap::progs` |
| `sock_map_prog_link_lookup()` | per-attach-type prog pointer dispatch | `SockMap::prog_link_lookup` |
| `sock_map_prog_update()` | per-attach/detach/replace | `SockMap::prog_update` |
| `sock_map_get_from_fd()` / `sock_map_prog_detach()` | per-PROG_ATTACH / DETACH syscall | `SockMap::prog_attach_syscall` / `prog_detach_syscall` |
| `sock_map_link_create()` | per-BPF_LINK_CREATE | `SockmapLink::create` |
| `sock_map_link_update_prog()` | per-BPF_LINK_UPDATE | `SockmapLink::update_prog` |
| `sock_map_bpf_prog_query()` | per-BPF_PROG_QUERY | `SockMap::bpf_prog_query` |
| `bpf_sock_map_update` / `bpf_sock_hash_update` | BPF helper from sock_ops | `SockMap::helper_update_sock_ops` |
| `bpf_sk_redirect_map` / `bpf_sk_redirect_hash` | BPF helper: skb verdict redirect | `SockMap::helper_sk_redirect` |
| `bpf_msg_redirect_map` / `bpf_msg_redirect_hash` | BPF helper: sk_msg redirect | `SockMap::helper_msg_redirect` |
| `sock_map_sk_is_suitable()` | per-sk_prot.psock_update_sk_prot != NULL | `SockMap::sk_is_suitable` |
| `sock_map_sk_state_allowed()` | per-state filter (TCP ESTAB/LISTEN, UDP, ...) | `SockMap::sk_state_allowed` |
| `sock_map_redirect_allowed()` | per-redirect-eligibility | `SockMap::redirect_allowed` |
| `sock_map_unhash()` | per-sk_prot.unhash interposition | `SockMap::sk_unhash` |
| `sock_map_destroy()` | per-sk_prot.destroy interposition | `SockMap::sk_destroy` |
| `sock_map_close()` | per-sk_prot.close interposition | `SockMap::sk_close` |
| `sock_map_remove_links()` | per-sk teardown: drop all map memberships | `SockMap::remove_links` |
| `sock_map_unlink()` | per-link dispatcher (SOCKMAP vs SOCKHASH) | `SockMap::unlink` |
| `sock_map_iter_attach_target()` / `_detach_target()` | per-BPF_ITER | `SockMap::iter_attach` / `iter_detach` |
| `sock_map_btf_ids` / `sock_hash_map_btf_ids` | per-BTF-ID registration | `SOCK_MAP_BTF_IDS` / `SOCK_HASH_BTF_IDS` |
| `sock_map_ops` / `sock_hash_ops` | per-map_ops vtable | `SOCK_MAP_OPS` / `SOCK_HASH_OPS` |

## Compatibility contract

REQ-1: struct bpf_stab:
- map: struct bpf_map (header).
- sks: struct sock ** (max_entries-sized array; entries NULL or refcounted sk).
- progs: struct sk_psock_progs (msg_parser, stream_parser, stream_verdict, skb_verdict + 4 bpf_link*).
- lock: spinlock_t (serializes xchg of stab.sks[i] under update/delete).

REQ-2: struct bpf_shtab + bucket + element:
- bpf_shtab: map, buckets, buckets_num (roundup_pow_of_two(max_entries)), elem_size (sizeof(elem) + round_up(key_size, 8)), progs, count (atomic_t).
- bpf_shtab_bucket: hlist_head head; spinlock_t lock.
- bpf_shtab_elem: rcu_head rcu; u32 hash; struct sock *sk; hlist_node node; u8 key[].

REQ-3: sock_map_alloc(attr) — SOCKMAP:
- if max_entries == 0 ∨ key_size != 4 ∨ (value_size != 4 ∧ value_size != 8) ∨ flags & ~(NUMA_NODE|RDONLY|WRONLY): return -EINVAL.
- stab = bpf_map_area_alloc(sizeof(*stab), NUMA_NO_NODE).
- bpf_map_init_from_attr(&stab.map, attr).
- spin_lock_init(&stab.lock).
- stab.sks = bpf_map_area_alloc(max_entries * sizeof(struct sock *), numa_node).

REQ-4: sock_hash_alloc(attr) — SOCKHASH:
- if max_entries == 0 ∨ key_size == 0 ∨ (value_size not in {4,8}) ∨ bad flags: return -EINVAL.
- if key_size > MAX_BPF_STACK: return -E2BIG.
- htab.buckets_num = roundup_pow_of_two(max_entries).
- htab.elem_size = sizeof(struct bpf_shtab_elem) + round_up(key_size, 8).
- if buckets_num == 0 ∨ buckets_num > U32_MAX / sizeof(bucket): return -EINVAL.
- htab.buckets = bpf_map_area_alloc(buckets_num * sizeof(bucket), numa_node).
- for i in 0..buckets_num: INIT_HLIST_HEAD(&buckets[i].head); spin_lock_init(&buckets[i].lock).

REQ-5: sock_map_progs(map):
- if map.map_type == BPF_MAP_TYPE_SOCKMAP: return &container_of(map, bpf_stab, map).progs.
- if map.map_type == BPF_MAP_TYPE_SOCKHASH: return &container_of(map, bpf_shtab, map).progs.
- else: return NULL.

REQ-6: sock_map_prog_link_lookup(map, &pprog, &plink, which):
- progs = sock_map_progs(map). If NULL: -EOPNOTSUPP.
- BPF_SK_MSG_VERDICT       → progs.msg_parser / .msg_parser_link.
- BPF_SK_SKB_STREAM_PARSER → progs.stream_parser / .stream_parser_link (only if CONFIG_BPF_STREAM_PARSER).
- BPF_SK_SKB_STREAM_VERDICT→ progs.stream_verdict / .stream_verdict_link (rejects if skb_verdict already set, -EBUSY).
- BPF_SK_SKB_VERDICT       → progs.skb_verdict / .skb_verdict_link (rejects if stream_verdict already set, -EBUSY).
- default: -EOPNOTSUPP.

REQ-7: sock_map_prog_update(map, prog, old, link, which) — handles 4 cases:
- prog_attach: prog != NULL, old == NULL, link == NULL.
- prog_detach: prog == NULL, old != NULL, link == NULL.
- link_attach: prog != NULL, old == NULL, link != NULL.
- link_detach: prog == NULL, old != NULL, link != NULL.
- If a bpf_link exists for that prog ((!link ∨ prog) ∧ *plink): -EBUSY.
- If old: psock_replace_prog(pprog, prog, old); on success *plink = NULL.
- Else: psock_set_prog(pprog, prog); if link: *plink = link.

REQ-8: sock_map_get_from_fd(attr, prog) — BPF_PROG_ATTACH syscall:
- if attr.attach_flags ∨ attr.replace_bpf_fd: -EINVAL.
- map = __bpf_map_get(target_fd).
- mutex_lock(&sockmap_mutex); sock_map_prog_update(map, prog, NULL, NULL, attr.attach_type); mutex_unlock.

REQ-9: sock_map_prog_detach(attr, ptype) — BPF_PROG_DETACH syscall:
- prog = bpf_prog_get(attr.attach_bpf_fd); ensure prog.type == ptype else -EINVAL.
- sock_map_prog_update(map, NULL, prog, NULL, attr.attach_type) under sockmap_mutex.

REQ-10: sock_map_link(map, sk) — install programs onto a socket joining the map:
- /* Take refs on each prog from map.progs */
- stream_verdict = bpf_prog_inc_not_zero(progs.stream_verdict) (if any); similarly stream_parser, msg_parser, skb_verdict.
- psock = sock_map_psock_get_checked(sk):
  - rcu_read_lock; psock = sk_psock(sk).
  - if psock ∧ sk.sk_prot.close != sock_map_close: -EBUSY.
  - else if !refcount_inc_not_zero(&psock.refcnt): -EBUSY.
- if psock ∧ overlap of (msg_parser, stream_parser, stream_verdict/skb_verdict) with existing psock.progs: -EBUSY.
- else if !psock: psock = sk_psock_init(sk, numa_node).
- Install via psock_set_prog: msg_parser, stream_parser, stream_verdict, skb_verdict.
- ret = sock_map_init_proto(sk, psock):
  - if !sk.sk_prot.psock_update_sk_prot: -EINVAL.
  - psock.psock_update_sk_prot = sk.sk_prot.psock_update_sk_prot.
  - ret = sk.sk_prot.psock_update_sk_prot(sk, psock, false). /* swap sk_prot pointer */
- write_lock_bh(&sk.sk_callback_lock):
  - if stream_parser ∧ stream_verdict ∧ !psock.saved_data_ready:
    - if sk_is_tcp(sk): sk_psock_init_strp(sk, psock); sk_psock_start_strp(sk, psock).
    - else: -EOPNOTSUPP.
  - else if !stream_parser ∧ stream_verdict ∧ !saved_data_ready: sk_psock_start_verdict(sk, psock).
  - else if !stream_verdict ∧ skb_verdict ∧ !saved_data_ready: sk_psock_start_verdict(sk, psock).
- write_unlock_bh.

REQ-11: sock_map_add_link(psock, link, map, link_raw):
- link.link_raw = link_raw; link.map = map.
- spin_lock_bh(&psock.link_lock); list_add_tail(&link.list, &psock.link); unlock.

REQ-12: sock_map_del_link(sk, psock, link_raw) — reverses REQ-11 + stops strp/verdict if last:
- For matching link: walk list, free; if last link with stream_parser → strp_stop; if last with stream_verdict ∨ skb_verdict → verdict_stop.
- If stops needed under write_lock_bh(&sk.sk_callback_lock):
  - if strp_stop: sk_psock_stop_strp(sk, psock).
  - if verdict_stop: sk_psock_stop_verdict(sk, psock).
  - if psock.psock_update_sk_prot: psock.psock_update_sk_prot(sk, psock, false). /* restore sk_prot */

REQ-13: sock_map_update_elem_sys(map, key, value, flags) — syscall path:
- value carries u32 or u64 sockfd ufd.
- sock = sockfd_lookup(ufd); sk = sock.sk.
- if !sock_map_sk_is_suitable(sk): -EOPNOTSUPP. /* needs sk_prot.psock_update_sk_prot */
- sock_map_sk_acquire(sk) = lock_sock(sk) + rcu_read_lock().
- if !sock_map_sk_state_allowed(sk): -EOPNOTSUPP.
- if map_type == SOCKMAP: sock_map_update_common(map, *(u32 *)key, sk, flags).
- else: sock_hash_update_common(map, key, sk, flags).
- sock_map_sk_release(sk) = rcu_read_unlock + release_sock.

REQ-14: sock_map_update_common(map, idx, sk, flags):
- /* called under rcu_read_lock + lock_sock(sk) or bh_lock_sock(sk) */
- if flags > BPF_EXIST: -EINVAL.
- if idx >= max_entries: -E2BIG.
- link = sk_psock_init_link(). On failure -ENOMEM.
- ret = sock_map_link(map, sk). /* attaches psock + progs */
- psock = sk_psock(sk).
- spin_lock_bh(&stab.lock):
  - osk = stab.sks[idx].
  - if osk ∧ flags == BPF_NOEXIST: -EEXIST.
  - if !osk ∧ flags == BPF_EXIST: -ENOENT.
  - sock_map_add_link(psock, link, map, &stab.sks[idx]).
  - stab.sks[idx] = sk.
  - if osk: sock_map_unref(osk, &stab.sks[idx]).

REQ-15: sock_hash_update_common(map, key, sk, flags):
- link = sk_psock_init_link(). sock_map_link(map, sk).
- hash = jhash(key, key_size, 0); bucket = htab.buckets[hash & (buckets_num - 1)].
- spin_lock_bh(&bucket.lock):
  - elem = sock_hash_lookup_elem_raw(&bucket.head, hash, key, key_size).
  - if elem ∧ flags == BPF_NOEXIST: -EEXIST.
  - if !elem ∧ flags == BPF_EXIST: -ENOENT.
  - elem_new = sock_hash_alloc_elem(htab, key, key_size, hash, sk, elem):
    - if atomic_inc_return(&htab.count) > max_entries ∧ !old: -E2BIG.
    - new = bpf_map_kmalloc_node(htab, elem_size, GFP_ATOMIC | __GFP_NOWARN, numa_node).
    - memcpy(new.key, key, key_size); new.sk = sk; new.hash = hash.
  - sock_map_add_link(psock, link, map, elem_new).
  - hlist_add_head_rcu(&elem_new.node, &bucket.head).
  - if elem: hlist_del_rcu(&elem.node); sock_map_unref(elem.sk, elem); sock_hash_free_elem(htab, elem) /* kfree_rcu */.

REQ-16: __sock_map_lookup_elem(map, key) / __sock_hash_lookup_elem(map, key):
- rcu_read_lock held by caller (BPF program context).
- SOCKMAP: bounds check key vs max_entries; READ_ONCE(stab.sks[key]).
- SOCKHASH: bucket = jhash → bucket; sock_hash_lookup_elem_raw walks hlist with rcu_dereference; returns elem.sk or NULL.

REQ-17: sock_map_delete_elem / sock_hash_delete_elem:
- SOCKMAP __sock_map_delete(stab, sk_test, psk):
  - spin_lock_bh(&stab.lock); if !sk_test ∨ sk_test == *psk: sk = xchg(psk, NULL).
  - if sk: sock_map_unref(sk, psk). else: -EINVAL.
- SOCKHASH sock_hash_delete_elem(map, key):
  - bucket = jhash → bucket; spin_lock_bh(&bucket.lock); find elem; hlist_del_rcu, sock_map_unref(elem.sk, elem), sock_hash_free_elem (kfree_rcu).

REQ-18: redirect helpers:
- bpf_sk_redirect_map(skb, map, key, flags):
  - if flags & ~BPF_F_INGRESS: SK_DROP.
  - sk = __sock_map_lookup_elem(map, key); if !sk ∨ !sock_map_redirect_allowed(sk): SK_DROP.
  - if (flags & BPF_F_INGRESS) ∧ sk_is_vsock(sk): SK_DROP.
  - skb_bpf_set_redir(skb, sk, flags & BPF_F_INGRESS); return SK_PASS.
- bpf_msg_redirect_map(msg, map, key, flags):
  - if flags & ~BPF_F_INGRESS: SK_DROP.
  - sk = __sock_map_lookup_elem(map, key); if !sk ∨ !sock_map_redirect_allowed(sk): SK_DROP.
  - if !(flags & BPF_F_INGRESS) ∧ !sk_is_tcp(sk): SK_DROP.
  - if sk_is_vsock(sk): SK_DROP.
  - msg.flags = flags; msg.sk_redir = sk; return SK_PASS.
- _hash variants are identical modulo lookup function.

REQ-19: bpf_sock_map_update / bpf_sock_hash_update — callable from sock_ops BPF only:
- WARN_ON_ONCE(!rcu_read_lock_held).
- sock_map_op_okay(ops) iff ops.op ∈ {BPF_SOCK_OPS_PASSIVE_ESTABLISHED_CB, BPF_SOCK_OPS_ACTIVE_ESTABLISHED_CB, BPF_SOCK_OPS_TCP_LISTEN_CB}.
- if sock_map_sk_is_suitable(ops.sk) ∧ sock_map_op_okay(ops): sock_map_update_common / sock_hash_update_common with ops.sk.
- else: -EOPNOTSUPP.

REQ-20: sock_map_redirect_allowed(sk):
- if sk_is_tcp(sk): sk.sk_state != TCP_LISTEN.
- else: READ_ONCE(sk.sk_state) == TCP_ESTABLISHED.

REQ-21: sock_map_sk_state_allowed(sk):
- TCP: state ∈ {ESTABLISHED, LISTEN}.
- Stream-UNIX: state == ESTABLISHED.
- vsock (SOCK_STREAM/SOCK_SEQPACKET): state == ESTABLISHED.
- Other: true.

REQ-22: sock_map_unhash / _destroy / _close — interposed sk_prot callbacks:
- All three: rcu_read_lock; psock = sk_psock(sk).
- _unhash: if psock: saved_unhash = psock.saved_unhash; sock_map_remove_links(sk, psock). Call saved_unhash(sk).
- _destroy: psock_get; saved_destroy; remove_links; sk_psock_stop(psock); sk_psock_put. Call saved_destroy(sk).
- _close: lock_sock; psock_get; saved_close; remove_links; sk_psock_stop; release_sock; cancel_delayed_work_sync(&psock.work); sk_psock_put. Call saved_close(sk, timeout).
- All three WARN_ON_ONCE if saved_* points back at themselves (recursion).

REQ-23: sock_map_free / sock_hash_free:
- synchronize_rcu (no in-flight update/delete).
- For each non-NULL entry: xchg(psk, NULL); sock_hold; lock_sock; rcu_read_lock; sock_map_unref; rcu_read_unlock; release_sock; sock_put.
- For SOCKHASH: spin_lock_bh(bucket); sock_hold(elem.sk) for all; hlist_move_list to local; process out of lock (lock_sock + sock_map_unref + release_sock + sock_put + sock_hash_free_elem); cond_resched per bucket.
- synchronize_rcu (drain psock readers).
- bpf_map_area_free(stab.sks / htab.buckets); bpf_map_area_free(stab / htab).

REQ-24: BPF_LINK_TYPE_SOCKMAP — sock_map_link_create:
- if attr.link_create.flags: -EINVAL.
- map = bpf_map_get_with_uref(target_fd); must be SOCKMAP or SOCKHASH.
- sockmap_link = kzalloc; bpf_link_init(&sockmap_link.link, BPF_LINK_TYPE_SOCKMAP, &sock_map_link_ops, prog, attach_type).
- bpf_link_prime; sock_map_prog_update(map, prog, NULL, &sockmap_link.link, attach_type) under sockmap_mutex.
- bpf_prog_inc(prog); bpf_link_settle.

REQ-25: sock_map_link_ops:
- .release = sock_map_link_release: sock_map_prog_update(map, NULL, link.prog, link, link.attach_type); bpf_map_put_with_uref(map).
- .detach = sock_map_link_detach → sock_map_link_release.
- .dealloc = kfree.
- .update_prog = sock_map_link_update_prog:
  - if old ∧ link.prog != old: -EPERM.
  - if link.prog.type != prog.type ∨ expected_attach_type mismatch: -EINVAL.
  - psock_replace_prog(pprog, prog, old) or psock_set_prog.
  - bpf_prog_inc(prog); old_link_prog = xchg(&link.prog, prog); bpf_prog_put(old_link_prog).
- .fill_link_info / .show_fdinfo: map_id + attach_type.

REQ-26: sock_map_bpf_prog_query — BPF_PROG_QUERY:
- if attr.query.query_flags: -EINVAL.
- map = __bpf_map_get(target_fd).
- sock_map_prog_link_lookup(map, &pprog, NULL, attr.query.attach_type).
- prog = *pprog; prog_cnt = !prog ? 0 : 1.
- id = data_race(prog.aux.id); if id == 0: prog_cnt = 0.
- copy_to_user(prog_ids[0] = id) + prog_cnt + attach_flags = 0.

REQ-27: BPF_ITER on sockmap/sockhash:
- sock_map_iter_reg.target = "sockmap"; attach_target validates SOCKMAP/SOCKHASH, max_rdonly_access ≤ key_size.
- ctx_arg_info: bpf_iter__sockmap { meta; map; key; sk }. key = PTR_TO_BUF|MAYBE_NULL|RDONLY; sk = PTR_TO_BTF_ID_OR_NULL (BTF_SOCK_TYPE_SOCK).
- seq_ops: rcu_read_lock around start/stop; iterate stab.sks[index] or htab buckets.

## Acceptance Criteria

- [ ] AC-1: sock_map_alloc rejects key_size != 4 / value_size ∉ {4,8} / max_entries == 0.
- [ ] AC-2: sock_hash_alloc rejects key_size == 0 or key_size > MAX_BPF_STACK.
- [ ] AC-3: sock_map_update_elem_sys: sockfd_lookup → sk; rejects sk with no psock_update_sk_prot.
- [ ] AC-4: sock_map_update_common with BPF_NOEXIST on existing slot returns -EEXIST.
- [ ] AC-5: sock_map_update_common with BPF_EXIST on empty slot returns -ENOENT.
- [ ] AC-6: Insert refcounted: previous occupant's psock back-link removed via sock_map_unref.
- [ ] AC-7: sock_map_link: -EBUSY when sk already has a sockmap-style psock with overlapping progs.
- [ ] AC-8: After insert, sk.sk_prot.close == sock_map_close; psock.psock_update_sk_prot recorded.
- [ ] AC-9: bpf_sk_redirect_map: TCP_LISTEN socket as target → SK_DROP.
- [ ] AC-10: bpf_msg_redirect_map: non-TCP target with egress (no BPF_F_INGRESS) → SK_DROP.
- [ ] AC-11: vsock target rejected by both sk_redirect (with INGRESS) and msg_redirect.
- [ ] AC-12: sock_map_close: links removed before sk_psock_stop; saved_close called exactly once.
- [ ] AC-13: sock_map_free: synchronize_rcu before+after walk; every non-NULL entry has lock_sock+unref+release_sock+sock_put.
- [ ] AC-14: BPF_PROG_QUERY returns id=0/prog_cnt=0 when no prog attached to (map, attach_type).
- [ ] AC-15: BPF_LINK_TYPE_SOCKMAP: same map can hold only one bpf_link per attach_type (re-attach with link existing → -EBUSY).
- [ ] AC-16: stream_verdict and skb_verdict are mutually exclusive (REQ-6 -EBUSY).

## Architecture

```
struct BpfStab {
  map:   BpfMap,
  sks:   *mut *mut Sock,            // max_entries slots
  progs: SkPsockProgs,
  lock:  SpinLock,
}

struct BpfShtab {
  map:          BpfMap,
  buckets:      *mut BpfShtabBucket,
  buckets_num:  u32,                 // roundup_pow_of_two(max_entries)
  elem_size:    u32,                 // sizeof(elem) + round_up(key_size, 8)
  progs:        SkPsockProgs,
  count:        AtomicU32,
}

struct BpfShtabBucket {
  head: HlistHead<BpfShtabElem>,
  lock: SpinLock,
}

#[repr(C)]
struct BpfShtabElem {
  rcu:  RcuHead,
  hash: u32,
  sk:   *mut Sock,
  node: HlistNode,
  key:  [u8; 0],                     // VLA
}

struct SockmapLink {
  link: BpfLink,
  map:  *mut BpfMap,
}
```

`BpfStab::map_alloc(attr) -> Result<*mut BpfMap>`:
1. if attr.max_entries == 0 ∨ attr.key_size != 4 ∨ attr.value_size ∉ {4,8} ∨ attr.map_flags & !(NUMA_NODE|RDONLY|WRONLY): Err(-EINVAL).
2. stab = bpf_map_area_alloc(sizeof(BpfStab), NUMA_NO_NODE).
3. bpf_map_init_from_attr(&stab.map, attr).
4. spin_lock_init(&stab.lock).
5. stab.sks = bpf_map_area_alloc(max_entries * size_of::<*mut Sock>(), stab.map.numa_node).
6. Ok(&stab.map).

`BpfShtab::map_alloc(attr) -> Result<*mut BpfMap>`:
1. Validate as REQ-4 (key_size > 0, key_size ≤ MAX_BPF_STACK).
2. htab.buckets_num = roundup_pow_of_two(max_entries).
3. htab.elem_size = size_of::<BpfShtabElem>() + round_up(key_size, 8).
4. Validate buckets_num != 0 ∧ ≤ U32_MAX / size_of::<BpfShtabBucket>().
5. htab.buckets = bpf_map_area_alloc(buckets_num * size_of::<BpfShtabBucket>(), numa_node).
6. for i in 0..buckets_num: INIT_HLIST_HEAD; spin_lock_init.

`SockMap::link_psock(map, sk) -> Result<()>`:
1. progs = SockMap::progs(map).
2. Take bpf_prog_inc_not_zero on each of progs.{stream_verdict, stream_parser, msg_parser, skb_verdict} (cascading goto-undo on each failure).
3. psock = SockMap::psock_get_checked(sk):
   - rcu_read_lock; psock = sk_psock(sk).
   - if psock ∧ sk.sk_prot.close != sock_map_close: Err(-EBUSY).
   - else if !refcount_inc_not_zero(&psock.refcnt): Err(-EBUSY).
4. if psock:
   - /* No overlapping attach types allowed */
   - if msg_parser ∧ psock.progs.msg_parser
     ∨ stream_parser ∧ psock.progs.stream_parser
     ∨ (skb_verdict ∨ stream_verdict) ∧ (psock.progs.skb_verdict ∨ psock.progs.stream_verdict): Err(-EBUSY).
5. else: psock = sk_psock_init(sk, map.numa_node).
6. psock_set_prog(&psock.progs.msg_parser, msg_parser); same for stream_parser, stream_verdict, skb_verdict.
7. /* swap sk_prot to psock-shim variant */
8. if !sk.sk_prot.psock_update_sk_prot: Err(-EINVAL).
9. psock.psock_update_sk_prot = sk.sk_prot.psock_update_sk_prot.
10. sk.sk_prot.psock_update_sk_prot(sk, psock, false).
11. write_lock_bh(&sk.sk_callback_lock):
    - if stream_parser ∧ stream_verdict ∧ !psock.saved_data_ready:
      - if !sk_is_tcp(sk): Err(-EOPNOTSUPP).
      - sk_psock_init_strp(sk, psock); sk_psock_start_strp(sk, psock).
    - else if !stream_parser ∧ stream_verdict ∧ !psock.saved_data_ready: sk_psock_start_verdict(sk, psock).
    - else if !stream_verdict ∧ skb_verdict ∧ !psock.saved_data_ready: sk_psock_start_verdict(sk, psock).
12. write_unlock_bh.

`BpfStab::update_common(map, idx, sk, flags) -> Result<()>`:
1. WARN if !rcu_read_lock_held.
2. if flags > BPF_EXIST: Err(-EINVAL).
3. if idx >= max_entries: Err(-E2BIG).
4. link = sk_psock_init_link(). On NULL Err(-ENOMEM).
5. SockMap::link_psock(map, sk)?
6. psock = sk_psock(sk). WARN if !psock.
7. spin_lock_bh(&stab.lock):
   - osk = stab.sks[idx].
   - if osk ∧ flags == BPF_NOEXIST: Err(-EEXIST).
   - if !osk ∧ flags == BPF_EXIST: Err(-ENOENT).
   - SockMap::add_link(psock, link, map, &stab.sks[idx]).
   - stab.sks[idx] = sk.
   - if osk: SockMap::unref(osk, &stab.sks[idx]).
8. Ok(()).

`BpfShtab::update_common(map, key, sk, flags) -> Result<()>`:
1. Validate flags ≤ BPF_EXIST; rcu_read_lock_held.
2. link = sk_psock_init_link(). On NULL Err(-ENOMEM).
3. SockMap::link_psock(map, sk)?
4. hash = jhash(key, key_size, 0).
5. bucket = htab.buckets[hash & (buckets_num - 1)].
6. spin_lock_bh(&bucket.lock):
   - elem = hlist scan for (hash, memcmp(key) == 0).
   - if elem ∧ flags == BPF_NOEXIST: Err(-EEXIST).
   - if !elem ∧ flags == BPF_EXIST: Err(-ENOENT).
   - elem_new = alloc_elem(htab, key, key_size, hash, sk, elem):
     - if atomic_inc_return(&htab.count) > max_entries ∧ !elem: atomic_dec; Err(-E2BIG).
     - bpf_map_kmalloc_node(htab, elem_size, GFP_ATOMIC|__GFP_NOWARN, numa_node).
     - memcpy(new.key, key, key_size); new.sk = sk; new.hash = hash.
   - SockMap::add_link(psock, link, map, elem_new).
   - hlist_add_head_rcu(&elem_new.node, &bucket.head).
   - if elem: hlist_del_rcu(&elem.node); SockMap::unref(elem.sk, elem); kfree_rcu(elem).
7. Ok(()).

`SockMap::helper_sk_redirect(skb, map, key, flags) -> i32`:
1. if flags & !BPF_F_INGRESS: return SK_DROP.
2. sk = lookup_raw(map, key). If !sk ∨ !SockMap::redirect_allowed(sk): return SK_DROP.
3. if (flags & BPF_F_INGRESS) ∧ sk_is_vsock(sk): return SK_DROP.
4. skb_bpf_set_redir(skb, sk, flags & BPF_F_INGRESS).
5. return SK_PASS.

`SockMap::helper_msg_redirect(msg, map, key, flags) -> i32`:
1. if flags & !BPF_F_INGRESS: return SK_DROP.
2. sk = lookup_raw(map, key). If !sk ∨ !redirect_allowed(sk): return SK_DROP.
3. if !(flags & BPF_F_INGRESS) ∧ !sk_is_tcp(sk): return SK_DROP. /* egress requires TCP */
4. if sk_is_vsock(sk): return SK_DROP.
5. msg.flags = flags; msg.sk_redir = sk; return SK_PASS.

`SockMap::sk_close(sk, timeout)`:
1. lock_sock(sk); rcu_read_lock.
2. psock = sk_psock(sk).
3. if psock:
   - saved_close = psock.saved_close.
   - SockMap::remove_links(sk, psock).
   - psock = sk_psock_get(sk).
   - if !psock: goto no_psock.
   - rcu_read_unlock.
   - sk_psock_stop(psock).
   - release_sock(sk).
   - cancel_delayed_work_sync(&psock.work).
   - sk_psock_put(sk, psock).
4. else: saved_close = sk.sk_prot.close; no_psock: rcu_read_unlock; release_sock.
5. WARN_ON_ONCE(saved_close == sock_map_close).  /* prevent recursion */
6. saved_close(sk, timeout).

`SockMap::prog_update(map, prog, old, link, which) -> Result<()>`:
1. SockMap::prog_link_lookup(map, &pprog, &plink, which)?
2. if (!link ∨ prog) ∧ *plink: Err(-EBUSY).  /* prog_attach/detach can't displace link-owned slot */
3. if old:
   - psock_replace_prog(pprog, prog, old)?
   - on success: *plink = NULL.
4. else:
   - psock_set_prog(pprog, prog).
   - if link: *plink = link.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `sks_array_xchg_under_stab_lock` | INVARIANT | per-update_common: stab.sks[idx] xchg only with stab.lock held. |
| `hlist_mutation_under_bucket_lock` | INVARIANT | per-shtab: hlist_add_head_rcu / hlist_del_rcu only with bucket.lock held. |
| `psock_progs_overlap_rejected` | INVARIANT | link_psock: overlapping msg_parser/stream_parser/verdict on existing psock ⟹ -EBUSY. |
| `sk_prot_swap_atomic` | INVARIANT | sock_map_init_proto: sk_prot replaced before any data_ready event sees new psock. |
| `saved_close_not_self` | INVARIANT | sock_map_close: WARN if saved_close == sock_map_close (recursion guard). |
| `link_back_pointer_consistent` | INVARIANT | sock_map_add_link: link.map / link.link_raw = caller args; psock.link contains every map membership. |
| `redirect_drops_listen_tcp` | INVARIANT | bpf_sk_redirect_*: sk_is_tcp ∧ sk_state == TCP_LISTEN ⟹ SK_DROP. |
| `redirect_drops_vsock_ingress` | INVARIANT | bpf_sk_redirect_*: BPF_F_INGRESS ∧ sk_is_vsock ⟹ SK_DROP. |
| `msg_redirect_drops_nontcp_egress` | INVARIANT | bpf_msg_redirect_*: !INGRESS ∧ !sk_is_tcp ⟹ SK_DROP. |
| `stream_verdict_skb_verdict_exclusive` | INVARIANT | prog_link_lookup: stream_verdict slot busy ⟹ skb_verdict attach -EBUSY (and vice versa). |
| `link_owned_slot_protected` | INVARIANT | prog_update: bpf_link-owned slot rejects prog_attach/detach. |
| `count_atomicity` | INVARIANT | sock_hash_alloc_elem: count > max_entries ⟹ refuse new insert (replace allowed). |
| `synchronize_rcu_before_walk_on_free` | INVARIANT | sock_map_free / sock_hash_free: synchronize_rcu called both before and after the bucket walk. |

### Layer 2: TLA+

`net/core/sock-map.tla`:
- Per-attach + per-insert + per-delete + per-redirect + per-close.
- Properties:
  - `safety_no_overlapping_progs_on_psock` — per-link_psock: psock never holds two competing progs for same attach point.
  - `safety_one_link_per_attach_type` — per-map: at most one bpf_link per (attach_type) slot.
  - `safety_listen_tcp_not_redirected` — per-sk_redirect: TCP_LISTEN never SK_PASS.
  - `safety_vsock_not_redirected_ingress` — per-sk_redirect: vsock+INGRESS always SK_DROP.
  - `safety_msg_redirect_egress_tcp_only` — per-msg_redirect: !INGRESS ⟹ sk_is_tcp.
  - `safety_psock_destroyed_after_close` — per-sock_map_close: sk_psock_stop + sk_psock_put before saved_close.
  - `safety_no_recursive_close` — per-sock_map_close: saved_close != sock_map_close.
  - `safety_rcu_lookup_safe` — per-redirect: __sock_map_lookup_elem inside rcu_read_lock, elem freed via kfree_rcu.
  - `liveness_per_insert_terminates` — per-update_common: bounded by spinlock + finite progs setup.
  - `liveness_per_close_eventually_drops_psock` — per-sk close: psock refcount reaches 0.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `BpfStab::map_alloc` post: key_size == 4 ∧ value_size ∈ {4,8} ∧ max_entries > 0 | `BpfStab::map_alloc` |
| `BpfShtab::map_alloc` post: buckets_num is power-of-two ≥ max_entries; elem_size = sizeof(elem)+round_up(key_size,8) | `BpfShtab::map_alloc` |
| `SockMap::link_psock` post: ret = Ok ⟹ psock_set_prog called for each non-NULL prog in map.progs | `SockMap::link_psock` |
| `BpfStab::update_common` post: stab.sks[idx] == sk ∧ link.list ∈ psock.link list ∧ old occupant unref'd | `BpfStab::update_common` |
| `BpfShtab::update_common` post: htab.count ≤ max_entries (or replaced); bucket head's first elem matches new key | `BpfShtab::update_common` |
| `SockMap::delete_elem` post: stab.sks[idx] == NULL after; old sk unref'd; ret = Ok | `SockMap::delete_elem` |
| `SockMap::helper_sk_redirect` post: ret ∈ {SK_DROP, SK_PASS}; SK_PASS ⟹ redirect_allowed(sk) | `SockMap::helper_sk_redirect` |
| `SockMap::helper_msg_redirect` post: msg.sk_redir set ⟹ ret == SK_PASS ∧ sk valid | `SockMap::helper_msg_redirect` |
| `SockMap::sk_close` post: psock.refcnt decremented; saved_close called once; cancel_delayed_work_sync(&psock.work) executed | `SockMap::sk_close` |
| `SockMap::prog_update` post: link-owned slot not overwritten by prog_attach (-EBUSY) | `SockMap::prog_update` |
| `SockmapLink::create` post: bpf_link_settle returns fd; map_id, attach_type stored | `SockmapLink::create` |

### Layer 4: Verus/Creusot functional

`Per-PROG_ATTACH(map, prog, attach_type) → progs.<slot> = prog → on subsequent map_update_elem(sk, ...): sk_psock allocated/located → progs copied to psock → sk_prot swapped → data_ready calls strparser/verdict path → BPF returns SK_PASS|SK_DROP with optional redirect → recipient sk's ingress queue receives skb` semantic equivalence: per-Documentation/bpf/sockmap.rst.

`Per-BPF_LINK_CREATE(SOCKMAP) → sock_map_link_ops registered → release_link triggers sock_map_prog_update(map, NULL, prog, link, attach_type) → progs.<slot> cleared, *plink cleared → map_put_with_uref` semantic equivalence: per-libbpf bpf_link_create() documentation.

## Hardening

(Inherits row-1 features from `net/core/00-overview.md` § Hardening — when authored.)

Sockmap/sockhash reinforcement:

- **Per-rcu_read_lock around all sk lookups in redirect path** — defense against per-UAF after concurrent map_delete_elem.
- **Per-kfree_rcu on bpf_shtab_elem** — defense against per-bucket-walker-vs-deleter race.
- **Per-stab.lock + per-bucket-lock isolation** — defense against per-concurrent-insert torn writes.
- **Per-sk_prot.psock_update_sk_prot != NULL gate** — defense against per-unsupported-proto attach (must be TCP/Unix-stream/vsock w/ patched sk_prot).
- **Per-sock_map_sk_state_allowed filter** — defense against per-attach-to-half-open / TIME_WAIT.
- **Per-saved_close recursion guard (WARN_ON_ONCE)** — defense against per-double-interpose stack overflow.
- **Per-cancel_delayed_work_sync on close** — defense against per-deferred-work-after-free.
- **Per-stream_verdict / skb_verdict mutual exclusion** — defense against per-conflicting-prog confusion.
- **Per-bpf_link displaces prog_attach (-EBUSY)** — defense against per-uncoordinated detach of link-owned prog.
- **Per-vsock ingress redirect denied + per-non-TCP egress redirect denied** — defense against per-cross-family confused-deputy.
- **Per-CLASS(fd) auto-close in syscall paths** — defense against per-fd leak on error path.
- **Per-WARN_ON_ONCE(!rcu_read_lock_held) in lookup_raw** — defense against per-non-RCU caller hitting freed elem.
- **Per-bpf_prog_inc_not_zero before install** — defense against per-prog freed concurrently with attach.

## Grsecurity/PaX-style Reinforcement

Rationale: `BPF_MAP_TYPE_SOCKMAP` / `SOCKHASH` insert eBPF programs (`stream_verdict`, `stream_parser`, `skb_verdict`) into the TCP/UDP receive path, redirecting packets across sockets — a malicious or buggy program lets the operator pivot a packet from socket A's owner to socket B's owner. Compromise here is privilege escalation across cgroup/netns boundaries.

Baseline (cross-ref `net/00-overview.md` § Hardening):
- **PAX_USERCOPY**: `bpf_attr` from `bpf(BPF_MAP_UPDATE_ELEM, ...)` copy via `bpf_check_uarg_tail_zero` + bounded `copy_from_user`; map-value buffers USERCOPY-whitelisted per-element-size.
- **PAX_KERNEXEC**: `bpf_map_ops` (sock_map_ops, sock_hash_ops) live in `__ro_after_init`; `sk_psock_progs.stream_verdict`, `->stream_parser`, `->skb_verdict` are atomic-xchg-replaced under `sk_psock->ingress_lock`, never patched in-place.
- **PAX_RANDKSTACK**: every entry into `sock_map_update_elem`, `sock_map_delete_elem`, `sk_psock_drop` re-randomises kernel-stack offset.
- **PAX_REFCOUNT**: `sk_psock.refcnt`, `bpf_prog.aux->refcnt`, and per-map `usercnt` use saturating `Refcount`; defends sockmap-update storm.
- **PAX_MEMORY_SANITIZE**: `sk_psock_destroy` zero-fills the `sk_psock` slab block including the `progs.stream_verdict` pointer slot before slab-return.
- **PAX_UDEREF**: bpfsyscall uargs accessed only through `bpf_check_uarg_tail_zero`-validated copies; no raw `__user` deref.
- **PAX_RAP / kCFI**: indirect calls through `sk_psock_progs.{stream_verdict,stream_parser,skb_verdict}` (BPF dispatch) are kCFI-tagged via the BPF dispatcher; per-`bpf_map_ops` methods (lookup_elem, update_elem, delete_elem, free) likewise.
- **GRKERNSEC_HIDESYM**: `bpf_prog` JIT image address, `sk_psock*`, map fd-table pointer never rendered into `bpftool prog show` output for non-CAP_SYS_ADMIN.
- **GRKERNSEC_DMESG**: BPF verifier rejection messages ratelimited; CAP_SYSLOG to read.

sock-map-specific reinforcement:
- **BPF sockmap CAP_BPF strict** — `bpf(BPF_MAP_CREATE, BPF_MAP_TYPE_SOCKMAP)` requires `CAP_BPF` AND `CAP_NET_ADMIN` in `init_user_ns` (grsec policy beyond upstream's CAP_BPF-only post-5.8 default); refuse from non-init userns.
- **sk_psock_progs kCFI** — `stream_verdict` / `stream_parser` / `skb_verdict` indirect calls validated by BPF dispatcher kCFI tag; mismatch → `BUG()` not type-confused jump into wrong prog type.
- **bpf_prog_inc_not_zero + atomic-xchg install** — defends against attach/free TOCTOU on prog refcount.
- **stream_verdict / skb_verdict mutual exclusion** — refuse simultaneous attach (already in row-2); grsec audit-log on attempted bypass.
- **Cross-family redirect refusal** — sockmap-redirect from TCP to vsock/AF_UNIX → `BPF_DROP` + grsec audit; defends against confused-deputy across protocol families.
- **CLASS(fd) auto-close** — `sock_map_get_from_fd` uses `CLASS(fd)` RAII, defends fd-leak on partial-failure paths.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- net/core/skmsg.c sk_psock lifecycle, strparser integration, sk_psock_init/_destroy/_stop (covered separately if expanded)
- net/ipv4/tcp_bpf.c TCP sk_prot psock_update_sk_prot implementation (covered separately if expanded)
- net/ipv4/udp_bpf.c / net/unix/unix_bpf.c / net/vmw_vsock/vsock_bpf.c per-protocol shims (covered separately)
- kernel/bpf/verifier.c verifier integration for ARG_PTR_TO_CTX / sk redirect helpers (covered in `verifier.md` Tier-3)
- kernel/bpf/syscall.c BPF_MAP_CREATE / BPF_PROG_ATTACH dispatch (covered in `bpf-core.md` Tier-3)
- include/linux/skmsg.h struct sk_msg pipeline (covered separately if expanded)
- Implementation code
