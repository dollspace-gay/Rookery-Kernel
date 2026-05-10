---
title: "Tier-3: net/sctp/socket.c — SCTP (Stream Control Transmission Protocol) socket family"
tags: ["tier-3", "net", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

SCTP (RFC 4960) is a reliable, message-oriented transport protocol with **multi-streaming** (per-association multiple independent ordered/unordered streams) and **multi-homing** (per-association multiple peer IP addresses). Per-socket types: SOCK_SEQPACKET (one-to-many; UDP-style multi-association), SOCK_STREAM (one-to-one; TCP-style single-association). Per-association has primary path + alternative paths; per-path heartbeat detects failure → failover. Per-msg has stream-id + payload-protocol-id + ttl + ordered/unordered. Per-chunk types: DATA, INIT, SACK, HEARTBEAT, etc. Critical for: telecom signaling (SS7-over-IP, Diameter, M3UA), H.248, peer-to-peer with HA.

This Tier-3 covers `socket.c` (~9720 lines).

### Acceptance Criteria

- [ ] AC-1: socket(AF_INET, SOCK_STREAM, IPPROTO_SCTP): succeeds.
- [ ] AC-2: bindx with 2 IPs: per-endpoint multi-homed.
- [ ] AC-3: connect: 4-way INIT/INIT-ACK/COOKIE-ECHO/COOKIE-ACK exchanges.
- [ ] AC-4: sendmsg with sinfo_stream=3: per-stream-3 in-order delivery.
- [ ] AC-5: Per-primary path failure (heartbeat timeout): failover to alternate.
- [ ] AC-6: Per-msg fragmentation: DATA chunks reassembled.
- [ ] AC-7: SACK per-association: gap-ack-blocks for selective retransmit.
- [ ] AC-8: SCTP_NOTIFICATION SCTP_ASSOC_CHANGE delivered on assoc state-change.
- [ ] AC-9: SOCK_SEQPACKET 1-to-many: per-msg-recv has sndinfo with assoc_id.
- [ ] AC-10: shutdown(): SHUTDOWN-COMPLETE handshake.

### Architecture

Per-socket state:

```
struct SctpSock {
  inet: InetSock,                                 // base AF_INET
  ep: *SctpEndpoint,
  pf: *SctpPf,                                    // per-AF (v4/v6)
  type_: u8,                                      // SOCK_SEQPACKET / SOCK_STREAM
  initmsg: SctpInitmsg,
  paddrparam: SctpPaddrparams,
  rtoinfo: SctpRtoinfo,
  assocparams: SctpAssocparams,
  default_*: ...,
  ...
}

struct SctpEndpoint {
  base: SctpEndpointCommon,
  bind_addr: SctpBindAddr,                       // multi-IP local addresses
  asocs: ListHead<SctpAssociation>,
  ...
}

struct SctpAssociation {
  base: SctpEndpointCommon,
  ep: *SctpEndpoint,
  peer: SctpPeer,                                // multi-IP peer
  primary_path: *SctpTransport,
  transports: ListHead<SctpTransport>,
  state: SctpState,
  ...
}

struct SctpTransport {
  asoc_link: ListLink,
  asoc: *SctpAssociation,
  ipaddr: IpAddr,
  pmtu: u32,
  rtt: u32,
  hb_timer: TimerList,
  state: SctpTransportState,
  ...
}
```

`Sctp::init_sock(sk) -> Result<()>`:
1. sp = (SctpSock*)sk.
2. sp.ep = sctp_endpoint_init(sk).
3. sp.initmsg.sinit_num_ostreams = SCTP_DEFAULT_OUTSTREAMS (10).
4. sp.initmsg.sinit_max_instreams = SCTP_DEFAULT_INSTREAMS (10).

`Sctp::bind(sk, addr, alen) -> Result<()>`:
1. sp = (SctpSock*)sk.
2. err = sctp_do_bind(sk, addr, alen).
3. /* validate addr; insert into endpoint's bind_addr */

`Sctp::sendmsg(sk, msg, len) -> Result<usize>`:
1. sp = (SctpSock*)sk.
2. assoc = lookup-or-create-association.
3. /* parse cmsg for sinfo */
4. Per-DATA chunk build.
5. sctp_outq_tail(assoc.outqueue, chunk).
6. /* Trigger send via output.c */

`Sctp::accept(sock, newsock, flags, kern) -> Result<()>`:
1. /* SOCK_STREAM only */
2. assoc = pop-from-pending-assoc-queue.
3. newsk = sctp_sock_migrate(sk, newsock, assoc).

`SctpAssoc::new(ep, sk, gfp) -> *SctpAssociation`:
1. asoc = sctp_association_new(ep, sk, scope, gfp).
2. /* state machine starts in CLOSED */

`SctpTransport::create(asoc, peer_addr, gfp) -> *SctpTransport`:
1. t = sctp_transport_new(net, peer_addr, gfp).
2. list_add(&t.asoc_link, &asoc.transports).

### Out of Scope

- net/sctp/{associola, transport, output, input, sm_*}.c (covered separately if expanded)
- DTLS-over-SCTP (separate)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct sctp_sock` | per-socket state | `SctpSock` |
| `struct sctp_endpoint` | per-(local-IP-set, port) endpoint | `SctpEndpoint` |
| `struct sctp_association` | per-conn association | `SctpAssociation` |
| `struct sctp_transport` | per-(association, peer-IP) path | `SctpTransport` |
| `sctp_init_sock()` | per-socket init | `Sctp::init_sock` |
| `sctp_close()` | close() | `Sctp::close` |
| `sctp_bind()` | bind | `Sctp::bind` |
| `sctp_bindx()` | multi-IP bind | `Sctp::bindx` |
| `sctp_connect()` | per-stream connect | `Sctp::connect` |
| `sctp_connectx()` | multi-IP connect | `Sctp::connectx` |
| `sctp_listen()` | per-stream listen | `Sctp::listen` |
| `sctp_accept()` | per-stream accept | `Sctp::accept` |
| `sctp_sendmsg()` | per-msg send | `Sctp::sendmsg` |
| `sctp_recvmsg()` | per-msg recv | `Sctp::recvmsg` |
| `sctp_setsockopt()` / `getsockopt()` | per-SCTP_* | `Sctp::setsockopt` / `getsockopt` |
| `sctp_endpoint_init()` | per-endpoint init | `SctpEndpoint::init` |
| `sctp_association_new()` | per-assoc init | `SctpAssoc::new` |
| `sctp_transport_route()` | per-path routing | `SctpTransport::route` |
| `sctp_assoc_lookup_by_addr_paddr()` | per-(local, peer) lookup | `Sctp::assoc_lookup` |
| `SCTP_INIT` / `SCTP_DATA` / `SCTP_SACK` / `SCTP_HEARTBEAT` / etc. | chunk types | UAPI |
| `SCTP_INITMSG` / `SCTP_PEER_ADDR_PARAMS` / etc. | sockopts | UAPI |

### compatibility contract

REQ-1: socket(AF_INET, SOCK_STREAM, IPPROTO_SCTP) or SOCK_SEQPACKET:
- Per-init: alloc sctp_sock; per-endpoint registered; default stream count.
- Per-SOCK_SEQPACKET: 1-to-many (multi-association in single socket).
- Per-SOCK_STREAM: 1-to-1 (TCP-style).

REQ-2: bind / bindx:
- Per-(local-IP, port).
- bindx: SCTP_BINDX_ADD_ADDR / REM_ADDR for multi-homing.

REQ-3: connect / connectx:
- per-(peer-IP-list, port): start INIT-handshake.
- 4-way handshake: INIT → INIT-ACK (with cookie) → COOKIE-ECHO → COOKIE-ACK.

REQ-4: Per-association:
- 1+ transports (per-peer-IP).
- Per-stream count negotiated during INIT.
- Per-association sequence numbers + cumulative-TSN-ack.
- Per-PMTU-discovery per-transport.
- Per-heartbeat interval per-transport.

REQ-5: Per-multi-streaming:
- Per-stream-id distinct sequencing.
- Stream-2 head-of-line block doesn't stall stream-7.
- Per-msg setsockopt SCTP_SNDINFO.sinfo_stream.

REQ-6: Per-multi-homing:
- Per-association up to N transports (peer-IP-list).
- Per-PRIMARY transport selected at INIT.
- Per-heartbeat-fail: failover to alternate.

REQ-7: Per-msg framing:
- Per-msg may span multiple DATA chunks (fragmentation).
- Per-DATA-chunk has TSN, stream-id, ssn, payload-protocol-id, fragment-flags.

REQ-8: SACK (Selective ACK):
- Per-association: cumulative ack + gap-ack-blocks.

REQ-9: Per-message-options sockopts:
- SCTP_INITMSG: per-association init params (out-streams, in-streams, max-init-attempts).
- SCTP_PEER_ADDR_PARAMS: per-transport heartbeat params.
- SCTP_RTOINFO: RTO timing.
- SCTP_ASSOCINFO: per-association params.

REQ-10: Per-namespace:
- Per-net SCTP instance with per-port hash.

REQ-11: Per-CMSG:
- SCTP_SNDINFO: per-msg stream/proto-id/ttl/flags.
- SCTP_RCVINFO: per-recv-msg stream/proto-id/ssn/tsn.
- SCTP_NXTINFO: peek-ahead.

REQ-12: Per-NOTIFY events:
- SCTP_ASSOC_CHANGE: per-assoc state change.
- SCTP_PEER_ADDR_CHANGE: per-transport state.
- SCTP_REMOTE_ERROR: per-error-chunk.
- SCTP_SEND_FAILED: per-msg-send fail.
- SCTP_SHUTDOWN_EVENT: graceful shutdown.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `assoc_state_in_set` | INVARIANT | per-assoc.state ∈ SCTP_STATE_*. |
| `transport_in_assoc_list` | INVARIANT | per-transport.asoc_link ∈ asoc.transports. |
| `bind_addr_unique` | INVARIANT | per-(local-IP, port) bound at most once. |
| `streams_within_negotiated` | INVARIANT | per-msg sinfo_stream < negotiated stream-count. |
| `tsn_monotonic_per_assoc` | INVARIANT | per-DATA chunk TSN strictly-increasing per-association. |

### Layer 2: TLA+

`net/sctp/sctp.tla`:
- Per-association 4-way INIT handshake + per-stream send/recv + per-multi-homing failover.
- Properties:
  - `safety_4way_handshake_required` — per-DATA: only after COOKIE-ACK.
  - `safety_per_stream_in_order` — per-stream-N: SSN delivered in-order.
  - `safety_failover_to_alternate` — per-primary heartbeat-fail ⟹ alternate-primary selected.
  - `liveness_per_msg_eventually_acked` — per-DATA + alive-path: SACK eventually received.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Sctp::init_sock` post: ep allocated; default stream count | `Sctp::init_sock` |
| `Sctp::bind` post: addr in ep.bind_addr | `Sctp::bind` |
| `Sctp::sendmsg` post: per-msg fragmented if > path-MTU; chunks queued | `Sctp::sendmsg` |
| `SctpAssoc::new` post: assoc in CLOSED state; primary_path NULL | `SctpAssoc::new` |
| `SctpTransport::create` post: transport in asoc.transports | `SctpTransport::create` |

### Layer 4: Verus/Creusot functional

`Per-association RFC 4960 4-way INIT + multi-streaming + multi-homing failover` semantic equivalence: per-RFC 4960 SCTP base.

### hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

SCTP-specific reinforcement:

- **Per-COOKIE-ECHO HMAC validation** — defense against per-cookie-spoof.
- **Per-INIT min-cookie-life** — defense against per-replay-attack.
- **Per-association count cap** — defense against per-init flood DoS.
- **Per-transport heartbeat timeout bounded** — defense against per-stuck-path silent.
- **Per-stream count negotiated capped** — defense against per-stream-bomb resource exhaustion.
- **Per-msg fragmentation respects path-MTU** — defense against per-frag IP-fragment DoS.
- **Per-CAP_NET_RAW for some sockopts** — defense against per-unprivileged manipulation.
- **Per-namespace SCTP scoped** — defense against cross-ns leakage.
- **Per-RCU-free of associations** — defense against per-walk UAF.
- **Per-cmsg validated** — defense against per-malformed-cmsg.
- **Per-transport state-machine bounded** — defense against per-state-loop.

