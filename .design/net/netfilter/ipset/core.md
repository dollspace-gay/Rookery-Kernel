# Tier-3: net/netfilter/ipset/core — ipset framework + NFNL_SUBSYS_IPSET handler

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/netfilter/ipset/ip_set_core.c
  - net/netfilter/ipset/ip_set_getport.c
  - net/netfilter/ipset/pfxlen.c
  - include/linux/netfilter/ipset/ip_set.h
  - include/uapi/linux/netfilter/ipset/ip_set.h
-->

## Summary
Tier-3 design for the ipset core: per-system set-type registry, per-netns named-set table, NFNL_SUBSYS_IPSET userspace control interface (`IPSET_CMD_*` messages), per-set-type protocol-revision negotiation, atomic SWAP/RENAME/DESTROY semantics, per-element-extension framework for TIMEOUT/COUNTERS/COMMENT/SKBINFO/SKBINFO_ETHERTYPE, in-kernel hot-path `ip_set_test` / `_add` / `_del` for `xt_set` / `nft lookup` / `em_ipset` consumers, and the per-set-type protocol-version (currently 7) compatibility negotiation.

Sub-tier-3 of `net/netfilter/ipset/00-overview.md`. Pairs with `net/netfilter/ipset/types-hash.md` (the 11-variant hash family — most-used).

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Core: per-set-type registry, NFNL handler, in-kernel test/add/del | `net/netfilter/ipset/ip_set_core.c` |
| L4-port extraction helper (used by hash:ip,port etc.) | `net/netfilter/ipset/ip_set_getport.c` |
| Prefix-length lookup table for hash:net family | `net/netfilter/ipset/pfxlen.c` |
| Public API | `include/linux/netfilter/ipset/ip_set.h` |
| UAPI: IPSET_CMD_*, IPSET_ATTR_*, IPSET_FLAG_* | `include/uapi/linux/netfilter/ipset/ip_set.h` |

## Compatibility contract

### `struct ip_set` (per-set state)

```c
struct ip_set {
    struct list_head list;
    char name[IPSET_MAXNAMELEN];
    spinlock_t lock;
    refcount_t ref;
    refcount_t ref_netlink;
    u32 timeout;
    u32 elements;
    u32 ext_size;
    u32 quota;
    u8 family;
    u8 revision;
    u8 extensions;
    u8 flags;
    u32 markmask;
    u8 netmask;
    u8 type;
    void *data;
    const struct ip_set_type *type_t;
    const struct ip_set_type_variant *variant;
};
```

Layout-equivalent for first cache-line.

### `struct ip_set_type` (per-type registration)

```c
struct ip_set_type {
    struct list_head list;
    char name[IPSET_MAXNAMELEN];
    u8 family;
    u8 revision_min;
    u8 revision_max;
    u8 features;
    u8 dimension;
    int (*create)(struct net *, struct ip_set *, struct nlattr *tb[], u32 flags);
    const struct nla_policy create_policy[IPSET_ATTR_CREATE_MAX + 1];
    const struct nla_policy adt_policy[IPSET_ATTR_ADT_MAX + 1];
    struct module *me;
};
```

Layout-byte-identical so existing per-type modules' static instances work.

### `struct ip_set_type_variant` (per-type vtable)

```c
struct ip_set_type_variant {
    int (*kadt)(struct ip_set *, const struct sk_buff *, const struct xt_action_param *, enum ipset_adt, struct ip_set_adt_opt *);
    int (*uadt)(struct ip_set *, struct nlattr *tb[], enum ipset_adt, u32 *lineno, u32 flags, bool retried);
    int (*adt)(struct ip_set *, void *value, u32 timeout, u32 flags, struct ip_set_ext *);
    int (*resize)(struct ip_set *, bool retried);
    void (*destroy)(struct ip_set *);
    void (*flush)(struct ip_set *);
    int (*head)(struct ip_set *, struct sk_buff *);
    int (*list)(const struct ip_set *, struct sk_buff *, struct netlink_callback *);
    bool (*same_set)(const struct ip_set *, const struct ip_set *);
    void (*cancel_gc)(struct ip_set *);
    size_t (*memsize)(const struct ip_set *);
};
```

`kadt` is the kernel-context add/del/test entry (called from `xt_set` / `nft lookup` / `em_ipset`). `uadt` is the userspace-context entry (called from NFNL handler). Both must yield identical results for same input.

### IPSET_CMD_* dispatch

Per-msg handler table in `ip_set_core.c`:

| Command | Handler |
|---|---|
| `CREATE` | `ip_set_create` (validate + alloc + init) |
| `DESTROY` | `ip_set_destroy` (refcount-drop check; reject if in-use) |
| `FLUSH` | `ip_set_flush` (per-type flush callback) |
| `RENAME` | `ip_set_rename` (atomic name change) |
| `SWAP` | `ip_set_swap` (atomic two-set swap) |
| `LIST` / `SAVE` | `ip_set_dump` (multi-part dump) |
| `ADD` / `DEL` / `TEST` | `ip_set_uadd_uadt` / `_uadt` |
| `HEADER` | `ip_set_header` (set metadata only) |
| `TYPE` | `ip_set_type` (registered types list) |
| `PROTOCOL` / `PROTOCOL_MIN` | protocol-version negotiation |
| `GET_BYNAME` / `GET_BYINDEX` | name↔index lookup |

Identical dispatch.

### Atomic SWAP

Two same-typed sets atomically swap their `data` + per-element state via per-set lock-pair. Index/name preserved; rules referencing the named set see new content atomically. RCU-pointer for in-kernel consumers' refcounting.

Identical algorithm.

### `ip_set_ext_*` per-element extensions

```c
struct ip_set_ext {
    u64 packets;          /* COUNTERS */
    u64 bytes;
    u32 timeout;          /* TIMEOUT */
    char *comment;        /* COMMENT */
    struct ip_set_skbinfo skbinfo;   /* SKBINFO: mark + priority + queue */
};
```

Per-set `extensions` flag-set determines which extensions per-element are actually allocated/used. Identical layout.

### Per-element timeout + GC

Per-set GC timer fires every `IPSET_GC_PERIOD` (default 1/4 of set's `timeout` setting); walks elements; expired ones removed. Per-element timestamp tracked via jiffies.

### Protocol version negotiation

Userspace queries `IPSET_CMD_PROTOCOL` → kernel returns `protocol = 7` (current). Userspace iterates back to find compatible version. Each set-type module declares `revision_min` and `revision_max`; userspace's requested revision must fall in that range.

Per-revision changes are documented in upstream ipset history (e.g., revision 4 added per-element comments).

### In-kernel `kadt` hot path

```c
int ip_set_test(ip_set_id_t index, const struct sk_buff *skb,
                const struct xt_action_param *par,
                struct ip_set_adt_opt *opt);
int ip_set_add(ip_set_id_t index, const struct sk_buff *skb,
               const struct xt_action_param *par,
               struct ip_set_adt_opt *opt);
int ip_set_del(ip_set_id_t index, const struct sk_buff *skb,
               const struct xt_action_param *par,
               struct ip_set_adt_opt *opt);
```

Index resolved via `ip_set_get_byname` (returns index for fast path) at rule-install time; subsequent per-skb lookups use index directly. Identical contract.

### `ip_set_getport` + `pfxlen` helpers

- `ip_set_getport.c`: extracts L4 port from skb (TCP/UDP/UDPLite/SCTP/DCCP) for hash:ip,port-style sets; NF-conntrack-aware
- `pfxlen.c`: precomputed prefix-length-to-mask table for hash:net family fast-path

Identical helpers.

## Requirements

- REQ-1: `struct ip_set` + `struct ip_set_type` + `struct ip_set_type_variant` byte-identical first cache-line.
- REQ-2: All `IPSET_CMD_*` message types dispatched identically.
- REQ-3: Protocol-version negotiation: kernel reports protocol=7; per-type `revision_min`/`max` honored.
- REQ-4: Atomic SWAP: two same-typed sets exchange `data` + per-element state under paired per-set locks; identical algorithm.
- REQ-5: RENAME: per-set name mutation; reject if new name exists.
- REQ-6: Per-element extensions: TIMEOUT / COUNTERS / COMMENT / SKBINFO; `ip_set_ext` layout-byte-identical.
- REQ-7: Per-element timeout GC: per-set GC timer; expired elements removed.
- REQ-8: In-kernel `ip_set_test` / `_add` / `_del`: index-resolved fast path used by xt_set / nft / em_ipset; identical contract.
- REQ-9: `ip_set_getport`: per-l4proto port extraction; NF-conntrack-aware fragment handling.
- REQ-10: `pfxlen`: precomputed prefix-length-to-IPv4/IPv6-mask table; identical contents.
- REQ-11: Multi-part dump for IPSET_CMD_LIST / SAVE; per-walker cursor identical.
- REQ-12: 32-bit userspace compat — DEFERRED for v0 (consistent with `x-tables.md` REQ-9, `xfrm_compat.c`).
- REQ-13: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `pahole struct ip_set` + `ip_set_type` + `ip_set_type_variant` byte-identical first cache-line. (covers REQ-1)
- [ ] AC-2: `ipset create test hash:ip; ipset add test 1.2.3.4` test: NLA-encoded IPSET_CMD_CREATE+ADD byte-identical to upstream's; `ipset list test` shows 1.2.3.4. (covers REQ-2)
- [ ] AC-3: SWAP test: build set A; build set B; `ipset swap A B`; rules referencing A now see B's content; semantics atomic. (covers REQ-4)
- [ ] AC-4: RENAME test: `ipset rename A C`; subsequent ops on C work; A no longer exists; rules referencing A fail. (covers REQ-5)
- [ ] AC-5: TIMEOUT test: `ipset create timed hash:ip timeout 5`; `ipset add timed 1.2.3.4`; wait 6s; `ipset test timed 1.2.3.4` returns 1 (not in set). (covers REQ-6, REQ-7)
- [ ] AC-6: COUNTERS test: `ipset create cnt hash:ip counters; ipset add cnt 1.2.3.4`; iptables-rule using set; matching packets bump per-element pcnt+bcnt. (covers REQ-6)
- [ ] AC-7: COMMENT test: `ipset add cnt 1.2.3.4 comment "blocked-by-policy-X"`; `ipset list cnt` shows comment. (covers REQ-6)
- [ ] AC-8: Protocol negotiation test: `ipset --protocol`; returns 7; per-type revision_min/max query works. (covers REQ-3)
- [ ] AC-9: In-kernel hot-path test: `iptables-legacy -A INPUT -m set --match-set blacklist src -j DROP`; matching packets pass through ip_set_test fast path; non-matching packets pass. (covers REQ-8)
- [ ] AC-10: ip_set_getport test: hash:ip,port set; TCP packet extracts port via getport; UDP extracts; ICMP rejects (no port). (covers REQ-9)
- [ ] AC-11: pfxlen test: hash:net set; prefix lookup uses precomputed mask table; ipset list shows correct CIDR notation. (covers REQ-10)
- [ ] AC-12: 32-bit compat test: explicitly skipped per REQ-12. (covers REQ-12)
- [ ] AC-13: Hardening section present and follows template. (covers REQ-13)

## Architecture

### Rust module organization

- `kernel::net::netfilter::ipset::core::TypeRegistry` — per-system set-type list
- `kernel::net::netfilter::ipset::core::Set` — `struct ip_set`
- `kernel::net::netfilter::ipset::core::TypeVariant` — `ip_set_type_variant` vtable
- `kernel::net::netfilter::ipset::core::Extensions` — per-element ext_* (TIMEOUT/COUNTERS/COMMENT/SKBINFO)
- `kernel::net::netfilter::ipset::core::nfnl::Dispatch` — IPSET_CMD_* dispatch table
- `kernel::net::netfilter::ipset::core::nfnl::Create`, `Destroy`, `Flush`, `Rename`, `Swap`, `List`, `Add`, `Del`, `Test`, `Header`, `TypeQuery`, `Protocol`
- `kernel::net::netfilter::ipset::core::Gc` — periodic GC for TIMEOUT-extension sets
- `kernel::net::netfilter::ipset::core::HotPath` — `ip_set_test/_add/_del` for in-kernel consumers
- `kernel::net::netfilter::ipset::core::Getport` — L4-port extraction
- `kernel::net::netfilter::ipset::core::Pfxlen` — prefix-length-to-mask table

### Locking and concurrency

- **`ip_set_type_mutex`** (per-system mutex): per-system set-type registry mutator
- **Per-netns `ip_set_list_lock`** (mutex): per-netns named-set list mutator
- **Per-set `lock`** (spinlock): per-set element mutator
- **Per-set `ref` + `ref_netlink` refcounts**: track in-kernel + netlink consumers
- **RCU**: in-kernel hot path is RCU-side per-element

### Error handling

- `Err(EEXIST)` — duplicate set name
- `Err(ENOENT)` — no such set
- `Err(EBUSY)` — DESTROY refused (in-use)
- `Err(EPROTOTYPE)` — protocol/revision mismatch
- `Err(EINVAL)` — bad NLA / type-specific param
- `Err(ENOMEM)` — alloc fail
- `Err(EPERM)` — non-CAP_NET_ADMIN
- `Err(EOVERFLOW)` — set full + no FORCEADD

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Per-system type registry insert/erase | `kani::proofs::net::netfilter::ipset::core::registry_safety` |
| SWAP atomic-pair locks (deadlock-free; per-set canonical-order locking) | `kani::proofs::net::netfilter::ipset::core::swap_safety` |
| Per-element extension layout (offset arithmetic via ip_set_ext_offsets) | `kani::proofs::net::netfilter::ipset::core::ext_safety` |
| Per-set GC walk (no use-after-free of just-removed elements) | `kani::proofs::net::netfilter::ipset::core::gc_safety` |
| `ip_set_getport` per-l4proto extraction (skb-pull bounds) | `kani::proofs::net::netfilter::ipset::core::getport_safety` |
| `pfxlen` prefix-mask lookup (no out-of-bounds for prefix > 128) | `kani::proofs::net::netfilter::ipset::core::pfxlen_safety` |

### Layer 2: TLA+ models

(none new owned at this granularity)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-system type registry | every entry has unique `name`; revision range non-empty | `kani::proofs::net::netfilter::ipset::core::registry_invariants` |
| Per-netns named-set list | every set has unique `name` within netns | `kani::proofs::net::netfilter::ipset::core::name_invariants` |
| Per-set refcount | `ref ≥ 0`; `ref_netlink ≥ 0`; DESTROY blocked while ref+ref_netlink > 0 | `kani::proofs::net::netfilter::ipset::core::refcount_invariants` |

### Layer 4: Functional correctness (declared in `net/netfilter/ipset/00-overview.md` Layer 4)

- **Atomic-swap soundness theorem** via Verus — proves: IPSET_CMD_SWAP is atomic from concurrent test/lookup's perspective.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-set + per-type refcounts use `Refcount` (saturating) | § Mandatory |
| **AUTOSLAB** | per-set slab cache | § Mandatory |
| **CONSTIFY** | per-type `ip_set_type` instances + per-variant `ip_set_type_variant` `static const` | § Mandatory |
| **MEMORY_SANITIZE** | freed sets + per-element ext data cleared (carries selectors + comments) | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, CONSTIFY, MEMORY_SANITIZE**: see above
- **USERCOPY**: NLA parsing uses bound-checked accessors
- **SIZE_OVERFLOW**: per-element timeout + counter + extension-offset arithmetic uses checked operators
- **KERNEXEC**: per-IPSET_CMD_* dispatch via `static const fn-ptr` array

### Row-2 / GR-RBAC integration

- LSM hook `security_capable(CAP_NET_ADMIN)` already required for IPSET_CMD_CREATE / DESTROY / ADD / DEL / FLUSH / SWAP.
- Default useful GR-RBAC policy: deny ipset mutations outside gradm-marked `firewall_admin` role.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Open Questions

(none — ipset core ABI exhaustively specified by upstream + libipset regression-test corpus)

## Out of Scope

- Per-set-type families (cross-ref `net/netfilter/ipset/types-bitmap.md`, `types-hash.md`, `types-list.md`)
- 32-bit userspace compat — DEFERRED for v0 (REQ-12)
- 32-bit-only paths
- Implementation code
