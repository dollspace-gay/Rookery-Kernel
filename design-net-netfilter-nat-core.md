---
title: "Tier-3: net/netfilter/nat-core — NAT mapping (SNAT / DNAT / MASQUERADE / REDIRECT)"
tags: ["design-doc", "tier-3", "net", "netfilter"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for the netfilter NAT subsystem: per-`nf_conn` NAT mapping installation (`nf_nat_setup_info`), per-direction translation (`nf_nat_packet`), the four NAT modes (Source NAT / Destination NAT / Masquerade / Redirect), per-protocol NAT (per-l4-proto port-mangling per RFC 3022 / RFC 7857), per-helper NAT (FTP / IRC / SIP / PPTP / TFTP / SANE / Amanda — protocol-aware payload-rewriting), and the NAT bidirectional-mapping correctness guarantee (forward path's NAT exactly inverted on return path).

Owns the mandatory `models/net/nat_translation.tla` TLA+ model declared by `net/netfilter/00-overview.md` (REQ-O4): bidirectional-mapping invariant + concurrent-flows-don't-share-state + reverse-de-NAT correctness.

Sub-tier-3 of `net/netfilter/00-overview.md`. Pairs with `net/netfilter/conntrack-core.md` (NAT lives as conntrack extension), `net/netfilter/nat-helpers.md` (per-helper modules), `net/sched/act-ct.md` (tc-side NAT alternative path).

### Requirements

- REQ-1: 4 NAT modes (SNAT / DNAT / MASQUERADE / REDIRECT) implemented per upstream semantics.
- REQ-2: `struct nf_nat_range2` UAPI byte-identical layout.
- REQ-3: `nf_nat_setup_info` allocation algorithm: alloc-protect (pick from pool, verify no conntrack-table conflict, install) per RFC 3022 + 7857.
- REQ-4: `nf_nat_packet` per-direction translation: mangle headers + update L4 checksum incrementally.
- REQ-5: Per-l4proto port mangling: TCP/UDP/SCTP/ICMP per `nf_nat_proto.c` vtable; identical algorithm.
- REQ-6: Per-helper NAT (FTP / IRC / SIP / PPTP / TFTP / SANE / Amanda / etc.) — protocol-aware payload rewriting; per-helper module identical signatures.
- REQ-7: MASQUERADE: src IP from outbound iface; iface-event listener tears down conntracks on iface-down.
- REQ-8: REDIRECT: dst IP rewritten to local; identical contract.
- REQ-9: Bidirectional-mapping invariant: forward NAT and reverse de-NAT are precise inverses; concurrent flows never share NAT state.
- REQ-10: TLA+ model `models/net/nat_translation.tla` (mandatory per `net/netfilter/00-overview.md` Layer 2) — proves: bidirectional-mapping invariant; concurrent-flows non-interference; reverse-de-NAT correctness.
- REQ-11: Per-skb checksum update incremental (RFC 1624) — no full-packet checksum recompute.
- REQ-12: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: `pahole struct nf_nat_range2` byte-identical layout. (covers REQ-2)
- [ ] AC-2: SNAT test: `nft add rule ... ip saddr 10.0.0.0/24 oifname "eth0" snat to 192.0.2.1`; outbound packet from 10.0.0.5 → tcpdump on eth0 shows src=192.0.2.1; reply path correctly de-NAT'd back to 10.0.0.5. (covers REQ-1, REQ-3, REQ-4, REQ-9)
- [ ] AC-3: DNAT test: `nft add rule ... iifname "eth0" tcp dport 80 dnat to 10.0.0.5:8080`; inbound packet to public-IP:80 → DNAT'd to 10.0.0.5:8080; reply path de-NAT'd. (covers REQ-1, REQ-3)
- [ ] AC-4: MASQUERADE test: `nft add rule ... oifname "eth0" masquerade`; outbound packet → src rewritten to eth0's primary IP; iface-IP-change tears down existing conntracks. (covers REQ-1, REQ-7)
- [ ] AC-5: REDIRECT test: `iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 8080`; inbound port-80 packets DNAT'd to local box port 8080. (covers REQ-1, REQ-8)
- [ ] AC-6: Per-l4proto test: TCP NAT preserves seq + computes checksum incrementally; UDP NAT preserves zero-csum-allowed semantics; SCTP NAT recomputes CRC32c fully. (covers REQ-5, REQ-11)
- [ ] AC-7: FTP NAT helper test: load `nf_nat_ftp`; FTP PORT command "1,2,3,4,17,135" → rewritten to NAT'd address+port; subsequent DCC connection works. (covers REQ-6)
- [ ] AC-8: Concurrent-flows test: 2 separate flows install distinct NAT mappings; mappings don't share state; reverse path of flow A doesn't de-NAT to flow B's source. (covers REQ-9)
- [ ] AC-9: Alloc-conflict test: NAT pool with single available port; first conntrack gets it; second conntrack with conflicting tuple → alloc fails (or returns alternative within range). (covers REQ-3)
- [ ] AC-10: TLA+ `models/net/nat_translation.tla` proves: bidirectional-mapping invariant + non-interference + reverse-de-NAT correctness. (covers REQ-10)
- [ ] AC-11: Hardening section present and follows template. (covers REQ-12)

### Architecture

### Rust module organization

- `kernel::net::netfilter::nat::Nat` — top-level NAT entrypoint
- `kernel::net::netfilter::nat::ManipType` — `NF_NAT_MANIP_SRC | DST` enum
- `kernel::net::netfilter::nat::Range` — `struct nf_nat_range2` + flag set
- `kernel::net::netfilter::nat::Setup` — `nf_nat_setup_info` allocation
- `kernel::net::netfilter::nat::Packet` — `nf_nat_packet` per-direction translation
- `kernel::net::netfilter::nat::proto::TcpUdpSctpIcmp` — per-l4proto vtable
- `kernel::net::netfilter::nat::Masquerade` — masquerade-mode + iface-event listener
- `kernel::net::netfilter::nat::Redirect` — redirect-mode
- `kernel::net::netfilter::nat::helpers::HelperRegistry` — per-helper NAT vtable
- `kernel::net::netfilter::nat::ChecksumIncremental` — RFC 1624 per-field checksum update

### Locking and concurrency

- **Per-`nf_conn` `lock`** (spinlock, inherited from conntrack-core): held during NAT setup
- **`nf_nat_lock`** (per-netns spinlock): per-NAT-pool allocation atomicity (prevents two flows from racing on same pool entry)
- **RCU**: per-helper registry is RCU-side
- **Conntrack hashtable per-bucket lock**: held during NAT'd-tuple insertion

### Error handling

- `Err(EAGAIN)` — NAT pool exhausted (caller may retry with different range)
- `Err(EINVAL)` — bad range / mismatched l3/l4
- `Err(EBUSY)` — race lost; retry
- `Err(ENOMEM)` — alloc fail

### Out of Scope

- Per-helper NAT detail (cross-ref `net/netfilter/nat-helpers.md`)
- Conntrack core (cross-ref `net/netfilter/conntrack-core.md`)
- 32-bit-only paths
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| NAT core: mapping installation, per-direction translation, alloc-protect-tuple | `net/netfilter/nf_nat_core.c` |
| Per-l4proto port-mangling (TCP/UDP/SCTP/ICMP) | `net/netfilter/nf_nat_proto.c` |
| Per-helper NAT framework | `net/netfilter/nf_nat_helper.c` |
| MASQUERADE-mode helpers (use outbound iface's primary IP) | `net/netfilter/nf_nat_masquerade.c` |
| REDIRECT-mode helpers (rewrite dst to local) | `net/netfilter/nf_nat_redirect.c` |
| Public API | `include/net/netfilter/nf_nat.h` |

### compatibility contract

### Four NAT modes

| Mode | Constant | Effect |
|---|---|---|
| **SNAT** | `NF_NAT_MANIP_SRC` | Rewrite outbound `src.addr:src.port` to chosen NAT-pool entry; reverse-direction de-NAT inverts |
| **DNAT** | `NF_NAT_MANIP_DST` | Rewrite outbound `dst.addr:dst.port` to chosen target; reverse-direction de-NAT inverts |
| **MASQUERADE** | (not its own bit; uses SNAT with iface-derived src) | Like SNAT but `src.addr` derived dynamically from outbound iface's primary IPv4/IPv6 address; auto-handles iface-IP changes |
| **REDIRECT** | (uses DNAT with self-loopback target) | Like DNAT but rewrites `dst.addr` to local box's IP for the inbound iface |

### `struct nf_nat_range2` (UAPI for NAT pool spec)

```c
struct nf_nat_range2 {
    unsigned int flags;
    union nf_inet_addr min_addr;
    union nf_inet_addr max_addr;
    union nf_conntrack_man_proto min_proto;
    union nf_conntrack_man_proto max_proto;
    union nf_conntrack_man_proto base_proto;     /* for full-cone NAT */
};
```

`flags` ∈ `{NF_NAT_RANGE_MAP_IPS, _PROTO_SPECIFIED, _PROTO_RANDOM, _PERSISTENT, _PROTO_RANDOM_FULLY, _PROTO_OFFSET, _NETMAP, _PROTO_RANDOM_ALL}`. Identical bit semantics.

### NAT installation: `nf_nat_setup_info`

```c
unsigned int nf_nat_setup_info(struct nf_conn *ct,
                                const struct nf_nat_range2 *range,
                                enum nf_nat_manip_type maniptype);
```

Algorithm:
1. Allocate from pool: pick a `(addr, port)` pair from `range` that, when applied to `ct->tuplehash[OPPOSITE_DIR]`, produces a tuple not currently in conntrack hashtable
2. Install: rewrite `ct->tuplehash[REPLY_DIR]` per the chosen mapping
3. Set `IPS_SRC_NAT` or `IPS_DST_NAT` flag in `ct->status`
4. Insert `ct->tuplehash[REPLY_DIR]` into hashtable so reverse-path packets find conntrack via NAT'd 5-tuple

Identical algorithm. Per-RFC 3022 + RFC 7857 (NAT behavior recommendations).

### `nf_nat_packet` (per-direction translation)

Called from per-AF NAT hook at `NF_INET_POST_ROUTING` (SNAT side) and `NF_INET_PRE_ROUTING` (DNAT side):
1. Extract skb's tuple
2. Lookup conntrack via tuple
3. If NAT'd direction → mangle skb headers per `ct->tuplehash[ORIG_DIR]` (forward) or `[REPLY_DIR]` (reverse)
4. Update L4 checksum (TCP/UDP/ICMP) for the mangled fields

Identical algorithm.

### Per-l4proto NAT specifics

| Proto | Mangling needed |
|---|---|
| TCP | src/dst port + checksum + sequence-number adjust (if connection-track-based modifications mid-flow) |
| UDP | src/dst port + checksum |
| SCTP | src/dst port + (no checksum-incremental — SCTP has CRC32c, fully recomputed) |
| ICMP | id field (echo) + embedded-error-quote-tuple if applicable |

Per `nf_nat_proto.c` per-l4proto vtable. Identical.

### Per-helper NAT (`nf_nat_helper.c`)

Conntrack helpers (FTP / IRC / SIP / etc.) parse application-layer payload; when payload contains addresses/ports of expected related connections, NAT helper rewrites those as well. Example: FTP PORT command contains "1,2,3,4,5,6" (IP+port of expected DCC); SNAT rewrites the ports/IP in the PORT command to the NAT'd values.

Per-helper `nf_nat_helper`:
```c
int nf_nat_amanda(struct sk_buff *, enum ip_conntrack_info, unsigned int, unsigned int, struct nf_conntrack_expect *);
int nf_nat_ftp(struct sk_buff *, ...);
int nf_nat_sip_*(...);
/* etc. */
```

Identical signatures.

### MASQUERADE specifics

`nf_nat_masquerade_*`:
- Source IP picked dynamically from outbound iface's `IP_AF_INET` primary address at install-time
- On iface-down or IP-change, kernel walks all MASQUERADE conntracks and tears them down (hot-unplug correctness)
- Per-netns iface-event listener registered

### REDIRECT specifics

`nf_nat_redirect_*`:
- Destination IP rewritten to inbound iface's primary IP (or 127.0.0.1 for non-routing setups)
- Used by `tc filter ... action mirred ingress redirect` interplay (cross-ref `act-mirred.md`); also classic `iptables -t nat -A PREROUTING -j REDIRECT --to-port 8080` for transparent proxy

### Bidirectional-mapping invariant (KEY)

For any conntrack `ct` with NAT installed:
- Forward direction: `ct->tuplehash[ORIG_DIR].tuple = original_tuple`
- Reverse direction: `ct->tuplehash[REPLY_DIR].tuple = nat_inverted_tuple = invert(applied_NAT(original_tuple))`

When forward packet arrives matching `original_tuple`: NAT'd to `applied_NAT(original_tuple)`.
When reverse packet arrives matching `nat_inverted_tuple = invert(applied_NAT(original))`: de-NAT'd to `invert(original_tuple)` (the natural reverse of the original).

Ergo: bidirectional flow is consistent. Identical mapping invariant.

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| `nf_nat_setup_info` allocation under pool lock (no double-allocate of same tuple) | `kani::proofs::net::netfilter::nat::setup_safety` |
| Per-l4proto checksum incremental update arithmetic | `kani::proofs::net::netfilter::nat::checksum_safety` |
| Per-helper NAT payload rewriting (skb_pull bounds; L4 checksum recompute) | `kani::proofs::net::netfilter::nat::helper_safety` |
| MASQUERADE iface-event teardown (RCU-side walk; no use-after-free) | `kani::proofs::net::netfilter::nat::masq_safety` |
| Bidirectional-tuple installation (`nat_inverted_tuple = invert(applied_NAT(original))` precisely) | `kani::proofs::net::netfilter::nat::bidir_safety` |

### Layer 2: TLA+ models

- `models/net/nat_translation.tla` (mandatory per `net/netfilter/00-overview.md`) — proves bidirectional-mapping invariant + non-interference + reverse-de-NAT correctness. Owned here.

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-conntrack NAT mapping | when IPS_SRC_NAT or IPS_DST_NAT set, both tuplehash[ORIG_DIR] and tuplehash[REPLY_DIR] are in hashtable | `kani::proofs::net::netfilter::nat::tuple_invariants` |
| NAT range allocation | per-(netns, range) at most one conntrack per allocated (addr, port) pair | `kani::proofs::net::netfilter::nat::alloc_invariants` |
| Per-l4proto checksum | post-NAT checksum equals pre-NAT minus old-field plus new-field (RFC 1624 invariant) | `kani::proofs::net::netfilter::nat::checksum_invariants` |

### Layer 4: Functional correctness (declared in `net/netfilter/00-overview.md` Layer 4)

- **NAT bidirectional-mapping theorem** via Verus — proves: ∀ NAT'd flow, `(orig.src, orig.dst, orig.sport, orig.dport)` maps to consistent `(nat.src, nat.dst, nat.sport, nat.dport)` and reverse direction inverts correctly. Owned here.
- **NAT non-interference theorem** via Verus — proves: ∀ pair of distinct flows, NAT pool allocations don't overlap (no two flows share `(nat.src, nat.sport)` for SNAT or `(nat.dst, nat.dport)` for DNAT).

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-helper module refcount uses `Refcount` (saturating) | § Mandatory |
| **SIZE_OVERFLOW** | NAT range arithmetic + checksum incremental computation uses checked operators (CVE class: CVE-class NAT range-bypass via overflow) | § Mandatory |
| **CONSTIFY** | per-l4proto + per-helper NAT vtables `static const` | § Mandatory |
| **LATENT_ENTROPY** | NAT port-allocation randomization seeded from kernel CSPRNG (per `NF_NAT_RANGE_PROTO_RANDOM_FULLY` flag) | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: per-conntrack (cross-ref `conntrack-core.md`)
- **CONSTIFY, SIZE_OVERFLOW, LATENT_ENTROPY**: see above
- **USERCOPY**: NLA parsing uses bound-checked accessors
- **KERNEXEC**: per-l4proto + per-helper dispatch via `static const fn-ptr` arrays

### Row-2 / GR-RBAC integration

- LSM hook: same as `conntrack-core.md` (CAP_NET_ADMIN gate).
- Useful default GR-RBAC policy: deny NAT mutations outside gradm-marked `firewall_admin` role; NAT-pool-bypass via misconfigured rule is a privilege-escalation primitive.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

