# Tier-3: arch/x86/kvm/{vmx/vmx.c, svm/svm.c} (MSR-bitmap passthrough subset) — KVM per-vCPU MSR access intercept policy

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/x86-msr.md
upstream-paths:
  - arch/x86/kvm/vmx/vmx.c (msr_bitmap mgmt: ~3026, ~3044, ~4059..4129, ~2331, ~2418..2423, ~4080..4099)
  - arch/x86/kvm/svm/svm.c (svm_set_msr_interception_bitmap, MSRPM)
  - arch/x86/include/asm/svm.h (MSRPM layout)
-->

## Summary

VMX uses `vmcs.msr-bitmap` (4KB per-vCPU page) and SVM uses `vmcb.msrpm` (8KB per-vCPU page) to selectively passthrough or intercept guest RDMSR/WRMSR. Per-MSR bit set: vmexit on access; clear: HW direct access without exit. Critical for: hot-MSR perf (passthrough TSC, FS_BASE, GS_BASE, etc.) while intercepting security/policy-relevant MSRs (MTRR, IA32_DEBUGCTL when LBR off, IA32_PRED_CMD for spec-control-policy, SYSCALL/SYSENTER MSRs for syscall hardening). Per-vCPU dynamic toggling (e.g. enable LBR-passthrough only after guest enables LBR via DEBUGCTL).

This Tier-3 covers MSR-bitmap mgmt subset of vmx.c + svm.c (~150 lines combined).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `loaded_vmcs->msr_bitmap` | per-VMX msr-bitmap page | `LoadedVmcs::msr_bitmap` |
| `vmx_disable_intercept_for_msr()` | per-MSR clear bitmap bit | `VmxOps::disable_intercept_for_msr` |
| `vmx_enable_intercept_for_msr()` | per-MSR set bitmap bit | `VmxOps::enable_intercept_for_msr` |
| `vmx_set_msr_bitmap_read()` / `write()` | per-MSR per-direction set | `VmxOps::set_msr_bitmap_*` |
| `vmx_clear_msr_bitmap_read()` / `write()` | per-MSR per-direction clear | `VmxOps::clear_msr_bitmap_*` |
| `vmx_test_msr_bitmap_read()` / `write()` | per-MSR test | `VmxOps::test_msr_bitmap_*` |
| `vmx_msr_bitmap_l01_changed()` | per-vCPU dirty-flag for nested | `VmxOps::msr_bitmap_l01_changed` |
| `vmx_update_msr_bitmap_x2apic()` | per-x2APIC MSR rebuild | `VmxOps::update_msr_bitmap_x2apic` |
| `cpu_has_vmx_msr_bitmap()` | per-host capability | `VmxOps::cpu_has_msr_bitmap` |
| `set_msr_interception()` (SVM) | per-MSR per-direction set | `SvmOps::set_msr_interception` |
| `svm_set_msr_interception_bitmap()` | per-MSRPM update | `SvmOps::set_msr_interception_bitmap` |

## Compatibility contract

REQ-1: VMX msr-bitmap layout (4096 bytes = 4 × 1024 bytes):
- Bytes [0..1024]: read-bitmap for MSRs [0x00000000..0x00001FFF].
- Bytes [1024..2048]: read-bitmap for MSRs [0xC0000000..0xC0001FFF].
- Bytes [2048..3072]: write-bitmap for MSRs [0x00000000..0x00001FFF].
- Bytes [3072..4096]: write-bitmap for MSRs [0xC0000000..0xC0001FFF].
- Per-bit: 1 = intercept (vmexit); 0 = passthrough.

REQ-2: SVM msrpm layout (8192 bytes = 8 × 1024 bytes; 2 bits per MSR):
- Each MSR has 2 bits: bit-0 = read-intercept; bit-1 = write-intercept.
- 4 ranges of 8192 MSRs (covering 0x0000_0000..0x0000_1FFF and 0xC000_0000..0xC000_1FFF and 0xC001_0000..0xC001_1FFF and 0xDA00_0000..0xDB00_1FFF).

REQ-3: Per-vCPU msr_bitmap allocation:
- Per-vmx alloc_loaded_vmcs: alloc 4KB page; default fill 0xff (all-intercept).
- Per-vCPU init: per-MSR optional disable_intercept_for_msr (passthrough policy).

REQ-4: Default passthrough MSRs (VMX):
- MSR_IA32_TSC (RDTSC fast).
- MSR_FS_BASE / MSR_GS_BASE / MSR_KERNEL_GS_BASE.
- MSR_IA32_SYSENTER_CS / EIP / ESP.
- MSR_LSTAR / MSR_CSTAR / MSR_STAR / MSR_SYSCALL_MASK.
- MSR_IA32_SPEC_CTRL (R only; W intercepted for IBPB tracking).
- MSR_IA32_PRED_CMD (W only).
- Per-PMU MSRs when PMU enabled.

REQ-5: Default-intercept MSRs (VMX):
- All MTRR MSRs.
- All architectural-PMU MSRs when PMU disabled.
- Microcode update MSR (MSR_IA32_UCODE_REV / MSR_IA32_UCODE_WRITE).
- IA32_DEBUGCTLMSR (intercepted by default; passthrough when LBR enabled).
- All Hyper-V emulated MSRs.
- All PV-clock MSRs (always intercepted; emulated by KVM).

REQ-6: vmx_disable_intercept_for_msr(vcpu, msr, type):
- type & MSR_TYPE_R: clear read-bit.
- type & MSR_TYPE_W: clear write-bit.
- vmx_msr_bitmap_l01_changed(vmx) → marks nested.force_msr_bitmap_recalc = true.

REQ-7: vmx_enable_intercept_for_msr(vcpu, msr, type):
- Set read/write bits.
- Same dirty-flag.

REQ-8: Per-x2APIC mode toggling:
- Per-x2APIC: passthrough APIC MSRs 0x800..0x83F (or subset for x2apic-virtualization).
- Per-xAPIC: intercept all 0x800..0x83F.
- vmx_update_msr_bitmap_x2apic invoked on lapic-mode change.

REQ-9: Per-nested vmcs02:
- L0 builds vmcs02.msr_bitmap from L1's vmcs12.msr_bitmap AND vmcs01.msr_bitmap.
- Per-MSR: vmcs02 intercepts iff L1-intercepts ∨ L0-intercepts.

REQ-10: SVM MSRPM:
- svm_set_msr_interception(vcpu, msr, read, write) updates 2 bits per MSR.
- Per-vmcb allocated msrpm page; vmcb.control.msrpm_base_pa = phys(msrpm).

## Acceptance Criteria

- [ ] AC-1: Boot Linux guest: vmcs.msr-bitmap-pa set; tracepoint vmexit count for RDMSR low.
- [ ] AC-2: Guest RDMSR(MSR_IA32_TSC): no vmexit (default passthrough).
- [ ] AC-3: Guest WRMSR(MSR_IA32_PRED_CMD, IBPB): no vmexit; WRMSR completes.
- [ ] AC-4: Guest WRMSR(MSR_MTRRdefType): vmexit; KVM emulates.
- [ ] AC-5: Guest enables LBR via DEBUGCTLMSR.LBR=1: vmx_passthrough_lbr_msrs disables intercept for LBR_FROM/TO.
- [ ] AC-6: Per-x2APIC switch: APIC MSRs 0x800..0x83F passthrough.
- [ ] AC-7: Per-VM with PMU disabled: PMU MSRs intercepted; #GP injected.
- [ ] AC-8: Per-VM with PMU enabled: PMU MSRs passthrough.
- [ ] AC-9: Per-nested L2 RDMSR: bitmap is OR(L1, L0) intercept.
- [ ] AC-10: SVM: svm_set_msr_interception(MSR_IA32_TSC, false, false) clears 2 MSRPM bits.

## Architecture

Per-vCPU loaded_vmcs:

```
struct LoadedVmcs {
  vmcs: PhysAddr,
  msr_bitmap: Box<[u8; 4096]>,                    // VMX: 4KB
  ...
}
```

Per-VMX-vCPU:

```
struct VmxVcpu {
  ...
  loaded_vmcs: &mut LoadedVmcs,
  shadow_msr_intercept: ShadowMsrIntercept,       // per-MSR shadow tracking
  x2apic_msr_bitmap_mode: X2ApicMsrBitmapMode,
  guest_uret_msrs: GuestUretMsrs,
}
```

Per-SVM-vCPU:

```
struct SvmVcpu {
  ...
  msrpm: Box<[u8; 8192]>,                         // SVM: 8KB
  msrpm_pa: PhysAddr,
}
```

`VmxOps::msr_bitmap_offsets(msr) -> (range_offset, byte_idx, bit_idx)`:
1. If msr in [0..0x1FFF]: range_offset = 0; bit = msr.
2. If msr in [0xC0000000..0xC0001FFF]: range_offset = 1024; bit = msr - 0xC0000000.
3. Else: panic (invalid MSR for bitmap).
4. byte_idx = bit / 8; bit_idx = bit % 8.

`VmxOps::set_msr_bitmap_read(bitmap, msr)`:
1. (range_offset, byte_idx, bit_idx) = msr_bitmap_offsets(msr).
2. bitmap[range_offset + byte_idx] |= 1 << bit_idx.

`VmxOps::clear_msr_bitmap_read(bitmap, msr)`:
1. bitmap[range_offset + byte_idx] &= !(1 << bit_idx).

`VmxOps::set_msr_bitmap_write(bitmap, msr)`:
1. Same as read but +2048 offset.

`VmxOps::disable_intercept_for_msr(vcpu, msr, type)`:
1. bitmap = vcpu.loaded_vmcs.msr_bitmap.
2. If type & MSR_TYPE_R: clear_msr_bitmap_read(bitmap, msr).
3. If type & MSR_TYPE_W: clear_msr_bitmap_write(bitmap, msr).
4. msr_bitmap_l01_changed(vmx).

`VmxOps::msr_bitmap_l01_changed(vmx)`:
1. vmx.nested.force_msr_bitmap_recalc = true.
2. If vmx is per-Hyper-V evmcs: evmcs->hv_clean_fields &= !MSR_BITMAP_CLEAN.

`VmxOps::update_msr_bitmap_x2apic(vcpu)`:
1. mode = vmx.x2apic_msr_bitmap_mode.
2. If mode == DISABLED: enable_intercept all 0x800..0x83F.
3. If mode == X2APIC ∨ X2APIC_APICV: disable_intercept select APIC MSRs.

`SvmOps::set_msr_interception(vcpu, msr, read_intercept, write_intercept)`:
1. (offset, bit_pos) = svm_msrpm_offset(msr).
2. byte = svm.msrpm[offset].
3. byte &= !(0b11 << bit_pos).
4. If read_intercept: byte |= (1 << bit_pos).
5. If write_intercept: byte |= (1 << (bit_pos + 1)).
6. svm.msrpm[offset] = byte.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `bitmap_size_eq_4kb_vmx` | INVARIANT | per-VMX msr_bitmap 4096 bytes. |
| `msrpm_size_eq_8kb_svm` | INVARIANT | per-SVM msrpm 8192 bytes. |
| `bitmap_offset_in_range` | INVARIANT | per-set/clear: byte_idx + range_offset < bitmap_size. |
| `default_intercept_all` | INVARIANT | per-init bitmap memset 0xff. |
| `disable_intercept_clears_bit` | INVARIANT | post-disable_intercept_for_msr: bit clear. |

### Layer 2: TLA+

`virt/kvm/msr_passthrough.tla`:
- Per-MSR per-vCPU intercept-state lattice + per-vmenter bitmap loaded.
- Properties:
  - `safety_passthrough_msr_no_vmexit` — per-MSR with bit clear ⟹ guest WRMSR/RDMSR doesn't vmexit.
  - `safety_intercepted_msr_vmexits` — per-MSR with bit set ⟹ guest WRMSR/RDMSR vmexits.
  - `safety_nested_l2_more_strict` — per-MSR vmcs02-intercept ⊇ vmcs01-intercept ∪ vmcs12-intercept.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `VmxOps::disable_intercept_for_msr` post: bitmap-bit clear; dirty-flag set | `VmxOps::disable_intercept_for_msr` |
| `VmxOps::enable_intercept_for_msr` post: bitmap-bit set; dirty-flag set | `VmxOps::enable_intercept_for_msr` |
| `VmxOps::update_msr_bitmap_x2apic` post: APIC-MSR per-mode policy applied | `VmxOps::update_msr_bitmap_x2apic` |
| `SvmOps::set_msr_interception` post: 2 MSRPM bits updated atomically | `SvmOps::set_msr_interception` |

### Layer 4: Verus/Creusot functional

`Per-MSR with intercept-bit clear → guest direct HW access; per-set → vmexit + KVM handler` semantic equivalence: per-MSR-bitmap matches Intel VT-x SDM §25.1.3 + AMD APM-Vol2 §15.11.

## Hardening

(Inherits row-1 features from `virt/kvm/x86-msr.md` § Hardening.)

MSR-passthrough-specific reinforcement:

- **Default-intercept all** — defense against per-MSR forgotten intercept exposing host MSR.
- **Per-MSR offset bounds-checked** — defense against out-of-range MSR causing OOB write.
- **Per-x2APIC mode-toggling validated** — defense against per-mode-switch race exposing wrong MSRs.
- **Per-vmcs02 OR(L1, L0) recompute** — defense against L2 escaping L0 intercept policy.
- **Per-evmcs Hyper-V dirty-flag** — defense against per-evmcs partial-update.
- **Per-LBR passthrough only when guest-LBR-enabled** — defense against guest accidentally accessing host LBR state.
- **Per-PMU passthrough only when PMU advertised** — defense against per-MSR access without PMU CPUID.
- **Per-PRED_CMD W passthrough only** — defense against guest reading host SPEC_CTRL state.
- **Per-x2APIC APIC-MSR passthrough requires APICv** — defense against missing APICv-required infra.
- **Per-VMX MSR-bitmap-page locked in host phys-addr** — defense against host swap moving page mid-vmenter.
- **Per-init memset 0xff guards future MSRs** — defense against forward-MSR-leak when host kernel adds new MSRs.

## Grsecurity/PaX-style Reinforcement

Baseline hardening (always applied):

- **PAX_USERCOPY** — bitmap pages not user-touched; assertion enforced.
- **PAX_KERNEXEC** — vmx/svm bitmap-flip helpers RO after init.
- **PAX_RANDKSTACK** — randomized kstack per vmenter.
- **PAX_REFCOUNT** — bitmap-page ref + per-vCPU passthrough slot refcount saturating.
- **PAX_MEMORY_SANITIZE** — bitmap page memset(0xff) at alloc; memset(0xff) at vCPU destroy (default-intercept).
- **PAX_UDEREF** — no user pointer in bitmap path; assertion.
- **PAX_RAP / kCFI** — bitmap update callbacks type-checked.
- **GRKERNSEC_HIDESYM** — bitmap PA / MSR addresses redacted.
- **GRKERNSEC_DMESG** — passthrough-flip floods rate-limited.

Passthrough-specific:

- **CAP_SYS_ADMIN on KVM_RUN** — privileged VMM only.
- **PRED_CMD passthrough write-only** — guest cannot read host SPEC_CTRL.
- **x2APIC passthrough gated by APICv enable** — defense against missing infra.
- **memset(0xff) on init guards future-MSR forward-leak**.

Rationale: MSR passthrough is the most dangerous default — sanitize-to-deny on alloc + destroy ensures any forgotten MSR defaults to intercept, blocking accidental host-MSR exposure.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- KVM MSR core handling (covered in `x86-msr.md` Tier-3)
- KVM MSR-filter (covered in `x86-msr-filter.md` Tier-3)
- KVM PMU MSRs (covered in `x86-pmu.md` Tier-3)
- KVM LBR MSRs (covered in `x86-vmx-lbr.md` Tier-3)
- Implementation code
