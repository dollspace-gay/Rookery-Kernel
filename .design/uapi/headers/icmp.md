# Tier-5: include/uapi/linux/icmp.h — ICMPv4 ABI

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - include/uapi/linux/icmp.h (~162 lines)
-->

## Summary

`<linux/icmp.h>` is the **wire-format ABI** for ICMPv4 (RFC 792 + RFC 4884 + RFC 8335). It defines `struct icmphdr` (the 8-byte ICMPv4 header with a 4-byte payload-purpose union), the `ICMP_*` type/code constants used in `icmphdr.type` and `icmphdr.code`, `struct icmp_filter` (per-socket type bitmask for `IPPROTO_ICMP` `SOCK_RAW` filtering via `setsockopt(SOL_RAW, ICMP_FILTER)`), and the RFC 4884 / RFC 8335 extension structures used for ICMP Extended Echo (PROBE). Per-`icmphdr.type` selects one of 16 well-known type codes; per-`icmphdr.code` further distinguishes (e.g. 16 codes for `ICMP_DEST_UNREACH`, 4 for `ICMP_REDIRECT`, 2 for `ICMP_TIME_EXCEEDED`). Per-`un` union carries either an Echo `(id, sequence)` pair, a Redirect `gateway` address, a Frag-Needed `(unused, mtu)` pair, or a generic 4-byte `reserved[4]`. Per-`ICMP_FILTER` lets a `SOCK_RAW IPPROTO_ICMP` socket subscribe to a subset of types via a 32-bit mask. Per-CAP_NET_RAW required for raw ICMP; otherwise `IPPROTO_ICMP` opened as `SOCK_DGRAM` requires no privilege but only delivers `ICMP_ECHO` Echo-replies (per `net.ipv4.ping_group_range`).

This Tier-5 covers `include/uapi/linux/icmp.h` (~162 lines).

## ABI surface

### Wire-format structures

| Symbol | Size | Field layout | Endianness |
|---|---|---|---|
| `struct icmphdr` | 8 B | `type` (1 B), `code` (1 B), `checksum` (`__sum16`), `un { echo {id (BE16), sequence (BE16)} \| gateway (BE32) \| frag {__unused (BE16), mtu (BE16)} \| reserved[4] }` | network |
| `struct icmp_filter` | 4 B | `data` (`u32`) | host |
| `struct icmp_ext_hdr` | 4 B | bitfield `version:4 + reserved1:4` (1 B), `reserved2` (1 B), `checksum` (`__sum16`) | network |
| `struct icmp_extobj_hdr` | 4 B | `length` (`__be16`), `class_num` (1 B), `class_type` (1 B) | network |
| `struct icmp_ext_echo_ctype3_hdr` | 4 B | `afi` (`__be16`), `addrlen` (1 B), `reserved` (1 B) | network |
| `struct icmp_ext_echo_iio` | var | `extobj_hdr` + `ident` union (name[IFNAMSIZ] \| ifindex (BE32) \| addr struct) | network |

### ICMP types (`icmphdr.type`)

| Symbol | Value | Direction | Meaning |
|---|---|---|---|
| `ICMP_ECHOREPLY` | 0 | rx | Echo Reply |
| `ICMP_DEST_UNREACH` | 3 | rx | Destination Unreachable |
| `ICMP_SOURCE_QUENCH` | 4 | rx | Source Quench (deprecated, RFC 6633) |
| `ICMP_REDIRECT` | 5 | rx | Redirect |
| `ICMP_ECHO` | 8 | tx | Echo Request (ping) |
| `ICMP_TIME_EXCEEDED` | 11 | rx | Time Exceeded |
| `ICMP_PARAMETERPROB` | 12 | rx | Parameter Problem |
| `ICMP_TIMESTAMP` | 13 | tx/rx | Timestamp Request |
| `ICMP_TIMESTAMPREPLY` | 14 | tx/rx | Timestamp Reply |
| `ICMP_INFO_REQUEST` | 15 | obsolete | Information Request (RFC 6918) |
| `ICMP_INFO_REPLY` | 16 | obsolete | Information Reply |
| `ICMP_ADDRESS` | 17 | obsolete | Address Mask Request (RFC 6918) |
| `ICMP_ADDRESSREPLY` | 18 | obsolete | Address Mask Reply |
| `NR_ICMP_TYPES` | 18 | — | sentinel (max defined type value) |
| `ICMP_EXT_ECHO` | 42 | tx | Extended Echo (PROBE; RFC 8335) |
| `ICMP_EXT_ECHOREPLY` | 43 | rx | Extended Echo Reply |

### Codes for `ICMP_DEST_UNREACH` (`icmphdr.code`)

| Symbol | Value | Meaning |
|---|---|---|
| `ICMP_NET_UNREACH` | 0 | Network Unreachable |
| `ICMP_HOST_UNREACH` | 1 | Host Unreachable |
| `ICMP_PROT_UNREACH` | 2 | Protocol Unreachable |
| `ICMP_PORT_UNREACH` | 3 | Port Unreachable |
| `ICMP_FRAG_NEEDED` | 4 | Fragmentation Needed + DF set (uses `un.frag.mtu`) |
| `ICMP_SR_FAILED` | 5 | Source Route failed |
| `ICMP_NET_UNKNOWN` | 6 | Net Unknown |
| `ICMP_HOST_UNKNOWN` | 7 | Host Unknown |
| `ICMP_HOST_ISOLATED` | 8 | Host Isolated |
| `ICMP_NET_ANO` | 9 | Net Administratively prohibited |
| `ICMP_HOST_ANO` | 10 | Host Administratively prohibited |
| `ICMP_NET_UNR_TOS` | 11 | Net Unreachable for TOS |
| `ICMP_HOST_UNR_TOS` | 12 | Host Unreachable for TOS |
| `ICMP_PKT_FILTERED` | 13 | Packet filtered |
| `ICMP_PREC_VIOLATION` | 14 | Precedence violation |
| `ICMP_PREC_CUTOFF` | 15 | Precedence cut off |
| `NR_ICMP_UNREACH` | 15 | sentinel |

### Codes for `ICMP_REDIRECT`

| Symbol | Value | Meaning |
|---|---|---|
| `ICMP_REDIR_NET` | 0 | Redirect for Network |
| `ICMP_REDIR_HOST` | 1 | Redirect for Host |
| `ICMP_REDIR_NETTOS` | 2 | Redirect for Network+TOS |
| `ICMP_REDIR_HOSTTOS` | 3 | Redirect for Host+TOS |

### Codes for `ICMP_TIME_EXCEEDED`

| Symbol | Value | Meaning |
|---|---|---|
| `ICMP_EXC_TTL` | 0 | TTL count exceeded in transit |
| `ICMP_EXC_FRAGTIME` | 1 | Fragment reassembly time exceeded |

### Codes for `ICMP_EXT_ECHO` / `ICMP_EXT_ECHOREPLY` (RFC 8335)

| Symbol | Value | Meaning |
|---|---|---|
| `ICMP_EXT_CODE_MAL_QUERY` | 1 | Malformed Query |
| `ICMP_EXT_CODE_NO_IF` | 2 | No such Interface |
| `ICMP_EXT_CODE_NO_TABLE_ENT` | 3 | No such Table Entry |
| `ICMP_EXT_CODE_MULT_IFS` | 4 | Multiple Interfaces |
| `ICMP_EXT_ECHOREPLY_ACTIVE` | 0x04 | "active" bit in reply |
| `ICMP_EXT_ECHOREPLY_IPV4` | 0x02 | "ipv4" bit in reply |
| `ICMP_EXT_ECHOREPLY_IPV6` | 0x01 | "ipv6" bit in reply |
| `ICMP_EXT_ECHO_CTYPE_NAME` | 1 | identifier = ifname |
| `ICMP_EXT_ECHO_CTYPE_INDEX` | 2 | identifier = ifindex |
| `ICMP_EXT_ECHO_CTYPE_ADDR` | 3 | identifier = address |
| `ICMP_AFI_IP` | 1 | AFI = IPv4 |
| `ICMP_AFI_IP6` | 2 | AFI = IPv6 |

### Socket option

| Symbol | Value | Level | Payload |
|---|---|---|---|
| `ICMP_FILTER` | 1 | `SOL_RAW` (IPPROTO_RAW) on `SOCK_RAW IPPROTO_ICMP` | `struct icmp_filter` (32-bit type bitmask) |

`icmp_filter.data`: bit `n` set means "drop type `n`" (e.g. `data = ~0u & ~(1u << ICMP_ECHOREPLY)` accepts only Echo replies).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct icmphdr` | per-ICMP wire header | `IcmpHdr` |
| `struct icmp_filter` | per-raw-socket filter | `IcmpFilter` |
| `struct icmp_ext_hdr` | per-RFC4884 extension | `IcmpExtHdr` |
| `struct icmp_extobj_hdr` | per-object header | `IcmpExtobjHdr` |
| `struct icmp_ext_echo_ctype3_hdr` | per-c-type-3 ident | `IcmpExtEchoCtype3Hdr` |
| `struct icmp_ext_echo_iio` | per-Interface-Identification Object | `IcmpExtEchoIio` |
| `ICMP_*` type constants | per-type-id | `IcmpType` |
| Unreach / Redirect / Exceeded codes | per-code-id | `IcmpCode` |
| `ICMP_FILTER` sockopt | per-filter-name | `IcmpSockopt` |
| `NR_ICMP_TYPES`/`NR_ICMP_UNREACH` | per-sentinel | shared |

## Compatibility contract

REQ-1: `struct icmphdr` is **exactly 8 bytes**. Field order: `type` (1 B), `code` (1 B), `checksum` (`__sum16` network order), then a 4-byte tagged union `un { echo {id (__be16), sequence (__be16)} | gateway (__be32) | frag {__unused (__be16), mtu (__be16)} | reserved[4] }`.

REQ-2: `checksum` is the one's-complement sum over the entire ICMP message (header + body) with the checksum field set to zero during computation.

REQ-3: The interpretation of `un` depends on `type`/`code`:
- `ICMP_ECHO` / `ICMP_ECHOREPLY` / `ICMP_TIMESTAMP` / `ICMP_TIMESTAMPREPLY`: `echo { id, sequence }`.
- `ICMP_REDIRECT`: `gateway` (advertised next-hop).
- `ICMP_DEST_UNREACH` with `code == ICMP_FRAG_NEEDED`: `frag.mtu` (next-hop MTU).
- All other codes/types: `reserved[4]` (sender sets 0; receiver ignores).

REQ-4: All multi-byte fields inside `un` are **network byte order**.

REQ-5: `NR_ICMP_TYPES = 18` bounds the legacy type table. Extended types (`ICMP_EXT_ECHO = 42`, `ICMP_EXT_ECHOREPLY = 43`) are outside that range and handled by a separate dispatch.

REQ-6: `NR_ICMP_UNREACH = 15` bounds the Dest-Unreach code table.

REQ-7: `ICMP_SOURCE_QUENCH` (type 4) is deprecated by RFC 6633; the kernel **must not generate** it and **must silently ignore** incoming Source Quench.

REQ-8: `ICMP_INFO_REQUEST`/`ICMP_INFO_REPLY`/`ICMP_ADDRESS`/`ICMP_ADDRESSREPLY` are obsolete (RFC 6918); the kernel does not respond.

REQ-9: `ICMP_REDIRECT` is accepted only when `IPV4_DEVCONF_ACCEPT_REDIRECTS == 1`; with `IPV4_DEVCONF_SECURE_REDIRECTS == 1`, only redirects from the current default gateway are honoured.

REQ-10: `ICMP_FRAG_NEEDED` (Dest-Unreach code 4) updates the PMTU cache only if the receiving socket has not opted out via `IP_PMTUDISC_INTERFACE` / `_PROBE`.

REQ-11: `struct icmp_filter` is **4 bytes** (`__u32 data`); set via `setsockopt(sk, SOL_RAW, ICMP_FILTER, &filter, 4)`; bit N set ⟹ type N filtered out. `SOL_RAW == 255` (= `IPPROTO_RAW`).

REQ-12: A `SOCK_RAW`/`IPPROTO_ICMP` socket requires `CAP_NET_RAW` (in the appropriate user namespace) to create.

REQ-13: A `SOCK_DGRAM`/`IPPROTO_ICMP` socket ("Linux ping socket") requires the caller's GID ∈ `net.ipv4.ping_group_range`; it only sends `ICMP_ECHO` and receives only the matching `ICMP_ECHOREPLY`.

REQ-14: `struct icmp_ext_hdr` carries `version=2` per RFC 4884; total length must include 8-byte header padding ahead of the extensions.

REQ-15: `struct icmp_extobj_hdr.length` includes the 4-byte object header itself (RFC 4884 § 7).

REQ-16: `struct icmp_ext_echo_iio` (RFC 8335 Interface-Identification Object): `class_num = 3`, `class_type ∈ {1,2,3}` selects union arm (`name` / `ifindex` / `addr`). When `class_type == 3`, `ip_addr` length is governed by `ctype3_hdr.addrlen` and family by `ctype3_hdr.afi` (`ICMP_AFI_IP` = 1 ⟹ `ipv4_addr`; `ICMP_AFI_IP6` = 2 ⟹ `ipv6_addr`).

REQ-17: ICMP Echo on a SOCK_DGRAM (ping socket): `id` field is **overwritten** by the kernel with the socket's source port so that replies can be demuxed; userspace cannot set arbitrary `id`.

REQ-18: ICMP error generation is rate-limited (`net.ipv4.icmp_ratelimit`, `icmp_ratemask`); userspace cannot bypass via raw ICMP since outgoing raw ICMP is treated as an injection and counted toward the global token bucket only for kernel-originated errors.

REQ-19: All `ICMP_*` numeric values are **frozen** ABI; new types are appended (e.g. 42/43 for EXT_ECHO).

REQ-20: `icmp_filter.data` is in **host byte order**; not a wire field.

## Acceptance Criteria

- [ ] AC-1: `sizeof(struct icmphdr) == 8`.
- [ ] AC-2: `offsetof(struct icmphdr, un) == 4`.
- [ ] AC-3: `sizeof(struct icmp_filter) == 4`.
- [ ] AC-4: ICMP type constants bit-exact (0,3,4,5,8,11,12,13,14,15,16,17,18; 42, 43).
- [ ] AC-5: `NR_ICMP_TYPES == 18`, `NR_ICMP_UNREACH == 15`.
- [ ] AC-6: All `ICMP_*_UNREACH` codes 0..15 bit-exact.
- [ ] AC-7: `ICMP_EXC_TTL == 0`, `ICMP_EXC_FRAGTIME == 1`.
- [ ] AC-8: `ICMP_REDIR_NET == 0` .. `ICMP_REDIR_HOSTTOS == 3`.
- [ ] AC-9: `socket(AF_INET, SOCK_RAW, IPPROTO_ICMP)` without CAP_NET_RAW → `-EPERM`.
- [ ] AC-10: `socket(AF_INET, SOCK_DGRAM, IPPROTO_ICMP)` outside `ping_group_range` → `-EACCES`.
- [ ] AC-11: `setsockopt(SOL_RAW, ICMP_FILTER, …, 4)` accepted; non-4 optlen → `-EINVAL`.
- [ ] AC-12: Receive of `ICMP_SOURCE_QUENCH` silently dropped, no socket delivery.
- [ ] AC-13: Receive of `ICMP_REDIRECT` with `ACCEPT_REDIRECTS=0` ignored.
- [ ] AC-14: Receive of `ICMP_FRAG_NEEDED` with `mtu < 68` rejected (RFC 1191 floor).

## Architecture

```
#[repr(C)]
pub struct IcmpHdr {
    pub r#type:   u8,
    pub code:     u8,
    pub checksum: Sum16,
    pub un:       IcmpUn,            // 4 B union
}                                    // == 8 B

#[repr(C)]
pub union IcmpUn {
    pub echo:     IcmpEcho,          // { id: BE16, sequence: BE16 }
    pub gateway:  BE32,
    pub frag:     IcmpFrag,          // { __unused: BE16, mtu: BE16 }
    pub reserved: [u8; 4],
}

#[repr(C)] pub struct IcmpFilter { pub data: u32 }     // host order
```

`IcmpHdr::validate(buf: &[u8]) -> Result<&Self, IcmpErr>`:
1. if buf.len() < 8: return Err(Truncated).
2. let h = read_as::<IcmpHdr>(buf).
3. if !checksum_ok(buf): return Err(BadChecksum).
4. return Ok(h).

`IcmpHdr::body_for_type(&self) -> IcmpBody`:
1. match self.r#type:
   - ICMP_ECHO | ECHOREPLY | TIMESTAMP | TIMESTAMPREPLY: Body::Echo(self.un.echo).
   - ICMP_REDIRECT: Body::Gateway(self.un.gateway).
   - ICMP_DEST_UNREACH if self.code == ICMP_FRAG_NEEDED: Body::Frag(self.un.frag).
   - _: Body::Reserved(self.un.reserved).

`IcmpFilter::is_filtered(&self, t: u8) -> bool`:
1. if t >= 32: return false.
2. return (self.data & (1u32 << t)) != 0.

`SetSockOpt::icmp_filter(sk, optval, optlen) -> Result<(), Errno>`:
1. if sk.type != SOCK_RAW ∨ sk.protocol != IPPROTO_ICMP: return -ENOPROTOOPT.
2. if optlen != 4: return -EINVAL.
3. sk.icmp_filter = read_u32(optval).
4. return Ok.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `icmphdr_size_locked` | INVARIANT | `mem::size_of::<IcmpHdr>() == 8`. |
| `icmphdr_union_size` | INVARIANT | `mem::size_of::<IcmpUn>() == 4`. |
| `icmp_filter_size` | INVARIANT | `mem::size_of::<IcmpFilter>() == 4`. |
| `nr_icmp_types_bounded` | INVARIANT | `NR_ICMP_TYPES == 18`. |
| `unreach_codes_in_range` | INVARIANT | per-code: 0..=15. |
| `icmp_filter_optlen_is_4` | INVARIANT | per-setsockopt: rejects optlen != 4. |
| `cap_net_raw_required` | INVARIANT | per-create: SOCK_RAW IPPROTO_ICMP needs CAP_NET_RAW. |
| `source_quench_dropped` | INVARIANT | per-rx: type 4 silently dropped. |

### Layer 2: TLA+

`uapi/headers/icmp.tla`:
- Models receive-path dispatch + filter application.
- Properties:
  - `safety_unreach_code_in_table` — per-receive: `NR_ICMP_UNREACH` bound respected.
  - `safety_redirect_only_from_gw` — per-`ICMP_REDIRECT`: with SECURE_REDIRECTS=1, source must be default gw.
  - `safety_filter_bitmask_consistent` — per-`ICMP_FILTER`: filter drops match bit pattern.
  - `liveness_echo_reply_delivered` — per-`ICMP_ECHO`: reply emitted within rate-limit window.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `IcmpHdr::validate` post: ret OK ⟹ buf.len() ≥ 8 ∧ checksum valid | `IcmpHdr::validate` |
| `IcmpHdr::body_for_type` post: union arm selected per type/code | `IcmpHdr::body_for_type` |
| `IcmpFilter::is_filtered` post: matches bit `1u32 << t` | `IcmpFilter::is_filtered` |
| `SetSockOpt::icmp_filter` post: optlen == 4 ∨ -EINVAL | `SetSockOpt::icmp_filter` |

### Layer 4: Verus/Creusot functional

RFC 792 wire-format equivalence: per-type/code pair, struct layout matches RFC 792 § 3.1..3.4 and RFC 4884/8335 layouts. All `ICMP_*` constants proven bit-exact via const-asserts.

## Hardening — Grsecurity/PaX-style Reinforcement

- **GRKERNSEC_NO_SIMULT_CONNECT** — per-ICMP: drop ICMP errors that reference TCP flows in mid-simultaneous-open (rare) to avoid covert resets.
- **GRKERNSEC_BLACKHOLE** — per-ICMP-Echo to closed-port host: optionally suppress emission of `ICMP_DEST_UNREACH(PORT)` for unreachable UDP ports, halving port-scan signal.
- **GRKERNSEC_RANDNET** — per-ICMP Echo: randomize the ICMP `id` chosen by SOCK_DGRAM ping sockets (already source-port-keyed); randomize ICMPv4 Timestamp `originate` to avoid host fingerprinting.
- **PaX UDEREF on ICMP_FILTER setsockopt** — per-`ICMP_FILTER`: copy_from_user the 4-byte filter under UDEREF; reject any non-4 optlen pre-copy to avoid OOB read.
- **CAP_NET_RAW gating** — per-`SOCK_RAW IPPROTO_ICMP`: refuse from non-init user-ns even with delegated CAP_NET_RAW; raw ICMP can spoof PMTU updates, RAs, and redirects.
- **GRKERNSEC_PROC for /proc/net** — per-`/proc/net/icmp`: hide foreign-netns ICMP sockets and per-type counters to avoid leak of scan/probe activity.
- **RANDKSTACK at icmp_rcv entry** — per-`icmp_rcv`: re-randomize kernel-stack offset before parsing untrusted ICMP headers; resists ROP via crafted ICMP options.
- **TCP MD5 / AO key handling** — per-ICMP-PMTUD: `ICMP_FRAG_NEEDED` for a TCP-AO flow must be cross-checked against the AO key tuple before mutating PMTU; refuse PMTU updates from non-authenticated ICMP for AO-protected flows (`accept_icmps` in `tcp_ao_info_opt`).
- **ACCEPT_REDIRECTS forced off in init-ns** — per-init: `IPV4_DEVCONF_ACCEPT_REDIRECTS = 0`; ICMP_REDIRECT cannot reroute traffic.
- **ICMP error rate-limit tightened** — per-`net.ipv4.icmp_ratelimit`: lower default token-bucket; avoid being used as an amplifier.
- **ICMP_FRAG_NEEDED MTU floor 68/1280** — per-rx: refuse PMTU below 68 (IPv4) / 1280 (IPv6) — defeats classic "tiny MTU" amplification.
- **ping_group_range default-empty** — per-init: SOCK_DGRAM IPPROTO_ICMP requires explicit GID grant; prevents unprivileged ping reconnaissance.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `<linux/icmpv6.h>` ICMPv6 ABI (covered in `icmpv6.md` Tier-5 when added)
- ICMP error rate-limit sysctls (covered under `net/ipv4/sysctls.md`)
- ICMP socket implementation (`net/ipv4/icmp.c`, Tier-3)
- Ping socket implementation (`net/ipv4/ping.c`, Tier-3)
- Linux ICMP-Extended-Echo PROBE userland (RFC 8335 client logic)
