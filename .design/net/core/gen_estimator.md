# Tier-3: net/core/gen_estimator.c — packets-per-sec / bytes-per-sec rate estimator (TC stats)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/core/00-overview.md
upstream-paths:
  - net/core/gen_estimator.c (~280 lines)
  - include/net/gen_stats.h (~84 lines)
-->

## Summary

`gen_estimator` provides a per-(qdisc/class/filter) rate estimator computing exponentially-weighted-moving-average (EWMA) of packets-per-second (pps) and bytes-per-second (bps) over per-tc-class traffic counters. Per-estimator armed with timer at 1, 0.25, 1, 4, ... seconds (configurable interval bucket idx 0..5). Each tick: compute `delta = current - previous; rate = (1-w) * prev_rate + w * (delta / interval)`. Used by `tc class show` to display per-class current rate, and by HTB/HFSC bandwidth-shapers to enforce target rate. Critical for: traffic control quality-of-service.

This Tier-3 covers `gen_estimator.c` (~280 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct net_rate_estimator` | per-(stats, est) state | `NetRateEstimator` |
| `gen_new_estimator()` | per-stats add estimator | `Gen::new_estimator` |
| `gen_kill_estimator()` | per-stats remove | `Gen::kill_estimator` |
| `gen_replace_estimator()` | per-stats reload | `Gen::replace_estimator` |
| `gen_estimator_active()` | per-stats has-estimator | `Gen::estimator_active` |
| `gen_estimator_read()` | per-stats current rate | `Gen::estimator_read` |
| `est_timer()` | per-tick callback | `Gen::est_timer` |
| `gnet_stats_basic_sync` | per-(pkts, bytes) atomic counter | `GnetStatsBasicSync` |
| `gnet_estimator` | per-tc UAPI rate config | UAPI |
| `TCA_RATE` | per-tc-msg attr | UAPI |

## Compatibility contract

REQ-1: Per-tc UAPI gnet_estimator:
- `interval` (s8): per-bucket idx (0..5; -2..3 in Linux semantics).
- `ewma_log` (u8): EWMA averaging factor.
- Per-bucket interval: 0.25s, 0.5s, 1s, 2s, 4s, 8s.

REQ-2: gen_new_estimator(bstats, running, rate_est, lock, est_attr) -> Result:
- bstats: per-class basic-stats (pkts/bytes counter).
- running: optional running-state pointer.
- rate_est: per-stats rate estimator output ptr.
- lock: per-class lock for stats read.
- est_attr: gnet_estimator config.
- Allocates net_rate_estimator; arms per-bucket timer.

REQ-3: Per-tick est_timer callback:
- Read current bstats (atomically); compute deltas.
- bps_rate = (delta_bytes * 8) >> est.intvl; (intvl in shift form).
- pps_rate = delta_pkts >> est.intvl.
- est.last_pps = ((est.last_pps << ewma_log) - est.last_pps + pps_rate) >> ewma_log.
- est.last_bps = ((est.last_bps << ewma_log) - est.last_bps + bps_rate) >> ewma_log.
- Re-arm timer at next interval boundary.

REQ-4: Per-stats read:
- gen_estimator_read(rate_est, sample) → fills sample.bps + sample.pps.

REQ-5: Per-stats kill:
- gen_kill_estimator(rate_est) → cancel timer; free.

REQ-6: Per-stats lock:
- bstats may be per-CPU; per-tick atomic-read.
- est holds rcu_read_lock + bstats_basic_sync rd-lock.

REQ-7: Per-tick scheduler:
- All estimators share per-bucket timers (1 timer per (interval, ewma_log) bucket).
- Per-bucket timer iter walks per-bucket-list of estimators.

REQ-8: Per-EWMA semantics:
- Smaller ewma_log: more responsive (fewer past samples).
- Larger ewma_log: smoother (more past samples).

REQ-9: Per-rate computation overflow:
- 64-bit accumulation; per-tick deltas signed-clamped.
- Per-rate clamped to ≥ 0.

## Acceptance Criteria

- [ ] AC-1: `tc qdisc add dev eth0 root htb` + `tc class add ... rate 100mbit ceil 100mbit estimator 250ms 1sec`: gen_new_estimator allocates per-class.
- [ ] AC-2: After 1s of traffic at ~100Mbit/s: `tc -s class show dev eth0` shows rate ≈ 100Mbit/s.
- [ ] AC-3: gen_kill_estimator after class delete: timer cancelled.
- [ ] AC-4: Per-bucket idx 2 (1s interval): est_timer fires every ~1s.
- [ ] AC-5: gen_replace_estimator: replaces config without reset.
- [ ] AC-6: Per-tick under high-rate: no overflow in u64 accumulator.
- [ ] AC-7: gen_estimator_read returns last-computed pps + bps.
- [ ] AC-8: Per-class with running counter: rate calculation uses running ↔ stopped state.
- [ ] AC-9: kfree_rcu on est: no UAF for concurrent reader.

## Architecture

Per-(stats, est) state:

```
struct NetRateEstimator {
  bstats: &Mutex<GnetStatsBasicSync>,
  ewma_log: u8,
  intvl_log: u8,                                  // bucket idx
  next_jiffies: u64,
  last_packets: u64,
  last_bytes: u64,
  avpps: u32,                                     // last EWMA pps
  avbps: u32,                                     // last EWMA bps
  rcu: RcuHead,
}

struct GnetStatsBasicSync {
  pkts: AtomicU64,
  bytes: AtomicU64,
  // OR per-CPU counters
}
```

Per-bucket timer scheduler:

```
const ESTIMATOR_BUCKETS: usize = 6;              // bucket idx 0..5

struct EstimatorBucket {
  list: Mutex<Vec<&NetRateEstimator>>,
  timer: HrTimer,
}

static BUCKETS: [EstimatorBucket; ESTIMATOR_BUCKETS] = ...;
```

`Gen::new_estimator(bstats, running, rate_est, lock, est_attr) -> Result<()>`:
1. Validate est_attr.interval ∈ [-2, 3]; intvl_log = est_attr.interval + 2.
2. Validate est_attr.ewma_log < 32.
3. Allocate NetRateEstimator; init bstats ref + intvl_log + ewma_log.
4. Read initial bstats: last_packets / last_bytes.
5. *rate_est = est.
6. Add to BUCKETS[intvl_log].list under lock.
7. If !BUCKETS[intvl_log].timer running: arm.

`Gen::est_timer(bucket)`:
1. For each est in bucket.list:
   - Read bstats: cur_packets, cur_bytes.
   - delta_pkts = cur_packets - est.last_packets.
   - delta_bytes = cur_bytes - est.last_bytes.
   - intvl_seconds = 1 << est.intvl_log; (per-bucket: 0.25 = 0; 0.5 = 1; 1 = 2; 2 = 3; 4 = 4; 8 = 5)
   - pps = (delta_pkts << 10) / intvl_in_jiffies.
   - bps = (delta_bytes << 10 * 8) / intvl_in_jiffies.
   - est.avpps = (est.avpps * (1 << est.ewma_log) - est.avpps + pps) >> est.ewma_log.
   - est.avbps = ... similar.
   - est.last_packets = cur_packets; est.last_bytes = cur_bytes.
2. Re-arm timer at next tick.

`Gen::estimator_read(rate_est, sample)`:
1. sample.bps = est.avbps.
2. sample.pps = est.avpps.

`Gen::kill_estimator(rate_est)`:
1. est = *rate_est; *rate_est = NULL.
2. Remove from bucket.list.
3. kfree_rcu(est, rcu).

`Gen::replace_estimator(rate_est, ...)`:
1. Atomically swap est out and re-init with new attr.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `intvl_log_lt_buckets` | INVARIANT | per-est.intvl_log < ESTIMATOR_BUCKETS. |
| `ewma_log_lt_32` | INVARIANT | per-est.ewma_log < 32. |
| `bucket_list_membership` | INVARIANT | per-est in exactly one BUCKETS[intvl_log].list. |
| `delta_non_negative` | INVARIANT | per-tick: cur_packets ≥ last_packets. |
| `rate_non_negative` | INVARIANT | per-tick: avpps ≥ 0; avbps ≥ 0. |

### Layer 2: TLA+

`net/core/gen_estimator.tla`:
- Per-est registration + per-tick computation + per-EWMA update.
- Properties:
  - `safety_no_tick_after_kill` — post-kill: no further est_timer reads est.
  - `safety_ewma_bounded` — per-tick: avpps ≤ max(rate-history); rate-history bounded.
  - `liveness_tick_eventually_runs` — per-est in bucket ⟹ est_timer eventually called.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Gen::new_estimator` post: est in BUCKETS[intvl_log].list; timer armed | `Gen::new_estimator` |
| `Gen::est_timer` post: est.avpps / avbps EWMA-updated; last_* current | `Gen::est_timer` |
| `Gen::estimator_read` post: sample.bps == est.avbps; sample.pps == est.avpps | `Gen::estimator_read` |
| `Gen::kill_estimator` post: est removed from list; freed via RCU | `Gen::kill_estimator` |

### Layer 4: Verus/Creusot functional

`Per-class @ steady-rate R for ≥ 5 intervals → est.avbps converges to R within EWMA-lag` semantic equivalence: per-EWMA matches signal-processing first-order low-pass filter.

## Hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

Estimator-specific reinforcement:

- **Per-tick atomic bstats read** — defense against torn-read causing rate-spike.
- **Per-est intvl_log/ewma_log validated** — defense against malformed UAPI causing OOB.
- **Per-RCU-free** — defense against concurrent-reader UAF.
- **Per-bucket timer single-shared** — defense against per-est timer-allocation DoS.
- **Per-est removal under lock** — defense against per-tick walking deleted entry.
- **Per-EWMA u32 saturating** — defense against integer-overflow in rate.
- **Per-tick interval read from bucket-config (not est)** — defense against per-est tampering.
- **Per-tc UAPI nl_capable check at gen_new_estimator caller** — defense against unprivileged adding estimator.
- **Per-bucket bound iter to ≤ N est per tick** — defense against per-bucket DoS.
- **Per-replace atomic swap** — defense against per-replace observed in inconsistent state.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- TC qdisc / class infra (covered separately)
- HTB / HFSC bandwidth shapers (covered separately)
- TC filters (covered separately)
- Implementation code
