# Tier-2: net/sched — packet scheduler (qdiscs, classifiers, actions)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/sched/
  - include/net/sch_generic.h
  - include/net/pkt_cls.h
  - include/net/pkt_sched.h
  - include/net/act_api.h
  - include/uapi/linux/pkt_sched.h
  - include/uapi/linux/pkt_cls.h
-->

## Summary
Tier-2 overview for the Linux packet scheduler (`tc`) — the per-netdev TX queueing-discipline ("qdisc") tree, the classifier framework that decides per-packet routing within that tree, the action framework that mutates packets in the path, and the upstream rate-limiting + traffic-shaping + flow-isolation toolbox.

Three composable layers (per `man tc`):
1. **Qdiscs** (`sch_*`): per-netdev or per-class queue + scheduler. Examples: `pfifo_fast` (default), `fq` (fair queueing), `fq_codel` (default for systemd), `cake` (laptop/router default), `htb` (hierarchical token bucket), `mqprio` (multi-queue priority + DCB), `taprio` (time-aware shaper for IEEE 802.1Qbv TSN), `bpf_qdisc` (BPF-driven scheduling).
2. **Classifiers** (`cls_*`): packet-to-class mapping. Examples: `u32` (32-bit-key matcher), `flower` (5-tuple + meta), `bpf` (BPF-program-driven), `flow` (per-flow), `cgroup` (per-cgroup), `matchall` (all packets), `route` (route-key driven), `basic` (ematch-tree driven).
3. **Actions** (`act_*`): per-packet mutators invoked by classifiers. Examples: `mirred` (mirror/redirect to another iface), `gact` (drop/pass), `police` (rate limit), `pedit` (packet edit), `nat` (stateless NAT), `tunnel_key` (set tunnel metadata), `bpf` (BPF program), `skbedit` (skb metadata edit), `vlan` (VLAN tag mutate), `ct` (connection-track lookup → flowtable populate).

Used by: every modern Linux distro for queue management (FQ-CoDel + CAKE for desktop bufferbloat fix), Kubernetes CNI plugins (Cilium uses `cls_bpf` extensively), QoS tooling, container TC (Docker `--cpus`), high-frequency-trading deterministic latency setups (`taprio`), and SD-WAN / DPDK alternatives.

Sub-tier-2 of `net/00-overview.md`.

## Scope

This Tier-2 governs **all** of `/home/doll/linux-src/net/sched/` (~50 source files) plus the public API + UAPI headers. Per-file Tier-3 docs live under `.design/net/sched/`.

## Compatibility contract — outline

### `tc` UAPI: NETLINK_ROUTE qdisc/class/filter messages

Per `include/uapi/linux/pkt_sched.h` + `include/uapi/linux/pkt_cls.h`:
- `RTM_NEWQDISC` / `DELQDISC` / `GETQDISC`
- `RTM_NEWTCLASS` / `DELTCLASS` / `GETTCLASS`
- `RTM_NEWTFILTER` / `DELTFILTER` / `GETTFILTER`
- `RTM_NEWCHAIN` / `DELCHAIN` / `GETCHAIN`

Wire format byte-identical so iproute2's `tc` works unchanged.

### Per-qdisc configuration via NLA TCA_OPTIONS-nested attributes

Each qdisc has a per-type NLA schema (e.g., `TCA_FQ_CODEL_LIMIT` / `INTERVAL` / `TARGET` / `CE_THRESHOLD` for fq_codel). Wire format byte-identical.

### Per-classifier configuration via NLA TCA_OPTIONS-nested attributes

Same model for classifiers (e.g., `TCA_FLOWER_FLAGS` / `KEY_*` for flower).

### Per-action configuration via NLA TCA_OPTIONS-nested attributes

Same for actions.

### `/proc/net/psched`

Per-system clock-tick + jiffy-equivalent reference for tc rate calculations. Format-byte-identical so `tc` rate parameters compute identically.

### `qdisc_priv()` / `tcf_proto`/`tcf_chain`/`tcf_block` shared infrastructure

Shared across all qdiscs/classifiers; layout-equivalent.

### `struct Qdisc` layout

`include/net/sch_generic.h`:
- `enqueue`, `dequeue` ops (function pointers)
- `flags`, `limit`, `pad`, `parent`, `handle`
- `next_sched`, `gso_skb`, `q` (skb_queue), `bstats` (rate stats), `running`, `refcnt`, `qstats`
- `priv` (per-qdisc private data)
- `ops` (struct Qdisc_ops vtable: `init`/`reset`/`destroy`/`change`/`attach`/`enqueue`/`dequeue`/`peek`/`graft`/`leaf`/`find`/`walk`/`dump`/`dump_stats`)

Layout-equivalent for first cache-line.

### `struct tcf_proto` layout

Per-classifier-instance state. Fields: `next`, `root`, `classify`, `bind_class`, `protocol`, `prio`, `chain`, `data`, `ops`, `block`, `qsd_classify`, `lock`, `refcnt`, `rcu`, `deleting`. Layout-equivalent.

### `struct tc_action` layout

Per-action-instance state. Fields: `tcfa_pad`, `tcfa_id` (TCA_ID_*), `tcfa_lock` (spinlock), `tcfa_index` (u32), `tcfa_refcnt`, `tcfa_bindcnt`, `tcfa_action`, `tcfa_tm`, `tcfa_bstats`/`tcfa_qstats`, `goto_chain`, `hw_stats`, `used_hw_stats`. Layout-equivalent.

## Tier-3 docs governed by this Tier-2

(Phase C will add these incrementally.)

| Tier-3 doc | Scope |
|---|---|
| `net/sched/sch-api.md` | qdisc API: register/lookup, NETLINK_ROUTE wire (`sch_api.c`) |
| `net/sched/sch-generic.md` | per-qdisc generic helpers + default qdiscs (`sch_generic.c`) |
| `net/sched/cls-api.md` | classifier API + chain/block infrastructure (`cls_api.c`) |
| `net/sched/act-api.md` | action API (`act_api.c`) |
| `net/sched/sch-fq.md` | FQ qdisc (`sch_fq.c`) |
| `net/sched/sch-fq-codel.md` | FQ-CoDel qdisc (`sch_fq_codel.c`) |
| `net/sched/sch-cake.md` | CAKE qdisc (`sch_cake.c`) |
| `net/sched/sch-htb.md` | HTB qdisc (`sch_htb.c`) |
| `net/sched/sch-mqprio.md` | mqprio + DCB (`sch_mqprio.c`) |
| `net/sched/sch-taprio.md` | TAPRIO TSN scheduler (`sch_taprio.c`) |
| `net/sched/sch-bpf.md` | BPF qdisc (`bpf_qdisc.c`) |
| `net/sched/cls-u32.md` | u32 classifier (`cls_u32.c`) |
| `net/sched/cls-flower.md` | flower classifier (`cls_flower.c`) |
| `net/sched/cls-bpf.md` | BPF classifier (`cls_bpf.c`) |
| `net/sched/act-mirred.md` | mirror/redirect action (`act_mirred.c`) |
| `net/sched/act-police.md` | rate-policer action (`act_police.c`) |
| `net/sched/act-ct.md` | connection-track action (`act_ct.c`) |
| `net/sched/em-meta.md` | extended-match meta predicates (`em_meta.c`) |

## Compatibility outline (top-level)

- REQ-O1: NETLINK_ROUTE wire format for RTM_*QDISC/TCLASS/TFILTER/CHAIN byte-identical.
- REQ-O2: `struct Qdisc`, `struct tcf_proto`, `struct tc_action` first-cache-line layout-equivalent.
- REQ-O3: Per-qdisc + per-classifier + per-action NLA TCA_OPTIONS schemas byte-identical.
- REQ-O4: Per-Tier-3 child documents define per-component requirements — total tc ABI fidelity covered cumulatively.
- REQ-O5: `/proc/net/psched` content byte-identical.
- REQ-O6: Per-netdev TX qdisc tree dispatch identical: enqueue → dequeue → driver xmit.
- REQ-O7: Per-classifier `tcf_classify` walk: per-priority order; first-match-wins; fall-through to next chain on TC_ACT_UNSPEC.
- REQ-O8: Per-action invocation: classifier-driven; per-action `tcfa_refcnt` shared across classifiers (action sharing).
- REQ-O9: Hardware-offload (TC offload): per-qdisc + per-classifier + per-action offload to NIC (e.g., Mellanox ASAP², Intel ice/i40e). Identical contract.
- REQ-O10: TLA+ models declared at this Tier-2 (see § Verification — Layer 2 below).
- REQ-O11: Hardening: row-1 features applied; default-on configurable per `00-security-principles.md`.

## Acceptance Criteria (top-level)

- [ ] AC-O1: `tc qdisc show` + `tc filter show` + `tc actions list action <id>` byte-identical content after equivalent setup. (covers REQ-O1)
- [ ] AC-O2: `pahole struct Qdisc` + `struct tcf_proto` + `struct tc_action` byte-identical first cache-line. (covers REQ-O2)
- [ ] AC-O3: Per-Tier-3 ACs cumulatively cover the tc ABI surface. (meta-AC)
- [ ] AC-O4: `cat /proc/net/psched` byte-identical content. (covers REQ-O5)
- [ ] AC-O5: TX path test: install fq_codel root qdisc + flower classifier + mirred action; tcpdump on output shows correctly-classified packets redirected. (covers REQ-O6, REQ-O7, REQ-O8)
- [ ] AC-O6: HW-offload test on supporting NIC: install flower classifier with tc offload flag; `tc filter show` shows in-hw counter; throughput exceeds CPU-only path. (covers REQ-O9)
- [ ] AC-O7: TLA+ models per § Verification all pass `make tla`. (covers REQ-O10)
- [ ] AC-O8: Hardening section per Tier-3 child docs is non-empty. (covers REQ-O11)

## Architecture (top-level)

Each Tier-3 child doc declares its own Rust module organization. The shared abstractions:

- `kernel::net::sched::qdisc::Qdisc` — `struct Qdisc` core
- `kernel::net::sched::qdisc::ops::QdiscOps` — `struct Qdisc_ops` vtable
- `kernel::net::sched::cls::TcfProto` — `struct tcf_proto`
- `kernel::net::sched::cls::Block` — `tcf_block` per-shared-qdisc-or-block
- `kernel::net::sched::cls::Chain` — `tcf_chain` per-priority chain
- `kernel::net::sched::act::TcAction` — `struct tc_action`
- `kernel::net::sched::act::ops::TcActionOps` — `struct tc_action_ops` vtable
- `kernel::net::sched::netlink::TcNetlink` — RTM_*QDISC/TCLASS/TFILTER/CHAIN handlers
- `kernel::net::sched::offload::HwOffload` — per-qdisc/cls/act HW offload bridge

## Verification (top-level)

### Layer 1: Kani SAFETY proofs

Each Tier-3 child doc declares its own.

### Layer 2: TLA+ models — mandatory list

| Model | Owned by |
|---|---|
| `models/net/sch_fq_codel.tla` | `net/sched/sch-fq-codel.md` (FQ-CoDel: per-flow queue isolation + CoDel target/interval invariants — provably no flow can starve another, dequeue order satisfies fair-queueing definition) |
| `models/net/sch_htb.tla` | `net/sched/sch-htb.md` (HTB: per-class rate + ceil + burst + cburst invariants — token-bucket non-negative, hierarchy rate-sum bounded) |
| `models/net/cls_chain_eval.tla` | `net/sched/cls-api.md` (classifier-chain evaluation: priority order + TC_ACT_UNSPEC fall-through; concurrent chain mutation atomicity) |

### Layer 3: invariant harnesses

Per Tier-3.

### Layer 4: functional correctness (opt-in)

- **FQ-CoDel fair-queueing theorem** via Verus — proves: ∀ pair of flows (A, B), bandwidth share converges to 1:1 ratio under continuous traffic.
- **HTB rate-bound theorem** via Verus — proves: per-class actual bandwidth ≤ class.rate + ceil-driven borrow within class.cburst window.
- **TAPRIO time-window correctness** via TLA+ refinement — proves: each per-traffic-class send window matches the configured admin/oper schedule per IEEE 802.1Qbv.

## Hardening (top-level)

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this Tier-2

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | Qdisc, tcf_proto, tcf_chain, tcf_block, tc_action refcounts use `Refcount` (saturating) | § Mandatory |
| **AUTOSLAB** | per-type slab caches | § Mandatory |
| **MEMORY_SANITIZE** | freed Qdisc/cls/act state cleared (qdiscs may queue in-flight skbs with sensitive payload) | § Default-on configurable off |
| **SIZE_OVERFLOW** | rate-table arithmetic + token-bucket arithmetic uses checked operators (CVE class: integer-overflow rate-limit-bypass) | § Mandatory |

### Row-1 features consumed by this Tier-2

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE, SIZE_OVERFLOW**: see above
- **CONSTIFY**: per-qdisc/cls/act ops vtables `static const`
- **USERCOPY**: NLA parsing uses bound-checked accessors
- **KERNEXEC**: per-qdisc/cls/act dispatch via `static const fn-ptr` arrays

### Row-2 / GR-RBAC integration

- LSM hook `security_capable(CAP_NET_ADMIN)` already required for `tc qdisc`/`filter`/`actions` mutations.
- LSM hook on BPF-driven qdiscs/classifiers/actions: `security_capable(CAP_BPF)` for `bpf_qdisc` / `cls_bpf` / `act_bpf`.
- Default useful GR-RBAC policy: deny `tc` mutations outside gradm-marked `network_admin` role.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Open Questions

(none at Tier-2 — defer to per-Tier-3)

## Out of Scope

- Per-driver TC-offload implementations (cross-ref individual driver Tier-4 docs once Phase D begins)
- Per-IPSec qdisc considerations (cross-ref `net/xfrm/00-overview.md`)
- Implementation code
