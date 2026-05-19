# Tier-3: net/netfilter/nf_conntrack_proto.c — Conntrack protocol dispatch

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/netfilter/00-overview.md
upstream-paths:
  - net/netfilter/nf_conntrack_proto.c (~693 lines)
  - include/net/netfilter/nf_conntrack_l4proto.h
  - include/net/netfilter/nf_conntrack.h
  - include/uapi/linux/netfilter/nf_conntrack_common.h
-->

## Summary

`nf_conntrack_proto.c` is the **L4-protocol dispatch + per-AF hook registration** layer for Netfilter connection tracking. It owns: per-`l4proto` lookup (`nf_ct_l4proto_find` returns a `const struct nf_conntrack_l4proto *` for TCP / UDP / UDPLite / ICMP / ICMPv6 / SCTP / GRE / generic), per-net per-AF hook activation (`nf_ct_netns_get` / `_put` ref-counting `users4` / `users6` / `users_bridge` and registering the `ipv4_conntrack_ops` / `ipv6_conntrack_ops` arrays at `NF_INET_PRE_ROUTING`, `_LOCAL_OUT`, `_POST_ROUTING`, `_LOCAL_IN`), the canonical `nf_confirm()` hook that drives helpers + sequence-adjust at confirm time, IPv4-fragment-aware `ipv4_conntrack_local`, `getsockopt(SO_ORIGINAL_DST)` / `IP6T_SO_ORIGINAL_DST` for masqueraded sockets, bridge-conntrack glue (`nf_ct_bridge_register` / `_unregister`), and per-net per-l4proto state initialisation (`nf_conntrack_proto_pernet_init` chains generic / udp / tcp / icmp[/icmpv6/sctp/gre]). Critical for: dispatching every conntrack packet to the right state-machine, scoping conntrack lifetime to namespace usage, exposing pre-NAT 4-tuple to applications (proxies), keeping bridge-NF on a separate refcount.

This Tier-3 covers `net/netfilter/nf_conntrack_proto.c` (~693 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `nf_ct_l4proto_find(l4proto)` | per-protonum dispatch | `Proto::l4proto_find` |
| `nf_conntrack_l4proto_generic` | per-fallback ops | `Proto::L4_GENERIC` |
| `nf_conntrack_l4proto_tcp` | per-TCP ops | `Proto::L4_TCP` |
| `nf_conntrack_l4proto_udp` | per-UDP ops | `Proto::L4_UDP` |
| `nf_conntrack_l4proto_icmp` | per-ICMPv4 ops | `Proto::L4_ICMP` |
| `nf_conntrack_l4proto_icmpv6` | per-ICMPv6 ops | `Proto::L4_ICMPV6` |
| `nf_conntrack_l4proto_sctp` | per-SCTP ops (CONFIG) | `Proto::L4_SCTP` |
| `nf_conntrack_l4proto_gre` | per-GRE ops (CONFIG) | `Proto::L4_GRE` |
| `nf_l4proto_log_invalid()` | per-`sysctl_log_invalid` log | `Proto::log_invalid` |
| `nf_ct_l4proto_log_invalid()` | per-ct log | `Proto::ct_log_invalid` |
| `nf_confirm()` | confirm-hook entry | `Proto::confirm_hook` |
| `ipv4_conntrack_in()` | PRE_ROUTING/LOCAL_OUT v4 | `Proto::v4_in` |
| `ipv4_conntrack_local()` | LOCAL_OUT v4 (frag-aware) | `Proto::v4_local` |
| `ipv6_conntrack_in()` | PRE_ROUTING/LOCAL_OUT v6 | `Proto::v6_in` |
| `ipv6_conntrack_local()` | LOCAL_OUT v6 | `Proto::v6_local` |
| `ipv4_conntrack_ops[]` | per-AF nf_hook_ops table | `Proto::V4_HOOKS` |
| `ipv6_conntrack_ops[]` | per-AF nf_hook_ops table | `Proto::V6_HOOKS` |
| `nf_ct_netns_get(net, nfproto)` | per-net per-AF refcount up | `Proto::netns_get` |
| `nf_ct_netns_put(net, nfproto)` | per-net per-AF refcount down | `Proto::netns_put` |
| `nf_ct_netns_do_get()` | per-AF inner refcount + register | `Proto::netns_do_get` |
| `nf_ct_netns_do_put()` | per-AF inner refcount + unregister | `Proto::netns_do_put` |
| `nf_ct_netns_inet_get()` | per-`NFPROTO_INET` v4+v6 combo | `Proto::netns_inet_get` |
| `nf_ct_tcp_fixup()` | per-fixup TCP max-win on enable | `Proto::tcp_fixup` |
| `nf_ct_bridge_register()` / `_unregister()` | per-`nf_ct_bridge_info` | `Proto::bridge_register` / `_unregister` |
| `getorigdst()` | per-`SO_ORIGINAL_DST` v4 | `Proto::getorigdst_v4` |
| `ipv6_getorigdst()` | per-`IP6T_SO_ORIGINAL_DST` v6 | `Proto::getorigdst_v6` |
| `so_getorigdst` / `so_getorigdst6` | per-`nf_sockopt_ops` | `Proto::SO_GETORIGDST` / `_V6` |
| `nf_conntrack_proto_init()` | per-module init | `Proto::module_init` |
| `nf_conntrack_proto_fini()` | per-module fini | `Proto::module_fini` |
| `nf_conntrack_proto_pernet_init()` | per-net L4 init chain | `Proto::pernet_init` |
| `nf_ct_bridge_info` (static) | per-bridge plugin slot | `Proto::BRIDGE_INFO` |
| `nf_ct_proto_mutex` | per-registry mutex | `Proto::PROTO_MUTEX` |

## Compatibility contract

REQ-1: `struct nf_conntrack_l4proto` (per-protocol vtable) MUST expose:
- `l4proto: u8` — IANA protonum (or pseudo-`IPPROTO_RAW` for generic).
- `allow_clash: bool` — per-`udp.c` / `icmp.c` clash-resolve permit.
- `pkt_to_tuple(skb, dataoff, net, tuple) -> bool` — per-L4-header parse into tuple.
- `invert_tuple(inverse, orig) -> bool` — per-direction inversion.
- `packet(ct, skb, dataoff, ctinfo, state) -> int` — per-state-machine step; returns `NF_ACCEPT` / `NF_DROP` / negative-errno.
- `can_early_drop(ct) -> bool` — per-table-pressure eviction permit.
- `nlattr_size: usize` / `to_nlattr(skb, nlh, ct, destroy) -> int` / `from_nlattr(...)` — per-ctnetlink dump/load.
- `tuple_to_nlattr` / `nlattr_tuple_size` / `nlattr_to_tuple` — per-ctnetlink tuple I/O.
- `nla_policy: *const nla_policy` / `nla_size` — per-policy.
- `print_tuple(seq, tuple)` — per-/proc/net/nf_conntrack.

REQ-2: `nf_ct_l4proto_find(l4proto: u8) -> &'static nf_conntrack_l4proto`:
- /* Compile-time-built switch (NOT a runtime registry — registration is implicit at module link) */
- match `l4proto`:
  - `IPPROTO_UDP` (17) → `&nf_conntrack_l4proto_udp`.
  - `IPPROTO_TCP` (6) → `&nf_conntrack_l4proto_tcp`.
  - `IPPROTO_ICMP` (1) → `&nf_conntrack_l4proto_icmp`.
  - `IPPROTO_SCTP` (132) if `CONFIG_NF_CT_PROTO_SCTP` → `&nf_conntrack_l4proto_sctp`.
  - `IPPROTO_GRE` (47) if `CONFIG_NF_CT_PROTO_GRE` → `&nf_conntrack_l4proto_gre`.
  - `IPPROTO_ICMPV6` (58) if `CONFIG_IPV6` → `&nf_conntrack_l4proto_icmpv6`.
  - else → `&nf_conntrack_l4proto_generic`.
- /* MUST NOT return NULL — generic is the always-present fallback */

REQ-3: `nf_l4proto_log_invalid(skb, state, protonum, fmt, ...)`:
- if `state->net->ct.sysctl_log_invalid != protonum ∧ != IPPROTO_RAW`: return (per-sysctl filter).
- `nf_log_packet(net, state.pf, 0, skb, state.in, state.out, NULL, "nf_ct_proto_%d: %pV ", protonum, &vaf)`.
- /* `IPPROTO_RAW` (255) is the "log-everything" magic value */

REQ-4: `nf_ct_l4proto_log_invalid(skb, ct, state, fmt, ...)`:
- `net = nf_ct_net(ct)`.
- if `likely(net->ct.sysctl_log_invalid == 0)`: return.
- forwards to `nf_l4proto_log_invalid` with `nf_ct_protonum(ct)`.

REQ-5: `nf_confirm(priv, skb, state) -> u32` — POST_ROUTING / LOCAL_IN hook:
- `ct = nf_ct_get(skb, &ctinfo)`.
- if `!ct ∨ in_vrf_postrouting(state)`: return `NF_ACCEPT`.
- /* `in_vrf_postrouting`: `CONFIG_NET_L3_MASTER_DEV` ∧ `state.hook == POST_ROUTING` ∧ `netif_is_l3_master(state.out)` */
- `help = nfct_help(ct)`.
- `seqadj_needed = test_bit(IPS_SEQ_ADJUST_BIT, &ct.status) ∧ !nf_is_loopback_packet(skb)`.
- if `!help ∧ !seqadj_needed`: return `nf_conntrack_confirm(skb)`.
- if `ctinfo == IP_CT_RELATED_REPLY`: return `nf_conntrack_confirm(skb)`  /* helpers don't see ICMP errors */.
- compute `protoff` per `nf_ct_l3num(ct)`:
  - `NFPROTO_IPV4`: `protoff = skb_network_offset(skb) + ip_hdrlen(skb)`.
  - `NFPROTO_IPV6`: `pnum = ipv6_hdr(skb).nexthdr`; `start = ipv6_skip_exthdr(skb, sizeof(ipv6hdr), &pnum, &frag_off)`; if `start < 0 ∨ (frag_off & htons(~0x7)) != 0`: return `nf_conntrack_confirm(skb)`; `protoff = start`.
  - default: return `nf_conntrack_confirm(skb)`.
- if `help`: `helper = rcu_dereference(help.helper)`; if `helper`: `ret = helper.help(skb, protoff, ct, ctinfo)`; if `ret != NF_ACCEPT`: return `ret`.
- if `seqadj_needed ∧ !nf_ct_seq_adjust(skb, ct, ctinfo, protoff)`: `NF_CT_STAT_INC_ATOMIC(net, drop)`; return `NF_DROP`.
- return `nf_conntrack_confirm(skb)`.

REQ-6: `ipv4_conntrack_in(priv, skb, state) -> u32`:
- thin forwarder: `return nf_conntrack_in(skb, state)`.

REQ-7: `ipv4_conntrack_local(priv, skb, state) -> u32`:
- /* LOCAL_OUT entry — must tolerate `IP_NODEFRAG` sockets that ship fragments out */
- if `ip_is_fragment(ip_hdr(skb))`:
  - `tmpl = nf_ct_get(skb, &ctinfo)`.
  - if `tmpl ∧ nf_ct_is_template(tmpl)`:
    - `skb._nfct = 0`  /* clear template — later matches must not be fooled */.
    - `nf_ct_put(tmpl)`.
  - return `NF_ACCEPT`.
- return `nf_conntrack_in(skb, state)`.

REQ-8: `ipv4_conntrack_ops[]` (ordered, priority-tagged):
1. `ipv4_conntrack_in` @ `NFPROTO_IPV4`, `NF_INET_PRE_ROUTING`, `NF_IP_PRI_CONNTRACK`.
2. `ipv4_conntrack_local` @ `NFPROTO_IPV4`, `NF_INET_LOCAL_OUT`, `NF_IP_PRI_CONNTRACK`.
3. `nf_confirm` @ `NFPROTO_IPV4`, `NF_INET_POST_ROUTING`, `NF_IP_PRI_CONNTRACK_CONFIRM`.
4. `nf_confirm` @ `NFPROTO_IPV4`, `NF_INET_LOCAL_IN`, `NF_IP_PRI_CONNTRACK_CONFIRM`.

REQ-9: `ipv6_conntrack_ops[]` (CONFIG_IPV6, ordered, priority-tagged):
1. `ipv6_conntrack_in` @ `NFPROTO_IPV6`, `NF_INET_PRE_ROUTING`, `NF_IP6_PRI_CONNTRACK`.
2. `ipv6_conntrack_local` @ `NFPROTO_IPV6`, `NF_INET_LOCAL_OUT`, `NF_IP6_PRI_CONNTRACK`.
3. `nf_confirm` @ `NFPROTO_IPV6`, `NF_INET_POST_ROUTING`, `NF_IP6_PRI_LAST`.
4. `nf_confirm` @ `NFPROTO_IPV6`, `NF_INET_LOCAL_IN`, `NF_IP6_PRI_LAST - 1`.

REQ-10: `nf_ct_netns_do_get(net, nfproto) -> int` (per-AF refcount + register):
- `cnet = nf_ct_pernet(net)`.
- `fixup_needed = false`; `retry = true`.
- `retry:` `mutex_lock(&nf_ct_proto_mutex)`.
- match `nfproto`:
  - `NFPROTO_IPV4`: `cnet.users4 += 1`; if `users4 > 1`: goto `out_unlock`. `err = nf_defrag_ipv4_enable(net)`; if `err`: `users4 = 0`; goto unlock. `err = nf_register_net_hooks(net, ipv4_conntrack_ops, ARRAY_SIZE)`; if err: `users4 = 0`; else `fixup_needed = true`.
  - `NFPROTO_IPV6` (CONFIG_IPV6): same shape with `users6` + `nf_defrag_ipv6_enable` + `ipv6_conntrack_ops`.
  - `NFPROTO_BRIDGE`: if `!nf_ct_bridge_info`: if `!retry`: `err = -EPROTO`; else unlock + `request_module("nf_conntrack_bridge")` + `retry = false` + goto retry. if `!try_module_get(nf_ct_bridge_info.me)`: `err = -EPROTO`. `users_bridge += 1`; if `> 1`: goto unlock. `err = nf_register_net_hooks(net, nf_ct_bridge_info.ops, ops_size)`; if err: `users_bridge = 0`; else `fixup_needed = true`.
  - default: `err = -EPROTO`.
- `out_unlock:` `mutex_unlock`.
- if `fixup_needed`: `nf_ct_iterate_cleanup_net(nf_ct_tcp_fixup, &{net, nfproto})`.
- return `err`.

REQ-11: `nf_ct_netns_do_put(net, nfproto)` (per-AF refcount + unregister):
- `mutex_lock`.
- `NFPROTO_IPV4`: if `users4 ∧ --users4 == 0`: `nf_unregister_net_hooks(ipv4_conntrack_ops)`; `nf_defrag_ipv4_disable(net)`.
- `NFPROTO_IPV6`: symmetric with `users6` + `nf_defrag_ipv6_disable`.
- `NFPROTO_BRIDGE`: if `!nf_ct_bridge_info`: break. if `users_bridge ∧ --users_bridge == 0`: `nf_unregister_net_hooks(bridge.ops)`. `module_put(nf_ct_bridge_info.me)`.
- `mutex_unlock`.

REQ-12: `nf_ct_netns_inet_get(net) -> int` (per-`NFPROTO_INET`):
- `err = nf_ct_netns_do_get(net, NFPROTO_IPV4)`.
- if `CONFIG_IPV6`: if `err < 0`: goto err1. `err = nf_ct_netns_do_get(net, NFPROTO_IPV6)`; if `err < 0`: goto err2.
- `err2: nf_ct_netns_put(net, NFPROTO_IPV4)`; `err1:`; return `err`.

REQ-13: `nf_ct_netns_get(net, nfproto) -> int` (public entry):
- match `nfproto`:
  - `NFPROTO_INET`: `nf_ct_netns_inet_get(net)`.
  - `NFPROTO_BRIDGE`: `nf_ct_netns_do_get(net, NFPROTO_BRIDGE)`; if ok: `nf_ct_netns_inet_get(net)` (chain INET); if INET fails: undo BRIDGE.
  - else: `nf_ct_netns_do_get(net, nfproto)`.

REQ-14: `nf_ct_netns_put(net, nfproto)`:
- `NFPROTO_BRIDGE`: `nf_ct_netns_do_put(net, NFPROTO_BRIDGE)`; fallthrough.
- `NFPROTO_INET`: `nf_ct_netns_do_put(net, NFPROTO_IPV4)`; `nf_ct_netns_do_put(net, NFPROTO_IPV6)`.
- else: `nf_ct_netns_do_put(net, nfproto)`.

REQ-15: `nf_ct_tcp_fixup(ct, _nfproto) -> int` (callback for `nf_ct_iterate_cleanup_net`):
- `nfproto = (unsigned long)_nfproto as u8`.
- if `nf_ct_l3num(ct) != nfproto`: return 0.
- if `nf_ct_protonum(ct) == IPPROTO_TCP ∧ ct.proto.tcp.state == TCP_CONNTRACK_ESTABLISHED`:
  - `ct.proto.tcp.seen[0].td_maxwin = 0`.
  - `ct.proto.tcp.seen[1].td_maxwin = 0`  /* per-newly-enabled conntrack — allow re-learn of window */.
- return 0.

REQ-16: `getorigdst(sk, optval, user, len) -> int` (per-`SO_ORIGINAL_DST` IPv4):
- `inet = inet_sk(sk)`.
- zero `tuple`.
- `lock_sock(sk)`; populate `tuple.src.u3.ip = inet.inet_rcv_saddr`; `tuple.src.u.tcp.port = inet.inet_sport`; `tuple.dst.u3.ip = inet.inet_daddr`; `tuple.dst.u.tcp.port = inet.inet_dport`; `tuple.src.l3num = PF_INET`; `tuple.dst.protonum = sk.sk_protocol`; `release_sock`.
- if `protonum != IPPROTO_TCP ∧ != IPPROTO_SCTP`: return `-ENOPROTOOPT`.
- if `*len < sizeof(sockaddr_in)`: return `-EINVAL`.
- `h = nf_conntrack_find_get(sock_net(sk), &nf_ct_zone_dflt, &tuple)`.
- if `!h`: return `-ENOENT`.
- `ct = nf_ct_tuplehash_to_ctrack(h)`.
- build `sockaddr_in` from `ct.tuplehash[IP_CT_DIR_ORIGINAL].tuple.dst`: `sin_port`, `sin_addr.s_addr`.
- `nf_ct_put(ct)`.
- `copy_to_user(user, &sin, sizeof(sin))` — return 0 or `-EFAULT`.

REQ-17: `ipv6_getorigdst(sk, optval, user, len) -> int` (per-`IP6T_SO_ORIGINAL_DST`):
- `tuple.src.l3num = NFPROTO_IPV6`; populate from `sk->sk_v6_rcv_saddr` / `sk_v6_daddr` / `inet.inet_sport` / `inet.inet_dport`; capture `bound_dev_if = sk.sk_bound_dev_if` and `flow_label = inet6.flow_label` under `lock_sock`.
- protonum check (TCP/SCTP), `*len` check, `nf_conntrack_find_get`.
- build `sockaddr_in6`: `port`, `flowinfo = flow_label & IPV6_FLOWINFO_MASK`, `addr = ct.dst.u3.in6`, `scope_id = ipv6_iface_scope_id(&addr, bound_dev_if)`.
- `copy_to_user`.

REQ-18: `so_getorigdst` / `so_getorigdst6` (`struct nf_sockopt_ops`):
- v4: `pf = PF_INET`, `get_optmin = SO_ORIGINAL_DST` (80), `get_optmax = SO_ORIGINAL_DST + 1`, `.get = getorigdst`.
- v6: `pf = NFPROTO_IPV6`, `get_optmin = IP6T_SO_ORIGINAL_DST`, `get_optmax = + 1`, `.get = ipv6_getorigdst`.

REQ-19: `nf_ct_bridge_register(info)` / `_unregister(info)`:
- WARN on prior occupancy / NULL respectively.
- `mutex_lock(&nf_ct_proto_mutex)`; assign / clear `nf_ct_bridge_info`; unlock.
- /* Bridge integration is plug-in: `nf_conntrack_bridge` module sets this at init time */

REQ-20: `nf_conntrack_proto_init() -> int`:
- `nf_register_sockopt(&so_getorigdst)`.
- if CONFIG_IPV6: `nf_register_sockopt(&so_getorigdst6)` (cleanup v4 on failure).

REQ-21: `nf_conntrack_proto_fini()`:
- `nf_unregister_sockopt(&so_getorigdst)`; if CONFIG_IPV6: `nf_unregister_sockopt(&so_getorigdst6)`.

REQ-22: `nf_conntrack_proto_pernet_init(net)` (called from `nf_conntrack_pernet_init`):
- `nf_conntrack_generic_init_net(net)`.
- `nf_conntrack_udp_init_net(net)`.
- `nf_conntrack_tcp_init_net(net)`.
- `nf_conntrack_icmp_init_net(net)`.
- if CONFIG_IPV6: `nf_conntrack_icmpv6_init_net(net)`.
- if CONFIG_NF_CT_PROTO_SCTP: `nf_conntrack_sctp_init_net(net)`.
- if CONFIG_NF_CT_PROTO_GRE: `nf_conntrack_gre_init_net(net)`.

REQ-23: Module aliases — `ip_conntrack`, `nf_conntrack-2` (AF_INET), `nf_conntrack-10` (AF_INET6); GPL.

REQ-24: `module_param_call(hashsize, nf_conntrack_set_hashsize, param_get_uint, &nf_conntrack_htable_size, 0600)` exposed here (table belongs to core, but the writeable knob is anchored here).

## Acceptance Criteria

- [ ] AC-1: `nf_ct_l4proto_find(IPPROTO_TCP)` returns `&nf_conntrack_l4proto_tcp` and never NULL.
- [ ] AC-2: `nf_ct_l4proto_find(IPPROTO_UDPLITE)` returns generic (UDPLite folds into UDP ops at protonum dispatch).
- [ ] AC-3: First `nf_ct_netns_get(net, NFPROTO_IPV4)` enables defrag-ipv4 and registers 4 IPv4 hooks; second call only bumps `users4`.
- [ ] AC-4: First `nf_ct_netns_get(net, NFPROTO_INET)` enables both IPv4 and IPv6 atomically; IPv6 failure rolls back IPv4.
- [ ] AC-5: Matching `nf_ct_netns_put` decrements; final put unregisters hooks and calls `nf_defrag_ipvX_disable`.
- [ ] AC-6: `NFPROTO_BRIDGE` get with unloaded bridge module triggers `request_module("nf_conntrack_bridge")` and a single retry.
- [ ] AC-7: `nf_confirm` returns `NF_ACCEPT` for VRF post-routing and for packets without a tracked ct.
- [ ] AC-8: `nf_confirm` returns `NF_DROP` and increments `drop` stat when `nf_ct_seq_adjust` fails on a seqadj-marked TCP ct.
- [ ] AC-9: `ipv4_conntrack_local` clears `skb._nfct` for fragmented `IP_NODEFRAG` packets that carry a conntrack template.
- [ ] AC-10: `getorigdst` on a SNATted TCP socket returns the ORIGINAL `dst.ip` + `dst.port`.
- [ ] AC-11: `getorigdst` on a UDP socket returns `-ENOPROTOOPT` (TCP/SCTP only).
- [ ] AC-12: `ipv6_getorigdst` returns `scope_id` derived from `sk_bound_dev_if` for link-local addresses.
- [ ] AC-13: `nf_l4proto_log_invalid` is silent unless `net.ct.sysctl_log_invalid` matches `protonum` or equals `IPPROTO_RAW` (255).
- [ ] AC-14: After IPv4 hooks are first registered, `nf_ct_iterate_cleanup_net(nf_ct_tcp_fixup, ...)` zeroes `td_maxwin` on established TCP cts of that family.
- [ ] AC-15: `nf_ct_bridge_register` warns and refuses if a bridge plugin is already registered.

## Architecture

```
struct L4ProtoOps {
  l4proto: u8,
  allow_clash: bool,
  pkt_to_tuple: fn(*SkBuff, usize, *Net, *mut Tuple) -> bool,
  invert_tuple: fn(*mut Tuple, *const Tuple) -> bool,
  packet:       fn(*mut Conn, *SkBuff, usize, CtInfo, *HookState) -> i32,
  can_early_drop: fn(*const Conn) -> bool,
  nlattr_size:  usize,
  to_nlattr:    Option<fn(...)>,
  from_nlattr:  Option<fn(...)>,
  tuple_to_nlattr: Option<fn(...)>,
  nlattr_to_tuple: Option<fn(...)>,
  nlattr_tuple_size: Option<fn() -> u32>,
  nla_policy:   *const NlaPolicy,
  print_tuple:  Option<fn(*SeqFile, *const Tuple)>,
}

static L4_TCP:      L4ProtoOps = ...;
static L4_UDP:      L4ProtoOps = ...;
static L4_ICMP:     L4ProtoOps = ...;
static L4_ICMPV6:   L4ProtoOps = ...;
static L4_SCTP:     L4ProtoOps = ...;   // cfg(NF_CT_PROTO_SCTP)
static L4_GRE:      L4ProtoOps = ...;   // cfg(NF_CT_PROTO_GRE)
static L4_GENERIC:  L4ProtoOps = ...;
```

`Proto::l4proto_find(l4proto: u8) -> &'static L4ProtoOps`:
1. match `l4proto`:
   - 17 → &L4_UDP.
   - 6  → &L4_TCP.
   - 1  → &L4_ICMP.
   - 132 if cfg → &L4_SCTP.
   - 47  if cfg → &L4_GRE.
   - 58  if cfg(IPV6) → &L4_ICMPV6.
   - _ → &L4_GENERIC.

`Proto::confirm_hook(priv, skb, state) -> Verdict`:
1. `(ct, ctinfo) = nf_ct_get(skb)`.
2. if `ct.is_none() || in_vrf_postrouting(state)`: return Accept.
3. `help = nfct_help(ct)`; `seqadj_needed = ct.status & IPS_SEQ_ADJUST != 0 && !is_loopback(skb)`.
4. if `help.is_none() && !seqadj_needed`: return `nf_conntrack_confirm(skb)`.
5. if `ctinfo == IP_CT_RELATED_REPLY`: return `nf_conntrack_confirm(skb)`.
6. `protoff = match nf_ct_l3num(ct) { IPv4 => skb_network_offset(skb)+ip_hdrlen(skb), IPv6 => ipv6_skip_exthdr(...)?, _ => return nf_conntrack_confirm(skb) }`.
7. if `help.is_some()`: `helper = rcu_deref(help.helper)`; if `helper.is_some()`: `ret = helper.help(skb, protoff, ct, ctinfo)`; if `ret != Accept`: return ret.
8. if `seqadj_needed && !nf_ct_seq_adjust(skb, ct, ctinfo, protoff)`: stat++drop; return Drop.
9. return `nf_conntrack_confirm(skb)`.

`Proto::netns_do_get(net, nfproto: u8) -> Result<(), errno_t>`:
1. `cnet = nf_ct_pernet(net)`.
2. `fixup_needed = false`; `retry = true`.
3. `'retry:` lock `PROTO_MUTEX`.
4. match `nfproto`:
   - IPv4: `cnet.users4 += 1`; if `users4 > 1`: unlock+return Ok.
            `nf_defrag_ipv4_enable(net)?`; if Err: `users4 = 0` unlock return Err.
            `nf_register_net_hooks(net, V4_HOOKS)?`; if Err: `users4 = 0`; else `fixup_needed = true`.
   - IPv6: symmetric.
   - BRIDGE: if `BRIDGE_INFO.is_none()`: if `!retry`: unlock return -EPROTO; else unlock+`request_module("nf_conntrack_bridge")` + retry=false + goto 'retry.
            `try_module_get(BRIDGE_INFO.me)?`. `users_bridge += 1`; if `> 1`: unlock return Ok.
            `nf_register_net_hooks(net, BRIDGE_INFO.ops, BRIDGE_INFO.ops_size)?`; if Err: `users_bridge = 0`; else `fixup_needed = true`.
   - _: unlock return -EPROTO.
5. unlock `PROTO_MUTEX`.
6. if `fixup_needed`: `nf_ct_iterate_cleanup_net(tcp_fixup, &{net, nfproto})`.
7. return Ok / Err.

`Proto::netns_do_put(net, nfproto)`:
1. lock `PROTO_MUTEX`.
2. match nfproto:
   - IPv4: if `cnet.users4 > 0 && --users4 == 0`: `nf_unregister_net_hooks(V4_HOOKS)`; `nf_defrag_ipv4_disable(net)`.
   - IPv6: symmetric.
   - BRIDGE: if `BRIDGE_INFO.is_some()`: if `users_bridge > 0 && --users_bridge == 0`: `nf_unregister_net_hooks(BRIDGE_INFO.ops)`. `module_put(BRIDGE_INFO.me)`.
3. unlock.

`Proto::netns_get(net, nfproto) -> Result<()>`:
1. INET: `netns_inet_get(net)`.
2. BRIDGE: `netns_do_get(net, BRIDGE)?`; `netns_inet_get(net)?` (rollback BRIDGE on failure).
3. _: `netns_do_get(net, nfproto)`.

`Proto::netns_put(net, nfproto)`:
1. BRIDGE: `netns_do_put(BRIDGE)`; fallthrough.
2. INET: `netns_do_put(IPv4)`; `netns_do_put(IPv6)`.
3. _: `netns_do_put(nfproto)`.

`Proto::tcp_fixup(ct, nfproto)`:
1. if `nf_ct_l3num(ct) != nfproto`: 0.
2. if `nf_ct_protonum(ct) == IPPROTO_TCP && tcp.state == TCP_CONNTRACK_ESTABLISHED`:
   - `tcp.seen[0].td_maxwin = 0`.
   - `tcp.seen[1].td_maxwin = 0`.
3. 0.

`Proto::getorigdst_v4(sk, optval, user, len) -> Result<()>`:
1. populate tuple from `inet_sk(sk)` under `lock_sock`/`release_sock`.
2. if `protonum != TCP && != SCTP`: -ENOPROTOOPT.
3. if `*len < size_of::<sockaddr_in>()`: -EINVAL.
4. `h = nf_conntrack_find_get(sock_net(sk), &ZONE_DFLT, &tuple).ok_or(-ENOENT)?`.
5. `ct = tuplehash_to_ctrack(h)`.
6. build sockaddr_in from `ct.tuplehash[ORIGINAL].dst`.
7. `nf_ct_put(ct)`.
8. `copy_to_user(user, &sin)?`.

`Proto::module_init() -> Result<()>`:
1. `nf_register_sockopt(&SO_GETORIGDST)?`.
2. if cfg(IPV6): `nf_register_sockopt(&SO_GETORIGDST_V6)?` (rollback v4 on failure).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `l4proto_find_never_null` | INVARIANT | per-`l4proto_find`: returns `&L4_GENERIC` for unknown protonums; never NULL. |
| `confirm_no_ct_accepts` | INVARIANT | per-`confirm_hook`: `nf_ct_get` returns None ⟹ Accept. |
| `confirm_vrf_short_circuit` | INVARIANT | per-`confirm_hook`: `in_vrf_postrouting` ⟹ Accept (no helper / no seqadj). |
| `confirm_no_helper_no_seqadj` | INVARIANT | per-`confirm_hook`: `help.is_none() && !seqadj_needed` ⟹ direct `nf_conntrack_confirm`. |
| `confirm_ipv6_frag_short_circuit` | INVARIANT | per-`confirm_hook`: IPv6 with frag_off & ~0x7 ⟹ direct confirm (no helper run on fragment). |
| `netns_get_refcount_balanced` | INVARIANT | per-`netns_get`/`_put` pair: net users counter returns to initial value. |
| `netns_inet_get_rollback` | INVARIANT | per-`netns_inet_get`: IPv6 failure rolls back IPv4 get. |
| `bridge_get_module_ref_balanced` | INVARIANT | per-`netns_do_get` BRIDGE: `try_module_get` paired with `module_put` on every path. |
| `tcp_fixup_only_established` | INVARIANT | per-`tcp_fixup`: zeroes maxwin only when state==ESTABLISHED. |
| `getorigdst_protonum_filter` | INVARIANT | per-`getorigdst`: only TCP/SCTP succeed; rest -ENOPROTOOPT. |
| `getorigdst_ct_ref_balanced` | INVARIANT | per-`getorigdst`: `nf_conntrack_find_get` paired with `nf_ct_put` on every exit. |
| `log_invalid_sysctl_gated` | INVARIANT | per-`nf_l4proto_log_invalid`: emits only if `sysctl_log_invalid == protonum ∨ == IPPROTO_RAW`. |
| `proto_mutex_held_for_bridge_registry` | INVARIANT | per-`bridge_register` / `_unregister`: `PROTO_MUTEX` held during slot mutation. |

### Layer 2: TLA+

`net/netfilter/conntrack-proto-netns.tla`:
- Per-net per-AF refcount FSM: states `Down`, `Defragging`, `Up`, `Bridge_LoadingModule`.
- Properties:
  - `safety_v4_hooks_only_when_users4_positive` — per-net: IPv4 hooks registered iff users4 > 0.
  - `safety_v6_hooks_only_when_users6_positive` — per-net: IPv6 hooks registered iff users6 > 0.
  - `safety_defrag_paired_with_hooks` — per-net: defrag-enabled iff users{4,6} > 0.
  - `safety_inet_atomic` — per-`netns_inet_get`: either both v4 and v6 succeed or neither remains incremented.
  - `safety_bridge_module_ref_balanced` — per-net: BRIDGE_INFO module refcount returns to 0 after final put.
  - `liveness_request_module_terminates` — per-`netns_do_get(BRIDGE)`: at most one retry; loops do not unbound.
  - `liveness_proto_mutex_progress` — per-mutex: no thread holds across `request_module` (unlock before request).

`net/netfilter/conntrack-confirm.tla`:
- Per-`nf_confirm` packet flow: states `Enter`, `NoCt`, `VrfBypass`, `NoHelperFastPath`, `RelatedReplyFastPath`, `IPv6FragFastPath`, `HelperRun`, `SeqAdj`, `Confirm`, `Drop`.
- Properties:
  - `safety_no_helper_for_related_reply` — per-`ctinfo==IP_CT_RELATED_REPLY`: never enters HelperRun.
  - `safety_seqadj_drop_increments_stat` — per-seqadj fail: `NF_CT_STAT.drop` ++.
  - `safety_confirm_terminates_with_verdict` — per-flow: exactly one verdict in {Accept, Drop}.
  - `safety_protoff_well_formed_on_helper_run` — per-HelperRun precondition: `protoff` computed from L3 dispatch.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Proto::l4proto_find` post: `result.l4proto == query || result.is_generic()` | `Proto::l4proto_find` |
| `Proto::confirm_hook` post: returns Accept ∨ Drop ∨ helper-defined verdict | `Proto::confirm_hook` |
| `Proto::netns_do_get` post: Ok ⟹ users-counter > 0 ∧ hooks registered | `Proto::netns_do_get` |
| `Proto::netns_do_put` post: dec-to-zero ⟹ hooks unregistered + defrag disabled | `Proto::netns_do_put` |
| `Proto::netns_inet_get` post: Ok ⟹ both v4 and v6 incremented; Err ⟹ neither | `Proto::netns_inet_get` |
| `Proto::getorigdst_v4` post: Ok(0) ⟹ user-buffer holds ORIGINAL.dst sockaddr_in | `Proto::getorigdst_v4` |
| `Proto::getorigdst_v6` post: Ok(0) ⟹ user-buffer holds sockaddr_in6 with computed scope_id | `Proto::getorigdst_v6` |
| `Proto::tcp_fixup` post: ct.l3num != nfproto ⟹ no-op | `Proto::tcp_fixup` |
| `Proto::module_init` post: Ok ⟹ sockopt registered for all enabled families | `Proto::module_init` |
| `Proto::pernet_init` post: per-l4-init called once for every enabled L4 | `Proto::pernet_init` |

### Layer 4: Verus/Creusot functional

`Per-namespace-conntrack-lifecycle → per-AF refcount → hook (un)register → defrag (en)disable → l4proto pernet init` semantic equivalence: per-`Documentation/networking/nf_conntrack-sysctl.rst` and per-RFC-5382 (NAT behavioural req for TCP — observed via fixup of `td_maxwin` on first-enable).

`Per-confirm-hook flow → helper.help(skb, protoff, ct, ctinfo) → nf_ct_seq_adjust → nf_conntrack_confirm` semantic equivalence: per-Documentation/networking/nf_conntrack-sysctl.rst (sysctl_log_invalid, sysctl_acct, sysctl_tstamp) and per-`include/net/netfilter/nf_conntrack_l4proto.h` vtable contract.

## Hardening

(Inherits row-1 features from `net/netfilter/00-overview.md` § Hardening.)

Conntrack-proto reinforcement:

- **Per-`PROTO_MUTEX` strict ordering** — defense against per-bridge-registry race / per-double-load.
- **Per-`request_module` unlocked + bounded retry=1** — defense against per-`PROTO_MUTEX` deadlock during modprobe.
- **Per-AF refcount monotonic + WARN on dec-from-zero** — defense against per-leak / per-double-put unhooking hooks under live traffic.
- **Per-`netns_inet_get` atomic rollback** — defense against per-half-enabled net (v4 up, v6 unregistered) producing asymmetric verdicts.
- **Per-`l4proto_find` total function (always `&L4_GENERIC` fallback)** — defense against per-NULL-deref on unknown protonum.
- **Per-`nf_confirm` `IP_CT_RELATED_REPLY` skip-helper** — defense against per-helper-on-ICMP-error crash.
- **Per-IPv6 frag-fast-path in `nf_confirm`** — defense against per-helper-on-fragment misparse.
- **Per-VRF post-routing bypass** — defense against per-VRF-leaf masquerade collision with L3-master device.
- **Per-`getorigdst` TCP/SCTP-only filter** — defense against per-UDP-misuse leaking ct state.
- **Per-`copy_to_user` length-checked (`sizeof(sockaddr_in)` / `sockaddr_in6`)** — defense against per-userspace stack-write OOB.
- **Per-`ipv4_conntrack_local` template-clear on fragment** — defense against per-stale-template fooling later match.
- **Per-`tcp_fixup` only-on-ESTABLISHED** — defense against per-spurious-window-zero on SYN-SENT.
- **Per-sysctl-gated `nf_l4proto_log_invalid`** — defense against per-log-flood DoS.
- **Per-`nf_ct_bridge_info` single-slot WARN on register-while-occupied** — defense against per-stale-bridge-ops.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `nf_conntrack_core.c` packet ingress (`nf_conntrack_in`, `__nf_conntrack_confirm`) — covered in `nf-conntrack-core.md` Tier-3.
- Per-L4 state machines (`nf_conntrack_proto_tcp.c`, `_udp.c`, `_icmp.c`, `_icmpv6.c`, `_sctp.c`, `_gre.c`, `_generic.c`) — covered separately if expanded.
- `nf_defrag_ipv4.c` / `_ipv6.c` reassembly — covered separately if expanded.
- `nf_conntrack_bridge.c` — covered separately if expanded.
- `nf_conntrack_helper.c` — covered in `conntrack-helper.md` Tier-3.
- `nf_conntrack_seqadj.c` — covered separately if expanded.
- ctnetlink dump/load — covered in `conntrack-netlink.md` Tier-3.
- Implementation code
