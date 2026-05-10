---
title: "Tier-3: drivers/iommu/intel/nested.c — Intel VT-d nested-translation backend (vIOMMU 2-stage page-table)"
tags: ["tier-3", "iommu", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Intel VT-d nested translation provides 2-stage IOMMU page-tables: stage-1 (S1) is guest-managed (per-guest-PASID) and stage-2 (S2) is host-managed; an in-flight DMA from a passthrough device gets walked through S1 (guest-IOVA → guest-PA) then S2 (guest-PA → host-PA) by the IOMMU hardware. nested.c is the Intel vendor backend for the iommufd vIOMMU framework — it builds nested-domain context entries with both S1 + S2 page-table pointers, handles per-S1 invalidation requests batched from guest, and synthesizes per-vIOMMU faults (translation faults, S1 PRI faults) for guest delivery via iommufd-eventq.

This Tier-3 covers `drivers/iommu/intel/nested.c` (~241 lines, but each line is dense vendor logic).

### Acceptance Criteria

- [ ] AC-1: Boot host with VT-d ECAP.NEST=1 (Sapphire Rapids+) + cap.SLADS=1: `dmesg | grep "VT-d nested"` shows nested support.
- [ ] AC-2: vIOMMU+SR-IOV test: VFIO-passthrough VF + iommufd vIOMMU; guest enables vIOMMU, allocates guest-IOVA via S1; DMA succeeds with 2-stage walk.
- [ ] AC-3: S1 cache-invalidate batch: guest issues 16 IOTLB-invalidates; host batches into single QI submit; counter shows reduction.
- [ ] AC-4: Nested + PASID: guest with vSVA + nested vIOMMU; guest-PASID handled with both S1+S2 walk.
- [ ] AC-5: Fault injection: rogue guest-IOVA → S1-walk fault → iommufd-eventq → guest vIOMMU fault register populated.
- [ ] AC-6: Detach lifecycle: guest detaches device; nested domain ref dropped; host PASID-table entry zeroed + IOTLB flushed.
- [ ] AC-7: Spec-violation defense: malformed S1-PGD pointer (not page-aligned) rejected at HWPT-alloc with -EINVAL.
- [ ] AC-8: Live migration: nested domain checkpoint + restore reproduces same S1+S2 effective translation.

### Architecture

`Intel::NestedOps` is a const vtable matching `iommu_domain_ops`:

```
const NESTED_OPS: IommuDomainOps = IommuDomainOps {
  attach_dev: Intel::nested_attach_dev,
  set_dev_pasid: Intel::nested_set_dev_pasid,
  cache_invalidate_user: Intel::nested_cache_invalidate_user,
  free: Intel::nested_domain_free,
};

struct IntelNestedDomain {
  base: IntelDomain,                        // type=NESTED
  parent: KArc<IntelDomain>,                // S2 parent
  s1_pgd: u64,                              // guest-supplied S1 PGD phys-addr (in guest-PA, but we treat as host-PA after vIOMMU translation)
  flags: u32,                               // SLADE, WPE, EAFE, etc.
  aw: u8,                                   // address-width (39/48/57)
}
```

`Intel::nested_domain_alloc(parent, user_data)`:
1. Validate `parent.type == DOMAIN_DOMAIN_NESTED_PARENT` (S2 hwpt).
2. Validate user_data is `IOMMU_HWPT_DATA_VTD_S1`; cast to `iommu_hwpt_vtd_s1`.
3. Validate s1_pgd page-aligned + within guest-PA range.
4. Allocate `IntelNestedDomain`; populate parent + s1_pgd + flags + aw.
5. `domain.ops = &NESTED_OPS`.
6. Return as IommuDomain.

`Intel::nested_set_dev_pasid(domain, dev, pasid)`:
1. Cast domain to `IntelNestedDomain`.
2. Look up per-device `intel_iommu`, `pasid_table`.
3. Compose pasid-table entry:
   - PGTT (Page-Table-Translation-Type) = NESTED.
   - SLPTPTR (S2 page-table) = parent.s2_pgd.
   - FLPTPTR (S1 page-table) = domain.s1_pgd.
   - AW = domain.aw.
   - PWT/PCD/etc. from domain.flags.
4. Write pasid-table entry atomically (with present-bit-clear-then-set if needed).
5. QI: PASID-cache invalidate (type=pasid_cache, granularity=per_device_per_pasid).
6. QI: IOTLB invalidate (type=iotlb, granularity=per_pasid).
7. Wait for QI completion.

`Intel::nested_cache_invalidate_user(domain, info)`:
1. Cast info to `iommu_hwpt_vtd_s1_invalidate`.
2. For each request in info.requests:
   - Decode type (IOTLB / DEV_IOTLB / PIOTLB).
   - Decode granularity (per-pasid / per-page-range).
   - Compose QI desc; push to per-iommu QI batch.
3. `qi_submit_sync(iommu, &batch, count)`; wait for completion.
4. Update info.req_num to indicate consumed requests.

`Intel::nested_attach_dev(domain, dev)`:
1. Validate device + IOMMU support nested.
2. For RID-PASID-only (non-PASID device): `nested_set_dev_pasid(domain, dev, RID_PASID)`.

### Out of Scope

- Generic IOMMU API (covered in `iommu-core.md` Tier-3)
- Intel IOMMU driver core (covered in `intel-iommu.md` Tier-3)
- iommufd vIOMMU framework (covered in `iommufd-viommu.md` Tier-3)
- iommufd HWPT object (covered in `iommufd-hwpt.md` Tier-3)
- iommufd eventq fault delivery (covered in `iommufd-eventq.md` Tier-3)
- Intel SVM (covered in `intel-svm.md` Tier-3)
- AMD/ARM nested translation (different vendors)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `intel_iommu_nested_ops` | iommu-domain-ops vtable for nested S1 | `drivers::iommu::intel::NestedOps` |
| `intel_nested_attach_dev(domain, dev)` | per-device attach via S1+S2 nested-domain | `Intel::nested_attach_dev` |
| `intel_nested_set_dev_pasid(domain, dev, pasid)` | per-PASID nested attach | `Intel::nested_set_dev_pasid` |
| `intel_nested_cache_invalidate_user(domain, info)` | per-S1 invalidation batch | `Intel::nested_cache_invalidate_user` |
| `intel_nested_domain_alloc(parent, user_data)` | alloc S1-nested domain | `Intel::nested_domain_alloc` |
| `domain_setup_first_level(...)` | program S1 page-table-base + PGTT in pasid-entry | `IntelDomain::setup_s1` |
| `domain_setup_second_level(...)` | program S2 page-table-base in pasid-entry | `IntelDomain::setup_s2` |
| `qi_flush_dev_iotlb_pasid(...)` (qi.c) | DEV-IOTLB invalidate per-PASID | `Iommu::qi_flush_dev_iotlb_pasid` |
| `qi_flush_piotlb(...)` (qi.c) | PIOTLB invalidate per-PASID-per-PageRange | `Iommu::qi_flush_piotlb` |
| `vtd_pasid_dir_entry` / `pasid_table_entry` | PASID-table layout (intel-pasid Tier-3) | shared |
| `iommu_hwpt_vtd_s1` (UAPI struct) | per-S1 user-data: flags + S1-page-table-base + addr-width | shared with iommufd |

### compatibility contract

REQ-1: Nested domain alloc:
- iommufd userspace requests HWPT-alloc with parent=S2-domain + user_data containing IOMMU_HWPT_DATA_VTD_S1 (S1-PGD + flags + AW).
- `intel_nested_domain_alloc(parent, &iommu_hwpt_vtd_s1)` allocates Intel `dmar_domain` with type=DOMAIN_DOMAIN_NESTED + S1-PGD pointer + parent-S2 pointer.

REQ-2: S1-nested domain attach (per-PASID):
- `intel_nested_set_dev_pasid(domain, dev, pasid)`:
  - Look up per-device pasid-table.
  - Write pasid-table entry: PGTT=NESTED, S1-PGD=domain.s1_pgd, S2-PGD=parent.s2_pgd, AW=domain.aw.
  - Issue PASID-cache invalidate via QI.
  - Issue IOTLB invalidate for old PASID range.

REQ-3: Per-S1 invalidation batch from guest:
- iommufd routes guest-issued IOTLB invalidate requests via `intel_nested_cache_invalidate_user(domain, info)`.
- `info.requests[]` array; per-request:
  - DEV_IOTLB: `qi_flush_dev_iotlb_pasid` for guest-pasid + range.
  - PIOTLB: `qi_flush_piotlb` for guest-pasid + range.
  - IOTLB: `qi_flush_iotlb_psi` for guest-pasid + addr + size.
- Batched submission to QI for efficiency.

REQ-4: Per-domain S1 attributes:
- AW (Address Width): 39-bit / 48-bit / 57-bit (4-level / 5-level paging).
- SLADE (Second-Level Access/Dirty Enable): if guest enables; passes through to S2.
- WPE (Write Protect Enable): per-S1 enforcement.
- EAFE (Execute-Above-Forbidden-Enable): per-S1 NX-enforcement.

REQ-5: Per-device support gating:
- Nested attach only valid if device + IOMMU support PASID + nested-translation (ECAP.NEST + ECAP.PSS).
- If not supported: `intel_nested_attach_dev` returns -EOPNOTSUPP.

REQ-6: S1 fault routing:
- Hardware fault during S1-walk → IOMMU writes fault descriptor to fault-recording register (FRR).
- Driver: `intel_iommu_handle_iotlb_fault(...)` reads FRR.
- If fault belongs to nested domain: dispatch to `iommu_report_device_fault(...)` with domain.iommufd-eventq.
- Userspace consumes via iommufd-eventq + injects into guest vIOMMU.

REQ-7: PRI fault routing for nested:
- PRQ entry with nested-PASID → `intel_iommu_drain_pasid_prq` cleanup on detach.
- Live PRI fault → iommufd-eventq dispatch (covered in iommufd-eventq Tier-3).

REQ-8: Cache invalidation correctness:
- After every guest-S1-PTE update, guest must issue IOTLB-invalidate via vIOMMU register-write.
- vIOMMU emulation forwards to host via `cache_invalidate_user` ioctl.
- Host calls into nested.c which submits to QI.

REQ-9: Per-S1 nested HWPT lifecycle tied to parent S2 HWPT:
- S1 cannot outlive parent S2; ref-count enforced.
- Detach-S2-while-S1-attached: return -EBUSY.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `s1_pgd_aligned` | INVARIANT | per-IntelNestedDomain.s1_pgd 4KiB-aligned; defense against IOMMU walk-OOB on misaligned PGD. |
| `parent_refcount_held` | UAF | per-IntelNestedDomain holds KArc<parent>; parent freed only after all nested children freed. |
| `pasid_entry_atomic` | INVARIANT | pasid-table-entry write atomic via cmpxchg or present-clear-then-set; defense against IOMMU reading torn entry. |
| `qi_batch_no_oob` | OOB | per-batch desc count bounded by per-iommu QI ring size. |

### Layer 2: TLA+

`drivers/iommu/intel/nested_invalidate.tla` (already exists from earlier Tier-3) models per-domain S1 invalidation sequencing.

`drivers/iommu/intel/nested_attach.tla`: per-PASID nested-attach state ∈ {Detached, Attaching, Attached, Detaching, Faulted}; transitions per pasid-entry write + QI flush + fault.

Properties:
- `safety_no_attached_with_zeroed_pasid_entry` — Attached implies pasid-entry has nonzero PGD pointers.
- `safety_detach_completes_invalidation` — Detaching → Detached requires QI invalidate-complete.
- `liveness_attaching_terminates` — every Attaching eventually Attached or Faulted.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Intel::nested_domain_alloc` post: domain.parent populated; s1_pgd validated; ops vtable bound | `Intel::nested_domain_alloc` |
| `Intel::nested_set_dev_pasid` post: pasid-table-entry has S1 + S2 PGDs; QI flushed | `Intel::nested_set_dev_pasid` |
| `Intel::nested_cache_invalidate_user` post: all info.requests submitted to QI; req_num updated | `Intel::nested_cache_invalidate_user` |
| Per-nested-domain at most one parent S2; ref held until free | `IntelNestedDomain` |
| Per-pasid-entry transition atomic (one writer at a time per (dev, pasid)) | `Intel::nested_set_dev_pasid` |

### Layer 4: Verus/Creusot functional

`Guest device DMA at IOVA → S1 walk → guest-PA → S2 walk → host-PA` round-trip equivalence: per-IOVA the host-PA reached by 2-stage IOMMU walk equals composition of guest-managed S1 + host-managed S2.

### hardening

(Inherits row-1 features from `drivers/iommu/intel-iommu.md` § Hardening.)

intel-nested-specific reinforcement:

- **Per-S1 PGD address validated as page-aligned + within guest-PA range** — defense against malicious guest pointing S1 to host kernel memory.
- **Per-pasid-entry write atomic** — defense against IOMMU walking torn entry mid-update.
- **Per-domain parent ref-count** — defense against parent-S2 free-while-S1-attached.
- **Per-cache-invalidate batch req_num verified** — defense against userspace under-reporting consumed entries causing replay.
- **Per-QI submission timeout-bounded** — defense against stuck QI on malicious vIOMMU invalidate-storm.
- **PASID-cache invalidate before pasid-entry write** — defense against IOMMU using stale cached entry post-update.
- **IOTLB invalidate before pasid-entry write** — same, for IOTLB cache.
- **Per-device nested-cap check** — defense against attaching nested-domain to non-supporting device.
- **Per-S1 fault rate-limit** — defense against malicious guest generating fault flood saturating iommufd-eventq.
- **Per-S1 PGTT field constrained to {NESTED, FIRST_LEVEL}** — defense against PGTT mismatch with PGD type.
- **Per-detach S1 PASID drain** before pasid-entry zero — defense against in-flight DMA after detach.

