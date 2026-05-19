# Tier-3: net/sched/sch-htb — HTB classful qdisc (hierarchical token bucket)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/sched/sch_htb.c
  - include/net/sch_generic.h
  - include/uapi/linux/pkt_sched.h
-->

## Summary
Tier-3 design for HTB (Hierarchical Token Bucket) — the workhorse classful qdisc used by virtually all bandwidth-shaping deployments (web hosting traffic-shaping, ISP CPE rate-limiting, Linux-on-server tenant isolation, CIRC-style tcil-shaped DSL configurations, embedded routers via OpenWrt SQM with HTB+fq_codel inner). Lets userspace declare a tree of classes with per-class rate (committed information rate) + ceil (peak), with parent classes lending their unused tokens to their children's borrowing children — implementing both reservation (rate is guaranteed) and excess-sharing (ceil is fair).

Each class has its own inner qdisc (often `pfifo` for the leaf, or `fq_codel`/`sfq` for fair distribution among that class's flows). HTB's per-class `tc_htb_rate` (rate-table-encoded byte/sec → time/byte mapping) drives token replenishment timers.

Sub-tier-3 of `net/sched/00-overview.md`. Owns the mandatory `models/net/sch_htb.tla` TLA+ model proving rate-bound invariants. Pairs with `net/sched/sch-fq-codel.md` (frequently the leaf qdisc), `net/sched/sch-cake.md` (alternative shaper that bundles HTB-style + FQ-CoDel into a single integrated unit).

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| HTB qdisc: per-class state, token-bucket arithmetic, hierarchy walk, NLA parser, dump | `net/sched/sch_htb.c` |
| Public API | `include/net/sch_generic.h` |
| UAPI: `TCA_HTB_*` NLAs + `struct tc_htb_opt`, `struct tc_htb_glob` | `include/uapi/linux/pkt_sched.h` |

## Compatibility contract

### `tc_htb_glob` (root-level config)

```c
struct tc_htb_glob {
    __u32 version;
    __u32 rate2quantum;     /* bps→quantum divisor */
    __u32 defcls;           /* default classid for unclassified packets */
    __u32 debug;
    __u32 direct_pkts;      /* count of pkts going directly to dev (root-direct) */
};
```

`TCA_HTB_INIT` carries this for `tc qdisc add ... htb`. Layout-byte-identical.

### `tc_htb_opt` (per-class config)

```c
struct tc_htb_opt {
    struct tc_ratespec rate;     /* CIR rate spec */
    struct tc_ratespec ceil;     /* PIR ceil spec */
    __u32 buffer;                 /* CIR token bucket size (~burst) */
    __u32 cbuffer;                /* PIR token bucket size (~cburst) */
    __u32 quantum;                /* DRR quantum for sibling distribution */
    __u32 level;                  /* class level in tree (leaf=0) */
    __u32 prio;                   /* priority among siblings */
};
```

`TCA_HTB_PARMS` carries this for `tc class add ... htb rate <X> ceil <Y> burst <Z>`.

### `struct tc_ratespec`

```c
struct tc_ratespec {
    unsigned char  cell_log;
    __u8           linklayer;     /* TC_LINKLAYER_ETHERNET / ATM */
    unsigned short overhead;
    short          cell_align;
    unsigned short mpu;            /* min packet unit */
    __u32          rate;           /* bytes/sec */
};
```

Layout-byte-identical for all rate-spec users (HTB, CBQ-deprecated, TBF, …).

### Per-class state (token bucket)

Per-class `htb_class`:
- `tokens` (s64): current CIR-bucket fill in nanoseconds-of-rate
- `ctokens` (s64): current PIR-bucket fill in nanoseconds-of-rate
- `mbuffer` / `mcbuffer`: max burst (CIR-buffer in nanoseconds)
- `cmode`: `HTB_CANT_SEND | HTB_MAY_BORROW | HTB_CAN_SEND`
- `feed[]`: per-priority round-robin list of children
- `wait_q`: wait-queue for tokens to refill
- `t_c`: timestamp of last update

Token replenishment per-dequeue:
```
elapsed = now - t_c
tokens   = min(tokens   + (elapsed × rate),   buffer)
ctokens  = min(ctokens  + (elapsed × ceil),   cbuffer)
```

If `tokens >= 0` → `HTB_CAN_SEND`; if `tokens < 0 && ctokens >= 0` → `HTB_MAY_BORROW`; if `ctokens < 0` → `HTB_CANT_SEND`.

Identical state machine.

### Hierarchy walk on dequeue

1. Iterate priorities high→low (HTB_PRIO_LEVELS=8)
2. For each priority, walk the per-prio feed[] DRR list
3. For each leaf class:
   - If CAN_SEND or MAY_BORROW (and parent has CIR/borrow tokens): dequeue from leaf inner qdisc
   - Subtract dequeued bytes from this class's tokens; if MAY_BORROW, also subtract from parent chain via `htb_charge_class`
4. If no class can send → schedule wakeup at next-replenish-time

Identical algorithm.

### `htb_charge_class` — borrow charging

When MAY_BORROW: walk up parents charging each level's `tokens` (CIR) until a level has CAN_SEND. The PIR-bucket (`ctokens`) is charged at every level always.

Identical algorithm.

### NLA configuration

`TCA_OPTIONS` nested NLAs:
- `TCA_HTB_PARMS` (struct tc_htb_opt)
- `TCA_HTB_INIT` (struct tc_htb_glob)
- `TCA_HTB_CTAB` (CIR rate table)
- `TCA_HTB_RTAB` (PIR rate table — pre-computed time/byte for fast lookup)
- `TCA_HTB_DIRECT_QLEN`
- `TCA_HTB_RATE64` (64-bit rate when rate exceeds 32-bit `tc_ratespec.rate`)
- `TCA_HTB_CEIL64`
- `TCA_HTB_OFFLOAD` (HW offload mode)

Wire format byte-identical so iproute2's `tc qdisc/class add ... htb` works unchanged.

### Per-class /sys/class statistics

Per `tc -s class show`:
- `tokens` / `ctokens` (current values)
- `bytes` / `pkts` / `drops` / `overlimits` / `requeues`
- `lended` (bytes lent to children)
- `borrowed` (bytes borrowed from parent)
- `giants` (oversize packets that wouldn't fit in any single token-grant)
- `direct_pkts`

Format byte-identical.

### HW offload (`TCA_HTB_OFFLOAD`)

When set, HTB delegates per-class shaping to the NIC's queueing engine (when supported by the driver — Mellanox, Intel ice). Identical contract.

## Requirements

- REQ-1: All `TCA_HTB_*` NLAs (per the table) parsed identically; `tc_htb_opt` + `tc_htb_glob` layout-byte-identical.
- REQ-2: Per-class token bucket: `tokens` / `ctokens` semantics + replenishment formula identical.
- REQ-3: `cmode` state machine: `HTB_CANT_SEND | HTB_MAY_BORROW | HTB_CAN_SEND` transitions identical.
- REQ-4: Hierarchy dequeue walk: priority-ordered (8 levels) DRR among siblings; first eligible class wins.
- REQ-5: Borrow charging via `htb_charge_class`: walk up parents until CAN_SEND; PIR always charged at every level.
- REQ-6: Rate tables (`TCA_HTB_RTAB` / `CTAB`): pre-computed rate-spec → time-per-byte fast-lookup table.
- REQ-7: 64-bit rate support via `TCA_HTB_RATE64` / `TCA_HTB_CEIL64` (when rate > 32-bit limit ~34 Gbit/s).
- REQ-8: Per-class statistics: `tokens` / `ctokens` / `lended` / `borrowed` / `giants` byte-identical to upstream.
- REQ-9: HW offload via `TCA_HTB_OFFLOAD`; per-driver shaper-engine delegation; identical contract.
- REQ-10: TLA+ model `models/net/sch_htb.tla` (mandatory per `net/sched/00-overview.md` Layer 2) — proves: per-class actual bandwidth ≤ class.rate + ceil-driven borrow within class.cburst window; hierarchy rate-sum bounded.
- REQ-11: `direct_pkts` semantics: packets without classid → directly enqueued on netdev (bypasses HTB hierarchy, counted in TCA_HTB_GLOB direct_pkts).
- REQ-12: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `pahole struct tc_htb_opt` + `struct tc_htb_glob` + `struct tc_ratespec` byte-identical layout. (covers REQ-1)
- [ ] AC-2: `tc qdisc add dev eth0 root handle 1: htb default 30; tc class add dev eth0 parent 1: classid 1:1 htb rate 100mbit ceil 100mbit; tc class add dev eth0 parent 1:1 classid 1:10 htb rate 30mbit ceil 100mbit; tc class add dev eth0 parent 1:1 classid 1:20 htb rate 70mbit ceil 100mbit` test: NLA byte-identical; classes visible in `tc -s class show dev eth0`. (covers REQ-1, REQ-4)
- [ ] AC-3: Rate-respect test: with above hierarchy + iperf3 to 1:10 + iperf3 to 1:20, sum throughput ≤ 100mbit; per-class throughput respects rate (≥ 30 / 70 mbit) + ceil-borrow when other class idle. (covers REQ-2, REQ-3, REQ-5)
- [ ] AC-4: 64-bit rate test: `tc class add ... rate 50gbit ceil 50gbit` succeeds; rate stored in TCA_HTB_RATE64 / TCA_HTB_CEIL64; throughput on 100 Gbps NIC tracks. (covers REQ-7)
- [ ] AC-5: `direct_pkts` test: send packet not matching any classifier → directly enqueued on netdev; `tc -s qdisc show` shows direct_pkts increment. (covers REQ-11)
- [ ] AC-6: Token-replenishment test: idle class for 1s → `tokens` reaches `buffer` cap; subsequent dequeue allowed up to burst. (covers REQ-2)
- [ ] AC-7: Borrow test: leaf 1:10 (rate 30, ceil 100); other class idle → 1:10 can dequeue at 100mbit (using parent's CIR borrow); `borrowed` counter increments. (covers REQ-5, REQ-8)
- [ ] AC-8: HW-offload test on supporting NIC: install HTB with TCA_HTB_OFFLOAD; `tc -s class show` shows `in_hw_count` increment. (covers REQ-9)
- [ ] AC-9: TLA+ `models/net/sch_htb.tla` proves: ∀ class `c` with continuous traffic, time-averaged dequeue rate ≤ c.rate + (ceil-borrow contribution within cburst); hierarchy rate-sum ≤ root.ceil. (covers REQ-10)
- [ ] AC-10: Hardening section present and follows template. (covers REQ-12)

## Architecture

### Rust module organization

- `kernel::net::sched::sch_htb::Htb` — qdisc instance
- `kernel::net::sched::sch_htb::HtbClass` — per-class state
- `kernel::net::sched::sch_htb::TokenBucket` — CIR/PIR token-bucket arithmetic
- `kernel::net::sched::sch_htb::HierarchyWalker` — dequeue priority + DRR walk
- `kernel::net::sched::sch_htb::Borrow` — `htb_charge_class` walk up parents
- `kernel::net::sched::sch_htb::RateTable` — RTAB/CTAB pre-computed lookup
- `kernel::net::sched::sch_htb::Direct` — direct-pkt path
- `kernel::net::sched::sch_htb::Offload` — HW offload bridge

### Locking and concurrency

- **Per-qdisc `q.lock`** (spinlock, inherited): held during enqueue/dequeue
- **No new locks**: per-class state accessed only under qdisc lock

### Error handling

- `Err(EINVAL)` — bad NLA / inconsistent params (e.g., ceil < rate)
- `Err(EBUSY)` — can't change rate while children present
- `Err(ENOMEM)` — alloc fail

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Token-bucket replenishment arithmetic (no overflow on 64-bit elapsed × rate) | `kani::proofs::net::sched::sch_htb::tokens_safety` |
| Hierarchy walk priority levels (≤ HTB_PRIO_LEVELS=8) | `kani::proofs::net::sched::sch_htb::walk_safety` |
| `htb_charge_class` parent-walk (no infinite loop on broken hierarchy) | `kani::proofs::net::sched::sch_htb::charge_safety` |
| RTAB/CTAB lookup (no out-of-bounds) | `kani::proofs::net::sched::sch_htb::rtab_safety` |
| Per-class DRR `quantum` arithmetic | `kani::proofs::net::sched::sch_htb::drr_safety` |

### Layer 2: TLA+ models

- `models/net/sch_htb.tla` (mandatory per `net/sched/00-overview.md`) — proves rate-bound invariants. Owned here.

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| HTB hierarchy tree | every non-root class has a non-NULL parent; tree depth ≤ ~10 levels practically; no cycles | `kani::proofs::net::sched::sch_htb::tree_invariants` |
| Per-class token bucket | `tokens ≤ buffer`; `ctokens ≤ cbuffer`; both ≥ minimum (negative for DEBT-state but bounded) | `kani::proofs::net::sched::sch_htb::tokens_invariants` |
| Per-priority feed list | classes in feed[i] all have prio == i; DRR-deficit ≥ 0 | `kani::proofs::net::sched::sch_htb::feed_invariants` |

### Layer 4: Functional correctness (opt-in; declared in `net/sched/00-overview.md` Layer 4)

- **HTB rate-bound theorem** via Verus — proves: per-class actual bandwidth ≤ class.rate + ceil-driven-borrow within class.cburst window.
- **Hierarchy work-conservation theorem** via Verus — proves: when any class has packets to send AND the link is idle, HTB dequeues something (no deadlock).

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **SIZE_OVERFLOW** | Token-bucket arithmetic (rate × elapsed) + RTAB lookup uses checked operators (CVE class: rate-overflow → unbounded throughput) | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: per-class slabs (cross-ref `net/sched/sch-api.md`); per-skb (cross-ref `net/skbuff.md`)
- **CONSTIFY**: `Qdisc_ops` for HTB `static const`
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

Beyond the Row-1 defaults above, sch_htb (Hierarchical Token Bucket) inherits the following PaX/grsec primitives:

- **PAX_USERCOPY**: TCA_HTB_PARMS / TCA_HTB_INIT / TCA_HTB_CTAB / TCA_HTB_RTAB attributes parsed via `nla_*` whitelisted accessors.
- **PAX_KERNEXEC**: `htb_qdisc_ops` and `htb_class_ops` are `__ro_after_init`; W^X enforced.
- **PAX_RANDKSTACK**: RTM_NEWQDISC + RTM_NEWTCLASS + per-packet enqueue/dequeue (softirq) protected.
- **PAX_REFCOUNT**: `Qdisc.refcnt`, every `htb_class.refcnt`, per-class `bstats` / `qstats` use checked `refcount_t` / `u64_stats_*`.
- **PAX_MEMORY_SANITIZE**: freed `htb_class` slab entries zeroed on `htb_destroy_class`.
- **PAX_UDEREF**: rtnetlink + class-walker fenced.
- **PAX_RAP / kCFI**: `Qdisc_ops` and `Qdisc_class_ops` callbacks type-tagged + CFI-checked.
- **GRKERNSEC_HIDESYM**: HTB internal symbols hidden from `/proc/kallsyms`.
- **GRKERNSEC_DMESG**: parse-failure / hierarchy-loop printks rate-limited and CAPSYSLOG-gated.

Component-specific reinforcement:

- RTM_NEWQDISC / RTM_NEWTCLASS for HTB require **CAP_NET_ADMIN** in the netns owner; HTB hierarchy mutation through `htb_change_class` re-checks CAP on every call.
- `htb_class.refcnt` (the children + filter refcount) uses checked `refcount_t`; destroying a class with non-zero refcount is refused (`-EBUSY`).
- Hierarchy depth is bounded by `TC_HTB_MAXDEPTH = 8`; class-id graph is validated for loops at `htb_change_class` time before insertion.
- `rate64` / `ceil64` clamped to `[0, U64_MAX / 8]` bps; `burst` / `cburst` clamped to `[mtu, 1<<28]`. Out-of-range fails `-EINVAL`.
- Per-class `tokens` / `ctokens` accounting uses signed saturating arithmetic; cannot wrap to grant a class unbounded service.
- `quantum` clamped per upstream `[mtu, 1<<20]`; default-quantum fallback is computed from `rate` with bounded arithmetic.

Rationale: HTB is the most complex classful shaper in mainline; its hierarchy + per-class refcount + per-class rate table are all attacker-reachable from `tc class add`. Bounding depth + rate + tokens, plus refusing destruction of a refcounted class, plus the PaX + RAP/kCFI envelope, keeps the hierarchy from being weaponized into an OOM or a token-overflow service-bypass.

## Open Questions

(none — HTB semantics exhaustively specified by upstream + the original "HTB classfull qdisc" paper)

## Out of Scope

- Other shaping qdiscs (cross-ref `net/sched/sch-cake.md`, `sch-tbf.md`)
- 32-bit-only paths
- Implementation code
