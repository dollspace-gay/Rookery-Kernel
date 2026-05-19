# Tier-3: net/sched/sch_red.c — RED (Random Early Detection) + ARED qdisc

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/sched/sch-api.md
upstream-paths:
  - net/sched/sch_red.c (~580 lines)
  - include/net/red.h (RED parameters / EWMA)
  - include/uapi/linux/pkt_sched.h (RED UAPI)
-->

## Summary

RED (Random Early Detection; Floyd-Jacobson 1993) probabilistically drops/marks packets when avg-queue-length crosses thresholds, *before* tail-drop, to provide AQM (Active Queue Management) signal to TCP-cong. Per-qdisc EWMA-tracks `qavg` (over `Wq` weight); per-(min, max) thresholds: below `min` no drop; between `min` and `max` linearly increasing prob drop (max-prob at `max`); above `max` always drop ("hard"). ARED extension auto-tunes `max_p` adaptively. ECN-mode: marks (CE bit) instead of drops. GRED extension: per-VQ + DSCP-classified tracking. Critical for: TCP-friendly AQM. Modern kernels prefer CoDel/PIE but RED is still reference.

This Tier-3 covers `sch_red.c` (~580 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct red_sched_data` | per-qdisc state | `RedSchedData` |
| `struct red_parms` | per-qdisc EWMA + thresholds | `RedParms` |
| `struct red_vars` | per-qdisc avg-queue + count | `RedVars` |
| `red_init()` | per-qdisc init | `Red::init` |
| `red_destroy()` | per-qdisc destroy | `Red::destroy` |
| `red_enqueue()` | per-pkt enqueue (RED logic) | `Red::enqueue` |
| `red_dequeue()` | per-pkt dequeue | `Red::dequeue` |
| `red_drop()` | tail-drop fallback | `Red::drop` |
| `red_change()` | per-qdisc reconfigure | `Red::change` |
| `red_set_parms()` | precompute Wq + max_p | `Red::set_parms` |
| `red_calc_qavg()` | per-pkt EWMA update | `Red::calc_qavg` |
| `red_action()` | per-pkt drop/mark/pass decision | `Red::action` |
| `RED_HARD_MARK` / `RED_PROB_MARK` / `RED_DONT_MARK` | per-decision | shared |
| `red_adaptive_algo()` | per-ARED adapt max_p | `Red::adaptive_algo` |
| `TCA_RED_PARMS` / `TCA_RED_STAB` / `TCA_RED_FLAGS` | UAPI attrs | UAPI |

## Compatibility contract

REQ-1: Per-tc UAPI tc_red_qopt:
- `limit`: queue cap (bytes/pkts).
- `qth_min`: min threshold (bytes).
- `qth_max`: max threshold (bytes).
- `Wlog`: log2(1/Wq) (EWMA averaging-period).
- `Plog`: log2(qth_max - qth_min) related shift.
- `Scell_log`: idle-period scale.
- `flags`: ECN / HARDMARK / ADAPTATIVE.

REQ-2: RED parameters (red_parms):
- `qth_min`, `qth_max`, `qth_delta` (= qth_max - qth_min).
- `Scell_log`, `Stab[256]`: per-idle scaling (after busy → idle).
- `target_min`, `target_max`: ARED target.

REQ-3: RED EWMA (red_calc_qavg):
- `qavg = (qavg * (1 - Wq)) + (cur_qlen * Wq)`.
- Implementation: `qavg += (cur_qlen - qavg) >> Wlog`.

REQ-4: red_action (per-pkt decision):
- If qavg < qth_min: RED_DONT_MARK.
- Elif qavg ≥ qth_max: RED_HARD_MARK (drop or mark).
- Else: probability `p = max_p × (qavg - qth_min) / qth_delta`.
  - Increment `count`; effective_p = p / (1 - count × p) (Floyd anti-burst).
  - If random < effective_p: RED_PROB_MARK; reset count.
  - Else: count++.

REQ-5: Per-flag ECN:
- If RED_PROB_MARK ∧ ECN ∧ skb has CE-capable ECN bit: set CE; pass.
- Else: drop.

REQ-6: red_enqueue:
- Compute qavg via red_calc_qavg.
- decision = red_action(qavg).
- switch decision:
  - DONT_MARK: inner.enqueue(skb).
  - PROB_MARK: if ECN ∧ INET_ECN_set_ce(skb): inner.enqueue (mark); else: drop.
  - HARD_MARK: drop.

REQ-7: red_dequeue:
- skb = inner.dequeue().
- If !skb: idle marker; set red_vars.qidlestart = ktime_get_ns().
- Return skb.

REQ-8: Per-idle Stab adjustment:
- After idle: m = (now - qidlestart) >> Scell_log.
- qavg ×= (1 - Wq)^m → use Stab[m] precomputed table.

REQ-9: Per-ARED:
- Periodic timer (~500ms) adjusts max_p: if qavg > target_max: max_p ×= 1.5; if qavg < target_min: max_p /= 1.5.

REQ-10: Per-multiqueue-MQRED:
- Multiple inner-qdiscs; per-queue VQ tracking.

## Acceptance Criteria

- [ ] AC-1: `tc qdisc add dev eth0 root red limit 100k min 10k max 30k avpkt 1500 burst 50 ecn`: RED allocated.
- [ ] AC-2: At qavg < qth_min: enqueue passes.
- [ ] AC-3: At qavg ≥ qth_max: enqueue drops.
- [ ] AC-4: Between qth_min and qth_max: per-pkt random drop with linear prob.
- [ ] AC-5: ECN flag + CE-capable skb: mark instead of drop.
- [ ] AC-6: Anti-burst: count tracked; consecutive drops more spread out.
- [ ] AC-7: After idle: qavg decays per Stab table.
- [ ] AC-8: ARED auto-tunes max_p toward target.
- [ ] AC-9: tc -s qdisc show dev eth0: per-stat early/marked/forced/pdrop counts.
- [ ] AC-10: red_change updates parameters atomically.

## Architecture

Per-qdisc state:

```
struct RedSchedData {
  parms: RedParms,
  vars: RedVars,
  flags: RedFlags,                                // ECN | HARDMARK | ADAPTATIVE
  userbits: u32,
  limit: u32,
  qstats: RedStats,
  qdisc: &Qdisc,                                  // inner (default pfifo)
  adapt_timer: HrTimer,
}

struct RedParms {
  qth_min: u32,
  qth_max: u32,
  Wlog: u8,                                       // log2(1/Wq)
  Plog: u8,
  Scell_log: u8,
  Stab: [u8; 256],                                // per-idle decay table
  max_P: u32,                                     // max-mark prob
  max_P_reciprocal: u32,
  qth_delta: u32,
  target_min: u32,                                // ARED
  target_max: u32,
}

struct RedVars {
  qcount: i32,
  qR: u32,                                        // last random
  qavg: u32,                                      // EWMA queue length
  qidlestart: KtimeT,                             // start-of-idle
}
```

`Red::set_parms(parms, qth_min, qth_max, Wlog, Plog, Scell_log, Stab, max_P)`:
1. parms.qth_min = qth_min.
2. parms.qth_max = qth_max.
3. parms.qth_delta = qth_max - qth_min.
4. parms.Wlog = Wlog.
5. parms.Plog = Plog.
6. parms.max_P = max_P.
7. memcpy(parms.Stab, Stab, 256).

`Red::calc_qavg(parms, vars, backlog) -> u32`:
1. now = ktime_get_ns().
2. If vars.qidlestart != 0:
   - m = (now - vars.qidlestart) >> parms.Scell_log.
   - If m >= 256: vars.qavg = 0.
   - Else: vars.qavg = (vars.qavg * (1 - parms.Stab[m])) >> 8.
   - vars.qidlestart = 0.
3. vars.qavg += (backlog - vars.qavg) >> parms.Wlog.
4. Return vars.qavg.

`Red::action(parms, vars, qavg) -> RedAction`:
1. If qavg < parms.qth_min: vars.qcount = -1; return DONT_MARK.
2. If qavg >= parms.qth_max: vars.qcount = -1; return HARD_MARK.
3. vars.qcount++.
4. p = (parms.max_P × (qavg - parms.qth_min)) / parms.qth_delta.
5. effective_p = p / (1 - vars.qcount × p).  (Floyd anti-burst)
6. If random_u32() < effective_p × U32_MAX:
   - vars.qcount = 0.
   - Return PROB_MARK.
7. Return DONT_MARK.

`Red::enqueue(sch, skb, to_free) -> Result<()>`:
1. backlog = sch.qstats.backlog + skb.len.
2. qavg = Red::calc_qavg(&q.parms, &q.vars, backlog).
3. decision = Red::action(&q.parms, &q.vars, qavg).
4. switch decision:
   - DONT_MARK: ret = q.qdisc.enqueue(skb)?.
   - PROB_MARK:
     - q.qstats.prob_mark++.
     - if q.flags.ecn ∧ INET_ECN_set_ce(skb): ret = q.qdisc.enqueue(skb)?.
     - else: q.qstats.prob_drop++; qdisc_drop(skb); return Err.
   - HARD_MARK:
     - q.qstats.forced_mark++.
     - if q.flags.ecn ∧ INET_ECN_set_ce(skb): ret = q.qdisc.enqueue(skb)?.
     - else: q.qstats.forced_drop++; qdisc_drop(skb); return Err.
5. sch.qstats.backlog += skb.len.
6. Ok.

`Red::dequeue(sch) -> Option<&Skb>`:
1. skb = q.qdisc.dequeue();
2. If !skb: q.vars.qidlestart = ktime_get_ns(); return None.
3. sch.qstats.backlog -= skb.len.
4. Return skb.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `qth_min_le_qth_max` | INVARIANT | parms.qth_min ≤ parms.qth_max. |
| `qavg_le_max_backlog` | INVARIANT | per-qavg bounded by max-observed backlog. |
| `decision_well_defined` | INVARIANT | per-action returns DONT_MARK ∨ PROB_MARK ∨ HARD_MARK. |
| `qth_delta_positive` | INVARIANT | parms.qth_delta > 0 (after init). |
| `Stab_idx_lt_256` | INVARIANT | per-idle: m < 256 (else qavg = 0). |

### Layer 2: TLA+

`net/sched/sch_red.tla`:
- Per-qdisc EWMA + decision + drop/mark.
- Properties:
  - `safety_drop_above_max` — qavg ≥ qth_max ⟹ HARD_MARK.
  - `safety_pass_below_min` — qavg < qth_min ⟹ DONT_MARK.
  - `liveness_qavg_decays_idle` — per-idle: qavg decays toward 0.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Red::calc_qavg` post: vars.qavg EWMA-updated; idle-decay applied | `Red::calc_qavg` |
| `Red::action` post: returned decision per-threshold consistent | `Red::action` |
| `Red::enqueue` post: skb in inner-qdisc OR dropped/marked per-decision | `Red::enqueue` |
| `Red::set_parms` post: parms.qth_delta == qth_max - qth_min | `Red::set_parms` |

### Layer 4: Verus/Creusot functional

`Per-burst-traffic over RED → TCP-cong sees ECN-marks before tail-drop → reduces cwnd → fairer share` semantic equivalence: per-RED matches Floyd-Jacobson 1993 AQM signal model.

## Hardening

(Inherits row-1 features from `net/sched/sch-api.md` § Hardening.)

RED-specific reinforcement:

- **Per-qth_min ≤ qth_max enforced** — defense against per-config inverted thresholds.
- **Per-Wlog ∈ [0, 31] enforced** — defense against per-shift undefined.
- **Per-Stab table read-only** — defense against per-update during decay computation.
- **Per-Floyd anti-burst count reset** — defense against burst-loss skew.
- **Per-ECN-mark requires CE-capable** — defense against marking non-ECN traffic.
- **Per-adaptive timer rate-limited** — defense against per-tick uncontrolled max_P drift.
- **Per-VFIO non-coherent DMA: no impact** — RED is per-qdisc, not per-DMA.
- **Per-stat counters atomically incremented** — defense against per-stat race.
- **Per-change atomic-swap parms+vars** — defense against per-change torn read.
- **Per-RED ECN flag CAP_NET_ADMIN-checked** — defense against per-unprivileged ECN-marking.

## Grsecurity/PaX-style Reinforcement

Baseline PaX/grsecurity mitigations applicable to the RED qdisc:

- **PAX_USERCOPY** — `TCA_RED_PARMS` / `TCA_RED_STAB` netlink blobs traverse whitelisted `copy_{to,from}_user` boundaries.
- **PAX_KERNEXEC** — `red_enqueue` and `red_action` execute from W^X .text with no live-patch trampolines on the AQM hot path.
- **PAX_RANDKSTACK** — per-syscall stack randomization frustrates inference of `red_random()` / Floyd-rng state via timing side channels.
- **PAX_REFCOUNT** — RED `Qdisc` and the chained inner qdisc use saturating refcounts on `tc qdisc change` storms.
- **PAX_MEMORY_SANITIZE** — `red_sched_data`, `red_parms`, `red_vars`, and the `stab[]` lookup table are sanitized on free.
- **PAX_UDEREF** — RED tune attribute parsing traverses user pointers only under UDEREF.
- **PAX_RAP / kCFI** — `red_qdisc_ops` is `static const`; ECN-mark and drop callbacks dispatched via CFI-checked indirect calls.
- **GRKERNSEC_HIDESYM** — `red_random`, `red_calc_qavg` hidden from unprivileged kallsyms.
- **GRKERNSEC_DMESG** — RED early-drop / ECN-mark warnings rate-limited via CAP_SYSLOG gating.

RED-specific reinforcement:

- **TC qdisc CAP_NET_ADMIN** — RED create/change/destroy requires CAP_NET_ADMIN in the owning netns.
- **`red_qdisc_ops` PAX_RAP-typed** — dispatch type-checked on every enqueue/dequeue path.
- **Adaptive `max_P` timer rate-limited** — userspace cannot drive the auto-tuning callback faster than its bounded period.
- **Stab table read-only after change** — atomic-swap pattern prevents torn read during decay computation.
- **GRKERNSEC_HIDESYM on Floyd anti-burst counter** — internal counter exposure restricted.

Rationale: RED's security envelope is the qavg accumulator, the Stab lookup, and the ECN-mark gate; under grsec the stab[] read-only-after-publish and bounded adaptive timer remove the two main classes of AQM-tampering vectors.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- TC qdisc API (covered in `sch-api.md` Tier-3)
- GRED (multi-VQ extension; covered separately if needed)
- CoDel / PIE (covered separately)
- Implementation code
