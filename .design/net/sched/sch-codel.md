# Tier-3: net/sched/sch_codel.c — CoDel (Controlled Delay) qdisc

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/sched/sch-api.md
upstream-paths:
  - net/sched/sch_codel.c (~286 lines)
  - include/net/codel.h (~167 lines)
  - include/net/codel_impl.h (state machine)
  - include/uapi/linux/pkt_sched.h (CoDel UAPI)
-->

## Summary

CoDel (Controlled Delay; Nichols-Jacobson 2012) is parameterless AQM that targets a per-pkt sojourn-time (default 5ms). Per-skb time-stamped at enqueue; per-skb dequeue checks `now - enqueue_time` ≥ target → "above-target" state. While above-target ≥ `interval` (100ms): start dropping with progressively shrinking inter-drop times (`interval / sqrt(count)`). Per-good state: count cleared. Self-tuning, no thresholds. Critical for: bufferbloat mitigation; baseline AQM in modern Linux distros (preferred over RED).

This Tier-3 covers `sch_codel.c` (~286 lines) + `include/net/codel.h` (~167 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct codel_sched_data` | per-qdisc state | `CodelSchedData` |
| `struct codel_params` | target + interval + ce_threshold | `CodelParams` |
| `struct codel_vars` | per-qdisc state machine | `CodelVars` |
| `struct codel_stats` | per-qdisc stats | `CodelStats` |
| `codel_init()` | per-qdisc init | `Codel::init` |
| `codel_destroy()` | per-qdisc destroy | `Codel::destroy` |
| `codel_enqueue()` | per-pkt timestamp + enqueue | `Codel::enqueue` |
| `codel_dequeue()` | per-pkt sojourn-check + drop-or-pass | `Codel::dequeue` |
| `codel_change()` | per-qdisc reconfigure | `Codel::change` |
| `codel_should_drop()` | per-pkt above-target test | `Codel::should_drop` |
| `codel_dequeue_func` | per-pkt inner-dequeue callback | `CodelDequeueFn` |
| `codel_control_law()` | per-state inter-drop time | `Codel::control_law` |
| `TCA_CODEL_TARGET` (5ms) | per-config target | UAPI |
| `TCA_CODEL_INTERVAL` (100ms) | per-config interval | UAPI |
| `TCA_CODEL_CE_THRESHOLD` (optional) | per-skb ECN-mark threshold | UAPI |

## Compatibility contract

REQ-1: Per-tc UAPI tc_codel_qopt:
- `target`: u32 sojourn-time target (microseconds; default 5ms = 5000us).
- `interval`: u32 control-loop interval (microseconds; default 100ms).
- `limit`: pkts queue cap.
- `ecn`: bool ECN-mark instead of drop.
- `ce_threshold`: optional u32 sojourn for forced CE-mark (lower than drop).

REQ-2: Per-skb timestamp:
- skb.tstamp = ktime_get_ns() at enqueue.
- Per-dequeue: sojourn = ktime_get_ns() - skb.tstamp.

REQ-3: codel_should_drop(skb, sch, vars, params, now):
- sojourn = now - skb.tstamp.
- If sojourn < params.target ∨ sch.qstats.backlog ≤ params.target_pkt_size: return False; vars.first_above_time = 0.
- Else (sojourn ≥ target):
  - If vars.first_above_time == 0: vars.first_above_time = now + interval.
  - Elif now ≥ vars.first_above_time: return True.
  - Else: return False.

REQ-4: codel_dequeue (state machine):
- vars.dropping = current state.
- Per-state DROPPING:
  - If !should_drop: dropping = 0.
  - Elif now ≥ vars.drop_next:
    - drop skb (or ECN-mark).
    - vars.count++.
    - vars.drop_next = control_law(vars.drop_next, params.interval, vars.count).
    - retry dequeue.
- Per-state NON-DROPPING:
  - If should_drop:
    - drop skb (or ECN-mark).
    - retry-dequeue: skb_next.
    - if vars.count > 2 ∧ now - vars.drop_next < 16 × interval: vars.count -= 2.
    - else: vars.count = 1.
    - dropping = 1.
    - vars.drop_next = control_law(now, interval, vars.count).
- Return dequeued skb.

REQ-5: control_law(t, interval, count):
- Return t + interval / sqrt(count).
- Implementation: codel_inv_sqrt(count) precomputed table (256 entries).

REQ-6: ECN-mark integration:
- If params.ecn ∧ INET_ECN_set_ce(skb): mark instead of drop; pass skb.

REQ-7: ce_threshold mode:
- If sojourn ≥ ce_threshold ∧ INET_ECN_set_ce(skb): force-mark even before above-target drop.

REQ-8: Per-state vars:
- count: drop-history.
- lastcount: prev-state count for re-entry boost.
- dropping: current state bit.
- rec_inv_sqrt: cached 1/sqrt(count) (u16; precomputed).
- first_above_time: time-of-first-target-violation.
- drop_next: time-of-next-drop (when in dropping state).

REQ-9: Per-FQ-CoDel composition:
- FQ-CoDel uses CoDel as inner per-flow qdisc.
- Each flow has independent codel_vars.

REQ-10: Per-newton-iteration inv_sqrt:
- vars.rec_inv_sqrt updated on count++ via Newton-Raphson:
  - new = (rec * (3 - count * rec * rec)) / 2.

## Acceptance Criteria

- [ ] AC-1: `tc qdisc add dev eth0 root codel target 5ms interval 100ms`: CoDel allocated.
- [ ] AC-2: Per-skb sojourn < 5ms: pass through.
- [ ] AC-3: Per-skb sojourn ≥ 5ms continuously for 100ms: enter dropping state.
- [ ] AC-4: In dropping state: drops at intervals = 100ms / sqrt(count).
- [ ] AC-5: After good period: state reverts to non-dropping; count cleared.
- [ ] AC-6: ECN-mark variant: instead of drop, mark CE bit on ECT-capable skbs.
- [ ] AC-7: ce_threshold (e.g. 1ms): pre-drop CE-mark at lower sojourn.
- [ ] AC-8: control_law: drop-times follow inverse-sqrt curve.
- [ ] AC-9: Re-entry within 16 × interval: count reduced by 2 (not reset to 1).
- [ ] AC-10: codel_change updates target/interval atomically.

## Architecture

Per-qdisc state:

```
struct CodelSchedData {
  params: CodelParams,
  vars: CodelVars,
  stats: CodelStats,
  drop_overlimit: u32,
  qdisc: &Qdisc,                                 // inner (default pfifo)
}

struct CodelParams {
  target: u32,                                   // ns (5_000_000)
  ce_threshold: u32,                             // ns (optional)
  interval: u32,                                 // ns (100_000_000)
  ecn: bool,
  target_pkt_size: u32,                          // soft minimum backlog
}

struct CodelVars {
  count: u32,                                    // drop-history
  lastcount: u32,
  rec_inv_sqrt: u16,                             // 1/sqrt(count)
  dropping: bool,
  first_above_time: KtimeT,
  drop_next: KtimeT,
  ldelay: u32,                                   // last sojourn observed
}

struct CodelStats {
  maxpacket: u32,
  drop_count: u32,
  drop_len: u32,
  ecn_mark: u32,
  ce_mark: u32,
}
```

`Codel::should_drop(skb, sch, vars, params, now) -> bool`:
1. sojourn = now - skb.tstamp.
2. vars.ldelay = sojourn.
3. If sojourn < params.target ∨ sch.qstats.backlog ≤ params.target_pkt_size:
   - vars.first_above_time = 0; return false.
4. If vars.first_above_time == 0:
   - vars.first_above_time = now + params.interval.
   - return false.
5. If now ≥ vars.first_above_time: return true.
6. Return false.

`Codel::control_law(t, interval, rec_inv_sqrt) -> KtimeT`:
1. Return t + reciprocal_scale(interval, rec_inv_sqrt).

`Codel::Newton_step(vars)`:
1. invsqrt = vars.rec_inv_sqrt.
2. new = (invsqrt * (3 - vars.count * invsqrt² / 0x10000)) / 2.
3. vars.rec_inv_sqrt = new.

`Codel::dequeue(sch) -> Option<&Skb>`:
1. now = ktime_get_ns().
2. skb = q.qdisc.dequeue(); if !skb: vars.dropping=false; return None.
3. drop = Codel::should_drop(skb, sch, vars, params, now).
4. If vars.dropping:
   - If !drop: vars.dropping = false.
   - Elif now ≥ vars.drop_next:
     - drop_or_mark(skb, params).
     - vars.count++; Codel::Newton_step(vars).
     - vars.drop_next = Codel::control_law(vars.drop_next, params.interval, vars.rec_inv_sqrt).
     - skb = q.qdisc.dequeue(); if !skb: return None.
5. Elif drop:
   - drop_or_mark(skb, params).
   - skb = q.qdisc.dequeue(); if !skb: return None.
   - if vars.count > 2 ∧ now - vars.drop_next < 16 × params.interval: vars.count -= 2.
   - else: vars.count = 1.
   - Codel::Newton_step(vars).
   - vars.dropping = true.
   - vars.drop_next = Codel::control_law(now, params.interval, vars.rec_inv_sqrt).
6. If params.ce_threshold ∧ vars.ldelay > params.ce_threshold ∧ INET_ECN_set_ce(skb): stats.ce_mark++.
7. Return skb.

`Codel::enqueue(sch, skb, to_free) -> Result<()>`:
1. skb.tstamp = ktime_get_ns().
2. q.qdisc.enqueue(skb)?
3. sch.qstats.backlog += skb.len.
4. Ok.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `count_ge_zero` | INVARIANT | vars.count ≥ 0. |
| `rec_inv_sqrt_le_max` | INVARIANT | vars.rec_inv_sqrt ≤ u16::MAX. |
| `drop_next_increases_in_dropping` | INVARIANT | per-drop-while-dropping: drop_next' > drop_next. |
| `first_above_time_zero_when_below` | INVARIANT | sojourn < target ⟹ first_above_time = 0. |
| `dropping_implies_count_gt_0` | INVARIANT | vars.dropping ⟹ vars.count > 0. |

### Layer 2: TLA+

`net/sched/sch_codel.tla`:
- Per-skb enqueue + per-dequeue state machine.
- Properties:
  - `safety_no_drop_below_target` — per-skb sojourn < target ⟹ no drop.
  - `safety_drop_rate_decreases_with_count` — control_law(t, count+1) - control_law(t, count) decreases.
  - `liveness_long_overload_eventually_drops` — per-sojourn ≥ target for ≥ interval ⟹ drop.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Codel::should_drop` post: returned-true ⟹ sojourn ≥ target ∧ now ≥ first_above_time | `Codel::should_drop` |
| `Codel::dequeue` post: per-state-transition consistent with should_drop | `Codel::dequeue` |
| `Codel::control_law` post: returned-time = t + interval × rec_inv_sqrt | `Codel::control_law` |
| `Codel::Newton_step` post: rec_inv_sqrt converges toward 1/sqrt(count) | `Codel::Newton_step` |

### Layer 4: Verus/Creusot functional

`Per-bufferbloat workload → CoDel keeps per-pkt sojourn ≈ target via inverse-sqrt drop schedule` semantic equivalence: per-CoDel matches Nichols-Jacobson 2012 control law.

## Hardening

(Inherits row-1 features from `net/sched/sch-api.md` § Hardening.)

CoDel-specific reinforcement:

- **Per-skb tstamp set at enqueue under sch.lock** — defense against per-tstamp race.
- **Per-vars u32 count saturating** — defense against u32-overflow on long-overload.
- **Per-Newton step bounded iters** — defense against iteration-divergence.
- **Per-control_law u64 monotonic** — defense against ktime overflow.
- **Per-count = 1 reset on long-good-period** — defense against per-state stuck-dropping.
- **Per-ECN-mark requires CE-capable** — defense against per-non-ECN traffic mark.
- **Per-ce_threshold validated < target** — defense against per-config inversion.
- **Per-codel_change atomic-swap** — defense against per-config torn read.
- **Per-Newton inv_sqrt precomputed table fallback** — defense against per-init Newton not yet converged.
- **Per-AQM sojourn-based** — defense against per-burst-spike that traditional RED would over-drop.

## Grsecurity/PaX-style Reinforcement

Beyond the Row-1 defaults above, sch_codel (RFC 8289) inherits the following PaX/grsec primitives:

- **PAX_USERCOPY**: TCA_CODEL_* attributes parsed via `nla_*` whitelisted accessors.
- **PAX_KERNEXEC**: `codel_qdisc_ops` is `__ro_after_init`; W^X enforced on the AQM dispatch path.
- **PAX_RANDKSTACK**: RTM_NEWQDISC + per-packet enqueue/dequeue (softirq) protected.
- **PAX_REFCOUNT**: `Qdisc.refcnt`, `q->stats.drops`, `q->stats.ecn_mark` use checked `refcount_t` / `u64_stats_*`.
- **PAX_MEMORY_SANITIZE**: freed `codel_sched_data` and per-packet timestamp scratch zeroed.
- **PAX_UDEREF**: rtnetlink + skb walkers fenced.
- **PAX_RAP / kCFI**: `Qdisc_ops` callbacks type-tagged + CFI-checked.
- **GRKERNSEC_HIDESYM**: CoDel internal symbols hidden from `/proc/kallsyms`.
- **GRKERNSEC_DMESG**: CoDel parse-failure printks rate-limited, CAPSYSLOG-gated.

Component-specific reinforcement:

- RTM_NEWQDISC attaching CoDel requires **CAP_NET_ADMIN** in the netns owner.
- `target`, `interval`, `ce_threshold` clamped: `target ∈ [1us, 1s]`, `interval ∈ [1us, 1s]`; out-of-range fails `-EINVAL`.
- `limit` (packets) clamped to `[1, 1<<20]`; prevents unbounded `qlen` and the associated `kmalloc` pressure on attach.
- Drop / ECN-mark policy: when `qdelay > target` for `> interval`, drop or mark deterministically; no attacker-controllable branch into the dropper from a non-`ECN_TX_OK` path.
- Per-Qdisc drop counter uses `u64_stats_*` saturating; no wrap-on-overflow gives a false low-drop signal.

Rationale: CoDel is the canonical AQM applied to bufferbloat-prone egress queues; an unchecked `limit` or wrap on the drop counter would either OOM the box or hide a flood. Bounding the four config knobs plus saturating stats keeps the AQM honest, and PaX + RAP/kCFI keep its dispatch fail-closed.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- TC qdisc API (covered in `sch-api.md` Tier-3)
- FQ-CoDel (covered in `sch-fq-codel.md` Tier-3)
- PIE (Proportional Integral Enhanced; covered separately)
- Implementation code
