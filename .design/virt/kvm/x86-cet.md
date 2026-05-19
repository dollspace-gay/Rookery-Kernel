# Tier-3: arch/x86/kvm/x86.c (CET subset) — KVM CET (Control-flow Enforcement Technology) virtualization (shadow-stack + IBT)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/kvm-core.md
upstream-paths:
  - arch/x86/kvm/x86.c (CET sections; see grep "MSR_IA32_S_CET\|MSR_IA32_U_CET\|MSR_IA32_PL[0-3]_SSP\|XFEATURE_CET")
  - arch/x86/include/asm/cet.h
  - arch/x86/kvm/cpuid.c (CET_SS, CET_IBT)
-->

## Summary

Intel CET (Control-flow Enforcement Technology; Tiger Lake+) provides hardware Shadow Stack (CET-SS) for return-address protection + Indirect Branch Tracking (CET-IBT) via ENDBR64/ENDBR32 instructions for indirect-branch validation. KVM virtualizes via per-vCPU MSR_IA32_S_CET (supervisor) + MSR_IA32_U_CET (user) + MSR_IA32_PL[0-3]_SSP (per-privilege-level shadow-stack pointers) + MSR_IA32_INT_SSP_TAB (per-IDT shadow-stack table). XSAVE state-component bits 11+12 cover CET. Per-vCPU CR4.CET enable. Critical for: hardening guest kernel + userspace against ROP/JOP attacks; modern Windows/Linux uses CET when available.

This Tier-3 covers CET virtualization in `arch/x86/kvm/x86.c` (~150-200 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `MSR_IA32_S_CET` (0x6A2) | supervisor CET control | UAPI |
| `MSR_IA32_U_CET` (0x6A0) | user CET control | UAPI |
| `MSR_IA32_PL0_SSP` (0x6A4) | ring-0 shadow-stack pointer | UAPI |
| `MSR_IA32_PL1_SSP` (0x6A5) | ring-1 SSP | UAPI |
| `MSR_IA32_PL2_SSP` (0x6A6) | ring-2 SSP | UAPI |
| `MSR_IA32_PL3_SSP` (0x6A7) | ring-3 SSP | UAPI |
| `MSR_IA32_INT_SSP_TAB` (0x6A8) | interrupt shadow-stack table | UAPI |
| `XFEATURE_MASK_CET_USER` (bit 11) | CET-user XSAVE component | UAPI |
| `XFEATURE_MASK_CET_KERNEL` (bit 12) | CET-supervisor XSAVE component | UAPI |
| `X86_CR4_CET` | CR4.CET enable bit | shared |
| `X86_FEATURE_SHSTK` | CPUID 0x07.0:ECX[7] CET-Shadow-Stack | shared |
| `X86_FEATURE_IBT` | CPUID 0x07.0:EDX[20] CET-Indirect-Branch-Tracking | shared |
| `kvm_cet_supported(vcpu)` | per-vCPU CET-cap query | `Vcpu::cet_supported` |
| `kvm_cet_user_supported(vcpu)` | per-vCPU CET-user-cap query | `Vcpu::cet_user_supported` |

## Compatibility contract

REQ-1: CR4.CET (bit 23):
- Master enable.
- Required for: any S_CET / U_CET / PL[N]_SSP / INT_SSP_TAB activity.
- Per-vCPU CR4.CET write requires CPUID.SHSTK or CPUID.IBT.

REQ-2: MSR_IA32_S_CET / U_CET fields:
- Bit[0] SH_STK_EN (Shadow-Stack Enable).
- Bit[1] WR_SHSTK_EN (WRSS instruction allowed).
- Bit[2] ENDBR_EN (IBT enable).
- Bit[3] LEG_IW_EN (Legacy-IW-Wait).
- Bit[4] NO_TRACK_EN (NoTrack prefix-allowed).
- Bit[5] SUPPRESS_DIS (Suppress-Disable).
- Bit[10] SUPPRESS (Suppress IBT).
- Bit[11] TRACKER (per-current-track state).
- Bits[12..63]: EB_LEG_BITMAP (per-legacy-IW bitmap; vendor-specific).

REQ-3: PL[0-3]_SSP MSRs:
- Per-privilege-level shadow-stack pointer (8-byte aligned).
- Used by HW on privilege transitions: per-INT/SYSCALL/etc.: SSP loaded from PL[CPL]_SSP.

REQ-4: MSR_IA32_INT_SSP_TAB:
- Pointer to 8-entry SSP-table for interrupt-vectors.
- Per-vector (bits[7:0] in INT_SSP_TAB) overrides PL[0]_SSP.
- Used for IDT-style shadow-stack switching.

REQ-5: XSAVE state component CET-USER (bit 11):
- 16 bytes: U_CET_MSR + PL3_SSP_MSR.
- Per-vCPU saved/restored via XSAVE/XRSTOR around vmenter/vmexit.

REQ-6: XSAVE state component CET-SUPERVISOR (bit 12):
- 24 bytes: S_CET_MSR + PL0_SSP_MSR + INT_SSP_TAB_MSR.
- Per-vCPU XSAVES (supervisor-XSAVE).

REQ-7: SHSTK semantics (when SH_STK_EN=1):
- Per-CALL: HW also pushes return-addr to shadow-stack.
- Per-RET: HW compares stack-return-addr vs shadow-stack-return-addr.
- Mismatch: #CP (Control-Protection exception, vector 21).

REQ-8: IBT semantics (when ENDBR_EN=1):
- Per-indirect-jump (CALL r/m / JMP r/m): HW expects ENDBR* opcode at target.
- Missing ENDBR: #CP exception.
- ENDBR is NOP on non-CET CPU.

REQ-9: KVM virtualization:
- Per-vCPU CR4.CET write: validate CPUID-cap; update.
- Per-vCPU MSR_IA32_*_CET write: validate reserved-bits; update shadow.
- PL[0-3]_SSP: shadow-stack-pointer 8-byte-aligned.
- INT_SSP_TAB: 8-byte-aligned table pointer.

REQ-10: Per-vCPU CPUID gating:
- CPUID 0x07.0:ECX[7] = 1 iff KVM advertises CET-SS.
- CPUID 0x07.0:EDX[20] = 1 iff CET-IBT.
- CPUID 0x0D sub-leaf 11 = CET-USER XSAVE size.
- CPUID 0x0D sub-leaf 12 = CET-SUPERVISOR XSAVE size.

REQ-11: #CP (vector 21) injection:
- Per-CET-violation triggers HW #CP.
- Error code:
  - Bit[0] NEAR_RET / FAR_RET / ENDBR / RSTORSSP / SETSSBSY / etc.
- KVM intercepts via vmcs.exception_bitmap if guest_debug.

REQ-12: Live migration:
- Per-vCPU CET MSRs migrated via XSAVE2 + KVM_GET_MSRS.
- Destination CPUID validated.

## Acceptance Criteria

- [ ] AC-1: Boot Linux guest with CET host: CPUID 0x07.0:ECX bit 7 set.
- [ ] AC-2: Guest WRMSR(MSR_IA32_U_CET, SH_STK_EN | ENDBR_EN): SHSTK + IBT enabled.
- [ ] AC-3: Per-vCPU CR4.CET write: master CET enabled.
- [ ] AC-4: SHSTK fault: guest mismatches stack-vs-shadow-stack return; #CP delivered.
- [ ] AC-5: IBT fault: guest indirect-branch to non-ENDBR target; #CP delivered.
- [ ] AC-6: PL3_SSP write: per-vCPU pointer updated; subsequent SYSCALL pushes to new SSP.
- [ ] AC-7: INT_SSP_TAB: per-vCPU table pointer set; interrupt-vector switches SSP per-table.
- [ ] AC-8: Live migration: CET state preserved.
- [ ] AC-9: kvm-unit-tests `cet` test passes.
- [ ] AC-10: Multi-vCPU: per-vCPU CET MSRs distinct.

## Architecture

Per-vCPU CET state in `vcpu.arch`:

```
struct VcpuArch {
  ...
  msr_ia32_s_cet: u64,
  msr_ia32_u_cet: u64,                          // (also in XSAVE)
  pl0_ssp: u64,                                  // (also in XSAVE)
  pl1_ssp: u64,
  pl2_ssp: u64,
  pl3_ssp: u64,                                  // (also in XSAVE)
  int_ssp_tab: u64,                              // (also in XSAVE)
}
```

`Vcpu::cet_supported(vcpu)`:
1. Return guest_cpuid_has(vcpu, X86_FEATURE_SHSTK) || guest_cpuid_has(vcpu, X86_FEATURE_IBT).

`Vcpu::handle_msr` MSR_IA32_S_CET branch:
1. WRMSR (guest):
   - If !cet_supported(vcpu): #GP.
   - Validate reserved bits.
   - vcpu.arch.msr_ia32_s_cet = data.
2. RDMSR (guest):
   - Return vcpu.arch.msr_ia32_s_cet.

`Vcpu::handle_msr` MSR_IA32_U_CET branch:
1. WRMSR: per-XSAVE-component update vcpu.arch.guest_fpu xstate.
2. RDMSR: return per-XSAVE component.

`Vcpu::handle_msr` MSR_IA32_PL[0-3]_SSP branch:
1. WRMSR: validate ssp 8-byte-aligned; update shadow.
2. RDMSR: return shadow.

`Vcpu::set_cr4` CR4.CET branch:
1. If new_cr4 & X86_CR4_CET && !cet_supported(vcpu): #GP.
2. Update CR4 normally.

`Mmu::permission_fault` (per-page-fault):
- Per-#CP from CET-violation: error_code includes CET_NEAR_RET / FAR_RET / ENDBR / etc.
- Inject #CP to guest via kvm_queue_exception_e.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cet_msr_reserved_bits_zero` | INVARIANT | per-MSR reserved-bits zero. |
| `pl_ssp_8byte_aligned` | INVARIANT | per-PL[N]_SSP value 8-byte-aligned. |
| `int_ssp_tab_8byte_aligned` | INVARIANT | INT_SSP_TAB pointer 8-byte-aligned. |
| `cet_cpuid_required_for_cr4` | INVARIANT | CR4.CET writable iff CPUID.SHSTK or .IBT. |
| `s_cet_u_cet_distinct` | INVARIANT | per-vCPU S_CET vs U_CET independent. |

### Layer 2: TLA+

`virt/kvm/cet_shadow_stack.tla`:
- Per-vCPU shadow-stack-pointer state.
- Properties:
  - `safety_shstk_match_at_ret` — per-RET requires SS-return-addr matches stack-return-addr.
  - `safety_endbr_at_indirect_target` — per-indirect-branch target must have ENDBR.
  - `safety_cp_on_violation` — per-violation #CP injected with correct error code.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Vcpu::handle_msr` S/U_CET post: per-MSR shadow updated; reserved-bits zero | `Vcpu::handle_msr` |
| `Vcpu::handle_msr` PL[N]_SSP post: 8-byte-aligned shadow updated | `Vcpu::handle_msr` |
| `Vcpu::set_cr4` CET post: CR4.CET writable iff CPUID-supported | `Vcpu::set_cr4` |
| Per-vCPU CET state in XSAVE: components 11/12 saved at vmexit | invariants on FPU swap |

### Layer 4: Verus/Creusot functional

`Per-CALL: shadow-stack push; per-RET: SS-vs-stack mismatch → #CP. Per-indirect-branch: ENDBR-required → missing causes #CP` semantic equivalence: per-instruction the CET semantics match Intel SDM CET spec.

## Hardening

(Inherits row-1 features from `virt/kvm/kvm-core.md` § Hardening.)

CET-specific reinforcement:

- **CR4.CET CPUID-gated** — defense against guest enabling without HW.
- **Per-MSR reserved-bits enforced** — defense against guest writing reserved.
- **PL_SSP 8-byte-aligned** — defense against unaligned causing CPU exception.
- **INT_SSP_TAB validated** — defense against table pointer to host-kernel-mem.
- **Per-vCPU CET in XSAVE** — defense against per-vCPU state leak across VMs.
- **#CP error-code propagated correctly** — defense against guest mis-attributing fault.
- **Live-migrate per-vCPU CET state** — defense against post-migrate broken protection.
- **CR4.CET requires CR0.WP=1** (per Intel-SDM) — defense against CET-bypass via ROP via writable kernel-pages.
- **WRSS / SETSSBSY / RSTORSSP intercept policy** — defense against guest abuse of supervisor-CET ops from user-mode.
- **CPUID 0x07.0:ECX[7] / EDX[20] gated by KVM module-param** — defense against unexpected CET-on per-VM.

## Grsecurity/PaX-style Reinforcement

Baseline hardening (always applied):

- **PAX_USERCOPY** — CET MSR get/set ioctl payloads bounded; no oversized SSP/INT_SSP_TAB write.
- **PAX_KERNEXEC** — CET MSR write handler table RO after init.
- **PAX_RANDKSTACK** — randomized kstack per KVM_RUN; CET-save/restore sees fresh stack.
- **PAX_REFCOUNT** — vCPU XSAVE-area pin refcount saturating.
- **PAX_MEMORY_SANITIZE** — guest CET XSAVE bytes (S_CET/U_CET/PL_SSP/INT_SSP_TAB) zeroed on vCPU destroy.
- **PAX_UDEREF** — INT_SSP_TAB and PL_SSP host-side validation rejects host-kernel hva.
- **PAX_RAP / kCFI** — #CP injection / WRSS-intercept callbacks type-checked.
- **GRKERNSEC_HIDESYM** — SSP / INT_SSP_TAB values redacted in trace.
- **GRKERNSEC_DMESG** — CET intercept floods rate-limited.

CET-specific:

- **CAP_SYS_ADMIN on KVM_SET_MSR(CET_*)** — defense against unprivileged CET MSR flip.
- **CR0.WP=1 hard-enforced before CR4.CET=1** — closes ROP-via-writable-kernel-pages bypass.
- **WRSS supervisor-only intercept** — guest user-mode WRSS abuse blocked.
- **Live-migrate CET XSAVE PAX_MEMORY_SANITIZE on source** — no shadow-stack pointer leak across VMs.

Rationale: CET shadow-stack state is high-value forensic data; sanitize on destroy + UDEREF on table pointers prevents shadow-stack-pointer leaks that would enable cross-VM ROP-chain reuse.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- KVM core (covered in `kvm-core.md` Tier-3)
- KVM FPU / XSAVE (covered in `x86-fpu.md` Tier-3)
- CR0 / CR4 (covered in `x86-cr0-cr4.md` Tier-3)
- Per-arch CET userspace API (arch/x86/kernel/cet.c; covered separately)
- AMD-Shadow-Stack (similar; covered in `x86-svm.md` future Tier-3)
- Implementation code
