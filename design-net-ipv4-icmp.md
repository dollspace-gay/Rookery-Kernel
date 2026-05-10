---
title: "Tier-3: net/ipv4/icmp.c — ICMPv4 receive/send (echo + error reports + per-CPU socket pool + rate-limit)"
tags: ["tier-3", "net", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

`net/ipv4/icmp.c` is the ICMPv4 protocol implementation: per-skb ICMP RX (echo-request/_reply, dest-unreach, time-exceeded, redirect, frag-needed, etc.) + ICMP TX via per-CPU control socket pool. Per-protocol icmp_protocol registered as inet_proto for IPPROTO_ICMP=1. Per-error-type rate-limit (sysctl_icmp_ratemask + token bucket per-dst). icmp_send used by IP layer + drivers to emit error reports. icmp_socket per-CPU pool avoids socket-alloc overhead for fast emit. Critical for: ping, traceroute, PMTU discovery, error reporting throughout networking stack.

This Tier-3 covers `net/ipv4/icmp.c` (~1766 lines).

### Acceptance Criteria

- [ ] AC-1: ping 8.8.8.8: ECHO_REQUEST sent; ECHO_REPLY received; data+seq+id round-trip.
- [ ] AC-2: ICMP-redirect: per-route gw updated; subsequent flow uses new gw.
- [ ] AC-3: PMTU: FRAG_NEEDED received; dst.metrics[RTAX_MTU] reduced.
- [ ] AC-4: TTL-expired packet → ICMP_TIME_EXCEEDED sent back to source.
- [ ] AC-5: Port-unreachable: UDP packet to no-listener; ICMP_DEST_UNREACH sent.
- [ ] AC-6: Rate-limit: 1000 errors/sec stress; per-dst rate-limit kicks in.
- [ ] AC-7: Per-CPU socket pool: 32-CPU host emitting ICMP errors; per-CPU sock used.
- [ ] AC-8: sysctl_icmp_echo_ignore_all=1: ECHO_REQUEST silently dropped.
- [ ] AC-9: per-namespace ICMP: per-netns errors isolated.
- [ ] AC-10: linux test project icmp tests pass.

### Architecture

`Icmp::rcv(skb)`:
1. icmph := icmp_hdr(skb).
2. If !pskb_may_pull(skb, sizeof(*icmph)): drop.
3. csum_chk: validate icmp checksum.
4. If filter: drop per sysctl.
5. icmp_pointers[icmph.type].handler(skb).

`Icmp::echo(skb)`:
1. param := { type=ECHO_REPLY, code=0, ...}.
2. Compute reply: swap src/dst, recompute checksum.
3. icmp_reply(&param, skb).

`Icmp::send(skb, type, code, info)`:
1. Compute reply route.
2. icmp_global_allow → may rate-limit.
3. sk := icmp_xmit_lock(net).
4. __icmp_send: build ICMP error pkt + send via ip_build_and_send_pkt.
5. icmp_xmit_unlock(sk).

`Icmp::xmit_lock(net)`:
1. local_bh_disable.
2. cpu := smp_processor_id().
3. sk := icmp_sk(net, cpu).
4. spin_lock(&sk->sk_lock.slock) (or similar).
5. Return sk.

`Icmp::redirect(skb)`:
1. icmph := icmp_hdr(skb).
2. iph := original IP header in payload.
3. gw := icmp_redirect.gateway.
4. ipv4_redirect(&iph, sk, gw, ...).

### Out of Scope

- IPv6 ICMP (ICMPv6; covered separately)
- ping(8) / traceroute(8) (userspace concern)
- PMTU details (covered in `net/ipv4/route.md` Tier-3)
- IGMP (covered in `net/ipv4/igmp.md` future Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `icmp_protocol` | per-protocol vtable | `net::ipv4::icmp::ICMP_PROTOCOL` |
| `icmp_init()` | module init: register | `Icmp::init` |
| `icmp_rcv(skb)` | per-skb RX entry | `Icmp::rcv` |
| `icmp_echo(skb)` | per-skb echo-request reply | `Icmp::echo` |
| `icmp_redirect(skb)` | per-skb ICMP-redirect handler | `Icmp::redirect` |
| `icmp_unreach(skb)` | per-skb dest-unreach handler | `Icmp::unreach` |
| `icmp_time_exceed(skb)` | per-skb time-exceeded handler | `Icmp::time_exceed` |
| `icmp_send(skb, type, code, info)` | per-error emit | `Icmp::send` |
| `__icmp_send(skb, type, code, info, &opt)` | low-level emit | `Icmp::__send` |
| `icmp_xmit_lock(net)` | per-CPU socket lock | `Icmp::xmit_lock` |
| `icmp_xmit_unlock(sk)` | unlock | `Icmp::xmit_unlock` |
| `icmpv4_global_allow(net, type, code)` | global rate-limit | `Icmp::global_allow` |
| `icmp_global_allow(sk, dst)` | per-dst rate-limit | `Icmp::global_allow_dst` |
| `icmp_socket_table[NR_CPUS]` | per-CPU control socket | `Icmp::SOCKET_TABLE` |
| `icmp_push_reply(...)` | per-reply push | `Icmp::push_reply` |
| `icmp_route_lookup(...)` | per-reply route | `Icmp::route_lookup` |
| `__icmp_socket_table_alloc()` | per-CPU socket alloc | `Icmp::__socket_alloc` |

### compatibility contract

REQ-1: ICMP types (RFC792):
- 0 ECHO_REPLY.
- 3 DEST_UNREACH (codes 0..15: NET/HOST/PROT/PORT_UNREACH, FRAG_NEEDED, SR_FAILED, etc.).
- 4 SOURCE_QUENCH (deprecated).
- 5 REDIRECT.
- 8 ECHO_REQUEST.
- 9/10 RTR_ADV/RTR_SOLIC.
- 11 TIME_EXCEEDED (codes 0/1: TTL/FRAG_REASSEMBLY).
- 12 PARAM_PROB.
- 13/14 TIMESTAMP/REPLY.
- 15/16 INFO_REQUEST/REPLY (deprecated).
- 17/18 ADDRESS_REQUEST/REPLY (deprecated).

REQ-2: Per-skb `icmp_rcv` flow:
1. icmp_filter(skb) — filter sysctl_icmp_echo_ignore_all + sysctl_icmp_echo_ignore_broadcasts.
2. Validate ICMP header + checksum.
3. icmp_pointers[type].handler(skb) — per-type dispatch.

REQ-3: Per-type handler table (`icmp_pointers[]`):
- ECHO_REQUEST → icmp_echo.
- ECHO_REPLY → ping_rcv (per-RAW-socket).
- REDIRECT → icmp_redirect.
- DEST_UNREACH / TIME_EXCEEDED / SOURCE_QUENCH → icmp_unreach (notify upper-layer-sock).
- PARAM_PROB → icmp_unreach (similar).

REQ-4: `icmp_echo(skb)` flow:
1. Validate ECHO_REQUEST per sysctl_icmp_echo_ignore_*.
2. Construct ECHO_REPLY:
   - type = ECHO_REPLY.
   - swap source ↔ dest addrs.
   - copy data + identifier + sequence.
3. icmp_push_reply(skb, &icmp_param, ...) → IP layer.

REQ-5: `icmp_send` (error reporting from IP layer):
1. orig_skb := triggering skb.
2. type / code / info := error specifier.
3. Compose ICMP error: header + first 8 bytes of orig_skb.
4. ip_route_output for reply route.
5. ip_build_and_send_pkt.

REQ-6: Per-CPU control socket pool:
- icmp_socket_table[NR_CPUS]: per-CPU kernel-socket.
- icmp_xmit_lock acquires; icmp_xmit_unlock releases.
- Avoids per-emit socket alloc.

REQ-7: Per-error rate-limit:
- sysctl_icmp_ratemask: bitmap of types subject to rate-limit.
- icmpv4_global_allow: token bucket per-net.
- icmp_global_allow_dst: per-dst rate-limit.

REQ-8: PMTU update via FRAG_NEEDED:
- Per-FRAG_NEEDED: extract Next-Hop MTU from icmphdr.
- ipv4_update_pmtu(...) updates dst.metrics[RTAX_MTU].

REQ-9: ICMP-redirect security:
- Per-net sysctl_icmp_redirects gating.
- Per-redirect-source must be on-link gateway.
- Defense against malicious-router redirecting traffic.

REQ-10: ECHO_REPLY (received):
- Delivered to RAW socket if any (ping(8) uses).
- ICMP-error correlation: lookup by id+seq.

REQ-11: ICMP-error → upper-layer notify:
- icmp_unreach → __ip_error → per-protocol handler->err_handler.
- TCP / UDP / etc. update sk->sk_err for sock-level error report.

REQ-12: Per-namespace socket:
- Per-net: icmp_sk(net, cpu).
- Per-namespace error counts.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `icmp_type_bounded` | OOB | per-type idx ∈ [0, NR_ICMP_TYPES]; defense against handler-table OOB. |
| `icmp_csum_validated` | INVARIANT | per-skb csum verified before handler dispatch. |
| `xmit_lock_balanced` | INVARIANT | per-CPU icmp_xmit_lock + _unlock paired. |
| `rate_limit_token_bounded` | INVARIANT | per-dst rate-limit token bucket bounded. |
| `redirect_source_validated` | INVARIANT | ICMP-redirect from on-link gw only. |

### Layer 2: TLA+

`net/ipv4/icmp_state.tla`:
- Per-skb state ∈ {Received, Filtered, Validated, Dispatched, Errored}.
- Properties:
  - `safety_csum_invalid_no_dispatch` — invalid csum drops; no handler call.
  - `safety_filter_drops_silently` — per-sysctl filter drops without reply.
  - `liveness_dispatched_eventually_handled` — dispatched skb invokes handler.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Icmp::rcv` post: skb dispatched OR dropped per sysctl/filter | `Icmp::rcv` |
| `Icmp::echo` post: ECHO_REPLY constructed + sent | `Icmp::echo` |
| `Icmp::send` post: per-CPU sock used; ICMP error packet emitted | `Icmp::send` |
| Per-CPU per-net icmp_sk distinct | invariants on socket pool |

### Layer 4: Verus/Creusot functional

`Per-skb ICMP processing matches RFC792 + RFC4884 + Linux extensions` semantic equivalence: per-type handler invocation matches RFC.

### hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

icmp-specific reinforcement:

- **Per-CPU socket pool** — defense against socket-alloc overhead causing PPS bottleneck.
- **Per-error rate-limit** — defense against ICMP-flood amplification.
- **Per-redirect source validation** — defense against malicious-redirect attack.
- **Per-PMTU lock-rate-limit** — defense against rapid-PMTU-update flapping.
- **Per-skb csum validation** — defense against accepting corrupted ICMP.
- **sysctl_icmp_echo_ignore_*** — defense against ICMP-echo-amplification.
- **Per-namespace isolation** — defense against cross-netns ICMP leak.
- **Per-error per-dst dst_link_failure** — defense against silent-error.
- **icmpv4_global_allow per-net** — defense against global ICMP-storm.
- **Per-skb pull bounded before icmphdr access** — defense against truncated packet OOB.

