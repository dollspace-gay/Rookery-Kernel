# Tier-3: arch/x86/kvm/mtrr.c — KVM MTRR (Memory-Type-Range-Register) virtualization + PAT integration

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/kvm-core.md
upstream-paths:
  - arch/x86/kvm/mtrr.c
  - arch/x86/kvm/x86.c (MTRR-MSR dispatch)
  - arch/x86/include/asm/mtrr.h (UAPI mtrr ranges)
  - arch/x86/include/uapi/asm/msr-index.h (MSR_IA32_MTRR_*, MSR_IA32_PAT)
-->

## Summary

MTRR (Memory-Type-Range-Register) and PAT (Page-Attribute-Table) are x86 mechanisms that let firmware/OS specify per-physical-range memory caching attributes (UC=uncacheable, WB=writeback, WC=write-combining, WT=writethrough, WP=write-protect). KVM virtualizes both for guest: per-guest fixed-MTRR (8 fixed-range MSRs covering legacy 0-1MB), variable-MTRR (8 var-range MSR-pairs each describing a 2^k-aligned range), MTRR-default (default cache type), MTRR-cap (capability MSR), and PAT (8-entry table indexed by guest PTE PWT/PCD/PAT bits). Critical for: guest device-passthrough (frame-buffer must be WC; MMIO must be UC); guest performance (incorrect WB-on-MMIO causes write-merging-corruption).

This Tier-3 covers `arch/x86/kvm/mtrr.c` (~133 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `kvm_set_msr_common(vcpu, msr_info)` MTRR-MSR dispatch | RDMSR/WRMSR for MTRR/PAT MSRs | covered by `Vcpu::handle_msr` |
| `kvm_mtrr_set_msr(vcpu, msr, data)` | per-MTRR-MSR set | `KvmMtrr::set_msr` |
| `kvm_mtrr_get_msr(vcpu, msr, &data)` | per-MTRR-MSR get | `KvmMtrr::get_msr` |
| `kvm_mtrr_check_gfn_range_consistency(vcpu, gfn, page_count)` | per-range consistency check | `KvmMtrr::check_gfn_range_consistency` |
| `kvm_mtrr_get_guest_memory_type(vcpu, gfn)` | per-gfn guest memory-type query | `KvmMtrr::get_guest_memory_type` |
| `kvm_mtrr_valid(vcpu, msr, data)` | per-MSR write validity | `KvmMtrr::valid` |
| `update_mtrr(vcpu, msr)` | post-write MTRR-cache rebuild | `KvmMtrr::update` |
| `kvm_mtrr_init(vcpu)` | per-vCPU MTRR init | `KvmMtrr::init` |
| `var_mtrr_range(...)` | per-var-MTRR range decode | `KvmMtrr::var_range_decode` |

## Compatibility contract

REQ-1: MTRR MSRs:
- `MSR_IA32_MTRRCAP` (0xFE): read-only; bits[7:0] = vcnt (var-MTRR count, typically 8); bit 8 = FIX (fixed-MTRR support); bit 10 = WC (WC type support); bit 11 = SMRR.
- `MSR_IA32_MTRR_DEF_TYPE` (0x2FF): bits[7:0] = default type; bit 10 = FE (fixed-MTRR enable); bit 11 = E (overall enable).
- `MSR_IA32_MTRR_PHYSBASE0..7` (0x200, 0x202, ...): per-var-MTRR base + type.
- `MSR_IA32_MTRR_PHYSMASK0..7` (0x201, 0x203, ...): per-var-MTRR mask + valid bit.
- 8 fixed MSRs covering 0..1MB:
  - MSR_IA32_MTRR_FIX64K_00000 (0x250): 8 sub-ranges of 64KB each.
  - MSR_IA32_MTRR_FIX16K_80000 (0x258), MSR_IA32_MTRR_FIX16K_A0000 (0x259): 16 sub-ranges of 16KB each.
  - MSR_IA32_MTRR_FIX4K_C0000..F8000 (0x268..0x26F): 64 sub-ranges of 4KB each.

REQ-2: PAT MSR:
- `MSR_IA32_PAT` (0x277): 64-bit; 8 entries each 8-bit; per-entry encodes a memory type. Linux default: PAT0=WB, PAT1=WT, PAT2=UC-, PAT3=UC, PAT4=WB, PAT5=WT, PAT6=UC-, PAT7=UC.

REQ-3: Guest WRMSR(MSR_IA32_MTRR_*) emulation:
1. Validate per-MSR via `KvmMtrr::valid`:
   - PHYSBASE: type ∈ {0=UC, 1=WC, 4=WT, 5=WP, 6=WB}; base aligned to ≥ 4KB.
   - PHYSMASK: lower bits zero per arch-spec; valid-bit at position 11.
   - Fixed: per-byte type validation.
   - DEF_TYPE: type valid + reserved bits zero.
2. Update per-vCPU MTRR shadow.
3. `update_mtrr(vcpu, msr)`:
   - Recompute per-gfn cache-type lookup table (cached for fast lookup).
   - Per-cached SPTE memory-type bits may need refresh; `kvm_mmu_zap_all` if mtrr-state changed substantially.

REQ-4: Per-gfn memory-type lookup:
- `kvm_mtrr_get_guest_memory_type(vcpu, gfn)`:
  - Walk fixed-MTRR if gfn < 1MB and FIX-enabled.
  - Walk var-MTRR; per-var: if (gfn << PAGE_SHIFT) match (base & mask): match found; record type.
  - On overlap: per-spec, UC > WT > else; KVM follows.
  - If no match: return DEF_TYPE.

REQ-5: SPTE memory-type bits (TDP path):
- KVM_X86_QUIRK_MWAIT_NEVER_UD_FAULTS variant: KVM uses guest's effective MTRR-PAT effective-type to populate SPTE memory-type bits.
- For non-coherent-DMA guests: KVM may force UC if guest-MTRR allows broader caching but device requires UC.

REQ-6: PAT WRMSR validation:
- 8-entry table; each entry < 8 (memory-type encoding).
- Reserved-types raise #GP.

REQ-7: Per-VM MTRR state:
- `vcpu.arch.mtrr_state`:
  - `mtrr_state_t` (Linux mtrr struct): per-fixed + per-var entries + def_type.
  - `pat` (per-vCPU PAT shadow).

REQ-8: Live migration:
- KVM_GET_MSRS / KVM_SET_MSRS of MTRR + PAT MSRs round-trip.
- Per-MSR included in migration-required-MSR list.

REQ-9: SMRR (System Management Range Registers, optional):
- MSR_IA32_SMRR_PHYSBASE / _PHYSMASK for SMM range protection.
- KVM advertises SMRR if guest CPUID supports.

REQ-10: NoT-implemented optional features:
- WC-MTRR is supported (bit 10 of CAP set).
- SMRR support gated on guest-OS opt-in.
- WT type rare but supported.

REQ-11: Per-vCPU effective-type derivation (for SPTE):
- Effective = combine(MTRR_type[gfn], PAT_type[pte_pat_bits]).
- Combine table per Intel SDM Vol 3 Sec 11.5.2 Table 11-7.
- Implementation: per-(MTRR, PAT) pair → effective.

## Acceptance Criteria

- [ ] AC-1: Boot Linux guest under qemu+kvm: BIOS programs default MTRR (WB for system RAM); guest perceives correct memory-type.
- [ ] AC-2: Guest enables MTRR + WRMSR-set DEF_TYPE = WB: SPTE memory-type bits reflect WB.
- [ ] AC-3: Guest writes var-MTRR for 0xFEC00000+ (IOAPIC) as UC: SPTE for that range becomes UC.
- [ ] AC-4: PAT modifications: guest sets PAT0=WC; PTE with pwt=0,pcd=0,pat=0 maps as WC.
- [ ] AC-5: Live migration: MTRR + PAT state preserved across migrate.
- [ ] AC-6: SR-IOV passthrough: VF device's MMIO BAR forced UC despite guest MTRR set WB.
- [ ] AC-7: kvm-unit-tests `mtrr` test passes.
- [ ] AC-8: Spec-violation defense: guest WRMSR(MTRR) with reserved-bit set raises #GP to guest.

## Architecture

`KvmMtrr` per-vCPU:

```
struct KvmMtrr {
  fixed_ranges: KBox<[u8; KVM_NR_FIXED_MTRR_REGION * 8]>,  // 11 fixed-MSRs × 8 bytes = 88 entries
  var_ranges: [VarMtrr; KVM_NR_VAR_MTRR],                  // typically 8
  def_type: u8,                                            // bits[7:0] of MSR_IA32_MTRR_DEF_TYPE
  fixed_enabled: bool,
  enabled: bool,
  pat: u64,                                                // PAT shadow (8 × 8-bit entries)
  cached_lookup: KBox<MtrrLookupCache>,                    // per-gfn memory-type cache
}

struct VarMtrr {
  base: u64,                                                // PHYSBASE: addr | type
  mask: u64,                                                // PHYSMASK: mask | valid_bit
}
```

`KvmMtrr::set_msr(vcpu, msr_idx, data)`:
1. Switch on msr_idx:
   - MSR_IA32_MTRR_DEF_TYPE: validate; vcpu.arch.mtrr.def_type = data & 0xFF; .enabled = !!(data & MTRR_DEF_TYPE_E); .fixed_enabled = !!(data & MTRR_DEF_TYPE_FE).
   - MSR_IA32_MTRRCAP: read-only; raise #GP.
   - MSR_IA32_MTRR_PHYSBASE0..7: var_ranges[idx].base = data; validate.
   - MSR_IA32_MTRR_PHYSMASK0..7: var_ranges[idx].mask = data; validate.
   - MSR_IA32_MTRR_FIX*: fixed_ranges[range_offset .. +8] = unpack(data).
   - MSR_IA32_PAT: validate per-byte; vcpu.arch.mtrr.pat = data.
2. `update_mtrr(vcpu, msr_idx)`:
   - Invalidate cached_lookup.
   - If mtrr-state-change affects SPTE memory-type bits: kvm_zap_gfn_range(...) for affected ranges OR kvm_mmu_zap_all().

`KvmMtrr::get_guest_memory_type(vcpu, gfn)`:
1. If !mtrr.enabled: return MTRR_TYPE_UNCACHABLE.
2. If gfn < 1MB && mtrr.fixed_enabled: return per-fixed-MSR-entry-type.
3. For each var_ranges[i]:
   - If (var.mask >> PAGE_SHIFT) & valid_bit set:
     - If ((gfn << PAGE_SHIFT) ^ var.base.addr) & var.mask.mask == 0:
       - record var.base.type.
4. Combine multiple matches per Intel-spec precedence (UC > WT > else).
5. If no match: return mtrr.def_type.

`KvmMtrr::init(vcpu)`:
1. Default def_type = MTRR_TYPE_UNCACHABLE.
2. Default fixed/var ranges all-zero.
3. Default pat = 0x0007040600070406ULL (Linux default).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `mtrr_type_valid` | INVARIANT | per-MTRR.type ∈ {0=UC, 1=WC, 4=WT, 5=WP, 6=WB}; defense against guest writing reserved type causing UB on lookup. |
| `var_mtrr_count_bounded` | INVARIANT | KVM_NR_VAR_MTRR ≤ 10 (per-spec max); var-range index always in-bounds. |
| `pat_entry_valid` | INVARIANT | each PAT-entry < 8 (memory-type encoding); enforced at WRMSR. |
| `cached_lookup_consistent` | INVARIANT | cached_lookup either invalidated (None) or matches current MTRR state. |

### Layer 2: TLA+

`virt/kvm/mtrr_lookup.tla`:
- Per-gfn memory-type derivation as deterministic function of (MTRR state, PAT state, gfn).
- Properties:
  - `safety_lookup_total` — for every gfn, lookup terminates with single type result.
  - `safety_disabled_returns_uc` — !enabled implies result == UC.
  - `safety_match_precedence` — multi-match resolved per Intel-spec precedence.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `KvmMtrr::set_msr` post: MTRR shadow updated; cached_lookup invalidated | `KvmMtrr::set_msr` |
| `KvmMtrr::get_guest_memory_type` post: returned type consistent with current MTRR + PAT state | `KvmMtrr::get_guest_memory_type` |
| Per-MTRR PHYSBASE.addr aligned to 4KiB | `KvmMtrr::valid` |
| Per-MTRR PHYSMASK reserved-bits zero | `KvmMtrr::valid` |
| `KvmMtrr::update` post: SPTE memory-type bits refreshed via mmu-zap if state changed substantially | `KvmMtrr::update` |

### Layer 4: Verus/Creusot functional

`Guest WRMSR(MTRR/PAT) → Spte::make memory-type bits → guest CPU caches at correct type`: per-gfn the SPTE memory-type bits match `combine(MTRR-type[gfn], PAT-type[pat-idx])` per Intel-SDM combine table.

## Hardening

(Inherits row-1 features from `virt/kvm/kvm-core.md` § Hardening.)

MTRR-specific reinforcement:

- **MTRR/PAT WRMSR validation** at hot-path — defense against guest writing reserved-type causing combine-table-OOB.
- **Per-vCPU PHYSBASE.type validated** — defense against type values 2,3,7 (reserved per Intel-spec).
- **Per-vCPU PHYSMASK valid-bit-only meaningful** — defense against guest setting valid-bit but mask-bits all-zero (matches whole address space causing global override).
- **mtrr-state-change forces SPTE refresh** — defense against stale-cached SPTE memory-type after MTRR update.
- **Cached lookup invalidated on every WRMSR(MTRR/PAT)** — defense against stale lookup giving wrong type.
- **MTRR def_type initialized to UC at vCPU reset** — per-Intel-SDM safe-default; defense against accidentally-WB before guest BIOS programs MTRR.
- **PAT default = 0x0007040600070406ULL** — Linux-spec default; defense against PAT-zero causing all-WB classification.
- **MTRRCAP read-only** — defense against guest claiming WC/SMRR support host doesn't have.
- **SR-IOV passthrough force-UC for device-MMIO** — defense against guest-MTRR-WB allowing write-combining on MMIO causing IO-corruption.
- **MTRR + PAT state included in live-migration MSR list** — defense against post-migrate cache-config inconsistency.
- **Per-vCPU MTRR re-init on KVM_VCPU_RESET** — defense against carryover MTRR state across guest INIT.

## Grsecurity/PaX-style Reinforcement

Baseline hardening (always applied):

- **PAX_USERCOPY** — MTRR/PAT MSR set via KVM_SET_MSRS bounded.
- **PAX_KERNEXEC** — MTRR combine table + walker RO after init.
- **PAX_RANDKSTACK** — randomized kstack per WRMSR(MTRR/PAT) emulation.
- **PAX_REFCOUNT** — MTRR cache invalidate refcount saturating.
- **PAX_MEMORY_SANITIZE** — kvm_mtrr per-vCPU struct zeroed at INIT and destroy.
- **PAX_UDEREF** — KVM_SET_MSRS user pointer validated.
- **PAX_RAP / kCFI** — MTRR walk/lookup callbacks type-checked.
- **GRKERNSEC_HIDESYM** — MTRR base/mask values redacted in trace.
- **GRKERNSEC_DMESG** — MTRR-misconfig warnings rate-limited.

MTRR-specific:

- **CAP_SYS_ADMIN on KVM_SET_MSRS for MTRR/PAT** — defense against unprivileged memory-type flip.
- **Reserved-type 2/3/7 rejected at WRMSR** — closes combine-table OOB.
- **Passthrough device-MMIO force-UC** — defense against guest WC-on-MMIO IO corruption.

Rationale: MTRR/PAT control memory-type for every SPTE; sanitize on destroy + reserved-type rejection prevent combine-table OOB and post-INIT stale cache-policy carryover.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- KVM core (covered in `kvm-core.md` Tier-3)
- TDP MMU memory-type integration (covered in `x86-mmu-tdp.md` Tier-3)
- SPTE bit layout (covered in `x86-mmu-spte.md` Tier-3)
- VMX vendor (covered in `x86-vmx.md` Tier-3)
- SVM vendor (covered in `x86-svm.md` Tier-3)
- Guest-side MTRR programming (guest-driver concern)
- Implementation code
