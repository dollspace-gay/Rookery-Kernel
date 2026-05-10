---
title: "Tier-3: net/sched/sch_fifo.c — PFIFO/BFIFO/PFIFO_HEAD_DROP simple FIFO qdiscs"
tags: ["tier-3", "net", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

`pfifo` (packet-FIFO; pkt-count-limited) and `bfifo` (byte-FIFO; byte-count-limited) are the simplest qdiscs: per-pkt enqueue at tail; per-pkt dequeue at head. `pfifo_head_drop` variant drops oldest when full (BFB / TCP-friendly). Used as default leaf-qdisc under PRIO/MULTIQ/CBQ/MQPRIO/etc. Per-qdisc has `limit` (packets or bytes); per-overflow drops new (pfifo/bfifo) or drops head (pfifo_head_drop). Critical for: simple defaulting; per-queue baseline; HTB/HFSC leaf-qdisc.

This Tier-3 covers `sch_fifo.c` (~276 lines).

### Acceptance Criteria

- [ ] AC-1: `tc qdisc add dev eth0 root pfifo limit 1000`: pfifo allocated.
- [ ] AC-2: 1001-th enqueue: dropped (NET_XMIT_DROP).
- [ ] AC-3: bfifo limit=1MB: 1MB+1-byte dropped.
- [ ] AC-4: pfifo_head_drop limit=1000: 1001-th enqueued; oldest dropped.
- [ ] AC-5: dequeue returns FIFO order.
- [ ] AC-6: fifo_init without opt: uses default tx_queue_len.
- [ ] AC-7: fifo_dump: per-attr TCA_FIFO_LIMIT.
- [ ] AC-8: stats.drops increments per drop.
- [ ] AC-9: Per-PRIO leaf: pfifo serves per-band default.
- [ ] AC-10: pfifo as leaf of HTB: HTB rate-shapes; pfifo queues.

### Architecture

`Pfifo::enqueue(sch, skb, to_free) -> Result<()>`:
1. If sch.q.qlen >= sch.limit:
   - qdisc_drop(skb, sch, to_free).
   - return Err(NET_XMIT_DROP).
2. __qdisc_enqueue_tail(sch, skb).
3. sch.qstats.backlog += skb.len.
4. Return Ok(NET_XMIT_SUCCESS).

`Bfifo::enqueue(sch, skb, to_free) -> Result<()>`:
1. If sch.qstats.backlog + skb.len > sch.limit:
   - qdisc_drop(skb, sch, to_free).
   - return Err(NET_XMIT_DROP).
2. __qdisc_enqueue_tail(sch, skb).
3. sch.qstats.backlog += skb.len.
4. Return Ok(NET_XMIT_SUCCESS).

`PfifoHeadDrop::enqueue(sch, skb, to_free) -> Result<()>`:
1. __qdisc_enqueue_tail(sch, skb).
2. sch.qstats.backlog += skb.len.
3. If sch.q.qlen > sch.limit:
   - dropped = qdisc_dequeue_head(sch).
   - sch.qstats.backlog -= dropped.len.
   - sch.qstats.drops++.
   - qdisc_qstats_drop(sch).
   - kfree_skb_reason(dropped, SKB_DROP_REASON_QDISC_OVERLIMIT).
4. Return Ok(NET_XMIT_SUCCESS).

`Fifo::init(sch, opt) -> Result<()>`:
1. If !opt: sch.limit = sch.dev_queue.dev.tx_queue_len (× mtu for bfifo).
2. Else: sch.limit = parse(opt).
3. Ok.

`Fifo::dump(sch, skb) -> Result<()>`:
1. nla_put_u32(skb, TCA_FIFO_LIMIT, sch.limit).
2. Ok.

### Out of Scope

- TC qdisc API (covered in `sch-api.md` Tier-3)
- pfifo_fast (covered in `sch-generic.md` Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `pfifo_qdisc_ops` | per-class ops for pfifo | `Pfifo::OPS` |
| `bfifo_qdisc_ops` | per-class ops for bfifo | `Bfifo::OPS` |
| `pfifo_head_drop_qdisc_ops` | per-class ops for pfifo_head_drop | `PfifoHeadDrop::OPS` |
| `pfifo_enqueue()` | per-pkt enqueue | `Pfifo::enqueue` |
| `bfifo_enqueue()` | per-pkt enqueue (byte-limit) | `Bfifo::enqueue` |
| `pfifo_tail_enqueue()` | per-pkt tail-drop enqueue | `Pfifo::tail_enqueue` |
| `pfifo_head_drop_enqueue()` | per-pkt head-drop enqueue | `PfifoHeadDrop::enqueue` |
| `qdisc_dequeue_head()` | per-pkt head dequeue | `Qdisc::dequeue_head` |
| `__qdisc_enqueue_tail()` | per-pkt tail-list-add | `Qdisc::enqueue_tail` |
| `fifo_init()` | per-qdisc init (sets limit) | `Fifo::init` |
| `fifo_dump()` | per-qdisc dump | `Fifo::dump` |

### compatibility contract

REQ-1: pfifo limit:
- u32 packet count.
- Default: tx_queue_len (typically 1000).

REQ-2: bfifo limit:
- u32 byte count.
- Default: tx_queue_len × dev.mtu.

REQ-3: pfifo_enqueue:
- If sch.q.qlen >= sch.limit: qdisc_drop(skb); return Err(NET_XMIT_DROP).
- __qdisc_enqueue_tail(sch, skb).
- sch.qstats.backlog += skb.len.
- Return NET_XMIT_SUCCESS.

REQ-4: bfifo_enqueue:
- If sch.qstats.backlog + skb.len > sch.limit: qdisc_drop(skb); return Err.
- __qdisc_enqueue_tail(sch, skb).
- sch.qstats.backlog += skb.len.
- Return NET_XMIT_SUCCESS.

REQ-5: pfifo_head_drop_enqueue:
- __qdisc_enqueue_tail(sch, skb).
- sch.qstats.backlog += skb.len.
- If sch.q.qlen > sch.limit:
  - dropped = qdisc_dequeue_head(sch).
  - sch.qstats.backlog -= dropped.len.
  - kfree_skb(dropped).
  - sch.qstats.drops++.
- Return NET_XMIT_SUCCESS.

REQ-6: pfifo_dequeue / bfifo_dequeue (default):
- skb = qdisc_dequeue_head(sch).
- If skb: sch.qstats.backlog -= skb.len.
- Return skb.

REQ-7: fifo_init:
- Parse opt or use default.
- sch.limit = computed limit.
- pfifo: limit in pkts; bfifo: limit in bytes.

REQ-8: Per-NIC default leaf:
- pfifo_fast (default if no qdisc) is in `sch_generic.c`, not this file.
- This file's pfifo is set when user explicitly chooses.

REQ-9: Per-stats:
- qstats.qlen / qstats.backlog / qstats.drops tracked.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pfifo_qlen_le_limit_post_drop` | INVARIANT | post-pfifo_enqueue: sch.q.qlen ≤ sch.limit. |
| `bfifo_backlog_le_limit_post_drop` | INVARIANT | post-bfifo_enqueue: sch.qstats.backlog ≤ sch.limit. |
| `head_drop_drops_oldest` | INVARIANT | per-head-drop: dropped == was-head. |
| `dequeue_returns_head` | INVARIANT | per-dequeue: returned == was-head. |
| `qstats_consistent` | INVARIANT | qstats.qlen == #queued. |

### Layer 2: TLA+

`net/sched/sch_fifo.tla`:
- Per-pkt enqueue + per-pkt dequeue + per-overflow drop.
- Properties:
  - `safety_no_overcommit_pfifo` — sch.q.qlen ≤ sch.limit always.
  - `safety_fifo_order` — per-(enq A, enq B, deq, deq): A first if enqueued first.
  - `liveness_pkt_eventually_dequeued` — per-enqueue + dequeue called ⟹ pkt out.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Pfifo::enqueue` post: sch.q.qlen ≤ sch.limit OR drop returned | `Pfifo::enqueue` |
| `Bfifo::enqueue` post: sch.qstats.backlog ≤ sch.limit OR drop returned | `Bfifo::enqueue` |
| `PfifoHeadDrop::enqueue` post: enqueued; head-dropped if over | `PfifoHeadDrop::enqueue` |
| `Fifo::init` post: sch.limit set per opt or default | `Fifo::init` |

### Layer 4: Verus/Creusot functional

`Per-FIFO order: enqueue at tail; dequeue at head; per-overflow drop new (pfifo/bfifo) or oldest (pfifo_head_drop)` semantic equivalence: per-FIFO matches RFC2917 simple-queue semantics.

### hardening

(Inherits row-1 features from `net/sched/sch-api.md` § Hardening.)

FIFO-specific reinforcement:

- **Per-limit u32 bounded** — defense against per-config insane limit causing OOM.
- **Per-pfifo qlen-cap honored** — defense against per-pkt-buffer overflow.
- **Per-bfifo backlog-cap honored** — defense against per-byte-buffer overflow.
- **Per-head_drop reduces TCP TimeOut** — defense against per-tail-drop causing slow-start.
- **Per-fifo init validates limit > 0** — defense against per-config zero-limit.
- **Per-stats counters atomic** — defense against per-stat race.
- **Per-leaf integration honors parent limit** — defense against per-parent-leaf-mismatch.
- **Per-init defaults to tx_queue_len** — defense against per-config missing limit.
- **Per-overflow drop reason logged** — defense against per-debug-loss.
- **Per-NETIF_F_LLTX safe** — defense against LLTX-NIC bypass leaving qdisc starved.

