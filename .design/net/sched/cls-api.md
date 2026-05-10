# Tier-3: net/sched/cls-api — classifier API (chain / block / RTM_*TFILTER + RTM_*CHAIN wire)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/sched/cls_api.c
  - include/net/pkt_cls.h
  - include/net/sch_generic.h
  - include/uapi/linux/pkt_cls.h
  - include/uapi/linux/rtnetlink.h
-->

## Summary
Tier-3 design for the tc classifier framework: per-system `tcf_proto_ops` registry (one entry per classifier type — u32, flower, bpf, …), per-block `tcf_chain` priority list, per-chain `tcf_proto` filter list, the `tcf_classify` hot-path that walks chains in order and returns a per-packet `tcf_result` (which an action then consumes), and the `RTM_NEWTFILTER / DELTFILTER / GETTFILTER / NEWCHAIN / DELCHAIN / GETCHAIN` NETLINK_ROUTE wire format.

Classifiers run during `__qdisc_run_begin` (or directly under `__netif_receive_skb_core` for clsact ingress). Per-classifier `classify` hook returns one of: TC_ACT_OK (accept; result populated), TC_ACT_SHOT (drop), TC_ACT_UNSPEC (no match; fall through to next classifier in the same chain), TC_ACT_RECLASSIFY (start chain over with new context), TC_ACT_PIPE (continue with action chain), TC_ACT_TRAP (cooperative-pull to userspace).

Sub-tier-3 of `net/sched/00-overview.md`. Owns the mandatory `models/net/cls_chain_eval.tla` TLA+ model. Pairs with `net/sched/sch-api.md` (qdisc framework), `net/sched/act-api.md` (action framework that classifiers populate), per-classifier Tier-3s (cls-u32.md, cls-flower.md, cls-bpf.md).

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Classifier registry, chain/block primitives, `tcf_classify` hot path, RTM_*TFILTER + RTM_*CHAIN handlers, dump | `net/sched/cls_api.c` |
| Public API | `include/net/pkt_cls.h`, `include/net/sch_generic.h` (tcf_block, tcf_chain, tcf_proto types) |
| UAPI: TCA_*, TC_ACT_* constants | `include/uapi/linux/pkt_cls.h`, `include/uapi/linux/rtnetlink.h` |

## Compatibility contract

### `tcf_register_proto` / `tcf_unregister_proto`

```c
int tcf_register_proto(struct tcf_proto_ops *ops);
int tcf_unregister_proto(struct tcf_proto_ops *ops);
```

Per-system `tcf_proto_ops` linked-list. Each `ops` has unique `kind[IFNAMSIZ]` (e.g., `"flower"`); duplicate-register fails -EEXIST.

### `struct tcf_proto_ops` vtable

`include/net/pkt_cls.h`:
```c
struct tcf_proto_ops {
    struct list_head next;
    char kind[IFNAMSIZ];
    int (*classify)(struct sk_buff *, const struct tcf_proto *, struct tcf_result *);
    int (*init)(struct tcf_proto *);
    void (*destroy)(struct tcf_proto *, bool, struct netlink_ext_ack *);
    void *(*get)(struct tcf_proto *, u32);
    int (*change)(...);
    int (*delete)(...);
    bool (*delete_empty)(...);
    void (*walk)(...);
    int (*reoffload)(...);
    void (*hw_add)(...);
    void (*hw_del)(...);
    void (*bind_class)(...);
    void *(*tmplt_create)(...);
    void (*tmplt_destroy)(...);
    void (*tmplt_reoffload)(...);
    int (*tmplt_dump)(...);
    struct module *owner;
    int flags;
};
```

Layout-equivalent for first cache-line.

### `struct tcf_chain` and `struct tcf_block`

`include/net/sch_generic.h`:
- `tcf_chain` per-priority filter list within a block; `index` (chain ID); list of `tcf_proto` per priority
- `tcf_block` shared filter group; can be referenced by multiple qdiscs via `TCA_INGRESS_BLOCK` / `TCA_EGRESS_BLOCK`; carries `chains` list + per-block lock

Layout-equivalent.

### `struct tcf_proto` (per-instance filter state)

```c
struct tcf_proto {
    struct tcf_proto __rcu *next;
    void __rcu       *root;
    int (*classify)(...);
    void (*bind_class)(...);
    __be16  protocol;
    u32     prio;
    void   *data;
    const struct tcf_proto_ops *ops;
    struct tcf_chain *chain;
    spinlock_t lock;
    bool deleting;
    refcount_t refcnt;
    struct rcu_head rcu;
    struct hlist_node destroy_ht_node;
};
```

Layout-equivalent.

### `tcf_classify` hot path

For each skb entering ingress or qdisc enqueue:
1. Walk per-priority `tcf_proto` list within the active chain
2. For each: invoke `proto->classify(skb, proto, &result)`
3. On `TC_ACT_OK / SHOT / RECLASSIFY / PIPE / TRAP / STOLEN / QUEUED / REDIRECT`: stop walk, return result
4. On `TC_ACT_UNSPEC`: continue to next `proto` in same priority order
5. On `TC_ACT_GOTO_CHAIN`: jump to chain `result.chain` and restart walk
6. End-of-chain with no match: TC_ACT_UNSPEC bubbles up (qdisc treats as no-classify)

Identical algorithm so action / drop / mirror behavior is byte-equivalent.

### `RTM_NEWTFILTER / DELTFILTER / GETTFILTER` wire format

NLA attributes:
- `TCA_KIND` (classifier kind, e.g., `"flower"`)
- `TCA_OPTIONS` (nested per-classifier-specific attrs)
- `TCA_RATE` (rate estimator)
- `TCA_FCNT`, `TCA_STATS`, `TCA_XSTATS`
- `TCA_CHAIN` (chain ID within block)
- `TCA_HW_OFFLOAD`
- `TCA_DUMP_FLAGS`
- `TCA_EXT_WARN_MSG`

Wire format byte-identical so iproute2 + tc-ebpf libraries work.

### `RTM_NEWCHAIN / DELCHAIN / GETCHAIN` wire format

NLA attributes for per-chain template (e.g., flower template applied to all filters in a chain):
- `TCA_CHAIN` (chain ID)
- `TCA_KIND` (template-classifier kind)
- `TCA_OPTIONS` (nested per-classifier template attrs)
- `TCA_DUMP_FLAGS`

Wire format byte-identical.

### TC action constants (`TC_ACT_*`)

| Constant | Value | Semantic |
|---|---|---|
| `TC_ACT_UNSPEC` | -1 | no match; fall through |
| `TC_ACT_OK` | 0 | classify match; result populated |
| `TC_ACT_RECLASSIFY` | 1 | restart classification with new context |
| `TC_ACT_SHOT` | 2 | drop |
| `TC_ACT_PIPE` | 3 | continue to next action |
| `TC_ACT_STOLEN` | 4 | take ownership, don't free |
| `TC_ACT_QUEUED` | 5 | queue for later |
| `TC_ACT_REPEAT` | 6 | re-run same classifier |
| `TC_ACT_REDIRECT` | 7 | redirect to another netdev |
| `TC_ACT_TRAP` | 8 | trap to userspace |
| `TC_ACT_VALUE_MAX` | … | sentinel |
| `TC_ACT_GOTO_CHAIN` | … | bit-flag for chain jump |
| `TC_ACT_EXT_VAL_MASK` | … | extended-value mask for goto-chain |
| `TC_ACT_EXT_OPCODE_MAX` | … | sentinel |

Identical bit values + semantics.

### Multi-part dump protocol

`RTM_GETTFILTER` and `RTM_GETCHAIN` walks support `NLM_F_DUMP`; per-walker cursor across multi-part replies with `NLM_F_MULTI` + `NLMSG_DONE`. Identical.

### HW offload via `tcf_block_offload_*`

Per-classifier instance can request HW offload via `TCA_HW_OFFLOAD`; classifier's `reoffload` callback negotiates with driver. Identical contract.

## Requirements

- REQ-1: `tcf_register_proto` / `tcf_unregister_proto` semantics identical: per-system list; duplicate `kind` rejected with -EEXIST.
- REQ-2: `struct tcf_proto_ops` first-cache-line + per-method-pointer layout-equivalent.
- REQ-3: `struct tcf_chain` + `struct tcf_block` + `struct tcf_proto` first-cache-line layout-equivalent.
- REQ-4: `tcf_classify` hot path: per-priority-walk + per-`classify` invocation + TC_ACT_* dispatch identical; TC_ACT_GOTO_CHAIN jump with bounded cycle detection.
- REQ-5: `RTM_NEWTFILTER / DELTFILTER / GETTFILTER` wire format byte-identical (TCA_KIND / OPTIONS / RATE / FCNT / STATS / CHAIN / HW_OFFLOAD / DUMP_FLAGS / EXT_WARN_MSG).
- REQ-6: `RTM_NEWCHAIN / DELCHAIN / GETCHAIN` wire format byte-identical.
- REQ-7: `TC_ACT_*` constants byte-identical.
- REQ-8: Multi-part dump for RTM_GETTFILTER + RTM_GETCHAIN; walker cursor identical.
- REQ-9: HW offload: `TCA_HW_OFFLOAD` flag + per-classifier `reoffload` callback + per-driver `tc_setup_cb_call`; identical contract.
- REQ-10: Per-block + per-chain ref counts via `Refcount` (saturating); destruction via RCU.
- REQ-11: Filter-replace atomicity: RTM_NEWTFILTER replacing existing filter at same (chain, prio, protocol) is atomic from `tcf_classify`'s perspective (RCU-side reads see either old or new, never partial).
- REQ-12: TLA+ model `models/net/cls_chain_eval.tla` (mandatory per `net/sched/00-overview.md` Layer 2) — proves: classifier-chain evaluation atomicity under concurrent NEWTFILTER + tcf_classify; first-match-wins; TC_ACT_GOTO_CHAIN cycles bounded.
- REQ-13: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `pahole struct tcf_proto_ops` + `struct tcf_proto` + `struct tcf_chain` + `struct tcf_block` byte-identical first cache-line. (covers REQ-2, REQ-3)
- [ ] AC-2: `tc filter add dev eth0 protocol ip parent 1: prio 1 u32 match ip dst 1.2.3.4/32 flowid 1:10` test: NLA-encoded RTM_NEWTFILTER byte-identical to upstream's; filter visible in `tc filter show`. (covers REQ-1, REQ-5)
- [ ] AC-3: Chain test: `tc chain add dev eth0 chain 42 flower template ip_proto tcp` → chain template visible via `tc chain show dev eth0`. (covers REQ-6)
- [ ] AC-4: Classify-hot-path test: install u32 filter matching dst IP; receive matching packet → tcpdump-on-classified shows TC_ACT_OK result; non-matching packet → TC_ACT_UNSPEC fall-through. (covers REQ-4)
- [ ] AC-5: GOTO_CHAIN test: chain A filter 1 → goto chain B; chain B filter 1 matches → action fires; cycle detection: goto B → goto A → bounded by MAX_RECLASSIFY_LOOP, returns drop after limit. (covers REQ-4)
- [ ] AC-6: Replace-atomicity test: install filter at (chain=0, prio=1); concurrent classify hot-path during RTM_NEWTFILTER replace → reader sees either the old or new filter, never crashes / no partial state. (covers REQ-11)
- [ ] AC-7: HW-offload test on supporting NIC: install flower filter with TCA_HW_OFFLOAD; `tc filter show` shows in-hw counter; CPU classifier path skipped per skb. (covers REQ-9)
- [ ] AC-8: Multi-part dump test: 100 filters across 10 chains; `tc filter show` returns all; multi-part replies. (covers REQ-8)
- [ ] AC-9: TLA+ `models/net/cls_chain_eval.tla` proves: classify never reads inconsistent filter state during NEWTFILTER replace; first-match-wins; goto-chain cycle bounded. (covers REQ-12)
- [ ] AC-10: Hardening section present and follows template. (covers REQ-13)

## Architecture

### Rust module organization

- `kernel::net::sched::cls_api::ProtoRegistry` — per-system tcf_proto_ops list
- `kernel::net::sched::cls_api::ProtoOps` — `struct tcf_proto_ops` wrapper
- `kernel::net::sched::cls_api::Block` — `struct tcf_block` per-shared-block
- `kernel::net::sched::cls_api::Chain` — `struct tcf_chain` per-priority chain
- `kernel::net::sched::cls_api::Proto` — `struct tcf_proto` per-filter
- `kernel::net::sched::cls_api::classify::Classify` — `tcf_classify` hot path
- `kernel::net::sched::cls_api::netlink::RtmTfilter` — RTM_*TFILTER handlers
- `kernel::net::sched::cls_api::netlink::RtmChain` — RTM_*CHAIN handlers
- `kernel::net::sched::cls_api::dump::Dump` — multi-part dump protocol
- `kernel::net::sched::cls_api::offload::HwOffload` — `tcf_block_offload_*`
- `kernel::net::sched::cls_api::tc_act::TcActConstants` — TC_ACT_* enum

### Locking and concurrency

- **`tcf_proto_mod_lock`** (mutex): per-system tcf_proto_ops registry mutator
- **Per-block `tcf_block_lock`** (mutex): per-block chains + filters mutation
- **Per-chain `filter_chain_lock`** (mutex): per-chain filter add/remove
- **Per-`tcf_proto` `lock`** (spinlock): per-filter state mutation
- **RCU**: classify hot-path reads `tcf_proto` and embedded data structures RCU-side; mutators use RCU-defer

### Error handling

- `Err(EEXIST)` — duplicate-register / RTM_NEWTFILTER for existing (chain, prio, protocol)
- `Err(ENOENT)` — RTM_DELTFILTER / RTM_GETTFILTER for non-existent
- `Err(EINVAL)` — bad NLA / unknown classifier kind
- `Err(EOPNOTSUPP)` — classifier doesn't support requested feature
- `Err(EPERM)` — non-CAP_NET_ADMIN
- `Err(ELOOP)` — TC_ACT_GOTO_CHAIN cycle exceeds MAX_RECLASSIFY_LOOP

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Per-system tcf_proto_ops list insert/erase under tcf_proto_mod_lock | `kani::proofs::net::sched::cls_api::registry_safety` |
| Per-chain filter insert/erase under filter_chain_lock | `kani::proofs::net::sched::cls_api::chain_safety` |
| `tcf_classify` per-priority walk + TC_ACT_GOTO_CHAIN handling | `kani::proofs::net::sched::cls_api::classify_safety` |
| RTM_NEWTFILTER NLA parser | `kani::proofs::net::sched::cls_api::nla_parse_safety` |
| Multi-part dump cursor (no out-of-bounds walker advance) | `kani::proofs::net::sched::cls_api::dump_safety` |
| RCU-side read of `tcf_proto` (no read-after-free during replace) | `kani::proofs::net::sched::cls_api::rcu_safety` |

### Layer 2: TLA+ models

- `models/net/cls_chain_eval.tla` (mandatory per `net/sched/00-overview.md`) — proves classifier-chain-evaluation atomicity. Owned here.

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-system tcf_proto_ops list | every entry's `kind[IFNAMSIZ]` is unique | `kani::proofs::net::sched::cls_api::registry_invariants` |
| Per-block chains list | chain index is unique within block; ordered ascending | `kani::proofs::net::sched::cls_api::chain_index_invariants` |
| Per-chain filters | priority is unique-or-tied-by-protocol; ordered ascending | `kani::proofs::net::sched::cls_api::filter_priority_invariants` |
| Per-filter `refcnt` | matches walk-side+RX-side+RTM holders | `kani::proofs::net::sched::cls_api::refcount_invariants` |

### Layer 4: Functional correctness (opt-in; declared in `net/sched/00-overview.md` Layer 4)

- **Classify-replace atomicity refinement theorem** via TLA+ refinement — proves: implementation refines the abstract atomic-replace model.
- **GOTO_CHAIN termination theorem** via Verus — proves: any classify call terminates within `MAX_RECLASSIFY_LOOP` chain hops; never infinite-loops.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-tcf_proto + per-tcf_chain + per-tcf_block refcounts use `Refcount` (saturating) | § Mandatory |
| **AUTOSLAB** | per-tcf_proto + per-chain + per-block slab caches | § Mandatory |
| **CONSTIFY** | `tcf_proto_ops` vtables `static const` | § Mandatory |
| **SIZE_OVERFLOW** | per-priority + per-chain index arithmetic uses checked operators | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, CONSTIFY, SIZE_OVERFLOW**: see above
- **MEMORY_SANITIZE**: freed tcf_proto state cleared (carries match keys including potentially-sensitive packet selectors)
- **USERCOPY**: NLA parsing uses bound-checked accessors
- **KERNEXEC**: per-classifier dispatch via `static const fn-ptr` array per `tcf_proto_ops`

### Row-2 / GR-RBAC integration

- LSM hook `security_capable(CAP_NET_ADMIN)` already required for all RTM_*TFILTER + RTM_*CHAIN mutations.
- LSM hook `security_capable(CAP_BPF)` for `cls_bpf` filter installation.
- Default useful GR-RBAC policy: deny tc-filter mutation outside gradm-marked `network_admin` role.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Open Questions

(none — classifier API is exhaustively specified by upstream + iproute2 wire-test corpus)

## Out of Scope

- Per-classifier implementations (cross-ref `net/sched/cls-u32.md`, `cls-flower.md`, `cls-bpf.md`)
- Action framework (cross-ref `net/sched/act-api.md`)
- 32-bit-only paths
- Implementation code
