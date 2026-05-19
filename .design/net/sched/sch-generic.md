# Tier-3: net/sched/sch-generic — qdisc generic helpers + default qdiscs (pfifo_fast / noqueue / noop / mq)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/sched/sch_generic.c
  - net/sched/sch_mq.c
  - include/net/sch_generic.h
  - include/uapi/linux/pkt_sched.h
-->

## Summary
Tier-3 design for the qdisc generic-helpers layer + the kernel's built-in baseline qdiscs that virtually every netdev uses on boot before userspace replaces them. Provides:
- `noop_qdisc` / `noop_enqueue`/`dequeue`: pre-init placeholder used during netdev creation (drops everything)
- `noqueue_qdisc`: pseudo-qdisc for virtual netdevs that don't need queueing (loopback, dummy, veth in some configurations)
- `pfifo_fast_qdisc` / `pfifo_qdisc` / `bfifo_qdisc`: legacy 3-band priority FIFO; default before `fq_codel` became kernel-default in 4.12 (still configurable as fallback default via `net.core.default_qdisc=pfifo_fast`)
- `mq_qdisc` (`sch_mq.c`): multi-queue meta-qdisc that one-qdisc-per-TX-queue auto-creates when netdev has > 1 TX queue (no explicit `tc qdisc` needed); the inner per-queue qdiscs are typically `pfifo_fast` or `fq_codel`
- Per-qdisc helper primitives: `__qdisc_enqueue_tail`, `__qdisc_dequeue_head`, `qdisc_drop`, `qdisc_drop_all`, `qdisc_tree_reduce_backlog`, `qdisc_watchdog_*`
- The per-qdisc `gso_skb` slot for retry-after-driver-reject of GSO-segmented skbs
- The per-qdisc lock + state machine (`__QDISC_STATE_SCHED / DEACTIVATED / MISSED / DRAINING`)

Sub-tier-3 of `net/sched/00-overview.md`. Foundational layer underneath every other qdisc Tier-3 (sch_fq_codel, sch_htb, sch_cake, …).

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Generic helpers + noop/noqueue/pfifo_fast/pfifo/bfifo qdiscs + per-qdisc state machine | `net/sched/sch_generic.c` |
| `mq` multi-queue meta-qdisc | `net/sched/sch_mq.c` |
| Public API | `include/net/sch_generic.h` |
| UAPI: TCA_PRIO_*, struct tc_prio_qopt, struct tc_fifo_qopt | `include/uapi/linux/pkt_sched.h` |

## Compatibility contract

### `noop_qdisc` (placeholder)

Single per-system `noop_qdisc` instance used during netdev create + early init. `noop_enqueue` returns NET_XMIT_CN (drop with congestion notification). `noop_dequeue` returns NULL.

Identical to upstream so netdev-init sequence identical.

### `noqueue_qdisc` (no-queueing virtual netdevs)

For virtual netdevs that don't queue (e.g., `loopback`, `bridge`, `veth` in some configurations): set `noqueue_qdisc` so packets pass through without buffering. dev_queue_xmit goes straight to driver xmit.

### `pfifo_fast_qdisc` (3-band priority FIFO)

`include/uapi/linux/pkt_sched.h`:
```c
struct tc_prio_qopt {
    int    bands;
    __u8   priomap[TC_PRIO_MAX + 1];   /* 16 entries: priority → band */
};
```

Default `bands=3` with `priomap = {1, 2, 2, 2, 1, 2, 0, 0, 1, 1, 1, 1, 1, 1, 1, 1}` mapping skb priority → band:
- Band 0 (highest priority): skb priorities 6, 7
- Band 1 (default): skb priorities 0, 4, 8, 9, ..., 15
- Band 2 (lowest priority): skb priorities 1, 2, 3, 5

Strict priority among bands; FIFO within each band. `txq->qdisc->q.qlen` shared.

Layout-byte-identical so legacy `tc qdisc add ... pfifo_fast` works unchanged.

### `pfifo_qdisc` and `bfifo_qdisc`

- `pfifo`: simple FIFO with a per-qdisc packet limit (`TCA_FIFO_PACKET_LIMIT`, default = `tx_queue_len`)
- `bfifo`: simple FIFO with a per-qdisc byte limit

Configured via `tc qdisc add ... pfifo limit 1000`. UAPI `struct tc_fifo_qopt`:
```c
struct tc_fifo_qopt {
    __u32 limit;     /* either packets or bytes; per-qdisc */
};
```

Layout-byte-identical.

### `mq` (multi-queue meta-qdisc)

`sch_mq.c`: When a netdev has `real_num_tx_queues > 1`, kernel auto-creates a root `mq_qdisc` with one inner qdisc per TX queue (default `pfifo_fast`, configurable). The inner qdiscs are individually replaceable via `tc qdisc replace dev eth0 parent <X>:<Y>` where `<Y>` is the per-queue handle.

The mq qdisc itself doesn't enqueue/dequeue (it's a meta-shell); per-skb dispatch lands directly on the per-TX-queue inner qdisc.

Identical model.

### Qdisc state machine

Per-qdisc `state` field:
- `__QDISC_STATE_SCHED` (1<<0): set when qdisc is in __netif_schedule list
- `__QDISC_STATE_DEACTIVATED` (1<<1): set during qdisc tear-down; rejects new packets
- `__QDISC_STATE_MISSED` (1<<2): set when dequeue returned NULL but packet appeared mid-call (concurrency hint)
- `__QDISC_STATE_DRAINING` (1<<3): set during draining for replace

### Per-qdisc helpers (used by every qdisc Tier-3)

- `__qdisc_enqueue_tail(skb, q)` — enqueue at tail of skb_queue
- `__qdisc_dequeue_head(q)` — dequeue head
- `qdisc_drop(skb, sch, to_free)` — drop with stat update
- `qdisc_drop_all(sch, to_free)` — drop all queued skbs
- `qdisc_tree_reduce_backlog(sch, n_packets, n_bytes)` — propagate backlog reduction up tree
- `qdisc_watchdog_init/schedule_ns/cancel` — per-qdisc hrtimer for delayed dequeue (used by token-bucket qdiscs)

### `gso_skb` retry slot

Per-qdisc `gso_skb` field caches the most-recently-dequeued GSO skb if driver returned BUSY/COLLISION. Next dequeue returns this cached skb first before peeking inner qdisc. Identical retry semantics.

### Default-qdisc selection

Per `qdisc_create_dflt`:
1. Read `net.core.default_qdisc` sysctl
2. If non-empty + ops registered → use that
3. Else → `pfifo_fast` (legacy default)

Identical to upstream so distros' `default_qdisc=fq_codel` setting persists identically.

## Requirements

- REQ-1: `noop_qdisc` + `noop_enqueue` + `noop_dequeue` semantics identical (drop + return NULL).
- REQ-2: `noqueue_qdisc` semantics identical (no buffering; pass-through to driver).
- REQ-3: `pfifo_fast_qdisc`: 3-band strict-priority FIFO with default `priomap` byte-identical to upstream.
- REQ-4: `pfifo_qdisc` + `bfifo_qdisc`: simple FIFO with per-qdisc packet/byte limit; `struct tc_fifo_qopt` byte-identical.
- REQ-5: `mq_qdisc` auto-create on `real_num_tx_queues > 1`; one-inner-qdisc-per-TX-queue (default pfifo_fast); identical algorithm.
- REQ-6: Qdisc state machine: `__QDISC_STATE_*` bits identical; transitions match upstream.
- REQ-7: Per-qdisc helpers (`__qdisc_enqueue_tail`, `__qdisc_dequeue_head`, `qdisc_drop`, `qdisc_drop_all`, `qdisc_tree_reduce_backlog`, `qdisc_watchdog_*`) semantics identical.
- REQ-8: `gso_skb` retry-cache: per-qdisc cached GSO skb on driver BUSY; next dequeue returns cached first.
- REQ-9: Default-qdisc selection per `net.core.default_qdisc` sysctl; fallback to `pfifo_fast`.
- REQ-10: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `pahole struct Qdisc` first cache-line byte-identical (cross-ref `sch-api.md`). Includes `noop_qdisc` instance pointers. (covers REQ-1)
- [ ] AC-2: noop_qdisc test: netdev created without explicit qdisc → root qdisc is noop_qdisc; packets enqueued during init → dropped + NET_XMIT_CN. (covers REQ-1)
- [ ] AC-3: noqueue test: dummy netdev with noqueue_qdisc → ip-link traffic passes through without buffering; loopback round-trip works. (covers REQ-2)
- [ ] AC-4: pfifo_fast priomap test: send 16 skbs with priorities 0..15 through `pfifo_fast` root → tcpdump shows skbs dequeued in band 0 (prio 6,7) → band 1 (prio 0,4,8..15) → band 2 (prio 1,2,3,5) order. (covers REQ-3)
- [ ] AC-5: pfifo limit test: `tc qdisc add ... pfifo limit 100`; enqueue 101 skbs → 101st dropped + counter increment. (covers REQ-4)
- [ ] AC-6: bfifo limit test: `tc qdisc add ... bfifo limit 10000` → byte-limit enforced; aggregate byte-sum at-cap drops next skb. (covers REQ-4)
- [ ] AC-7: mq auto-create test: netdev with `ethtool -L combined 8` → root qdisc is `mq`; `tc qdisc show` shows 8 per-queue inner qdiscs. (covers REQ-5)
- [ ] AC-8: mq inner-replace test: `tc qdisc replace dev eth0 parent <X>:1 fq_codel`; only TX queue 0's inner qdisc replaced. (covers REQ-5)
- [ ] AC-9: gso_skb retry test: driver returns NETDEV_TX_BUSY for an oversize GSO skb → next dequeue returns same skb (from gso_skb cache); driver eventually accepts. (covers REQ-8)
- [ ] AC-10: default_qdisc test: `sysctl net.core.default_qdisc=pfifo_fast`; create new netdev → root inner qdiscs are pfifo_fast; reset to fq_codel → new netdev's qdiscs are fq_codel. (covers REQ-9)
- [ ] AC-11: Hardening section present and follows template. (covers REQ-10)

## Architecture

### Rust module organization

- `kernel::net::sched::sch_generic::Noop` — placeholder qdisc
- `kernel::net::sched::sch_generic::Noqueue` — pass-through qdisc
- `kernel::net::sched::sch_generic::PfifoFast` — 3-band priority FIFO
- `kernel::net::sched::sch_generic::Pfifo` — simple FIFO with packet limit
- `kernel::net::sched::sch_generic::Bfifo` — simple FIFO with byte limit
- `kernel::net::sched::sch_generic::Mq` — multi-queue meta-qdisc
- `kernel::net::sched::sch_generic::Helpers` — `__qdisc_enqueue_tail`, etc.
- `kernel::net::sched::sch_generic::StateMachine` — `__QDISC_STATE_*` bit ops
- `kernel::net::sched::sch_generic::GsoRetry` — `gso_skb` cache + retry
- `kernel::net::sched::sch_generic::DefaultSelector` — `default_qdisc` sysctl

### Locking and concurrency

- **Per-qdisc `q.lock`** (spinlock, declared in struct Qdisc): per-qdisc state mutator
- **Per-netdev `qdisc_tree_mutex`** (cross-ref `sch-api.md`): held during qdisc replace
- **Per-qdisc `gso_skb` access**: under `q.lock`

### Error handling

- `NET_XMIT_CN` — congestion notification on overflow
- `NET_XMIT_DROP` — drop without notification
- `Err(EINVAL)` — bad NLA / bad sysctl value
- `Err(ENOMEM)` — alloc fail

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| `__qdisc_enqueue_tail` + `__qdisc_dequeue_head` skb_queue manipulation | `kani::proofs::net::sched::sch_generic::queue_safety` |
| `qdisc_tree_reduce_backlog` walk up qdisc tree (no infinite loop) | `kani::proofs::net::sched::sch_generic::backlog_safety` |
| State-machine bit-ops (atomic test_and_set / clear) | `kani::proofs::net::sched::sch_generic::state_safety` |
| `gso_skb` retry slot (no double-dequeue + no leak) | `kani::proofs::net::sched::sch_generic::gso_safety` |
| `mq_qdisc` per-TX-queue inner-qdisc bind/unbind | `kani::proofs::net::sched::sch_generic::mq_safety` |

### Layer 2: TLA+ models

(none new owned at this granularity; per-qdisc Tier-3s declare per-qdisc TLA+ models)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-qdisc state bits | mutually-exclusive bit pairs honored: `DEACTIVATED ⇒ ¬ SCHED`; `DRAINING ⇒ ¬ enqueue-allowed` | `kani::proofs::net::sched::sch_generic::state_invariants` |
| pfifo_fast priomap | every entry in priomap[] is < bands; total queue length = sum of per-band lengths | `kani::proofs::net::sched::sch_generic::priomap_invariants` |
| mq inner qdiscs | exactly `real_num_tx_queues` inner qdiscs allocated; each has a unique handle | `kani::proofs::net::sched::sch_generic::mq_invariants` |

### Layer 4: Functional correctness (opt-in)

- **pfifo_fast strict-priority theorem** via Verus — proves: ∀ skbs S1 in higher-priority band, S2 in lower band: S1 dequeued before S2 if both queued at same time.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **CONSTIFY** | `Qdisc_ops` for noop/noqueue/pfifo_fast/pfifo/bfifo/mq `static const` | § Mandatory |
| **SIZE_OVERFLOW** | per-qdisc qlen + backlog arithmetic uses checked operators | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: per-Qdisc (cross-ref `net/sched/sch-api.md`); per-skb (cross-ref `net/skbuff.md`)
- **CONSTIFY, SIZE_OVERFLOW**: see above
- **KERNEXEC**: per-qdisc dispatch via `static const fn-ptr` array

### Row-2 / GR-RBAC integration

- LSM hook: same as `sch-api.md` (CAP_NET_ADMIN gate).
- Default GR-RBAC policy: empty.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Grsecurity/PaX-style Reinforcement

Beyond the Row-1 defaults above, sch_generic (the dequeue / tx-softirq core for every qdisc) inherits the following PaX/grsec primitives:

- **PAX_USERCOPY**: sch_generic itself takes no user input, but every skb it carries was constructed through a USERCOPY-checked allocator (`pskb_*`, `__alloc_skb`).
- **PAX_KERNEXEC**: `noop_qdisc`, `pfifo_fast_ops`, `mq_qdisc_ops` all `__ro_after_init`; W^X enforced on the generic dequeue path.
- **PAX_RANDKSTACK**: every entry into `__qdisc_run` from `NET_TX_SOFTIRQ` participates in kstack randomization.
- **PAX_REFCOUNT**: `Qdisc.refcnt`, `Qdisc.q.qlen`, every `qstats` counter use checked `refcount_t` / `u64_stats_*`; saturating.
- **PAX_MEMORY_SANITIZE**: `qdisc_alloc` + `qdisc_destroy` zero the private region.
- **PAX_UDEREF**: dequeue path never dereferences a userspace pointer; enforced by sparse + objtool.
- **PAX_RAP / kCFI**: every `Qdisc_ops` callback (`enqueue`, `dequeue`, `peek`, `reset`) is invoked via CFI-checked indirect; the per-cpu `pfifo_fast` and `noop` op tables are type-tagged.
- **GRKERNSEC_HIDESYM**: dequeue-loop symbols hidden from `/proc/kallsyms` for unprivileged readers.
- **GRKERNSEC_DMESG**: `Dead loop on netdevice ...` and watchdog printks rate-limited and gated to CAPSYSLOG.

Component-specific reinforcement:

- `__qdisc_run` bounds its loop by `quota = dev_tx_weight` and `time_after_eq(jiffies, ...)` to prevent a single qdisc dequeue from starving NET_TX_SOFTIRQ.
- `NET_TX_SOFTIRQ` is scheduled with `raise_softirq_irqoff`; tx-watchdog (`dev_watchdog`) fires on stuck TX with PaX-bounded `tx_timeout` callback dispatch.
- `noop_qdisc` is mapped into `dev->qdisc` for un-initialized devices — it is a literal `.rodata` constant; a corrupted `dev->qdisc` cannot point at attacker memory without tripping kCFI on the next dequeue.
- `qdisc_lookup` and `qdisc_match_from_root` traverse a refcounted hash; orphan / dangling Qdiscs are released through RCU after their refcount drops, never freed under the dequeue.
- Per-cpu `gnet_stats_basic_sync` for `pfifo_fast` uses `u64_stats_*` to remain SMP-safe under saturating arithmetic.

Rationale: sch_generic is *the* per-packet kernel dispatch surface; any RAP/kCFI miss or refcount wrap here is a kernel-RCE primitive that fires on every NIC tx. The PaX layer plus saturating qstats plus the bounded dequeue loop keep this path fail-closed.

## Open Questions

(none — sch_generic semantics are exhaustively specified by upstream)

## Out of Scope

- Other qdiscs (cross-ref `net/sched/sch-fq-codel.md`, `sch-htb.md`, `sch-cake.md`, `sch-fq.md`, `sch-mqprio.md`, `sch-taprio.md`, `sch-bpf.md`)
- 32-bit-only paths
- Implementation code
