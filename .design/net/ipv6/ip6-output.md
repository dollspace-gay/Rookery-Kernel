# Tier-3: net/ipv6/ip6-output — IPv6 TX path (build → frag → neigh → xmit)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/ipv6/ip6_output.c
  - net/ipv6/output_core.c
  - net/ipv6/ip6_offload.c
  - net/ipv6/ip6_checksum.c
  - net/ipv6/ip6_flowlabel.c
-->

## Summary
Tier-3 design for the IPv6 transmit path: socket-level `ip6_xmit` (called from TCP/UDP/RAW upper layers via `inet6_csk_xmit` and friends), the local-vs-route-driven egress decision (`ip6_local_out` → `ip6_output` → `ip6_finish_output` → `ip6_finish_output2`), per-flow flow-label assignment, IPv6 fragmentation (RFC 8200 § 4.5 — only by source; routers never fragment), per-netfilter `LOCAL_OUT` / `POST_ROUTING` hooks, neighbour resolution (calls into `net/ipv6/ndisc.md`), GSO/TSO segmentation offload, and per-netdev `xmit` invocation.

Sub-tier-3 of `net/ipv6/00-overview.md`. Pairs with `net/ipv6/ip6-input.md` (RX), `net/ipv6/route.md` (route selection), `net/ipv6/ndisc.md` (neighbour resolution), `net/ipv6/ip6_flowlabel.md` (per-socket flow-label management).

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| `ip6_xmit`, `ip6_output`, `ip6_finish_output(2)`, `ip6_send_skb`, `ip6_local_out`, fragmentation | `net/ipv6/ip6_output.c` |
| Identification + checksum + helper primitives | `net/ipv6/output_core.c` |
| GSO/TSO offload | `net/ipv6/ip6_offload.c` |
| Checksum primitives | `net/ipv6/ip6_checksum.c` |
| Per-socket flow-label management | `net/ipv6/ip6_flowlabel.c` |

## Compatibility contract

### `ip6_xmit` socket-driven egress

Called from TCP/UDP/RAW upper-layers; takes a `struct sock`, an skb, and a `flowi6` (flow descriptor with saddr/daddr/proto/sport/dport/flowlabel/oif/mark). Outputs:
1. Build IPv6 header from sock state + flowi6
2. Apply per-socket extension headers (HBH, RH, DestOpt) from `np->opt` per-socket setsockopt
3. Set hop_limit from `np->hop_limit` or default `64`
4. Set traffic class from `np->tclass`
5. Set flow label from `np->flow_label` or auto-generated per-flow
6. Compute pseudo-header checksum if not offloaded
7. Push to `ip6_local_out`

Identical semantics so userspace observed wire format matches upstream.

### Route lookup

`flowi6` resolves via `ip6_route_output` to an `rt6_info`; sets skb->dst. Identical algorithm.

### `ip6_output` local-or-fwd dispatch

After route resolves, `ip6_output` invokes:
- `ip6_finish_output` (after netfilter `LOCAL_OUT`/`POST_ROUTING`)
- For multicast: `ip6_mc_finish_output`

`ip6_finish_output` chooses:
- If skb fits MTU → `ip6_finish_output2` (direct neigh resolve + xmit)
- If skb exceeds MTU AND DF unset (default for UDP/RAW): `ip6_fragment` then per-fragment `ip6_finish_output2`
- If skb exceeds MTU AND DF set (e.g., for PMTU-discovery TCP): drop + ICMP PTB to source

Identical behavior so existing PMTU-discovery semantics + UDP-fragmentation match.

### Fragmentation (RFC 8200 § 4.5)

IPv6 fragmentation is source-only (intermediate routers MUST NOT fragment). Source-fragmentation builds Fragment Header (nexthdr=44) chained to the original nexthdr; per-fragment offset + identification field. `ip6_fragment` algorithm identical to upstream.

### Netfilter hooks

`NF_INET_LOCAL_OUT` (between `ip6_local_out` and `ip6_output`), `NF_INET_POST_ROUTING` (after fragmentation, before `ip6_finish_output2`). Each hook can ACCEPT/DROP/STOLEN/QUEUE/REPEAT identically.

### Neighbour resolution

`ip6_finish_output2` calls `__neigh_lookup_noref` then `neigh_output`. If neigh in INCOMPLETE state → skb queued to `arp_queue` (cross-ref `net/ipv6/ndisc.md`); if FAILED → drop + ICMP DestUnreach Code 3 (Address Unreachable).

### GSO/TSO

For sockets with `SOCK_GSO`-equivalent: `ip6_finish_output_gso` segments the skb into MTU-sized fragments before per-fragment xmit. Algorithm identical (allows hardware offload via `dev->features`).

### Flow label management

Per-socket `np->flow_label` from `IPV6_FLOWLABEL_MGR` setsockopt or per-flow auto-generated (RFC 6437) from `flowi6`. Identical algorithm.

### Per-netns counters

`Ip6OutRequests`, `Ip6OutDiscards`, `Ip6OutNoRoutes`, `Ip6OutMcastPkts`, `Ip6OutOctets`, `Ip6OutMcastOctets`, `Ip6OutBcastOctets`, `Ip6FragOKs`, `Ip6FragFails`, `Ip6FragCreates`. Format-identical (cross-ref `/proc/net/snmp6`).

## Requirements

- REQ-1: `ip6_xmit` builds IPv6 header from sock + flowi6 identical to upstream.
- REQ-2: Per-socket extension headers (`np->opt`): HBH/RH/DestOpt set via `IPV6_HOPOPTS`/`IPV6_RTHDR`/`IPV6_DSTOPTS` setsockopts; injected into TX skb.
- REQ-3: Hop-limit from `np->hop_limit` or default `64`; traffic class from `np->tclass`.
- REQ-4: Flow label: per-socket `np->flow_label` or auto-generated per-flow (RFC 6437 hash of flowi6 5-tuple).
- REQ-5: `ip6_output` netfilter `LOCAL_OUT` / `POST_ROUTING` invocations at identical points.
- REQ-6: `ip6_finish_output`: MTU check → fragmentation (DF=0) or PTB drop (DF=1).
- REQ-7: `ip6_fragment`: source-only fragmentation per RFC 8200 § 4.5; identical algorithm.
- REQ-8: `ip6_finish_output2`: neighbour lookup → neigh_output; backlog if INCOMPLETE; drop + ICMP if FAILED.
- REQ-9: GSO/TSO offload: per-`dev->features` segmentation; algorithm identical.
- REQ-10: Multicast TX: `ip6_mc_finish_output`; respects `IPV6_MULTICAST_LOOP` + `IPV6_MULTICAST_HOPS` setsockopts.
- REQ-11: Per-netns Ip6Out* counters bumped at identical points; format byte-identical.
- REQ-12: Per-socket flow-label management (`IPV6_FLOWLABEL_MGR`): label allocation / sharing / freeing + per-netns label table; cross-ref `net/ipv6/flowlabel.md`.
- REQ-13: Identification field generation: per-destination per-netns counter + jiffies-mix; identical algorithm to defeat ID-prediction attacks.
- REQ-14: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: A TCP-over-IPv6 sendmsg from socket S to dst D → tcpdump shows IPv6 header with saddr/daddr/hop_limit/tclass/flowlabel matching socket state. (covers REQ-1, REQ-3, REQ-4)
- [ ] AC-2: With `setsockopt IPV6_RTHDR` setting a Type-2 routing header on socket S → tcpdump shows the routing header injected. (covers REQ-2)
- [ ] AC-3: A UDP-over-IPv6 send of a 2 KB packet over a 1500-MTU iface → kernel fragments into 2 IPv6 fragments with Fragment Header; receiver reassembles. (covers REQ-7)
- [ ] AC-4: A TCP-over-IPv6 send of a 2 KB packet over a 1500-MTU iface (DF=1) → drop + ICMP PTB to src; subsequent socket sends respect the lower MTU. (covers REQ-6)
- [ ] AC-5: A send to an unresolved neigh → skb queued in arp_queue while NS is sent; on NA → skb drained. (covers REQ-8)
- [ ] AC-6: A send to a permanently-unresolved neigh (no NA after retries) → drop + ICMP Dest Unreach Code 3. (covers REQ-8)
- [ ] AC-7: GSO test: a 64KB TCP send over a 1500-MTU iface with `dev->features` indicating TSO support → kernel hands a single 64KB skb to the driver; without TSO support → kernel pre-segments into MTU chunks. (covers REQ-9)
- [ ] AC-8: Multicast TX test with `IPV6_MULTICAST_LOOP=1`: send to ff02::1 → also delivered locally; with =0 → not. (covers REQ-10)
- [ ] AC-9: `cat /proc/net/snmp6` Ip6Out* counters byte-identical after equivalent traffic. (covers REQ-11)
- [ ] AC-10: A per-socket `IPV6_FLOWLABEL_MGR` allocation → assigned label visible in tcpdump on every TX from that socket. (covers REQ-12)
- [ ] AC-11: Identification-field generation: 1000 sends to the same dst → IDs not monotonic-increment; pass entropy test ≥ 7.5 bits/8 bits per ID byte. (covers REQ-13)
- [ ] AC-12: Hardening section present and follows template. (covers REQ-14)

## Architecture

### Rust module organization

- `kernel::net::ipv6::output::Ip6Xmit` — socket-driven egress entry
- `kernel::net::ipv6::output::HeaderBuilder` — struct ipv6hdr from sock+flowi6
- `kernel::net::ipv6::output::ExthdrInject` — per-socket EH injection
- `kernel::net::ipv6::output::Ip6Output` — netfilter LOCAL_OUT + dispatch
- `kernel::net::ipv6::output::Ip6FinishOutput` — MTU check
- `kernel::net::ipv6::output::Ip6Fragment` — source-only fragmentation
- `kernel::net::ipv6::output::Ip6FinishOutput2` — neigh resolve + xmit
- `kernel::net::ipv6::output::Gso` — GSO/TSO segmentation
- `kernel::net::ipv6::output::McOutput` — multicast TX
- `kernel::net::ipv6::output::IdentGen` — Identification field generator

### Locking and concurrency

- **No new locks**: TX is per-skb; lock-state inherited from upper-layer (sock lock) + lower-layer (netdev queue lock)
- **Per-`dst_entry` cookie**: lockless rcu-side ref via `skb_dst_set_noref`
- **Per-netns ident counter**: per-CPU + per-destination atomic; `READ_ONCE`/`atomic64_inc_return`

### Error handling

- skb dropped via kfree_skb_reason (Ip6OutDiscards / Ip6FragFails)
- Counter bumped via `IP6_INC_STATS_BH(net, idev, …)`
- ICMP PTB / Dest Unreach generated for the right error classes
- Backlog overflow → drop + EAGAIN to upper layer

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| `ip6_xmit` skb header build (no overrun) | `kani::proofs::net::ipv6::output::xmit_safety` |
| Per-socket EH injection (skb_push bounds) | `kani::proofs::net::ipv6::output::exthdr_inject_safety` |
| `ip6_fragment` per-fragment offset arithmetic | `kani::proofs::net::ipv6::output::frag_safety` |
| Identification-field generator (no shared-state corruption) | `kani::proofs::net::ipv6::output::ident_safety` |
| `ip6_finish_output2` neigh-state-machine integration | `kani::proofs::net::ipv6::output::finish_safety` |

### Layer 2: TLA+ models

(none owned at this granularity; cross-ref `net/ipv6/ip6-input.md` for frag-reasm model and `net/ipv6/ndisc.md` for neighbour state)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-fragment Fragment Header chain | offsets are 8-byte-aligned + strictly increasing; final fragment has More=0 | `kani::proofs::net::ipv6::output::frag_header_invariants` |
| Per-socket EH options | `np->opt`'s computed `tot_len` matches Σ per-EH lengths | `kani::proofs::net::ipv6::output::opt_invariants` |
| Identification counter | per-(saddr, daddr) ID never repeats within `MAX_FRAG_REASM_TIMEOUT` (60s) | `kani::proofs::net::ipv6::output::ident_invariants` |

### Layer 4: Functional correctness (opt-in)

- **Source-only fragmentation theorem** via Verus — proves: ip6_fragment never fragments a packet whose nexthdr indicates an already-fragmented chain (no double-fragmentation).
- **Identification non-collision theorem** via Verus — proves: under per-(saddr, daddr) atomic+jiffies-mix, P(collision within 60s) ≤ 2^-32 for typical traffic rates.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **SIZE_OVERFLOW** | fragmentation offset + EH-length arithmetic uses checked operators | § Mandatory |
| **LATENT_ENTROPY** | Identification-field generator seeded from kernel CSPRNG (defense vs. ID-prediction) | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: per-skb infrastructure (cross-ref `net/skbuff.md`)
- **CONSTIFY**: per-EH-type ops vtables `static const`
- **SIZE_OVERFLOW**: see above
- **LATENT_ENTROPY**: see above

### Row-2 / GR-RBAC integration

- LSM hook `security_skb_classify_flow` for per-socket flow tagging
- LSM hook on netfilter LOCAL_OUT (via secmark)
- Default GR-RBAC policy: empty so behavior matches upstream

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Open Questions

(none — IPv6 TX path semantics are exhaustively specified by RFC 8200 + upstream)

## Out of Scope

- IPv6 RX path (cross-ref `net/ipv6/ip6-input.md`)
- IPv6 routing (cross-ref `net/ipv6/route.md`)
- Neighbour resolution (cross-ref `net/ipv6/ndisc.md`)
- GSO/TSO algorithm details (cross-ref `net/core/00-overview.md` GSO Tier-3)
- Flow-label management details (cross-ref `net/ipv6/flowlabel.md`)
- AH/ESP TX (cross-ref `net/xfrm/00-overview.md`)
- 32-bit-only paths
- Implementation code
