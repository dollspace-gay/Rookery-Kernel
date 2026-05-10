---
title: "Tier-3: net/ipv4/raw.c — IPv4 RAW socket (per-protocol IP-header passthrough + per-skb hash dispatch + IP_HDRINCL)"
tags: ["tier-3", "net", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

`net/ipv4/raw.c` is the IPv4 RAW socket implementation — userspace `socket(AF_INET, SOCK_RAW, IPPROTO_FOO)` creates a per-protocol RAW socket that receives every per-skb matching the protocol number + can transmit arbitrary IPv4 packets. Per-RX skb: raw_local_deliver searches per-protocol raw_v4_hashinfo for matching socks; per-match clones skb to deliver. Per-TX: optional IP_HDRINCL lets userspace supply IP header directly (otherwise kernel constructs). Critical for: ping(8), traceroute(8), dhcp-client, custom-protocol implementations, network-monitoring tools.

This Tier-3 covers `net/ipv4/raw.c` (~1126 lines).

### Acceptance Criteria

- [ ] AC-1: socket(AF_INET, SOCK_RAW, IPPROTO_ICMP) (CAP_NET_RAW): creates RAW socket.
- [ ] AC-2: ICMP echo via RAW: per-skb received via raw_local_deliver.
- [ ] AC-3: IP_HDRINCL: userspace constructs IP header; transmits as-is.
- [ ] AC-4: Per-protocol multi-RAW: 2 socks for same protocol; both receive copy.
- [ ] AC-5: Per-sock BPF filter: SO_ATTACH_FILTER; non-matching skbs dropped.
- [ ] AC-6: ICMP error → sk_err: traceroute-style use case.
- [ ] AC-7: Per-namespace: 2 netns with distinct RAW socks.
- [ ] AC-8: ss(8) + ss --raw: shows RAW socket state.
- [ ] AC-9: 100 RAW socks stress: per-bucket hash walk performance acceptable.
- [ ] AC-10: linux test project ipv4-raw tests pass.

### Architecture

`RawSock`:

```
struct RawSock {
  inet: InetSock,
  filter: KArc<SkFilter>,
  ipproto: u8,
  hdrincl: bool,
}
```

`Raw::init()` (module-level):
1. raw_v4_hashinfo := init_hashinfo.
2. inet_register_protosw(&raw_inet_protosw).

`Raw::local_deliver(skb, protocol)`:
1. read_lock(&raw_v4_hashinfo.lock).
2. hash := raw_hashfunc(net, protocol).
3. sk_iter := raw_v4_hashinfo.ht[hash].
4. While sk_iter:
   - If raw_v4_match(net, sk_iter, ...):
     - clone := skb_clone(skb).
     - raw_rcv(sk_iter, clone).
   - sk_iter = sk_iter->next.
5. read_unlock.

`Raw::rcv(sk, skb)`:
1. If sk_filter(sk, skb): drop.
2. ipv4_pktinfo_prepare(sk, skb, ...).
3. ret := sock_queue_rcv_skb(sk, skb).
4. If ret: drop.
5. Return 0.

`Raw::sendmsg(sk, msg, len)`:
1. Lock sock.
2. If sk->hdrincl: raw_send_hdrinc(sk, &fl4, msg, len, &rt, flags, &sockc).
3. Else:
   - ip_route_output_flow → rt.
   - sock_alloc_send_skb.
   - Construct IP header.
   - Copy data.
   - ip_send_skb.
4. release_sock.

`Raw::send_hdrinc(sk, &fl4, msg, len, &rt, flags, &sockc)`:
1. Allocate skb.
2. memcpy_from_msg(skb_put(skb, len), msg).
3. iph := ip_hdr(skb).
4. iph.frag_off = htons(IP_DF) (or per-options).
5. iph.id = htons(...).
6. ip_local_out(skb).

### Out of Scope

- IPv6 RAW (covered separately)
- BPF socket filter (covered in `kernel/bpf/bpf-core.md` Tier-3)
- Netfilter (covered in `net/netfilter.md` Tier-2)
- struct sock (covered in `net/struct-sock.md` Tier-2)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct raw_sock` | per-RAW-sock | `net::ipv4::raw::RawSock` |
| `raw_v4_hashinfo` | global per-protocol hash | `Raw::V4_HASHINFO` |
| `raw_init()` | module init | `Raw::init` |
| `raw_v4_match(net, sk, num, raddr, laddr, dif, sdif)` | per-sock match | `Raw::v4_match` |
| `raw_local_deliver(skb, protocol)` | per-skb deliver to matching socks | `Raw::local_deliver` |
| `raw_v4_input(net, skb, &iph, hash)` | per-protocol RX | `Raw::v4_input` |
| `raw_rcv(sk, skb)` | per-sock receive | `Raw::rcv` |
| `raw_sendmsg(sk, msg, len)` | per-sock send | `Raw::sendmsg` |
| `raw_recvmsg(sk, msg, len, flags, &addr_len)` | per-sock recv | `Raw::recvmsg` |
| `raw_send_hdrinc(sk, &fl4, msg, length, &rt, flags, &sockc)` | IP_HDRINCL send | `Raw::send_hdrinc` |
| `raw_bind(sk, &addr, addr_len)` | sys_bind impl | `Raw::bind` |
| `raw_close(sk, timeout)` | sys_close impl | `Raw::close` |
| `raw_setsockopt(sk, level, optname, optval, optlen)` | per-RAW setsockopt | `Raw::setsockopt` |
| `raw_getsockopt(sk, level, optname, optval, optlen)` | per-RAW getsockopt | `Raw::getsockopt` |
| `raw_v4_err(sk, skb, info)` | per-sock error report | `Raw::v4_err` |
| `raw_seq_*` (raw_diag.c) | /proc/net/raw seq-file | `RawDiag::seq_*` |
| `raw_v4_lookup(net, sk, num, daddr, dif, sdif)` | per-sock direct lookup | `Raw::v4_lookup` |

### compatibility contract

REQ-1: Per-sock `raw_sock`:
- `inet` (struct inet_sock; embeds struct sock).
- `protocol` (per-RAW IPPROTO_* number).
- `filter` (BPF socket-filter; per-skb).
- `ipproto` (alias of protocol).

REQ-2: Per-protocol `raw_v4_hashinfo`:
- `lock` (rwlock).
- `ht[RAW_HTABLE_SIZE]` (256-bucket hash by protocol).
- Per-bucket: list of raw socks for that protocol.

REQ-3: Per-sock `socket(AF_INET, SOCK_RAW, IPPROTO_FOO)` flow:
1. inet_create_sock with type=SOCK_RAW.
2. Per-RAW prot (raw_prot) bound; per-AF inet_sockraw_ops.
3. Per-protocol added to raw_v4_hashinfo.

REQ-4: Per-skb `raw_local_deliver` (called from ip_local_deliver_finish):
1. hash := raw_hashfunc(net, protocol).
2. read_lock(&raw_v4_hashinfo.lock).
3. sk := __raw_v4_lookup(...) per-bucket walk.
4. If sk: skb_clone + raw_rcv(sk, clone).
5. Continue per-bucket walk to deliver to all matching socks.
6. read_unlock.

REQ-5: Per-sock `raw_rcv(sk, skb)`:
1. If sk_filter(sk, skb): drop.
2. ipv4_pktinfo_prepare_errqueue (per-skb info preserved).
3. sock_queue_rcv_skb(sk, skb).
4. sk->sk_data_ready(sk).

REQ-6: Per-sock `raw_sendmsg`:
1. Validate per-RAW protocol.
2. Allocate skb via sock_alloc_send_skb.
3. Compose IP header (per IP_HDRINCL):
   - If IP_HDRINCL: copy from userspace as-is.
   - Else: kernel constructs IP header.
4. ip_send_skb / ip_local_out for transmit.

REQ-7: IP_HDRINCL semantics:
- Per-sock setsockopt(IPPROTO_IP, IP_HDRINCL, 1).
- Subsequent send: userspace supplies IP header.
- Useful for: custom IP options, source-routing, traceroute.

REQ-8: Per-RAW capability check:
- AF_INET + SOCK_RAW requires CAP_NET_RAW.
- Defense against unauthorized RAW socket creation.

REQ-9: Per-skb match (`raw_v4_match`):
- Match: sock.protocol == skb.protocol.
- Match: sock.sk_bound_dev_if matches if set.
- Match: sock.local_addr matches dst-addr if bound.
- Match: sock.remote_addr matches src-addr if connected.

REQ-10: Per-sock BPF filter:
- Per-RAW socket may install BPF filter via SO_ATTACH_FILTER.
- Per-skb evaluated; non-matching dropped.

REQ-11: ICMP error handling (`raw_v4_err`):
- Per-skb error → per-sock sk_err.
- sk_error_report dispatches.

REQ-12: /proc/net/raw:
- raw_diag exposes per-sock state.
- Used by ss(8), netstat.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `raw_hashinfo_per_protocol_isolated` | INVARIANT | per-protocol hash bucket distinct. |
| `raw_v4_match_correct` | INVARIANT | per-sock match matches all RFC793-style fields. |
| `cap_net_raw_required` | INVARIANT | RAW socket creation requires CAP_NET_RAW. |
| `ip_hdrincl_validates_iph` | INVARIANT | IP_HDRINCL validates IP-header fields before transmit. |
| `per_skb_filter_pre_deliver` | INVARIANT | sk_filter evaluated before sock_queue_rcv_skb. |

### Layer 2: TLA+

`net/ipv4/raw_lifecycle.tla`:
- Per-sock state ∈ {Created, Bound, Listening, Connected, Closing}.
- Properties:
  - `safety_per_protocol_dispatch` — per-skb dispatched to all matching socks.
  - `safety_per_sock_filter_evaluated` — per-sock BPF eval before queue.
  - `liveness_received_eventually_dispatched` — every received-skb eventually dispatched or dropped.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Raw::local_deliver` post: per-matching-sock clone delivered | `Raw::local_deliver` |
| `Raw::rcv` post: skb queued OR filtered | `Raw::rcv` |
| `Raw::sendmsg` post: skb transmitted via IP layer | `Raw::sendmsg` |
| Per-sock raw_v4_hashinfo membership consistent across bind/close | invariants on bind |

### Layer 4: Verus/Creusot functional

`Per-RAW-skb: every matching sock receives a clone; per-IP_HDRINCL TX preserves user-supplied header bits` semantic equivalence: per-skb the dispatch + transmit semantics match Linux RAW socket spec.

### hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

raw-specific reinforcement:

- **CAP_NET_RAW required** — defense against unauthorized RAW socket creation.
- **Per-RAW hash lookup RCU** — defense against close-during-deliver UAF.
- **Per-skb filter pre-queue** — defense against unauthorized data-receive.
- **IP_HDRINCL bounds checked** — defense against userspace-supplied malformed header.
- **Per-protocol hash bucket isolated** — defense against cross-protocol leak.
- **Per-namespace isolation** — defense against cross-netns RAW leak.
- **Per-sock receive-buffer bounded** — defense against unbounded queue OOM.
- **ICMP error → sk_err atomic** — defense against torn error report.
- **Multiple-deliver via skb_clone** — defense against shared-skb modification.
- **/proc/net/raw permission gated** — defense against ss-via-unprivileged sock list.

