---
title: "Tier-3: net/sched/sch_sfq.c — SFQ (Stochastic Fairness Queueing) qdisc"
tags: ["tier-3", "net", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

SFQ (Stochastic Fairness Queueing) provides per-flow fair queueing without per-flow state explosion: per-skb hashed (jhash with periodic perturbation) into one of `divisor` (default 1024) flow buckets; per-bucket FIFO; round-robin dequeue across active buckets in deficit-round-robin order. Per-bucket capped at `depth` (default 127 pkts). Per-perturbation-period jhash key reseeded to mitigate hash-flooding adversaries. Approximate per-flow fairness with O(1) per-skb cost. Critical for: simple multi-flow fairness without HTB-style classification.

This Tier-3 covers `sch_sfq.c` (~982 lines).

### Acceptance Criteria

- [ ] AC-1: `tc qdisc add dev eth0 root sfq perturb 10`: SFQ allocated; perturbation timer armed.
- [ ] AC-2: 4 concurrent TCP flows: each ≈ 1/4 bandwidth.
- [ ] AC-3: One flow with 1000-pkt burst + one with 1-pkt: each gets fair share via DRR.
- [ ] AC-4: Per-bucket overflow > maxdepth: oldest dropped (headdrop).
- [ ] AC-5: Per-perturbation: q.perturbation reseeded.
- [ ] AC-6: Per-empty bucket: removed from active-list; slot freed.
- [ ] AC-7: tc -s class show: per-bucket stats.
- [ ] AC-8: divisor=2048: power-of-2 enforced.
- [ ] AC-9: limit reached: longest-bucket head-dropped.
- [ ] AC-10: sfq_change updates quantum atomically.

### Architecture

Per-qdisc state:

```
struct SfqSchedData {
  perturbation: SipHashKey,
  quantum: u32,
  perturb_period: u64,
  perturb_timer: HrTimer,
  divisor: u32,
  maxdepth: u32,
  maxflows: u32,
  limit: u32,
  headdrop: bool,
  ht: Vec<i32>,                                  // bucket_idx → slot_idx (-1 = empty)
  slots: Vec<SfqSlot>,
  dep: SfqDepArray,                              // free/busy slot lists
  tail: i32,                                     // head of active-list
  red_parms: Option<RedParms>,
}

struct SfqSlot {
  next: i32,
  prev: i32,
  allot: i32,                                    // DRR deficit (signed bytes)
  hash: u32,
  qlen: u32,
  head: SkbPtr,
  tail: SkbPtr,
  backlog: u32,
  vars: Option<RedVars>,
}
```

`Sfq::hash(skb, q) -> u32`:
1. h = skb_get_hash_perturb(skb, &q.perturbation).
2. Return h & (q.divisor - 1).

`Sfq::enqueue(sch, skb, to_free) -> Result<()>`:
1. bucket_idx = Sfq::hash(skb, q).
2. slot_idx = q.ht[bucket_idx].
3. If slot_idx == SFQ_EMPTY_SLOT:
   - slot_idx = q.dep.alloc_free()?
   - q.ht[bucket_idx] = slot_idx.
   - q.slots[slot_idx].init(bucket_idx, q.quantum).
   - sfq_link(q, slot_idx).
4. slot.fifo_push_back(skb); slot.qlen++; slot.backlog += skb.len.
5. sch.q.qlen++; sch.qstats.backlog += skb.len.
6. If slot.qlen > q.maxdepth:
   - dropped = if q.headdrop: slot.fifo_pop_front() else: slot.fifo_pop_back().
   - qdisc_drop(dropped); return Err.
7. If sch.q.qlen > q.limit: sfq_drop_longest(q, to_free).
8. Ok.

`Sfq::dequeue(sch) -> Option<&Skb>`:
1. If q.tail < 0: return None.
2. slot_idx = q.tail; slot = &mut q.slots[slot_idx].
3. skb = slot.fifo_peek().
4. bytes = qdisc_pkt_len(skb).
5. If slot.allot < bytes:
   - slot.allot += q.quantum.
   - q.tail = slot.next.
   - Goto 1.
6. slot.fifo_pop_front(); slot.allot -= bytes; slot.qlen--; slot.backlog -= skb.len.
7. sch.q.qlen--; sch.qstats.backlog -= skb.len.
8. If slot.qlen == 0:
   - sfq_dec(q, slot_idx).
   - q.ht[slot.hash] = SFQ_EMPTY_SLOT.
   - q.dep.return_free(slot_idx).
9. Return Some(skb).

`Sfq::perturbation(timer)`:
1. q.perturbation = random_u32().
2. q.perturb_timer rearmed at now + perturb_period.

### Out of Scope

- TC qdisc API (covered in `sch-api.md` Tier-3)
- FQ-CoDel (covered in `sch-fq-codel.md` Tier-3)
- FQ (covered in `sch-fq.md` Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct sfq_sched_data` | per-qdisc state | `SfqSchedData` |
| `struct sfq_slot` | per-bucket flow state | `SfqSlot` |
| `sfq_init()` | per-qdisc init | `Sfq::init` |
| `sfq_destroy()` | per-qdisc destroy | `Sfq::destroy` |
| `sfq_enqueue()` | per-pkt hash-and-enqueue | `Sfq::enqueue` |
| `sfq_dequeue()` | per-pkt round-robin dequeue | `Sfq::dequeue` |
| `sfq_drop()` | per-bucket-overflow drop | `Sfq::drop` |
| `sfq_change()` | per-qdisc reconfigure | `Sfq::change` |
| `sfq_link()` / `sfq_dec()` | per-bucket link/del | `Sfq::link` / `dec` |
| `sfq_perturbation()` | per-period rehash | `Sfq::perturbation` |
| `sfq_hash()` | per-skb → bucket-idx | `Sfq::hash` |
| `SFQ_MAX_FLOWS` (128) | max active flows | shared |
| `SFQ_DEFAULT_DEPTH` (127) | per-bucket cap | shared |
| `SFQ_MAX_DIVISOR` (32768) | max hash divisor | shared |

### compatibility contract

REQ-1: Per-tc UAPI tc_sfq_qopt:
- `quantum`: deficit per round (bytes; default ~MTU).
- `perturb_period`: rehash interval (seconds; default 0 = no perturb).
- `limit`: total queue cap (pkts).
- `divisor`: hash-table size (default 1024; max SFQ_MAX_DIVISOR).
- `flows`: max simultaneously-active flows (default 127).
- `headdrop`: bool: drop oldest in bucket (head) vs newest.
- `red_parms`: optional per-bucket RED.

REQ-2: Per-bucket sfq_slot:
- `next`, `prev`: doubly-linked list of active buckets (allot-DRR ring).
- `allot`: deficit credit (signed bytes).
- `hash`: bucket-idx into ht.
- `qlen`: per-bucket pkt count.
- `head`, `tail`: per-bucket FIFO ptr.
- backlog: per-bucket bytes.

REQ-3: Per-skb hash:
- bucket_idx = skb_get_hash_perturb(skb, &q.perturbation) & (q.divisor - 1).
- ht[bucket_idx] = slot_idx (i32; -1 if not active).

REQ-4: sfq_enqueue:
- bucket_idx = Sfq::hash(skb, q).
- slot_idx = q.ht[bucket_idx].
- If slot_idx == SFQ_EMPTY_SLOT (-1):
  - slot_idx = next-free-slot.
  - q.ht[bucket_idx] = slot_idx.
  - q.slots[slot_idx].hash = bucket_idx.
  - q.slots[slot_idx].allot = q.quantum.
  - sfq_link(q, slot_idx) into active-list tail.
- Per-bucket FIFO append skb.
- q.slots[slot_idx].qlen++.
- If q.slots[slot_idx].qlen > q.maxdepth: drop-from-bucket (head or tail).
- If sch.q.qlen > q.limit: sfq_drop(longest bucket).

REQ-5: sfq_dequeue (Deficit Round Robin):
- Get head of active-list.
- skb = bucket FIFO head.
- bytes = qdisc_pkt_len(skb).
- If slot.allot < bytes: slot.allot += q.quantum; rotate to tail; retry.
- Else: dequeue skb; slot.allot -= bytes.
- If slot.qlen == 0: sfq_dec; free slot.

REQ-6: Per-perturbation:
- After perturb_period seconds: q.perturbation = random_u32().
- New jhash key changes per-skb bucket-mapping.
- Existing buckets' skbs continue to drain at old mapping.

REQ-7: Per-headdrop vs taildrop:
- TCA_SFQ_HEAD_DROP: drop oldest in bucket (better TCP recovery).
- Default: tail-drop.

REQ-8: Per-bucket RED:
- Optional TCA_SFQ_RED_PARMS: per-bucket EWMA + threshold.

REQ-9: Per-class API:
- sfq_class_op: sfq exposes per-flow as classes via sfq_walk; per-class stats.

REQ-10: Per-divisor constraint:
- Power-of-2; ≤ SFQ_MAX_DIVISOR (32768).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `divisor_power_of_2` | INVARIANT | q.divisor is power-of-2 ≤ SFQ_MAX_DIVISOR. |
| `bucket_idx_lt_divisor` | INVARIANT | per-hash: bucket_idx < q.divisor. |
| `slot_idx_lt_max_flows` | INVARIANT | per-alloc: slot_idx < SFQ_MAX_FLOWS. |
| `qlen_le_maxdepth` | INVARIANT | per-slot.qlen ≤ q.maxdepth (post-drop). |
| `total_qlen_le_limit` | INVARIANT | sch.q.qlen ≤ q.limit (post-drop). |
| `active_list_consistent` | INVARIANT | per-active-list slot has qlen > 0. |

### Layer 2: TLA+

`net/sched/sch_sfq.tla`:
- Per-skb hash + per-bucket FIFO + DRR dequeue + perturbation.
- Properties:
  - `safety_drr_fairness` — per-fully-active flows: long-term bytes-served ≈ equal.
  - `safety_no_starve_active_bucket` — per-bucket-with-skb eventually dequeued.
  - `liveness_perturb_eventually_rehashes` — perturb_period > 0 ⟹ q.perturbation changes.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Sfq::enqueue` post: skb in slot.fifo; slot in active-list | `Sfq::enqueue` |
| `Sfq::dequeue` post: returned-skb from non-starved slot | `Sfq::dequeue` |
| `Sfq::perturbation` post: q.perturbation reseeded | `Sfq::perturbation` |
| `Sfq::change` post: per-quantum/divisor updated; existing buckets preserved | `Sfq::change` |

### Layer 4: Verus/Creusot functional

`Per-N-flows over time T → each flow served ≈ T/N bytes via DRR` semantic equivalence: per-SFQ matches Stochastic-Fairness-Queueing approximation.

### hardening

(Inherits row-1 features from `net/sched/sch-api.md` § Hardening.)

SFQ-specific reinforcement:

- **Per-perturbation periodic rehash** — defense against hash-flooding adversarial flows.
- **Per-bucket maxdepth cap** — defense against per-flow unbounded queueing.
- **Per-divisor power-of-2 + max** — defense against per-config invalid.
- **Per-DRR allot sign-bounded** — defense against deficit-overflow.
- **Per-active-list O(1) link/dec** — defense against per-skb-dispatch O(N).
- **Per-perturb_timer rate-limit** — defense against per-tick excessive rehashing.
- **Per-headdrop preserves TCP recovery** — defense against per-tail-drop causing timeout.
- **Per-divisor ≤ SFQ_MAX_DIVISOR** — defense against per-config OOM.
- **Per-slot init zeroes** — defense against per-stale state from freed slot.
- **Per-bucket RED gating** — defense against per-bucket overflow not signaling congestion.

