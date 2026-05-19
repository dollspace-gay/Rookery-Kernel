# Tier-3: net/ipv4/tcp_timer.c — TCP timers (RTO, delack, keepalive, probe)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/ipv4/tcp.md
upstream-paths:
  - net/ipv4/tcp_timer.c (~905 lines)
  - net/ipv4/tcp.c (timer init bits)
  - include/net/inet_connection_sock.h (icsk_pending, icsk_timeout)
-->

## Summary

TCP runs four classes of timers per-connection: **RTO** (retransmission), **delayed-ACK** (DACK), **probe** (zero-window probe + loss-probe), and **keepalive** (idle-conn detection). Per-tcp_sock has retransmit_timer (multiplexed for RTO / probe / loss-probe / RACK / reo-timeout via icsk_pending), delack_timer (DACK only), and keepalive_timer. Per-RTO uses smoothed RTT (srtt_us) + RTT variance (mdev_us) per Karn's algorithm. Per-keepalive sends ACK probes after tcp_keepalive_time idle. Critical for: TCP retransmission correctness, congestion-control input, half-open connection cleanup.

This Tier-3 covers `net/ipv4/tcp_timer.c` (~905 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `tcp_init_xmit_timers()` | per-init | `Tcp::init_xmit_timers` |
| `tcp_write_timer_handler()` | per-RTO/probe dispatch | `Tcp::write_timer_handler` |
| `tcp_write_timer()` | per-retransmit_timer fn | `Tcp::write_timer` |
| `tcp_delack_timer_handler()` | per-DACK dispatch | `Tcp::delack_timer_handler` |
| `tcp_delack_timer()` | per-delack fn | `Tcp::delack_timer` |
| `tcp_keepalive_timer()` | per-keepalive fn | `Tcp::keepalive_timer` |
| `tcp_compressed_ack_kick()` | per-compressed-ACK timer | `Tcp::compressed_ack_kick` |
| `tcp_retransmit_timer()` | per-RTO action | `Tcp::retransmit_timer` |
| `tcp_probe_timer()` | per-probe (zero-window) | `Tcp::probe_timer` |
| `tcp_send_loss_probe()` | per-TLP send | `Tcp::send_loss_probe` |
| `tcp_rack_reo_timeout()` | per-RACK | `Tcp::rack_reo_timeout` |
| `tcp_orphan_retries()` | per-orphan retries policy | `Tcp::orphan_retries` |
| `tcp_out_of_resources()` | per-resource-pressure | `Tcp::out_of_resources` |
| `tcp_set_keepalive()` | per-enable | `Tcp::set_keepalive` |
| `tcp_clamp_rto_to_user_timeout()` | per-clamp | `Tcp::clamp_rto_to_user_timeout` |
| `tcp_pacing_check()` | per-pacing | `Tcp::pacing_check` |

## Compatibility contract

REQ-1: tcp_init_xmit_timers(sk):
- timer_setup(&icsk.icsk_retransmit_timer, tcp_write_timer, 0).
- timer_setup(&icsk.icsk_delack_timer, tcp_delack_timer, 0).
- timer_setup(&sk.sk_timer, tcp_keepalive_timer, 0).

REQ-2: tcp_write_timer fires:
- /* Demux on icsk.icsk_pending */
- switch icsk.icsk_pending:
  - ICSK_TIME_RETRANS: tcp_retransmit_timer.
  - ICSK_TIME_PROBE0: tcp_probe_timer.
  - ICSK_TIME_LOSS_PROBE: tcp_send_loss_probe.
  - ICSK_TIME_REO_TIMEOUT: tcp_rack_reo_timeout.

REQ-3: tcp_retransmit_timer:
- if !packets_out: return.
- /* Check for orphan / out-of-resources */
- if sk_state == TCP_FIN_WAIT_1 ∨ TCP_LAST_ACK ∨ TCP_CLOSING:
  - if tcp_out_of_resources(sk, do_reset): return.
- /* Per-retransmit */
- icsk.retransmits++.
- tcp_enter_loss(sk).  // CA-state -> LOSS.
- tcp_xmit_retransmit_queue(sk, ...).  // resend lost segments.
- /* Per-RTO backoff */
- icsk.rto = min(icsk.rto * 2, TCP_RTO_MAX).
- inet_csk_reset_xmit_timer(sk, ICSK_TIME_RETRANS, icsk.rto, TCP_RTO_MAX).

REQ-4: TCP_RTO_INIT, TCP_RTO_MIN, TCP_RTO_MAX:
- TCP_RTO_INIT = 1 sec (HZ).
- TCP_RTO_MIN = 200 ms (HZ/5).
- TCP_RTO_MAX = 120 sec (120 * HZ).
- tcp_rto_min: per-net min (sysctl_tcp_rto_min_us).

REQ-5: tcp_probe_timer (zero-window probe):
- /* PROBE0: peer advertised zero-window */
- if probes_out >= max_probes: tcp_done(sk).
- else: tcp_xmit_probe_skb (1-byte probe).
- icsk.probes_out++.
- icsk.rto *= 2 (cap RTO_MAX).
- reset PROBE0 timer.

REQ-6: tcp_delack_timer (delayed-ACK):
- if !icsk.ack.pending: return.
- icsk.ack.pending &= ~ICSK_ACK_TIMER.
- /* Send pending ACK */
- tcp_send_ack(sk).
- /* Possibly schedule next */

REQ-7: ICSK_ACK_* flags:
- ICSK_ACK_SCHED: ACK is scheduled.
- ICSK_ACK_TIMER: timer running.
- ICSK_ACK_PUSHED: PSH bit was set on incoming.
- ICSK_ACK_PUSHED2: piggyback.
- ICSK_ACK_NOW: send ACK immediately.

REQ-8: tcp_keepalive_timer:
- if sk_state == TCP_LISTEN: handle SYN-RECV reqs (deprecated path).
- if sk_state ∈ {ESTABLISHED, FIN_WAIT_2}:
  - elapsed = jiffies - tp.rcv_tstamp.
  - if elapsed ≥ keepalive_time:
    - if probes_out >= keepalive_probes: tcp_send_active_reset; tcp_done.
    - else: tcp_send_active_reset_or_keepalive_probe; reset timer for keepalive_intvl.
- reset timer for keepalive_time.

REQ-9: TCP keepalive defaults:
- keepalive_time: 7200 sec (2 hours).
- keepalive_intvl: 75 sec.
- keepalive_probes: 9.

REQ-10: tcp_send_loss_probe (TLP):
- /* RFC 8985 RACK + TLP */
- Send last-segment retransmit.
- icsk.icsk_pending = 0.

REQ-11: tcp_rack_reo_timeout:
- /* Per-RACK reordering timeout */
- tcp_rack_mark_lost.

REQ-12: tcp_clamp_rto_to_user_timeout:
- Clamp icsk.rto to user_timeout (TCP_USER_TIMEOUT setsockopt).

REQ-13: tcp_orphan_retries(sk, alive):
- per-orphan_retries sysctl + alive flag.
- Returns max retries before tcp_done.

REQ-14: tcp_out_of_resources(sk, do_reset):
- if too-many-orphans: send RST + tcp_done; return 1.

REQ-15: tcp_compressed_ack_kick:
- per-compressed-ACK in TCP-LL (loss-less) mode.
- Sends compressed ACK if pending.

REQ-16: Pacing-aware:
- tcp_pacing_check: defer xmit if pacing-rate exceeded.

REQ-17: Timer sources:
- timer_list (jiffies).
- per-CONFIG_HIGH_RES_TIMERS: hrtimer for tcp_pacing_timer.

## Acceptance Criteria

- [ ] AC-1: tcp_init_xmit_timers: per-sk has 3 initialized timers.
- [ ] AC-2: send packet, no ACK: RTO fires after icsk.rto jiffies; retransmit issued.
- [ ] AC-3: RTO retransmit: rto *= 2 capped at TCP_RTO_MAX.
- [ ] AC-4: peer advertised zero-window: PROBE0 timer; sends 1-byte probe.
- [ ] AC-5: delayed-ACK: timer fires after ATO; sends ACK.
- [ ] AC-6: keepalive elapsed: send keepalive probe; if 9 probes lost: RST + done.
- [ ] AC-7: TCP_USER_TIMEOUT setsockopt: tcp_done after that elapsed.
- [ ] AC-8: TLP: send last-seg retransmit at TLP-time.
- [ ] AC-9: orphaned sk + retries exhausted: RST + tcp_done.
- [ ] AC-10: ICSK_TIME_RETRANS while DACK pending: independent timers.

## Architecture

```
struct TcpTimer;
impl TcpTimer {
  fn init_xmit_timers(sk: &Sock);
  fn write_timer(t: &TimerList);
  fn write_timer_handler(sk: &Sock);
  fn delack_timer(t: &TimerList);
  fn delack_timer_handler(sk: &Sock);
  fn keepalive_timer(t: &TimerList);
  fn retransmit_timer(sk: &Sock);
  fn probe_timer(sk: &Sock);
  fn send_loss_probe(sk: &Sock);
  fn rack_reo_timeout(sk: &Sock);
}
```

`Tcp::init_xmit_timers(sk)`:
1. icsk = inet_csk(sk).
2. timer_setup(&icsk.icsk_retransmit_timer, Tcp::write_timer, 0).
3. timer_setup(&icsk.icsk_delack_timer, Tcp::delack_timer, 0).
4. timer_setup(&sk.sk_timer, Tcp::keepalive_timer, 0).
5. icsk.icsk_pending = 0.
6. icsk.ack.pending = 0.

`Tcp::write_timer(t)`:
1. icsk = container_of(t, InetConnSock, icsk_retransmit_timer).
2. sk = &icsk.base.sock.
3. bh_lock_sock(sk).
4. if sock_owned_by_user(sk):
   - /* User holds; defer */
   - icsk.icsk_retransmit_timer.expires = jiffies + (HZ / 20).
   - sock_hold(sk).
5. else:
   - Tcp::write_timer_handler(sk).
   - if !sk_destruct_in_progress(sk): sock_put(sk).
6. bh_unlock_sock(sk).

`Tcp::write_timer_handler(sk)`:
1. icsk = inet_csk(sk).
2. event = icsk.icsk_pending.
3. if !event: return.
4. if event >= ICSK_TIME_RETRANS:
   - icsk.icsk_pending = 0.
   - tcp_clamp_rto_to_user_timeout(sk).
5. switch event:
   - ICSK_TIME_RETRANS: Tcp::retransmit_timer(sk).
   - ICSK_TIME_PROBE0: Tcp::probe_timer(sk).
   - ICSK_TIME_LOSS_PROBE: Tcp::send_loss_probe(sk).
   - ICSK_TIME_REO_TIMEOUT: Tcp::rack_reo_timeout(sk).

`Tcp::retransmit_timer(sk)`:
1. tp = tcp_sk(sk).
2. if !tp.packets_out: return.
3. /* Per-state: orphan handling */
4. if sk_state ∈ {FIN_WAIT_1, LAST_ACK, CLOSING}:
   - if tcp_out_of_resources(sk, true): return.
5. /* Enter LOSS */
6. tcp_enter_loss(sk).
7. /* Retransmit */
8. tcp_xmit_retransmit_queue(sk).
9. /* Backoff */
10. icsk.icsk_backoff++.
11. icsk.icsk_rto = min(icsk.icsk_rto * 2, TCP_RTO_MAX).
12. inet_csk_reset_xmit_timer(sk, ICSK_TIME_RETRANS, icsk.icsk_rto, TCP_RTO_MAX).

`Tcp::probe_timer(sk)`:
1. icsk = inet_csk(sk).
2. tp = tcp_sk(sk).
3. /* Max probes */
4. if icsk.icsk_probes_out > tp.tcp_max_probes(sk):
   - if !tp.tcp_close_state(sk): tcp_done(sk).
   - return.
5. tcp_xmit_probe_skb(sk, 0).
6. icsk.icsk_probes_out++.
7. icsk.icsk_rto = min(icsk.icsk_rto * 2, TCP_RTO_MAX).
8. inet_csk_reset_xmit_timer(sk, ICSK_TIME_PROBE0, icsk.icsk_rto, TCP_RTO_MAX).

`Tcp::delack_timer(t)`:
1. icsk = container_of(t, InetConnSock, icsk_delack_timer).
2. sk = &icsk.base.sock.
3. bh_lock_sock(sk).
4. if !sock_owned_by_user(sk):
   - Tcp::delack_timer_handler(sk).
5. else:
   - icsk.icsk_ack.blocked = 1.
   - sock_hold(sk).
6. bh_unlock_sock(sk).

`Tcp::delack_timer_handler(sk)`:
1. icsk = inet_csk(sk).
2. if !(icsk.icsk_ack.pending & ICSK_ACK_TIMER): return.
3. /* Send pending ACK */
4. icsk.icsk_ack.pending &= ~ICSK_ACK_TIMER.
5. tcp_send_ack(sk).
6. icsk.icsk_ack.ato = TCP_ATO_MIN.

`Tcp::keepalive_timer(t)`:
1. sk = container_of(t, Sock, sk_timer).
2. tp = tcp_sk(sk).
3. icsk = inet_csk(sk).
4. /* Idle elapsed */
5. elapsed = keepalive_time_elapsed(tp).
6. if elapsed >= keepalive_time(tp):
   - if icsk.icsk_probes_out >= keepalive_probes(tp):
     - tcp_send_active_reset(sk, GFP_ATOMIC).
     - tcp_done(sk).
     - return.
   - tcp_write_wakeup(sk, LINUX_MIB_TCPKEEPALIVE).
   - icsk.icsk_probes_out++.
   - elapsed = keepalive_intvl(tp).
7. else:
   - elapsed = keepalive_time(tp) - elapsed.
8. inet_csk_reset_keepalive_timer(sk, elapsed).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `rto_within_min_max` | INVARIANT | per-icsk.rto ∈ [TCP_RTO_MIN, TCP_RTO_MAX]. |
| `pending_exclusive_per_handler` | INVARIANT | per-write_timer_handler: clears pending before action. |
| `delack_pending_cleared_after_send` | INVARIANT | per-delack_handler: pending &= ~ICSK_ACK_TIMER. |
| `keepalive_probes_capped` | INVARIANT | per-keepalive: probes_out ≤ keepalive_probes; else done. |
| `xmit_timer_set_per_state` | INVARIANT | per-RTO/probe: timer reset post-action. |
| `bh_lock_held` | INVARIANT | per-handler: bh_lock_sock held. |

### Layer 2: TLA+

`net/ipv4/tcp-timer.tla`:
- Per-RTO + per-probe + per-DACK + per-keepalive interplay.
- Per-orphan timer + per-user_timeout.
- Properties:
  - `safety_RTO_advances_with_backoff` — per-retransmit: rto = min(rto*2, RTO_MAX).
  - `safety_keepalive_terminates_after_probes` — per-keepalive: probes ≥ N ⟹ done.
  - `safety_DACK_emits_ACK` — per-DACK: tcp_send_ack called.
  - `liveness_per_RTO_eventually_progresses_or_done` — per-RTO: ack-arrives ∨ tcp_done.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Tcp::write_timer_handler` post: pending dispatched; backoff applied | `Tcp::write_timer_handler` |
| `Tcp::retransmit_timer` post: rto-doubled, retransmit issued | `Tcp::retransmit_timer` |
| `Tcp::probe_timer` post: probe sent or done | `Tcp::probe_timer` |
| `Tcp::delack_timer_handler` post: ACK sent if pending | `Tcp::delack_timer_handler` |
| `Tcp::keepalive_timer` post: probe sent or done after probes_out limit | `Tcp::keepalive_timer` |

### Layer 4: Verus/Creusot functional

`Per-RTO/probe/DACK/keepalive timer dispatch + per-Karn-RTO + per-TLP + per-RACK-reorder` semantic equivalence: per-RFC 793 / RFC 6298 / RFC 5681 / RFC 8985.

## Hardening

(Inherits row-1 features from `net/ipv4/00-overview.md` § Hardening.)

TCP timer reinforcement:

- **Per-RTO clamped to TCP_RTO_MAX** — defense against per-runaway-backoff.
- **Per-keepalive probes capped** — defense against per-zombie-connection.
- **Per-orphan retries policy** — defense against per-orphan-resource-DOS.
- **Per-bh_lock_sock during handler** — defense against per-concurrent-mutation.
- **Per-icsk_pending cleared on dispatch** — defense against per-double-fire.
- **Per-user_timeout clamp on RTO** — defense against per-user-policy violation.
- **Per-tcp_xmit_retransmit_queue per-LOSS state** — defense against per-spurious-retransmit-during-CWR.
- **Per-tcp_pacing_check before xmit** — defense against per-pacing-violation.
- **Per-PROBE0 max_probes** — defense against per-zero-window peer-DOS.
- **Per-write_timer_handler defers if user-owned** — defense against per-handler-vs-user race.
- **Per-DACK_TIMER cleared post-send** — defense against per-redundant-ACK.
- **Per-tcp_done releases sk reference** — defense against per-leak.

## Grsecurity/PaX-style Reinforcement

Baseline hardening features applied across the TCP timer subsystem:

- **PAX_USERCOPY** — no user copies on the timer hot path; `inet_diag` export of timer state uses `nla_put` only.
- **PAX_KERNEXEC** — `tcp_write_timer`, `tcp_keepalive_timer`, `tcp_delack_timer`, and `tcp_compressed_ack_kick` handler addresses in `__ro_after_init`; `timer_setup` resolves to a `.rodata` function pointer.
- **PAX_RANDKSTACK** — kstack offset re-randomized on each softirq entry into the timer handler; defeats predictable-stack timer-spray primitives.
- **PAX_REFCOUNT** — `sock_hold` in `tcp_write_timer_handler` / `tcp_keepalive_timer` uses saturating refcount; double-fire-vs-close UAF degrades to controlled leak.
- **PAX_MEMORY_SANITIZE** — `tcp_sock`/`inet_csk` slab objects zeroed on `tcp_done` → free; freed timer state cannot leak into a freshly-allocated socket.
- **PAX_UDEREF** — `setsockopt(TCP_USER_TIMEOUT, TCP_KEEPIDLE, TCP_KEEPINTVL, TCP_KEEPCNT)` reads via `copy_from_sockptr`.
- **PAX_RAP / kCFI** — `timer_list->function` indirect call type-signature-checked; the four TCP timer handlers carry a verified prototype.
- **GRKERNSEC_HIDESYM** — `tcp_write_timer`, `tcp_keepalive_timer`, `tcp_delack_timer` symbols hidden from `/proc/kallsyms` for non-root.
- **GRKERNSEC_DMESG** — `pr_warn_ratelimited` on RTO clamp and PROBE0 cap gated from unprivileged readers.

tcp-timer-specific reinforcement:

- **RTO / probe / keepalive `sock_hold`/`sock_put` use PAX_REFCOUNT** — defense against double-fire timer vs. socket-close UAF.
- **Retransmit timeout bounded** to `[TCP_RTO_MIN, TCP_RTO_MAX]` with `icsk_user_timeout` overriding; `tcp_retries2` cap enforced and `inet_csk_rto_backoff` shift bounded under SIZE_OVERFLOW (no 64-bit shift overflow).
- **`icsk_pending` cleared **atomically** before handler dispatch** with `bh_lock_sock`; defense against double-fire during `softirq` re-entry.
- **PROBE0 (`icsk_probes_out`) capped at `sysctl_tcp_retries2`** with `tcp_write_err` on overrun; defense against zero-window peer-DoS.
- **Delack timer cancellation under `bh_lock_sock`** with `inet_csk_clear_xmit_timer`; defense against handler-vs-user-thread race producing a stale ACK.

Rationale: TCP timers fire in softirq on every connection and race the socket close path; historic timer-vs-close UAFs (CVE-2019-11479-class) are the standard primitive. REFCOUNT on the `sock_hold` pair plus RAP on `timer_list->function` plus bounded RTO arithmetic make the timer ladder a closed-loop state machine rather than a privesc primitive.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- net/ipv4/tcp_input.c (covered in `tcp-input.md` Tier-3)
- net/ipv4/tcp_output.c (covered in `tcp-output.md` Tier-3)
- net/ipv4/tcp_recovery.c RACK (covered separately if expanded)
- net/ipv4/tcp_rate.c (covered separately if expanded)
- Implementation code
