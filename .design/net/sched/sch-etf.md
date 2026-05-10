# Tier-3: net/sched/sch_etf.c — ETF (Earliest TxTime First) qdisc — TSN per-skb scheduled transmit

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/sched/sch-api.md
upstream-paths:
  - net/sched/sch_etf.c (~517 lines)
  - include/uapi/linux/pkt_sched.h (ETF UAPI + SO_TXTIME)
-->

## Summary

ETF (Earliest TxTime First; TSN class B) provides per-skb scheduled-transmit timestamps via SO_TXTIME socket-option. Per-skb userspace sets `skb.tstamp = future-deadline-ns` (CLOCK_TAI or CLOCK_REALTIME). Per-qdisc maintains red-black-tree ordered by tstamp; per-dequeue gates per-watchdog at next earliest tstamp. Optional **deadline_mode** treats tstamp as a deadline: drop if past-deadline + delta-ns. Per-qdisc may **offload** to NIC HW with LaunchTime-Engine (Intel I210, etc.) via TC_SETUP_QDISC_ETF. Critical for: TSN scheduled traffic; AVB class A/B precise transmit timing.

This Tier-3 covers `sch_etf.c` (~517 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct etf_sched_data` | per-qdisc state | `EtfSchedData` |
| `etf_init()` | per-qdisc init | `Etf::init` |
| `etf_destroy()` | per-qdisc destroy | `Etf::destroy` |
| `etf_enqueue_timesortedlist()` | per-skb tstamp-ordered insert | `Etf::enqueue` |
| `etf_dequeue_timesortedlist()` | per-skb dequeue + deadline check | `Etf::dequeue` |
| `etf_change()` | per-qdisc reconfigure | `Etf::change` |
| `etf_peek_timesortedlist()` | per-peek earliest | `Etf::peek` |
| `is_packet_valid()` | per-skb tstamp-validation | `Etf::is_packet_valid` |
| `report_sock_error()` | per-pkt drop notify | `Etf::report_sock_error` |
| `TCA_ETF_PARMS` | UAPI per-config | UAPI |
| `SO_TXTIME` | per-socket sockopt for tstamp + flags | UAPI |
| `SCM_TXTIME` | per-cmsg per-pkt | UAPI |

## Compatibility contract

REQ-1: Per-tc UAPI tc_etf_qopt:
- `delta`: u32 max-time before tstamp to send (slack window).
- `clockid`: i32 CLOCK_TAI / CLOCK_REALTIME / etc.
- `flags`: bitmap (DEADLINE_MODE / SKIP_SOCK_CHECK / OFFLOAD).

REQ-2: Per-skb sockopt SO_TXTIME:
- struct sock_txtime { clockid_t clockid, u32 flags }.
- Per-skb cmsg SCM_TXTIME = u64 ns-tstamp.
- skb.tstamp = ktime from cmsg.

REQ-3: etf_enqueue_timesortedlist:
- If !is_packet_valid(skb): drop + return Err.
- rb_insert(&q.tree, skb) ordered by skb.tstamp.
- If skb.tstamp < earliest pending: rearm watchdog at min(skb.tstamp, earliest) - delta.

REQ-4: etf_dequeue_timesortedlist:
- skb = rb_first(&q.tree); if !skb: return None.
- now = ktime_get_clockid(q.clockid).
- If skb.tstamp - delta > now: rearm watchdog at skb.tstamp - delta; return None.
- If q.deadline_mode ∧ skb.tstamp + delta < now (past-deadline):
  - drop skb; report_sock_error(skb, ECANCELED, SO_EE_CODE_TXTIME_INVALID_PARAM); return None (or recurse).
- rb_erase(skb); return Some(skb).

REQ-5: is_packet_valid(skb):
- If skb.tstamp == 0: return false.
- If !skip_sock_check ∧ skb.sk.txtime_clockid != q.clockid: return false.
- Return true.

REQ-6: deadline_mode flag:
- ETF_F_DEADLINE_MODE: tstamp is deadline; per-past-deadline drop.
- Else: tstamp is launch-time; per-past-tstamp send-immediately.

REQ-7: Per-offload (TC_SETUP_QDISC_ETF):
- Driver programs HW LaunchTime-Engine.
- Kernel becomes pass-through.

REQ-8: Per-clockid:
- CLOCK_TAI most common (PTP-synchronized).
- CLOCK_REALTIME for non-PTP setups.
- ktime_get_clockid + delta_ns precision.

REQ-9: Per-skb redirect handling:
- skb_is_redirected: skipped is_packet_valid sock check (no original sk).

REQ-10: Per-mq-prio integration:
- ETF typically leaf under mqprio (per-class scheduled traffic).

REQ-11: Per-watchdog:
- qdisc_watchdog_schedule_ns(&q.watchdog, abs_tstamp - delta).
- Per-fire: __netif_schedule the qdisc to retry dequeue.

## Acceptance Criteria

- [ ] AC-1: `tc qdisc add dev eth0 parent ... etf clockid CLOCK_TAI delta 200000`: ETF allocated.
- [ ] AC-2: skb.tstamp = future-CLOCK_TAI: enqueued; sorted in rbtree.
- [ ] AC-3: At skb.tstamp - delta: watchdog fires; dequeue returns skb.
- [ ] AC-4: skb.tstamp = past + deadline_mode: skb dropped; SO_EE_CODE_TXTIME_INVALID_PARAM reported.
- [ ] AC-5: skb.tstamp = past + launch-mode: skb sent immediately.
- [ ] AC-6: skb.tstamp = 0: enqueue returns -EINVAL.
- [ ] AC-7: skb.sk.txtime_clockid != q.clockid + !skip_sock_check: drop.
- [ ] AC-8: Multi-skb tstamps interleaved: dequeue returns earliest first.
- [ ] AC-9: Offload + supported driver: TC_SETUP_QDISC_ETF issued; soft path bypassed.
- [ ] AC-10: etf_change updates clockid/delta atomically.

## Architecture

Per-qdisc state:

```
struct EtfSchedData {
  delta: i64,                                    // slack-ns
  last: i64,
  tree: RbRoot<&Skb>,                            // ordered by skb.tstamp
  watchdog: QdiscWatchdog,
  clockid: ClockidT,
  offload: bool,
  deadline_mode: bool,
  skip_sock_check: bool,
}
```

`Etf::init(sch, opt) -> Result<()>`:
1. Parse qopt → q.delta/clockid/flags.
2. Validate clockid.
3. q.tree = rb_root_init.
4. q.deadline_mode = DEADLINE_MODE_IS_ON(qopt).
5. q.skip_sock_check = SKIP_SOCK_CHECK_IS_ON(qopt).
6. If qopt.flags & ETF_F_OFFLOAD: install offload via ndo_setup_tc(TC_SETUP_QDISC_ETF).

`Etf::is_packet_valid(skb, q) -> bool`:
1. If skb.tstamp == 0: return false.
2. If q.skip_sock_check ∨ skb_is_redirected(skb): return true.
3. sk = skb.sk.
4. If !sk: return false.
5. If sk.txtime_clockid != q.clockid: return false.
6. If q.deadline_mode ∧ !(sk.txtime_flags & SOF_TXTIME_DEADLINE_MODE): return false.
7. Return true.

`Etf::enqueue(skb, sch, to_free) -> Result<()>`:
1. If !is_packet_valid(skb, q): qdisc_drop; return Err.
2. tstamp = skb.tstamp.
3. rb_link_node ordered by tstamp.
4. sch.q.qlen++; sch.qstats.backlog += skb.len.
5. earliest = rb_first(&q.tree).
6. If earliest == skb: qdisc_watchdog_schedule_ns(&q.watchdog, tstamp - q.delta).
7. Ok.

`Etf::dequeue(sch) -> Option<&Skb>`:
1. p = rb_first(&q.tree); if !p: return None.
2. skb = container_of(p).
3. now = ktime_get_clockid(q.clockid).
4. If skb.tstamp - q.delta > now:
   - qdisc_watchdog_schedule_ns(&q.watchdog, skb.tstamp - q.delta); return None.
5. If q.deadline_mode ∧ skb.tstamp + q.delta < now:
   - rb_erase(p, &q.tree).
   - sch.q.qlen--; sch.qstats.backlog -= skb.len.
   - report_sock_error(skb, ECANCELED, SO_EE_CODE_TXTIME_INVALID_PARAM).
   - kfree_skb(skb).
   - Return Etf::dequeue(sch); // recurse for next.
6. rb_erase(p, &q.tree).
7. sch.q.qlen--; sch.qstats.backlog -= skb.len.
8. Return Some(skb).

`Etf::peek(sch) -> Option<&Skb>`:
1. p = rb_first(&q.tree); if !p: return None.
2. Return Some(container_of(p)).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `delta_pos` | INVARIANT | q.delta > 0. |
| `tree_ordered_by_tstamp` | INVARIANT | per-rb_node: left.tstamp ≤ self.tstamp ≤ right.tstamp. |
| `tstamp_nonzero_post_enqueue` | INVARIANT | per-enqueued skb tstamp > 0. |
| `dequeue_returns_earliest_due` | INVARIANT | per-dequeue-Some: skb.tstamp - delta ≤ now. |
| `deadline_drop_logged` | INVARIANT | per-deadline-mode past-deadline drop ⟹ sock_error reported. |

### Layer 2: TLA+

`net/sched/sch_etf.tla`:
- Per-skb enqueue + watchdog + dequeue + deadline-check.
- Properties:
  - `safety_no_send_before_tstamp` — per-dequeue-Some: now ≥ tstamp - delta.
  - `safety_past_deadline_dropped` — per-deadline-mode skb past tstamp + delta ⟹ dropped.
  - `liveness_eventual_dequeue` — per-skb tstamp arrives ⟹ eventually dequeued.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Etf::enqueue` post: skb in tree ordered by tstamp; watchdog rearmed if earliest | `Etf::enqueue` |
| `Etf::dequeue` post: returned skb tstamp - delta ≤ now; OR None+watchdog | `Etf::dequeue` |
| `Etf::is_packet_valid` post: returns true ⟺ skb has valid tstamp + sock check | `Etf::is_packet_valid` |
| `Etf::change` post: clockid/delta updated under lock | `Etf::change` |

### Layer 4: Verus/Creusot functional

`Per-skb scheduled at T → dequeued at T - delta..T → sent within slack window; per-deadline mode past-deadline → dropped + ECANCELED` semantic equivalence: per-ETF matches TSN class-B scheduled-traffic spec.

## Hardening

(Inherits row-1 features from `net/sched/sch-api.md` § Hardening.)

ETF-specific reinforcement:

- **Per-skb tstamp validated non-zero** — defense against per-skb fall-through with no tstamp.
- **Per-skb sock txtime_clockid match** — defense against per-clockid mismatch causing wrong-time send.
- **Per-deadline drop reported via sock-error** — defense against per-userspace silent-loss of scheduled pkt.
- **Per-tree balanced rb-tree O(log N)** — defense against per-skb O(N) insertion.
- **Per-watchdog absolute-ns** — defense against per-tick rounding.
- **Per-clockid validated** — defense against per-config invalid clockid.
- **Per-offload requires driver support** — defense against per-offload silent-fail.
- **Per-redirect skip-sock-check** — defense against per-redirected-skb missing tstamp.
- **Per-etf_change atomic-swap** — defense against per-config torn read.
- **Per-leaf integration with mqprio** — defense against per-traffic-class isolation breach.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- TC qdisc API (covered in `sch-api.md` Tier-3)
- mqprio / taprio (covered separately)
- CBS (covered in `sch-cbs.md` Tier-3)
- PTP / timekeeping (covered separately)
- Implementation code
