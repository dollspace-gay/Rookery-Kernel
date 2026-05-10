# Tier-3: drivers/net/vxlan/vxlan_core.c — VXLAN (Virtual eXtensible LAN) tunnel driver

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/00-overview.md
upstream-paths:
  - drivers/net/vxlan/vxlan_core.c (~5023 lines)
  - drivers/net/vxlan/vxlan_multicast.c
  - drivers/net/vxlan/vxlan_mdb.c
  - drivers/net/vxlan/vxlan_vnifilter.c
  - drivers/net/vxlan/vxlan_private.h
  - include/net/vxlan.h
  - include/uapi/linux/if_link.h (IFLA_VXLAN_*)
-->

## Summary

VXLAN (RFC 7348) is an L2-in-UDP tunnel encapsulation. Per-vxlan-iface bound to (UDP-port, group/peer-IP, VNI). Per-skb tx wraps inner-Ethernet frame in: outer-eth + outer-IP + outer-UDP + VXLAN-header(8B; flags + VNI) → inner-eth + inner-payload. Per-skb rx: UDP-encap socket receives; strips VXLAN-header; classifies by VNI to per-VNI vxlan_dev; up-pass as ethernet frame. Per-FDB maps inner-MAC → outer-IP for unicast tunnel. Per-multicast forwards unknown-MAC over IP multicast group. Per-VNIFILTER mode: single iface multiplexes multiple VNIs. Critical for: cloud-overlay networking (Kubernetes Calico/Flannel/Cilium, OpenStack Neutron, AWS VPC, Azure VNet equivalent).

This Tier-3 covers `vxlan_core.c` (~5023 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct vxlan_dev` | per-vxlan netdev | `VxlanDev` |
| `struct vxlan_sock` | per-(UDP-port, AF) socket | `VxlanSock` |
| `struct vxlan_fdb` | per-(VNI, inner-MAC) FDB entry | `VxlanFdb` |
| `vxlan_init()` | per-module init | `Vxlan::init` |
| `vxlan_setup()` | per-vxlan netdev setup | `Vxlan::setup` |
| `vxlan_xmit()` | per-skb TX (encap) | `Vxlan::xmit` |
| `vxlan_rcv()` | per-UDP-encap RX (decap) | `Vxlan::rcv` |
| `vxlan_open_socket()` | per-(port, AF) UDP socket | `Vxlan::open_socket` |
| `vxlan_fdb_create()` | per-FDB add | `VxlanFdb::create` |
| `vxlan_fdb_find_rdst()` | per-MAC lookup | `VxlanFdb::find_rdst` |
| `vxlan_fdb_dump()` | per-rtnetlink dump | `VxlanFdb::dump` |
| `vxlan_xmit_one()` | per-rdst encap-and-send | `Vxlan::xmit_one` |
| `udp_tunnel_xmit_skb()` | UDP-encap helper | shared |
| `IFLA_VXLAN_*` | per-rtnetlink attrs | UAPI |
| `VXLAN_F_*` | per-feature flags | shared |
| `VXLAN_HF_VNI` | per-header VNI flag | shared |

## Compatibility contract

REQ-1: VXLAN header (8 bytes):
- byte[0]: flags (bit-3 VXLAN_HF_VNI = 1).
- bytes[1..4]: reserved.
- bytes[4..7]: VNI (24-bit).
- byte[7]: reserved.

REQ-2: Per-vxlan_dev:
- vni: u32 default-VNI.
- remote: union { ip4 / ip6 } default-remote.
- saddr: source-IP (or per-rdst).
- src_port_min / src_port_max: per-flow UDP-src-port range.
- dst_port: outer-UDP-dst-port (default 4789 IANA).
- ttl, tos: outer.
- flags: VXLAN_F_LEARN / VXLAN_F_PROXY / VXLAN_F_RSC / VXLAN_F_L2MISS / VXLAN_F_L3MISS / VXLAN_F_COLLECT_METADATA / VXLAN_F_VNIFILTER / VXLAN_F_REMCSUM_TX / etc.
- ttl_inherit: from inner-IP.
- ageing: FDB ageing time.

REQ-3: Per-rtnetlink IFLA_VXLAN_*:
- IFLA_VXLAN_ID: VNI.
- IFLA_VXLAN_GROUP: multicast group / unicast peer.
- IFLA_VXLAN_PORT: UDP dst-port.
- IFLA_VXLAN_TTL / TOS / SRC_PORT_MIN / MAX.
- IFLA_VXLAN_LEARNING / PROXY / RSC / L2MISS / L3MISS.
- IFLA_VXLAN_COLLECT_METADATA: per-skb tunnel_info-driven (no fixed VNI).
- IFLA_VXLAN_VNIFILTER: per-vxlan_dev multiplexes multiple VNIs.

REQ-4: vxlan_xmit (per-skb TX):
- inner_eth = skb_eth_hdr.
- fdb = vxlan_fdb_find_rdst(inner_eth.h_dest, vni) ∨ default-rdst.
- For each rdst (multi-remote possible):
  - vxlan_xmit_one(skb_clone, vxlan_dev, rdst).

REQ-5: vxlan_xmit_one:
- src_port = vxlan_src_port (computed from per-skb hash).
- Build VXLAN header (flags=0x08 | VNI).
- udp_tunnel_xmit_skb (or udp_tunnel6_xmit_skb): wraps in UDP+IP.
- Per-IPv4: ip_local_out.
- Per-IPv6: ip6_local_out.

REQ-6: vxlan_rcv (per-UDP-encap RX):
- Per-vxlan_sock receives via udp_tunnel_encap.
- Validate VXLAN-header (flags + VNI).
- Lookup vxlan_dev by (sock, vni).
- skb_pull(VXLAN_HLEN); skb.protocol = htons(ETH_P_TEB).
- skb.dev = vxlan_dev.
- /* Strip outer + per-VNI inner */
- netif_rx(skb).

REQ-7: vxlan_fdb (per-vxlan-dev hashtable):
- Key: (h_dest, vni).
- Value: rdst-list { remote_ip, remote_port, vni, ifindex }.
- Per-rxed-skb learning: if (VXLAN_F_LEARN ∧ inner-eth.h_source not in FDB): create entry.

REQ-8: Per-multicast unknown-MAC:
- Per-vxlan-dev with multicast group: skb forwarded to group.
- Per-receiver per-MAC populates FDB.

REQ-9: Per-VNIFILTER:
- Single vxlan-dev with multiple VNIs (per-VNI vlan-like).
- Per-skb metadata: vni in skb.tunnel_info.

REQ-10: Per-FDB notification:
- RTM_NEWNEIGH / DELNEIGH on FDB-changes.

REQ-11: Per-checksum offload:
- VXLAN_F_REMCSUM_TX / RX: remote-CSUM.
- NETIF_F_GSO_UDP_TUNNEL: GSO-aware encap.

REQ-12: Per-namespace:
- Per-net VXLAN instance.

## Acceptance Criteria

- [ ] AC-1: `ip link add vxlan0 type vxlan id 100 group 239.1.1.1 dev eth0`: vxlan_dev created.
- [ ] AC-2: `ip link set vxlan0 up`: vxlan_sock opened on UDP port 4789.
- [ ] AC-3: TX skb on vxlan0: outer-eth + outer-IP + UDP + VXLAN-hdr + inner.
- [ ] AC-4: RX UDP packet on port 4789 with matching VNI: skb decapped + delivered to vxlan0.
- [ ] AC-5: Per-FDB entry: lookup MAC → rdst.
- [ ] AC-6: Unknown-MAC: per-multicast group forwards.
- [ ] AC-7: VXLAN_F_LEARN + RX: src-MAC FDB-learned.
- [ ] AC-8: VXLAN_F_VNIFILTER: 100 VNIs on single iface.
- [ ] AC-9: GSO UDP-tunnel: large skb segmented at NIC.
- [ ] AC-10: bridge fdb add ... vxlan0: static FDB; user-controlled VNI mapping.

## Architecture

Per-vxlan_dev priv:

```
struct VxlanDev {
  hlist: HListLink,                              // per-vs.vni_list
  vs: *VxlanSock,                                 // per-(port, AF)
  remote_addr: VxlanAddr,                         // ip4/ipv6
  saddr: VxlanAddr,
  vni: u32,
  default_dst: VxlanRdst,
  cfg: VxlanCfg,
  ttl: u8,
  tos: u8,
  src_port_min: u16,
  src_port_max: u16,
  dst_port: __be16,
  flags: u32,                                    // VXLAN_F_*
  age_interval: u32,
  age_timer: TimerList,
  fdb_head: HListHead<VxlanFdb>,                 // per-(MAC, VNI)
  ...
}
```

Per-FDB:

```
struct VxlanFdb {
  rhnode: RhashNode,
  key: VxlanFdbKey,                              // (eth_addr, vni)
  remotes: ListHead<VxlanRdst>,
  state: u16,                                    // NUD_*
  flags: u8,
  used: AtomicLong,
  updated: AtomicLong,
  vni: u32,
  ifindex: u32,
}

struct VxlanRdst {
  list: ListLink,
  remote_ip: VxlanAddr,
  remote_port: __be16,
  remote_vni: u32,
  ifindex: u32,
  ...
}
```

Per-vxlan_sock:

```
struct VxlanSock {
  sock: *Sock,
  port: __be16,
  flags: u32,
  vni_list: HListHead<VxlanDev>,                  // per-VNI dispatch
  refcnt: AtomicI64,
}
```

`Vxlan::open_socket(port, family, flags) -> *VxlanSock`:
1. /* Re-use existing per-(port, family) socket */
2. vs = lookup-vs(port, family).
3. if !vs:
   - sock = udp_tunnel_sock_create(net, &cfg, IPPROTO_UDP, ...).
   - vs = alloc; vs.sock = sock; vs.port = port.
   - sock.encap_type = UDP_ENCAP_VXLAN; sock.encap_rcv = vxlan_rcv.
4. atomic_inc(&vs.refcnt).
5. Return vs.

`Vxlan::xmit(skb, dev) -> NetdevTxT`:
1. vxlan = vxlan_priv(dev).
2. eth = eth_hdr(skb).
3. fdb = VxlanFdb::find(vxlan, eth.h_dest, vxlan.vni).
4. if !fdb:
   - if !(vxlan.flags & VXLAN_F_LEARN ∧ vxlan.flags & VXLAN_F_PROXY): drop.
   - rdst = vxlan.default_dst.
5. else: rdst = first-rdst.
6. for each rdst in fdb.remotes:
   - skb_clone for additional rdst.
   - Vxlan::xmit_one(vxlan, skb, rdst).

`Vxlan::xmit_one(vxlan, skb, rdst)`:
1. vxh = (VxlanHdr*)skb_push(skb, VXLAN_HLEN).
2. vxh.flags = VXLAN_HF_VNI.
3. vxh.vni = htonl(rdst.remote_vni << 8).
4. src_port = vxlan_src_port(vxlan.src_port_min, vxlan.src_port_max, skb_hash).
5. udp_tunnel_xmit_skb(rt, vxlan.vs.sock, skb, ..., src_port, rdst.remote_port).

`Vxlan::rcv(sk, skb)`:
1. vxh = (VxlanHdr*)skb.data.
2. validate vxh.flags & VXLAN_HF_VNI.
3. vni = ntohl(vxh.vni) >> 8.
4. vs = (VxlanSock*)sk.user_data.
5. vxlan = vxlan_vs_lookup(vs, vni).
6. if !vxlan: drop.
7. /* Per-LEARN: src-MAC learning */
8. skb_pull(skb, VXLAN_HLEN).
9. skb.protocol = htons(ETH_P_TEB).
10. skb.dev = vxlan.dev.
11. netif_rx(skb).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `vni_24bit` | INVARIANT | per-vxlan_dev.vni < (1 << 24). |
| `flags_vni_set_per_xmit` | INVARIANT | per-VXLAN-hdr-emit: flags & VXLAN_HF_VNI. |
| `udp_dst_port_default_4789` | INVARIANT | per-default cfg: dst_port == 4789. |
| `fdb_unique_per_mac_vni` | INVARIANT | per-(eth_addr, vni): at most one VxlanFdb. |
| `vs_refcount_balanced` | INVARIANT | per-open_socket inc; per-close dec. |

### Layer 2: TLA+

`net/vxlan/vxlan.tla`:
- Per-vxlan_dev RX/TX + per-FDB learning + per-multicast flood.
- Properties:
  - `safety_round_trip_vni_preserved` — per-tx vni → encap → rx vni: same.
  - `safety_no_loopback` — per-rx skb: not delivered back to source-vxlan.
  - `liveness_unknown_mac_eventually_learned` — per-multicast flood + reply: FDB-entry created.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Vxlan::xmit` post: per-skb encapsulated + udp_tunnel_xmit_skb invoked | `Vxlan::xmit` |
| `Vxlan::rcv` post: per-skb decapped + delivered to vxlan_dev | `Vxlan::rcv` |
| `Vxlan::open_socket` post: vs exists; per-(port, family) unique | `Vxlan::open_socket` |
| `VxlanFdb::create` post: entry in fdb_head; key unique | `VxlanFdb::create` |

### Layer 4: Verus/Creusot functional

`Per-VNI L2-in-UDP encapsulation: inner-eth+inner-payload → outer-eth+outer-IP+UDP+VXLAN-hdr+inner` semantic equivalence: per-RFC 7348 VXLAN.

## Hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

VXLAN-specific reinforcement:

- **Per-VNI 24-bit bounded** — defense against per-VNI overflow.
- **Per-FDB rhashtable RCU** — defense against per-walk UAF.
- **Per-FDB ageing periodic** — defense against per-stale-MAC unbounded growth.
- **Per-VXLAN-hdr-flag validation** — defense against per-malformed-VXLAN-pkt crash.
- **Per-multicast-group bounded** — defense against per-flood unbounded.
- **Per-namespace VXLAN scoped** — defense against cross-ns leak.
- **Per-VNIFILTER per-VNI ACL** — defense against per-tenant cross-VNI leak.
- **Per-CAP_NET_ADMIN for create/delete** — defense against unprivileged tunnel-create.
- **Per-checksum-offload validated by NIC cap** — defense against per-config invalid.
- **Per-source-IP anti-spoof** — defense against per-rx with mismatched src.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- VXLAN multicast (vxlan_multicast.c; covered separately if expanded)
- VXLAN MDB (vxlan_mdb.c)
- VXLAN VNIFILTER (vxlan_vnifilter.c)
- udp_tunnel framework (covered separately)
- Implementation code
