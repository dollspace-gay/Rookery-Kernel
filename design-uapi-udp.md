---
title: "Tier-5: include/uapi/linux/udp.h — UDP ABI"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`<linux/udp.h>` is the UDP **wire-format and socket-option ABI**. It defines `struct udphdr` (the fixed 8-byte UDP datagram header: source port, destination port, length, checksum), the `UDP_*` `setsockopt(2)`/`getsockopt(2)` option numbers at level `IPPROTO_UDP`, and the `UDP_ENCAP_*` encapsulation-type values consumed by `setsockopt(UDP_ENCAP, …)` to tell the kernel that received UDP datagrams carry a known L2/L3 payload (IPsec NAT-T, L2TPv3, GTP, RXRPC, OpenVPN, etc.). Per-`udphdr.source`/`dest` are 16-bit network-byte-order port numbers. Per-`udphdr.len` is the **total** UDP length (header + payload) in bytes, network order. Per-`udphdr.check` is the network-order one's-complement checksum over a pseudo-header (IPv4/IPv6) plus the UDP header and payload; for IPv4 a value of 0 means "no checksum"; for IPv6 the checksum is mandatory (RFC 2460) unless overridden via `UDP_NO_CHECK6_TX`/`_RX`. Per-`UDP_CORK` defers transmission until uncorked. Per-`UDP_SEGMENT` enables UDP-GSO (RFC 768 segmentation by NIC). Per-`UDP_GRO` enables UDP-GRO on receive (coalesces datagrams).

This Tier-5 covers `include/uapi/linux/udp.h` (~50 lines).

### Acceptance Criteria

- [ ] AC-1: `sizeof(struct udphdr) == 8`.
- [ ] AC-2: `offsetof(struct udphdr, len) == 4 && offsetof(struct udphdr, check) == 6`.
- [ ] AC-3: All four fields are `__be16` / `__sum16` (network order, 16-bit).
- [ ] AC-4: `UDP_CORK == 1`, `UDP_ENCAP == 100`, `UDP_NO_CHECK6_TX == 101`, `UDP_NO_CHECK6_RX == 102`, `UDP_SEGMENT == 103`, `UDP_GRO == 104`.
- [ ] AC-5: `UDP_ENCAP_ESPINUDP_NON_IKE == 1` .. `UDP_ENCAP_OVPNINUDP == 8`.
- [ ] AC-6: `TCP_ENCAP_ESPINTCP == 7` (cross-listed value matches xfrm encap-type table).
- [ ] AC-7: `setsockopt(UDP_ENCAP, TCP_ENCAP_ESPINTCP)` returns `-EINVAL`.
- [ ] AC-8: `setsockopt(UDP_ENCAP, 0)` clears the encap handler.
- [ ] AC-9: `setsockopt(UDP_NO_CHECK6_TX, 1)` on AF_INET socket → `-EAFNOSUPPORT`.
- [ ] AC-10: IPv6 receive of `udphdr.check == 0` without `UDP_NO_CHECK6_RX` → drop + counter.
- [ ] AC-11: `setsockopt(UDP_SEGMENT, 65508)` → `-EINVAL`.
- [ ] AC-12: `setsockopt(UDP_ENCAP, UDP_ENCAP_L2TPINUDP)` without CAP_NET_ADMIN → `-EPERM`.
- [ ] AC-13: `setsockopt(UDP_CORK, 1)` queues until cleared; `getsockopt(UDP_CORK)` returns 1.
- [ ] AC-14: IPv4 receive of `udphdr.check == 0` accepted without drop.

### Architecture

```
#[repr(C)]
pub struct UdpHdr {
    pub source: BE16,
    pub dest:   BE16,
    pub len:    BE16,
    pub check:  Sum16,
}                              // == 8 B

#[repr(u32)]
pub enum UdpEncap {
    None              = 0,
    EspInUdpNonIke    = 1,
    EspInUdp          = 2,
    L2tpInUdp         = 3,
    Gtp0              = 4,
    Gtp1U             = 5,
    RxRpc             = 6,
    EspInTcp          = 7,    // xfrm encap-type, not valid via UDP_ENCAP
    OvpnInUdp         = 8,
}
```

`UdpHdr::validate(buf: &[u8], pseudo: &Pseudo, af: AddrFamily) -> Result<&Self, UdpErr>`:
1. if buf.len() < 8: return Err(Truncated).
2. let h = read_as::<UdpHdr>(buf).
3. let l = h.len.to_host() as usize.
4. if l < 8 ∨ buf.len() < l: return Err(BadLength).
5. if h.check.to_host() == 0:
   - if af == AddrFamily::Inet6 ∧ !sk.no_check6_rx: return Err(MissingChecksumV6).
6. else if !pseudo_checksum_ok(pseudo, &buf[..l]): return Err(BadChecksum).
7. return Ok(h).

`SetSockOpt::udp_encap(sk, v) -> Result<(), Errno>`:
1. match v:
   - 0: sk.encap = None.
   - 1 | 2: sk.encap = match v { 1 => EspInUdpNonIke, 2 => EspInUdp, _ => unreachable!() }.
   - 3 | 4 | 5: if !ns_capable(CAP_NET_ADMIN): return -EPERM.
   - 6: sk.encap = RxRpc.
   - 7: return -EINVAL.   // ESPINTCP is xfrm, not UDP encap
   - 8: sk.encap = OvpnInUdp.
   - _: return -EINVAL.
2. return Ok.

`SetSockOpt::udp_no_check6_tx(sk, on) -> Result<(), Errno>`:
1. if sk.family != AF_INET6: return -EAFNOSUPPORT.
2. sk.no_check6_tx = on.

`SetSockOpt::udp_segment(sk, mss) -> Result<(), Errno>`:
1. if mss > 65507: return -EINVAL.   // max UDP payload (65535 - 20 IP - 8 UDP)
2. sk.gso_size = mss as u16.

### Out of Scope

- `<linux/udplite.h>` UDP-Lite cscov options (covered separately)
- `<linux/l2tp.h>` L2TPv3 (covered separately)
- `<linux/gtp.h>` GTP-U / GTP-C control (covered separately)
- xfrm encap (`<linux/xfrm.h>`) — `TCP_ENCAP_ESPINTCP` lives there
- UDP implementation (`net/ipv4/udp.c`, Tier-3)
- AF_RXRPC implementation (`net/rxrpc/`, Tier-3)
- OpenVPN data-channel offload (`drivers/net/ovpn/`, Tier-4 driver-doc)

### abi surface

### Wire-format structure

| Symbol | Size | Field layout | Endianness |
|---|---|---|---|
| `struct udphdr` | 8 B | `source` (`__be16`), `dest` (`__be16`), `len` (`__be16`), `check` (`__sum16`) | network |

### UDP_* socket options (level `IPPROTO_UDP`)

| Symbol | Value | Direction | Semantic |
|---|---|---|---|
| `UDP_CORK` | 1 | rw | Defer transmission until cleared (corked-output) |
| `UDP_ENCAP` | 100 | rw | Set encapsulation type (`UDP_ENCAP_*` value) |
| `UDP_NO_CHECK6_TX` | 101 | rw | IPv6: send with checksum field set to 0 |
| `UDP_NO_CHECK6_RX` | 102 | rw | IPv6: accept incoming with checksum 0 |
| `UDP_SEGMENT` | 103 | rw | UDP-GSO segment size (per-`sendmsg` cmsg also) |
| `UDP_GRO` | 104 | rw | Enable UDP-GRO on receive |

(Values 2..99 are reserved; values 10..11 in particular are reserved for UDP-Lite `UDPLITE_SEND_CSCOV` / `UDPLITE_RECV_CSCOV` and intentionally not redeclared here.)

### UDP_ENCAP_* (`UDP_ENCAP` setsockopt values)

| Symbol | Value | Reference |
|---|---|---|
| `UDP_ENCAP_ESPINUDP_NON_IKE` | 1 | draft-ietf-ipsec-nat-t-ike-00/01 (unused; legacy) |
| `UDP_ENCAP_ESPINUDP` | 2 | RFC 3948 (IPsec ESP-in-UDP NAT-T, draft-ietf-ipsec-udp-encaps-06) |
| `UDP_ENCAP_L2TPINUDP` | 3 | RFC 2661 (L2TPv2 in UDP) |
| `UDP_ENCAP_GTP0` | 4 | GSM TS 09.60 (GTP version 0) |
| `UDP_ENCAP_GTP1U` | 5 | 3GPP TS 29.060 (GTP-U) |
| `UDP_ENCAP_RXRPC` | 6 | Linux AF_RXRPC |
| `TCP_ENCAP_ESPINTCP` | 7 | (xfrm encap-type cross-listed here; ESP-in-TCP, not UDP) |
| `UDP_ENCAP_OVPNINUDP` | 8 | OpenVPN data-channel encapsulation |

(The task brief lists `GENEVE`, `RTP`, `DTLS` — these are **not** present in the upstream `<linux/udp.h>` at baseline `7.1.0-rc2`. GENEVE encap is handled via `tunnel`/`fou` not `UDP_ENCAP`; RTP and DTLS are pure userspace protocols carried over UDP without kernel encap-type tagging. Documented here for the brief; ABI lists only the 8 values above.)

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct udphdr` | per-UDP wire header | `UdpHdr` |
| `UDP_CORK` / `UDP_ENCAP` / `UDP_NO_CHECK6_TX` / `UDP_NO_CHECK6_RX` / `UDP_SEGMENT` / `UDP_GRO` | per-sockopt | `UdpSockoptName` |
| `UDP_ENCAP_*` constants | per-encap-type | `UdpEncap` |
| `TCP_ENCAP_ESPINTCP` (cross-listed) | per-xfrm-encap | `XfrmEncapType` |

### compatibility contract

REQ-1: `struct udphdr` is **exactly 8 bytes**. Field order is `source` (`__be16`), `dest` (`__be16`), `len` (`__be16`), `check` (`__sum16`). All four fields are **network byte order**.

REQ-2: `udphdr.len` is the **total length** (header + payload) in bytes, in network order. Minimum value 8 (header only, zero payload). Maximum value 65535. The kernel rejects datagrams whose `len` does not match the IPv4 `tot_len - ihl*4` (or the IPv6 payload length minus extension headers).

REQ-3: `udphdr.check` is the one's-complement Internet checksum over the IPv4/IPv6 pseudo-header, the UDP header (with `check` zeroed), and the payload. For **IPv4**: value 0 means "no checksum"; the kernel will accept such datagrams. For **IPv6**: value 0 is illegal (RFC 2460) unless either:
- `UDP_NO_CHECK6_RX = 1` on the receiving socket (accept), or
- `UDP_NO_CHECK6_TX = 1` on the sending socket (transmit with 0).

REQ-4: `setsockopt(UDP_CORK, 1)` queues outgoing data until `UDP_CORK = 0` is set; combined with `MSG_MORE` to coalesce small writes into a single datagram. Cleared on socket close.

REQ-5: `setsockopt(UDP_ENCAP, type)` accepts one of the `UDP_ENCAP_*` values (1..8) or 0 to disable. Each non-zero value activates a kernel-side decapsulation handler. Some types (`UDP_ENCAP_ESPINUDP`, `_L2TPINUDP`, `_GTP0`, `_GTP1U`, `_RXRPC`, `_OVPNINUDP`) require the corresponding module to be available; without the module the call may return `-ENOPROTOOPT`.

REQ-6: `UDP_ENCAP_L2TPINUDP` (3) and `UDP_ENCAP_GTP0`/`GTP1U` (4/5) require `CAP_NET_ADMIN` in the network namespace (since they hijack receive flow for tunneling).

REQ-7: `UDP_ENCAP_ESPINUDP` (2) is used by `pluto` / IKE userspace; setting it requires the socket be `IPPROTO_UDP` `SOCK_DGRAM` bound to port 4500 (the NAT-T port).

REQ-8: `UDP_ENCAP_OVPNINUDP` (8) is a Linux-specific OpenVPN data-channel encap; introduced in mainline 6.16+ — present in 7.1.0-rc2 baseline.

REQ-9: `TCP_ENCAP_ESPINTCP` (7) is **not** a UDP encap-type; it appears in this header as a cross-listed xfrm constant. The kernel uses it from xfrm netlink, **not** via `UDP_ENCAP` setsockopt. Userspace passing it to `UDP_ENCAP` receives `-EINVAL`.

REQ-10: `setsockopt(UDP_NO_CHECK6_TX, 1)` requires the socket to be IPv6 (`AF_INET6`) and not connected to a peer that requires checksums (UDP-Lite over IPv6 ignores the flag).

REQ-11: `setsockopt(UDP_SEGMENT, mss)` sets a default GSO segmentation size for subsequent `sendmsg(2)` calls; range 0..65507. Per-send override via `cmsg{SOL_UDP, UDP_SEGMENT}` with payload `u16`. The NIC or stack splits a large `sendmsg` into multiple datagrams of `mss` bytes (last one ≤ `mss`).

REQ-12: `setsockopt(UDP_GRO, 1)` enables UDP-GRO on receive: the stack coalesces consecutive datagrams with the same flow-key into a single skb; userspace receives a single `recvmsg(2)` carrying the original gso-size as `cmsg{SOL_UDP, UDP_GRO}`.

REQ-13: All `UDP_*` setsockopt values are **stable ABI**; never renumbered. Values 10–11 are reserved for UDP-Lite (declared in `<linux/udplite.h>`); 2..99 otherwise reserved.

REQ-14: `udphdr` over the wire is **never** truncated below 8 bytes; the IP layer drops short datagrams before they reach UDP.

REQ-15: `getsockopt(UDP_CORK)` returns 0/1 (current cork state). `getsockopt(UDP_ENCAP)` returns the currently configured encap type as `int`.

REQ-16: UDP-Lite (`IPPROTO_UDPLITE`, value 136) shares wire format with UDP but reinterprets `udphdr.len` as the **checksum coverage**; this `<linux/udp.h>` does not declare UDP-Lite options (see `<linux/udplite.h>`).

REQ-17: Source port 0 in `udphdr.source` is allowed on **transmit** (means "no reply expected") but the kernel will replace it with an ephemeral port when the socket is bound via `connect(2)` to enable two-way flow.

REQ-18: `UDP_SEGMENT` interacts with `MSG_ZEROCOPY` only on supported NICs; otherwise GSO is software-emulated by the stack.

REQ-19: `UDP_GRO` enabled flow may receive a single datagram whose total length exceeds the network MTU (it represents N coalesced packets); userspace must use the `UDP_GRO` cmsg to recover the original segment size.

REQ-20: Setting `UDP_ENCAP` on a socket already in use as a regular UDP socket has no effect on already-queued data, only on subsequent receives.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `udphdr_size_locked` | INVARIANT | `mem::size_of::<UdpHdr>() == 8`. |
| `udphdr_field_offsets` | INVARIANT | `offset_of!(UdpHdr, len) == 4 && offset_of!(UdpHdr, check) == 6`. |
| `udp_sockopt_values_stable` | INVARIANT | `UDP_CORK==1, UDP_ENCAP==100..UDP_GRO==104`. |
| `udp_encap_in_range` | INVARIANT | per-set: `v ∈ {0,1,2,3,4,5,6,8}`; 7 rejected. |
| `udp_no_check6_tx_v6_only` | INVARIANT | per-set: refuses AF_INET. |
| `udp_segment_max_payload` | INVARIANT | per-set: `mss ≤ 65507`. |
| `udp_len_consistent` | INVARIANT | per-validate: `len ≥ 8 ∧ buf.len() ≥ len`. |
| `v6_zero_checksum_requires_opt` | INVARIANT | per-validate v6: `check==0 ∧ !no_check6_rx ⟹ drop`. |

### Layer 2: TLA+

`uapi/headers/udp.tla`:
- Models receive-validate pipeline + UDP_ENCAP handler dispatch.
- Properties:
  - `safety_encap_dispatch_correct` — per-encap-type: correct handler invoked.
  - `safety_v6_checksum_required` — per-v6-rx: refuse `check==0` unless opted in.
  - `safety_l2tp_gtp_require_admin` — per-set: privileged encaps gated.
  - `liveness_cork_drains` — per-`UDP_CORK=0` after data: datagram emitted.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `UdpHdr::validate` post: ret OK ⟹ buf.len() ≥ 8 ∧ len ≥ 8 ∧ checksum policy honored | `UdpHdr::validate` |
| `SetSockOpt::udp_encap` post: v ∈ stable set ∨ -EINVAL/-EPERM | `SetSockOpt::udp_encap` |
| `SetSockOpt::udp_no_check6_tx` post: family == AF_INET6 | `SetSockOpt::udp_no_check6_tx` |
| `SetSockOpt::udp_segment` post: `mss ≤ 65507` | `SetSockOpt::udp_segment` |

### Layer 4: Verus/Creusot functional

RFC 768 wire-format equivalence: `udphdr` byte layout matches RFC 768 §header; pseudo-header checksum matches RFC 768 / RFC 8200. Encap dispatch matches xfrm / l2tp / gtp / rxrpc / ovpn module interfaces.

### hardening — grsecurity/pax-style reinforcement

- **GRKERNSEC_NO_SIMULT_CONNECT** — per-UDP-`connect(2)`: refuse "simultaneous-connect" where two unprivileged sockets in different netns race to bind the same ephemeral port via SO_REUSEPORT; removes a niche covert channel.
- **GRKERNSEC_BLACKHOLE** — per-UDP: silently drop datagrams to closed ports rather than reply with `ICMP_DEST_UNREACH(PORT)`; halves port-scan signal and removes ICMP-amplification side-effect.
- **GRKERNSEC_RANDNET** — per-UDP: randomize ephemeral source ports within `IP_LOCAL_PORT_RANGE`; randomize `udphdr.source` for unconnected `sendto`; jitter on UDP-RTP / DNS source-port choice (Kaminsky-grade randomness).
- **PaX UDEREF on `recvmsg(UDP_GRO|UDP_SEGMENT) cmsg`** — per-recvmsg: copy cmsg control buffer under UDEREF; pre-bound check `msg_controllen`.
- **CAP_NET_RAW gating on raw-UDP spoofing** — per-`IPPROTO_UDP SOCK_RAW` + `IP_HDRINCL`: refuse from non-init user-ns; raw UDP injection can forge `udphdr.source` for DNS/amplification attacks.
- **GRKERNSEC_PROC for /proc/net** — per-`/proc/net/udp`, `/proc/net/udp6`, `/proc/net/udplite`: hide foreign-netns sockets and `(saddr,daddr,sport,dport)` tuples to deny remote-process fingerprinting.
- **RANDKSTACK at recv/send entry** — per-`udp_rcv` / `udp_sendmsg`: re-randomize kernel-stack offset before parsing untrusted udphdr; resists ROP via crafted GRO-coalesced large skbs.
- **TCP MD5 / AO key handling** — n/a directly to UDP, but: `UDP_ENCAP_ESPINUDP` carries ESP packets whose key material (loaded via xfrm netlink) is locked in non-swappable kmem; constant-time HMAC compare in the ESP path.
- **UDP_ENCAP privileged types confined to init-ns** — per-set: `UDP_ENCAP_L2TPINUDP`, `UDP_ENCAP_GTP0`, `UDP_ENCAP_GTP1U`, `UDP_ENCAP_OVPNINUDP` require **init**-namespace `CAP_NET_ADMIN`, not a delegated user-ns capability.
- **UDP_NO_CHECK6_RX audit-logged** — per-set: log adoption of v6 zero-checksum acceptance; this turns a v6 socket into a checksum-bypass receiver and is a forensically interesting event.
- **UDP-Lite explicit allow-list** — per-`IPPROTO_UDPLITE` socket create: gate behind sysctl; deters fingerprinting and avoids partial-checksum bypass on intermediate appliances.
- **UDP_SEGMENT GSO cap tightened** — per-set: refuse `mss < TCP_MSS_DEFAULT` (536) outside debug builds; tiny-segment storms otherwise act as fragmentation amplifier.
- **UDP_GRO coalesce limit** — per-flow: cap coalesced datagram count and total bytes; prevents an attacker from synthesizing an oversized GRO skb to trigger OOM.

