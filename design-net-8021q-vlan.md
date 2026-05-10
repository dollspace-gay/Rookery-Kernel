---
title: "Tier-3: net/8021q/vlan.c — IEEE 802.1Q VLAN driver (per-VLAN virtual netdev)"
tags: ["tier-3", "net", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

802.1Q VLAN driver creates a per-VID virtual netdev (e.g. `eth0.10`) over a real netdev. Per-VLAN-dev acts as a normal netdev: per-skb tx prepends 4-byte VLAN tag (TPID + TCI{PCP, DEI, VID}); per-skb rx with matching VID stripped + delivered to per-VLAN dev. Per-real-dev maintains per-VLAN group bitmap + per-VLAN linked list. Per-real-dev with NETIF_F_HW_VLAN_CTAG_FILTER offloads filtering to NIC. Per-VLAN-dev inherits MAC + propagates UC/MC adds to underlying. Critical for: VLAN-trunking on a single physical port + container/VM isolation by VLAN.

This Tier-3 covers `vlan.c` (~770 lines) + `vlan_dev.c` skim.

### Acceptance Criteria

- [ ] AC-1: `ip link add link eth0 name eth0.10 type vlan id 10`: vlan_dev created.
- [ ] AC-2: `ip link set eth0.10 up`: VID 10 registered with real_dev (HW-filter if supported).
- [ ] AC-3: TX on eth0.10: outgoing skb has 4-byte 802.1Q tag; egresses real_dev.
- [ ] AC-4: RX on real_dev with VID 10: delivered to eth0.10.
- [ ] AC-5: ETH_P_8021AD: QinQ outer-tag handled.
- [ ] AC-6: HW-filter: VID 11 (not registered) dropped at NIC.
- [ ] AC-7: vlan_dev_set_mac different from real_dev: dev_uc_add to real_dev.
- [ ] AC-8: VID 4095: rejected.
- [ ] AC-9: `ip link del eth0.10`: vlan_dev unregistered; vlan_group[10] cleared.
- [ ] AC-10: PCP marking via egress_priority_map: skb.priority → 802.1p PCP bits.

### Architecture

Per-VLAN-dev priv:

```
struct VlanDevPriv {
  vlan_id: u16,
  vlan_proto: __be16,                            // 0x8100 / 0x88A8
  flags: u32,                                    // VLAN_FLAG_*
  real_dev: *NetDev,
  dent: *ProcDirEntry,                           // /proc/net/vlan
  ingress_priority_map: [u32; 8],                // PCP → skb.priority
  egress_priority_map: ListHead<VlanPriorityTci>,
  nr_ingress_mappings: u32,
  nr_egress_mappings: u32,
  vlan_pcpu_stats: *PcpuStats,
}
```

Per-real-dev VLAN info:

```
struct VlanInfo {
  real_dev: *NetDev,
  grp: VlanGroup,
  vid_list: ListHead<VlanVidInfo>,
  nr_vids: u32,
}

struct VlanGroup {
  grp_array: [[*NetDev; VLAN_N_VID]; 4],         // per-proto, per-vid
  ...
}
```

`Vlan::register_device(real_dev, vid) -> Result<()>`:
1. snprintf("eth0.10").
2. new_dev = alloc_netdev(priv_size = sizeof(VlanDevPriv), ifname, ..., vlan_setup).
3. priv = vlan_dev_priv(new_dev).
4. priv.real_dev = real_dev.
5. priv.vlan_id = vid.
6. priv.vlan_proto = htons(ETH_P_8021Q).
7. err = Vlan::register_dev(new_dev, NULL).

`Vlan::register_dev(vlan_dev, extack) -> Result<()>`:
1. priv = vlan_dev_priv(vlan_dev).
2. real_dev = priv.real_dev.
3. err = vlan_check_real_dev(real_dev, htons(priv.vlan_proto), priv.vlan_id, extack)?
4. err = register_netdevice(vlan_dev)?
5. vlan_group_set_device(real_dev.vlan_info.grp, htons(priv.vlan_proto), priv.vlan_id, vlan_dev).
6. If real_dev.features & NETIF_F_HW_VLAN_CTAG_FILTER: real_dev.netdev_ops.ndo_vlan_rx_add_vid(real_dev, htons(proto), vid).

`Vlan::unregister_dev(vlan_dev, head)`:
1. priv = vlan_dev_priv(vlan_dev).
2. vlan_group_set_device(real_dev.vlan_info.grp, ..., NULL).
3. If real_dev.features & NETIF_F_HW_VLAN_CTAG_FILTER: ndo_vlan_rx_kill_vid.
4. unregister_netdevice_queue(vlan_dev, head).

`Vlan::xmit(skb, vlan_dev) -> NetdevTxT`:
1. priv = vlan_dev_priv(vlan_dev).
2. tci = (priv.vlan_id & VLAN_VID_MASK) | (skb_priority_to_pcp(skb.priority) << VLAN_PRIO_SHIFT).
3. If real_dev.features & NETIF_F_HW_VLAN_CTAG_TX:
   - __vlan_hwaccel_put_tag(skb, htons(priv.vlan_proto), tci).
4. Else:
   - skb_vlan_push(skb, htons(priv.vlan_proto), tci).
5. skb.dev = real_dev.
6. dev_queue_xmit(skb).

`Vlan::skb_recv(skb, dev, pt, orig_dev) -> i32`:
1. tci = vlan_tx_tag_get_id(skb).
2. vid = tci & VLAN_VID_MASK.
3. vlan_proto = skb.protocol.
4. proto_idx = Vlan::proto_idx(vlan_proto).
5. vlan_dev = real_dev.vlan_info.grp.grp_array[proto_idx][vid].
6. If vlan_dev:
   - skb.dev = vlan_dev.
   - skb.protocol = inner_eth_proto.
   - netif_rx(skb).
7. Else: drop.

### Out of Scope

- net/core/dev (covered in `dev.md` Tier-3)
- net/bridge VLAN-aware (covered separately)
- ethtool ops (covered separately)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct vlan_dev_priv` | per-VLAN-dev priv | `VlanDevPriv` |
| `struct vlan_group` | per-real-dev VID-bitmap | `VlanGroup` |
| `vlan_proto_init()` | per-net VLAN proto-list register | `Vlan::proto_init` |
| `vlan_setup()` | per-vlan-dev setup callback | `Vlan::setup` |
| `register_vlan_dev()` | per-VLAN-dev register | `Vlan::register_dev` |
| `unregister_vlan_dev()` | per-VLAN-dev unregister | `Vlan::unregister_dev` |
| `register_vlan_device()` | per-(real_dev, VID) create | `Vlan::register_device` |
| `vlan_dev_init()` | per-vlan-dev init op | `Vlan::dev_init` |
| `vlan_dev_uninit()` | per-vlan-dev uninit op | `Vlan::dev_uninit` |
| `vlan_dev_set_mac_address()` | per-vlan-dev MAC change | `Vlan::dev_set_mac` |
| `vlan_dev_set_lockdep_one()` | per-vlan-dev lockdep nesting | `Vlan::dev_set_lockdep` |
| `vlan_dev_hard_start_xmit()` | per-vlan-dev TX | `Vlan::xmit` |
| `vlan_dev_set_rx_mode()` | per-vlan-dev rx-mode propagate | `Vlan::set_rx_mode` |
| `vlan_skb_recv()` | per-real-dev RX → per-VLAN-dev | `Vlan::skb_recv` |
| `__vlan_hwaccel_put_tag()` | per-skb HW-offload tag | helper |
| `vlan_proto_idx()` | proto → vlan_group_array index | `Vlan::proto_idx` |
| `VLAN_GROUP_ARRAY_LEN` (4096) | per-VID space | shared |

### compatibility contract

REQ-1: ETH_P_8021Q (0x8100) and ETH_P_8021AD (0x88A8) registration:
- vlan_proto_init: per-net.dev.ptype-list register vlan_packet_type for both.
- Per-skb arriving at real-dev with VLAN-tag: vlan_skb_recv called.

REQ-2: Per-real-dev vlan_group:
- 4096 entries × 4 protos (in vlan_group_array).
- vlan_group[proto_idx][vid] = vlan_dev (or NULL).

REQ-3: Per-VLAN-dev rtnl_link:
- vlan_link_ops registered.
- newlink: register_vlan_device(real_dev, vid).

REQ-4: register_vlan_device(real_dev, vid):
- alloc_netdev with priv_size = sizeof(VlanDevPriv).
- vlan_setup invoked: dev.netdev_ops = vlan_netdev_ops; ethtool_ops; flags.
- vlan.real_dev = real_dev.
- vlan.vlan_id = vid.
- Insert into real_dev.vlan_info.

REQ-5: vlan_dev_hard_start_xmit:
- If NETIF_F_HW_VLAN_CTAG_TX: __vlan_hwaccel_put_tag(skb, htons(vid_proto), tci); skb.dev = real_dev; dev_queue_xmit.
- Else: skb_vlan_push (insert 4-byte tag in eth-header); skb.dev = real_dev; dev_queue_xmit.

REQ-6: vlan_skb_recv (per-real-dev RX hook):
- If skb.protocol == ETH_P_8021Q | ETH_P_8021AD ∧ skb has tag:
  - vid = vlan_tx_tag_get_id(skb).
  - vlan_dev = real_dev.vlan_info.grp[proto_idx][vid].
  - If vlan_dev: skb.dev = vlan_dev; skb.protocol = inner_proto; netif_receive_skb.
  - Else: drop.

REQ-7: Per-vlan-dev MAC:
- Default: inherit real_dev.dev_addr.
- vlan_dev_set_mac: dev_uc_add(real_dev, new_mac).

REQ-8: Per-vlan-dev rx-mode propagation:
- vlan_dev_set_rx_mode: propagate UC + MC list to real_dev.

REQ-9: Per-NIC HW-VLAN filter:
- NETIF_F_HW_VLAN_CTAG_FILTER: vid registered via ndo_vlan_rx_add_vid(real_dev, proto, vid).
- HW filter accepts only registered VIDs.

REQ-10: Per-NIC HW-VLAN-CTAG-TX/RX:
- NETIF_F_HW_VLAN_CTAG_TX: tag inserted by HW.
- NETIF_F_HW_VLAN_CTAG_RX: tag stripped by HW into skb.vlan_tci.

REQ-11: Per-rtnl ioctl SIOCSIFVLAN:
- Legacy: vconfig add/del/set_egress_priority_map.
- Per-rtnl_link: modern.

REQ-12: Per-VID range:
- 0..4094 valid (4095 reserved).
- Per-VLAN-dev unique per-(real_dev, proto, vid).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `vid_in_range` | INVARIANT | per-vlan_dev: vid ∈ [0, 4094]. |
| `vlan_group_unique_per_proto_vid` | INVARIANT | per-(real_dev, proto, vid): at most one vlan_dev. |
| `proto_idx_lt_4` | INVARIANT | proto_idx ∈ {0=8021Q, 1=8021AD, ...}. |
| `vlan_dev_real_dev_consistent` | INVARIANT | priv.real_dev == registered. |
| `hw_filter_register_per_vid` | INVARIANT | per-NETIF_F_HW_VLAN_CTAG_FILTER: ndo_vlan_rx_add_vid called. |

### Layer 2: TLA+

`net/8021q/vlan.tla`:
- Per-VLAN-dev register/unregister + per-skb tx/rx tag handling.
- Properties:
  - `safety_no_cross_vid_leak` — per-skb with VID X delivered only to vlan_dev with VID X.
  - `safety_hw_filter_consistent` — per-VID registered ⟺ HW-filter registered.
  - `liveness_subordinate_inherits_state` — per-vlan_dev MAC change propagates to real_dev UC list.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Vlan::register_dev` post: vlan_group[proto][vid] == vlan_dev; HW-filter registered if available | `Vlan::register_dev` |
| `Vlan::unregister_dev` post: vlan_group entry cleared; HW-filter unregistered | `Vlan::unregister_dev` |
| `Vlan::xmit` post: skb has 802.1Q tag; skb.dev == real_dev | `Vlan::xmit` |
| `Vlan::skb_recv` post: per-VID delivered to matching vlan_dev | `Vlan::skb_recv` |

### Layer 4: Verus/Creusot functional

`Per-vlan_dev TX prepends VID tag → real_dev egress; per-real_dev RX with tag → per-VID vlan_dev → strip tag → up-pass` semantic equivalence: per-IEEE 802.1Q.

### hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

VLAN-specific reinforcement:

- **Per-vid bound to [0, 4094]** — defense against per-config invalid VID.
- **Per-(proto, vid) uniqueness enforced** — defense against per-collision on register.
- **Per-HW-filter add/remove paired with register/unregister** — defense against per-NIC stale filter.
- **Per-skb tag inserted via HW or skb_vlan_push** — defense against per-tag manual insert OOB.
- **Per-MAC propagation to real_dev UC list** — defense against per-mac mismatch.
- **Per-rx vid filtered to registered** — defense against unregistered VID up-pass.
- **Per-PCP map validates priority** — defense against per-prio-map invalid.
- **Per-rtnl_lock during reg/unreg** — defense against per-list mutation race.
- **Per-skb_vlan_push checks headroom** — defense against per-tag push OOM.
- **Per-VLAN-dev NETIF_F_HW_VLAN inheritance** — defense against per-feature mismatch.

