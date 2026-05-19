# Tier-5: include/uapi/linux/ip.h — IPv4 header ABI

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - include/uapi/linux/ip.h (~197 lines)
-->

## Summary

`<linux/ip.h>` is the **wire-format ABI** for IPv4. It defines `struct iphdr` (the 20-byte IPv4 datagram header, plus optional options), `struct ip_auth_hdr` (IPsec AH, RFC 4302), `struct ip_esp_hdr` (IPsec ESP, RFC 4303), `struct ip_comp_hdr` (IPComp, RFC 3173), the legacy `IPTOS_*` Type-of-Service bits and precedence values, and the `IPOPT_*` IP-option numbering scheme. Per-`iphdr` is read by every receive path (IP fragmentation, conntrack, netfilter hook, routing lookup) and written by every transmit path (UDP/TCP send, raw socket with `IP_HDRINCL=0`); per-field endianness is **network byte order** for all multi-byte members. Per-`ihl` × 4 = header length in bytes (min 20, max 60). Per-`frag_off` packs three flag bits (`IP_DF`, `IP_MF`, reserved) into the top of a 16-bit field whose remaining 13 bits are the fragment offset in 8-byte units, accessed via the `IP_OFFSET` mask. Per-`__struct_group(addrs)` lets the kernel hash/copy `(saddr, daddr)` as a single 8-byte unit while keeping the C ABI intact.

This Tier-5 covers `include/uapi/linux/ip.h` (~197 lines).

## ABI surface

### Wire-format structures

| Symbol | Size | Field layout | Endianness |
|---|---|---|---|
| `struct iphdr` | 20 B + options | bitfield `version:4 + ihl:4` (1 B; bitfield order arch-dependent), `tos` (1 B), `tot_len` (`__be16`), `id` (`__be16`), `frag_off` (`__be16`), `ttl` (1 B), `protocol` (1 B), `check` (`__sum16`), `addrs { saddr (__be32); daddr (__be32); }` | network |
| `struct ip_auth_hdr` | 12 B + auth_data | `nexthdr` (1 B), `hdrlen` (1 B, 32-bit words), `reserved` (`__be16`), `spi` (`__be32`), `seq_no` (`__be32`), `auth_data[]` (≥ 4 B, 64-bit aligned) | network |
| `struct ip_esp_hdr` | 8 B + enc_data | `spi` (`__be32`), `seq_no` (`__be32`), `enc_data[]` (≥ 8 B, 64-bit aligned) | network |
| `struct ip_comp_hdr` | 4 B | `nexthdr` (1 B), `flags` (1 B), `cpi` (`__be16`) | network |
| `struct ip_beet_phdr` | 4 B | `nexthdr` (1 B), `hdrlen` (1 B), `padlen` (1 B), `reserved` (1 B) | host |
| `struct ip_iptfs_hdr` | 4 B | `subtype` (1 B), `flags` (1 B), `block_offset` (`__be16`) | network |
| `struct ip_iptfs_cc_hdr` | 24 B | `subtype` + `flags` + `block_offset` + `loss_rate` (`__be32`) + `rtt_adelay_xdelay` (`__be64`) + `tval` (`__be32`) + `techo` (`__be32`) | network |

### IPv4 protocol constants

| Symbol | Value | Meaning |
|---|---|---|
| `IPVERSION` | 4 | `iphdr.version` value |
| `MAXTTL` | 255 | upper bound on `iphdr.ttl` |
| `IPDEFTTL` | 64 | default TTL for outgoing packets |
| `MAX_IPOPTLEN` | 40 | maximum IPv4 options length (60-20) |
| `IPV4_BEET_PHMAXLEN` | 8 | max BEET pseudo-header length |

### Type-of-Service (`iphdr.tos`)

#### TOS field (lower 5 bits used; bits 1..4 select profile, bit 0 reserved)

| Symbol | Value | Meaning |
|---|---|---|
| `IPTOS_TOS_MASK` | 0x1E | TOS mask (bits 1..4) |
| `IPTOS_TOS(t)` | `(t) & 0x1E` | extract TOS |
| `IPTOS_LOWDELAY` | 0x10 | minimize delay |
| `IPTOS_THROUGHPUT` | 0x08 | maximize throughput |
| `IPTOS_RELIABILITY` | 0x04 | maximize reliability |
| `IPTOS_MINCOST` | 0x02 | minimize monetary cost |

#### Precedence (upper 3 bits, top of `tos`)

| Symbol | Value | Meaning |
|---|---|---|
| `IPTOS_PREC_MASK` | 0xE0 | precedence mask |
| `IPTOS_PREC(t)` | `(t) & 0xE0` | extract precedence |
| `IPTOS_PREC_NETCONTROL` | 0xE0 | 7 |
| `IPTOS_PREC_INTERNETCONTROL` | 0xC0 | 6 |
| `IPTOS_PREC_CRITIC_ECP` | 0xA0 | 5 |
| `IPTOS_PREC_FLASHOVERRIDE` | 0x80 | 4 |
| `IPTOS_PREC_FLASH` | 0x60 | 3 |
| `IPTOS_PREC_IMMEDIATE` | 0x40 | 2 |
| `IPTOS_PREC_PRIORITY` | 0x20 | 1 |
| `IPTOS_PREC_ROUTINE` | 0x00 | 0 (default) |

### IP options (per-option octet layout)

The first option octet contains: copy flag (bit 7), class (bits 5..6), number (bits 0..4).

| Symbol | Value | Meaning |
|---|---|---|
| `IPOPT_COPY` | 0x80 | copy-into-fragments flag |
| `IPOPT_CLASS_MASK` | 0x60 | class mask |
| `IPOPT_NUMBER_MASK` | 0x1F | number mask |
| `IPOPT_CONTROL` | 0x00 | class 0: control |
| `IPOPT_RESERVED1` | 0x20 | class 1 |
| `IPOPT_MEASUREMENT` | 0x40 | class 2: debug/measurement |
| `IPOPT_RESERVED2` | 0x60 | class 3 |

| Symbol | Encoded value | Number | Description |
|---|---|---|---|
| `IPOPT_END` | 0x00 | 0 | End of option list |
| `IPOPT_NOOP` | 0x01 | 1 | No operation |
| `IPOPT_SEC` | 0x82 | 2 | Security (DoD) |
| `IPOPT_LSRR` | 0x83 | 3 | Loose Source-and-Record Route |
| `IPOPT_TIMESTAMP` | 0x44 | 4 | Timestamp |
| `IPOPT_CIPSO` | 0x86 | 6 | Commercial IP Security Option |
| `IPOPT_RR` | 0x07 | 7 | Record Route |
| `IPOPT_SID` | 0x88 | 8 | Stream ID |
| `IPOPT_SSRR` | 0x89 | 9 | Strict Source-and-Record Route |
| `IPOPT_RA` | 0x94 | 20 | Router Alert |

Aliases: `IPOPT_NOP = IPOPT_NOOP`, `IPOPT_EOL = IPOPT_END`, `IPOPT_TS = IPOPT_TIMESTAMP`.

Per-option layout offsets: `IPOPT_OPTVAL` = 0, `IPOPT_OLEN` = 1, `IPOPT_OFFSET` = 2, `IPOPT_MINOFF` = 4.

Timestamp flags (low nibble of TS option `flag` byte):
| Symbol | Value | Meaning |
|---|---|---|
| `IPOPT_TS_TSONLY` | 0 | timestamps only |
| `IPOPT_TS_TSANDADDR` | 1 | timestamps + addrs |
| `IPOPT_TS_PRESPEC` | 3 | prespec'd modules only |

### `frag_off` bit-fields (16-bit network-order word)

Bit layout (host-view after `ntohs`): bit 15 = reserved (must be 0), bit 14 = DF, bit 13 = MF, bits 0..12 = offset (in 8-byte units).

| Symbol | Value | Mask role |
|---|---|---|
| `IP_CE` | 0x8000 | flag — congestion (reserved/evil) |
| `IP_DF` | 0x4000 | flag — Don't Fragment |
| `IP_MF` | 0x2000 | flag — More Fragments |
| `IP_OFFSET` | 0x1FFF | mask — fragment offset bits |

(Note: `IP_CE`, `IP_DF`, `IP_MF`, `IP_OFFSET` are defined in `<linux/ip.h>` indirectly via netfilter / kernel `<net/ip.h>`; the uapi header relies on the masks documented in RFC 791. Documented here for completeness of `frag_off` interpretation.)

### IPv4 devconf indices (sysctl backing for `net.ipv4.conf.<dev>.*`)

`enum { IPV4_DEVCONF_FORWARDING=1, MC_FORWARDING, PROXY_ARP, ACCEPT_REDIRECTS, SECURE_REDIRECTS, SEND_REDIRECTS, SHARED_MEDIA, RP_FILTER, ACCEPT_SOURCE_ROUTE, BOOTP_RELAY, LOG_MARTIANS, TAG, ARPFILTER, MEDIUM_ID, NOXFRM, NOPOLICY, FORCE_IGMP_VERSION, ARP_ANNOUNCE, ARP_IGNORE, PROMOTE_SECONDARIES, ARP_ACCEPT, ARP_NOTIFY, ACCEPT_LOCAL, SRC_VMARK, PROXY_ARP_PVLAN, ROUTE_LOCALNET, IGMPV2_UNSOLICITED_REPORT_INTERVAL, IGMPV3_UNSOLICITED_REPORT_INTERVAL, IGNORE_ROUTES_WITH_LINKDOWN, DROP_UNICAST_IN_L2_MULTICAST, DROP_GRATUITOUS_ARP, BC_FORWARDING, ARP_EVICT_NOCARRIER, __IPV4_DEVCONF_MAX }`. `IPV4_DEVCONF_MAX = __IPV4_DEVCONF_MAX - 1`.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct iphdr` | per-IPv4 wire header | `IpHdr` |
| `struct ip_auth_hdr` | per-AH | `IpAuthHdr` |
| `struct ip_esp_hdr` | per-ESP | `IpEspHdr` |
| `struct ip_comp_hdr` | per-IPComp | `IpCompHdr` |
| `struct ip_beet_phdr` | per-BEET | `IpBeetPhdr` |
| `struct ip_iptfs_hdr` | per-IPTFS | `IpIptfsHdr` |
| `struct ip_iptfs_cc_hdr` | per-IPTFS-CC | `IpIptfsCcHdr` |
| `IPVERSION`/`MAXTTL`/`IPDEFTTL` | per-defaults | shared |
| `IPOPT_*` numbers | per-option-id | `IpOpt` |
| `IPTOS_*` bits | per-TOS-byte | `IpTos` |
| `IPV4_DEVCONF_*` | per-sysctl index | `Ipv4Devconf` |

## Compatibility contract

REQ-1: `struct iphdr` is **exactly 20 bytes** before options (verified via `BUILD_BUG_ON(sizeof(iphdr) != 20)` in kernel). Field order is bit-exact across all architectures; only the `version`/`ihl` bitfield **order within the first byte** is endianness-conditional and chosen so the first byte byte-equals the wire octet on both LE and BE arches.

REQ-2: On wire, the first byte of an IPv4 header is `(version << 4) | ihl`. The bitfield split must produce the same byte on both `__LITTLE_ENDIAN_BITFIELD` (ihl:4, version:4) and `__BIG_ENDIAN_BITFIELD` (version:4, ihl:4).

REQ-3: `ihl` is in **32-bit words**; the header occupies `ihl * 4` bytes. Valid range 5..15 (20..60 B). `ihl < 5` is invalid; the kernel drops the packet.

REQ-4: `tot_len` is in **bytes**, network byte order, including the header and payload. Min `ihl*4`, max `65535`.

REQ-5: `frag_off` is a single network-order `__be16` packing: bit 15 reserved (must be 0), bit 14 = DF, bit 13 = MF, bits 0..12 = fragment offset in **8-byte units**. Masks `IP_DF = 0x4000`, `IP_MF = 0x2000`, `IP_OFFSET = 0x1FFF`.

REQ-6: `ttl` is decremented by 1 at every router. The kernel sets outgoing `ttl = IPDEFTTL = 64` unless overridden by `IP_TTL` or `IP_MULTICAST_TTL`. Receiving `ttl == 0` ⟹ drop.

REQ-7: `protocol` is one of the `IPPROTO_*` values from `<linux/in.h>`; the receive path dispatches via `inet_protos[protocol]`.

REQ-8: `check` is the **one's-complement** sum of the header (with `check` set to 0). Recomputed on every TTL decrement; offloadable to NIC.

REQ-9: `saddr` and `daddr` form a `__struct_group(addrs, ...)` so the kernel may treat the 8-byte address pair as a single field for hashing / copying without breaking C ABI (the field names `saddr` / `daddr` remain individually accessible).

REQ-10: `struct ip_auth_hdr.hdrlen` is in **32-bit words minus 2** (per RFC 4302 § 2.2); total AH length = `(hdrlen + 2) * 4` bytes. `auth_data[]` is at least 4 bytes and must be 64-bit-aligned.

REQ-11: `struct ip_esp_hdr` has 8 bytes of fixed header (`spi`, `seq_no`); `enc_data[]` is ≥ 8 bytes and 64-bit-aligned.

REQ-12: `struct ip_comp_hdr.cpi` (Compression Parameter Index) is a `__be16` allocated by IANA (well-known 0..63, negotiated 256..61439).

REQ-13: `IPOPT_*` encoded values include the copy bit (`0x80`) and class bits in the top 3 bits and the number in the bottom 5; the kernel decodes via `IPOPT_COPIED`, `IPOPT_CLASS`, `IPOPT_NUMBER` macros.

REQ-14: `IPOPT_RA` (Router Alert) MUST be honoured by intermediate routers handling the packet locally (RFC 2113); used by IGMP / RSVP.

REQ-15: `IPOPT_SSRR` / `IPOPT_LSRR` source-route options are off by default (`IPV4_DEVCONF_ACCEPT_SOURCE_ROUTE` = 0) — accepting them re-enables a classical DDoS reflector.

REQ-16: `MAX_IPOPTLEN` (40 B) caps the option area; per-option `len` byte ≤ remaining space; out-of-range options ⟹ packet dropped.

REQ-17: `IPTOS_TOS_MASK = 0x1E` covers the legacy 4-bit TOS field; per RFC 2474 the byte is now interpreted as DSCP (bits 7..2) + ECN (bits 1..0). Userspace setting via `IP_TOS` writes the raw byte.

REQ-18: `IPV4_DEVCONF_*` indices are stable; appending new ones must keep prior values fixed. `__IPV4_DEVCONF_MAX` is a sentinel.

REQ-19: `iphdr` reads from user buffers (raw socket with `IP_HDRINCL`) must validate `ihl >= 5`, `version == 4`, `tot_len >= ihl*4` before any pointer arithmetic.

REQ-20: All multi-byte fields (`tot_len`, `id`, `frag_off`, `check`, `saddr`, `daddr`, `spi`, `seq_no`, `cpi`) are **network byte order**.

## Acceptance Criteria

- [ ] AC-1: `sizeof(struct iphdr) == 20`.
- [ ] AC-2: `offsetof(struct iphdr, saddr) == 12 && offsetof(struct iphdr, daddr) == 16`.
- [ ] AC-3: First-byte equation `byte = (version << 4) | ihl` holds on both LE and BE.
- [ ] AC-4: `sizeof(struct ip_comp_hdr) == 4`.
- [ ] AC-5: `sizeof(struct ip_esp_hdr) == 8`.
- [ ] AC-6: `sizeof(struct ip_auth_hdr) == 12` (excluding flex `auth_data`).
- [ ] AC-7: `IPVERSION == 4`, `MAXTTL == 255`, `IPDEFTTL == 64`.
- [ ] AC-8: All `IPOPT_*` encoded values bit-exact (e.g. `IPOPT_RA == 0x94`).
- [ ] AC-9: All `IPTOS_*` bits bit-exact (`IPTOS_LOWDELAY == 0x10` etc.).
- [ ] AC-10: `IP_DF == 0x4000`, `IP_MF == 0x2000`, `IP_OFFSET == 0x1FFF`.
- [ ] AC-11: Receive of `iphdr.ihl < 5` ⟹ silent drop + counter inc.
- [ ] AC-12: Receive of `iphdr.version != 4` ⟹ silent drop.
- [ ] AC-13: `IPV4_DEVCONF_MAX == __IPV4_DEVCONF_MAX - 1`.

## Architecture

```
#[repr(C)]
pub struct IpHdr {
    pub ver_ihl:    u8,           // (version << 4) | ihl
    pub tos:        u8,
    pub tot_len:    BE16,
    pub id:         BE16,
    pub frag_off:   BE16,
    pub ttl:        u8,
    pub protocol:   u8,
    pub check:      Sum16,
    pub saddr:      BE32,
    pub daddr:      BE32,
}   // == 20 B; static_assert(sizeof::<IpHdr>() == 20)

impl IpHdr {
    pub fn version(&self) -> u8 { self.ver_ihl >> 4 }
    pub fn ihl(&self)     -> u8 { self.ver_ihl & 0x0F }
    pub fn header_len(&self) -> usize { (self.ihl() as usize) * 4 }

    pub fn df(&self) -> bool { (self.frag_off.to_host() & 0x4000) != 0 }
    pub fn mf(&self) -> bool { (self.frag_off.to_host() & 0x2000) != 0 }
    pub fn offset_bytes(&self) -> u16 { (self.frag_off.to_host() & 0x1FFF) * 8 }
}
```

`IpHdr::validate_received(buf: &[u8]) -> Result<&Self, IpErr>`:
1. if buf.len() < 20: return Err(Truncated).
2. let h = read_as::<IpHdr>(buf).
3. if h.version() != 4: return Err(BadVersion).
4. if h.ihl() < 5: return Err(BadIhl).
5. let hlen = h.header_len();
6. if buf.len() < hlen: return Err(Truncated).
7. let tot = h.tot_len.to_host() as usize.
8. if tot < hlen ∨ buf.len() < tot: return Err(BadLength).
9. if !checksum_ok(buf[..hlen]): return Err(BadChecksum).
10. return Ok(h).

`IpHdr::encode_first_byte(version: u8, ihl: u8) -> u8`:
1. assert!(version <= 0x0F ∧ ihl <= 0x0F).
2. return (version << 4) | ihl.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `iphdr_size_locked` | INVARIANT | `mem::size_of::<IpHdr>() == 20`. |
| `iphdr_addrs_offsets` | INVARIANT | per-iphdr: `offset_of!(IpHdr, saddr) == 12`. |
| `ihl_min_5` | INVARIANT | per-validate: rejects `ihl < 5`. |
| `version_is_4` | INVARIANT | per-validate: rejects `version != 4`. |
| `frag_off_masks_disjoint` | INVARIANT | `IP_DF & IP_MF == 0`, `IP_OFFSET & (IP_DF|IP_MF) == 0`. |
| `ipopt_bits_consistent` | INVARIANT | `IPOPT_RA == (20 | IPOPT_CONTROL | IPOPT_COPY)`. |
| `ah_hdrlen_words_minus_2` | INVARIANT | per-AH: total = (hdrlen+2)*4. |

### Layer 2: TLA+

`uapi/headers/ip.tla`:
- Models IPv4 receive validation pipeline.
- Properties:
  - `safety_no_oob_read_after_validate` — per-validate: subsequent reads stay within `tot_len`.
  - `safety_checksum_recomputed_on_ttl_dec` — per-forward: checksum updated.
  - `liveness_validated_packet_dispatched` — per-validate-ok: handed to `inet_protos[protocol]`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `IpHdr::validate_received` post: ret OK ⟹ `ihl >= 5 ∧ version == 4 ∧ tot_len >= ihl*4` | `IpHdr::validate_received` |
| `IpHdr::encode_first_byte` post: high nibble = version, low nibble = ihl | `IpHdr::encode_first_byte` |
| `IpOpt::decode` post: returns one of the stable 10 options or Unknown | `IpOpt::decode` |
| `IpTos::extract_dscp` post: low 2 bits cleared | `IpTos::extract_dscp` |

### Layer 4: Verus/Creusot functional

`iphdr` layout matches RFC 791 byte-for-byte; constant `IPVERSION = 4`, `MAXTTL = 255`, `IPDEFTTL = 64`, `MAX_IPOPTLEN = 40` proven via const-asserts.

## Hardening — Grsecurity/PaX-style Reinforcement

- **GRKERNSEC_NO_SIMULT_CONNECT** — per-iphdr: detect and drop forged "simultaneous-open" SYN pairs whose `saddr`/`daddr` cross-match an outgoing half-open before delivery.
- **GRKERNSEC_BLACKHOLE** — per-iphdr receive: drop packets to closed ports without emitting an ICMP-port-unreach or TCP-RST so port-scan attackers see only a single behaviour.
- **GRKERNSEC_RANDNET** — per-iphdr emit: randomize `iphdr.id` per-flow rather than per-socket-counter; defeats Salzberg-style off-path IPID inference.
- **PaX UDEREF on raw-socket sendmsg** — per-`IP_HDRINCL`: validate the caller-supplied `iphdr` from user memory under UDEREF, then re-validate after copy (no TOCTOU on `ihl`).
- **CAP_NET_RAW gating on header forging** — per-`IP_HDRINCL`, per-`IPOPT_LSRR`/`SSRR`: refuse forged source-route or arbitrary `protocol` from non-init user-ns.
- **GRKERNSEC_PROC for /proc/net** — per-`/proc/net/igmp`, `/proc/net/route`, `/proc/net/dev`: hide foreign-netns counters that leak L3 topology (`saddr`/`daddr` pairs).
- **RANDKSTACK at iphdr parse** — per-`ip_rcv` entry: re-randomize kernel-stack offset before parsing untrusted `iphdr` to defeat stack-layout disclosure via crafted options.
- **TCP MD5 / AO key handling** — per-iphdr+`tcphdr` pair: lock TCP-AO MAC keys keyed on `(saddr,daddr,sport,dport)` in non-swappable kmem; constant-time HMAC compare.
- **IPV4_DEVCONF_ACCEPT_SOURCE_ROUTE forced off** — per-init: reject `IPOPT_LSRR`/`IPOPT_SSRR` on receive regardless of sysctl, to neutralize reflection attacks.
- **IPV4_DEVCONF_SECURE_REDIRECTS forced on** — per-init: only accept ICMP-Redirects from the current default gateway.
- **iphdr-options bound + audit-logged** — per-receive: any options frame logs `(saddr, daddr, option-list)` to audit; deters reconnaissance.
- **IPOPT_RA path confined** — per-Router-Alert: only IGMP/RSVP handlers may consume; otherwise drop to prevent option-driven dispatch escalation.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `<linux/ipv6.h>` (covered in `ipv6.md` Tier-5 when added)
- Netfilter hooks (`<linux/netfilter_ipv4.h>`) — covered separately
- IPsec key management (xfrm netlink) — covered separately
- IGMP wire format (`<linux/igmp.h>`) — covered separately
- TCP/UDP/ICMP wire formats (covered in `tcp.md`, `udp.md`, `icmp.md`)
