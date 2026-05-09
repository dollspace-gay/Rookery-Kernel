---
title: "Tier-3: net/ipv6/flowlabel — IPv6 Flow Label management (RFC 6437)"
tags: ["design-doc", "tier-3", "net"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for IPv6 flow label management (RFC 6437): the per-netns label-allocation table accessed via `IPV6_FLOWLABEL_MGR` setsockopt, the per-socket flow-label state, the `auto_flowlabel` per-netns mode (kernel auto-generates a flow label per outgoing 5-tuple per RFC 6437 § 3), `auto_flowlabel_state_table` for stable per-flow labels, and `/proc/net/ip6_flowlabel` enumeration.

The 20-bit flow label appears in the lower 20 bits of `struct ipv6hdr.flow_lbl[3]`. Used for ECMP hashing, intermediate-router-stateless flow identification, and (on egress) by network-policy applied to flows.

Sub-tier-3 of `net/ipv6/00-overview.md`.

### Requirements

- REQ-1: `IPV6_FLOWLABEL_MGR` setsockopt: actions GET/PUT/RENEW; sharing NONE/EXCL/PROCESS/USER/ANY; flags CREATE/EXCL/REFLECT/REMOTE — semantics identical.
- REQ-2: Per-netns flow-label table: per-(label, dst) entries; sharing-mode rules enforced; expiry + linger handled.
- REQ-3: `IPV6_FLOWINFO_SEND` setsockopt: per-socket; if set, kernel emits `np->flow_label` in TX flow-label field; if clear, kernel uses 0 (no flow label).
- REQ-4: `auto_flowlabel=1`: per-flow hash-based label; identical hashing algorithm + per-netns `flowlabel_hashrnd` seed.
- REQ-5: `auto_flowlabel=2`: per-flow state-table cached label; per-netns table; identical eviction policy.
- REQ-6: `flowlabel_consistency=1`: kernel rejects mid-flow change of label; rejection signaled per RFC 6437.
- REQ-7: `flowlabel_state_ranges=1`: per-netns flow-label-space split into [low, mid) auto-only + [mid, high) app-only ranges; user requests in auto range fail with EINVAL.
- REQ-8: `flowlabel_reflect`: RX cache of remote flow label per (saddr, daddr) tuple; on TX response, reflect the cached label per RFC 7690.
- REQ-9: Per-netns `fl_max_size` cap; on overflow → EAGAIN to user; existing entries continue to function.
- REQ-10: `IPV6_RECVFLOWINFO` setsockopt: when set, cmsg `IPV6_FLOWINFO` carries the received flow info to recvmsg.
- REQ-11: `/proc/net/ip6_flowlabel` format byte-identical.
- REQ-12: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: `IPV6_FLOWLABEL_MGR` GET test: setsockopt with `flr_action=IPV6_FL_A_GET`, `flr_label=0`, `flr_flags=IPV6_FL_F_CREATE` → label allocated; subsequent setsockopt with `flr_label=alloc'd` returns same. (covers REQ-1)
- [ ] AC-2: SHARE_USER test: socket A allocates label with `IPV6_FL_S_USER`; socket B (same uid) GET on same label → succeeds. (covers REQ-1, REQ-2)
- [ ] AC-3: Auto-flowlabel test: with `auto_flowlabel=1`, two TCP connections from same src to different dsts have different labels; same connection has stable label across packets. (covers REQ-4)
- [ ] AC-4: Auto-flowlabel mode 2 test: across kernel reboots (with stable state table per-netns), same 5-tuple always yields same label. (covers REQ-5)
- [ ] AC-5: Consistency test: with `flowlabel_consistency=1`, a TCP connection's mid-stream IPv6 packet with different label triggers rejection per RFC 6437. (covers REQ-6)
- [ ] AC-6: State-ranges test: with `flowlabel_state_ranges=1`, IPV6_FL_A_GET request for a label in the auto range → EINVAL. (covers REQ-7)
- [ ] AC-7: Reflect test: with `flowlabel_reflect=3` (TCP+UDP), incoming flow label cached; outgoing reply uses cached value. (covers REQ-8)
- [ ] AC-8: Max-size test: allocate `fl_max_size + 1` labels → EAGAIN on the +1th. (covers REQ-9)
- [ ] AC-9: `IPV6_RECVFLOWINFO=1`: cmsg from recvmsg carries `IPV6_FLOWINFO` with received-flowinfo. (covers REQ-10)
- [ ] AC-10: `cat /proc/net/ip6_flowlabel` byte-identical content. (covers REQ-11)
- [ ] AC-11: Hardening section present and follows template. (covers REQ-12)

### Architecture

### Rust module organization

- `kernel::net::ipv6::flowlabel::FlowLabelMgr` — top-level manager
- `kernel::net::ipv6::flowlabel::Table` — per-netns flow-label table
- `kernel::net::ipv6::flowlabel::Entry` — `struct ip6_flowlabel`
- `kernel::net::ipv6::flowlabel::Sharing` — IPV6_FL_S_* sharing rules
- `kernel::net::ipv6::flowlabel::Auto` — `auto_flowlabel` (mode 1 hash)
- `kernel::net::ipv6::flowlabel::AutoState` — `auto_flowlabel=2` state table
- `kernel::net::ipv6::flowlabel::Reflect` — `flowlabel_reflect` cache
- `kernel::net::ipv6::flowlabel::netlink::FlMgrNetlink` — IPV6_FLOWLABEL_MGR setsockopt
- `kernel::net::ipv6::flowlabel::proc::ProcFlowlabel` — /proc/net/ip6_flowlabel

### Locking and concurrency

- **Per-netns `ip6_fl_lock`** (rwlock): per-netns flow-label table
- **Per-netns `flowlabel_state_table_lock`** (spinlock): auto-mode-2 state table
- **Per-netns `flowlabel_reflect_lock`** (per-bucket spinlock): reflect cache
- **RCU**: per-flow-label entry refcount; lookups RCU-side

### Error handling

- `Err(EINVAL)` — bad action / bad sharing / range violation
- `Err(EEXIST)` — `IPV6_FL_F_EXCL` and label exists
- `Err(EAGAIN)` — `fl_max_size` reached
- `Err(EPERM)` — sharing-mode violation (e.g., NONE → another socket)

### Out of Scope

- IPv6 RX/TX path integration (cross-ref `net/ipv6/ip6-input.md`, `ip6-output.md`)
- 32-bit-only paths
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Per-netns flow-label table + IPV6_FLOWLABEL_MGR + auto_flowlabel | `net/ipv6/ip6_flowlabel.c` |
| Public API | `include/net/ipv6.h` (struct ip6_flowlabel) |
| UAPI | `include/uapi/linux/in6.h` (struct in6_flowlabel_req, IPV6_FL_*) |

### compatibility contract

### `IPV6_FLOWLABEL_MGR` setsockopt

`struct in6_flowlabel_req`:
```c
struct in6_flowlabel_req {
    struct in6_addr flr_dst;
    __be32          flr_label;
    __u8            flr_action;
    __u8            flr_share;
    __u16           flr_flags;
    __u16           flr_expires;
    __u16           flr_linger;
    __u32           __flr_pad;
};
```

Actions:
- `IPV6_FL_A_GET` (1) — get/allocate label
- `IPV6_FL_A_PUT` (2) — release label
- `IPV6_FL_A_RENEW` (3) — renew/refresh

Sharing modes:
- `IPV6_FL_S_NONE` (0) — exclusive to this socket
- `IPV6_FL_S_EXCL` (1) — exclusive (synonym)
- `IPV6_FL_S_PROCESS` (2) — shareable within process
- `IPV6_FL_S_USER` (3) — shareable by uid
- `IPV6_FL_S_ANY` (255) — anyone

Flags:
- `IPV6_FL_F_CREATE` (1) — create if absent
- `IPV6_FL_F_EXCL` (2) — fail if present
- `IPV6_FL_F_REFLECT` (4) — passive reflection of received label
- `IPV6_FL_F_REMOTE` (8) — share with remote peer

Wire format byte-identical so `ip -6 fl` userspace works unchanged.

### `/proc/net/ip6_flowlabel`

Per-flow-label line: `Label dst saddr fl owner expires linger sharing`. Format-byte-identical so existing parsers work.

### Auto flow-label

When `/proc/sys/net/ipv6/auto_flowlabel=1` (default on), kernel auto-generates per-flow labels for outgoing TCP/UDP/etc. via `ip6_make_flowlabel`:
- Hash of (saddr, daddr, sport, dport, ipproto) plus per-netns hashrnd → 20-bit label
- For TCP: stable across the connection (cached in tcp_sock)
- For UDP: per-flow consistency by re-hashing same 5-tuple

Per RFC 6437 § 3 stateless-ID requirement.

### `auto_flowlabel_state_table` (mode 2)

When `auto_flowlabel=2`, kernel maintains a per-(saddr, daddr, sport, dport, proto) 5-tuple → label state table for stable cross-system labels. Identical algorithm.

### Flow-label scope on receive

Flow label received in incoming IPv6 header is accessible via `IPV6_RECVFLOWINFO` cmsg. Identical.

### sysctls

- `/proc/sys/net/ipv6/auto_flowlabel` (default `1`): 0=off, 1=hash-based, 2=hash-based + state table
- `/proc/sys/net/ipv6/flowlabel_consistency` (default `1`): per RFC 6437 forbid mid-flow change
- `/proc/sys/net/ipv6/flowlabel_state_ranges` (default `0`): split flow-label space into (auto-only, app-only) ranges
- `/proc/sys/net/ipv6/flowlabel_reflect` (default `0`): bitmap controlling RFC 7690 + reply reflection of received labels
- `/proc/sys/net/ipv6/fl_max_size` (default `4096`): max per-netns flowlabel-table size

Format byte-identical.

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Per-netns flow-label table insert/erase under ip6_fl_lock | `kani::proofs::net::ipv6::flowlabel::table_safety` |
| Sharing-mode predicates (NONE / PROCESS / USER / ANY) | `kani::proofs::net::ipv6::flowlabel::share_safety` |
| Auto-flowlabel hash arithmetic (no overflow on 5-tuple inputs) | `kani::proofs::net::ipv6::flowlabel::auto_safety` |
| Reflect-cache insert/lookup | `kani::proofs::net::ipv6::flowlabel::reflect_safety` |

### Layer 2: TLA+ models

(none mandatory at this level)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-netns flow-label table | every entry's hash matches `flowlabel_hashfn(label, dst)` | `kani::proofs::net::ipv6::flowlabel::table_invariants` |
| Per-entry sharing | sharing-mode is in allowed set; owner-pid/uid populated correspondingly | `kani::proofs::net::ipv6::flowlabel::sharing_invariants` |
| `fl_max_size` accounting | per-netns count = Σ live entries in the per-netns table | `kani::proofs::net::ipv6::flowlabel::count_invariants` |

### Layer 4: Functional correctness (opt-in)

- **Auto-flowlabel determinism theorem** via Verus — proves: same `(saddr, daddr, sport, dport, proto, hashrnd)` tuple always yields same label; no state pollution.
- **Reflect-soundness theorem** via Verus — proves: reflected label on TX equals last-cached label for incoming traffic on the same (saddr, daddr) tuple within `MAX_FLOWLABEL_REFLECT_AGE`.

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-flow-label entry refcount uses `Refcount` (saturating) | § Mandatory |
| **AUTOSLAB** | per-`ip6_flowlabel` slab cache | § Mandatory |
| **MEMORY_SANITIZE** | freed flow-label entries cleared (label is privacy-sensitive on shared machines) | § Default-on configurable off |
| **LATENT_ENTROPY** | per-netns `flowlabel_hashrnd` initialized from kernel CSPRNG (RFC 6437 § 6.1) | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE, LATENT_ENTROPY**: see above
- **CONSTIFY**: setsockopt action dispatch table `static const`
- **USERCOPY**: setsockopt buffer copy uses `copy_from_user`
- **SIZE_OVERFLOW**: hash + per-netns count arithmetic uses checked operators

### Row-2 / GR-RBAC integration

- LSM hook `security_socket_setsockopt` for IPV6_FLOWLABEL_MGR; GR-RBAC policy can deny per-subject (default empty).
- Useful default policy: deny IPV6_FLOWLABEL_MGR with sharing-modes USER/ANY for non-root subjects (privacy: stable cross-process flow IDs are tracking primitives).

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

