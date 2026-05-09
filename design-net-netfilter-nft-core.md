---
title: "Tier-3: net/netfilter/nft-core — nftables VM expression evaluation engine"
tags: ["design-doc", "tier-3", "net", "netfilter"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for the nftables VM evaluation engine — the per-skb hot-path that walks rules in a chain, evaluates expressions in each rule, and applies the verdict. Expressions form a register-based VM (16 32-bit registers — `NFT_REG_VERDICT` plus `NFT_REG32_00..15`) with per-expression `eval` callbacks reading/writing register slots. Rule traversal halts on `NFT_BREAK` (rule didn't match → next rule), `NFT_CONTINUE` (continue chain), `NFT_GOTO` (jump to chain), `NFT_JUMP` (call chain w/ return), or terminal verdict (`NFT_DROP / ACCEPT / QUEUE / STOLEN / RETURN`).

Three chain types: `filter` (default), `nat` (NAT-aware; rejects multi-hook), `route` (route-revalidation needed after rewrite — used by mangle-equivalent rules). Each chain registers an NF hook callback (cross-ref `net/netfilter/core.md`) that invokes `nft_do_chain` to drive the VM.

Owns the mandatory `models/net/nft_vm_eval.tla` TLA+ model declared by `net/netfilter/00-overview.md`: per-rule expression chain executes deterministically; concurrent ruleset transactions are atomic via NFT_MSG_NEWRULE/COMMIT.

Sub-tier-3 of `net/netfilter/00-overview.md`. Pairs with `net/netfilter/nft-api.md` (rule installation), `net/netfilter/nft-expressions.md` (per-expression types), `net/netfilter/nft-sets.md` (set lookup expression).

### Requirements

- REQ-1: Register file: `NFT_REG_VERDICT` (reg 0) + `NFT_REG32_00..15` (16 × 4-byte regs); layout-byte-identical to upstream.
- REQ-2: `struct nft_expr_ops` vtable layout-equivalent for first cache-line.
- REQ-3: Verdict constants (NFT_CONTINUE / BREAK / JUMP / GOTO / RETURN + NF_DROP / ACCEPT / QUEUE / STOLEN) byte-identical bit values.
- REQ-4: `nft_do_chain` algorithm: rule-walk + per-rule expression-walk + per-verdict dispatch identical.
- REQ-5: JUMP-stack: bounded by NFT_JUMP_STACK_SIZE=16; per-jump push/pop sound.
- REQ-6: Three chain types (filter / nat / route): per-AF hook registration; constraints enforced at NEWCHAIN time.
- REQ-7: Per-rule expression list: packed array of `nft_expr` with per-expression private data; in-order evaluation.
- REQ-8: Transaction commit: RCU-pointer-swap of rule lists; generation-counter increment; atomic commit-or-rollback.
- REQ-9: Per-expression `validate` callback: pre-commit semantic validation; reject batch on validate failure.
- REQ-10: Trace facility: per-rule trace bit drives `__nft_trace_packet` emission to NFT_MSG_TRACE listeners.
- REQ-11: TLA+ model `models/net/nft_vm_eval.tla` (mandatory per `net/netfilter/00-overview.md` Layer 2) — proves: per-rule expression chain executes deterministically; concurrent ruleset transactions are atomic via NFT_MSG_NEWRULE/COMMIT; concurrent classify never sees mid-swap state.
- REQ-12: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: Register-file size + layout test: `pahole struct nft_regs` byte-identical. (covers REQ-1)
- [ ] AC-2: Simple rule evaluation: `nft add rule inet filter input ip saddr 1.2.3.4 drop`; matching packet → DROP; non-matching → ACCEPT (chain default). (covers REQ-4)
- [ ] AC-3: JUMP test: `nft add chain inet filter custom ; nft add rule inet filter input jump custom`; main chain JUMPs to custom; on RETURN from custom → resume main. (covers REQ-5)
- [ ] AC-4: GOTO test: `nft add rule inet filter input goto custom`; jumps and never returns to main. (covers REQ-4)
- [ ] AC-5: NAT-chain test: `nft add chain ip nat post { type nat hook postrouting priority srcnat; }`; SNAT rule applies. (covers REQ-6)
- [ ] AC-6: route-chain test: `nft add chain ip mangle out { type route hook output priority mangle; }`; route revalidation occurs after rule modifies skb. (covers REQ-6)
- [ ] AC-7: Atomic commit test: 100-rule transaction; concurrent classify sees either old-rules or new-rules, never partial. (covers REQ-8, REQ-11)
- [ ] AC-8: Validate test: rule with expression accessing undefined register → batch rejected with EINVAL at commit time. (covers REQ-9)
- [ ] AC-9: Trace test: `nft add rule ... meta nftrace set 1 accept`; `nft monitor trace` shows per-skb trace events for that rule. (covers REQ-10)
- [ ] AC-10: JUMP-stack overflow test: 17-deep JUMP chain → NFT_JUMP_STACK_SIZE=16 cap exceeded → drop with EINVAL. (covers REQ-5)
- [ ] AC-11: TLA+ `models/net/nft_vm_eval.tla` proves: deterministic eval + atomic transaction commit. (covers REQ-11)
- [ ] AC-12: Hardening section present and follows template. (covers REQ-12)

### Architecture

### Rust module organization

- `kernel::net::netfilter::nft::core::DoChain` — `nft_do_chain` VM driver
- `kernel::net::netfilter::nft::core::Regs` — register file
- `kernel::net::netfilter::nft::core::Verdict` — NFT_* / NF_* verdict enum
- `kernel::net::netfilter::nft::core::JumpStack` — JUMP/RETURN stack
- `kernel::net::netfilter::nft::core::ExprOps` — `struct nft_expr_ops` wrapper
- `kernel::net::netfilter::nft::core::Rule` — per-rule expression list
- `kernel::net::netfilter::nft::core::ChainFilter` — filter-type chain
- `kernel::net::netfilter::nft::core::ChainNat` — nat-type chain (with per-AF hook constraints)
- `kernel::net::netfilter::nft::core::ChainRoute` — route-type chain
- `kernel::net::netfilter::nft::core::Validate` — per-expression validate callback orchestration
- `kernel::net::netfilter::nft::core::Trace` — `__nft_trace_packet` emit

### Locking and concurrency

- **Per-skb VM execution**: thread-local register file (no shared state)
- **RCU**: chain → rule list traversal RCU-side; mutators in `nft-api.md` use RCU-defer
- **Per-netns generation counter**: atomic increment on commit (cross-ref `nft-api.md` REQ-4)

### Error handling

- VM internal errors return NFT_BREAK (rule didn't match, continue) — no error path through VM
- Batch validation errors (`Err(EINVAL)`, `Err(ELOOP)` for cycle, `Err(ENOSPC)` for stack-overflow rejection) returned at commit time

### Out of Scope

- Per-expression types (cross-ref `net/netfilter/nft-expressions.md`)
- Set storage backends (cross-ref `net/netfilter/nft-sets.md`)
- API + transaction commit detail (cross-ref `net/netfilter/nft-api.md`)
- 32-bit-only paths
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| nftables VM core: `nft_do_chain`, register file, expression dispatch | `net/netfilter/nf_tables_core.c` |
| Chain types: filter / nat / route | `net/netfilter/nft_chain_filter.c`, `nft_chain_nat.c`, `nft_chain_route.c` |
| Public API | `include/net/netfilter/nf_tables.h` |

### compatibility contract

### Register file

```c
struct nft_regs {
    union {
        u32        data[NFT_REG_VERDICT_SIZE / sizeof(u32)];
        struct nft_verdict verdict;
    };
};
```

Plus 32-bit register slots: `NFT_REG32_00..15` (16 × 4-byte regs). Wider operations use multiple consecutive slots (e.g., 16-byte IPv6 address occupies REG32_00..03).

`NFT_REG_VERDICT` (always reg 0) holds verdict + chain-jump info; final value of this register at end of evaluation drives the NF hook return.

Layout-byte-identical so wire-format compat with userspace `nft` (which encodes register references in NLA) holds.

### `struct nft_expr_ops` per-expression vtable

```c
struct nft_expr_ops {
    void  (*eval)(const struct nft_expr *, struct nft_regs *, const struct nft_pktinfo *);
    int   (*clone)(struct nft_expr *, const struct nft_expr *, gfp_t);
    unsigned int   size;
    int   (*init)(const struct nft_ctx *, const struct nft_expr *, const struct nlattr *const tb[]);
    void  (*activate)(const struct nft_ctx *, const struct nft_expr *);
    void  (*deactivate)(const struct nft_ctx *, const struct nft_expr *, enum nft_trans_phase);
    void  (*destroy)(const struct nft_ctx *, const struct nft_expr *);
    void  (*destroy_clone)(const struct nft_ctx *, const struct nft_expr *);
    int   (*dump)(struct sk_buff *, const struct nft_expr *, bool);
    int   (*validate)(const struct nft_ctx *, const struct nft_expr *);
    bool  (*reduce)(struct nft_regs_track *, const struct nft_expr *);
    bool  (*gc)(struct net *, const struct nft_expr *);
    int   (*offload)(struct nft_offload_ctx *, struct nft_flow_rule *, const struct nft_expr *);
    bool  (*offload_action)(const struct nft_expr *);
    void  (*offload_stats)(struct nft_expr *, const struct flow_stats *);
    const struct nft_expr_type *type;
    void  *data;
};
```

Layout-equivalent. Per-expression types (payload / meta / immediate / lookup / cmp / bitwise / numgen / random / ct / counter / nat / target / ...) register via `nft_register_expr`.

### Verdict constants (`NFT_*` returned by eval, bridged to `NF_*`)

| Verdict | Constant | Effect on chain walk |
|---|---|---|
| `NFT_CONTINUE` | -1 | Move to next rule |
| `NFT_BREAK` | -2 | Same as CONTINUE; rule didn't match (default after expr fail) |
| `NFT_JUMP` | -3 | Call sub-chain; on RETURN, resume next rule |
| `NFT_GOTO` | -4 | Jump to other chain; no return |
| `NFT_RETURN` | -5 | Return from JUMP'd chain |
| `NF_DROP` | 0 | Terminal: drop |
| `NF_ACCEPT` | 1 | Terminal: accept |
| `NF_QUEUE` | 3 | Terminal: queue to userspace |
| `NF_STOLEN` | 4 | Terminal: callback owns skb |

Bridged to NF hook return at chain exit.

### `nft_do_chain` algorithm

```rust
fn nft_do_chain(pkt, chain) -> NF_VERDICT {
    regs = fresh_regs();
    rule = chain.rules.first();
    'outer: loop {
        for expr in rule.expressions {
            expr.ops.eval(expr, &mut regs, pkt);
            if regs.verdict.code != NFT_CONTINUE {
                match regs.verdict.code {
                    NFT_BREAK => break 'outer (continue next-rule),
                    NFT_JUMP => { push_stack(rule.next); chain = regs.verdict.chain; rule = chain.rules.first(); continue 'outer; }
                    NFT_GOTO => { chain = regs.verdict.chain; rule = chain.rules.first(); continue 'outer; }
                    NFT_RETURN => { rule = pop_stack(); chain = rule.chain; continue 'outer; }
                    NF_* terminal => return that;
                }
            }
        }
        rule = rule.next();
        if rule == NULL { return NF_ACCEPT; }
    }
}
```

Identical algorithm. JUMP-stack depth bounded by `NFT_JUMP_STACK_SIZE = 16`.

### Chain types

| Type | Purpose | Constraints |
|---|---|---|
| `filter` | Default — packet filtering, action chains | Any hook |
| `nat` | NAT manipulation | Single hook per type per AF (PRE_ROUTING for DNAT, POST_ROUTING for SNAT) |
| `route` | Route-revalidation needed (mangle-equivalent) | OUTPUT hook only |

Each chain type registers per-AF hook ops via `nft_chain_filter_init` / `_nat_init` / `_route_init`.

### Per-rule expression list

Per-rule `expressions` is a packed array of `nft_expr` structs:
```c
struct nft_expr {
    const struct nft_expr_ops *ops;
    unsigned char data[];   /* per-expression private data */
};
```

Layout-equivalent. Each rule's expressions evaluated in order; first-match-wins per expression.

### Transaction commit (consumed from `nft-api.md`)

When `NFT_MSG_BEGIN..COMMIT` batch is committed, kernel atomically:
1. Validates all transaction items (NEWRULE / DELRULE / NEWCHAIN / etc.)
2. RCU-pointer-swaps each affected chain's `rules` list head from old-rules to new-rules
3. Increments `net->nft.gencursor` (generation counter)
4. Schedules per-RCU-grace-period free of old rules

Concurrent classify reads RCU-side: either sees old-rules entirely, or new-rules entirely; never sees mid-swap state.

Identical atomicity model.

### `validate` callback per-expression

Pre-commit walk: for each expression in the transaction, call `expr->ops->validate(ctx, expr)` to verify rule semantics (e.g., expression doesn't access undefined registers, register sizes match). Reject batch with -EINVAL if any validate fails.

### Trace facility

When per-rule trace bit set: invoke `__nft_trace_packet` per expression eval to emit `NFT_MSG_TRACE` event to userspace listeners (cross-ref `nft-api.md` REQ-6).

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Register-file access (no out-of-bounds reg read/write) | `kani::proofs::net::netfilter::nft::core::reg_safety` |
| `nft_do_chain` algorithm (no infinite loop; JUMP stack bounded) | `kani::proofs::net::netfilter::nft::core::do_chain_safety` |
| Per-expression eval dispatch (RCU-side fn-ptr; no use-after-free) | `kani::proofs::net::netfilter::nft::core::eval_dispatch_safety` |
| Atomic commit RCU-pointer swap | `kani::proofs::net::netfilter::nft::core::commit_safety` |

### Layer 2: TLA+ models

- `models/net/nft_vm_eval.tla` (mandatory per `net/netfilter/00-overview.md`) — proves deterministic eval + atomic transaction commit. Owned here.

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Register file | NFT_REG_VERDICT always at register 0; subsequent slots are 32-bit aligned | `kani::proofs::net::netfilter::nft::core::reg_invariants` |
| Per-rule expression list | every expression has non-NULL `ops`; per-expression `size` ≥ sizeof(struct nft_expr) | `kani::proofs::net::netfilter::nft::core::expr_invariants` |
| JUMP stack | depth ≤ NFT_JUMP_STACK_SIZE=16 always | `kani::proofs::net::netfilter::nft::core::stack_invariants` |
| Per-chain rules list | RCU-pointer is valid; either points to a complete list (post-commit) or NULL (pre-init) | `kani::proofs::net::netfilter::nft::core::chain_invariants` |

### Layer 4: Functional correctness (declared in `net/netfilter/00-overview.md` Layer 4)

- **nftables transaction atomicity theorem** via TLA+ refinement — proves: NFT_MSG_BEGIN..COMMIT is atomic from concurrent-classify's perspective; concurrent classify never sees mid-swap. Owned here.

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **CONSTIFY** | per-expression `nft_expr_ops` vtables `static const`; per-chain-type `nft_chain_type` `static const` | § Mandatory |
| **SIZE_OVERFLOW** | register-slot indexing + JUMP-stack-depth arithmetic uses checked operators (CVE class: CVE-2022-class register-overflow) | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB**: per-rule + per-expression slabs (cross-ref `net/netfilter/nft-api.md`)
- **CONSTIFY, SIZE_OVERFLOW**: see above
- **MEMORY_SANITIZE**: freed expression state cleared (carries match keys + selectors)
- **USERCOPY**: NLA parsing in init/dump uses bound-checked accessors
- **KERNEXEC**: per-expression dispatch via `static const fn-ptr` array

### Row-2 / GR-RBAC integration

- LSM hook: same as `nft-api.md` (CAP_NET_ADMIN gate; mutations only via API).
- Default GR-RBAC policy: empty.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

