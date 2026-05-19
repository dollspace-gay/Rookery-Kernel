# Tier-3: drivers/iommu/intel/svm.c — Intel SVM (PASID-bound shared address space + per-mm-PASID install + mm-notifier integration)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/iommu/00-overview.md
upstream-paths:
  - drivers/iommu/intel/svm.c
  - drivers/iommu/intel/iommu.h (sva-related)
  - include/linux/intel-svm.h
-->

## Summary

Intel-specific Shared Virtual Addressing (SVM, also called SVA — Shared Virtual Addressing) bind glue. Wires per-mm-bound device into the generic iommu-sva framework (cross-ref `iommu-sva.md`) and per-PASID PT-root configuration via `intel-pasid.md`. SVM lets a PASID-capable PCIe device walk the same page tables as the bound process's `mm_struct` — every device DMA sees user-virtual addresses + faults via PRI on non-present pages.

Backbone of: Intel IDXD (Data Streaming Accelerator) + Intel DSA + Intel-IAA (In-Memory Analytics Accelerator) + Intel-QAT (QuickAssist Technology) + Intel-FPGA + Intel SGX-IO; AMD has parallel `AMD-V SVM` via separate `amd/iommu.c` paths but the API surface is identical via `iommu_sva_bind_device`.

This Tier-3 covers `drivers/iommu/intel/svm.c` (~240 lines) — Intel-specific bind/unbind + iommu-sva domain hookup.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `intel_svm_bind_mm(iommu, dev, mm)` | per-mm bind of dev via PASID | `Device::intel_svm_bind_mm` |
| `intel_svm_unbind_mm(handle)` | inverse | `Sva::intel_unbind` (Drop) |
| `intel_iommu_set_dev_pasid(domain, dev, pasid, old)` | per-PASID install of SVA-domain | `Domain::intel_set_dev_pasid` |
| `intel_iommu_remove_dev_pasid(dev, pasid, domain)` | inverse | `Domain::intel_remove_dev_pasid` |
| `intel_svm_unbind` (replacing-domain path) | drop old domain when replacing | `Domain::intel_svm_unbind_replace` |
| `intel_iommu_sva_supported(dev)` | check device + IOMMU support SVA | `Device::intel_sva_supported` |
| `intel_svm_check_pasid(dev, mm)` | per-mm PASID lookup helper | `Device::svm_check_pasid` |
| `intel_iommu_sva_invalidate(domain, addr, size)` | per-domain invalidate (called from MMU notifier) | `Domain::sva_invalidate` |
| `intel_iommu_sva_release_pasid(handle)` | release PASID after drain | `Sva::release_pasid` |

## Compatibility contract

REQ-1: Intel SVM bind succeeds only when:
- IOMMU has SM (Scalable Mode) enabled (CAP.ECAP.SMTS bit set).
- Device exposes PCIe PASID + PRI + ATS capabilities.
- Per-vendor `iommu_ops->dev_enable_feat(dev, IOMMU_DEV_FEAT_SVA)` succeeds.

REQ-2: Per-bind PASID alloc via Intel PASID-allocator (cross-ref `intel-pasid.md`); per-IOMMU 20-bit PASID space (~1M).

REQ-3: Per-PASID PT-root install via `pasid_setup_first_level`: ptr to mm.pgd (4-level for 48-bit-VA / 5-level for 57-bit-VA per CPU CR4.LA57); flags include FLPM + AW (address-width) + DID + FLPTPTR.

REQ-4: MMU notifier registration on bound mm: per-mm `mmu_interval_notifier` covers entire VA range [0, ULONG_MAX]; on host pte change (any of: page migration / KSM / NUMA balancing / mprotect / unmap / madvise(DONTNEED)) → notifier's `invalidate` callback fires per-vendor `intel_iommu_sva_invalidate(domain, range.start, range.end - range.start)` → per-IOMMU TLB flush for affected range.

REQ-5: Per-PASID drain on mm exit: `mmu_notifier_release(mm)` (called from `__mmput`) → per-bind release callback → `intel_iommu_drain_pasid_prq` (cross-ref `intel-pasid.md`) before clearing PASID-entry to avoid PRQ-race.

REQ-6: Per-PASID drain on explicit unbind: `iommu_sva_unbind_device(handle)` → drain PRQ for this PASID → clear PASID-entry → free PASID.

REQ-7: Per-IOMMU SVA domain shared across bindings: `iommu_get_dma_domain(group)` returns same SVA domain for multiple devices in same group.

REQ-8: Per-bind per-process SVM auditing: per-process binding count tracked via mm refcount; defense against mm exit while bind active.

REQ-9: SVA domain cleanup on last-unbind: when last PASID for this domain unbound, domain freed via `iommu_domain_free`.

REQ-10: PRQ fault dispatch (cross-ref `intel-pasid.md` PRQ + `iommu-sva.md` iopf): per-PASID fault routed to per-bind handler → `handle_mm_fault(mm, vma, address, flags)` → page-response sent.

## Acceptance Criteria

- [ ] AC-1: Intel IDXD test program: open `/dev/dsa/wq0.0` chardev → mmap WQ → submit work descriptor with addr = malloc'd buffer; PASID-bound DMA accesses buffer via SVA without explicit dma_map.
- [ ] AC-2: PRI fault test: device DMA at non-present user-VA → PRI fault → kernel handles via handle_mm_fault → page-faulted-in → device retries.
- [ ] AC-3: madvise(DONTNEED) test: madvise on SVA-bound user range → MMU notifier invalidates IOMMU TLB → subsequent device access PRI-faults + reads zero page.
- [ ] AC-4: Process-exit cleanup: SVA-bound process killed → all PASID bindings auto-released; subsequent PASID alloc reuses freed PASID.
- [ ] AC-5: Cross-process isolation: SVA bind in process A; process B cannot DMA via A's PASID.
- [ ] AC-6: Multi-bind on same mm: 2 devices both bind to same mm → each gets unique PASID; both DMA to user-VA correctly.
- [ ] AC-7: kselftest IOMMU SVA-related Intel-specific subset passes.

## Architecture

Intel SVM bind flow `Device::intel_svm_bind_mm(iommu, dev, mm)`:
1. Check `iommu.sm` (scalable mode enabled).
2. Check device PASID + PRI + ATS caps via `pci_pasid_features` + `pci_pri_features` + `pci_ats_supported`.
3. Per-vendor `iommu_ops->dev_enable_feat(dev, IOMMU_DEV_FEAT_SVA)`:
   - Enable PASID + PRI + ATS on device.
   - For PRI: `pci_enable_pri(dev, max-pri-reqs)`.
4. `pasid = intel_pasid_alloc_id(...)` — alloc PASID from per-IOMMU pool.
5. `intel_pasid_setup_first_level(iommu, dev, mm.pgd, pasid, did, flags=PASID_FLAG_SUPERVISOR_MODE * (cpl == 0))` (cross-ref `intel-pasid.md`).
6. Allocate `iommu_sva` handle; install pasid + mm + dev.
7. Register MMU notifier on mm:
   - `mmu_interval_notifier_insert(&sva.mn, &mm.notifier_subscriptions, 0, ULONG_MAX, &intel_sva_mn_ops)`.
   - On invalidate: `intel_iommu_sva_invalidate(domain, range.start, size)` → per-IOMMU TLB flush.
8. Return iommu_sva handle.

Per-PASID install `Domain::intel_set_dev_pasid(domain, dev, pasid, old)`:
1. Lookup `iommu` from dev.
2. Validate domain.type == SVA (cross-ref `iommu-core.md`).
3. `intel_pasid_setup_first_level(iommu, dev, mm.pgd, pasid, did, flags)`.
4. Add dev to domain.attached_devs (RCU-protected).
5. If old non-NULL: drain old PASID's PRQ + clear old's entry.

MMU notifier callback `intel_sva_mn_ops::invalidate(notifier, range)`:
1. `sva = container_of(notifier, IntelSva, mn)`.
2. Per-iommu (multi-domain shared SVA): walk all attached IOMMUs.
3. `qi_flush_iotlb_pasid(iommu, did, range.start, mask, sva.pasid)`.
4. For each attached ATS-capable device: `qi_flush_dev_iotlb_pasid(iommu, sid, qdep, range.start, mask, sva.pasid)`.
5. Wait for completion via QI Wait descriptor.
6. Return 0 (allow host pte change to proceed).

Per-bind unbind `Sva::intel_unbind(handle)` (called from `iommu_sva_unbind_device`):
1. Stop new PRI faults for this PASID: per-vendor pasid-block (intel: temporary INVALID PASID-entry).
2. `intel_iommu_drain_pasid_prq(iommu, pasid)` — drain pending PRI faults; per-fault response = INVALID.
3. `intel_pasid_tear_down_entry(iommu, dev, pasid, fault_ignore=false)` — clear PASID-entry + cache flush + final drain.
4. Unregister MMU notifier.
5. `intel_pasid_free_id(pasid)`.
6. Drop `iommu_sva` handle.

mm exit cleanup: `mmu_notifier_release(mm)` (from `__mmput`) → per-bind notifier's `release` callback fires → calls into iommu-sva framework → per-bind unbind sequence.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pasid_no_collision` | UNIQUENESS | per-IOMMU PASID alloc bitmap-tracked; never re-issues active PASID. |
| `mm_no_uaf` | UAF | mm refcount held via mmgrab while sva-bound; mm exit auto-unbinds via mmu_notifier_release. |
| `prq_drain_completes` | TERMINATION | per-PASID PRQ drain has bounded retry; over-bound triggers WARN + force-clear. |
| `domain_attach_consistent` | INVARIANT | Domain.attached_devs reflects actual per-PASID-table-entry installations. |

### Layer 2: TLA+

`models/iommu/pasid_lifecycle.tla` (parent-declared): proves PASID alloc → bind → mm-notifier-invalidate → unbind → free state machine; mm exit + concurrent device DMA never produce use-after-free.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Device::intel_svm_bind_mm` post: returned handle has handle.pasid == per-iommu allocated; handle.mm == provided mm; mmu_notifier installed; PASID-entry P=1 with mm.pgd | `Device::intel_svm_bind_mm` |
| `Sva::intel_unbind` post: PASID-entry P=0; PRQ drained; PASID freed; mmu_notifier removed; mm refcount dropped | `Sva::intel_unbind` |
| `intel_sva_mn_ops::invalidate` invariant: every host pte change in [start, end) is preceded by IOMMU TLB invalidate for [start, end) on bound PASIDs (parent's pasid_lifecycle.tla is the proof) | MMU notifier callback |

### Layer 4: Verus/Creusot functional

`Device::intel_svm_bind_mm(dev, mm) → device DMA via PASID at user_va → handle_mm_fault if non-present → device retries → access succeeds` round-trip equivalence: SVA-bound device's DMA effectively shares mm.pgd with CPU; per-page semantics match CPU access.

## Hardening

(Inherits row-1 features from `drivers/iommu/00-overview.md` § Hardening.)

intel-svm specific reinforcement:

- **Per-IOMMU SM (scalable mode) required** — bind rejected on legacy-only IOMMU; defense against config-mismatch crash.
- **Device cap validation strict** — bind requires PASID + PRI + ATS all present; missing any returns -EOPNOTSUPP.
- **Per-PASID drain on every unbind** mandatory — defense against UAF-via-PASID-recycle race.
- **mm refcount held via mmgrab** — defense against mm freed while SVA-bound.
- **MMU notifier ordering enforced** — invalidate-range-start happens BEFORE host pte change visible to SVA device; defense against device DMA reading stale-but-physically-already-freed page.
- **PRQ drain timeout bounded** — per-drain attempt bounded by per-IOMMU timeout; over-timeout WARN + force-PASID-entry-clear.
- **Cross-process bind isolation** — per-bind handle ties PASID to specific mm; cross-mm DMA via stale PASID rejected at per-vendor PRQ entry source-id check.
- **PASID-entry mode validation** — Intel SVM uses first-level-only mode; nested + identity rejected at intel_svm_bind_mm.

## Grsecurity/PaX-style Reinforcement

Hardened-policy supplement above baseline `## Hardening`. SVM binds a user `mm_struct` directly into device DMA via PASID — any drift between the IOMMU TLB and the host MMU translates to cross-process info-leak or kernel UAF, so SVA bind ergonomics are gated tightly.

- **PAX_USERCOPY** on SVA bind ioctl return paths and PASID-info leaks via debugfs.
- **PAX_KERNEXEC** on `intel_svm_bind_mm`, MMU-notifier callbacks, and unbind paths (RO post-init).
- **PAX_RANDKSTACK** on bind/unbind entry chains crossing mm-notifier registration.
- **PAX_REFCOUNT** on `iommu_sva` handles, mm refs (mmgrab), and per-PASID device refs.
- **PAX_MEMORY_SANITIZE** zeroes SVA-domain memory + PASID-entries on unbind.
- **PAX_UDEREF** on copy_from_user paths in SVA bind UAPI (iommufd ioctls).
- **PAX_RAP/kCFI** on `intel_sva_mn_ops` (MMU-notifier vtable) and SVA domain-ops.
- **GRKERNSEC_HIDESYM** hides per-mm PASID assignments and bound-device PASID-tables.
- **GRKERNSEC_DMESG** restricts SVA bind/unbind decoded printouts to CAP_SYSLOG.
- **CAP_SYS_ADMIN strict** on SVA-bind from outside init userns; nested-userns blocked from raw SVA.
- **GRKERNSEC_DMA strict-mode** — SVA-capable devices default-blocked until PASID+PRI+ATS triple-verified.
- **ATS/PASID capability gating** — devices must expose all three (PASID, PRI, ATS) via verified PCIe caps; ECAP-only path refused.
- **VT-d PRQ rate-limit** per PASID; defense against malicious DSA workload PRI-flooding the host scheduler.
- **DMAR ACPI signature verify** before honoring SMTS bit / scalable-mode enablement.
- **VFIO-compat path refused** for SVM-bound devices under hardened policy; iommufd-native bind only.
- **Per-bind PASID-mode constrained to first-level-only** — nested / identity rejected at SVA bind path.

Rationale: SVM erases the boundary between a user process's mm and a hostile device's DMA. Hardened Rookery enforces every PCIe + VT-d cap re-validation on every bind, never trusts cached vendor flags, and refuses each fallback (legacy mode, VFIO-compat, missing PRI) that upstream tolerates.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Generic IOMMU SVA framework (covered in `iommu-sva.md` Tier-3)
- Intel PASID-table mgmt (covered in `intel-pasid.md` Tier-3)
- iopf event framework (covered in `iommu-sva.md` Tier-3 + `iommufd-eventq.md` Tier-3)
- Intel VT-d core (covered in `intel-iommu.md` Tier-3)
- AMD SVM (covered in `amd-pasid.md` future Tier-3)
- 32-bit-only paths
- Implementation code
