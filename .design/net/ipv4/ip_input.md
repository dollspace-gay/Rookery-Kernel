# Tier-3: net/ipv4/ip_input.c — IPv4 RX dispatch (per-skb header parse + ip_route_input + ip_local_deliver)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/ipv4/00-overview.md
upstream-paths:
  - net/ipv4/ip_input.c
  - net/ipv4/ip_forward.c
  - include/net/ip.h
-->

## Summary

`net/ipv4/ip_input.c` is the IPv4 receive-path entry — every IPv4 skb dequeued from netdev RX softirq dispatches via `ip_rcv` → `ip_rcv_finish` → routing-decision → `ip_local_deliver` (local) or `ip_forward` (forward). Per-skb: validate IP header (version, IHL, total-length, checksum); strip per-NIC L2; dispatch via per-protocol IPPROTO handler (TCP, UDP, ICMP, GRE, etc.). Critical for: every IPv4 packet arriving at a network host.

This Tier-3 covers `ip_input.c` (~718 lines) + `ip_forward.c` (~181 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `ip_rcv(skb, dev, pt, orig_dev)` | per-skb IPv4 RX entry | `IpInput::rcv` |
| `ip_rcv_finish(net, sk, skb)` | post-NETFILTER PREROUTING | `IpInput::rcv_finish` |
| `ip_rcv_core(skb, net)` | per-skb header validate + parse | `IpInput::rcv_core` |
| `ip_local_deliver(skb)` | per-skb local L4 dispatch | `IpInput::local_deliver` |
| `ip_local_deliver_finish(net, sk, skb)` | per-skb post-NETFILTER LOCAL_IN | `IpInput::local_deliver_finish` |
| `ip_forward(skb)` | per-skb forward path | `IpForward::forward` |
| `ip_forward_finish(net, sk, skb)` | per-skb post-NETFILTER FORWARD | `IpForward::forward_finish` |
| `ip_options_rcv_srr(skb, dev)` | per-skb source-route-record | `IpInput::options_rcv_srr` |
| `ip_route_input_noref(skb, daddr, saddr, tos, dev)` | route-lookup (covered in route.md) | shared |
| `inet_protos[]` | per-protocol handler table | `Inet::PROTOS` |
| `raw_local_deliver(skb, protocol)` | per-RAW-socket delivery | `Raw::local_deliver` |
| `ip_call_ra_chain(skb)` | per-router-alert chain | `IpInput::call_ra_chain` |

## Compatibility contract

REQ-1: Per-skb `ip_rcv` flow (per-NETIF protocol-handler):
1. orig_dev := skb->dev (input NIC).
2. ip_rcv_core: validate IP header.
3. NF_HOOK(NFPROTO_IPV4, NF_INET_PRE_ROUTING, skb, dev, NULL, ip_rcv_finish).
4. Return NET_RX_SUCCESS.

REQ-2: Per-skb header validation (`ip_rcv_core`):
1. Pull 20 bytes (min IPv4 header).
2. iph := ip_hdr(skb).
3. Validate version == 4.
4. Validate IHL >= 5.
5. Pull full header (IHL * 4 bytes).
6. Validate ip_total_length >= IHL*4.
7. Validate checksum (ip_fast_csum).
8. Trim skb to IP-total-length.
9. Stash per-skb info (skb->protocol = ETH_P_IP, etc.).

REQ-3: `ip_rcv_finish`:
1. ip_route_input_noref(skb, iph->daddr, iph->saddr, iph->tos, dev) — sets skb->_skb_refdst.
2. dst := skb_dst(skb).
3. dst->input(skb): typically ip_local_deliver OR ip_forward.

REQ-4: `ip_local_deliver`:
1. If iph->frag_off & ~MF: ip_defrag (reassemble fragments).
2. NF_HOOK(NF_INET_LOCAL_IN, skb, ..., ip_local_deliver_finish).

REQ-5: `ip_local_deliver_finish`:
1. raw_local_deliver(skb, iph->protocol) — per-RAW-socket dispatch.
2. ipprot := rcu_dereference(inet_protos[iph->protocol]).
3. If ipprot:
   - ipprot->handler(skb) — TCP/UDP/ICMP/etc.
4. Else:
   - icmp_send(skb, ICMP_DEST_UNREACH, ICMP_PROT_UNREACH, 0).

REQ-6: `ip_forward`:
1. Validate hop-limit (iph->ttl > 1; else ICMP_TIME_EXCEEDED).
2. NF_HOOK(NF_INET_FORWARD, skb, ..., ip_forward_finish).
3. ip_decrease_ttl(iph): TTL--.
4. ip_send_check: recompute checksum.

REQ-7: `ip_forward_finish`:
1. dst_output(net, sk, skb) — passes to dst-output (typically ip_output).
2. ip_output → ip_finish_output → ip_finish_output2 → neigh_output → dev_queue_xmit.

REQ-8: Per-protocol handler registration:
- inet_add_protocol(prot, protocol_num).
- TCP: tcp_protocol; UDP: udp_protocol; ICMP: icmp_protocol; etc.

REQ-9: Source-route-record (SRR):
- IP options field; per-source-route entry.
- Specially handled to update routing decision.

REQ-10: Router-alert chain:
- IP options "Router Alert" → call all subscribed handlers (e.g., RSVP, IGMP).
- Per-skb ip_call_ra_chain.

REQ-11: NETFILTER hooks:
- NF_INET_PRE_ROUTING (before routing decision).
- NF_INET_LOCAL_IN (locally-destined packet).
- NF_INET_FORWARD (forwarded packet).
- NF_INET_LOCAL_OUT (locally-generated outbound).
- NF_INET_POST_ROUTING (after routing).

REQ-12: Per-skb Conntrack:
- ct hook in NF_INET_PRE_ROUTING.
- Per-flow ct established/related/new/invalid.

## Acceptance Criteria

- [ ] AC-1: Per-skb IPv4 packet to host: dispatched via inet_protos[iph->protocol] handler.
- [ ] AC-2: Per-skb IPv4 packet to forward: ip_forward → dst_output → next-hop dev_queue_xmit.
- [ ] AC-3: Per-skb invalid IPv4 (bad checksum): dropped; counter increment.
- [ ] AC-4: Per-skb fragment: ip_defrag reassembles; reassembled packet delivered.
- [ ] AC-5: TTL-expired packet: dropped + ICMP_TIME_EXCEEDED sent.
- [ ] AC-6: Per-skb IP options SRR: source-route honored.
- [ ] AC-7: NETFILTER hook PREROUTING: rule matches; conntrack establishes flow.
- [ ] AC-8: Per-skb RAW socket: matching raw-socket per-protocol receives copy.
- [ ] AC-9: Multi-AF host: IPv4 + IPv6 packets dispatched via separate paths.
- [ ] AC-10: Per-CPU softirq: 32-CPU host with high RX-PPS; per-CPU input independent.

## Architecture

`IpInput::rcv(skb, dev, pt, orig_dev)`:
1. skb := skb_share_check(skb, GFP_ATOMIC).
2. ret := ip_rcv_core(skb, net).
3. If ret < 0: return NET_RX_DROP.
4. NF_HOOK(NFPROTO_IPV4, NF_INET_PRE_ROUTING, net, NULL, skb, dev, NULL, ip_rcv_finish).
5. Return NET_RX_SUCCESS.

`IpInput::rcv_core(skb, net)`:
1. If !pskb_may_pull(skb, sizeof(struct iphdr)): return -1.
2. iph := ip_hdr(skb).
3. If iph.version != 4 OR iph.ihl < 5: return -1.
4. ihl := iph.ihl * 4.
5. If !pskb_may_pull(skb, ihl): return -1.
6. If ip_fast_csum(iph, iph.ihl): return -1.
7. tot_len := ntohs(iph.tot_len).
8. If skb.len < tot_len OR tot_len < ihl: return -1.
9. pskb_trim_rcsum(skb, tot_len - ihl + ihl).
10. skb->transport_header = skb->network_header + ihl.
11. Return 0.

`IpInput::local_deliver_finish(net, sk, skb)`:
1. raw_local_deliver(skb, iph->protocol).
2. ipprot := rcu_dereference(inet_protos[iph->protocol]).
3. If ipprot && ipprot->handler:
   - ret := ipprot->handler(skb).
   - If ret < 0: ICMP send; goto drop.
4. Else: send ICMP_PROT_UNREACH.

`IpForward::forward(skb)`:
1. iph := ip_hdr(skb).
2. If iph->ttl <= 1: ICMP_TIME_EXCEEDED; drop.
3. If !ip_options_rcv_srr(skb, dev): return.
4. NF_HOOK(NF_INET_FORWARD, skb, ..., ip_forward_finish).

`IpForward::forward_finish(net, sk, skb)`:
1. ip_decrease_ttl(ip_hdr(skb)).
2. dst_output(net, sk, skb).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `ip_header_pull_safe` | OOB | per-skb pskb_may_pull bounds checked before iph access. |
| `ip_csum_validated_before_route` | INVARIANT | per-skb checksum verified before further processing. |
| `tot_len_le_skb_len` | INVARIANT | per-skb skb.len ≥ iph.tot_len. |
| `ttl_decrement_atomic` | INVARIANT | per-forward TTL decrement atomic w.r.t. checksum. |
| `inet_protos_rcu_protected` | UAF | per-protocol handler registered/unregistered via RCU. |

### Layer 2: TLA+

`net/ipv4/ip_rx_dispatch.tla`:
- Per-skb state ∈ {Received, HeaderValidated, Routed, LocalDelivered, Forwarded, Dropped}.
- Properties:
  - `safety_no_local_deliver_without_local_route` — LocalDelivered implies route-decision local.
  - `safety_no_forward_without_remote_route` — Forwarded implies route-decision remote.
  - `liveness_received_eventually_dispositioned` — every Received eventually Delivered/Forwarded/Dropped.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `IpInput::rcv` post: skb dispatched via NF_HOOK to ip_rcv_finish OR dropped | `IpInput::rcv` |
| `IpInput::rcv_core` post: returns 0 iff valid IPv4 header | `IpInput::rcv_core` |
| `IpInput::local_deliver_finish` post: per-protocol handler called OR ICMP sent | `IpInput::local_deliver_finish` |
| `IpForward::forward` post: TTL >= 1 + ICMP-sent if expired | `IpForward::forward` |

### Layer 4: Verus/Creusot functional

`Per-skb IPv4 RX: kernel-level demux + L4 dispatch matches IPv4 RFC791 + RFC2474` semantic equivalence: per-skb the resulting handler invocation matches RFC-defined per-protocol number.

## Hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

ip_input-specific reinforcement:

- **Per-skb pskb_may_pull bounds** — defense against truncated header OOB read.
- **Per-skb csum validation** — defense against corrupted-packet acceptance.
- **TTL-decrement before forward-output** — defense against IP-loop forwarding.
- **NETFILTER hook ordering** — defense against bypass of conntrack/iptables.
- **inet_protos RCU-protected** — defense against unregister-during-dispatch UAF.
- **ICMP send rate-limited** — defense against ICMP flood reflection.
- **Per-skb skb_share_check** — defense against shared-skb modification corrupting clone.
- **iph fields validated** — defense against IHL=0 or version!=4.
- **Per-namespace socket lookup** — defense against cross-netns packet leak.
- **Source-route-record gated** by sysctl — defense against SRR-amplification attacks.
- **Router-alert chain ref-counted** — defense against handler-unregister-during-walk.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- IPv6 RX (covered separately)
- Routing (covered in `net/ipv4/route.md` Tier-3)
- TCP / UDP / ICMP per-protocol RX (covered separately)
- Conntrack (covered in `net/netfilter.md` Tier-2)
- IP fragmentation reassembly (covered in `net/ipv4/ip_fragment.md` future Tier-3)
- Implementation code
