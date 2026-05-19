# Tier-3: net/ipv6/ip6-input — IPv6 RX path (recv → exthdr-walk → reasm → deliver)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/ipv6/ip6_input.c
  - net/ipv6/exthdrs.c
  - net/ipv6/exthdrs_core.c
  - net/ipv6/reassembly.c
  - net/ipv6/ip6_checksum.c
-->

## Summary
Tier-3 design for the IPv6 receive path: the entry point `ipv6_rcv` (called from `__netif_receive_skb_core` for `ETH_P_IPV6`-marked skbs), the IPv6 header validation + checksum, the extension-header walker (Hop-by-Hop → Routing → Fragment → AH → ESP → DestOpt → upper-layer), fragment reassembly (RFC 8200 § 4.5), the local-vs-forward decision (`ip6_input_finish` → `ip6_local_deliver` or `ip6_forward`), per-CPU netfilter `NF_INET_PRE_ROUTING` / `LOCAL_IN` hooks, and final delivery to upper-layer protocols via `inet6_protos[]` dispatch.

Sub-tier-3 of `net/ipv6/00-overview.md`. Pairs with `net/ipv6/ip6-output.md` (TX), `net/ipv6/route.md` (route lookup), `net/ipv6/exthdrs.md` (TLV option detail). Hot path for every received IPv6 packet — performance-critical and security-critical (parser-driven attack surface).

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| `ipv6_rcv` entry, header validation, local-vs-forward dispatch | `net/ipv6/ip6_input.c` |
| Extension-header walker + per-EH-type handlers | `net/ipv6/exthdrs.c` |
| EH skip helpers (used by GRO + checksum) | `net/ipv6/exthdrs_core.c` |
| IPv6 fragment reassembly | `net/ipv6/reassembly.c` |
| IPv6 checksum primitives | `net/ipv6/ip6_checksum.c` |

## Compatibility contract

### `ipv6_rcv` entry contract

Called for every skb whose protocol is `htons(ETH_P_IPV6)`. Validates:
1. skb is non-NULL, has at least `sizeof(struct ipv6hdr)` linear bytes
2. `ipv6_hdr(skb)->version == 6`
3. `ipv6_hdr(skb)->payload_len` matches skb length minus header (or zero for jumbograms)
4. saddr is not multicast (RFC 4291 § 2.7)
5. ttl-equivalent (`hop_limit`) — accepted in `ipv6_rcv` but decremented in `ip6_forward` for forwarding
6. Source-address validity (no `::1` from non-loopback iface)

Identical validation order so observed errors and counters match.

### Extension-header processing order (RFC 8200 § 4.1)

1. **Hop-by-Hop Options** (must be first if present)
2. **Destination Options** (intermediate; presented if Routing follows)
3. **Routing Header**
4. **Fragment Header** (after walk-down; reassembly point)
5. **Authentication Header** (AH)
6. **Encapsulating Security Payload** (ESP)
7. **Destination Options** (final; only consumed by destination)
8. **Upper-Layer** (TCP / UDP / ICMPv6 / SCTP / …)

The walker bounds total option count + total bytes parsed to defeat extension-header explosion attacks (CVE-2017-7542-class).

### Fragment reassembly

Per-(saddr, daddr, identification, nexthdr) reassembly queue; per-netns hash table; per-netns `frags.high_thresh` (default 4 MB) + `frags.low_thresh` (default 3 MB) backpressure. Per-RFC, fragments must be reassembled before further processing.

Fragment timeout: `frags.timeout` (default 60s). Identical defaults.

### Local-vs-forward decision

After exthdr walk + reassembly:
- If destination matches a local address (per `ipv6_chk_addr` against `inet6_dev->addr_list`) → `ip6_local_deliver` → upper-layer
- If destination is link-local multicast → `ip6_local_deliver` if joined (per `inet6_dev->mc_list`) AND/OR forward (per RFC for routers)
- Otherwise → `ip6_forward` (gated by `forwarding=1` per netdev)

Identical decision order.

### Netfilter hooks

`NF_INET_PRE_ROUTING` (after exthdr walk + reasm; before route lookup), `NF_INET_LOCAL_IN` (after route resolves to local), `NF_INET_FORWARD` (after route resolves to remote). Each hook can ACCEPT/DROP/STOLEN/QUEUE/REPEAT skbs identically.

### `struct ipv6hdr` layout

`include/uapi/linux/ipv6.h`: `version:4`, `priority:4`, `flow_lbl[3]`, `payload_len`, `nexthdr`, `hop_limit`, `saddr`, `daddr`. Layout-byte-identical (40 bytes total).

## Requirements

- REQ-1: `ipv6_rcv` validates skb + struct ipv6hdr per RFC 4291 + 8200; identical error counters bumped on validation failure.
- REQ-2: Extension-header walker: handles HBH, RH, Fragment, AH, ESP, DestOpt in RFC 8200 order; bounded total walk count.
- REQ-3: HBH option processing: walks TLV options; recognized types: PadN (1), RouterAlert (5), Jumbo Payload (194), Calipso, IOAM. Unrecognized type's high-2-bits dictate behavior (skip / discard / discard+ICMP per RFC 8200 § 4.2).
- REQ-4: Routing Header (Type 0 deprecated per RFC 5095; Type 2 for Mobile IPv6; Type 4 for SR; Type 1 for nimrod) — per-RFC; Type 0 dropped per RFC 5095.
- REQ-5: Fragment reassembly: per-netns hash; per-frag-queue timeout; high/low thresholds with backpressure-eviction. Identical defaults.
- REQ-6: AH/ESP: pass-through to xfrm (cross-ref `net/xfrm/00-overview.md`); not handled in this Tier-3.
- REQ-7: Final dispatch: `inet6_protos[nexthdr]->handler(skb)`; identical table contents.
- REQ-8: Local-vs-forward decision via per-netdev `forwarding` sysctl + per-iface address check; identical.
- REQ-9: Netfilter `NF_INET_PRE_ROUTING/LOCAL_IN/FORWARD` invoked at identical points.
- REQ-10: Multicast input: pkt with multicast daddr → `ip6_mc_input` → forwarded to all listening sockets per `mc_list`; identical.
- REQ-11: Per-netns counters bumped: `Ip6InReceives`, `Ip6InHdrErrors`, `Ip6InAddrErrors`, `Ip6InDiscards`, `Ip6InNoRoutes`, `Ip6InTruncatedPkts`, `Ip6InMcastPkts`, `Ip6InOctets`, `Ip6InMcastOctets`, `Ip6InBcastOctets`, `Ip6InNoECTPkts`, `Ip6InECT0Pkts`, `Ip6InECT1Pkts`, `Ip6InCEPkts`. Format-identical (cross-ref `/proc/net/snmp6`).
- REQ-12: TLA+ model `models/net/ipv6_frag_reasm.tla` (mandatory per `net/ipv6/00-overview.md` Layer 2) — proves: fragment reassembly never delivers a partially-overlapping fragment chain; per-frag-queue completion is total-order on `(offset, more)`.
- REQ-13: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `pahole struct ipv6hdr` byte-identical (40 bytes). (covers REQ-1)
- [ ] AC-2: Adversarial extension-header chain test: 64 nested HBH + 32 nested DestOpt → rejected with `Ip6InHdrErrors++`; CPU bounded; no kernel hang. (covers REQ-2)
- [ ] AC-3: HBH unrecognized-type test: Type 0xFF (high-2 bits = `11`, ICMP-and-discard) → packet dropped + ICMP Parameter Problem generated. (covers REQ-3)
- [ ] AC-4: Routing Header Type 0 test: per RFC 5095, dropped + ICMP Parameter Problem. (covers REQ-4)
- [ ] AC-5: Fragment reassembly test: send 3-fragment IPv6 packet (offsets 0/1280/2560); after all 3 received, kernel delivers reassembled packet to upper layer. (covers REQ-5)
- [ ] AC-6: Fragment timeout test: send 1 of 3 fragments; wait `frags.timeout + 1s`; queue evicted; counter `Ip6ReasmTimeout++`. (covers REQ-5)
- [ ] AC-7: Forwarding test: enable `net.ipv6.conf.all.forwarding=1` on a host; packet to off-link host → routed via configured route. (covers REQ-8)
- [ ] AC-8: Multicast test: a socket joined to `ff02::1`; packet to `ff02::1` → delivered. (covers REQ-10)
- [ ] AC-9: `cat /proc/net/snmp6` byte-identical to upstream after equivalent traffic. (covers REQ-11)
- [ ] AC-10: TLA+ `models/net/ipv6_frag_reasm.tla` proves: ∀ fragment queue, reassembled output is contiguous + total-ordered offset-sequence; never partial-overlap. (covers REQ-12)
- [ ] AC-11: Hardening section present and follows template. (covers REQ-13)

## Architecture

### Rust module organization

- `kernel::net::ipv6::input::Ipv6Rcv` — entry point
- `kernel::net::ipv6::input::HeaderValidator` — struct ipv6hdr validation
- `kernel::net::ipv6::input::ExthdrWalker` — bounded EH walker
- `kernel::net::ipv6::input::HbhHandler` — Hop-by-Hop options
- `kernel::net::ipv6::input::RoutingHandler` — Routing Header (Types 0/2/4)
- `kernel::net::ipv6::input::DestOptHandler` — Destination Options
- `kernel::net::ipv6::input::FragReasm` — fragment reassembly
- `kernel::net::ipv6::input::FragQueue` — per-(saddr, daddr, ident) queue
- `kernel::net::ipv6::input::LocalDeliver` — local-deliver path
- `kernel::net::ipv6::input::Forward` — forward path (cross-ref TX)
- `kernel::net::ipv6::input::McInput` — multicast input

### Locking and concurrency

- **Per-fragment-queue spinlock**: `q->lock`; protects per-queue fragment list
- **Per-netns frags hash bucket lock**: `inet_frags->frags_pcpu`-style
- **Per-netdev `inet6_dev->lock`**: read-side for address check
- **RCU**: `inet6_protos[]` lookup is RCU-side

### Error handling

- skb dropped with kfree_skb_reason for telemetry (Ip6InHdrErrors / Ip6InAddrErrors / Ip6InDiscards / Ip6ReasmFails)
- Counter bumped via `IP6_INC_STATS_BH(net, idev, …)` per-netns + per-idev
- ICMP Parameter Problem / Time Exceeded generated for the right error classes (per RFC 4443 + RFC 8200)

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| `ipv6_rcv` skb_pull / skb_set_network_header bounds | `kani::proofs::net::ipv6::input::rcv_safety` |
| Extension-header walker iteration count cap | `kani::proofs::net::ipv6::input::exthdr_walk_safety` |
| HBH TLV option-walk under bounded iter cap | `kani::proofs::net::ipv6::input::hbh_walk_safety` |
| Fragment-queue insert/erase + reasm output-skb build | `kani::proofs::net::ipv6::input::frag_reasm_safety` |
| Per-netns frag GC + timeout-driven eviction | `kani::proofs::net::ipv6::input::frag_gc_safety` |

### Layer 2: TLA+ models

- `models/net/ipv6_frag_reasm.tla` (mandatory per `net/ipv6/00-overview.md` Layer 2) — proves fragment reassembly invariants. Owned here.

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-netns frags hash | every queue's hash matches `inet6_frag_hashfn(saddr, daddr, id, …)` | `kani::proofs::net::ipv6::input::frag_hash_invariants` |
| Per-frag-queue offset-sorted list | offsets strictly increasing; final fragment has `more=0` | `kani::proofs::net::ipv6::input::frag_order_invariants` |
| Per-netns frag memory accounting | `frags.mem` = Σ `skb->truesize` over all queues + queued skbs | `kani::proofs::net::ipv6::input::frag_mem_invariants` |

### Layer 4: Functional correctness (opt-in; declared in `net/ipv6/00-overview.md` Layer 4)

- **Extension-header bounded-walk theorem** via Verus — proves: walker terminates within ≤ `MAX_HEADER_OPT_PARSED` iterations on any input; rejects 0-length options that would loop.
- **Fragment reassembly soundness** via TLA+ refinement — proves: reassembled output equals the byte-stream concatenation of fragments in offset order; rejects overlapping fragments.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **SIZE_OVERFLOW** | extension-header length + frag offset arithmetic uses checked operators (CVE class: CVE-2017-7542) | § Mandatory |
| **MEMORY_SANITIZE** | freed frag-queue skbs cleared (sensitive: payload-in-flight) | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: per-frag-queue + per-skb infrastructure (cross-ref `net/skbuff.md`)
- **CONSTIFY**: extension-header type-handler dispatch table `static const`
- **SIZE_OVERFLOW**: see above
- **KERNEXEC**: EH-handler dispatch via `static const fn-ptr` array

### Row-2 / GR-RBAC integration

- LSM hook `security_skb_classify_flow` (already standard) — flow-label-tagging for socket egress
- LSM hook on netfilter PRE_ROUTING (via secmark)
- Default GR-RBAC policy: empty so behavior matches upstream

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Grsecurity/PaX-style Reinforcement

Baseline hardening features applied across the IPv6 RX path:

- **PAX_USERCOPY** — any `skb_copy_*` from linear/frag area into user buffer bound-checked; refuse slab-boundary straddling.
- **PAX_KERNEXEC** — `ipv6_protocol[]`, `inet6_protos`, `ip6_input_finish` dispatch in `__ro_after_init`/`.rodata`.
- **PAX_RANDKSTACK** — kstack offset re-randomized on each `ip6_rcv` / `ip6_rcv_core` entry.
- **PAX_REFCOUNT** — `skb->users`, `dst->__refcnt`, and `inet_frag_queue->refcnt` saturating.
- **PAX_MEMORY_SANITIZE** — freed frag-queue skbs zeroed; payload-in-flight cannot leak across slab reuse.
- **PAX_UDEREF** — no user-pointer follow on the input path; netfilter user-attached objects only via verified accessors.
- **PAX_RAP / kCFI** — `ipv6_protocol[]->handler`, `inet6_protos[].handler`, and netfilter `nf_hook_entry->hook` signature-checked.
- **GRKERNSEC_HIDESYM** — `ip6_rcv`, `ip6_input_finish`, `ipv6_protocol`, `inet6_protos` hidden from non-root.
- **GRKERNSEC_DMESG** — `pr_warn_ratelimited` on bad-version, bad-payload-length, bad-hop-by-hop gated.

ip6-input-specific reinforcement:

- **`ip6_rcv` user-visible copies covered by PAX_USERCOPY** — defense against linear-vs-frag boundary-straddling read.
- **Hop-by-hop options strict-parse**: only known options accepted; unknown options handled per RFC 8200 `1xxx` (Discard + ICMP) bits; option-length walks under SIZE_OVERFLOW; defense against CVE-2017-7542-class.
- **Source-address validation**: drop multicast-source, loopback-from-non-lo, unspecified-source-on-non-DAD; martian-source counted via `IPSTATS_MIB_INADDRERRORS`.
- **Frag-queue per-namespace memory cap** (`ip6frag_high_thresh`/`low_thresh`) with PAX_REFCOUNT on `inet_frag_queue`; per-queue evict on LRU; defense against frag-slab exhaustion.
- **Per-skb `IP6CB(skb)` scratch metadata** stored in skb control block (not user-visible); flags validated before use; defense against attacker-influenced `IP6CB` field.

Rationale: `ip6_rcv` is reachable by every adjacent host and is the gate to all upper-layer IPv6 protocols; a single hop-by-hop option parse bug or a slab-boundary copy is a remote primitive. USERCOPY+SIZE_OVERFLOW+RAP on the dispatch path plus strict hop-by-hop processing plus bounded fragment memory close the standard remote-RX exploit surfaces.

## Open Questions

(none — IPv6 RX path semantics are exhaustively specified by RFC 4291 + 8200 + 5095 + upstream)

## Out of Scope

- IPv6 TX path (cross-ref `net/ipv6/ip6-output.md`)
- Per-EH-type protocol implementation (cross-ref `net/ipv6/exthdrs.md` for option semantics)
- AH/ESP handling (cross-ref `net/xfrm/00-overview.md`)
- 32-bit-only paths
- Implementation code
