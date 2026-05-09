---
title: "Tier-3: net/xfrm/interface — XFRMi virtual netdev (`xfrm_interface_core.c`)"
tags: ["design-doc", "tier-3", "net", "ipsec"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for the XFRMi virtual netdev (`type xfrm`) — a routing-and-marking abstraction that lets userspace bind XFRM policies to a per-interface `if_id` rather than to a packet selector. Created via `ip link add foo type xfrm dev <underlying> if_id <id>`; packets routed through `foo` get classified with `if_id` for SPD lookup, and policies installed with `XFRMA_IF_ID=<id>` match only those packets. Replaces the older `vti`/`vti6` virtual-tunnel interfaces with a unified, AF-agnostic model.

Useful for multi-tenant IPSec deployments (same SAs, separate policy per tenant via `if_id`), SD-WAN, and route-based VPN topologies (e.g., FortiGate-style "interface-based VPN"). Pairs with `net/xfrm/state.md` (SAs carry `if_id`), `net/xfrm/policy.md` (policies match on `if_id`).

Sub-tier-3 of `net/xfrm/00-overview.md`. Replaces older `vti.c` / `vti6.c` virtual-tunnel netdevs with a unified model. The xfrm_interface_bpf.c sub-component exposes per-XFRMi BPF integration for traffic classification.

### Requirements

- REQ-1: rtnl link type `xfrm` registered; `IFLA_XFRM_LINK` / `IFLA_XFRM_IF_ID` / `IFLA_XFRM_COLLECT_METADATA` NLAs parsed identically.
- REQ-2: XFRMi xmit: sets `flowi.flowi_xfrm_if_id = if_id`; SPD lookup matches policies with `policy.if_id == if_id`; bundle build proceeds.
- REQ-3: XFRMi RX: post-decap routing classifies packets by `xfrm_state.if_id` to XFRMi netdev; subsequent RX path treats packet as XFRMi-arrived.
- REQ-4: Per-XFRMi mark/mask configuration: when set, packet `skb->mark` adjusted via `(mark & ~mask) | (if_id & mask)` semantics. Identical to upstream.
- REQ-5: COLLECT_METADATA mode: kernel attaches `xfrm_md_info` to skbs; BPF programs can read.
- REQ-6: Per-XFRMi BPF integration via `xfrm_interface_bpf.c`: kfunc set exposes per-XFRMi metadata accessors to BPF-XDP / BPF-TC programs.
- REQ-7: Per-netns XFRMi registration: created in netns N → policies + SAs scoped to netns N.
- REQ-8: `struct xfrm_md_info` byte-identical layout.
- REQ-9: XFRMi feature parity with deprecated `vti` / `vti6` virtual-tunnel: GRE-encap / arp / multicast disabled per upstream policy (XFRMi is a routing aid, not an encapsulator).
- REQ-10: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: `pahole struct xfrm_md_info` byte-identical layout. (covers REQ-8)
- [ ] AC-2: `ip link add foo type xfrm dev eth0 if_id 0x42` test: XFRMi netdev created; `ip -d link show foo` shows `xfrm if_id 0x42` link info. (covers REQ-1)
- [ ] AC-3: Policy-match test: install policy with `XFRMA_IF_ID=0x42`; route `ip route add 10.0.0.0/24 dev foo`; TX to 10.0.0.5 → matches policy; TX to 192.168.0.5 (default route via eth0) → no match. (covers REQ-2)
- [ ] AC-4: RX classification test: install SA with `if_id=0x42`; receive ESP packet with matching SPI → kernel decapsulates + sets `skb->dev = foo`; tcpdump on `foo` sees decap'd traffic. (covers REQ-3)
- [ ] AC-5: Mark-mask test: configure XFRMi with mark=0x100, mask=0xff00; outgoing packet's mark becomes `(orig_mark & ~0xff00) | (0x100 & 0xff00)`. (covers REQ-4)
- [ ] AC-6: COLLECT_METADATA test: enable IFLA_XFRM_COLLECT_METADATA; BPF-XDP program reads `xfrm_md_info.if_id` from per-skb metadata. (covers REQ-5, REQ-6)
- [ ] AC-7: Per-netns test: create XFRMi `foo` in netns A with if_id=0x42; netns B's policies with if_id=0x42 don't match netns-A traffic. (covers REQ-7)
- [ ] AC-8: Multi-tenant test: 2 XFRMis (foo if_id=0x42, bar if_id=0x43); per-XFRMi distinct policy; tcpdump shows traffic correctly classified per `if_id`. (covers REQ-2)
- [ ] AC-9: vti-feature-deprecation test: `ip link add foo type xfrm` → `ip -d link show foo` does not list `gre`/`arp`/`multicast` features (XFRMi is no-encap routing). (covers REQ-9)
- [ ] AC-10: Hardening section present and follows template. (covers REQ-10)

### Architecture

### Rust module organization

- `kernel::net::xfrm::iface::Xfrmi` — netdev type root
- `kernel::net::xfrm::iface::Rtnl` — rtnl link ops (alloc/setup/dellink/changelink)
- `kernel::net::xfrm::iface::Xmit` — `xfrmi_xmit` TX path
- `kernel::net::xfrm::iface::Rx` — `xfrmi_input` RX classification
- `kernel::net::xfrm::iface::MarkMask` — per-XFRMi mark/mask transformation
- `kernel::net::xfrm::iface::Metadata` — `xfrm_md_info` attach + BPF accessors
- `kernel::net::xfrm::iface::bpf::XfrmiBpfKfunc` — kfunc set for BPF programs
- `kernel::net::xfrm::iface::Registry` — per-netns (underlying-link, if_id) → XFRMi map

### Locking and concurrency

- **Per-netns XFRMi registry** (rwlock): protects (underlying, if_id) → XFRMi map mutator
- **rtnl_lock** (system-wide): held during XFRMi create/destroy via netlink
- **Per-XFRMi `dev_lock`**: standard netdev locking
- **RCU**: lookup hot path is RCU-side

### Error handling

- `Err(EINVAL)` — bad NLA / bad if_id (e.g., 0 reserved)
- `Err(EEXIST)` — XFRMi with same (underlying, if_id) already exists
- `Err(ENODEV)` — bad IFLA_XFRM_LINK
- `Err(ENOMEM)` — alloc fail
- `Err(EPERM)` — non-CAP_NET_ADMIN

### Out of Scope

- Underlying SA/policy storage (cross-ref `net/xfrm/state.md`, `net/xfrm/policy.md`)
- Per-AF transform handling (cross-ref `net/xfrm/ah-esp-ipv4.md`, `net/xfrm/ah-esp-ipv6.md`)
- Older `vti.c` / `vti6.c` (deprecated; XFRMi is the replacement)
- 32-bit-only paths
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| XFRMi netdev: rtnl ops, xmit, RX, link configuration | `net/xfrm/xfrm_interface_core.c` |
| Per-XFRMi BPF integration | `net/xfrm/xfrm_interface_bpf.c` |
| Public API | `include/net/xfrm.h` |
| UAPI: rtnetlink IFLA_XFRM_LINK / IFLA_XFRM_IF_ID / IFLA_XFRM_COLLECT_METADATA | `include/uapi/linux/if_link.h` |

### compatibility contract

### `ip link add foo type xfrm dev <underlying> if_id <id>`

Creates an XFRMi netdev `foo`:
- `IFLA_XFRM_LINK` — ifindex of underlying physical netdev
- `IFLA_XFRM_IF_ID` — 32-bit if_id (matches `xfrm_state.if_id` + `xfrm_policy.if_id`)
- `IFLA_XFRM_COLLECT_METADATA` — boolean; when set, kernel attaches `xfrm_md` metadata to skbs (BPF-readable)

After creation, `ip route add ... dev foo` routes traffic through this XFRMi; SPD lookup uses `if_id` as a selector match field.

Wire format byte-identical so iproute2 / NetworkManager-strongswan / SD-WAN tooling work unchanged.

### XFRMi xmit path

When a packet's `skb_dst()` resolves to an XFRMi netdev:
1. `xfrmi_xmit` invoked
2. Set `skb->mark = if_id` (via per-XFRMi mark/mask configuration if set, or directly)
3. Look up `flowi.flowi_xfrm_if_id = if_id` 
4. `xfrm_lookup` with this flowi → SPD lookup matches policies where `policy.if_id == if_id`
5. Bundle build → standard XFRM TX

Identical sequence so policies installed with `XFRMA_IF_ID=<id>` match only XFRMi-routed traffic.

### XFRMi RX path

When IPSec-decapsulated packet has `xfrm_state.if_id` matching a configured XFRMi netdev:
1. `__xfrm_input` post-decap calls `xfrmi_input` if SA's `if_id` is non-zero
2. Look up XFRMi by (underlying, if_id) → set `skb->dev = xfrmi_netdev`
3. Re-inject via standard RX path; subsequent routing-stack treats packet as having arrived on XFRMi

Identical so received traffic is classifiable by ifindex of XFRMi.

### `IFLA_XFRM_COLLECT_METADATA` mode

When set, kernel attaches a `struct xfrm_md_info` per-skb metadata block carrying `if_id` + decap'd SA pointer. BPF programs (per `xfrm_interface_bpf.c`) can read this metadata for fine-grained classification (e.g., per-tenant policy enforcement at netfilter level).

Identical mode + bpf-tap-point.

### Per-namespace XFRMi registration

XFRMi netdevs are per-netns; created in netns N can route to netns-N SPD/SADB only. Identical netns isolation.

### `struct xfrm_md_info` (BPF-visible metadata)

```c
struct xfrm_md_info {
    u32 if_id;
    int link;
    struct dst_entry *dst_orig;
};
```

Layout-byte-identical for BPF-program access stability.

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| rtnl link create/destroy under rtnl_lock | `kani::proofs::net::xfrm::iface::rtnl_safety` |
| Per-netns registry insert/erase | `kani::proofs::net::xfrm::iface::registry_safety` |
| `xfrmi_xmit` flowi if_id propagation (no stale read) | `kani::proofs::net::xfrm::iface::xmit_safety` |
| `xfrmi_input` post-decap dev assignment | `kani::proofs::net::xfrm::iface::rx_safety` |
| Mark-mask arithmetic (no overflow) | `kani::proofs::net::xfrm::iface::mark_safety` |

### Layer 2: TLA+ models

(none new owned at this granularity)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-netns (underlying, if_id) → XFRMi map | uniqueness: `(link, if_id)` maps to ≤1 XFRMi | `kani::proofs::net::xfrm::iface::registry_invariants` |
| Per-XFRMi metadata | `xfrm_md_info.if_id` matches owning XFRMi's `if_id` | `kani::proofs::net::xfrm::iface::md_invariants` |

### Layer 4: Functional correctness (opt-in)

- **XFRMi classification soundness theorem** via Verus — proves: ∀ packet `p` routed through XFRMi `foo` with if_id `i`, `__xfrm_policy_lookup_bytype` matches only policies with `policy.if_id == i` (or unrestricted policies); never matches other-tenant policies.

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-XFRMi netdev refcount uses `Refcount` (saturating) | § Mandatory |
| **AUTOSLAB** | per-XFRMi private-data slab cache | § Mandatory |
| **CONSTIFY** | rtnl link-ops vtable + netdev-ops vtable `static const` | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, CONSTIFY**: see above
- **MEMORY_SANITIZE**: freed XFRMi private data cleared (carries if_id + per-tenant identifier in some deployments)
- **USERCOPY**: rtnl NLA parsing uses bound-checked accessors
- **SIZE_OVERFLOW**: mark/mask arithmetic uses checked operators

### Row-2 / GR-RBAC integration

- LSM hook `security_capable(CAP_NET_ADMIN)` already required for `ip link add type xfrm`; GR-RBAC policy can deny per-subject (default empty).
- Useful default GR-RBAC policy: deny XFRMi creation outside gradm-marked `network_admin` role; multi-tenant XFRMi setups need careful auth gate.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

