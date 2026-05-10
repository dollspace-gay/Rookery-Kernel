# Tier-3: arch/x86/kvm/mmu/spte.c — per-SPTE bit-layout + make_spte construction (TDP + shadow shared)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/x86-mmu-tdp.md
upstream-paths:
  - arch/x86/kvm/mmu/spte.c
  - arch/x86/kvm/mmu/spte.h
  - arch/x86/kvm/mmu/mmu_internal.h (SPTE-related defs)
-->

## Summary

SPTEs (Shadow Page-Table Entries) are 64-bit entries that mirror guest-PA→host-PFN translation, used by both legacy shadow MMU and TDP MMU. spte.c (~576 lines) defines the per-SPTE bit layout and provides the `make_spte` constructor that takes vcpu state + (gfn, pfn, level, role, write_fault) and produces the final 64-bit SPTE value. spte.h (~572 lines) declares the bit constants + helper queries (is_present, is_writable, is_executable, is_mmio, etc.). MMIO SPTEs are special: encoded with a unique pattern that triggers EPT-misconfig vmexit on guest access, allowing KVM to emulate the access via in-kernel device.

This Tier-3 covers `spte.c` (~576 lines) + `spte.h` (~572 lines) + relevant mmu_internal.h definitions.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `union kvm_mmu_page_role` | per-SP role bitfield | `kernel::kvm::x86::mmu::SpRole` |
| `make_spte(vcpu, sp, slot, pte_access, gfn, pfn, old_spte, prefetch, can_unsync, host_writable, new_spte)` | per-leaf SPTE constructor | `Spte::make` |
| `make_huge_page_split_spte(...)` | per-split-SPTE during huge-page-collapse-undo | `Spte::make_split` |
| `make_mmio_spte(vcpu, gfn, access)` | construct MMIO-SPTE (special pattern) | `Spte::make_mmio` |
| `make_spte_executable(spte)` | clear NX bit | `Spte::make_executable` |
| `mark_spte_for_access_track(spte)` | set per-SPTE access-tracking bit (idle-page-tracking) | `Spte::mark_access_track` |
| `restore_acc_track_spte(spte)` | clear access-track + restore present | `Spte::restore_access_track` |
| `is_shadow_present_pte(spte)` | per-SPTE present-bit check | `Spte::is_present` |
| `is_writable_pte(spte)` | per-SPTE writable check | `Spte::is_writable` |
| `is_mmio_spte(spte)` | per-SPTE MMIO detection | `Spte::is_mmio` |
| `is_access_track_spte(spte)` | per-SPTE access-track detection | `Spte::is_access_track` |
| `is_last_spte(spte, level)` | leaf-or-intermediate detection | `Spte::is_last` |
| `is_dirty_spte(spte)` | per-SPTE dirty-bit check | `Spte::is_dirty` |
| `spte_to_pfn(spte)` | extract PFN | `Spte::pfn` |
| `kvm_mmu_set_spte_notrack_writes(...)` | non-tracked SPTE install (rare) | `Spte::set_notrack` |
| `__handle_changed_spte(...)` | per-SPTE change side-effects (rmap, dirty-tracking, page_track) | `Spte::handle_change` |
| `kvm_mmu_set_mmio_spte_mask(mmio_value, mmio_mask, access_mask)` | per-VM MMIO encoding setup | `KvmArchX86::set_mmio_spte_mask` |
| `make_nx_huge_page_split_spte(...)` | NX-huge-page-split fallback | `Spte::make_nx_huge_split` |

## Compatibility contract

REQ-1: SPTE bit layout (64-bit):

| Bits | Meaning |
|---|---|
| 0 | Present (P) |
| 1 | Writable (RW) |
| 2 | User-supervisor (US) |
| 3 | Page-Write-Through (PWT) |
| 4 | Page-Cache-Disable (PCD) |
| 5 | Accessed (A) |
| 6 | Dirty (D) |
| 7 | Page-Size (PS) — set means leaf at non-PT0 level |
| 8 | Global (G) |
| 9-11 | sw-only: MMU-Writable / Host-Writable / Special-Mask |
| 12 | EPT-only: ignored; legacy: sw-mid bit |
| 12..51 | Physical addr (host PFN << 12) |
| 52..62 | EPT-only: more flags + KVM access-tracking saved bits |
| 63 | NX (Execute-Disable) |

REQ-2: For EPT (Intel): bit-layout differs slightly:
- Bit 0 = read-allowed.
- Bit 1 = write-allowed.
- Bit 2 = exec-allowed (or exec-for-supervisor when mode-based-exec).
- Bits 3-5 = memory-type (UC/WB/WC/etc.).
- Bits 8/9 = A/D (when EPT A/D enabled).

REQ-3: For NPT (AMD): same as legacy paging.

REQ-4: MMU-Writable (sw bit 9): KVM marks SPTE as logically-writable per guest PTE; cleared during write-protect-for-dirty-tracking; restored on dirty-flush.

REQ-5: Host-Writable (sw bit 10): host page is writable (not COW/shared/zero-page); cleared if KVM cannot guarantee host-write.

REQ-6: SPTE.RW = MMU-Writable AND Host-Writable.

REQ-7: MMIO SPTEs (special encoding):
- Pattern: `mmio_value | (gfn << 12) | (access_mask << some_offset)`.
- Encoded gfn + access flags directly in SPTE's "physical addr" bits.
- Pattern designed to trigger EPT-misconfig vmexit (e.g., reserved-bits set, R/W/X all clear) on guest access.
- KVM detects via `is_mmio_spte` and emulates the access.
- `mmio_mask` per-VM-config + `mmio_value` per-VM-config; depends on host CPU max_phyaddr.

REQ-8: Access-track SPTEs:
- KVM may mark SPTE as access-tracked (clear present + set special bit) when host detects idle-page.
- Subsequent guest access → page-not-present vmexit → KVM observes access-track + restores present.
- Used for /sys/kernel/mm/idle_page_tracking integration.

REQ-9: Per-SPTE A/D bit shadowing (when EPT/NPT lacks A/D support):
- KVM uses sw-bits to track guest A/D state; updates via emulator.

REQ-10: NX huge-page split: when KVM detects guest writes to huge-page-mapped-as-rwx-NX-allowed, splits to PT0 + marks NX-protected on subset; defense against transient-execution exploits via huge-page-prefetch.

REQ-11: SP role (16-bit bitfield):
- level (4 bits).
- direct (1 bit) — direct-map mode (TDP).
- has_4_byte_gpte (1 bit) — 32-bit non-PAE guest paging.
- access (3 bits) — RWX permissions.
- invalid (1 bit) — generation-stale marker.
- efer_nx (1 bit), cr0_wp (1 bit), smep (1 bit), smap (1 bit).
- ad_disabled (1 bit) — A/D bits absent in guest CR4.
- guest_mode (1 bit) — nested-virt context.

REQ-12: `make_spte` semantics:
- Inputs: vcpu (for cpu role), sp (SP role), slot (memslot for permissions), pte_access (guest-derived), gfn, pfn, old_spte, prefetch, can_unsync, host_writable.
- Output: new SPTE value.
- Internal logic:
  - Start with present=1, R=1, A=1.
  - Apply NX from sp.role.efer_nx + pte_access.
  - Apply RW from pte_access ∩ host_writable.
  - Apply US from pte_access.US.
  - Apply memory-type from slot (UC for MMIO, WB default).
  - Apply page-size from sp.role.level > PG_LEVEL_4K.
  - Apply MMU-Writable / Host-Writable sw-bits.
  - Set physical addr = pfn << 12.

## Acceptance Criteria

- [ ] AC-1: SPTE construction round-trip: per-SP role + pte_access produces SPTE with matching is_writable / is_executable / is_present.
- [ ] AC-2: MMIO SPTE: virtio-mmio access from guest → EPT-misconfig vmexit → emulated.
- [ ] AC-3: Access-track: idle-page-tracking marks SPTE access-tracked; subsequent access restores present.
- [ ] AC-4: NX-huge-page-split: guest-write to huge-page-NX-region triggers split + per-PT0 NX set on relevant subset.
- [ ] AC-5: A/D bit propagation (when EPT supports): host A/D bits update visible via KVM_GET_DIRTY_LOG.
- [ ] AC-6: A/D bit emulation (when EPT lacks A/D): per-write triggers emulation that updates sw-A/D bit.
- [ ] AC-7: SP role hash collision: distinct guest mappings same gfn but different role get distinct SPs.
- [ ] AC-8: kvm-unit-tests `mmu` test passes.

## Architecture

`SpRole` packed bitfield:

```
struct SpRole(u16) {
  level: u4,
  direct: u1,
  has_4_byte_gpte: u1,
  access: u3,
  invalid: u1,
  efer_nx: u1,
  cr0_wp: u1,
  smep_andnot_wp: u1,
  smap_andnot_wp: u1,
  ad_disabled: u1,
  guest_mode: u1,
}
```

`Spte::make(...)` flow (refines REQ-12):
1. Initialize spte = SPTE_PRESENT | SPTE_ACCESSED.
2. If sp.role.level > PG_LEVEL_4K: spte |= SPTE_PAGE_SIZE.
3. Apply NX: if !pte_access.X || sp.role.efer_nx: spte |= SPTE_NX.
4. Apply RW: if pte_access.W && host_writable: spte |= SPTE_WRITABLE; spte |= SPTE_MMU_WRITABLE; spte |= SPTE_HOST_WRITABLE.
5. Apply US: if pte_access.U: spte |= SPTE_USER.
6. Apply pfn: spte |= (pfn << 12) & SPTE_PFN_MASK.
7. Apply memory-type: spte |= host_memory_type(slot, gfn).
8. If !can_unsync && SP-shared-write: write-protect (clear SPTE_WRITABLE keeping MMU_WRITABLE).
9. If write_fault && pte_access.W: spte |= SPTE_DIRTY.
10. Return final spte.

`Spte::make_mmio(vcpu, gfn, access)`:
1. Compose value = mmio_value | (gfn << SPTE_MMIO_GFN_SHIFT) | (access << SPTE_MMIO_ACCESS_SHIFT).
2. Some reserved-bits set (per-CPU max_phyaddr-based) so HW triggers EPT-misconfig.
3. Return.

`Spte::handle_change(old_spte, new_spte, sp, level)`:
1. If old.present + new.present + old.pfn != new.pfn: BUG (SPTE-PFN-mid-flight; should never happen).
2. If old.present + !new.present: per-rmap-remove(old.pfn).
3. If !old.present + new.present: per-rmap-add(new.pfn).
4. If old.dirty != new.dirty: dirty-tracking update.
5. If page_track-watcher present: callback dispatch.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `make_spte_present_iff_pfn_valid` | INVARIANT | spte.present == 1 implies pfn ∈ host RAM. |
| `mmio_spte_pattern_distinct` | INVARIANT | mmio_value pattern + reserved-bits ensure HW always EPT-misconfigs on access; defense against MMIO-SPTE-aliasing-with-real-mapping. |
| `pfn_no_oob` | OOB | per-SPTE.pfn extracted via mask; bounded by SPTE_PFN_MASK. |
| `nx_implies_x_clear` | INVARIANT | spte.NX==1 implies spte.X==0 (mutual). |

### Layer 2: TLA+

`virt/kvm/spte_lifecycle.tla` (already exists from earlier Tier-3 collection) models SPTE state:
- States: NotPresent, Present(ro), Present(rw), AccessTracked, Mmio.
- Transitions:
  - NotPresent → Present(ro/rw) via make_spte.
  - Present(rw) → Present(ro) via write-protect.
  - Present(ro) → Present(rw) via permission-grant.
  - Any → AccessTracked via mark_access_track.
  - AccessTracked → Present via restore_access_track.
- Properties:
  - `safety_present_implies_pfn_valid`.
  - `safety_mmio_distinct_from_present` — Mmio state not transitioning to Present without explicit reset.
  - `liveness_eventually_evicted` — under memory pressure, Present eventually reaches NotPresent.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Spte::make` post: returned spte respects RW = MMU_WRITABLE ∧ HOST_WRITABLE | `Spte::make` |
| `Spte::make_mmio` post: spte triggers EPT-misconfig on HW (verified via spec model) | `Spte::make_mmio` |
| `Spte::handle_change` post: rmap consistent (add iff new.present, remove iff old.present) | `Spte::handle_change` |
| Per-SPTE.pfn within memslot if SPTE.present | `Spte::make` |

### Layer 4: Verus/Creusot functional

`Guest VA → Spte::make(...) → final SPTE → guest MMU walk produces correct PA + permissions` semantic equivalence: per-VA the SPTE's effective permission match composition of guest-PTE permission ∧ host-page permission.

## Hardening

(Inherits row-1 features from `virt/kvm/x86-mmu-tdp.md` § Hardening.)

SPTE-specific reinforcement:

- **MMIO-SPTE pattern includes reserved-bits-set** — defense against HW silently treating MMIO-SPTE as real mapping.
- **Per-SPTE NX-huge-page split** for transient-execution mitigation — defense against Spectre-style speculative-execution via huge-page-prefetch.
- **MMU-Writable + Host-Writable AND-coupling** — defense against KVM-marking-writable while host page COW.
- **Per-SPTE pfn validated against memslot** — defense against PFN pointing outside guest-owned memory.
- **`__handle_changed_spte` BUG on present-pfn-mid-flight** — defense against torn-SPTE-update causing rmap inconsistency.
- **Per-SPTE access-track preserves real PFN in saved-bits** — defense against access-track losing pfn.
- **Per-VM mmio_value derived from host CPU max_phyaddr** — defense against generic mmio_value colliding with real PA on small-phyaddr hosts.
- **Per-SP role.invalid bit honored** — generation-stale SP detected; subsequent SPTE-walk skip.
- **MMIO-SPTE never installed for memslot pages** — defense against cross-mode-confusion mapping real RAM as MMIO.
- **Per-SPTE atomic install** (cmpxchg) — defense against torn-SPTE-update causing transient inconsistency.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- TDP MMU iter (covered in `x86-mmu-tdp.md` Tier-3)
- Shadow MMU + paging_tmpl (covered in `x86-mmu-shadow.md` Tier-3)
- KVM core (covered in `kvm-core.md` Tier-3)
- VMX vendor (covered in `x86-vmx.md` Tier-3)
- SVM vendor (covered in `x86-svm.md` Tier-3)
- mmu_internal.h SP-allocator details (covered in `x86-mmu-shadow.md` Tier-3)
- Implementation code
