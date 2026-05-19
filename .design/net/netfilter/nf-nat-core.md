# Tier-3: net/netfilter/nf_nat_core.c — NAT core engine

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/netfilter/00-overview.md
upstream-paths:
  - net/netfilter/nf_nat_core.c (~1369 lines)
  - include/net/netfilter/nf_nat.h
  - include/uapi/linux/netfilter/nf_nat.h
-->

## Summary

`nf_nat_core.c` is the **Network Address Translation engine** for Netfilter — sitting on top of conntrack, it implements per-connection address/port rewriting. It owns: the global `nf_nat_bysource` siphash table (keyed by ORIGINAL `(src, protonum, zone, net_mix)` → list of NATted `nf_conn` via `nat_bysource` hlist), per-bucket striped `nf_nat_locks[CONNTRACK_LOCKS]` spinlocks, the `nf_nat_setup_info(ct, range, maniptype)` core that picks a unique tuple via `get_unique_tuple` + `find_appropriate_src` (per-srcip mapping memoisation) + `find_best_ips_proto` (jhash-distributed IP selection) + `nf_nat_l4proto_unique_tuple` (per-l4 port/id allocation with `NF_NAT_MAX_ATTEMPTS=128` ceiling and `NF_NAT_HARDER_THRESH=32` TIME_WAIT eviction), `nf_nat_inet_fn` PRE/POST/LOCAL hook that runs registered per-table NAT chains then falls back to `nf_nat_alloc_null_binding`, `nf_nat_packet` packet rewriter dispatching to `nf_nat_manip_pkt`, `nf_nat_register_fn`/`_unregister_fn` per-net per-`NFPROTO_*` hook entry table (the `nat_net.nat_proto_net[NFPROTO_NUMPROTO]` array of `nf_nat_hooks_net` with `nf_hook_entries_insert_raw` chaining), per-`CONFIG_XFRM` session-decode (`__nf_nat_decode_session` writing `flowi4`/`flowi6` from ct status bits), per-`CONFIG_NF_CT_NETLINK` `CTA_NAT_V4_MIN/MAX_IP`/`V6_MIN/MAX_IP` and `CTA_PROTONAT_PORT_MIN/MAX` parsers, and `nf_nat_cleanup_conntrack` bysource-removal at ct destroy. Critical for: SNAT (POST_ROUTING source-rewrite), DNAT (PRE_ROUTING / LOCAL_OUT dest-rewrite), MASQUERADE (interface-bound SNAT with auto-flush), and PPTP/GRE-key NAT.

This Tier-3 covers `net/netfilter/nf_nat_core.c` (~1369 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `nf_nat_bysource` (static) | per-srcip hashtable | `Nat::BYSOURCE` |
| `nf_nat_htable_size` | per-table bucket count | `Nat::HTABLE_SIZE` |
| `nf_nat_hash_rnd` (siphash key) | per-`get_random_once` hash key | `Nat::HASH_RND` |
| `nf_nat_locks[CONNTRACK_LOCKS]` | per-bucket-stripe lock array | `Nat::LOCKS` |
| `nf_nat_proto_mutex` | per-registry mutex | `Nat::PROTO_MUTEX` |
| `struct nf_nat_lookup_hook_priv` | per-`nf_hook_ops.priv` lookup chain | `Nat::LookupHookPriv` |
| `struct nf_nat_hooks_net` | per-net per-pf nat hook ops | `Nat::HooksNet` |
| `struct nat_net` | per-net pernet generic data | `Nat::PerNet` |
| `hash_by_src(net, zone, tuple)` | per-tuple bysource hash | `Nat::hash_by_src` |
| `nf_nat_used_tuple()` | per-collision check (inverse lookup) | `Nat::used_tuple` |
| `nf_nat_used_tuple_new()` | per-new-ct collision check + clash-allow | `Nat::used_tuple_new` |
| `nf_nat_used_tuple_harder()` | per-aggressive TIME_WAIT eviction | `Nat::used_tuple_harder` |
| `nf_nat_may_kill()` | per-TIME_WAIT kill eligibility | `Nat::may_kill` |
| `nf_seq_has_advanced()` | per-TCP seq compare | `Nat::seq_has_advanced` |
| `nf_nat_allow_clash()` | per-l4proto clash flag | `Nat::allow_clash` |
| `nf_nat_inet_in_range()` | per-AF IP range test | `Nat::inet_in_range` |
| `l4proto_in_range()` | per-l4 port/id range test | `Nat::l4proto_in_range` |
| `nf_in_range()` | per-`nf_nat_range2` membership | `Nat::in_range` |
| `same_src()` | per-bysource matcher | `Nat::same_src` |
| `find_appropriate_src()` | per-bysource lookup for SRC reuse | `Nat::find_appropriate_src` |
| `find_best_ips_proto()` | per-jhash IP picker | `Nat::find_best_ips_proto` |
| `nf_nat_l4proto_unique_tuple()` | per-l4 port/id picker | `Nat::l4proto_unique_tuple` |
| `get_unique_tuple()` | per-overall tuple selector | `Nat::get_unique_tuple` |
| `nf_ct_nat_ext_add()` | per-ct NAT-extension allocator | `Nat::ct_nat_ext_add` |
| `nf_nat_setup_info()` | per-ct NAT binding initialiser | `Nat::setup_info` |
| `__nf_nat_alloc_null_binding()` | per-no-rule passthrough binding | `Nat::alloc_null_binding_inner` |
| `nf_nat_alloc_null_binding()` | per-hook wrapper | `Nat::alloc_null_binding` |
| `nf_nat_packet()` | per-packet rewrite dispatch | `Nat::packet` |
| `nf_nat_inet_fn()` | per-`NF_INET_*` hook fn | `Nat::inet_fn` |
| `in_vrf_postrouting()` (static, NAT-local) | per-VRF POST_ROUTING bypass | `Nat::in_vrf_postrouting` |
| `__nf_nat_decode_session()` | per-XFRM session decode | `Nat::decode_session` |
| `nf_nat_ipv4_decode_session()` | per-XFRM v4 decode | `Nat::decode_session_v4` |
| `nf_nat_ipv6_decode_session()` | per-XFRM v6 decode | `Nat::decode_session_v6` |
| `nf_nat_proto_remove()` | per-ct match-by-pf+proto | `Nat::proto_remove` |
| `nf_nat_cleanup_conntrack()` | per-ct bysource removal | `Nat::cleanup_conntrack` |
| `nf_nat_proto_clean()` | per-ct rmmod cleanup | `Nat::proto_clean` |
| `nfnetlink_parse_nat()` | per-`CTA_NAT_*` parser | `Nat::nl_parse_nat` |
| `nfnetlink_parse_nat_proto()` | per-`CTA_PROTONAT` parser | `Nat::nl_parse_nat_proto` |
| `nfnetlink_parse_nat_setup()` | per-ctnetlink setup-info hook | `Nat::nl_parse_nat_setup` |
| `nf_nat_register_fn()` | per-net per-pf hook chain insert | `Nat::register_fn` |
| `nf_nat_unregister_fn()` | per-net per-pf hook chain remove | `Nat::unregister_fn` |
| `nf_nat_init()` (module_init) | per-module init | `Nat::module_init` |
| `nf_nat_cleanup()` (module_exit) | per-module fini | `Nat::module_fini` |
| `nat_net_ops` (pernet) | per-net subsys | `Nat::PERNET_OPS` |
| `nat_hook` (`struct nf_nat_hook`) | per-NF callbacks (parse / decode / remove) | `Nat::NAT_HOOK` |
| `follow_master_nat` | per-helper-expectfn for PPTP | `Nat::FOLLOW_MASTER` |

## Compatibility contract

REQ-1: Per-module constants:
- `NF_NAT_MAX_ATTEMPTS = 128` — per-`l4proto_unique_tuple` outer loop cap.
- `NF_NAT_HARDER_THRESH = NF_NAT_MAX_ATTEMPTS / 4 = 32` — per-attempts-left threshold below which `used_tuple_harder` may evict TIME_WAIT TCP ct.
- `nf_nat_bysource: *mut [hlist_head]` — per-net-shared (NOT per-net) global hashtable.
- `nf_nat_htable_size` — set at `module_init` from `nf_conntrack_htable_size`, lower-bounded by `CONNTRACK_LOCKS`.
- `nf_nat_hash_rnd: siphash_key_t` — initialised lazily via `get_random_once`.
- `nf_nat_locks[CONNTRACK_LOCKS]` — striped spinlocks; bucket `h` uses `nf_nat_locks[h % CONNTRACK_LOCKS]`.

REQ-2: `hash_by_src(net, zone, tuple) -> u32`:
- `combined = { src: tuple.src, net_mix: net_hash_mix(net), protonum: tuple.dst.protonum, zone: 0 }`.
- if `zone.dir == NF_CT_DEFAULT_ZONE_DIR`: `combined.zone = zone.id`.
- `get_random_once(&NF_NAT_HASH_RND, sizeof)`.
- `hash = siphash(&combined, sizeof(combined), &NF_NAT_HASH_RND)`.
- return `reciprocal_scale(hash, nf_nat_htable_size)`.

REQ-3: `nf_nat_used_tuple(tuple, ignored_ct) -> bool`:
- /* Reply tuple lookup — conntrack tracks reply, not outgoing */
- `nf_ct_invert_tuple(&reply, tuple)`.
- return `nf_conntrack_tuple_taken(&reply, ignored_ct)`.

REQ-4: `nf_nat_used_tuple_new(tuple, ignored_ct) -> bool`:
- /* `noinline`. Two-step: first check forward-collision via `used_tuple`; if collision but `nf_nat_allow_clash` (UDP/ICMP allow clash) then check whether the colliding ct itself is subject to NAT.
   Only returns false (tuple usable) when both ct's are non-NAT and ORIGINAL of new matches REPLY-of-ignored — clash-resolve at insertion time. */
- if `!nf_nat_used_tuple(tuple, ignored_ct)`: return false.
- if `!nf_nat_allow_clash(ignored_ct)`: return true.
- if `READ_ONCE(ignored_ct.status) & (IPS_NAT_MASK | IPS_SEQ_ADJUST)`: return true.
- `net = nf_ct_net(ignored_ct)`; `zone = nf_ct_zone(ignored_ct)`.
- `thash = nf_conntrack_find_get(net, zone, tuple)`.
- if `!thash`: `nf_ct_invert_tuple(&reply, tuple)`; `thash = nf_conntrack_find_get(net, zone, &reply)`; if `!thash`: return false /* clash gone */.
- `ct = nf_ct_tuplehash_to_ctrack(thash)`.
- `taken = true`.
- if `READ_ONCE(ct.status) & (IPS_NAT_MASK | IPS_SEQ_ADJUST)`: goto out.
- if `nf_ct_tuple_equal(&ct.tuplehash[ORIGINAL].tuple, &ignored_ct.tuplehash[REPLY].tuple)`: `taken = false`.
- out: `nf_ct_put(ct)`; return `taken`.

REQ-5: `nf_nat_may_kill(ct, status_flags) -> bool` — per-TIME_WAIT TCP evictability:
- `old_state = READ_ONCE(ct.proto.tcp.state)`.
- if `old_state < TCP_CONNTRACK_TIME_WAIT`: return false.
- if `status_flags & (IPS_FIXED_TIMEOUT | IPS_DYING)`: return false.
- return `(status_flags & IPS_SRC_NAT) == IPS_SRC_NAT`.

REQ-6: `nf_seq_has_advanced(old, new) -> bool`:
- return `(__s32)(new.proto.tcp.seen[0].td_end - old.proto.tcp.seen[0].td_end) > 0`.

REQ-7: `nf_nat_used_tuple_harder(tuple, ignored_ct, attempts_left) -> bool`:
- `nf_ct_invert_tuple(&reply, tuple)`.
- if `attempts_left > NF_NAT_HARDER_THRESH ∨ tuple.dst.protonum != IPPROTO_TCP ∨ ignored_ct.proto.tcp.state != TCP_CONNTRACK_SYN_SENT`:
  - return `nf_conntrack_tuple_taken(&reply, ignored_ct)` /* mild path */.
- /* Aggressive path: search for evictable TIME_WAIT collision */
- `thash = nf_conntrack_find_get(net, zone, &reply)`.
- if `!thash`: return false.
- `ct = nf_ct_tuplehash_to_ctrack(thash)`.
- `taken = true`.
- if `thash.tuple.dst.dir == IP_CT_DIR_ORIGINAL`: goto out.
- WARN if `ct == ignored_ct` and goto out.
- `flags = READ_ONCE(ct.status)`.
- if `!nf_nat_may_kill(ct, flags)`: goto out.
- if `!nf_seq_has_advanced(ct, ignored_ct)`: goto out.
- if `nf_ct_kill(ct)`: `taken = flags & (IPS_OFFLOAD | IPS_HW_OFFLOAD)` /* offload-pinned cts cannot be reused */.
- out: `nf_ct_put(ct)`; return `taken`.

REQ-8: `nf_nat_inet_in_range(t, range) -> bool`:
- IPv4: `ntohl(t.src.u3.ip) ∈ [ntohl(range.min_addr.ip), ntohl(range.max_addr.ip)]`.
- IPv6: `ipv6_addr_cmp(&t.src.u3.in6, &range.min_addr.in6) >= 0 ∧ ipv6_addr_cmp(..., &range.max_addr.in6) <= 0`.

REQ-9: `l4proto_in_range(tuple, maniptype, min, max) -> bool`:
- ICMP/ICMPV6: `ntohs(tuple.src.u.icmp.id) ∈ [ntohs(min.icmp.id), ntohs(max.icmp.id)]`.
- GRE/TCP/UDP/SCTP: `port = (maniptype == SRC) ? tuple.src.u.all : tuple.dst.u.all`; `ntohs(port) ∈ [ntohs(min.all), ntohs(max.all)]`.
- default: true.

REQ-10: `nf_in_range(tuple, range) -> bool`:
- if `range.flags & NF_NAT_RANGE_MAP_IPS ∧ !nf_nat_inet_in_range(tuple, range)`: return false.
- if `!(range.flags & NF_NAT_RANGE_PROTO_SPECIFIED)`: return true.
- return `l4proto_in_range(tuple, NF_NAT_MANIP_SRC, &range.min_proto, &range.max_proto)`.

REQ-11: `find_appropriate_src(net, zone, tuple, result_out, range) -> bool`:
- /* Memoise SRC-mapping: walk bysource bucket */
- `h = hash_by_src(net, zone, tuple)`.
- for `ct` in `hlist_for_each_entry_rcu(&BYSOURCE[h], nat_bysource)`:
  - if `same_src(ct, tuple) ∧ net_eq(net, nf_ct_net(ct)) ∧ nf_ct_zone_equal(ct, zone, IP_CT_DIR_ORIGINAL)`:
    - `nf_ct_invert_tuple(result_out, &ct.tuplehash[REPLY].tuple)` /* invert REPLY to get prior SRC mapping */.
    - `result_out.dst = tuple.dst`.
    - if `nf_in_range(result_out, range)`: return true.
- return false.

REQ-12: `find_best_ips_proto(zone, tuple_io, range, ct, maniptype)`:
- if `!(range.flags & NF_NAT_RANGE_MAP_IPS)`: return.
- `var_ipp = (maniptype == SRC) ? &tuple.src.u3 : &tuple.dst.u3`.
- /* Fast path: single-IP range */
- if `nf_inet_addr_cmp(&range.min_addr, &range.max_addr)`: `*var_ipp = range.min_addr`; return.
- `max = (l3num == IPv4) ? sizeof(ip)/sizeof(u32)-1 : sizeof(ip6)/sizeof(u32)-1`.
- `j = jhash2(&tuple.src.u3, sizeof/sizeof(u32), (range.flags & PERSISTENT) ? 0 : (tuple.dst.u3.all[max] ^ zone.id))`.
- /* Per-word IP selection */
- `full_range = false`.
- for `i ∈ 0..=max`:
  - if `!full_range`: `minip = ntohl(range.min_addr.all[i])`; `maxip = ntohl(range.max_addr.all[i])`; `dist = maxip - minip + 1`.
  - else: `minip = 0`; `dist = ~0`.
  - `var_ipp.all[i] = htonl(minip + reciprocal_scale(j, dist))`.
  - if `var_ipp.all[i] != range.max_addr.all[i]`: `full_range = true`.
  - if `!(range.flags & PERSISTENT)`: `j ^= tuple.dst.u3.all[i]`.

REQ-13: `nf_nat_l4proto_unique_tuple(tuple_io, range, maniptype, ct)`:
- per-protonum keyptr + min + range_size:
  - ICMP/ICMPV6: `keyptr = &tuple.src.u.icmp.id`; if `!PROTO_SPECIFIED`: `min = 0, range_size = 65536`; else `min = ntohs(min.icmp.id)`, `range_size = ntohs(max)-ntohs(min)+1`.
  - GRE: if `!ct.master`: return /* not PPTP child — leave alone */. `keyptr = (SRC) ? &src.u.gre.key : &dst.u.gre.key`; if `!PROTO_SPECIFIED`: `min=1, range_size=65535`; else `min=ntohs(min.gre.key)`, `range_size=ntohs(max)-min+1`.
  - UDP/TCP/SCTP: `keyptr = (SRC) ? &src.u.all : &dst.u.all`.
  - default: return.
- For UDP/TCP/SCTP if `!PROTO_SPECIFIED`:
  - if `MANIP_DST`: return (no DST port rewrite without explicit range).
  - per-RFC-6056-ish port-class tiering:
    - `port < 512`: `min = 1, range_size = 511`.
    - `512 ≤ port < 1024`: `min = 600, range_size = 424` (i.e. `1023 - 600 + 1`).
    - `port ≥ 1024`: `min = 1024, range_size = 64512` (i.e. `65535 - 1024 + 1`).
- else `min = ntohs(range.min_proto.all)`, `max = ntohs(range.max_proto.all)`; if `max < min`: `swap(max, min)`; `range_size = max - min + 1`.
- `find_free_id:`
  - if `PROTO_OFFSET`: `off = ntohs(*keyptr) - ntohs(range.base_proto.all)`.
  - else if `PROTO_RANDOM_ALL ∨ maniptype != DST`: `off = get_random_u16()`.
  - else: `off = 0`.
- `attempts = min(range_size, NF_NAT_MAX_ATTEMPTS)`.
- `'another_round:` for `i ∈ 0..attempts; off += 1`:
  - `*keyptr = htons(min + off % range_size)`.
  - if `!nf_nat_used_tuple_harder(tuple, ct, attempts - i)`: return.
- if `attempts >= range_size ∨ attempts < 16`: return.
- `attempts /= 2`; `off = get_random_u16()`; goto another_round.

REQ-14: `get_unique_tuple(tuple_out, orig_tuple, range, ct, maniptype)`:
- `zone = nf_ct_zone(ct)`; `net = nf_ct_net(ct)`.
- /* 1) SRC reuse */
- if `maniptype == NF_NAT_MANIP_SRC ∧ !(range.flags & PROTO_RANDOM_ALL)`:
  - if `nf_in_range(orig_tuple, range)`:
    - if `!nf_nat_used_tuple_new(orig_tuple, ct)`: `*tuple_out = *orig_tuple`; return.
  - else if `find_appropriate_src(net, zone, orig_tuple, tuple_out, range)`:
    - if `!nf_nat_used_tuple(tuple_out, ct)`: return.
- /* 2) IP selection */
- `*tuple_out = *orig_tuple`.
- `find_best_ips_proto(zone, tuple_out, range, ct, maniptype)`.
- /* 3) Try IP-only / proto-already-in-range fast path */
- if `!(range.flags & PROTO_RANDOM_ALL)`:
  - if `range.flags & PROTO_SPECIFIED`:
    - if `!(range.flags & PROTO_OFFSET) ∧ l4proto_in_range(tuple_out, maniptype, &range.min_proto, &range.max_proto) ∧ (range.min_proto.all == range.max_proto.all ∨ !nf_nat_used_tuple(tuple_out, ct))`: return.
  - else if `!nf_nat_used_tuple(tuple_out, ct)`: return.
- /* 4) L4 port/id allocation */
- `nf_nat_l4proto_unique_tuple(tuple_out, range, maniptype, ct)`.

REQ-15: `nf_ct_nat_ext_add(ct) -> Option<*nf_conn_nat>`:
- `nat = nfct_nat(ct)`.
- if `nat`: return `Some(nat)`.
- if `!nf_ct_is_confirmed(ct)`: `nat = nf_ct_ext_add(ct, NF_CT_EXT_NAT, GFP_ATOMIC)`.
- return `nat`.

REQ-16: `nf_nat_setup_info(ct, range, maniptype) -> u32`:
- if `nf_ct_is_confirmed(ct)`: return `NF_ACCEPT` /* no-op on confirmed ct */.
- WARN if `maniptype ∉ {SRC, DST}`.
- if WARN `nf_nat_initialized(ct, maniptype)`: return `NF_DROP`.
- `nf_ct_invert_tuple(&curr_tuple, &ct.tuplehash[REPLY].tuple)`.
- `get_unique_tuple(&new_tuple, &curr_tuple, range, ct, maniptype)`.
- if `!nf_ct_tuple_equal(&new_tuple, &curr_tuple)`:
  - `nf_ct_invert_tuple(&reply, &new_tuple)`.
  - `nf_conntrack_alter_reply(ct, &reply)`.
  - if `maniptype == SRC`: `ct.status |= IPS_SRC_NAT`; else `|= IPS_DST_NAT`.
  - if `nfct_help(ct) ∧ !nfct_seqadj(ct)`: if `!nfct_seqadj_ext_add(ct)`: return `NF_DROP`.
- if `maniptype == SRC`:
  - `srchash = hash_by_src(net, nf_ct_zone(ct), &ct.tuplehash[ORIGINAL].tuple)`.
  - `lock = &nf_nat_locks[srchash % CONNTRACK_LOCKS]`.
  - `spin_lock_bh(lock)`.
  - `hlist_add_head_rcu(&ct.nat_bysource, &nf_nat_bysource[srchash])`.
  - `spin_unlock_bh(lock)`.
- if `maniptype == DST`: `ct.status |= IPS_DST_NAT_DONE`; else `|= IPS_SRC_NAT_DONE`.
- return `NF_ACCEPT`.

REQ-17: `__nf_nat_alloc_null_binding(ct, manip) -> u32`:
- `ip = (manip == SRC) ? ct.tuplehash[REPLY].tuple.dst.u3 : ct.tuplehash[REPLY].tuple.src.u3`.
- `range = { flags: NF_NAT_RANGE_MAP_IPS, min_addr: ip, max_addr: ip }`.
- return `nf_nat_setup_info(ct, &range, manip)`.

REQ-18: `nf_nat_alloc_null_binding(ct, hooknum) -> u32`:
- return `__nf_nat_alloc_null_binding(ct, HOOK2MANIP(hooknum))`.
- /* `HOOK2MANIP`: POST_ROUTING/LOCAL_IN ⟹ SRC; PRE_ROUTING/LOCAL_OUT ⟹ DST */.

REQ-19: `nf_nat_packet(ct, ctinfo, hooknum, skb) -> u32`:
- `mtype = HOOK2MANIP(hooknum)`; `dir = CTINFO2DIR(ctinfo)`.
- `statusbit = (mtype == SRC) ? IPS_SRC_NAT : IPS_DST_NAT`.
- if `dir == IP_CT_DIR_REPLY`: `statusbit ^= IPS_NAT_MASK`.
- if `ct.status & statusbit`: `verdict = nf_nat_manip_pkt(skb, ct, mtype, dir)`.
- return `verdict`.

REQ-20: `nf_nat_inet_fn(priv, skb, state) -> u32`:
- `maniptype = HOOK2MANIP(state.hook)`.
- `(ct, ctinfo) = nf_ct_get(skb)`.
- if `!ct ∨ in_vrf_postrouting(state)`: return `NF_ACCEPT`.
- `nat = nfct_nat(ct)`.
- switch `ctinfo`:
  - `IP_CT_RELATED` / `_RELATED_REPLY` / `IP_CT_NEW`:
    - if `!nf_nat_initialized(ct, maniptype)`:
      - `lpriv = priv as *nf_nat_lookup_hook_priv`.
      - `e = rcu_dereference(lpriv.entries)`.
      - if `!e`: goto null_bind.
      - for `i ∈ 0..e.num_hook_entries`:
        - `ret = e.hooks[i].hook(e.hooks[i].priv, skb, state)`.
        - if `ret != NF_ACCEPT`: return `ret`.
        - if `nf_nat_initialized(ct, maniptype)`: goto do_nat.
      - `null_bind:` `ret = nf_nat_alloc_null_binding(ct, state.hook)`; if `ret != NF_ACCEPT`: return `ret`.
    - else: if `nf_nat_oif_changed(state.hook, ctinfo, nat, state.out)`: goto oif_changed.
  - default: /* ESTABLISHED */ WARN if `ctinfo ∉ {ESTABLISHED, ESTABLISHED_REPLY}`. if `nf_nat_oif_changed(...)`: goto oif_changed.
- `do_nat:` return `nf_nat_packet(ct, ctinfo, state.hook, skb)`.
- `oif_changed:` `nf_ct_kill_acct(ct, ctinfo, skb)`; return `NF_DROP`.

REQ-21: `in_vrf_postrouting(state) -> bool`:
- if `CONFIG_NET_L3_MASTER_DEV ∧ state.hook == NF_INET_POST_ROUTING ∧ netif_is_l3_master(state.out)`: return true.
- else return false.

REQ-22: `__nf_nat_decode_session(skb, fl)` (CONFIG_XFRM):
- `(ct, ctinfo) = nf_ct_get(skb)`; if `!ct`: return.
- `family = nf_ct_l3num(ct)`; `dir = CTINFO2DIR(ctinfo)`.
- `statusbit = (dir == ORIGINAL) ? IPS_DST_NAT : IPS_SRC_NAT`.
- IPv4: `nf_nat_ipv4_decode_session(skb, ct, dir, statusbit, fl)`.
- IPv6: `nf_nat_ipv6_decode_session(skb, ct, dir, statusbit, fl)`.

REQ-23: `nf_nat_ipv4_decode_session(skb, ct, dir, statusbit, fl)`:
- `t = &ct.tuplehash[dir].tuple`; `fl4 = &fl.u.ip4`.
- if `ct.status & statusbit`: `fl4.daddr = t.dst.u3.ip`; if TCP/UDP/SCTP: `fl4.fl4_dport = t.dst.u.all`.
- `statusbit ^= IPS_NAT_MASK`.
- if `ct.status & statusbit`: `fl4.saddr = t.src.u3.ip`; if TCP/UDP/SCTP: `fl4.fl4_sport = t.src.u.all`.

REQ-24: `nf_nat_ipv6_decode_session(...)`: symmetric using `fl.u.ip6`, `in6` addresses.

REQ-25: `nf_nat_proto_remove(ct, data) -> i32`:
- `clean = data as *nf_nat_proto_clean`.
- if `(clean.l3proto ∧ nf_ct_l3num(ct) != clean.l3proto) ∨ (clean.l4proto ∧ nf_ct_protonum(ct) != clean.l4proto)`: return 0.
- return `(ct.status & IPS_NAT_MASK) ? 1 : 0`.

REQ-26: `nf_nat_cleanup_conntrack(ct)`:
- `h = hash_by_src(nf_ct_net(ct), nf_ct_zone(ct), &ct.tuplehash[ORIGINAL].tuple)`.
- `spin_lock_bh(&nf_nat_locks[h % CONNTRACK_LOCKS])`.
- `hlist_del_rcu(&ct.nat_bysource)`.
- `spin_unlock_bh(...)`.

REQ-27: `nf_nat_proto_clean(ct, data) -> i32`:
- if `nf_nat_proto_remove(ct, data)`: return 1.
- if `test_and_clear_bit(IPS_SRC_NAT_DONE_BIT, &ct.status)`: `nf_nat_cleanup_conntrack(ct)` /* rmmod: detach before table freed */.
- return 0.

REQ-28: `nfnetlink_parse_nat_proto(attr, ct, range) -> i32`:
- `nla_parse_nested_deprecated(tb, CTA_PROTONAT_MAX, attr, protonat_nla_policy, NULL)`.
- if `tb[CTA_PROTONAT_PORT_MIN]`: `range.min_proto.all = nla_get_be16(tb[MIN])`; `range.max_proto.all = min`; `flags |= PROTO_SPECIFIED`.
- if `tb[CTA_PROTONAT_PORT_MAX]`: `range.max_proto.all = nla_get_be16(tb[MAX])`; `flags |= PROTO_SPECIFIED`.

REQ-29: `nfnetlink_parse_nat(nat, ct, range) -> i32`:
- zero `range`.
- `nla_parse_nested_deprecated(tb, CTA_NAT_MAX, nat, nat_nla_policy, NULL)`.
- per `nf_ct_l3num(ct)`:
  - IPv4: `nf_nat_ipv4_nlattr_to_range(tb, range)` — parses `CTA_NAT_V4_MINIP` / `_MAXIP` (MAXIP defaults to MIN).
  - IPv6: `nf_nat_ipv6_nlattr_to_range(tb, range)` — parses `CTA_NAT_V6_MINIP` / `_MAXIP`.
  - default: `-EPROTONOSUPPORT`.
- if `tb[CTA_NAT_PROTO]`: forward to `nfnetlink_parse_nat_proto`.

REQ-30: `nfnetlink_parse_nat_setup(ct, manip, attr) -> i32`:
- WARN if `nf_nat_initialized(ct, manip)`: return -EEXIST.
- if `!attr`: `__nf_nat_alloc_null_binding(ct, manip)`; return 0 or -ENOMEM.
- `nfnetlink_parse_nat(attr, ct, &range)?`.
- return `nf_nat_setup_info(ct, &range, manip) == NF_DROP ? -ENOMEM : 0`.

REQ-31: `nf_nat_register_fn(net, pf, ops, orig_nat_ops, ops_count) -> i32`:
- `nat_net = net_generic(net, nat_net_id)`.
- WARN if `pf >= NFPROTO_NUMPROTO`: return -EINVAL.
- `nat_proto_net = &nat_net.nat_proto_net[pf]`.
- find `i` such that `orig_nat_ops[i].hooknum == ops.hooknum`; WARN if not found.
- lock `PROTO_MUTEX`.
- if `!nat_proto_net.nat_hook_ops`:
  - WARN if `users != 0`.
  - `nat_ops = kmemdup_array(orig_nat_ops, ops_count, ...)` /* deep clone for per-net mutation */.
  - for each `i`: allocate `priv: nf_nat_lookup_hook_priv` and `nat_ops[i].priv = priv`. Rollback prior privs on failure.
  - `nf_register_net_hooks(net, nat_ops, ops_count)?` (rollback privs on failure).
  - `nat_proto_net.nat_hook_ops = nat_ops`.
- `nat_ops = nat_proto_net.nat_hook_ops`.
- `priv = nat_ops[hooknum].priv` (WARN if NULL).
- `nf_hook_entries_insert_raw(&priv.entries, ops)?`.
- on success: `users += 1`.
- unlock `PROTO_MUTEX`.

REQ-32: `nf_nat_unregister_fn(net, pf, ops, ops_count)`:
- `nat_proto_net = &nat_net.nat_proto_net[pf]`.
- lock `PROTO_MUTEX`.
- WARN if `users == 0`: unlock.
- `users -= 1`.
- find matching `i` for `ops.hooknum`; `priv = nat_ops[i].priv`.
- `nf_hook_entries_delete_raw(&priv.entries, ops)`.
- if `users == 0`:
  - `nf_unregister_net_hooks(net, nat_ops, ops_count)`.
  - for each `i`: `kfree_rcu(priv, rcu_head)`.
  - `nat_proto_net.nat_hook_ops = NULL`; `kfree_rcu(nat_ops, rcu)`.
- unlock.

REQ-33: `nat_net_ops` (pernet_operations): `id = &nat_net_id`, `size = sizeof(struct nat_net)`.

REQ-34: `struct nf_nat_hook nat_hook`:
- `.parse_nat_setup = nfnetlink_parse_nat_setup`.
- if XFRM: `.decode_session = __nf_nat_decode_session`.
- `.remove_nat_bysrc = nf_nat_cleanup_conntrack`.
- /* Published via `RCU_INIT_POINTER(nf_nat_hook, &nat_hook)` at init */.

REQ-35: `nf_nat_init()` (module_init):
- `nf_nat_htable_size = nf_conntrack_htable_size`; lower-clamp to `CONNTRACK_LOCKS`.
- `nf_nat_bysource = nf_ct_alloc_hashtable(&nf_nat_htable_size, 0)` (vmalloc-fallback hashtable); fail -ENOMEM.
- `spin_lock_init` each `nf_nat_locks[0..CONNTRACK_LOCKS]`.
- `register_pernet_subsys(&nat_net_ops)?` (free table on failure).
- `nf_ct_helper_expectfn_register(&follow_master_nat)`.
- WARN `nf_nat_hook != NULL`; `RCU_INIT_POINTER(nf_nat_hook, &nat_hook)`.
- `register_nf_nat_bpf()?` — on failure: NULL out hook, unregister expectfn, sync_net, unregister pernet, free table.

REQ-36: `nf_nat_cleanup()` (module_exit):
- `nf_ct_iterate_destroy(nf_nat_proto_clean, &{})` — visit every ct, drop NAT bindings, detach bysource.
- `nf_ct_helper_expectfn_unregister(&follow_master_nat)`.
- `RCU_INIT_POINTER(nf_nat_hook, NULL)`.
- `synchronize_net()`.
- `kvfree(nf_nat_bysource)`.
- `unregister_pernet_subsys(&nat_net_ops)`.

REQ-37: Module: GPL, description "Network address translation core".

## Acceptance Criteria

- [ ] AC-1: `nf_nat_setup_info` on a confirmed ct returns `NF_ACCEPT` no-op (no rewrite).
- [ ] AC-2: First SNAT setup: `IPS_SRC_NAT` and `IPS_SRC_NAT_DONE` bits set; ct inserted into `nf_nat_bysource` bucket under stripe lock.
- [ ] AC-3: First DNAT setup: `IPS_DST_NAT` and `IPS_DST_NAT_DONE` bits set; ct NOT inserted into bysource (DST only).
- [ ] AC-4: `get_unique_tuple` SRC path: if `orig_tuple` already in range and not taken, returns it unchanged.
- [ ] AC-5: `get_unique_tuple` SRC path: if same `src/protonum` already mapped (memoised via bysource), reuses prior mapping when `nf_in_range`.
- [ ] AC-6: `nf_nat_l4proto_unique_tuple` UDP without `PROTO_SPECIFIED` and port<512 picks from `[1, 511]`.
- [ ] AC-7: `nf_nat_l4proto_unique_tuple` UDP without `PROTO_SPECIFIED` and 512≤port<1024 picks from `[600, 1023]`.
- [ ] AC-8: `nf_nat_l4proto_unique_tuple` UDP without `PROTO_SPECIFIED` and port≥1024 picks from `[1024, 65535]`.
- [ ] AC-9: `nf_nat_l4proto_unique_tuple` after `NF_NAT_MAX_ATTEMPTS=128` failures with `range_size > 128` halves attempts and retries from a fresh random offset.
- [ ] AC-10: `nf_nat_used_tuple_harder` with `attempts_left ≤ 32` and TCP SYN_SENT may evict an EVICTABLE TIME_WAIT entry (seq advanced).
- [ ] AC-11: `nf_nat_used_tuple_harder` refuses to evict OFFLOAD/HW_OFFLOAD ct.
- [ ] AC-12: `nf_nat_packet` on SRC binding rewrites src; reply direction inverts statusbit and rewrites dst.
- [ ] AC-13: `nf_nat_inet_fn` on `IP_CT_NEW` with no installed NAT rules calls `nf_nat_alloc_null_binding` (range = REPLY.dst.ip / src.ip).
- [ ] AC-14: `nf_nat_inet_fn` on `IP_CT_NEW` runs registered hook chain in order, stops at non-ACCEPT or once initialised.
- [ ] AC-15: `nf_nat_inet_fn` ESTABLISHED with `oif_changed` returns `NF_DROP` and kills ct.
- [ ] AC-16: `nf_nat_inet_fn` on VRF POST_ROUTING (`netif_is_l3_master(state.out)`) returns ACCEPT bypass.
- [ ] AC-17: `nf_nat_cleanup_conntrack` removes ct from bysource under correct stripe lock.
- [ ] AC-18: `nfnetlink_parse_nat` with `CTA_NAT_V4_MINIP` only: `range.max_addr.ip = range.min_addr.ip` (default-to-MIN).
- [ ] AC-19: `nf_nat_register_fn` second call (same hooknum, same pf) reuses `nat_hook_ops` and bumps `users` without re-registering.
- [ ] AC-20: `nf_nat_unregister_fn` final put on `users==0` unregisters hooks + RCU-frees `nat_hook_ops` + per-hook privs.
- [ ] AC-21: `nf_nat_init` allocates `nf_nat_bysource` with `htable_size ≥ CONNTRACK_LOCKS`.
- [ ] AC-22: `nf_nat_cleanup` iterates every ct and clears bysource before freeing the table.

## Architecture

```
const NF_NAT_MAX_ATTEMPTS:   u32 = 128;
const NF_NAT_HARDER_THRESH:  u32 = NF_NAT_MAX_ATTEMPTS / 4;   // 32

static BYSOURCE:    *mut [HlistHead] = ptr::null_mut();        // length HTABLE_SIZE
static HTABLE_SIZE: AtomicUsize       = AtomicUsize::new(0);
static HASH_RND:    SiphashAlignedKey = SiphashAlignedKey::new();   // lazy via get_random_once
static LOCKS:       [SpinLock; CONNTRACK_LOCKS] = ...;
static PROTO_MUTEX: Mutex = Mutex::new();
static NAT_NET_ID:  AtomicU32         = AtomicU32::new(0);

struct NatNet {
  nat_proto_net: [NfNatHooksNet; NFPROTO_NUMPROTO],
}

struct NfNatHooksNet {
  nat_hook_ops: *mut [NfHookOps],   // per-net deep clone of orig_nat_ops, with priv = NfNatLookupHookPriv
  users:        u32,
}

struct NfNatLookupHookPriv {
  entries: *const NfHookEntries,    // RCU
  rcu_head: RcuHead,
}
```

`Nat::hash_by_src(net, zone, tuple) -> u32`:
1. `combined = NatHashKey { src: tuple.src, net_mix: net_hash_mix(net), protonum: tuple.dst.protonum, zone: 0 }`.
2. if `zone.dir == NF_CT_DEFAULT_ZONE_DIR`: `combined.zone = zone.id`.
3. `get_random_once(&mut HASH_RND, size_of::<HashKey>())`.
4. `h = siphash(&combined, size_of::<HashKey>(), &HASH_RND)`.
5. `reciprocal_scale(h, HTABLE_SIZE)`.

`Nat::get_unique_tuple(tuple_out, orig, range, ct, maniptype)`:
1. `zone = nf_ct_zone(ct)`; `net = nf_ct_net(ct)`.
2. if `maniptype == SRC && !(range.flags & PROTO_RANDOM_ALL)`:
   - if `Nat::in_range(orig, range)` and not used: `*tuple_out = *orig`; return.
   - else `Nat::find_appropriate_src(net, zone, orig, tuple_out, range)?` and not used: return.
3. `*tuple_out = *orig`; `Nat::find_best_ips_proto(zone, tuple_out, range, ct, maniptype)`.
4. if `!(range.flags & PROTO_RANDOM_ALL)`:
   - if `range.flags & PROTO_SPECIFIED`: l4_in_range + single-port-or-not-used ⟹ return.
   - else if not used ⟹ return.
5. `Nat::l4proto_unique_tuple(tuple_out, range, maniptype, ct)`.

`Nat::setup_info(ct, range, maniptype) -> Verdict`:
1. if `nf_ct_is_confirmed(ct)`: return Accept.
2. WARN if maniptype ∉ {SRC,DST}; WARN if `nf_nat_initialized(ct, maniptype)` ⟹ Drop.
3. `Nat::get_unique_tuple(&mut new_tuple, invert(REPLY), range, ct, maniptype)`.
4. if `new_tuple != curr_tuple`:
   - `nf_conntrack_alter_reply(ct, &invert(new_tuple))`.
   - set `IPS_SRC_NAT | IPS_DST_NAT` bit per maniptype.
   - if helper attached and no seqadj: `nfct_seqadj_ext_add(ct)?`.
5. if maniptype == SRC: bysource-insert under `LOCKS[srchash % CONNTRACK_LOCKS]`.
6. set `IPS_*_NAT_DONE`.
7. Accept.

`Nat::inet_fn(priv, skb, state) -> Verdict`:
1. `maniptype = HOOK2MANIP(state.hook)`.
2. `(ct, ctinfo) = nf_ct_get(skb)`.
3. if `ct.is_none() || in_vrf_postrouting(state)`: Accept.
4. match ctinfo:
   - NEW / RELATED / RELATED_REPLY:
     - if `!initialized(ct, maniptype)`:
       - run registered hook chain (`lpriv.entries`) until non-ACCEPT or initialised.
       - if still uninitialised: `Nat::alloc_null_binding(ct, state.hook)`.
     - else if `nf_nat_oif_changed(...)`: oif_changed.
   - ESTABLISHED / ESTABLISHED_REPLY: if `nf_nat_oif_changed(...)`: oif_changed.
5. `Nat::packet(ct, ctinfo, state.hook, skb)`.
6. `oif_changed:` `nf_ct_kill_acct(ct, ctinfo, skb)`; return Drop.

`Nat::packet(ct, ctinfo, hooknum, skb) -> Verdict`:
1. `mtype = HOOK2MANIP(hooknum)`; `dir = CTINFO2DIR(ctinfo)`.
2. `statusbit = (mtype == SRC) ? IPS_SRC_NAT : IPS_DST_NAT`.
3. if `dir == REPLY`: `statusbit ^= IPS_NAT_MASK`.
4. if `ct.status & statusbit`: `nf_nat_manip_pkt(skb, ct, mtype, dir)` else Accept.

`Nat::register_fn(net, pf, ops, orig_nat_ops, ops_count) -> Result<()>`:
1. validate `pf < NFPROTO_NUMPROTO`.
2. find `hooknum` index in `orig_nat_ops`.
3. `PROTO_MUTEX.lock()`.
4. if first user: deep-clone `nat_ops`, allocate per-hook priv, `nf_register_net_hooks(...)`, install.
5. `nf_hook_entries_insert_raw(&priv.entries, ops)`.
6. `users += 1`.
7. unlock.

`Nat::module_init() -> Result<()>`:
1. `HTABLE_SIZE = max(nf_conntrack_htable_size, CONNTRACK_LOCKS)`.
2. `BYSOURCE = nf_ct_alloc_hashtable(&HTABLE_SIZE, 0)`.
3. init `LOCKS[..]`.
4. `register_pernet_subsys(&PERNET_OPS)?`.
5. `nf_ct_helper_expectfn_register(&FOLLOW_MASTER)`.
6. publish `NAT_HOOK` via `RCU_INIT_POINTER(nf_nat_hook, ...)`.
7. `register_nf_nat_bpf()?` (full rollback on failure).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `bysource_lock_correct_stripe` | INVARIANT | per-`setup_info` / `cleanup_conntrack`: bucket `h` accessed under `LOCKS[h % CONNTRACK_LOCKS]`. |
| `bysource_rcu_protected_walk` | INVARIANT | per-`find_appropriate_src`: walk uses `hlist_for_each_entry_rcu` under RCU read lock. |
| `setup_info_no_double_init` | INVARIANT | per-`setup_info`: WARN+Drop if `nf_nat_initialized(ct, maniptype)`. |
| `setup_info_confirmed_noop` | INVARIANT | per-`setup_info`: confirmed ct ⟹ Accept no-op. |
| `setup_info_src_inserts_bysource` | INVARIANT | per-`setup_info` maniptype==SRC ⟹ exactly one `hlist_add_head_rcu` to `BYSOURCE[h]`. |
| `setup_info_dst_skip_bysource` | INVARIANT | per-`setup_info` maniptype==DST ⟹ no bysource insert. |
| `setup_info_statusbits_correct` | INVARIANT | per-`setup_info`: SRC ⟹ `IPS_SRC_NAT|IPS_SRC_NAT_DONE`; DST ⟹ `IPS_DST_NAT|IPS_DST_NAT_DONE`. |
| `l4proto_unique_tuple_bounded` | INVARIANT | per-`l4proto_unique_tuple`: total iterations ≤ `NF_NAT_MAX_ATTEMPTS` per round; rounds halve attempts to ≥16. |
| `harder_evict_only_timewait` | INVARIANT | per-`used_tuple_harder`: eviction requires `tcp.state >= TIME_WAIT ∧ !(status & FIXED_TIMEOUT|DYING) ∧ (status & SRC_NAT)`. |
| `harder_evict_skip_offload` | INVARIANT | per-`used_tuple_harder`: `(status & (IPS_OFFLOAD | IPS_HW_OFFLOAD)) != 0` ⟹ tuple reported as taken. |
| `inet_fn_oif_changed_drops` | INVARIANT | per-`inet_fn`: oif change ⟹ `nf_ct_kill_acct` then Drop. |
| `inet_fn_vrf_bypass` | INVARIANT | per-`inet_fn`: `in_vrf_postrouting` ⟹ Accept (no NAT). |
| `register_fn_first_user_clones_ops` | INVARIANT | per-`register_fn`: first user duplicates `orig_nat_ops`; subsequent users share. |
| `register_fn_priv_rcu_freed` | INVARIANT | per-`unregister_fn`: priv freed via `kfree_rcu`. |
| `decode_session_xfrm_direction_correct` | INVARIANT | per-`__nf_nat_decode_session`: `dir==ORIGINAL` ⟹ probe `IPS_DST_NAT` first; `REPLY` ⟹ `IPS_SRC_NAT` first. |
| `null_binding_uses_reply_address` | INVARIANT | per-`__nf_nat_alloc_null_binding`: range pinned to `REPLY.tuple.{dst|src}.u3` (no actual rewrite needed). |

### Layer 2: TLA+

`net/netfilter/nat-bysource.tla`:
- Per-bucket state machine: `Empty`, `Inserted(ct)`, `Removed(ct)`.
- Properties:
  - `safety_bysource_stripe_lock` — every insert/del uses `LOCKS[bucket % CONNTRACK_LOCKS]`.
  - `safety_no_double_insert` — `nat_bysource` node never inserted twice without intervening remove.
  - `safety_cleanup_on_destroy` — ct destruction with `IPS_SRC_NAT_DONE` triggers exactly one bysource remove.
  - `liveness_rcu_walk_eventually_observes_insert` — readers reach grace period within bounded steps.

`net/netfilter/nat-port-allocation.tla`:
- Per-`l4proto_unique_tuple` allocator FSM: `OffsetSelect`, `LinearSearch`, `Halve`, `Found`, `Exhausted`.
- Properties:
  - `safety_attempts_bounded` — total iterations across all rounds ≤ `NF_NAT_MAX_ATTEMPTS * log2(range_size/16)`.
  - `safety_terminates` — `attempts < 16` exit guarantees termination.
  - `liveness_finds_free_when_available` — non-saturated range yields Found.
  - `safety_proto_random_all_picks_random` — `PROTO_RANDOM_ALL` ⟹ initial off via `get_random_u16`.

`net/netfilter/nat-inet-fn.tla`:
- Per-packet hook FSM: `Enter`, `NoCt`, `VrfBypass`, `RunHookChain`, `NullBind`, `OifChanged`, `DoNat`, `Drop`.
- Properties:
  - `safety_uninitialized_runs_chain_then_null_binds` — `NEW ∧ !initialized` ⟹ chain run; fallback `alloc_null_binding`.
  - `safety_oif_changed_kills_ct` — `nf_nat_oif_changed` ⟹ `nf_ct_kill_acct`.
  - `liveness_packet_reaches_verdict` — every path terminates with Accept/Drop/chain-verdict.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Nat::setup_info` post: confirmed ⟹ Accept no-op | `Nat::setup_info` |
| `Nat::setup_info` post: SRC ⟹ bysource-inserted ∧ IPS_SRC_NAT[_DONE] set | `Nat::setup_info` |
| `Nat::setup_info` post: DST ⟹ IPS_DST_NAT[_DONE] set ∧ NOT bysource-inserted | `Nat::setup_info` |
| `Nat::get_unique_tuple` post: `tuple_out` ∈ range or `nf_in_range` false on output (l4proto allocator may extend beyond) | `Nat::get_unique_tuple` |
| `Nat::l4proto_unique_tuple` post: iterations ≤ `NF_NAT_MAX_ATTEMPTS` per round | `Nat::l4proto_unique_tuple` |
| `Nat::used_tuple_harder` post: `attempts_left > 32 ∨ proto != TCP ∨ state != SYN_SENT` ⟹ degrades to `nf_conntrack_tuple_taken` | `Nat::used_tuple_harder` |
| `Nat::inet_fn` post: returns Accept / Drop / chain-verdict (exactly one) | `Nat::inet_fn` |
| `Nat::packet` post: rewrite occurs iff `ct.status & statusbit` after dir-XOR | `Nat::packet` |
| `Nat::register_fn` post: Ok ⟹ `users` incremented; first-user installed hooks | `Nat::register_fn` |
| `Nat::unregister_fn` post: `users==0` ⟹ hooks unregistered + priv kfree_rcu'd | `Nat::unregister_fn` |
| `Nat::cleanup_conntrack` post: ct no longer in `BYSOURCE[h]` after return | `Nat::cleanup_conntrack` |
| `Nat::module_init` post: Ok ⟹ `BYSOURCE != NULL ∧ nf_nat_hook == &NAT_HOOK ∧ pernet+bpf registered` | `Nat::module_init` |

### Layer 4: Verus/Creusot functional

`Per-NAT setup_info → get_unique_tuple → (find_appropriate_src ∨ find_best_ips_proto ∨ l4proto_unique_tuple) → nf_conntrack_alter_reply → bysource-insert (SRC) → IPS_*_NAT_DONE` semantic equivalence: per-`include/uapi/linux/netfilter/nf_nat.h` (`NF_NAT_RANGE_MAP_IPS`, `_PROTO_SPECIFIED`, `_PROTO_RANDOM_ALL`, `_PROTO_RANDOM_FULLY`, `_PERSISTENT`, `_PROTO_OFFSET`, `_NETMAP`).

`Per-packet inet_fn → ctinfo-dispatch → registered chain | alloc_null_binding → nf_nat_packet → nf_nat_manip_pkt` semantic equivalence: per-RFC-3022 (Traditional NAT), RFC-4787 (UDP NAT BEHAVE), RFC-5382 (TCP NAT requirements) — Endpoint-Independent Mapping observed via `find_appropriate_src` memoisation, Hairpinning via `nf_nat_used_tuple_new` clash-resolution, Port-Preservation observed via "try the original tuple first" branch.

## Hardening

(Inherits row-1 features from `net/netfilter/00-overview.md` § Hardening.)

NAT-core reinforcement:

- **Per-`NF_NAT_MAX_ATTEMPTS` cap + attempts-halving rounds with `< 16` exit** — defense against per-softirq soft-lockup when all ports in saturated range are in use.
- **Per-`NF_NAT_HARDER_THRESH` TIME_WAIT eviction gated on TCP SYN_SENT only** — defense against per-DoS of TIME_WAIT pools by aggressive port reuse from non-NAT-needed flows.
- **Per-`IPS_OFFLOAD | IPS_HW_OFFLOAD` eviction veto in `used_tuple_harder`** — defense against per-evict-of-offloaded-flow-then-reuse leading to hardware/software state desync.
- **Per-`siphash` keyed by `get_random_once` HASH_RND** — defense against per-hash-collision DoS via crafted src-port enumeration.
- **Per-bucket stripe lock `LOCKS[h % CONNTRACK_LOCKS]`** — defense against per-global-lock contention; bounds critical section to one stripe.
- **Per-RCU bysource walk + `kfree_rcu` of priv** — defense against per-UAF when removing ct under live `find_appropriate_src` traversal.
- **Per-`nf_nat_used_tuple_new` clash-resolution path requires both ct's non-NAT** — defense against per-silent-rewrite of established NAT mapping by parallel new flow.
- **Per-`WARN_ON nf_nat_initialized` re-init refusal** — defense against per-double-binding corrupting `IPS_*_NAT_DONE` semantics.
- **Per-`PROTO_MUTEX` for register/unregister + WARN on `users == 0` underflow** — defense against per-leak / per-double-put of `nat_hook_ops`.
- **Per-`oif_changed` ⟹ `nf_ct_kill_acct` + Drop** — defense against per-stale-route-after-routing-table-change leaking packets to wrong interface.
- **Per-VRF POST_ROUTING bypass** — defense against per-VRF-leaf NAT collision with L3-master masquerade.
- **Per-CONFIG_XFRM `__nf_nat_decode_session` direction-bit-XOR** — defense against per-XFRM-flowi mismatch in reply direction.
- **Per-`nf_nat_proto_clean` at module exit iterates every ct + detaches bysource before kvfree** — defense against per-UAF when ct destroy callback fires post-table-free.
- **Per-`register_nf_nat_bpf` full rollback on failure** — defense against per-partially-initialised module exposing dangling `nf_nat_hook`.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `nf_nat_proto.c` per-l4-protocol packet manip (`nf_nat_manip_pkt` family: IPv4/IPv6/TCP/UDP/UDPLite/ICMP/ICMPv6/SCTP/GRE) — covered separately if expanded.
- `nf_nat_masquerade.c` masquerade target + auto-flush on ifdown — covered in `nat-helpers.md` if expanded.
- `nf_nat_redirect.c` — covered separately if expanded.
- `nf_nat_helper.c` `nf_nat_seq_adjust` + payload mangler — covered separately if expanded.
- `nf_nat_amanda.c` / `_ftp.c` / `_irc.c` / `_sip.c` / `_tftp.c` / `_h323.c` ALG helpers — covered in `nat-helpers.md` Tier-3.
- `nf_nat_bpf.c` BPF NAT — covered separately if expanded.
- ctnetlink protocol-level dump/load — covered in `conntrack-netlink.md` Tier-3.
- `nf_conntrack_core.c` ct lifecycle — covered in `nf-conntrack-core.md` Tier-3.
- Per-l4proto state machines — covered separately.
- Implementation code
