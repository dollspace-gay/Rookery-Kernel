---
title: "Tier-3: net/sched/act-mirred — mirror / redirect action"
tags: ["design-doc", "tier-3", "net"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for `act_mirred` — the most-used tc action, providing mirror (clone-and-send-to-target) and redirect (move-to-target) semantics on either ingress or egress of a target netdev. The cornerstone of every modern container-networking dataplane: Cilium uses `mirred redirect ingress` to inject packets into per-pod veth peers; OvS uses it to bridge OVS datapath with kernel datapath; SPAN/RSPAN port-mirroring setups use `mirred mirror egress`.

Four actions exposed:
- **`TCA_EGRESS_REDIRECT`**: pop skb from current path; send out target netdev's egress; original processing stops
- **`TCA_INGRESS_REDIRECT`**: pop skb from current path; inject into target netdev's ingress; original processing stops
- **`TCA_EGRESS_MIRROR`**: clone skb; send clone out target netdev's egress; original continues
- **`TCA_INGRESS_MIRROR`**: clone skb; inject clone into target netdev's ingress; original continues

Sub-tier-3 of `net/sched/00-overview.md`. Pairs with `net/sched/act-api.md` (action framework). Driver-side HW offload for many NICs (Mellanox ASAP², Intel ice/i40e, Marvell) supports redirect/mirror at line rate via TCAM rules.

### Requirements

- REQ-1: `struct tc_mirred` byte-identical UAPI layout.
- REQ-2: All 4 `eaction` types (`EGRESS_REDIRECT / EGRESS_MIRROR / INGRESS_REDIRECT / INGRESS_MIRROR`) with identical semantics.
- REQ-3: `TCA_MIRRED_*` NLAs (per the table) parsed identically; `TCA_MIRRED_BLOCKID` shared-block-target alternative supported.
- REQ-4: Per-skb recursion depth cap `MIRRED_RECURSION_LIMIT=4`; per-CPU counter via `__this_cpu_inc/dec`.
- REQ-5: Target-netdev lookup via ifindex (or blockid); netdev-down → drop with counter increment.
- REQ-6: Mirror semantics: `skb_clone` + send-to-target; original returns TC_ACT_PIPE (continue chain).
- REQ-7: Redirect semantics: skb itself transferred; original returns TC_ACT_REDIRECT (terminate chain on this path).
- REQ-8: HW offload via `flow_action_mirred` per-filter integration; identical contract.
- REQ-9: Per-action dump byte-identical.
- REQ-10: Per-action stats (bytes, packets, drops) tracked via per-CPU `cpu_bstats` / `cpu_qstats` (cross-ref `act-api.md`).
- REQ-11: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: `pahole struct tc_mirred` byte-identical layout. (covers REQ-1)
- [ ] AC-2: Egress redirect test: `tc filter add dev eth0 ingress flower ip_proto tcp dst_port 8080 action mirred egress redirect dev eth1` → matching packet emerges on eth1; original eth0 path doesn't continue. (covers REQ-2, REQ-7)
- [ ] AC-3: Ingress redirect test: `tc filter add dev eth0 ingress flower ... action mirred ingress redirect dev veth-pod` → matching packet appears as ingress on veth-pod (typical Cilium setup). (covers REQ-2, REQ-7)
- [ ] AC-4: Egress mirror test: `tc filter ... action mirred egress mirror dev mon0` (port mirror) → packet appears on mon0 AND continues original processing on eth0. (covers REQ-2, REQ-6)
- [ ] AC-5: Ingress mirror test: clone arrives on target's ingress; original continues unchanged. (covers REQ-2, REQ-6)
- [ ] AC-6: Recursion-cap test: chain of 5 mirred actions (eth0 → eth1 → eth2 → eth3 → eth4 → eth5) → 5th hop drops + counter increments (limit is 4). (covers REQ-4)
- [ ] AC-7: Down-target test: `ip link set eth1 down`; matching packet routed via mirred to eth1 → drop + counter. (covers REQ-5)
- [ ] AC-8: Block-target test: filter with `TCA_MIRRED_BLOCKID=42` instead of ifindex; redirect targets all members of block 42. (covers REQ-3)
- [ ] AC-9: HW-offload test on supporting NIC (Mellanox ASAP² / Intel ice): mirred-redirect filter offloads to TCAM; line-rate redirect verified. (covers REQ-8)
- [ ] AC-10: Per-action dump test: `tc -s actions list action mirred` byte-identical content. (covers REQ-9)
- [ ] AC-11: Per-CPU stats test: 1000 packets through mirred → `tcfa_bstats.bytes + packets` aggregated correctly across CPUs. (covers REQ-10)
- [ ] AC-12: Hardening section present and follows template. (covers REQ-11)

### Architecture

### Rust module organization

- `kernel::net::sched::act_mirred::Mirred` — action instance
- `kernel::net::sched::act_mirred::Eaction` — 4-variant enum (EGRESS_REDIRECT etc.)
- `kernel::net::sched::act_mirred::TargetLookup` — ifindex or blockid → netdev
- `kernel::net::sched::act_mirred::RecursionLimit` — per-CPU counter
- `kernel::net::sched::act_mirred::Mirror` — skb_clone + send
- `kernel::net::sched::act_mirred::Redirect` — skb transfer
- `kernel::net::sched::act_mirred::HwOffload` — `flow_action_mirred` bridge

### Locking and concurrency

- **Per-action `tcfa_lock`** (spinlock, inherited from act-api): held during config mutation
- **Per-CPU recursion counter**: lockless `__this_cpu_*`
- **RCU**: target-netdev lookup hot path is RCU-side

### Error handling

- `Err(EINVAL)` — bad NLA / invalid eaction
- `Err(ENODEV)` — bad ifindex → target down → drop with counter
- `Err(ELOOP)` — recursion-cap exceeded
- `Err(ENOMEM)` — skb_clone failure (mirror) → drop

### Out of Scope

- Other actions (cross-ref `net/sched/act-police.md`, `act-ct.md`, etc.)
- 32-bit-only paths
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| mirred action: target-netdev redirect/mirror, ingress/egress dispatch, per-action dump, HW offload | `net/sched/act_mirred.c` |
| Public API | `include/net/tc_act/tc_mirred.h`, `include/net/act_api.h` |
| UAPI: `TCA_MIRRED_*` NLAs + `struct tc_mirred` + redirect-action constants | `include/uapi/linux/tc_act/tc_mirred.h` |

### compatibility contract

### `struct tc_mirred` (UAPI per-action config)

```c
struct tc_mirred {
    tc_gen;                  /* common: action, refcnt, bindcnt, capab */
    int  eaction;            /* TCA_EGRESS_REDIRECT / INGRESS_REDIRECT / EGRESS_MIRROR / INGRESS_MIRROR */
    __u32 ifindex;           /* target netdev */
};
```

Where `tc_gen` is the common per-action header (action, refcnt, bindcnt, capab — shared across all act_*).

Layout-byte-identical so iproute2's `tc actions add action mirred ...` works unchanged.

### `eaction` constants

| Constant | Value | Semantic |
|---|---|---|
| `TCA_EGRESS_REDIRECT` | 1 | Move skb to target netdev's egress; stop processing on this path |
| `TCA_EGRESS_MIRROR` | 2 | Clone skb, send clone to target's egress; original continues |
| `TCA_INGRESS_REDIRECT` | 3 | Move skb to target's ingress; stop processing |
| `TCA_INGRESS_MIRROR` | 4 | Clone skb, inject clone to target's ingress; original continues |

Identical bit values + semantics.

### NLA configuration

`TCA_OPTIONS` nested NLAs:
- `TCA_MIRRED_PARMS` (struct tc_mirred + extras)
- `TCA_MIRRED_TM` (read-only timestamp/usage)
- `TCA_MIRRED_PAD`
- `TCA_MIRRED_BLOCKID` (alternative to ifindex — target a shared block instead of a single netdev; gives "to whichever block this filter applies to" semantics)

Wire format byte-identical so iproute2 + tc-test-suite work unchanged.

### Recursion bounds

`mirred` may itself fire from within an action chain that was triggered by another `mirred` (e.g., redirect to veth-peer that has its own ingress qdisc with another mirred). Per-skb recursion depth tracked via per-CPU `mirred_recursion` counter; capped at `MIRRED_RECURSION_LIMIT=4` to prevent infinite loops.

Identical limit + counter.

### HW offload

For drivers supporting `flow_action_mirred` (TC_SETUP_CLSFLOWER + TC_ACT_REDIRECT path): per-action mirred installs as part of the per-filter flow-action descriptor; HW programs TCAM to redirect/mirror packets matching the filter. On miss → software fallback.

Identical contract.

### `act` callback semantics

```rust
fn act(skb, action_state) -> int {
    if recursion_depth > 4 { drop; return TC_ACT_SHOT; }
    target = lookup_netdev(action_state.ifindex);
    if target is NULL { drop; return TC_ACT_SHOT; }
    if eaction is MIRROR:
        clone = skb_clone(skb);
        if eaction is EGRESS_MIRROR:
            dev_queue_xmit(target, clone);
        else (INGRESS_MIRROR):
            netif_rx(target, clone);
        return TC_ACT_PIPE;  // original continues
    if eaction is REDIRECT:
        // skb itself transferred
        if eaction is EGRESS_REDIRECT:
            dev_queue_xmit(target, skb);
        else (INGRESS_REDIRECT):
            netif_rx(target, skb);
        return TC_ACT_REDIRECT; // original path stops, skb consumed
}
```

Identical algorithm.

### Per-action dump

`tcfa_dump` returns NLA-encoded `TCA_MIRRED_PARMS`, `TCA_MIRRED_TM`, `TCA_MIRRED_BLOCKID` (when set). Format byte-identical so `tc -s actions list action mirred` works.

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Per-CPU recursion counter (no underflow on dec; bounded ≤ MIRRED_RECURSION_LIMIT) | `kani::proofs::net::sched::act_mirred::recursion_safety` |
| Target-netdev lookup (RCU-side; no use-after-free of netdev pointer) | `kani::proofs::net::sched::act_mirred::lookup_safety` |
| skb_clone for mirror (refcount sound; clone freed on send-failure) | `kani::proofs::net::sched::act_mirred::clone_safety` |
| HW-offload bridge | `kani::proofs::net::sched::act_mirred::offload_safety` |

### Layer 2: TLA+ models

(none new owned at this granularity)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Recursion counter | per-CPU value ≥ 0 always; ≤ MIRRED_RECURSION_LIMIT before each invoke | `kani::proofs::net::sched::act_mirred::counter_invariants` |
| Per-action target | `ifindex != 0` XOR `blockid != 0` (exactly one of the two set) | `kani::proofs::net::sched::act_mirred::target_invariants` |

### Layer 4: Functional correctness (opt-in)

- **Recursion-bound theorem** via Verus — proves: ∀ mirred-chain, total mirred invocations per skb ≤ MIRRED_RECURSION_LIMIT; never infinite-loops.

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-action + per-target-netdev refcounts use `Refcount` (saturating) | § Mandatory |
| **CONSTIFY** | `tc_action_ops` for mirred `static const` | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: per-action (cross-ref `net/sched/act-api.md`); per-skb (cross-ref `net/skbuff.md`)
- **CONSTIFY**: see above
- **USERCOPY**: NLA parsing uses bound-checked accessors
- **SIZE_OVERFLOW**: per-CPU stats aggregation arithmetic uses checked operators
- **KERNEXEC**: action dispatch via `static const fn-ptr`

### Row-2 / GR-RBAC integration

- LSM hook: same as `act-api.md` (CAP_NET_ADMIN gate).
- Useful default GR-RBAC policy: deny mirred-redirect across security-zone boundaries (e.g., from container to host iface) outside gradm-marked `network_admin` role; mirred is a frequent vector for sandbox escape via packet-injection.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

