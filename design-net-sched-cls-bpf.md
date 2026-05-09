---
title: "Tier-3: net/sched/cls-bpf — BPF-driven classifier (Cilium dataplane foundation)"
tags: ["design-doc", "tier-3", "net"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for `cls_bpf` — the BPF-driven classifier that runs a user-supplied (cBPF or eBPF) program at classify-time. The program receives an `__sk_buff` context, returns a TC_ACT_* value, and optionally populates `tcf_result.classid`. The foundation of every modern container-networking dataplane: Cilium's BPF datapath uses cls_bpf at ingress and egress; tc-bpf programs implement custom packet rewriting, redirection, and policy enforcement that classical classifiers can't express.

Two BPF program types:
- **`BPF_PROG_TYPE_SCHED_CLS`** (eBPF): rich access to skb metadata, helpers, maps, kfuncs; can mutate skb (push/pop headers via bpf helpers); can be HW-offloaded via XDP/eBPF-in-NIC paths
- **cBPF** (legacy): classic Berkeley Packet Filter; read-only; less expressive but smaller verifier surface

`cls_bpf` integrates tightly with `kernel/bpf/00-overview.md` for verifier soundness + `kernel/bpf/00-overview.md` Layer 4 helpers/kfuncs.

Sub-tier-3 of `net/sched/00-overview.md`. Pairs with `net/sched/act-bpf.md` (BPF action — sibling pattern), `kernel/bpf/00-overview.md` (verifier + maps + helpers).

### Requirements

- REQ-1: All `TCA_BPF_*` NLAs (per the table) parsed identically.
- REQ-2: Two program-type modes (cBPF via `OPS_LEN`+`OPS` / eBPF via `FD`); identical dispatch.
- REQ-3: Direct-action mode (`TCA_BPF_FLAG_ACT_DIRECT`): program return = TC_ACT_* directly; without flag → return = classid + TC_ACT_OK.
- REQ-4: eBPF program type `BPF_PROG_TYPE_SCHED_CLS`; `__sk_buff` context with documented field accesses; verifier checks bounds.
- REQ-5: Helper / kfunc access set per the table; identical signatures to upstream.
- REQ-6: HW offload via `TCA_CLS_FLAGS_SKIP_SW` / `SKIP_HW` / `IN_HW`; driver-side `ndo_setup_tc(TC_SETUP_CLSBPF, ...)` bridge.
- REQ-7: Per-CPU per-filter hit counter; aggregated via `READ_ONCE` for dump.
- REQ-8: Action chain integration: per-filter `tcf_exts` invoked when classify returns TC_ACT_OK or non-zero classid (without direct-action).
- REQ-9: Per-filter dump: NLA-encoded program metadata (TAG, ID, NAME, FLAGS) byte-identical.
- REQ-10: Verifier integration: eBPF program load checked against `BPF_PROG_TYPE_SCHED_CLS` rules; reject programs with disallowed accesses.
- REQ-11: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: cBPF test: `tc filter add dev eth0 ingress bpf bytecode-file /path/to/prog.cBPF flowid 1:10` → matching packets classified to 1:10. (covers REQ-1, REQ-2)
- [ ] AC-2: eBPF test: load eBPF program with `bpftool prog load` returning FD; `tc filter add ... bpf da fd <FD>` → program runs on every packet. (covers REQ-2, REQ-4)
- [ ] AC-3: Direct-action test: eBPF program returns TC_ACT_SHOT → packet dropped (without classid lookup). (covers REQ-3)
- [ ] AC-4: Helper test: eBPF program calls `bpf_skb_store_bytes(skb, offset, value)` to overwrite a packet byte; verify packet emerges with mutated byte. (covers REQ-5)
- [ ] AC-5: Verifier-rejection test: eBPF program with out-of-bounds access to `__sk_buff` → verifier rejects at load. (covers REQ-10)
- [ ] AC-6: HW-offload test on supporting NIC (Netronome / Mellanox BlueField): `tc filter ... bpf ... skip_sw` → driver compiles eBPF to TCAM; `IN_HW` flag set. (covers REQ-6)
- [ ] AC-7: pcnt test: 1000 packets through eBPF filter → `tc -s filter show` reports pkts=1000. (covers REQ-7)
- [ ] AC-8: Action chain test: eBPF returns TC_ACT_OK + sets classid in result → mirred action in chain redirects to eth1. (covers REQ-8)
- [ ] AC-9: Per-filter dump test: TCA_BPF_TAG + TCA_BPF_NAME byte-identical in `tc filter show`. (covers REQ-9)
- [ ] AC-10: Cilium-equivalent test: eBPF program implementing 5-tuple → ENI-policy-decision via map lookup → action; full Cilium dataplane works. (covers REQ-2, REQ-4, REQ-5)
- [ ] AC-11: Hardening section present and follows template. (covers REQ-11)

### Architecture

### Rust module organization

- `kernel::net::sched::cls_bpf::Bpf` — classifier instance
- `kernel::net::sched::cls_bpf::CbpfMode` — legacy cBPF
- `kernel::net::sched::cls_bpf::EbpfMode` — modern eBPF
- `kernel::net::sched::cls_bpf::DirectAction` — `TCA_BPF_FLAG_ACT_DIRECT` semantics
- `kernel::net::sched::cls_bpf::Helpers` — per-prog-type helper allow-list (consults verifier)
- `kernel::net::sched::cls_bpf::HwOffload` — `ndo_setup_tc(TC_SETUP_CLSBPF)` bridge
- `kernel::net::sched::cls_bpf::Pcnt` — per-CPU hit counter
- `kernel::net::sched::cls_bpf::Dump` — NLA emit

### Locking and concurrency

- **Per-block `tcf_block_lock`** (mutex, inherited): held during per-filter mutation
- **RCU**: classify hot path RCU-side; mutator side under tcf_block_lock with RCU-defer
- **Per-CPU `pcnt`**: lockless per-CPU counters

### Error handling

- `Err(EEXIST)` — duplicate filter
- `Err(EINVAL)` — bad NLA / bad cBPF / bad eBPF FD
- `Err(EBADF)` — invalid FD
- `Err(EPERM)` — non-CAP_BPF (eBPF) or non-CAP_NET_ADMIN
- `Err(EOPNOTSUPP)` — TCA_CLS_FLAGS_SKIP_SW + driver doesn't support

### Out of Scope

- BPF verifier internals (cross-ref `kernel/bpf/00-overview.md`)
- BPF helpers / kfuncs implementation (cross-ref per-helper Tier-3 docs)
- Other classifiers (cross-ref `net/sched/cls-flower.md`, `cls-u32.md`)
- 32-bit-only paths
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| BPF classifier: per-filter program registration, classify hot path, NLA parser, dump | `net/sched/cls_bpf.c` |
| Public API | `include/net/pkt_cls.h` |
| UAPI: `TCA_BPF_*` NLAs | `include/uapi/linux/pkt_cls.h`, `include/uapi/linux/bpf.h` |

### compatibility contract

### NLA configuration

`TCA_OPTIONS` nested NLAs:
- `TCA_BPF_CLASSID` (target classid for parent qdisc; combined with prog return)
- `TCA_BPF_OPS_LEN` + `TCA_BPF_OPS` (legacy cBPF program — sock_filter array)
- `TCA_BPF_FD` (eBPF program FD obtained from `BPF_PROG_LOAD`)
- `TCA_BPF_NAME` (eBPF program name string for dumps)
- `TCA_BPF_FLAGS` (`TCA_BPF_FLAG_ACT_DIRECT` — eBPF return value is action directly, no classid lookup)
- `TCA_BPF_FLAGS_GEN` (`TCA_CLS_FLAGS_SKIP_HW / SKIP_SW / IN_HW / NOT_IN_HW`)
- `TCA_BPF_TAG` (4-byte BPF program tag for verification)
- `TCA_BPF_ID` (per-namespace BPF prog ID for FD-less reference)

Wire format byte-identical so iproute2's `tc filter add ... bpf` works unchanged.

### Two program-type modes

| Mode | NLA | Prog type | Mutable? |
|---|---|---|---|
| **cBPF (legacy)** | `TCA_BPF_OPS_LEN` + `TCA_BPF_OPS` | n/a (in-kernel JIT-or-interpret) | no |
| **eBPF (modern)** | `TCA_BPF_FD` (FD from BPF_PROG_LOAD) | `BPF_PROG_TYPE_SCHED_CLS` | yes |

eBPF is the modern path; cBPF retained for legacy filters from older iproute2 versions.

### Direct-action mode (`TCA_BPF_FLAG_ACT_DIRECT`)

When set, eBPF program's return value is interpreted directly as `TC_ACT_*` (no classid lookup). Used by Cilium-style dataplanes where the BPF program does all action logic in-program.

### Classify hot path

```rust
let res = run_program(prog, skb);  // BPF execution
if direct_action {
    return res; // TC_ACT_*
} else {
    classid = res;
    if classid != 0 {
        result.classid = classid;
        return TC_ACT_OK;
    } else {
        return TC_ACT_UNSPEC;
    }
}
```

Identical algorithm.

### eBPF program access

Per `BPF_PROG_TYPE_SCHED_CLS`: program receives `struct __sk_buff` context with read-write access to selected skb fields (data, data_end, len, pkt_type, mark, queue_mapping, protocol, vlan_present, vlan_tci, vlan_proto, priority, ingress_ifindex, ifindex, tc_index, cb[5], hash, tc_classid, tstamp, wire_len, gso_segs, gso_size, hwtstamp, family, remote_ip4, local_ip4, remote_ip6, local_ip6, remote_port, local_port). Per `kernel/bpf/00-overview.md` verifier checks all access bounds.

### Helper / kfunc access

`BPF_PROG_TYPE_SCHED_CLS` programs can call:
- `bpf_skb_store_bytes`, `bpf_skb_load_bytes` (mutate / read packet bytes)
- `bpf_skb_pull_data` (linearize)
- `bpf_skb_change_head` / `_change_tail` (push/pop headers)
- `bpf_skb_adjust_room` (insert/remove tunnel headers)
- `bpf_redirect`, `bpf_redirect_map` (action)
- `bpf_clone_redirect` (per-hop decision)
- `bpf_perf_event_output` (telemetry)
- `bpf_get_hash_recalc`, `bpf_set_hash_invalid`
- `bpf_skb_get/set_tunnel_key`, `_opt` (tunnel-aware)
- ct kfuncs: `bpf_ct_lookup_*`, `bpf_xdp_ct_alloc`, `bpf_ct_release` (cross-ref `net/netfilter/nf_conntrack_bpf.c`)
- `bpf_xdp_get_xfrm_state` etc. (cross-ref `net/xfrm/bpf.md`)

Identical helper set so existing programs work unchanged.

### HW offload

`TCA_CLS_FLAGS_SKIP_SW` requests hardware offload (NIC's eBPF processor / smart-NIC TCAM compilation); on driver failure → ENOTSUPP. `IN_HW` flag set when offloaded.

### Per-filter pcnt

Per-CPU hit counter; aggregated for `tc -s filter show`.

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| eBPF program execution wrapper (no use-after-free during async kfunc) | `kani::proofs::net::sched::cls_bpf::exec_safety` |
| cBPF JIT/interpret dispatch | `kani::proofs::net::sched::cls_bpf::cbpf_safety` |
| Direct-action vs. classid-lookup branch | `kani::proofs::net::sched::cls_bpf::action_safety` |
| HW-offload bridge | `kani::proofs::net::sched::cls_bpf::offload_safety` |

### Layer 2: TLA+ models

(none new owned at this granularity; relies on `models/net/cls_chain_eval.tla` from `cls-api.md` for chain-eval atomicity + verifier soundness from `kernel/bpf/00-overview.md`)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-filter program ref | when `prog != NULL`, refcount ≥ 1 (program won't be freed during classify) | `kani::proofs::net::sched::cls_bpf::prog_invariants` |
| Per-CPU pcnt | aggregated equals Σ per-CPU values | `kani::proofs::net::sched::cls_bpf::pcnt_invariants` |

### Layer 4: Functional correctness (depends on `kernel/bpf/00-overview.md` Layer 4)

- **Verifier-soundness theorem** (cross-ref `kernel/bpf/00-overview.md`) — proves: any program accepted by verifier doesn't violate kernel-memory access bounds. Owned by BPF-verifier Tier-3, consumed here.

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-filter program refcount uses `Refcount` (saturating) | § Mandatory |
| **CONSTIFY** | `tcf_proto_ops` for cls_bpf `static const` | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: per-filter (cross-ref `net/sched/cls-api.md`)
- **CONSTIFY**: see above
- **USERCOPY**: NLA parsing uses bound-checked accessors
- **SIZE_OVERFLOW**: per-CPU stats aggregation arithmetic uses checked operators
- **KERNEXEC**: BPF dispatch via `static const` JIT-or-interpret entry; eBPF runs in JIT-mapped W^X memory (verifier-checked)

### Row-2 / GR-RBAC integration

- LSM hook `security_capable(CAP_BPF)` already required for eBPF program load.
- LSM hook `security_capable(CAP_NET_ADMIN)` for cls_bpf filter installation.
- Default useful GR-RBAC policy: deny cls_bpf installation outside gradm-marked `bpf_users` role; eBPF in cls_bpf is a frequent capability-escape vector.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

