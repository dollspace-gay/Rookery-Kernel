---
title: "Tier-3: net/ipv4/tcp — TCP (RFC 9293)"
tags: ["design-doc", "tier-3", "net"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for TCP (Transmission Control Protocol) on IPv4. Owns the TCP state machine (per RFC 9293), connection establishment + teardown, congestion control framework + per-pluggable-algorithm hooks (CUBIC default, BBR, Reno, DCTCP, etc.), retransmission timer + RTO calculation, RACK + TLP loss recovery, SACK + DSACK, fast-path receive, segmentation + reassembly, window scale, timestamps, MD5 + AO authentication, and TCP Fast Open.

Sub-tier-3 of `net/ipv4/00-overview.md`. **Strong Layer-4 Verus opt-in candidate** (declared in `net/00-overview.md`): TCP state machine functional-correctness vs. RFC 9293 is one of the most-impactful proofs in the entire project.

### Requirements

- REQ-1: TCP wire format byte-identical to upstream (RFC 9293 + Linux extensions).
- REQ-2: TCP state machine implements RFC 9293's CLOSED → LISTEN → SYN_SENT → SYN_RECV → ESTABLISHED → FIN_WAIT_1/2 → CLOSE_WAIT → CLOSING → LAST_ACK → TIME_WAIT → CLOSED transitions byte-identically. Listen-side req-sock + accept-queue semantics match upstream.
- REQ-3: `struct tcp_sock` layout-equivalent for first-cache-line + commonly-accessed fields.
- REQ-4: TCP option parsing + composition: MSS, Window Scale, Timestamps, SACK, MD5, AO. Per RFC + RFC 5925 (AO) + Linux's MD5 (RFC 2385).
- REQ-5: TCP_* sockopts: every documented opt byte-identical numeric value + struct layout (TCP_INFO etc.).
- REQ-6: Congestion control framework: pluggable algorithms via `tcp_register_congestion_control`; CUBIC default; BBR, Reno, DCTCP, Cubic, BIC, CDG, Highspeed, HTCP, Hybla, Illinois, LP, NV, Scalable, Vegas, Veno, Westwood, YeaH, PLB all available.
- REQ-7: RTT estimation + RTO calculation per RFC 6298; SRTT smoothing constants identical.
- REQ-8: Loss recovery: RACK + TLP per RFC 8985; F-RTO per RFC 5682; SACK + DSACK per RFC 2018 + 2883.
- REQ-9: TCP Fast Open per RFC 7413: cookie issuance + verification, TFO server + client side.
- REQ-10: TCP Authentication Option (TCP-AO) per RFC 5925 + Linux's recent implementation.
- REQ-11: TCP-MD5 (legacy; RFC 2385) preserved for BGP compat.
- REQ-12: GSO + GRO for TCP: tcp_gso_segment + tcp_gro_receive identical.
- REQ-13: BPF integration (sockmap, sk_msg, BPF socket-iter) preserved.
- REQ-14: TCP per-route metrics caching: rtt + cwnd + ssthresh persisted across short reconnect; `tcp_no_metrics_save` toggle preserved.
- REQ-15: TLA+ model `models/net/tcp_state.tla` (mandatory per `net/00-overview.md` Layer 2) — proves TCP RFC 9293 state machine implementation matches the spec under concurrent recv-from-network + app-write.
- REQ-16: **Layer-4 functional correctness MANDATORY-recommended via Verus** (declared in `net/00-overview.md` Layer 4 candidates) — proves TCP state-machine implementation conforms to RFC 9293. High-leverage; TCP is a frequent CVE source.
- REQ-17: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: A `tcpdump` capture of a TCP connection setup + data transfer + teardown byte-identical (modulo timestamps + SeqNo deltas) on Rookery vs. upstream. (covers REQ-1, REQ-2, REQ-4)
- [ ] AC-2: `pahole struct tcp_sock` first-cache-line layout byte-identical. (covers REQ-3)
- [ ] AC-3: An iperf3 TCP test on every available cong-control algorithm (`net.ipv4.tcp_congestion_control = X`) achieves throughput within ±5% of upstream. (covers REQ-6)
- [ ] AC-4: A high-RTT loss-injection test (drop 1% of packets, RTT 100ms) shows TCP recovery semantics matching upstream within ±10% throughput. (covers REQ-7, REQ-8)
- [ ] AC-5: A TCP Fast Open client + server test exchanges TFO cookies + bypasses 3WHS on subsequent connections. (covers REQ-9)
- [ ] AC-6: A BGP-style TCP-MD5 session works correctly between Rookery + a BGP daemon. (covers REQ-11)
- [ ] AC-7: A TCP-AO test (RFC 5925) exchanges authenticated segments correctly. (covers REQ-10)
- [ ] AC-8: GSO + GRO selftests pass. (covers REQ-12)
- [ ] AC-9: A sockmap test (BPF redirect) routes TCP traffic between two sockets via sockmap. (covers REQ-13)
- [ ] AC-10: A reconnect test against the same destination after `net.ipv4.tcp_no_metrics_save=0` reuses cached cwnd. (covers REQ-14)
- [ ] AC-11: `make tla` passes `models/net/tcp_state.tla`. (covers REQ-15)
- [ ] AC-12: **Layer-4 Verus proof of TCP state-machine implementation conforms to RFC 9293** compiles and verifies. (covers REQ-16)
- [ ] AC-13: Every documented `net.ipv4.tcp_*` sysctl round-trips correctly. (covers REQ-5)
- [ ] AC-14: Hardening section present and follows template. (covers REQ-17)

### Architecture

### Rust module organization

- `kernel::net::ipv4::tcp::TcpSock` — `struct tcp_sock` wrapper
- `kernel::net::ipv4::tcp::state_machine::TcpStateMachine` — RFC 9293 state machine
- `kernel::net::ipv4::tcp::input` — RX path (tcp_input.c analog)
- `kernel::net::ipv4::tcp::output` — TX path (tcp_output.c analog)
- `kernel::net::ipv4::tcp::timer` — retrans + persist + keepalive timers
- `kernel::net::ipv4::tcp::cong::CongControl` — pluggable cong-control framework
- `kernel::net::ipv4::tcp::cong::cubic` (and others) — per-algorithm modules
- `kernel::net::ipv4::tcp::recovery` — RACK + TLP + F-RTO loss recovery
- `kernel::net::ipv4::tcp::sack` — SACK + DSACK processing
- `kernel::net::ipv4::tcp::offload` — GSO + GRO
- `kernel::net::ipv4::tcp::fastopen` — TFO
- `kernel::net::ipv4::tcp::ao` — TCP Authentication Option
- `kernel::net::ipv4::tcp::md5` — TCP-MD5 (legacy)
- `kernel::net::ipv4::tcp::diag` — inet_diag dump
- `kernel::net::ipv4::tcp::metrics` — per-route metrics
- `kernel::net::ipv4::tcp::ulp::Ulp` — Upper Layer Protocol (kTLS hooks)
- `kernel::net::ipv4::tcp::bpf` — BPF integration

### Locking and concurrency

- **Per-`tcp_sock` `sk_lock`** (composite spin + waitqueue): held during sendmsg / recvmsg / setsockopt / state transitions
- **Per-listener accept queue** (spinlock): protects `request_sock` queue
- **Per-route metrics hash** (RCU): readers RCU; writer per-bucket lock
- **Cong-control list** (spinlock): per-protocol algorithm-list mutator

TLA+ model `models/net/tcp_state.tla` covers the full state machine under concurrent ack-arrival + app-write + timer-fire.

### Error handling

- `Err(ECONNREFUSED)` — destination port closed
- `Err(ECONNRESET)` — RST received
- `Err(ETIMEDOUT)` — connection timed out
- `Err(EAGAIN)` — would-block on non-blocking sock
- `Err(EPIPE)` — write on closed sock
- `Err(EMSGSIZE)` — write exceeds sock sndbuf

### Out of Scope

- TCP over IPv6 (cross-ref `net/ipv6/tcp6.md` Tier-3 to be authored; mostly shares this Tier-3's logic)
- MPTCP (cross-ref `net/mptcp.md`)
- 32-bit-only paths
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| TCP core (sendmsg, recvmsg, setsockopt, getsockopt) | `net/ipv4/tcp.c` |
| RX path (state machine + ack processing) | `net/ipv4/tcp_input.c` |
| TX path (skb composition + segmentation) | `net/ipv4/tcp_output.c` |
| IPv4-specific TCP (icsk + connect/listen/accept) | `net/ipv4/tcp_ipv4.c`, `net/ipv4/inet_connection_sock.c` |
| Retransmission + persist + keepalive timers | `net/ipv4/tcp_timer.c` |
| Loss recovery (RACK, TLP, F-RTO) | `net/ipv4/tcp_recovery.c` |
| Congestion control framework | `net/ipv4/tcp_cong.c` |
| Pluggable cong algorithms | `tcp_cubic.c` (default), `tcp_bbr.c`, `tcp_dctcp.c`, `tcp_bic.c`, `tcp_cdg.c`, `tcp_highspeed.c`, `tcp_htcp.c`, `tcp_hybla.c`, `tcp_illinois.c`, `tcp_lp.c`, `tcp_nv.c`, `tcp_scalable.c`, `tcp_vegas.c`, `tcp_veno.c`, `tcp_westwood.c`, `tcp_yeah.c`, `tcp_plb.c` |
| Mini-socks (TIME_WAIT + req-sock) | `net/ipv4/tcp_minisocks.c` |
| Per-route metrics caching | `net/ipv4/tcp_metrics.c` |
| `inet_diag` netlink dump | `net/ipv4/tcp_diag.c` |
| GSO/GRO offload | `net/ipv4/tcp_offload.c` |
| TCP Fast Open | `net/ipv4/tcp_fastopen.c` |
| Upper Layer Protocol shim | `net/ipv4/tcp_ulp.c` |
| TCP Authentication Option (RFC 5925) | `net/ipv4/tcp_ao.c` |
| TCP-MD5 + AO sigpool | `net/ipv4/tcp_sigpool.c` |
| BPF integration (sockmap, sk_msg) | `net/ipv4/tcp_bpf.c` |
| AF_INET registration | `net/ipv4/af_inet.c` |
| Public types | `include/net/tcp.h` |
| UAPI | `include/uapi/linux/tcp.h` |

### compatibility contract

### Wire format

TCP wire format is fixed by RFC 9293 (consolidating 793 + 7323 + others); Linux's specific options (Window Scale, Timestamps, SACK, MSS, MD5, AO) per RFC. Wire format byte-identical with upstream — Wireshark/tcpdump captures match.

### `struct tcp_sock` layout

`include/linux/tcp.h` (extends `inet_connection_sock` extends `inet_sock` extends `sock`). First-cache-line + commonly-accessed fields layout-equivalent to upstream. Fields: snd_una, snd_nxt, rcv_nxt, rcv_wnd, snd_wnd, snd_cwnd, snd_ssthresh, srtt_us, mdev_us, rttvar_us, mss_cache, write_seq, snd_sml, retrans_stamp, undo_marker, undo_retrans, total_retrans, lost, sacked, fackets, retransmits, prior_ssthresh, high_seq, repair_*, rcv_tsval, rcv_tsecr, ts_recent, ts_recent_stamp, advmss, snd_one_byte_recv, max_window, snd_cwnd_clamp, snd_cwnd_used, snd_cwnd_stamp, prior_cwnd, prr_delivered, prr_out, delivered, delivered_ce, lost_skb_hint, retrans_out, lost_out, rate_*, ecn_flags, retrans_count, syn_retries, ack_pending, ato, rcv_*, sa_state, ss_*, rcv_wup, copied_seq, ulp_data, ...

Layout-equivalent.

### TCP options (UAPI bit-identical)

`include/uapi/linux/tcp.h`: TCP_NODELAY, TCP_MAXSEG, TCP_CORK, TCP_KEEPIDLE, TCP_KEEPINTVL, TCP_KEEPCNT, TCP_SYNCNT, TCP_LINGER2, TCP_DEFER_ACCEPT, TCP_WINDOW_CLAMP, TCP_INFO, TCP_QUICKACK, TCP_CONGESTION, TCP_MD5SIG, TCP_THIN_LINEAR_TIMEOUTS, TCP_THIN_DUPACK, TCP_USER_TIMEOUT, TCP_REPAIR, TCP_REPAIR_QUEUE, TCP_QUEUE_SEQ, TCP_REPAIR_OPTIONS, TCP_FASTOPEN, TCP_TIMESTAMP, TCP_NOTSENT_LOWAT, TCP_CC_INFO, TCP_SAVE_SYN, TCP_SAVED_SYN, TCP_REPAIR_WINDOW, TCP_FASTOPEN_CONNECT, TCP_ULP, TCP_MD5SIG_EXT, TCP_FASTOPEN_KEY, TCP_FASTOPEN_NO_COOKIE, TCP_ZEROCOPY_RECEIVE, TCP_INQ, TCP_TX_DELAY, TCP_AO_ADD_KEY, TCP_AO_DEL_KEY, TCP_AO_INFO, TCP_AO_GET_KEYS, TCP_AO_REPAIR. Numeric byte-identical.

### `struct tcp_info` (TCP_INFO sockopt)

UAPI struct exposing per-socket statistics: tcpi_state, tcpi_ca_state, tcpi_retransmits, tcpi_probes, tcpi_backoff, tcpi_options, tcpi_snd_wscale, tcpi_rcv_wscale, tcpi_delivery_rate_app_limited, tcpi_fastopen_client_fail, tcpi_rto, tcpi_ato, tcpi_snd_mss, tcpi_rcv_mss, tcpi_unacked, tcpi_sacked, tcpi_lost, tcpi_retrans, tcpi_fackets, tcpi_last_data_sent, tcpi_last_ack_sent, tcpi_last_data_recv, tcpi_last_ack_recv, tcpi_pmtu, tcpi_rcv_ssthresh, tcpi_rtt, tcpi_rttvar, tcpi_snd_ssthresh, tcpi_snd_cwnd, tcpi_advmss, tcpi_reordering, tcpi_rcv_rtt, tcpi_rcv_space, tcpi_total_retrans, tcpi_pacing_rate, tcpi_max_pacing_rate, tcpi_bytes_acked, tcpi_bytes_received, tcpi_segs_out, tcpi_segs_in, tcpi_notsent_bytes, tcpi_min_rtt, tcpi_data_segs_in, tcpi_data_segs_out, tcpi_delivery_rate, tcpi_busy_time, tcpi_rwnd_limited, tcpi_sndbuf_limited, tcpi_delivered, tcpi_delivered_ce, tcpi_bytes_sent, tcpi_bytes_retrans, tcpi_dsack_dups, tcpi_reord_seen, tcpi_rcv_ooopack, tcpi_snd_wnd, tcpi_rcv_wnd, tcpi_rehash, tcpi_total_rto, tcpi_total_rto_recoveries, tcpi_total_rto_time. Layout byte-identical so `ss -i` / `nstat` work unmodified.

### Sysctls

`net.ipv4.tcp_*` per `Documentation/networking/ip-sysctl.rst`: `tcp_window_scaling`, `tcp_timestamps`, `tcp_sack`, `tcp_dsack`, `tcp_fack`, `tcp_ecn`, `tcp_keepalive_time`, `tcp_keepalive_intvl`, `tcp_keepalive_probes`, `tcp_syn_retries`, `tcp_synack_retries`, `tcp_max_orphans`, `tcp_max_syn_backlog`, `tcp_retries1`, `tcp_retries2`, `tcp_orphan_retries`, `tcp_fin_timeout`, `tcp_max_tw_buckets`, `tcp_tw_reuse`, `tcp_low_latency`, `tcp_no_delay_ack`, `tcp_no_metrics_save`, `tcp_moderate_rcvbuf`, `tcp_mem`, `tcp_wmem`, `tcp_rmem`, `tcp_app_win`, `tcp_adv_win_scale`, `tcp_tw_recycle`, `tcp_abort_on_overflow`, `tcp_stdurg`, `tcp_rfc1337`, `tcp_max_syn_recv_buckets`, `tcp_min_tso_segs`, `tcp_tso_win_divisor`, `tcp_tso_rtt_log`, `tcp_pacing_ss_ratio`, `tcp_pacing_ca_ratio`, `tcp_invalid_ratelimit`, `tcp_l3mdev_accept`, `tcp_min_snd_mss`, `tcp_min_rtt_wlen`, `tcp_autocorking`, `tcp_thin_linear_timeouts`, `tcp_thin_dupack`, `tcp_early_retrans`, `tcp_recovery`, `tcp_fwmark_accept`, `tcp_workaround_signed_windows`, `tcp_dsack_filter`, `tcp_limit_output_bytes`, `tcp_challenge_ack_limit`, `tcp_min_tso_segs`, `tcp_notsent_lowat`, `tcp_fastopen`, `tcp_fastopen_blackhole_timeout_sec`, `tcp_congestion_control`, `tcp_available_congestion_control`, `tcp_allowed_congestion_control`, `tcp_pingpong_thresh`, `tcp_ehash_entries`, `tcp_max_accept_backlog_conns_per_listener`, `tcp_plb_enabled`, `tcp_migrate_req`, `tcp_no_ssthresh_metrics_save`, `tcp_rto_min_us`, plus more recent additions.

Format-identical.

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| TCP option parser (handle malformed options) | `kani::proofs::net::tcp::opt_parse_safety` |
| Send-queue skb manipulation | `kani::proofs::net::tcp::sendq_safety` |
| Receive-queue skb manipulation | `kani::proofs::net::tcp::recvq_safety` |
| RACK/TLP timer state | `kani::proofs::net::tcp::recovery_timer_safety` |
| MD5/AO sigpool buffer | `kani::proofs::net::tcp::sig_buffer_safety` |

### Layer 2: TLA+ models

- `models/net/tcp_state.tla` (mandatory per `net/00-overview.md` Layer 2) — proves TCP RFC 9293 state-machine implementation. Owned here.
- `models/net/tcp_recovery.tla` (NEW) — proves RACK+TLP recovery doesn't lose retransmits across nested-loss scenarios.
- `models/net/tcp_cong_pluggable.tla` (NEW) — proves: switching cong-control algorithm mid-flight via `setsockopt(TCP_CONGESTION)` doesn't violate per-algorithm invariants.

### Layer 3: Kani harnesses for data-structure invariants (declared in `net/00-overview.md`)

| Data structure | Invariant | Harness |
|---|---|---|
| TCP send queue | skb chain by sequence number; no overlap; no gap until rcv_nxt | `kani::proofs::net::tcp::sendq_invariants` |
| TCP receive ofo (out-of-order) queue | skb chain ordered; ranges non-overlapping | `kani::proofs::net::tcp::ofo_invariants` |
| Cong-control algorithm registration | Each algorithm registered at most once; lookup-by-name unique | `kani::proofs::net::tcp::cong_registry_invariants` |

### Layer 4: Functional correctness (mandatory-recommended per `net/00-overview.md`)

- **TCP state-machine RFC 9293 conformance** via Verus — proves: for any sequence of (incoming-segment, app-action, timer-fire) inputs, the resulting state-transition is one of those allowed by RFC 9293's diagram. Highest-leverage Layer-4 proof in the project alongside BPF verifier soundness + crypto algorithm correctness.
- **CUBIC cong-control algebra** via Creusot — proves: CUBIC's cwnd update follows RFC 8312 (or its successor).

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | tcp_sock refcount + per-listener req-sock refcount use `Refcount` | § Mandatory |
| **AUTOSLAB** | tcp_sock allocated via dedicated tcp_hashinfo + tcp_metrics caches; type-tagged | § Mandatory |
| **SYN-cookie defense** (against SYN flood) | Default-on per `net.ipv4.tcp_syncookies=1`; matches upstream | § Default-on |
| **TFO blackhole detection** (defends against TFO-cookie-based DoS) | `net.ipv4.tcp_fastopen_blackhole_timeout_sec` non-zero default | § Default-on |
| **TCP_AO** (modern crypto-authenticated TCP per RFC 5925) | Available via `setsockopt(TCP_AO_ADD_KEY)`; defends BGP/route-server long-lived TCP | (existing; preserved) |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: per-tcp_sock + per-skb (cross-ref `net/skbuff.md`)
- **UDEREF**: sendmsg/recvmsg via iov_iter (cross-ref `lib/usercopy.md`)
- **CONSTIFY**: per-cong-control algorithm vtables are `static const`
- **SIZE_OVERFLOW**: TCP-seq + cwnd + buffer-size arithmetic uses checked operators (32-bit seq wraps are explicit `wrapping_*`)

### Row-2 / GR-RBAC integration

LSM hooks (cross-ref `net/socket-api.md`): `security_socket_accept`, `security_socket_connect`, `security_inet_conn_request`, `security_inet_conn_established`, `security_inet_csk_clone`. GR-RBAC's policy can deny TCP connections by source/dest, classify flows, etc.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above; Layer 4 RFC 9293 conformance is the keystone proof.)

