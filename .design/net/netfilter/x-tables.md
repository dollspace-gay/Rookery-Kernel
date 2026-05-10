# Tier-3: net/netfilter/x-tables — legacy x_tables match/target framework

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/netfilter/x_tables.c
  - include/linux/netfilter/x_tables.h
  - include/uapi/linux/netfilter/x_tables.h
-->

## Summary
Tier-3 design for the x_tables shared framework underpinning iptables / ip6tables / ebtables / arptables. Provides per-AF table-registration (`xt_table`), per-table-rule-blob installation (`xt_replace_table`), per-system match/target module registry (`xt_register_match`/`_target` family), and the per-skb evaluation entry point (`xt_action_param` based per-match invocation). Modern distros run `iptables-nft` (CLI translates to nftables), but `iptables-legacy` still uses x_tables in many embedded / vendor / RHEL-classic deployments.

Per-AF iptables core (`net/ipv4/netfilter/ip_tables.c` etc.) layers atop x_tables, registering per-AF rule format (`struct ipt_entry`) and per-table init.

Sub-tier-3 of `net/netfilter/00-overview.md`. Pairs with `net/netfilter/iptables.md` (per-AF iptables core), `net/netfilter/ebtables.md` (Ethernet bridge), `net/netfilter/arptables.md` (ARP). Many xt_match / xt_target modules are loadable separately (`xt_conntrack`, `xt_state`, `xt_recent`, `xt_set`, etc.).

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| x_tables shared framework: per-AF table register, match/target registry, `xt_replace_table`, NF hook bridging | `net/netfilter/x_tables.c` |
| Public API | `include/linux/netfilter/x_tables.h` |
| UAPI: xt_entry_match, xt_entry_target, xt_counters, xt_table_info, xt_get_revision | `include/uapi/linux/netfilter/x_tables.h` |

## Compatibility contract

### `struct xt_table` (per-table descriptor)

```c
struct xt_table {
    struct list_head list;
    unsigned int valid_hooks;       /* bitmask of NF_INET_* hooks the table covers */
    struct xt_table_info __rcu *private;
    struct nf_hook_ops *ops;
    struct module *me;
    u_int8_t af;                     /* NFPROTO_IPV4 / IPV6 / BRIDGE / ARP */
    int priority;                    /* per-AF NF_IP_PRI_* */
    int (*table_init)(struct net *);
    const char name[XT_TABLE_MAXNAMELEN];
};
```

Layout-byte-identical so existing per-AF table modules work.

### `struct xt_table_info` (per-table rule blob)

```c
struct xt_table_info {
    unsigned int size;       /* total bytes of all rule entries */
    unsigned int number;     /* per-hook chain entry counts */
    unsigned int initial_entries;
    unsigned int hook_entry[NF_INET_NUMHOOKS];     /* per-hook offset into entries */
    unsigned int underflow[NF_INET_NUMHOOKS];      /* per-hook policy offset */
    unsigned int stacksize;
    void ***jumpstack;       /* per-CPU per-hook jump stack */
    void *entries[];        /* the rule blob */
};
```

Per-hook `hook_entry[h]` points to first rule of hook h; `underflow[h]` points to the policy rule (final rule in chain). RCU-protected swap on `xt_replace_table`.

Layout-byte-identical.

### `struct xt_entry_match` + `struct xt_entry_target`

Per-rule packed array of match modules + a single target. Each carries:
```c
struct xt_entry_match {
    union {
        struct {
            __u16 match_size;
            char name[XT_EXTENSION_MAXNAMELEN];
            __u8 revision;
        } user;
        struct {
            __u16 match_size;
            struct xt_match *match;
        } kernel;
    } u;
    unsigned char data[];   /* per-match private data */
};
```

Same structure for `xt_entry_target` (with `target` instead of `match`). Layout-byte-identical so iptables-legacy's NLA encoding works.

### `struct xt_match` + `struct xt_target` registration

```c
struct xt_match {
    struct list_head list;
    const char name[XT_EXTENSION_MAXNAMELEN];
    u_int8_t revision;
    u_int8_t family;
    bool (*match)(const struct sk_buff *, struct xt_action_param *);
    int  (*checkentry)(const struct xt_mtchk_param *);
    void (*destroy)(const struct xt_mtdtor_param *);
    struct module *me;
    const char *table;
    unsigned int matchsize;
    unsigned int hooks;
    unsigned short proto;
    unsigned short usersize;
    /* compat */
    void (*compat_from_user)(void *, const void *);
    int  (*compat_to_user)(void __user *, const void *);
    unsigned int compatsize;
};
```

`struct xt_target` analogous (with `target(skb, par)` callback returning verdict). Per-revision (multiple revisions of same name) supported via `revision` field — userspace declares which revision it understands; kernel selects the matching one.

Identical signatures.

### `xt_register_match` / `_unregister_match` / `_target` / `_*matches` / `_targets`

```c
int xt_register_match(struct xt_match *match);
int xt_unregister_match(struct xt_match *match);
int xt_register_target(struct xt_target *target);
int xt_unregister_target(struct xt_target *target);
int xt_register_matches(struct xt_match *match, unsigned int n);
int xt_register_targets(struct xt_target *target, unsigned int n);
int xt_register_template(...);
```

Per-match / per-target listed in per-AF registry. Identical contract.

### `xt_replace_table` (atomic rule blob swap)

Userspace provides full new rule blob; kernel:
1. Validates per-rule (each match `checkentry`, each target `checkentry`)
2. Prepares new `xt_table_info`
3. RCU-pointer-swaps `table->private` from old to new
4. Frees old after RCU sync

Identical atomicity model — existing rule traversals see either old-blob or new-blob, never partial.

### Per-skb evaluation: `ipt_do_table` / `ip6t_do_table` (per-AF)

Per-AF wrapper invokes per-rule chain walk: for each rule, check IP-header match (via per-AF match), then walk per-rule `xt_entry_match` array (calling each `match->match` callback), then if all match → invoke target.

Returns NF_VERDICT or per-target XT_RETURN / XT_CONTINUE.

Identical algorithm.

### Per-CPU jump stack

Each `xt_table_info` has a per-CPU per-hook jump stack for CHAIN-jump targets (e.g., `-j custom_chain`). Stack depth bounded by `XT_RECURSION_LIMIT`. Identical.

### Per-table counters

Per-rule `struct xt_counters { __u64 pcnt; __u64 bcnt; }` updated on each match. Read via getsockopt SO_GET_COUNTERS.

Identical UAPI.

### setsockopt / getsockopt UAPI

Per-AF userspace control via setsockopt SOL_IP/IPV6 (or per-bridge family):
- `SO_SET_REPLACE` (rule-blob replace)
- `SO_GET_INFO` (table info)
- `SO_GET_ENTRIES` (entries dump)
- `SO_GET_REVISION_MATCH / TARGET` (revision query)
- `SO_GET_COUNTERS`
- `SO_ADD_COUNTERS`

Wire format byte-identical so iptables-legacy CLI works unchanged.

### Module versioning via `revision`

When a match/target's semantics change, upstream bumps `revision`; userspace iptables specifies the revision it built against; kernel returns -EPROTOTYPE if no matching revision is registered. Identical mechanism.

## Requirements

- REQ-1: `struct xt_table` + `xt_table_info` + `xt_entry_match` + `xt_entry_target` byte-identical layout.
- REQ-2: `struct xt_match` + `struct xt_target` registration via `xt_register_match` / `_target` (and bulk variants); per-AF + per-revision dispatch identical.
- REQ-3: `xt_replace_table` atomic swap: validate-then-RCU-swap; concurrent traversal sees old-or-new, never partial.
- REQ-4: Per-AF evaluation entry (`ipt_do_table` / `ip6t_do_table` etc.): per-rule chain walk + per-match invocation + target dispatch identical.
- REQ-5: Per-CPU jump stack for CHAIN target jumps; depth bounded by XT_RECURSION_LIMIT=512.
- REQ-6: Per-rule counters (`xt_counters`): pcnt + bcnt updated atomically per per-CPU shard, aggregated on read.
- REQ-7: setsockopt UAPI: `SO_SET_REPLACE / GET_INFO / GET_ENTRIES / GET_REVISION_* / GET_COUNTERS / ADD_COUNTERS` byte-identical.
- REQ-8: Module-revision dispatch via `xt_get_revision` UAPI: query match/target by name → returns `revision`; reject SO_SET_REPLACE if userspace requests revision not registered.
- REQ-9: 32-bit-userspace compat (compat_from_user / compat_to_user / compatsize) — DEFERRED for v0 (consistent with `xfrm_compat.c` decision in `net/xfrm/00-overview.md` REQ-O3).
- REQ-10: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `pahole struct xt_table` + `xt_table_info` + `xt_entry_match` + `xt_entry_target` byte-identical layout. (covers REQ-1)
- [ ] AC-2: `iptables-legacy -A FORWARD -j ACCEPT` test: NLA-encoded SO_SET_REPLACE byte-identical to upstream's; rule visible via `iptables-legacy -L`. (covers REQ-2, REQ-7)
- [ ] AC-3: Atomic replace test: 1000-rule transaction; concurrent traversal sees old-rules or new-rules, never partial. (covers REQ-3)
- [ ] AC-4: Match invocation test: install rule with `-m conntrack --ctstate ESTABLISHED -j ACCEPT`; matching established packets pass; new packets don't match. (covers REQ-4)
- [ ] AC-5: CHAIN jump test: install user-defined chain `myrules`; `-j myrules` jumps and returns; jump-stack depth never exceeds XT_RECURSION_LIMIT. (covers REQ-5)
- [ ] AC-6: Counters test: 1000 packets through rule; SO_GET_COUNTERS returns pcnt=1000 + bcnt=Σ(packet sizes). (covers REQ-6)
- [ ] AC-7: Revision dispatch test: register two revisions of `xt_state` (revision 0 + 1); userspace requests revision 1; kernel selects revision-1 match. (covers REQ-8)
- [ ] AC-8: 32-bit compat test on x86_64: explicitly skipped per REQ-9 (deferred for v0). (covers REQ-9)
- [ ] AC-9: SO_GET_INFO test: returns table size + per-hook entry counts byte-identical to upstream. (covers REQ-7)
- [ ] AC-10: Hardening section present and follows template. (covers REQ-10)

## Architecture

### Rust module organization

- `kernel::net::netfilter::xtables::Registry` — per-AF table + per-system match/target registries
- `kernel::net::netfilter::xtables::Table` — `struct xt_table` wrapper
- `kernel::net::netfilter::xtables::TableInfo` — `struct xt_table_info`
- `kernel::net::netfilter::xtables::Match` — `struct xt_match` + per-revision dispatch
- `kernel::net::netfilter::xtables::Target` — `struct xt_target`
- `kernel::net::netfilter::xtables::Replace` — `xt_replace_table` atomic swap
- `kernel::net::netfilter::xtables::JumpStack` — per-CPU per-hook jump stack
- `kernel::net::netfilter::xtables::Counters` — per-CPU per-rule counters
- `kernel::net::netfilter::xtables::Sockopt` — setsockopt/getsockopt UAPI
- `kernel::net::netfilter::xtables::Revision` — `xt_get_revision` dispatch

### Locking and concurrency

- **`xt_mutex[NFPROTO_NUMPROTO]`** (per-AF mutex): mutator side for table/match/target registries
- **RCU**: traversal hot-path RCU-side via `table->private`
- **Per-CPU jump stack**: lockless per-CPU per-hook stacks
- **Per-CPU counters**: lockless per-CPU shards; aggregated on read

### Error handling

- `Err(EEXIST)` — duplicate match/target name + revision
- `Err(ENOENT)` — referenced match/target not registered
- `Err(EPROTOTYPE)` — requested revision not available
- `Err(EINVAL)` — bad rule blob / out-of-bounds offsets
- `Err(ELOOP)` — CHAIN jump exceeds XT_RECURSION_LIMIT
- `Err(EPERM)` — non-CAP_NET_ADMIN
- `Err(ENOMEM)` — alloc fail

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Per-AF table list insert/erase under xt_mutex | `kani::proofs::net::netfilter::xtables::registry_safety` |
| `xt_replace_table` RCU-swap (no use-after-free during readers) | `kani::proofs::net::netfilter::xtables::replace_safety` |
| Per-rule chain walk (jump-stack bounded; offsets validated) | `kani::proofs::net::netfilter::xtables::traverse_safety` |
| Per-CPU counter aggregation | `kani::proofs::net::netfilter::xtables::counter_safety` |
| Setsockopt buffer parsing (bounds-checked accessors) | `kani::proofs::net::netfilter::xtables::sockopt_safety` |

### Layer 2: TLA+ models

(none new owned at this granularity)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-AF match/target registry | every entry has unique (name, revision); per-AF lists distinct | `kani::proofs::net::netfilter::xtables::registry_invariants` |
| `xt_table_info` rule blob | `hook_entry[h] < underflow[h] ≤ size` for all `h ∈ valid_hooks`; offsets are 8-byte-aligned | `kani::proofs::net::netfilter::xtables::blob_invariants` |
| Per-CPU jump stack | depth ≤ XT_RECURSION_LIMIT throughout traversal | `kani::proofs::net::netfilter::xtables::stack_invariants` |

### Layer 4: Functional correctness (opt-in)

- **xt_replace_table atomicity theorem** via TLA+ refinement — proves: implementation refines abstract atomic-replace model; concurrent classify never sees mid-swap state.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-match + per-target module refcounts use `Refcount` (saturating) | § Mandatory |
| **CONSTIFY** | per-match + per-target ops vtables `static const` | § Mandatory |
| **SIZE_OVERFLOW** | rule-blob offset + jump-stack arithmetic uses checked operators (CVE class: CVE-2018-class iptables-blob-overflow) | § Mandatory |
| **MEMORY_SANITIZE** | freed `xt_table_info` blob cleared (carries match keys + LSM secctx) | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT, CONSTIFY, SIZE_OVERFLOW, MEMORY_SANITIZE**: see above
- **AUTOSLAB**: per-table_info slab cache
- **USERCOPY**: setsockopt blob copy uses `copy_from_user` with explicit size validation
- **KERNEXEC**: per-match + per-target dispatch via `static const fn-ptr`

### Row-2 / GR-RBAC integration

- LSM hook `security_capable(CAP_NET_ADMIN)` already required for setsockopt mutations.
- Default useful GR-RBAC policy: deny iptables-legacy mutations outside gradm-marked `firewall_admin` role; legacy x_tables has accumulated significant attack surface and many CVEs in xt_modules.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Open Questions

(none — x_tables ABI exhaustively specified by upstream + iptables-legacy regression-test suite)

## Out of Scope

- Per-AF iptables core (cross-ref `net/netfilter/iptables.md`)
- Per-AF ebtables / arptables (cross-ref `net/netfilter/ebtables.md`, `net/netfilter/arptables.md`)
- 32-bit userspace compat — DEFERRED for v0 (REQ-9)
- 32-bit-only paths
- Implementation code
