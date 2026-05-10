---
title: "Tier-2: net/sctp — SCTP (RFC 4960 multi-streamed reliable + IPv4/IPv6 + auth + sock-diag)"
tags: ["tier-2", "net", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 wrapper for SCTP (Stream Control Transmission Protocol, RFC 4960) — message-oriented + reliable + multi-stream + multi-homing + Path MTU discovery socket transport, used by SS7 over IP signaling (M2PA / M2UA / M3UA / SUA), Diameter telco AAA, mobile-network 5G N2 + S1AP, RDS/IB. Components: **socket family** (`socket.c` + `protocol.c` + `ipv6.c`: AF_INET/AF_INET6 SCTP socket impl), **association** (`associola.c` + `endpointola.c` + `bind_addr.c`), **transport + path mgmt** (`transport.c` + `output.c` + `outqueue.c` + `input.c` + `inqueue.c` + `sm_*.c` + `transport.c`), **chunks + parsing** (`chunk.c` + `sm_make_chunk.c`), **state machine** (`sm_*.c`: per-event state machine), **stream + scheduler** (`stream.c` + `stream_sched*.c`: per-stream prio scheduling FCFS/PRIO/RR), **auth** (`auth.c`: SCTP-AUTH RFC 4895 chunk auth via shared keys), **diag** (`diag.c`: sock-diag for `ss -S`), **proc + sysfs** (`proc.c` + `sysctl.c`), **offload** (`offload.c`: GRO/GSO/LRO offload), **uls + tsnmap** (`ulpqueue.c` + `tsnmap.c`: per-stream upper-layer queue + TSN map), **misc** (`debug.c`).

### Out of Scope

- Implementation code; 32-bit-only paths

### compatibility contract — outline

- AF_INET[6] socket types `SOCK_STREAM`, `SOCK_SEQPACKET` w/ IPPROTO_SCTP byte-identical UAPI.
- SCTP-specific socket options: `SCTP_PARTIAL_DELIVERY_POINT`, `SCTP_FRAGMENT_INTERLEAVE`, `SCTP_PRIMARY_ADDR`, `SCTP_PEER_ADDR_PARAMS`, `SCTP_INITMSG`, `SCTP_DEFAULT_SEND_PARAM`, `SCTP_EVENTS`, `SCTP_AUTOCLOSE`, `SCTP_RTOINFO`, `SCTP_MAXSEG`, `SCTP_NODELAY`, `SCTP_DISABLE_FRAGMENTS`, `SCTP_AUTH_*`, `SCTP_STREAM_*`, `SCTP_RESET_*`. Wire format byte-identical (lksctp-tools + every SCTP userspace consumes unchanged).
- `/proc/net/sctp/{snmp,assocs,eps,remaddr}` byte-identical.
- `/proc/sys/net/sctp/{addip_enable,addip_noauth_enable,assoc_valid_cookie_life,association_max_retrans,auth_enable,cookie_hmac_alg,default_auto_asconf,encap_port,hb_interval,intl_enable,max_autoclose,max_burst,max_init_retransmits,nat_friendly,path_max_retrans,pf_enable,pf_expose,pf_retrans,prsctp_enable,reconf_enable,rcvbuf_policy,rto_alpha_exp_divisor,rto_beta_exp_divisor,rto_initial,rto_max,rto_min,rwnd_update_shift,sack_timeout,sctp_mem,sctp_rmem,sctp_wmem,sndbuf_policy,strict_init_resp_validation,valid_cookie_life}` byte-identical.

### tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `net/sctp/socket.md` | `socket.c` + `protocol.c` + `ipv6.c`: socket impl |
| `net/sctp/associola.md` | `associola.c` + `endpointola.c` + `bind_addr.c`: association + endpoint + multi-bind |
| `net/sctp/transport.md` | `transport.c`: per-path transport |
| `net/sctp/output-input.md` | `output.c` + `outqueue.c` + `input.c` + `inqueue.c`: I/O paths |
| `net/sctp/state-machine.md` | `sm_*.c`: per-event state machine |
| `net/sctp/chunk.md` | `chunk.c` + `sm_make_chunk.c`: chunk parsing + emission |
| `net/sctp/stream.md` | `stream.c` + `stream_sched*.c`: per-stream prio scheduling |
| `net/sctp/auth.md` | `auth.c`: SCTP-AUTH RFC 4895 |
| `net/sctp/diag.md` | `diag.c`: sock-diag |
| `net/sctp/proc-sysctl.md` | `proc.c` + `sysctl.c` |
| `net/sctp/offload.md` | `offload.c`: GRO/GSO/LRO |
| `net/sctp/ulpq-tsnmap.md` | `ulpqueue.c` + `tsnmap.c` |

### compatibility outline / ac / verification / hardening

- REQ-O1: SCTP socket UAPI byte-identical (lksctp-tools consumes unchanged).
- REQ-O2: `/proc/net/sctp/*` + sysctls byte-identical.
- REQ-O3: SCTP-AUTH RFC 4895 + SCTP add-IP RFC 5061 + DATA chunk fragmentation per RFC 4960 byte-identical.
- REQ-O4: TLA+ models (association state machine COOKIE-WAIT → COOKIE-ECHOED → ESTABLISHED → SHUTDOWN-PENDING → SHUTDOWN-SENT → SHUTDOWN-ACK-SENT → CLOSED; SACK + retransmit selective-ack correctness; multi-homing path-failure detection).
- REQ-O5: AC: lksctp-tools test suite passes; M3UA daemon connects + signals to remote SCTP peer.
- Hardening: row-1 features per `00-security-principles.md`; SCTP-AUTH default-on for new endpoints; nat_friendly default-off (defense against blind-spoof); per-LSM hook on socket-create.

