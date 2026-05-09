---
title: "Tier-3: net/sched/sch-fq — sender-side fair queueing (TCP pacing backbone)"
tags: ["design-doc", "tier-3", "net"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for `fq` — Sender-side Fair Queueing optimized for TCP pacing. Per-flow rb-tree of pending TCP / UDP socket flows; per-flow `time_next_packet` field tracks when the flow may next emit (computed from `sk->sk_pacing_rate` × payload_size). Dequeue picks the flow with the smallest `time_next_packet` among ready flows. Combines sender pacing (BBR / TCP_INTERNAL_PACING) with per-flow fair-queueing without the AQM machinery of `fq_codel` (the AQM is left to the receiver path or to upper-layer congestion control).

Critical for: TCP_BBR (depends on pacing for the inflight-bound algorithm), TCP_INTERNAL_PACING (skb-level pacing baked into recent TCP — used when fq isn't installed), bulk-data servers (NFS, video CDN, ML-training storage backends). Default qdisc on many enterprise distros for high-bandwidth-low-latency workloads.

Sub-tier-3 of `net/sched/00-overview.md`. Pairs with `net/sched/sch-fq-codel.md` (sibling but different design — fq is sender-pacing-aware while fq_codel is generic-AQM). Heavily integrated with `net/ipv4/tcp.md` (TCP_PACING / `sk_pacing_rate` consumption).

### Requirements

- REQ-1: All `TCA_FQ_*` NLAs (per the table) parsed identically; defaults match upstream.
- REQ-2: Per-flow rb-tree (`delayed`) keyed on `time_next_packet`; per-flow ready FIFO (`internal`); identical structure.
- REQ-3: Dequeue: EDT-scheduled — flows with `time_next_packet ≤ now` ready; flows with future `tnp` delayed; identical algorithm.
- REQ-4: Per-skb `tnp` computation: `now + skb_len / sk->sk_pacing_rate` when `RATE_ENABLE=1`; honor `skb->tstamp` if set (EDT).
- REQ-5: Per-socket hash bucketing: `socket_hash = hash(sk, ...) mod 2^buckets_log`; identical hash function.
- REQ-6: Horizon enforcement: skb with `tstamp - now > horizon` → drop (default) or cap to horizon.
- REQ-7: Low-rate-threshold deprioritization: flows with rate < `LOW_RATE_THRESHOLD` moved to throttled list.
- REQ-8: Priority class integration: `priomap[16]` + `weights[]`; WRR over priority classes.
- REQ-9: Per-flow + per-qdisc statistics: drop_overlimit, throttled, flows_plimit, ce_marks (when CE_THRESHOLD set).
- REQ-10: Per-flow refill delay (`FLOW_REFILL_DELAY` default 40ms): minimum gap between flow getting a fresh credit; identical timer.
- REQ-11: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: `tc qdisc add dev eth0 root fq` test: NLA byte-identical to upstream's; visible in `tc -s qdisc show`. (covers REQ-1)
- [ ] AC-2: TCP_BBR pacing test: install fq + enable BBR (`net.ipv4.tcp_congestion_control=bbr`); long-haul TCP transfer → tcpdump shows packets emitted at BBR's pacing rate (not bursty); per-RTT cwnd sees flat throughput. (covers REQ-3, REQ-4)
- [ ] AC-3: EDT skb test: socket with `SO_TXTIME` set; skb with `skb->tstamp = future_ns` → fq emits at exactly that time ± slack. (covers REQ-3, REQ-4)
- [ ] AC-4: Per-flow fairness test: 10 TCP flows on same socket priorities; per-flow throughput converges to 1/10 of link bandwidth. (covers REQ-2)
- [ ] AC-5: Horizon test: skb with `skb->tstamp = now + 100s` (100s > horizon=10s) → drop with HORIZON_DROP=1 or capped to now+10s with =0. (covers REQ-6)
- [ ] AC-6: Low-rate test: flow with `sk_pacing_rate=200kbps` (below 550kbps) → flow tagged low-rate; high-rate flows preferred during contention. (covers REQ-7)
- [ ] AC-7: Priomap test: `tc qdisc add ... fq priomap 0 1 2 3 ...`; skb->priority=5 routed to priority class 5; per-class weights respected. (covers REQ-8)
- [ ] AC-8: Buckets-log test: `tc qdisc add ... fq buckets_log 14` → 16384 buckets; verify per-flow lookup distributes evenly. (covers REQ-5)
- [ ] AC-9: `tc -s qdisc show dev eth0` byte-identical content (drop_overlimit, throttled, ce_marks). (covers REQ-9)
- [ ] AC-10: Hardening section present and follows template. (covers REQ-11)

### Architecture

### Rust module organization

- `kernel::net::sched::sch_fq::Fq` — qdisc instance
- `kernel::net::sched::sch_fq::Flow` — per-flow state
- `kernel::net::sched::sch_fq::DelayedTree` — rb-tree of `delayed` flows (keyed on time_next_packet)
- `kernel::net::sched::sch_fq::InternalList` — FIFO of `internal` (ready) flows
- `kernel::net::sched::sch_fq::Edt` — EDT scheduling computation
- `kernel::net::sched::sch_fq::PacingRate` — sk_pacing_rate consumption
- `kernel::net::sched::sch_fq::Hash` — per-socket hash + bucketing
- `kernel::net::sched::sch_fq::Horizon` — future-tstamp enforcement
- `kernel::net::sched::sch_fq::LowRate` — low-rate flow deprioritization
- `kernel::net::sched::sch_fq::Priomap` — priority class WRR

### Locking and concurrency

- **Per-qdisc `q.lock`** (spinlock, inherited): held during enqueue/dequeue
- **No new locks**: per-flow + rb-tree state accessed only under qdisc lock

### Error handling

- `Err(ENOBUFS)` — flow plimit hit
- `Err(EOVERFLOW)` — qdisc plimit hit
- `Err(EINVAL)` — bad NLA / inconsistent params
- `Err(EHORIZON)` — skb tstamp exceeds horizon (returned via drop counter, not callback)

### Out of Scope

- TCP_BBR + TCP_INTERNAL_PACING (cross-ref `net/ipv4/tcp.md` Tier-3)
- Other qdiscs (cross-ref `net/sched/sch-fq-codel.md`, `sch-cake.md`)
- 32-bit-only paths
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| FQ qdisc: per-flow rb-tree, time-next-packet scheduling, NLA parser, dump | `net/sched/sch_fq.c` |
| Public API | `include/net/sch_generic.h` |
| UAPI: `TCA_FQ_*` NLAs | `include/uapi/linux/pkt_sched.h` |

### compatibility contract

### NLA configuration

`TCA_OPTIONS` nested NLAs:
- `TCA_FQ_PLIMIT` (default 10000): total qdisc packet limit
- `TCA_FQ_FLOW_PLIMIT` (default 100): per-flow packet limit
- `TCA_FQ_QUANTUM` (default 2 × MTU): per-flow per-round byte quantum
- `TCA_FQ_INITIAL_QUANTUM` (default 10 × MTU): initial-burst quantum (TCP slow-start)
- `TCA_FQ_RATE_ENABLE` (default 1): enable per-flow rate-limiting (consumes `sk->sk_pacing_rate`)
- `TCA_FQ_FLOW_DEFAULT_RATE` (default 0): per-flow default rate when sk pacing rate is 0
- `TCA_FQ_FLOW_MAX_RATE` (default ~unlimited): per-flow rate cap
- `TCA_FQ_BUCKETS_LOG` (default 10 → 1024 buckets): hash-bucket count log2
- `TCA_FQ_FLOW_REFILL_DELAY` (default 40ms): minimum gap between flow refills
- `TCA_FQ_LOW_RATE_THRESHOLD` (default 550kbps): threshold below which flows are deprioritized as "low-rate"
- `TCA_FQ_CE_THRESHOLD` (default unlimited — disabled): ECN-CE-mark on per-flow sojourn > threshold (rare; `fq` doesn't normally do AQM)
- `TCA_FQ_TIMER_SLACK` (default 10us): hrtimer slack for wakeups
- `TCA_FQ_HORIZON` (default 10s): max future scheduling horizon
- `TCA_FQ_HORIZON_DROP` (default 1): drop-or-cap policy when EDT exceeds horizon
- `TCA_FQ_PRIOMAP` (per-priority TC mapping for prio classes 0..15)
- `TCA_FQ_WEIGHTS` (per-band weight array)

Wire format byte-identical so iproute2's `tc qdisc add fq ...` works unchanged.

### Per-flow state

Per-flow node in the rb-tree:
- `sk`: pointer to socket (or NULL for non-socket flows like ARP)
- `head`, `tail`: skb queue
- `qlen`: queued packet count
- `time_next_packet` (u64 ns): earliest time next packet may emit
- `credit` (s64): per-quantum credit (DRR-style)
- `socket_hash`: per-socket hash for bucketing
- `next`: rb-node linkage in `delayed` (waiting) tree or `internal` (ready) FIFO
- `age`: monotonic counter for tie-breaking

### Dequeue: time-ordered with EDT (Earliest Departure Time)

1. If a flow's `time_next_packet > now` → flow is in the `delayed` rb-tree (sorted by time)
2. If `time_next_packet ≤ now` → flow is in the `internal` ready list
3. Dequeue picks from `internal` list head; if empty + `delayed` non-empty + earliest-time ≤ now → move flow to internal
4. After dequeue: per-skb `tnp = now + skb_len / sk->sk_pacing_rate`; if `tnp > now` → flow re-inserted into `delayed`; else stays in `internal`

Identical EDT scheduler.

### `sk->sk_pacing_rate` consumption

When `RATE_ENABLE=1`: each skb's pacing rate comes from `skb->sk->sk_pacing_rate` (or `tcp_sk(sk)->...` for TCP-specific). TCP updates this field per ACK based on its congestion-control state (BBR sets it to ~bottleneck-bandwidth × 1.25).

When skb has `skb->tstamp` set (EDT skb timestamp), fq honors that as `time_next_packet` directly (overriding rate-derived). Used by TCP-internal-pacing path.

### Hash bucketing

Per-socket flow lookup via `socket_hash = hash_of(sk, sk_pacing_rate, ...) mod 2^buckets_log`. Per-bucket linked-list of flows; collision-resolution via list walk + `socket_hash` match.

### Horizon enforcement

Skb with `skb->tstamp - now > horizon` → drop (`TCA_FQ_HORIZON_DROP=1`) or cap to `now + horizon` (`TCA_FQ_HORIZON_DROP=0`). Defends against "send 1 hour from now" buggy senders.

### Per-flow throttling state

When per-flow drops or pacing-rate falls below `LOW_RATE_THRESHOLD`, flow is moved to a per-bucket "throttled list" deprioritized vs. high-rate flows. Identical heuristic.

### Priority class integration (PRIOMAP + WEIGHTS)

Per `skb->priority` (0..15) → priority class via `priomap[]`; per-class weight via `weights[]`. WRR over priority classes within `internal` dequeue. Identical algorithm.

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| EDT computation arithmetic (no overflow on now + skb_len / rate) | `kani::proofs::net::sched::sch_fq::edt_safety` |
| rb-tree insert/erase under q.lock | `kani::proofs::net::sched::sch_fq::rbtree_safety` |
| Per-flow hash bucket walk (no out-of-bounds) | `kani::proofs::net::sched::sch_fq::hash_safety` |
| Horizon comparison arithmetic (signed/unsigned correct) | `kani::proofs::net::sched::sch_fq::horizon_safety` |
| Priority WRR weight arithmetic | `kani::proofs::net::sched::sch_fq::priomap_safety` |

### Layer 2: TLA+ models

(none new owned at this granularity; relies on `models/net/sch_fq_codel.tla` for fair-queueing semantics)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-flow `time_next_packet` | flow in `delayed` rb-tree ⇔ tnp > now (at last move-time); flow in `internal` ⇔ tnp ≤ now | `kani::proofs::net::sched::sch_fq::placement_invariants` |
| rb-tree | sorted by tnp ascending; size = `delayed_count` | `kani::proofs::net::sched::sch_fq::rb_invariants` |
| Per-bucket flow list | every flow in bucket B has `socket_hash mod 2^buckets_log == B` | `kani::proofs::net::sched::sch_fq::bucket_invariants` |

### Layer 4: Functional correctness (opt-in)

- **EDT pacing-rate-bound theorem** via Verus — proves: time-averaged dequeue rate per-flow ≤ `sk->sk_pacing_rate`.
- **Per-flow fairness theorem** via Verus — proves: under continuous traffic on N equal-rate flows, each gets ~1/N of link bandwidth.

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **SIZE_OVERFLOW** | EDT arithmetic + rate × time computations use checked operators | § Mandatory |
| **LATENT_ENTROPY** | per-qdisc per-bucket hash perturbation seeded from kernel CSPRNG | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: per-skb (cross-ref `net/skbuff.md`); per-Qdisc (cross-ref `net/sched/sch-api.md`)
- **CONSTIFY**: `Qdisc_ops` for fq `static const`
- **SIZE_OVERFLOW, LATENT_ENTROPY**: see above
- **KERNEXEC**: per-method dispatch via `static const fn-ptr` array

### Row-2 / GR-RBAC integration

- LSM hook: same as `sch-api.md` (CAP_NET_ADMIN gate).
- Default GR-RBAC policy: empty.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

