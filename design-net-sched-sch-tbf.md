---
title: "Tier-3: net/sched/sch_tbf.c — TBF (Token Bucket Filter) qdisc"
tags: ["tier-3", "net", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

TBF (Token Bucket Filter) is a classic rate-limiter: per-qdisc maintains a token bucket of size `buffer` bytes refilled at `rate` bytes/sec, with optional second-bucket of size `mtu` (peak) refilled at `peakrate` bytes/sec. Per-pkt enqueue queues; per-pkt dequeue requires `pkt_size + ip_overhead` tokens — if available: send + decrement; else schedule watchdog timer for token-replenishment-time. Critical for: simple bandwidth-shaping (egress shapers in containers, VM throttling, policing).

This Tier-3 covers `sch_tbf.c` (~632 lines).

### Acceptance Criteria

- [ ] AC-1: `tc qdisc add dev eth0 root tbf rate 10mbit burst 8000 limit 100kb`: TBF allocated.
- [ ] AC-2: 100MB egress: `tc -s qdisc show dev eth0` shows ~10Mbit/s sustained.
- [ ] AC-3: Per-pkt > buffer-derived max_size: dropped at enqueue.
- [ ] AC-4: 50ms-burst at line-rate then idle: bucket replenishes at rate.
- [ ] AC-5: Per-peakrate config: throttles short-bursts to peakrate.
- [ ] AC-6: Per-fire watchdog: dequeue retried.
- [ ] AC-7: Per-empty inner queue: dequeue returns NULL silently.
- [ ] AC-8: tbf_change(rate=20mbit) without re-init: rate doubled; existing tokens scaled.
- [ ] AC-9: tbf_dump: per-attr TCA_TBF_PARMS + RATE64 + PRATE64.
- [ ] AC-10: tbf_reset on dev down: tokens reset.

### Architecture

Per-qdisc state:

```
struct TbfSchedData {
  rate: PschedRatecfg,
  peak: PschedRatecfg,
  max_size: u32,
  buffer: u64,                                   // bucket size in ns
  mtu: u64,                                       // peak-bucket size in ns
  tokens: i64,                                    // available main tokens (ns)
  ptokens: i64,                                   // available peak tokens (ns)
  t_c: KtimeT,                                   // last token-refresh
  qdisc: &Qdisc,                                  // inner
  watchdog: QdiscWatchdog,
}

struct PschedRatecfg {
  rate_bytes_ps: u64,
  mult: u32,
  shift: u8,
  linklayer: u8,
  overhead: u16,
}
```

`Tbf::init(sch, opt)`:
1. Parse TCA_TBF_PARMS: rate, buffer, mtu, peakrate, limit.
2. psched_ratecfg_precompute(&q.rate, &qopt.rate, qopt.rate64).
3. If peakrate: psched_ratecfg_precompute(&q.peak, &qopt.peakrate, qopt.prate64).
4. q.buffer = qopt.buffer; q.mtu = qopt.mtu; q.max_size = (q.buffer * NSEC_PER_SEC) / q.rate.rate_bytes_ps.
5. q.qdisc = qdisc_create_dflt(BFIFO, limit).
6. q.tokens = q.buffer; q.ptokens = q.mtu.
7. q.t_c = ktime_get_ns().

`Tbf::enqueue(sch, skb, to_free) -> Result<()>`:
1. If skb.len > q.max_size: qdisc_drop(skb, sch, to_free); return Err(NetDropPackets).
2. ret = q.qdisc.enqueue(skb, q.qdisc, to_free)?.
3. sch.qstats.backlog += skb.len.
4. Ok(NetXmitSuccess).

`Tbf::dequeue(sch) -> Option<&Skb>`:
1. skb = q.qdisc.peek()?.
2. now = ktime_get_ns().
3. toks = min(now - q.t_c, q.buffer) + q.tokens.
4. ptoks = if peak: min(now - q.t_c, q.mtu) + q.ptokens; else: i64::MAX.
5. len = qdisc_pkt_len(skb).
6. toks_for_skb = psched_l2t_ns(&q.rate, len).
7. ptoks_for_skb = if peak: psched_l2t_ns(&q.peak, len); else: 0.
8. If toks ≥ toks_for_skb ∧ ptoks ≥ ptoks_for_skb:
   - skb = q.qdisc.dequeue().
   - q.tokens = toks - toks_for_skb.
   - q.ptokens = ptoks - ptoks_for_skb.
   - q.t_c = now.
   - sch.qstats.backlog -= skb.len.
   - Return Some(skb).
9. Else: qdisc_watchdog_schedule_ns(&q.watchdog, now + max(toks_for_skb - toks, ptoks_for_skb - ptoks)); return None.

`Psched::l2t_ns(rate, bytes) -> u64`:
1. Adjusted = bytes + rate.overhead.
2. Per-linklayer ATM: ceil(adjusted / 48) * 53.
3. ns = (adjusted * NSEC_PER_SEC) / rate.rate_bytes_ps.
4. Return ns.

`Tbf::change(sch, opt) -> Result<()>`:
1. Parse new params; recompute rates + max_size.
2. Atomic-swap q-state under sch.lock.

### Out of Scope

- TC qdisc API (covered in `sch-api.md` Tier-3)
- HTB / HFSC (covered separately)
- BPF qdisc (covered in `sch-bpf.md` Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct tbf_sched_data` | per-qdisc state | `TbfSchedData` |
| `tbf_init()` | per-qdisc init | `Tbf::init` |
| `tbf_destroy()` | per-qdisc destroy | `Tbf::destroy` |
| `tbf_enqueue()` | per-pkt enqueue | `Tbf::enqueue` |
| `tbf_dequeue()` | per-pkt dequeue (rate-limited) | `Tbf::dequeue` |
| `tbf_change()` | per-qdisc change params | `Tbf::change` |
| `tbf_dump()` | per-qdisc dump | `Tbf::dump` |
| `tbf_reset()` | per-qdisc reset | `Tbf::reset` |
| `psched_l2t_ns()` | bytes → ns conversion | `Psched::l2t_ns` |
| `psched_ratecfg_precompute()` | per-rate precompute lookup | `Psched::ratecfg_precompute` |
| `qdisc_watchdog_schedule_ns()` | per-qdisc timer arm | `Qdisc::watchdog_schedule_ns` |
| `TCA_TBF_PARMS` / `TCA_TBF_RATE64` / `TCA_TBF_PRATE64` | per-tc UAPI attrs | UAPI |

### compatibility contract

REQ-1: Per-tc UAPI tc_tbf_qopt:
- `rate`: rate config (struct tc_ratespec).
- `peakrate`: optional peak-rate config.
- `limit`: queue size in bytes.
- `buffer`: maximum tokens (bytes) main bucket holds.
- `mtu`: maximum tokens (bytes) peak bucket holds.

REQ-2: Per-qdisc state:
- `tokens`: i64 main-bucket tokens.
- `ptokens`: i64 peak-bucket tokens.
- `t_c`: ktime of last token-refresh.
- `rate`: psched rate config (psched_ratecfg).
- `peak`: psched peak-rate config.
- `qdisc`: inner queue (typically pfifo/sfq).
- `watchdog`: per-qdisc timer for token-replenishment.
- `max_size`: max packet size accepted (limited by buffer/mtu).

REQ-3: tbf_enqueue:
- If skb.len > tbf.max_size: drop + qdisc_pkt_len_init.
- inner_qdisc.enqueue(skb).
- Update qstats.

REQ-4: tbf_dequeue:
- skb = inner.peek().
- now = ktime_get_ns().
- toks = min(now - t_c, buffer); ptoks = min(now - t_c, mtu).
- toks_for_skb = psched_l2t_ns(rate, skb.len + ip_overhead).
- ptoks_for_skb = psched_l2t_ns(peak, skb.len + ip_overhead).
- If toks ≥ toks_for_skb ∧ (peak == NULL ∨ ptoks ≥ ptoks_for_skb):
  - tokens -= toks_for_skb.
  - ptokens -= ptoks_for_skb.
  - t_c = now.
  - Return skb (dequeued from inner).
- Else: schedule watchdog at max(toks_needed, ptoks_needed) - elapsed.
- Return None.

REQ-5: psched_l2t_ns(rate, len):
- Convert bytes-of-pkt to nanoseconds-at-rate.
- Per-rate.mult / rate.shift precomputed for fast u64 multiply.

REQ-6: Per-buffer/mtu constraints:
- buffer ≥ MTU + IP-overhead.
- max_size = (buffer × NSEC_PER_SEC) / rate.

REQ-7: Per-rate config:
- rate.rate_bytes_ps: bytes/sec.
- rate.linklayer: ethernet / atm.
- For ATM: per-cell padding accounted.

REQ-8: Per-watchdog:
- tbf.watchdog rearmed at next token-replenishment.
- Per-fire: __netif_schedule the qdisc to re-attempt dequeue.

REQ-9: Per-pkt-burst tolerance:
- tokens ≤ buffer (cap).
- Max burst = buffer / rate seconds at line-rate.

REQ-10: Per-MQ-prio integration:
- TBF sits as leaf qdisc; per-class shaping via embedded inner qdisc.

REQ-11: Per-tc-exts (BPF):
- Optional TCA_TBF_TBURST_HZ / TCA_TBF_PBURST etc.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `tokens_le_buffer` | INVARIANT | per-q.tokens ≤ q.buffer (after refresh-cap). |
| `ptokens_le_mtu` | INVARIANT | per-q.ptokens ≤ q.mtu. |
| `max_size_pos` | INVARIANT | q.max_size > 0. |
| `enqueue_drops_oversize` | INVARIANT | per-skb.len > max_size ⟹ enqueue drops. |
| `dequeue_decrements_tokens` | INVARIANT | per-successful-dequeue: tokens -= toks_for_skb. |

### Layer 2: TLA+

`net/sched/sch_tbf.tla`:
- Per-qdisc enqueue/dequeue/refresh/watchdog cycle.
- Properties:
  - `safety_no_send_without_tokens` — per-dequeue-success ⟹ tokens ≥ toks_for_skb.
  - `safety_long_term_rate_ge_input` — per-burst then idle: avg-output ≤ rate.
  - `liveness_eventually_dequeue` — per-pending-skb + tokens-replenishing ⟹ eventually dequeued.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Tbf::enqueue` post: skb in inner-qdisc OR dropped | `Tbf::enqueue` |
| `Tbf::dequeue` post: returned-Some ⟹ tokens decremented | `Tbf::dequeue` |
| `Tbf::dequeue` post: returned-None ⟹ watchdog scheduled | `Tbf::dequeue` |
| `Tbf::change` post: rate/buffer/mtu updated atomically | `Tbf::change` |

### Layer 4: Verus/Creusot functional

`Per-qdisc with rate R + buffer B → over interval T: bytes-sent ≤ R × T + B` semantic equivalence: per-token-bucket model satisfies leaky-bucket bandwidth contract.

### hardening

(Inherits row-1 features from `net/sched/sch-api.md` § Hardening.)

TBF-specific reinforcement:

- **Per-pkt size > max_size dropped** — defense against oversize-pkt starving bucket.
- **Per-tokens cap at buffer** — defense against bucket overflow under long-idle.
- **Per-watchdog absolute-ns** — defense against per-tick rounding drift.
- **Per-rate mult/shift precomputed** — defense against per-pkt costly division.
- **Per-ATM linklayer cell-padding** — defense against ATM rate undershoot.
- **Per-peakrate optional** — defense against per-feature mandatory-config bug.
- **Per-tbf change atomic-swap under sch.lock** — defense against per-config update race.
- **Per-qdisc reset clears tokens** — defense against stale tokens after iface restart.
- **Per-pkt overhead accounted** — defense against IP-tunnel-encap rate undershoot.
- **Per-inner qdisc dropped if can't fit** — defense against silent enqueue OOM.

