---
title: "Tier-3: net/sched/sch-fq-codel — FQ-CoDel qdisc (default since 4.12)"
tags: ["design-doc", "tier-3", "net"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for `fq_codel` (RFC 8290): the kernel-default qdisc since v4.12. Combines per-flow queueing (defeats bufferbloat between flows) with the CoDel AQM (Active Queue Management; defeats per-flow bufferbloat). Per-skb hash-classified into one of `flows` flow buckets (default 1024); each bucket has its own CoDel state. Dequeue order is round-robin among non-empty flows with per-flow byte-quantum (default 1514 — one MTU-sized packet per round). When a flow's queue grows past CoDel's `target` for longer than `interval`, CoDel starts marking ECN / dropping packets to signal congestion to upstream TCP/QUIC senders.

The `models/net/sch_fq_codel.tla` TLA+ model (mandatory per `net/sched/00-overview.md` Layer 2) is owned here; it proves per-flow queue isolation (no flow can starve another) and CoDel target/interval invariants.

Sub-tier-3 of `net/sched/00-overview.md`. Pairs with `net/sched/sch-cake.md` (extends fq_codel with shaping + per-host fairness; default for laptop/router setups), `net/sched/sch-fq.md` (sender-side fair queueing without AQM, used in TCP_PACING contexts).

### Requirements

- REQ-1: All `TCA_FQ_CODEL_*` NLAs (per the table) parsed identically; defaults match upstream.
- REQ-2: Per-flow hash classification: `skb_get_hash_perturb(skb, perturbation) % flows_cnt` identical.
- REQ-3: CoDel state machine per RFC 8289: `count` / `dropping` / `next_drop` / `last_above_time` semantics identical; geometric backoff `interval / sqrt(count)`.
- REQ-4: ECN marking precedence (`ECN` flag enabled): IP_ECN_set_ce instead of drop when `ip_hdr.tos & 0x3 == ECT_*`.
- REQ-5: CE_THRESHOLD: when configured, additionally CE-mark per-flow whose sojourn exceeds CE_THRESHOLD (prior to CoDel's target).
- REQ-6: Dequeue scheduler: deficit round-robin between new_flows + old_flows; identical algorithm.
- REQ-7: Per-qdisc memory limit (`memory_limit` default 32 MiB): on overflow → drop_batch_size packets dropped from longest-backlog flow.
- REQ-8: Perturbation rotation: `perturbation` reseeded every 10 minutes (or per-config); identical rotation cadence.
- REQ-9: Per-`/proc` + TCA_XSTATS counters byte-identical.
- REQ-10: TLA+ model `models/net/sch_fq_codel.tla` (mandatory per `net/sched/00-overview.md` Layer 2) — proves: per-flow queue isolation (no flow can starve another); CoDel target invariants (sojourn > target ⇒ eventual drop within interval).
- REQ-11: HW offload: per-NIC `qdisc-graft` to HW only when full feature set is HW-supported; otherwise software path.
- REQ-12: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: `pahole struct fq_codel_sched_data` byte-identical first cache-line. (covers REQ-1)
- [ ] AC-2: `tc qdisc add dev eth0 root fq_codel target 5ms interval 100ms ecn` test: NLA byte-identical; visible in `tc -s qdisc show dev eth0`. (covers REQ-1)
- [ ] AC-3: Per-flow isolation test: 10 TCP flows + 1 UDP-flood flow into single eth0 with fq_codel root → TCP throughput stable; UDP flood gets 1/11 share, doesn't starve others (verifiable via `tc -s qdisc` flow-bucket counters). (covers REQ-2, REQ-6, REQ-10)
- [ ] AC-4: CoDel marking test: TCP single-flow into fq_codel with 50ms RTT + 100Mbps bottleneck; sojourn exceeds 5ms-target after fill → CoDel marks/drops; throughput converges to ~95Mbps (cwnd-bounded). (covers REQ-3)
- [ ] AC-5: ECN test: with `ecn=1`, ECN-Capable-Transport (ECT(1)) marked TCP flow → CoDel CE-marks; flow throughput unaffected (CE != drop). (covers REQ-4)
- [ ] AC-6: Hash-perturbation rotation test: track flow-bucket assignments over 11 minutes → perturbation salt observably rotates at minute 10; flows reassigned. (covers REQ-8)
- [ ] AC-7: Memory-limit test: configure memory_limit=1MiB; flood with traffic → packets dropped when memory_usage exceeds limit; `drop_overlimit` counter increments. (covers REQ-7)
- [ ] AC-8: TLA+ `models/net/sch_fq_codel.tla` proves: ∀ pair of flows (A, B) with continuous traffic, A's dequeue rate is within 1 quantum of B's; CoDel target invariants hold. (covers REQ-10)
- [ ] AC-9: TCA_XSTATS counters test: enqueue 1000 packets through 100 flows → `new_flow_count = 100`, `maxpacket = max-skb-len`, etc. byte-identical. (covers REQ-9)
- [ ] AC-10: Hardening section present and follows template. (covers REQ-12)

### Architecture

### Rust module organization

- `kernel::net::sched::sch_fq_codel::FqCodel` — qdisc instance
- `kernel::net::sched::sch_fq_codel::Flow` — per-flow state
- `kernel::net::sched::sch_fq_codel::Hash` — per-skb flow assignment
- `kernel::net::sched::sch_fq_codel::Codel` — CoDel state machine (shared with sch-codel)
- `kernel::net::sched::sch_fq_codel::Drr` — deficit round-robin scheduler
- `kernel::net::sched::sch_fq_codel::Limits` — memory + packet limits
- `kernel::net::sched::sch_fq_codel::Perturb` — hash perturbation rotation
- `kernel::net::sched::sch_fq_codel::Stats` — per-qdisc + per-flow stats

### Locking and concurrency

- **Per-qdisc `q.lock`** (spinlock, inherited from sch_generic): held during enqueue/dequeue
- **No new locks**: per-flow state accessed only under qdisc lock

### Error handling

- `Err(ENOBUFS)` — qdisc memory_limit hit; oldest packet dropped
- `Err(EINVAL)` — bad NLA / invalid combination (e.g., `target > interval`)
- `Err(EOVERFLOW)` — packet limit hit

### Out of Scope

- Other qdiscs (cross-ref `net/sched/sch-cake.md`, `sch-fq.md`, `sch-htb.md`, etc.)
- Classifier framework (cross-ref `net/sched/cls-api.md`)
- Action framework (cross-ref `net/sched/act-api.md`)
- 32-bit-only paths
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| FQ-CoDel qdisc: per-flow buckets, CoDel state, enqueue/dequeue, NLA parser, dump | `net/sched/sch_fq_codel.c` |
| CoDel algorithm primitives (shared with sch_codel + sch_cake) | `include/net/codel.h`, `include/net/codel_impl.h`, `include/net/codel_qdisc.h` |
| UAPI: `TCA_FQ_CODEL_*` NLAs | `include/uapi/linux/pkt_sched.h` |

### compatibility contract

### NLA configuration

`TCA_OPTIONS` nested NLAs:
- `TCA_FQ_CODEL_TARGET` (default 5ms): per-flow target queue sojourn time before CoDel starts marking
- `TCA_FQ_CODEL_INTERVAL` (default 100ms): per-flow CoDel sliding window
- `TCA_FQ_CODEL_LIMIT` (default 10240): total qdisc packet limit
- `TCA_FQ_CODEL_ECN` (default 1): enable ECN marking instead of drop when possible
- `TCA_FQ_CODEL_FLOWS` (default 1024): number of per-flow buckets
- `TCA_FQ_CODEL_QUANTUM` (default 1514): per-flow per-round byte quantum
- `TCA_FQ_CODEL_CE_THRESHOLD` (default disabled): CE-mark even before CoDel target
- `TCA_FQ_CODEL_DROP_BATCH_SIZE` (default 64): max packets to drop in one over-limit-trigger
- `TCA_FQ_CODEL_MEMORY_LIMIT` (default 32 MiB): total qdisc memory cap
- `TCA_FQ_CODEL_CE_THRESHOLD_SELECTOR` (per-DSCP CE-marking threshold)

Wire format byte-identical so iproute2's `tc qdisc add fq_codel ...` works unchanged.

### Per-flow hash classification

`skb_get_hash_perturb(skb, perturbation)` → flow-bucket index mod `flows`. Per-flow `perturbation` salt rotates every 10 minutes (or per-`TCA_FQ_CODEL_PERTURB`-seconds) to defeat hash-collision attacks.

Identical hash + perturbation strategy.

### CoDel algorithm (RFC 8289)

Per-flow CoDel maintains:
- `count`: drop-count incremented per-marking event; resets when queue drains below target
- `dropping`: in dropping state (vs. monitoring state)
- `next_drop`: time until next drop (geometric backoff: `interval / sqrt(count)`)
- `last_above_time`: time queue first exceeded target

Dequeue path:
1. If queue empty → return NULL
2. Compute sojourn time `now - skb->tstamp`
3. If sojourn ≤ target OR queue ≤ MTU → not above target
4. If above target for ≥ interval → enter dropping state; drop or ECN-mark
5. Per-iteration: `count++`, `next_drop = now + interval / sqrt(count)`

Identical state machine + arithmetic.

### Dequeue: deficit round-robin among flows

Two flow lists:
- `new_flows`: flows that just received their first packet
- `old_flows`: flows that had at least one packet served

Per-round:
1. Pick the next flow from `new_flows`; if empty, pick from `old_flows`
2. If flow's `deficit` ≤ 0 → add `quantum` to `deficit`; move to `old_flows`; pick next
3. Dequeue one packet from flow; subtract `len` from `deficit`
4. If flow now empty: remove from list; if was in new_flows, move to old_flows; else discard

Identical scheduler.

### Per-flow + per-qdisc statistics

Per-`/proc` and TCA_XSTATS:
- `maxpacket`: largest seen packet
- `drop_overlimit`: drops at total-limit
- `ecn_mark`: ECN marks emitted
- `new_flow_count`: new-to-old transitions
- `qdisc_stats`: standard q-stats

Identical format.

### `struct fq_codel_sched_data` layout

`net/sched/sch_fq_codel.c` (kernel-internal, but `pahole` introspectable):
- `flows`: per-flow array (size `flows`)
- `backlogs`: per-flow backlog array
- `flows_cnt`: number of flows
- `quantum`: per-flow quantum
- `drop_batch_size`
- `memory_limit`, `memory_usage`
- `perturbation`
- `cparams` (codel_params): `target` / `interval` / `ce_threshold` / `ecn`
- `cstats` (codel_stats): aggregate codel events
- `qdisc_stats`
- `new_flows`, `old_flows`: list_head per-flow lists

Layout-equivalent.

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Per-flow hash mod flows_cnt (no out-of-bounds bucket index) | `kani::proofs::net::sched::sch_fq_codel::hash_safety` |
| CoDel arithmetic (no underflow on count++ then decrement; sqrt-table bounded) | `kani::proofs::net::sched::sch_fq_codel::codel_safety` |
| DRR deficit arithmetic | `kani::proofs::net::sched::sch_fq_codel::drr_safety` |
| Per-flow list (new_flows / old_flows) move | `kani::proofs::net::sched::sch_fq_codel::list_safety` |
| Memory accounting on enqueue/dequeue | `kani::proofs::net::sched::sch_fq_codel::memory_safety` |

### Layer 2: TLA+ models

- `models/net/sch_fq_codel.tla` (mandatory per `net/sched/00-overview.md`) — proves per-flow queue isolation + CoDel invariants. Owned here.

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-flow buckets | every flow appears in exactly one of {new_flows, old_flows, empty}; new_flows ⊆ flows-with-recent-arrival | `kani::proofs::net::sched::sch_fq_codel::list_invariants` |
| CoDel `count` | when `dropping=true`, `count > 0`; when `dropping=false`, `count == 0 OR last_above_time set` | `kani::proofs::net::sched::sch_fq_codel::codel_invariants` |
| Per-qdisc memory accounting | `memory_usage = Σ skb->truesize over enqueued skbs`; never negative | `kani::proofs::net::sched::sch_fq_codel::mem_invariants` |

### Layer 4: Functional correctness (opt-in; declared in `net/sched/00-overview.md` Layer 4)

- **FQ-CoDel fair-queueing theorem** via Verus — proves: ∀ pair of flows (A, B) with continuous traffic + equal-MTU-packets, dequeue rates converge to 1:1 ratio over time.
- **CoDel sojourn bound theorem** via Verus — proves: ∀ flow `f`, expected steady-state sojourn time ≤ target + interval × (1 / drop-prob-per-target).

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **SIZE_OVERFLOW** | CoDel arithmetic + DRR deficit + memory_usage uses checked operators | § Mandatory |
| **LATENT_ENTROPY** | per-qdisc `perturbation` seeded from kernel CSPRNG; rotated on a periodic timer (defense vs. hash-collision DoS attacks) | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: per-skb (cross-ref `net/skbuff.md`); per-Qdisc (cross-ref `net/sched/sch-api.md`)
- **CONSTIFY**: `Qdisc_ops` vtable for fq_codel `static const`
- **SIZE_OVERFLOW, LATENT_ENTROPY**: see above
- **KERNEXEC**: per-method dispatch via `static const fn-ptr` array

### Row-2 / GR-RBAC integration

- LSM hook: same as `sch-api.md` (CAP_NET_ADMIN gate).
- Default GR-RBAC policy: empty.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

