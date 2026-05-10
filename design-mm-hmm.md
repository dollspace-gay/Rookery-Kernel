---
title: "Tier-3: mm/hmm.c — HMM (Heterogeneous Memory Management) — GPU / accelerator mirroring"
tags: ["tier-3", "mm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

HMM (Heterogeneous Memory Management) provides shared per-(CPU-host, device-GPU) virtual-memory: device drivers (NVidia GPU, AMD GPU, etc.) **mirror** CPU process page tables into device-MMU so CPU/GPU share virtual addresses. Per-`hmm_range_fault` driver invokes to populate device PTEs for a range. Per-`mmu_interval_notifier` device gets per-mm-write notifications to invalidate device cache + retry. Per-PG_DEVICE pages live in device-private memory; migrate-back to host on CPU access. Critical for: heterogeneous compute (CUDA-managed, ROCm, OpenCL SVM), unified-virtual-memory (UVM).

This Tier-3 covers `hmm.c` (~895 lines).

### Acceptance Criteria

- [ ] AC-1: Driver registers mmu_interval_notifier: ops.invalidate ready.
- [ ] AC-2: hmm_range_fault on present range: pfns populated.
- [ ] AC-3: hmm_range_fault with REQ_FAULT on swapped-out: handle_mm_fault triggered; pfns retrievable.
- [ ] AC-4: CPU munmap → invalidate fires; driver retries; range invalid.
- [ ] AC-5: Per-write-fault: REQ_WRITE pfns populated writable.
- [ ] AC-6: Per-read-only PTE + REQ_WRITE: -EFAULT.
- [ ] AC-7: migrate_vma_setup + pages + finalize: per-folio migrated to device.
- [ ] AC-8: CPU access to migrated device-private page: faulted-back to host.
- [ ] AC-9: nouveau / amdgpu HMM integration: SVM workloads execute.

### Architecture

Per-range descriptor:

```
struct HmmRange {
  notifier: &MmuIntervalNotifier,
  notifier_seq: u64,
  start: u64,
  end: u64,
  hmm_pfns: *u64,                                 // output pfn array
  default_flags: u64,                             // REQ_FAULT etc.
  pfn_flags_mask: u64,
  dev_private_owner: *void,
}
```

Per-mmu_interval_notifier:

```
struct MmuIntervalNotifier {
  interval_tree: IntervalNode,                    // start/end on tree
  ops: *MmuIntervalNotifierOps,                   // invalidate callback
  mm: *MmStruct,
  invalidate_seq: u64,
}

trait MmuIntervalNotifierOps {
  fn invalidate(notifier, range_info: &MmuNotifierRange, seq: u64) -> bool;
}
```

`Hmm::range_fault(range) -> Result<()>`:
1. mm = range.notifier.mm.
2. mmap_read_lock(mm).
3. /* Walk per-pfn in [start, end) */
4. for addr in [range.start, range.end) step PAGE_SIZE:
   - vma = find_vma(mm, addr).
   - if !vma: pfns[i] = HMM_PFN_ERROR; continue.
   - if range.default_flags & REQ_FAULT ∧ !pte_present:
     - ret = handle_mm_fault(vma, addr, FAULT_FLAG_*).
     - if ret & VM_FAULT_ERROR: pfns[i] = HMM_PFN_ERROR.
   - pte_present checks; populate pfns[i] = pfn | flags.
5. mmap_read_unlock(mm).

`Mmu::interval_notifier_insert(notifier, mm, start, end, ops) -> Result<()>`:
1. notifier.mm = mm.
2. notifier.ops = ops.
3. interval_tree_insert(&notifier.interval_tree, &mm.notifier.itree).
4. /* Per-mmput → mm_subscribe_invalidate_range_start/end */
5. Ok.

`Mig::vma_setup(args) -> Result<()>`:
1. /* For each pfn in src range: identify pages; capture state */
2. ...

### Out of Scope

- mm/00-overview (Tier-2)
- mm/migrate_device.c (covered separately if expanded)
- GPU drivers (drivers/gpu/drm/{nouveau, amdgpu}; covered separately)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct hmm_range` | per-(start, end, mm, pfns) range | `HmmRange` |
| `hmm_range_fault()` | driver entry to walk + fault | `Hmm::range_fault` |
| `hmm_pfns_to_map` | per-range output array | shared |
| `HMM_PFN_*` flags | per-pfn permission/state | UAPI |
| `struct mmu_interval_notifier` | per-range CPU-mm notifier | `MmuIntervalNotifier` |
| `mmu_interval_notifier_insert()` | per-driver register notifier | `Mmu::interval_notifier_insert` |
| `mmu_interval_notifier_remove()` | per-driver unregister | `Mmu::interval_notifier_remove` |
| `migrate_vma_setup()` | per-folio migrate-prepare (device ↔ host) | `Mig::vma_setup` |
| `migrate_vma_pages()` | per-folio migrate-pages | `Mig::vma_pages` |
| `migrate_vma_finalize()` | per-folio post-migrate | `Mig::vma_finalize` |
| `MIGRATE_PFN_*` flags | per-PFN migrate-state | shared |
| `struct dev_pagemap` | per-device memory region | `DevPagemap` |
| `MEMORY_DEVICE_PRIVATE` | per-device-private memory type | shared |

### compatibility contract

REQ-1: Per-hmm_range:
- start, end: virtual range.
- pfns: caller-provided u64 array.
- notifier: per-mmu_interval_notifier.
- timestamp: per-validity for retry.
- flags: HMM_PFN_REQ_FAULT / WRITE / READ.

REQ-2: HMM_PFN_* values:
- HMM_PFN_VALID: pfn populated.
- HMM_PFN_WRITE: writable.
- HMM_PFN_ERROR: lookup failed.
- HMM_PFN_REQ_FAULT: driver wants fault-in.
- HMM_PFN_REQ_WRITE: driver wants write-fault.

REQ-3: hmm_range_fault(range):
- /* Validate range within VMA */
- Walk per-pfn:
  - if !pte_present ∧ HMM_PFN_REQ_FAULT: handle_mm_fault.
  - if pte_present: extract pfn.
  - if writable ∧ HMM_PFN_REQ_WRITE: handle_mm_fault for write.
  - if write-required + read-only: -EFAULT.
- Per-pfn output flags.

REQ-4: Per-mmu_interval_notifier:
- driver: mmu_interval_notifier_insert(notifier, mm, start, end, ops).
- per-mm-change in range: ops.invalidate(notifier, range_info, seq).
- driver typically: stop device-tasks; unmap device-PTEs; on retry: re-fault.

REQ-5: Per-retry:
- Per-driver kernel-context: hold range_lock.
- Per-fault: invoke hmm_range_fault.
- If invalidate received: drop range_lock; retry.
- Pattern: mmu_interval_set_seq + read-side.

REQ-6: Per-device-private-pages:
- alloc'd via memremap_pages with MEMORY_DEVICE_PRIVATE.
- Per-page has ZONE_DEVICE memory-type.
- Per-CPU access faults → migrate-back via vma.vm_ops.fault.

REQ-7: migrate_vma_setup / pages / finalize:
- Per-VMA-range migrate between host ↔ device-private.
- setup: identify source pfns; isolate.
- pages: per-pfn copy + update PTEs.
- finalize: post-migrate cleanup.

REQ-8: Per-CPU-access-to-device-private:
- Per-fault on device-page: dev_pagemap.ops.page_free OR migrate-back.
- Driver implements .migrate_to_ram callback.

REQ-9: Per-userspace API:
- No direct syscall — used by GPU drivers (nouveau, amdgpu, kfd).
- Per-driver ioctl SVM-like.

REQ-10: Per-namespace:
- HMM tied to mm-struct; no per-net.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pfn_array_sized` | INVARIANT | range.hmm_pfns.len() ≥ (range.end - range.start) / PAGE_SIZE. |
| `notifier_seq_matches_on_success` | INVARIANT | hmm_range_fault success ⟹ caller checks notifier_seq still valid. |
| `mmap_read_lock_held` | INVARIANT | per-range_fault: mmap_read_lock held. |
| `interval_tree_consistent` | INVARIANT | per-mmu_notifier interval-tree balanced. |

### Layer 2: TLA+

`mm/hmm.tla`:
- Per-range_fault + per-notifier-invalidate retry-loop.
- Properties:
  - `safety_per_pfn_invalidated_on_mm_change` — per-CPU-mm-mutation in range ⟹ driver notified.
  - `safety_no_stale_pfn_used` — per-driver checks notifier_seq before consuming pfns.
  - `liveness_handler_eventually_completes` — per-driver retry-loop terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Hmm::range_fault` post: pfns populated for valid range; HMM_PFN_ERROR per-failure | `Hmm::range_fault` |
| `Mmu::interval_notifier_insert` post: notifier in mm interval-tree | `Mmu::interval_notifier_insert` |
| `Mig::vma_setup` post: per-pfn identified for migrate | `Mig::vma_setup` |

### Layer 4: Verus/Creusot functional

`Per-device GPU page-table mirroring + per-mm-change invalidation + device-private memory migration` semantic equivalence: per-Documentation/mm/hmm.rst.

### hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening.)

HMM-specific reinforcement:

- **Per-notifier_seq checked by driver** — defense against per-stale-pfn use.
- **Per-mmu_interval_notifier interval-tree balanced** — defense against per-notifier-explosion.
- **Per-fault-in invokes handle_mm_fault** — defense against per-driver bypassing fault-path.
- **Per-device-private vma.vm_ops.fault** — defense against per-CPU-access reading garbage.
- **Per-dev_pagemap refcount** — defense against per-region UAF.
- **Per-MEMORY_DEVICE_PRIVATE not addressable from CPU** — defense against per-mistaken CPU access.
- **Per-migrate-back race-free** — defense against per-concurrent CPU+GPU access torn.
- **Per-CAP_SYS_ADMIN for GPU drivers (separate)** — defense against per-unprivileged HMM-use.
- **Per-mmap_read_lock during range_fault** — defense against per-VMA mutation race.
- **Per-pfns array bounds-checked** — defense against per-out-of-range pfn-write.

