# Tier-3: net/sched/sch-taprio — IEEE 802.1Qbv time-aware shaper (TSN)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/sched/sch_taprio.c
  - net/sched/sch_mqprio_lib.c
  - include/net/sch_generic.h
  - include/uapi/linux/pkt_sched.h
-->

## Summary
Tier-3 design for `taprio` — Linux's IEEE 802.1Qbv "Enhancements for Scheduled Traffic" time-aware shaper, the cornerstone of Time-Sensitive Networking (TSN). Per-traffic-class gate-control list (GCL) defines a cyclic schedule where each (start_time, interval, gate_mask) entry opens or closes per-TC TX gates over a clock-driven cycle. Used by industrial control (PROFINET-IRT, EtherCAT, OPC-UA), automotive in-vehicle networks, audio/video bridging (AVB), and 5G fronthaul networks.

PTP-synchronized clock (cross-ref `net/ptp/00-overview.md` if exists; otherwise `drivers/ptp/`) provides the time base. taprio supports both software-only mode (kernel hrtimer-driven) and hardware-offload mode (NIC's HW scheduler — Intel ice/i40e, NXP, TI Sitara, etc.).

Sub-tier-3 of `net/sched/00-overview.md`. Layered atop `net/sched/sch-mqprio.md` (mqprio provides per-TC queues; taprio gates them per-cycle). The mandatory `models/net/sch_taprio.tla` TLA+ model proves time-window correctness against the IEEE 802.1Qbv admin/oper schedule.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| taprio qdisc: GCL, per-cycle scheduler, hrtimer/HW-offload modes, NLA parser, dump | `net/sched/sch_taprio.c` |
| Shared mqprio helpers (per-TC queue mapping) | `net/sched/sch_mqprio_lib.c` |
| Public API | `include/net/sch_generic.h` |
| UAPI: `TCA_TAPRIO_*` NLAs, `struct tc_mqprio_qopt`, GCL entries | `include/uapi/linux/pkt_sched.h` |

## Compatibility contract

### Gate Control List (GCL)

Per IEEE 802.1Qbv: cyclic schedule of GCL entries. Each entry:
- `cmd` (u8): `TC_TAPRIO_CMD_SET_GATES` (open gates for this interval) | `TC_TAPRIO_CMD_SET_AND_HOLD_MAC` | `TC_TAPRIO_CMD_SET_AND_RELEASE_MAC`
- `gate_mask` (u32): bitmask of TCs whose gates are open during `interval`
- `interval` (u32): nanoseconds for this entry

Cycle:
- `cycle_time` (u64 ns): total cycle length
- `cycle_time_extension` (u64 ns): extra time appended when needed for last-entry alignment

Schedule mutation transitions:
- **Admin schedule**: pending schedule loaded by user, scheduled to take effect at `base_time`
- **Oper schedule**: currently active schedule

Per IEEE 802.1Qbv: when `now >= base_time + N × cycle_time` for the smallest valid N, admin atomically becomes oper.

Identical state machine.

### NLA configuration

`TCA_OPTIONS` nested NLAs:
- `TCA_TAPRIO_ATTR_PRIOMAP` (struct tc_mqprio_qopt — same as mqprio: prio_tc_map + offset + count)
- `TCA_TAPRIO_ATTR_SCHED_BASE_TIME` (u64 ns — when this schedule starts)
- `TCA_TAPRIO_ATTR_SCHED_CYCLE_TIME` (u64 ns)
- `TCA_TAPRIO_ATTR_SCHED_CYCLE_TIME_EXTENSION` (u64 ns)
- `TCA_TAPRIO_ATTR_SCHED_ENTRY_LIST` (nested array of entries):
  - `TCA_TAPRIO_SCHED_ENTRY` (each):
    - `TCA_TAPRIO_SCHED_ENTRY_CMD` (u8)
    - `TCA_TAPRIO_SCHED_ENTRY_GATE_MASK` (u32)
    - `TCA_TAPRIO_SCHED_ENTRY_INTERVAL` (u32 ns)
- `TCA_TAPRIO_ATTR_FLAGS` (`TCA_TAPRIO_ATTR_FLAG_TXTIME_ASSIST | TCA_TAPRIO_ATTR_FLAG_FULL_OFFLOAD`)
- `TCA_TAPRIO_ATTR_TXTIME_DELAY` (u32 ns — for txtime-assist mode)
- `TCA_TAPRIO_ATTR_ADMIN_SCHED` / `TCA_TAPRIO_ATTR_OPER_SCHED` (read-only in dumps)
- `TCA_TAPRIO_ATTR_TC_ENTRY` (per-TC entry — fp/preemptable like mqprio)

Wire format byte-identical so iproute2's `tc qdisc add ... taprio` works unchanged.

### Three operating modes

| Mode | Trigger | Behavior |
|---|---|---|
| **Software** (default) | No flags | Kernel hrtimer fires per GCL entry; per-TC enqueue gated in software |
| **txtime-assist** | `TCA_TAPRIO_ATTR_FLAG_TXTIME_ASSIST` set | Software-side fills `skb->tstamp` with target send time; NIC's TX-time scheduler emits at exact time. Requires `etf` qdisc on inner queues. |
| **Full offload** | `TCA_TAPRIO_ATTR_FLAG_FULL_OFFLOAD` set | Driver implements `ndo_setup_tc(TC_SETUP_QDISC_TAPRIO, &offload)`; HW handles entire schedule + gate-control. |

Identical mode set.

### Per-TC gate enforcement

Per dequeue:
1. Compute current GCL entry: index `i = ((now - base_time) % cycle_time) / interval`
2. Check entry `i`'s `gate_mask` & (1 << tc): if zero → TC is closed; skip
3. Otherwise: dequeue from per-TC queue (delegating to mqprio sub-tree)

Identical algorithm.

### Cycle time validation

`cycle_time = Σ entry.interval` (with optional extension); admin schedule rejected with EINVAL if entries' total interval doesn't match declared `cycle_time`.

### `etf` (Earliest TxTime First) inner qdisc

In `txtime-assist` mode, `etf` (`net/sched/sch_etf.c`, separate Tier-3) is the inner qdisc on each per-TC queue; it sorts skbs by `skb->tstamp` and dequeues them in time order.

### Schedule transitions

User submits new admin schedule via NLA UPDATE; kernel installs it as pending; transition fires at the next `base_time + N × cycle_time` boundary. Pending → oper happens atomically (per IEEE 802.1Qbv § 8.6.9.2.1).

## Requirements

- REQ-1: GCL entries: `cmd / gate_mask / interval` per IEEE 802.1Qbv; layout-byte-identical.
- REQ-2: All `TCA_TAPRIO_*` NLAs (per the table) parsed identically; defaults match upstream.
- REQ-3: Three operating modes (software / txtime-assist / full-offload); identical mode dispatch.
- REQ-4: Per-TC gate enforcement: per-cycle GCL walk; gate_mask check per dequeue; identical arithmetic.
- REQ-5: Cycle time validation: `Σ entry.interval == cycle_time` (or extension applied); reject with EINVAL otherwise.
- REQ-6: Schedule transitions: admin → oper atomic at next `base_time + N × cycle_time`; identical IEEE 802.1Qbv § 8.6.9.2.1 semantics.
- REQ-7: PTP clock source: hrtimer scheduled against `CLOCK_TAI` (PTP-synchronized when ptp4l running); identical.
- REQ-8: Full HW offload via `ndo_setup_tc(TC_SETUP_QDISC_TAPRIO, &offload)`; on driver failure → return EOPNOTSUPP.
- REQ-9: txtime-assist mode: software-side computes per-skb send-time, fills `skb->tstamp`; requires `etf` inner qdisc.
- REQ-10: Per-TC frame preemption (IEEE 802.1Qbu) via `TCA_TAPRIO_ATTR_TC_ENTRY` shared with mqprio.
- REQ-11: TLA+ model `models/net/sch_taprio.tla` (per `net/sched/00-overview.md` Layer 4 — referenced theorem) — proves: each per-TC send-window matches the configured admin/oper schedule per IEEE 802.1Qbv; admin → oper transition atomicity.
- REQ-12: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: GCL wire-format test: `tc qdisc add dev eth0 root taprio num_tc 4 ... base-time 1000000000 cycle-time 1000000 sched-entry S 0xf 1000000` byte-identical NLA encoding. (covers REQ-1, REQ-2)
- [ ] AC-2: Software-mode gate test: configure 2-TC schedule with TC 0 open for 500us + TC 1 open for 500us; under traffic on both TCs → tcpdump on eth0 shows alternating-TC packets at 500us granularity ± hrtimer jitter. (covers REQ-3, REQ-4)
- [ ] AC-3: Cycle-validation test: `tc qdisc add ... taprio cycle-time 1000000 sched-entry S 1 600000 sched-entry S 2 500000` → EINVAL (sum > cycle_time). (covers REQ-5)
- [ ] AC-4: Admin → oper transition test: install initial schedule (oper); install new admin schedule with `base_time = now + 100ms`; tcpdump shows transition at exactly `base_time` boundary. (covers REQ-6)
- [ ] AC-5: PTP-driven test: with ptp4l synchronizing CLOCK_TAI; multi-host TSN setup; per-host TC-1 packets aligned to within ±1us of master-clock TC-1 window. (covers REQ-7)
- [ ] AC-6: Full-offload test on supporting NIC (Intel ice/i40e or NXP/TI TSN-capable): install with FULL_OFFLOAD flag → driver's `ndo_setup_tc` called; HW handles schedule. (covers REQ-8)
- [ ] AC-7: txtime-assist test: install with TXTIME_ASSIST + etf inner; per-skb `skb->tstamp` filled at expected send time; NIC TX-time scheduler emits within ±100ns of target. (covers REQ-9)
- [ ] AC-8: Frame-preemption test on supporting TSN NIC: per-TC FP=PREEMPTIBLE configured; preemptable frames are interrupted by express-frame arrival. (covers REQ-10)
- [ ] AC-9: TLA+ `models/net/sch_taprio.tla` proves: ∀ time `t` in oper schedule, per-TC gate state matches the GCL entry covering `t`; admin→oper transition is atomic. (covers REQ-11)
- [ ] AC-10: Hardening section present and follows template. (covers REQ-12)

## Architecture

### Rust module organization

- `kernel::net::sched::sch_taprio::Taprio` — qdisc instance
- `kernel::net::sched::sch_taprio::Gcl` — Gate Control List entries
- `kernel::net::sched::sch_taprio::Schedule` — admin + oper schedules + transition
- `kernel::net::sched::sch_taprio::CycleWalker` — per-cycle GCL walk
- `kernel::net::sched::sch_taprio::SwMode` — software hrtimer mode
- `kernel::net::sched::sch_taprio::TxtimeAssist` — txtime-assist mode w/ skb->tstamp fill
- `kernel::net::sched::sch_taprio::FullOffload` — driver-side `TC_SETUP_QDISC_TAPRIO`
- `kernel::net::sched::sch_taprio::PriorityMap` — shared with mqprio
- `kernel::net::sched::sch_taprio::FramePreemption` — IEEE 802.1Qbu integration

### Locking and concurrency

- **Per-qdisc `q.lock`** (spinlock, inherited): held during enqueue/dequeue
- **Schedule transition atomicity**: admin schedule pointer atomically swapped with oper at `base_time + N × cycle_time` boundary; readers use RCU
- **hrtimer**: per-cycle wakeup; CLOCK_TAI

### Error handling

- `Err(EINVAL)` — bad NLA / cycle_time mismatch / gate_mask refers to non-existent TC
- `Err(EOPNOTSUPP)` — full-offload + driver doesn't implement TC_SETUP_QDISC_TAPRIO
- `Err(ENOMEM)` — alloc fail
- `Err(ETIME)` — base_time in the past at install time (when configured strictly)

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| GCL walk arithmetic (no integer overflow on `(now - base_time) % cycle_time`) | `kani::proofs::net::sched::sch_taprio::gcl_walk_safety` |
| Schedule transition atomicity (admin → oper RCU-pointer swap; no readers see partial swap) | `kani::proofs::net::sched::sch_taprio::transition_safety` |
| Gate-mask bit access (no out-of-bounds; bit i < num_tc) | `kani::proofs::net::sched::sch_taprio::gate_mask_safety` |
| txtime-assist skb tstamp fill (no overflow on tstamp + delay) | `kani::proofs::net::sched::sch_taprio::txtime_safety` |

### Layer 2: TLA+ models

- `models/net/sch_taprio.tla` (referenced in `net/sched/00-overview.md` Layer 4 theorem) — proves time-window correctness. Owned here at this level (Layer 4 functional correctness gate).

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| GCL entries | Σ entry.interval ≤ cycle_time + cycle_time_extension; entry intervals all > 0 | `kani::proofs::net::sched::sch_taprio::gcl_invariants` |
| Schedule transition | exactly one of {admin alone, oper alone, admin+oper-pending-swap} at any observable instant | `kani::proofs::net::sched::sch_taprio::schedule_invariants` |
| GCL gate_mask | every set bit i in any gate_mask is < num_tc | `kani::proofs::net::sched::sch_taprio::mask_invariants` |

### Layer 4: Functional correctness (declared in `net/sched/00-overview.md`)

- **TAPRIO time-window correctness theorem** via TLA+ refinement — proves: each per-traffic-class send window matches the configured admin/oper schedule per IEEE 802.1Qbv. Owned here.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **SIZE_OVERFLOW** | cycle_time + base_time + interval arithmetic uses checked operators (CVE class: 64-bit time-overflow → wrap-around schedule) | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: per-skb (cross-ref `net/skbuff.md`); per-Qdisc (cross-ref `net/sched/sch-api.md`)
- **CONSTIFY**: `Qdisc_ops` for taprio `static const`
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

Baseline PaX/grsecurity mitigations applicable to taprio (time-aware shaper, IEEE 802.1Qbv):

- **PAX_USERCOPY** — `TCA_TAPRIO_ATTR_*` netlink schedule blobs (entries, base_time, cycle_time) traverse whitelisted `copy_{to,from}_user`.
- **PAX_KERNEXEC** — `taprio_enqueue`, `advance_sched`, and `find_entry_to_transmit` execute from W^X .text.
- **PAX_RANDKSTACK** — per-syscall stack randomization frustrates inference of gate-state timing.
- **PAX_REFCOUNT** — taprio `Qdisc`, child queue Qdiscs, and `taprio_sched` reference counts saturate on adversarial reschedule churn.
- **PAX_MEMORY_SANITIZE** — `sched_entry[]`, the swap-out admin schedule, and freed oper schedule sanitized on `taprio_destroy`.
- **PAX_UDEREF** — netlink schedule attribute walking under UDEREF.
- **PAX_RAP / kCFI** — `taprio_qdisc_ops` `static const`; `advance_sched` hrtimer callback CFI-checked.
- **GRKERNSEC_HIDESYM** — `taprio_get_time`, `find_entry_to_transmit` hidden from unprivileged kallsyms.
- **GRKERNSEC_DMESG** — schedule-mismatch / TXTIME-assist warnings CAP_SYSLOG-gated.

taprio-specific reinforcement:

- **TC qdisc CAP_NET_ADMIN** — `tc qdisc {add,replace,change} ... taprio` gated by CAP_NET_ADMIN in netdev's netns.
- **`taprio_qdisc_ops` PAX_RAP-typed** — vtable dispatch CFI-verified.
- **SIZE_OVERFLOW on `cycle_time + base_time` arithmetic** — checked operators prevent 64-bit time wrap that would invert gate decisions.
- **`hrtimer` advance-sched rate-limited via min cycle_time** — userspace cannot supply a near-zero cycle to flood softirq.
- **GRKERNSEC_HIDESYM on `taprio_get_time`** — clock-source inference restricted.

Rationale: taprio's correctness hinges on `cycle_time`/`base_time` arithmetic and the admin-vs-oper schedule swap; SIZE_OVERFLOW on the time math, sanitize-on-free of the admin schedule, and CFI on the hrtimer callback close the time-overflow and stale-schedule classes.

## Open Questions

(none — taprio semantics exhaustively specified by IEEE 802.1Qbv + upstream)

## Out of Scope

- Multi-queue priority below taprio (cross-ref `net/sched/sch-mqprio.md`)
- ETF (Earliest TxTime First) inner qdisc detail (cross-ref future `net/sched/sch-etf.md`)
- PTP-clock subsystem (cross-ref `drivers/ptp/00-overview.md`)
- 32-bit-only paths
- Implementation code
