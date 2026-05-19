---
title: "Tier-5: include/uapi/linux/in.h — IPv4 address family ABI"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`<linux/in.h>` is the IPv4 address-family ABI. It defines `struct sockaddr_in` (the BSD-style address used by `bind(2)` / `connect(2)` / `sendto(2)` / `recvfrom(2)` on AF_INET sockets), `struct in_addr` (32-bit network-order IPv4 address), the `IPPROTO_*` constants used as the third argument to `socket(2)` and as `iphdr.protocol` values, and the IP-level socket-option numbers consumed by `setsockopt(2)`/`getsockopt(2)` at level `SOL_IP` (= `IPPROTO_IP`). Per-`sockaddr_in` size is locked at 16 bytes (`__SOCK_SIZE__`). Per-`__be32 s_addr` is **network-order**: writing it requires `htonl()`. Per-`sin_port` is **network-order** `__be16`: `htons()`. Per-`sin_family` must equal `AF_INET`. Per-`__pad[8]` padding is exposed via the BSD alias `sin_zero` and must be zeroed by userspace per POSIX. Multicast group membership, packet-info ancillary data (`IP_PKTINFO` cmsg), TProxy redirection (`IP_TRANSPARENT`, `IP_ORIGDSTADDR`), Path-MTU-Discovery state machine (`IP_MTU_DISCOVER` + `IP_PMTUDISC_*`), and CAP_NET_RAW-gated raw-header injection (`IP_HDRINCL`) are all reached through this header.

This Tier-5 covers `include/uapi/linux/in.h` (~337 lines).

### Acceptance Criteria

- [ ] AC-1: `sizeof(struct sockaddr_in) == 16` on every supported arch.
- [ ] AC-2: `sizeof(struct in_addr) == 4`.
- [ ] AC-3: `IPPROTO_*` numeric values bit-exact vs. upstream header.
- [ ] AC-4: `IPPROTO_MAX` is a sentinel only; not used in stable ABI.
- [ ] AC-5: `IP_*` socket-option numbers 1..52 bit-exact.
- [ ] AC-6: `IP_PMTUDISC_*` values 0..5 bit-exact.
- [ ] AC-7: `IP_MULTICAST_*` and `MCAST_*` values 32..48 bit-exact.
- [ ] AC-8: `INADDR_*` constants bit-exact.
- [ ] AC-9: `socket(AF_INET, SOCK_RAW, IPPROTO_RAW)` without CAP_NET_RAW → `-EPERM`.
- [ ] AC-10: `setsockopt(IP_TRANSPARENT)` without CAP_NET_ADMIN → `-EPERM`.
- [ ] AC-11: `setsockopt(IP_HDRINCL, 1)` on SOCK_RAW without CAP_NET_RAW → `-EPERM`.
- [ ] AC-12: `setsockopt(IP_ADD_MEMBERSHIP)` with non-multicast group → `-EINVAL`.
- [ ] AC-13: `setsockopt(IP_MTU_DISCOVER, 6)` → `-EINVAL` (out of range).
- [ ] AC-14: `bind(sa.sin_family=AF_INET6, …)` on AF_INET socket → `-EAFNOSUPPORT`.
- [ ] AC-15: `sin_zero[8]` left non-zero by userspace is silently ignored (current behaviour).

### Architecture

```
#[repr(C)]
pub struct InAddr { pub s_addr: BE32 }      // 4 B network order

#[repr(C)]
pub struct SockaddrIn {
    pub sin_family: u16,                    // AF_INET = 2
    pub sin_port:   BE16,                   // network order
    pub sin_addr:   InAddr,
    pub _pad:       [u8; 8],                // sin_zero alias
}                                           // total 16 B = SOCK_ADDR_SIZE

#[repr(C)]
pub struct InPktinfo {
    pub ipi_ifindex:  i32,                  // host order
    pub ipi_spec_dst: InAddr,
    pub ipi_addr:     InAddr,
}                                           // 12 B
```

`SockaddrIn::validate(buf: &[u8]) -> Result<&Self, Errno>`:
1. if buf.len() < 16: return -EINVAL.
2. let sa = read_aligned::<SockaddrIn>(buf).
3. if sa.sin_family != AF_INET: return -EAFNOSUPPORT.
4. return Ok(sa).

`IpProto::from_u32(v) -> Option<Self>`:
1. match v against the IPPROTO_* set (0,1,2,4,6,8,12,17,22,29,33,41,46,47,50,51,92,94,98,103,108,115,132,136,137,143,144,255,256,262).
2. else return None.

`SetSockOpt::ip_mtu_discover(sk, v) -> Result<(), Errno>`:
1. match v:
   - 0..=2: sk.pmtud = v.
   - 3 (PROBE) | 4 (INTERFACE) | 5 (OMIT): if !ns_capable(CAP_NET_ADMIN) ∧ !ns_capable(CAP_NET_RAW): return -EPERM.
2. set sk.pmtud.

`SetSockOpt::ip_add_membership(sk, mreq) -> Result<(), Errno>`:
1. if !IN_MULTICAST(ntohl(mreq.imr_multiaddr.s_addr)): return -EINVAL.
2. let dev = resolve_iface(mreq).
3. if dev.is_none(): return -ENODEV.
4. igmp_join_group(sk, dev, mreq.imr_multiaddr).

`SetSockOpt::ip_hdrincl(sk, on) -> Result<(), Errno>`:
1. if sk.type != SOCK_RAW: return -ENOPROTOOPT.
2. if on ∧ !ns_capable(CAP_NET_RAW): return -EPERM.
3. sk.hdrincl = on.

### Out of Scope

- `<linux/in6.h>` IPv6 ABI (covered in `in6.md` Tier-5 when added)
- `<linux/sockios.h>` ioctl numbers (covered separately)
- Routing-table netlink (`rtnetlink`, `RTM_*`) ABI (covered separately)
- Implementation of IGMP / multicast state machine (Tier-3 `net/ipv4/igmp.md`)
- TCP/UDP per-protocol behaviour (covered in `tcp.md`, `udp.md`)

### abi surface

### Address-family structures (size-locked)

| Symbol | Type | Layout | Endianness | Notes |
|---|---|---|---|---|
| `struct in_addr` | 4 B | `__be32 s_addr` | network | sole field |
| `struct sockaddr_in` | 16 B | `sin_family` (`u16`) + `sin_port` (`__be16`) + `sin_addr` (4 B) + `__pad[8]` | mixed | `__SOCK_SIZE__ == 16`; `sin_zero` is an alias for `__pad` |
| `struct in_pktinfo` | 12 B | `ipi_ifindex` (`int`) + `ipi_spec_dst` (`in_addr`) + `ipi_addr` (`in_addr`) | host/network | IP_PKTINFO cmsg payload |
| `struct ip_mreq` | 8 B | 2 × `in_addr` | network | IP_ADD_MEMBERSHIP legacy |
| `struct ip_mreqn` | 12 B | 2 × `in_addr` + `int imr_ifindex` | network/host | IP_ADD_MEMBERSHIP modern |
| `struct ip_mreq_source` | 12 B | 3 × `__be32` | network | IP_BLOCK_SOURCE / IP_ADD_SOURCE_MEMBERSHIP |
| `struct ip_msfilter` | var | 3 × `__be32` + 2 × `__u32` + flex `__be32[]` | network/host | IP_MSFILTER set |
| `struct group_req` | var | `gr_interface` (`u32`) + `__kernel_sockaddr_storage` | host/family | MCAST_JOIN_GROUP |
| `struct group_source_req` | var | `gsr_interface` + 2 × storage | host/family | MCAST_JOIN_SOURCE_GROUP |
| `struct group_filter` | var | iface + group + fmode + numsrc + flex storage[] | host/family | MCAST_MSFILTER (union of `_aux` legacy vs flex) |

### IPPROTO_* (third arg of socket(2); iphdr.protocol)

| Symbol | Value | Description |
|---|---|---|
| `IPPROTO_IP` | 0 | Dummy / SOL_IP for setsockopt level |
| `IPPROTO_ICMP` | 1 | ICMPv4 |
| `IPPROTO_IGMP` | 2 | Internet Group Management Protocol |
| `IPPROTO_IPIP` | 4 | IPIP tunnels |
| `IPPROTO_TCP` | 6 | TCP |
| `IPPROTO_EGP` | 8 | Exterior Gateway Protocol |
| `IPPROTO_PUP` | 12 | PARC Universal Packet |
| `IPPROTO_UDP` | 17 | UDP |
| `IPPROTO_IDP` | 22 | XNS IDP |
| `IPPROTO_TP` | 29 | ISO TP4 |
| `IPPROTO_DCCP` | 33 | Datagram Congestion Control Protocol |
| `IPPROTO_IPV6` | 41 | IPv6-in-IPv4 |
| `IPPROTO_RSVP` | 46 | RSVP |
| `IPPROTO_GRE` | 47 | Cisco GRE (RFC 1701/1702) |
| `IPPROTO_ESP` | 50 | IPsec ESP |
| `IPPROTO_AH` | 51 | IPsec Authentication Header |
| `IPPROTO_MTP` | 92 | Multicast Transport Protocol |
| `IPPROTO_BEETPH` | 94 | BEET pseudo header (IPsec) |
| `IPPROTO_ENCAP` | 98 | Generic encapsulation |
| `IPPROTO_PIM` | 103 | Protocol Independent Multicast |
| `IPPROTO_COMP` | 108 | IPComp |
| `IPPROTO_L2TP` | 115 | L2TP |
| `IPPROTO_SCTP` | 132 | SCTP |
| `IPPROTO_UDPLITE` | 136 | UDP-Lite (RFC 3828) |
| `IPPROTO_MPLS` | 137 | MPLS-in-IP (RFC 4023) |
| `IPPROTO_ETHERNET` | 143 | Ethernet-in-IPv6 |
| `IPPROTO_AGGFRAG` | 144 | AGGFRAG in ESP (RFC 9347) |
| `IPPROTO_RAW` | 255 | Raw IP packets (CAP_NET_RAW) |
| `IPPROTO_SMC` | 256 | Shared Memory Communications |
| `IPPROTO_MPTCP` | 262 | Multipath TCP |
| `IPPROTO_MAX` | sentinel | bound for protocol-table arrays |

### IP_* socket options (level SOL_IP)

| Symbol | Value | Semantic |
|---|---|---|
| `IP_TOS` | 1 | Type-of-service byte (gets/sets `iphdr.tos`) |
| `IP_TTL` | 2 | Per-socket TTL override |
| `IP_HDRINCL` | 3 | Caller supplies IPv4 header (raw sockets; CAP_NET_RAW) |
| `IP_OPTIONS` | 4 | IPv4 options on transmit |
| `IP_ROUTER_ALERT` | 5 | Receive packets w/ Router-Alert option (raw) |
| `IP_RECVOPTS` | 6 | Receive all IP options as cmsg |
| `IP_RETOPTS` | 7 | Receive raw options stripped of timestamp/record-route |
| `IP_PKTINFO` | 8 | Receive `in_pktinfo` cmsg |
| `IP_PKTOPTIONS` | 9 | Get/set TCP-pinned option ancillary data |
| `IP_MTU_DISCOVER` | 10 | Set PMTUD policy (see IP_PMTUDISC_*) |
| `IP_RECVERR` | 11 | Enable extended error queue (MSG_ERRQUEUE) |
| `IP_RECVTTL` | 12 | Receive TTL as cmsg |
| `IP_RECVTOS` | 13 | Receive TOS as cmsg |
| `IP_MTU` | 14 | Get current PMTU (getsockopt only) |
| `IP_FREEBIND` | 15 | Bind to non-local address (CAP_NET_ADMIN) |
| `IP_IPSEC_POLICY` | 16 | Per-socket IPsec policy |
| `IP_XFRM_POLICY` | 17 | Per-socket XFRM policy (CAP_NET_ADMIN) |
| `IP_PASSSEC` | 18 | Pass SELinux peer security context as cmsg |
| `IP_TRANSPARENT` | 19 | Allow bind to foreign addresses (CAP_NET_ADMIN; TProxy) |
| `IP_ORIGDSTADDR` | 20 | Receive original dst addr (TProxy) as cmsg |
| `IP_RECVORIGDSTADDR` | == IP_ORIGDSTADDR | alias |
| `IP_MINTTL` | 21 | Drop packets with TTL below this |
| `IP_NODEFRAG` | 22 | Disable defragmentation on raw socket (CAP_NET_RAW) |
| `IP_CHECKSUM` | 23 | Inject checksum offset/value |
| `IP_BIND_ADDRESS_NO_PORT` | 24 | Defer ephemeral port allocation until connect |
| `IP_RECVFRAGSIZE` | 25 | Receive fragment size as cmsg |
| `IP_RECVERR_RFC4884` | 26 | RFC4884-aware error queue |
| `IP_RECVRETOPTS` | == IP_RETOPTS | BSD alias |

### IP_PMTUDISC_* (IP_MTU_DISCOVER values)

| Symbol | Value | Behaviour |
|---|---|---|
| `IP_PMTUDISC_DONT` | 0 | Never set DF |
| `IP_PMTUDISC_WANT` | 1 | Use per-route hint |
| `IP_PMTUDISC_DO` | 2 | Always set DF |
| `IP_PMTUDISC_PROBE` | 3 | Probe — set DF, ignore PMTU |
| `IP_PMTUDISC_INTERFACE` | 4 | Use interface MTU; ignore spoofable ICMP frag-needed |
| `IP_PMTUDISC_OMIT` | 5 | Like INTERFACE but allow fragmentation |

### IP_*_MULTICAST_* and group ops

| Symbol | Value | Meaning |
|---|---|---|
| `IP_MULTICAST_IF` | 32 | Pick outgoing iface for multicast |
| `IP_MULTICAST_TTL` | 33 | Multicast TTL |
| `IP_MULTICAST_LOOP` | 34 | Loopback control |
| `IP_ADD_MEMBERSHIP` | 35 | Join group (legacy IPv4) |
| `IP_DROP_MEMBERSHIP` | 36 | Leave group |
| `IP_UNBLOCK_SOURCE` | 37 | Source-specific multicast unblock |
| `IP_BLOCK_SOURCE` | 38 | Source-specific multicast block |
| `IP_ADD_SOURCE_MEMBERSHIP` | 39 | Join (G,S) |
| `IP_DROP_SOURCE_MEMBERSHIP` | 40 | Leave (G,S) |
| `IP_MSFILTER` | 41 | Set source filter |
| `MCAST_JOIN_GROUP` | 42 | RFC 3678 join (AF-agnostic) |
| `MCAST_BLOCK_SOURCE` | 43 | RFC 3678 block |
| `MCAST_UNBLOCK_SOURCE` | 44 | RFC 3678 unblock |
| `MCAST_LEAVE_GROUP` | 45 | RFC 3678 leave |
| `MCAST_JOIN_SOURCE_GROUP` | 46 | RFC 3678 SSM join |
| `MCAST_LEAVE_SOURCE_GROUP` | 47 | RFC 3678 SSM leave |
| `MCAST_MSFILTER` | 48 | RFC 3678 set filter |
| `IP_MULTICAST_ALL` | 49 | Receive all groups joined on iface |
| `IP_UNICAST_IF` | 50 | Pick outgoing iface for unicast |
| `IP_LOCAL_PORT_RANGE` | 51 | Per-socket ephemeral range |
| `IP_PROTOCOL` | 52 | Per-socket protocol override |
| `MCAST_EXCLUDE` | 0 | filter mode |
| `MCAST_INCLUDE` | 1 | filter mode |
| `IP_DEFAULT_MULTICAST_TTL` | 1 | default |
| `IP_DEFAULT_MULTICAST_LOOP` | 1 | default |

### In-network address constants

| Symbol | Value | Meaning |
|---|---|---|
| `INADDR_ANY` | 0x00000000 | Wildcard bind |
| `INADDR_BROADCAST` | 0xFFFFFFFF | All-1s broadcast |
| `INADDR_NONE` | 0xFFFFFFFF | Error return from `inet_addr(3)` |
| `INADDR_DUMMY` | 0xC0000008 | 192.0.0.8 (RFC 7600) |
| `INADDR_LOOPBACK` | 0x7F000001 | 127.0.0.1 |
| `IN_LOOPBACKNET` | 127 | 127/8 |
| `INADDR_UNSPEC_GROUP` | 0xE0000000 | 224.0.0.0 |
| `INADDR_ALLHOSTS_GROUP` | 0xE0000001 | 224.0.0.1 |
| `INADDR_ALLRTRS_GROUP` | 0xE0000002 | 224.0.0.2 |
| `INADDR_ALLSNOOPERS_GROUP` | 0xE000006A | 224.0.0.106 |
| `INADDR_MAX_LOCAL_GROUP` | 0xE00000FF | 224.0.0.255 |

### Class-network legacy macros

`IN_CLASSA(a)`, `IN_CLASSB(a)`, `IN_CLASSC(a)`, `IN_CLASSD(a)`, `IN_CLASSE(a)`, `IN_MULTICAST(a)` (alias of `IN_CLASSD`), `IN_BADCLASS(a)`, `IN_EXPERIMENTAL(a)`, `IN_LOOPBACK(a)`, plus per-class `_NET`, `_NSHIFT`, `_HOST`, `_MAX` masks. Userspace-only; the kernel ignores classful boundaries internally.

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct sockaddr_in` | per-AF_INET addr | `SockaddrIn` |
| `struct in_addr` | 32-bit IPv4 | `InAddr` |
| `struct in_pktinfo` | per-IP_PKTINFO cmsg | `InPktinfo` |
| `struct ip_mreq` / `ip_mreqn` | per-IGMPv2 join | `IpMreq` / `IpMreqN` |
| `struct ip_mreq_source` | per-SSM join | `IpMreqSource` |
| `struct ip_msfilter` | per-source-filter | `IpMsfilter` |
| `struct group_req` / `_source_req` / `_filter` | per-RFC3678 ops | `GroupReq` / `GroupSourceReq` / `GroupFilter` |
| `IPPROTO_*` enum | per-protocol-id | `IpProto` |
| `IP_*` setsockopt names | per-IP-sockopt | `IpSockoptName` |
| `IP_PMTUDISC_*` | per-PMTUD policy | `PmtudPolicy` |
| `INADDR_*` constants | per-special-addr | shared |
| `__SOCK_SIZE__` | size lock | `SOCK_ADDR_SIZE` |

### compatibility contract

REQ-1: `struct sockaddr_in` layout is **frozen** at 16 bytes (`__SOCK_SIZE__`). Field order is `sin_family` (`__kernel_sa_family_t`, `u16`), `sin_port` (`__be16`, network byte order), `sin_addr` (`struct in_addr`, 4 B network byte order), `__pad[8]`. The kernel **reads** `sin_family` and accepts the struct only if it equals `AF_INET` (2).

REQ-2: `sin_zero` is a `#define` alias for `__pad`. Userspace conventionally zeroes it; the kernel currently ignores its contents but must not break programs that compare `sin_zero` byte-for-byte.

REQ-3: `struct in_addr` contains a single `__be32 s_addr` field. The value is stored in **network byte order**: userspace must call `htonl()` to convert host-order constants (e.g. `htonl(INADDR_LOOPBACK)`).

REQ-4: `IPPROTO_*` constants are stable wire-protocol numbers (IANA-assigned for 0..255; Linux-specific extensions above 256). The numeric values **must not** change across kernel versions.

REQ-5: `IPPROTO_MAX` is a sentinel and may shift when new protocols are added; userspace must not embed it as a fixed ABI value.

REQ-6: `IPPROTO_RAW` (255) requires `CAP_NET_RAW` to bind on a `SOCK_RAW` socket; otherwise `socket(2)` returns `-EPERM`.

REQ-7: `IP_HDRINCL` on `SOCK_RAW` sockets requires `CAP_NET_RAW`. With it set, the caller-supplied IP header is used verbatim (except for `iphdr.id` and `iphdr.check` which the kernel may fill).

REQ-8: `IP_OPTIONS` setsockopt: option list ≤ 40 bytes (`MAX_IPOPTLEN`); per-option validation rejects unknown classes.

REQ-9: `IP_MTU_DISCOVER` accepts only the 6 `IP_PMTUDISC_*` values 0..5. `IP_PMTUDISC_PROBE`, `_INTERFACE`, `_OMIT` require `CAP_NET_ADMIN` or `CAP_NET_RAW`.

REQ-10: `IP_TRANSPARENT` requires `CAP_NET_ADMIN`; enables TProxy bind-to-foreign-address.

REQ-11: `IP_FREEBIND` requires `CAP_NET_ADMIN`; permits binding to a non-local address.

REQ-12: `IP_RECVERR` enables a per-socket error queue read via `recvmsg(MSG_ERRQUEUE)`. `IP_RECVERR_RFC4884` extends with RFC 4884 length/version semantics.

REQ-13: `IP_PKTINFO` cmsg buffer carries `struct in_pktinfo` (12 B): `ipi_ifindex` (host order), `ipi_spec_dst`, `ipi_addr` (both network order).

REQ-14: `IP_ORIGDSTADDR` (alias `IP_RECVORIGDSTADDR`) is mandatory for TProxy reply path: cmsg returns the original (pre-redirect) `struct sockaddr_in`.

REQ-15: `IP_TOS` and `IP_TTL` accept a `u8` value (0..255). `IP_TTL` 0 maps to "use default"; negative values rejected.

REQ-16: `IP_MULTICAST_TTL` accepts `0..255`; default = 1 (`IP_DEFAULT_MULTICAST_TTL`). `IP_MULTICAST_LOOP` accepts 0/1; default = 1.

REQ-17: `IP_ADD_MEMBERSHIP` accepts `struct ip_mreq` (8 B) or `struct ip_mreqn` (12 B) selected by `optlen`. Group address must be in 224.0.0.0/4 (multicast range).

REQ-18: RFC 3678 ops (`MCAST_*`) take `struct group_req` / `group_source_req` / `group_filter` using `__kernel_sockaddr_storage` to be AF-agnostic.

REQ-19: `IP_BIND_ADDRESS_NO_PORT` defers source-port allocation to `connect(2)` and selects from `IP_LOCAL_PORT_RANGE` if set.

REQ-20: `IP_LOCAL_PORT_RANGE` value is a `u32` packing `lo` (low 16) and `hi` (high 16); `lo <= hi`; both within `1..65535` or both zero (clear override).

REQ-21: `IP_PROTOCOL` (52) overrides the per-socket protocol seen by `getsockopt(SO_PROTOCOL)`; useful for `IPPROTO_MPTCP` fallback diagnostics.

REQ-22: Address-class macros (`IN_CLASSA`, etc.) are **userspace-only**; the kernel routes by destination address + routing table, not classfulness.

REQ-23: `INADDR_LOOPBACK` (0x7F000001) and `IN_LOOPBACK(a)` (127/8) are reserved for the loopback device `lo`; bind on these addresses requires the socket to use the loopback iface.

REQ-24: `INADDR_ANY` (0) used as bind address means "wildcard, listen on all interfaces". `INADDR_BROADCAST` is destination-only.

REQ-25: All multi-byte fields in IPv4 wire structs (`sin_port`, `sin_addr.s_addr`, `imr_*addr`) are network byte order; `ipi_ifindex` is **host order**.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `sockaddr_in_size_locked` | INVARIANT | `mem::size_of::<SockaddrIn>() == 16`. |
| `in_addr_size_locked` | INVARIANT | `mem::size_of::<InAddr>() == 4`. |
| `sin_family_validated` | INVARIANT | per-validate: rejects non-AF_INET. |
| `ipproto_max_not_in_abi` | INVARIANT | per-IpProto: MAX is sentinel, not parsed. |
| `pmtudisc_in_range` | INVARIANT | per-set: 0..=5 only. |
| `multicast_group_in_classd` | INVARIANT | per-add-membership: address ∈ 224/4. |
| `hdrincl_requires_raw` | INVARIANT | per-hdrincl: rejects on non-SOCK_RAW. |
| `transparent_requires_admin` | INVARIANT | per-IP_TRANSPARENT: CAP_NET_ADMIN check. |

### Layer 2: TLA+

`uapi/headers/in.tla`:
- Models setsockopt state machine for IP-level options.
- Properties:
  - `safety_no_capabilities_bypass` — per-option-set: privileged options reject unprivileged callers.
  - `safety_multicast_group_class_d` — per-IP_ADD_MEMBERSHIP: rejects non-mcast.
  - `safety_pmtud_state_machine_closed` — per-PMTUD: only 6 valid states.
  - `liveness_membership_eventually_joined` — per-join: IGMP report sent.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `SockaddrIn::validate` post: ret OK ⟹ sin_family == AF_INET | `SockaddrIn::validate` |
| `IpProto::from_u32` post: ret Some ⟺ v ∈ stable set | `IpProto::from_u32` |
| `SetSockOpt::ip_mtu_discover` post: privileged values require cap | `SetSockOpt::ip_mtu_discover` |
| `SetSockOpt::ip_add_membership` post: group ∈ 224/4 ∨ -EINVAL | `SetSockOpt::ip_add_membership` |

### Layer 4: Verus/Creusot functional

ABI numeric equivalence: each `IPPROTO_*`, `IP_*`, `IP_PMTUDISC_*`, `MCAST_*`, `INADDR_*` constant compiles to the bit-exact upstream value; verified by const-asserts at crate root.

### hardening — grsecurity/pax-style reinforcement

- **GRKERNSEC_NO_SIMULT_CONNECT** — per-AF_INET: refuse simultaneous-open (RFC 793 § 3.4) which is widely unused and a covert-channel vector; honored for `IPPROTO_TCP` sockets bound via `sockaddr_in`.
- **GRKERNSEC_BLACKHOLE** — per-AF_INET: silently drop incoming TCP/UDP traffic to closed ports rather than RST/ICMP-unreach; halves port-scan signal.
- **GRKERNSEC_RANDNET** — per-IPID/per-INADDR_ANY connect: randomize `iphdr.id`, ephemeral port choice (within `IP_LOCAL_PORT_RANGE`), and TCP ISN to defeat off-path attackers.
- **PaX UDEREF on `copy_from_user(sockaddr_in)`** — per-bind/connect: forbid implicit kernel→user dereference; strict 16-byte length check before reading `sin_family`.
- **CAP_NET_RAW gating** — per-`IPPROTO_RAW`, `IP_HDRINCL`, `IP_NODEFRAG`, `IP_ROUTER_ALERT`: never allow user-namespace-only `CAP_NET_RAW` to forge headers visible on the host bridge.
- **GRKERNSEC_PROC for /proc/net** — per-inet_diag: hide foreign sockets in `/proc/net/tcp`, `/proc/net/udp`, `/proc/net/raw`, `/proc/net/igmp` from non-root namespaces.
- **RANDKSTACK at recv/send entry** — per-recvfrom/sendto on AF_INET: re-randomize kernel-stack offset before parsing `sockaddr_in` to defeat stack-layout disclosure.
- **TCP MD5 / AO key handling** — per-`IP_*` does not key TCP-AO directly, but `sockaddr_in` is the keying tuple for `TCP_MD5SIG` / `TCP_AO_ADD_KEY`; lock keys in non-swappable kmem with constant-time compares.
- **IP_TRANSPARENT confined to init-userns** — per-CAP_NET_ADMIN check: require the **init** user-namespace, not a delegated one, to prevent container-foreign-bind escape.
- **IP_FREEBIND audit-logged** — per-set: emit an audit record naming the foreign address; deters silent ARP-spoofing prep.
- **PMTUD_PROBE/_INTERFACE/_OMIT cap-gated** — per-set: privileged PMTUD modes can bypass ICMP-frag-needed, used as DoS amplifier; require CAP_NET_ADMIN.
- **IPPROTO_RAW DCCP/SCTP injection denied to user-ns** — per-create: even with `CAP_NET_RAW`, refuse raw injection of `IPPROTO_DCCP`, `IPPROTO_SCTP`, `IPPROTO_AGGFRAG` from a non-init user-ns.

