# Tier-3: net/ipv4/tcp_input.c — TCP receive-path core (per-segment ack-processing + slow-path SACK + congestion-control + connection-state-machine)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/ipv4/tcp.md
upstream-paths:
  - net/ipv4/tcp_input.c
  - include/net/tcp.h
  - net/ipv4/tcp_output.c (paired)
  - net/ipv4/tcp_cong.c
-->

## Summary

`net/ipv4/tcp_input.c` is the TCP receive-path heart — every per-segment ACK / DATA / FIN drives the TCP state machine. Per-segment fast-path (header-prediction): segments arriving in-order with expected ACK get fast-path delivery without full state-machine walk. Slow-path: SACK processing, retransmission detection, congestion-control updates (Reno / CUBIC / BBR / DCTCP), out-of-order queue, F-RTO false-retransmit detection. ~7767 lines — single largest TCP file. Backbone of internet TCP performance + correctness.

This Tier-3 covers `net/ipv4/tcp_input.c` (~7767 lines) — focused on per-segment dispatch + state machine.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `tcp_v4_rcv(skb)` | per-skb TCP IPv4 RX entry | `Tcp::v4_rcv` |
| `tcp_rcv_state_process(sk, skb)` | per-state RX processing | `Tcp::rcv_state_process` |
| `tcp_rcv_established(sk, skb)` | ESTABLISHED-state fast-path | `Tcp::rcv_established` |
| `tcp_data_queue(sk, skb)` | per-skb queue to receive-queue | `Tcp::data_queue` |
| `tcp_data_queue_ofo(sk, skb)` | per-skb out-of-order queue | `Tcp::data_queue_ofo` |
| `tcp_ofo_queue(sk)` | flush ofo-queue | `Tcp::ofo_queue` |
| `tcp_validate_incoming(sk, skb, &th, true)` | per-segment validation | `Tcp::validate_incoming` |
| `tcp_clean_rtx_queue(sk, skb, &delivered, &acked, &ack)` | per-ACK cleanup retx-queue | `Tcp::clean_rtx_queue` |
| `tcp_ack(sk, skb, flag)` | per-ACK processing | `Tcp::ack` |
| `tcp_rearm_rto(sk)` | per-ACK re-arm retransmit-timer | `Tcp::rearm_rto` |
| `tcp_sack_remove(tp)` | per-segment SACK pruning | `Tcp::sack_remove` |
| `tcp_sacktag_write_queue(...)` | per-SACK write-queue mark | `Tcp::sacktag_write_queue` |
| `tcp_recvmsg(sk, msg, len, flags, &addr_len)` | recv-syscall entry | `Tcp::recvmsg` |
| `tcp_recvmsg_locked(...)` | locked recv-impl | `Tcp::recvmsg_locked` |
| `tcp_event_data_recv(sk, skb)` | per-segment data-recv event | `Tcp::event_data_recv` |
| `tcp_check_space(sk)` | per-recv space check | `Tcp::check_space` |
| `tcp_set_state(sk, state)` | per-state transition | `Tcp::set_state` |
| `tcp_done(sk)` | per-sock done | `Tcp::done` |
| `tcp_rcv_established_no_data_with_no_ack(...)` | header-prediction-fail detect | `Tcp::rcv_established_no_data_no_ack` |
| `tcp_fastopen_*` | TFO-specific receive | `Tcp::fastopen_*` |

## Compatibility contract

REQ-1: TCP states (per RFC793 + extensions):
- TCP_ESTABLISHED.
- TCP_SYN_SENT.
- TCP_SYN_RECV.
- TCP_FIN_WAIT1 / _WAIT2.
- TCP_TIME_WAIT.
- TCP_CLOSE.
- TCP_CLOSE_WAIT.
- TCP_LAST_ACK.
- TCP_LISTEN.
- TCP_CLOSING.
- TCP_NEW_SYN_RECV (Linux-specific; SYN-cookies + TCP fast-open).

REQ-2: Per-segment header-prediction fast-path:
- vector of conditions: ACK matches expected; data in-order; not SACK; small range.
- Skips per-state dispatch; direct queue-to-recv.

REQ-3: Per-ACK processing (`tcp_ack`):
1. Determine: is dup-ACK / partial / full ACK?
2. tcp_clean_rtx_queue: remove acked segments from retx-queue.
3. tcp_rearm_rto: re-arm retransmit-timer.
4. tcp_fastretrans_alert: detect 3-dup-ACK fast-retransmit.
5. Update CWND via congestion-control hook.
6. tcp_xmit_retransmit_queue: maybe retransmit.

REQ-4: SACK (Selective ACK; RFC2018):
- Per-segment ACK includes SACK-block list of received non-contiguous ranges.
- Per-write-queue: per-skb sacked bitmap (TCPCB_SACKED_ACKED, _LOST, _RETRANS).
- tcp_sacktag_write_queue marks per-segment received status.

REQ-5: Out-of-order queue:
- Per-sock skb_queue: out_of_order_queue (RB-tree).
- Per-arrival OOO segment inserted; flushed when missing-segment arrives.
- Holds until in-order continuity restored.

REQ-6: Congestion-control:
- Per-sock pluggable CC: tcp_cong_ops vtable.
- Reno (default): per-ACK CWND adjust + per-loss CWND halve.
- CUBIC (modern default): smooth W(t) = c(t-K)^3 + Wmax curve.
- BBR (Google): bandwidth-delay-product-based pacing.
- DCTCP (datacenter): ECN-marking-aware.

REQ-7: Per-RTT estimator:
- Per-sock smoothed RTT (srtt) + variance (rttvar).
- Per-ACK: update srtt + rttvar via Jacobson algorithm.
- RTO derived: srtt + 4*rttvar.

REQ-8: F-RTO (Forward-RTO; RFC5682):
- Detect spurious-retransmit after RTO timeout.
- Per-RTO: send 1-or-2 unsent segments; if their ACK arrives before retx-ACK: spurious-RTO detected; restore cwnd.

REQ-9: TCP timestamps (RFC1323 PAWS):
- Per-segment TS-Val + TS-EchoReply.
- PAWS: reject segments with TS-Val < last-seen.
- Used for RTT measurement + sequence-number-wrap-protection.

REQ-10: TCP fast-open (TFO; RFC7413):
- Pre-shared cookie in SYN allows immediate SYN+data delivery.
- Per-sock tcp_fastopen_*.

REQ-11: TCP-AO (Authentication-Option; RFC5925):
- Per-segment HMAC authenticator.
- Replaces deprecated TCP-MD5.

REQ-12: Per-sock TCP_DEFER_ACCEPT:
- Server-side: don't deliver to userspace until first data segment.
- Defends against zero-data SYN-flood probes.

## Acceptance Criteria

- [ ] AC-1: Boot Linux; bind TCP listener; accept connection; per-segment data delivered.
- [ ] AC-2: 3-way handshake: SYN → SYN+ACK → ACK; states LISTEN → SYN_RECV → ESTABLISHED.
- [ ] AC-3: Out-of-order: receive segments 1, 3, 2; in-order delivery 1, 2, 3.
- [ ] AC-4: SACK: bursty packet-loss; SACK-blocks identify received-segments; per-block fast-retransmit selective.
- [ ] AC-5: 3-dup-ACK: fast-retransmit before RTO; CWND halved.
- [ ] AC-6: RTO timeout: per-segment retransmit; CWND back to MSS.
- [ ] AC-7: PAWS: receive old-TS segment; rejected.
- [ ] AC-8: TFO: client SYN includes cookie; server immediately responds with SYN+ACK+data.
- [ ] AC-9: CC switch: cubic → bbr via setsockopt(TCP_CONGESTION); rate-control behavior changes.
- [ ] AC-10: 100K connection stress: per-sock state-machine transitions correct.

## Architecture

`Tcp::v4_rcv(skb)`:
1. Pull TCP header (sizeof(struct tcphdr)).
2. Validate: csum, port, len.
3. lookup sk via __inet_lookup_skb(net, skb, ...).
4. tcp_v4_do_rcv(sk, skb).

`Tcp::do_rcv(sk, skb)`:
1. If sk->sk_state == TCP_ESTABLISHED:
   - tcp_rcv_established(sk, skb).
2. Else:
   - tcp_rcv_state_process(sk, skb).

`Tcp::rcv_established(sk, skb)`:
1. Header-prediction check.
2. If predicted-fast-path:
   - Queue to receive-queue.
   - Update tp->rcv_nxt.
   - Wake recv-waiters.
3. Else:
   - tcp_validate_incoming(sk, skb, &th, true).
   - If valid: tcp_data_queue(sk, skb).
   - tcp_ack(sk, skb, FLAG_DATA | FLAG_DATA_ACKED).

`Tcp::data_queue(sk, skb)`:
1. If skb->seq == tp->rcv_nxt:
   - In-order: append to receive-queue.
   - Update tp->rcv_nxt += skb_seq_len.
   - tcp_ofo_queue(sk): flush any OFO segments now contiguous.
2. Else:
   - tcp_data_queue_ofo(sk, skb).

`Tcp::ack(sk, skb, flag)`:
1. tp := tcp_sk(sk).
2. If ack < tp->snd_una: dup-ACK; tcp_dup_ack.
3. Else:
   - tcp_clean_rtx_queue: remove acked.
   - tcp_rearm_rto.
   - Update CWND via tp->cong_ops->ack_event(sk, flag).
   - tcp_xmit_retransmit_queue if loss recovery.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `tcp_state_valid` | INVARIANT | per-sock state ∈ valid TCP-state set; transitions per RFC793. |
| `seq_arithmetic_modular` | INVARIANT | per-segment seq comparisons use seq_lt / before / after macros. |
| `rcv_nxt_monotonic` | INVARIANT | tp->rcv_nxt monotonically increases per in-order segment. |
| `ofo_queue_bounded` | INVARIANT | per-sock out_of_order_queue len ≤ sysctl_tcp_max_ofo_len. |
| `sack_blocks_bounded` | INVARIANT | per-segment SACK-block count ≤ MAX_SACK_BLOCKS. |

### Layer 2: TLA+

`net/ipv4/tcp_state_machine.tla`:
- TCP states + transitions per RFC793.
- Properties:
  - `safety_state_transition_per_rfc` — only valid transitions allowed.
  - `safety_close_eventually_terminates` — TIME_WAIT eventually CLOSE.
  - `liveness_three_way_handshake_completes` — SYN_SENT eventually ESTABLISHED (assuming peer responds).

`net/ipv4/tcp_sack.tla`:
- Per-segment sacked bitmap.
- Properties:
  - `safety_sacked_acked_implies_recv` — SACKED-ACKED segment received by peer.
  - `safety_lost_implies_retx` — LOST segment subsequently retransmitted.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Tcp::v4_rcv` post: skb dispatched to per-sock OR drop | `Tcp::v4_rcv` |
| `Tcp::data_queue` post: skb in receive-queue OR ofo-queue | `Tcp::data_queue` |
| `Tcp::ack` post: per-acked segment cleaned; cwnd updated; retx re-armed | `Tcp::ack` |
| Per-sock tp->snd_una monotonically advances on full-ACK | invariants on ack |

### Layer 4: Verus/Creusot functional

`Per-segment TCP RX: state machine + congestion control + receive-queue mgmt match RFC793 + RFC5681 (Reno) + per-CC algorithm` semantic equivalence: per-conn the state matches RFC-defined behavior under all input sequences.

## Hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

tcp_input-specific reinforcement:

- **Per-segment seq-validation** — defense against SYN-floods + sequence-spoofing.
- **PAWS sanity check** — defense against replay-style attacks.
- **OFO-queue bounded** — defense against per-conn ofo-flood OOM.
- **SACK-block validation** — defense against malicious SACK causing kernel crash.
- **Per-sock retx-queue len bounded** — defense against unbounded retx-queue OOM.
- **Per-sock RTO + CC clamped** — defense against pathological CC causing infinite cwnd.
- **TCP fast-open cookie validated** — defense against TFO-amplification attack.
- **TCP-AO HMAC verified** — defense against unauthenticated segment.
- **RST-flood rate-limit** — defense against RST-floods exhausting per-sock state.
- **Per-sock state-machine atomic transition** — defense against torn state during concurrent recv/send.
- **TCP_DEFER_ACCEPT gating** — defense against pre-data-receive accept overhead.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- TCP Tier-2 overview (`net/ipv4/tcp.md`)
- TCP output (covered in `net/ipv4/tcp_output.md` future Tier-3)
- TCP socket lookup (covered in `net/ipv4/tcp_ipv4.md` future Tier-3)
- Per-CC algorithms (covered in `net/ipv4/tcp_cong/<algo>.md` future Tier-3s)
- TCP-AO (covered in `net/ipv4/tcp_ao.md` future Tier-3)
- Implementation code
