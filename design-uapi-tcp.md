---
title: "Tier-5: include/uapi/linux/tcp.h — TCP ABI"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`<linux/tcp.h>` is the TCP **wire-format and socket-option ABI**. It defines `struct tcphdr` (the 20-byte TCP segment header with the 9-bit flag set FIN/SYN/RST/PSH/ACK/URG/ECE/CWR/AE plus 3 reserved bits and the 4-bit data-offset), `union tcp_word_hdr` (lets the kernel treat the header as five `__be32` words for fast flag manipulation), the `TCP_*` `setsockopt(2)`/`getsockopt(2)` option numbers (level `IPPROTO_TCP`), `struct tcp_info` (the large counters block returned by `getsockopt(TCP_INFO)`), `struct tcp_md5sig` and `struct tcp_diag_md5sig` (per-RFC 2385 keyed-MD5 signature), the TCP-AO (RFC 5925) keying structures `tcp_ao_add` / `tcp_ao_del` / `tcp_ao_info_opt` / `tcp_ao_getsockopt` / `tcp_ao_repair`, the TCP-Repair structures (`tcp_repair_opt`, `tcp_repair_window`), the zero-copy receive descriptor `tcp_zerocopy_receive`, and a family of enum tables (`tcp_ca_state`, TCP_FLAG_*, TCPF_CA_*, TCPI_OPT_*, TCPI_ECN_MODE_*, fastopen client-fail reasons). Per-`tcphdr.doff` × 4 = header length in bytes (min 20, max 60). Per-control bits live in the low 9 bits of the 16-bit half-word that also encodes `doff` and `res1`/`ae`. Per-`window`, `check`, `urg_ptr` are network byte order.

This Tier-5 covers `include/uapi/linux/tcp.h` (~524 lines).

### Acceptance Criteria

- [ ] AC-1: `sizeof(struct tcphdr) == 20`.
- [ ] AC-2: `offsetof(struct tcphdr, window) == 14`.
- [ ] AC-3: `sizeof(union tcp_word_hdr) == 20`.
- [ ] AC-4: `TCP_FLAG_SYN == htonl(0x00020000)` (bit pattern matches RFC 793 segment word 3).
- [ ] AC-5: All `TCP_*` socket-option values 1..46 bit-exact (skip 15).
- [ ] AC-6: `TCP_MD5SIG_MAXKEYLEN == 80`, `TCP_AO_MAXKEYLEN == 80`.
- [ ] AC-7: `TCPI_OPT_TIMESTAMPS == 1` .. `TCPI_OPT_TFO_CHILD == 128`.
- [ ] AC-8: `enum tcp_ca_state` values 0..4 bit-exact.
- [ ] AC-9: `TCP_REPAIR_OFF_NO_WP == -1`, `TCP_REPAIR_OFF == 0`, `TCP_REPAIR_ON == 1`.
- [ ] AC-10: `getsockopt(TCP_INFO, &buf, &len)` returns `min(sizeof(tcp_info), len)` bytes.
- [ ] AC-11: `setsockopt(TCP_MD5SIG, …)` without CAP_NET_ADMIN → `-EPERM`.
- [ ] AC-12: `setsockopt(TCP_REPAIR, 2)` (invalid) → `-EINVAL`.
- [ ] AC-13: `setsockopt(TCP_ZEROCOPY_RECEIVE)` with non-zero `reserved` → `-EINVAL`.
- [ ] AC-14: `setsockopt(TCP_AO_ADD_KEY)` with `keylen > 80` → `-EINVAL`.
- [ ] AC-15: `setsockopt(TCP_CONGESTION, "nonexistent")` → `-ENOENT`.
- [ ] AC-16: `tcphdr.doff < 5` on receive → drop.

### Architecture

```
#[repr(C)]
pub struct TcpHdr {
    pub source:  BE16,
    pub dest:    BE16,
    pub seq:     BE32,
    pub ack_seq: BE32,
    pub flags:   BE16,           // doff:4 + res1:3 + ae:1 + cwr/ece/urg/ack/psh/rst/syn/fin
    pub window:  BE16,
    pub check:   Sum16,
    pub urg_ptr: BE16,
}                                // == 20 B

#[repr(C)]
pub union TcpWordHdr {
    pub hdr:   TcpHdr,
    pub words: [BE32; 5],
}
```

`TcpHdr::doff(&self) -> u8`:
1. let w = self.flags.to_host();
2. return ((w >> 12) & 0x0F) as u8.

`TcpHdr::flags9(&self) -> u16`:
1. let w = self.flags.to_host();
2. return (w & 0x01FF) | (((w >> 8) & 0x01) << 8).   // FIN..AE

`TcpHdr::validate(buf: &[u8]) -> Result<&Self, TcpErr>`:
1. if buf.len() < 20: return Err(Truncated).
2. let h = read_as::<TcpHdr>(buf).
3. if h.doff() < 5: return Err(BadDoff).
4. let hlen = (h.doff() as usize) * 4.
5. if buf.len() < hlen: return Err(Truncated).
6. return Ok(h).

`SetSockOpt::md5sig(sk, opt, optlen) -> Result<(), Errno>`:
1. if !ns_capable(CAP_NET_ADMIN): return -EPERM.
2. if optlen < sizeof::<TcpMd5Sig>(): return -EINVAL.
3. let m = read_as::<TcpMd5Sig>(opt).
4. if m.tcpm_keylen > 80: return -EINVAL.
5. tcp_md5_install(sk, m).

`SetSockOpt::ao_add_key(sk, opt, optlen) -> Result<(), Errno>`:
1. if optlen < sizeof::<TcpAoAdd>(): return -EINVAL.
2. let a = read_as::<TcpAoAdd>(opt).
3. if a.keylen > TCP_AO_MAXKEYLEN: return -EINVAL.
4. if a.prefix > prefix_max(a.addr.family): return -EINVAL.
5. tcp_ao_add(sk, a).

`SetSockOpt::repair(sk, v) -> Result<(), Errno>`:
1. if !ns_capable(CAP_NET_ADMIN): return -EPERM.
2. match v: 1 | 0 | -1 ⟹ sk.repair = v; _ ⟹ return -EINVAL.

### Out of Scope

- TCP state-machine implementation (Tier-3 `net/ipv4/tcp_input.c` / `tcp_output.c`)
- Congestion-control modules (Tier-3 `net/ipv4/tcp_cong.c`, individual algorithms)
- MPTCP (`<linux/mptcp.h>` Tier-5 when added)
- INET_DIAG (`<linux/inet_diag.h>` covered separately)
- kTLS (`<linux/tls.h>` Tier-5 when added)
- TCP-AO crypto-algorithm registration (Tier-3)

### abi surface

### Wire-format structures

| Symbol | Size | Field layout |
|---|---|---|
| `struct tcphdr` | 20 B + options | `source` (`__be16`), `dest` (`__be16`), `seq` (`__be32`), `ack_seq` (`__be32`), bitfield `ae:1 + res1:3 + doff:4 + fin:1 + syn:1 + rst:1 + psh:1 + ack:1 + urg:1 + ece:1 + cwr:1` (16-bit; bit order arch-dependent), `window` (`__be16`), `check` (`__sum16`), `urg_ptr` (`__be16`) |
| `union tcp_word_hdr` | 20 B | `hdr` (tcphdr) ∪ `words[5]` (`__be32[5]`) |

The 9 control bits are FIN, SYN, RST, PSH, ACK, URG, ECE, CWR, AE (the most recent: Accurate-ECN bit, formerly NS / reserved).

### TCP_FLAG_* (mask for word 3 of `tcp_word_hdr`)

| Symbol | Value (big-endian u32) | Bit |
|---|---|---|
| `TCP_FLAG_AE` | `htonl(0x01000000)` | "ae" / AccECN bit |
| `TCP_FLAG_CWR` | `htonl(0x00800000)` | |
| `TCP_FLAG_ECE` | `htonl(0x00400000)` | |
| `TCP_FLAG_URG` | `htonl(0x00200000)` | |
| `TCP_FLAG_ACK` | `htonl(0x00100000)` | |
| `TCP_FLAG_PSH` | `htonl(0x00080000)` | |
| `TCP_FLAG_RST` | `htonl(0x00040000)` | |
| `TCP_FLAG_SYN` | `htonl(0x00020000)` | |
| `TCP_FLAG_FIN` | `htonl(0x00010000)` | |
| `TCP_RESERVED_BITS` | `htonl(0x0E000000)` | res1 |
| `TCP_DATA_OFFSET` | `htonl(0xF0000000)` | doff mask |

### TCP general constants

| Symbol | Value | Meaning |
|---|---|---|
| `TCP_MSS_DEFAULT` | 536 | IPv4 default MSS (RFC 1122) |
| `TCP_MSS_DESIRED` | 1220 | IPv6 tunneled / EDNS0 |

### TCP_* socket options (level `IPPROTO_TCP`)

| Symbol | Value | Direction | Semantic |
|---|---|---|---|
| `TCP_NODELAY` | 1 | rw | Disable Nagle |
| `TCP_MAXSEG` | 2 | rw | Limit MSS |
| `TCP_CORK` | 3 | rw | Defer partial segments |
| `TCP_KEEPIDLE` | 4 | rw | Idle before first keepalive (s) |
| `TCP_KEEPINTVL` | 5 | rw | Interval between keepalives (s) |
| `TCP_KEEPCNT` | 6 | rw | Probes before death |
| `TCP_SYNCNT` | 7 | rw | SYN retransmits |
| `TCP_LINGER2` | 8 | rw | FIN_WAIT2 timeout (s) |
| `TCP_DEFER_ACCEPT` | 9 | rw | Wake listener on data (s) |
| `TCP_WINDOW_CLAMP` | 10 | rw | Advertised window cap |
| `TCP_INFO` | 11 | ro | Get `struct tcp_info` |
| `TCP_QUICKACK` | 12 | rw | Disable/reenable delayed ACK |
| `TCP_CONGESTION` | 13 | rw | CC algorithm name (string) |
| `TCP_MD5SIG` | 14 | rw | RFC 2385 keyed MD5 |
| `TCP_THIN_LINEAR_TIMEOUTS` | 16 | rw | Linear RTO for thin streams |
| `TCP_THIN_DUPACK` | 17 | rw | Fast retrans on 1 dupack |
| `TCP_USER_TIMEOUT` | 18 | rw | Loss-retry timeout (ms) |
| `TCP_REPAIR` | 19 | rw | Enter/leave repair mode |
| `TCP_REPAIR_QUEUE` | 20 | rw | Select repair queue |
| `TCP_QUEUE_SEQ` | 21 | rw | Set queue sequence number |
| `TCP_REPAIR_OPTIONS` | 22 | wo | Set saved options |
| `TCP_FASTOPEN` | 23 | rw | Enable TFO listener (qlen) |
| `TCP_TIMESTAMP` | 24 | rw | Override TSval base |
| `TCP_NOTSENT_LOWAT` | 25 | rw | Unsent-bytes EPOLLOUT watermark |
| `TCP_CC_INFO` | 26 | ro | Per-CC info blob |
| `TCP_SAVE_SYN` | 27 | rw | Save incoming SYN |
| `TCP_SAVED_SYN` | 28 | ro | Retrieve saved SYN |
| `TCP_REPAIR_WINDOW` | 29 | rw | Get/set window params (`tcp_repair_window`) |
| `TCP_FASTOPEN_CONNECT` | 30 | rw | Attempt TFO on connect |
| `TCP_ULP` | 31 | rw | Attach Upper-Layer-Protocol (kTLS, MPTCP) |
| `TCP_MD5SIG_EXT` | 32 | rw | MD5 with prefix/ifindex (`tcp_md5sig`) |
| `TCP_FASTOPEN_KEY` | 33 | rw | TFO cookie key |
| `TCP_FASTOPEN_NO_COOKIE` | 34 | rw | TFO without cookie |
| `TCP_ZEROCOPY_RECEIVE` | 35 | rw | Zero-copy receive (`tcp_zerocopy_receive`) |
| `TCP_INQ` | 36 | rw | Cmsg with bytes in rxqueue (alias `TCP_CM_INQ`) |
| `TCP_TX_DELAY` | 37 | rw | Egress delay (usec) |
| `TCP_AO_ADD_KEY` | 38 | wo | RFC 5925 Master Key Tuple add |
| `TCP_AO_DEL_KEY` | 39 | wo | MKT delete |
| `TCP_AO_INFO` | 40 | rw | Per-socket TCP-AO options |
| `TCP_AO_GET_KEYS` | 41 | ro | List MKTs |
| `TCP_AO_REPAIR` | 42 | rw | SNE/ISN repair |
| `TCP_IS_MPTCP` | 43 | ro | Is MPTCP? |
| `TCP_RTO_MAX_MS` | 44 | rw | Max RTO (ms) |
| `TCP_RTO_MIN_US` | 45 | rw | Min RTO (us) |
| `TCP_DELACK_MAX_US` | 46 | rw | Max delayed-ACK (us) |

### TCP-Repair markers

| Symbol | Value | Meaning |
|---|---|---|
| `TCP_REPAIR_ON` | 1 | enter repair |
| `TCP_REPAIR_OFF` | 0 | leave repair, send window probes |
| `TCP_REPAIR_OFF_NO_WP` | -1 | leave repair, no window probes |

`struct tcp_repair_opt { __u32 opt_code; __u32 opt_val; }` — 8 B.
`struct tcp_repair_window { __u32 snd_wl1; __u32 snd_wnd; __u32 max_window; __u32 rcv_wnd; __u32 rcv_wup; }` — 20 B.
`enum { TCP_NO_QUEUE, TCP_RECV_QUEUE, TCP_SEND_QUEUE, TCP_QUEUES_NR }` — TCP_REPAIR_QUEUE values.

### Fastopen-client fail reason

`enum tcp_fastopen_client_fail { TFO_STATUS_UNSPEC, TFO_COOKIE_UNAVAILABLE, TFO_DATA_NOT_ACKED, TFO_SYN_RETRANSMITTED }`.

### TCPI_OPT_* (bits in `tcp_info.tcpi_options`)

| Symbol | Value | Meaning |
|---|---|---|
| `TCPI_OPT_TIMESTAMPS` | 1 | RFC 7323 timestamps negotiated |
| `TCPI_OPT_SACK` | 2 | SACK negotiated |
| `TCPI_OPT_WSCALE` | 4 | Window-scale negotiated |
| `TCPI_OPT_ECN` | 8 | ECN negotiated |
| `TCPI_OPT_ECN_SEEN` | 16 | ECT seen on receive |
| `TCPI_OPT_SYN_DATA` | 32 | SYN-ACK acked SYN data |
| `TCPI_OPT_USEC_TS` | 64 | µsec-resolution timestamps |
| `TCPI_OPT_TFO_CHILD` | 128 | Child socket from TFO-on-SYN |

### TCP CA state (`enum tcp_ca_state` + TCPF_CA_* bitmasks)

| Symbol | Value | TCPF mask |
|---|---|---|
| `TCP_CA_Open` | 0 | `TCPF_CA_Open` = 1 |
| `TCP_CA_Disorder` | 1 | `TCPF_CA_Disorder` = 2 |
| `TCP_CA_CWR` | 2 | `TCPF_CA_CWR` = 4 |
| `TCP_CA_Recovery` | 3 | `TCPF_CA_Recovery` = 8 |
| `TCP_CA_Loss` | 4 | `TCPF_CA_Loss` = 16 |

### TCPI_ECN_MODE_* / TCP_ACCECN_*

| Symbol | Value | Meaning |
|---|---|---|
| `TCPI_ECN_MODE_DISABLED` | 0 | |
| `TCPI_ECN_MODE_RFC3168` | 1 | classic ECN |
| `TCPI_ECN_MODE_ACCECN` | 2 | Accurate-ECN |
| `TCPI_ECN_MODE_PENDING` | 3 | |
| `TCP_ACCECN_OPT_NOT_SEEN` | 0 | |
| `TCP_ACCECN_OPT_EMPTY_SEEN` | 1 | |
| `TCP_ACCECN_OPT_COUNTER_SEEN` | 2 | |
| `TCP_ACCECN_OPT_FAIL_SEEN` | 3 | |
| `TCP_ACCECN_ACE_FAIL_SEND` | BIT(0) | |
| `TCP_ACCECN_ACE_FAIL_RECV` | BIT(1) | |
| `TCP_ACCECN_OPT_FAIL_SEND` | BIT(2) | |
| `TCP_ACCECN_OPT_FAIL_RECV` | BIT(3) | |

### `struct tcp_info` (extensible counters block)

Fixed ABI prefix (numeric layout — additions are append-only):
`tcpi_state` (u8), `tcpi_ca_state` (u8), `tcpi_retransmits` (u8), `tcpi_probes` (u8), `tcpi_backoff` (u8), `tcpi_options` (u8), `tcpi_snd_wscale:4 + tcpi_rcv_wscale:4` (u8), `tcpi_delivery_rate_app_limited:1 + tcpi_fastopen_client_fail:2` (u8), `tcpi_rto` / `tcpi_ato` / `tcpi_snd_mss` / `tcpi_rcv_mss` (u32×4), `tcpi_unacked` / `_sacked` / `_lost` / `_retrans` / `_fackets` (u32×5), `tcpi_last_data_sent` / `_ack_sent` / `_data_recv` / `_ack_recv` (u32×4), `tcpi_pmtu` / `_rcv_ssthresh` / `_rtt` / `_rttvar` / `_snd_ssthresh` / `_snd_cwnd` / `_advmss` / `_reordering` (u32×8), `tcpi_rcv_rtt` / `_rcv_space` (u32×2), `tcpi_total_retrans` (u32), `tcpi_pacing_rate` / `_max_pacing_rate` / `_bytes_acked` / `_bytes_received` (u64×4), `tcpi_segs_out` / `_segs_in` (u32×2), `tcpi_notsent_bytes` / `_min_rtt` / `_data_segs_in` / `_data_segs_out` (u32×4), `tcpi_delivery_rate` (u64), `tcpi_busy_time` / `_rwnd_limited` / `_sndbuf_limited` (u64×3), `tcpi_delivered` / `_delivered_ce` (u32×2), `tcpi_bytes_sent` / `_bytes_retrans` (u64×2), `tcpi_dsack_dups` / `_reord_seen` / `_rcv_ooopack` / `_snd_wnd` / `_rcv_wnd` / `_rehash` (u32×6), `tcpi_total_rto` / `_total_rto_recoveries` (u16×2), `tcpi_total_rto_time` / `_received_ce` (u32×2), `_delivered_e1_bytes` / `_e0_bytes` / `_ce_bytes` / `_received_e1_bytes` / `_e0_bytes` / `_ce_bytes` (u32×6), bitfield `tcpi_ecn_mode:2 + tcpi_accecn_opt_seen:2 + tcpi_accecn_fail_mode:4 + tcpi_options2:24` (u32).

### Netlink attrs for SCM_TIMESTAMPING_OPT_STATS

`enum { TCP_NLA_PAD, BUSY, RWND_LIMITED, SNDBUF_LIMITED, DATA_SEGS_OUT, TOTAL_RETRANS, PACING_RATE, DELIVERY_RATE, SND_CWND, REORDERING, MIN_RTT, RECUR_RETRANS, DELIVERY_RATE_APP_LMT, SNDQ_SIZE, CA_STATE, SND_SSTHRESH, DELIVERED, DELIVERED_CE, BYTES_SENT, BYTES_RETRANS, DSACK_DUPS, REORD_SEEN, SRTT, TIMEOUT_REHASH, BYTES_NOTSENT, EDT, TTL, REHASH }`.

### MD5 / AO keying

| Symbol | Value | Meaning |
|---|---|---|
| `TCP_MD5SIG_MAXKEYLEN` | 80 | MD5 key length cap |
| `TCP_MD5SIG_FLAG_PREFIX` | 0x1 | use `tcpm_prefixlen` |
| `TCP_MD5SIG_FLAG_IFINDEX` | 0x2 | use `tcpm_ifindex` |
| `TCP_AO_MAXKEYLEN` | 80 | AO key length cap |
| `TCP_AO_KEYF_IFINDEX` | 1 | ifindex set |
| `TCP_AO_KEYF_EXCLUDE_OPT` | 2 | exclude other TCP options from MAC |

Structs: `tcp_md5sig` (storage + flags + prefix + keylen + ifindex + key[80]), `tcp_diag_md5sig` (per-INET_DIAG dump), `tcp_ao_add` (storage + alg_name[64] + ifindex + flags + key[80], 8-byte aligned), `tcp_ao_del`, `tcp_ao_info_opt`, `tcp_ao_getsockopt`, `tcp_ao_repair { snt_isn (__be32), rcv_isn (__be32), snd_sne (u32), rcv_sne (u32) }`.

### Zero-copy receive

`struct tcp_zerocopy_receive { address (u64), length (u32 in/out), recv_skip_hint (u32 out), inq (u32 out), err (s32 out), copybuf_address (u64), copybuf_len (s32 in/out), flags (u32 in), msg_control (u64), msg_controllen (u64), msg_flags (u32), reserved (u32) }`. Flag: `TCP_RECEIVE_ZEROCOPY_FLAG_TLB_CLEAN_HINT = 0x1`.

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct tcphdr` | per-TCP wire header | `TcpHdr` |
| `union tcp_word_hdr` | per-flag-word view | `TcpWordHdr` |
| `TCP_FLAG_*` | per-flag-mask | `TcpFlag` |
| `TCP_*` sockopt names | per-option | `TcpSockoptName` |
| `struct tcp_info` | per-info-dump | `TcpInfo` |
| `struct tcp_md5sig` / `tcp_diag_md5sig` | per-RFC2385 key | `TcpMd5Sig` / `TcpDiagMd5Sig` |
| `struct tcp_ao_*` | per-RFC5925 keys | `TcpAoAdd` / `TcpAoDel` / `TcpAoInfoOpt` / `TcpAoGetsockopt` / `TcpAoRepair` |
| `struct tcp_repair_opt` / `tcp_repair_window` | per-CRIU repair | `TcpRepairOpt` / `TcpRepairWindow` |
| `struct tcp_zerocopy_receive` | per-ZC receive | `TcpZerocopyReceive` |
| `enum tcp_ca_state` | per-CC FSM | `TcpCaState` |
| `enum tcp_fastopen_client_fail` | per-TFO fail | `TfoClientFail` |
| `TCPI_OPT_*` | per-info-flags | `TcpiOpt` |
| `TCPI_ECN_MODE_*` | per-ECN mode | `TcpiEcnMode` |

### compatibility contract

REQ-1: `struct tcphdr` is **exactly 20 bytes**. Field order: `source` (`__be16`), `dest` (`__be16`), `seq` (`__be32`), `ack_seq` (`__be32`), the 16-bit flags-word, `window` (`__be16`), `check` (`__sum16`), `urg_ptr` (`__be16`).

REQ-2: The flags word, viewed as the wire `__be16`, has high nibble = `doff` (4 bits), then `res1` (3 bits), `ae` (1 bit), `cwr`, `ece`, `urg`, `ack`, `psh`, `rst`, `syn`, `fin` (each 1 bit, total 9 control bits). The C bitfield layout is conditional on `__LITTLE_ENDIAN_BITFIELD` / `__BIG_ENDIAN_BITFIELD` and chosen so the on-wire encoding is invariant.

REQ-3: `doff` is in **32-bit words**; header length = `doff * 4`. Valid range 5..15 (20..60 B).

REQ-4: `union tcp_word_hdr` provides a 5-word view; `tcp_flag_word(tp) == words[3]` so the `TCP_FLAG_*` masks can be ANDed in network byte order without per-byte shifts.

REQ-5: `TCP_FLAG_*` masks are pre-htonl'd 32-bit values for direct comparison against `tcp_flag_word(tp)`.

REQ-6: `TCP_RESERVED_BITS` (htonl 0x0E000000) covers `res1` (3 bits); senders must zero these bits; receivers must ignore.

REQ-7: `TCP_DATA_OFFSET` mask (htonl 0xF0000000) covers `doff`; receive path must reject `doff < 5`.

REQ-8: All `TCP_*` socket-option numbers 1..46 are **stable ABI**. Value 15 is intentionally skipped (formerly TCP_AO; now reassigned upstream — kept reserved).

REQ-9: `TCP_INFO` returns a `struct tcp_info`; the structure is **append-only**: new fields appended at the end. `getsockopt` `optlen` is in/out — userspace passes max buffer; kernel returns actual bytes written.

REQ-10: `TCP_CONGESTION` value is a NUL-terminated ASCII name; max length `TCP_CA_NAME_MAX = 16` (defined in `<net/tcp.h>` but observable via the ABI).

REQ-11: `TCP_MD5SIG` and `TCP_MD5SIG_EXT` accept a `struct tcp_md5sig` whose `tcpm_addr` is a `__kernel_sockaddr_storage`; key length 0..80 (`TCP_MD5SIG_MAXKEYLEN`); installing the key requires `CAP_NET_ADMIN`.

REQ-12: `TCP_AO_ADD_KEY` accepts a `struct tcp_ao_add` (8-byte aligned). `prefix` ≤ 32 (IPv4) or ≤ 128 (IPv6); `sndid`/`rcvid` are 8-bit; `alg_name[64]` NUL-terminated crypto algorithm name; `key[80]` raw key.

REQ-13: `TCP_AO_DEL_KEY` removes by `(addr, prefix, sndid, rcvid, ifindex)`. `del_async` is valid only on listen sockets.

REQ-14: `TCP_AO_INFO` `setsockopt`: each bit-flag is in/out depending on `set_*` bits. `accept_icmps` controls whether ICMP-PMTU is honoured for AO-protected flows.

REQ-15: `TCP_REPAIR` accepts {1, 0, -1} only. Toggling requires `CAP_NET_ADMIN`. While in repair mode, `TCP_REPAIR_QUEUE`, `TCP_QUEUE_SEQ`, `TCP_REPAIR_OPTIONS`, `TCP_REPAIR_WINDOW`, `TCP_AO_REPAIR` are valid.

REQ-16: `TCP_FASTOPEN` accepts a `int qlen` (listen-side qlen); `TCP_FASTOPEN_CONNECT` accepts `0|1`; `TCP_FASTOPEN_KEY` accepts 16-byte (one key) or 32-byte (primary+secondary) buffers.

REQ-17: `TCP_ZEROCOPY_RECEIVE` accepts `struct tcp_zerocopy_receive` and **must** zero `reserved`. Returned `length` ≤ requested length; `recv_skip_hint` indicates extra bytes to skip due to TLB alignment.

REQ-18: `TCP_INQ` enables a `SCM_TCP_INQ` cmsg on each `recvmsg(2)` reporting bytes still in the rx queue.

REQ-19: `TCP_NOTSENT_LOWAT` value 0 ⟹ disabled; otherwise sets the threshold in bytes below which `EPOLLOUT` is reported.

REQ-20: All multi-byte fields in `tcphdr` are network byte order; `tcp_info` fields are host byte order; AO `key[]` is raw bytes.

REQ-21: `TCP_MD5SIG_FLAG_IFINDEX` requires `tcpm_ifindex != 0`; `TCP_MD5SIG_FLAG_PREFIX` requires `tcpm_prefixlen ≤ 32/128` per AF.

REQ-22: `TCP_AO_GET_KEYS` is a vector dump; userspace passes a `(struct tcp_ao_getsockopt)[]` of size `nkeys`; kernel fills as many as fit and reports actual count via `nkeys` out.

REQ-23: `tcp_ao_repair { snt_isn, rcv_isn, snd_sne, rcv_sne }` lets CRIU restore the 64-bit sequence-number-extension (SNE) state for AO-protected sockets.

REQ-24: `TCP_CC_INFO` returns a CC-algorithm-specific struct; layout varies per algorithm.

REQ-25: `TCP_SAVED_SYN` returns the saved SYN segment (with options); cleared after read.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `tcphdr_size_locked` | INVARIANT | `mem::size_of::<TcpHdr>() == 20`. |
| `tcp_word_hdr_alias` | INVARIANT | union arms share storage; `mem::size_of::<TcpWordHdr>() == 20`. |
| `tcp_flag_masks_pre_htonl` | INVARIANT | `TCP_FLAG_SYN.to_host() == 0x00020000`. |
| `doff_min_5` | INVARIANT | per-validate: rejects `doff < 5`. |
| `md5_keylen_bounded` | INVARIANT | per-set: `keylen ≤ 80`. |
| `ao_keylen_bounded` | INVARIANT | per-set: `keylen ≤ 80`. |
| `repair_only_3_values` | INVARIANT | per-set: {-1, 0, 1}. |
| `info_optlen_in_out` | INVARIANT | per-getsockopt: returned len = min(req, sizeof). |

### Layer 2: TLA+

`uapi/headers/tcp.tla`:
- Models TCP socket-option state machine + repair lifecycle.
- Properties:
  - `safety_no_unprivileged_md5` — per-`TCP_MD5SIG`: rejects without `CAP_NET_ADMIN`.
  - `safety_ao_key_unique_per_id` — per-`TCP_AO_ADD_KEY`: rejects collision on (addr,prefix,sndid).
  - `safety_repair_off_drains` — per-`TCP_REPAIR=0`: window probes sent.
  - `liveness_info_returns_state` — per-`TCP_INFO`: returns within bounded buffer.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `TcpHdr::validate` post: ret OK ⟹ buf.len() ≥ 20 ∧ doff ≥ 5 ∧ buf.len() ≥ doff*4 | `TcpHdr::validate` |
| `SetSockOpt::md5sig` post: CAP_NET_ADMIN ∧ keylen ≤ 80 | `SetSockOpt::md5sig` |
| `SetSockOpt::ao_add_key` post: keylen ≤ 80 ∧ prefix ≤ family-max | `SetSockOpt::ao_add_key` |
| `SetSockOpt::repair` post: v ∈ {-1, 0, 1} ∨ -EINVAL | `SetSockOpt::repair` |
| `GetSockOpt::info` post: written ≤ min(req, sizeof::<TcpInfo>()) | `GetSockOpt::info` |

### Layer 4: Verus/Creusot functional

RFC 793 / RFC 7323 / RFC 2385 / RFC 5925 wire equivalence: `tcphdr` bit layout matches RFC 793 §3.1; option masks match RFC 793 §3.5. `tcp_md5sig` and `tcp_ao_add` MAC inputs match RFC 2385 / RFC 5925.

### hardening — grsecurity/pax-style reinforcement

- **GRKERNSEC_NO_SIMULT_CONNECT** — per-TCP: drop SYN-pair simultaneous-open at the inbound socket, removing a rare-but-covert connect path that bypasses firewall hole-punch policy.
- **GRKERNSEC_BLACKHOLE** — per-TCP: silently drop SYN to closed ports instead of sending RST; defeats classical `nmap -sS` port-scan.
- **GRKERNSEC_RANDNET** — per-TCP: RFC 6528-grade ISN randomization plus IP-id randomization, ephemeral port randomization within `IP_LOCAL_PORT_RANGE`, and TFO cookie key rotation.
- **PaX UDEREF on TCP_MD5SIG / TCP_AO_ADD_KEY** — per-setsockopt: copy key + addr storage under UDEREF; pre-copy `optlen` bound check; refuse on partial copy.
- **CAP_NET_RAW gating on raw-socket spoofing** — per-`IPPROTO_TCP SOCK_RAW`: refuse from non-init user-ns; `IP_HDRINCL` raw TCP injection capable of forging `tcphdr.source/dest` is gated.
- **GRKERNSEC_PROC for /proc/net** — per-`/proc/net/tcp`, `/proc/net/tcp6`: hide foreign-netns sockets including `(saddr,daddr,sport,dport)` tuples to deny remote-process fingerprinting.
- **RANDKSTACK at recv/send entry** — per-`tcp_v4_rcv` / `tcp_sendmsg`: re-randomize kernel-stack offset; resists ROP via crafted segment headers / large option blocks.
- **TCP MD5 / AO key handling** — keys stored in non-swappable kmem with `mlock`-equivalent protection; constant-time HMAC compares; `TCP_AO_KEYF_EXCLUDE_OPT` honoured so middlebox-mangled options do not break authentication; `accept_icmps=0` default — refuse to act on ICMP errors for AO flows.
- **TCP_FASTOPEN_KEY rotated** — per-init: refuse `TCP_FASTOPEN_NO_COOKIE` outside init-ns; rotate the TFO key on key-compromise indication; mlock the key buffer.
- **TCP_REPAIR confined** — per-init-ns CAP_NET_ADMIN: refuse `TCP_REPAIR_ON` from a delegated user-ns (CRIU running unprivileged); without this gate, repair forges arbitrary sequence numbers.
- **TCP_ZEROCOPY_RECEIVE address checks** — per-call: verify `address` is within the caller's user-VA range under UDEREF; refuse if mapping is non-anonymous-private or has `VM_LOCKED` mismatch.
- **TCP_ULP module load gated** — per-`TCP_ULP="tls"|"mptcp"`: refuse module autoload from non-init user-ns; prevents auto-loading a vulnerable module via setsockopt.
- **TCP_INFO leak-minimized** — per-`getsockopt(TCP_INFO)`: in hardened mode, zero kernel-only fields (e.g. precise `min_rtt`, `pacing_rate`) to reduce off-path traffic-analysis signal.

