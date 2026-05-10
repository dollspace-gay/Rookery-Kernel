---
title: "Tier-3: net/ipv4/tcp_output.c — TCP transmit-path core (per-skb queue + per-skb segmentation + retransmission + pacing)"
tags: ["tier-3", "net", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

`net/ipv4/tcp_output.c` is the TCP transmit-path — every per-sock send-syscall queues skb to write-queue + dispatches via `tcp_write_xmit` to construct + transmit per-segment TCP packets. Per-skb: TCP segmentation respecting MSS + CWND + receive-window, retransmission management (per-segment timer-armed retx-queue), TCP options (timestamp, SACK, MSS, window-scale), pacing (per-pkt rate-limit), TSO offload-prep. Critical for TCP TX correctness + performance. ~4658 lines second-largest TCP file.

This Tier-3 covers `net/ipv4/tcp_output.c` (~4658 lines).

### Acceptance Criteria

- [ ] AC-1: send(2): TCP-segment constructed; per-segment header + payload; transmitted via IP.
- [ ] AC-2: Multi-segment send: per-MSS-aligned skb's queued; per-segment SEQ number assigned.
- [ ] AC-3: TSO offload: skb with gso_size=MSS; NIC HW segments per-MSS.
- [ ] AC-4: Retransmit on RTO: per-skb retransmit; CWND reset.
- [ ] AC-5: TLP: per-sock TLP-timer fires; tail segment retransmitted.
- [ ] AC-6: Pacing: per-sock SO_PACING_RATE limits TX rate; per-skb hrtimer.
- [ ] AC-7: PMTU update: ICMP frag-needed; tp->mss_cache updated; subsequent skb's smaller.
- [ ] AC-8: TCP options: timestamp + SACK negotiated in SYN; per-segment options included.
- [ ] AC-9: Cubic / BBR / DCTCP: per-CC algorithm produces distinct CWND profile.
- [ ] AC-10: 100K conn stress: per-sock TX-loop drains write-queue without livelock.

### Architecture

`Tcp::sendmsg(sk, msg, size)`:
1. lock sock.
2. ret := tcp_sendmsg_locked(sk, msg, size).
3. release_sock(sk).
4. Return ret.

`Tcp::write_xmit(sk, mss_now, nonagle, push_one, gfp)`:
1. tp := tcp_sk(sk).
2. While true:
   - skb := tcp_send_head(sk).
   - If !skb: break.
   - cwnd_quota := tcp_cwnd_test(tp, skb).
   - If !cwnd_quota: break.
   - tcp_init_tso_segs(skb, mss_now).
   - If tcp_snd_wnd_test(tp, skb, mss_now): tso_fragment.
   - if tcp_pacing_check(sk): break.
   - tcp_transmit_skb(sk, skb, 1, gfp).
   - tcp_event_new_data_sent(sk, skb).
3. Return sent.

`Tcp::transmit_skb(sk, skb, clone_it, gfp, rcv_nxt)`:
1. tp := tcp_sk(sk).
2. clone := clone_it ? skb_clone(skb, gfp) : skb.
3. opts := compute TCP options.
4. tcp_options_write(...).
5. th := tcp_header(skb).
6. th.source = sk_port; th.dest = peer_port.
7. th.seq = htonl(skb->seq).
8. th.ack_seq = htonl(rcv_nxt).
9. th.window = htons(tp->rcv_wnd >> tp->rcv_wscale).
10. th.check = 0; tcp_v4_check.
11. err := sk_prot->sendmsg(sk, msg, size) actually → ip_queue_xmit.

`Tcp::__retransmit_skb(sk, skb, segs)`:
1. tcp_init_tso_segs(skb, mss_now).
2. tcp_event_skb_retransmit(skb).
3. tcp_transmit_skb(sk, skb, 1, gfp).
4. tp.total_retrans += segs.

### Out of Scope

- TCP RX (covered in `net/ipv4/tcp-input.md` Tier-3)
- Per-CC algorithms (covered separately)
- IP transmit (covered in `net/ipv4/ip_output.md` future Tier-3)
- Per-NIC TSO/GSO offload (covered in `net/core/dev.md` Tier-3)
- BPF socket-filter / sockmap (covered separately)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `tcp_write_xmit(sk, mss_now, nonagle, push_one, gfp)` | per-sock TX-loop entry | `Tcp::write_xmit` |
| `tcp_transmit_skb(sk, skb, clone_it, gfp_mask, rcv_nxt)` | per-skb construct + transmit | `Tcp::transmit_skb` |
| `tcp_sendmsg(sk, msg, size)` | sendmsg entry | `Tcp::sendmsg` |
| `tcp_sendmsg_locked(sk, msg, size)` | locked sendmsg | `Tcp::sendmsg_locked` |
| `tcp_push(sk, flags, mss_now, nonagle, size_goal)` | per-sock push pending data | `Tcp::push` |
| `tcp_write_queue_purge(sk)` | per-sock purge write-queue | `Tcp::write_queue_purge` |
| `__tcp_retransmit_skb(sk, skb, segs)` | per-skb retransmit | `Tcp::__retransmit_skb` |
| `tcp_xmit_retransmit_queue(sk)` | per-sock retransmit-queue dispatch | `Tcp::xmit_retransmit_queue` |
| `tcp_send_ack(sk)` | per-sock send pure-ACK | `Tcp::send_ack` |
| `tcp_send_fin(sk)` | per-sock send FIN | `Tcp::send_fin` |
| `tcp_send_synack(sk)` | per-sock SYN+ACK reply | `Tcp::send_synack` |
| `tcp_make_synack(sk, dst, &req, foc, synack_type, syn_skb)` | construct SYN+ACK skb | `Tcp::make_synack` |
| `__tcp_send_ack(sk, rcv_nxt, gfp)` | per-sock send-ACK with explicit rcv_nxt | `Tcp::__send_ack` |
| `tcp_init_tso_segs(skb, mss_now)` | per-skb TSO config | `Tcp::init_tso_segs` |
| `tso_fragment(sk, skb, len, mss_now, gfp)` | per-skb TSO split | `Tcp::tso_fragment` |
| `tcp_mtu_check_reprobe(sk)` | per-sock PMTU re-probe | `Tcp::mtu_check_reprobe` |
| `tcp_mtu_to_mss(sk, mtu)` | MTU → MSS | `Tcp::mtu_to_mss` |
| `tcp_send_loss_probe(sk)` | TLP (Tail-Loss-Probe) | `Tcp::send_loss_probe` |
| `tcp_pacing_check(sk)` | per-sock pacing-rate check | `Tcp::pacing_check` |
| `tcp_skb_pcount_set(skb, segs)` | per-skb segments-count | `Tcp::skb_pcount_set` |

### compatibility contract

REQ-1: Per-sock `sendmsg` flow:
1. Lock sock.
2. tcp_sendmsg_locked.
3. Per-msg-iov: alloc skb (or reuse partial-skb at queue-tail); copy data.
4. Update tp->write_seq.
5. Push pending data via tcp_push.

REQ-2: Per-sock `tcp_push`:
1. tcp_write_xmit(sk, mss_now, nonagle, push_one, gfp).
2. If !sent: tcp_check_probe_timer.

REQ-3: Per-sock `tcp_write_xmit` flow:
1. While sk->sk_write_queue has skb:
   - skb := skb_peek(write_queue).
   - cwnd_quota := tcp_cwnd_test (CWND vs in-flight).
   - If !cwnd_quota: break.
   - tso_segs := tcp_init_tso_segs(skb, mss_now).
   - If skb->len > mss_now * cwnd_quota: tso_fragment.
   - tcp_pacing_check: if rate-limited: defer.
   - tcp_transmit_skb(sk, skb, clone_it=1).
   - Move skb to retx-queue.
   - Update tp->snd_nxt.
2. Update tp->snd_cwnd_used.

REQ-4: Per-skb `tcp_transmit_skb` flow:
1. clone := clone_it ? skb_clone(skb) : skb.
2. tcp_options_size := tcp_options_write(...).
3. Construct TCP header: source port, dest port, seq, ack_seq, flags, window, checksum, urgent.
4. Add per-segment TCP options (timestamp / SACK / WSCALE).
5. Call sk->sk_prot->sendmsg → ip_queue_xmit (IP layer).
6. Update tp.bytes_sent + tp.segs_out.

REQ-5: TCP options in per-segment:
- MSS (only in SYN).
- Window scale (SYN/SYN+ACK only).
- SACK permitted (SYN/SYN+ACK).
- Timestamps (per-segment if negotiated).
- SACK blocks (per-non-SYN where applicable).
- TCP-AO authenticator (per-segment if configured).

REQ-6: TSO (TCP Segmentation Offload):
- Per-skb may carry skb_shinfo(skb)->gso_size = MSS.
- NIC offloads per-MSS segmentation.
- gso_segs = ceil(skb.len / mss).

REQ-7: Per-sock pacing:
- tp->sk_pacing_rate (bytes/sec).
- Per-skb: schedule_hrtimeout for next-pkt-time.
- Used by BBR + sysctl_tcp_pacing_ss_ratio for slow-start.

REQ-8: Per-sock retx-queue (write-queue split):
- skb's transmitted but unacked: in retx-queue.
- skb's queued for first-tx: in write-queue.
- After ACK: tcp_clean_rtx_queue removes from retx.

REQ-9: Per-RTO retransmit:
- RTO timer expires → tcp_retransmit_timer → __tcp_retransmit_skb.
- Per-skb at retx-queue head re-sent.
- Per-RTO event: tcp_enter_loss → CWND collapse.

REQ-10: TLP (Tail Loss Probe):
- Per-sock timer fires before RTO if pending segments.
- Probe re-sends one segment to detect tail-loss.

REQ-11: Path-MTU (PMTU):
- ICMP frag-needed → tcp_mtu_to_mss → per-sock cwnd reset.
- tcp_mtu_check_reprobe: periodic re-probe to discover MTU recovery.

REQ-12: Per-sock backlog dispatch:
- Per-sock backlog queue (sk->sk_backlog) for skb's during sk_lock-held.
- Dispatched at sk_unlock via tcp_write_xmit.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `seq_arithmetic_modular` | INVARIANT | per-segment seq comparisons via seq_lt / before / after. |
| `cwnd_quota_correct` | INVARIANT | per-write_xmit only sends within CWND. |
| `mss_per_skb_consistent` | INVARIANT | per-skb gso_size matches mss_now. |
| `retx_queue_no_uaf` | UAF | per-skb in retx-queue ref-counted. |
| `sndbuf_limit_enforced` | INVARIANT | per-sock sk_wmem_queued ≤ sk_sndbuf. |

### Layer 2: TLA+

`net/ipv4/tcp_tx.tla`:
- Per-segment state ∈ {Queued, Transmitted, Acked, Retransmitted, Lost}.
- Properties:
  - `safety_acked_unique` — per-segment Acked once.
  - `safety_lost_eventually_retx` — Lost → Retransmitted via timer / dup-ACK.
  - `liveness_queued_eventually_acked` — assuming peer acks, Queued → Acked.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Tcp::sendmsg` post: data queued; sk_wmem updated; tcp_push triggered | `Tcp::sendmsg` |
| `Tcp::write_xmit` post: per-skb in retx-queue OR remains in write-queue | `Tcp::write_xmit` |
| `Tcp::transmit_skb` post: TCP header constructed; ip_queue_xmit called | `Tcp::transmit_skb` |
| `Tcp::__retransmit_skb` post: per-skb header re-set + re-transmitted | `Tcp::__retransmit_skb` |

### Layer 4: Verus/Creusot functional

`Per-segment TX: skb constructed with TCP header + queued to IP layer; subsequent ACK matches expected SEQ` semantic equivalence: per-conn the bytes sent + bytes acked relationship matches RFC793.

### hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

tcp_output-specific reinforcement:

- **Per-sock CWND check before transmit** — defense against window-overflow.
- **Per-sock sndbuf bound** — defense against unbounded queue OOM.
- **Per-skb refcount** — defense against retx-queue-skb double-release.
- **TSO per-NIC capability check** — defense against fragmentation-fail on non-TSO NIC.
- **Per-segment TCP option valid** — defense against malformed-option causing peer-side parse fail.
- **Pacing rate clamped** — defense against pathological rate causing timer-storm.
- **PMTU lock-rate-limit** — defense against rapid-PMTU-change causing flapping.
- **Per-sock retx-count cap** — defense against unbounded retx causing peer-DoS.
- **TLP-timer bounded** — defense against premature loss-probe.
- **TFO cookie validated on send** — defense against TFO-amplification.
- **Per-segment TCP-AO HMAC** — defense against unauthenticated segment.
- **Per-sock backlog len bounded** — defense against per-sock backlog OOM.

