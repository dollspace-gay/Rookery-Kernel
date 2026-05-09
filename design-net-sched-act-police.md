---
title: "Tier-3: net/sched/act-police — rate-policer action (single-rate / two-rate token bucket)"
tags: ["design-doc", "tier-3", "net"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for `act_police` — a rate-policer action that drops or marks packets exceeding configured rate limits. Implements both Single Rate Three Color Marker (RFC 2697 srTCM) and Two Rate Three Color Marker (RFC 2698 trTCM) algorithms via per-action token-bucket state, with configurable conform/exceed/pir-exceed actions. The cornerstone of every ingress rate-limit setup, DDoS-mitigation primitive, and per-customer fair-share enforcement on telecom equipment.

Used by:
- `tc filter add dev eth0 ingress flower ... action police rate 100mbit burst 10kb conform-exceed drop`
- DDoS auto-mitigation scripts (`ip rule add fwmark 0x100 lookup 100; tc filter ... action police ...`)
- Per-customer-VLAN bandwidth shaping at ISP CPE
- BPF-classified flows that consume tokens from a shared police action via action sharing (cross-ref `act-api.md` REQ-4)

Sub-tier-3 of `net/sched/00-overview.md`. Pairs with `net/sched/act-api.md` (action framework), `net/sched/cls-flower.md` / `cls-bpf.md` (typical classifiers that install this action).

### Requirements

- REQ-1: `struct tc_police` byte-identical UAPI layout.
- REQ-2: All `TCA_POLICE_*` NLAs (per the table) parsed identically; 64-bit rates supported via `RATE64` / `PEAKRATE64` / `PKTRATE64`.
- REQ-3: Single-rate (RFC 2697 srTCM) mode: one CIR bucket; conform/exceed semantics identical.
- REQ-4: Two-rate (RFC 2698 trTCM) mode: CIR + PIR buckets; conform/partial-exceed/full-exceed semantics identical.
- REQ-5: Token-bucket replenishment arithmetic: `toks += (now - t_c) × rate`, capped at `burst`; identical formula.
- REQ-6: `result` action: 5 variants (`OK / DROP / RECLASSIFY / PIPE / CONTINUE`); identical dispatch.
- REQ-7: PPS-based mode via `TCA_POLICE_PKTRATE64` + `TCA_POLICE_PKTBURST64`; per-packet "size" = 1.
- REQ-8: HW offload via `flow_action_police`; identical contract.
- REQ-9: Per-action stats: per-CPU bstats/qstats; per-color drop counter.
- REQ-10: Action sharing: police action with `tcfa_index = N` shareable across multiple classifiers; tokens are shared (single bucket regardless of caller-count).
- REQ-11: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: `pahole struct tc_police` byte-identical layout. (covers REQ-1)
- [ ] AC-2: Single-rate test: `tc filter ... action police rate 1mbit burst 10kb conform-exceed pass/drop` → traffic at 500kbit fully conforms; at 2mbit half drops. (covers REQ-3, REQ-5)
- [ ] AC-3: Two-rate test: `tc filter ... action police rate 1mbit peakrate 2mbit burst 10kb mtu 1500 conform-exceed pass/pipe/drop` → traffic at 500kbit conforms; at 1.5mbit pipes (partial-exceed); at 3mbit drops (full-exceed). (covers REQ-4)
- [ ] AC-4: 64-bit rate test: `tc filter ... action police rate 50gbit ...` (over 32-bit limit ~34 Gbit) → uses `TCA_POLICE_RATE64`; rate honored on 100 Gbps NIC. (covers REQ-2)
- [ ] AC-5: PPS test: `tc filter ... action police pktrate 1000pps pktburst 100` → drops above 1000 packets/sec regardless of packet size. (covers REQ-7)
- [ ] AC-6: Result-RECLASSIFY test: police with `conform-exceed pass/reclassify` → exceed packets re-enter classifier from chain 0. (covers REQ-6)
- [ ] AC-7: Result-PIPE test: police is mid-chain (between mirred + drop); on conform, PIPE to next; on exceed, drop early without reaching final action. (covers REQ-6)
- [ ] AC-8: Action-sharing test: 2 classifiers reference same police action by index; combined traffic tokens consumed from single bucket. (covers REQ-10)
- [ ] AC-9: HW-offload test on supporting NIC (Mellanox ConnectX-6+): police filter offloads via `flow_action_police`; per-action `in_hw_count` increments. (covers REQ-8)
- [ ] AC-10: Per-color drops stats test: 1000 conform + 500 exceed → `tcfa_bstats` shows 1500 packets; per-color drop counter shows 500. (covers REQ-9)
- [ ] AC-11: Hardening section present and follows template. (covers REQ-11)

### Architecture

### Rust module organization

- `kernel::net::sched::act_police::Police` — action instance
- `kernel::net::sched::act_police::SingleRate` — RFC 2697 srTCM
- `kernel::net::sched::act_police::TwoRate` — RFC 2698 trTCM
- `kernel::net::sched::act_police::TokenBucket` — replenish + consume arithmetic
- `kernel::net::sched::act_police::Result` — per-color action dispatch
- `kernel::net::sched::act_police::PpsMode` — packets/sec mode
- `kernel::net::sched::act_police::HwOffload` — `flow_action_police` bridge
- `kernel::net::sched::act_police::Stats` — per-CPU bstats/qstats + per-color drops

### Locking and concurrency

- **Per-action `tcfa_lock`** (spinlock, inherited from act-api): held during token-bucket replenish + consume (atomic update of `tcfp_tok` / `tcfp_ptok` / `tcfp_t_c`)
- **No new locks**

### Error handling

- `Err(EINVAL)` — bad NLA / inconsistent (e.g., peakrate < rate)
- `Err(ENOMEM)` — alloc fail
- Per-color drops counted via per-CPU stats, not returned to caller (action-internal)

### Out of Scope

- Connection tracking action (cross-ref `net/sched/act-ct.md`)
- Other actions (cross-ref `net/sched/act-mirred.md`, `act-gact.md`, etc.)
- 32-bit-only paths
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Rate-policer action: srTCM / trTCM token-bucket, rate-table arithmetic, NLA parser, dump | `net/sched/act_police.c` |
| Public API | `include/net/tc_act/tc_police.h`, `include/net/act_api.h` |
| UAPI: `TCA_POLICE_*` NLAs + `struct tc_police` (lives in pkt_cls.h, not its own header) | `include/uapi/linux/pkt_cls.h` |

### compatibility contract

### `struct tc_police` (UAPI)

```c
struct tc_police {
    __u32 index;              /* action index */
    int   action;              /* default action (TC_ACT_*) for "exceed" path */
    __u32 limit;               /* bucket limit (legacy field; superseded by rates) */
    __u32 burst;               /* CIR-bucket size */
    __u32 mtu;                 /* per-packet max applied to bucket */
    struct tc_ratespec rate;   /* CIR rate spec */
    struct tc_ratespec peakrate; /* PIR rate spec (0 means single-rate) */
    int   refcnt;
    int   bindcnt;
    __u32 capab;
};
```

Layout-byte-identical so iproute2's `tc filter ... action police rate <X> burst <Y> ...` works unchanged.

### Two operating modes

| Mode | Trigger | Algorithm |
|---|---|---|
| **Single-rate** (RFC 2697) | `peakrate.rate == 0` | One CIR token bucket; conform if tokens ≥ pkt_size; exceed otherwise |
| **Two-rate** (RFC 2698) | `peakrate.rate > 0` | Two buckets (CIR + PIR); conform if both tokens ≥ pkt_size; partial-exceed if PIR available; full-exceed otherwise |

Identical algorithm.

### NLA configuration

`TCA_OPTIONS` nested NLAs:
- `TCA_POLICE_TBF` (struct tc_police — backward-compat carrier)
- `TCA_POLICE_RATE` (CIR rate table — for fast time→bytes lookup)
- `TCA_POLICE_PEAKRATE` (PIR rate table)
- `TCA_POLICE_AVRATE` (legacy: estimated rate-from-traffic; for "police by observed rate" mode)
- `TCA_POLICE_RESULT` (per-result action: `TCA_POLICE_RESULT_OK` / `_DROP` / `_RECLASSIFY` / `_PIPE` / `_CONTINUE`)
- `TCA_POLICE_TM` (read-only timestamp/usage)
- `TCA_POLICE_PAD`
- `TCA_POLICE_RATE64` (64-bit CIR for rates > 32-bit)
- `TCA_POLICE_PEAKRATE64`
- `TCA_POLICE_PKTRATE64` (PPS-based limit; alternative to byte-rate)
- `TCA_POLICE_PKTBURST64`

Wire format byte-identical so iproute2 + tc-test-suite work unchanged.

### Token-bucket state

Per-action state:
- `tcfp_tok` (s64): current CIR tokens (in nanoseconds-of-rate)
- `tcfp_ptok` (s64): current PIR tokens (only for two-rate)
- `tcfp_t_c` (s64): timestamp of last replenish
- `tcfp_burst` / `tcfp_mtu_ptoks` / `tcfp_mtu_max` for caps

Replenish per-act invocation:
```
toks = min(tcfp_tok + (now - t_c) × rate,   burst)
ptoks = min(tcfp_ptok + (now - t_c) × peakrate, mtu_ptoks)
```

If `toks - pkt_size ≥ 0` → conform; consume from `toks`. Otherwise → exceed; in two-rate, check `ptoks - pkt_size ≥ 0` → partial-exceed; consume from `ptoks` only. Otherwise → full-exceed.

Identical state machine.

### `result` field — per-color action

`TCA_POLICE_RESULT` enum:
- `TCA_POLICE_RESULT_OK` (legacy alias for conform)
- `TCA_POLICE_RESULT_DROP` (default exceed action)
- `TCA_POLICE_RESULT_RECLASSIFY` (jump to reclassify)
- `TCA_POLICE_RESULT_PIPE` (continue chain — used when policer is part of larger action sequence)
- `TCA_POLICE_RESULT_CONTINUE` (skip to next action regardless of color)

Identical action set.

### PPS-based mode (`TCA_POLICE_PKTRATE64` + `TCA_POLICE_PKTBURST64`)

When set: police by packets/sec rather than bytes/sec; per-packet "size" treated as 1 regardless of actual byte length. Used for SYN-flood limits etc.

### HW offload via `flow_action_police`

When `TCA_CLS_FLAGS_SKIP_SW`: per-action police installs as part of per-filter flow-action descriptor; HW programs internal token-bucket. On miss → software fallback.

Identical contract.

### Per-action stats

Per-CPU `cpu_bstats` / `cpu_qstats` (cross-ref `act-api.md` REQ-11): packets/bytes processed; per-color `drops` count.

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Token-bucket replenishment arithmetic (no overflow on (now - t_c) × rate; signed overflow guarded) | `kani::proofs::net::sched::act_police::tokens_safety` |
| Two-rate dual-bucket atomicity (toks + ptoks updated together under tcfa_lock) | `kani::proofs::net::sched::act_police::two_rate_safety` |
| Per-color result dispatch | `kani::proofs::net::sched::act_police::result_safety` |
| HW-offload bridge | `kani::proofs::net::sched::act_police::offload_safety` |

### Layer 2: TLA+ models

(none new owned at this granularity; cross-ref `models/net/sch_htb.tla` from `sch-htb.md` for token-bucket invariants)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-action token-bucket state | `tcfp_tok ≤ burst`; `tcfp_ptok ≤ mtu_ptoks` after replenish | `kani::proofs::net::sched::act_police::tokens_invariants` |
| Single-rate vs two-rate selection | `peakrate.rate == 0` ⇔ single-rate mode (no `tcfp_ptok` updates) | `kani::proofs::net::sched::act_police::mode_invariants` |

### Layer 4: Functional correctness (opt-in)

- **RFC 2697 srTCM correctness theorem** via Verus — proves: time-averaged conform-rate ≤ CIR; burst absorption ≤ `burst` bytes within rate-burst window.
- **RFC 2698 trTCM correctness theorem** via Verus — proves: time-averaged conform+partial-exceed rate ≤ PIR; conform-only rate ≤ CIR; full-exceed kicks in beyond PIR.

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **SIZE_OVERFLOW** | Token-bucket arithmetic (rate × elapsed) + per-color stats use checked operators (CVE class: rate-overflow → unbounded throughput) | § Mandatory |
| **CONSTIFY** | `tc_action_ops` for police `static const` | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: per-action (cross-ref `net/sched/act-api.md`)
- **CONSTIFY, SIZE_OVERFLOW**: see above
- **USERCOPY**: NLA parsing uses bound-checked accessors
- **KERNEXEC**: action dispatch via `static const fn-ptr`

### Row-2 / GR-RBAC integration

- LSM hook: same as `act-api.md` (CAP_NET_ADMIN gate).
- Useful default GR-RBAC policy: deny police-action installation outside gradm-marked `network_admin` role; rate-bypass via misconfigured police is a denial-of-service primitive.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

