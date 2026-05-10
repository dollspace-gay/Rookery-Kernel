---
title: "Tier-3: net/ipv4/tcp_cong.c — TCP congestion-control plug-in framework + Reno + CUBIC + BBR"
tags: ["tier-3", "net", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

TCP congestion-control (CC) is the algorithm-driven decision of how much data to keep in flight (CWND) — Linux supports plug-in CC via `tcp_register_congestion_control` + per-sock `tp->icsk_ca_ops` vtable. Default-policy: cubic. Per-CC: per-ACK CWND-update + per-loss CWND-collapse + per-RTO recovery + (optional) BPF/EBPF support. CUBIC: smooth W(t) = c(t-K)^3 + Wmax; BBR: bandwidth-delay-product paced; DCTCP: ECN-aware datacenter; Vegas: delay-based; Reno: classic AIMD. Per-VM tunable via setsockopt(TCP_CONGESTION) or sysctl_tcp_congestion_control.

This Tier-3 covers `tcp_cong.c` (~538) + `tcp_cubic.c` (~552) + `tcp_bbr.c` (~1200) — selective.

### Acceptance Criteria

- [ ] AC-1: setsockopt(TCP_CONGESTION, "cubic"): per-sock CC switched; subsequent ACK uses CUBIC algorithm.
- [ ] AC-2: Per-sock CC private: cubic state populated; per-ACK CWND advance per-curve.
- [ ] AC-3: BBR: per-RTT min-RTT + bandwidth estimated; pacing rate set.
- [ ] AC-4: DCTCP + ECN: per-segment ECN-marks tracked; per-RTT alpha computed; cwnd reduced proportionally.
- [ ] AC-5: Per-namespace default: net.ipv4.tcp_congestion_control = "bbr"; new conn uses BBR.
- [ ] AC-6: BPF CC: bpftool prog load CC.bpf.o; BPF struct_ops register; userspace conn uses BPF CC.
- [ ] AC-7: Per-CC undo: spurious-loss detected; cwnd restored.
- [ ] AC-8: 100K conn stress per CC: per-sock CC state correct; throughput appropriate.
- [ ] AC-9: tc-iperf3 matrix tests: cubic / bbr / dctcp throughput characterized.
- [ ] AC-10: sysctl change: net.ipv4.tcp_congestion_control = "newalgo"; takes effect for new sockets.

### Architecture

`TcpCcOps`:

```
struct TcpCcOps {
  list: ListNode,
  key: u32,
  flags: u32,
  init: TcpCcInit,
  release: TcpCcRelease,
  ssthresh: TcpCcSsthresh,
  cong_avoid: TcpCcCongAvoid,
  set_state: TcpCcSetState,
  cwnd_event: TcpCcCwndEvent,
  in_ack_event: TcpCcInAckEvent,
  pkts_acked: TcpCcPktsAcked,
  min_tso_segs: TcpCcMinTsoSegs,
  cong_control: TcpCcCongControl,
  undo_cwnd: TcpCcUndoCwnd,
  name: KStr,
  owner: KModule,
}
```

`TcpCc::register_type(type)`:
1. Validate type.name + key uniqueness.
2. spin_lock(&tcp_cong_list_lock).
3. list_add_rcu(&type.list, &tcp_cong_list).
4. unlock.
5. Return 0.

`TcpCc::assign(sk)` (per-sock at connection-init):
1. tp := tcp_sk(sk).
2. ca_ops := lookup default CC for net-namespace.
3. tp->icsk_ca_ops = ca_ops.
4. tcp_init_congestion_control(sk): ca_ops->init(sk).

`Reno::cong_avoid(sk, ack, acked)`:
1. tp := tcp_sk(sk).
2. If !tcp_is_cwnd_limited(sk): return.
3. If tp->snd_cwnd <= tp->snd_ssthresh: tcp_slow_start (cwnd += acked).
4. Else: tcp_cong_avoid_ai (cwnd += MSS / cwnd per RTT).

`Cubic::cong_avoid(sk, ack, acked)`:
1. ca := bictcp_priv(sk).
2. If tcp_in_slow_start(tp): tcp_slow_start.
3. Else: bictcp_update(ca, tp->snd_cwnd, acked).
   - Compute new cwnd via cubic curve.

`Bbr::main(sk, ack, &rs)`:
1. bbr := bbr_priv(sk).
2. bbr_update_model(sk, &rs):
   - Update min_rtt + max_bandwidth.
3. bbr_set_pacing_rate(sk).
4. bbr_set_cwnd(sk, &rs).

### Out of Scope

- TCP RX (covered in `tcp-input.md` Tier-3)
- TCP TX (covered in `tcp-output.md` Tier-3)
- BPF (covered in `kernel/bpf/bpf-core.md` Tier-3)
- Per-CC algorithm theory (network research literature)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct tcp_congestion_ops` | per-CC vtable | `net::ipv4::tcp_cong::TcpCcOps` |
| `tcp_register_congestion_control(type)` | per-CC register | `TcpCc::register_type` |
| `tcp_unregister_congestion_control(type)` | per-CC unregister | `TcpCc::unregister_type` |
| `tcp_assign_congestion_control(sk)` | per-sock pick CC | `TcpCc::assign` |
| `tcp_init_congestion_control(sk)` | per-sock init | `TcpCc::init` |
| `tcp_cleanup_congestion_control(sk)` | per-sock cleanup | `TcpCc::cleanup` |
| `tcp_set_congestion_control(sk, name, load, cap_net_admin)` | setsockopt-style change | `TcpCc::set` |
| `tcp_reno_cong_avoid(sk, ack, acked)` (Reno) | per-ACK CWND adjust | `Reno::cong_avoid` |
| `tcp_reno_ssthresh(sk)` | per-loss ssthresh | `Reno::ssthresh` |
| `tcp_reno_undo_cwnd(sk)` | per-spurious-loss undo | `Reno::undo_cwnd` |
| `bictcp_cong_avoid(sk, ack, acked)` (CUBIC; tcp_cubic.c) | per-ACK | `Cubic::cong_avoid` |
| `bictcp_state(sk, new_state)` | per-state transition | `Cubic::state` |
| `bbr_main(sk, ack, &rs)` (BBR) | per-ACK | `Bbr::main` |
| `bbr_pacing_gain` | per-cycle pacing | `Bbr::PACING_GAIN` |
| `dctcp_*` (tcp_dctcp.c) | per-ECN-mark dctcp | `Dctcp::*` |

### compatibility contract

REQ-1: `tcp_congestion_ops` (per-CC vtable):
- `init(sk)`: per-sock init.
- `release(sk)`: per-sock cleanup.
- `ssthresh(sk)` → return new ssthresh on loss.
- `cong_avoid(sk, ack, acked)`: per-ACK update CWND.
- `set_state(sk, new_state)`: per-state transition (TCP_CA_Open / _Disorder / _Recovery / _Loss).
- `cwnd_event(sk, ev)`: per-event (CWND_RESTART / CWND_TIMEOUT / etc.).
- `in_ack_event(sk, ev)`: per-ACK (CA_ACK_SLOW / FAST).
- `pkts_acked(sk, sample)`: per-batch packet sample.
- `min_tso_segs(sk)`: per-segment quantum.
- `cong_control(sk, ack, &rs)`: per-ACK rate control (BBR, etc.).
- `undo_cwnd(sk)`: per-spurious-loss recovery.
- `name[16]`: CC algorithm name.
- `flags`: TCP_CONG_NON_RESTRICTED, TCP_CONG_NEEDS_ECN.
- `key`: CC-specific per-sock key.

REQ-2: Per-sock CC private:
- tp->icsk_ca_priv (ICA_PRIV_SIZE bytes).
- Per-CC stores private state.

REQ-3: Reno CC:
- ssthresh: tp->snd_cwnd / 2.
- cong_avoid:
  - Slow-start: cwnd += acked (per-ACK +1MSS).
  - Congestion-avoidance: cwnd += MSS / cwnd (per-RTT +1MSS).

REQ-4: CUBIC CC (tcp_cubic.c):
- Per-sock `bictcp` private.
- Per-ACK W(t) = C * (t - K)^3 + Wmax where K = (Wmax * beta / C)^(1/3).
- ssthresh: cwnd * beta (default beta=0.7).

REQ-5: BBR CC (tcp_bbr.c):
- States: STARTUP / DRAIN / PROBE_BW / PROBE_RTT.
- Per-RTT: estimate min-RTT + max-bandwidth.
- Per-segment: pacing_rate = bandwidth * gain.
- cwnd ≈ bandwidth * min_rtt * 2.

REQ-6: DCTCP CC (tcp_dctcp.c):
- Per-RTT: alpha = ECN-marked-packets / total-packets.
- cwnd_reduction: cwnd *= (1 - alpha/2).
- Used in datacenter where ECN supported.

REQ-7: Per-CA state machine:
- TCP_CA_Open: normal operation.
- TCP_CA_Disorder: dup-ACK observed.
- TCP_CA_CWR: ECN echo received.
- TCP_CA_Recovery: in fast-retransmit.
- TCP_CA_Loss: RTO timeout.

REQ-8: Per-CC registration:
- tcp_register_congestion_control(&type): adds to global list.
- tp->icsk_ca_ops set per-sock at connection-init OR via setsockopt(TCP_CONGESTION, name).

REQ-9: Per-namespace default CC:
- net.ipv4.tcp_congestion_control sysctl per-namespace.
- Per-sock-bind: tp->icsk_ca_ops = lookup(default).

REQ-10: BPF CC support:
- BPF_PROG_TYPE_STRUCT_OPS allows BPF program registering as CC.
- Userspace can deploy custom CC without kernel-module.

REQ-11: Per-CC ECN cap:
- Per-CC.flags & TCP_CONG_NEEDS_ECN: only used when ECN negotiated.
- DCTCP requires ECN.

REQ-12: Per-CC pluggable hooks:
- `cong_control` (BBR-style): per-ACK rate-control replaces classic cwnd-based.
- `min_tso_segs`: per-segment min quantum (BBR uses larger).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cc_ops_rcu_protected` | UAF | per-CC list walked under rcu; module-unload respects active-sock refs. |
| `tcp_cong_list_unique_name` | INVARIANT | per-name uniqueness in registered list. |
| `cwnd_no_overflow` | INVARIANT | tp->snd_cwnd ≤ TCP_CWND_MAX. |
| `ssthresh_min_2_mss` | INVARIANT | per-RFC ssthresh ≥ 2 * MSS after loss. |
| `ca_ops_required_methods_set` | INVARIANT | per-CC.cong_avoid OR cong_control non-NULL. |

### Layer 2: TLA+

`net/ipv4/tcp_ca_state_machine.tla`:
- Per-sock CA state ∈ {Open, Disorder, CWR, Recovery, Loss}.
- Properties:
  - `safety_state_transitions_per_RFC` — only valid transitions.
  - `safety_recovery_eventually_open` — Recovery → Open on full-retx-acked.
  - `liveness_loss_eventually_handled` — Loss → Recovery via retransmit + ACK.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `TcpCc::register_type` post: type in tcp_cong_list; visible to subsequent assign | `TcpCc::register_type` |
| `TcpCc::assign` post: tp->icsk_ca_ops set; init called | `TcpCc::assign` |
| `Reno::cong_avoid` post: cwnd advances per slow-start OR AI | `Reno::cong_avoid` |
| `Cubic::cong_avoid` post: cwnd evolves per cubic curve | `Cubic::cong_avoid` |

### Layer 4: Verus/Creusot functional

`Per-sock CC: per-RTT cwnd evolution matches CC-algorithm spec (Reno AIMD / CUBIC W(t) / BBR pacing)` semantic equivalence: per-conn under given RTT/loss profile, cwnd matches algorithm output.

### hardening

(Inherits row-1 features from `net/ipv4/tcp.md` Tier-2 § Hardening.)

CC-specific reinforcement:

- **CC-list RCU + module-ref** — defense against module-unload race.
- **Per-name uniqueness** — defense against attacker-supplied bogus name.
- **Per-CC undo_cwnd opt-in** — defense against missing-undo on spurious-loss.
- **TCP_CONG_NEEDS_ECN gate** — defense against using ECN-required CC without ECN.
- **TCP_CONG_NON_RESTRICTED capability check** — defense against unauthorized CC selection.
- **BPF CC sandboxed** — defense against BPF runtime escape.
- **Per-CC private state size capped** — defense against module-supplied excessive private bloat.
- **Per-sock CC switch atomic** — defense against torn cwnd during change.
- **Per-namespace default validated** — defense against incorrect default propagating.
- **CC-state transition validated** — defense against invalid TCP_CA transitions.

