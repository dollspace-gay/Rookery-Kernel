# Tier-3: net/8021q/vlan_dev.c — VLAN-dev TX/RX/IOCTL ndo callbacks

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/8021q/vlan.md
upstream-paths:
  - net/8021q/vlan_dev.c (~1087 lines)
  - net/8021q/vlan.h
  - include/linux/if_vlan.h
-->

## Summary

`vlan_dev.c` implements the netdev_ops + ethtool_ops for per-VLAN-dev (e.g. `eth0.10`). Per-skb tx prepends 4-byte 802.1Q tag (TPID + TCI{PCP, DEI, VID}); per-tx-stat counts on per-vlan-dev. Per-skb rx — when called via `vlan_skb_recv` — strips tag, sets skb.dev = vlan_dev, dispatches to upper-stack. Per-MTU smaller than real_dev MTU due to tag overhead. Per-set_rx_mode propagates UC/MC to real_dev. Per-ioctl SIOCSHWTSTAMP/SIOCGHWTSTAMP forwarded. Per-ethtool: features inherited from real_dev. Critical for: per-VLAN-dev as a first-class netdev with full IP-stack integration.

This Tier-3 covers `vlan_dev.c` (~1087 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `vlan_netdev_ops` | per-vlan_dev netdev_ops | `Vlan::NETDEV_OPS` |
| `vlan_ethtool_ops` | per-vlan_dev ethtool_ops | `Vlan::ETHTOOL_OPS` |
| `vlan_dev_init()` | per-init op | `Vlan::dev_init` |
| `vlan_dev_uninit()` | per-uninit op | `Vlan::dev_uninit` |
| `vlan_dev_hard_start_xmit()` | per-tx op | `Vlan::xmit` |
| `vlan_dev_change_mtu()` | per-MTU change | `Vlan::change_mtu` |
| `vlan_dev_set_mac_address()` | per-mac change | `Vlan::set_mac` |
| `vlan_dev_set_rx_mode()` | per-rx-mode | `Vlan::set_rx_mode` |
| `vlan_dev_change_rx_flags()` | per-rx-flag | `Vlan::change_rx_flags` |
| `vlan_dev_ioctl()` | per-ioctl | `Vlan::ioctl` |
| `vlan_dev_neigh_setup()` | per-neighbor setup | `Vlan::neigh_setup` |
| `vlan_dev_select_queue()` | per-tx-queue select | `Vlan::select_queue` |
| `vlan_dev_get_stats64()` | per-stats | `Vlan::get_stats` |
| `vlan_skb_recv()` | per-tagged-rx dispatch | `Vlan::skb_recv` |
| `vlan_dev_hard_header()` | per-tx eth-hdr build | `Vlan::hard_header` |
| `__vlan_hwaccel_put_tag()` | per-skb HW-tag insertion | helper |
| `skb_vlan_push()` | per-skb sw-tag insertion | helper |

## Compatibility contract

REQ-1: Per-vlan_dev netdev_ops:
- ndo_init = vlan_dev_init.
- ndo_uninit = vlan_dev_uninit.
- ndo_open / ndo_stop = vlan_dev_open / stop (propagate to real_dev).
- ndo_start_xmit = vlan_dev_hard_start_xmit.
- ndo_change_mtu = vlan_dev_change_mtu.
- ndo_set_mac_address = vlan_dev_set_mac_address.
- ndo_set_rx_mode = vlan_dev_set_rx_mode.
- ndo_change_rx_flags = vlan_dev_change_rx_flags.
- ndo_select_queue = vlan_dev_select_queue.
- ndo_get_stats64 = vlan_dev_get_stats64.
- ndo_eth_ioctl = vlan_dev_ioctl.

REQ-2: vlan_dev_init:
- priv = vlan_dev_priv(dev).
- dev.priv_flags |= IFF_802_1Q_VLAN.
- dev.features |= subset of real_dev.features (HW-VLAN-CTAG-TX/RX/FILTER, GSO, GRO, ...).
- dev.dev_addr = real_dev.dev_addr (default).
- dev.broadcast = real_dev.broadcast.
- dev.mtu = real_dev.mtu - VLAN_HLEN (when not HW-CTAG-TX).
- dev.flags = real_dev.flags.
- dev.gso_max_size = real_dev.gso_max_size.

REQ-3: vlan_dev_hard_start_xmit:
- priv = vlan_dev_priv(dev).
- tci = priv.vlan_id | (skb_priority_to_pcp(skb.priority) << VLAN_PRIO_SHIFT).
- If real_dev.features & NETIF_F_HW_VLAN_CTAG_TX:
  - __vlan_hwaccel_put_tag(skb, htons(priv.vlan_proto), tci).
- Else:
  - skb_vlan_push(skb, htons(priv.vlan_proto), tci).
- skb.dev = real_dev.
- ret = dev_queue_xmit(skb).
- per-stats update.

REQ-4: vlan_skb_recv (called from net/core/dev rx-path):
- priv = vlan_dev_priv(vlan_dev).
- skb = vlan_check_reorder_header(skb).
- skb.dev = vlan_dev.
- skb.skb_iif = vlan_dev.ifindex.
- /* skb.protocol already set by net/core to inner */
- per-stats update.
- napi_gro_receive(napi, skb).

REQ-5: vlan_dev_set_mac_address:
- new_mac different from real_dev.dev_addr → dev_uc_add(real_dev, new_mac).
- old (saved) MAC: dev_uc_del.

REQ-6: vlan_dev_set_rx_mode:
- dev_uc_sync(real_dev, vlan_dev).
- dev_mc_sync(real_dev, vlan_dev).

REQ-7: vlan_dev_change_mtu:
- vlan_dev MTU ≤ real_dev MTU - VLAN_HLEN.
- Validate; return -ERANGE if exceeded.

REQ-8: vlan_dev_change_rx_flags:
- IFF_PROMISC: dev_set_promiscuity(real_dev, +/-1).
- IFF_ALLMULTI: dev_set_allmulti(real_dev, +/-1).

REQ-9: vlan_dev_select_queue:
- Forward to real_dev.netdev_ops.ndo_select_queue.

REQ-10: vlan_dev_ioctl (SIOCSHWTSTAMP):
- Forward to real_dev (ndo_eth_ioctl).

REQ-11: Per-vlan_dev features:
- vlan_dev.features = real_dev.vlan_features & ~NETIF_F_HW_VLAN_*.

REQ-12: Per-stats:
- per_cpu_ptr(priv.vlan_pcpu_stats).

## Acceptance Criteria

- [ ] AC-1: `ip link add ... type vlan id 10`: vlan_dev_init invoked; features inherited.
- [ ] AC-2: `ip link set eth0.10 up`: ndo_open propagates to real_dev.
- [ ] AC-3: TX skb on eth0.10: HW-CTAG-TX inserts tag OR sw skb_vlan_push.
- [ ] AC-4: RX tagged skb on eth0: vlan_skb_recv → eth0.10.
- [ ] AC-5: vlan_dev_set_mac different from real_dev: real_dev UC list +1.
- [ ] AC-6: vlan_dev_set_rx_mode IFF_PROMISC: real_dev promisc +1.
- [ ] AC-7: vlan_dev MTU > real_dev MTU - 4: -ERANGE.
- [ ] AC-8: ifconfig eth0.10 stats: per-vlan-dev rx_packets / tx_packets.
- [ ] AC-9: SIOCSHWTSTAMP on eth0.10: forwarded to eth0.
- [ ] AC-10: ethtool -k eth0.10: features-list inherited.

## Architecture

(Builds atop `Vlan::register_device` from `vlan.md`.)

`Vlan::dev_init(dev) -> Result<()>`:
1. priv = vlan_dev_priv(dev).
2. real_dev = priv.real_dev.
3. dev.gso_max_size = real_dev.gso_max_size.
4. dev.gso_max_segs = real_dev.gso_max_segs.
5. dev.priv_flags = (real_dev.priv_flags & ~IFF_XMIT_DST_RELEASE) | IFF_802_1Q_VLAN.
6. dev.features = real_dev.vlan_features & ~NETIF_F_HW_VLAN_FEATURES.
7. dev.hw_features = real_dev.hw_features & ~NETIF_F_HW_VLAN_FEATURES.
8. dev.gso_partial_features = real_dev.gso_partial_features.
9. dev.flags = (real_dev.flags & ~(IFF_UP | IFF_PROMISC | IFF_ALLMULTI | IFF_MASTER | IFF_SLAVE)) | (dev.flags & (IFF_UP | IFF_PROMISC | IFF_ALLMULTI)).
10. ether_addr_copy(dev.dev_addr, real_dev.dev_addr).
11. ether_addr_copy(dev.broadcast, real_dev.broadcast).
12. dev.mtu = real_dev.mtu.
13. priv.vlan_pcpu_stats = netdev_alloc_pcpu_stats(...).

`Vlan::dev_open(dev) -> Result<()>`:
1. priv = vlan_dev_priv(dev).
2. err = dev_uc_add(priv.real_dev, dev.dev_addr).  // if MAC differs
3. err = vlan_proto_open(priv.real_dev).

`Vlan::xmit(skb, dev) -> NetdevTxT`:
1. priv = vlan_dev_priv(dev).
2. skb = vlan_insert_inner_tag(skb, htons(priv.vlan_proto), priv.vlan_id, skb.priority).
3. skb.dev = priv.real_dev.
4. /* per-cpu stats */ stats.tx_packets++; stats.tx_bytes += skb.len.
5. dev_queue_xmit(skb).

`Vlan::skb_recv(skb, dev, pt, orig_dev) -> i32`:
1. tci = vlan_tx_tag_get(skb).
2. proto_idx = Vlan::proto_idx(skb.protocol).
3. vid = tci & VLAN_VID_MASK.
4. priv = real_dev.vlan_info.grp.grp_array[proto_idx][vid] -> vlan_dev_priv.
5. If !priv: drop.
6. skb.dev = priv.dev.
7. /* per-cpu stats */ stats.rx_packets++; stats.rx_bytes += skb.len.
8. /* PCP via ingress_priority_map */
9. skb.priority = priv.ingress_priority_map[(tci & VLAN_PRIO_MASK) >> VLAN_PRIO_SHIFT].
10. napi_gro_receive(...).

`Vlan::set_mac(dev, addr) -> Result<()>`:
1. priv = vlan_dev_priv(dev).
2. real_dev = priv.real_dev.
3. addr.sa_family = real_dev.type.
4. err = dev_uc_add(real_dev, addr.sa_data).
5. err = dev_uc_del(real_dev, dev.dev_addr).
6. ether_addr_copy(dev.dev_addr, addr.sa_data).

`Vlan::set_rx_mode(dev)`:
1. priv = vlan_dev_priv(dev).
2. dev_mc_sync(priv.real_dev, dev).
3. dev_uc_sync(priv.real_dev, dev).

`Vlan::change_mtu(dev, new_mtu) -> Result<()>`:
1. priv = vlan_dev_priv(dev).
2. max = priv.real_dev.mtu - (real_dev.features & NETIF_F_HW_VLAN_CTAG_TX ? 0 : VLAN_HLEN).
3. If new_mtu > max: return Err(ERANGE).
4. dev.mtu = new_mtu.

`Vlan::ioctl(dev, ifr, cmd) -> Result<()>`:
1. priv = vlan_dev_priv(dev).
2. real_dev = priv.real_dev.
3. /* SIOCSHWTSTAMP / SIOCGHWTSTAMP / SIOCSHWTSTAMP_RT etc. */
4. real_dev.netdev_ops.ndo_eth_ioctl(real_dev, ifr, cmd).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `vlan_mtu_le_real_mtu_minus_4` | INVARIANT | per-vlan_dev MTU ≤ real_dev MTU - VLAN_HLEN (sw-mode). |
| `xmit_real_dev_consistent` | INVARIANT | per-tx skb.dev = real_dev. |
| `rx_vlan_dev_consistent` | INVARIANT | per-rx skb.dev = vlan_dev. |
| `mac_propagated_to_real_dev` | INVARIANT | per-mac-change: real_dev UC list updated. |
| `rx_flag_promisc_propagated` | INVARIANT | per-IFF_PROMISC flip on vlan_dev: real_dev.promiscuity ±1. |

### Layer 2: TLA+

`net/8021q/vlan_dev.tla`:
- Per-vlan_dev tx (tag-insert) + per-vlan_dev rx (tag-strip) + per-state propagation.
- Properties:
  - `safety_tx_tag_correct` — per-tx skb has 802.1Q tag with correct VID/PCP.
  - `safety_rx_dispatch_unique` — per-rx tagged skb to single vlan_dev.
  - `safety_state_propagation` — per-mac/rx-mode change reflected in real_dev.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Vlan::dev_init` post: features inherited; MTU/MAC set | `Vlan::dev_init` |
| `Vlan::xmit` post: skb has tag; skb.dev=real_dev | `Vlan::xmit` |
| `Vlan::skb_recv` post: skb.dev=vlan_dev; per-cpu stats++ | `Vlan::skb_recv` |
| `Vlan::set_mac` post: real_dev.UC contains new MAC | `Vlan::set_mac` |

### Layer 4: Verus/Creusot functional

`Per-vlan_dev xmit → 802.1Q-tagged skb on real_dev egress; per-real_dev rx tagged → strip → vlan_dev → upper stack` semantic equivalence: per-IEEE 802.1Q packet flow.

## Hardening

(Inherits row-1 features from `net/8021q/vlan.md` § Hardening.)

VLAN-dev-specific reinforcement:

- **Per-MTU bound to real_dev MTU - HLEN** — defense against per-tag overflow MTU.
- **Per-MAC change via dev_uc_add/del** — defense against per-real_dev silent UC overflow.
- **Per-rx_flag PROMISC/ALLMULTI ±1 paired** — defense against per-flag mismatch.
- **Per-stats per-CPU** — defense against per-stat race.
- **Per-features filtered to non-VLAN-HW** — defense against per-feature double-handling.
- **Per-vlan_dev select_queue forwards** — defense against per-vlan-dev missing XPS info.
- **Per-skb tag insertion via hwaccel or push** — defense against per-tag-insert OOB.
- **Per-vlan_dev open propagates to real_dev** — defense against per-real-down state.
- **Per-vlan_dev neigh_setup forwards** — defense against per-neighbor lookup miss.
- **Per-vlan_dev unbind from real_dev on uninit** — defense against per-real-removed UAF.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- VLAN core (covered in `vlan.md` Tier-3)
- net/core/dev (covered in `dev.md` Tier-3)
- ethtool framework (covered separately)
- Implementation code
