---
title: "Tier-3: arch/x86/kvm/mmu/mmu.c (zap subset) — KVM MMU SPTE invalidation (kvm_mmu_invalidate + zap_gfn_range + zap_all + mmu_valid_gen)"
tags: ["tier-3", "virt-kvm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

KVM MMU SPTE invalidation ("zap") is the per-VM mechanism for clearing per-vCPU shadow / TDP page-tables when host-MM-state changes (mmu_notifier callbacks: invalidate_range_start / _end / change_pte / aging) or per-VM events (memslot delete, CR3 write, INVPCID, INVLPGB). Per-zap walks SPTEs covering the affected range; clears + flushes per-page TLB. KVM uses generation-counter (kvm.arch.mmu_valid_gen) for "fast-zap-all" — bumping the gen invalidates entire MMU; per-vmenter sees stale-gen + rebuilds. mmu_notifier hooks let host MM notify guest of page-mapping changes (e.g., swap-out, migration, COW). Critical for: correctness under host-side memory pressure + KVM-side memslot reconfiguration.

This Tier-3 covers MMU zap functions in `arch/x86/kvm/mmu/mmu.c` (~600-800 lines) + `tdp_mmu.c` analog.

### Acceptance Criteria

- [ ] AC-1: Memslot delete: per-memslot SPTEs zapped; subsequent guest access #PF.
- [ ] AC-2: Host swap-out: mmu_notifier fires; per-gfn SPTE invalidated; guest re-fault re-pins.
- [ ] AC-3: Guest INVLPG: per-vCPU TLB flushed for given GVA.
- [ ] AC-4: Live migration: per-VM zap_all_fast → background worker zaps; minimal vmexit interruption.
- [ ] AC-5: THP collapse: 512×4KB SPTEs collapsed to single 2MB SPTE.
- [ ] AC-6: Multi-vCPU TLB flush: per-vCPU IPI; all flushes complete before zap-return.
- [ ] AC-7: PCID INVPCID: per-PCID flush; other PCIDs unaffected.
- [ ] AC-8: mmu_valid_gen rollover: 256 fast-zaps; full-zap triggered.
- [ ] AC-9: kvm-unit-tests `mmu_zap` test passes.
- [ ] AC-10: Stress: 10K mmu_notifier events/sec; no UAF; throughput maintained.

### Architecture

`Mmu::zap_gfn_range(kvm, start, end)`:
1. lock(kvm.mmu_lock).
2. For each memslot:
   - per-gfn from start..end clamped to slot-range:
     - rmap_head := __gfn_to_rmap(slot, gfn).
     - kvm_zap_all_rmap_sptes(kvm, rmap_head).
3. kvm_flush_remote_tlbs(kvm).
4. unlock.

`Mmu::zap_all_fast(kvm)`:
1. lock(kvm.mmu_lock).
2. kvm.arch.mmu_valid_gen++.
3. queue_work(kvm.arch.mmu_zap_worker, &zap_obsolete_work).
4. unlock.

`Mmu::zap_obsolete_pages(kvm)` (worker):
1. lock(kvm.mmu_lock).
2. For each SP in active_mmu_pages:
   - If is_obsolete_sp(sp): zap; remove from list.
   - Yield-on-cond if many SPs.
3. kvm_flush_remote_tlbs(kvm).
4. unlock.

`Mmu::unmap_gfn_range(kvm, range)` (mmu-notifier):
1. lock(kvm.mmu_lock).
2. For each gfn in [range.start, range.end):
   - rmap_head := per-slot rmap.
   - zap matching SPTEs.
3. flush_remote_tlbs.
4. unlock.

`Mmu::invalidate_addr(vcpu, mmu, addr, root_hpa)`:
1. Walk per-vCPU MMU page-table for addr.
2. Per-leaf-SPTE: clear if matching.
3. kvm_x86_call(flush_tlb_gva)(vcpu, addr).

### Out of Scope

- Shadow MMU (covered in `x86-mmu-shadow.md` Tier-3)
- TDP MMU iter (covered in `x86-mmu-tdp.md` Tier-3)
- KVM core (covered in `kvm-core.md` Tier-3)
- mmu_notifier framework (covered in `mm/mmu-notifier.md` future Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `kvm_mmu_zap_all(kvm)` | per-VM zap all SPTEs | `Mmu::zap_all` |
| `kvm_mmu_zap_all_fast(kvm)` | per-VM fast-zap via gen-bump | `Mmu::zap_all_fast` |
| `kvm_mmu_zap_oldest_mmu_pages(kvm)` | per-VM evict oldest SPs | `Mmu::zap_oldest_mmu_pages` |
| `kvm_zap_obsolete_pages(kvm)` | per-VM zap stale-gen SPs | `Mmu::zap_obsolete_pages` |
| `kvm_zap_gfn_range(kvm, start, end)` | per-VM zap range | `Mmu::zap_gfn_range` |
| `__kvm_rmap_zap_gfn_range(kvm, slot, gfn_start, gfn_end, ...)` | per-rmap range-zap | `Mmu::__rmap_zap_gfn_range` |
| `kvm_zap_all_rmap_sptes(kvm, rmap_head)` | per-rmap-head zap-all | `Mmu::zap_all_rmap_sptes` |
| `kvm_unmap_gfn_range(kvm, range)` | per-mmu-notifier callback | `Mmu::unmap_gfn_range` |
| `kvm_test_age_gfn(kvm, range)` | per-mmu-notifier age-test | `Mmu::test_age_gfn` |
| `kvm_age_gfn(kvm, range)` | per-mmu-notifier age | `Mmu::age_gfn` |
| `kvm_set_pte_gfn(kvm, range)` | per-mmu-notifier change-pte | `Mmu::set_pte_gfn` |
| `kvm_arch_flush_shadow_all(kvm)` | per-arch-zap-all | `Mmu::flush_shadow_all` |
| `kvm_arch_flush_shadow_memslot(kvm, slot)` | per-memslot zap | `Mmu::flush_shadow_memslot` |
| `kvm_mmu_invalidate_addr(vcpu, mmu, addr, hpa)` | per-vCPU INVLPG | `Mmu::invalidate_addr` |
| `kvm_mmu_invalidate_gva(vcpu, mmu, gva, root_hpa)` | per-vCPU GVA invalidate | `Mmu::invalidate_gva` |
| `kvm_mmu_invalidate_all_pgds(vcpu)` | per-vCPU all-PCID invalidate | `Mmu::invalidate_all_pgds` |
| `kvm.arch.mmu_valid_gen` | per-VM generation counter | `KvmArch::mmu_valid_gen` |
| `is_obsolete_sp(sp)` | per-SP gen-stale check | `Sp::is_obsolete` |
| `kvm_mmu_zap_collapsible_sptes(kvm, slot)` | per-memslot collapse | `Mmu::zap_collapsible_sptes` |

### compatibility contract

REQ-1: Per-VM `mmu_valid_gen`:
- u64 atomic counter; per-VM.
- Bumped on per-zap-all-fast.
- Per-SP at create has snapshot.
- Per-vmenter compare; if stale: rebuild.

REQ-2: Zap categories:

**A. Range-based:**
- `kvm_zap_gfn_range(kvm, start, end)`:
  - Walk per-memslot rmap; per-rmap-entry zap matching SPTE.
  - TDP MMU: walk per-root TDP iter; per-leaf zap.
- Used: memslot delete, mmu_notifier invalidate-range.

**B. Per-VM all:**
- `kvm_mmu_zap_all(kvm)`:
  - Walk all per-VM SPs; zap each.
  - Slow but exhaustive.
- Used: rare; when generational fast-zap insufficient.

**C. Generational fast-zap:**
- `kvm_mmu_zap_all_fast(kvm)`:
  - Bump kvm.arch.mmu_valid_gen.
  - Worker zaps obsolete SPs in background (`kvm_zap_obsolete_pages`).
- Used: most common "all" path.

**D. Per-mmu-notifier:**
- Host MM hooks: invalidate_range_start / _end / change_pte / clear_young / clear_flush_young / test_young.
- Per-callback: kvm_unmap_gfn_range / kvm_age_gfn / kvm_test_age_gfn / kvm_set_pte_gfn.

**E. Per-vCPU TLB:**
- INVLPG, INVLPGB, INVPCID, MOV-to-CR3 → per-vCPU TLB-flush.
- Doesn't zap SPTEs; just invalidates per-CPU TLB.

REQ-3: SPTE rmap (reverse-map):
- Per-gfn-in-memslot list of SPTE-addresses pointing to that gfn.
- Used to find all SPTEs for a gfn during zap.

REQ-4: TDP MMU zap path:
- Per-TDP-root pgd: walk via `kvm_tdp_mmu_iter`.
- Per-iter: leaf SPTE → atomic-clear; flush.

REQ-5: Per-mmu_notifier hooks:
- Host kernel registers KVM as mmu_notifier on guest-mm.
- Per-host-MM-event: KVM callback fires.
- KVM acquires kvm->mmu_lock; performs zap.

REQ-6: Per-zap TLB-flush:
- Per-zapped-SPTE: kvm_flush_remote_tlbs(kvm) or kvm_flush_remote_tlbs_with_address.
- Per-vCPU IPI to flush TLB.

REQ-7: Per-VM mmu_notifier_seq:
- Bumped on every mmu_notifier event.
- Per-vCPU page-fault checks seq to detect concurrent host-MM-event.

REQ-8: Per-vCPU INVLPG handling:
- Guest INVLPG instruction → vmexit handle_invlpg.
- KVM: kvm_mmu_invalidate_addr per-vCPU + flush TLB.

REQ-9: Per-vCPU INVPCID handling:
- Per-PCID invalidation (with PCID enabled).
- KVM: per-PCID kvm_mmu_invalidate_pcid.

REQ-10: Per-VM mmu_valid_gen mod-256:
- Per-SP uses 8-bit gen field.
- Bumped mod-256; periodic full-zap on rollover.

REQ-11: zap_collapsible_sptes:
- Post-host-page-merge (THP): collapse 4KB SPTEs to 2MB or 1GB.
- Per-memslot iteration; per-SP demoted-then-promoted.

REQ-12: Per-zap kvm_mmu_lock:
- Required for non-RCU walks.
- TDP MMU has separate spinlock for fast-path.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `mmu_lock_held_during_zap` | INVARIANT | per-zap kvm.mmu_lock held; defense against concurrent SPTE-create-during-zap. |
| `mmu_valid_gen_monotonic` | INVARIANT | per-VM gen counter monotonically increases. |
| `obsolete_sp_eventually_freed` | INVARIANT | per-stale-gen SP eventually zapped + freed. |
| `tlb_flush_after_spte_clear` | INVARIANT | per-zapped-SPTE: kvm_flush_remote_tlbs called. |
| `mmu_notifier_seq_per_event` | INVARIANT | mmu_notifier_seq bumped on every notifier event. |

### Layer 2: TLA+

`virt/kvm/mmu_zap_lifecycle.tla`:
- Per-SPTE state ∈ {Present, Zapping, Zapped, Stale}.
- Properties:
  - `safety_no_present_after_zap` — Zapped → not subsequently observed Present without re-create.
  - `safety_tlb_flush_after_zap` — Zapped → kvm_flush_remote_tlbs eventually called.
  - `liveness_obsolete_eventually_zapped` — assuming worker runs, Stale eventually Zapped.

`virt/kvm/mmu_notifier_seq.tla`:
- Per-vCPU pf_seq vs kvm.mmu_notifier_seq.
- Properties:
  - `safety_pf_seq_consistency` — per-vCPU page-fault detects concurrent notifier-event via seq mismatch.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Mmu::zap_gfn_range` post: per-gfn SPTEs cleared; TLB flushed | `Mmu::zap_gfn_range` |
| `Mmu::zap_all_fast` post: gen incremented; worker queued | `Mmu::zap_all_fast` |
| `Mmu::zap_obsolete_pages` post: per-stale SP removed from active list | `Mmu::zap_obsolete_pages` |
| `Mmu::unmap_gfn_range` post: per-mmu-notifier range-zapped | `Mmu::unmap_gfn_range` |
| Per-zap mmu_notifier_seq advanced | invariants on notifier hooks |

### Layer 4: Verus/Creusot functional

`Per-zap: post-zap guest re-fault on cleared GFN OR proceeds with refreshed SPTE` semantic equivalence: per-zap the next guest access to affected GFN triggers KVM_EXIT_MMIO or page-fault handler.

### hardening

(Inherits row-1 features from `virt/kvm/x86-mmu-tdp.md` § Hardening.)

zap-specific reinforcement:

- **kvm.mmu_lock for non-RCU walks** — defense against torn SPTE state.
- **mmu_valid_gen 8-bit + rollover handling** — defense against stuck-gen.
- **TDP MMU separate lock** — defense against single-mutex bottleneck.
- **kvm_flush_remote_tlbs synchronous** — defense against pre-flush guest accessing stale TLB.
- **mmu_notifier_seq under lock** — defense against torn seq read.
- **Per-zap rmap walk RCU-protected** — defense against rmap-mod during walk.
- **Per-vCPU pf_seq compare** — defense against TLB stale across notifier-event.
- **zap_collapsible serialized with active fault** — defense against fault-during-collapse UAF.
- **Per-VM zap_obsolete worker bounded yield** — defense against zap-storm starving other workers.
- **Per-INVPCID type validated** — defense against guest issuing invalid INVPCID type.
- **Per-INVLPGB instruction validated** (newer) — defense against guest using before CPUID feature.
- **Per-mmu_notifier registration ref-counted** — defense against use-after-deregister.

