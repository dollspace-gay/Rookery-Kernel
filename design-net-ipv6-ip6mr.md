---
title: "Tier-3: net/ipv6/ip6mr — IPv6 multicast routing (MFC + PIM-SM kernel side)"
tags: ["design-doc", "tier-3", "net"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for the IPv6 multicast routing subsystem — the kernel side of PIM-SMv6 / MLD-snoop / static multicast forwarding, controlled by a userspace daemon (pim6d, mrd6, smcrouted). Implements the multicast forwarding cache (MFC6), virtual interface (mif6) registry, the `MRT6_*` setsockopts that the userspace daemon uses to install MFC entries + register MIFs, kernel-driven MFC miss notification (sent up to the daemon as an `MRT6MSG_NOCACHE`-style msg over the registered raw socket), and PIM-SM-relevant tunnel-encapsulation. Cooperates with `net/ipv6/mcast.c` (MLDv1/v2 listener-side, sender-side joins) and the netfilter conntrack-style multicast helpers.

Sub-tier-3 of `net/ipv6/00-overview.md`. Hot path for IPTV / video distribution / financial-data multicast / IPv6 SSM. Not used by typical hosts (forwarding-disabled by default); essential for IPv6 multicast routers.

### Requirements

- REQ-1: All MRT6_* setsockopts (per the table above) implemented identically; CAP_NET_ADMIN gated.
- REQ-2: `struct mif6ctl`, `struct mf6cctl`, `struct sioc_sg_req6` byte-identical layout.
- REQ-3: Per-netns `mr6_table` primary + secondary tables (selectable via `MRT6_TABLE`); identical multi-table semantics.
- REQ-4: MFC entry storage: per-(origin, mcastgrp) hash bucket; per-entry `mif_count` + ifset bitmap of forward-out MIFs; identical algorithm.
- REQ-5: MIF6 registry: per-table per-MIF entry with vif/pifi mapping, threshold, rate-limit; identical.
- REQ-6: RX forwarding: on multicast packet receipt, lookup MFC; on hit → forward out per ifset bitmap (skb_clone per egress MIF + xmit) decremented hop-limit per RFC 8200 § 3; on miss → MFC-miss notification path.
- REQ-7: MFC-miss queue: bounded `MFC_QUEUE_LEN_MAX=3`; on overflow → drop oldest; expire after `MFC_TIMEOUT=10s`.
- REQ-8: MRT6MSG_NOCACHE / MRT6MSG_WHOLEPKT / MRT6MSG_WRONGMIF wire format byte-identical to upstream; emitted at identical decision points.
- REQ-9: PIM Register encapsulation when `MRT6_PIM=1`: RFC 7761 § 4.4 wire format identical.
- REQ-10: NETLINK_ROUTE RTM_NEWROUTE/DELROUTE/GETROUTE for AF_INET6 with `RTN_MULTICAST` family-route — wire format byte-identical (alternative to MRT6_ADD_MFC).
- REQ-11: `/proc/net/ip6_mr_cache` + `/proc/net/ip6_mr_vif` content format byte-identical.
- REQ-12: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: `pahole struct mif6ctl` + `pahole struct mf6cctl` byte-identical. (covers REQ-2)
- [ ] AC-2: pim6d test: spawn pim6d on a router with 2 IPv6-multicast-enabled ifaces; multicast traffic from src behind iface 0 to group `ff3e::1234` joined behind iface 1 → forwarded; tcpdump shows decremented hoplimit. (covers REQ-1, REQ-6)
- [ ] AC-3: MFC miss test: spawn pim6d (or mock daemon); send multicast to a (origin, group) with no MFC entry → daemon receives MRT6MSG_NOCACHE within `< MFC_TIMEOUT/2`; kernel queue length ≤ 3. (covers REQ-7, REQ-8)
- [ ] AC-4: PIM Register test: enable MRT6_PIM=1; multicast to a group with no MFC → kernel encapsulates in PIM Register and sends to RP. (covers REQ-9)
- [ ] AC-5: NETLINK RTN_MULTICAST install test: `ip -6 mroute add ...` (when iproute2 supports v6 mcast) → MFC entry visible in `/proc/net/ip6_mr_cache`. (covers REQ-10)
- [ ] AC-6: Multi-table test: open two MRT6_TABLE-distinct sockets; install entries in each; verify per-table isolation. (covers REQ-3)
- [ ] AC-7: Threshold test: MIF with `vifc_threshold=4`; packet with hoplimit < 4 → not forwarded out that MIF. (covers REQ-5)
- [ ] AC-8: `cat /proc/net/ip6_mr_cache` + `cat /proc/net/ip6_mr_vif` byte-identical. (covers REQ-11)
- [ ] AC-9: smcroute test (multicast-statically-configured): smcrouted with v6 entries → kernel forwards according to static rules. (covers REQ-1, REQ-6)
- [ ] AC-10: Hardening section present and follows template. (covers REQ-12)

### Architecture

### Rust module organization

- `kernel::net::ipv6::ip6mr::Ip6mr` — top-level entrypoint
- `kernel::net::ipv6::ip6mr::Mr6Table` — per-netns + per-table registry
- `kernel::net::ipv6::ip6mr::Mfc6Entry` — `struct mfc6_cache`
- `kernel::net::ipv6::ip6mr::Mif6Entry` — `struct vif_device` (mif equivalent)
- `kernel::net::ipv6::ip6mr::Forward` — multicast packet forwarding
- `kernel::net::ipv6::ip6mr::Unresolved` — MFC-miss queue + daemon notify
- `kernel::net::ipv6::ip6mr::PimRegister` — PIM Register encapsulator
- `kernel::net::ipv6::ip6mr::sockopt::Mrt6Sockopt` — MRT6_* setsockopt
- `kernel::net::ipv6::ip6mr::netlink::RtmMcastHandler` — RTM_*ROUTE with RTN_MULTICAST
- `kernel::net::ipv6::ip6mr::proc::ProcIp6Mr` — `/proc/net/ip6_mr_*`

### Locking and concurrency

- **Per-netns + per-table `mr6_lock`** (rwlock): protects MFC + MIF registries
- **Per-MFC-entry `lock`** (spinlock): protects unresolved-queue draining
- **`mr6t_unresolved_lock` per-table**: protects unresolved-list mutator
- **RCU**: MFC lookup hot path is RCU-side

### Error handling

- `Err(EPERM)` — non-CAP_NET_ADMIN
- `Err(EBUSY)` — MRT6_INIT on already-owned table
- `Err(EADDRINUSE)` — duplicate MIF/MFC
- `Err(ENOENT)` — del of non-existent
- `Err(ENOMEM)` — alloc fail

### Out of Scope

- MLDv1/v2 listener-side semantics in detail (cross-ref future `net/ipv6/mcast.md` Tier-3 — listener-side joins, IGMPv3-equivalent reports/queries; `mcast.c` is governed there)
- 32-bit-only paths
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| MFC6 cache + mif6 registry + MRT6_* setsockopt + RX forwarding decision | `net/ipv6/ip6mr.c` |
| MLDv1/v2 listener-side (per-socket joins) + sender-side filters + IGMPv3-equivalent | `net/ipv6/mcast.c` |
| Public API | `include/net/ip6_route.h` |
| UAPI | `include/uapi/linux/mroute6.h` (MRT6_*, struct mif6ctl, struct mf6cctl, struct sioc_sg_req6) |

### compatibility contract

### Userspace control via raw socket + setsockopt

Userspace daemon (pim6d, smcrouted) opens `socket(AF_INET6, SOCK_RAW, IPPROTO_ICMPV6)` (or with a cooperating module IPPROTO_PIM=103) and uses MRT6_* setsockopts to drive routing:

| Setsockopt | Operation |
|---|---|
| `MRT6_INIT` | take ownership; per-netns mr6_table primary |
| `MRT6_DONE` | release ownership |
| `MRT6_ADD_MIF` / `MRT6_DEL_MIF` | register / unregister virtual interface |
| `MRT6_ADD_MFC` / `MRT6_DEL_MFC` | add / delete forwarding cache entry |
| `MRT6_ADD_MFC_PROXY` / `MRT6_DEL_MFC_PROXY` | proxy/wildcard entries |
| `MRT6_FLUSH` | flush all entries |
| `MRT6_TABLE` | select non-default mr6 table |
| `MRT6_PIM` | enable PIM mode |
| `MRT6_ASSERT` | enable Assert messages |

Wire format byte-identical so existing pim6d/smcrouted/mrd6 work unchanged.

### `struct mif6ctl` layout

```c
struct mif6ctl {
    mifi_t      mif6c_mifi;
    unsigned char mif6c_flags;
    unsigned char vifc_threshold;
    __u16        mif6c_pifi;        /* physical iface ifindex */
    unsigned int vifc_rate_limit;
};
```

Layout-byte-identical.

### `struct mf6cctl` layout

```c
struct mf6cctl {
    struct sockaddr_in6 mf6cc_origin;
    struct sockaddr_in6 mf6cc_mcastgrp;
    mifi_t              mf6cc_parent;
    struct if_set       mf6cc_ifset;        /* IF_SETSIZE bits */
};
```

Layout-byte-identical.

### `/proc/net/ip6_mr_cache` + `/proc/net/ip6_mr_vif`

Per-MFC entry + per-MIF entry tabular formats. Format-byte-identical so `smcroute -f` and similar tooling work.

### MFC miss → daemon notification

When forwarding sees a multicast packet whose (origin, mcastgrp) doesn't match any MFC entry, kernel:
1. Allocates an MFC "unresolved" entry
2. Queues the skb to the unresolved entry's `q` list (bounded by `MFC_QUEUE_LEN_MAX = 3`)
3. Sends an MRT6MSG_NOCACHE message up the registered MRT6 raw socket to the daemon
4. Daemon installs the actual MFC entry; kernel drains the queue

Identical behavior so existing daemons see identical MFC-miss events.

### PIM register encapsulation

When PIM mode (`MRT6_PIM`) is enabled, kernel-side encapsulates MFC-miss-to-RP packets in PIM Register messages (RFC 7761 § 4.4). Identical encapsulation algorithm.

### NETLINK_ROUTE RTM_NEWROUTE/DELROUTE for AF_INET6 with `RTN_MULTICAST`

Alternative interface (vs. setsockopt) for installing MFC entries via netlink. Wire format byte-identical so `iproute2`-driven mcast routing works.

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| MFC bucket insert/erase under mr6_lock | `kani::proofs::net::ipv6::ip6mr::mfc_safety` |
| Multicast forward (skb_clone per MIF in ifset bitmap; no double-free) | `kani::proofs::net::ipv6::ip6mr::forward_safety` |
| Unresolved-queue insert + drain | `kani::proofs::net::ipv6::ip6mr::unresolved_safety` |
| PIM Register encapsulation header build | `kani::proofs::net::ipv6::ip6mr::pim_safety` |
| MRT6_* setsockopt option-table walk | `kani::proofs::net::ipv6::ip6mr::sockopt_safety` |

### Layer 2: TLA+ models

(none mandatory at this level)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| MFC entry | `mif_count` matches popcount of ifset bitmap | `kani::proofs::net::ipv6::ip6mr::mfc_invariants` |
| Unresolved-queue per-MFC | queue length ≤ `MFC_QUEUE_LEN_MAX=3`; entries are FIFO-ordered | `kani::proofs::net::ipv6::ip6mr::unresolved_invariants` |
| MIF6 registry | per-MIF `pifi` resolves to a real netdev's ifindex | `kani::proofs::net::ipv6::ip6mr::mif_invariants` |

### Layer 4: Functional correctness (opt-in)

- **Forwarding-out-correctness theorem** via Verus — proves: ∀ multicast skb hitting MFC entry `e`, the set of egress MIFs equals exactly the bits set in `e.mfc_ifset`; no MIF forwarded twice; no MIF skipped.

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | mfc6_cache + per-MIF refcounts use `Refcount` (saturating) | § Mandatory |
| **AUTOSLAB** | per-`mfc6_cache` slab cache | § Mandatory |
| **MEMORY_SANITIZE** | freed mfc6_cache + drained unresolved-queue skbs cleared | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: see above
- **CONSTIFY**: MRT6_* setsockopt dispatch table `static const`
- **USERCOPY**: setsockopt buffer copy via `copy_from_user`
- **SIZE_OVERFLOW**: ifset-bitmap arithmetic uses checked operators
- **KERNEXEC**: setsockopt dispatch via `static const fn-ptr` array

### Row-2 / GR-RBAC integration

- LSM hook `security_capable(CAP_NET_ADMIN)` already required for MRT6_*; GR-RBAC policy can deny per-subject (default empty).
- Useful default GR-RBAC policy: deny MRT6_* outside gradm-marked `mcast_router_admin` role; multicast routing can be a DoS amplifier.
- DoS-prevention: `MFC_QUEUE_LEN_MAX=3` per-unresolved-MFC bound prevents memory-exhaustion via flood of MFC misses.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

