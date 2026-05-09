---
title: "Tier-2 (sub): net/netfilter/ipset — named IP/port/MAC set framework"
tags: ["design-doc", "tier-2", "net", "netfilter"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Sub-Tier-2 overview for the `ipset` framework — named, typed sets of IPs / ports / MACs / network prefixes / concatenations queryable in O(1) (hash) or O(log n) (rbtree) for fast set-membership testing. Used by:
- iptables `xt_set` match (`-m set --match-set blacklist src`)
- nftables `lookup` expression (sets are nftables-internal but ipset coexists)
- tc `cls_basic` em_ipset extended-match
- Cilium / firewalld / fail2ban for dynamic block-/allow-lists with timeouts

Set types organized in three families:
- **bitmap:** for ≤8-bit key spaces (small finite ranges)
- **hash:** for unique-element keys (the most-used family — 9 variants)
- **list:** meta-set containing other named sets (set-of-sets)

Per-set-type supports per-element timeouts, per-element comments, per-element flags, per-element counters, per-element skbinfo (mark+priority+queue), and the COMPLEX nested types like `hash:ip,port,proto,ip` for 4-tuple matching.

Sub-tier-2 of `net/netfilter/00-overview.md`. Pairs with `net/sched/em-meta.md` (`em_ipset` predicate consumes ipset).

### Out of Scope

- Per-set-type implementations (cross-ref child Tier-3s)
- 32-bit-only paths
- Implementation code

### scope

This sub-Tier-2 governs **all** of `/home/doll/linux-src/net/netfilter/ipset/` (~25 source files) plus the public API + UAPI headers. Per-set-type Tier-3 docs would split by family (bitmap / hash / list).

### compatibility contract — outline

### `ipset` UAPI: NFNL_SUBSYS_IPSET via libipset

```
IPSET_CMD_NONE     = 0
IPSET_CMD_PROTOCOL = 1   /* protocol version negotiation */
IPSET_CMD_CREATE   = 2   /* create new named set */
IPSET_CMD_DESTROY  = 3   /* destroy set */
IPSET_CMD_FLUSH    = 4   /* flush set elements */
IPSET_CMD_RENAME   = 5   /* rename set */
IPSET_CMD_SWAP     = 6   /* swap two sets atomically */
IPSET_CMD_LIST     = 7   /* list sets / elements */
IPSET_CMD_SAVE     = 8   /* dump for save */
IPSET_CMD_ADD      = 9   /* add element */
IPSET_CMD_DEL      = 10  /* delete element */
IPSET_CMD_TEST     = 11  /* test membership */
IPSET_CMD_HEADER   = 12  /* list set headers only */
IPSET_CMD_TYPE     = 13  /* list registered set types */
IPSET_CMD_PROTOCOL_MIN = 14
IPSET_CMD_GET_BYNAME = 15
IPSET_CMD_GET_BYINDEX = 16
```

Wire format byte-identical so `ipset` CLI works unchanged.

### Set-type families

| Family | Variants | Best for |
|---|---|---|
| **bitmap** | `bitmap:ip`, `bitmap:ip,mac`, `bitmap:port` | Small (≤65k) integer-keyed sets |
| **hash** | `hash:ip`, `hash:mac`, `hash:net`, `hash:net,iface`, `hash:net,port`, `hash:net,port,net`, `hash:ip,mac`, `hash:ip,mark`, `hash:ip,port`, `hash:ip,port,ip`, `hash:ip,port,net` | Generic O(1) lookup; dominant in production |
| **list** | `list:set` | Meta-set of named sets |

Identical type set so existing scripts work.

### Per-set features

Per-set creation flags + extension support:
- `IPSET_FLAG_TIMEOUT` — per-element timeout
- `IPSET_FLAG_COUNTERS` — per-element packet/byte counters
- `IPSET_FLAG_COMMENT` — per-element comment string
- `IPSET_FLAG_SKBINFO` — per-element skb metadata (mark, priority, queue)
- `IPSET_FLAG_FORCEADD` — when set full + adding, evict oldest

Per-element extras attached as `ip_set_ext_*`. Identical UAPI.

### Per-set-type vtable

```c
struct ip_set_type_variant {
    int (*kadt)(struct ip_set *, const struct sk_buff *, const struct xt_action_param *, enum ipset_adt, struct ip_set_adt_opt *);
    int (*uadt)(struct ip_set *, struct nlattr *tb[], enum ipset_adt, u32 *lineno, u32 flags, bool retried);
    /* core operations */
    int (*adt)(struct ip_set *, void *, u32 timeout, u32 flags, struct ip_set_ext *);
    int (*resize)(struct ip_set *, bool retried);
    void (*destroy)(struct ip_set *);
    void (*flush)(struct ip_set *);
    int (*head)(struct ip_set *, struct sk_buff *);
    int (*list)(const struct ip_set *, struct sk_buff *, struct netlink_callback *);
    bool (*same_set)(const struct ip_set *, const struct ip_set *);
    void (*cancel_gc)(struct ip_set *);
    /* per-set memory accounting */
    size_t (*memsize)(const struct ip_set *);
};
```

Per-set-family code via `ip_set_hash_gen.h` / `ip_set_bitmap_gen.h` template-generation (similar to C++ templates via header includes).

### Per-set-type protocol revision

Per `IPSET_CMD_PROTOCOL`: kernel reports protocol version (currently 7); userspace iterates back to find compatible version. Identical mechanism.

### Atomic swap (`IPSET_CMD_SWAP`)

Two same-type sets can be atomically swapped via `IPSET_CMD_SWAP`; userspace builds new set in name-A-tmp, then swaps name-A-tmp ↔ name-A, gets atomic ruleset replacement without disturbing rules referencing the named set.

Identical wire format.

### Per-element timeouts

Per-set `IPSET_FLAG_TIMEOUT`: per-element `timeout` field; periodic GC every `IPSET_GC_PERIOD` (default 60s); expired elements removed.

### sub-Tier-3 docs governed

(Phase C will add these incrementally.)

| Tier-3 doc | Scope |
|---|---|
| `net/netfilter/ipset/core.md` | ip_set_core.c — registry + NFNL handler + protocol versioning |
| `net/netfilter/ipset/types-bitmap.md` | bitmap family (ip / ip,mac / port) |
| `net/netfilter/ipset/types-hash.md` | hash family (all 11 variants) |
| `net/netfilter/ipset/types-list.md` | list:set meta-set |

### compatibility outline (top-level)

- REQ-O1: `IPSET_CMD_*` message types parsed identically; full UAPI byte-identical.
- REQ-O2: All set-type families (bitmap / hash / list) with full variant set.
- REQ-O3: Per-set features (TIMEOUT / COUNTERS / COMMENT / SKBINFO / FORCEADD) byte-identical UAPI.
- REQ-O4: Per-set-type vtable (`ip_set_type_variant`) layout-identical.
- REQ-O5: Atomic swap (IPSET_CMD_SWAP) RCU-pointer-swap semantics.
- REQ-O6: Per-element timeout + GC every IPSET_GC_PERIOD.
- REQ-O7: Per-Tier-3 child documents define per-component requirements.
- REQ-O8: Hardening: row-1 features applied per `00-security-principles.md`.

### acceptance criteria (top-level)

- [ ] AC-O1: `ipset list` byte-identical content. (covers REQ-O1)
- [ ] AC-O2: Each set-type variant creatable + addable + testable; `ipset test` returns 0 on hit + 1 on miss. (covers REQ-O2)
- [ ] AC-O3: Per-feature test for TIMEOUT / COUNTERS / COMMENT / SKBINFO / FORCEADD. (covers REQ-O3)
- [ ] AC-O4: Atomic swap test: build new set in tmp + swap; rules referencing original-name see new content atomically. (covers REQ-O5)
- [ ] AC-O5: Per-element timeout test: add element with timeout 5s; wait 6s; element auto-removed. (covers REQ-O6)
- [ ] AC-O6: Per-Tier-3 ACs cumulatively cover ipset ABI. (meta-AC)
- [ ] AC-O7: Hardening section per Tier-3 child docs is non-empty. (covers REQ-O8)

### architecture (top-level)

Each Tier-3 child doc declares its own Rust module organization. Shared abstractions:

- `kernel::net::netfilter::ipset::IpSet` — `struct ip_set` (per-set state)
- `kernel::net::netfilter::ipset::TypeRegistry` — per-system type registry
- `kernel::net::netfilter::ipset::TypeVariant` — `ip_set_type_variant` vtable
- `kernel::net::netfilter::ipset::Extension` — per-element ext_* (timeout / counters / comment / skbinfo)
- `kernel::net::netfilter::ipset::nfnl::Api` — NFNL_SUBSYS_IPSET handler

### verification (top-level)

### Layer 1: Kani SAFETY proofs

Each Tier-3 child doc declares its own.

### Layer 2: TLA+ models — mandatory list

(none mandatory at this level; per-Tier-3 may add)

### Layer 3: invariant harnesses

Per Tier-3.

### Layer 4: functional correctness (opt-in)

- **Atomic-swap soundness** via Verus — proves: IPSET_CMD_SWAP is atomic from concurrent test/lookup's perspective.

### hardening (top-level)

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this Tier-2

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-set + per-element refcounts use `Refcount` (saturating) | § Mandatory |
| **AUTOSLAB** | per-set-type slab caches | § Mandatory |
| **MEMORY_SANITIZE** | freed elements cleared (carries IP/port/MAC selectors + per-element comments) | § Default-on configurable off |

### Row-1 features consumed by this Tier-2

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: see above
- **CONSTIFY**: per-set-type `ip_set_type` instances + per-variant `ip_set_type_variant` `static const`
- **USERCOPY**: NLA parsing uses bound-checked accessors
- **SIZE_OVERFLOW**: per-element timeout + counter + memory-accounting arithmetic uses checked operators
- **KERNEXEC**: per-set-type dispatch via `static const fn-ptr` arrays

### Row-2 / GR-RBAC integration

- LSM hook `security_capable(CAP_NET_ADMIN)` already required for IPSET_CMD_CREATE / ADD / DEL.
- Default useful GR-RBAC policy: deny ipset mutations outside gradm-marked `firewall_admin` role.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

