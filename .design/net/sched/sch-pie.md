# Tier-3: net/sched/sch_pie.c — PIE (Proportional Integral controller Enhanced) qdisc

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/sched/sch-api.md
upstream-paths:
  - net/sched/sch_pie.c (~583 lines)
  - include/net/pie.h (PIE state)
  - include/uapi/linux/pkt_sched.h (PIE UAPI)
-->

## Summary

PIE (Proportional Integral controller Enhanced; RFC 8033) is AQM that maintains low queueing delay via per-period (default 16ms) PI controller updates of a per-pkt drop probability. Per-period: estimates queuing delay from per-skb sojourn samples; computes drop_prob proportional to delay-target deviation + integral derivative. Per-pkt enqueue probabilistically drops based on current drop_prob (bypassed in burst-allowance window). Per-skb time-stamped at enqueue; per-dequeue measures sojourn for next-period estimate. Critical for: low-latency AQM in low-power devices where CoDel's per-pkt cost is too high (DOCSIS, broadband CPE).

This Tier-3 covers `sch_pie.c` (~583 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct pie_sched_data` | per-qdisc state | `PieSchedData` |
| `struct pie_params` | per-config | `PieParams` |
| `struct pie_vars` | per-state machine | `PieVars` |
| `struct pie_stats` | per-counter | `PieStats` |
| `pie_init()` | per-qdisc init | `Pie::init` |
| `pie_destroy()` | per-qdisc destroy | `Pie::destroy` |
| `pie_enqueue()` | per-pkt drop-or-enqueue | `Pie::enqueue` |
| `pie_qdisc_dequeue()` | per-pkt dequeue + sample | `Pie::dequeue` |
| `pie_change()` | per-qdisc reconfigure | `Pie::change` |
| `pie_calculate_probability()` | per-period PI controller update | `Pie::calculate_probability` |
| `pie_drop_early()` | per-pkt drop-decision | `Pie::drop_early` |
| `pie_process_dequeue()` | per-pkt sojourn-sample | `Pie::process_dequeue` |
| `TCA_PIE_TARGET` (15ms) | per-config delay target | UAPI |
| `TCA_PIE_TUPDATE` (16ms) | per-period | UAPI |
| `TCA_PIE_BETA` / `ALPHA` | PI controller gains | UAPI |

## Compatibility contract

REQ-1: Per-tc UAPI tc_pie_qopt:
- `target`: u32 target delay (us; default 15ms = 15000us).
- `tupdate`: u32 update period (us; default 15ms = 15000us).
- `limit`: pkts queue cap.
- `alpha`: u32 PI integral gain.
- `beta`: u32 PI proportional gain.
- `ecn`: bool ECN-mark.

REQ-2: Per-period PI controller:
- delay_estimate = qdelay_old (last sample).
- delta_p = alpha × (delay_estimate - target) + beta × (delay_estimate - delay_old).
- drop_prob_new = drop_prob_old + delta_p.
- Clamped to [0, MAX_PROB].

REQ-3: Per-pkt drop_early:
- If burst_protection: return false.
- If drop_prob ≤ 0.2 ∧ qdelay < target/2: return false (no drop).
- If drop_prob ≤ MAX_DROP_PROB ∧ qdelay > target × 2: return true (force drop).
- Else: random_u32() < drop_prob × U32_MAX → drop.

REQ-4: Per-skb timestamp:
- skb.tstamp = ktime_get_ns() at enqueue.
- Per-dequeue: sojourn = now - skb.tstamp; update qdelay sample.

REQ-5: Per-burst window:
- After enqueue-burst-clear: burst_time decremented per period.
- During burst: drop_prob effectively zero.

REQ-6: PI controller scaling:
- alpha / beta: 0..32 (Q-format scaling).
- Per-RFC: alpha = 0.125, beta = 1.25.

REQ-7: ECN-mark integration:
- If params.ecn ∧ INET_ECN_set_ce(skb): mark; pass through.
- Else: drop.

REQ-8: Per-period timer:
- HRTimer fires every `tupdate`.
- pie_calculate_probability invoked.

REQ-9: pie_qdisc_dequeue:
- skb = inner.dequeue(); if !skb: return None.
- pie_process_dequeue(skb): sojourn = now - skb.tstamp; update qdelay.
- Return skb.

REQ-10: Per-MQRED-PIE:
- Multiple inner qdiscs sharing single PIE state; or per-queue PIE.

## Acceptance Criteria

- [ ] AC-1: `tc qdisc add dev eth0 root pie target 15ms tupdate 15ms`: PIE allocated.
- [ ] AC-2: Per-skb sojourn ≪ target: drop_prob → 0; pass.
- [ ] AC-3: Per-skb sojourn ≫ target: drop_prob increases; per-pkt drops.
- [ ] AC-4: Per-period PI update: drop_prob adjusts proportionally.
- [ ] AC-5: ECN-mark variant: mark instead of drop on ECT-capable.
- [ ] AC-6: Burst window: no drops during initial burst-clear.
- [ ] AC-7: drop_prob clamped to [0, 1.0].
- [ ] AC-8: pie_change updates target/tupdate atomically.
- [ ] AC-9: HRTimer fires per tupdate.
- [ ] AC-10: tc -s qdisc show: drop_prob + delay stats visible.

## Architecture

Per-qdisc state:

```
struct PieSchedData {
  params: PieParams,
  vars: PieVars,
  stats: PieStats,
  qdisc: &Qdisc,                                 // inner
  adapt_timer: HrTimer,
}

struct PieParams {
  target: u32,                                   // ns (15_000_000)
  tupdate: u32,                                  // ns (15_000_000)
  alpha: u32,
  beta: u32,
  ecn: bool,
  bytemode: bool,
  dq_rate_estimator: bool,
}

struct PieVars {
  qdelay: u32,                                   // last sample (ns)
  qdelay_old: u32,
  burst_time: u32,
  drop_prob: u64,                                // Q40 fixed-point
  accu_prob: u64,
  dq_count: u64,                                 // bytes for rate estimate
  dq_tstamp: KtimeT,
  avg_dq_rate: u32,
  rate_check: bool,
}
```

`Pie::enqueue(sch, skb, to_free) -> Result<()>`:
1. If sch.qstats.qlen >= q.params.limit: drop; return Err.
2. drop = Pie::drop_early(sch, skb).
3. If drop:
   - If params.ecn ∧ INET_ECN_set_ce(skb): mark; pass to inner.
   - Else: qdisc_drop(skb, sch, to_free); return Err.
4. skb.tstamp = ktime_get_ns().
5. q.qdisc.enqueue(skb)?.
6. sch.qstats.backlog += skb.len.
7. Ok.

`Pie::drop_early(sch, skb) -> bool`:
1. If vars.burst_time > 0: return false.
2. If vars.drop_prob == 0: return false.
3. If vars.qdelay < params.target / 2 ∧ vars.drop_prob < (MAX_PROB / 5): return false.
4. If vars.qdelay > 2 × params.target: return true.
5. r = random_u32().
6. Return r < (vars.drop_prob >> SHIFT_TO_U32).

`Pie::dequeue(sch) -> Option<&Skb>`:
1. skb = q.qdisc.dequeue(); if !skb: return None.
2. Pie::process_dequeue(sch, skb).
3. Return skb.

`Pie::process_dequeue(sch, skb)`:
1. now = ktime_get_ns().
2. sojourn = now - skb.tstamp.
3. vars.qdelay = sojourn (or per-dq_rate_estimator path).

`Pie::calculate_probability(timer)`:
1. delta = (vars.qdelay - params.target) × params.alpha + (vars.qdelay - vars.qdelay_old) × params.beta.
2. vars.drop_prob = clamp(vars.drop_prob + delta, 0, MAX_PROB).
3. vars.qdelay_old = vars.qdelay.
4. If vars.burst_time > 0: vars.burst_time -= params.tupdate.
5. Re-arm timer at now + tupdate.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `drop_prob_in_range` | INVARIANT | vars.drop_prob ∈ [0, MAX_PROB]. |
| `target_pos` | INVARIANT | params.target > 0. |
| `tupdate_pos` | INVARIANT | params.tupdate > 0. |
| `qdelay_non_negative` | INVARIANT | vars.qdelay ≥ 0. |
| `burst_time_decreasing` | INVARIANT | per-period: vars.burst_time -= tupdate (saturating). |

### Layer 2: TLA+

`net/sched/sch_pie.tla`:
- Per-period PI update + per-pkt drop_early + per-skb sojourn.
- Properties:
  - `safety_drop_prob_increases_under_overload` — per-period qdelay > target ⟹ drop_prob ↑.
  - `safety_drop_prob_decreases_under_idle` — per-period qdelay < target ⟹ drop_prob ↓.
  - `liveness_target_reached` — per-PI converges qdelay → target.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Pie::drop_early` post: returned-true ⟹ probabilistic-or-force-drop conditions met | `Pie::drop_early` |
| `Pie::calculate_probability` post: drop_prob clamped; qdelay_old = qdelay | `Pie::calculate_probability` |
| `Pie::process_dequeue` post: vars.qdelay updated to current sample | `Pie::process_dequeue` |
| `Pie::enqueue` post: skb in inner OR dropped/marked per drop_early | `Pie::enqueue` |

### Layer 4: Verus/Creusot functional

`Per-overload qdelay → PI controller updates drop_prob → pkts probabilistically dropped → qdelay drives toward target` semantic equivalence: per-PIE matches RFC 8033 PI control law.

## Hardening

(Inherits row-1 features from `net/sched/sch-api.md` § Hardening.)

PIE-specific reinforcement:

- **Per-period drop_prob clamped** — defense against per-overflow runaway.
- **Per-burst-protection prevents over-drop** — defense against per-startup over-shoot.
- **Per-skb tstamp atomic-set** — defense against per-set race.
- **Per-pkt random gating** — defense against per-pkt deterministic-drop attack.
- **Per-period PI gains validated** — defense against per-config insane gains causing oscillation.
- **Per-ECN requires CE-capable** — defense against per-non-ECN traffic mark.
- **Per-pie_change atomic-swap** — defense against per-config torn read.
- **Per-tupdate ≥ minimum** — defense against per-tick excessive PI update.
- **Per-bytemode optional** — defense against per-pkt-vs-byte rate confusion.
- **Per-MAX_PROB constant** — defense against per-config invalid range.

## Grsecurity/PaX-style Reinforcement

Baseline PaX/grsecurity mitigations applicable to the PIE qdisc data path:

- **PAX_USERCOPY** — bounded `copy_{to,from}_user` for tc-netlink PIE parameter blobs (`TCA_PIE_*`), rejecting whitelist overruns on stat dumps and config sets.
- **PAX_KERNEXEC** — PIE classify and dequeue paths execute from W^X .text; no JIT, no patched trampolines.
- **PAX_RANDKSTACK** — per-syscall kernel-stack randomization frustrates PIE-tick / drop-decision timing inference.
- **PAX_REFCOUNT** — `Qdisc`, inner `Qdisc`, and `Qdisc_class_ops` use saturating refcounts; PIE attach/graft cannot wrap on adversarial RTNL storms.
- **PAX_MEMORY_SANITIZE** — `pie_sched_data`, vars (`drop_prob`, `prob`, `qdelay_old`) and the per-skb timestamp shadow are sanitized on free.
- **PAX_UDEREF** — netlink attribute parsing for `tc qdisc add ... pie` traverses user pointers only under UDEREF, no speculative deref.
- **PAX_RAP / kCFI** — `Qdisc_ops` for PIE (`pie_qdisc_ops`) is `static const`; enqueue/dequeue/dump dispatch is type-checked at call sites.
- **GRKERNSEC_HIDESYM** — PIE-internal symbols (`pie_drop_early`, `calculate_probability`) hidden from non-CAP_SYSLOG callers.
- **GRKERNSEC_DMESG** — drop-probability anomalies are rate-limited and gated by CAP_SYSLOG.

PIE-specific reinforcement:

- **TC qdisc CAP_NET_ADMIN** — `tc qdisc {add,change,del} ... pie` requires CAP_NET_ADMIN in the netns owning the netdev.
- **`pie_qdisc_ops` PAX_RAP-typed** — function-pointer mismatch on dispatch faults instead of speculatively executing.
- **Per-tupdate watchdog rate-limit bounded** — PI-controller tick cannot be driven below the configured minimum from userspace.
- **`drop_prob` clamp under PAX_REFCOUNT-equivalent saturating arithmetic** — adversarial enqueue cannot wrap probability accumulator.
- **GRKERNSEC_HIDESYM on PIE internal probes** — kprobe/tracepoint surface for `pie_calculate_probability` gated.

Rationale: PIE is a per-flow AQM running in softirq with attacker-controlled enqueue rate; clamping probability arithmetic, sanitizing per-skb shadow timestamps, and CFI-protecting the Qdisc_ops vtable closes the AQM-tampering surface that grsec historically hardens for sch-api children.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- TC qdisc API (covered in `sch-api.md` Tier-3)
- CoDel (covered in `sch-codel.md` Tier-3)
- FQ-CoDel (covered in `sch-fq-codel.md` Tier-3)
- Implementation code
