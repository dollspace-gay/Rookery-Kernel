---
title: "Tier-3: net/ipv4/tcp_bbr.c — BBR (Bottleneck Bandwidth and Round-trip propagation time) congestion control"
tags: ["tier-3", "net", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

BBR is a **model-based congestion control** that estimates the path's bottleneck **bandwidth (BtlBw)** and **minimum round-trip propagation time (min_rtt)** independently, and paces sending at BtlBw with a `cwnd = cwnd_gain * BtlBw * min_rtt` (BDP) cap. Per packet ACK: BBR samples delivered/elapsed via the rate-sample mechanism in tcp_input.c and feeds a windowed-max filter (`minmax`) for bw and a windowed-min filter for RTT. Per **state machine** (`bbr_mode`): BBR_STARTUP (high_gain 2885/1000 doubles rate each RTT) → BBR_DRAIN (1/high_gain drains startup queue) → BBR_PROBE_BW (8-phase pacing-gain cycle [5/4, 3/4, 1, 1, 1, 1, 1, 1] over min_rtt cycles to discover/share bw) → BBR_PROBE_RTT (every 10s if min_rtt not refreshed, cap cwnd=4 packets for 200ms to re-measure RTT). Per-LT (long-term) policer detector samples two consistent loss-heavy intervals and pins lt_bw to avoid losses. Per `tcp_congestion_ops` plug-in: registered via `tcp_register_congestion_control` with `.cong_control = bbr_main` invoking `bbr_update_model` then `bbr_set_pacing_rate` / `bbr_set_cwnd`. Critical for: high-BDP path utilization, loss-tolerant operation, deterministic pacing-based congestion control.

This Tier-3 covers `net/ipv4/tcp_bbr.c` (~1200 lines).

### Acceptance Criteria

- [ ] AC-1: `sysctl net.ipv4.tcp_congestion_control = bbr` activates plug-in; existing flows unaffected; new flows use BBR.
- [ ] AC-2: bbr_init on TCP_CA_OPEN sets mode = BBR_STARTUP, prior_cwnd = 0, snd_ssthresh = TCP_INFINITE_SSTHRESH, pacing request set.
- [ ] AC-3: BBR_STARTUP: pacing_gain = cwnd_gain ≈ 2.885; bw doubles each RTT until 3 consecutive rounds without >=25% growth.
- [ ] AC-4: STARTUP→DRAIN transition occurs when bbr_full_bw_reached; snd_ssthresh = bbr_inflight(max_bw, BBR_UNIT).
- [ ] AC-5: DRAIN→PROBE_BW transition occurs when bbr_packets_in_net_at_edt <= bbr_inflight(max_bw, BBR_UNIT).
- [ ] AC-6: PROBE_BW gain cycle: 8 phases of pacing_gain {5/4, 3/4, 1, 1, 1, 1, 1, 1}; phase advance after min_rtt + criteria.
- [ ] AC-7: Initial PROBE_BW cycle_idx uniformly random ∈ {0..6} (avoid 1 = drain phase) per bbr_cycle_rand.
- [ ] AC-8: min_rtt windowed-min over 10s; expiry triggers BBR_PROBE_RTT.
- [ ] AC-9: PROBE_RTT caps in-flight at bbr_cwnd_min_target=4 for >= 200ms + one round; then bbr_reset_mode restores prior mode.
- [ ] AC-10: bbr_max_bw uses minmax over bbr_bw_rtts=10 rounds.
- [ ] AC-11: bbr_set_pacing_rate writes sk->sk_pacing_rate; never reduces rate until full_bw_reached.
- [ ] AC-12: bbr_undo_cwnd resets full_bw / full_bw_cnt / lt sampling; returns current cwnd unchanged.
- [ ] AC-13: bbr_set_state(TCP_CA_Loss) sets full_bw = 0, round_start = 1, feeds a loss sample to LT detector.
- [ ] AC-14: LT detector pins lt_bw after 2 consecutive intervals with >= 50% loss and consistent (<=1/8 ratio or <=4 kbps diff) throughput.
- [ ] AC-15: bbr_get_info populates INET_DIAG_BBRINFO (bw, min_rtt, pacing_gain, cwnd_gain).
- [ ] AC-16: BUILD_BUG_ON sizeof(struct bbr) > ICSK_CA_PRIV_SIZE at compile time.

### Architecture

```
enum BbrMode { Startup = 0, Drain = 1, ProbeBw = 2, ProbeRtt = 3 }

struct BbrState {                          // fits in ICSK_CA_PRIV_SIZE
  min_rtt_us: u32,
  min_rtt_stamp: u32,                      // tcp_jiffies32
  probe_rtt_done_stamp: u32,
  bw: MinMax,                              // BtlBw, pkts/uS << BW_SCALE
  rtt_cnt: u32,
  next_rtt_delivered: u32,
  cycle_mstamp: u64,
  // packed bitfields:
  mode: BbrMode,                           // :3
  prev_ca_state: u8,                       // :3
  packet_conservation: bool,               // :1
  round_start: bool,                       // :1
  idle_restart: bool,                      // :1
  probe_rtt_round_done: bool,              // :1
  lt_is_sampling: bool,                    // :1
  lt_rtt_cnt: u8,                          // :7
  lt_use_bw: bool,                         // :1
  lt_bw: u32,
  lt_last_delivered: u32,
  lt_last_stamp: u32,
  lt_last_lost: u32,
  pacing_gain: u16,                        // :10
  cwnd_gain: u16,                          // :10
  full_bw_reached: bool,                   // :1
  full_bw_cnt: u8,                         // :2
  cycle_idx: u8,                           // :3
  has_seen_rtt: bool,                      // :1
  prior_cwnd: u32,
  full_bw: u32,
  ack_epoch_mstamp: u64,
  extra_acked: [u16; 2],
  ack_epoch_acked: u32,                    // :20
  extra_acked_win_rtts: u8,                // :5
  extra_acked_win_idx: u8,                 // :1
}
```

`Bbr::cong_ops()` returns a `&'static TcpCongestionOps` with the wired methods.

`Bbr::cong_control(sk, ack, flag, rs)` (per `bbr_main`):
1. Bbr::update_model(sk, rs).
2. bw = Bbr::bw(sk).
3. Bbr::set_pacing_rate(sk, bw, state.pacing_gain).
4. Bbr::set_cwnd(sk, rs, rs.acked_sacked, bw, state.cwnd_gain).

`Bbr::update_model(sk, rs)`:
1. Bbr::update_bw(sk, rs).
2. Bbr::update_ack_aggregation(sk, rs).
3. Bbr::update_cycle_phase(sk, rs).
4. Bbr::check_full_bw_reached(sk, rs).
5. Bbr::check_drain(sk, rs).
6. Bbr::update_min_rtt(sk, rs).
7. Bbr::update_gains(sk).

`Bbr::update_bw(sk, rs)`:
1. state.round_start = false.
2. if rs.delivered < 0 ∨ rs.interval_us <= 0: return.
3. if !before(rs.prior_delivered, state.next_rtt_delivered):
   - state.next_rtt_delivered = tp.delivered; state.rtt_cnt += 1; state.round_start = true; state.packet_conservation = false.
4. Bbr::lt_bw_sampling(sk, rs).
5. bw_sample = ((rs.delivered as u64) * BW_UNIT) / (rs.interval_us as u64).
6. if !rs.is_app_limited ∨ bw_sample >= Bbr::max_bw(sk):
   - minmax_running_max(&state.bw, BBR_BW_RTTS=10, state.rtt_cnt, bw_sample).

`Bbr::update_min_rtt(sk, rs)`:
1. filter_expired = after(tcp_jiffies32, state.min_rtt_stamp + 10 * HZ).
2. if rs.rtt_us >= 0 ∧ (rs.rtt_us < state.min_rtt_us ∨ (filter_expired ∧ !rs.is_ack_delayed)):
   - state.min_rtt_us = rs.rtt_us as u32; state.min_rtt_stamp = tcp_jiffies32.
3. if BBR_PROBE_RTT_MODE_MS > 0 ∧ filter_expired ∧ !state.idle_restart ∧ state.mode != BBR_PROBE_RTT:
   - state.mode = BBR_PROBE_RTT; Bbr::save_cwnd(sk); state.probe_rtt_done_stamp = 0.
4. if state.mode == BBR_PROBE_RTT:
   - tp.app_limited = ((tp.delivered + tcp_packets_in_flight(tp))).nonzero_or(1).
   - if !state.probe_rtt_done_stamp ∧ tcp_packets_in_flight(tp) <= BBR_CWND_MIN_TARGET:
     - state.probe_rtt_done_stamp = tcp_jiffies32 + msecs_to_jiffies(200).
     - state.probe_rtt_round_done = false.
     - state.next_rtt_delivered = tp.delivered.
   - else if state.probe_rtt_done_stamp:
     - if state.round_start: state.probe_rtt_round_done = true.
     - if state.probe_rtt_round_done: Bbr::check_probe_rtt_done(sk).
5. if rs.delivered > 0: state.idle_restart = false.

`Bbr::update_cycle_phase(sk, rs)`:
1. if state.mode == BBR_PROBE_BW ∧ Bbr::is_next_cycle_phase(sk, rs): Bbr::advance_cycle_phase(sk).

`Bbr::is_next_cycle_phase(sk, rs) -> bool`:
1. is_full_length = tcp_stamp_us_delta(tp.delivered_mstamp, state.cycle_mstamp) > state.min_rtt_us.
2. if state.pacing_gain == BBR_UNIT: return is_full_length.
3. inflight = Bbr::packets_in_net_at_edt(sk, rs.prior_in_flight); bw = Bbr::max_bw(sk).
4. if state.pacing_gain > BBR_UNIT:
   - return is_full_length ∧ (rs.losses ∨ inflight >= Bbr::inflight(sk, bw, state.pacing_gain)).
5. /* pacing_gain < BBR_UNIT */
   return is_full_length ∨ inflight <= Bbr::inflight(sk, bw, BBR_UNIT).

`Bbr::check_full_bw_reached(sk, rs)`:
1. if Bbr::full_bw_reached(sk) ∨ !state.round_start ∨ rs.is_app_limited: return.
2. bw_thresh = ((state.full_bw as u64) * BBR_FULL_BW_THRESH) >> BBR_SCALE.
3. if Bbr::max_bw(sk) >= bw_thresh as u32:
   - state.full_bw = Bbr::max_bw(sk); state.full_bw_cnt = 0; return.
4. state.full_bw_cnt += 1.
5. state.full_bw_reached = state.full_bw_cnt >= BBR_FULL_BW_CNT (3).

`Bbr::check_drain(sk, rs)`:
1. if state.mode == BBR_STARTUP ∧ Bbr::full_bw_reached(sk):
   - state.mode = BBR_DRAIN.
   - WRITE_ONCE(tp.snd_ssthresh, Bbr::inflight(sk, Bbr::max_bw(sk), BBR_UNIT)).
2. if state.mode == BBR_DRAIN ∧
   Bbr::packets_in_net_at_edt(sk, tcp_packets_in_flight(tp)) <= Bbr::inflight(sk, Bbr::max_bw(sk), BBR_UNIT):
   - Bbr::reset_probe_bw_mode(sk).

`Bbr::update_gains(sk)`:
1. match state.mode:
   - BBR_STARTUP: state.pacing_gain = state.cwnd_gain = BBR_HIGH_GAIN.
   - BBR_DRAIN: state.pacing_gain = BBR_DRAIN_GAIN; state.cwnd_gain = BBR_HIGH_GAIN.
   - BBR_PROBE_BW: state.pacing_gain = if state.lt_use_bw { BBR_UNIT } else { BBR_PACING_GAIN[state.cycle_idx as usize] }; state.cwnd_gain = BBR_CWND_GAIN.
   - BBR_PROBE_RTT: state.pacing_gain = state.cwnd_gain = BBR_UNIT.

`Bbr::set_pacing_rate(sk, bw, gain)`:
1. rate = Bbr::bw_to_pacing_rate(sk, bw, gain).
2. if !state.has_seen_rtt ∧ tcp_min_rtt(tp) != !0u32:
   - Bbr::init_pacing_rate_from_rtt(sk).
3. if Bbr::full_bw_reached(sk) ∨ rate > sk.sk_pacing_rate.load():
   - sk.sk_pacing_rate.store(rate).

`Bbr::set_cwnd(sk, rs, acked, bw, gain)`:
1. if Bbr::set_cwnd_to_recover_or_restore(sk, rs, acked, &mut cwnd): return cwnd.
2. target = Bbr::bdp(sk, bw, gain) + Bbr::ack_aggregation_cwnd(sk).
3. target = Bbr::quantization_budget(sk, target).
4. if Bbr::full_bw_reached(sk): tcp_snd_cwnd_set(tp, min(tcp_snd_cwnd(tp) + acked, target)).
5. else if tcp_snd_cwnd(tp) < target ∨ tp.delivered < TCP_INIT_CWND: tcp_snd_cwnd_set(tp, tcp_snd_cwnd(tp) + acked).
6. tcp_snd_cwnd_set(tp, max(tcp_snd_cwnd(tp), BBR_CWND_MIN_TARGET)).
7. if state.mode == BBR_PROBE_RTT: tcp_snd_cwnd_set(tp, min(tcp_snd_cwnd(tp), BBR_CWND_MIN_TARGET)).

`Bbr::lt_bw_sampling(sk, rs)` (per-policer):
1. if state.lt_use_bw:
   - if state.mode == BBR_PROBE_BW ∧ state.round_start ∧ (state.lt_rtt_cnt += 1) >= BBR_LT_BW_MAX_RTTS (48):
     - Bbr::reset_lt_bw_sampling(sk); Bbr::reset_probe_bw_mode(sk).
   - return.
2. if !state.lt_is_sampling: if !rs.losses: return; Bbr::reset_lt_bw_sampling_interval(sk); state.lt_is_sampling = true.
3. if rs.is_app_limited: Bbr::reset_lt_bw_sampling(sk); return.
4. if state.round_start: state.lt_rtt_cnt += 1.
5. if state.lt_rtt_cnt < BBR_LT_INTVL_MIN_RTTS (4): return.
6. if state.lt_rtt_cnt > 4 * BBR_LT_INTVL_MIN_RTTS: Bbr::reset_lt_bw_sampling(sk); return.
7. if !rs.losses: return.
8. lost = tp.lost - state.lt_last_lost; delivered = tp.delivered - state.lt_last_delivered.
9. if delivered == 0 ∨ (lost << BBR_SCALE) < BBR_LT_LOSS_THRESH * delivered: return.
10. t = ms(tp.delivered_mstamp) - state.lt_last_stamp.
11. if t < 1 ∨ overflow_check_fail(t): Bbr::reset_lt_bw_sampling(sk); return.
12. bw = delivered as u64 * USEC_PER_MSEC / t.
13. Bbr::lt_bw_interval_done(sk, bw):
    - if state.lt_bw != 0:
      - diff = abs(bw - state.lt_bw).
      - if diff * BBR_UNIT <= BBR_LT_BW_RATIO * state.lt_bw ∨ rate_bytes_per_sec(diff, BBR_UNIT) <= BBR_LT_BW_DIFF:
        - state.lt_bw = (bw + state.lt_bw) / 2; state.lt_use_bw = true; state.pacing_gain = BBR_UNIT; state.lt_rtt_cnt = 0; return.
    - state.lt_bw = bw; Bbr::reset_lt_bw_sampling_interval(sk).

`Bbr::set_state(sk, new_state)`:
1. if new_state == TCP_CA_Loss:
   - state.prev_ca_state = TCP_CA_Loss; state.full_bw = 0; state.round_start = true.
   - Bbr::lt_bw_sampling(sk, RateSample { losses: 1, ..default }).

### Out of Scope

- BBRv2 / BBRv3 (separate design document if upstreamed).
- net/ipv4/tcp_input.c rate-sample construction (covered in `tcp-input.md` Tier-3).
- net/ipv4/tcp_output.c pacing-rate consumption + EDT (covered in `tcp-output.md` Tier-3).
- net/sched/sch_fq.c fair-queue pacing scheduler (covered in `net/sched/sch_fq.md`).
- include/linux/win_minmax.h `struct minmax` (covered in `lib/win_minmax.md` if expanded).
- BPF struct_ops congestion control framework (covered in `bpf/struct-ops.md` if expanded).
- Implementation code.

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `enum bbr_mode` | STARTUP/DRAIN/PROBE_BW/PROBE_RTT | `BbrMode` |
| `struct bbr` | per-socket BBR state (ICSK_CA_PRIV_SIZE) | `BbrState` |
| `tcp_bbr_cong_ops` | per-tcp_congestion_ops registration | `Bbr::COng_OPS` |
| `bbr_init()` | per-init from `tcp_set_ca_state` | `Bbr::init` |
| `bbr_main()` | per-cong_control (per-ACK) | `Bbr::cong_control` |
| `bbr_update_model()` | per-ACK model refresh | `Bbr::update_model` |
| `bbr_update_bw()` | per-rate-sample BtlBw filter | `Bbr::update_bw` |
| `bbr_update_min_rtt()` | per-rate-sample min_rtt window | `Bbr::update_min_rtt` |
| `bbr_update_cycle_phase()` | per-PROBE_BW gain cycle advance | `Bbr::update_cycle_phase` |
| `bbr_is_next_cycle_phase()` | per-phase-completion predicate | `Bbr::is_next_cycle_phase` |
| `bbr_advance_cycle_phase()` | per-cycle_idx increment | `Bbr::advance_cycle_phase` |
| `bbr_check_full_bw_reached()` | per-STARTUP exit predicate | `Bbr::check_full_bw_reached` |
| `bbr_check_drain()` | per-STARTUP→DRAIN→PROBE_BW | `Bbr::check_drain` |
| `bbr_check_probe_rtt_done()` | per-PROBE_RTT exit | `Bbr::check_probe_rtt_done` |
| `bbr_update_gains()` | per-mode pacing/cwnd gain refresh | `Bbr::update_gains` |
| `bbr_update_ack_aggregation()` | per-extra_acked windowed-max | `Bbr::update_ack_aggregation` |
| `bbr_lt_bw_sampling()` | per-policer LT detector | `Bbr::lt_bw_sampling` |
| `bbr_lt_bw_interval_done()` | per-LT-interval close | `Bbr::lt_bw_interval_done` |
| `bbr_set_pacing_rate()` | per-pacing rate write | `Bbr::set_pacing_rate` |
| `bbr_init_pacing_rate_from_rtt()` | per-init pacing from srtt | `Bbr::init_pacing_rate_from_rtt` |
| `bbr_bw_to_pacing_rate()` | per-bw→rate conversion (gain) | `Bbr::bw_to_pacing_rate` |
| `bbr_rate_bytes_per_sec()` | per-rate * mss * (1-margin) | `Bbr::rate_bytes_per_sec` |
| `bbr_set_cwnd()` | per-cwnd derivation | `Bbr::set_cwnd` |
| `bbr_set_cwnd_to_recover_or_restore()` | per-recovery cwnd policy | `Bbr::set_cwnd_to_recover_or_restore` |
| `bbr_bdp()` | per-BDP computation | `Bbr::bdp` |
| `bbr_inflight()` | per-target inflight from bw, gain | `Bbr::inflight` |
| `bbr_quantization_budget()` | per-segmentation budget | `Bbr::quantization_budget` |
| `bbr_packets_in_net_at_edt()` | per-EDT-projected in-flight | `Bbr::packets_in_net_at_edt` |
| `bbr_ack_aggregation_cwnd()` | per-extra_acked cwnd add | `Bbr::ack_aggregation_cwnd` |
| `bbr_save_cwnd()` | per-pre-recovery snapshot | `Bbr::save_cwnd` |
| `bbr_tso_segs_goal()` / `bbr_min_tso_segs()` | per-TSO sizing | `Bbr::tso_segs_goal` |
| `bbr_max_bw()` / `bbr_bw()` / `bbr_extra_acked()` | per-getters | `Bbr::max_bw` / `bw` / `extra_acked` |
| `bbr_full_bw_reached()` | per-STARTUP-complete predicate | `Bbr::full_bw_reached` |
| `bbr_reset_startup_mode()` / `bbr_reset_probe_bw_mode()` / `bbr_reset_mode()` | per-mode-set | `Bbr::reset_*_mode` |
| `bbr_reset_lt_bw_sampling()` / `_interval()` | per-LT reset | `Bbr::reset_lt_*` |
| `bbr_cwnd_event()` | per-CA event (tx_start/idle) | `Bbr::cwnd_event` |
| `bbr_undo_cwnd()` | per-spurious-recovery undo | `Bbr::undo_cwnd` |
| `bbr_ssthresh()` | per-ssthresh + cwnd save | `Bbr::ssthresh` |
| `bbr_sndbuf_expand()` | per-3x sndbuf provision | `Bbr::sndbuf_expand` |
| `bbr_set_state()` | per-CA-state change | `Bbr::set_state` |
| `bbr_get_info()` | per-INET_DIAG_BBRINFO export | `Bbr::get_info` |
| `bbr_register()` / `bbr_unregister()` | per-module init/exit | `Bbr::register` / `unregister` |

### compatibility contract

REQ-1: `struct bbr` fits in `ICSK_CA_PRIV_SIZE` (per-`BUILD_BUG_ON` in `bbr_register`). Layout:
- min_rtt_us / min_rtt_stamp: per-windowed-min RTT (u32 jiffies stamp).
- probe_rtt_done_stamp: end-of-PROBE_RTT deadline.
- bw: `struct minmax` BtlBw filter, in pkts/uS << BW_SCALE.
- rtt_cnt: per-packet-timed round counter.
- next_rtt_delivered: tp->delivered at end of current round.
- cycle_mstamp: PROBE_BW phase start timestamp.
- mode:3 / prev_ca_state:3 / packet_conservation:1 / round_start:1 / idle_restart:1 / probe_rtt_round_done:1 / unused:13.
- lt_is_sampling:1 / lt_rtt_cnt:7 / lt_use_bw:1 / lt_bw / lt_last_delivered / lt_last_stamp / lt_last_lost.
- pacing_gain:10 / cwnd_gain:10 / full_bw_reached:1 / full_bw_cnt:2 / cycle_idx:3 / has_seen_rtt:1.
- prior_cwnd / full_bw.
- ack_epoch_mstamp / extra_acked[2] / ack_epoch_acked:20 / extra_acked_win_rtts:5 / extra_acked_win_idx:1.

REQ-2: `enum bbr_mode` values: BBR_STARTUP=0, BBR_DRAIN=1, BBR_PROBE_BW=2, BBR_PROBE_RTT=3. Stored in 3-bit `mode` field. ABI: not exposed to userspace except via tcp_bbr_info diag attr.

REQ-3: Constants (read-only, per-bbr_*):
- bbr_bw_rtts = CYCLE_LEN + 2 = 10 (BtlBw filter window in rounds).
- bbr_min_rtt_win_sec = 10 (min_rtt window in seconds).
- bbr_probe_rtt_mode_ms = 200 (PROBE_RTT duration).
- bbr_min_tso_rate = 1200000 bps (TSO threshold).
- bbr_pacing_margin_percent = 1 (1% under-pace).
- bbr_high_gain = BBR_UNIT * 2885 / 1000 + 1 (STARTUP pacing gain ≈ 2/ln(2)).
- bbr_drain_gain = BBR_UNIT * 1000 / 2885 (DRAIN gain = 1/high_gain).
- bbr_cwnd_gain = BBR_UNIT * 2 (PROBE_BW cwnd gain).
- bbr_pacing_gain[CYCLE_LEN=8] = { 5/4, 3/4, 1, 1, 1, 1, 1, 1 } * BBR_UNIT.
- bbr_cycle_rand = 7 (random initial phase 0..6 to spread BBR flows).
- bbr_cwnd_min_target = 4 (PROBE_RTT cwnd floor).
- bbr_full_bw_thresh = BBR_UNIT * 5 / 4 (25% bw growth threshold).
- bbr_full_bw_cnt = 3 (rounds without 25% growth → STARTUP done).
- bbr_lt_intvl_min_rtts = 4 / bbr_lt_loss_thresh = 50 / bbr_lt_bw_ratio = BBR_UNIT/8 / bbr_lt_bw_diff = 4000/8 / bbr_lt_bw_max_rtts = 48.
- bbr_extra_acked_gain = BBR_UNIT / bbr_extra_acked_win_rtts = 5 / bbr_ack_epoch_acked_reset_thresh = 1<<20 / bbr_extra_acked_max_us = 100ms.

REQ-4: `bbr_init(sk)` (per-CC-init):
- prior_cwnd = 0; tp->snd_ssthresh = TCP_INFINITE_SSTHRESH.
- rtt_cnt = 0; next_rtt_delivered = tp->delivered; prev_ca_state = TCP_CA_Open.
- min_rtt_us = tcp_min_rtt(tp); min_rtt_stamp = tcp_jiffies32.
- minmax_reset(&bw, rtt_cnt, 0).
- has_seen_rtt = 0; bbr_init_pacing_rate_from_rtt(sk).
- full_bw_reached = 0; full_bw = 0; full_bw_cnt = 0; cycle_mstamp = 0; cycle_idx = 0.
- bbr_reset_lt_bw_sampling; bbr_reset_startup_mode (mode = BBR_STARTUP).
- ack_epoch_mstamp = tp->tcp_mstamp; ack_epoch_acked = 0; extra_acked[0..1] = 0.
- `cmpxchg(&sk->sk_pacing_status, SK_PACING_NONE, SK_PACING_NEEDED)` — request pacing.

REQ-5: `bbr_main(sk, ack, flag, rs)` (per-cong_control hook, called once per ACK after rate_sample built):
- bbr_update_model(sk, rs).
- bw = bbr_bw(sk).
- bbr_set_pacing_rate(sk, bw, pacing_gain).
- bbr_set_cwnd(sk, rs, rs->acked_sacked, bw, cwnd_gain).

REQ-6: `bbr_update_model(sk, rs)` (orderly composition):
1. bbr_update_bw — feed BtlBw filter, count rounds, run LT sampling.
2. bbr_update_ack_aggregation — extra_acked windowed max.
3. bbr_update_cycle_phase — PROBE_BW gain cycle.
4. bbr_check_full_bw_reached — STARTUP plateau detection.
5. bbr_check_drain — STARTUP→DRAIN→PROBE_BW.
6. bbr_update_min_rtt — windowed-min + PROBE_RTT entry/exit.
7. bbr_update_gains — pacing_gain/cwnd_gain per current mode.

REQ-7: `bbr_update_bw(sk, rs)` (per-rate-sample BtlBw):
- round_start = 0; if rs->delivered < 0 || rs->interval_us <= 0: return.
- if !before(rs->prior_delivered, next_rtt_delivered): next_rtt_delivered = tp->delivered; rtt_cnt++; round_start = 1; packet_conservation = 0.
- bbr_lt_bw_sampling(sk, rs).
- bw = (rs->delivered * BW_UNIT) / rs->interval_us.
- if !rs->is_app_limited || bw >= bbr_max_bw(sk): minmax_running_max(&bw, bbr_bw_rtts, rtt_cnt, bw).

REQ-8: `bbr_update_min_rtt(sk, rs)` (per-windowed-min + PROBE_RTT):
- filter_expired = after(tcp_jiffies32, min_rtt_stamp + bbr_min_rtt_win_sec * HZ).
- if rs->rtt_us >= 0 ∧ (rs->rtt_us < min_rtt_us ∨ (filter_expired ∧ !rs->is_ack_delayed)): min_rtt_us = rs->rtt_us; min_rtt_stamp = tcp_jiffies32.
- if bbr_probe_rtt_mode_ms > 0 ∧ filter_expired ∧ !idle_restart ∧ mode != BBR_PROBE_RTT: mode = BBR_PROBE_RTT; bbr_save_cwnd; probe_rtt_done_stamp = 0.
- if mode == BBR_PROBE_RTT:
  - tp->app_limited = (tp->delivered + tcp_packets_in_flight(tp)) ?: 1.
  - if !probe_rtt_done_stamp ∧ tcp_packets_in_flight(tp) <= bbr_cwnd_min_target: probe_rtt_done_stamp = tcp_jiffies32 + msecs_to_jiffies(200); probe_rtt_round_done = 0; next_rtt_delivered = tp->delivered.
  - else if probe_rtt_done_stamp: if round_start: probe_rtt_round_done = 1; if probe_rtt_round_done: bbr_check_probe_rtt_done.
- if rs->delivered > 0: idle_restart = 0.

REQ-9: `bbr_check_probe_rtt_done(sk)`:
- if !(probe_rtt_done_stamp ∧ after(tcp_jiffies32, probe_rtt_done_stamp)): return.
- min_rtt_stamp = tcp_jiffies32; tcp_snd_cwnd_set(tp, max(tcp_snd_cwnd(tp), prior_cwnd)); bbr_reset_mode.

REQ-10: `bbr_check_full_bw_reached(sk, rs)`:
- if bbr_full_bw_reached(sk) ∨ !round_start ∨ rs->is_app_limited: return.
- bw_thresh = full_bw * bbr_full_bw_thresh >> BBR_SCALE.
- if bbr_max_bw(sk) >= bw_thresh: full_bw = bbr_max_bw; full_bw_cnt = 0; return.
- ++full_bw_cnt; full_bw_reached = full_bw_cnt >= bbr_full_bw_cnt.

REQ-11: `bbr_check_drain(sk, rs)`:
- if mode == BBR_STARTUP ∧ bbr_full_bw_reached: mode = BBR_DRAIN; WRITE_ONCE(tp->snd_ssthresh, bbr_inflight(sk, bbr_max_bw(sk), BBR_UNIT)).
- if mode == BBR_DRAIN ∧ bbr_packets_in_net_at_edt(sk, tcp_packets_in_flight(tp)) <= bbr_inflight(sk, bbr_max_bw(sk), BBR_UNIT): bbr_reset_probe_bw_mode.

REQ-12: `bbr_update_cycle_phase(sk, rs)`:
- if mode == BBR_PROBE_BW ∧ bbr_is_next_cycle_phase(sk, rs): bbr_advance_cycle_phase.
- bbr_advance_cycle_phase: cycle_idx = (cycle_idx + 1) & (CYCLE_LEN - 1); cycle_mstamp = tp->delivered_mstamp.
- bbr_is_next_cycle_phase predicate:
  - is_full_length = (delivered_mstamp - cycle_mstamp) > min_rtt_us.
  - pacing_gain == BBR_UNIT (1.0 cruise): return is_full_length.
  - pacing_gain > BBR_UNIT (probe-up 5/4): return is_full_length ∧ (rs->losses ∨ inflight >= bbr_inflight(sk, max_bw, pacing_gain)).
  - pacing_gain < BBR_UNIT (drain 3/4): return is_full_length ∨ inflight <= bbr_inflight(sk, max_bw, BBR_UNIT).

REQ-13: `bbr_update_gains(sk)`:
- BBR_STARTUP: pacing_gain = cwnd_gain = bbr_high_gain.
- BBR_DRAIN: pacing_gain = bbr_drain_gain; cwnd_gain = bbr_high_gain.
- BBR_PROBE_BW: pacing_gain = (lt_use_bw ? BBR_UNIT : bbr_pacing_gain[cycle_idx]); cwnd_gain = bbr_cwnd_gain.
- BBR_PROBE_RTT: pacing_gain = cwnd_gain = BBR_UNIT.
- default: WARN_ONCE.

REQ-14: `bbr_lt_bw_sampling(sk, rs)` (per-policer detector):
- if lt_use_bw: if mode == BBR_PROBE_BW ∧ round_start ∧ ++lt_rtt_cnt >= bbr_lt_bw_max_rtts (48): bbr_reset_lt_bw_sampling; bbr_reset_probe_bw_mode. return.
- if !lt_is_sampling: if !rs->losses: return; bbr_reset_lt_bw_sampling_interval; lt_is_sampling = true.
- if rs->is_app_limited: bbr_reset_lt_bw_sampling; return.
- if round_start: lt_rtt_cnt++.
- if lt_rtt_cnt < 4 (bbr_lt_intvl_min_rtts): return.
- if lt_rtt_cnt > 16: bbr_reset_lt_bw_sampling; return.
- if !rs->losses: return.
- lost = tp->lost - lt_last_lost; delivered = tp->delivered - lt_last_delivered.
- if !delivered ∨ (lost << BBR_SCALE) < bbr_lt_loss_thresh * delivered (50%): return.
- t = ms(delivered_mstamp) - lt_last_stamp; if t < 1: return; if overflow: reset.
- bw = (u64)delivered * USEC_PER_MSEC / t.
- bbr_lt_bw_interval_done(sk, bw): if old lt_bw within 1/8 ratio or 4kbps: lt_bw = (old+new)/2; lt_use_bw = 1; pacing_gain = BBR_UNIT. else lt_bw = bw; reset_interval.

REQ-15: `bbr_set_pacing_rate(sk, bw, gain)`:
- rate = bbr_bw_to_pacing_rate(sk, bw, gain) — = bw * mss * gain * (USEC_PER_SEC/100) * (100-margin) >> (BBR_SCALE + BW_SCALE).
- if !has_seen_rtt ∧ tcp_min_rtt(tp) != ~0: bbr_init_pacing_rate_from_rtt(sk).
- if bbr_full_bw_reached(sk) ∨ rate > READ_ONCE(sk->sk_pacing_rate): WRITE_ONCE(sk->sk_pacing_rate, rate).

REQ-16: `bbr_set_cwnd(sk, rs, acked, bw, gain)`:
- bbr_set_cwnd_to_recover_or_restore — handle recovery/restore-from-loss; if true: return.
- target_cwnd = bbr_bdp(sk, bw, gain) + bbr_ack_aggregation_cwnd(sk).
- target_cwnd = bbr_quantization_budget(sk, target_cwnd).
- if bbr_full_bw_reached: tcp_snd_cwnd_set(tp, min(tcp_snd_cwnd(tp) + acked, target_cwnd)).
- else if tcp_snd_cwnd(tp) < target_cwnd ∨ tp->delivered < TCP_INIT_CWND: tcp_snd_cwnd_set(tp, tcp_snd_cwnd(tp) + acked).
- tcp_snd_cwnd_set(tp, max(tcp_snd_cwnd(tp), bbr_cwnd_min_target)).
- if mode == BBR_PROBE_RTT: tcp_snd_cwnd_set(tp, min(tcp_snd_cwnd(tp), bbr_cwnd_min_target)).

REQ-17: `bbr_set_state(sk, new_state)`:
- if new_state == TCP_CA_Loss: prev_ca_state = TCP_CA_Loss; full_bw = 0; round_start = 1; bbr_lt_bw_sampling(sk, rs={ .losses = 1 }).

REQ-18: `tcp_bbr_cong_ops` registration:
- .flags = TCP_CONG_NON_RESTRICTED; .name = "bbr"; .owner = THIS_MODULE.
- .init = bbr_init; .cong_control = bbr_main; .sndbuf_expand = bbr_sndbuf_expand.
- .undo_cwnd = bbr_undo_cwnd; .cwnd_event_tx_start = bbr_cwnd_event_tx_start.
- .ssthresh = bbr_ssthresh; .min_tso_segs = bbr_min_tso_segs; .get_info = bbr_get_info; .set_state = bbr_set_state.
- Module init: register BTF kfunc set + tcp_register_congestion_control.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `bbr_state_fits_icsk_ca` | INVARIANT | per-build: sizeof(BbrState) <= ICSK_CA_PRIV_SIZE. |
| `cycle_idx_in_range` | INVARIANT | per-advance_cycle_phase: cycle_idx < CYCLE_LEN (8) — masked with `& (CYCLE_LEN - 1)`. |
| `mode_valid` | INVARIANT | per-update_gains: state.mode is one of {Startup, Drain, ProbeBw, ProbeRtt}. |
| `pacing_gain_table_indexed_safely` | INVARIANT | per-update_gains: cycle_idx as usize indexes BBR_PACING_GAIN[0..7]. |
| `full_bw_cnt_bounded` | INVARIANT | per-check_full_bw_reached: full_bw_cnt <= 3 (2-bit field, saturating). |
| `bw_sample_no_overflow` | INVARIANT | per-update_bw: (delivered * BW_UNIT) fits u64 since delivered <= 2^32, BW_UNIT = 2^24. |
| `rate_bytes_per_sec_no_overflow` | INVARIANT | per-bbr_rate_bytes_per_sec: ordering rate*mss*gain>>BBR_SCALE prevents u64 overflow up to 2.9Tbps. |
| `pacing_rate_monotonic_until_full_bw` | INVARIANT | per-set_pacing_rate: !full_bw_reached ⟹ sk_pacing_rate never decreases. |
| `lt_rtt_cnt_bounded` | INVARIANT | per-lt_bw_sampling: lt_rtt_cnt < 128 (7-bit field). |
| `extra_acked_win_idx_binary` | INVARIANT | per-update_ack_aggregation: extra_acked_win_idx ∈ {0, 1}. |
| `min_rtt_never_zero_after_first_sample` | INVARIANT | per-update_min_rtt: once rs.rtt_us >= 0 observed, min_rtt_us > 0. |
| `probe_rtt_cwnd_cap` | INVARIANT | per-set_cwnd: mode == PROBE_RTT ⟹ tcp_snd_cwnd(tp) <= BBR_CWND_MIN_TARGET (4) post-call. |

### Layer 2: TLA+

`net/ipv4/tcp-bbr.tla`:
- Per-ACK + per-rate-sample + per-mode-transition + per-cycle-advance.
- Properties:
  - `safety_startup_to_drain` — Startup ∧ full_bw_reached ⟹ next mode is Drain (after this ACK).
  - `safety_drain_to_probe_bw` — Drain ∧ inflight <= bdp(max_bw, 1.0) ⟹ next mode is ProbeBw.
  - `safety_probe_rtt_entry` — min_rtt filter expired ∧ mode != ProbeRtt ∧ !idle_restart ⟹ mode := ProbeRtt.
  - `safety_probe_rtt_duration` — ProbeRtt persists >= 200ms ∧ >= 1 round before exit via check_probe_rtt_done.
  - `safety_pacing_only_decreases_post_full_bw` — !full_bw_reached ⟹ pacing_rate non-decreasing.
  - `safety_lt_bw_pins_pacing_gain_unit` — lt_use_bw == true ⟹ ProbeBw pacing_gain == BBR_UNIT.
  - `safety_cycle_phase_indices_valid` — cycle_idx ∈ {0..7}; pacing_gain[cycle_idx] ∈ {5/4, 3/4, 1, 1, 1, 1, 1, 1} * BBR_UNIT.
  - `liveness_eventually_leaves_startup` — given a sufficiently long flow with bounded bw, eventually full_bw_reached.
  - `liveness_eventually_advances_probe_bw_cycle` — Per-min_rtt period the cycle advances at least once.
  - `liveness_eventually_probes_rtt` — min_rtt_stamp + 10s deadline eventually triggers ProbeRtt entry.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Bbr::cong_control` post: pacing_rate updated; cwnd updated per current mode | `Bbr::cong_control` |
| `Bbr::update_bw` post: round_start ⟹ rtt_cnt += 1 ∧ next_rtt_delivered = tp.delivered | `Bbr::update_bw` |
| `Bbr::update_min_rtt` post: rs.rtt_us < min_rtt_us ⟹ min_rtt_us := rs.rtt_us | `Bbr::update_min_rtt` |
| `Bbr::check_full_bw_reached` post: bw_grew_25pct ⟺ full_bw_cnt = 0; else full_bw_cnt += 1 | `Bbr::check_full_bw_reached` |
| `Bbr::check_drain` post: Startup ∧ full_bw_reached ⟹ mode = Drain ∧ snd_ssthresh = bdp | `Bbr::check_drain` |
| `Bbr::update_cycle_phase` post: PROBE_BW phase advance ⟹ cycle_idx = (old + 1) mod 8 | `Bbr::update_cycle_phase` |
| `Bbr::update_gains` post: mode == Startup ⟹ pacing_gain = cwnd_gain = high_gain | `Bbr::update_gains` |
| `Bbr::lt_bw_sampling` post: two consecutive consistent intervals ⟹ lt_use_bw = true | `Bbr::lt_bw_sampling` |
| `Bbr::set_cwnd` post: probe_rtt ⟹ cwnd = min(cwnd, BBR_CWND_MIN_TARGET) | `Bbr::set_cwnd` |
| `Bbr::set_state` post: TCP_CA_Loss ⟹ full_bw = 0 ∧ round_start = 1 | `Bbr::set_state` |

### Layer 4: Verus/Creusot functional

`Per-ACK → rate_sample → update_bw (BtlBw filter) → update_ack_aggregation → update_cycle_phase → check_full_bw_reached → check_drain → update_min_rtt (with PROBE_RTT entry/exit) → update_gains → set_pacing_rate → set_cwnd (with BDP target + extra_acked + quantization + min cap + PROBE_RTT cap)` semantic equivalence with the BBR draft (draft-cardwell-iccrg-bbr-congestion-control) and per-Documentation/networking/tcp.rst BBR section.

### hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

BBR reinforcement:

- **Per-`BUILD_BUG_ON` BbrState size <= ICSK_CA_PRIV_SIZE** — defense against per-CA-priv corruption.
- **Per-bitfield range invariants (mode:3, cycle_idx:3, full_bw_cnt:2, lt_rtt_cnt:7)** — defense against per-bit-overflow violating mode/gain invariants.
- **Per-cycle_idx masked with (CYCLE_LEN-1)** — defense against per-pacing-gain[] out-of-bounds.
- **Per-pacing_rate-monotone-during-STARTUP** — defense against per-startup-stall (premature rate cut).
- **Per-PROBE_RTT bounded duration (200ms + 1 round)** — defense against per-PROBE_RTT-livelock.
- **Per-min_rtt window 10s** — defense against per-stale RTT estimates causing pacing miscalibration.
- **Per-LT detector tied to actual losses + non-app-limited samples** — defense against per-false-policer detection.
- **Per-rs.is_app_limited filtered from BtlBw sampling** — defense against per-app-limited-bw-underestimate.
- **Per-rs.is_ack_delayed filtered from min_rtt sampling** — defense against per-delayed-ACK-RTT-inflation.
- **Per-bbr_rate_bytes_per_sec u64 overflow-safe operand order (verified by comment)** — defense against per-rate-wrap on 100+Gbps paths.
- **Per-`cmpxchg(SK_PACING_NONE, SK_PACING_NEEDED)` non-clobbering** — defense against per-pacing-conflict with FQ disc.
- **Per-tcp_register_congestion_control failure path unwinds BTF kfunc registration** — defense against per-module-load partial-state.
- **Per-MODULE_LICENSE Dual BSD/GPL preserved (Rookery clean-room rewrite uses same license)** — defense against per-license drift.

