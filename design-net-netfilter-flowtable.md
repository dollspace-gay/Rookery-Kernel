---
title: "Tier-3: net/netfilter/flowtable — fast-path bypass for established flows"
tags: ["design-doc", "tier-3", "net", "netfilter"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for nftables flowtable — the per-tuple fast-path that bypasses the full conntrack-lookup + nftables-rule-chain on established flows. When a TCP / UDP flow becomes established (typically after the 3-way handshake completes for TCP) and a rule installs the flow into a flowtable (`flow add @ft`), subsequent packets for that 5-tuple skip per-packet rule traversal entirely; instead they take a short path: tuple lookup → per-tuple cached forwarding decision (next-hop, output netdev, NAT-mangling, TTL decrement) → direct xmit.

Three offload tiers:
- **Software** (default): kernel runs the fast path (CPU)
- **HW** (`flowtable flags = offload`): NIC's TCAM / dataplane processor handles flow forwarding (Mellanox ASAP², Intel ice/i40e, Marvell Octeon, Broadcom)
- **XDP** (`xdp_flowtable_attach`): per-netdev XDP-driven flowtable lookup runs in driver's poll-mode-receive context for line-rate forwarding without kernel sk_buff allocation

Plus BPF integration (`nf_flow_table_bpf.c`) for BPF programs to look up + manipulate flowtable entries.

Owns the mandatory `models/net/flowtable_offload.tla` TLA+ model declared by `net/netfilter/00-overview.md`: a packet matching a flowtable entry produces output equivalent to the full conntrack+rule-chain path; HW-offloaded entries match SW path.

Sub-tier-3 of `net/netfilter/00-overview.md`. Pairs with `net/netfilter/conntrack-core.md` (consumes nf_conn refs), `net/netfilter/nft-api.md` (flowtable-management messages), `net/sched/act-ct.md` (alternative tc-side flowtable populator).

### Requirements

- REQ-1: `struct flow_offload` + `struct flow_offload_tuple` byte-identical first cache-line layout.
- REQ-2: `nf_flow_offload_add` semantics: idempotent re-install (IPS_OFFLOAD check); per-direction route + MAC + NAT cache; both-direction rhashtable insert.
- REQ-3: Fast-path lookup `nf_flow_offload_lookup`: 5-tuple → rhashtable lookup → cached transformations + xmit.
- REQ-4: Pre-cached transformations: TTL decrement, NAT mangle, incremental L4 checksum, L2 MAC replace.
- REQ-5: HW offload: `NF_FLOWTABLE_HW_OFFLOAD` flag drives `flow_setup_cb_call(TC_SETUP_FT)` to driver; fall back to software on driver error.
- REQ-6: XDP path: `bpf_xdp_flow_lookup` kfunc; per-netdev XDP-attached flowtable lookup; `bpf_xdp_redirect` on hit.
- REQ-7: BPF kfunc set: `bpf_xdp_flow_lookup` + `bpf_skb_flow_lookup` + `bpf_flowtable_offload_release` per upstream signatures.
- REQ-8: Per-tuple GC: `nf_flowtable_default_gc_step` (30s); timeout + conntrack-teardown drives eviction.
- REQ-9: `/proc/net/netfilter/nf_flowtable*` procfs entries byte-identical content.
- REQ-10: TLA+ model `models/net/flowtable_offload.tla` (mandatory per `net/netfilter/00-overview.md` Layer 2) — proves: a packet matching a flowtable entry produces output equivalent to the full conntrack+rule-chain path; HW-offloaded entries match SW path.
- REQ-11: Per-flowtable type registration: inet (v4+v6), ip (v4), ip6 (v6); per-AF setup callbacks identical.
- REQ-12: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: `pahole struct flow_offload` + `struct flow_offload_tuple` byte-identical first cache-line. (covers REQ-1)
- [ ] AC-2: Software fast-path test: install flowtable + rule `flow add @ft`; established TCP flow → first packet does conntrack + rule chain; subsequent packets bypass via flowtable lookup; verifiable via `tc -s qdisc` showing decreasing per-rule counters once offloaded. (covers REQ-2, REQ-3, REQ-4)
- [ ] AC-3: HW offload test on supporting NIC (Mellanox ASAP² ConnectX-5+): `nft add flowtable inet f1 { ... flags offload; }`; verify `nft list flowtables` shows offload flag; line-rate verified via iperf3. (covers REQ-5)
- [ ] AC-4: XDP path test: load XDP program calling `bpf_xdp_flow_lookup`; established flow → XDP redirects to xmit_dst's netdev; sk_buff never allocated. (covers REQ-6)
- [ ] AC-5: BPF kfunc test: `bpftool kfunc list` includes `bpf_xdp_flow_lookup` + `bpf_skb_flow_lookup` + `bpf_flowtable_offload_release`. (covers REQ-7)
- [ ] AC-6: GC test: install flow with 30s timeout; idle for 31s → flow evicted; subsequent packet → conntrack lookup + rule chain again. (covers REQ-8)
- [ ] AC-7: Conntrack-teardown test: TCP FIN/RST seen → conntrack tears down; flowtable entry evicted within RCU grace period. (covers REQ-8)
- [ ] AC-8: NAT-aware fast-path test: install flowtable + NAT rule; established NAT'd flow → fast-path applies cached NAT mangling identically. (covers REQ-4)
- [ ] AC-9: `cat /proc/net/netfilter/nf_flowtable*` byte-identical content. (covers REQ-9)
- [ ] AC-10: Per-AF test: install IPv6 flow into inet flowtable; lookup matches v6 5-tuple. (covers REQ-11)
- [ ] AC-11: TLA+ `models/net/flowtable_offload.tla` proves: fast-path output equivalent to slow-path; HW-offloaded matches SW. (covers REQ-10)
- [ ] AC-12: Hardening section present and follows template. (covers REQ-12)

### Architecture

### Rust module organization

- `kernel::net::netfilter::flowtable::Flowtable` — per-table flow_table
- `kernel::net::netfilter::flowtable::FlowOffload` — `struct flow_offload`
- `kernel::net::netfilter::flowtable::Tuple` — `struct flow_offload_tuple`
- `kernel::net::netfilter::flowtable::Add` — `nf_flow_offload_add`
- `kernel::net::netfilter::flowtable::Lookup` — `nf_flow_offload_lookup`
- `kernel::net::netfilter::flowtable::Transform` — TTL/NAT/checksum/MAC pre-cached transformations
- `kernel::net::netfilter::flowtable::Gc` — periodic GC
- `kernel::net::netfilter::flowtable::HwOffload` — `flow_setup_cb_call(TC_SETUP_FT)` bridge
- `kernel::net::netfilter::flowtable::Xdp` — XDP path (`nf_flow_table_xdp.c`)
- `kernel::net::netfilter::flowtable::bpf::Kfunc` — BPF kfunc set
- `kernel::net::netfilter::flowtable::Path` — per-tuple route-path resolution
- `kernel::net::netfilter::flowtable::Procfs` — `/proc/net/netfilter/nf_flowtable*`

### Locking and concurrency

- **Per-flowtable rhashtable**: per-bucket lockless insert/erase via rhashtable's spinhash; lookup RCU-side
- **Per-flow_offload `flags`** (atomic): tracks IPS_OFFLOAD-equivalent state + teardown gate
- **Per-driver HW-offload state**: per-driver locking; per-flowtable list of pending offload-installs
- **RCU**: lookup hot-path RCU-side; eviction via call_rcu

### Error handling

- `Err(EEXIST)` — flow already in flowtable (caller should retry without re-add)
- `Err(ENOMEM)` — alloc fail
- `Err(ENOSPC)` — flowtable size cap; or HW table full
- `Err(EINVAL)` — bad tuple (e.g., L3proto unsupported by flowtable type)

### Out of Scope

- Conntrack core (cross-ref `net/netfilter/conntrack-core.md`)
- nft API (cross-ref `net/netfilter/nft-api.md`)
- 32-bit-only paths
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Flowtable core: per-table flow_table, per-tuple flow_table_entry, lookup, GC | `net/netfilter/nf_flow_table_core.c` |
| Per-AF inet flowtable | `net/netfilter/nf_flow_table_inet.c` |
| Per-AF IPv4/IPv6 flowtable | `net/netfilter/nf_flow_table_ip.c` |
| HW offload bridge | `net/netfilter/nf_flow_table_offload.c` |
| Per-tuple route-path resolution | `net/netfilter/nf_flow_table_path.c` |
| /proc reporter | `net/netfilter/nf_flow_table_procfs.c` |
| XDP flowtable hook | `net/netfilter/nf_flow_table_xdp.c` |
| BPF kfunc set for flowtable | `net/netfilter/nf_flow_table_bpf.c` |
| Public API | `include/net/netfilter/nf_flow_table.h` |

### compatibility contract

### `struct nf_flowtable_type` per-AF

```c
struct nf_flowtable_type {
    struct list_head list;
    int family;
    int (*init)(struct nf_flowtable *ft);
    int (*setup)(struct nf_flowtable *ft, struct net_device *dev, enum flow_block_command cmd);
    int (*action)(struct net *net, struct flow_offload *flow, enum flow_offload_tuple_dir dir, struct nf_flow_rule *flow_rule);
    void (*free)(struct nf_flowtable *ft);
    nf_hookfn *hook;
    struct module *owner;
};
```

Per-AF flowtable types: inet (covering both v4 + v6), ip (v4-only), ip6 (v6-only).

Layout-equivalent.

### `struct flow_offload` (per-tuple flow entry)

```c
struct flow_offload {
    struct flow_offload_tuple_rhash tuplehash[FLOW_OFFLOAD_DIR_MAX];   /* original + reply */
    struct nf_conn *ct;                                                  /* parent conntrack */
    unsigned long flags;
    u16 type;
    __be16 inner_protonum;
    u32 timeout;
    struct rcu_head rcu_head;
};
```

Plus per-direction:
```c
struct flow_offload_tuple {
    union {
        struct in_addr src_v4;
        struct in6_addr src_v6;
    };
    union {
        struct in_addr dst_v4;
        struct in6_addr dst_v6;
    };
    union {
        struct {
            __be16 src_port;
            __be16 dst_port;
        };
        struct {
            u8 type;
            u8 code;
            __be16 id;
        };
    };
    u8 l3proto;
    u8 l4proto;
    u8 dir;
    u16 mtu;
    union {
        struct dst_entry *dst_cache;
        struct nf_flowtable *ft;
    };
    u32 dst_cookie;
    struct dst_entry *xmit_dst;
    /* per-direction encap (VLAN, GRE, etc.) */
    /* per-direction MAC addresses (for L2 forwarding) */
};
```

Layout-equivalent for first cache-line.

### `nf_flow_offload_add` (entry installation)

Called from nft `flow add @ft` rule expression:
1. Check `ct->status` for `IPS_OFFLOAD` — if already set, no-op
2. Allocate `flow_offload`; initialize per-direction tuple from `ct->tuplehash`
3. Resolve route for both directions: per-tuple `dst_entry` + per-tuple `xmit_dst` cached
4. Insert into per-flowtable rhashtable (both directions)
5. Set `IPS_OFFLOAD` on parent conntrack
6. If flowtable has `NF_FLOWTABLE_HW_OFFLOAD` flag: enqueue per-driver flow_setup_cb_call(TC_SETUP_FT)

Identical algorithm.

### Fast-path lookup (`nf_flow_offload_lookup`)

Per-skb at PRE_ROUTING (or per-netdev ingress for XDP path):
1. Extract 5-tuple from skb
2. Lookup in flowtable's rhashtable
3. On miss → return — packet falls through to slow path
4. On hit → use cached `xmit_dst` + per-direction MAC + NAT mangling
5. Apply pre-cached transformations:
   - Decrement TTL
   - NAT mangle src/dst per cached mapping
   - Update L4 checksum (incremental)
   - Replace L2 header with cached MACs
6. Direct xmit via `xmit_dst->output(skb)`

Bypasses entire conntrack lookup + nftables rule chain. Per-skb cost: 5-tuple hash + rhashtable lookup + ≤10 register operations.

### HW offload contract

When `NF_FLOWTABLE_HW_OFFLOAD` flag set:
1. Driver implements `xdo_dev_flowtable_*` ops
2. flowtable installs per-tuple via `flow_setup_cb_call(TC_SETUP_FT)` to driver
3. Driver programs hardware (TCAM lookup table; per-tuple SOC accelerator state)
4. Subsequent matching packets handled in NIC; never enter kernel
5. On fault (table-full, MAC-flush): driver returns -ENOSPC → fall back to software path
6. On flow teardown: driver evicts via TC_SETUP_FT/destroy

Identical contract.

### XDP path (`nf_flow_table_xdp.c`)

Per-netdev XDP program can call `bpf_xdp_flow_lookup` kfunc to perform flowtable lookup at XDP-receive context (before sk_buff allocation). On hit → bpf_xdp_redirect to xmit_dst's netdev. Lower latency + higher throughput than software flowtable path (no skb alloc).

### BPF integration

`nf_flow_table_bpf.c` exposes kfuncs:
- `bpf_xdp_flow_lookup` — XDP-context lookup
- `bpf_skb_flow_lookup` — TC/socket-context lookup
- `bpf_flowtable_offload_release` — refcount release after lookup

### Per-flow GC

Per-tuple `timeout` (set from conntrack timeout); GC every `nf_flowtable_default_gc_step` (default 30s). On timeout or conntrack-side teardown (FIN/RST seen in conntrack helper), flowtable entry evicted.

### `/proc/net/netfilter/nf_flowtable*` (procfs)

Per-flowtable summary + per-flow dump. Format-byte-identical so existing `iptables-save -t flow` / `nft -j list flowtables` parse correctly.

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| `nf_flow_offload_add` per-direction insert (refcount sound across both directions) | `kani::proofs::net::netfilter::flowtable::add_safety` |
| `nf_flow_offload_lookup` rhashtable lookup (RCU-side; no use-after-free) | `kani::proofs::net::netfilter::flowtable::lookup_safety` |
| Pre-cached transform application (skb header rewrite + checksum incremental) | `kani::proofs::net::netfilter::flowtable::transform_safety` |
| HW-offload bridge (driver fault → fallback path) | `kani::proofs::net::netfilter::flowtable::hw_offload_safety` |
| BPF kfunc refcount-acquire/release pairing | `kani::proofs::net::netfilter::flowtable::bpf_safety` |

### Layer 2: TLA+ models

- `models/net/flowtable_offload.tla` (mandatory per `net/netfilter/00-overview.md`) — proves fast-path equivalence + HW-offload soundness. Owned here.

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-flowtable rhashtable | both directions of an installed flow appear in respective buckets; no orphans | `kani::proofs::net::netfilter::flowtable::table_invariants` |
| Per-flow_offload state | when alive, parent conntrack has IPS_OFFLOAD set; xmit_dst non-NULL | `kani::proofs::net::netfilter::flowtable::offload_invariants` |
| HW-offload state | when offloaded, driver has corresponding entry; SW fallback when driver returns ENOSPC | `kani::proofs::net::netfilter::flowtable::hw_invariants` |

### Layer 4: Functional correctness (declared in `net/netfilter/00-overview.md` Layer 4)

- **Flowtable-fast-path equivalence theorem** via TLA+ refinement — proves: ∀ skb matching flowtable entry, fast-path output = slow-path output (same bytes on wire). Owned here.
- **HW-offload equivalence theorem** via Verus — proves: HW-offloaded flow's wire output equals SW path's; fault-then-fallback is correctness-preserving.

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-flow_offload + per-conntrack-ref refcounts use `Refcount` (saturating); HW-offload ref via paired BPF kfuncs | § Mandatory |
| **AUTOSLAB** | per-flow_offload slab cache | § Mandatory |
| **MEMORY_SANITIZE** | freed flow_offload cleared (carries 5-tuple + cached MAC + NAT mapping) | § Default-on configurable off |
| **SIZE_OVERFLOW** | per-flow timeout + GC interval arithmetic uses checked operators | § Mandatory |
| **CONSTIFY** | per-AF `nf_flowtable_type` instances `static const` | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE, SIZE_OVERFLOW, CONSTIFY**: see above
- **USERCOPY**: NLA parsing in API layer uses bound-checked accessors
- **KERNEXEC**: per-AF dispatch via `static const fn-ptr` array

### Row-2 / GR-RBAC integration

- LSM hook: same as `nft-api.md` (CAP_NET_ADMIN gate; flowtable mutations only via API).
- LSM hook `security_capable(CAP_BPF)` for XDP path + BPF kfuncs.
- Default useful GR-RBAC policy: deny BPF-XDP-flowtable-attach outside gradm-marked `bpf_users` role.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

