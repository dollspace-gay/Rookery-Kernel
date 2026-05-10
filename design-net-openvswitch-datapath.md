---
title: "Tier-3: net/openvswitch/datapath.c — Open vSwitch (OVS) datapath"
tags: ["tier-3", "net", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Open vSwitch (OVS) is a software multi-layer virtual switch with **flow-based forwarding**: per-skb extracts canonical 5-tuple+headers (`sw_flow_key`); `flow_table` lookup matches against installed rules; per-match executes per-action-list (output, set-field, push-vlan, conntrack, drop, recirculate). Per-miss: upcall to userspace (`ovs-vswitchd`) via genetlink; userspace installs new flow rule. Per-vport types: NETDEV (real NIC slave), INTERNAL (kernel netdev), TUNNEL (VXLAN, GRE, GENEVE), GRE_L2, etc. Per-conntrack action integrates Linux NF-conntrack. Critical for: KVM/Linux Bridge alternative; OpenStack Neutron, Kubernetes CNIs, NFV.

This Tier-3 covers `datapath.c` (~2891 lines).

### Acceptance Criteria

- [ ] AC-1: ovs-vsctl add-br br0: OVS_DP_CMD_NEW; datapath created.
- [ ] AC-2: ovs-vsctl add-port br0 eth0: vport NETDEV bound; rx_handler installed.
- [ ] AC-3: Per-skb on eth0: process_packet extracts key; lookups flow.
- [ ] AC-4: Flow miss: upcall to userspace.
- [ ] AC-5: ovs-ofctl add-flow: per-rule installed; subsequent matches hit.
- [ ] AC-6: OVS_ACTION_ATTR_OUTPUT to vport: skb forwarded.
- [ ] AC-7: OVS_ACTION_ATTR_PUSH_VLAN: skb has VLAN tag added.
- [ ] AC-8: OVS_ACTION_ATTR_CT: conntrack state populated.
- [ ] AC-9: OVS_ACTION_ATTR_RECIRC: per-recirc_id second-pass.
- [ ] AC-10: tunnel vport (VXLAN): per-skb decap on RX; encap on TX.
- [ ] AC-11: ovs-dpctl dump-flows: per-flow stats.
- [ ] AC-12: Per-namespace: nsA / nsB OVS instances independent.

### Architecture

Per-DP state:

```
struct Datapath {
  list_node: ListLink,                            // ovs_net.dps
  table: FlowTable,
  ports: [HListHead<Vport>; DP_VPORT_HASH_BUCKETS],
  stats_percpu: PerCpu<DpStats>,
  net: *Net,
  user_features: u32,
  max_headroom: u32,
}

struct Vport {
  dev: *NetDev,
  dp: *Datapath,
  port_no: u16,
  type_: VportType,                               // NETDEV / INTERNAL / TUNNEL / ...
  upcall_portids: [u32; UPCALL_NTHRDS],
  hash_node: HListLink,
  ...
}

struct SwFlow {
  rcu: RcuHead,
  flow_node: RhashNode,                           // flow_table
  ufid_node: RhashNode,                           // ufid table
  hash: u32,
  key: SwFlowKey,
  mask: *SwFlowMask,
  sf_acts: *SwFlowActions,
  stats: PerCpu<SwFlowStats>,
}

struct SwFlowKey {
  tun_proto: u8,
  tun_opts: TunOpts,
  tun_key: TunInfo,
  phy: { priority, in_port, skb_mark, vlan_tci, ... },
  eth: EthernetKey,
  ip: IpKey,
  ipv4: IpV4Key,
  ipv6: IpV6Key,
  tp: { src, dst, flags },
  ct_state: CtState,
  ct_zone: u16,
  ct_mark: u32,
  ct_label: CtLabels,
  recirc_id: u32,
}

struct FlowTable {
  ti: *TableInstance,                             // rhashtable
  ufid_ti: *TableInstance,
  mask_cache_index: u32,
  mask_cache: PerCpu<MaskCache>,
  mask_array: *MaskArray,
  ...
}
```

`OvsDp::new(net, parms) -> Result<*Datapath>`:
1. dp = kzalloc(sizeof(Datapath)).
2. ovs_flow_tbl_init(&dp.table).
3. for_each_dp_port: hash_init.
4. /* internal vport for DP */
5. internal_vport = new_vport(&internal_parms).
6. list_add(&dp.list_node, &ovs_net(net).dps).

`OvsDp::process_packet(skb, key)`:
1. dp = ovs_dp(skb.dev).
2. /* Per-flow lookup */
3. flow = FlowTable::lookup_stats(&dp.table, key, skb_get_hash(skb), &n_mask_hit, &cache_hit).
4. if !flow:
   - upcall_info = { cmd: OVS_PACKET_CMD_MISS, ... }.
   - err = OvsDp::upcall(dp, skb, key, &upcall_info, 0).
5. else:
   - Actions::do_execute_actions(dp, skb, key, flow.sf_acts).

`FlowTable::lookup_stats(tbl, key, skb_hash, n_mask_hit, cache_hit) -> *SwFlow`:
1. /* Per-mask-cache fast-path */
2. ti = rcu_dereference(tbl.ti).
3. for mask_idx in 0..mask_array.count:
   - mask = mask_array.masks[mask_idx].
   - masked_key = key_masked(key, mask).
   - flow = rhashtable_lookup(ti, &masked_key, ovs_rht_params).
   - if flow: *n_mask_hit += 1; return flow.
4. Return NULL.

`Actions::do_execute_actions(dp, skb, key, sf_acts)`:
1. nla_for_each_attr(a, sf_acts.actions, sf_acts.actions_len, ...):
   - switch nla_type(a):
     - OVS_ACTION_ATTR_OUTPUT: do_output(dp, skb, port_no).
     - OVS_ACTION_ATTR_SET: execute_set_action(skb, key, a).
     - OVS_ACTION_ATTR_PUSH_VLAN: push_vlan(skb, key, ...).
     - OVS_ACTION_ATTR_POP_VLAN: pop_vlan(skb).
     - OVS_ACTION_ATTR_CT: ovs_ct_execute(net, skb, key, ...).
     - OVS_ACTION_ATTR_RECIRC: recirc; goto top of pipeline with new recirc_id.
     - OVS_ACTION_ATTR_HASH: compute hash + set in skb.
     - OVS_ACTION_ATTR_DROP: kfree_skb; return.
     - OVS_ACTION_ATTR_USERSPACE: ovs_dp_upcall(dp, skb, key, ...).

`OvsDp::upcall(dp, skb, key, upcall_info, cutlen) -> i32`:
1. Build OVS_PACKET_CMD_MISS genl msg.
2. Embed key + skb data + upcall_info attrs.
3. genlmsg_unicast(net, skb_msg, upcall_portid).

### Out of Scope

- net/openvswitch/{flow, flow_table, actions, conntrack, vport, vport-*}.c (covered separately if expanded)
- ovs-vswitchd userspace (out-of-tree)
- OVSDB (out-of-tree)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct datapath` | per-bridge state | `Datapath` |
| `struct vport` | per-port (slave + virt iface) | `Vport` |
| `struct sw_flow` | per-flow rule | `SwFlow` |
| `struct sw_flow_key` | per-skb canonical key | `SwFlowKey` |
| `struct flow_table` | per-DP flow-table | `FlowTable` |
| `ovs_dp_process_packet()` | per-skb pipeline entry | `OvsDp::process_packet` |
| `ovs_dp_upcall()` | per-flow-miss userspace upcall | `OvsDp::upcall` |
| `ovs_flow_tbl_lookup_stats()` | per-skb table-lookup | `FlowTable::lookup_stats` |
| `new_vport()` | per-vport create | `Vport::new` |
| `ovs_dp_cmd_*` | per-genl userspace ABI | `OvsDp::cmd_*` |
| `do_execute_actions()` | per-action-list executor | `Actions::do_execute_actions` |
| `ovs_ct_execute()` | conntrack action | `OvsCt::execute` |
| `OVS_PACKET_CMD_*` | per-userspace genl-cmd | UAPI |
| `OVS_DP_CMD_*` / `OVS_VPORT_CMD_*` / `OVS_FLOW_CMD_*` | per-userspace cmd | UAPI |
| `ovs_dp_genl_family` | per-net genetlink family | `OvsDp::genl_family` |

### compatibility contract

REQ-1: Per-DP creation:
- userspace via OVS_DP_CMD_NEW genl: kernel allocates Datapath + creates internal vport.
- Per-DP has linked list of vports + flow_table.

REQ-2: Per-vport types:
- OVS_VPORT_TYPE_NETDEV: bind to existing NIC.
- OVS_VPORT_TYPE_INTERNAL: in-kernel virt netdev.
- OVS_VPORT_TYPE_GENEVE / VXLAN / GRE / GRE_L2: tunnel.
- Per-vport has rx_handler intercepted.

REQ-3: ovs_dp_process_packet (per-skb pipeline):
- Per-skb extract sw_flow_key from L2/L3/L4 headers.
- flow = ovs_flow_tbl_lookup_stats(&dp.table, &key, skb_hash, &n_mask_hit).
- if !flow: ovs_dp_upcall(dp, skb, &key, &upcall_info, 0).
- else: do_execute_actions(dp, skb, key, flow.sf_acts).

REQ-4: do_execute_actions:
- Per-NLA-encoded action list:
  - OVS_ACTION_ATTR_OUTPUT: forward to vport.
  - OVS_ACTION_ATTR_SET: per-field set.
  - OVS_ACTION_ATTR_PUSH_VLAN / POP_VLAN: VLAN ops.
  - OVS_ACTION_ATTR_PUSH_MPLS / POP_MPLS: MPLS.
  - OVS_ACTION_ATTR_CT: conntrack.
  - OVS_ACTION_ATTR_RECIRC: re-run pipeline with new key.
  - OVS_ACTION_ATTR_HASH: compute hash.
  - OVS_ACTION_ATTR_DROP.
  - OVS_ACTION_ATTR_USERSPACE: upcall.

REQ-5: Per-flow-table:
- mask_array: per-mask-pattern bitmask.
- per-mask: rhashtable of flows.
- Per-lookup: walk masks; per-mask compute masked-key + rhashtable lookup.

REQ-6: Per-key extract:
- per-skb headers walked: ETH+VLAN+CVLAN+IP/IPv6+TCP/UDP/ICMP+inner-tunnel.
- Per-tunnel-info from skb.tunnel_info.
- Per-conntrack-state from nf_conn.

REQ-7: Per-userspace genetlink:
- OVS_DATAPATH_FAMILY: ops for DP/VPORT/FLOW/PACKET.
- OVS_PACKET_CMD_EXECUTE: userspace inject.
- OVS_PACKET_CMD_MISS: kernel-to-userspace upcall.
- OVS_PACKET_CMD_ACTION: userspace-to-kernel action exec.

REQ-8: Per-upcall:
- skb cloned + reduced; ovs_packet_cmd_miss; queue to userspace genl-receive socket.
- userspace ovs-vswitchd processes; installs flow.

REQ-9: Per-recirculation:
- OVS_ACTION_ATTR_RECIRC: re-process packet through pipeline with recirc_id != 0.
- Useful for tunnel-decap then re-classify.

REQ-10: Per-conntrack integration:
- OVS_ACTION_ATTR_CT: apply nf_conntrack lookup; sets ct_state in skb metadata.
- Per-DNAT/SNAT support.

REQ-11: Per-namespace:
- Per-net OVS instance.
- Per-vport bound to per-net netdev.

REQ-12: Per-userspace ABI:
- /usr/sbin/ovs-vswitchd binds to genl family.
- OVSDB stores config; ovs-vswitchd installs flows.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `vport_unique_per_dp` | INVARIANT | per-dp.ports: vport.port_no unique. |
| `flow_in_table_iff_in_rhashtable` | INVARIANT | per-flow on table.ti ⟺ inserted via rhashtable. |
| `actions_within_attr_len` | INVARIANT | per-action: nla_len(a) ≤ remaining-actions-len. |
| `recirc_depth_bound` | INVARIANT | per-RECIRC: depth ≤ MAX_RECIRC (e.g. 4). |
| `key_extracted_per_packet` | INVARIANT | per-process_packet: key populated before lookup. |

### Layer 2: TLA+

`net/openvswitch/datapath.tla`:
- Per-skb pipeline: extract → lookup → action / upcall.
- Properties:
  - `safety_no_action_after_drop` — per-OVS_ACTION_ATTR_DROP: no further action.
  - `safety_recirc_eventually_terminates` — per-recirc: depth bounded.
  - `liveness_miss_eventually_handled` — per-flow-miss + userspace responsive: flow installed.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `OvsDp::process_packet` post: key extracted; flow OR upcall path | `OvsDp::process_packet` |
| `FlowTable::lookup_stats` post: returned-flow matches mask × key | `FlowTable::lookup_stats` |
| `Actions::do_execute_actions` post: per-action dispatched per nla_type | `Actions::do_execute_actions` |
| `OvsDp::upcall` post: genl msg unicast to upcall_portid | `OvsDp::upcall` |

### Layer 4: Verus/Creusot functional

`Per-skb OVS pipeline: extract key → flow_table lookup → match-then-action OR miss-then-upcall` semantic equivalence: per-Open vSwitch architecture (RFC 7047 OVSDB + OVS internals).

### hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

OVS-specific reinforcement:

- **Per-flow-add CAP_NET_ADMIN** — defense against per-userspace flow-spoof.
- **Per-RECIRC depth bounded** — defense against per-recirc infinite-loop DoS.
- **Per-flow_table mask_array RCU** — defense against per-walk UAF.
- **Per-vport rx_handler atomic-register/unregister** — defense against per-vport race.
- **Per-upcall_portid per-thread** — defense against per-userspace single-thread bottleneck.
- **Per-action-attr length validated** — defense against per-malformed-nla OOB.
- **Per-conntrack-zone scoped per-DP** — defense against cross-DP CT-conflict.
- **Per-genl OVS_DP_CMD_NEW CAP** — defense against per-unprivileged DP-create.
- **Per-namespace OVS instance** — defense against cross-ns leak.
- **Per-DP destroy waits-for-RCU** — defense against per-skb-in-flight UAF.
- **Per-upcall skb cutlen** — defense against per-userspace OOM on huge skb.

