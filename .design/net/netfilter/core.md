# Tier-3: net/netfilter/core — NF hook framework (per-AF / per-hook callback chains)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/netfilter/core.c
  - include/linux/netfilter.h
  - include/uapi/linux/netfilter.h
-->

## Summary
Tier-3 design for the netfilter hook framework — the foundational dispatch layer that runs at each of the 5 standard NF_INET_* hooks per AF, walking per-AF/per-hook callback chains in priority order and applying verdicts (`NF_ACCEPT / DROP / QUEUE / STOLEN / REPEAT`). Every netfilter consumer (conntrack, NAT, nftables, flowtable, x_tables/iptables/ip6tables/ebtables/arptables, IPVS) registers its callbacks at this layer.

Two registration paths:
- **Module-level** (`nf_register_net_hook` / `nf_register_net_hooks`): per-netns registration; per-AF + per-hook + priority
- **xtables-level**: registered via x_tables module's per-table chain registration → translates to netfilter hooks

Sub-tier-3 of `net/netfilter/00-overview.md`. The "front gate" of the entire netfilter subsystem; every Tier-3 child consumes this layer's hook-registration API.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Hook framework: per-AF/per-hook chains, registration, traversal, verdict dispatch | `net/netfilter/core.c` |
| Public API + UAPI | `include/linux/netfilter.h`, `include/uapi/linux/netfilter.h` |

## Compatibility contract

### 5 standard NF_INET_* hooks (per AF)

| Hook | Constant | Where invoked |
|---|---|---|
| `NF_INET_PRE_ROUTING` | 0 | After RX validation, before route lookup |
| `NF_INET_LOCAL_IN` | 1 | After route resolves to local |
| `NF_INET_FORWARD` | 2 | After route resolves to remote (forwarding) |
| `NF_INET_LOCAL_OUT` | 3 | TX from local socket, before route |
| `NF_INET_POST_ROUTING` | 4 | Just before driver xmit |

Plus AF-specific hooks:
- **Bridge**: `NF_BR_PRE_ROUTING / LOCAL_IN / FORWARD / LOCAL_OUT / POST_ROUTING / BROUTING`
- **ARP**: `NF_ARP_IN / OUT / FORWARD`
- **Netdev**: `NF_NETDEV_INGRESS / EGRESS` (per-netdev hooks for high-throughput TC-like setups)

### Verdict constants

```c
#define NF_DROP    0
#define NF_ACCEPT  1
#define NF_STOLEN  2
#define NF_QUEUE   3
#define NF_REPEAT  4
#define NF_STOP    5    /* deprecated; aliased to ACCEPT */
#define NF_MAX_VERDICT  NF_STOP
```

Plus encoding: `NF_QUEUE_NR(num) << 16 | NF_QUEUE` (queue-number embedded in verdict for NFQUEUE).

Identical bit values + semantics.

### `struct nf_hook_ops` (per-callback descriptor)

```c
struct nf_hook_ops {
    nf_hookfn      *hook;
    struct net_device *dev;
    void           *priv;
    u8              pf;          /* NFPROTO_INET / IPV4 / IPV6 / BRIDGE / ARP / NETDEV / ... */
    enum nf_inet_hooks hooknum;
    int             priority;
};
```

Layout-byte-identical so per-module registrations work.

### Per-netns / per-AF / per-hook chains

Per-netns `struct net.nf` tracks per-AF hook-chains. Each (pf, hooknum) tuple has a sorted-by-priority list of `nf_hook_entry` (refcount-protected RCU-side-readable). On packet traversal:
1. `nf_hook_slow(skb, pf, hooknum, indev, outdev)` invoked at hook point
2. Walk `nf_hook_entries_by_pf[pf][hooknum]` in priority order
3. For each entry: invoke `entry->hook(entry->priv, skb, state)`
4. On NF_DROP: free skb + return failure
5. On NF_ACCEPT: continue to next entry (or after all → return success)
6. On NF_QUEUE: handoff to nfnetlink_queue + return suspended
7. On NF_STOLEN: callback owns skb; don't continue
8. On NF_REPEAT: re-run callback chain from start

Identical algorithm + semantics.

### `nf_register_net_hook` / `nf_register_net_hooks` / `_unregister_*`

```c
int nf_register_net_hook(struct net *net, const struct nf_hook_ops *ops);
int nf_register_net_hooks(struct net *net, const struct nf_hook_ops *ops, unsigned int n);
void nf_unregister_net_hook(struct net *net, const struct nf_hook_ops *ops);
void nf_unregister_net_hooks(struct net *net, const struct nf_hook_ops *ops, unsigned int n);
```

Per-module per-netns registration. Identical signatures.

### Priority constants (defined per-AF in `include/uapi/linux/netfilter_ipv4.h` etc.)

For NFPROTO_IPV4:
- `NF_IP_PRI_RAW_BEFORE_DEFRAG = -450`
- `NF_IP_PRI_CONNTRACK_DEFRAG = -400`
- `NF_IP_PRI_RAW = -300`
- `NF_IP_PRI_SELINUX_FIRST = -225`
- `NF_IP_PRI_CONNTRACK = -200`
- `NF_IP_PRI_MANGLE = -150`
- `NF_IP_PRI_NAT_DST = -100`
- `NF_IP_PRI_FILTER = 0`
- `NF_IP_PRI_SECURITY = 50`
- `NF_IP_PRI_NAT_SRC = 100`
- `NF_IP_PRI_SELINUX_LAST = 225`
- `NF_IP_PRI_CONNTRACK_HELPER = 300`
- `NF_IP_PRI_LAST = INT_MAX`

Identical priorities — order of (conntrack-defrag, raw, conntrack, mangle, dnat, filter, security, snat, conntrack-helper) is established lore.

### Per-netdev hooks (NFPROTO_NETDEV ingress/egress)

Per-netdev `nf_hook_ingress` registered via `nf_register_net_hook` with `NFPROTO_NETDEV` + `NF_NETDEV_INGRESS`/`EGRESS`; bound to a single `dev`. Used for ingress-side filtering before regular IP pre-routing (e.g., `tc ingress` and `nft hook ingress` setups).

### Hook removal + RCU

Unregister waits for synchronize_net (RCU sync) so in-flight hook traversals complete before module unload. Identical lifecycle.

## Requirements

- REQ-1: 5 standard NF_INET_* hooks + per-AF AF-specific hooks (Bridge / ARP / Netdev) registered identically per upstream.
- REQ-2: Verdict constants `NF_DROP / ACCEPT / STOLEN / QUEUE / REPEAT` byte-identical; queue-number embedded encoding identical.
- REQ-3: `struct nf_hook_ops` byte-identical layout.
- REQ-4: Per-netns per-AF per-hook chain list ordered by priority ascending; concurrent walk via RCU; mutator under per-netns mutex.
- REQ-5: `nf_hook_slow` traversal: priority-ordered walk; per-verdict dispatch (ACCEPT/DROP/QUEUE/STOLEN/REPEAT); identical algorithm.
- REQ-6: `nf_register_net_hook` / `nf_register_net_hooks` / `_unregister_*` semantics identical.
- REQ-7: Priority constants per-AF (NF_IP_PRI_*, NF_IP6_PRI_*) byte-identical to upstream.
- REQ-8: NFPROTO_NETDEV ingress/egress per-netdev hook registration; identical contract.
- REQ-9: RCU synchronization on hook removal: synchronize_net before freeing.
- REQ-10: Per-netns isolation: hooks registered in netns A invisible to netns B.
- REQ-11: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `pahole struct nf_hook_ops` byte-identical layout. (covers REQ-3)
- [ ] AC-2: Hook traversal test: register 3 hooks at PRE_ROUTING with priorities -200, 0, 100; verify invocation order (-200 first, 100 last) via tracepoint counters. (covers REQ-4, REQ-5)
- [ ] AC-3: NF_DROP test: hook returns NF_DROP → skb freed; subsequent hooks in chain not called. (covers REQ-2, REQ-5)
- [ ] AC-4: NF_QUEUE test: hook returns `NF_QUEUE_NR(0) << 16 | NF_QUEUE`; packet handed to nfnetlink_queue with queue=0. (covers REQ-2)
- [ ] AC-5: NF_REPEAT test: hook returns NF_REPEAT → chain re-walked from start; verify via counter. (covers REQ-2, REQ-5)
- [ ] AC-6: Per-AF separation test: register hook at NFPROTO_IPV4; IPv6 traffic doesn't trigger it. (covers REQ-1)
- [ ] AC-7: Priority constants test: `NF_IP_PRI_CONNTRACK = -200` matches upstream's actual conntrack module priority. (covers REQ-7)
- [ ] AC-8: NFPROTO_NETDEV ingress test: register per-netdev ingress hook; matching packet on bound dev triggers; on other dev doesn't. (covers REQ-8)
- [ ] AC-9: Module-unload safety test: rapid register/unregister of 1000 hooks; RCU synchronize before free; no use-after-free crash. (covers REQ-9)
- [ ] AC-10: Per-netns test: hook registered in netns A invisible to packets in netns B (verifiable via packet-passes-through-A-but-not-B test). (covers REQ-10)
- [ ] AC-11: Hardening section present and follows template. (covers REQ-11)

## Architecture

### Rust module organization

- `kernel::net::netfilter::core::Hook` — `struct nf_hook_ops` wrapper
- `kernel::net::netfilter::core::Chain` — per-(netns, pf, hooknum) priority-sorted chain
- `kernel::net::netfilter::core::Registry` — per-netns chain table (per-AF × per-hooknum)
- `kernel::net::netfilter::core::Traversal` — `nf_hook_slow` walk + verdict dispatch
- `kernel::net::netfilter::core::Verdict` — `NF_*` enum + queue-number encoding
- `kernel::net::netfilter::core::PerNetdev` — NFPROTO_NETDEV ingress/egress hook
- `kernel::net::netfilter::core::Priority` — per-AF priority constants

### Locking and concurrency

- **Per-netns `nf_hook_mutex`** (mutex): chain mutator (register/unregister)
- **RCU**: chain traversal hot path is RCU-side (`nf_hook_entries`)
- **Per-netdev hook chain**: per-netdev `nf_hook_ingress` pointer; protected by RTNL during register

### Error handling

- `Err(EEXIST)` — duplicate hook registration with same (pf, hooknum, priority, hookfn)
- `Err(EINVAL)` — bad pf / bad hooknum
- `Err(ENOENT)` — unregister non-existent
- `Err(ENOMEM)` — alloc fail
- `Err(EPERM)` — non-CAP_NET_ADMIN

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Per-chain insert/erase under nf_hook_mutex | `kani::proofs::net::netfilter::core::registry_safety` |
| `nf_hook_slow` RCU-side traversal (no use-after-free during in-flight unregister) | `kani::proofs::net::netfilter::core::traverse_safety` |
| Verdict-dispatch state machine (REPEAT bounded; QUEUE handoff sound) | `kani::proofs::net::netfilter::core::verdict_safety` |
| Per-netdev hook bind/unbind under RTNL | `kani::proofs::net::netfilter::core::netdev_safety` |

### Layer 2: TLA+ models

(none new owned at this granularity; per-consumer Tier-3s declare per-consumer TLA+ models — conntrack state machine, NAT translation, etc.)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-(netns, pf, hooknum) chain | priority strictly increasing | `kani::proofs::net::netfilter::core::priority_invariants` |
| Per-netdev ingress hook | every netdev has at most one ingress hook chain registered | `kani::proofs::net::netfilter::core::netdev_invariants` |
| Per-netns isolation | hooks registered in netns A invisible to skbs from netns B | `kani::proofs::net::netfilter::core::isolation_invariants` |

### Layer 4: Functional correctness (opt-in)

- **Hook-traversal soundness theorem** via Verus — proves: `nf_hook_slow` calls hooks in strict priority order; on NF_REPEAT bounded by MAX_REPEAT_ITER; verdict dispatch is total (every verdict produces a deterministic next-state).

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-`nf_hook_entries` refcount uses `Refcount` (saturating); RCU-protected | § Mandatory |
| **CONSTIFY** | per-module `nf_hook_ops` instances `static const` | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, CONSTIFY**: see above
- **AUTOSLAB**: `nf_hook_entries` slab cache
- **MEMORY_SANITIZE**: freed hook entries cleared
- **USERCOPY**: NLA parsing uses bound-checked accessors at consumer Tier-3s
- **KERNEXEC**: hook dispatch via `static const fn-ptr` (each `nf_hookfn` registered as static const)

### Row-2 / GR-RBAC integration

- LSM hook `security_capable(CAP_NET_ADMIN)` already required for nf_register_net_hook from netlink-driven mutation paths.
- Default useful GR-RBAC policy: deny netfilter hook registration outside gradm-marked `firewall_admin` role; arbitrary hook registration is a sandbox-escape vector.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Open Questions

(none — netfilter hook framework exhaustively specified by upstream)

## Out of Scope

- Per-consumer modules (cross-ref `net/netfilter/conntrack-core.md`, `nft-api.md`, etc.)
- Per-AF AF-specific hook semantics (cross-ref `net/ipv4/netfilter/00-overview.md`, `net/ipv6/netfilter/00-overview.md`, `net/bridge/netfilter/00-overview.md` once added)
- 32-bit-only paths
- Implementation code
