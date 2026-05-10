---
title: "Tier-3: arch/x86/kvm/x86.c (CR0/CR4 subset) — Control-register virtualization (CR0/CR3/CR4 + paging-mode transitions + CR0/4 reserved-bits)"
tags: ["tier-3", "virt-kvm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Guest CR0/CR3/CR4 writes have side-effects (paging-mode change, NX-enable, SMEP/SMAP-enable, PCID-enable, etc.) that KVM must intercept, validate, and apply. Per-vCPU shadow `vcpu.arch.cr0/cr3/cr4`; vmcs/vmcb has GUEST_CR0 / GUEST_CR3 / GUEST_CR4 fields directly observable. KVM uses VMX's CR0_GUEST_HOST_MASK (vmcs.CR0_GUEST_HOST_MASK) + CR0_READ_SHADOW: bits in the mask cause vmexit on guest write; other bits guest can change directly. Per-write side-effects: kvm_post_set_cr0 / _cr4 update MMU mode, refresh CPUID-derived feature gating, recompute MSR bitmap.

This Tier-3 covers CR0/CR3/CR4 virtualization in `arch/x86/kvm/x86.c` (~400-500 lines).

### Acceptance Criteria

- [ ] AC-1: Boot Linux guest: BIOS sets CR0.PE → kernel sets CR0.PG → switch to long-mode via CR4.PAE + EFER.LMA + CR0.PG.
- [ ] AC-2: Guest WRMSR/MOV CR0 with reserved bits: #GP injected.
- [ ] AC-3: Guest enables SMEP via CR4.SMEP=1: kvm_post_set_cr4 refreshes MMU.
- [ ] AC-4: Guest enables PCIDE via CR4.PCIDE=1: TLB-invalidation policy changes.
- [ ] AC-5: Guest CR3 write with no-flush bit (PCID): TLB not flushed for that PCID.
- [ ] AC-6: LMSW: 16-bit guest BIOS uses LMSW; correctly sets low CR0 bits.
- [ ] AC-7: Live migration: CR0/CR3/CR4 round-trip; post-migrate guest sees same values.
- [ ] AC-8: CR0.WP toggle: guest sets WP=1; subsequent supervisor write to read-only page → #PF.
- [ ] AC-9: CR4.OSXSAVE toggle: guest enables; XSAVE feature visible in guest CPUID 0x01:ECX[27].
- [ ] AC-10: kvm-unit-tests `cr_access` test passes.

### Architecture

`Vcpu::set_cr0(vcpu, cr0)`:
1. Validate via is_valid_cr0(vcpu, cr0).
2. If !valid: inject #GP(0).
3. old_cr0 := vcpu.arch.cr0.
4. vcpu.arch.cr0 = cr0.
5. kvm_post_set_cr0(vcpu, old_cr0, cr0).
6. Vendor: vmx_set_cr0(vcpu, cr0) OR svm_set_cr0(vcpu, cr0).
7. Return 0.

`Vcpu::post_set_cr0(vcpu, old_cr0, cr0)`:
1. If (old_cr0 ^ cr0) & (X86_CR0_PG | X86_CR0_WP):
   - kvm_init_mmu(vcpu).
2. If (old_cr0 ^ cr0) & X86_CR0_PG:
   - vcpu.arch.efer effective LMA may change.
   - kvm_emulate_invalid_guest_state if invalid.
3. If (old_cr0 ^ cr0) & (X86_CR0_CD | X86_CR0_NW):
   - kvm_zap_gfn_range(vcpu.kvm, ...) — flush MMU memory-type cache.

`Vcpu::set_cr4(vcpu, cr4)`:
1. Validate via is_valid_cr4(vcpu, cr4).
2. If !valid: inject #GP(0).
3. old_cr4 := vcpu.arch.cr4.
4. vcpu.arch.cr4 = cr4.
5. kvm_post_set_cr4(vcpu, old_cr4, cr4).
6. Vendor: vmx_set_cr4(vcpu, cr4) OR svm_set_cr4(vcpu, cr4).

`Vcpu::is_valid_cr4(vcpu, cr4)`:
1. reserved_bits := compute per-vCPU reserved bits from CPUID.
2. If cr4 & reserved_bits: return false.
3. Per-feature checks:
   - PCIDE requires CR0.PG=1 long-mode.
   - OSXSAVE requires CR0.PE.
   - PKS requires PKE.
4. Return true.

`VcpuVmx::set_cr0(vcpu, cr0)`:
1. Compute hw_cr0 := cr0 | guest-must-have-bits & ~unrestricted-guest-bits.
2. VMWRITE GUEST_CR0 = hw_cr0.
3. VMWRITE CR0_READ_SHADOW = cr0.
4. Vmcs.CR0_GUEST_HOST_MASK reflects intercepted bits.

`VcpuSvm::set_cr0(vcpu, cr0)`:
1. svm.vmcb01.save.cr0 = cr0.
2. vmcb_mark_dirty(svm.vmcb01, CR).

### Out of Scope

- KVM core (covered in `kvm-core.md` Tier-3)
- VMX vendor (covered in `x86-vmx.md` Tier-3)
- SVM vendor (covered in `x86-svm.md` Tier-3)
- TDP MMU (covered in `x86-mmu-tdp.md` Tier-3)
- Shadow MMU (covered in `x86-mmu-shadow.md` Tier-3)
- EFER MSR (covered in `x86-vmx.md` / `x86-svm.md` Tier-3s)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `kvm_set_cr0(vcpu, cr0)` | guest CR0 write entry | `Vcpu::set_cr0` |
| `kvm_post_set_cr0(vcpu, old_cr0, cr0)` | post-CR0-set side-effects | `Vcpu::post_set_cr0` |
| `kvm_set_cr3(vcpu, cr3)` | guest CR3 write entry | `Vcpu::set_cr3` |
| `kvm_set_cr4(vcpu, cr4)` | guest CR4 write entry | `Vcpu::set_cr4` |
| `kvm_post_set_cr4(vcpu, old_cr4, cr4)` | post-CR4-set side-effects | `Vcpu::post_set_cr4` |
| `kvm_lmsw(vcpu, msw)` | LMSW instruction | `Vcpu::lmsw` |
| `kvm_is_valid_cr0(vcpu, cr0)` | per-CR0-bits validation | `Vcpu::is_valid_cr0` |
| `kvm_is_valid_cr4(vcpu, cr4)` | per-CR4-bits validation | `Vcpu::is_valid_cr4` |
| `kvm_read_cr0(vcpu)` / `_cr3(vcpu)` / `_cr4(vcpu)` | per-vCPU CR shadow read | `Vcpu::read_cr0` / `_cr3` / `_cr4` |
| `kvm_read_cr0_bits(vcpu, mask)` | per-bit-mask read | `Vcpu::read_cr0_bits` |
| `kvm_init_mmu(vcpu)` | per-vCPU MMU re-init on paging-mode change | `Mmu::init` |
| `vmx_set_cr0(vcpu, cr0)` (vmx.c) | VMX vmcs CR0 write | `VcpuVmx::set_cr0` |
| `vmx_set_cr4(vcpu, cr4)` | VMX vmcs CR4 write | `VcpuVmx::set_cr4` |
| `svm_set_cr0(vcpu, cr0)` (svm.c) | SVM vmcb CR0 write | `VcpuSvm::set_cr0` |
| `svm_set_cr4(vcpu, cr4)` | SVM vmcb CR4 write | `VcpuSvm::set_cr4` |
| `vmcs.CR0_GUEST_HOST_MASK` | which CR0 bits intercept | UAPI |
| `vmcs.CR0_READ_SHADOW` | guest-visible value of intercepted bits | UAPI |
| `vmcs.CR4_GUEST_HOST_MASK` | which CR4 bits intercept | UAPI |
| `vmcs.CR4_READ_SHADOW` | guest-visible CR4-bits | UAPI |

### compatibility contract

REQ-1: CR0 fields:
- Bit[0] PE (Protection Enable).
- Bit[1] MP (Monitor coProcessor).
- Bit[2] EM (Emulation).
- Bit[3] TS (Task Switched).
- Bit[4] ET (Extension Type).
- Bit[5] NE (Numeric Error).
- Bit[16] WP (Write Protect).
- Bit[18] AM (Alignment Mask).
- Bit[29] NW (Not Writethrough).
- Bit[30] CD (Cache Disable).
- Bit[31] PG (Paging).

REQ-2: CR0 write semantics:
1. Validate: PG requires PE.
2. Validate: PG enabled requires CR4.PAE if not 64-bit.
3. Validate: CR0.NW + CR0.CD combo.
4. Validate: KVM_X86_QUIRK_CR0_*: per-quirk handling.
5. Update vcpu.arch.cr0.
6. kvm_post_set_cr0:
   - If paging-mode change: kvm_init_mmu(vcpu).
   - If WP change: refresh per-vCPU MMU permission-tables.
   - If CD/NW change: refresh memory-type SPTE bits.
7. Vendor: vmcs.GUEST_CR0 / vmcb.cr0 = cr0.

REQ-3: CR4 fields:
- Bit[0] VME / Bit[1] PVI (legacy).
- Bit[2] TSD (Time-Stamp Disable).
- Bit[3] DE (Debugging Extensions).
- Bit[4] PSE (Page-Size Extension).
- Bit[5] PAE (Physical Address Extension).
- Bit[6] MCE (Machine-Check Enable).
- Bit[7] PGE (Page Global Enable).
- Bit[8] PCE (Performance Counter Enable).
- Bit[9] OSFXSR (XMM enable).
- Bit[10] OSXMMEXCPT.
- Bit[13] VMXE (Vmx Enable).
- Bit[14] SMXE (SMX Enable).
- Bit[16] FSGSBASE.
- Bit[17] PCIDE (PCID Enable).
- Bit[18] OSXSAVE.
- Bit[19] KL (Key-Locker).
- Bit[20] SMEP.
- Bit[21] SMAP.
- Bit[22] PKE (Protection-Key Enable).
- Bit[23] CET.
- Bit[24] PKS (Protection-Key Supervisor).
- Bit[25] UINTR.
- Bit[27] LAM_SUP.

REQ-4: CR4 write semantics:
1. Validate: bits ⊆ vcpu.arch.guest_supported_xcr4.
2. Validate: per-bit dependencies (PCIDE requires CR0.PG=1 long-mode; OSXSAVE requires CR0.PE; etc.).
3. Update vcpu.arch.cr4.
4. kvm_post_set_cr4:
   - If PAE change: kvm_init_mmu.
   - If PGE change: invalidate per-vCPU TLB (global pages).
   - If PCIDE change: invalidate TLB.
   - If OSXSAVE change: refresh per-vCPU CPUID bits visible to guest.
5. Vendor: vmcs.GUEST_CR4 / vmcb.cr4 = cr4.

REQ-5: CR3 write semantics:
1. Validate: low bits ≤ MAXPHYADDR.
2. Validate: PCID format (bit-63 = preserve-CR3-no-flush; lower 12 bits = PCID).
3. Update vcpu.arch.cr3.
4. mmu_load_pgd(vcpu): re-program TDP/shadow page-table root.
5. Vendor: vmcs.GUEST_CR3 / vmcb.cr3 = cr3.

REQ-6: CR0/CR4 reserved-bits per CPUID:
- Per-vCPU.arch.cr0_guest_owned_bits (bits guest can write directly).
- Per-vCPU.arch.cr4_guest_owned_bits.
- Reserved-bits (per CPUID-not-supported features) writes raise #GP.

REQ-7: VMX intercept policy:
- vmcs.CR0_GUEST_HOST_MASK: 1-bit = intercept on write.
- vmcs.CR0_READ_SHADOW: guest sees this when reading intercepted bit.
- KVM intercepts: CR0.PE / .PG / .NW / .CD / WP changes.
- KVM lets guest directly write: TS, MP, EM, ET, NE, AM.

REQ-8: SVM intercept policy:
- vmcb.control.intercept_cr: per-bit intercept-on-write CR0..7.
- KVM sets bit-0 (CR0 write), bit-3 (CR3 write), bit-4 (CR4 write).

REQ-9: Per-vmexit handler for CR write:
- VMX: VMEXIT_REASON_CR_ACCESS; KVM dispatches based on CR-num + access-type.
- SVM: VMEXIT_REASON_CR0_WRITE / _CR3_WRITE / _CR4_WRITE.

REQ-10: kvm_lmsw (LMSW instruction):
- 16-bit immediate; sets low 4 bits of CR0 (PE / MP / EM / TS).
- Used by 16-bit/32-bit guest BIOS startup.

REQ-11: Live migration:
- Per-vCPU CR0/CR3/CR4 migrated.
- Validated against destination CPUID + features.

REQ-12: Per-vmenter consistency check:
- Per-spec, vmenter requires CR0.PE=1 for non-real-mode; CR0.PG ↔ EFER.LMA / CR4.PAE compatibility.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cr0_pg_implies_pe` | INVARIANT | per-vCPU !(cr0 & PG && !(cr0 & PE)). |
| `cr4_features_subset_cpuid` | INVARIANT | cr4 bits ⊆ CPUID-supported features. |
| `cr3_within_maxphyaddr` | INVARIANT | per-vCPU cr3 high-bits == 0 above MAXPHYADDR. |
| `intercept_mask_consistent` | INVARIANT | vmcs.CR0_GUEST_HOST_MASK includes all bits KVM needs to virtualize. |
| `post_set_mmu_init` | INVARIANT | paging-mode change triggers kvm_init_mmu. |

### Layer 2: TLA+

`virt/kvm/cr0_cr4_transitions.tla`:
- Per-vCPU paging-mode state: real-mode / pmode / long-mode.
- Per-bit transitions allowed per Intel SDM.
- Properties:
  - `safety_paging_only_with_pe` — PG requires PE.
  - `safety_long_mode_requires_pae` — long-mode implies PAE.
  - `safety_efer_lma_consistent` — EFER.LMA matches CR0.PG && CR4.PAE && CS.L.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Vcpu::set_cr0` post: vcpu.arch.cr0 updated; vmcs/vmcb refreshed; MMU re-init if needed | `Vcpu::set_cr0` |
| `Vcpu::set_cr4` post: vcpu.arch.cr4 updated; per-bit-feature side-effects applied | `Vcpu::set_cr4` |
| `Vcpu::set_cr3` post: vcpu.arch.cr3 updated; mmu_load_pgd called; TLB flushed per PCID semantics | `Vcpu::set_cr3` |
| Per-vCPU.arch.cr0/cr4 mirror vmcs.GUEST_CR0/CR4_READ_SHADOW exactly | invariants on set/get |

### Layer 4: Verus/Creusot functional

`Per-CR-write: guest sees new value via subsequent MOV CR; KVM-side state matches HW vmcs/vmcb` semantic equivalence: per-write the guest-observable + HW-visible state synchronized.

### hardening

(Inherits row-1 features from `virt/kvm/kvm-core.md` § Hardening.)

CR-virtualization-specific reinforcement:

- **Per-CR0/CR4 reserved-bits enforced** — defense against guest writing reserved bits causing CPU exception.
- **Per-bit dependency check** — defense against partial-feature config (e.g., PCIDE without long-mode).
- **PG/PE consistency** — defense against PG=1 PE=0 causing #GP at vmenter.
- **Per-paging-mode change MMU-re-init** — defense against stale SPTE after mode change.
- **Per-PCID write no-flush bit honored** — defense against unnecessary TLB flush.
- **Per-vCPU intercept-mask reflects KVM-virtualized bits** — defense against guest bypassing intercept on critical bit.
- **CR3 high-bits validated against MAXPHYADDR** — defense against physical-addr OOB.
- **Per-CR4.VMXE set requires guest VMX-cap CPUID** — defense against guest-VMX without explicit support.
- **kvm_lmsw bounded to 4 bits** — defense against partial CR0-modification leaking.
- **Per-WP toggle MMU-permission-refresh** — defense against stale supervisor-write-protect.
- **Live-migrate CR0/CR3/CR4 validated against destination CPUID** — defense against post-migrate invalid state.
- **Per-vmenter consistency check** — defense against guest entering with invalid CR-state.

