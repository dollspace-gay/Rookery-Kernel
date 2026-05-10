# Tier-3: arch/x86/kvm/x86.c (PKU subset) — KVM Protection-Keys virtualization (PKU + PKS via XSAVE PKRU + MSR_IA32_PKRS)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/kvm-core.md
upstream-paths:
  - arch/x86/kvm/x86.c (PKU/PKS sections; see grep "PKRU|PKRS|X86_FEATURE_PKU")
  - arch/x86/kvm/cpuid.c (PKU CPUID gating)
  - arch/x86/include/asm/pkru.h
-->

## Summary

Intel PKU (Protection Keys for User-pages; Skylake-X+) and PKS (Protection Keys for Supervisor; Tiger Lake+) are 16-key memory-tagging features. Each 4KB page-table entry's bits[62:59] encode a 4-bit protection-key; per-key access permissions controlled by PKRU register (user; via XSAVE feature bit 9) for PKU and MSR_IA32_PKRS (supervisor) for PKS. Per-key access-disable (AD) + write-disable (WD) bits in PKRU can revoke read/write permission per protection-domain at instruction-cycle granularity (much faster than mprotect()). KVM virtualizes via per-vCPU PKRU (in guest_fpu) + MSR_IA32_PKRS shadow + per-vCPU CPUID gating + CR4.PKE/_PKS enable bits + per-page-fault PKRU/PKRS check.

This Tier-3 covers PKU/PKS in `arch/x86/kvm/x86.c` (~150-200 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `X86_FEATURE_PKU` | PKU CPUID feature bit | shared |
| `X86_FEATURE_PKS` | PKS CPUID feature bit | shared |
| `vcpu.arch.pkru` | per-vCPU PKRU (in guest_fpu via XSAVE) | covered in `x86-fpu.md` |
| `vcpu.arch.pkrs` | per-vCPU PKRS (MSR shadow) | `VcpuArch::pkrs` |
| `MSR_IA32_PKRS` | PKS MSR | UAPI |
| `kvm_pku_supported(vcpu)` | per-vCPU PKU-cap query | `Vcpu::pku_supported` |
| `kvm_pks_supported(vcpu)` | per-vCPU PKS-cap query | `Vcpu::pks_supported` |
| `kvm_x86_call(get_protected_pkrs)(vcpu)` | per-vendor PKRS get | `Vcpu::get_pkrs` |
| `pku_check(vcpu, pte, fault)` (in MMU) | per-page-fault PKU check | covered in `x86-mmu-tdp.md` |

## Compatibility contract

REQ-1: PKU enabling:
- CR4.PKE = 1 enables PKU.
- Per-PTE bits[62:59] = protection-key (0..15).
- Per-key 2-bit access in PKRU: bit[2k]=AD (Access Disable), bit[2k+1]=WD (Write Disable).
- Per user-mode access (CPL=3): if pte.PK matches, PKRU bits checked.

REQ-2: PKS enabling:
- CR4.PKS = 1 enables PKS.
- Per-supervisor-mode access (CPL<3): MSR_IA32_PKRS instead of PKRU.
- Same 4-bit PK in PTE, 2-bit per-key in MSR_IA32_PKRS.

REQ-3: PKRU register:
- 32-bit; 16 keys × 2 bits.
- Per-key encoding: bit[2k] = AD; bit[2k+1] = WD.
- AD=1 → all access (R/W) blocked.
- WD=1 (with AD=0) → write blocked, read allowed.

REQ-4: PKRU as XSAVE component:
- XCR0 bit 9 = PKRU XSAVE component.
- Per-vCPU PKRU stored in guest_fpu->fpstate's XSAVE area.
- Saved/restored at vmenter/vmexit via XSAVE/XRSTOR.

REQ-5: MSR_IA32_PKRS for PKS:
- Per-CPU MSR.
- Same 16-key × 2-bit layout.
- Used for supervisor-mode access.

REQ-6: Per-vCPU CPUID gating:
- CPUID 0x07.0:ECX bit 3 = OSPKE (OS PKU enable).
- CPUID 0x07.0:ECX bit 31 = PKS.
- Guest sees PKU/PKS only if KVM advertises + host supports.

REQ-7: Per-vCPU CR4.PKE write semantics:
- PKE write requires CPUID.OSPKE.
- KVM rejects PKE=1 without PKU-cap CPUID.

REQ-8: Per-vCPU CR4.PKS write semantics:
- PKS write requires CPUID.PKS.
- KVM rejects PKS=1 without PKS-cap.

REQ-9: Per-page-fault PKU check (in MMU walker):
1. If !CR4.PKE: skip PK check.
2. pk := pte.bits[62:59].
3. Per-PKRU: if AD set: page-fault with PFERR_PK_MASK.
4. If WD set + write-access: page-fault.

REQ-10: WRPKRU / RDPKRU instructions:
- User-mode instructions (no privilege needed for WRPKRU per Intel-spec).
- KVM lets guest execute natively (no intercept).
- Per-vmenter: PKRU loaded from guest_fpu (XSAVE).

REQ-11: WRMSR/RDMSR(MSR_IA32_PKRS):
- Privileged (CPL=0 only).
- Per-vCPU shadow updated.
- Vendor-specific MSR pass-through (similar to PKRU).

REQ-12: Live migration:
- Per-vCPU PKRU + PKRS migrated as part of XSAVE2 + MSR_IA32_PKRS.

## Acceptance Criteria

- [ ] AC-1: Boot Linux guest with PKU host: guest CPUID 0x07.0:ECX bit 3 set.
- [ ] AC-2: Guest WRMSR(MSR_IA32_PKRS, X) (PKS): per-vCPU pkrs shadow updated.
- [ ] AC-3: Guest WRPKRU (PKU): per-vCPU PKRU updated via XSAVE.
- [ ] AC-4: Per-page PKU access-block: guest sets AD=1 for key K; subsequent access to PTE-with-PK=K causes #PF with PFERR_PK_MASK.
- [ ] AC-5: PKS supervisor: kernel-mode access via PKRS-controlled key.
- [ ] AC-6: CR4.PKE write without OSPKE CPUID: #GP injected.
- [ ] AC-7: Multi-vCPU: per-vCPU PKRU distinct.
- [ ] AC-8: Live migration: PKRU + PKRS preserved.
- [ ] AC-9: kvm-unit-tests `pku` test passes.
- [ ] AC-10: Performance: PKU page-fault dispatched without VMM round-trip cost (HW handles per Intel-spec).

## Architecture

Per-vCPU PKRU (in guest_fpu):

```
struct Mmu {
  ...
  // PKRU stored in XSAVE area at known offset (component 9)
}

struct VcpuArch {
  ...
  pkrs: u64,                                    // MSR_IA32_PKRS shadow
}
```

`Vcpu::pku_supported(vcpu)`:
1. Return cpu_has_pku(vcpu) && cpu_feature_enabled(X86_FEATURE_PKU).

`Vcpu::pks_supported(vcpu)`:
1. Return guest_cpuid_has(vcpu, X86_FEATURE_PKS) && cpu_feature_enabled(X86_FEATURE_PKS).

`Vcpu::handle_msr` MSR_IA32_PKRS branch:
1. WRMSR (guest):
   - If !pks_supported(vcpu): -EINVAL.
   - vcpu.arch.pkrs = data & PKRS_VALID_MASK.
2. RDMSR (guest):
   - Return vcpu.arch.pkrs.

`Vcpu::set_cr4` CR4.PKE / PKS branch:
1. If new_cr4 & X86_CR4_PKE && !pku_supported(vcpu): #GP.
2. If new_cr4 & X86_CR4_PKS && !pks_supported(vcpu): #GP.
3. Set CR4 normally; kvm_init_mmu refresh permissions.

`Mmu::permission_fault` (per-page-fault):
1. If CR4.PKE && cpl == 3:
   - pk := (pte >> 59) & 0xF.
   - pkru := vcpu.arch.guest_fpu_pkru().
   - access_disable := (pkru >> (2*pk)) & 1.
   - write_disable := (pkru >> (2*pk + 1)) & 1.
   - If access_disable: return PFERR_PK_MASK.
   - If write && write_disable: return PFERR_PK_MASK.
2. If CR4.PKS && cpl < 3:
   - Similar with vcpu.arch.pkrs.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pkrs_valid_mask` | INVARIANT | per-vCPU pkrs bits[63:32] = 0; lower 32 bits valid. |
| `pku_cpuid_required_for_cr4` | INVARIANT | CR4.PKE writable iff CPUID OSPKE. |
| `pks_cpuid_required_for_cr4` | INVARIANT | CR4.PKS writable iff CPUID PKS. |
| `per_page_pk_idx_bounded` | OOB | per-PTE pk ∈ [0, 15]. |
| `pku_pks_independent` | INVARIANT | per-vCPU PKRU + PKRS distinct registers. |

### Layer 2: TLA+

`virt/kvm/pku_pks_protection.tla`:
- Per-page protection-key + per-key permission state.
- Properties:
  - `safety_no_access_when_AD_set` — per-key AD=1 blocks read/write.
  - `safety_no_write_when_WD_set` — per-key WD=1 blocks write.
  - `safety_user_uses_pkru_supervisor_uses_pkrs` — per-cpl correct register consulted.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Vcpu::handle_msr` MSR_IA32_PKRS post: vcpu.arch.pkrs updated; subsequent RDMSR returns same | `Vcpu::handle_msr` |
| `Vcpu::set_cr4` PKE/PKS post: per-vCPU CR4 reflects guest write iff CPUID-supported | `Vcpu::set_cr4` |
| `Mmu::permission_fault` post: per-PK + per-cpl access checked vs PKRU/PKRS | `Mmu::permission_fault` |
| Per-vCPU PKRU saved/restored via XSAVE component-9 | invariants on FPU swap |

### Layer 4: Verus/Creusot functional

`Per-page access: read/write blocked iff (CR4.PKE && cpl=3 && PKRU.AD/WD per pte.PK) OR (CR4.PKS && cpl<3 && PKRS.AD/WD per pte.PK)` semantic equivalence: per-access the result matches Intel SDM PKU/PKS rules.

## Hardening

(Inherits row-1 features from `virt/kvm/kvm-core.md` § Hardening.)

PKU/PKS-specific reinforcement:

- **PKE/PKS CPUID gating** — defense against guest using unsupported feature.
- **PKRS supervisor-only access** — defense against user-mode reading kernel PKRS.
- **WRPKRU non-intercepted** — defense against per-instruction overhead; HW enforces.
- **Per-vCPU PKRU in guest_fpu** — defense against cross-vCPU PKRU leak.
- **PKRS reserved bits enforced** — defense against guest writing reserved bits.
- **Per-page-fault PFERR_PK_MASK** — defense against guest mis-classifying PK fault.
- **Live-migrate PKRU via XSAVE2** — defense against post-migrate inconsistent state.
- **PKE + PKS independent** — defense against feature-leakage between user/supervisor.
- **CR4.PKE write rejected without OSPKE** — defense against guest forcing PKU via CR4 alone.
- **Per-vCPU CPUID 0x07 mask gates PKU/PKS** — defense against per-VM feature mis-advertise.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- KVM core (covered in `kvm-core.md` Tier-3)
- KVM FPU/XSAVE (covered in `x86-fpu.md` Tier-3)
- Shadow MMU permissions (covered in `x86-mmu-shadow.md` Tier-3)
- TDP MMU permissions (covered in `x86-mmu-tdp.md` Tier-3)
- CR0/CR4 (covered in `x86-cr0-cr4.md` Tier-3)
- Implementation code
