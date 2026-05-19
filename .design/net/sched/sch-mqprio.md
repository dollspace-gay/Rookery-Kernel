# Tier-3: net/sched/sch-mqprio — multi-queue priority qdisc + DCB integration

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/sched/sch_mqprio.c
  - net/sched/sch_mqprio_lib.c
  - include/net/sch_generic.h
  - include/uapi/linux/pkt_sched.h
  - include/uapi/linux/dcbnl.h
-->

## Summary
Tier-3 design for `mqprio` — a multi-queue priority qdisc that maps the netdev's TX queues into priority traffic classes (TCs). Unlike single-queue qdiscs (`fq_codel`, `htb`), `mqprio` is multi-queue-aware: it places one inner qdisc per netdev TX queue, then groups those queues into TCs whose mapping comes from the `tc_to_priority[]` and `priority_to_queue[]` tables. Hardware-offloadable on virtually every modern multi-queue NIC.

Two operating modes:
- **`MQPRIO_MODE_DCB`** (default): IEEE 802.1Q Data Center Bridging. Bandwidth allocation per-TC via DCBNL via `dcb_setapp` / `dcb_getapp`; PCP-based prioritization on Ethernet trunks.
- **`MQPRIO_MODE_CHANNEL`**: per-TC bandwidth shaping (min_rate + max_rate per TC), HW-offloaded.

Used by: high-throughput multi-queue setups (10/40/100 GbE), DCB-converged Ethernet (FCoE / RoCE / iSCSI in same fabric), TSN setups (mqprio frequently the inner qdisc beneath taprio time-aware scheduling).

Sub-tier-3 of `net/sched/00-overview.md`. Pairs with `net/sched/sch-taprio.md` (often layered atop mqprio for TSN time-aware shaping).

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| mqprio qdisc: per-TC mapping, multi-queue dispatch, NLA parser, dump | `net/sched/sch_mqprio.c` |
| mqprio shared helpers (used by taprio + mqprio) | `net/sched/sch_mqprio_lib.c` |
| Public API | `include/net/sch_generic.h` |
| UAPI: `TCA_MQPRIO_*` NLAs, `struct tc_mqprio_qopt` | `include/uapi/linux/pkt_sched.h` |
| DCBNL UAPI | `include/uapi/linux/dcbnl.h` |

## Compatibility contract

### `struct tc_mqprio_qopt` (root config)

```c
struct tc_mqprio_qopt {
    __u8 num_tc;
    __u8 prio_tc_map[TC_QOPT_MAX_QUEUE];   /* 16 entries: TC for each priority 0..15 */
    __u8 hw;                                /* 0 = software-only; 1 = HW-offload requested */
    __u16 count[TC_QOPT_MAX_QUEUE];         /* per-TC queue count */
    __u16 offset[TC_QOPT_MAX_QUEUE];        /* per-TC starting queue offset */
};
```

`TCA_OPTIONS` carries this for `tc qdisc add ... mqprio num_tc 4 map 0 1 2 3 ... queues 1@0 2@1 3@3 ...`. Layout-byte-identical.

### NLA configuration (extended attributes)

`TCA_OPTIONS` nested NLAs:
- `TCA_MQPRIO_MODE` (`MQPRIO_MODE_DCB | MQPRIO_MODE_CHANNEL`)
- `TCA_MQPRIO_SHAPER` (`TC_MQPRIO_SHAPER_DCB | TC_MQPRIO_SHAPER_BW_RATE`)
- `TCA_MQPRIO_MIN_RATE64` (per-TC min rate in bytes/sec — 64-bit; only with `SHAPER_BW_RATE`)
- `TCA_MQPRIO_MAX_RATE64` (per-TC max rate; only with `SHAPER_BW_RATE`)
- `TCA_MQPRIO_TC_ENTRY` (nested per-TC entry — fp/preemptable, gate, etc. for TSN integration)

Wire format byte-identical so iproute2's `tc qdisc add ... mqprio` works unchanged.

### Per-priority → per-TC mapping

`prio_tc_map[16]` maps `skb->priority` ∈ {0..15} → TC ∈ {0..num_tc-1}. Per-TC `offset[tc]` + `count[tc]` define the range of TX queues belonging to that TC; per-skb queue selected via `priority_to_queue` then within-TC ordinal.

Identical mapping algorithm.

### Per-TX-queue inner qdisc

Each netdev TX queue gets its own inner qdisc (default `pfifo_fast`, configurable via `tc qdisc replace dev eth0 parent <X>:<Y> ...`). mqprio's enqueue selects the queue then defers to the inner qdisc.

### HW offload (`hw=1`)

When `hw=1`, mqprio passes `tc_mqprio_qopt_offload` to driver via `ndo_setup_tc(TC_SETUP_QDISC_MQPRIO, ...)`. Driver returns success/failure; on success → driver's HW shaper handles per-TC rate-limiting + queue mapping. On failure → fall back to software path with `hw=0`.

Identical contract.

### DCBNL integration (`MQPRIO_MODE_DCB`)

In DCB mode: per-TC bandwidth allocation comes from netdev's DCB state via `dcb_getapp` / `dcb_setapp` instead of from per-TC NLAs. DCBx (LLDP-Application TLV) negotiates with peer.

### `MQPRIO_MODE_CHANNEL` per-TC rate-shaping

Per-TC `min_rate` + `max_rate` shaping; HW-offloaded if supported. Identical to per-class HTB shaping but at the multi-queue scheduler level.

### `TCA_MQPRIO_TC_ENTRY` (TSN integration)

Per-TC entry with extended attrs:
- `TCA_MQPRIO_TC_ENTRY_INDEX` (TC index)
- `TCA_MQPRIO_TC_ENTRY_FP` (frame-preemption — IEEE 802.1Qbu): `TC_FP_EXPRESS | TC_FP_PREEMPTIBLE`

Used by TSN-aware NICs that support frame preemption per IEEE 802.1Qbu.

## Requirements

- REQ-1: `struct tc_mqprio_qopt` byte-identical layout.
- REQ-2: All `TCA_MQPRIO_*` NLAs (per the table) parsed identically; defaults match upstream.
- REQ-3: Per-priority → per-TC mapping via `prio_tc_map[16]` + `offset[]` + `count[]`; identical algorithm.
- REQ-4: Per-TX-queue inner qdisc allocation; default `pfifo_fast`; replaceable via `tc qdisc replace`.
- REQ-5: HW offload via `ndo_setup_tc(TC_SETUP_QDISC_MQPRIO, &offload)` when `hw=1`; on driver failure → fall back to software path.
- REQ-6: `MQPRIO_MODE_DCB`: DCBNL integration via `dcb_getapp` / `dcb_setapp`; per-TC bandwidth from netdev DCB state.
- REQ-7: `MQPRIO_MODE_CHANNEL`: per-TC `min_rate` + `max_rate` 64-bit shaping (`TCA_MQPRIO_MIN_RATE64` / `MAX_RATE64`).
- REQ-8: `TCA_MQPRIO_TC_ENTRY` parser: per-TC frame-preemption (IEEE 802.1Qbu) `TC_FP_EXPRESS / PREEMPTIBLE` configuration.
- REQ-9: Per-TC stats: aggregated bytes/packets/drops returned via `tc -s class show` per-TC.
- REQ-10: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `pahole struct tc_mqprio_qopt` byte-identical (16 + 16 + 1 + 1 + 32 + 32 = 98 bytes plus padding). (covers REQ-1)
- [ ] AC-2: `tc qdisc add dev eth0 root mqprio num_tc 4 map 0 0 0 0 1 1 1 1 2 2 2 2 3 3 3 3 queues 4@0 4@4 4@8 4@12 hw 1` test: NLA byte-identical; visible in `tc qdisc show dev eth0`. (covers REQ-2, REQ-3, REQ-5)
- [ ] AC-3: Per-priority routing test: `skb->priority=5` → mapped to TC 1 (per the map); enqueued to one of queues 4..7; tcpdump on eth0 shows TC 1's packets routed to TC-1 queue range. (covers REQ-3)
- [ ] AC-4: Per-queue inner qdisc test: `tc qdisc replace dev eth0 parent 1:1 fq_codel`; per-queue qdisc replaceable; subsequent traffic through queue 0 sees fq_codel-classified output. (covers REQ-4)
- [ ] AC-5: HW-offload test on supporting NIC (Intel ixgbe/i40e/ice or Mellanox ConnectX-5+): `tc qdisc add ... mqprio ... hw 1`; `tc qdisc show` shows offload flag set; per-TC counters increment on traffic. (covers REQ-5)
- [ ] AC-6: DCB mode test: configure netdev with DCB IEEE 802.1Q app TLV per-TC bandwidth via `dcb` tooling; mqprio in DCB mode reads + applies. (covers REQ-6)
- [ ] AC-7: Channel mode rate test: `mqprio mode channel min_rate 100mbit 200mbit 300mbit 400mbit max_rate ...`; per-TC throughput respects rate. (covers REQ-7)
- [ ] AC-8: TC_FP_PREEMPTIBLE test on supporting TSN NIC: `tc qdisc add ... mqprio ... fp E E P P`; driver enables per-TC frame preemption via `TCA_MQPRIO_TC_ENTRY_FP`. (covers REQ-8)
- [ ] AC-9: Per-TC stats test: `tc -s class show dev eth0` returns per-TC byte/packet counters byte-identical to upstream. (covers REQ-9)
- [ ] AC-10: Hardening section present and follows template. (covers REQ-10)

## Architecture

### Rust module organization

- `kernel::net::sched::sch_mqprio::Mqprio` — qdisc instance
- `kernel::net::sched::sch_mqprio::PriorityMap` — `prio_tc_map[16]`
- `kernel::net::sched::sch_mqprio::TcQueueRange` — per-TC `offset` + `count`
- `kernel::net::sched::sch_mqprio::InnerQdiscs` — per-TX-queue inner qdisc
- `kernel::net::sched::sch_mqprio::HwOffload` — `ndo_setup_tc` bridge
- `kernel::net::sched::sch_mqprio::DcbMode` — DCBNL integration
- `kernel::net::sched::sch_mqprio::ChannelShaper` — per-TC min/max rate shaping
- `kernel::net::sched::sch_mqprio::FramePreemption` — IEEE 802.1Qbu integration
- `kernel::net::sched::sch_mqprio::lib::Shared` — `sch_mqprio_lib.c` shared helpers

### Locking and concurrency

- **Per-TX-queue qdisc lock** (inherited): held during enqueue/dequeue
- **No new global locks**: per-TC mapping is RCU-readable from hot path; mutator side under rtnl_lock during NLA-driven mutation

### Error handling

- `Err(EINVAL)` — bad NLA / inconsistent (e.g., sum of `count[]` != netdev's real_num_tx_queues)
- `Err(EOPNOTSUPP)` — driver doesn't support requested HW offload
- `Err(ENOMEM)` — alloc fail

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| `prio_tc_map` array bounds (priority ≤ TC_PRIO_MAX=15) | `kani::proofs::net::sched::sch_mqprio::priomap_safety` |
| Per-TC offset+count range (offset+count ≤ real_num_tx_queues) | `kani::proofs::net::sched::sch_mqprio::range_safety` |
| HW offload bridge (no use-after-free during driver-callback) | `kani::proofs::net::sched::sch_mqprio::offload_safety` |
| DCBNL bandwidth-table read (no stale pointer) | `kani::proofs::net::sched::sch_mqprio::dcb_safety` |

### Layer 2: TLA+ models

(none new owned)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-TC offset+count tiling | ∀ TX queue index `q`, exists exactly one TC with `offset[tc] ≤ q < offset[tc] + count[tc]` (queues are partitioned across TCs without gap or overlap) | `kani::proofs::net::sched::sch_mqprio::tiling_invariants` |
| Per-priority → per-TC mapping | `prio_tc_map[i] < num_tc` for all i ∈ 0..15 | `kani::proofs::net::sched::sch_mqprio::map_invariants` |

### Layer 4: Functional correctness (opt-in)

(deferred — same gate as `net/sched/00-overview.md`)

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **SIZE_OVERFLOW** | per-TC offset+count + per-TC rate arithmetic uses checked operators | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: per-skb (cross-ref `net/skbuff.md`); per-Qdisc (cross-ref `net/sched/sch-api.md`)
- **CONSTIFY**: `Qdisc_ops` for mqprio `static const`
- **SIZE_OVERFLOW**: see above
- **KERNEXEC**: per-method dispatch via `static const fn-ptr` array

### Row-2 / GR-RBAC integration

- LSM hook: same as `sch-api.md` (CAP_NET_ADMIN gate).
- Default GR-RBAC policy: empty.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Grsecurity/PaX-style Reinforcement

Beyond the Row-1 defaults above, sch_mqprio (multi-queue priority, DCB/802.1Q) inherits the following PaX/grsec primitives:

- **PAX_USERCOPY**: TCA_MQPRIO_* attributes (`mqprio_qopt`, `TCA_MQPRIO_TC_ENTRY`, `TCA_MQPRIO_MIN_RATE64`, `TCA_MQPRIO_MAX_RATE64`) parsed via `nla_*` whitelisted accessors with strict policy.
- **PAX_KERNEXEC**: `mqprio_qdisc_ops` is `__ro_after_init`; W^X enforced.
- **PAX_RANDKSTACK**: RTM_NEWQDISC + per-packet enqueue/dequeue protected.
- **PAX_REFCOUNT**: `Qdisc.refcnt` + per-TC sub-qdisc refcnts + per-TC qstats use checked `refcount_t` / `u64_stats_*`.
- **PAX_MEMORY_SANITIZE**: freed `mqprio_sched` (per-TC array + offload state) zeroed on destroy.
- **PAX_UDEREF**: rtnetlink walkers fenced.
- **PAX_RAP / kCFI**: `Qdisc_ops` + per-driver `ndo_setup_tc(TC_SETUP_QDISC_MQPRIO)` callbacks type-tagged + CFI-checked.
- **GRKERNSEC_HIDESYM**: mqprio internal symbols hidden from `/proc/kallsyms`.
- **GRKERNSEC_DMESG**: parse-failure / offload-fail printks rate-limited.

Component-specific reinforcement:

- RTM_NEWQDISC attaching mqprio requires **CAP_NET_ADMIN** in the netns owner; full-offload mode additionally requires the driver to advertise the feature, gating attacker-controlled offload calls.
- `num_tc` clamped to `[1, TC_QOPT_MAX_QUEUE = 16]`; `prio_tc_map[]` indexed by `skb->priority & TC_PRIO_MAX` only; out-of-range fails `-EINVAL`.
- Per-TC `(count, offset)` validated such that `offset + count ≤ dev->num_tx_queues` and ranges do not overlap; prevents one TC from poisoning another's TX ring.
- `min_rate64` / `max_rate64` clamped to `[0, U64_MAX / 8]` bps and `min_rate ≤ max_rate ≤ link_speed` (where link speed is known).
- Hardware-offload path (`TC_SETUP_QDISC_MQPRIO`) is dispatched via the kCFI-checked `ndo_setup_tc`; failure rolls back to in-kernel mqprio rather than leaving the NIC half-programmed.

Rationale: mqprio is the standard way to map skb priorities to NIC TX queues with hardware offload; a corrupted `prio_tc_map` or an out-of-range `(count, offset)` pair becomes a cross-queue DMA / RAP target. Bounding the map plus the driver offload kCFI + CAP gating keeps the multi-queue dispatch fail-closed.

## Open Questions

(none — mqprio semantics exhaustively specified by upstream + IEEE 802.1Q DCB)

## Out of Scope

- Time-aware shaping atop mqprio (cross-ref `net/sched/sch-taprio.md`)
- 32-bit-only paths
- Implementation code
