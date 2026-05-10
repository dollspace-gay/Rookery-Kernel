# Tier-3: net/netfilter/conntrack-netlink — NFNL_SUBSYS_CTNETLINK control interface

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/netfilter/nf_conntrack_netlink.c
  - net/netfilter/nf_conntrack_acct.c
  - net/netfilter/nf_conntrack_ecache.c
  - net/netfilter/nf_conntrack_labels.c
  - net/netfilter/nf_conntrack_seqadj.c
  - net/netfilter/nf_conntrack_timeout.c
  - net/netfilter/nf_conntrack_timestamp.c
  - include/uapi/linux/netfilter/nfnetlink_conntrack.h
-->

## Summary
Tier-3 design for the NETLINK_NETFILTER + NFNL_SUBSYS_CTNETLINK userspace interface for conntrack-tools. Lets userspace daemons (`conntrack` / `conntrackd` / `flowsync` / OVS / Cilium control-plane) inspect, mutate, expire, and synchronize conntrack state. Covers the message types `IPCTNL_MSG_CT_*`, `IPCTNL_MSG_EXP_*`, plus the supporting conntrack-extension modules that make this surface useful: accounting (per-direction counters), event-cache (asynchronous event delivery), labels (128-bit per-flow tags), seqadj (TCP sequence adjustment), per-flow timeout policies, and per-flow timestamps.

Used by:
- `conntrack -L / -E / -F / -G / -I / -D / -U` (conntrack-tools CLI)
- `conntrackd` (cluster-conntrack-replication daemon)
- libnetfilter_conntrack-using monitoring (Suricata, Zeek)
- Cilium / OVS userspace control plane (read-side mostly)

Sub-tier-3 of `net/netfilter/00-overview.md`. Pairs with `net/netfilter/conntrack-core.md` (storage), `net/netfilter/conntrack-helper.md` (helper events emitted via ctnetlink), `net/netfilter/conntrack-proto.md` (per-proto attribute encoding).

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| ctnetlink core: per-IPCTNL_MSG_* handlers, dump, multicast events | `net/netfilter/nf_conntrack_netlink.c` |
| Accounting extension (per-direction byte/packet counters) | `net/netfilter/nf_conntrack_acct.c` |
| Event cache (asynchronous event delivery via ecache) | `net/netfilter/nf_conntrack_ecache.c` |
| Per-flow 128-bit labels | `net/netfilter/nf_conntrack_labels.c` |
| TCP sequence-adjustment for connection-track-based modifications mid-flow | `net/netfilter/nf_conntrack_seqadj.c` |
| Per-flow timeout policy attach (cttimeout) | `net/netfilter/nf_conntrack_timeout.c` |
| Per-flow timestamps | `net/netfilter/nf_conntrack_timestamp.c` |
| UAPI: IPCTNL_MSG_*, CTA_* attrs | `include/uapi/linux/netfilter/nfnetlink_conntrack.h` |

## Compatibility contract

### IPCTNL_MSG_* message types

Per `include/uapi/linux/netfilter/nfnetlink_conntrack.h`:

| Message | Operation |
|---|---|
| `IPCTNL_MSG_CT_NEW` | Create or update conntrack |
| `IPCTNL_MSG_CT_GET` | Lookup or dump |
| `IPCTNL_MSG_CT_DELETE` | Delete |
| `IPCTNL_MSG_CT_GET_CTRZERO` | Dump and zero counters |
| `IPCTNL_MSG_CT_GET_STATS_CPU` | Per-CPU statistics |
| `IPCTNL_MSG_CT_GET_STATS` | Global statistics |
| `IPCTNL_MSG_CT_GET_DYING` | Dump dying entries |
| `IPCTNL_MSG_CT_GET_UNCONFIRMED` | Dump unconfirmed entries |
| `IPCTNL_MSG_EXP_NEW` | Create expectation |
| `IPCTNL_MSG_EXP_GET` | Lookup expectation |
| `IPCTNL_MSG_EXP_DELETE` | Delete expectation |
| `IPCTNL_MSG_EXP_GET_STATS_CPU` | Per-CPU expectation stats |

Wire format byte-identical so `conntrack` works unchanged.

### CTA_* NLA attributes (for IPCTNL_MSG_CT_*)

Top-level attrs:
- `CTA_TUPLE_ORIG / CTA_TUPLE_REPLY` (nested: IP/PROTO inner attrs)
- `CTA_STATUS` (IPS_* status flags as u32)
- `CTA_PROTOINFO` (per-l4proto state — TCP state etc.)
- `CTA_HELP` (helper attached)
- `CTA_NAT_SRC / CTA_NAT_DST` (NAT mapping)
- `CTA_TIMEOUT` (lifetime)
- `CTA_MARK` (32-bit conntrack mark) + `CTA_MARK_MASK`
- `CTA_COUNTERS_ORIG / CTA_COUNTERS_REPLY` (acct counters)
- `CTA_USE` (refcount)
- `CTA_ID` (kernel-assigned 32-bit ID)
- `CTA_NAT_DST_TUPLE` (post-NAT target tuple)
- `CTA_TUPLE_MASTER` (parent for RELATED flows)
- `CTA_SEQ_ADJ_ORIG / CTA_SEQ_ADJ_REPLY` (TCP seq adjustment)
- `CTA_SECMARK` / `CTA_SECCTX` (LSM)
- `CTA_TIMESTAMP` (start/stop)
- `CTA_LABELS` / `CTA_LABELS_MASK` (128-bit labels)
- `CTA_SYNPROXY` (syn-proxy state)
- `CTA_FILTER` (per-dump filter)
- `CTA_ZONE` (per-conntrack zone)
- `CTA_ZONE_MASK`
- `CTA_PAD`

Per-tuple nested attrs:
- `CTA_TUPLE_IP` (CTA_IP_V4_SRC/DST or CTA_IP_V6_SRC/DST)
- `CTA_TUPLE_PROTO` (CTA_PROTO_NUM + per-proto: CTA_PROTO_SRC_PORT/DST_PORT for TCP/UDP, ICMP_TYPE/CODE/ID for ICMP, etc.)

Per-protoinfo nested attrs:
- TCP: CTA_PROTOINFO_TCP_STATE, FLAGS_ORIG/REPLY, WSCALE_ORIG/REPLY
- DCCP / SCTP: per-proto state
- GRE: CTA_PROTOINFO_GRE_TIMEOUT_STREAM, KEY_TIMEOUT, etc.

Wire format byte-identical so libnetfilter_conntrack-built tools work unchanged.

### CTA_EXPECT_* NLA attributes (for IPCTNL_MSG_EXP_*)

- `CTA_EXPECT_TUPLE` (expected tuple)
- `CTA_EXPECT_MASTER` (parent conntrack tuple)
- `CTA_EXPECT_MASK` (mask for partial matching)
- `CTA_EXPECT_TIMEOUT`
- `CTA_EXPECT_ID`
- `CTA_EXPECT_HELP_NAME`
- `CTA_EXPECT_ZONE`
- `CTA_EXPECT_FLAGS`
- `CTA_EXPECT_CLASS`
- `CTA_EXPECT_NAT` (per-NAT-mode expectation)
- `CTA_EXPECT_FN` (per-helper expect function name)

Wire format byte-identical.

### Multicast event groups

```
NFNLGRP_CONNTRACK_NEW      = 1
NFNLGRP_CONNTRACK_UPDATE   = 2
NFNLGRP_CONNTRACK_DESTROY  = 3
NFNLGRP_CONNTRACK_EXP_NEW  = 4
NFNLGRP_CONNTRACK_EXP_UPDATE = 5
NFNLGRP_CONNTRACK_EXP_DESTROY = 6
```

Daemons subscribe via `setsockopt(NETLINK_ADD_MEMBERSHIP, NFNLGRP_CONNTRACK_*)`. Identical groups.

### Event cache (`nf_conntrack_ecache.c`)

Per-conntrack `nf_conntrack_ecache` extension caches per-event flags; deferred event emission via per-CPU work queue to avoid blocking conntrack hot path. Event types: `IPCT_NEW`, `IPCT_RELATED`, `IPCT_DESTROY`, `IPCT_REPLY`, `IPCT_ASSURED`, `IPCT_PROTOINFO`, `IPCT_HELPER`, `IPCT_MARK`, `IPCT_SEQADJ`, `IPCT_NATSEQADJ`, `IPCT_SECMARK`, `IPCT_LABEL`, `IPCT_SYNPROXY`.

Per-event-type opt-in via per-listener subscription bitmap.

### Conntrack accounting (`nf_conntrack_acct.c`)

Per-direction byte/packet counters as conntrack extension. `nf_conntrack_acct` sysctl gates whether new conntracks get acct extension allocated (default: 0, per upstream 4.7+).

### TCP seqadj (`nf_conntrack_seqadj.c`)

When connection-track-based modifications mid-flow modify TCP payload (e.g., FTP NAT helper rewriting PORT command, changing payload length), TCP sequence numbers post-modification need adjustment. seqadj extension tracks per-direction seq offsets and applies on each TCP packet.

### Conntrack timeout (`nf_conntrack_timeout.c` + ctnetlink subsys)

Per-flow timeout policy: instead of using global per-proto-state timeouts, attach a named timeout policy that overrides. Per-policy NLAs: name + family + L4 proto + per-state timeouts. Used for fine-grained per-flow lifetime control (e.g., custom "long-lived TCP" policy).

### Conntrack labels (`nf_conntrack_labels.c`)

128-bit per-flow label bitmap. Used by `cls_flower KEY_CT_LABELS` matching (cross-ref `cls-flower.md`) and OVS for application-tagging.

### Conntrack timestamps (`nf_conntrack_timestamp.c`)

Per-flow start/stop nanosecond timestamps. Useful for flow-duration reporting; gated by `nf_conntrack_timestamp` sysctl (default off).

## Requirements

- REQ-1: All `IPCTNL_MSG_CT_*` + `IPCTNL_MSG_EXP_*` message types parsed identically.
- REQ-2: All `CTA_*` + `CTA_EXPECT_*` NLA attribute schemas byte-identical.
- REQ-3: Multicast event groups (`NFNLGRP_CONNTRACK_*`) registered identically.
- REQ-4: Event cache (`nf_conntrack_ecache.c`): per-conntrack ecache extension; per-event-type bitmap; per-CPU deferred emission.
- REQ-5: Conntrack accounting (`nf_conntrack_acct.c`): per-direction counters; sysctl-gated default off.
- REQ-6: TCP seqadj (`nf_conntrack_seqadj.c`): per-direction seq offset; applied at NF_INET_LOCAL_OUT/POST_ROUTING.
- REQ-7: Conntrack timeout policies (`nf_conntrack_timeout.c`): per-flow timeout-policy attach via cttimeout NLAs; identical UAPI.
- REQ-8: Conntrack labels (`nf_conntrack_labels.c`): 128-bit per-flow bitmap; read/write via CTA_LABELS + CTA_LABELS_MASK.
- REQ-9: Conntrack timestamps (`nf_conntrack_timestamp.c`): per-flow start/stop ns; sysctl-gated default off.
- REQ-10: GET_DYING + GET_UNCONFIRMED dumps: walk per-netns dying-list / unconfirmed-list; identical algorithm.
- REQ-11: Multi-part dump for IPCTNL_MSG_CT_GET / EXP_GET; per-walker cursor identical.
- REQ-12: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `conntrack -L` byte-identical content (modulo timing). (covers REQ-1, REQ-2)
- [ ] AC-2: `conntrack -E` (event mode) test: subscribe NFNLGRP_CONNTRACK_NEW; trigger new flow → daemon receives event with byte-identical CTA_* encoding. (covers REQ-3, REQ-4)
- [ ] AC-3: Acct test: enable `nf_conntrack_acct=1`; new conntrack carries CTA_COUNTERS_ORIG/REPLY; `conntrack -L -o extended` shows them. (covers REQ-5)
- [ ] AC-4: Seqadj test: install FTP NAT helper; FTP control connection PORT command rewritten by helper; subsequent TCP segments have seq adjusted to compensate; receiver sees consistent bytestream. (covers REQ-6)
- [ ] AC-5: Cttimeout test: `nfct add timeout long-tcp inet tcp established 86400`; attach via `iptables CT --timeout long-tcp`; matched flows use 86400s ESTABLISHED timeout. (covers REQ-7)
- [ ] AC-6: Labels test: write CTA_LABELS = 0x1 to existing conntrack; subsequent `conntrack -L` shows label set. (covers REQ-8)
- [ ] AC-7: Timestamp test: enable `nf_conntrack_timestamp=1`; new conntrack carries CTA_TIMESTAMP nested with start_ns; on close, stop_ns set. (covers REQ-9)
- [ ] AC-8: GET_DYING test: trigger conntrack to enter DYING state (e.g., FIN+ACK exchange); IPCTNL_MSG_CT_GET_DYING dump returns it; subsequent GC reaps it. (covers REQ-10)
- [ ] AC-9: Multi-part dump test: 100K conntracks; `conntrack -L` returns all with NLM_F_MULTI + NLMSG_DONE. (covers REQ-11)
- [ ] AC-10: conntrackd test: 2 hosts running conntrackd; conntrack changes on host A propagate to host B via UDP-mcast or TCP synchronization. (covers REQ-1, REQ-3, REQ-4)
- [ ] AC-11: Hardening section present and follows template. (covers REQ-12)

## Architecture

### Rust module organization

- `kernel::net::netfilter::conntrack::netlink::Api` — top-level NFNL_SUBSYS_CTNETLINK handler
- `kernel::net::netfilter::conntrack::netlink::Dispatch` — per-IPCTNL_MSG_* dispatch
- `kernel::net::netfilter::conntrack::netlink::nla::CtaParser` — per-CTA_* NLA parser
- `kernel::net::netfilter::conntrack::netlink::nla::CtaBuilder` — reply NLA builder
- `kernel::net::netfilter::conntrack::netlink::Dump` — multi-part dump protocol
- `kernel::net::netfilter::conntrack::netlink::Notify` — multicast event broadcast
- `kernel::net::netfilter::conntrack::ext::Ecache` — event cache extension
- `kernel::net::netfilter::conntrack::ext::Acct` — accounting extension
- `kernel::net::netfilter::conntrack::ext::Seqadj` — TCP seqadj extension
- `kernel::net::netfilter::conntrack::ext::Timeout` — per-flow timeout policy
- `kernel::net::netfilter::conntrack::ext::Labels` — 128-bit labels
- `kernel::net::netfilter::conntrack::ext::Timestamp` — per-flow start/stop ns
- `kernel::net::netfilter::conntrack::netlink::Cttimeout` — cttimeout subsystem (per-policy registry)

### Locking and concurrency

- **Per-netns netlink mutex** (inherited from netlink core): per-socket message processing
- **Per-conntrack `lock`** (cross-ref `conntrack-core.md`): held during NLA-driven mutations
- **Per-CPU ecache work-queue**: deferred event emission; lockless per-CPU enqueue
- **Per-netns conntrack hashtable per-bucket lock**: held during walk for dump

### Error handling

- `Err(EINVAL)` — bad CTA_* / unsupported attribute combination
- `Err(EEXIST)` — IPCTNL_MSG_CT_NEW for duplicate
- `Err(ENOENT)` — non-existent
- `Err(EPERM)` — non-CAP_NET_ADMIN
- `Err(ENOMEM)` — alloc fail

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Per-IPCTNL_MSG_* dispatch | `kani::proofs::net::netfilter::conntrack::netlink::dispatch_safety` |
| CTA_* NLA parser per-attribute bounds | `kani::proofs::net::netfilter::conntrack::netlink::nla_parse_safety` |
| Multi-part dump cursor across DYING / UNCONFIRMED / regular hashtables | `kani::proofs::net::netfilter::conntrack::netlink::dump_safety` |
| Per-CPU ecache enqueue (lockless; no double-emit) | `kani::proofs::net::netfilter::conntrack::netlink::ecache_safety` |
| TCP seqadj arithmetic (no signed overflow) | `kani::proofs::net::netfilter::conntrack::netlink::seqadj_safety` |

### Layer 2: TLA+ models

(none new; cross-ref `models/net/conntrack_state_machine.tla` from `conntrack-proto.md`)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-CPU ecache work-queue | per-event ordering preserved (no out-of-order emission) | `kani::proofs::net::netfilter::conntrack::netlink::ecache_invariants` |
| Per-flow seqadj state | offset_orig + offset_reply applied consistently across reorderings | `kani::proofs::net::netfilter::conntrack::netlink::seqadj_invariants` |
| Per-flow labels | 128-bit bitmap consistent across read-modify-write under per-conn lock | `kani::proofs::net::netfilter::conntrack::netlink::labels_invariants` |

### Layer 4: Functional correctness (opt-in)

- **Event-cache reliability theorem** via Verus — proves: ∀ subscribed event class + ∀ matching conntrack mutation, exactly one event is delivered to each subscriber (no drops, no duplicates).

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **CONSTIFY** | per-message dispatch table + per-attribute NLA validation policies `static const` | § Mandatory |
| **MEMORY_SANITIZE** | freed-after-emit event-cache skbs cleared (carries flow 5-tuple + LSM secctx) | § Default-on configurable off |
| **SIZE_OVERFLOW** | NLA-length + seqadj offset arithmetic uses checked operators | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: per-skb (cross-ref `net/skbuff.md`); per-extension (cross-ref `conntrack-core.md`)
- **CONSTIFY, SIZE_OVERFLOW**: see above
- **USERCOPY**: NLA parsing uses bound-checked accessors (CVE-2014-class CTNETLINK-overflow defense)
- **KERNEXEC**: dispatch via `static const fn-ptr` array

### Row-2 / GR-RBAC integration

- LSM hook `security_capable(CAP_NET_ADMIN)` already required for IPCTNL_MSG_CT_NEW / DELETE / EXP_NEW / DELETE.
- Default useful GR-RBAC policy: deny IPCTNL_MSG_CT_FLUSH (bulk delete) + DELETE outside gradm-marked `firewall_admin` role; conntrack flush is a state-clearing primitive that may help malicious processes mask flow correlation.

### Userspace-visible behavior changes

None beyond upstream defaults (acct + timestamp default off per upstream 4.7+).

### Verification

(See § Verification above.)

## Open Questions

(none — ctnetlink + extensions exhaustively specified by upstream + libnetfilter_conntrack test corpus + conntrackd cluster-replication test coverage)

## Out of Scope

- Conntrack core (cross-ref `net/netfilter/conntrack-core.md`)
- Conntrack helpers (cross-ref `net/netfilter/conntrack-helper.md`)
- Per-protocol state machines (cross-ref `net/netfilter/conntrack-proto.md`)
- 32-bit-only paths
- Implementation code
