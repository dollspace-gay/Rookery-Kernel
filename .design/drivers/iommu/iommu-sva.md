# Tier-3: drivers/iommu/iommu-sva.c + io-pgfault.c — Shared Virtual Addressing (PASID-bound device DMA via CPU page tables + PRI fault handling)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/iommu/00-overview.md
upstream-paths:
  - drivers/iommu/iommu-sva.c
  - drivers/iommu/io-pgfault.c
  - drivers/iommu/iommu-priv.h
  - include/linux/iommu.h
-->

## Summary

SVA (Shared Virtual Addressing) — PASID-bound device DMA where the device walks the same page tables as the CPU's `mm_struct`. Originally created for Intel IDXD (Data Streaming Accelerator) + Intel DSA + Intel-IAA + Intel-QAT; now also used by FPGA accelerators (Intel-FPGA + Xilinx Versal), AMD-vIOMMU + AMD-SVM consumers, ARM SMMUv3 SSV. Critical because: (a) zero-copy + zero-pin — userspace allocates buffer normally, hands ptr to device, device reads/writes it directly without any extra DMA-mapping; (b) per-process isolation — devices bound via PASID can only access their owner process's memory, never other processes'.

PRI (Page Request Interface) is the PCIe spec extension that lets devices fault on non-present pages: device sends Page Request → IOMMU dispatches to host CPU → kernel walks `mm_struct`, faults page in, sends Page Response → device retries access.

This Tier-3 covers `drivers/iommu/iommu-sva.c` (~340 lines) + `io-pgfault.c` (~550 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct iommu_sva` | per-bind handle | `kernel::iommu::sva::Sva` |
| `iommu_sva_bind_device(dev, mm)` | bind dev to mm via PASID | `Device::sva_bind` |
| `iommu_sva_unbind_device(handle)` | unbind | `Sva::unbind` (Drop) |
| `iommu_sva_get_pasid(handle)` | extract PASID number | `Sva::pasid` |
| `iopf_queue_alloc(name)` / `_free(queue)` | per-IOMMU PRI fault queue init | `IopfQueue::alloc` / `_free` |
| `iopf_queue_add_device(queue, dev)` / `_remove_device(...)` | per-device fault subscription | `IopfQueue::add_device` / `_remove_device` |
| `iopf_queue_flush_dev(dev)` | wait for pending faults to drain | `IopfQueue::flush_dev` |
| `iommu_register_device_fault_handler(dev, handler, data)` | per-device fault handler install | `Device::register_fault_handler` |
| `iommu_unregister_device_fault_handler(dev)` | inverse | `Device::unregister_fault_handler` |
| `iommu_report_device_fault(dev, evt)` | per-IOMMU vendor entry-point: deliver fault to handler | `Device::report_fault` |
| `iommu_page_response(dev, msg)` | per-device PRI response | `Device::page_response` |
| `iopf_handle_group(work)` | workqueue handler for PRI fault group | `IopfQueue::handle_group_work` |
| `iopf_handle_single(fault)` | single-fault handler dispatch | `IopfQueue::handle_single` |
| `iopf_complete_group(dev, iopf_group, status)` | response to PRI group | `IopfQueue::complete_group` |
| `mmu_notifier_*` family on per-bind mm_struct | invalidate device pgtable on host pte change (cross-ref `mm/00-overview.md`) | `Sva::mmu_notifier_*` |

## Compatibility contract

REQ-1: `iommu_sva_bind_device(dev, mm)` per-spec:
- Allocate PASID via per-IOMMU PASID-table.
- Per-vendor: install per-PASID PT root pointing at mm's PGD (Intel) OR sub-table-entry pointing at mm's PGD (AMD).
- Register MMU notifier on mm so host pte changes invalidate device pgtable.
- Return `iommu_sva` handle.

REQ-2: Bind-time validation: dev must support PASID (PCIe PASID capability) AND PRI (Page Request Interface) AND ATS (Address Translation Service); else return -EOPNOTSUPP.

REQ-3: `iommu_sva_get_pasid(handle)` returns the PASID number assigned to this bind; consumed by per-driver code that programs the device with this PASID for subsequent transactions.

REQ-4: PRI fault flow:
1. Device fault on non-present page → ATS-PRQ packet emitted (PCIe-spec).
2. IOMMU receives via PRQ (Page Request Queue) MMIO.
3. Per-IOMMU IRQ handler: drain PRQ, build `iommu_fault_event`, call `iommu_report_device_fault(dev, evt)`.
4. Per-device handler (installed via `iommu_register_device_fault_handler`): typically `iopf_handle_event` from io-pgfault subsystem.
5. iopf core: queue fault into per-device fault group; schedule workqueue.
6. Workqueue: walk per-fault PASID → per-fault `mm_struct` → `handle_mm_fault(mm, vma, address, fault_flags)` to fault page in.
7. Per-fault page-response sent back to device via `iommu_page_response(dev, msg)`.
8. Device retries access.

REQ-5: Per-fault group: PRQ fault entries with PRGI (Page Request Group Index) coalesced into a group; group response sent atomically (defense against partial-response confusing device).

REQ-6: Per-device fault rate-limit: per-device PRI fault rate capped (default 100k/sec); over-cap WARN + auto-disable PRI on the device.

REQ-7: MMU notifier integration: per-bind `mmu_interval_notifier` on `mm`; host pte changes (page-migration / KSM / NUMA balancing / mprotect / unmap) trigger per-vendor IOMMU pgtable invalidation before host-pte change becomes visible.

REQ-8: Per-bind unbind: `iommu_sva_unbind_device(handle)`:
- Stop new PRI faults for this PASID (per-vendor block PASID).
- Drain pending PRI faults.
- Unregister MMU notifier.
- Per-vendor unbind PT-root.
- Free PASID.

REQ-9: PASID lifetime tied to `mm_struct`: when last user of mm exits, all SVA bindings auto-unbind via mm-exit notifier.

REQ-10: Per-device fault handler invoked once per fault (deduplication across PRGI groups).

## Acceptance Criteria

- [ ] AC-1: Intel IDXD + idxd-cdev test program: `iommu_sva_bind_device` succeeds; subsequent device DMA via PASID accesses user-allocated buffer without explicit dma_map.
- [ ] AC-2: PRI fault test: device DMA to non-present page → kernel handles fault → page faulted in → device retries successfully.
- [ ] AC-3: MMU notifier test: madvise(MADV_DONTNEED) on SVA-bound user range → IOMMU invalidates device pgtable → subsequent device access PRI-faults.
- [ ] AC-4: Process-exit cleanup: SVA-bound process killed → all PASID bindings auto-released; subsequent PASID alloc reuses freed PASID.
- [ ] AC-5: Cross-process isolation: SVA bind in process A; process B cannot DMA via A's PASID (defense in depth).
- [ ] AC-6: PRI fault rate-limit: synthetic flood of PRI requests → rate-limited; persistent over-rate disables PRI on device + WARN logged.
- [ ] AC-7: kselftest IOMMU SVA-related tests pass.

## Architecture

`Sva` lives in `kernel::iommu::sva::Sva`:

```
struct Sva {
  refcount: Refcount,
  dev: Arc<Device>,
  domain: Arc<IommuDomain>,
  mm: Arc<MmStruct>,
  pasid: u32,
  mmu_notifier: KBox<MmuIntervalNotifier>,
  released: AtomicBool,
}

struct IopfQueue {
  refcount: Refcount,
  name: KString,
  wq: Arc<Workqueue>,                  // per-queue workqueue ("iopf-")
  partial_groups: Mutex<Vec<KBox<IopfGroup>>>,  // groups still being assembled
  flush_completion: WaitQueue,
}

struct IopfGroup {
  prgi: u32,
  pasid: u32,
  dev: Arc<Device>,
  faults: Vec<KBox<IommuFaultEvent>>,
  is_last: bool,
  work: WorkStruct,
}

struct IommuFaultEvent {
  type_: FaultType,                    // PAGE_REQ / UNRECOVERABLE
  pasid: u32,
  prgi: u32,
  iova: u64,
  flags: FaultFlags,                   // READ / WRITE / EXEC / PRIV
  source_id: u32,
}
```

SVA bind `Device::sva_bind(dev, mm)`:
1. Verify dev supports PASID + PRI + ATS:
   - `pci_pasid_features(dev)` returns supported caps.
   - `pci_pri_features(dev)` returns PRI cap version.
   - `pci_ats_supported(dev)` returns ATS cap.
2. Per-vendor `iommu_ops->sva_bind(dev, mm)`:
   - Intel: `intel_iommu_sva_bind` allocs PASID via `intel_pasid_alloc_id`; install per-PASID PT root pointing at mm.pgd; flush PASID cache; enable PRI on device.
   - AMD: similar via `amd_iommu_bind_pasid`.
3. Allocate `Sva` handle: bind refcount, MMU notifier installed.
4. Install MMU notifier:
   - `mmu_interval_notifier_insert(&sva.mmu_notifier, &mm.notifier_subscriptions, 0, ULONG_MAX, &iommu_sva_mn_ops)`.
   - On invalidate-range-start: `iommu_ops->iotlb_sync(domain, iova, size)` → flush IOMMU TLB for affected range.
5. Return `iommu_sva` handle.

PRI fault flow `iopf_handle_event(dev, evt)`:
1. `evt = page-request fault descriptor extracted from per-IOMMU PRQ`.
2. Look up per-PASID `Sva` binding via `iommufd_object_get(...)`.
3. `iopf_group = iopf_partial_groups.find_or_create(evt.prgi, evt.pasid)`.
4. `iopf_group.faults.push(evt)`.
5. If `evt.is_last`:
   - `iopf_partial_groups.remove(iopf_group)`.
   - `queue_work(iopf_q.wq, &iopf_group.work)` → schedule handler.

Workqueue handler `iopf_handle_group_work(work)`:
1. `iopf_group = container_of(work, IopfGroup, work)`.
2. For each fault in iopf_group.faults:
   - `mm = iopf_group.sva.mm`.
   - `down_read(&mm->mmap_lock)`.
   - `vma = find_vma(mm, fault.iova)`.
   - If vma valid + fault.flags compatible with vma.vm_flags:
     - `vm_fault_t result = handle_mm_fault(vma, fault.iova, fault.flags & FAULT_FLAG_*, NULL)`.
     - response = SUCCESS if result is page-served else INVALID.
   - Else: response = INVALID.
   - `up_read(&mm->mmap_lock)`.
3. `iopf_complete_group(dev, iopf_group, status)`:
   - Per-vendor `iommu_ops->page_response(dev, msg)` sends PCIe-spec PRG-Response with PRGI + PASID + status.
4. Free iopf_group + faults.

MMU notifier callback `iommu_sva_mn_ops::invalidate(notifier, range)`:
1. `sva = container_of(notifier, Sva, mmu_notifier)`.
2. `iommu_ops->cache_invalidate_user(sva.domain, range.start, range.end - range.start, sva.pasid)`.
3. Wait for any in-flight device DMA to drain (vendor-specific).
4. Return 0 (allow host to proceed with pte change).

Process-exit cleanup: `mmu_notifier_release(mm)` is called from `__mmput`; per-bind notifier's `release` callback fires → `Device::sva_unbind(sva)` for each bound SVA on this mm.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `sva_no_uaf` | UAF | `Arc<Sva>` outlives all in-flight PRI faults; unbind drains pending faults before drop. |
| `pasid_no_collision` | UNIQUENESS | per-IOMMU PASID alloc bitmap-tracked; never two simultaneous SVAs with same PASID. |
| `iopf_group_no_dup` | INVARIANT | per-PRGI iopf_group unique per-device; faults coalesced before workqueue dispatch. |
| `mmu_notifier_no_uaf` | UAF | `mmu_interval_notifier` outlives mm via mm-mmgrab; release safe. |

### Layer 2: TLA+

`models/iommu/pasid_lifecycle.tla` (parent-declared): proves PASID alloc → bind → mm-notifier-invalidate → unbind → free state machine; mm exit + concurrent device DMA never produce use-after-free of pgtable; mmu-notifier range-invalidate flushed before underlying page reuse.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Device::sva_bind` post: returned handle has handle.pasid == per-iommu allocated PASID; handle.mm == provided mm; mmu_notifier installed | `Device::sva_bind` |
| `iopf_handle_event` post: every event eventually completes via `iopf_complete_group` (within bounded wallclock); fault rate-limit applies if exceeded | `iopf_handle_event` |
| MMU notifier invariant: every host pte change in [start, end) is preceded by IOMMU TLB invalidate for [start, end) on bound PASIDs | `iommu_sva_mn_ops::invalidate` |

### Layer 4: Verus/Creusot functional

`Device::sva_bind(dev, mm) → device DMA via PASID at user_va → handle_mm_fault if non-present → device retries → access succeeds` round-trip equivalence: SVA-bound device's DMA effectively shares mm.pgd with CPU; per-page semantics match CPU access.

## Hardening

(Inherits row-1 features from `drivers/iommu/00-overview.md` § Hardening.)

iommu-sva specific reinforcement:

- **Per-IOMMU PASID alloc bounded** by per-IOMMU MAX_PASID (typically 1M); per-bind PASID alloc in per-mm context; defense against PASID-exhaustion DoS.
- **Per-device PRI fault rate-limit** — default 100k/sec; over-rate triggers per-device WARN + PRI auto-disable; defense against device-bug-causing-PRI-flood DoS.
- **Per-fault response status validated** — only SUCCESS / INVALID / FAILURE allowed; defense against malformed response bytes corrupting device state.
- **Cross-process PASID isolation** — per-PASID PT root tied to specific mm; cross-mm PASID use rejected at iommu_ops layer.
- **MMU notifier ordering enforced** — invalidate-range-start happens BEFORE host pte change; defense against device DMA reading stale-but-physically-already-freed page.
- **Per-bind handle refcount + RCU** — concurrent unbind + in-flight fault never UAF.
- **Bind capability check** — dev MUST advertise PASID + PRI + ATS; missing any → -EOPNOTSUPP.
- **Per-process bind cap** — per-process maximum SVA bindings (default 64); defense against unbounded bind attack.
- **iopf workqueue priority** — high-priority workqueue ensures fault response within bounded latency; defense against device-side timeout while waiting.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-vendor PASID-table mgmt (covered in `intel-pasid.md` + `amd-pasid.md` future Tier-3s)
- Per-vendor IOMMU TLB-invalidate impl (covered in `intel-iommu.md` + `amd-iommu.md` Tier-3s)
- MMU notifier framework (covered in `mm/00-overview.md` parent Tier-2)
- handle_mm_fault (covered in `mm/page-fault.md` future Tier-3)
- 32-bit-only paths
- Implementation code
