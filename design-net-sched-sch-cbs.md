---
title: "Tier-3: net/sched/sch_cbs.c — CBS (Credit-Based Shaper) qdisc — IEEE 802.1Qav AVB"
tags: ["tier-3", "net", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

CBS (Credit-Based Shaper; IEEE 802.1Qav for AVB / 802.1Qcc for TSN) is a deterministic-rate qdisc for time-sensitive networking. Per-qdisc maintains a "credits" counter (signed bytes): per-idle-tick refills at `idleslope` bytes/sec; per-pkt-send drains at `sendslope = idleslope - port_rate` bytes/sec. Send only when credits ≥ 0; cap at `hicredit` (idle excess); floor at `locredit` (send excess). Per-qdisc may run as **soft** (kernel computes credits) or **offloaded** (NIC HW computes via `TC_SETUP_QDISC_CBS`). Critical for: AVB streaming-classes A/B (1ms latency budget), TSN scheduled traffic.

This Tier-3 covers `sch_cbs.c` (~578 lines).

### Acceptance Criteria

- [ ] AC-1: `tc qdisc add dev eth0 root cbs idleslope 98304 sendslope -901696 hicredit 153 locredit -1389 offload 0`: CBS allocated soft.
- [ ] AC-2: At sustained line-rate input: credit balance oscillates around zero.
- [ ] AC-3: Credits hit hicredit during idle: capped.
- [ ] AC-4: Credits hit locredit during burst: capped.
- [ ] AC-5: Negative credits: dequeue returns None until next-credit-time.
- [ ] AC-6: Watchdog fires at delay_from_credits time.
- [ ] AC-7: Offload mode + supported driver: TC_SETUP_QDISC_CBS issued; soft path bypassed.
- [ ] AC-8: idleslope > port_rate: -EINVAL (sendslope would be positive).
- [ ] AC-9: Per-class under MQPRIO: per-CBS independent.
- [ ] AC-10: cbs_change atomic-swap params.

### Architecture

Per-qdisc state:

```
struct CbsSchedData {
  enqueue: fn(&mut Skb, &mut Qdisc, &mut Vec<&Skb>) -> Result<()>,
  dequeue: fn(&mut Qdisc) -> Option<&Skb>,
  watchdog: QdiscWatchdog,
  qstats_lock: SpinLock,
  port_rate: u64,                                // bytes/s
  last: KtimeT,
  credits: i64,                                  // bytes
  hicredit: i32,
  locredit: i32,
  sendslope: i64,                                // bytes/s (negative)
  idleslope: i64,                                // bytes/s
  offload: bool,
}
```

`Cbs::init(sch, opt) -> Result<()>`:
1. Parse qopt → q.idleslope/sendslope/hicredit/locredit/offload.
2. q.port_rate = ndo_get_dev_speed(sch.dev) × bytes_per_mbit.
3. Validate: idleslope < port_rate; sendslope < 0; hicredit ≥ 0; locredit ≤ 0.
4. q.last = ktime_get_ns().
5. q.credits = 0.
6. If offload: install offload via ndo_setup_tc(TC_SETUP_QDISC_CBS).
7. Else: q.enqueue = enqueue_soft; q.dequeue = dequeue_soft.

`Cbs::enqueue(sch, skb, to_free) -> Result<()>`:
1. Return q.enqueue(skb, sch, to_free).

`Cbs::enqueue_soft(skb, sch, to_free) -> Result<()>`:
1. inner.enqueue(skb, inner, to_free)?.
2. sch.qstats.backlog += skb.len.
3. Ok.

`Cbs::dequeue_soft(sch) -> Option<&Skb>`:
1. skb = inner.peek().
2. If !skb: return None.
3. now = ktime_get_ns().
4. credits = q.credits.
5. If credits < 0:
   - credits_added = (now - q.last) × q.idleslope / NSEC_PER_SEC.
   - credits = min(credits + credits_added, q.hicredit).
6. If credits < 0:
   - delay = (-credits × NSEC_PER_SEC) / q.idleslope.
   - qdisc_watchdog_schedule_ns(&q.watchdog, now + delay).
   - q.credits = credits.
   - q.last = now.
   - return None.
7. inner.dequeue().
8. frame_time = (skb.len × NSEC_PER_SEC) / q.port_rate.
9. credits_used = (q.sendslope × frame_time / NSEC_PER_SEC).abs().
10. credits -= credits_used.
11. credits = max(credits, q.locredit).
12. q.credits = credits.
13. q.last = now.
14. sch.qstats.backlog -= skb.len.
15. Return Some(skb).

`Cbs::dequeue_offload(sch) -> Option<&Skb>`:
1. skb = inner.dequeue().
2. If skb: sch.qstats.backlog -= skb.len.
3. Return skb.

### Out of Scope

- TC qdisc API (covered in `sch-api.md` Tier-3)
- mqprio / taprio (covered in `sch-mqprio.md` / `sch-taprio.md` Tier-3)
- ETS / ETF (Earliest TxTime First; covered separately)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct cbs_sched_data` | per-qdisc state | `CbsSchedData` |
| `cbs_init()` | per-qdisc init | `Cbs::init` |
| `cbs_destroy()` | per-qdisc destroy | `Cbs::destroy` |
| `cbs_enqueue()` | per-pkt enqueue dispatcher | `Cbs::enqueue` |
| `cbs_enqueue_soft()` | per-pkt soft-mode enqueue | `Cbs::enqueue_soft` |
| `cbs_enqueue_offload()` | per-pkt offload-mode | `Cbs::enqueue_offload` |
| `cbs_dequeue()` | per-pkt dequeue dispatcher | `Cbs::dequeue` |
| `cbs_dequeue_soft()` | per-pkt soft-mode credit-check | `Cbs::dequeue_soft` |
| `cbs_dequeue_offload()` | per-pkt offload-pass-through | `Cbs::dequeue_offload` |
| `cbs_change()` | per-qdisc reconfigure | `Cbs::change` |
| `timediff_to_credits()` | per-time-elapsed credit-add | `Cbs::timediff_to_credits` |
| `credits_from_len()` | per-skb credit-debit | `Cbs::credits_from_len` |
| `delay_from_credits()` | per-deficit deadline | `Cbs::delay_from_credits` |
| `TCA_CBS_PARMS` | UAPI per-qdisc params | UAPI |

### compatibility contract

REQ-1: Per-tc UAPI tc_cbs_qopt:
- `offload`: bool (request HW offload).
- `hicredit`: s32 max-credits (bytes).
- `locredit`: s32 min-credits (bytes).
- `idleslope`: s32 idle credit-refill rate (bytes/s).
- `sendslope`: s32 send credit-drain rate (bytes/s; negative).

REQ-2: Per-qdisc state:
- `last`: ktime of last credit-update.
- `credits`: s64 current credits (bytes).
- `port_rate`: u64 port transmit rate.
- `enqueue`, `dequeue`: per-mode function pointers.

REQ-3: Per-formula (RFC: 802.1Qav §34.5):
- sendslope = idleslope - port_rate (always negative).
- hicredit ≥ 0.
- locredit ≤ 0.
- max_interference_size × idleslope / port_rate ≤ hicredit.
- max_frame_size × sendslope / port_rate ≤ locredit (min, since neg).

REQ-4: cbs_dequeue_soft (credit gate):
- Peek inner.skb.
- now = ktime_get_ns().
- If credits < 0:
  - credits_added = timediff_to_credits(now - last, idleslope).
  - credits = min(credits + credits_added, hicredit).
  - If credits < 0:
    - delay = delay_from_credits(credits, idleslope).
    - watchdog_schedule(now + delay).
    - return None.
- Dequeue skb.
- credits -= credits_from_len(skb.len, sendslope, port_rate).
- credits = max(credits, locredit).
- last = now.
- return Some(skb).

REQ-5: cbs_enqueue_soft:
- inner.enqueue(skb).
- Per-qstats backlog += skb.len.

REQ-6: Per-offload mode:
- TC_SETUP_QDISC_CBS ndo command sent to driver.
- Driver programs HW shaper; kernel becomes pass-through.
- enqueue/dequeue both pure inner-passthrough.

REQ-7: timediff_to_credits(diff_ns, slope):
- Return (diff_ns × slope) / NSEC_PER_SEC.

REQ-8: credits_from_len(len, slope, port_rate):
- frame_time_ns = (len × NSEC_PER_SEC) / port_rate.
- Return |slope × frame_time / NSEC_PER_SEC|.

REQ-9: delay_from_credits(deficit, slope):
- Return (-deficit × NSEC_PER_SEC) / slope.

REQ-10: Per-multiqueue-mqprio leaf:
- CBS typically root of leaf-class under MQPRIO root.
- Per-traffic-class has independent CBS (per-AVB-class).

REQ-11: HW offload negotiation:
- offload=1 + driver supports: kernel installs offload mode.
- Else: -EOPNOTSUPP.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `sendslope_negative` | INVARIANT | q.sendslope < 0. |
| `idleslope_lt_port_rate` | INVARIANT | q.idleslope < q.port_rate. |
| `credits_within_bounds` | INVARIANT | q.credits ∈ [q.locredit, q.hicredit]. |
| `hicredit_ge_zero` | INVARIANT | q.hicredit ≥ 0. |
| `locredit_le_zero` | INVARIANT | q.locredit ≤ 0. |

### Layer 2: TLA+

`net/sched/sch_cbs.tla`:
- Per-pkt enqueue + per-tick credit refill + dequeue gating.
- Properties:
  - `safety_no_send_with_negative_credits` — per-dequeue-success ⟹ credits ≥ 0 (pre-debit).
  - `safety_credits_capped` — per-credit update: credits ≤ hicredit ∧ credits ≥ locredit.
  - `liveness_credits_eventually_replenish` — per-idle-time credits → max(prev + idleslope×t, hicredit).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Cbs::dequeue_soft` post: returned-Some ⟹ credits debited; returned-None ⟹ watchdog scheduled | `Cbs::dequeue_soft` |
| `Cbs::init` post: q.credits=0; q.last=now | `Cbs::init` |
| `Cbs::change` post: params updated under lock | `Cbs::change` |
| `Cbs::credits_from_len` post: returned ≥ 0 (since slope is negative, abs taken) | `Cbs::credits_from_len` |

### Layer 4: Verus/Creusot functional

`Per-AVB stream at idleslope rate over interval T → bytes-sent ≤ idleslope × T + (hicredit - locredit) / port_rate × frame_size` semantic equivalence: per-CBS matches IEEE 802.1Qav §34.5 credit law.

### hardening

(Inherits row-1 features from `net/sched/sch-api.md` § Hardening.)

CBS-specific reinforcement:

- **Per-credits clamped to [locredit, hicredit]** — defense against per-overflow runaway send / send-deficit.
- **Per-watchdog absolute-ns** — defense against per-tick jitter.
- **Per-port_rate retrieved per-init** — defense against per-link-speed change after init.
- **Per-offload requires driver support** — defense against per-offload silent-fail.
- **Per-soft mode iota-precision** — defense against per-credit accumulator overflow.
- **Per-frame_time u64 saturating** — defense against per-pkt-len × port_rate overflow.
- **Per-cbs_change atomic-swap params** — defense against per-config torn read.
- **Per-MQPRIO leaf isolated state** — defense against per-class cross-leak.
- **Per-init validates sendslope = idleslope - port_rate** — defense against per-config inconsistent.
- **Per-AVB-class delay-budget enforced via locredit** — defense against per-pkt deferral exceeding budget.

