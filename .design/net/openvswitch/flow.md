# Tier-3: net/openvswitch/flow.c — OVS flow-key extraction

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/openvswitch/00-overview.md
upstream-paths:
  - net/openvswitch/flow.c (~1116 lines)
  - net/openvswitch/flow.h (struct sw_flow_key, struct sw_flow_mask, struct sw_flow_match)
  - include/uapi/linux/openvswitch.h (OVS_KEY_ATTR_*, OVS_FRAG_TYPE_*, OFPIEH12_*)
-->

## Summary

OVS classification depends on a canonical **`struct sw_flow_key`** built from the per-skb wire frame and per-skb metadata. `flow.c` is the L2/L3/L4 header parser: `ovs_flow_key_extract(tun_info, skb, key)` is the ingress hot-path entry; `ovs_flow_key_extract_userspace(net, attr, skb, key, log)` is the userspace-inject path that hydrates the key from netlink `OVS_KEY_ATTR_*` attributes before calling the same `key_extract` parser; `ovs_flow_key_update(skb, key)` is the post-mutation re-extract used after push/pop or before a CT/RECIRC action when the cached key has been marked `SW_FLOW_KEY_INVALID`. The key is laid out so a single memcmp over a `[start, end)` byte range is the masked-match primitive: per-mask `sw_flow_mask.range` selects which prefix of the key participates in the masked AND-and-compare. Per-skb-stats (`flow.stats[cpu]`) are also maintained here via `ovs_flow_stats_update` / `_get` / `_clear`. Per-fragment policy: IPv4 `OVS_FRAG_TYPE_LATER` from `IP_OFFSET`, `OVS_FRAG_TYPE_FIRST` from `IP_MF` or GSO-UDP, `OVS_FRAG_TYPE_NONE` otherwise; IPv6 derived from `ipv6_find_hdr` and the `IP6_FH_F_FRAG` flag with `frag_off`. Critical for: every-packet classification correctness, flow-cache shape stability across kernel revisions, CT-tuple consistency.

This Tier-3 covers `net/openvswitch/flow.c` (~1116 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct sw_flow_key` | per-skb canonical key | `SwFlowKey` |
| `struct sw_flow_mask` / `sw_flow_key_range` | per-mask wildcard pattern | `SwFlowMask` / `SwFlowKeyRange` |
| `struct sw_flow_match` | per-extract match-set used by userspace | `SwFlowMatch` |
| `ovs_flow_key_extract()` | per-skb canonical extract | `OvsFlow::key_extract` |
| `ovs_flow_key_extract_userspace()` | per-netlink hydrate + extract | `OvsFlow::key_extract_userspace` |
| `ovs_flow_key_update()` | per-mutation re-extract | `OvsFlow::key_update` |
| `ovs_flow_key_update_l3l4()` | per-CT-frag L3/L4 re-extract | `OvsFlow::key_update_l3l4` |
| `key_extract()` | per-frame walker (eth → vlan → ethertype → L3 → L4) | `OvsFlow::key_extract_frame` |
| `key_extract_l3l4()` | per-IPv4/IPv6/ARP/MPLS/NSH branch | `OvsFlow::key_extract_l3l4` |
| `key_extract_mac_proto()` | per-skb.dev->type → MAC_PROTO_{ETHERNET,NONE} | `OvsFlow::key_extract_mac_proto` |
| `parse_vlan()` / `parse_vlan_tag()` | per-tagged-frame walker | `OvsFlow::parse_vlan` |
| `parse_ethertype()` | per-LLC/SNAP unwrap | `OvsFlow::parse_ethertype` |
| `parse_ipv6hdr()` | per-IPv6 header + ext-hdrs | `OvsFlow::parse_ipv6hdr` |
| `get_ipv6_ext_hdrs()` | per-IPv6 extension-header flags (OFPIEH12_*) | `OvsFlow::get_ipv6_ext_hdrs` |
| `parse_icmpv6()` | per-ND-option walker (target, sll, tll) | `OvsFlow::parse_icmpv6` |
| `parse_nsh()` | per-NSH base + context (MD-type1/2) | `OvsFlow::parse_nsh` |
| `check_iphdr()` / `check_header()` | per-pskb-pull guard | `OvsFlow::check_*` |
| `tcphdr_ok()` / `udphdr_ok()` / `sctphdr_ok()` / `icmphdr_ok()` / `icmp6hdr_ok()` / `arphdr_ok()` | per-L4 sanity | `OvsFlow::*_ok` |
| `clear_vlan()` | per-key VLAN clear | `OvsFlow::clear_vlan` |
| `ovs_flow_stats_update()` / `_get()` / `_clear()` | per-flow stats RCU+per-CPU | `SwFlow::stats_*` |
| `ovs_flow_used_time()` | per-jiffies-to-ms | `OvsFlow::used_time` |
| `ovs_ct_fill_key()` | per-ct-state ← nf_conn at extract finish | `OvsCt::fill_key` |

## Compatibility contract

REQ-1: `struct sw_flow_key` layout (must be `__aligned(BITS_PER_LONG/8)` and packed in declaration order so a `[start, end)` range comparison via word-stride memcmp is valid):
- `tun_opts[IP_TUNNEL_OPTS_MAX]`, `tun_opts_len`, `tun_key: ip_tunnel_key` — encapsulating tunnel metadata.
- `phy { priority, skb_mark, in_port } __packed` — ingress metadata.
- `mac_proto: u8` (MAC_PROTO_ETHERNET / MAC_PROTO_NONE) with high-bit `SW_FLOW_KEY_INVALID` cache flag.
- `tun_proto: u8` (AF_INET / AF_INET6) — outer tunnel family.
- `ovs_flow_hash: u32` — populated by OVS_ACTION_ATTR_HASH.
- `recirc_id: u32` — RECIRC chain id.
- `eth { src, dst, vlan: vlan_head, cvlan: vlan_head, type: __be16 }`.
- `ct_state: u8` + `ct_orig_proto: u8` (CT original-direction L4 proto).
- union { `ip { proto, tos, ttl, frag }` } — L3 common.
- `ct_zone: u16`.
- `tp { src, dst, flags: __be16 }` — L4 ports + TCP flags.
- union { `ipv4 { addr.{src,dst}, union { ct_orig.{src,dst} | arp.{sha,tha} } }`, `ipv6 { addr.{src,dst}, label, exthdrs, union { ct_orig.{src,dst} | nd.{target,sll,tll} } }`, `mpls { num_labels_mask, lse[MPLS_LABEL_DEPTH] }`, `nsh: ovs_key_nsh` }.
- `ct { orig_tp.{src,dst}, mark, labels }`.

REQ-2: `SW_FLOW_KEY_INVALID` (high bit of mac_proto): set by `invalidate_flow_key` in actions; checked by `is_flow_key_valid`; cleared by `ovs_flow_key_update` once re-extract succeeds.

REQ-3: `sw_flow_key_range`:
- `start, end: unsigned short int` — half-open byte range within `struct sw_flow_key`.
- `recirc_id` byte offset participates: `SW_FLOW_KEY_RANGE` macro covers `[start, offsetof(key, recirc_id) + sizeof(recirc_id))`.

REQ-4: `sw_flow_mask`:
- `ref_count: int`.
- `rcu: rcu_head`.
- `range: sw_flow_key_range`.
- `key: sw_flow_key` — wildcard pattern: 1-bits = match, 0-bits = wildcard.

REQ-5: `sw_flow_match`:
- `key: *sw_flow_key`.
- `range: sw_flow_key_range`.
- `mask: *sw_flow_mask`.

REQ-6: `sw_flow_key_is_nd(key)` (inline):
- `eth.type == ETH_P_IPV6 ∧ ip.proto == NEXTHDR_ICMP ∧ tp.dst == 0 ∧ (tp.src == NDISC_NEIGHBOUR_SOLICITATION ∨ tp.src == NDISC_NEIGHBOUR_ADVERTISEMENT)`.

REQ-7: `ovs_flow_key_extract(tun_info, skb, key)` (ingress entry):
1. /* Tunnel metadata */
2. if tun_info:
   - key.tun_proto = ip_tunnel_info_af(tun_info).
   - memcpy(&key.tun_key, &tun_info.key, sizeof(key.tun_key)).
   - if tun_info.options_len:
     - BUILD_BUG_ON((1 << (sizeof(tun_info.options_len) * 8)) - 1 > sizeof(key.tun_opts)).
     - ip_tunnel_info_opts_get(TUN_METADATA_OPTS(key, tun_info.options_len), tun_info).
     - key.tun_opts_len = tun_info.options_len.
   - else: key.tun_opts_len = 0.
3. else: key.tun_proto = 0; key.tun_opts_len = 0; memset(&key.tun_key, 0).
4. /* Phy metadata */
5. key.phy.priority = skb.priority.
6. key.phy.in_port = OVS_CB(skb).input_vport.port_no.
7. key.phy.skb_mark = skb.mark.
8. key.ovs_flow_hash = 0.
9. /* MAC proto */
10. res = key_extract_mac_proto(skb); if res < 0: return res; key.mac_proto = res.
11. /* TC SKB-extension hand-off (CONFIG_NET_TC_SKB_EXT) */
12. if tc_skb_ext_tc_enabled():
    - tc_ext = skb_ext_find(skb, TC_SKB_EXT).
    - key.recirc_id = tc_ext ∧ !tc_ext.act_miss ? tc_ext.chain : 0.
    - OVS_CB(skb).mru = tc_ext ? tc_ext.mru : 0.
    - post_ct = tc_ext ? tc_ext.post_ct : false; post_ct_snat/dnat per tc_ext.
    - zone = post_ct ? tc_ext.zone : 0.
13. else: key.recirc_id = 0.
14. /* Frame walk */
15. err = key_extract(skb, key); if !err:
    - ovs_ct_fill_key(skb, key, post_ct).  /* Must be after key_extract. */
    - if post_ct:
      - if !skb_get_nfct(skb): key.ct_zone = zone.
      - else: if !post_ct_dnat: key.ct_state &= ~OVS_CS_F_DST_NAT; if !post_ct_snat: key.ct_state &= ~OVS_CS_F_SRC_NAT.
16. return err.

REQ-8: `key_extract_mac_proto(skb)`:
- switch skb.dev.type:
  - ARPHRD_ETHER → MAC_PROTO_ETHERNET.
  - ARPHRD_NONE → if skb.protocol == ETH_P_TEB: MAC_PROTO_ETHERNET; else MAC_PROTO_NONE.
- default: WARN_ON_ONCE(1); return -EINVAL.

REQ-9: `key_extract(skb, key)` (per-frame walker):
1. key.tp.flags = 0  /* Always cleared even for non-TCP. */
2. skb_reset_mac_header(skb).
3. clear_vlan(key)  /* {tci, tpid} = 0 for both vlan and cvlan. */
4. if ovs_key_mac_proto(key) == MAC_PROTO_NONE:
   - /* L3-only packet (e.g., tunnel decap leaves L3 at skb->data) */
   - if eth_type_vlan(skb.protocol): return -EINVAL.
   - skb_reset_network_header(skb).
   - key.eth.type = skb.protocol.
5. else:  /* Ethernet frame */
   - eth = eth_hdr(skb).
   - ether_addr_copy(key.eth.src, eth.h_source); ether_addr_copy(key.eth.dst, eth.h_dest).
   - __skb_pull(skb, 2 * ETH_ALEN)  /* leaves skb->data at the ethertype/VLAN-TPID slot. */
   - if parse_vlan(skb, key): return -ENOMEM.
   - key.eth.type = parse_ethertype(skb); if key.eth.type == 0: return -ENOMEM.
   - /* Per-multi-tag retain TPID so skb_vlan_pop can later see it */
   - if key.eth.cvlan.tci & VLAN_CFI_MASK: skb.protocol = key.eth.cvlan.tpid.
   - else: skb.protocol = key.eth.type.
   - skb_reset_network_header(skb).
   - __skb_push(skb, skb.data - skb_mac_header(skb))  /* leave skb->data at MAC header. */
6. skb_reset_mac_len(skb).
7. return key_extract_l3l4(skb, key).

REQ-10: `parse_vlan(skb, key)`:
- /* HW-accelerated outer tag */
- if skb_vlan_tag_present(skb):
  - key.eth.vlan.tci = htons(skb.vlan_tci) | htons(VLAN_CFI_MASK).
  - key.eth.vlan.tpid = skb.vlan_proto.
- else:
  - res = parse_vlan_tag(skb, &key.eth.vlan, untag_vlan=true).
  - if res ≤ 0: return res.
- /* Inner C-VLAN: do not untag (consumed later) */
- res = parse_vlan_tag(skb, &key.eth.cvlan, untag_vlan=false).
- if res ≤ 0: return res.
- return 0.

REQ-11: `parse_vlan_tag(skb, key_vh, untag_vlan)`:
- vh = (vlan_head *)skb.data.
- if !eth_type_vlan(vh.tpid): return 0.
- if skb.len < sizeof(vlan_head) + sizeof(__be16): return 0.
- if !pskb_may_pull(skb, sizeof(vlan_head) + sizeof(__be16)): return -ENOMEM.
- key_vh.tci = vh.tci | VLAN_CFI_MASK; key_vh.tpid = vh.tpid.
- if untag_vlan:
  - offset = skb.data - skb_mac_header(skb).
  - __skb_push(skb, offset); __skb_vlan_pop(skb, &tci); __skb_pull(skb, offset).
  - __vlan_hwaccel_put_tag(skb, key_vh.tpid, tci).
- else: __skb_pull(skb, sizeof(vlan_head)).
- return 1.

REQ-12: `parse_ethertype(skb)`:
- proto = *(__be16 *)skb.data; __skb_pull(skb, sizeof(__be16)).
- if eth_proto_is_802_3(proto): return proto.
- /* LLC/SNAP unwrap */
- if skb.len < sizeof(llc_snap_hdr): return ETH_P_802_2.
- if !pskb_may_pull(skb, sizeof(llc_snap_hdr)): return htons(0).
- llc = (llc_snap_hdr *)skb.data.
- if llc.dsap != LLC_SAP_SNAP ∨ llc.ssap != LLC_SAP_SNAP ∨ (llc.oui[0]|llc.oui[1]|llc.oui[2]) != 0: return ETH_P_802_2.
- __skb_pull(skb, sizeof(llc_snap_hdr)).
- if eth_proto_is_802_3(llc.ethertype): return llc.ethertype.
- return htons(ETH_P_802_2).

REQ-13: `key_extract_l3l4(skb, key)` per-ethertype:
- ETH_P_IP branch:
  - check_iphdr; on -EINVAL → memset key.ip+key.ipv4; transport_header = network_header; return 0.
  - nh = ip_hdr(skb); key.ipv4.addr.{src,dst} = saddr/daddr; key.ip.{proto,tos,ttl} = …
  - offset = nh.frag_off & IP_OFFSET.
  - if offset: key.ip.frag = OVS_FRAG_TYPE_LATER; memset(key.tp, 0); return 0.
  - if nh.frag_off & IP_MF ∨ gso_type & SKB_GSO_UDP: key.ip.frag = OVS_FRAG_TYPE_FIRST.
  - else: key.ip.frag = OVS_FRAG_TYPE_NONE.
  - L4 per key.ip.proto: TCP (tcphdr_ok ⟹ src,dst,flags=TCP_FLAGS_BE16), UDP (src,dst), SCTP (src,dst), ICMP (type→tp.src, code→tp.dst); else key.tp memset.
- ETH_P_ARP / ETH_P_RARP:
  - arp_available = arphdr_ok(skb).
  - if arp_available ∧ ar_hrd == ARPHRD_ETHER ∧ ar_pro == ETH_P_IP ∧ ar_hln == ETH_ALEN ∧ ar_pln == 4:
    - key.ip.proto = lower-8-bits of ntohs(ar_op) (else 0).
    - key.ipv4.addr.{src,dst} from ar_sip / ar_tip.
    - key.ipv4.arp.{sha,tha} from ar_sha / ar_tha.
  - else: memset key.ip + key.ipv4.
- eth_p_mpls(key.eth.type):
  - memset key.mpls.
  - skb_set_inner_network_header(skb, skb.mac_len).
  - loop: check_header(mac_len + label_count * MPLS_HLEN); memcpy lse; if label_count ≤ MPLS_LABEL_DEPTH: stash into key.mpls.lse[label_count-1]; advance inner-network-header; if lse & MPLS_LS_S_MASK (bottom-of-stack) break; label_count++.
  - clamp label_count to MPLS_LABEL_DEPTH.
  - key.mpls.num_labels_mask = GENMASK(label_count - 1, 0).
- ETH_P_IPV6:
  - nh_len = parse_ipv6hdr(skb, key).
  - if nh_len < 0: -EINVAL → memset key.ip + key.ipv6.addr; fallthrough; -EPROTO → transport_header=network_header; return 0.
  - if key.ip.frag == OVS_FRAG_TYPE_LATER: memset key.tp; return 0.
  - if gso_type & SKB_GSO_UDP: key.ip.frag = OVS_FRAG_TYPE_FIRST.
  - L4 per key.ip.proto: NEXTHDR_TCP/UDP/SCTP analogous; NEXTHDR_ICMP → parse_icmpv6.
- ETH_P_NSH: parse_nsh.

REQ-14: `parse_ipv6hdr(skb, key)`:
1. check_header(skb, nh_ofs + sizeof(ipv6hdr)).
2. nh = ipv6_hdr(skb).
3. get_ipv6_ext_hdrs(skb, nh, &key.ipv6.exthdrs).
4. key.ip.proto = NEXTHDR_NONE; key.ip.tos = ipv6_get_dsfield(nh); key.ip.ttl = nh.hop_limit.
5. key.ipv6.label = *(__be32 *)nh & IPV6_FLOWINFO_FLOWLABEL.
6. key.ipv6.addr.src = nh.saddr; key.ipv6.addr.dst = nh.daddr.
7. nexthdr = ipv6_find_hdr(skb, &payload_ofs, -1, &frag_off, &flags).
8. if flags & IP6_FH_F_FRAG:
   - if frag_off: key.ip.frag = OVS_FRAG_TYPE_LATER; key.ip.proto = NEXTHDR_FRAGMENT; return 0.
   - else: key.ip.frag = OVS_FRAG_TYPE_FIRST.
9. else: key.ip.frag = OVS_FRAG_TYPE_NONE.
10. if nexthdr < 0: return -EPROTO.
11. nh_len = payload_ofs - nh_ofs.
12. skb_set_transport_header(skb, nh_ofs + nh_len).
13. key.ip.proto = nexthdr.
14. return nh_len.

REQ-15: `get_ipv6_ext_hdrs(skb, nh, *ext_hdrs)`:
- Walk `next_type = nh.nexthdr`; at each ipv6_ext_hdr type set OFPIEH12_{HOP,DEST,ROUTER,FRAG,AUTH,ESP,NONEXT}.
- OFPIEH12_UNREP set when the same ext-hdr appears twice (except DSTOPTS which may legitimately appear twice).
- OFPIEH12_UNSEQ set when ext-hdrs violate RFC 2460 order.
- IPPROTO_NONE → set OFPIEH12_NONEXT and stop.
- IPPROTO_DSTOPTS counted; ≥ 2 ⟹ third sets UNREP.
- skb_header_pointer at running offset; advance by ipv6_optlen(hp).

REQ-16: `parse_icmpv6(skb, key, nh_len)`:
- key.tp.src = htons(icmp6_type); key.tp.dst = htons(icmp6_code).
- if icmp6_code == 0 ∧ icmp6_type ∈ {NDISC_NEIGHBOUR_SOLICITATION, NDISC_NEIGHBOUR_ADVERTISEMENT}:
  - memset(&key.ipv6.nd, 0).
  - if icmp_len < sizeof(nd_msg): return 0.
  - skb_linearize(skb); on fail return -ENOMEM.
  - nd = (nd_msg *)skb_transport_header(skb).
  - key.ipv6.nd.target = nd.target.
  - Walk options (8-byte units):
    - ND_OPT_SOURCE_LL_ADDR len==8 ⟹ key.ipv6.nd.sll (twice ⟹ goto invalid).
    - ND_OPT_TARGET_LL_ADDR len==8 ⟹ key.ipv6.nd.tll (twice ⟹ goto invalid).
- invalid: zero key.ipv6.nd.{target,sll,tll}; return 0.

REQ-17: `parse_nsh(skb, key)`:
- check_header for NSH_BASE_HDR_LEN.
- nh = nsh_hdr(skb); version = nsh_get_ver(nh); length = nsh_hdr_len(nh).
- if version != 0: -EINVAL.
- check_header for `nh_ofs + length`.
- key.nsh.base.{flags,ttl,mdtype,np,path_hdr} from header.
- switch mdtype:
  - NSH_M_TYPE1: length must equal NSH_M_TYPE1_LEN; memcpy 16 bytes of md1 context.
  - NSH_M_TYPE2: zero key.nsh.context (sized to md1).
  - default: -EINVAL.

REQ-18: `ovs_flow_key_extract_userspace(net, attr, skb, key, log)`:
1. parse_flow_nlattrs(attr, a, &attrs, log).
2. ovs_nla_get_flow_metadata(net, a, attrs, key, log)  /* hydrate metadata from netlink. */
3. /* skb.protocol may need to come from the parsed key.eth.type since the caller crafted skb */
4. skb.protocol = key.eth.type.
5. err = key_extract(skb, key); on err return.
6. /* CT-original-direction-tuple cross-check */
7. if attrs & (1 << OVS_KEY_ATTR_CT_ORIG_TUPLE_IPV4) ∧ key.eth.type != ETH_P_IP: return -EINVAL.
8. if attrs & (1 << OVS_KEY_ATTR_CT_ORIG_TUPLE_IPV6) ∧ (key.eth.type != ETH_P_IPV6 ∨ sw_flow_key_is_nd(key)): return -EINVAL.
9. return 0.

REQ-19: `ovs_flow_key_update(skb, key)`:
- res = key_extract(skb, key).
- if !res: key.mac_proto &= ~SW_FLOW_KEY_INVALID.  /* clear cache-invalidation bit */
- return res.

REQ-20: `ovs_flow_key_update_l3l4(skb, key)`:
- /* Used by conntrack frag-handler after frag reassembly: skip L2 re-walk. */
- return key_extract_l3l4(skb, key).

REQ-21: `ovs_flow_stats_update(flow, tcp_flags, skb)` per-CPU RCU stats:
- cpu = smp_processor_id(); len = skb.len + (skb_vlan_tag_present ? VLAN_HLEN : 0).
- stats = rcu_dereference(flow.stats[cpu]).
- if stats:
  - spin_lock(&stats.lock).
  - if cpu == 0 ∧ flow.stats_last_writer != cpu: flow.stats_last_writer = cpu.
- else:
  - stats = rcu_dereference(flow.stats[0])  /* pre-allocated */.
  - spin_lock(&stats.lock).
  - if flow.stats_last_writer != cpu:
    - if flow.stats_last_writer != -1 ∧ !rcu_access_pointer(flow.stats[cpu]):
      - new_stats = kmem_cache_alloc_node(flow_stats_cache, GFP_NOWAIT|__GFP_THISNODE|__GFP_NOWARN|__GFP_NOMEMALLOC, numa_node_id()).
      - if new_stats: init {used=jiffies, packet_count=1, byte_count=len, tcp_flags=tcp_flags, spin_lock_init}; rcu_assign_pointer(flow.stats[cpu], new_stats); cpumask_set_cpu(cpu, flow.cpu_used_mask); goto unlock.
    - flow.stats_last_writer = cpu.
- stats.used = jiffies; stats.packet_count++; stats.byte_count += len; stats.tcp_flags |= tcp_flags.
- unlock: spin_unlock(&stats.lock).

REQ-22: `ovs_flow_stats_get(flow, ovs_stats, used, tcp_flags)`:
- *used = 0; *tcp_flags = 0; memset(ovs_stats, 0).
- for_each_cpu(cpu, flow.cpu_used_mask):
  - stats = rcu_dereference_ovsl(flow.stats[cpu]).
  - if stats:
    - spin_lock_bh(&stats.lock).
    - if !*used ∨ time_after(stats.used, *used): *used = stats.used.
    - *tcp_flags |= stats.tcp_flags.
    - ovs_stats.n_packets += packet_count; ovs_stats.n_bytes += byte_count.
    - spin_unlock_bh(&stats.lock).

REQ-23: `ovs_flow_stats_clear(flow)`:
- Held under ovs_mutex.
- for_each_cpu(cpu, flow.cpu_used_mask):
  - stats = ovsl_dereference(flow.stats[cpu]).
  - if stats: lock-bh; zero used/packet_count/byte_count/tcp_flags; unlock-bh.

REQ-24: `ovs_flow_used_time(flow_jiffies) -> u64`:
- ktime_get_ts64(&cur_ts).
- idle_ms = jiffies_to_msecs(jiffies - flow_jiffies).
- cur_ms = (u64)(u32)cur_ts.tv_sec * MSEC_PER_SEC + cur_ts.tv_nsec / NSEC_PER_MSEC.
- return cur_ms - idle_ms.

REQ-25: Per-IPv4 fragment policy:
- IP_OFFSET non-zero ⟹ key.ip.frag = OVS_FRAG_TYPE_LATER; key.tp memset; do not touch L4.
- IP_MF set or SKB_GSO_UDP ⟹ OVS_FRAG_TYPE_FIRST (L4 ports still extracted).
- else ⟹ OVS_FRAG_TYPE_NONE.

REQ-26: Per-IPv6 fragment policy:
- ipv6_find_hdr called with skip_rh-style flags; sets `flags & IP6_FH_F_FRAG` and `frag_off`.
- frag_off non-zero ⟹ OVS_FRAG_TYPE_LATER; key.ip.proto = NEXTHDR_FRAGMENT; return early.
- frag_off zero with IP6_FH_F_FRAG ⟹ OVS_FRAG_TYPE_FIRST.
- no IP6_FH_F_FRAG ⟹ OVS_FRAG_TYPE_NONE.
- SKB_GSO_UDP later upgrades NONE → FIRST.

REQ-27: Per-`SW_FLOW_KEY_INVALID` cache discipline:
- mac_proto high bit is SW_FLOW_KEY_INVALID.
- `ovs_key_mac_proto(key)` strips it: `mac_proto & ~SW_FLOW_KEY_INVALID`.
- `is_flow_key_valid(key)` ≡ `!(mac_proto & SW_FLOW_KEY_INVALID)`.
- `invalidate_flow_key(key)` ≡ `mac_proto |= SW_FLOW_KEY_INVALID` (from actions.c).
- `ovs_flow_key_update` is the only sanctioned path to clear the bit.

REQ-28: Per-mask compare semantics (`sw_flow_mask` consumer in flow_table.c):
- For lookup: compute masked = key & mask.key over `[mask.range.start, mask.range.end)`; hash; rhashtable_lookup_fast against pre-masked stored keys.
- Per `flow.h` macro SW_FLOW_KEY_RANGE: any non-trivial range must include up through recirc_id.
- Per-mask reference-counted (`ref_count`); freed via call_rcu when last reference dropped.

REQ-29: Per-CT extract finalization (`ovs_ct_fill_key`):
- Always called after `key_extract` returns 0.
- Reads `nf_conn` (if any) attached to skb; populates `key.ct_state`, `key.ct_zone`, `key.ct_mark`, `key.ct.labels`, `key.ct.orig_tp.{src,dst}`, `key.ct_orig_proto`.
- For IPv4: also fills `key.ipv4.ct_orig.{src,dst}`; for IPv6: `key.ipv6.ct_orig.{src,dst}`.
- When skb is post-CT (per tc-skb-ext): override `ct_zone` and possibly mask DST_NAT / SRC_NAT bits in `ct_state`.

REQ-30: Per-NSH MD-type compatibility:
- MD-type 1 (fixed 16-byte context). length must equal NSH_M_TYPE1_LEN.
- MD-type 2 (variable TLV) — context is zeroed in key (TLV walk is in flow_netlink).

## Acceptance Criteria

- [ ] AC-1: IPv4-TCP ingress: key.eth.type == ETH_P_IP; key.ip.proto == IPPROTO_TCP; key.tp.src/dst from header; key.tp.flags == TCP_FLAGS_BE16.
- [ ] AC-2: IPv4 fragment with offset != 0: key.ip.frag == OVS_FRAG_TYPE_LATER; key.tp memset.
- [ ] AC-3: IPv4 first fragment (IP_MF=1, offset=0): key.ip.frag == OVS_FRAG_TYPE_FIRST; L4 ports still present.
- [ ] AC-4: IPv6 with Fragment ext-hdr and frag_off=0: key.ip.frag == OVS_FRAG_TYPE_FIRST.
- [ ] AC-5: IPv6 with Fragment ext-hdr and frag_off!=0: key.ip.frag == OVS_FRAG_TYPE_LATER; key.ip.proto == NEXTHDR_FRAGMENT.
- [ ] AC-6: Double-tagged frame (802.1ad outer + 802.1Q inner): key.eth.vlan + key.eth.cvlan populated; skb.protocol == cvlan.tpid.
- [ ] AC-7: LLC/SNAP frame with non-SNAP DSAP/SSAP: parse_ethertype returns ETH_P_802_2.
- [ ] AC-8: ARP request: key.ip.proto == ARPOP_REQUEST; key.ipv4.arp.sha/tha and ipv4.addr.src/dst populated.
- [ ] AC-9: MPLS stack of 3 labels: key.mpls.lse[0..2] valid; key.mpls.num_labels_mask == GENMASK(2,0).
- [ ] AC-10: ICMPv6 NS with target+SLL options: key.ipv6.nd.target and .sll populated; tll zero.
- [ ] AC-11: ICMPv6 NS with two SLL options: nd fields zeroed (invalid case).
- [ ] AC-12: NSH MD-type 1: key.nsh.base populated; key.nsh.context filled from md1.
- [ ] AC-13: `ovs_flow_key_update` after push_mpls: re-extracts; clears SW_FLOW_KEY_INVALID.
- [ ] AC-14: `ovs_flow_key_extract_userspace` with attrs containing OVS_KEY_ATTR_CT_ORIG_TUPLE_IPV4 but eth.type==ETH_P_IPV6: returns -EINVAL.
- [ ] AC-15: Per-CPU stats: two writers on same CPU coalesce; second CPU allocates per-cpu stats lazily.

## Architecture

```
struct SwFlowKey {
  tun_opts: [u8; IP_TUNNEL_OPTS_MAX],
  tun_opts_len: u8,
  tun_key: IpTunnelKey,
  phy: PhyMeta { priority: u32, skb_mark: u32, in_port: u16 },   // __packed
  mac_proto: u8,                                                  // high bit = SW_FLOW_KEY_INVALID
  tun_proto: u8,
  ovs_flow_hash: u32,
  recirc_id: u32,
  eth: EthMeta { src: [u8;6], dst: [u8;6], vlan: VlanHead, cvlan: VlanHead, type: Be16 },
  ct_state: u8,
  ct_orig_proto: u8,
  ip: IpMeta { proto: u8, tos: u8, ttl: u8, frag: u8 },          // union with future protos
  ct_zone: u16,
  tp: TpMeta { src: Be16, dst: Be16, flags: Be16 },
  l3l4: union {
    ipv4: Ipv4 { addr: { src: Be32, dst: Be32 }, ct_orig | arp },
    ipv6: Ipv6 { addr: { src: In6Addr, dst: In6Addr }, label: Be32, exthdrs: u16, ct_orig | nd },
    mpls: Mpls { num_labels_mask: u32, lse: [Be32; MPLS_LABEL_DEPTH] },
    nsh: OvsKeyNsh,
  },
  ct: CtMeta { orig_tp: { src: Be16, dst: Be16 }, mark: u32, labels: OvsKeyCtLabels },
}  // __aligned(BITS_PER_LONG / 8)

struct SwFlowKeyRange { start: u16, end: u16 }
struct SwFlowMask     { ref_count: i32, rcu: RcuHead, range: SwFlowKeyRange, key: SwFlowKey }
struct SwFlowMatch    { key: *SwFlowKey, range: SwFlowKeyRange, mask: *SwFlowMask }
```

`OvsFlow::key_extract(tun_info, skb, key) -> Result<(), Errno>`:
1. /* Tunnel metadata or zero */
2. if let Some(ti) = tun_info: key.tun_proto = ip_tunnel_info_af(ti); copy tun_key; opts if any.
3. else: zero tun_proto / tun_opts_len / tun_key.
4. /* Phy metadata */
5. key.phy = PhyMeta { priority: skb.priority, skb_mark: skb.mark, in_port: OVS_CB(skb).input_vport.port_no }.
6. key.ovs_flow_hash = 0.
7. /* MAC proto from skb.dev->type */
8. key.mac_proto = OvsFlow::key_extract_mac_proto(skb)?.
9. /* TC-SKB-EXT hand-off */
10. {post_ct, post_ct_snat, post_ct_dnat, zone, recirc_id, mru} = tc_skb_ext_decode(skb).
11. key.recirc_id = recirc_id.
12. /* Walk frame */
13. OvsFlow::key_extract_frame(skb, key)?.
14. ovs_ct_fill_key(skb, key, post_ct).
15. if post_ct: per-zone / SRC_NAT / DST_NAT mask.
16. Ok(()).

`OvsFlow::key_extract_frame(skb, key) -> Result<(), Errno>`:
1. key.tp.flags = 0.
2. skb_reset_mac_header(skb).
3. clear_vlan(key).
4. if ovs_key_mac_proto(key) == MAC_PROTO_NONE: /* L3-only entry */ skb_reset_network_header; key.eth.type = skb.protocol.
5. else:
   - copy h_source/h_dest into key.eth.{src,dst}.
   - __skb_pull(skb, 2 * ETH_ALEN).
   - OvsFlow::parse_vlan(skb, key)?.
   - key.eth.type = OvsFlow::parse_ethertype(skb)?.
   - skb.protocol = if cvlan present { cvlan.tpid } else { eth.type }.
   - skb_reset_network_header; __skb_push back to MAC header.
6. skb_reset_mac_len(skb).
7. OvsFlow::key_extract_l3l4(skb, key).

`OvsFlow::key_extract_l3l4(skb, key) -> Result<(), Errno>`:
1. dispatch on key.eth.type:
   - ETH_P_IP → check_iphdr; copy addr + ip.{proto,tos,ttl}; per-fragment classify; per-L4 parse TCP/UDP/SCTP/ICMP.
   - ETH_P_ARP / ETH_P_RARP → if header valid: opcode → proto, addr + arp.{sha,tha}.
   - eth_p_mpls(eth.type) → loop check_header + read LSE up to MPLS_LABEL_DEPTH; set num_labels_mask.
   - ETH_P_IPV6 → parse_ipv6hdr (sets ip.{proto,tos,ttl,frag} + ipv6.{addr,label,exthdrs}); per-fragment classify; per-L4 parse.
   - ETH_P_NSH → parse_nsh.
2. Ok(()).

`OvsFlow::key_update(skb, key)`:
- res = OvsFlow::key_extract_frame(skb, key).
- if Ok: key.mac_proto &= !SW_FLOW_KEY_INVALID.
- res.

`OvsFlow::key_update_l3l4(skb, key)` ≡ `key_extract_l3l4`.

`OvsFlow::key_extract_userspace(net, attr, skb, key, log)`:
1. parse_flow_nlattrs(attr, a, &attrs, log)?.
2. ovs_nla_get_flow_metadata(net, a, attrs, key, log)?.
3. skb.protocol = key.eth.type.
4. OvsFlow::key_extract_frame(skb, key)?.
5. /* CT-orig-tuple cross-check */
6. if (attrs >> OVS_KEY_ATTR_CT_ORIG_TUPLE_IPV4) & 1 ∧ key.eth.type != ETH_P_IP: -EINVAL.
7. if (attrs >> OVS_KEY_ATTR_CT_ORIG_TUPLE_IPV6) & 1 ∧ (key.eth.type != ETH_P_IPV6 ∨ sw_flow_key_is_nd(key)): -EINVAL.
8. Ok(()).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pskb_pull_before_parse` | INVARIANT | per-check_header / check_iphdr / *_ok: pskb_may_pull succeeds before header deref. |
| `mpls_label_bound` | INVARIANT | per-MPLS loop: writes to key.mpls.lse[i] only when i < MPLS_LABEL_DEPTH. |
| `vlan_tag_consistency` | INVARIANT | per-parse_vlan: outer VLAN sets VLAN_CFI_MASK; inner cvlan preserved without untag. |
| `frag_later_zeros_tp` | INVARIANT | per-IPv4/IPv6 fragment offset != 0: key.tp is memset; no L4 parse. |
| `key_invalid_strict` | INVARIANT | per-ovs_key_mac_proto: high bit stripped; per-is_flow_key_valid agrees. |
| `nd_options_no_overflow` | INVARIANT | per-parse_icmpv6: option walk bounded by icmp_len; opt_len validated. |
| `nsh_md_type1_length_match` | INVARIANT | per-parse_nsh: MD-type 1 only on length == NSH_M_TYPE1_LEN. |
| `tun_opts_within_bounds` | INVARIANT | per-key_extract: BUILD_BUG_ON ensures (1 << (sizeof(tun_info.options_len) * 8)) - 1 ≤ sizeof(key.tun_opts). |
| `flow_stats_per_cpu_locked` | INVARIANT | per-ovs_flow_stats_update: stats.lock held during mutation. |
| `flow_stats_writer_invariant` | INVARIANT | per-stats_last_writer: stays consistent under concurrent writers. |

### Layer 2: TLA+

`net/openvswitch/flow.tla`:
- Per-skb walk = sequence of (eth, vlan*, ethertype, l3, l4); per-error returns at any layer leave key in well-defined state.
- Properties:
  - `safety_key_initialized` — per-key_extract: every field of key reachable from the chosen ethertype branch is initialized or explicitly memset.
  - `safety_fragment_classification_total` — per-IP/IPv6 fragment policy: every frame falls into exactly one of NONE/FIRST/LATER.
  - `safety_invalid_bit_lifecycle` — per-cycle: invalidate_flow_key (in actions) → ovs_flow_key_update → clear.
  - `safety_extension_header_flags_monotonic` — per-OFPIEH12_*: each bit set at most once except UNREP/UNSEQ which monotonically accumulate.
  - `liveness_extract_terminates` — per-ipv6_find_hdr / per-MPLS-stack-walk / per-ND-option-walk: terminate.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `OvsFlow::key_extract` post: ret == Ok ⟹ key.mac_proto stripped of SW_FLOW_KEY_INVALID by caller path | `OvsFlow::key_extract` |
| `OvsFlow::key_extract_frame` post: skb-cursor restored to MAC-header on Ethernet branch | `OvsFlow::key_extract_frame` |
| `OvsFlow::parse_vlan` post: outer.tci & VLAN_CFI_MASK iff tag present | `OvsFlow::parse_vlan` |
| `OvsFlow::parse_ipv6hdr` post: ret == nh_len ≥ sizeof(ipv6hdr) ∨ ret ∈ {-EINVAL, -EPROTO} | `OvsFlow::parse_ipv6hdr` |
| `OvsFlow::parse_icmpv6` post: ND option walk consumes ≤ icmp_len bytes | `OvsFlow::parse_icmpv6` |
| `OvsFlow::parse_nsh` post: mdtype1 ⟹ context bytes 0..15 copied | `OvsFlow::parse_nsh` |
| `OvsFlow::key_update` post: key.mac_proto &= !SW_FLOW_KEY_INVALID iff Ok | `OvsFlow::key_update` |
| `OvsFlow::key_extract_userspace` post: CT-orig-tuple match-mode-eth.type check enforced | `OvsFlow::key_extract_userspace` |
| `OvsFlow::stats_update` post: stats.{packet_count, byte_count, used, tcp_flags} updated under stats.lock | `OvsFlow::stats_update` |

### Layer 4: Verus/Creusot functional

`Per-ingress-skb pipeline: ovs_flow_key_extract → tunnel-info + phy + mac_proto + tc-skb-ext + key_extract (eth → vlan → ethertype → l3 → l4) → ovs_ct_fill_key`. Per `Documentation/networking/openvswitch.rst` and Open vSwitch user-kernel ABI: flow-key bit-for-bit equivalent across IPv4/IPv6/ARP/MPLS/NSH/VLAN/CVLAN matrix; per-fragment classification matches OpenFlow 1.5 semantics; per-CT-orig-tuple constraint matches `OVS_KEY_ATTR_CT_ORIG_TUPLE_*` validation.

## Hardening

(Inherits row-1 features from `net/openvswitch/00-overview.md` § Hardening.)

OVS-flow-extract reinforcement:

- **Per-pskb_may_pull / check_header before every deref** — defense against per-truncated-skb read-overrun.
- **Per-MPLS_LABEL_DEPTH stash cap** — defense against unbounded MPLS-stack write.
- **Per-ND-option opt_len + icmp_len validation** — defense against malformed-ND OOB read.
- **Per-NSH version-zero + MD-type-1-length strict check** — defense against malformed NSH parse.
- **Per-fragment-later L4 memset** — defense against stale-L4 in continuation fragment.
- **Per-`SW_FLOW_KEY_INVALID` re-extract discipline** — defense against post-mutation matching on stale key.
- **Per-CT-orig-tuple eth.type cross-check** — defense against userspace-injected tuple/key-shape mismatch (key-field overlap corruption).
- **Per-OFPIEH12_UNREP / UNSEQ flags surfaced to userspace** — defense against silent IPv6 ext-hdr-ordering exploitation.
- **Per-CPU stats lazy-alloc with __GFP_NOMEMALLOC** — defense against stats-alloc reentering page-allocator in skb path.
- **Per-stats spin_lock_bh on read** — defense against torn-read of u64 counters on 32-bit.
- **Per-flow_stats_cache pre-allocated stats[0]** — defense against per-CPU alloc-fail black-hole.
- **Per-BUILD_BUG_ON tun_opts size** — defense against ABI drift in IP_TUNNEL_OPTS_MAX.
- **Per-WARN_ON_ONCE on unknown skb.dev->type** — defense against silent acceptance of new dev types.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `net/openvswitch/actions.c` (covered in `actions.md` Tier-3)
- `net/openvswitch/datapath.c` (covered in `datapath.md` Tier-3)
- `net/openvswitch/flow_table.c` mask-array hashing (covered in `flow_table.md` Tier-3)
- `net/openvswitch/flow_netlink.c` netlink-attribute serialization / validation (Tier-3 separately)
- `net/openvswitch/conntrack.c` ovs_ct_fill_key body (covered in `conntrack.md` Tier-3)
- `net/openvswitch/vport*.c` per-vport rx/tx (Tier-3 separately)
- `net/core/skbuff.c` skb / pskb / hash primitives (covered in `net/core/skbuff.md`)
- `net/ipv6/exthdrs_core.c` ipv6_find_hdr / ipv6_ext_hdr (covered separately if expanded)
- `net/llc/` LLC/SNAP framing (Tier-3 separately if expanded)
- Implementation code
