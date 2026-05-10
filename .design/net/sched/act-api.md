# Tier-3: net/sched/act-api — action framework (RTM_*TACT, action sharing, action graph)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/sched/act_api.c
  - include/net/act_api.h
  - include/net/pkt_cls.h
  - include/uapi/linux/pkt_cls.h
-->

## Summary
Tier-3 design for the tc action framework: per-system `tc_action_ops` registry (one entry per action type — mirred, gact, police, …), per-action-instance state (`struct tc_action`), action sharing across multiple classifiers (per-action `tcfa_index` pulled from a per-netns IDR with refcount-based sharing), the `tcf_action_exec` invocation path called by classifiers after a match, the `RTM_NEWACTION / DELACTION / GETACTION` NETLINK_ROUTE wire format, the multi-part dump protocol for action listing, and the action-goto-chain mechanism that lets actions instruct the classifier framework to jump to a different chain.

Actions are invoked sequentially in a per-classifier-result `tcf_exts` array (an array of `tc_action *`). After one action returns TC_ACT_PIPE, the next runs; TC_ACT_OK / SHOT / STOLEN / REDIRECT terminate; TC_ACT_GOTO_CHAIN restarts at chain N.

Sub-tier-3 of `net/sched/00-overview.md`. Pairs with `net/sched/cls-api.md` (consumer; classifiers populate `tcf_exts`), per-action Tier-3s (`act-mirred.md`, `act-police.md`, `act-ct.md`, …).

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Action registry, sharing IDR, `tcf_action_exec`, RTM_*TACT handlers, dump | `net/sched/act_api.c` |
| Public API | `include/net/act_api.h`, `include/net/pkt_cls.h` (tcf_exts) |
| UAPI: TCA_ACT_* attrs, TC_ACT_* constants | `include/uapi/linux/pkt_cls.h` |

## Compatibility contract

### `tcf_register_action` / `tcf_unregister_action`

```c
int tcf_register_action(struct tc_action_ops *act, struct pernet_operations *ops);
int tcf_unregister_action(struct tc_action_ops *act, struct pernet_operations *ops);
```

Per-system + per-netns linked-list of `tc_action_ops`. Each `act` has unique `kind[IFNAMSIZ]` (e.g., `"mirred"`); duplicate-register fails -EEXIST. Per-netns ops alongside system-wide registration.

### `struct tc_action_ops` vtable

`include/net/act_api.h`:
```c
struct tc_action_ops {
    struct list_head head;
    char kind[IFNAMSIZ];
    enum tca_id id;
    unsigned int net_id;
    size_t size;
    struct module *owner;
    int (*act)(struct sk_buff *, const struct tc_action *, struct tcf_result *);
    int (*dump)(struct sk_buff *, struct tc_action *, int, int);
    void (*cleanup)(struct tc_action *);
    int (*lookup)(struct net *, struct tc_action **, u32);
    int (*init)(struct net *, struct nlattr *, struct nlattr *, struct tc_action **, struct tcf_proto *, u32, struct netlink_ext_ack *);
    int (*walk)(struct net *, struct sk_buff *, struct netlink_callback *, int, const struct tc_action_ops *, struct netlink_ext_ack *);
    void (*stats_update)(struct tc_action *, u64, u64, u64, u64, bool);
    size_t (*get_fill_size)(const struct tc_action *);
    struct net_device *(*get_dev)(const struct tc_action *, netdevice_tracker *);
    int (*offload_act_setup)(struct tc_action *, void *, u32 *, bool, struct netlink_ext_ack *);
};
```

Layout-equivalent for first cache-line.

### `struct tc_action` (per-instance)

```c
struct tc_action {
    const struct tc_action_ops *ops;
    __u32  type;
    struct tcf_idrinfo *idrinfo;
    u32 tcfa_index;
    refcount_t tcfa_refcnt;
    atomic_t tcfa_bindcnt;
    int tcfa_action;
    struct tcf_t tcfa_tm;
    struct gnet_stats_basic_sync tcfa_bstats;
    struct gnet_stats_basic_sync tcfa_bstats_hw;
    struct gnet_stats_queue tcfa_qstats;
    struct net_rate_estimator __rcu *tcfa_rate_est;
    spinlock_t tcfa_lock;
    struct gnet_stats_basic_sync __percpu *cpu_bstats;
    struct gnet_stats_basic_sync __percpu *cpu_bstats_hw;
    struct gnet_stats_queue __percpu *cpu_qstats;
    struct tc_cookie __rcu *act_cookie;
    struct tcf_chain __rcu *goto_chain;
    u32 tcfa_flags;
    u8 hw_stats;
    u8 used_hw_stats;
    bool used_hw_stats_valid;
    u32 in_hw_count;
};
```

Layout-equivalent for first cache-line.

### Per-action sharing (`tcfa_refcnt` + `tcfa_bindcnt`)

Two refcounts per-action:
- `tcfa_refcnt`: total references including direct ownership + classifier bindings
- `tcfa_bindcnt`: subset of `tcfa_refcnt` covering classifier bindings specifically

Action with `tcfa_index = N` can be referenced from multiple `tcf_exts` arrays (action sharing). Created once via RTM_NEWACTION; classifiers refer by index. Identical sharing model.

### `tcf_action_exec` invocation

Called by classifier on match:
```c
int tcf_action_exec(struct sk_buff *skb, struct tc_action *const *actions,
                    int nr_actions, struct tcf_result *res);
```

Walks `actions[0..nr_actions]`, invokes each `act->ops->act(skb, act, res)`:
- Returns TC_ACT_PIPE → continue with next action
- Returns TC_ACT_OK / SHOT / STOLEN / REDIRECT / TRAP → stop, return that
- Returns TC_ACT_GOTO_CHAIN → set `res->goto_tp` from `act->goto_chain`, return TC_ACT_GOTO_CHAIN to caller

Identical sequence.

### `RTM_NEWACTION / DELACTION / GETACTION` wire format

NLA attributes:
- `TCA_ACT_KIND` (action kind string, e.g., `"mirred"`)
- `TCA_ACT_OPTIONS` (nested per-action-specific attrs)
- `TCA_ACT_INDEX` (action's `tcfa_index` in shared-action IDR; 0 → assign)
- `TCA_ACT_STATS`, `TCA_ACT_XSTATS`
- `TCA_ACT_HW_STATS` (HW offload counters)
- `TCA_ACT_HW_STATS_TYPE`
- `TCA_ACT_USED_HW_STATS`
- `TCA_ACT_FLAGS`
- `TCA_ACT_PAD`
- `TCA_ACT_COOKIE` (per-action opaque cookie for userspace)
- `TCA_ACT_DUMP_FLAGS`
- `TCA_ACT_EXT_WARN_MSG`

Wire format byte-identical so iproute2 + tc-actions tooling work unchanged.

### Per-action goto-chain

Action can specify `tcfa_action = TC_ACT_GOTO_CHAIN | (chain_id << TC_ACT_EXT_SHIFT)`. On execution, classifier's `tcf_classify` jumps to that chain. Identical mechanism.

### Per-netns IDR for action-by-index lookup

Per-netns `tcf_idrinfo` maps `tcfa_index → tc_action`. Per-action ops register their own per-netns IDR via `pernet_operations`. RTM_GETACTION with `TCA_ACT_INDEX=N` returns the action.

Identical IDR model.

### Multi-part dump

`RTM_GETACTION` with `NLM_F_DUMP` walks the per-netns IDR, returning multi-part replies per-action with `NLM_F_MULTI` + `NLMSG_DONE` final. Identical.

### HW offload via `offload_act_setup`

Per-action `offload_act_setup` callback negotiates per-action HW offload with driver (per `flow_action` API). Identical contract.

## Requirements

- REQ-1: `tcf_register_action` / `tcf_unregister_action` semantics identical: per-system + per-netns `tc_action_ops` linked-list; duplicate `kind` rejected with -EEXIST.
- REQ-2: `struct tc_action_ops` first-cache-line + per-method-pointer layout-equivalent.
- REQ-3: `struct tc_action` first-cache-line layout-equivalent.
- REQ-4: Per-action sharing: `tcfa_refcnt` + `tcfa_bindcnt` track total + classifier-binding refs respectively; identical accounting.
- REQ-5: `tcf_action_exec` walk: per-action invocation; TC_ACT_* dispatch identical.
- REQ-6: `RTM_NEWACTION / DELACTION / GETACTION` wire format byte-identical (TCA_ACT_KIND/OPTIONS/INDEX/STATS/HW_STATS/HW_STATS_TYPE/USED_HW_STATS/FLAGS/PAD/COOKIE/DUMP_FLAGS/EXT_WARN_MSG NLAs).
- REQ-7: Per-action goto-chain: `tcfa_action = TC_ACT_GOTO_CHAIN | (chain_id << TC_ACT_EXT_SHIFT)` parsed; chain pointer resolved + cached in `goto_chain`.
- REQ-8: Per-netns IDR for action-by-index lookup; `pernet_operations` registration per-action-type.
- REQ-9: Multi-part dump for RTM_GETACTION; walker cursor identical.
- REQ-10: HW offload: `offload_act_setup` callback bridges to driver `flow_action` API; per-action `in_hw_count` + `hw_stats` reported back.
- REQ-11: Per-action `cpu_bstats` / `cpu_qstats` per-CPU stats infrastructure; aggregated in `tcfa_bstats` / `tcfa_qstats` for export via TCA_ACT_STATS.
- REQ-12: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `pahole struct tc_action_ops` + `struct tc_action` byte-identical first cache-line. (covers REQ-2, REQ-3)
- [ ] AC-2: `tc actions add action mirred egress redirect dev eth1` test: NLA-encoded RTM_NEWACTION byte-identical to upstream's; action listed via `tc actions list action mirred`. (covers REQ-1, REQ-6)
- [ ] AC-3: Action-sharing test: install action with `TCA_ACT_INDEX=0` → kernel assigns N; install second classifier filter referencing index N; both filters share the same action; `tcfa_refcnt = 2`, `tcfa_bindcnt = 2`. (covers REQ-4)
- [ ] AC-4: tcf_action_exec test: install filter with mirred + gact actions; on match → mirred runs (returns TC_ACT_PIPE), then gact runs (returns TC_ACT_SHOT, drop). (covers REQ-5)
- [ ] AC-5: Goto-chain action test: install gact action with `tcfa_action = TC_ACT_GOTO_CHAIN | (5 << TC_ACT_EXT_SHIFT)`; on classification match, classifier jumps to chain 5. (covers REQ-7)
- [ ] AC-6: Per-netns IDR test: action with index N in netns A; netns B's RTM_GETACTION with index=N returns ENOENT. (covers REQ-8)
- [ ] AC-7: Multi-part dump test: 100 actions installed; `tc actions list action mirred` returns all 100; multi-part replies. (covers REQ-9)
- [ ] AC-8: HW-offload test on supporting NIC: install mirred action with HW offload flag; `tc actions list` shows `in_hw_count` increment after offload. (covers REQ-10)
- [ ] AC-9: Per-CPU stats test: under load, per-CPU `cpu_bstats` aggregate sum equals `tcfa_bstats.bytes` returned via TCA_ACT_STATS. (covers REQ-11)
- [ ] AC-10: Hardening section present and follows template. (covers REQ-12)

## Architecture

### Rust module organization

- `kernel::net::sched::act_api::ActionRegistry` — per-system + per-netns tc_action_ops list
- `kernel::net::sched::act_api::ActionOps` — `struct tc_action_ops` wrapper
- `kernel::net::sched::act_api::TcAction` — `struct tc_action` wrapper
- `kernel::net::sched::act_api::Sharing` — tcfa_refcnt / tcfa_bindcnt accounting
- `kernel::net::sched::act_api::Idr` — per-netns IDR for action-by-index
- `kernel::net::sched::act_api::Exec` — `tcf_action_exec` walk + dispatch
- `kernel::net::sched::act_api::GotoChain` — `goto_chain` pointer mgmt
- `kernel::net::sched::act_api::netlink::RtmTact` — RTM_*TACT handlers
- `kernel::net::sched::act_api::dump::Dump` — multi-part dump
- `kernel::net::sched::act_api::offload::OffloadActSetup` — `offload_act_setup` bridge
- `kernel::net::sched::act_api::stats::PerCpuStats` — per-CPU bstats/qstats

### Locking and concurrency

- **`tcf_action_mod_lock`** (mutex): per-system tc_action_ops registry mutator
- **Per-netns `tcf_idrinfo->lock`** (spinlock): per-netns IDR mutator
- **Per-`tc_action` `tcfa_lock`** (spinlock): per-action mutable state (cookie, stats config)
- **RCU**: action-exec hot path is RCU-side; mutators use RCU-defer free

### Error handling

- `Err(EEXIST)` — duplicate-register / RTM_NEWACTION for existing index
- `Err(ENOENT)` — RTM_DELACTION / RTM_GETACTION for non-existent
- `Err(EINVAL)` — bad NLA / unknown action kind
- `Err(EBUSY)` — RTM_DELACTION while bindcnt > 0
- `Err(EPERM)` — non-CAP_NET_ADMIN

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Per-system tc_action_ops list insert/erase under tcf_action_mod_lock | `kani::proofs::net::sched::act_api::registry_safety` |
| Per-netns IDR alloc/free | `kani::proofs::net::sched::act_api::idr_safety` |
| `tcf_action_exec` walk with TC_ACT_GOTO_CHAIN handling | `kani::proofs::net::sched::act_api::exec_safety` |
| RTM_NEWACTION NLA parser | `kani::proofs::net::sched::act_api::nla_parse_safety` |
| Multi-part dump cursor | `kani::proofs::net::sched::act_api::dump_safety` |
| Per-action refcount + bindcnt arithmetic (no underflow on UPDPOLICY-replace) | `kani::proofs::net::sched::act_api::refcount_safety` |

### Layer 2: TLA+ models

(none new owned at this granularity; classifier+action sequencing covered by `models/net/cls_chain_eval.tla` from `cls-api.md`)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-system tc_action_ops list | every entry's `kind[IFNAMSIZ]` is unique | `kani::proofs::net::sched::act_api::registry_invariants` |
| Per-netns IDR | every action's `tcfa_index` is unique within netns | `kani::proofs::net::sched::act_api::idr_invariants` |
| Per-action `tcfa_bindcnt` | `tcfa_bindcnt ≤ tcfa_refcnt` always | `kani::proofs::net::sched::act_api::bindcnt_invariants` |
| Per-action `goto_chain` | when set, points to a valid chain in the same block | `kani::proofs::net::sched::act_api::goto_invariants` |

### Layer 4: Functional correctness (opt-in)

- **tcf_action_exec sequencing soundness theorem** via Verus — proves: per-skb action sequence is deterministic; TC_ACT_PIPE actions execute in array order; TC_ACT_GOTO_CHAIN terminates the local action sequence.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-tc_action `tcfa_refcnt` + `tcfa_bindcnt` use `Refcount` (saturating) | § Mandatory |
| **AUTOSLAB** | per-tc_action slab cache; per-action-type slab caches | § Mandatory |
| **CONSTIFY** | `tc_action_ops` vtables `static const` | § Mandatory |
| **MEMORY_SANITIZE** | freed tc_action state cleared (cookies + per-action config may carry sensitive selectors) | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, CONSTIFY, MEMORY_SANITIZE**: see above
- **USERCOPY**: NLA parsing uses bound-checked accessors
- **SIZE_OVERFLOW**: per-CPU stats aggregation arithmetic uses checked operators
- **KERNEXEC**: per-action dispatch via `static const fn-ptr` array per `tc_action_ops`

### Row-2 / GR-RBAC integration

- LSM hook `security_capable(CAP_NET_ADMIN)` already required for all RTM_*TACT mutations.
- LSM hook `security_capable(CAP_BPF)` for `act_bpf` action installation.
- Default useful GR-RBAC policy: deny tc-action mutation outside gradm-marked `network_admin` role.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Open Questions

(none — action API is exhaustively specified by upstream + iproute2 wire-test corpus)

## Out of Scope

- Per-action implementations (cross-ref `net/sched/act-mirred.md`, `act-police.md`, `act-ct.md`, etc.)
- Classifier framework (cross-ref `net/sched/cls-api.md`)
- Qdisc framework (cross-ref `net/sched/sch-api.md`)
- 32-bit-only paths
- Implementation code
