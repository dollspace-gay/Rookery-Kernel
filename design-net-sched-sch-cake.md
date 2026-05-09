---
title: "Tier-3: net/sched/sch-cake — CAKE qdisc (Common Applications Kept Enhanced)"
tags: ["design-doc", "tier-3", "net"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for CAKE — the integrated traffic-shaper-plus-AQM that became the de-facto standard for residential/laptop/CPE bufferbloat-fixing in 2017+. CAKE bundles into a single qdisc what would otherwise be HTB-rate-shaping + 8 DiffServ-tin priority bands + per-tin FQ-CoDel + per-host fairness. Per-skb path: classify into one of 8 DiffServ tins → hash into per-host bucket within tin → hash into per-flow bucket within host → enqueue with flow-aware FQ-CoDel.

CAKE's key differentiators vs. plain `fq_codel`:
- **Built-in shaping**: `bandwidth` parameter accurately rate-limits below the link's wire speed (avoids ISP-side bufferbloat by keeping the bottleneck inside the local kernel)
- **Per-host fairness**: a single host with many flows doesn't dominate the link
- **DiffServ-tin priority**: 8-tin (or selectable 3/4/8-tin) priority among DSCP classes
- **Auto-rate**: estimates wire bandwidth from observed throughput (when configured)
- **ACK filtering**: drops "unnecessary" cumulative-ACKs to reduce ACK-train overhead in asymmetric links (DSL upstream)

Sub-tier-3 of `net/sched/00-overview.md`. Pairs with `net/sched/sch-fq-codel.md` (per-flow inner mechanism shared), `net/sched/sch-htb.md` (alternative shaping approach).

### Requirements

- REQ-1: All `TCA_CAKE_*` NLAs (per the table) parsed identically; defaults match upstream.
- REQ-2: DiffServ tin selection: 5 modes (`BESTEFFORT / PRECEDENCE / DIFFSERV3 / DIFFSERV4 / DIFFSERV8`); per-tin rate-weighting identical.
- REQ-3: Per-host hash + per-flow hash 2-level hierarchy: 8 hash modes (`NONE / SRC_IP / DST_IP / HOSTS / FLOWS / DUAL_SRC / DUAL_DST / TRIPLE`).
- REQ-4: Shaper algorithm: per-tin clock-driven dequeue; `time_next_send = now + skb_size / tin_rate`; identical arithmetic.
- REQ-5: Auto-rate: when `base_rate = 0 + autorate_ingress = 1`, EWMA-estimate from observed throughput; identical algorithm.
- REQ-6: ACK filter: 3 modes (`NONE / FILTER / AGGRESSIVE`); per-flow queue scan + redundant-ACK drop.
- REQ-7: GSO split (`split_gso = 1`): pre-segment GSO skbs into per-flow MSS-sized for accurate per-skb shaper accounting.
- REQ-8: ATM/PTM overhead modeling: per `TCA_CAKE_ATM`, accurately account 53-byte ATM cells / 64-byte PPP-over-ATM headers.
- REQ-9: Per-fwmark override (`fwmark`): override DiffServ tin from packet's fwmark.
- REQ-10: NAT-aware classification (`nat`): peek at conntrack NAT state to identify "real" srcip/dstip for hash.
- REQ-11: Per-tin + per-flow + per-qdisc statistics byte-identical.
- REQ-12: Memory limit (`memory` default 32 MiB) enforced; over-limit drops oldest from longest-backlog flow.
- REQ-13: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: `tc qdisc add dev eth0 root cake bandwidth 100mbit diffserv4 triple-isolate` test: NLA byte-identical; visible in `tc -s qdisc show`. (covers REQ-1)
- [ ] AC-2: DiffServ tin classification test: send packets with DSCP=EF (voice) + DSCP=BE → packets enqueued in voice + best-effort tins respectively; tcpdump shows voice tin's packets dequeued before BE under contention. (covers REQ-2)
- [ ] AC-3: Per-host fairness test: 1 host with 10 TCP flows + 1 host with 1 TCP flow, all into same DSCP tin → both hosts get equal share (50%); 10-flow host's flows compete among themselves for that 50%. (covers REQ-3)
- [ ] AC-4: Shaper accuracy test: configure `bandwidth 100mbit`; run iperf3 → measured throughput tracks 100 mbit ± 1%. (covers REQ-4)
- [ ] AC-5: Auto-rate test: `bandwidth 0 autorate-ingress`; under variable RX → CAKE's tracked rate adapts within seconds. (covers REQ-5)
- [ ] AC-6: ACK-filter test: TCP flow with `ack-filter` enabled → tcpdump shows fewer ACKs than baseline; throughput maintained. (covers REQ-6)
- [ ] AC-7: GSO split test: TCP send 64KB skbs through CAKE with `split_gso=1` → driver receives MSS-sized fragments; per-fragment shaper accounting accurate. (covers REQ-7)
- [ ] AC-8: ATM overhead test: configure `atm`; per-packet effective rate accounts for 5/53 ATM-cell-overhead. (covers REQ-8)
- [ ] AC-9: NAT-aware test: with `nat` mode + iptables MASQUERADE, classifier hashes by post-NAT srcip; container behind NAT shares fairly with host. (covers REQ-10)
- [ ] AC-10: `tc -s qdisc show dev eth0` byte-identical content for per-tin + per-flow + per-qdisc stats. (covers REQ-11)
- [ ] AC-11: Memory-limit test: configure `memory 1MiB`; flood; `drop_overlimit` increments when memory_used > 1MiB. (covers REQ-12)
- [ ] AC-12: Hardening section present and follows template. (covers REQ-13)

### Architecture

### Rust module organization

- `kernel::net::sched::sch_cake::Cake` — qdisc instance
- `kernel::net::sched::sch_cake::DiffservMode` — per-mode tin selection
- `kernel::net::sched::sch_cake::Tin` — per-tin state
- `kernel::net::sched::sch_cake::HostBucket` — per-host bucket within tin
- `kernel::net::sched::sch_cake::Flow` — per-flow bucket within host
- `kernel::net::sched::sch_cake::Shaper` — per-tin clock-driven dequeue
- `kernel::net::sched::sch_cake::AutoRate` — EWMA rate estimator
- `kernel::net::sched::sch_cake::AckFilter` — redundant-ACK detection + drop
- `kernel::net::sched::sch_cake::Atm` — ATM/PTM overhead modeling
- `kernel::net::sched::sch_cake::Codel` — CoDel primitives (shared with sch-fq-codel)
- `kernel::net::sched::sch_cake::Stats` — per-tin + per-flow + per-qdisc stats

### Locking and concurrency

- **Per-qdisc `q.lock`** (spinlock, inherited): held during enqueue/dequeue
- **No new locks**: per-tin / per-host / per-flow state accessed only under qdisc lock

### Error handling

- `Err(EINVAL)` — bad NLA / inconsistent params
- `Err(ENOBUFS)` — memory limit hit
- `Err(EOVERFLOW)` — packet limit hit

### Out of Scope

- Other qdiscs (cross-ref `net/sched/sch-fq-codel.md`, `sch-htb.md`)
- 32-bit-only paths
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| CAKE qdisc: tin classifier, per-host hash, per-flow FQ-CoDel state, shaper, NLA parser, dump | `net/sched/sch_cake.c` |
| CoDel primitives (shared with sch_fq_codel) | `include/net/codel.h` |
| UAPI: `TCA_CAKE_*` NLAs | `include/uapi/linux/pkt_sched.h` |

### compatibility contract

### NLA configuration

`TCA_OPTIONS` nested NLAs:
- `TCA_CAKE_BASE_RATE64` (default 0 — auto-rate): wire-rate in bits/sec
- `TCA_CAKE_DIFFSERV_MODE` (default `CAKE_DIFFSERV_DIFFSERV3`): tin selection — `BESTEFFORT / PRECEDENCE / DIFFSERV3 / DIFFSERV4 / DIFFSERV8`
- `TCA_CAKE_ATM` (default `CAKE_ATM_NONE`): link-overhead model — `NONE / ATM / PTM`
- `TCA_CAKE_FLOW_MODE` (default `CAKE_FLOW_TRIPLE`): hash mode — `NONE / SRC_IP / DST_IP / HOSTS / FLOWS / DUAL_SRC / DUAL_DST / TRIPLE`
- `TCA_CAKE_OVERHEAD` (default 0): per-packet overhead (e.g., -56 for shaper-corrected for FreeWAN ATM)
- `TCA_CAKE_RTT` (default 100ms): expected RTT (used for CoDel target tuning)
- `TCA_CAKE_TARGET` (default per-RTT computation)
- `TCA_CAKE_AUTORATE_INGRESS` (default 0): auto-rate from observed RX
- `TCA_CAKE_MEMORY` (default 32 MiB): memory limit
- `TCA_CAKE_MPU` (default 0): minimum per-packet unit
- `TCA_CAKE_INGRESS` (default 0): ingress shaping mode
- `TCA_CAKE_ACK_FILTER` (default `CAKE_ACK_NONE`): ACK-filter — `NONE / FILTER / AGGRESSIVE`
- `TCA_CAKE_SPLIT_GSO` (default 1): pre-segment GSO skbs to per-flow MSS-sized
- `TCA_CAKE_FWMARK` (default 0): per-fwmark overrides
- `TCA_CAKE_NAT` (default 0): apply NAT-aware classification

Wire format byte-identical so iproute2's `tc qdisc add cake ...` works unchanged.

### DiffServ tin selection

CAKE classifies each packet into one of 1, 3, 4, or 8 priority "tins" based on DSCP byte:

| Mode | Tins | Rationale |
|---|---|---|
| `BESTEFFORT` | 1 | All traffic equal |
| `PRECEDENCE` | 8 | Per legacy IP precedence (3-bit) |
| `DIFFSERV3` | 3 | Voice (EF/CS6/CS7), Best Effort, Bulk (CS1/CS2) |
| `DIFFSERV4` | 4 | Voice, Best Effort, Bulk, Background |
| `DIFFSERV8` | 8 | Per all 8 DSCP classes |

Each tin has its own per-tin shaping rate (computed from total `base_rate` × tin-weight). Identical mapping.

### Per-host hash

Per-tin: hash `(srcip, dstip)` (per `flow_mode == CAKE_FLOW_TRIPLE`) → host-bucket index. Per-host CoDel state.

### Per-flow hash within host

Per-host bucket: hash `(srcip, dstip, sport, dport, proto)` 5-tuple → per-flow bucket. Per-flow CoDel state.

Identical 2-level hash hierarchy.

### Shaper: `cake_dequeue_one`

Per-dequeue:
1. Compute per-tin bandwidth (allotted from total `base_rate`)
2. Walk tins in priority order
3. For each tin: walk flows DRR-style (similar to fq_codel)
4. For each candidate skb: compute `time_next_send = now + (skb_size / tin_rate)`
5. If `time_next_send` is in the future → schedule wakeup; current dequeue returns NULL
6. Else → dequeue + advance per-tin shaper clock

Identical algorithm.

### Auto-rate (when `bandwidth = 0` AND `autorate_ingress = 1`)

Track recent throughput per-flow-bucket; estimate sustainable wire-rate. Adjust `base_rate` over time. Per RFC-style EWMA.

Identical estimator.

### ACK filtering

When `ack_filter` is enabled, on enqueue, scan the per-flow queue for older redundant ACKs (same TCP flow with cumulatively-superseded ACK numbers) and drop them. AGGRESSIVE mode also drops some non-redundant ACKs. Reduces ACK-train overhead on asymmetric links.

Identical algorithm.

### Per-tin + per-flow + per-qdisc statistics

Per `tc -s qdisc show`:
- Per-tin: `bytes`, `pkts`, `drops`, `marks`, `ack_drops`, `way_indirect_hits` (per-host hash collisions resolved via fallback)
- Per-flow: dequeue rate
- Per-qdisc: `memory_used`, `drop_overlimit`, `drop_no_destination`

Format byte-identical.

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Per-tin clock arithmetic (no overflow on `now + skb_size / rate`) | `kani::proofs::net::sched::sch_cake::shaper_safety` |
| 2-level hash dispatch (tin index ≤ MAX_TINS=8; host index ≤ flows_per_tin; flow index ≤ flows_per_host) | `kani::proofs::net::sched::sch_cake::hash_safety` |
| ACK-filter scan (per-flow queue walk; no use-after-free) | `kani::proofs::net::sched::sch_cake::ack_safety` |
| ATM overhead arithmetic (no integer overflow) | `kani::proofs::net::sched::sch_cake::atm_safety` |
| AutoRate EWMA arithmetic | `kani::proofs::net::sched::sch_cake::autorate_safety` |

### Layer 2: TLA+ models

(none new owned at this granularity; relies on `models/net/sch_fq_codel.tla` from `sch-fq-codel.md` for fair-queueing semantics + `models/net/sch_htb.tla` from `sch-htb.md` for shaping rate-bound)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-tin priority order | tin priorities are strictly ordered (tin 0 highest); per-tin rate-weights sum to 1.0 | `kani::proofs::net::sched::sch_cake::tin_invariants` |
| Per-host bucket | every host bucket's flow-list is non-overlapping with other host buckets | `kani::proofs::net::sched::sch_cake::host_invariants` |
| Per-flow CoDel state | every flow's CoDel state is independent of other flows' (no shared mutable state) | `kani::proofs::net::sched::sch_cake::flow_invariants` |

### Layer 4: Functional correctness (opt-in)

- **CAKE per-host fairness theorem** via Verus — proves: ∀ pair of hosts (A, B) with continuous traffic in same tin, dequeue rates converge to 1:1 ratio.
- **CAKE shaper accuracy theorem** via Verus — proves: time-averaged dequeue rate ≤ configured `base_rate` within ±1 packet of granularity.

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **SIZE_OVERFLOW** | shaper clock + ATM overhead + AutoRate EWMA arithmetic uses checked operators | § Mandatory |
| **LATENT_ENTROPY** | per-qdisc hash perturbation seeded from kernel CSPRNG | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: per-skb (cross-ref `net/skbuff.md`); per-Qdisc (cross-ref `net/sched/sch-api.md`)
- **CONSTIFY**: `Qdisc_ops` for CAKE `static const`
- **SIZE_OVERFLOW, LATENT_ENTROPY**: see above
- **KERNEXEC**: per-method dispatch via `static const fn-ptr` array

### Row-2 / GR-RBAC integration

- LSM hook: same as `sch-api.md` (CAP_NET_ADMIN gate).
- Default GR-RBAC policy: empty.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

