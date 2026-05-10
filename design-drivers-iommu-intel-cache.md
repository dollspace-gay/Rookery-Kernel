---
title: "Tier-3: drivers/iommu/intel/cache.c — Intel VT-d invalidation cache mgmt API (IOTLB + DEV-IOTLB + PIOTLB + PASID-cache + IEC)"
tags: ["tier-3", "iommu", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Intel VT-d hardware maintains five distinct caches that must be invalidated when host changes per-domain or per-PASID translation tables: (1) IOTLB (per-IOMMU IOVA→PA cache), (2) DEV-IOTLB (per-endpoint ATS cache), (3) PIOTLB (per-PASID IOTLB for SVA), (4) PASID-cache (per-IOMMU pasid-table-entry cache), (5) IEC (interrupt-entry-cache). cache.c (~534 lines) provides the per-domain abstraction layer above per-IOMMU QI submission: `cache_tag_assign_domain` registers a per-domain "tag" that tracks which IOMMUs+devices need flush; on per-domain page-table-update, `cache_tag_flush_range` walks tags and submits batched QI commands.

This Tier-3 covers `drivers/iommu/intel/cache.c` (~534 lines).

### Acceptance Criteria

- [ ] AC-1: Per-attach tag-add: VFIO-passthrough device attach → cache_tags has [IOTLB, DEVTLB] entries; verify via debugfs.
- [ ] AC-2: Per-detach tag-remove: detach drops matching tags; flush after detach skips removed entries.
- [ ] AC-3: Range flush correctness: 4MB IOMMU-mapped range with TLB-warmed; unmap + flush; subsequent device-DMA gets fault (no stale TLB hit).
- [ ] AC-4: IH-flush optimization: RW→RO permission-tighten triggers IH=1 flush; cycle-counter shows shorter flush duration vs full-range.
- [ ] AC-5: Non-present cache flush: ioasid map of previously-unmapped range, immediate device-DMA succeeds (no stale negative cache).
- [ ] AC-6: SVA + PIOTLB: VTOL-DSA + SVA-bind per-PASID; per-PASID page-table-update flushes PIOTLB; subsequent device read sees update.
- [ ] AC-7: Multi-IOMMU domain: domain spanning 2 IOMMUs → flush submits to both via batched QI.
- [ ] AC-8: kvm-unit-test / vfio-test cache-coherence suite passes.

### Architecture

`CacheTag` per-(iommu, dev, pasid):

```
struct CacheTag {
  type_: CacheTagType,                       // IOTLB / DEVTLB / PIOTLB / NESTING_IOTLB / NESTING_DEVTLB
  domain_id: u16,                            // DID assigned per-IOMMU
  pasid: u32,                                // 20-bit; or RID_PASID for non-PASID devices
  iommu: KArc<Iommu>,
  dev: Option<KWeak<PciDev>>,                // for DEVTLB tags
  link: ListNode,                            // per-domain cache_tags list
}

enum CacheTagType {
  Iotlb,
  Devtlb,
  Piotlb,
  NestingIotlb,
  NestingDevtlb,
}
```

`Intel::cache_tag_assign_domain(domain, dev, pasid)` flow:
1. Acquire `domain.cache_tag_lock` (writer).
2. Compute per-(IOMMU, type) needed:
   - Always: IOTLB tag for IOMMU.
   - If endpoint has ATS: DEVTLB tag for (IOMMU, dev).
   - If pasid != RID_PASID + endpoint has PRI: PIOTLB tag for (IOMMU, dev, pasid).
   - If domain is nested: NESTING_IOTLB / NESTING_DEVTLB analogous.
3. Per-tag-needed: check if existing tag; if not, allocate + push to domain.cache_tags.
4. Release lock.

`Intel::cache_tag_flush_range(domain, start, end, ih)` flow:
1. Acquire `cache_tag_lock` (reader).
2. Per-IOMMU batch: group tags by iommu; per-IOMMU build qi_desc-list.
3. Per-tag in domain.cache_tags:
   - Compute flush command per-type:
     - IOTLB: page-selective with ih=ih, addr=start, mask=log2(end-start) clamped to MAMV.
     - DEVTLB: per-(sid, addr, size).
     - PIOTLB: per-(did, pasid, addr, mask).
   - Push to per-IOMMU batch.
4. Per-IOMMU: `qi_submit_sync(iommu, batch, count)`; wait for IWC.
5. Release reader-lock.

`Intel::cache_tag_flush_all(domain)`:
1. Acquire reader-lock.
2. Per-tag: emit global flush per-type:
   - IOTLB: per-DID flush.
   - DEVTLB: per-(sid) full-flush.
   - PIOTLB: per-(did, pasid) full-flush.
3. Per-IOMMU `qi_submit_sync`; release.

`Intel::iotlb_sync` (iommu-API callback):
- Called by generic iommu code after batched unmap operations.
- Walks per-domain queue of pending range-invalidations.
- Calls `cache_tag_flush_range` per-range.

### Out of Scope

- Generic IOMMU API (covered in `iommu-core.md` Tier-3)
- Intel IOMMU driver core (covered in `intel-iommu.md` Tier-3)
- Intel nested-translation backend (covered in `intel-nested.md` Tier-3)
- Intel PASID mgmt (covered in `intel-pasid.md` Tier-3)
- Intel IRQ remap (covered in `intel-irq-remap.md` Tier-3)
- Intel PRQ (covered in `intel-prq.md` Tier-3)
- AMD/ARM cache-mgmt (different drivers)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct cache_tag` | per-(domain, iommu, dev, pasid) tag | `drivers::iommu::intel::CacheTag` |
| `cache_tag_assign_domain(domain, dev, pasid)` | per-attach: register tag(s) | `Intel::cache_tag_assign_domain` |
| `cache_tag_unassign_domain(domain, dev, pasid)` | per-detach: deregister tag(s) | `Intel::cache_tag_unassign_domain` |
| `cache_tag_flush_range(domain, start, end, ih)` | range-invalidate per-domain | `Intel::cache_tag_flush_range` |
| `cache_tag_flush_all(domain)` | full-flush per-domain | `Intel::cache_tag_flush_all` |
| `cache_tag_flush_range_np(domain, start, end)` | non-present (write-back-only) flush | `Intel::cache_tag_flush_range_np` |
| `qi_flush_iotlb(iommu, did, addr, mask, type)` | submit IOTLB flush via QI | `Iommu::qi_flush_iotlb` |
| `qi_flush_dev_iotlb(iommu, sid, ...)` | submit DEV-IOTLB flush | `Iommu::qi_flush_dev_iotlb` |
| `qi_flush_piotlb(iommu, did, pasid, addr, mask)` | submit PIOTLB flush | `Iommu::qi_flush_piotlb` |
| `qi_flush_pasid_cache(iommu, did, granu, pasid)` | submit PASID-cache flush | `Iommu::qi_flush_pasid_cache` |
| `qi_flush_iec(iommu, index, mask)` | submit IEC flush (covered in irq-remap Tier-3) | shared |
| `cache_tag_type` enum | tag-kind {IOTLB, DEVTLB, NESTING_IOTLB, NESTING_DEVTLB, PASIDTLB} | `CacheTagType` |
| `cache_tag_lock` | per-domain spinlock for tag-list mutation | `IntelDomain::cache_tag_lock` |
| `intel_iommu_iotlb_sync(...)` | iommu-API hook callback for batched flush | `Intel::iotlb_sync` |
| `intel_iommu_iotlb_sync_map(...)` | post-map flush (for non-present-entry caching) | `Intel::iotlb_sync_map` |

### compatibility contract

REQ-1: Per-domain cache_tag list:
- `IntelDomain.cache_tags: KVec<CacheTag>` (each tag = (iommu, did, dev, pasid, type)).
- On per-device attach: append tags for IOTLB + DEV-IOTLB (if endpoint has ATS) + PIOTLB (if PASID).
- On per-device detach: remove matching tags.

REQ-2: Per-tag flush dispatch in `cache_tag_flush_range`:
- For each tag in domain.cache_tags:
  - If tag.type == IOTLB: `qi_flush_iotlb(iommu, did, addr, mask, page-selective)`.
  - If tag.type == DEVTLB: `qi_flush_dev_iotlb(iommu, sid, addr, mask)`.
  - If tag.type == PIOTLB: `qi_flush_piotlb(iommu, did, pasid, addr, mask)`.
  - If tag.type == NESTING_IOTLB / NESTING_DEVTLB: nested-specific flush (covered via intel-nested.c).
- Batched into per-IOMMU QI desc-array; single `qi_submit_sync` per IOMMU.

REQ-3: Per-tag flush granularity:
- `qi_flush_iotlb` granularity ∈ {global, per-DID, per-DID-PASID, page-selective-PSI}; cache.c uses page-selective by default + falls back to per-DID if range > MAMV.
- `qi_flush_dev_iotlb` always per-(sid, addr, size).
- `qi_flush_piotlb` always per-(did, pasid, addr, size).

REQ-4: IH (Invalidation Hint) bit:
- For range-flush after page-permission-tightening (e.g., RW→RO), IH=1 hints IOMMU to flush only PA-update not the entire range.
- `cache_tag_flush_range(domain, start, end, ih)` propagates IH to per-tag flush.

REQ-5: Non-present entry caching (NWFS bit):
- IOMMU may cache "page not present" results (negative cache).
- After map (non-present → present): must flush via `cache_tag_flush_range_np` to invalidate negative cache.
- Otherwise: device-DMA may continue to see "page not mapped" until cache evicts.

REQ-6: Per-domain DID assignment:
- DID (Domain-ID) 16-bit; per-IOMMU unique; assigned by `domain_alloc`.
- One domain may have multiple DIDs (one per IOMMU it spans).

REQ-7: Lock discipline:
- `cache_tag_lock` (per-domain spinlock) serializes tag-list mutation.
- Flush operations may walk under reader-lock; mutation under writer-lock.

REQ-8: ATS+PRI integration:
- DEV-IOTLB only registered if endpoint has ATS.
- PIOTLB only registered if endpoint has PRI + SVA bind.
- Per-attach examines per-device cap (via PCIe ATS + PRI cap structs).

REQ-9: SR-IOV VF tagging:
- VF inherits PF's IOMMU + DID; per-VF separate DEV-IOTLB tag (different SID).

REQ-10: Hot-add/hot-remove:
- Per-PCI hotplug: `cache_tag_assign_domain` adds tags; `cache_tag_unassign_domain` removes.
- Removed tags do not contribute to subsequent flushes.

REQ-11: Failure modes:
- QI submission failure (timeout): driver logs + bumps per-IOMMU error counter.
- IOMMU-fatal-error during QI: per-IOMMU disable; per-domain mark broken.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cache_tag_list_no_uaf` | UAF | per-tag.dev as KWeak; upgrade-on-use; defense against PCI-bus-del while flushing. |
| `qi_batch_no_oob` | OOB | per-IOMMU qi_desc batch ≤ ring-size; defense against batch-overflow. |
| `cache_tag_lock_serialized` | INVARIANT | tag-list mutation always under writer-lock; flush always under reader-lock. |
| `did_per_iommu_unique` | INVARIANT | per-IOMMU at most one tag per (domain, dev, pasid, type). |

### Layer 2: TLA+

`drivers/iommu/intel/cache_tag_lifecycle.tla`:
- per-tag state ∈ {Unassigned, Assigned, Flushing, Removed}.
- Properties:
  - `safety_no_flushing_after_removed` — once Removed, no further Flushing.
  - `safety_assign_implies_iommu_active` — Assigned implies tag.iommu is currently-active IOMMU.
  - `liveness_pending_flush_completes` — every queued flush eventually completes via QI.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Intel::cache_tag_assign_domain` post: tag pushed iff not already present; lock held throughout | `Intel::cache_tag_assign_domain` |
| `Intel::cache_tag_unassign_domain` post: matching tag removed; subsequent flush skips | `Intel::cache_tag_unassign_domain` |
| `Intel::cache_tag_flush_range` post: every tag in list submitted exactly once to QI batch | `Intel::cache_tag_flush_range` |
| Per-IOMMU batch after flush: IWC observed (qi_submit_sync waited) | `Intel::cache_tag_flush_range` |
| Per-domain.cache_tags free-list invariant: no duplicate tags | `Intel::cache_tag_assign_domain` |

### Layer 4: Verus/Creusot functional

`Per-domain page-table-update + cache_tag_flush_range → subsequent device DMA observes update` round-trip equivalence: post-flush device-IOTLB does not contain stale entry for [start..end].

### hardening

(Inherits row-1 features from `drivers/iommu/intel-iommu.md` § Hardening.)

cache-tag-specific reinforcement:

- **Per-tag.dev as KWeak** — defense against PCI-bus-del-during-flush UAF.
- **cache_tag_lock RWLock** — defense against tag-list mutation-during-flush iterator invalidation.
- **Per-IOMMU batch ≤ ring-size** — defense against batch overflow.
- **QI submit-sync timeout-bounded** — defense against stuck QI on faulty IOMMU.
- **Per-tag dedup at assign** — defense against duplicate tags causing duplicate QI commands.
- **Per-flush IH bit conservatively zero** when uncertain — defense against IH-incorrect-use causing missed flush.
- **Non-present cache flush after every map** — defense against stale negative-cache.
- **Per-detach tag-remove before pasid-entry-zero** — defense against stale tag flushing zeroed pasid causing IOMMU error.
- **Per-IOMMU error-counter on QI failure** — defense against silent flush-loss.
- **NESTING_IOTLB tags only for nested domains** — defense against tag-type / domain-type mismatch.

