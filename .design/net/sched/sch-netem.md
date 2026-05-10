# Tier-3: net/sched/sch_netem.c — NETEM (Network Emulator) qdisc — delay/loss/dup/reorder/corruption

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/sched/sch-api.md
upstream-paths:
  - net/sched/sch_netem.c (~1426 lines)
  - include/uapi/linux/pkt_sched.h (NETEM UAPI)
-->

## Summary

NETEM (Network Emulator) introduces controlled per-pkt impairments: **delay** (mean+jitter, optional correlation, tabledist distribution: uniform/normal/pareto/paretonormal), **loss** (Gilbert-Elliot 4-state model OR uncorrelated random), **duplication** (per-pkt copy + immediate re-enqueue), **corruption** (per-pkt random bit flip), **reorder** (per-pkt with prob, send immediate; rest delayed), **rate** (token-bucket above shaping), **slot** (per-time-slot transmit-batch). Per-skb time-stamped at enqueue; per-skb dequeue gated by `time_to_send`; rb-tree ordered by send-time. Critical for: testing TCP-cong / app robustness / netcode under flaky network.

This Tier-3 covers `sch_netem.c` (~1426 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct netem_sched_data` | per-qdisc state | `NetemSchedData` |
| `struct netem_skb_cb` | per-skb time_to_send | `NetemSkbCb` |
| `netem_init()` | per-qdisc init | `Netem::init` |
| `netem_destroy()` | per-qdisc destroy | `Netem::destroy` |
| `netem_enqueue()` | per-pkt timestamp + impair + insert | `Netem::enqueue` |
| `netem_dequeue()` | per-pkt deliver-when-due | `Netem::dequeue` |
| `tabledist()` | distribution-fn delay-sample | `Netem::tabledist` |
| `loss_4state()` | Gilbert-Elliot 4-state loss model | `Netem::loss_4state` |
| `loss_gilb_ell()` | Gilbert-Elliot simplified | `Netem::loss_gilb_ell` |
| `netem_change()` | per-qdisc reconfigure | `Netem::change` |
| `clg_state` | Gilbert-Elliot state | `ClgState` |
| `TCA_NETEM_*` | per-attr UAPI | UAPI |
| `NETEM_DIST_SCALE` (8192) | per-tabledist precision | shared |

## Compatibility contract

REQ-1: Per-tc UAPI tc_netem_qopt:
- `latency`: u32 mean delay (jiffies).
- `limit`: queue cap.
- `loss`: u32 prob ([0..UINT32_MAX]).
- `gap`: per-N-th pkt impair-only.
- `duplicate`: u32 prob.
- `jitter`: u32 stddev.

REQ-2: Per-attr extensions:
- TCA_NETEM_CORR: latency_corr / loss_corr / dup_corr (correlation 0..100%).
- TCA_NETEM_REORDER: reorder_prob / reorder_corr.
- TCA_NETEM_CORRUPT: corrupt_prob / corrupt_corr.
- TCA_NETEM_DELAY_DIST: ptr to distribution table.
- TCA_NETEM_LOSS: loss model (Gilbert-Elliot OR Gilbert-Elliot-simplified).
- TCA_NETEM_RATE: rate-shape config.
- TCA_NETEM_ECN: ECN-mark instead of drop.
- TCA_NETEM_SLOT: slot-config (transmit batch).

REQ-3: tabledist(mu, sigma, state, dist):
- t = rand_u32().
- If dist: t = dist[t & (NETEM_DIST_SCALE - 1)].  // 8192-entry table
- Else: t = (t * sigma * SCALE_DIV) >> 16 (uniform).
- Return mu + (s64)t.

REQ-4: Per-skb time_to_send computed at enqueue:
- delay = tabledist(q.latency, q.jitter, &q.delay_cor, q.delay_dist).
- now = ktime_get_ns().
- If q.last_send && q.last_send > now: now = q.last_send (anti-reorder).
- skb_cb.time_to_send = now + delay.
- q.last_send = skb_cb.time_to_send.
- If reorder ∧ rand-uniform < reorder_prob: skb_cb.time_to_send = now (immediate).

REQ-5: Per-skb stored in rb-tree by time_to_send.

REQ-6: Loss models:
- Uncorrelated: per-skb prob ≤ loss; drop.
- Gilbert-Elliot 4-state (CLG_4_STATES):
  - Good (1): no-loss with prob (1 - p13); transition to Burst (3) with p13.
  - Burst (3): loss with prob (1 - p32); transition to Good (1) with p32; or Bad (4) with p23.
  - Bad (4): loss with prob (1 - p41); transition to Bad-Good (2) with p41.
  - Bad-Good (2): no-loss with prob (1 - p14); transition to Bad (4) with p14.
- Gilbert-Elliot simplified (CLG_GILB_ELL): 2-state.

REQ-7: Per-skb duplication:
- If rand-uniform < dup_prob: clone skb; immediately enqueue twice.

REQ-8: Per-skb corruption:
- If rand-uniform < corrupt_prob: pick random byte; XOR with random bit.

REQ-9: netem_dequeue:
- Pop earliest in rb-tree.
- If skb_cb.time_to_send > now: re-arm watchdog; return None.
- Else: pop; return.

REQ-10: Per-rate control:
- Optional: enforce rate via per-skb time_to_send adjusted by skb.len/rate.
- Per-slot: batch sends within slot duration.

REQ-11: Per-watchdog:
- qdisc_watchdog_schedule_ns(&q.watchdog, earliest.time_to_send).

REQ-12: Per-correlated random:
- LCG: cur = (cur * a + c) % m.
- Correlated rand: out = (1 - corr) * cur + corr * prev.

## Acceptance Criteria

- [ ] AC-1: `tc qdisc add dev eth0 root netem delay 100ms 20ms`: latency=100ms ± 20ms.
- [ ] AC-2: 1000-pkt egress: mean delay ≈ 100ms; stddev ≈ 20ms.
- [ ] AC-3: `tc qdisc add ... netem loss 10%`: ~10% per-pkt drop.
- [ ] AC-4: `tc qdisc add ... netem loss gemodel 1% 99% 5% 95%`: 4-state markov.
- [ ] AC-5: `tc qdisc add ... netem duplicate 1%`: ~1% pkts dup.
- [ ] AC-6: `tc qdisc add ... netem reorder 25% 50%`: ~25% pkts immediate; rest delayed.
- [ ] AC-7: `tc qdisc add ... netem corrupt 0.1%`: ~0.1% pkts corrupted.
- [ ] AC-8: Distribution table `paretonormal`: per-pkt delay follows tabledist.
- [ ] AC-9: ECN flag: per-marked pkt instead of drop.
- [ ] AC-10: limit reached: subsequent enqueue dropped.
- [ ] AC-11: netem_change updates correlations atomically.

## Architecture

Per-qdisc state:

```
struct NetemSchedData {
  qdisc: &Qdisc,
  watchdog: QdiscWatchdog,
  latency: i64,                                  // mean delay (ns)
  jitter: s32,                                   // stddev
  delay_cor: u32,                                // correlation
  loss: u32,                                     // prob
  loss_cor: u32,
  duplicate: u32,
  dup_cor: u32,
  reorder: u32,
  reorder_cor: u32,
  corrupt: u32,
  corrupt_cor: u32,
  ecn: bool,
  delay_dist: Option<&[i16; NETEM_DIST_MAX]>,
  loss_model: ClgLossModel,                      // None / GeMonkey / 4State
  clg: ClgState,                                  // GE state
  rate_per_byte: u64,
  slot: SlotConfig,
  last_send: Option<KtimeT>,                     // anti-reorder
  t_root: RbRoot<NetemSkbCb>,                    // skbs by time_to_send
}

struct NetemSkbCb {
  time_to_send: KtimeT,
  rbnode: RbNode,
}
```

`Netem::tabledist(mu, sigma, state, dist) -> i64`:
1. r = lcg_next(state).
2. If dist:
   - idx = r & (NETEM_DIST_SCALE - 1).
   - x = dist[idx] as i64.
3. Else:
   - x = ((r as i64) * (sigma as i64) * SCALE_DIV) >> 16.
4. Return mu + x.

`Netem::loss_4state(q) -> bool`:
1. switch q.clg.state:
   - 1: rand < q.clg.a4 ? state ← 4 : (rand < q.clg.a1 ? state ← 3 : state ← 1).
   - 2: state remains 2 unless rand < q.clg.a3 → state ← 3.
   - 3: state ← 1 if rand > q.clg.a2 else 3.
   - 4: state ← 1 if rand > q.clg.a3 else 4.
2. Return state ∈ {3, 4} ? loss : pass.

`Netem::enqueue(sch, skb, to_free) -> Result<()>`:
1. If qstats.qlen ≥ q.limit: drop; return Err.
2. count = 1.
3. If q.duplicate ∧ rand < q.duplicate: count = 2.
4. If q.loss_model:
   - If loss_check(q): drop_or_ecn(skb); return Ok (silent-drop).
5. delay = tabledist(q.latency, q.jitter, &q.delay_cor, q.delay_dist).
6. now = ktime_get_ns().
7. If q.last_send && q.last_send > now: now = q.last_send.
8. skb_cb.time_to_send = now + delay.
9. q.last_send = skb_cb.time_to_send.
10. If q.reorder ∧ rand < q.reorder: skb_cb.time_to_send = now.
11. If q.corrupt ∧ rand < q.corrupt: corrupt_skb(skb).
12. rb_insert(&q.t_root, skb_cb).
13. sch.qstats.backlog += skb.len.
14. Re-arm watchdog at earliest.

`Netem::dequeue(sch) -> Option<&Skb>`:
1. p = rb_first(&q.t_root).
2. If !p: return None.
3. skb = container_of(p).
4. now = ktime_get_ns().
5. If skb.time_to_send > now: qdisc_watchdog_schedule_ns(&q.watchdog, skb.time_to_send); return None.
6. rb_erase(&p, &q.t_root).
7. sch.qstats.backlog -= skb.len.
8. Return Some(skb).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `loss_prob_le_max` | INVARIANT | per-q.loss ≤ UINT32_MAX. |
| `time_to_send_monotonic` | INVARIANT | per-skb time_to_send ≥ q.last_send (with reorder=0). |
| `rb_tree_ordered` | INVARIANT | per-rb_node: left.time_to_send ≤ self.time_to_send ≤ right.time_to_send. |
| `clg_state_in_range` | INVARIANT | q.clg.state ∈ {1, 2, 3, 4}. |
| `corrupt_byte_within_skb` | INVARIANT | per-corrupt: byte_offset < skb.len. |

### Layer 2: TLA+

`net/sched/sch_netem.tla`:
- Per-skb enqueue + watchdog + dequeue.
- Properties:
  - `safety_no_dequeue_before_due` — per-dequeue: skb.time_to_send ≤ now.
  - `safety_loss_drops_skb` — per-loss-decision: skb dropped or ecn-marked.
  - `liveness_pending_skb_eventually_dequeued` — per-pending skb + watchdog ⟹ eventually dequeued.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Netem::tabledist` post: returned ∈ [mu - sigma*RANGE, mu + sigma*RANGE] | `Netem::tabledist` |
| `Netem::enqueue` post: skb in t_root with computed time_to_send | `Netem::enqueue` |
| `Netem::dequeue` post: returned skb has time_to_send ≤ now | `Netem::dequeue` |
| `Netem::change` post: q-config updated atomically | `Netem::change` |

### Layer 4: Verus/Creusot functional

`Per-skb queued at T → delivered at T + tabledist(latency, jitter, ...) modulo loss/reorder/dup` semantic equivalence: per-NetEm matches RFC 4814 / RFC 8126 network impairment models.

## Hardening

(Inherits row-1 features from `net/sched/sch-api.md` § Hardening.)

NETEM-specific reinforcement:

- **Per-q.limit caps queue depth** — defense against per-skb-queue OOM.
- **Per-tabledist sampled within bounds** — defense against per-distribution OOB lookup.
- **Per-LCG seeded per-qdisc** — defense against reproducible attack.
- **Per-loss-model state validated** — defense against per-state corruption.
- **Per-corrupt byte_offset bounds-checked** — defense against per-corrupt OOB write in skb.
- **Per-skb dup rate-limited** — defense against per-dup unbounded queue growth.
- **Per-time_to_send anti-reorder enforced (unless reorder)** — defense against per-skb reorder leak.
- **Per-rb_tree balanced** — defense against per-rb worst-case O(n).
- **Per-ECN-mark requires CAP_ECN** — defense against per-config breaching ECN protocol.
- **Per-tc UAPI nl_capable check at change** — defense against unprivileged setting netem.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- TC qdisc API (covered in `sch-api.md` Tier-3)
- Per-distribution (paretonormal/etc.) data tables (out-of-tree)
- Per-skb random-rate-PRNG seeding (out-of-tree)
- Implementation code
