# Tier-3: net/netfilter/ebtables — Ethernet bridge filtering (legacy ebtables ABI)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/bridge/netfilter/ebtables.c
  - net/bridge/netfilter/ebtable_filter.c
  - net/bridge/netfilter/ebtable_nat.c
  - net/bridge/netfilter/ebtable_broute.c
  - net/bridge/netfilter/ebt_*.c
  - include/uapi/linux/netfilter_bridge/ebtables.h
-->

## Summary
Tier-3 design for `ebtables` — Ethernet-frame-level packet filtering for Linux bridge ports. Operates at NF_BR_* hooks (PREROUTING / LOCAL_IN / FORWARD / LOCAL_OUT / POSTROUTING / BROUTING) on bridge-managed netdevs. Per-AF rule format (`struct ebt_entry`) carries Ethernet-header match (src/dst MAC, EtherType, VLAN tag, 802.1Q PCP) plus per-rule packed array of ebt_match + ebt_target.

Three standard tables: `filter` (default), `nat` (Ethernet-level SNAT/DNAT), `broute` (BRouting — decide bridge vs route per packet). Modern alternative: `nft add table bridge` rules (nftables for bridge family). `ebtables-nft` translator (libxtables) translates ebtables CLI to nftables in-kernel; ebtables-legacy retains kernel x_tables-like dataplane.

Per-match modules: `ebt_arp` (ARP-specific), `ebt_among` (MAC set membership), `ebt_ip` / `ebt_ip6` (encapsulated IP/IPv6 match), `ebt_802_3` (802.3 LLC), `ebt_stp` (Spanning Tree Protocol), `ebt_vlan`, `ebt_mark_m`, `ebt_pkttype`, `ebt_limit`, etc. Per-target: `ebt_dnat` / `ebt_snat` / `ebt_redirect` / `ebt_arpreply` / `ebt_log` / `ebt_nflog` / `ebt_mark`.

Sub-tier-3 of `net/netfilter/00-overview.md`. Pairs with `net/netfilter/x-tables.md` (similar shared framework, but ebtables predates x_tables and has its own slightly-different per-rule format), `net/netfilter/iptables.md` (sibling for IP-level).

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| ebtables core: per-table register, ebt_do_table, NF_BR_* hooks | `net/bridge/netfilter/ebtables.c` |
| ebt_filter table | `net/bridge/netfilter/ebtable_filter.c` |
| ebt_nat table (Ethernet-level NAT) | `net/bridge/netfilter/ebtable_nat.c` |
| ebt_broute table (bridge-vs-route decision) | `net/bridge/netfilter/ebtable_broute.c` |
| Per-match modules: arp, among, ip, ip6, 802_3, stp, vlan, mark_m, pkttype, limit | `net/bridge/netfilter/ebt_*.c` |
| Per-target modules: dnat, snat, redirect, arpreply, log, nflog, mark | `net/bridge/netfilter/ebt_*.c` |
| UAPI | `include/uapi/linux/netfilter_bridge/ebtables.h` |

## Compatibility contract

### Six bridge hooks (NF_BR_*)

| Hook | Constant | When |
|---|---|---|
| `NF_BR_PRE_ROUTING` | 0 | After RX validation, before bridge forwarding decision |
| `NF_BR_LOCAL_IN` | 1 | Frame destined to a local netdev |
| `NF_BR_FORWARD` | 2 | Frame forwarded between bridge ports |
| `NF_BR_LOCAL_OUT` | 3 | Frame sourced by a local netdev |
| `NF_BR_POST_ROUTING` | 4 | Just before xmit on bridge port |
| `NF_BR_BROUTING` | 5 | Special: bridge-or-route decision |

Identical bit values + semantics.

### `struct ebt_entry` (per-rule)

```c
struct ebt_entry {
    unsigned int bitmask;
    unsigned int invflags;
    __be16 ethproto;
    char in[IFNAMSIZ];
    char logical_in[IFNAMSIZ];
    char out[IFNAMSIZ];
    char logical_out[IFNAMSIZ];
    unsigned char sourcemac[ETH_ALEN];
    unsigned char sourcemsk[ETH_ALEN];
    unsigned char destmac[ETH_ALEN];
    unsigned char destmsk[ETH_ALEN];
    unsigned int watchers_offset;
    unsigned int target_offset;
    unsigned int next_offset;
    unsigned char elems[0];
};
```

Note: ebtables predates xtables; "watchers" are like matches but explicit (legacy term). Layout-byte-identical so ebtables-legacy CLI works unchanged.

### Three standard tables

| Table | Hooks | Purpose |
|---|---|---|
| `filter` | INPUT + FORWARD + OUTPUT | Frame filtering (default policy ACCEPT) |
| `nat` | PREROUTING (DNAT) + OUTPUT (SNAT) + POSTROUTING (SNAT) | Ethernet-level NAT (rare; for L2 anonymization) |
| `broute` | BROUTING (special) | Decide whether frame is bridged or routed |

Identical priorities + valid-hooks per upstream.

### `BROUTING` hook (special)

Unique to ebtables. Frames in the broute table can be routed (re-injected at IP layer) instead of bridged via `EBT_BROUTING` target. Used for transparent proxy setups.

### setsockopt UAPI

Per `SOL_IP_BRIDGE`:
- `EBT_SO_SET_ENTRIES` / `EBT_SO_SET_COUNTERS`
- `EBT_SO_GET_INFO / GET_ENTRIES / GET_INIT_INFO / GET_INIT_ENTRIES`
- `EBT_SO_SET_ENTRIES`

Wire format byte-identical so ebtables-legacy CLI works unchanged.

### NAT-table semantics

Ethernet-level NAT rewrites src/dst MAC. Useful for:
- Bridge-as-MAC-anonymizer (`-A POSTROUTING -j snat --to-source 02:..`)
- Bridge-as-source-MAC-rewriter for VPN tunnel setups

### `ebt_arp` matching

```
struct ebt_arp_info {
    __be16 htype;
    __be16 ptype;
    __be16 opcode;
    struct in_addr saddr;
    struct in_addr daddr;
    struct in_addr smsk;
    struct in_addr dmsk;
    unsigned char smaddr[ETH_ALEN];
    unsigned char dmaddr[ETH_ALEN];
    unsigned char smmsk[ETH_ALEN];
    unsigned char dmmsk[ETH_ALEN];
    __u8 bitmask;
    __u8 invflags;
};
```

Per-ARP-field match. Used for ARP-spoofing prevention rules.

### `ebt_among` (MAC set membership)

Per-set named list of MACs (or MAC+IP pairs). Used to filter against allow-/deny-lists at line rate (similar to ipset for L3, but L2-specific).

## Requirements

- REQ-1: 6 NF_BR_* hooks (PRE_ROUTING / LOCAL_IN / FORWARD / LOCAL_OUT / POST_ROUTING / BROUTING) registered identically.
- REQ-2: 3 standard tables (filter / nat / broute) with byte-identical valid_hooks.
- REQ-3: `struct ebt_entry` byte-identical layout.
- REQ-4: Per-rule traversal: per-rule Ethernet-header match (src/dst MAC + EtherType + IFNAME) → walk ebt_entry_match (watchers) → target dispatch.
- REQ-5: BROUTING hook target `EBT_BROUTING`: marks frame for IP-routing instead of bridging; identical to upstream.
- REQ-6: ebt_arp match per `struct ebt_arp_info` byte-identical layout.
- REQ-7: ebt_among match: per-set named MAC (+IP) list; identical UAPI.
- REQ-8: Per-table init via `ebt_register_table` / `_unregister_table`; identical contract.
- REQ-9: setsockopt UAPI byte-identical (EBT_SO_SET_ENTRIES / GET_INFO / etc.).
- REQ-10: Per-rule counters (`struct ebt_counter`) byte-identical UAPI.
- REQ-11: 32-bit userspace compat — DEFERRED for v0 (consistent with `x-tables.md` REQ-9).
- REQ-12: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `pahole struct ebt_entry` byte-identical layout. (covers REQ-3)
- [ ] AC-2: `ebtables-legacy -A FORWARD --among-src @allowed -j ACCEPT` test: NLA-encoded EBT_SO_SET_ENTRIES byte-identical; rule visible via `ebtables-legacy -L FORWARD`. (covers REQ-1, REQ-2, REQ-7, REQ-9)
- [ ] AC-3: BROUTING test: `ebtables-legacy -t broute -A BROUTING --proto ipv4 -j redirect`; matching IPv4 frames routed via IP layer, not bridged. (covers REQ-5)
- [ ] AC-4: ebt_arp test: `ebtables-legacy -A INPUT -p arp --arp-op Request -j DROP`; ARP requests dropped. (covers REQ-6)
- [ ] AC-5: NAT test: `ebtables-legacy -t nat -A POSTROUTING -j snat --to-source 02:00:00:00:00:01`; outbound frames have src MAC rewritten. (covers REQ-2)
- [ ] AC-6: Counters test: 1000 frames through rule; counters byte/packet aggregate matches. (covers REQ-10)
- [ ] AC-7: `ebtables-legacy -L --Lc -t filter` byte-identical to upstream. (covers REQ-9)
- [ ] AC-8: 32-bit compat: explicitly skipped per REQ-11. (covers REQ-11)
- [ ] AC-9: Hardening section present and follows template. (covers REQ-12)

## Architecture

### Rust module organization

- `kernel::net::netfilter::ebtables::Ebtables` — top-level
- `kernel::net::netfilter::ebtables::EbtEntry` — `struct ebt_entry` wrapper
- `kernel::net::netfilter::ebtables::DoTable` — `ebt_do_table` per-skb walk
- `kernel::net::netfilter::ebtables::tables::Filter`, `Nat`, `Broute`
- `kernel::net::netfilter::ebtables::matches::Arp`, `Among`, `Ip`, `Ip6`, `Lb_802_3`, `Stp`, `Vlan`, `MarkM`, `PkttypeM`, `Limit`
- `kernel::net::netfilter::ebtables::targets::Dnat`, `Snat`, `Redirect`, `Arpreply`, `Log`, `Nflog`, `Mark`
- `kernel::net::netfilter::ebtables::Sockopt` — setsockopt UAPI

### Locking and concurrency

- (Inherited; pattern similar to `x-tables.md`): per-table mutex for mutator; RCU for traversal hot path
- **No new global locks**

### Error handling

- (Inherited from xtables-like pattern)

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Ethernet-header match (src/dst MAC + EtherType bounds-check) | `kani::proofs::net::netfilter::ebtables::eth_match_safety` |
| Per-rule traversal (offset arithmetic + jump-stack inherited) | `kani::proofs::net::netfilter::ebtables::traverse_safety` |
| BROUTING target re-inject (skb->dev change, no use-after-free) | `kani::proofs::net::netfilter::ebtables::brouting_safety` |
| ebt_among set lookup (per-set MAC list) | `kani::proofs::net::netfilter::ebtables::among_safety` |

### Layer 2: TLA+ models

(none new owned)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-AF (NFPROTO_BRIDGE) ebt_table list | every entry has unique name; per-table valid_hooks honored | `kani::proofs::net::netfilter::ebtables::table_invariants` |
| Per-rule offsets | `watchers_offset < target_offset < next_offset ≤ table_size` | `kani::proofs::net::netfilter::ebtables::offset_invariants` |

### Layer 4: Functional correctness (opt-in)

(deferred — same gate as `net/netfilter/00-overview.md`)

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **CONSTIFY** | per-table `ebt_table` instances + per-match/target ops vtables `static const` | § Mandatory |
| **SIZE_OVERFLOW** | per-rule offset + ebt_among set-size arithmetic uses checked operators | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: per-rule blob (cross-ref `x-tables.md`); per-skb (cross-ref `net/skbuff.md`)
- **CONSTIFY, SIZE_OVERFLOW**: see above
- **USERCOPY**: setsockopt blob copy uses bound-checked accessors
- **KERNEXEC**: per-match + per-target dispatch via `static const fn-ptr`

### Row-2 / GR-RBAC integration

- LSM hook: same as `x-tables.md` (CAP_NET_ADMIN gate).
- Default useful GR-RBAC policy: deny ebtables-legacy mutations outside gradm-marked `firewall_admin` role.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Open Questions

(none — ebtables ABI exhaustively specified by upstream + ebtables regression-test suite)

## Out of Scope

- Modern alternative `nft add table bridge` (cross-ref `net/netfilter/nft-api.md`)
- Per-match / per-target deep-dives (deferred to per-module Tier-4 if needed)
- 32-bit userspace compat — DEFERRED for v0 (REQ-11)
- 32-bit-only paths
- Implementation code
