# Tier-3: net/sched/sch_prio.c — PRIO multi-band priority qdisc

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/sched/sch-api.md
upstream-paths:
  - net/sched/sch_prio.c (~437 lines)
  - include/uapi/linux/pkt_sched.h (PRIO UAPI)
-->

## Summary

`PRIO` is a strict-priority N-band qdisc (default 3 bands; max 16). Per-skb mapped to band via `priomap[skb.priority]` (8-bit priority → band-idx). Per-dequeue: walk bands [0..N-1]; first non-empty band wins. Per-band has its own inner qdisc (default pfifo). Strict priority: band 0 dominates band 1+. Per-tc-class: each band exposed as class N:1, N:2, ..., N:N for hierarchical filtering. Critical for: simple QoS prioritization (e.g. control-plane traffic > bulk).

This Tier-3 covers `sch_prio.c` (~437 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct prio_sched_data` | per-qdisc state | `PrioSchedData` |
| `prio_init()` | per-qdisc init | `Prio::init` |
| `prio_destroy()` | per-qdisc destroy | `Prio::destroy` |
| `prio_enqueue()` | per-pkt enqueue (priomap dispatch) | `Prio::enqueue` |
| `prio_dequeue()` | per-pkt dequeue (strict-priority) | `Prio::dequeue` |
| `prio_peek()` | per-pkt peek | `Prio::peek` |
| `prio_change()` | per-qdisc reconfigure | `Prio::change` |
| `prio_classify()` | per-skb band classify | `Prio::classify` |
| `tc_prio_qopt` | UAPI per-qdisc opt | UAPI |
| `TC_PRIO_*` (8 priorities) | per-skb prio | UAPI |
| `TCQ_PRIO_BANDS` (3 default) | per-qdisc band count | shared |
| `priomap` | u8[16] map skb.prio → band | UAPI |

## Compatibility contract

REQ-1: Per-qdisc tc_prio_qopt:
- `bands`: 1..16 number of bands.
- `priomap[16]`: per-skb-priority → band-index.

REQ-2: Default priomap:
- TC_PRIO_BESTEFFORT (0) → band 1.
- TC_PRIO_FILLER (1) → band 2.
- TC_PRIO_BULK (2) → band 2.
- TC_PRIO_INTERACTIVE_BULK (4) → band 1.
- TC_PRIO_INTERACTIVE (6) → band 0.
- TC_PRIO_CONTROL (7) → band 0.

REQ-3: prio_classify(sch, skb) -> u32 (band):
- skb_prio = skb.priority & TC_PRIO_MAX.
- Per-tc-cls programs: invoke filters; if class returned & TC_H_MIN(class) ≤ bands: use class - 1 as band.
- Else: priomap[skb.priority & 0xF].

REQ-4: prio_enqueue:
- band = prio_classify(sch, skb).
- inner_qdisc = q.queues[band].
- ret = inner_qdisc.enqueue(skb).
- If !ret error: sch.qstats.backlog += skb.len; q.qstats.qlen++.

REQ-5: prio_dequeue (strict-priority):
- For band in 0..q.bands:
  - skb = q.queues[band].dequeue();
  - If skb: sch.qstats.backlog -= skb.len; q.qstats.qlen--; return skb.
- Return None.

REQ-6: prio_peek (strict-priority):
- For band in 0..q.bands:
  - skb = q.queues[band].peek();
  - If skb: return skb.
- Return None.

REQ-7: Per-band inner qdisc:
- Default: pfifo with limit = sch.dev.tx_queue_len.
- Per-tc-class user can replace via `tc qdisc add dev eth0 parent 1:1 sfq`.

REQ-8: prio_init:
- Allocate per-band q.queues[band] = qdisc_create_dflt(sch.dev_queue, &pfifo_qdisc_ops, ...).
- Set q.bands = qopt.bands.
- Copy priomap.

REQ-9: prio_change:
- New qopt.bands ≤ 16.
- If decreasing bands: kfree per-removed inner qdiscs.
- If increasing: alloc per-new pfifo.
- Update priomap.

REQ-10: prio_class_op:
- per-tc-class: each band has class id (parent-major:band+1).
- prio_walk: iterate per-band class.
- prio_graft: replace per-band inner qdisc.
- prio_leaf: return per-band inner qdisc.

## Acceptance Criteria

- [ ] AC-1: `tc qdisc add dev eth0 root prio bands 4 priomap 0 1 2 3 0 1 2 3 0 1 2 3 0 1 2 3`: PRIO allocated.
- [ ] AC-2: skb with prio=6 (TC_PRIO_INTERACTIVE): classified to band 0.
- [ ] AC-3: Multiple skbs in band 0 + band 1: dequeue drains band 0 first.
- [ ] AC-4: Empty band 0 + queued band 1: dequeue band 1.
- [ ] AC-5: Per-band inner qdisc replaceable: `tc qdisc add dev eth0 parent 1:1 sfq`.
- [ ] AC-6: prio_change reducing bands: per-removed inner qdiscs freed.
- [ ] AC-7: Per-class stats: `tc -s class show dev eth0` shows per-band pkts/bytes.
- [ ] AC-8: prio_destroy on dev-down: per-band qdiscs freed.
- [ ] AC-9: Concurrent enqueue across bands: lockless via inner-qdisc lock.
- [ ] AC-10: Per-band classifier: `tc filter add dev eth0 ... action skbedit priority 7`: skb.prio overrides.

## Architecture

Per-qdisc state:

```
struct PrioSchedData {
  bands: u32,
  prio2band: [u8; TC_PRIO_MAX + 1],              // priomap
  queues: [&Qdisc; TCQ_PRIO_BANDS_MAX],          // 16 bands max
}

const TCQ_PRIO_BANDS_MAX: usize = 16;
const TC_PRIO_MAX: usize = 15;
```

`Prio::init(sch, opt) -> Result<()>`:
1. Parse qopt.bands; validate ∈ [2..16].
2. q = sch.priv as *mut PrioSchedData.
3. For band in 0..q.bands:
   - q.queues[band] = qdisc_create_dflt(sch.dev_queue, &pfifo_qdisc_ops, TC_H_MAKE(sch.handle, band+1)).
4. memcpy(&q.prio2band, &qopt.priomap, sizeof(priomap)).

`Prio::classify(sch, skb) -> u32`:
1. tcf = rcu_dereference(sch.tcf_chain).
2. result = tcf_classify(skb, tcf).
3. If result == TC_ACT_OK ∨ TC_ACT_RECLASSIFY:
   - band = TC_H_MIN(result.classid) - 1.
   - If band < q.bands: return band.
4. prio = skb.priority & TC_PRIO_MAX.
5. Return q.prio2band[prio].

`Prio::enqueue(sch, skb, to_free) -> Result<()>`:
1. band = Prio::classify(sch, skb).
2. inner = q.queues[band].
3. ret = inner.enqueue(skb, inner, to_free)?.
4. sch.qstats.backlog += qdisc_pkt_len(skb).
5. q.qstats.qlen++.
6. Ok(NetXmitSuccess).

`Prio::dequeue(sch) -> Option<&Skb>`:
1. For band in 0..q.bands:
   - skb = q.queues[band].dequeue();
   - If skb is Some:
     - sch.qstats.backlog -= qdisc_pkt_len(skb).
     - q.qstats.qlen--.
     - Return skb.
2. Return None.

`Prio::change(sch, opt) -> Result<()>`:
1. Parse new qopt; validate.
2. lock sch.q_lock.
3. If new.bands < q.bands: drop + free queues[new.bands..].
4. If new.bands > q.bands: alloc queues[q.bands..new.bands].
5. q.bands = new.bands; q.prio2band = new.priomap.
6. unlock.

`Prio::graft(sch, arg, new_qdisc) -> Result<&Qdisc>`:
1. band = arg - 1.
2. Validate band < q.bands.
3. old = q.queues[band].
4. q.queues[band] = new_qdisc.
5. qdisc_purge(old).
6. Return old.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `bands_in_range` | INVARIANT | q.bands ∈ [2, 16]. |
| `prio2band_lt_bands` | INVARIANT | per-prio: q.prio2band[p] < q.bands. |
| `band_idx_in_dequeue` | INVARIANT | per-iter: band < q.bands. |
| `enqueue_then_dequeue_band_match` | INVARIANT | per-skb enqueue band == per-skb dequeue band. |
| `strict_priority_band0_first` | INVARIANT | per-dequeue: band[i] empty for all i < returned-band. |

### Layer 2: TLA+

`net/sched/sch_prio.tla`:
- Per-skb classify + enqueue + per-band dequeue.
- Properties:
  - `safety_strict_priority` — per-non-empty-band[0] never starves higher-numbered bands.
  - `safety_band_assignment_persistent` — per-skb band stays consistent.
  - `liveness_band_drains` — per-band non-empty + dequeue calls ⟹ drained.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Prio::classify` post: returned band < q.bands | `Prio::classify` |
| `Prio::enqueue` post: skb in q.queues[band].queue | `Prio::enqueue` |
| `Prio::dequeue` post: returned-skb from highest-priority non-empty band | `Prio::dequeue` |
| `Prio::change` post: new bands count == new.bands; per-removed freed | `Prio::change` |

### Layer 4: Verus/Creusot functional

`Per-skb prio mapped to band ∈ [0..bands) → strict-priority dequeue serves bands in order` semantic equivalence: per-PRIO matches strict-priority QoS class semantics.

## Hardening

(Inherits row-1 features from `net/sched/sch-api.md` § Hardening.)

PRIO-specific reinforcement:

- **Per-bands ∈ [2,16] enforced** — defense against per-config invalid count.
- **Per-priomap[i] < bands enforced** — defense against per-skb mapping to non-existent band.
- **Per-band inner qdisc default pfifo** — defense against per-config missing inner-qdisc.
- **Per-classify TC filters use RCU** — defense against per-filter-update during classify.
- **Per-graft purges old qdisc** — defense against per-graft UAF.
- **Per-decrease bands frees inner qdiscs atomically** — defense against per-decrease leaving stale qdisc with queued skbs.
- **Per-band qdisc lock independent** — defense against per-band-stall blocking other bands.
- **Per-skb prio masked to [0..15]** — defense against per-skb prio overflow indexing priomap.
- **Per-qstats backlog updated atomically** — defense against per-stats race.
- **Per-class walk validates band index** — defense against per-walk crashing on stale state.

## Grsecurity/PaX-style Reinforcement

Baseline PaX/grsecurity mitigations applicable to the PRIO qdisc:

- **PAX_USERCOPY** — `TCA_PRIO_*` netlink payload moves use whitelisted `copy_{to,from}_user`; the 16-entry priomap blob is size-checked under USERCOPY.
- **PAX_KERNEXEC** — `prio_enqueue` / `prio_dequeue` execute from immutable .text; no patchable trampolines on the per-band dispatch.
- **PAX_RANDKSTACK** — per-syscall stack randomization frustrates priomap-index inference via timing.
- **PAX_REFCOUNT** — outer PRIO `Qdisc` and each per-band inner `Qdisc` use saturating refcounts; rapid `tc qdisc change ... bands N` storms cannot wrap.
- **PAX_MEMORY_SANITIZE** — `prio_sched_data`, `prio2band[16]`, and freed per-band inner qdiscs sanitized on detach.
- **PAX_UDEREF** — `prio_tune` traverses `TCA_PRIO_MQ` user pointers only under UDEREF.
- **PAX_RAP / kCFI** — `prio_class_ops` and `prio_qdisc_ops` are `static const`; the per-band-graft callback is CFI-checked.
- **GRKERNSEC_HIDESYM** — `prio_classify` and per-band lookup symbols hidden from unprivileged kallsyms readers.
- **GRKERNSEC_DMESG** — PRIO band-graft/leaf-walk warnings rate-limited and CAP_SYSLOG-gated.

PRIO-specific reinforcement:

- **TC qdisc CAP_NET_ADMIN** — PRIO create/change/del requires CAP_NET_ADMIN over the netdev's netns.
- **`prio_qdisc_ops` PAX_RAP-typed** — vtable dispatch type-checked on every enqueue/dequeue/peek/dump.
- **Priomap bounded write under PAX_USERCOPY** — `priomap[i]` ≤ bands enforced before any classify can use it.
- **Per-band inner qdisc refcount saturating** — `qdisc_put` on band decrease cannot underflow.
- **GRKERNSEC_HIDESYM on `prio_classify`** — fast-path classifier not exposed for offset inference attacks.

Rationale: PRIO is a strict-priority dispatcher whose security envelope is the priomap and the per-band Qdisc pointer array; sanitizing freed inner qdiscs, saturating refcounts on rapid graft churn, and CFI-protecting the band-dispatch table closes the residual UAF / type-confusion surface.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- TC qdisc API (covered in `sch-api.md` Tier-3)
- Per-band inner qdisc impl (pfifo, sfq, etc.; covered separately)
- TC filters (covered in `cls-api.md` Tier-3)
- Implementation code
