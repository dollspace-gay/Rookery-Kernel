# Tier-3: drivers/iommu/intel/{iommu,cache}.c — Intel VT-d driver (DMAR + per-IOMMU domain + cache invalidation + scalable mode)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/iommu/00-overview.md
upstream-paths:
  - drivers/iommu/intel/iommu.c
  - drivers/iommu/intel/iommu.h
  - drivers/iommu/intel/cache.c
  - drivers/iommu/intel/dmar.c
-->

## Summary

The Intel VT-d (Virtualization Technology for Directed I/O) driver — every Intel server + workstation + most laptops have a VT-d-capable IOMMU. Implements `iommu_ops` per the generic IOMMU API (cross-ref `iommu-core.md`). Supports: legacy mode (LM, used pre-scalable-mode, single PT root per device, identity / DMA / nested translation modes), scalable mode (SM, post-Sapphire Rapids, per-PASID PT roots enabling SVA + nested translation simultaneously), interrupt remapping (IR), Posted Interrupts integration (KVM AVIC equivalent for KVM-Intel posted-interrupts), Page Request Interface (PRI) for PCIe page-faults from PASID-bound devices, queued cache invalidation (QI for batched async TLB flush), and per-IOMMU performance counters (perfmon).

This Tier-3 covers `drivers/iommu/intel/iommu.c` (~4200 lines: per-IOMMU init, per-device attach/detach, page-table mgmt) + `iommu.h` (header) + `cache.c` (~530 lines: QI batched cache invalidation).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct intel_iommu` | per-IOMMU-unit control block | `drivers::intel_iommu::Iommu` |
| `struct dmar_domain` | per-domain (per-IOAS) state | `drivers::intel_iommu::Domain` |
| `struct device_domain_info` | per-device-attach record | `drivers::intel_iommu::DeviceInfo` |
| `intel_iommu_init()` | subsystem init: parse DMAR ACPI table, init each IOMMU | `Subsystem::init` |
| `intel_iommu_init_qi(iommu)` / `_init_pri(iommu)` / `_init_smmu(iommu)` | per-IOMMU init phases | `Iommu::init_qi` / `_init_pri` |
| `intel_iommu_alloc_domain(parent)` / `_free_domain(domain)` | domain alloc/free | `Domain::alloc` / `_free` |
| `intel_iommu_attach_device(domain, dev)` / `_detach_device(...)` | per-device attach | `Domain::attach_device` / `_detach_device` |
| `intel_iommu_map_pages(domain, iova, paddr, sz, count, prot, gfp, &mapped)` / `_unmap_pages(...)` | per-domain map/unmap | `Domain::map_pages` / `_unmap_pages` |
| `intel_iommu_iotlb_sync_map(domain, iova, size)` / `_iotlb_sync(domain, gather)` | per-batch invalidation | `Domain::iotlb_sync_map` / `_iotlb_sync` |
| `intel_iommu_iova_to_phys(domain, iova)` | translate | `Domain::iova_to_phys` |
| `intel_iommu_dev_disable_pasid(...)` / `_enable_pasid(...)` | per-device PASID enable/disable | `DeviceInfo::pasid_*` |
| `qi_submit_sync(iommu, desc, count, options)` | submit batch of QI descriptors + wait | `Iommu::qi_submit_sync` |
| `qi_flush_context(iommu, did, sid, fm, type)` | flush context-cache (DID granularity) | `Iommu::qi_flush_context` |
| `qi_flush_iotlb(iommu, did, addr, mask, type)` | flush IOTLB | `Iommu::qi_flush_iotlb` |
| `qi_flush_dev_iotlb(iommu, sid, qdep, addr, mask)` | flush device-IOTLB (ATS-capable devices) | `Iommu::qi_flush_dev_iotlb` |
| `qi_flush_pasid_cache(iommu, did, granu, pasid)` | flush PASID cache | `Iommu::qi_flush_pasid_cache` |
| `intel_iommu_drain_pasid_prq(iommu, pasid)` | drain pending PRQ for a PASID | `Iommu::drain_pasid_prq` |
| `intel_iommu_set_dirty_tracking(domain, enable)` / `_read_and_clear_dirty(...)` | per-domain dirty-tracking (SLADE — Second-Level Access/Dirty Enable) | `Domain::set_dirty_tracking` / `_read_and_clear_dirty` |
| `device_block_translation(dev)` / `_unblock_translation(dev)` | per-device blocked-domain attach | `DeviceInfo::block_translation` / `_unblock_translation` |
| `iommu_calculate_max_sagaw(iommu)` | compute max SAGAW (Supported Adjusted Guest Address Width) | `Iommu::calculate_max_sagaw` |
| `intel_iommu_init_pasid(iommu, dev)` | per-device PASID alloc + bind | `Iommu::init_pasid` |

## Compatibility contract

REQ-1: DMAR ACPI table parsing (cross-ref `dmar.md`): per-IOMMU-unit instance discovered + initialized; per-RMRR (Reserved Memory Region Reporting) range marked as identity-mapped for boot devices (USB legacy, integrated GPU).

REQ-2: Per-IOMMU register layout per VT-d spec: capability registers, extended capability, root-table address, context-table address, IOTLB invalidation, IR table address, queued invalidation queue, page-request queue.

REQ-3: Legacy mode: per-device root-table-entry → context-table-entry → 2nd-level page-table; supports 4-level (48-bit IOVA) or 5-level (57-bit IOVA) per CPUID.

REQ-4: Scalable mode (SM): per-device PASID-table-entry → per-PASID 1st-level + 2nd-level page-tables; supports SVA (1st-level uses CPU page-tables shared with `mm_struct`), nested (1st-level guest-managed, 2nd-level host-managed).

REQ-5: Cache invalidation via QI (Queued Invalidation) — register-based legacy IOTLB invalidate replaced by descriptor-queue MMIO writes; per-iommu QI queue + completion-wait descriptors.

REQ-6: Per-VT-d generation feature support: VT-d 1.0 / 2.0 / 3.0 / 4.0 / 4.1 / 5.0 — 5.0 adds first-level translation, scalable mode, PASID, dirty tracking; per-feature CAP/ECAP register bit checked at init.

REQ-7: Page Request Interface (PRI): per-IOMMU PRQ (Page Request Queue) MMIO drain via `intel_pri_irq` handler; routes per-PASID page-fault to consumer (cross-ref `iommu-sva.md`).

REQ-8: Posted-Interrupt + IR integration: when CONFIG_IRQ_REMAP=y AND VT-d IR enabled, per-vector IRTE programmed in IRT; KVM IPI-bypass via posted-interrupt-descriptor.

REQ-9: Per-IOMMU perfmon (CONFIG_INTEL_IOMMU_PERF): per-IOMMU PMU registered as perf-event source for IOTLB hit/miss + pasid-cache stats.

REQ-10: Default-domain selection per cmdline `intel_iommu={on,off,igfx_off,nobounce,sm_on,sm_off}` + per-RMRR-required devices forced to identity mode.

REQ-11: Dirty-page tracking via SLADE (5.0+): per-page `dirty` bit set on writes; `read_and_clear_dirty` walks pgtable + clears + reports bitmap (used by iommufd live-migration).

REQ-12: ATS / PRI integration with `drivers/pci/ats.c`: per-device ATS enabled via `pci_enable_ats` after attach; PRI enabled via `pci_enable_pri` for SVA-capable devices.

## Acceptance Criteria

- [ ] AC-1: `dmesg | grep -i 'DMAR\|VT-d\|intel-iommu'` shows correct per-IOMMU-unit init on Intel reference HW.
- [ ] AC-2: vfio-pci passthrough of NVMe device to qemu KVM guest works (cross-ref `iommu-core.md` AC).
- [ ] AC-3: SR-IOV VF passthrough works on Intel ice/igb NIC.
- [ ] AC-4: Scalable Mode test on supported HW (Sapphire Rapids+): SVA (`iommu_sva_bind_device`) succeeds; PCIe device PASID-bound to userspace `mm_struct`.
- [ ] AC-5: PRI test on PRI-capable device: device page-fault → host PRQ entry → SVA bound consumer handles + responds.
- [ ] AC-6: IR test: KVM with x2APIC + posted-interrupt mode → IRTE programmed; vmexit-bypass observable.
- [ ] AC-7: Dirty-tracking test: iommufd HWPT_GET_DIRTY_BITMAP correctly reports written pages on Sapphire Rapids+ HW.
- [ ] AC-8: cmdline test: `intel_iommu=on iommu=pt` boots with default-IDENTITY domain; `intel_iommu=on iommu=strict` boots with strict-flush mode.
- [ ] AC-9: kselftest `tools/testing/selftests/iommu/` Intel-specific subset passes.

## Architecture

`Iommu` lives in `drivers::intel_iommu::Iommu`:

```
struct Iommu {
  reg: NonNull<u8>,            // per-IOMMU MMIO region (memremap'd from drhd entry)
  cap: u64,                    // CAP register cache
  ecap: u64,                   // ECAP register cache
  drhd: Arc<DmarDrhdUnit>,     // ACPI DMAR DRHD unit descriptor
  agaw: u8,                    // adjusted guest address width
  msagaw: u8,                  // max SAGAW
  node: NumaNode,
  flush: IommuFlush,           // per-IOMMU flush callbacks (legacy register-based or QI)
  iommu_state: [u32; 32],      // for suspend/resume
  qi: Option<KBox<QInvalidation>>,
  irt: Option<KBox<IrTable>>,
  prq: Option<KBox<PageRequestQueue>>,
  perf: Option<KBox<IommuPerfmon>>,
  sm: bool,                    // scalable-mode enabled
  pasid_table: Option<NonNull<PasidTable>>,
  root_table: NonNull<RootEntry>,
  ir_pre_enabled: bool,
  rmrr_list: Mutex<Vec<Arc<DmarRmrrUnit>>>,
}

struct Domain {
  refcount: Refcount,
  iommu_ops: &'static IntelIommuOps,
  type_: DomainType,           // DMA / DMA_FQ / IDENTITY / NESTED / UNMANAGED / BLOCKED
  pgd: NonNull<u64>,            // 2nd-level page-directory root (for legacy or SM 2nd level)
  agaw: u8,
  flags: u32,                  // DOMAIN_FLAG_INTEL_IOMMU_NESTED / _STATIC_IDENTITY / etc.
  iommu_did: [u16; MAX_IOMMUS], // per-IOMMU domain id (DID)
  attached: Mutex<HashSet<Arc<DeviceInfo>>>,
  iommu_count: AtomicU32,
  super_pages_disabled: AtomicBool,
  iommu_snoop: AtomicBool,
  iommu_coherency: AtomicBool,
  has_iotlb_device: AtomicBool,
  is_dirty_tracking_enabled: AtomicBool,
  cache: KBox<DomainCache>,    // per-domain invalidation cache batch
}
```

Subsystem init `Subsystem::init`:
1. `dmar_table_init()` — parse ACPI DMAR table (cross-ref `drivers/iommu/intel-dmar.md`).
2. For each DRHD (DMA Remapping Hardware Definition):
   - `Iommu::alloc(&drhd)` → memremap MMIO + read CAP/ECAP.
   - `Iommu::init_qi(&iommu)` → alloc QI queue, enable QIE bit.
   - If ECAP.PRS: `Iommu::init_pri(&iommu)` → alloc PRQ.
   - If ECAP.SMTS: enable scalable mode.
   - If CAP.SLLPS: enable super-page support.
3. For each device under each DRHD: `iommu_probe_device(dev)` from generic IOMMU core (cross-ref `iommu-core.md`) which calls back into `intel_iommu_probe_device`.
4. Apply RMRR identity mapping for boot-required devices.

Per-device attach `Domain::attach_device`:
1. `find_iommu_for_dev(dev)` → owning IOMMU.
2. Allocate per-domain DID for this IOMMU (per-IOMMU DID space limited to 64K).
3. Per-mode setup:
   - **Legacy mode**: write context-table entry (root_table[bus] → ctx[devfn]) with 2nd-level pgd + DID + AGAW + present-bit.
   - **Scalable mode**: write PASID-directory + per-PASID-table entry with both 1st-level (if any) + 2nd-level pgd + DID.
4. `qi_flush_context(iommu, did, sid, FM_GLOBAL, DOMAIN_SEL)` to invalidate context-cache.
5. `qi_flush_iotlb(iommu, did, ...)` to invalidate IOTLB.
6. Register dev to domain.attached.

Per-domain page mapping `Domain::map_pages`:
1. Validate iova + size against domain.agaw + spec-max.
2. Walk 2nd-level pgtable from `domain.pgd`, allocating intermediate page-table pages if absent.
3. At leaf: write 4 KB / 2 MB / 1 GB superpages per `pgsize` arg + per-CAP.SLLPS support.
4. If domain.iommu_snoop: SNOOP bit set on each leaf PTE.
5. Bump `mapped` count.

Per-batch invalidation `Domain::iotlb_sync(gather)`:
1. For each IOMMU in `domain.iommu_did`:
   - Build QI descriptor from `gather` (per-iova range + per-PASID).
   - Append to per-IOMMU QI queue.
2. Submit + wait via `qi_submit_sync` (write tail register, wait completion descriptor).

Page Request Interface IRQ handler `intel_pri_irq`:
1. Drain PRQ via MMIO read until empty.
2. Per-entry: extract PASID + Source-Id + IOVA + flags.
3. Look up SVA-bound `mm_struct` via PASID table.
4. `handle_mm_fault(mm, vma, iova, fault_flags)` — host page-walk fault handler.
5. Build PR-Response descriptor + submit to QI to ack page-grant or page-fault to device.

QI submission `qi_submit_sync(iommu, desc, count, options)`:
1. Append `count` desc + 1 Wait descriptor to QI queue.
2. Write tail register (MMIO).
3. Spin (or sleep on completion var) until Wait descriptor's status field is non-zero.

Cache invalidation modes:
- **Per-DID**: invalidate all entries for a domain.
- **Per-IOVA**: invalidate single IOVA + neighborhood (via mask).
- **Global**: invalidate everything (used at IOMMU init + after major reconfig).
- **Device-IOTLB**: ATS-capable devices have local IOTLB; invalidate via QI device-IOTLB descriptor.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pgd_walk_no_oob` | OOB | per-level page-table walk bounded by domain.agaw; index per-level always < 512 (4 KB level) or < 512 (PMD/PUD/PGD). |
| `qi_queue_no_oob` | OOB | QI tail/head pointers bounded by queue size; wrap correctly. |
| `did_no_collision` | UNIQUENESS | per-IOMMU DID allocator bitmap-tracked; never re-issues active DID. |
| `iommu_state_no_uaf` | UAF | `Arc<Iommu>` outlives all domains; per-domain detach removes did before iommu free. |

### Layer 2: TLA+

`models/iommu/qi_descriptor.tla` (parent-declared): proves Intel descriptor-queue submit + completion + tail-fence + invalidation-wait observe ordering needed by spec; concurrent submit from N CPUs serializes correctly via head pointer.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Domain::map_pages` post: every page in [iova, iova+sz) translates to corresponding paddr via `Domain::iova_to_phys` | `Domain::map_pages` |
| `Domain::unmap_pages` post: every page in unmapped range returns 0 from `iova_to_phys` | `Domain::unmap_pages` |
| `qi_submit_sync` post: Wait descriptor's status field == 1 (HW completed all preceding descriptors) | `Iommu::qi_submit_sync` |
| Per-domain `attached` invariant: every attached device has corresponding per-IOMMU DID assigned | `Domain::attach_device` |

### Layer 4: Verus/Creusot functional

Host pte change → MMU notifier invalidate_range_start → domain page-table invalidation → `qi_submit_sync` returns → device IOTLB drained → guest fault-on-next-access. Encoded as Verus invariant chained with `iommu-core.md`'s map↔unmap inverse.

## Hardening

(Inherits row-1 features from `drivers/iommu/00-overview.md` § Hardening.)

intel-iommu specific reinforcement:

- **DMAR table parsing validates per-DRHD address against ACPI DMAR-region attribute** — defense against malformed DMAR table pointing to invalid MMIO.
- **Per-IOMMU MMIO mapping CAP_SYS_RAWIO** at init — defense against misuse of memremap.
- **QI queue size bounded** — default 4 KB (256 descriptors); defense against per-fault queue-flood DoS.
- **PRQ queue size bounded** — default 4 KB; per-PASID fault rate-limited.
- **Per-DID alloc bitmap** — DID space 64K per-IOMMU; alloc strictly bounded; defense against DID-exhaustion DoS.
- **Page-table page free deferred via RCU** — concurrent device DMA + host pgtable walk safe across teardown.
- **SNOOP bit forced-on for non-coherent system caches** — defense against MMU-cache-incoherence bug exposing stale data.
- **Scalable-mode PASID-table entry validation** — per-PASID PT root validated against PASID-bounded `mm_struct` lifetime; PASID never re-bound while in-flight DMA.
- **IRTE programming validates target-CPU + vector** — defense against malformed IRTE redirecting interrupts to invalid CPU/vector.
- **PRQ IRQ handler bounded per-invocation** — defense against PRQ-flood causing soft-lockup.
- **Per-IOMMU `iommu_state` save/restore on suspend/resume** — defense against state-loss on S3 cycles.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- AMD-Vi (covered in `amd-iommu.md` future Tier-3)
- DMAR ACPI parser (covered in `intel-dmar.md` future Tier-3)
- Intel SVM details (covered in `intel-svm.md` future Tier-3)
- Intel nested mode (covered in `intel-nested.md` future Tier-3)
- IRQ remapping details (covered in `intel-irq-remap.md` future Tier-3)
- Per-IOMMU perfmon (covered in `intel-perfmon.md` future Tier-3)
- 32-bit-only paths
- Implementation code
