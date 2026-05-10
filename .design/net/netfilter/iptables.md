# Tier-3: net/netfilter/iptables — per-AF iptables/ip6tables (legacy ABI core)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/ipv4/netfilter/ip_tables.c
  - net/ipv4/netfilter/iptable_filter.c
  - net/ipv4/netfilter/iptable_nat.c
  - net/ipv4/netfilter/iptable_mangle.c
  - net/ipv4/netfilter/iptable_raw.c
  - net/ipv4/netfilter/iptable_security.c
  - net/ipv6/netfilter/ip6_tables.c
  - net/ipv6/netfilter/ip6table_filter.c
  - net/ipv6/netfilter/ip6table_nat.c
  - net/ipv6/netfilter/ip6table_mangle.c
  - net/ipv6/netfilter/ip6table_raw.c
  - net/ipv6/netfilter/ip6table_security.c
  - include/uapi/linux/netfilter_ipv4/ip_tables.h
  - include/uapi/linux/netfilter_ipv6/ip6_tables.h
-->

## Summary
Tier-3 design for the per-AF iptables / ip6tables legacy-ABI core. Each AF (IPv4 + IPv6) registers per-table modules implementing the standard 5-table layout: `filter` (default), `nat`, `mangle`, `raw`, `security`. Each table consists of a per-hook chain (PREROUTING / INPUT / FORWARD / OUTPUT / POSTROUTING — only `valid_hooks` per-table). The per-AF rule format adds AF-specific match (struct `ipt_entry` for IPv4, `ip6t_entry` for IPv6) on top of x_tables' generic match/target framework.

Modern distros ship `iptables-nft` (CLI translates rules to nftables); `iptables-legacy` retains x_tables-based dataplane for backward-compat. Both use the same userspace tooling (`iptables` / `ip6tables` / `iptables-save` / `iptables-restore`).

Sub-tier-3 of `net/netfilter/00-overview.md`. Pairs with `net/netfilter/x-tables.md` (shared framework), `net/netfilter/conntrack-core.md` (consumed by `xt_conntrack`).

## Upstream references in scope

| AF | Paths |
|---|---|
| IPv4 core (rule format, ipt_do_table, table init) | `net/ipv4/netfilter/ip_tables.c` |
| IPv4 filter table | `net/ipv4/netfilter/iptable_filter.c` |
| IPv4 nat table | `net/ipv4/netfilter/iptable_nat.c` |
| IPv4 mangle table | `net/ipv4/netfilter/iptable_mangle.c` |
| IPv4 raw table | `net/ipv4/netfilter/iptable_raw.c` |
| IPv4 security table (LSM-aware) | `net/ipv4/netfilter/iptable_security.c` |
| IPv6 analogues | `net/ipv6/netfilter/ip6*.c` (mirrored) |
| UAPI v4 | `include/uapi/linux/netfilter_ipv4/ip_tables.h` |
| UAPI v6 | `include/uapi/linux/netfilter_ipv6/ip6_tables.h` |

## Compatibility contract

### Five standard tables

| Table | Hooks | Priority | Purpose |
|---|---|---|---|
| `raw` | PREROUTING + OUTPUT | NF_IP_PRI_RAW (-300) | Pre-conntrack: NOTRACK target etc. |
| `mangle` | all 5 hooks | NF_IP_PRI_MANGLE (-150) | Per-skb header mangling (TOS, TTL, MARK) |
| `nat` | PREROUTING (DNAT) + INPUT + OUTPUT (DNAT) + POSTROUTING (SNAT) | NF_IP_PRI_NAT_DST (-100) / NAT_SRC (100) | NAT |
| `filter` | INPUT + FORWARD + OUTPUT | NF_IP_PRI_FILTER (0) | Packet filtering (default table) |
| `security` | INPUT + FORWARD + OUTPUT | NF_IP_PRI_SECURITY (50) | LSM-aware (SELinux secmark) |

Per-table `valid_hooks` bitmask determines which NF_INET_* hooks the table registers. Identical priorities + valid-hooks-per-table.

### `struct ipt_entry` (IPv4 per-rule)

```c
struct ipt_entry {
    struct ipt_ip ip;                    /* per-AF IP-header match */
    unsigned int nfcache;
    __u16 target_offset;                 /* offset to target from start of entry */
    __u16 next_offset;                   /* offset to next entry */
    unsigned int comefrom;
    struct xt_counters counters;
    unsigned char elems[0];              /* per-rule packed array of xt_entry_match + xt_entry_target */
};
```

`struct ipt_ip` carries IPv4-specific match fields:
```c
struct ipt_ip {
    struct in_addr src, dst;
    struct in_addr smsk, dmsk;
    char iniface[IFNAMSIZ], outiface[IFNAMSIZ];
    unsigned char iniface_mask[IFNAMSIZ], outiface_mask[IFNAMSIZ];
    __u16 proto;
    __u8 flags;
    __u8 invflags;
};
```

Layout-byte-identical so iptables-legacy CLI works unchanged.

### `struct ip6t_entry` + `struct ip6t_ip6` (IPv6)

Analogous layout with IPv6 addresses (struct in6_addr) and IPv6-specific fields. Layout-byte-identical.

### Per-AF entry traversal: `ipt_do_table` / `ip6t_do_table`

Per-skb at NF hook:
1. Read `table->private` RCU-side
2. Find first rule for current hook via `hook_entry[hook]` offset
3. For each rule: check IP-header match (per-AF `ip` field); on match, walk `xt_entry_match` array (each match's `match()` callback); if all match, invoke target
4. Handle target verdict: `XT_RETURN` (return from chain), `XT_CONTINUE` (next rule), `NF_*` (terminal verdict)
5. Update per-rule `xt_counters`

Identical algorithm to upstream.

### Per-AF table init: `ipt_register_table` / `ip6t_register_table`

Per-AF + per-table init: registers x_tables `xt_table` for the AF + table; provides per-AF rule-blob init (default policy = ACCEPT for filter; per-table specifics for others).

Identical signatures.

### Built-in policies + chains

Per-table per-hook default policy:
- filter table: ACCEPT for INPUT + FORWARD + OUTPUT
- nat table: ACCEPT for PRE + INPUT + OUTPUT + POSTROUTING
- mangle: ACCEPT for all 5 hooks
- raw: ACCEPT for PRE + OUTPUT
- security: ACCEPT for INPUT + FORWARD + OUTPUT

Identical defaults so iptables-save with no rules dumps identical output.

### Compatibility with conntrack via `xt_conntrack` / `xt_state` / `xt_recent`

Match modules attach to conntrack (for `--ctstate`, `--state`) — these live as separate `xt_match` modules registered globally. Per-AF iptables only needs to ensure conntrack hooks are registered at NF_IP_PRI_CONNTRACK and the rule chain uses xt_conntrack matches when configured.

### NAT table is special: hook only at PRE + INPUT + OUTPUT + POSTROUTING

NAT table's `valid_hooks` excludes FORWARD (per RFC 3022 NAT happens at boundary, not per-flow forwarding). Per-AF NAT tables register the `nat` table with these specific hooks.

### iptable_security LSM integration

`iptable_security` table is registered AFTER filter (priority NF_IP_PRI_SECURITY=50) and provides LSM-driven secmark assignment. SELinux-policy-driven access control on packets via `SECMARK` target.

## Requirements

- REQ-1: Per-AF (IPv4 + IPv6) iptables core: `ipt_do_table` / `ip6t_do_table` per-skb chain walk; identical algorithm.
- REQ-2: Five standard tables (filter / nat / mangle / raw / security) per-AF; `valid_hooks` + priorities byte-identical.
- REQ-3: `struct ipt_entry` + `struct ipt_ip` (IPv4) + `struct ip6t_entry` + `struct ip6t_ip6` (IPv6) byte-identical layout.
- REQ-4: Per-AF default policies match upstream defaults.
- REQ-5: Per-rule traversal: per-AF IP-header match → per-rule xt_entry_match walk → target dispatch. Identical algorithm.
- REQ-6: NAT table `valid_hooks` excludes FORWARD per RFC 3022.
- REQ-7: Security table LSM integration: SELinux secmark assignment via `SECMARK` target; identical contract.
- REQ-8: Per-AF `ipt_register_table` / `ip6t_register_table` per-table init; per-table default-policy initialization.
- REQ-9: setsockopt UAPI: `SO_SET_REPLACE / GET_INFO / GET_ENTRIES / GET_REVISION_* / GET_COUNTERS / ADD_COUNTERS` per-AF byte-identical (cross-ref `x-tables.md` REQ-7).
- REQ-10: Per-AF compat layer (`compat_ipt_replace` etc.) — DEFERRED for v0 (consistent with `xfrm_compat.c` + `x-tables.md` REQ-9).
- REQ-11: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `pahole struct ipt_entry` + `struct ipt_ip` + `struct ip6t_entry` + `struct ip6t_ip6` byte-identical layout. (covers REQ-3)
- [ ] AC-2: `iptables-legacy -A INPUT -p tcp --dport 80 -j ACCEPT` test: NLA-encoded SO_SET_REPLACE byte-identical to upstream's; rule visible via `iptables-legacy -L INPUT`. (covers REQ-1, REQ-2, REQ-5, REQ-9)
- [ ] AC-3: ip6tables test: `ip6tables-legacy -A INPUT ... -j ACCEPT`; per-AF v6 rule visible. (covers REQ-1, REQ-3)
- [ ] AC-4: NAT-table FORWARD rejection test: `iptables-legacy -t nat -A FORWARD -j ACCEPT` returns -EINVAL (FORWARD not in valid_hooks for nat). (covers REQ-6)
- [ ] AC-5: Default policy test: `iptables-legacy -L INPUT` shows policy ACCEPT (for filter table); chain default behavior matches. (covers REQ-4)
- [ ] AC-6: Mangle test: `iptables-legacy -t mangle -A POSTROUTING -j MARK --set-mark 0x100`; outbound packets get skb->mark = 0x100. (covers REQ-1, REQ-5)
- [ ] AC-7: Raw NOTRACK test: `iptables-legacy -t raw -A PREROUTING -p tcp --dport 22 -j NOTRACK`; SSH packets bypass conntrack. (covers REQ-2)
- [ ] AC-8: Security secmark test: `iptables-legacy -t security -A INPUT -j SECMARK --selctx system_u:object_r:firewall_t:s0`; matching packets tagged with SELinux secmark. (covers REQ-7)
- [ ] AC-9: iptables-save / iptables-restore round-trip: dump + restore byte-identical configuration. (covers REQ-9)
- [ ] AC-10: Hardening section present and follows template. (covers REQ-11)

## Architecture

### Rust module organization

- `kernel::net::netfilter::iptables::v4::IpTables` — IPv4 iptables core
- `kernel::net::netfilter::iptables::v4::IptEntry` — `struct ipt_entry` wrapper
- `kernel::net::netfilter::iptables::v4::IptIp` — IPv4 header match
- `kernel::net::netfilter::iptables::v4::DoTable` — `ipt_do_table` per-skb walk
- `kernel::net::netfilter::iptables::v4::tables::Filter`, `Nat`, `Mangle`, `Raw`, `Security` — per-table modules
- `kernel::net::netfilter::iptables::v6::Ip6Tables` — IPv6 mirror
- `kernel::net::netfilter::iptables::v6::Ip6tEntry`, `Ip6tIp6`, `DoTable6`
- `kernel::net::netfilter::iptables::v6::tables::*` — per-table v6 modules
- `kernel::net::netfilter::iptables::Sockopt` — per-AF setsockopt (consumes `xtables::Sockopt`)

### Locking and concurrency

- (Inherited from `x-tables.md`): per-AF mutex for table mutations, RCU for traversal hot path
- **No new global locks**

### Error handling

- (Inherited from `x-tables.md`): `EINVAL`, `ENOMEM`, `EEXIST`, `ENOENT`, `EPERM`, `EPROTOTYPE`, `ELOOP`

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Per-AF IP-header match (saddr/daddr/proto bounds-check) | `kani::proofs::net::netfilter::iptables::ip_match_safety` |
| Per-AF rule traversal (jump-stack inherited from xtables) | `kani::proofs::net::netfilter::iptables::traverse_safety` |
| Per-table init + default policy installation | `kani::proofs::net::netfilter::iptables::table_init_safety` |
| NAT table FORWARD-hook rejection | `kani::proofs::net::netfilter::iptables::nat_hook_safety` |

### Layer 2: TLA+ models

(none new owned at this granularity; consumed via `xt_replace_table` atomicity from `x-tables.md`)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-AF `struct ipt_entry` rule blob | per-rule `target_offset < next_offset ≤ table_size`; byte-aligned | `kani::proofs::net::netfilter::iptables::blob_invariants` |
| Per-table `valid_hooks` | filter ⇒ {INPUT, FORWARD, OUTPUT}; nat ⇒ {PRE, INPUT, OUTPUT, POSTROUTING}; etc. per table | `kani::proofs::net::netfilter::iptables::hooks_invariants` |
| Per-table priority | matches per-AF NF_IP_PRI_* constant | `kani::proofs::net::netfilter::iptables::priority_invariants` |

### Layer 4: Functional correctness (opt-in)

(deferred — same gate as `net/netfilter/00-overview.md`)

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **CONSTIFY** | per-table module's `xt_table` instances `static const` | § Mandatory |
| **SIZE_OVERFLOW** | per-AF rule-blob offset arithmetic uses checked operators | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: per-table_info (cross-ref `x-tables.md`); per-skb (cross-ref `net/skbuff.md`)
- **CONSTIFY, SIZE_OVERFLOW**: see above
- **USERCOPY**: setsockopt copy via `x-tables.md`'s bound-checked accessors
- **KERNEXEC**: per-AF dispatch via `static const fn-ptr`

### Row-2 / GR-RBAC integration

- LSM hook: same as `x-tables.md` (CAP_NET_ADMIN gate); `iptable_security` table specifically interacts with LSM secmark.
- Default useful GR-RBAC policy: deny iptables-legacy mutations outside gradm-marked `firewall_admin` role.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Open Questions

(none — iptables/ip6tables ABI exhaustively specified by upstream + decades of iptables regression test corpus)

## Out of Scope

- ebtables / arptables (cross-ref `net/netfilter/ebtables.md`, `net/netfilter/arptables.md`)
- Per-xt_match / per-xt_target module implementations (deferred to per-module Tier-4 docs if needed)
- 32-bit userspace compat — DEFERRED for v0 (REQ-10)
- 32-bit-only paths
- Implementation code
