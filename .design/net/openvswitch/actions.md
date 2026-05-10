# Tier-3: net/openvswitch/actions.c — OVS action executor

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/openvswitch/00-overview.md
upstream-paths:
  - net/openvswitch/actions.c (~1598 lines)
  - net/openvswitch/datapath.h (OVS_RECURSION_LIMIT, OVS_DEFERRED_ACTION_THRESHOLD, DEFERRED_ACTION_FIFO_SIZE)
  - include/uapi/linux/openvswitch.h (enum ovs_action_attr)
-->

## Summary

OVS post-classification execution: per-flow `sw_flow_actions.actions` is a netlink-encoded list of `OVS_ACTION_ATTR_*` attributes; `do_execute_actions(dp, skb, key, attr, len)` walks the list with `nla_next` and dispatches each typed attribute to a per-action handler. Per-action consumes-or-mutates `skb` and may update the cached `sw_flow_key` in place; if the packet structure is changed by push/pop, the cached key is marked `SW_FLOW_KEY_INVALID` via `invalidate_flow_key` and the next dependent action triggers a re-extract through `ovs_flow_key_update`. Per-recursive actions (RECIRC, SAMPLE, CLONE, CHECK_PKT_LEN) take a per-CPU `exec_level` counter; per-`OVS_RECURSION_LIMIT = 5` hard bound on packet pipeline recursion; per-`OVS_DEFERRED_ACTION_THRESHOLD = 3` switches deeper-level forks to a deferred FIFO so the call stack does not blow up. Critical for: OVS data-path correctness, NFV chains, conntrack-action and recirc-loop containment, kernel-stack-bound guarantee.

This Tier-3 covers `net/openvswitch/actions.c` (~1598 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `ovs_execute_actions()` | per-skb action-list entry from `ovs_dp_process_packet` | `Actions::execute` |
| `do_execute_actions()` | per-NLA dispatch loop | `Actions::do_execute_actions` |
| `clone_execute()` | per-fork (sample/recirc/clone/check_pkt_len) | `Actions::clone_execute` |
| `clone_key()` | per-CPU flow-key clone for recursion | `Actions::clone_key` |
| `process_deferred_actions()` | per-FIFO drain after stack unwound | `Actions::process_deferred` |
| `add_deferred_actions()` | per-FIFO enqueue | `Actions::add_deferred` |
| `invalidate_flow_key()` / `is_flow_key_valid()` | per-mutation cache-invalidate | `Actions::invalidate_key` |
| `do_output()` | OVS_ACTION_ATTR_OUTPUT | `Actions::do_output` |
| `output_userspace()` | OVS_ACTION_ATTR_USERSPACE | `Actions::output_userspace` |
| `execute_hash()` | OVS_ACTION_ATTR_HASH | `Actions::execute_hash` |
| `push_mpls()` / `pop_mpls()` / `set_mpls()` | MPLS push/pop/set | `Actions::mpls_*` |
| `push_vlan()` / `pop_vlan()` | VLAN push/pop | `Actions::vlan_*` |
| `push_eth()` / `pop_eth()` | OVS_ACTION_ATTR_PUSH_ETH / _POP_ETH | `Actions::eth_*` |
| `push_nsh()` / `pop_nsh()` | OVS_ACTION_ATTR_PUSH_NSH / _POP_NSH | `Actions::nsh_*` |
| `execute_set_action()` | OVS_ACTION_ATTR_SET (tunnel-only) | `Actions::execute_set` |
| `execute_masked_set_action()` | OVS_ACTION_ATTR_SET_MASKED | `Actions::execute_set_masked` |
| `set_ipv4()` / `set_ipv6()` / `set_tcp()` / `set_udp()` / `set_sctp()` / `set_eth_addr()` | per-header masked-set | `Actions::set_*` |
| `set_ip_addr()` / `set_ipv6_addr()` | per-L3 set with checksum fold | `Actions::set_ip_addr*` |
| `set_tp_port()` | per-L4 port set with checksum fold | `Actions::set_tp_port` |
| `execute_recirc()` | OVS_ACTION_ATTR_RECIRC | `Actions::execute_recirc` |
| `sample()` | OVS_ACTION_ATTR_SAMPLE | `Actions::sample` |
| `clone()` | OVS_ACTION_ATTR_CLONE | `Actions::clone` |
| `execute_check_pkt_len()` | OVS_ACTION_ATTR_CHECK_PKT_LEN | `Actions::execute_check_pkt_len` |
| `execute_dec_ttl()` / `dec_ttl_exception_handler()` | OVS_ACTION_ATTR_DEC_TTL | `Actions::execute_dec_ttl` |
| `execute_psample()` | OVS_ACTION_ATTR_PSAMPLE | `Actions::execute_psample` |
| `ovs_fragment()` / `prepare_frag()` / `ovs_vport_output()` | per-egress MTU-fragment path | `Actions::fragment` / `Actions::vport_output` |
| `ovs_ct_execute()` | OVS_ACTION_ATTR_CT (delegates to conntrack.c) | `OvsCt::execute` |
| `ovs_ct_clear()` | OVS_ACTION_ATTR_CT_CLEAR | `OvsCt::clear` |
| `ovs_meter_execute()` | OVS_ACTION_ATTR_METER (delegates to meter.c) | `OvsMeter::execute` |
| `OVS_CB(skb)` | per-skb action-context (mru, cutlen, probability, acts_origlen, upcall_pid, input_vport) | `OvsCb` |
| `struct ovs_pcpu_storage` | per-CPU action-recursion + FIFO + frag scratch | `OvsPcpuStorage` |
| `OVS_RECURSION_LIMIT = 5` | per-pipeline recursion cap | const |
| `OVS_DEFERRED_ACTION_THRESHOLD = 3` | per-defer-switch threshold | const |
| `DEFERRED_ACTION_FIFO_SIZE = 10` | per-CPU FIFO depth | const |
| `MAX_L2_LEN = VLAN_ETH_HLEN + 3 * MPLS_HLEN` | per-frag L2-snapshot bound | const |

## Compatibility contract

REQ-1: `ovs_execute_actions(dp, skb, acts, key)` per-entry:
- /* Per-CPU exec_level bump and guard */
- level = `__this_cpu_inc_return(ovs_pcpu_storage->exec_level)`.
- if level > OVS_RECURSION_LIMIT:
  - net_crit_ratelimited("ovs: recursion limit reached on datapath %s, probable configuration error\n").
  - `ovs_kfree_skb_reason(skb, OVS_DROP_RECURSION_LIMIT)`.
  - err = -ENETDOWN; goto out.
- OVS_CB(skb).acts_origlen = acts.orig_len.
- err = `do_execute_actions(dp, skb, key, acts.actions, acts.actions_len)`.
- if level == 1: `process_deferred_actions(dp)`.
- out: `__this_cpu_dec(ovs_pcpu_storage->exec_level)`.
- return err.

REQ-2: `do_execute_actions(dp, skb, key, attr, len)` per-iteration:
- /* Walk netlink TLV list */
- for (a = attr, rem = len; rem > 0; a = nla_next(a, &rem)):
  - if `trace_ovs_do_execute_action_enabled()`: trace.
  - dispatch on `nla_type(a)`:
    - OVS_ACTION_ATTR_OUTPUT → per-OUTPUT.
    - OVS_ACTION_ATTR_TRUNC → set OVS_CB(skb).cutlen.
    - OVS_ACTION_ATTR_USERSPACE → output_userspace + consume-if-last.
    - OVS_ACTION_ATTR_HASH → execute_hash.
    - OVS_ACTION_ATTR_PUSH_MPLS → push_mpls(mpls_lse, mpls_ethertype, skb->mac_len).
    - OVS_ACTION_ATTR_ADD_MPLS → push_mpls with mac_len conditional on OVS_MPLS_L3_TUNNEL_FLAG_MASK.
    - OVS_ACTION_ATTR_POP_MPLS → pop_mpls(ethertype).
    - OVS_ACTION_ATTR_PUSH_VLAN → push_vlan.
    - OVS_ACTION_ATTR_POP_VLAN → pop_vlan.
    - OVS_ACTION_ATTR_RECIRC → execute_recirc; return-if-last.
    - OVS_ACTION_ATTR_SET → execute_set_action (tunnel only).
    - OVS_ACTION_ATTR_SET_MASKED / _SET_TO_MASKED → execute_masked_set_action.
    - OVS_ACTION_ATTR_SAMPLE → sample(last=nla_is_last); return-if-last.
    - OVS_ACTION_ATTR_CT → ovs_flow_key_update-if-invalid; ovs_ct_execute. Treat -EINPROGRESS as 0 (stolen frag).
    - OVS_ACTION_ATTR_CT_CLEAR → ovs_ct_clear.
    - OVS_ACTION_ATTR_PUSH_ETH → push_eth.
    - OVS_ACTION_ATTR_POP_ETH → pop_eth.
    - OVS_ACTION_ATTR_PUSH_NSH → push_nsh.
    - OVS_ACTION_ATTR_POP_NSH → pop_nsh.
    - OVS_ACTION_ATTR_METER → ovs_meter_execute; drop-on-conform-fail.
    - OVS_ACTION_ATTR_CLONE → clone(last=nla_is_last); return-if-last.
    - OVS_ACTION_ATTR_CHECK_PKT_LEN → execute_check_pkt_len; return-if-last.
    - OVS_ACTION_ATTR_DEC_TTL → execute_dec_ttl; -EHOSTUNREACH → dec_ttl_exception_handler.
    - OVS_ACTION_ATTR_DROP → ovs_kfree_skb_reason(skb, OVS_DROP_EXPLICIT[_WITH_ERROR]); return 0.
    - OVS_ACTION_ATTR_PSAMPLE → execute_psample + consume-if-last.
  - if unlikely(err):
    - `ovs_kfree_skb_reason(skb, OVS_DROP_ACTION_ERROR)`.
    - return err.
- /* End of list reached without consuming skb */
- `ovs_kfree_skb_reason(skb, OVS_DROP_LAST_ACTION)`; return 0.

REQ-3: Per-OVS_ACTION_ATTR_OUTPUT:
- port = nla_get_u32(a).
- /* Last-action optimization: avoid clone */
- if nla_is_last(a, rem): `do_output(dp, skb, port, key)`; return 0.
- else: clone = skb_clone(skb, GFP_ATOMIC); if clone: do_output(dp, clone, port, key); OVS_CB(skb).cutlen = 0.

REQ-4: `do_output(dp, skb, out_port, key)`:
- vport = `ovs_vport_rcu(dp, out_port)`.
- /* Per-output requires running and link-up */
- if !(vport ∧ netif_running(vport.dev) ∧ netif_carrier_ok(vport.dev)):
  - kfree_skb_reason(skb, SKB_DROP_REASON_DEV_READY); return.
- mru = OVS_CB(skb).mru; cutlen = OVS_CB(skb).cutlen.
- /* OVS_ACTION_ATTR_TRUNC apply */
- if cutlen > 0:
  - if skb->len - cutlen > ovs_mac_header_len(key): pskb_trim(skb, skb->len - cutlen).
  - else: pskb_trim(skb, ovs_mac_header_len(key)).
- /* Per-MTU emit-or-fragment */
- if !mru ∨ skb->len ≤ mru + dev->hard_header_len: ovs_vport_send(vport, skb, ovs_key_mac_proto(key)).
- else if mru ≤ vport.dev.mtu: `ovs_fragment(net, vport, skb, mru, key)`.
- else: kfree_skb_reason(skb, SKB_DROP_REASON_PKT_TOO_BIG).

REQ-5: `ovs_fragment(net, vport, skb, mru, key)`:
- /* Per-MPLS network-header rewind */
- if eth_p_mpls(skb->protocol): orig_network_offset = skb_network_offset(skb); skb.network_header = skb.inner_network_header.
- if skb_network_offset(skb) > MAX_L2_LEN: drop OVS_DROP_FRAG_L2_TOO_LONG.
- if key->eth.type == ETH_P_IP:
  - prepare_frag(vport, skb, orig_network_offset, ovs_key_mac_proto(key)).
  - synth rtable ovs_rt with ovs_dst_ops + DST_NOCOUNT; skb_dst_set_noref.
  - IPCB(skb).frag_max_size = mru.
  - `ip_do_fragment(net, skb->sk, skb, ovs_vport_output)`.
- else if key->eth.type == ETH_P_IPV6:
  - synth rt6_info; `ip6_fragment(net, skb->sk, skb, ovs_vport_output)`.
- else: WARN_ONCE("Failed fragment"); drop OVS_DROP_FRAG_INVALID_PROTO.

REQ-6: `prepare_frag(vport, skb, orig_network_offset, mac_proto)`:
- /* Per-CPU snapshot of L2 + VLAN + cb to be replayed on each fragment */
- data = this_cpu_ptr(&ovs_pcpu_storage->frag_data).
- data.dst = skb._skb_refdst; data.vport = vport; data.cb = *OVS_CB(skb).
- data.inner_protocol = skb.inner_protocol; data.network_offset = orig_network_offset.
- if skb_vlan_tag_present(skb): data.vlan_tci = skb_vlan_tag_get(skb) | VLAN_CFI_MASK; else 0.
- data.vlan_proto = skb.vlan_proto; data.mac_proto = mac_proto; data.l2_len = skb_network_offset(skb).
- memcpy(&data.l2_data, skb.data, data.l2_len).
- memset(IPCB(skb), 0, sizeof(struct inet_skb_parm)); skb_pull(skb, hlen).

REQ-7: `ovs_vport_output(net, sk, skb)` (per-fragment trailer):
- /* Restore L2 from per-CPU frag_data */
- data = this_cpu_ptr(&ovs_pcpu_storage->frag_data).
- if skb_cow_head(skb, data.l2_len) < 0: kfree_skb_reason(skb, SKB_DROP_REASON_NOMEM); return -ENOMEM.
- __skb_dst_copy(skb, data.dst); *OVS_CB(skb) = data.cb; skb.inner_protocol = data.inner_protocol.
- /* Restore VLAN-hwaccel state */
- if data.vlan_tci & VLAN_CFI_MASK: __vlan_hwaccel_put_tag(skb, data.vlan_proto, data.vlan_tci & ~VLAN_CFI_MASK).
- else: __vlan_hwaccel_clear_tag(skb).
- skb_push(skb, data.l2_len); memcpy(skb.data, &data.l2_data, data.l2_len).
- skb_postpush_rcsum(skb, skb.data, data.l2_len); skb_reset_mac_header(skb).
- if eth_p_mpls(skb.protocol): skb.inner_network_header = skb.network_header; skb_set_network_header(skb, data.network_offset); skb_reset_mac_len(skb).
- `ovs_vport_send(vport, skb, data.mac_proto)`; return 0.

REQ-8: `output_userspace(dp, skb, key, attr, actions, actions_len, cutlen)`:
- Init `struct dp_upcall_info upcall = { 0 }`; upcall.cmd = OVS_PACKET_CMD_ACTION; upcall.mru = OVS_CB(skb).mru.
- nla_for_each_nested(a, attr): per OVS_USERSPACE_ATTR_*:
  - _USERDATA → upcall.userdata = a.
  - _PID → upcall.portid = OVS_CB(skb).upcall_pid ?? (per-cpu portid if OVS_DP_F_DISPATCH_UPCALL_PER_CPU) ?? nla_get_u32(a).
  - _EGRESS_TUN_PORT → look up vport; dev_fill_metadata_dst; upcall.egress_tun_info = skb_tunnel_info(skb).
  - _ACTIONS → upcall.actions = actions; upcall.actions_len = actions_len.
- return `ovs_dp_upcall(dp, skb, key, &upcall, cutlen)`.

REQ-9: Per-VLAN ops:
- `push_vlan(skb, key, vlan)`:
  - if skb_vlan_tag_present(skb): invalidate_flow_key (already-tagged ⟹ unknown nesting).
  - else: key.eth.vlan.tci = vlan.vlan_tci; key.eth.vlan.tpid = vlan.vlan_tpid.
  - err = skb_vlan_push(skb, vlan.vlan_tpid, ntohs(vlan.vlan_tci) & ~VLAN_CFI_MASK).
  - skb_reset_mac_len(skb).
- `pop_vlan(skb, key)`:
  - err = skb_vlan_pop(skb).
  - if skb_vlan_tag_present(skb): invalidate_flow_key (more tags remain).
  - else: key.eth.vlan.tci = 0; key.eth.vlan.tpid = 0.

REQ-10: Per-MPLS ops:
- `push_mpls(skb, key, mpls_lse, mpls_ethertype, mac_len)`:
  - err = `skb_mpls_push(skb, mpls_lse, mpls_ethertype, mac_len, !!mac_len)`.
  - if !mac_len: key.mac_proto = MAC_PROTO_NONE.
  - invalidate_flow_key(key); return 0.
- `pop_mpls(skb, key, ethertype)`:
  - err = `skb_mpls_pop(skb, ethertype, skb.mac_len, ovs_key_mac_proto(key) == MAC_PROTO_ETHERNET)`.
  - if ethertype == ETH_P_TEB: key.mac_proto = MAC_PROTO_ETHERNET.
  - invalidate_flow_key(key); return 0.
- `set_mpls(skb, flow_key, mpls_lse, mask)`:
  - pskb_may_pull to skb_network_offset + MPLS_HLEN.
  - lse = OVS_MASKED(stack.label_stack_entry, *mpls_lse, *mask).
  - skb_mpls_update_lse(skb, lse).
  - flow_key.mpls.lse[0] = lse.

REQ-11: Per-Ethernet push/pop:
- `push_eth(skb, key, ethh)`:
  - err = skb_eth_push(skb, ethh.addresses.eth_dst, ethh.addresses.eth_src).
  - key.mac_proto = MAC_PROTO_ETHERNET.
  - invalidate_flow_key(key).
- `pop_eth(skb, key)`:
  - err = skb_eth_pop(skb).
  - key.mac_proto = MAC_PROTO_NONE.
  - invalidate_flow_key(key).
- Per-pop_eth comment: "does not support VLAN packets as this action is never called for them" — validator must reject VLAN-before-POP_ETH in flow_netlink.

REQ-12: Per-NSH push/pop:
- `push_nsh(skb, key, a)`:
  - On-stack `u8 buffer[NSH_HDR_MAX_LEN]`; nsh_hdr_from_nlattr.
  - err = nsh_push(skb, nh).
  - key.mac_proto = MAC_PROTO_NONE; invalidate_flow_key(key).
- `pop_nsh(skb, key)`:
  - err = nsh_pop(skb).
  - if skb.protocol == ETH_P_TEB: key.mac_proto = MAC_PROTO_ETHERNET; else MAC_PROTO_NONE.
  - invalidate_flow_key(key).

REQ-13: Per-OVS_ACTION_ATTR_SET (tunnel only):
- if nla_type(a) == OVS_KEY_ATTR_TUNNEL_INFO:
  - skb_dst_drop(skb); dst_hold((dst_entry *)tun.tun_dst); skb_dst_set(skb, tun.tun_dst).
- else: -EINVAL.

REQ-14: `execute_masked_set_action(skb, flow_key, a)`:
- switch nla_type(a):
  - OVS_KEY_ATTR_PRIORITY → OVS_SET_MASKED(skb.priority, …); flow_key.phy.priority = skb.priority.
  - OVS_KEY_ATTR_SKB_MARK → OVS_SET_MASKED(skb.mark, …); flow_key.phy.skb_mark = skb.mark.
  - OVS_KEY_ATTR_TUNNEL_INFO → -EINVAL (masked tunnel-info forbidden).
  - OVS_KEY_ATTR_ETHERNET → set_eth_addr(key, mask).
  - OVS_KEY_ATTR_IPV4 → set_ipv4.
  - OVS_KEY_ATTR_IPV6 → set_ipv6.
  - OVS_KEY_ATTR_TCP → set_tcp.
  - OVS_KEY_ATTR_UDP → set_udp.
  - OVS_KEY_ATTR_SCTP → set_sctp.
  - OVS_KEY_ATTR_MPLS → set_mpls.
  - OVS_KEY_ATTR_CT_STATE / _ZONE / _MARK / _LABELS / _CT_ORIG_TUPLE_* / _NSH → -EINVAL.
- get_mask macro: `((const type)nla_data(a) + 1)` — mask follows masked-value at midpoint of attribute data.

REQ-15: Per-L3/L4 masked-set:
- `set_ipv4`: per-mask non-zero fields rewrite saddr/daddr/tos/ttl; per-write recompute IP checksum via csum_replace4 / csum_replace2; per-L4 protocol (TCP/UDP) checksum-replace via inet_proto_csum_replace4. Per-set clears skb hash and invalidates ct via ovs_ct_clear.
- `set_ipv6`: per-mask non-zero rewrite saddr/daddr/tclass/flowlabel/hop_limit; per-saddr always recalc L4-csum; per-daddr suppress recalc if NEXTHDR_ROUTING present (RFC 8200 dst-address mutation).
- `set_tcp` / `set_udp` / `set_sctp`: rewrite src/dst port; per-TCP/UDP via set_tp_port which inet_proto_csum_replace2 the existing L4 checksum; per-UDP zero-checksum special case (don't bother updating). Per-SCTP recompute full crc32c via sctp_compute_cksum, propagate any prior checksum error via XOR.
- `set_eth_addr`: ether_addr_copy_masked src/dst; rcsum-pull then rcsum-push; flow_key.eth.{src,dst} updated.

REQ-16: Per-OVS_ACTION_ATTR_RECIRC:
- `execute_recirc(dp, skb, key, a, last)`:
  - if !is_flow_key_valid(key): err = ovs_flow_key_update(skb, key); return err on error.
  - BUG_ON(!is_flow_key_valid(key)).
  - recirc_id = nla_get_u32(a).
  - return `clone_execute(dp, skb, key, recirc_id, NULL=actions, 0=len, last, true=clone_flow_key)`.

REQ-17: Per-OVS_ACTION_ATTR_HASH:
- `execute_hash(skb, key, a)`:
  - hash_act = nla_data(a).
  - if hash_alg == OVS_HASH_ALG_L4: hash = skb_get_hash(skb).
  - else if == OVS_HASH_ALG_SYM_L4: hash = __skb_get_hash_symmetric(skb).
  - hash = jhash_1word(hash, hash_basis).
  - if !hash: hash = 0x1.
  - key.ovs_flow_hash = hash.

REQ-18: Per-OVS_ACTION_ATTR_SAMPLE:
- `sample(dp, skb, key, attr, last)`:
  - sample_arg = nla_data(attr); arg = nla_data(sample_arg); actions = nla_next(sample_arg, &rem).
  - init_probability = OVS_CB(skb).probability.
  - if arg.probability != U32_MAX ∧ (!arg.probability ∨ get_random_u32() > arg.probability):
    - if last: ovs_kfree_skb_reason(skb, OVS_DROP_LAST_ACTION).
    - return 0.
  - OVS_CB(skb).probability = arg.probability.
  - clone_flow_key = !arg.exec.
  - err = clone_execute(dp, skb, key, 0, actions, rem, last, clone_flow_key).
  - if !last: OVS_CB(skb).probability = init_probability.

REQ-19: Per-OVS_ACTION_ATTR_CLONE:
- `clone(dp, skb, key, attr, last)`:
  - clone_arg = nla_data(attr); dont_clone_flow_key = nla_get_u32(clone_arg); actions = nla_next(clone_arg, &rem).
  - return clone_execute(dp, skb, key, 0, actions, rem, last, !dont_clone_flow_key).

REQ-20: `clone_execute(dp, skb, key, recirc_id, actions, len, last, clone_flow_key)`:
- /* Per-last clone-or-reuse */
- skb = last ? skb : skb_clone(skb, GFP_ATOMIC); if !skb: return 0 (OOM-skip).
- /* Per-key clone (or reuse if caller declared key won't be mutated) */
- clone = clone_flow_key ? clone_key(key) : key.
- /* In-line fast path */
- if clone:
  - if actions:  /* Sample / Clone / Check_pkt_len */
    - if clone_flow_key: __this_cpu_inc(ovs_pcpu_storage->exec_level).
    - err = do_execute_actions(dp, skb, clone, actions, len).
    - if clone_flow_key: __this_cpu_dec(ovs_pcpu_storage->exec_level).
  - else:  /* Recirc */
    - clone.recirc_id = recirc_id.
    - ovs_dp_process_packet(skb, clone).
  - return err.
- /* Per-out-of-flow-key-slots: enqueue to deferred FIFO */
- da = add_deferred_actions(skb, key, actions, len).
- if da:
  - if !actions:  /* Recirc */ key = &da.pkt_key; key.recirc_id = recirc_id.
- else:
  - ovs_kfree_skb_reason(skb, OVS_DROP_DEFERRED_LIMIT).
  - if net_ratelimit(): pr_warn("deferred action limit reached, drop {sample|recirc} action").
- return 0.

REQ-21: `clone_key(key)` per-CPU stash:
- ovs_pcpu = this_cpu_ptr(ovs_pcpu_storage); keys = &ovs_pcpu.flow_keys; level = ovs_pcpu.exec_level.
- if level ≤ OVS_DEFERRED_ACTION_THRESHOLD (== 3):
  - key = &keys.key[level - 1]; *key = *key_; return key.
- else: return NULL  /* triggers deferral */.

REQ-22: Deferred-action FIFO:
- `struct action_fifo` per-CPU `fifo[DEFERRED_ACTION_FIFO_SIZE = 10]` with head/tail.
- action_fifo_put: if fifo->head ≥ FIFO_SIZE - 1: return NULL (full).
- action_fifo_get: if empty: return NULL; else return &fifo[tail++].
- `add_deferred_actions(skb, key, actions, actions_len)`: da = action_fifo_put; da fields = skb, actions, actions_len, *key.

REQ-23: `process_deferred_actions(dp)`:
- if action_fifo_is_empty(fifo): return.
- do:
  - da = action_fifo_get(fifo); skb = da.skb; key = &da.pkt_key.
  - if da.actions: do_execute_actions(dp, skb, key, da.actions, da.actions_len).
  - else: ovs_dp_process_packet(skb, key).  /* deferred recirc */
- while !action_fifo_is_empty(fifo).
- action_fifo_init(fifo); /* reset head=tail=0 */.

REQ-24: Per-OVS_ACTION_ATTR_CHECK_PKT_LEN:
- `execute_check_pkt_len(dp, skb, key, attr, last)`:
  - cpl_arg = nla_data(attr); arg = nla_data(cpl_arg).
  - len = OVS_CB(skb).mru ? mru + skb.mac_len : skb.len.
  - max_len = arg.pkt_len.
  - if (skb_is_gso(skb) ∧ skb_gso_validate_mac_len(skb, max_len)) ∨ len ≤ max_len:
    - actions = nla_next(cpl_arg, &rem); clone_flow_key = !arg.exec_for_lesser_equal.
  - else:
    - actions = nla_next(nla_next(cpl_arg, &rem), &rem); clone_flow_key = !arg.exec_for_greater.
  - return clone_execute(dp, skb, key, 0, nla_data(actions), nla_len(actions), last, clone_flow_key).

REQ-25: Per-OVS_ACTION_ATTR_DEC_TTL:
- `execute_dec_ttl(skb, key)`:
  - if skb.protocol == ETH_P_IPV6: skb_ensure_writable; if hop_limit ≤ 1 return -EHOSTUNREACH; key.ip.ttl = --nh->hop_limit.
  - else if skb.protocol == ETH_P_IP: skb_ensure_writable; if ttl ≤ 1 return -EHOSTUNREACH; old_ttl = ttl--; csum_replace2(&check, old_ttl<<8, new_ttl<<8); key.ip.ttl = nh->ttl.
- On -EHOSTUNREACH: `dec_ttl_exception_handler(dp, skb, key, a)`:
  - actions = nla_data(a) (first attribute is OVS_DEC_TTL_ATTR_ACTION).
  - if nla_len(actions): return clone_execute(dp, skb, key, 0, nla_data(actions), nla_len(actions), true=last, false=clone_flow_key).
  - else: ovs_kfree_skb_reason(skb, OVS_DROP_IP_TTL); return 0.

REQ-26: Per-OVS_ACTION_ATTR_METER:
- if `ovs_meter_execute(dp, skb, key, nla_get_u32(a))`: ovs_kfree_skb_reason(skb, OVS_DROP_METER); return 0.

REQ-27: Per-OVS_ACTION_ATTR_CT:
- if !is_flow_key_valid(key): err = ovs_flow_key_update(skb, key); return-on-error.
- err = `ovs_ct_execute(ovs_dp_get_net(dp), skb, key, nla_data(a))`.
- if err: return err == -EINPROGRESS ? 0 : err.  /* Stolen IP fragment hidden from userspace */

REQ-28: Per-OVS_ACTION_ATTR_DROP:
- reason = nla_get_u32(a) ? OVS_DROP_EXPLICIT_WITH_ERROR : OVS_DROP_EXPLICIT.
- ovs_kfree_skb_reason(skb, reason); return 0.

REQ-29: Per-OVS_ACTION_ATTR_PSAMPLE (CONFIG_PSAMPLE):
- `execute_psample(dp, skb, attr)`:
  - psample_group_t pg = {}; psample_metadata md = {}.
  - nla_for_each_attr OVS_PSAMPLE_ATTR_GROUP / _COOKIE.
  - md.in_ifindex = OVS_CB(skb).input_vport.dev.ifindex.
  - md.trunc_size = skb.len - OVS_CB(skb).cutlen; md.rate_as_probability = 1.
  - rate = OVS_CB(skb).probability ?? U32_MAX.
  - psample_sample_packet(&pg, skb, rate, &md).
- if !CONFIG_PSAMPLE: no-op stub.

REQ-30: Per-OVS_RECURSION_LIMIT enforcement:
- exec_level is per-CPU counter incremented in ovs_execute_actions and again in clone_execute (only when clone_flow_key is true and actions != NULL).
- Hard cap OVS_RECURSION_LIMIT == 5.
- Per-recirc-loop: clone_execute(actions=NULL) does *not* bump exec_level; instead it re-enters ovs_dp_process_packet which itself calls ovs_execute_actions and bumps. Net effect: each recirc costs +1.
- On overflow: skb dropped OVS_DROP_RECURSION_LIMIT; -ENETDOWN returned; rate-limited crit log.

REQ-31: Per-OVS_DEFERRED_ACTION_THRESHOLD switchover:
- clone_key returns NULL once level > OVS_DEFERRED_ACTION_THRESHOLD (3).
- That triggers add_deferred_actions; the work runs after stack unwind in process_deferred_actions.
- FIFO size = 10 per CPU; overflow ⟹ drop OVS_DROP_DEFERRED_LIMIT, pr_warn rate-limited.

REQ-32: Per-`OVS_CB(skb)` field lifecycle:
- mru: ingress-set; consumed by do_output and check_pkt_len; preserved across recirc.
- cutlen: set by OVS_ACTION_ATTR_TRUNC; consumed by do_output / psample / userspace; cleared after each output/userspace/psample.
- probability: nesting state for sample; saved+restored by sample() across non-last invocations.
- acts_origlen: set at ovs_execute_actions entry from acts.orig_len.
- upcall_pid: ingress-set per-flow-net; used by output_userspace.
- input_vport: ingress-set; used by output_userspace + execute_psample.

REQ-33: Per-error handling per-iteration:
- After dispatch, if unlikely(err): ovs_kfree_skb_reason(skb, OVS_DROP_ACTION_ERROR); return err.
- "Actions that rightfully have to consume the skb should do it and return directly" (USERSPACE-last, OUTPUT-last, RECIRC-last, SAMPLE-last, CLONE-last, CHECK_PKT_LEN-last, DROP, METER-drop, PSAMPLE-last).
- End-of-list fallthrough: ovs_kfree_skb_reason(skb, OVS_DROP_LAST_ACTION).

REQ-34: Per-trace point: `trace_ovs_do_execute_action(dp, skb, key, a, rem)` emitted before each action when tracing is enabled.

## Acceptance Criteria

- [ ] AC-1: Single OUTPUT action: skb is forwarded to vport without clone (last-action optimization).
- [ ] AC-2: Two consecutive OUTPUT actions: first OUTPUT clones the skb, second is the last and consumes the original.
- [ ] AC-3: PUSH_VLAN then matching POP_VLAN: skb returns to original L2; flow_key.eth.vlan.{tci,tpid} = 0.
- [ ] AC-4: PUSH_MPLS then POP_MPLS: skb returns to original ethertype; key invalidated then re-extracted on next dependent action.
- [ ] AC-5: SET_MASKED ipv4 destination + checksum re-folded: nh.check valid, TCP/UDP L4-csum valid.
- [ ] AC-6: RECIRC twice nested: exec_level reaches 3, third packet observation enters deferred-FIFO and runs after unwind.
- [ ] AC-7: RECIRC nested beyond OVS_RECURSION_LIMIT == 5: -ENETDOWN; skb dropped OVS_DROP_RECURSION_LIMIT; per-net crit log emitted.
- [ ] AC-8: SAMPLE with probability = U32_MAX: always executes branch; probability = 0: never executes.
- [ ] AC-9: CHECK_PKT_LEN: len ≤ pkt_len picks LESS_EQUAL branch; len > pkt_len picks GREATER branch.
- [ ] AC-10: CT action with invalid key: ovs_flow_key_update triggered before ovs_ct_execute.
- [ ] AC-11: CT action returning -EINPROGRESS: treated as 0 (stolen frag); subsequent action list aborted.
- [ ] AC-12: METER over-conform: skb dropped OVS_DROP_METER; action list aborted.
- [ ] AC-13: DEC_TTL with ttl ≤ 1: dec_ttl_exception_handler runs nested actions or drops OVS_DROP_IP_TTL.
- [ ] AC-14: DROP: skb dropped with OVS_DROP_EXPLICIT_WITH_ERROR or OVS_DROP_EXPLICIT.
- [ ] AC-15: OUTPUT with mru < skb.len: ovs_fragment + ip_do_fragment / ip6_fragment + ovs_vport_output per-fragment restores L2.
- [ ] AC-16: Deferred-FIFO overflow (>10 entries on a CPU): skb dropped OVS_DROP_DEFERRED_LIMIT; pr_warn rate-limited.
- [ ] AC-17: End-of-list without consuming skb: skb dropped OVS_DROP_LAST_ACTION.

## Architecture

```
struct OvsPcpuStorage {
  exec_level: u32,                              // per-CPU recursion counter
  flow_keys: [SwFlowKey; OVS_DEFERRED_ACTION_THRESHOLD], // == 3
  action_fifos: ActionFifo,
  frag_data: OvsFragData,
}

struct ActionFifo {
  head: u16,
  tail: u16,
  fifo: [DeferredAction; DEFERRED_ACTION_FIFO_SIZE], // == 10
}

struct DeferredAction {
  skb: *SkBuff,
  actions: *NlAttr,       // NULL ⟹ Recirc
  actions_len: u32,
  pkt_key: SwFlowKey,
}

struct OvsCb {
  input_vport: *Vport,
  acts_origlen: u32,
  mru: u16,
  cutlen: u32,
  probability: u32,
  upcall_pid: u32,
}
```

`Actions::execute(dp, skb, acts, key) -> Result<(), Errno>`:
1. level = `per_cpu_inc_return(ovs_pcpu_storage.exec_level)`.
2. if level > OVS_RECURSION_LIMIT (== 5):
   - net_crit_ratelimited("ovs: recursion limit reached on datapath %s, probable configuration error", dp.name).
   - drop(skb, OVS_DROP_RECURSION_LIMIT); err = -ENETDOWN; goto out.
3. OVS_CB(skb).acts_origlen = acts.orig_len.
4. err = `Actions::do_execute_actions(dp, skb, key, acts.actions, acts.actions_len)`.
5. if level == 1: `Actions::process_deferred(dp)`.
6. out: per_cpu_dec(ovs_pcpu_storage.exec_level).
7. return err.

`Actions::do_execute_actions(dp, skb, key, attr, len) -> Result<(), Errno>`:
1. for each nla in attr-list:
   - emit trace.
   - dispatch on nla_type(a) (see REQ-2).
   - if action consumed skb (returned 0 + last) — return.
   - if err: drop(skb, OVS_DROP_ACTION_ERROR); return err.
2. /* End-of-list */ drop(skb, OVS_DROP_LAST_ACTION); return 0.

`Actions::clone_execute(dp, skb, key, recirc_id, actions, len, last, clone_flow_key) -> Result<(), Errno>`:
1. /* Reuse or clone skb */
2. skb = if last { skb } else { skb_clone(skb, GFP_ATOMIC) }; if !skb { return Ok(()) /* OOM-skip */ }.
3. clone = if clone_flow_key { clone_key(key) } else { Some(key) }.
4. if let Some(clone):
   - if actions:
     - if clone_flow_key: per_cpu_inc(exec_level).
     - err = do_execute_actions(dp, skb, clone, actions, len).
     - if clone_flow_key: per_cpu_dec(exec_level).
   - else:  /* Recirc */
     - clone.recirc_id = recirc_id.
     - ovs_dp_process_packet(skb, clone).
   - return err.
5. /* No key slot: defer */
6. da = add_deferred_actions(skb, key, actions, len).
7. if let Some(da):
   - if !actions: { key = &da.pkt_key; key.recirc_id = recirc_id }.
8. else:
   - drop(skb, OVS_DROP_DEFERRED_LIMIT).
   - net_ratelimit ⟹ pr_warn("deferred action limit reached, drop {sample|recirc}").
9. return Ok(()).

`Actions::clone_key(key) -> Option<*SwFlowKey>`:
1. ovs_pcpu = this_cpu_ptr.
2. level = ovs_pcpu.exec_level.
3. if level ≤ OVS_DEFERRED_ACTION_THRESHOLD:
   - slot = &ovs_pcpu.flow_keys[level - 1].
   - *slot = *key.
   - return Some(slot).
4. else: return None.

`Actions::process_deferred(dp)`:
1. if fifo.is_empty(): return.
2. loop:
   - da = fifo.get(); skb = da.skb; key = &da.pkt_key.
   - if da.actions: do_execute_actions(dp, skb, key, da.actions, da.actions_len).
   - else: ovs_dp_process_packet(skb, key).
   - if fifo.is_empty(): break.
3. fifo.init().  /* head = tail = 0 */

`Actions::do_output(dp, skb, port, key)`:
1. vport = ovs_vport_rcu(dp, port).
2. if !(vport ∧ netif_running(vport.dev) ∧ netif_carrier_ok(vport.dev)):
   - kfree_skb_reason(skb, SKB_DROP_REASON_DEV_READY); return.
3. apply cutlen (pskb_trim).
4. if !mru ∨ skb.len ≤ mru + dev.hard_header_len: ovs_vport_send(vport, skb, ovs_key_mac_proto(key)).
5. else if mru ≤ vport.dev.mtu: ovs_fragment(net, vport, skb, mru, key).
6. else: kfree_skb_reason(skb, SKB_DROP_REASON_PKT_TOO_BIG).

`Actions::execute_recirc(dp, skb, key, a, last)`:
1. if !is_flow_key_valid(key): err = ovs_flow_key_update(skb, key); return err.
2. BUG_ON(!is_flow_key_valid(key)).
3. recirc_id = nla_get_u32(a).
4. return clone_execute(dp, skb, key, recirc_id, None, 0, last, true).

`Actions::execute_check_pkt_len(dp, skb, key, attr, last)`:
1. arg = nla_data(nla_data(attr)).
2. len = mru ? mru + mac_len : skb.len.
3. take_branch = (skb_is_gso ∧ skb_gso_validate_mac_len(skb, arg.pkt_len)) ∨ len ≤ arg.pkt_len.
4. actions = if take_branch { ACTIONS_IF_LESS_EQUAL } else { ACTIONS_IF_GREATER }.
5. clone_flow_key = !arg.exec_for_{lesser_equal | greater}.
6. return clone_execute(dp, skb, key, 0, actions_data, actions_len, last, clone_flow_key).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `exec_level_balanced` | INVARIANT | per-ovs_execute_actions: inc → dec balanced even on error paths. |
| `recursion_limit_enforced` | INVARIANT | per-ovs_execute_actions: level > OVS_RECURSION_LIMIT ⟹ -ENETDOWN + drop, no further dispatch. |
| `clone_key_bounded` | INVARIANT | per-clone_key: returns None when level > OVS_DEFERRED_ACTION_THRESHOLD. |
| `fifo_bounded` | INVARIANT | per-action_fifo_put: head ≥ FIFO_SIZE - 1 ⟹ NULL; no out-of-bounds write. |
| `last_consumes_skb` | INVARIANT | per-do_execute_actions: nla_is_last + consuming-action ⟹ no further dereference of skb. |
| `invalidate_after_structural_mutation` | INVARIANT | per-push/pop {VLAN, MPLS, ETH, NSH}: invalidate_flow_key called. |
| `key_reextract_before_ct` | INVARIANT | per-OVS_ACTION_ATTR_CT: !is_flow_key_valid ⟹ ovs_flow_key_update first. |
| `csum_fold_consistent` | INVARIANT | per-set_ipv4/set_ipv6/set_tcp/set_udp: per-field rewrite ⟹ matching csum_replace. |
| `frag_l2_snapshot_bounded` | INVARIANT | per-prepare_frag: l2_len ≤ MAX_L2_LEN (= VLAN_ETH_HLEN + 3 * MPLS_HLEN). |
| `cutlen_one_shot` | INVARIANT | per-OUTPUT / USERSPACE / PSAMPLE: cutlen cleared after consumption. |

### Layer 2: TLA+

`net/openvswitch/actions.tla`:
- Per-action-list = sequence; per-skb consumed-once; per-recirc bounded; per-deferred-FIFO drained.
- Properties:
  - `safety_recursion_bounded` — per-exec_level ≤ OVS_RECURSION_LIMIT always.
  - `safety_skb_consumed_once` — per-skb: kfree / consume / vport_send happens exactly once.
  - `safety_deferred_fifo_bounded` — per-CPU FIFO depth ≤ DEFERRED_ACTION_FIFO_SIZE.
  - `safety_key_invalidated_after_structural_mutation` — per-push/pop ⟹ invalidate followed by re-extract before any structurally-dependent action.
  - `safety_exec_level_balanced` — every per-CPU inc paired with dec.
  - `liveness_eventually_drains_fifo` — per-process_deferred_actions: terminates when fifo empty.
  - `liveness_actions_terminate` — per-do_execute_actions: terminates when rem ≤ 0 or err or consuming-action.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Actions::execute` post: ret ⟹ skb is consumed XOR err == -ENETDOWN-and-dropped | `Actions::execute` |
| `Actions::do_execute_actions` post: rem == 0 after loop ∨ consuming-action returned | `Actions::do_execute_actions` |
| `Actions::clone_execute` post: skb consumed (cloned-and-executed or deferred-or-dropped) | `Actions::clone_execute` |
| `Actions::clone_key` post: result == None iff exec_level > OVS_DEFERRED_ACTION_THRESHOLD | `Actions::clone_key` |
| `Actions::process_deferred` post: fifo.head == fifo.tail == 0 | `Actions::process_deferred` |
| `Actions::do_output` post: skb consumed (sent, fragmented, or dropped) | `Actions::do_output` |
| `Actions::execute_recirc` post: key.recirc_id == recirc_id when clone_key succeeds | `Actions::execute_recirc` |
| `Actions::set_ipv4` post: mask non-zero ∧ value changed ⟹ checksum re-folded | `Actions::set_ipv4` |
| `Actions::set_ipv6` post: per-NEXTHDR_ROUTING ⟹ csum not recomputed for daddr | `Actions::set_ipv6` |

### Layer 4: Verus/Creusot functional

`Per-flow-hit pipeline: ovs_execute_actions → do_execute_actions (NLA-walk) → per-action handler → maybe clone_execute → maybe defer → process_deferred_actions on unwind`. Semantic equivalence to upstream per `Documentation/networking/openvswitch.rst` and `include/uapi/linux/openvswitch.h` action contract. Per-RECIRC pipeline composes with `ovs_dp_process_packet` recursively bounded by OVS_RECURSION_LIMIT.

## Hardening

(Inherits row-1 features from `net/openvswitch/00-overview.md` § Hardening.)

OVS-actions reinforcement:

- **Per-OVS_RECURSION_LIMIT == 5 hard cap** — defense against per-recirc-loop kernel-stack overflow / DoS.
- **Per-OVS_DEFERRED_ACTION_THRESHOLD switch-to-FIFO** — defense against deep-stack recursion when sample/clone/check_pkt_len nest.
- **Per-DEFERRED_ACTION_FIFO_SIZE == 10 + rate-limited pr_warn** — defense against unbounded deferred queue + log flood.
- **Per-OVS_DROP_RECURSION_LIMIT explicit drop-reason** — defense against silent black-hole on misconfig.
- **Per-invalidate_flow_key on every push/pop** — defense against stale-key matching post-encap.
- **Per-key-revalidate (ovs_flow_key_update) before CT/RECIRC** — defense against CT lookup on invalid key.
- **Per-skb_clone(GFP_ATOMIC) OOM-skip semantics** — defense against memory-pressure-induced action skip leaving stale skb.
- **Per-OVS_CB.cutlen one-shot** — defense against double-trim across OUTPUT/USERSPACE/PSAMPLE.
- **Per-vport readiness (running ∧ carrier_ok) before send** — defense against send-to-dead-port.
- **Per-ovs_fragment MAX_L2_LEN bound** — defense against L2-header overflow in per-CPU frag_data.l2_data.
- **Per-METER over-conform drop** — defense against meter-bypass.
- **Per-DROP-with-reason emitted to skb-drop-monitor** — defense against silent-drop blind-spots.
- **Per-pop_eth validator-rejection of VLAN-tagged ingress** — defense against malformed pop_eth on VLAN frame.
- **Per-trace-point gated on `trace_ovs_do_execute_action_enabled`** — defense against trace-induced perf cliff.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `net/openvswitch/conntrack.c` (covered in `conntrack.md` Tier-3)
- `net/openvswitch/datapath.c` (covered in `datapath.md` Tier-3)
- `net/openvswitch/flow.c` (covered in `flow.md` Tier-3)
- `net/openvswitch/flow_table.c` (covered in `flow_table.md` Tier-3)
- `net/openvswitch/flow_netlink.c` action validation / serialization (Tier-3 separately)
- `net/openvswitch/meter.c` ovs_meter_execute body (Tier-3 separately if expanded)
- `net/openvswitch/vport*.c` per-vport-type send/recv (Tier-3 separately)
- `net/core/skbuff.c` skb_mpls_push / skb_vlan_push / skb_eth_push primitives (covered in `net/core/skbuff.md`)
- `net/ipv4/ip_output.c` ip_do_fragment and `net/ipv6/ip6_output.c` ip6_fragment (covered in respective Tier-3)
- `net/psample/psample.c` (covered separately if expanded)
- Implementation code
