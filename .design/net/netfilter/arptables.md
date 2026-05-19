# Tier-3: net/netfilter/arptables — ARP packet filtering (legacy arptables ABI)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/ipv4/netfilter/arp_tables.c
  - net/ipv4/netfilter/arptable_filter.c
  - net/ipv4/netfilter/arpt_mangle.c
  - include/uapi/linux/netfilter_arp/arp_tables.h
-->

## Summary
Tier-3 design for `arptables` — legacy ARP packet filtering, the ARP-protocol counterpart to iptables. Operates at NF_ARP_* hooks (NF_ARP_IN / OUT / FORWARD) on every netdev that handles ARP traffic. Used to defend against ARP-spoofing, restrict ARP advertisements, or rewrite ARP-reply MACs (via `arpt_mangle` target).

Per-AF rule format (`struct arpt_entry`) carries ARP-header match (src/target IP + MAC, ARP opcode) plus per-rule packed array of arpt_match + arpt_target via x_tables-like format.

One standard table: `filter` (only). Modern alternative: `nft add table arp` rules (nftables for arp family). arptables-legacy retains kernel x_tables-like dataplane.

Sub-tier-3 of `net/netfilter/00-overview.md`. Pairs with `net/netfilter/x-tables.md` (shared framework — arptables uses xtables match/target framework). Sibling of `net/netfilter/iptables.md` and `net/netfilter/ebtables.md`.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| arptables core: arp_tables.c — per-table register, do_table, NF_ARP_* hooks | `net/ipv4/netfilter/arp_tables.c` |
| filter table | `net/ipv4/netfilter/arptable_filter.c` |
| arpt_mangle target (rewrite ARP reply MACs) | `net/ipv4/netfilter/arpt_mangle.c` |
| UAPI | `include/uapi/linux/netfilter_arp/arp_tables.h` |

## Compatibility contract

### Three ARP hooks (NF_ARP_*)

| Hook | Constant | When |
|---|---|---|
| `NF_ARP_IN` | 0 | ARP frame received |
| `NF_ARP_OUT` | 1 | ARP frame about to xmit |
| `NF_ARP_FORWARD` | 2 | ARP frame forwarded (rare; bridge w/ arp-forward) |

Identical bit values + semantics.

### `struct arpt_entry` (per-rule)

```c
struct arpt_entry {
    struct arpt_arp arp;
    unsigned int target_offset;
    unsigned int next_offset;
    unsigned int comefrom;
    struct xt_counters counters;
    unsigned char elems[0];
};
```

`struct arpt_arp` carries ARP-specific match fields:
```c
struct arpt_arp {
    struct in_addr src;
    struct in_addr tgt;
    struct in_addr smsk;
    struct in_addr tmsk;
    __u8 arhln;        /* hardware address length */
    __u8 arhln_mask;
    struct arpt_devaddr_info src_devaddr;   /* hw src + mask */
    struct arpt_devaddr_info tgt_devaddr;   /* hw tgt + mask */
    __be16 arpop;
    __be16 arpop_mask;
    __be16 arhrd;
    __be16 arhrd_mask;
    __be16 arpro;       /* protocol type */
    __be16 arpro_mask;
    char iniface[IFNAMSIZ], outiface[IFNAMSIZ];
    unsigned char iniface_mask[IFNAMSIZ], outiface_mask[IFNAMSIZ];
    __u16 flags;
    __u16 invflags;
};
```

`struct arpt_devaddr_info`: hardware address + mask (16 bytes).

Layout-byte-identical so arptables-legacy CLI works unchanged.

### One standard table: `filter`

Default policy ACCEPT for all 3 hooks. The only built-in table; no NAT or mangle tables (unlike iptables/ebtables).

### `arpt_mangle` target

Rewrites fields of an ARP packet:
- `src_ip` (sender protocol address)
- `tgt_ip` (target protocol address)
- `src_hw` (sender hardware address)
- `tgt_hw` (target hardware address)
- `target_action` (verdict — ACCEPT / DROP / CONTINUE)

Used for ARP-NAT-like setups (rare). UAPI byte-identical.

### setsockopt UAPI

Per `SOL_IP`:
- `ARPT_SO_SET_REPLACE` / `_SET_ADD_COUNTERS`
- `ARPT_SO_GET_INFO / _GET_ENTRIES / _GET_REVISION_TARGET`

Wire format byte-identical so arptables-legacy CLI works.

### Use cases

- ARP-spoofing prevention: drop ARP-replies claiming a non-allowed MAC for a critical IP
- Restrict ARP advertisements: only allow ARP-reply from authorized hosts
- ARP rate-limiting via `arpt_mangle` + drop-on-overflow

## Requirements

- REQ-1: 3 NF_ARP_* hooks (IN / OUT / FORWARD) registered identically.
- REQ-2: 1 standard table (filter) with default ACCEPT policy.
- REQ-3: `struct arpt_entry` + `struct arpt_arp` + `struct arpt_devaddr_info` byte-identical layout.
- REQ-4: Per-rule traversal: per-rule ARP-header match → walk match modules → target dispatch. Identical algorithm.
- REQ-5: `arpt_mangle` target: rewrite src_ip / tgt_ip / src_hw / tgt_hw + per-target verdict; identical contract.
- REQ-6: Per-rule counters via `xt_counters`; aggregated per-CPU.
- REQ-7: setsockopt UAPI: ARPT_SO_SET_REPLACE / GET_INFO / GET_ENTRIES / GET_REVISION_TARGET byte-identical.
- REQ-8: 32-bit userspace compat — DEFERRED for v0 (consistent with `x-tables.md` REQ-9).
- REQ-9: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `pahole struct arpt_entry` + `struct arpt_arp` byte-identical layout. (covers REQ-3)
- [ ] AC-2: `arptables -A INPUT --source-mac 00:11:22:33:44:55 -j DROP` test: NLA-encoded ARPT_SO_SET_REPLACE byte-identical; rule visible via `arptables -L INPUT`. (covers REQ-1, REQ-2, REQ-4, REQ-7)
- [ ] AC-3: ARP-spoofing-prevention test: install rule blocking ARP-reply with mismatched src MAC; spoofed reply dropped + counter incremented; legit reply passes. (covers REQ-4)
- [ ] AC-4: arpt_mangle test: install rule rewriting src_hw of outbound ARP-replies; tcpdump shows rewritten MAC. (covers REQ-5)
- [ ] AC-5: Counters test: 1000 ARP frames matching rule → per-CPU aggregated counters byte=Σ(frame sizes), packets=1000. (covers REQ-6)
- [ ] AC-6: `arptables -L --line-numbers -v` byte-identical to upstream. (covers REQ-7)
- [ ] AC-7: 32-bit compat: explicitly skipped per REQ-8. (covers REQ-8)
- [ ] AC-8: Hardening section present and follows template. (covers REQ-9)

## Architecture

### Rust module organization

- `kernel::net::netfilter::arptables::Arptables` — top-level
- `kernel::net::netfilter::arptables::ArptEntry` — `struct arpt_entry` wrapper
- `kernel::net::netfilter::arptables::ArptArp` — ARP-header match
- `kernel::net::netfilter::arptables::DoTable` — per-skb chain walk
- `kernel::net::netfilter::arptables::tables::Filter` — only built-in table
- `kernel::net::netfilter::arptables::targets::Mangle` — `arpt_mangle`
- `kernel::net::netfilter::arptables::Sockopt` — setsockopt UAPI

### Locking and concurrency

- (Inherited from `x-tables.md`): per-AF mutex for mutations, RCU for traversal hot path
- **No new global locks**

### Error handling

- (Inherited from xtables-like pattern)

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| ARP-header match (src/tgt IP + MAC bounds-check) | `kani::proofs::net::netfilter::arptables::arp_match_safety` |
| Per-rule traversal | `kani::proofs::net::netfilter::arptables::traverse_safety` |
| arpt_mangle skb mangling (skb_pull bounds + checksum incremental) | `kani::proofs::net::netfilter::arptables::mangle_safety` |

### Layer 2: TLA+ models

(none new owned)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-AF (NFPROTO_ARP) arpt_table list | filter table is present; valid_hooks = {IN, OUT, FORWARD} | `kani::proofs::net::netfilter::arptables::table_invariants` |
| Per-rule offsets | `target_offset < next_offset ≤ table_size`; byte-aligned | `kani::proofs::net::netfilter::arptables::offset_invariants` |

### Layer 4: Functional correctness (opt-in)

(deferred — same gate as `net/netfilter/00-overview.md`)

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **CONSTIFY** | per-table arpt_table instance + arpt_mangle ops `static const` | § Mandatory |
| **SIZE_OVERFLOW** | per-rule offset arithmetic uses checked operators | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: per-rule blob (cross-ref `x-tables.md`); per-skb (cross-ref `net/skbuff.md`)
- **CONSTIFY, SIZE_OVERFLOW**: see above
- **USERCOPY**: setsockopt copy via `x-tables.md`'s bound-checked accessors
- **KERNEXEC**: dispatch via `static const fn-ptr`

### Row-2 / GR-RBAC integration

- LSM hook: same as `x-tables.md` (CAP_NET_ADMIN gate).
- Default useful GR-RBAC policy: deny arptables-legacy mutations outside gradm-marked `firewall_admin` role.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Grsecurity/PaX-style Reinforcement

Rationale: arptables is the legacy ARP filtering ABI (ip_tables-style with `ARPT_*` setsockopt) — long deprecated in favour of nftables, but still shipped for back-compat. The setsockopt path takes a userspace-supplied `xt_table_info` blob with per-entry offsets and target verdicts that must be validated before any indirect call, and the `xt_table` global blob is the canonical "swap entire ruleset" surface — a malformed table can pivot to OOB target dispatch.

Baseline (cross-ref `net/netfilter/00-overview.md` § Hardening):
- **PAX_USERCOPY**: `xt_table_info` blob copy_from_user via slab-USERCOPY-whitelisted; per-entry `arpt_entry` walk bounded.
- **PAX_KERNEXEC**: `arpt_standard_target`, `arpt_error_target`, per-match/target `xt_match`/`xt_target` registrations placed `__ro_after_init`.
- **PAX_RANDKSTACK**: re-randomise on every `arpt_do_table` invocation from the netfilter hook.
- **PAX_REFCOUNT**: `xt_table` + per-cpu counter refs saturating.
- **PAX_MEMORY_SANITIZE**: freed table info + per-cpu counters zero-filled before slab reuse.
- **PAX_UDEREF**: setsockopt `ARPT_SO_SET_REPLACE` / `ARPT_SO_GET_INFO` via `copy_from_sockptr`.
- **PAX_RAP / kCFI**: `xt_target->target` and `xt_match->match` indirect calls kCFI-tagged.
- **GRKERNSEC_HIDESYM**: `xt_table` + per-cpu counter pointers never in `/proc/net/arp_tables_*` (only %pK).
- **GRKERNSEC_DMESG**: setsockopt-parse-fail warns ratelimited.

arptables-specific reinforcement:
- **Legacy CAP_NET_ADMIN strict-in-userns** — refuse arptables setsockopt from non-init userns under default GR policy.
- **x_table PAX_USERCOPY** — ruleset blob copy bounded by `xt_table_info.size`; refuse on length-mismatch + audit-log.
- **Per-entry offset chain bound** — verify each `arpt_entry.next_offset` is monotonically increasing and ≤ blob size before any target dispatch.
- **Target verdict bound** — `arpt_entry.target_offset + sizeof(struct arpt_entry_target)` ≤ next_offset; refuse otherwise.
- **Deprecated `pr_warn_once` on first ruleset load** — encourages migration to nftables.
- **GR-RBAC `legacy_iptables_users` role** — only marked subjects may use `ARPT_SO_*` setsockopt; CAP_NET_ADMIN alone insufficient under default policy.
- **Ruleset swap atomicity** — old-table free uses `synchronize_rcu`-then-sanitize sequence (defends use-after-swap).

## Open Questions

(none — arptables ABI exhaustively specified by upstream)

## Out of Scope

- Modern alternative `nft add table arp` (cross-ref `net/netfilter/nft-api.md`)
- 32-bit userspace compat — DEFERRED for v0 (REQ-8)
- 32-bit-only paths
- Implementation code
