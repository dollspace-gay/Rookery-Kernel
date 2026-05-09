---
title: "Tier-3: net/netfilter/conntrack-proto — per-protocol conntrack state machines"
tags: ["design-doc", "tier-3", "net", "netfilter"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for the per-protocol conntrack state machines: TCP (RFC 793 + 6298 + 5681 + 7414), UDP (stateless 2-direction), ICMP (request/reply pair tracking), ICMPv6, SCTP (per RFC 4960 + 9260), GRE (key-based pseudo-flow tracking), and generic (proto-agnostic 2-direction). Each protocol module registers a `nf_conntrack_l4proto` vtable; the conntrack core dispatches per-packet via `l4proto->packet`.

The TCP state machine — most complex — tracks SYN/SYN-ACK/ESTABLISHED/FIN_WAIT/CLOSE_WAIT/LAST_ACK/TIME_WAIT/CLOSED + window/seq/ACK validation per RFC. Handles edge cases (out-of-window segments, zero-window probing, Wscale option, SACK negotiation, retransmits). Defines the strictness vs. permissiveness behavior that `nf_conntrack_tcp_be_liberal` sysctl tunes.

Owns the mandatory `models/net/conntrack_state_machine.tla` TLA+ model declared by `net/netfilter/00-overview.md`.

Sub-tier-3 of `net/netfilter/00-overview.md`. Pairs with `net/netfilter/conntrack-core.md` (consumer; lifecycle + storage), various conntrack helpers (consumers of per-protocol state).

### Requirements

- REQ-1: `struct nf_conntrack_l4proto` vtable layout-equivalent for first cache-line.
- REQ-2: TCP state machine: 12 states (NONE → SYN_SENT → SYN_RECV → ESTABLISHED → FIN_WAIT → CLOSE_WAIT → LAST_ACK → TIME_WAIT → CLOSE + SYN_SENT2 + RETRANS + UNACK) per RFC 793 + 6298 + 5961.
- REQ-3: TCP per-direction window + seq + ACK tracking via `td_end / td_maxend / td_maxwin / td_scale / td_flags`; per-RFC 5961 validation rules.
- REQ-4: TCP per-state timeouts byte-identical to upstream defaults; sysctl-tunable per-state.
- REQ-5: `nf_conntrack_tcp_be_liberal` sysctl: 0 strict, 1 liberal; identical effect on out-of-window handling.
- REQ-6: UDP stateless tracking: `IPS_SEEN_REPLY`-driven; unreplied vs. replied timeouts.
- REQ-7: ICMP / ICMPv6 request/reply pair: tuple matching identical; non-request/reply types tagged IP_CT_RELATED.
- REQ-8: SCTP state machine per RFC 4960 + 9260; per-direction `vtag` tracking; per-state timeouts.
- REQ-9: GRE key-based tracking for PPTP + L2TPv3-GRE; explicit registration from helpers.
- REQ-10: Generic per-proto module for unknown protos; 600s default timeout.
- REQ-11: Per-proto error checking (`error` callback): rejects malformed packets pre-state-machine.
- REQ-12: TLA+ model `models/net/conntrack_state_machine.tla` (mandatory per `net/netfilter/00-overview.md` Layer 2) — proves: TCP state machine refines RFC 793 + 6298 abstract spec; transitions are sound under concurrent RX/TX; UDP/ICMP/SCTP/GRE state machines have no stuck states.
- REQ-13: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: `pahole struct nf_conntrack_l4proto` byte-identical first cache-line. (covers REQ-1)
- [ ] AC-2: TCP handshake test: SYN → SYN-ACK → ACK → conntrack transitions NONE → SYN_SENT → SYN_RECV → ESTABLISHED; verifiable via `conntrack -L | grep tcp`. (covers REQ-2)
- [ ] AC-3: TCP teardown test: FIN-ACK + ACK + FIN-ACK + ACK → conntrack ends at TIME_WAIT for 120s then reaped. (covers REQ-2, REQ-4)
- [ ] AC-4: Out-of-window test (strict mode): inject TCP segment with seq beyond receiver's window; conntrack marks INVALID; nft INVALID rule drops. (covers REQ-3, REQ-5)
- [ ] AC-5: Out-of-window test (liberal mode): same scenario with `tcp_be_liberal=1`; segment accepted; conntrack state unchanged. (covers REQ-5)
- [ ] AC-6: UDP tracking test: send UDP from A to B; conntrack created; reply from B → IPS_SEEN_REPLY set + timeout extended. (covers REQ-6)
- [ ] AC-7: ICMP echo test: ping flow → echo-request/echo-reply pair conntracked; `ICMP TYPE_DEST_UNREACH` to flow → tagged IP_CT_RELATED. (covers REQ-7)
- [ ] AC-8: SCTP test: 4-way handshake (INIT, INIT-ACK, COOKIE-ECHO, COOKIE-ACK) → ESTABLISHED state; per-direction vtags tracked. (covers REQ-8)
- [ ] AC-9: PPTP test: load `nf_conntrack_pptp`; PPTP control connection → expectation registered; subsequent GRE flow with matching key → child conntrack. (covers REQ-9)
- [ ] AC-10: Generic proto test: send IPPROTO_OSPF (89) packet; generic conntrack created; default 600s timeout. (covers REQ-10)
- [ ] AC-11: Malformed-pkt test: TCP packet with bogus header (e.g., data offset < 5) → `error` callback rejects; conntrack not created. (covers REQ-11)
- [ ] AC-12: TLA+ `models/net/conntrack_state_machine.tla` passes; refinement of RFC TCP state machine verified. (covers REQ-12)
- [ ] AC-13: Hardening section present and follows template. (covers REQ-13)

### Architecture

### Rust module organization

- `kernel::net::netfilter::conntrack::proto::Registry` — per-l4proto registration
- `kernel::net::netfilter::conntrack::proto::L4Proto` — `struct nf_conntrack_l4proto` wrapper
- `kernel::net::netfilter::conntrack::proto::tcp::Tcp` — TCP state machine
- `kernel::net::netfilter::conntrack::proto::tcp::Window` — per-direction window/seq/ACK tracking
- `kernel::net::netfilter::conntrack::proto::tcp::Rfc5961` — RFC 5961 mitigation logic
- `kernel::net::netfilter::conntrack::proto::udp::Udp` — UDP stateless
- `kernel::net::netfilter::conntrack::proto::icmp::Icmp`, `Icmpv6` — request/reply pair
- `kernel::net::netfilter::conntrack::proto::sctp::Sctp` — SCTP state machine + vtag tracking
- `kernel::net::netfilter::conntrack::proto::gre::Gre` — GRE key-based
- `kernel::net::netfilter::conntrack::proto::generic::Generic` — proto-agnostic
- `kernel::net::netfilter::conntrack::proto::Sysctl` — per-proto timeout sysctls

### Locking and concurrency

- **Per-`nf_conn` `lock`** (spinlock, inherited from conntrack-core): held during state transition + window update
- **No new global locks**

### Error handling

- `Err(EINVAL)` — malformed packet (e.g., bad TCP header length)
- `Err(EPROTONOSUPPORT)` — unknown l4proto (handled via generic module)
- `Err(EBADMSG)` — out-of-window TCP in strict mode
- skb dropped via kfree_skb_reason for telemetry

### Out of Scope

- Conntrack core lifecycle + storage (cross-ref `net/netfilter/conntrack-core.md`)
- Helpers (cross-ref `net/netfilter/conntrack-helper.md`)
- 32-bit-only paths
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Per-proto registration framework | `net/netfilter/nf_conntrack_proto.c` |
| TCP state machine (RFC 793/6298/5681/7414) | `net/netfilter/nf_conntrack_proto_tcp.c` |
| UDP (stateless 2-direction) | `net/netfilter/nf_conntrack_proto_udp.c` |
| ICMP request/reply | `net/netfilter/nf_conntrack_proto_icmp.c` |
| ICMPv6 | `net/netfilter/nf_conntrack_proto_icmpv6.c` |
| SCTP (RFC 4960/9260) | `net/netfilter/nf_conntrack_proto_sctp.c` |
| GRE (key-based) | `net/netfilter/nf_conntrack_proto_gre.c` |
| Generic (proto-agnostic) | `net/netfilter/nf_conntrack_proto_generic.c` |

### compatibility contract

### `struct nf_conntrack_l4proto` vtable

```c
struct nf_conntrack_l4proto {
    u_int8_t l4proto;
    bool   allow_clash;
    /* per-direction: extract tuple from packet */
    bool   (*pkt_to_tuple)(const struct sk_buff *, unsigned int, struct nf_conntrack_tuple *);
    bool   (*invert_tuple)(struct nf_conntrack_tuple *, const struct nf_conntrack_tuple *);
    /* state machine on packet */
    int    (*packet)(struct nf_conn *, struct sk_buff *, unsigned int, enum ip_conntrack_info, const struct nf_hook_state *);
    /* on new flow */
    bool   (*new)(struct nf_conn *, const struct sk_buff *, unsigned int);
    /* error checking */
    int    (*error)(struct net *, struct nf_conn *, struct sk_buff *, unsigned int, const struct nf_hook_state *);
    /* per-protocol netlink dump support */
    int    (*to_nlattr)(struct sk_buff *, struct nlattr *, struct nf_conn *, bool);
    int    (*from_nlattr)(struct nlattr *, struct nf_conn *);
    /* per-direction timeout handling */
    void   (*destroy)(struct nf_conn *);
    /* per-protocol netfilter conntrack timeout policy */
    int    (*get_timeouts)(struct net *, unsigned int *, unsigned int *);
    /* event reporting */
    int    nlattr_size;
    int    nlattr_tuple_size;
    /* etc. */
};
```

Layout-equivalent for first cache-line.

### TCP state machine (12 states — RFC 793 + 6298)

| State | Constant | Transition |
|---|---|---|
| `NONE` | 0 | initial; replaced on first packet |
| `SYN_SENT` | 1 | sent SYN, awaiting SYN-ACK |
| `SYN_RECV` | 2 | sent SYN-ACK, awaiting final ACK |
| `ESTABLISHED` | 3 | both sides exchanged data |
| `FIN_WAIT` | 4 | sent FIN; awaiting peer FIN |
| `CLOSE_WAIT` | 5 | received peer FIN; awaiting local close |
| `LAST_ACK` | 6 | sent FIN after CLOSE_WAIT; awaiting ACK |
| `TIME_WAIT` | 7 | both FINed; in 2*MSL wait |
| `CLOSE` | 8 | terminal |
| `SYN_SENT2` | 9 | simultaneous open |
| `RETRANS` | 10 | (special) retransmit-detected |
| `UNACK` | 11 | (special) unacked outbound after FIN |

Plus per-state timeouts (in seconds, all configurable via sysctl):
- SYN_SENT: 120 (`nf_conntrack_tcp_timeout_syn_sent`)
- SYN_RECV: 60
- ESTABLISHED: 432000 (5 days)
- FIN_WAIT: 120
- CLOSE_WAIT: 60
- LAST_ACK: 30
- TIME_WAIT: 120
- CLOSE: 10
- RETRANS: 300
- UNACK: 300
- (Loose mode `unacknowledged`: 300)

Plus window/seq tracking via per-direction `td_end / td_maxend / td_maxwin / td_scale / td_flags`. Validates incoming TCP segments per RFC 5961 (mitigates blind-injection attacks).

Identical state machine + timeouts + validation rules.

### UDP state (stateless)

UDP has no real state — but conntrack tracks "have we seen reply direction?" via `IPS_SEEN_REPLY`. Default timeout 30s (unreplied) → 180s (replied). Configurable via `nf_conntrack_udp_timeout` / `_udp_timeout_stream`.

### ICMP / ICMPv6 (request/reply pair)

For Echo request/reply, Timestamp, Address Mask: tuple is (saddr, daddr, type, code, id); reply direction is (daddr, saddr, reply_type, reply_code, id). Only request/reply types are conntracked; other ICMP types are tagged `IP_CT_RELATED` referencing the original conntrack they relate to.

Default timeout: 30s.

### SCTP state machine (RFC 4960 + 9260)

Tracks INIT / INIT-ACK / COOKIE-ECHO / COOKIE-ACK / ESTABLISHED / SHUTDOWN_SENT / SHUTDOWN_RECD / SHUTDOWN_ACK_SENT / CLOSED. Per-association `vtag` (Verification Tag) tracking — both directions have independent vtags. Per-direction-CSN (Cumulative Stream Number) for chunk validation.

Default timeouts per state.

### GRE (key-based)

For PPTP / OpenVPN-GRE / L2TPv3-GRE: tuple is (saddr, daddr, key, version). PPTP requires explicit `nf_conntrack_proto_gre` registration via call from `nf_conntrack_pptp` helper. Stateless (2-direction).

### Generic (proto-agnostic)

For proto numbers without dedicated module (e.g., proto 89 OSPF, proto 97 ETHERIP). Tuple is just (saddr, daddr, protonum); no port-equivalent. 600s default timeout.

### `nf_conntrack_tcp_be_liberal` sysctl (default 0)

When 0: strict — out-of-window TCP segments → INVALID, packet dropped via ipt_INVALID match.
When 1: liberal — out-of-window allowed (compensates for asymmetric routing where conntrack misses some packets).

### TCP RFC 5961 mitigation

For SYN-after-ESTABLISHED, pure-ACK-with-bad-seq, etc. — implements RFC 5961 recommendations to ignore + send challenge-ACK rather than reset connection.

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| TCP state-transition table walk (no out-of-bounds index) | `kani::proofs::net::netfilter::conntrack::proto::tcp_state_safety` |
| TCP window arithmetic (no overflow on td_end + skb_len) | `kani::proofs::net::netfilter::conntrack::proto::tcp_window_safety` |
| SCTP vtag check (per-direction independent vtags; no cross-leak) | `kani::proofs::net::netfilter::conntrack::proto::sctp_vtag_safety` |
| Generic per-l4proto dispatch (no NULL fn-ptr deref on unknown proto) | `kani::proofs::net::netfilter::conntrack::proto::dispatch_safety` |

### Layer 2: TLA+ models

- `models/net/conntrack_state_machine.tla` (mandatory per `net/netfilter/00-overview.md` Layer 2) — proves TCP/UDP/ICMP/SCTP/GRE state machines refine their respective RFC specs. Owned here.

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| TCP per-direction window | `td_maxend ≥ td_end ≥ initial_seq` always; `td_maxwin ≥ td_end - td_initial_seq` | `kani::proofs::net::netfilter::conntrack::proto::tcp_window_invariants` |
| TCP state machine | reachable states form a DAG (modulo retransmit cycles); CLOSE is terminal | `kani::proofs::net::netfilter::conntrack::proto::tcp_state_invariants` |
| SCTP per-direction vtag | both directions' vtags non-zero after INIT-ACK; never reset mid-association | `kani::proofs::net::netfilter::conntrack::proto::sctp_vtag_invariants` |

### Layer 4: Functional correctness (declared in `net/netfilter/00-overview.md` Layer 4)

- **TCP-conntrack RFC compliance theorem** via TLA+ refinement — proves: kernel TCP-state-machine refines RFC 793/6298 + 5961 + 7414 reordering rules. Owned here.

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **CONSTIFY** | per-l4proto `nf_conntrack_l4proto` vtables `static const` | § Mandatory |
| **SIZE_OVERFLOW** | TCP window arithmetic + SCTP vtag arithmetic + per-state-timeout addition uses checked operators (CVE class: CVE-2014-class window-overflow) | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB**: per-`nf_conn`'s embedded `proto` union (cross-ref `conntrack-core.md`)
- **CONSTIFY, SIZE_OVERFLOW**: see above
- **MEMORY_SANITIZE**: per-protocol state cleared on conntrack free (TCP windows + SCTP vtags can be sensitive)
- **USERCOPY**: NLA parsing for to_nlattr/from_nlattr uses bound-checked accessors
- **KERNEXEC**: per-l4proto dispatch via `static const fn-ptr` array

### Row-2 / GR-RBAC integration

- LSM hook: per-protocol mutations all go through `conntrack-core.md` LSM gate (CAP_NET_ADMIN).
- Default useful GR-RBAC policy: deny `nf_conntrack_tcp_be_liberal` sysctl write outside gradm-marked `firewall_admin` role; liberal mode reduces attack-surface defenses.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

