---
title: "Tier-3: net/sched/act-ct — connection-tracking action (netfilter conntrack from tc)"
tags: ["design-doc", "tier-3", "net"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for `act_ct` — the tc action that performs connection tracking lookup + state-setup directly in the tc datapath, populating the per-skb `nf_conn` (`skb->_nfct`) tag without traversing the full netfilter framework. The cornerstone of every modern stateful container-networking dataplane: lets a tc-bpf or `cls_flower` filter ask "is this an established connection? new? related? invalid?" and route accordingly without per-packet iptables traversal cost.

Combined with `cls_flower` `KEY_CT_*` matching (cross-ref `cls-flower.md` REQ-6) and the netfilter flowtable offload (cross-ref `net/netfilter/nf_flow_table.md` Tier-3 once netfilter is covered), `act_ct` enables wire-rate stateful firewalling on commodity NICs.

Two operating modes:
- **Conntrack lookup**: read-only — looks up existing nf_conn for skb's 5-tuple; tags skb with state
- **Conntrack commit**: also installs the conntrack on first packet of a flow; with `+nat` flag, applies SNAT/DNAT mapping
- **Conntrack clear**: removes conntrack tag from skb (e.g., after policy decision)

Sub-tier-3 of `net/sched/00-overview.md`. Pairs with `net/sched/cls-flower.md` (consumer of CT_STATE / CT_ZONE / CT_MARK / CT_LABELS keys), `net/netfilter/00-overview.md` Tier-2 (once added — netfilter conntrack underlying mechanism).

### Requirements

- REQ-1: `TCA_CT_ACTION` flag set (`COMMIT / CLEAR / NAT / NAT_SRC / NAT_DST / FORCE`); identical bit values + combination semantics.
- REQ-2: Per-skb `skb->_nfct` tagging with `IP_CT_*` state; identical to netfilter conntrack contract.
- REQ-3: Conntrack zones via `TCA_CT_ZONE` (16-bit); per-zone separate state tables.
- REQ-4: Conntrack mark + label setting via `TCA_CT_MARK` / `_MASK` and `TCA_CT_LABELS` / `_MASK` (128-bit); read-modify-write semantics with masks.
- REQ-5: NAT integration: `TCA_CT_ACT_NAT` + IP/port range parameters; per-RFC-3022 SNAT/DNAT semantics.
- REQ-6: Conntrack helper attachment via `TCA_CT_HELPER_NAME` + `_FAMILY` + `_PROTO`; helper provides per-flow related-flow registration.
- REQ-7: Flowtable offload: when established flow seen + flowtable enabled, insert via `nf_flow_offload_add`; subsequent packets fast-path through flowtable.
- REQ-8: Per-action stats: per-CPU bstats/qstats; per-action `installed_count` (number of conntrack entries installed by this action).
- REQ-9: HW offload via `flow_action_ct` (when supported by NIC — Mellanox ConnectX-6 DX+ supports CT-offload).
- REQ-10: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: Lookup-only test: `tc filter ... action ct nocommit pipe action gact pass`; first packet → `skb->_nfct` is NULL (not tracked); subsequent packet to existing conntracked flow → tagged `IP_CT_ESTABLISHED`. (covers REQ-1, REQ-2)
- [ ] AC-2: Commit test: `tc filter ... action ct commit`; first packet installs conntrack; verifiable via `conntrack -L` showing the flow. (covers REQ-1, REQ-2)
- [ ] AC-3: Clear test: skb already tagged → `action ct clear` → `skb->_nfct` becomes NULL. (covers REQ-1)
- [ ] AC-4: Zone test: `action ct zone 1 commit` + `action ct zone 2 commit`; same 5-tuple in zones 1 + 2 are distinct conntrack entries. (covers REQ-3)
- [ ] AC-5: Mark test: `action ct commit mark 0x100/0xff00`; conntrack entry has mark 0x100 (with mask applied to existing). (covers REQ-4)
- [ ] AC-6: NAT test: `action ct commit nat src addr 10.0.0.1-10.0.0.10 port 1024-65535`; outgoing connection rewritten to chosen IP/port; reverse path correctly de-NATs. (covers REQ-5)
- [ ] AC-7: Helper test: `action ct helper name ftp family inet proto tcp commit`; FTP control connection → DCC data connection auto-conntracked as `RELATED`. (covers REQ-6)
- [ ] AC-8: Flowtable offload test: enable nf_flow_table; established flow + `action ct` → flow inserted into flowtable; subsequent packets bypass tc filter chain (verifiable via `tc -s qdisc` showing decreasing per-filter counters once offloaded). (covers REQ-7)
- [ ] AC-9: HW-offload test on supporting NIC: full CT offload to Mellanox ConnectX-6 DX; conntrack entries appear in HW table; line-rate stateful firewall verified. (covers REQ-9)
- [ ] AC-10: Cilium-equivalent test: BPF + cls_flower + act_ct + act_mirred chain implementing pod-to-pod stateful firewall; full Cilium dataplane works at line rate. (covers REQ-1, REQ-2, REQ-3, REQ-4, REQ-7)
- [ ] AC-11: Hardening section present and follows template. (covers REQ-10)

### Architecture

### Rust module organization

- `kernel::net::sched::act_ct::Ct` — action instance
- `kernel::net::sched::act_ct::ActionFlags` — `TCA_CT_ACTION` enum
- `kernel::net::sched::act_ct::Lookup` — read-only conntrack lookup
- `kernel::net::sched::act_ct::Commit` — conntrack-install on first-packet
- `kernel::net::sched::act_ct::Clear` — `skb->_nfct` clear
- `kernel::net::sched::act_ct::Zone` — per-zone state isolation
- `kernel::net::sched::act_ct::MarkLabels` — read-modify-write mark + labels
- `kernel::net::sched::act_ct::Nat` — `TCA_CT_ACT_NAT` integration
- `kernel::net::sched::act_ct::Helper` — conntrack-helper attach
- `kernel::net::sched::act_ct::FlowtableOffload` — `nf_flow_offload_add` bridge
- `kernel::net::sched::act_ct::HwOffload` — `flow_action_ct` driver bridge

### Locking and concurrency

- **Per-action `tcfa_lock`** (spinlock, inherited from act-api): held during config mutation
- **Per-zone netfilter conntrack table locks** (cross-ref `net/netfilter/00-overview.md` once added): hot-path RCU-side
- **No new global locks** at this Tier-3

### Error handling

- `Err(EINVAL)` — bad NLA / inconsistent (e.g., NAT without commit)
- `Err(ENOENT)` — referenced helper not loaded
- `Err(EOPNOTSUPP)` — HW offload requested but not supported
- `Err(ENOMEM)` — alloc fail

### Out of Scope

- Netfilter conntrack underlying mechanism (cross-ref `net/netfilter/00-overview.md` once Tier-2 is added)
- Flowtable internal storage (cross-ref `net/netfilter/nf_flow_table.md` once Tier-3 is added)
- 32-bit-only paths
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Conntrack action: lookup, commit, clear, NAT integration, flowtable offload, NLA parser, dump | `net/sched/act_ct.c` |
| Public API | `include/net/tc_act/tc_ct.h`, `include/net/netfilter/nf_conntrack.h` |
| UAPI: `TCA_CT_*` NLAs + `TCA_CT_FORCE_COMMIT` etc. | `include/uapi/linux/tc_act/tc_ct.h` |

### compatibility contract

### NLA configuration

`TCA_OPTIONS` nested NLAs:
- `TCA_CT_PARMS` (struct tc_ct + extras)
- `TCA_CT_ACTION` (`TCA_CT_ACT_COMMIT | TCA_CT_ACT_CLEAR | TCA_CT_ACT_NAT | TCA_CT_ACT_NAT_SRC | TCA_CT_ACT_NAT_DST | TCA_CT_ACT_FORCE`)
- `TCA_CT_ZONE` (16-bit conntrack zone — for multi-tenant isolation)
- `TCA_CT_MARK` + `TCA_CT_MARK_MASK` (set conntrack mark with mask)
- `TCA_CT_LABELS` + `TCA_CT_LABELS_MASK` (128-bit conntrack labels with mask)
- `TCA_CT_NAT_IPV4_MIN` / `TCA_CT_NAT_IPV4_MAX` (NAT IP range — IPv4)
- `TCA_CT_NAT_IPV6_MIN` / `TCA_CT_NAT_IPV6_MAX` (NAT IP range — IPv6)
- `TCA_CT_NAT_PORT_MIN` / `TCA_CT_NAT_PORT_MAX` (NAT port range)
- `TCA_CT_HELPER_NAME` / `TCA_CT_HELPER_FAMILY` / `TCA_CT_HELPER_PROTO` (conntrack-helper module attach: ftp / sip / pptp / etc.)
- `TCA_CT_TM` (read-only timestamp)
- `TCA_CT_PAD`

Wire format byte-identical so iproute2's `tc filter ... action ct` works unchanged.

### `TCA_CT_ACTION` flag bits

| Flag | Constant | Semantic |
|---|---|---|
| `COMMIT` | 1<<0 | Install conntrack entry on first packet (vs. read-only lookup-and-tag) |
| `CLEAR` | 1<<1 | Remove existing conntrack tag from skb |
| `NAT` | 1<<2 | Apply NAT mapping per `TCA_CT_NAT_*` parameters |
| `NAT_SRC` | 1<<3 | NAT mode source-only |
| `NAT_DST` | 1<<4 | NAT mode destination-only |
| `FORCE` | 1<<5 | Override existing conntrack tag if present |

Per-action flag combinations: `lookup-only` (default; no flags), `commit`, `commit + nat`, `clear`. Identical bit semantics.

### Per-skb conntrack tagging

`skb->_nfct` (cross-ref netfilter conntrack) holds a (nf_conn pointer, ctinfo bits) tuple. After `act_ct` lookup/commit:
- `IP_CT_NEW` — first packet of flow
- `IP_CT_ESTABLISHED` — return-direction or established flow
- `IP_CT_RELATED` — related to existing flow (e.g., ICMP for an existing TCP)
- `IP_CT_NEW_REPLY` / `_RELATED_REPLY` — return-direction variants
- `IP_CT_INVALID` — invalid (couldn't track; e.g., out-of-window TCP segment)

After tagging, downstream classifiers can match on `KEY_CT_STATE` (cross-ref `cls-flower.md`).

Identical per-skb tagging.

### Flowtable offload integration

When kernel sees an established TCP/UDP flow with conntrack, `act_ct` may insert into `nf_flow_table` (cross-ref future `net/netfilter/nf_flow_table.md`). Once in flowtable, subsequent packets bypass conntrack lookup + tc filter chain entirely, taking a fast-path through `nf_flow_offload_xmit` (or HW-offload to NIC). Per-flow eviction on FIN/RST or timeout.

Identical contract.

### Conntrack zones

Per-tenant isolation: each conntrack lookup is scoped to a zone (default 0). Two flows with same 5-tuple but different zones are distinct conntrack entries. Used heavily in container networking for per-pod/per-tenant isolation.

### NAT integration (`TCA_CT_ACT_NAT`)

When `+nat` is set + commit: kernel installs NAT mapping from skb's original 5-tuple to a randomly-chosen IP/port from `[TCA_CT_NAT_IPV4_MIN, _MAX]` × `[TCA_CT_NAT_PORT_MIN, _MAX]`. NAT-source mode rewrites src; NAT-destination mode rewrites dst; both modes can be combined.

Identical per-RFC-3022 NAT semantics.

### Helper modules (`TCA_CT_HELPER_*`)

Conntrack helpers parse application-protocol semantics (FTP DCC, SIP, RTSP, PPTP, etc.) to identify "related" flows and pre-register them in conntrack. `TCA_CT_HELPER_NAME` selects from registered helpers; `_FAMILY` + `_PROTO` constrain.

Identical UAPI.

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Conntrack lookup (RCU-side; no use-after-free of nf_conn) | `kani::proofs::net::sched::act_ct::lookup_safety` |
| Commit path (refcount sound on freshly-allocated nf_conn) | `kani::proofs::net::sched::act_ct::commit_safety` |
| NAT mapping arithmetic (no overflow on IP / port range) | `kani::proofs::net::sched::act_ct::nat_safety` |
| Mark + labels read-modify-write (atomicity under conntrack lock) | `kani::proofs::net::sched::act_ct::mark_safety` |
| Flowtable offload-add (refcount sound) | `kani::proofs::net::sched::act_ct::flowtable_safety` |

### Layer 2: TLA+ models

(none new owned at this granularity; relies on netfilter conntrack TLA+ models from `net/netfilter/00-overview.md` once added)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-action flags | mutually exclusive flag combinations enforced (CLEAR ⇏ COMMIT, NAT ⇒ COMMIT) | `kani::proofs::net::sched::act_ct::flags_invariants` |
| Per-skb _nfct tagging | non-NULL ⇒ refcount ≥ 1 on the tagged nf_conn | `kani::proofs::net::sched::act_ct::tag_invariants` |

### Layer 4: Functional correctness (opt-in)

(deferred — relies on netfilter conntrack functional correctness theorem from `net/netfilter/00-overview.md`)

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-nf_conn ref acquired by lookup uses `Refcount` (saturating) | § Mandatory |
| **CONSTIFY** | `tc_action_ops` for ct `static const` | § Mandatory |
| **MEMORY_SANITIZE** | freed nf_conn (via netfilter side) cleared; per-action labels (which can carry sensitive identifiers) cleared on free | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, CONSTIFY, MEMORY_SANITIZE**: see above
- **USERCOPY**: NLA parsing uses bound-checked accessors
- **SIZE_OVERFLOW**: NAT range arithmetic uses checked operators
- **KERNEXEC**: action dispatch via `static const fn-ptr`

### Row-2 / GR-RBAC integration

- LSM hook: same as `act-api.md` (CAP_NET_ADMIN gate); plus per-zone access-control via netfilter LSM hooks once netfilter Tier-2 is covered.
- Useful default GR-RBAC policy: deny `act_ct` outside gradm-marked `network_admin` role; act_ct gives stateful-firewall manipulation primitive that can mask malicious flows behind ESTABLISHED state.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

