# Tier-3: net/sched/sch_multiq.c — MULTIQ multi-queue dispatcher qdisc

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/sched/sch-api.md
upstream-paths:
  - net/sched/sch_multiq.c (~414 lines)
  - include/uapi/linux/pkt_sched.h (MULTIQ UAPI)
-->

## Summary

MULTIQ is a multi-band qdisc that aligns per-band with the underlying NIC's TX-queues (one band per HW TX queue). Per-skb routed to band `skb_get_queue_mapping(skb)`; per-band has its own inner qdisc (default pfifo); per-dequeue round-robins across active bands. Useful when NIC has multiple HW TX queues (multi-queue NICs) and stack wants per-CPU/per-flow steering. Does NOT do classification beyond `skb.queue_mapping`. Critical for: multi-queue NIC fairness across HW queues; XPS/RFS integration.

This Tier-3 covers `sch_multiq.c` (~414 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct multiq_sched_data` | per-qdisc state | `MultiqSchedData` |
| `multiq_init()` | per-qdisc init | `Multiq::init` |
| `multiq_destroy()` | per-qdisc destroy | `Multiq::destroy` |
| `multiq_enqueue()` | per-skb queue-mapping dispatch | `Multiq::enqueue` |
| `multiq_dequeue()` | per-band round-robin dequeue | `Multiq::dequeue` |
| `multiq_change()` | per-qdisc reconfigure | `Multiq::change` |
| `multiq_classify()` | per-skb band classify | `Multiq::classify` |
| `multiq_tune()` | per-qdisc resize on dev change | `Multiq::tune` |
| `skb_get_queue_mapping()` | per-skb HW queue idx | shared |
| `TCA_MULTIQ_PARMS` | UAPI per-config | UAPI |

## Compatibility contract

REQ-1: Per-tc UAPI tc_multiq_qopt:
- `bands`: u16 number of bands.
- `max_bands`: u16 max (typically dev.real_num_tx_queues).

REQ-2: Per-band inner qdisc:
- Default pfifo with limit = sch.dev.tx_queue_len.
- Per-tc-class user can replace via `tc qdisc add dev eth0 parent root:1 sfq`.

REQ-3: Per-skb classify:
- band = skb_get_queue_mapping(skb).
- If band ≥ q.bands: band = 0 (default).

REQ-4: multiq_enqueue:
- band = Multiq::classify(skb).
- inner = q.queues[band].
- ret = inner.enqueue(skb).
- sch.q.qlen++; sch.qstats.backlog += skb.len.

REQ-5: multiq_dequeue (round-robin):
- For i in 0..q.bands:
  - band = (q.curband + i) % q.bands.
  - skb = q.queues[band].dequeue().
  - If skb: q.curband = (band + 1) % q.bands; return skb.
- Return None.

REQ-6: multiq_change:
- New bands ≤ max_bands ≤ dev.real_num_tx_queues.
- If decreasing: free per-removed inner-qdiscs.
- If increasing: alloc per-new pfifo.

REQ-7: multiq_tune:
- Per-NETDEV_CHANGE event: re-tune bands.
- Re-bind to dev.real_num_tx_queues if changed.

REQ-8: Per-class API:
- multiq_walk: per-band class.
- multiq_graft: replace per-band inner.
- multiq_leaf: return per-band inner.

REQ-9: Per-NIC integration:
- `dev_pick_tx` resolves skb.queue_mapping based on XPS/skb-hash/etc.
- MULTIQ trusts this and dispatches to the corresponding inner qdisc.

REQ-10: Per-NETIF_F_LLTX:
- LLTX devices may bypass; qdisc still functions for shaping.

## Acceptance Criteria

- [ ] AC-1: `tc qdisc add dev eth0 root multiq`: per-NIC TX-queue count bands.
- [ ] AC-2: skb with queue_mapping=2: enqueued to q.queues[2].
- [ ] AC-3: queue_mapping out-of-range: defaults to band 0.
- [ ] AC-4: 2 bands with skbs: dequeue round-robins.
- [ ] AC-5: Per-band sfq replacement: tc qdisc add ... parent 1:3 sfq.
- [ ] AC-6: Dev TX-queue count change: multiq_tune re-bands.
- [ ] AC-7: multiq_change reducing bands: per-removed inner qdiscs freed.
- [ ] AC-8: tc -s class show: per-band stats.
- [ ] AC-9: multiq_destroy: per-band qdiscs freed.
- [ ] AC-10: bands > max_bands: -EINVAL.

## Architecture

Per-qdisc state:

```
struct MultiqSchedData {
  bands: u16,
  max_bands: u16,
  curband: u16,                                  // round-robin pointer
  queues: Vec<&Qdisc>,                           // per-band
}
```

`Multiq::init(sch, opt) -> Result<()>`:
1. Parse qopt.bands; validate ≤ qopt.max_bands ≤ sch.dev.real_num_tx_queues.
2. q.bands = qopt.bands.
3. q.max_bands = qopt.max_bands.
4. For band in 0..q.bands:
   - q.queues[band] = qdisc_create_dflt(sch.dev_queue.netdev_get_tx_queue(band), &pfifo_qdisc_ops, TC_H_MAKE(sch.handle, band+1)).

`Multiq::classify(sch, skb) -> u32`:
1. band = skb_get_queue_mapping(skb).
2. If band >= q.bands: band = 0.
3. Return band.

`Multiq::enqueue(sch, skb, to_free) -> Result<()>`:
1. band = Multiq::classify(sch, skb).
2. inner = q.queues[band].
3. ret = inner.enqueue(skb, inner, to_free)?.
4. sch.qstats.backlog += skb.len.
5. sch.q.qlen++.
6. Ok.

`Multiq::dequeue(sch) -> Option<&Skb>`:
1. for i in 0..q.bands:
   - band = (q.curband + i) % q.bands.
   - skb = q.queues[band].dequeue();
   - if skb is Some:
     - q.curband = (band + 1) % q.bands.
     - sch.qstats.backlog -= qdisc_pkt_len(skb).
     - sch.q.qlen--.
     - Return skb.
2. Return None.

`Multiq::change(sch, opt) -> Result<()>`:
1. Parse new qopt.
2. lock sch.q_lock.
3. If new.bands < q.bands: drop + free queues[new.bands..].
4. If new.bands > q.bands: alloc queues[q.bands..new.bands].
5. q.bands = new.bands.
6. unlock.

`Multiq::tune(sch)`:
1. real_q = sch.dev.real_num_tx_queues.
2. If real_q != q.bands: Multiq::change with new bands = real_q.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `bands_le_max_bands` | INVARIANT | q.bands ≤ q.max_bands. |
| `bands_le_real_num_tx_queues` | INVARIANT | q.bands ≤ sch.dev.real_num_tx_queues. |
| `band_idx_in_range` | INVARIANT | per-classify: band < q.bands. |
| `curband_lt_bands` | INVARIANT | q.curband < q.bands. |

### Layer 2: TLA+

`net/sched/sch_multiq.tla`:
- Per-skb classify + per-band enqueue + round-robin dequeue.
- Properties:
  - `safety_band_consistent` — per-skb classified band same in enqueue and dequeue.
  - `safety_round_robin` — per-pass: each band gets 1 dequeue chance.
  - `liveness_active_band_drains` — per-band non-empty + dequeue ⟹ drained.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Multiq::classify` post: returned band < q.bands | `Multiq::classify` |
| `Multiq::enqueue` post: skb in q.queues[band] | `Multiq::enqueue` |
| `Multiq::dequeue` post: returned-skb from active band; curband advanced | `Multiq::dequeue` |
| `Multiq::change` post: per-bands count == new.bands; per-removed freed | `Multiq::change` |

### Layer 4: Verus/Creusot functional

`Per-skb queue_mapping → per-NIC TX-queue band → per-band qdisc → HW TX-queue` semantic equivalence: per-multiq matches NIC TX-queue alignment for XPS.

## Hardening

(Inherits row-1 features from `net/sched/sch-api.md` § Hardening.)

MULTIQ-specific reinforcement:

- **Per-bands ≤ real_num_tx_queues** — defense against per-bands beyond NIC capability.
- **Per-classify defaults to band 0 on invalid mapping** — defense against per-skb crash on out-of-range queue_mapping.
- **Per-tune handles dev-change** — defense against per-NIC reconfiguration leaving stale bands.
- **Per-graft purges old qdisc** — defense against per-graft UAF.
- **Per-band qdisc lock independent** — defense against per-band-stall blocking other bands.
- **Per-curband advance after dequeue** — defense against per-band starvation (round-robin).
- **Per-decrease bands frees inner qdiscs atomically** — defense against per-decrease leaving stale qdisc.
- **Per-MULTIQ leaf integration with mqprio** — defense against per-traffic-class isolation breach.

## Grsecurity/PaX-style Reinforcement

Beyond the Row-1 defaults above, sch_multiq (multi-queue dispatch qdisc) inherits the following PaX/grsec primitives:

- **PAX_USERCOPY**: `tc_multiq_qopt` attribute parsed via `nla_get_*` whitelisted accessors.
- **PAX_KERNEXEC**: `multiq_qdisc_ops` and `multiq_class_ops` are `__ro_after_init`; W^X enforced on the dispatch path.
- **PAX_RANDKSTACK**: RTM_NEWQDISC + per-packet enqueue/dequeue protected.
- **PAX_REFCOUNT**: `Qdisc.refcnt`, per-band sub-qdisc refcnts, per-band `bstats` / `qstats` use checked `refcount_t` / `u64_stats_*`.
- **PAX_MEMORY_SANITIZE**: freed band array zeroed on `multiq_destroy`.
- **PAX_UDEREF**: rtnetlink walkers fenced.
- **PAX_RAP / kCFI**: `Qdisc_ops` and `Qdisc_class_ops` callbacks (`enqueue`, `dequeue`, `graft`, `leaf`) type-tagged + CFI-checked.
- **GRKERNSEC_HIDESYM**: multiq internal symbols hidden from `/proc/kallsyms`.
- **GRKERNSEC_DMESG**: parse-failure printks rate-limited.

Component-specific reinforcement:

- RTM_NEWQDISC attaching multiq requires **CAP_NET_ADMIN** in the netns owner.
- `bands` clamped to `[1, dev->num_tx_queues]` and to `TCQ_MULTIQ_MAX_QUEUES`; out-of-range or `bands > num_tx_queues` fails `-EINVAL`.
- `max_bands` (the per-attach kmalloc-array size) is bounded to `dev->num_tx_queues`; no unbounded class allocation on attach.
- Per-band sub-qdisc graft (`multiq_graft`) refcounts the *old* child before swap and releases through RCU; class lookup (`multiq_find`) bounds the class id to `[1, q->bands]`.
- Per-band drop / overlimit counters saturate via `u64_stats_*`.

Rationale: multiq is the simplest multi-queue dispatch — it maps `skb->queue_mapping` directly to a sub-qdisc. The hardening surface is a `bands` integer and an array of child qdiscs; an unchecked `bands` is OOM, and a raw-indexed `band[]` lookup is OOB. Bounding the band count plus refcounted graft plus the standard PaX + RAP/kCFI envelope keeps the multi-queue dispatch fail-closed.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- TC qdisc API (covered in `sch-api.md` Tier-3)
- mqprio (covered in `sch-mqprio.md` Tier-3)
- generic (covered in `sch-generic.md` Tier-3)
- Implementation code
