---
title: "Tier-3: net/sched/sch-api — qdisc API (register / lookup / RTM_*QDISC + RTM_*TCLASS wire)"
tags: ["design-doc", "tier-3", "net"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for the qdisc framework's user-facing API: per-system `Qdisc_ops` registry (one entry per qdisc type — fq, fq_codel, htb, …), per-netdev qdisc-tree mutation primitives (`qdisc_create`, `qdisc_destroy`, `qdisc_graft`, `qdisc_change`), per-class mutation (`tc_class_*`), the `RTM_NEWQDISC / DELQDISC / GETQDISC` and `RTM_NEWTCLASS / DELTCLASS / GETTCLASS` NETLINK_ROUTE wire format, and the multi-part dump protocol for tree walks.

The "front door" for every qdisc and class operation: every `tc qdisc add` / `tc class add` lands here before per-qdisc `init/change/attach` callbacks fire. Sub-tier-3 of `net/sched/00-overview.md`. Pairs with `net/sched/sch-generic.md` (per-qdisc generic helpers + default qdiscs), `net/sched/cls-api.md` (filter framework), per-qdisc Tier-3s (sch-fq.md, sch-htb.md, etc.).

### Requirements

- REQ-1: `register_qdisc` / `unregister_qdisc` semantics identical: per-system `Qdisc_ops` linked-list; duplicate `id` rejected with -EEXIST.
- REQ-2: `struct Qdisc_ops` first-cache-line + per-method-pointer layout-equivalent.
- REQ-3: `struct Qdisc_class_ops` first-cache-line layout-equivalent.
- REQ-4: `RTM_NEWQDISC / DELQDISC / GETQDISC` wire format byte-identical (TCA_KIND / OPTIONS / STATS / RATE / FCNT / STAB / PAD / DUMP_INVISIBLE / CHAIN / HW_OFFLOAD / INGRESS_BLOCK / EGRESS_BLOCK / DUMP_FLAGS / EXT_WARN_MSG).
- REQ-5: `RTM_NEWTCLASS / DELTCLASS / GETTCLASS` wire format byte-identical.
- REQ-6: `qdisc_create` / `qdisc_destroy` / `qdisc_graft` / `qdisc_change` semantics identical; per-qdisc `init/change/destroy/attach` callbacks invoked at correct points.
- REQ-7: Per-class `tc_class_*` mutation; for non-classful qdiscs, RTM_NEWTCLASS returns EINVAL.
- REQ-8: Multi-part dump protocol for RTM_GETQDISC + RTM_GETTCLASS with per-walker cursor.
- REQ-9: `qdisc_create_dflt`: per-netdev default qdisc allocated from `net.core.default_qdisc` sysctl (default `fq_codel`).
- REQ-10: Rate estimator (`gen_estimator`) per-qdisc + per-class via `TCA_RATE` NLA; sliding-window EWMA algorithm identical.
- REQ-11: `tcf_block` shared-block infrastructure: per-netns IDR allocator; ref-counted; multiple qdiscs can share via `TCA_INGRESS_BLOCK` / `TCA_EGRESS_BLOCK`.
- REQ-12: HW-offload flag `TCA_HW_OFFLOAD`: passes through to driver-side; reported back via `RTM_GETQDISC` after offload attempt.
- REQ-13: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: `pahole struct Qdisc_ops` + `struct Qdisc_class_ops` byte-identical first cache-line. (covers REQ-2, REQ-3)
- [ ] AC-2: `tc qdisc add dev eth0 root fq_codel limit 1000` test: NLA-encoded RTM_NEWQDISC byte-identical to upstream's; qdisc visible in `tc qdisc show dev eth0`. (covers REQ-1, REQ-4, REQ-6)
- [ ] AC-3: `tc class add dev eth0 parent 1:0 classid 1:10 htb rate 10mbit` test: NLA-encoded RTM_NEWTCLASS byte-identical; class visible. (covers REQ-5, REQ-7)
- [ ] AC-4: Dump test: 100 qdiscs across 100 netdevs; `tc -s qdisc show` returns all; multi-part replies with NLM_F_MULTI + NLMSG_DONE final. (covers REQ-8)
- [ ] AC-5: Default qdisc test: `sysctl net.core.default_qdisc=pfifo_fast`; create new netdev → default is pfifo_fast; reset to fq_codel → new netdev is fq_codel. (covers REQ-9)
- [ ] AC-6: Rate-estimator test: install qdisc with `TCA_RATE` ewma_log=3 + interval=1s; send 1Mbps traffic; `tc -s qdisc` shows ~1Mbps reported in RATE stats. (covers REQ-10)
- [ ] AC-7: Shared-block test: 2 qdiscs on different netdevs share `TCA_INGRESS_BLOCK=42`; install filter on block 42 once → applies to both qdiscs' ingress. (covers REQ-11)
- [ ] AC-8: HW-offload test on supporting NIC: install qdisc with `TCA_HW_OFFLOAD`; `RTM_GETQDISC` returns the flag asserted; driver-counters increment on traffic. (covers REQ-12)
- [ ] AC-9: Duplicate-register test: load module providing qdisc id "fq_codel" while it's already registered → returns -EEXIST. (covers REQ-1)
- [ ] AC-10: Hardening section present and follows template. (covers REQ-13)

### Architecture

### Rust module organization

- `kernel::net::sched::sch_api::QdiscRegistry` — per-system Qdisc_ops list
- `kernel::net::sched::sch_api::QdiscOps` — `struct Qdisc_ops` wrapper + vtable
- `kernel::net::sched::sch_api::QdiscClassOps` — `struct Qdisc_class_ops` wrapper
- `kernel::net::sched::sch_api::Mutate` — qdisc_create/destroy/graft/change
- `kernel::net::sched::sch_api::ClassMutate` — tc_class_*
- `kernel::net::sched::sch_api::netlink::RtmQdisc` — RTM_*QDISC handlers
- `kernel::net::sched::sch_api::netlink::RtmTclass` — RTM_*TCLASS handlers
- `kernel::net::sched::sch_api::dump::Dump` — multi-part dump protocol
- `kernel::net::sched::sch_api::estimator::RateEstimator` — gen_estimator
- `kernel::net::sched::sch_api::block::TcfBlock` — shared-block IDR
- `kernel::net::sched::sch_api::default::DefaultQdisc` — qdisc_create_dflt

### Locking and concurrency

- **`qdisc_mod_lock`** (mutex): per-system Qdisc_ops registry mutator
- **Per-netdev `qdisc_tree_lock`** (rtnl_lock-class): per-netdev tree mutation; held during qdisc_graft etc.
- **Per-netdev `qdisc_sleeping->q.lock`** (per-qdisc spinlock): protects per-qdisc state during enqueue/dequeue
- **RCU**: hot-path lookups (qdisc_lookup_rcu, tc_class_walk_rcu) RCU-side

### Error handling

- `Err(EEXIST)` — duplicate-register / RTM_NEWQDISC for existing parent:handle
- `Err(ENOENT)` — RTM_DELQDISC / RTM_GETQDISC for non-existent
- `Err(EINVAL)` — bad NLA / non-classful qdisc + RTM_NEWTCLASS
- `Err(EOPNOTSUPP)` — qdisc doesn't support requested feature
- `Err(EPERM)` — non-CAP_NET_ADMIN
- `Err(ENOMEM)` — alloc fail

### Out of Scope

- Per-qdisc implementations (cross-ref `net/sched/sch-fq.md`, `sch-htb.md`, etc.)
- Classifier framework (cross-ref `net/sched/cls-api.md`)
- Action framework (cross-ref `net/sched/act-api.md`)
- 32-bit-only paths
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| qdisc registry, mutation primitives, RTM_*QDISC + RTM_*TCLASS handlers, dump | `net/sched/sch_api.c` |
| Public API | `include/net/sch_generic.h`, `include/net/pkt_sched.h` |
| UAPI: TCA_* attrs, struct tc_qopt, struct tc_estimator | `include/uapi/linux/pkt_sched.h`, `include/uapi/linux/rtnetlink.h` |

### compatibility contract

### `register_qdisc` / `unregister_qdisc`

Per-system `Qdisc_ops` linked-list. Registration on module load; unregistration on module unload.

```c
int register_qdisc(struct Qdisc_ops *qops);
int unregister_qdisc(struct Qdisc_ops *qops);
```

Each `Qdisc_ops` has unique `id[IFNAMSIZ]` (e.g., `"fq_codel"`); duplicate-register fails with -EEXIST. Identical contract.

### `struct Qdisc_ops` vtable

`include/net/sch_generic.h`:
```c
struct Qdisc_ops {
    struct Qdisc_ops *next;
    const struct Qdisc_class_ops *cl_ops;
    char id[IFNAMSIZ];
    int priv_size;
    unsigned int static_flags;
    int (*enqueue)(...);
    struct sk_buff *(*dequeue)(...);
    struct sk_buff *(*peek)(...);
    int (*init)(...);
    void (*reset)(...);
    void (*destroy)(...);
    int (*change)(...);
    void (*attach)(...);
    int (*change_tx_queue_len)(...);
    int (*change_real_num_tx)(...);
    int (*dump)(...);
    int (*dump_stats)(...);
    void (*ingress_block_set)(...);
    void (*egress_block_set)(...);
    u32 (*ingress_block_get)(...);
    u32 (*egress_block_get)(...);
    struct module *owner;
};
```

Layout-equivalent for first cache-line.

### `struct Qdisc_class_ops` vtable (per-class operations)

For classful qdiscs (htb, mqprio, drr): per-class lookup + walk + attach. Layout-equivalent.

### `RTM_NEWQDISC / DELQDISC / GETQDISC` wire format

NLA attributes per `TCA_*` enum:
- `TCA_KIND` (qdisc id string, e.g., `"fq_codel"`)
- `TCA_OPTIONS` (nested per-qdisc-specific attrs)
- `TCA_STATS` / `TCA_XSTATS` (read-only)
- `TCA_RATE` (rate estimator config)
- `TCA_FCNT` (block count)
- `TCA_STATS2` (extended stats)
- `TCA_STAB` (stab/qsd table)
- `TCA_PAD`
- `TCA_DUMP_INVISIBLE` (skip in dumps)
- `TCA_CHAIN` (block chain index)
- `TCA_HW_OFFLOAD` (HW-offload flag)
- `TCA_INGRESS_BLOCK` / `TCA_EGRESS_BLOCK` (shared-block IDs)
- `TCA_DUMP_FLAGS`
- `TCA_EXT_WARN_MSG`

Wire format byte-identical so iproute2 + bcc-tools' tc-bind work unchanged.

### `RTM_NEWTCLASS / DELTCLASS / GETTCLASS` wire format

NLA attributes mirror `RTM_*QDISC` (TCA_KIND/OPTIONS/STATS/RATE/etc.) plus class-specific selectors. Wire format byte-identical.

### Multi-part dump protocol

`RTM_GETQDISC` and `RTM_GETTCLASS` walks support `NLM_F_DUMP` for full-tree dump; per-walker cursor across multi-part replies with `NLM_F_MULTI` + final `NLMSG_DONE`. Identical multi-part protocol.

### `qdisc_create_dflt` (default qdisc construction)

Per-netdev default qdisc allocation when no explicit qdisc is set; default differs per kernel sysctl `net.core.default_qdisc` (default `fq_codel` since 4.12; configurable to `pfifo_fast` for legacy). Identical default selection.

### Rate estimator (`gen_estimator`)

Per-qdisc + per-class optional rate-estimator backed by `gnet_estimator` + sliding-window EWMA. Per `TCA_RATE`. Identical algorithm.

### Per-shared-block infrastructure

Multiple qdiscs can share a `tcf_block` for filter consolidation (e.g., shared ingress block across many netdevs). Block IDs allocated from per-netns IDR; ref-counted across consuming qdiscs.

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Per-system Qdisc_ops list insert/erase under qdisc_mod_lock | `kani::proofs::net::sched::sch_api::registry_safety` |
| Per-netdev qdisc-tree graft (refcount sound across attach) | `kani::proofs::net::sched::sch_api::graft_safety` |
| RTM_NEWQDISC NLA parser (per-attribute bounds-check) | `kani::proofs::net::sched::sch_api::nla_parse_safety` |
| Multi-part dump cursor (no out-of-bounds walker advance) | `kani::proofs::net::sched::sch_api::dump_safety` |
| Rate-estimator EWMA arithmetic (no overflow on 64-bit byte counter) | `kani::proofs::net::sched::sch_api::estimator_safety` |
| TcfBlock IDR alloc/free | `kani::proofs::net::sched::sch_api::block_safety` |

### Layer 2: TLA+ models

(none new owned at this granularity; per-qdisc Tier-3s declare per-qdisc TLA+ models)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-system Qdisc_ops list | every entry's `id[IFNAMSIZ]` is unique | `kani::proofs::net::sched::sch_api::registry_invariants` |
| Per-netdev qdisc-tree | every non-root qdisc has a non-NULL parent; tree depth bounded | `kani::proofs::net::sched::sch_api::tree_invariants` |
| Per-block ref count | matches Σ qdiscs holding it | `kani::proofs::net::sched::sch_api::block_invariants` |

### Layer 4: Functional correctness (opt-in)

(deferred — same gate as `net/sched/00-overview.md`)

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-Qdisc, per-Block, per-class refcounts use `Refcount` (saturating) | § Mandatory |
| **AUTOSLAB** | per-Qdisc + per-Block slab caches | § Mandatory |
| **CONSTIFY** | `Qdisc_ops` + `Qdisc_class_ops` vtables `static const` | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, CONSTIFY**: see above
- **MEMORY_SANITIZE**: freed Qdisc state cleared (qdiscs may queue in-flight skbs with sensitive payload)
- **USERCOPY**: NLA parsing uses bound-checked accessors
- **SIZE_OVERFLOW**: rate-table + IDR-arithmetic uses checked operators
- **KERNEXEC**: per-Qdisc dispatch via `static const fn-ptr` array per Qdisc_ops

### Row-2 / GR-RBAC integration

- LSM hook `security_capable(CAP_NET_ADMIN)` already required for all RTM_*QDISC + RTM_*TCLASS mutations.
- Default useful GR-RBAC policy: deny qdisc/class mutation outside gradm-marked `network_admin` role.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

