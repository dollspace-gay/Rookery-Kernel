# Tier-3: drivers/iommu/intel/{pasid,prq,svm}.c — Intel PASID-table mgmt + PRQ MMIO drain + SVM bind

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/iommu/00-overview.md
upstream-paths:
  - drivers/iommu/intel/pasid.c
  - drivers/iommu/intel/pasid.h
  - drivers/iommu/intel/prq.c
  - drivers/iommu/intel/svm.c
-->

## Summary

Intel-specific support for Process Address Space ID (PASID) and the Page Request Interface (PRI) integration. PASID is a PCIe-spec 20-bit identifier that lets one device run multiple parallel address spaces simultaneously (one per process bound via SVA). Intel VT-d's scalable mode adds per-PASID 1st-level + 2nd-level page-table roots in a 4 KB per-device PASID-table, indexed by PASID value.

Three connected files:
- **pasid.c** (~980 lines): per-IOMMU PASID-table management, per-device PASID-table-entry programming for legacy/SVM/nested/identity modes, per-PASID cache invalidation.
- **prq.c** (~400 lines): per-IOMMU Page Request Queue MMIO drain — the IRQ handler that pulls fault entries out of the HW-posted ring and dispatches to per-vendor or generic iopf framework.
- **svm.c** (~240 lines): Intel-specific SVM bind/unbind glue — wires per-mm-bound device into the iommu-sva framework and per-PASID PT-root configuration.

This Tier-3 covers all three files (~1600 lines combined).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct pasid_table` | per-device PASID-table descriptor (4 KB hosting 256 entries; expandable to multi-page for >256 PASIDs) | `kernel::iommu::intel::PasidTable` |
| `struct pasid_entry` | per-PASID 64-byte entry (split into Present + per-mode bits + PT-root pointer + flags) | `kernel::iommu::intel::PasidEntry` |
| `intel_pasid_alloc_table(dev)` / `_free_table(dev)` | per-device PASID-table alloc/free | `Device::pasid_alloc_table` / `_free_table` |
| `intel_pasid_get_entry(dev, pasid)` / `_lookup_entry(...)` | per-PASID entry lookup | `PasidTable::get_entry` |
| `intel_pasid_setup_first_level(iommu, dev, &pgd, pasid, did, flags)` | install 1st-level (SVA) PASID-entry pointing at user mm.pgd | `Device::pasid_setup_first_level` |
| `intel_pasid_setup_second_level(iommu, domain, dev, pasid)` | install 2nd-level (legacy) PASID-entry pointing at host pgd | `Device::pasid_setup_second_level` |
| `intel_pasid_setup_pass_through(iommu, dev, pasid)` | install identity-mapped PASID-entry | `Device::pasid_setup_pass_through` |
| `intel_pasid_setup_nested(iommu, dev, gpgd, pasid, &nested_attrs)` | install nested PASID-entry (1st-level guest-managed + 2nd-level host-managed) | `Device::pasid_setup_nested` |
| `intel_pasid_tear_down_entry(iommu, dev, pasid, fault_ignore)` | clear PASID-entry + flush caches | `Device::pasid_tear_down_entry` |
| `intel_iommu_drain_pasid_prq(iommu, pasid)` | drain pending PRQ entries for a PASID before unbind | `Iommu::drain_pasid_prq` |
| `prq_event_thread(irq, iommu)` | per-IOMMU PRQ IRQ handler (workqueue-driven) | `Iommu::prq_event_thread` |
| `intel_iommu_handle_prq(iommu)` | drain PRQ + dispatch to iopf framework | `Iommu::handle_prq` |
| `intel_iommu_page_response(dev, msg)` | per-vendor PRG-Response sender | `Device::page_response` |
| `intel_svm_bind_mm(iommu, dev, mm)` / `_unbind_mm(handle)` | Intel-specific SVA bind | `Device::svm_bind_mm` / `_unbind_mm` |
| `intel_svm_set_dev_pasid(domain, dev, pasid, old)` | per-PASID device-attach for SVA domain | `Domain::svm_set_dev_pasid` |

## Compatibility contract

REQ-1: Per-device PASID-table layout per VT-d spec § 9.3:
- Default size: 4 KB (256 entries × 16 bytes scalable mode entry, OR 8 entries × 512-byte legacy mode entry — actually scalable mode is 16-byte directory entry → per-PASID 64-byte entry in second level table).
- For PASID > 256: per-device extended PASID directory + multi-page PASID-table.

REQ-2: Per-PASID-entry layout per VT-d spec § 9.4:
- Bit 0: Present (P).
- Bit 6: SuperPage (always set).
- Bit 8-9: PT (1st-level / 2nd-level only / nested).
- Bits 12-N: First-Level PT root (1st-level only / nested).
- Bit 73-117: Domain-ID + Address Width.
- Bits 128-189: Second-Level PT root (2nd-level only / nested).
- Bits 192-255: Per-mode flags (Cache Disable / Memory Type / FPD / etc.).

REQ-3: Per-mode setup:
- **First-level only** (SVA): pgd ptr is user-mm's pgd; addr-width matches CPU; CR3-equivalent invalidation tied to mmu_notifier.
- **Second-level only** (legacy DMA / passthrough): pgd ptr is per-domain 2nd-level pagetable.
- **Pass-through (identity)**: special PT mode = 1 (identity-mapped).
- **Nested**: 1st-level = guest-managed (gpgd) + 2nd-level = host-managed.

REQ-4: PASID alloc/free: per-IOMMU PASID-allocator (bitmap-based) returns 20-bit PASID; per-device association via PASID-table-entry programming.

REQ-5: PASID cache invalidation: after PASID-entry modification, must `qi_flush_pasid_cache(iommu, did, granularity, pasid)` followed by `qi_flush_iotlb_pasid(...)` then `qi_flush_dev_iotlb_pasid(...)` for ATS-capable devices.

REQ-6: Per-IOMMU PRQ register layout per VT-d spec § 7.5:
- PRQ_HEAD (PQH_REG): MMIO-write head pointer (consumer = kernel).
- PRQ_TAIL (PQT_REG): MMIO-read tail pointer (producer = HW).
- PRS_REG: Page Request Status.
- PR-Cap reservoir.

REQ-7: PRQ entry layout per VT-d spec:
- 16 bytes per entry: Source-ID (BDF) + PASID + RW + Privileged + Address + Group-Index + LAST-bit.

REQ-8: PRQ drain flow:
1. IRQ from PRQ-overflow OR PRQ-pending: schedule prq_event_thread.
2. prq_event_thread loops: read PRQ-head + PRQ-tail registers; for each entry [head, tail):
   - Build `iommu_fault_event` from entry fields.
   - Look up per-device fault handler → typically `iopf_handle_event` (cross-ref `iommu-sva.md` + `iommufd-eventq.md`).
3. Update PRQ-head register to indicate consumed.

REQ-9: Per-PASID drain on unbind: `intel_iommu_drain_pasid_prq(iommu, pasid)` walks PRQ for in-flight requests for this PASID; per-spec must complete drain before clearing PASID-entry to avoid race.

REQ-10: SVM bind: per-mm-bound device's PASID-entry installed via `pasid_setup_first_level` with mm.pgd; mmu_notifier registered (cross-ref `iommu-sva.md`).

## Acceptance Criteria

- [ ] AC-1: Intel IDXD device: `iommu_sva_bind_device(dev, mm)` succeeds; PASID-entry installed with mm.pgd; subsequent device DMA via PASID accesses user-allocated buffer.
- [ ] AC-2: PRI fault test: device DMA at non-present PASID-mapped page → PRQ entry posted → prq_event_thread → iopf_handle_event → handle_mm_fault → page-response → device retries.
- [ ] AC-3: PASID alloc + free + re-alloc cycle: 256 SVA binds, then unbinds; subsequent re-alloc reuses freed PASID values.
- [ ] AC-4: Per-PASID drain on unbind: synthesize PRQ entry for pasid X; before drain completes, attempt unbind → blocks; after drain done, unbind succeeds.
- [ ] AC-5: Per-mode PASID test: SVA + identity + 2nd-level + nested all programmable on same device for different PASIDs.
- [ ] AC-6: kvm-unit-tests + kselftest iommufd intel-vt-d subset passes.

## Architecture

`PasidTable` lives in `kernel::iommu::intel::PasidTable`:

```
struct PasidTable {
  table: NonNull<u8>,                 // DMA-coherent 4 KB (or multi-page) buffer
  table_dma: dma_addr_t,
  size: u32,                          // current allocated entries
  refcount: Refcount,
}

#[repr(C)]
struct PasidEntry {
  val: [u64; 8],                       // 64-byte entry (8 u64s)
}
```

`Device::pasid_alloc_table(dev)`:
1. Allocate 4 KB DMA-coherent buffer; zero-init.
2. Per-device store ptr in `dev->iommu_dev->pasid_table`.

`Device::pasid_setup_first_level(iommu, dev, pgd, pasid, did, flags)`:
1. Look up per-device PasidTable.
2. `entry = PasidTable::get_entry(table, pasid)`.
3. Build `pasid_entry` per mode-bits:
   - val[0]: P=1, FLPM=mode (4-level / 5-level), FLPTPTR (pgd phys-addr), AW (addr-width).
   - val[1]: DID, FPD, PGTT (PT mode = first-level-only=2).
4. Memcpy into entry slot atomically.
5. `qi_flush_pasid_cache(iommu, did, GRAN_ALL, pasid)`.
6. `qi_flush_iotlb_pasid(iommu, did, all-iova, mask, pasid)`.
7. For ATS-capable: `qi_flush_dev_iotlb_pasid(iommu, sid, qdep, all-iova, mask, pasid)`.

`Device::pasid_tear_down_entry(iommu, dev, pasid, fault_ignore)`:
1. `entry = PasidTable::get_entry`.
2. Atomically write 0 to entry.val[0] (clear P bit).
3. `qi_flush_pasid_cache + iotlb + dev_iotlb` (with PASID).
4. `intel_iommu_drain_pasid_prq(iommu, pasid)` to wait for in-flight PRQ.
5. If !fault_ignore: process any remaining PRQ entries with INVALID response.

`Iommu::prq_event_thread(irq, iommu)`:
1. Read PRS (Page Request Status) MMIO; ack consumed bits.
2. Loop:
   - Read head + tail from PQH_REG + PQT_REG.
   - If head == tail: break.
   - For each entry in [head, tail):
     - Decode: source-id, pasid, address, RW, priv, last-bit.
     - Look up per-device fault handler via `iommu_report_device_fault(dev, evt)` (cross-ref `iommu-sva.md`).
   - Update PQH_REG to consumed position.
3. Re-arm interrupt.

`Device::page_response(dev, msg)`:
1. Build PRG-Response descriptor per VT-d spec.
2. Submit via per-IOMMU descriptor queue (separate from QI; uses Page Group Response submission).
3. Wait for completion-bit.

`Device::svm_bind_mm(iommu, dev, mm)`:
1. Allocate PASID via `intel_pasid_alloc_id(...)`.
2. `Device::pasid_setup_first_level(iommu, dev, &mm.pgd, pasid, did, flags)`.
3. Register mmu_notifier on mm (cross-ref `iommu-sva.md`).
4. Return iommu_sva handle.

`Device::svm_unbind_mm(handle)`:
1. `Device::pasid_tear_down_entry(iommu, dev, pasid, false)`.
2. Unregister mmu_notifier.
3. `intel_pasid_free_id(pasid)`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pasid_id_no_collision` | UNIQUENESS | per-IOMMU PASID alloc bitmap-tracked; never re-issues active PASID. |
| `pasid_table_no_uaf` | UAF | per-device PasidTable refcount; freed only after all PASID entries cleared. |
| `prq_head_tail_no_oob` | OOB | PRQ head/tail wrap modulo queue size; wraparound consistent. |
| `entry_atomic_write` | ATOMICITY | per-pasid_entry write atomic via 64-bit-aligned store; HW reader sees torn-free. |

### Layer 2: TLA+

`models/iommu/pasid_lifecycle.tla` (parent-declared): proves PASID alloc → bind → mm-notifier-invalidate → unbind → free state machine.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `pasid_setup_first_level` post: entry P=1; FLPTPTR matches pgd phys-addr; cache invalidation completed | `Device::pasid_setup_first_level` |
| `pasid_tear_down_entry` post: entry P=0; per-PASID PRQ drained | `Device::pasid_tear_down_entry` |
| Per-IOMMU PRQ head/tail invariant: `head <= tail (modulo size)`; advanced exactly once per consumed entry | `Iommu::prq_event_thread` |
| `intel_iommu_drain_pasid_prq` post: no in-flight PRQ entry for given pasid remains in queue | `Iommu::drain_pasid_prq` |

### Layer 4: Verus/Creusot functional

`Device::svm_bind_mm(dev, mm) → device DMA via PASID at user_va → handle_mm_fault if non-present → device retries → access succeeds` round-trip equivalence: SVM-bound device's DMA shares mm.pgd with CPU; per-page semantics match CPU.

## Hardening

(Inherits row-1 features from `drivers/iommu/00-overview.md` § Hardening.)

intel-pasid specific reinforcement:

- **Per-IOMMU PASID alloc bitmap** bounded by per-IOMMU MAX_PASID (typically 1M); defense against PASID-exhaustion.
- **Per-PASID drain on unbind mandatory** — defense against UAF-via-PASID-recycle race (new bind reuses just-released PASID before old bind's PRQ drained).
- **PASID-entry write atomic** via 64-bit-aligned u64 store; defense against torn-write race during reprogramming.
- **PRQ overflow handler** sets PRO bit in PRS register; per-IOMMU WARN + drain attempt; defense against PRQ-flood causing missed PRI faults.
- **Per-PASID cache invalidation 3-tuple sequence** (PASID-cache + IOTLB + Dev-IOTLB) ordered correctly per VT-d spec; defense against stale-entry race.
- **PRQ entry source-id validation** — per-entry BDF must correspond to a registered device; malformed entries dropped + WARN.
- **PASID-entry mode validation** — only one mode (first-only / second-only / pass-through / nested) set per entry; conflicting bits rejected at setup.

## Grsecurity/PaX-style Reinforcement

Hardened-policy supplement above baseline `## Hardening`. Intel PASID-table programming is the gateway by which user/guest address spaces become device-visible; PASID-entry corruption directly causes cross-process DMA leak, so PaX/grsec-equivalent mitigations are layered aggressively.

- **PAX_USERCOPY** on PASID-info-leaks via `/proc/<pid>/iommu`, debugfs `pasid_table`, and SVA bind UAPI return values.
- **PAX_KERNEXEC** on PASID-table writers, PRQ drain, and SVM bind/unbind paths (RO post-init).
- **PAX_RANDKSTACK** on `svm_bind_mm` / `pasid_setup_*` entry chains.
- **PAX_REFCOUNT** on `PasidTable`, per-device PASID refs, and SVA handles.
- **PAX_MEMORY_SANITIZE** zeroes `pasid_entry.val[]` on tear-down before P=0 commit + flushes.
- **PAX_UDEREF** on copy_from_user paths for SVA bind ioctl args.
- **PAX_RAP/kCFI** on Intel page_response / fault dispatch indirect calls.
- **GRKERNSEC_HIDESYM** hides PASID-table base addrs and per-PASID mm.pgd pointers.
- **GRKERNSEC_DMESG** restricts PRQ-fault decoded printouts to CAP_SYSLOG.
- **CAP_SYS_ADMIN strict** on raw PASID alloc/setup via iommufd; userspace ATS/PASID-bind requires init-userns.
- **GRKERNSEC_DMA strict-mode** — PASID-entries default-zero until full setup + invalidate sequence completes.
- **ATS/PASID capability gating** — devices without proper PCIe ATS+PRI advertisement refused PASID setup regardless of ECAP.
- **PRQ rate-limit** per-IOMMU + per-source-id; defense against malicious device PRI-storm targeting host scheduler.
- **DMAR ACPI signature verify** before honoring ECAP.PSS / scalable-mode bits.
- **VFIO-compat path deprecated** for PASID-using devices under hardened policy; iommufd-native only.

Rationale: PASID-tables map user/guest virtual memory directly into device DMA. Hardened Rookery refuses every "trust the firmware/device" shortcut here — every cap bit is re-verified, every entry is sanitized, and every degraded-compat mode is locked behind explicit capability + namespace gates.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Generic IOMMU SVA framework (covered in `iommu-sva.md` Tier-3)
- iopf event framework (covered in `iommu-sva.md` Tier-3 + `iommufd-eventq.md` Tier-3)
- Intel VT-d core (covered in `intel-iommu.md` Tier-3)
- Intel nested mode (covered in `intel-nested.md` future Tier-3)
- AMD PASID (covered in `amd-pasid.md` future Tier-3)
- 32-bit-only paths
- Implementation code
