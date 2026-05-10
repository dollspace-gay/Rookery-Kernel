# Tier-3: arch/x86/kvm/mmu/tdp_mmu.c — TDP MMU (EPT/NPT page-table walk + invalidation + concurrent fault handling)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/00-overview.md
upstream-paths:
  - arch/x86/kvm/mmu/tdp_mmu.c
  - arch/x86/kvm/mmu/tdp_iter.c
  - arch/x86/kvm/mmu/tdp_mmu.h
  - arch/x86/kvm/mmu/tdp_iter.h
  - arch/x86/kvm/mmu/spte.c
  - arch/x86/kvm/mmu/spte.h
-->

## Summary

The Two-Dimensional Paging MMU — modern KVM's EPT (Intel) / NPT (AMD) page-table walker. Default since Linux 5.10 (replaces the legacy shadow MMU for non-nested guests). Owns the second-level page table that translates guest-physical-address (GPA) → host-physical-address (HPA) on every guest memory access; the CPU walks both first-level (guest) and second-level (host TDP) page tables in hardware (the "two-dimensional" part). Critical hot path: any guest page fault that misses TDP pte traps to KVM here.

This Tier-3 covers `arch/x86/kvm/mmu/tdp_mmu.c` (~2000 lines) + `tdp_iter.c` (the page-table iterator) + `spte.c` (SPTE construction). The legacy shadow MMU lives in a separate Tier-3 (`x86-mmu-shadow.md`).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct tdp_iter` | per-walk page-table iterator state | `kernel::kvm::x86::TdpIter` |
| `kvm_tdp_mmu_init_vm(kvm)` / `_uninit_vm` | per-VM TDP MMU init/teardown | `Vm::tdp_mmu_init` / `_uninit` |
| `kvm_tdp_mmu_get_root(kvm, role)` / `_put_root` | acquire/release per-vm-role TDP root | `TdpMmu::get_root` / `_put_root` |
| `kvm_tdp_mmu_alloc_root(vcpu)` | allocate new TDP root for a vCPU | `TdpMmu::alloc_root` |
| `kvm_tdp_mmu_map(vcpu, fault)` | install TDP pte on guest page-fault | `TdpMmu::map_fault` |
| `kvm_tdp_mmu_zap_all(kvm)` / `_zap_invalidated_roots(kvm)` | zap all TDP ptes (e.g., for VM destroy or memslot delete) | `TdpMmu::zap_all` / `_zap_invalidated_roots` |
| `kvm_tdp_mmu_zap_leafs(kvm, root, start, end, can_yield, flush)` | zap leaf SPTEs in [start, end) GFN range | `TdpMmu::zap_leafs` |
| `kvm_tdp_mmu_unmap_gfn_range(kvm, range, flush)` | per-gfn-range unmap (called by mmu-notifier invalidate) | `TdpMmu::unmap_gfn_range` |
| `kvm_tdp_mmu_age_gfn_range(kvm, range)` | mark Accessed bits + return age (for LRU) | `TdpMmu::age_gfn_range` |
| `kvm_tdp_mmu_test_age_gfn(kvm, range)` | test Accessed without clearing | `TdpMmu::test_age_gfn` |
| `kvm_tdp_mmu_set_spte(kvm, ...)` | atomic SPTE update (cmpxchg-based) | `TdpMmu::set_spte` |
| `kvm_tdp_mmu_set_spte_gfn(kvm, range)` | mmu-notifier change-pte handler | `TdpMmu::set_spte_gfn` |
| `kvm_tdp_mmu_clear_dirty_slot(kvm, slot)` / `_clear_dirty_pt_masked` | clear dirty-bit for slot (live migration) | `TdpMmu::clear_dirty_slot` / `_clear_dirty_pt_masked` |
| `kvm_tdp_mmu_write_protect_gfn(kvm, slot, gfn, min_level)` | write-protect a single GFN | `TdpMmu::write_protect_gfn` |
| `kvm_tdp_mmu_walk_lockless(vcpu, addr, sptes, &leaf_level)` | lockless page-table walk (for instruction emulator) | `TdpMmu::walk_lockless` |
| `kvm_tdp_mmu_get_walk(vcpu, addr, sptes, &leaf_level)` | locked walk variant | `TdpMmu::get_walk` |
| `tdp_iter_start(iter, root, min_level, gfn)` | initialize walk iterator | `TdpIter::start` |
| `tdp_iter_next(iter)` | advance to next pte | `TdpIter::next` |
| `make_spte(vcpu, sp, slot, ...)` | build SPTE value from GFN + flags + access | `Spte::make` |
| `make_huge_page_split_spte(...)` | split a huge-page SPTE into 4K children | `Spte::make_split` |
| `is_shadow_present_pte(spte)` / `is_dirty_spte(spte)` / `is_writable_pte(spte)` | per-bit accessors | `Spte::is_*` |

## Compatibility contract

REQ-1: TDP MMU enabled by default (`kvm.tdp_mmu=true` cmdline, default Y) on EPT/NPT-capable HW; legacy shadow MMU only used for nested-virt guests + early HW without EPT.

REQ-2: Per-vCPU TDP root selected per `kvm_mmu_role` (per-mode 4-level vs 5-level paging, SMM vs non-SMM, guest-mode vs host-mode); roots cached + reused across vCPUs with same role.

REQ-3: Page-fault handler (`kvm_tdp_mmu_map`) installs SPTE atomically via cmpxchg; concurrent vCPU faults on the same GFN serialized correctly (one wins, others see the new SPTE on retry).

REQ-4: SPTE format byte-identical to Intel EPT / AMD NPT spec for every architectural bit (Present/W/R/X/A/D/PSE/IPAT/Memtype/etc.) — guest CPU walks see same bit semantics as upstream.

REQ-5: Huge-page support: 2MB / 1GB SPTEs auto-promoted when host-physical region is contiguous + not split-by-host-NUMA + `kvm.nx_huge_pages` policy permits. Auto-split on partial-write protection (live-migration dirty-tracking).

REQ-6: MMU-notifier integration: `kvm_tdp_mmu_unmap_gfn_range` invoked from host MMU-notifier (page migration / KSM / NUMA balancing); zaps overlapping SPTEs synchronously before returning.

REQ-7: Dirty-tracking: `kvm_tdp_mmu_clear_dirty_slot` walks all SPTEs in slot, clears Dirty bit + Write-Protects (manual mode) OR sets PML log destination (Intel PML mode) for live-migration page-copy.

REQ-8: TLB flush: every SPTE removal/clearing followed by per-vCPU TLB flush via `kvm_flush_remote_tlbs[_with_address]`; per-vCPU lazy flush via TLB-version-counter.

REQ-9: Lockless walk: `kvm_tdp_mmu_walk_lockless` traverses TDP pgtable without taking mmu_lock (RCU-protected); used by instruction emulator on hot path.

REQ-10: NX-Huge-Pages mitigation: per-iTLB-multihit-CVE workaround optionally splits all huge-page SPTEs into 4K + marks NX, then progressively reclaims back to huge after recovery period.

## Acceptance Criteria

- [ ] AC-1: Boot Linux guest under qemu KVM on Intel/AMD HW with EPT/NPT enabled; verify `dmesg | grep "TDP MMU"` shows TDP enabled.
- [ ] AC-2: 1GB huge-page test: guest with `transparent_hugepage=always` on host with 1GB hugetlbfs backing → guest sees PMD/PUD-level mappings.
- [ ] AC-3: Live-migration test: dirty-tracking via `KVM_GET_DIRTY_LOG` correctly identifies dirty pages; migrated guest resumes correctly.
- [ ] AC-4: NUMA-balancing test: host AutoNUMA migrates guest backing pages → MMU notifier fires → TDP SPTEs invalidated → guest re-faults + accesses migrated page.
- [ ] AC-5: Concurrent vCPU fault test: 8 vCPUs simultaneously faulting on same 2MB region → one wins SPTE installation, others see installed SPTE on retry; no spurious crashes.
- [ ] AC-6: NX-Huge-Pages test: with `kvm.nx_huge_pages=1`, executable-page split into 4K NX subpages observed; no iTLB multihit observed.
- [ ] AC-7: kvm-unit-tests `tools/testing/kvm/x86/mmu_*` subset passes.

## Architecture

`TdpMmu` lives in `kernel::kvm::x86::TdpMmu` (per-VM):

```
struct TdpMmu {
  roots: Mutex<Vec<Arc<TdpMmuRoot>>>,          // per-role TDP roots
  next_root_id: AtomicU64,
  zapped_roots_to_free: SpinLock<Vec<Arc<TdpMmuRoot>>>,
  pages: AtomicI64,                             // total TDP pgtable pages allocated
  pages_for_root: PerCpu<i64>,
}

struct TdpMmuRoot {
  refcount: Refcount,
  role: KvmMmuRole,
  hpa: u64,                                     // physical addr of root page
  mmu_role: KvmMmuRole,
  is_invalidated: AtomicBool,
  shadow_root_level: u32,                       // 4 or 5 (4-level vs 5-level paging)
  page_table_paths: KBox<TdpPgtablePages>,      // per-level pgtable allocations
}

struct TdpIter {
  next_last_level_gfn: u64,
  yielded_gfn: u64,
  pt_path: [NonNull<TdpPte>; PT64_ROOT_MAX_LEVEL],
  sptep: NonNull<TdpPte>,
  gfn: u64,
  root_level: u32,
  min_level: u32,
  level: u32,
  pte_index: u32,
  walk_index: u32,
  old_spte: u64,
  iterator_state: TdpIterState,
}
```

Page-fault handler `TdpMmu::map_fault(vcpu, fault)`:
1. Lookup or alloc per-vcpu mmu role.
2. `TdpMmu::get_root(kvm, role)` → existing or new root.
3. `TdpIter::start(iter, root, fault.req_level, fault.gfn)` → walk to fault GFN at requested level.
4. Loop:
   - `TdpIter::next(iter)`: advance to leaf or split point.
   - At leaf: `Spte::make(...)` build new SPTE (Present + Read + Write|RO + AccessFlags + memory-type + dirty-track bits + NX|X + huge-page-split-needed bit).
   - Atomic compare-exchange `iter.sptep` from `iter.old_spte` to `new_spte`:
     - On success: SPTE installed; if `iter.old_spte` was present, flush TLB.
     - On failure: another vCPU installed concurrently; re-read + retry.
5. If split was needed (huge → 4K): `Spte::make_split(...)` build split-SPTE structure; atomic publish.

MMU-notifier `TdpMmu::unmap_gfn_range(kvm, range, flush)`:
1. For each TdpMmuRoot in `kvm->tdp_mmu->roots`:
   - `TdpIter::start(iter, root, PG_LEVEL_4K, range.start)`.
   - Loop until `iter.gfn >= range.end`:
     - At leaf: `TdpMmu::set_spte(kvm, iter, 0)` (atomic clear).
     - Increment flush counter.
2. Flush TLB if any SPTEs were zapped.

Dirty-tracking `TdpMmu::clear_dirty_slot(kvm, slot)`:
1. For each TdpMmuRoot:
   - `TdpIter::start(iter, root, PG_LEVEL_4K, slot.base_gfn)`.
   - Loop while `iter.gfn < slot.base_gfn + slot.npages`:
     - At leaf with Dirty bit set: atomic CAS to clear D-bit + Write-protect (manual mode) OR enable PML logging.
2. Flush TLB.

NX-Huge-Pages mitigation: when `kvm.nx_huge_pages=1`, on TDP page-fault for executable region with huge-mapping candidate, split into 4K + mark NX on each (defense against iTLB multihit CVE-2018-12126 family). After recovery period, walker re-promotes to huge if execute-bits cleared.

Concurrent-fault correctness: each SPTE update is a single-instruction cmpxchg (atomic per-pte); RCU-protected page-table-page lifetime ensures stale walker pointers safe; mmu_lock taken only for batch operations (zap_all, root alloc/free).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `iter_no_oob` | OOB | `TdpIter::next` walks bounded by root_level descent + per-level pte_index < 512; never indexes past page-table page. |
| `spte_atomic_update` | ATOMICITY | `TdpMmu::set_spte` cmpxchg ensures no torn SPTE read; concurrent updaters serialized by hardware. |
| `root_no_uaf` | UAF | `Arc<TdpMmuRoot>` outlives any in-flight TdpIter referencing it. |
| `page_table_no_uaf` | UAF | RCU-deferred free of pgtable pages waits for grace period after last reader. |

### Layer 2: TLA+

`models/kvm/spte_lifecycle.tla` (parent-declared): proves SPTE state machine — non-present → present → R/W → dirty → flushed → unmap; concurrent host MMU notifier invalidation + guest fault never produce a stale SPTE pointing to a freed pfn.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `TdpMmu::map_fault` post: SPTE at `iter.sptep` is either `0` (concurrent zap won) OR contains a present mapping to `fault.pfn` with correct access bits | `TdpMmu::map_fault` |
| `TdpMmu::unmap_gfn_range` post: every SPTE in [start, end) either zero OR was already zero before invocation | `TdpMmu::unmap_gfn_range` |
| `TdpMmu::clear_dirty_slot` post: every SPTE in slot has D-bit clear + W-bit clear (manual mode) OR PML enabled (Intel PML mode) | `TdpMmu::clear_dirty_slot` |
| MMU-notifier ordering: every host pte change visible to KVM before the corresponding guest page is reused (parent's spte_lifecycle.tla is the proof) | `TdpMmu::unmap_gfn_range` |

### Layer 4: Verus/Creusot functional

`TdpMmu::map_fault(vcpu, fault) → guest accesses GPA` round-trip equivalence: post-map, guest CPU walking EPT/NPT for `fault.gpa` reaches the installed SPTE producing `fault.pfn` with `fault.access` permissions. Encoded as Verus model: `forall vcpu fault. installed_spte(vcpu, fault.gpa) maps to (fault.pfn, fault.access)`.

## Hardening

(Inherits row-1 features from `virt/kvm/00-overview.md` § Hardening.)

tdp-mmu-specific reinforcement:

- **Atomic SPTE updates** — every per-pte mutation via cmpxchg; never torn read by guest CPU walker. Defense against guest seeing inconsistent EPT entries.
- **RCU-protected pgtable lifetime** — pgtable pages freed only after RCU grace period; stale TdpIter walker pointers safe across concurrent zaps.
- **MMU-notifier completeness** — every host pte change zaps overlapping SPTEs before returning to MM (Layer-2 spte_lifecycle.tla is the proof). Defense against guest reading freed page.
- **NX-Huge-Pages mitigation default-on** for affected Intel CPUs (CVE-2018-12126 / -12130 / -12127 / -11091 / -12376 family). Splits exec-mapped huge pages to 4K NX subpages; recovery period progressively re-promotes.
- **TLB flush before SPTE reuse** — every cleared SPTE followed by per-vcpu (or remote-tlbs) flush before pgtable page reused for another GFN. Defense against stale TLB entry from prior guest.
- **Per-VM page-table-page count cap** — `kvm.max_tdp_mmu_pages` soft cap; defense against per-VM TDP page-table memory exhaustion.
- **SPTE write-protection during dirty-tracking** — atomic CAS sequence ensures no concurrent vCPU writes lost during tracking enable.
- **Lockless walk RCU-protected** — `kvm_tdp_mmu_walk_lockless` reads pgtable under rcu_read_lock; pgtable-page free deferred via call_rcu.
- **PML overflow handling** — Intel PML buffer fills → vmexit → KVM drains buffer + re-arms; never silent log loss.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Legacy shadow MMU (covered in `virt/kvm/x86-mmu-shadow.md` future Tier-3)
- Page-track callbacks (covered in `virt/kvm/x86-mmu-notifier.md` future Tier-3)
- Per-vendor VMX EPT specifics (covered in `virt/kvm/vmx-core.md` future Tier-3)
- Per-vendor SVM NPT specifics (covered in `virt/kvm/svm-core.md` future Tier-3)
- Nested TDP (covered in `virt/kvm/vmx-nested.md` + `svm-nested.md` future Tier-3s)
- 32-bit-only paths
- Implementation code
