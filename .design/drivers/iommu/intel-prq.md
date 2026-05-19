# Tier-3: drivers/iommu/intel/prq.c — PCIe PRI (Page-Request Interface) request-queue + per-PASID fault dispatch

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/iommu/intel-iommu.md
upstream-paths:
  - drivers/iommu/intel/prq.c
  - drivers/iommu/intel/iommu.h (PRQ register defs)
  - drivers/iommu/iommu-sva.c (SVA frontend that consumes PRI faults)
  - drivers/iommu/iommufd/eventq.c (iommufd-PRI event delivery)
-->

## Summary

PCIe PRI (Page Request Interface) lets a device with PASID-capable ATS request the IOMMU to demand-page in a missing translation — the IOMMU writes a PRQ entry into a per-IOMMU page-request queue, the kernel's PRQ thread reads entries, walks the host MM via `handle_mm_fault`, then sends a Page-Response back to the device. Without PRI, SVA-capable accelerators (Intel DSA, GPU shared-virtual-memory, IDXD) cannot demand-page; they would have to pin the entire host VA range up front.

This Tier-3 covers `drivers/iommu/intel/prq.c` (~396 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct page_req_dsc` | per-PRQ-entry 32-byte descriptor | `drivers::iommu::intel::PageReqDsc` |
| `intel_iommu_enable_prq(iommu)` | per-IOMMU init: alloc queue + irq | `Iommu::enable_prq` |
| `intel_iommu_finish_prq(iommu)` | per-IOMMU teardown | `Iommu::finish_prq` |
| `intel_prq_thread_handler(...)` | per-IOMMU PRQ kthread main loop | `Iommu::prq_thread` |
| `intel_iommu_drain_pasid_prq(...)` | per-PASID drain on detach | `Iommu::drain_pasid_prq` |
| `prq_event_thread(...)` | per-event dispatch + handle_mm_fault | `PrqHandler::process_event` |
| `intel_svm_drain_prq(...)` | SVA detach-time drain | `Iommu::svm_drain_prq` |
| `iommu_report_device_fault(...)` (iommu-core) | dispatch to per-device fault handler | `IommuCore::report_device_fault` |
| `dmar_response_pri(...)` | submit Page-Response to device via QI | `Iommu::response_pri` |
| `qi_submit_sync(iommu, &desc, count)` | invalidation-queue submit (used for PRI response) | `Iommu::qi_submit` |

## Compatibility contract

REQ-1: Per-IOMMU PRQ ring buffer:
- 32-byte entries (`page_req_dsc`).
- Default 4KB queue (128 entries); programmable via PQA register.
- Wraps via head/tail pointers in PQH/PQT MMIO registers.

REQ-2: Per-PRQ-entry layout (`page_req_dsc`):
- type (PASID-PRESENT bit + PRG-RESP-NEEDED bit).
- pasid (20-bit guest/host PASID).
- addr (host-virtual addr requested; 4KB aligned).
- requester-id (PCI BDF of source device).
- page-req-group-index (PRG index for response correlation).
- read/write/exec flags.

REQ-3: PRQ MSI interrupt allocation:
- Per-IOMMU MSI allocated for PRQ delivery; vector → `prq_event_thread`.
- IRQ thread context to allow `handle_mm_fault` (cannot run in IRQ context).

REQ-4: Per-event processing flow:
- Read PRQ head; for each entry from head to tail:
  - Validate PASID present + valid.
  - Look up per-PASID `iommu_sva` binding to find target mm.
  - Call `handle_mm_fault(mm, vma, addr, flags)` (host MM page-fault handler).
  - On success: device should retry; send Page-Response = SUCCESS.
  - On failure: send Page-Response = FAILURE.
  - Update head pointer (advance past consumed entry).

REQ-5: Page-Response submit via QI:
- Compose QI descriptor type = page-response.
- Fields: requester-id, PASID, PRG-index, response-code (SUCCESS/INVALID/FAILURE).
- `qi_submit_sync(iommu, desc)` to QI; wait for completion (wait-descriptor + IWC bit).

REQ-6: Per-PASID drain:
- On SVA-bind detach OR PASID release: ensure no in-flight PRQ entry references that PASID.
- `drain_pasid_prq(iommu, pasid)`:
  - Walk PRQ from head to tail; per-matching entry: emit RESPONSE=FAILURE.
  - Wait for tail to advance past last-pre-drain entry.
  - Submit additional QI flush to ensure all in-flight invalidated.

REQ-7: PRQ-overflow handling:
- IOMMU PFOR bit (Page-Fault Overflow Recoverable) set when queue full; IOMMU silently drops further requests.
- Driver: clear PFOR; rate-limit log; per-device fault may need handler to retry.
- On PFNR (Page-Fault Non-Recoverable): IOMMU may have lost requests; driver per-device-context-flush.

REQ-8: PRQ thread NUMA affinity: per-IOMMU PRQ kthread bound to per-IOMMU NUMA node (from RHSA).

REQ-9: PRQ → iommufd dispatch:
- For VFIO+iommufd, per-PRQ-entry needs userspace dispatch via iommufd-eventq fd (covered in iommufd-eventq Tier-3).
- PRQ handler calls `iommu_report_device_fault(dev, &fault_event)`; iommufd-eventq registered as handler routes to userspace.

REQ-10: PRQ for guest-PASID (vSVA):
- Guest IOVA fault → IOMMU PRQ entry with guest-PASID.
- Driver translates guest-PASID → host-PASID via vIOMMU mapping; injects fault into guest PRQ.
- Used by IOMMUFD viommu (covered in iommufd-viommu Tier-3).

REQ-11: PRQ + KVM guest passthrough:
- For SR-IOV passthrough where guest manages its own SVA: PRQ entry from VF gets pasid that is host-shadow of guest's pasid.
- vIOMMU emulation propagates entry to guest's vIOMMU PRQ.

## Acceptance Criteria

- [ ] AC-1: Boot host with VT-d PRI-cap CPU (Sapphire Rapids+) + DSA accelerator: `dmesg | grep "PRQ"` shows per-IOMMU PRQ init.
- [ ] AC-2: Intel DSA / IDXD test: workqueue with shared-virtual-memory; user submits work descriptor pointing to non-resident page; PRQ fires; page faulted in; descriptor completes.
- [ ] AC-3: GPU shared-virtual-memory test: i915 with PASID + non-resident user buffer; first GPU access triggers PRQ; gpu retry succeeds.
- [ ] AC-4: PRQ overflow recovery: synthetic stress producing >128 in-flight requests; driver detects PFOR, flushes, recovers; no permanent device hang.
- [ ] AC-5: SVA detach drain: process exit while DSA descriptor in-flight; PRQ drain completes within 500ms; no PRQ entries left referencing freed mm.
- [ ] AC-6: Per-PASID isolation: 2 processes both using DSA; PRQ for proc-A handled with proc-A's mm; PRQ for proc-B handled with proc-B's mm; no cross-contamination.
- [ ] AC-7: Page-Response correctness: success/failure responses match outcome of handle_mm_fault.
- [ ] AC-8: vSVA test (iommufd-viommu): guest with vDSA; guest-PASID PRQ injected to guest vIOMMU.

## Architecture

`Iommu` per-IOMMU adds PRQ fields:

```
struct Iommu {
  ...
  prq_supported: bool,                       // ECAP.PRS bit
  prq: Option<KArc<Prq>>,
}

struct Prq {
  ring: KBox<[PageReqDsc; PRQ_ENTRIES]>,    // 128-entry default
  ring_phys: u64,                            // physical addr programmed to PQA register
  thread: KThread,                           // per-IOMMU kthread
  msi_irq: u32,
}

struct PageReqDsc {
  type_pasid: u32,                           // {type[31:28], pasid[27:8], pasid_present, prg_resp_needed, ...}
  addr: u64,
  requester_id: u16,                         // PCI BDF
  prg_index: u16,
  flags: u32,                                // r/w/x bits
  reserved: u64,
}
```

`Iommu::enable_prq` flow:
1. Allocate ring (page-aligned, contiguous).
2. Init head=0, tail=0.
3. Program PQA register = ring_phys | log2(size).
4. Allocate MSI for PRQ delivery via intel-IR (alloc IRTE-handle).
5. Spawn kthread `intel_prq_thread_<iommu_id>`; bind to RHSA NUMA node.
6. Enable PRQ via GCMD register; wait for GSTS.PRQE bit.

`Iommu::prq_thread` main loop:
1. Wait on PRQ-MSI eventfd (per-IOMMU).
2. Read PQH (head); read PQT (tail).
3. For each entry from head to tail:
   - `process_event(&prq.ring[idx])`:
     - Decode pasid + addr + flags.
     - Look up `iommu.pasid_table[pasid]` → `iommu_sva` binding.
     - If binding has `dev_iommu.flags & IOMMU_DEV_FEAT_SVA` user mm:
       - `handle_mm_fault(mm, vma_for_addr, addr, fault_flags)`.
     - Else if iommufd-eventq registered: dispatch fault to iommufd userspace.
     - Compose `qi_desc` for page-response with result code.
     - `qi_submit_sync(iommu, &desc, 1)`.
   - Advance head.
4. Update PQH register.
5. Repeat.

`Iommu::drain_pasid_prq(pasid)`:
1. Acquire prq.lock.
2. Read PQT (tail); store as drain-target.
3. Loop until PQH == drain-target:
   - Per-entry: if entry.pasid == target pasid: send response=FAILURE.
   - Else process normally.
4. Submit additional QI page-selective-invalidation for pasid.
5. Wait for QI completion.

`Iommu::response_pri(req_id, pasid, prg_index, code)`:
1. Compose QI desc: type=PRG_RESP, requester_id, pasid, prg_index, code.
2. `qi_submit_sync(iommu, &desc, 1)`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `prq_ring_no_oob` | OOB | per-PRQ-entry indexed by head/tail mod PRQ_ENTRIES; ring access bounded. |
| `prq_head_advance_monotonic` | INVARIANT | per-iter PQH advances by ≥1 per consumed entry; never moves backward. |
| `pasid_lookup_safe` | UAF | per-PRQ-entry pasid lookup via KArc; defense against pasid-detach-during-lookup. |
| `qi_submit_no_double_free` | UAF | qi-desc allocated stack-local; defense against use-after-stack-frame. |

### Layer 2: TLA+

`drivers/iommu/intel/prq_state.tla` models per-IOMMU PRQ states:
- Per-entry state ∈ {Empty, Pending, Processing, Responded, Drained}.
- Per-IOMMU queue head/tail invariants: head ≤ tail mod ring-size.
- `liveness_eventually_processed` — every Pending entry eventually transitions to Responded or Drained.
- `safety_no_resp_without_processing` — Responded only reachable via Processing path.
- `safety_drain_terminates` — Drain (per-pasid) eventually reaches state where no entry has target pasid.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Iommu::enable_prq` post: ring allocated; PQA register programmed; thread running | `Iommu::enable_prq` |
| `Iommu::prq_thread` per-iteration: head advances by exactly count-consumed | `Iommu::prq_thread` |
| `Iommu::drain_pasid_prq` post: no PRQ entry with that pasid pending | `Iommu::drain_pasid_prq` |
| Per-entry processing ↔ per-entry response (1:1 correspondence) | `PrqHandler::process_event` |
| Per-PRQ thread NUMA-affinity matches RHSA | `Iommu::prq_thread` setup |

### Layer 4: Verus/Creusot functional

`Device DMA-fault → PRQ-entry → handle_mm_fault → page-response` round-trip equivalence: device sees retry-success iff target VA was faultable in target mm; sees retry-failure otherwise.

## Hardening

(Inherits row-1 features from `drivers/iommu/intel-iommu.md` § Hardening.)

PRQ-specific reinforcement:

- **Per-PRQ ring in IOMMU-mapped IO-coherent memory** — defense against torn-write of 32-byte entry by IOMMU vs CPU concurrent read.
- **Per-PASID drain on SVA detach** — defense against in-flight fault referencing freed mm.
- **PFOR overflow detected + recovered** — defense against silent PRQ stall.
- **Per-entry PASID validity check** — defense against IOMMU writing invalid PASID into queue.
- **Per-event handle_mm_fault gated by per-PASID mm validity** — defense against use-after-mm-free during fault handling.
- **Per-IOMMU PRQ MSI bound to thread context** — defense against PRQ-irq-handler running in atomic ctx + crashing in handle_mm_fault.
- **Per-page-response QI completion-wait bounded** (timeout ~1s) — defense against stuck QI causing PRQ thread hang.
- **Per-PASID rate-limit on PRQ overflow log** — defense against malicious device flooding dmesg via PRQ floods.
- **iommufd-eventq dispatch validated** — only registered eventq receives faults; defense against cross-process leak.
- **Per-PRQ-thread NUMA-bound** — defense against cross-NUMA cacheline ping-pong on hot fault path.

## Grsecurity/PaX-style Reinforcement

Hardened-policy supplement above baseline `## Hardening`. The PRQ is the only IOMMU subsystem where DMA-capable devices drive host-MM page-fault handling synchronously — Rookery therefore treats PRQ entry validation, drain ordering, and rate-limiting as TCB-critical.

- **PAX_USERCOPY** on PRQ fault info exposed to iommufd-eventq userspace consumers.
- **PAX_KERNEXEC** on `prq_thread`, `process_event`, `response_pri`, and QI submit (RO post-init).
- **PAX_RANDKSTACK** on `prq_thread` per-iteration entry into `handle_mm_fault` callchain.
- **PAX_REFCOUNT** on `Prq`, per-PASID mm refs, and SVA handle lifetimes.
- **PAX_MEMORY_SANITIZE** zeroes consumed PRQ entries after head-advance.
- **PAX_UDEREF** on any user-pointer fields surfaced from PRQ to userspace eventq.
- **PAX_RAP/kCFI** on fault-handler indirect dispatch (`report_device_fault` ops).
- **GRKERNSEC_HIDESYM** hides PRQ ring base, per-IOMMU thread pointers, and PASID-table refs.
- **GRKERNSEC_DMESG** restricts PFOR/PFNR overflow logs to CAP_SYSLOG.
- **CAP_SYS_ADMIN strict** on `enable_prq` runtime knobs and PRQ debugfs introspection.
- **GRKERNSEC_DMA strict-mode** — PRQ-capable devices default-blocked from SVA until full PRI capability re-validated.
- **VT-d PRQ rate-limit** per source-id + per-PASID; defense against malicious device PRI-storms saturating mm-fault path.
- **ATS/PASID gating** — devices without verified PCIe PRI capability bits refused PRQ enablement.
- **DMAR ACPI signature verify** before honoring ECAP.PRS bit.
- **VFIO-compat refused** for PRI-using devices under hardened policy; iommufd-native eventq only.
- **Per-entry source-id sanity check** against registered device list; spoofed BDFs dropped + ratelimited WARN.

Rationale: PRQ couples a hostile device's DMA stream directly to host kernel mm-fault handling. Hardened Rookery enforces every PCIe-spec validation upstream treats as optional and refuses every compatibility fallback that would let an unverified device speak PRI.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Generic IOMMU API (covered in `iommu-core.md` Tier-3)
- Intel SVA bind glue (covered in `intel-svm.md` Tier-3)
- iommu-sva framework (covered in `iommu-sva.md` Tier-3)
- iommufd eventq frontend (covered in `iommufd-eventq.md` Tier-3)
- iommufd vIOMMU PRQ injection (covered in `iommufd-viommu.md` Tier-3)
- AMD/ARM equivalents (different drivers)
- Implementation code
