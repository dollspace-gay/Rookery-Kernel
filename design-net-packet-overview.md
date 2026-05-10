---
title: "Tier-2: net/packet — AF_PACKET (raw L2 + tpacket_v1/v2/v3 ringbuf + fanout)"
tags: ["tier-2", "net", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 wrapper for AF_PACKET — the raw layer-2 socket family used by tcpdump/Wireshark/libpcap, dhclient (raw frames pre-IP), iwd hostapd EAPOL frames, every userspace packet sniffer, every userspace VPN/network-overlay implementation. Components: `af_packet.c` (AF_PACKET socket impl + tpacket_v1/v2/v3 mmap'd ringbuf + PACKET_FANOUT load-balancing across N sockets + per-socket BPF cBPF filter + per-socket xmit + recv hooks + VLAN tag handling + IPv4/IPv6 helpers integration), `diag.c` (NETLINK_PACKET_DIAG for `ss -A packet`), `internal.h`. Three packet socket modes: SOCK_DGRAM (no L2 header), SOCK_RAW (full L2 header), SOCK_SEQPACKET (rare).

### Out of Scope

- Implementation code; 32-bit-only paths

### compatibility contract — outline

- AF_PACKET socket family + sockaddr_ll + tpacket_hdr / tpacket2_hdr / tpacket3_hdr UAPI byte-identical (libpcap + tcpdump + Wireshark + iwd + dhclient consume unchanged).
- Setsockopt PACKET_*: `_RX_RING`, `_TX_RING`, `_VERSION`, `_RESERVE`, `_LOSS`, `_VNET_HDR`, `_TX_HAS_OFF`, `_QDISC_BYPASS`, `_FANOUT`, `_FANOUT_DATA`, `_HDRLEN`, `_AUXDATA`, `_TIMESTAMP`, `_VLAN_*`, `_RECV_OUTPUT`, `_COPY_THRESH`, `_ROLLOVER_STATS`, `_FANOUT_DATA` byte-identical.
- PACKET_FANOUT modes: `_HASH`, `_LB`, `_CPU`, `_RND`, `_ROLLOVER`, `_QM`, `_CBPF`, `_EBPF` byte-identical.
- TPACKET_V3 block-mode w/ block_size + block_nr + frame_size + sleeping_for_frames byte-identical (tpacket_v3 ringbuf consumed by libpcap-mmap, gulp, daq).
- `/proc/net/packet` byte-identical.
- NETLINK_PACKET_DIAG byte-identical.

### tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `net/packet/af-packet.md` | `af_packet.c`: socket impl + setsockopt + xmit/recv hooks |
| `net/packet/tpacket-ringbuf.md` | `af_packet.c` (tpacket_v1/v2/v3 portion): mmap'd ringbuf |
| `net/packet/fanout.md` | `af_packet.c` (fanout portion): per-fanout group load-balance modes |
| `net/packet/diag.md` | `diag.c`: NETLINK_PACKET_DIAG |

### compatibility outline / ac / verification / hardening

- REQ-O1: AF_PACKET UAPI byte-identical.
- REQ-O2: tpacket_v3 ringbuf wire format byte-identical.
- REQ-O3: PACKET_FANOUT load-balance modes byte-identical.
- REQ-O4: TLA+ models (tpacket_v3 producer-consumer block-mode + frame-mode; fanout per-cpu rollover; cBPF/eBPF filter execution race-free).
- REQ-O5: AC: tcpdump captures on physical iface; Wireshark dumpcap; libpcap mmap-ringbuf path achieves line-rate on 10G NIC.
- Hardening: AF_PACKET socket open requires CAP_NET_RAW (defends against unprivileged sniffing); per-namespace scoping; PACKET_TX bypass-qdisc requires CAP_NET_ADMIN.

