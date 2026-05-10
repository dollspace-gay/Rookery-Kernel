---
title: "Tier-3: arch/x86/kvm/smm.c — KVM SMM (System Management Mode) emulation (vCPU SMM-entry/exit + state-save area)"
tags: ["tier-3", "virt-kvm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

SMM (System Management Mode) is x86's highest-privilege CPU mode entered via SMI (System Management Interrupt) — used by firmware for power-management, ECC handling, hot-plug, ACPI, secure-boot, etc. KVM emulates SMM for guest BIOS/firmware (e.g., qemu's SMI handler in OVMF for secure-boot Variable storage). Per SMI: vCPU saves complete CPU state to SMRAM at SMBASE+0xFE00 + sets CPU mode to "SMM" + jumps to SMI handler at SMBASE+0x8000. Per RSM: vCPU restores from SMRAM + returns to pre-SMI state.

This Tier-3 covers `arch/x86/kvm/smm.c` (~663 lines) + `smm.h` (~171 lines).

### Acceptance Criteria

- [ ] AC-1: KVM_CAP_X86_SMM advertised to userspace; verify via KVM_CHECK_EXTENSION.
- [ ] AC-2: OVMF UEFI guest with secure-boot: BIOS variable writes trigger SMI; SMI handler runs; secure-boot variables persist correctly.
- [ ] AC-3: KVM_INJECT_SMI: userspace sends SMI; vCPU enters SMM; CS:EIP at SMBASE+0x8000.
- [ ] AC-4: Per-vCPU SMBASE relocation: SMI handler writes new SMBASE; subsequent SMI uses new SMBASE.
- [ ] AC-5: 32-bit + 64-bit save-area: 32-bit guest SMI uses 32-bit save layout; 64-bit guest uses 64-bit.
- [ ] AC-6: Live migration: in-SMM vCPU migrated correctly; resumes SMI handler post-migrate.
- [ ] AC-7: SMI vs NMI priority: pending NMI + SMI; SMI taken first; NMI taken on RSM.
- [ ] AC-8: kvm-unit-tests `smm` test passes.
- [ ] AC-9: Spec-violation defense: malformed save-area (invalid CR0 PG=1 PE=0) raises #GP at RSM.

### Architecture

`KvmSmm` per-vCPU additions:

```
struct VcpuArch {
  ...
  hflags: u32,                                // HF_SMM_MASK + others
  smbase: u32,                                // per-vCPU SMBASE (default 0x30000)
  smi_pending: bool,
  smi_count: u64,
}
```

`Vcpu::enter_smm` flow:
1. smram_addr = vcpu.arch.smbase + 0xFE00.
2. If is_long_mode(vcpu):
   - save_state_64(vcpu, smram_addr).
3. Else:
   - save_state_32(vcpu, smram_addr).
4. Pre-SMM intercept setup via vendor:
   - Set CR0 to SMM-mode value (typically PE=1 PG=0; KVM uses flat 32-bit emulation by setting CR0.PE=1, CR0.PG=0; reset CR4 features).
   - Set segments to flat:
     - CS.base = vcpu.arch.smbase; CS.selector = vcpu.arch.smbase >> 4; CS.limit = 0xFFFFFFFF.
     - DS/ES/SS/FS/GS.base = 0; .limit = 0xFFFFFFFF; .selector = 0.
   - EIP = 0x8000.
   - EFER = 0 (no LMA in SMM).
   - DR7 = 0x400.
   - Clear EFLAGS.IF, RF, TF, VM.
5. vcpu.arch.hflags |= HF_SMM_MASK.
6. vcpu.arch.smi_count++.
7. vendor->kvm_smm_changed(vcpu, true) — VMX/SVM update intercepts to recognize SMM-context (e.g., let RSM emulate, etc.).

`Emulator::leave_smm(ctxt)` (RSM-instruction emulation):
1. smram_addr = vcpu.arch.smbase + 0xFE00.
2. Determine SMM-entry was 32-bit or 64-bit from save-area CPU-mode field.
3. If 64-bit-entry: load_state_64(vcpu, smram_addr).
4. Else: load_state_32(vcpu, smram_addr).
5. vcpu.arch.smbase = save-area-SMBASE-field (allow relocation).
6. vcpu.arch.hflags &= ~HF_SMM_MASK.
7. vendor->kvm_smm_changed(vcpu, false).

`Vcpu::process_smi(vcpu)`:
1. If !vcpu.arch.smi_pending: return.
2. vcpu.arch.smi_pending = false.
3. enter_smm(vcpu).

### Out of Scope

- KVM core (covered in `kvm-core.md` Tier-3)
- VMX vendor (covered in `x86-vmx.md` Tier-3)
- SVM vendor (covered in `x86-svm.md` Tier-3)
- x86 emulator (covered in `x86-emulate.md` Tier-3)
- LAPIC (covered in `x86-lapic.md` Tier-3)
- STM (SMI-Transfer-Monitor) — out-of-scope at this Tier-3
- OVMF UEFI integration (firmware concern)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `enter_smm(vcpu)` | SMI entry: save state to SMRAM + flip mode | `Vcpu::enter_smm` |
| `emulator_leave_smm(ctxt)` (RSM) | RSM exit: restore state from SMRAM | `Emulator::leave_smm` |
| `process_smi(vcpu)` | SMI request handler | `Vcpu::process_smi` |
| `kvm_smm_changed(vcpu, entering_smm)` | post-state-change hook (vendor) | `Vcpu::smm_changed` |
| `kvm_inject_smi(vcpu)` | userspace request SMI inject | `Vcpu::inject_smi` |
| `kvm_x86_ops.send_smi(vcpu)` (vendor) | per-vendor SMI delivery | `vendor::send_smi` |
| `is_smm(vcpu)` | per-vCPU SMM-mode query | `Vcpu::is_smm` |
| `kvm_smm_init(vm, region)` | per-VM SMRAM allocation | `KvmSmm::init` |
| `enter_smm_save_state_64(vcpu, smram)` | 64-bit save area construction | `Vcpu::save_state_64` |
| `enter_smm_save_state_32(vcpu, smram)` | 32-bit save area construction | `Vcpu::save_state_32` |
| `rsm_load_state_64(ctxt, smram)` | 64-bit restore | `Emulator::load_state_64` |
| `rsm_load_state_32(ctxt, smram)` | 32-bit restore | `Emulator::load_state_32` |
| `KVM_SMM_CMD_SET_LBR` (vendor-LBR-save quirk) | per-vendor LBR-MSR-save | `vendor::smm_lbr` |

### compatibility contract

REQ-1: Per-vCPU SMM state:
- `vcpu.arch.hflags & HF_SMM_MASK` (in-SMM flag).
- `vcpu.arch.smbase` (per-vCPU SMBASE; per-Intel-SDM default 0x30000; relocatable via SMM-handler).
- `vcpu.arch.smi_pending` (deferred-SMI flag).
- `vcpu.arch.smi_count` (lifetime SMI counter; for migration).

REQ-2: SMRAM layout:
- Save area at SMBASE+0xFE00 to SMBASE+0xFFFF (512 bytes for 32-bit, expanded for 64-bit).
- SMI handler entry at SMBASE+0x8000 (typical; firmware-defined).
- Per-vCPU per-VM dedicated SMRAM page (host-allocated).

REQ-3: SMI entry sequence (`enter_smm`):
1. Save current CPU state into 32-bit or 64-bit save area at SMBASE+0xFE00.
   - 32-bit save: ~512 bytes; per Intel SDM Vol 3 Sec 31.4.
   - 64-bit save: ~512 bytes; long-mode save area; per AMD APM (Intel uses similar but slightly different layout; KVM follows Intel SDM rev).
2. Set vcpu.arch.hflags |= HF_SMM_MASK.
3. Set vcpu.arch.cr0 = SMM-entry CR0 (typically PE=0, but with KVM's flat-mode emulation: CR0 keeps PE for performance; flat-segment).
4. Reset segment registers to flat 32-bit: CS = 0x3000 (SMBASE >> 4), EIP = 0x8000.
5. Disable interrupts: clear EFLAGS.IF, clear IDTR.
6. Set vCPU run-state to executing SMI handler.
7. vendor->kvm_smm_changed callback (VMX/SVM update intercepts).

REQ-4: RSM exit sequence (`emulator_leave_smm`):
1. Read save area at SMBASE+0xFE00.
2. Restore CR0/CR3/CR4/EFER/segments/GPRs from save area.
3. Clear vcpu.arch.hflags & HF_SMM_MASK.
4. vendor->kvm_smm_changed callback.
5. Resume guest at saved CS:EIP.

REQ-5: SMBASE relocation:
- During SMI handler, firmware may write to save-area offset 0xFEF8 (SMBASE field) to relocate per-vCPU SMBASE for next SMI.
- On RSM: vcpu.arch.smbase = save-area-SMBASE.

REQ-6: SMI inject from userspace (KVM_INJECT_SMI ioctl):
1. vcpu.arch.smi_pending = true.
2. kvm_make_request(KVM_REQ_SMI, vcpu).
3. kvm_vcpu_kick(vcpu).
4. Next vmenter: process_smi clears pending + executes enter_smm.

REQ-7: SMI ⇄ NMI/IRQ priority:
- SMI has higher priority than NMI per Intel SDM.
- If both pending: SMI delivered first.

REQ-8: Migration:
- KVM_GET_MSRS includes MSR_SMI_COUNT.
- KVM_GET_VCPU_EVENTS includes smi_pending + in-SMM-flag.
- Save-area memory part of guest RAM (handled via memslot).

REQ-9: Vendor-specific:
- Intel (VMX): SMI-Transfer-Monitor (STM) support gated; KVM advertises HF_SMM_MASK guest-visible via separate CPUID enumeration.
- AMD (SVM): smm_save / smm_restore hooks.

REQ-10: Per-VM SMI gate:
- KVM_CAP_X86_SMM enables SMM emulation for VM.
- If disabled: SMI inject returns -EINVAL.

REQ-11: STM (SMI-Transfer-Monitor) - Intel:
- Allows VMM to interpose between SMI and SMI handler (super-firmware-isolation).
- Out-of-scope at this Tier-3 level.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `smram_addr_aligned` | INVARIANT | smbase + 0xFE00 4KiB-aligned; defense against save-area straddling page boundary mid-write. |
| `save_area_size_bounded` | OOB | save-area write bounded by 512 bytes (32) or 1024 bytes (64); within memslot. |
| `smi_count_monotonic` | INVARIANT | smi_count monotonically increases per SMI. |
| `hflags_smm_mask_atomic` | INVARIANT | HF_SMM_MASK set/clear paired exactly with state save/restore. |

### Layer 2: TLA+

`virt/kvm/smm_state.tla`:
- States: NormalMode, EnteringSmm, InSmm, ExitingSmm, NormalMode'.
- Transitions per SMI delivery + RSM execution.
- Properties:
  - `safety_state_save_complete_before_in_smm` — InSmm only after full save-area write.
  - `safety_state_restore_complete_before_normal` — NormalMode' only after full save-area read.
  - `safety_smi_count_increments_at_entry`.
  - `liveness_in_smm_eventually_exits` — assuming SMI handler executes RSM.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Vcpu::enter_smm` post: hflags has HF_SMM_MASK; CS.base == smbase; EIP == 0x8000 | `Vcpu::enter_smm` |
| `Emulator::leave_smm` post: hflags HF_SMM_MASK clear; saved CS:EIP restored | `Emulator::leave_smm` |
| `Vcpu::process_smi` post: smi_pending == false; if was true, smi_count incremented | `Vcpu::process_smi` |
| Per-vCPU smbase always 32KiB-aligned per Intel-SDM | `Vcpu::enter_smm` |

### Layer 4: Verus/Creusot functional

`Per-SMI: enter_smm(vcpu) + handler-execution + RSM → vcpu state matches pre-SMI state` round-trip equivalence: post-RSM all GPR/segments/flags/CR0-4 match pre-SMI snapshot exactly (modulo handler-mediated state changes that handler explicitly stored).

### hardening

(Inherits row-1 features from `virt/kvm/kvm-core.md` § Hardening.)

SMM-specific reinforcement:

- **Save-area in IOMMU-protected memslot** — defense against device-DMA writing to SMRAM mid-SMI causing corruption.
- **HF_SMM_MASK transitions paired with vendor-callback** — defense against vendor missing SMM-mode change causing wrong intercept-bitmap.
- **RSM only valid in SMM** — defense against guest invoking RSM outside SMM (would be #UD per Intel SDM).
- **Save-area validated on RSM** — invalid CR0 (PG=1 PE=0), invalid CR4 reserved bits → #GP injected to guest.
- **Per-vCPU SMBASE init at vCPU reset to 0x30000** — defense against carryover SMBASE between guest INIT.
- **KVM_CAP_X86_SMM gating** — defense against SMI-inject in non-SMM-aware VMs.
- **SMI vs NMI priority** strict per spec — defense against priority-inversion delivering NMI before pending SMI.
- **smi_count migration-included** — defense against post-migrate inconsistency causing duplicate-SMI counting.
- **Per-vCPU SMI rate limit** at userspace boundary — defense against SMI flood causing guest livelock.
- **Save-area write atomic from KVM perspective** — defense against partial-save on host-IRQ between save fields.
- **SMM-aware shadow MMU vs TDP MMU** — defense against SPTE-stale referring to non-SMM VA when guest in SMM (vCPU-shadow-mmu rebuild on smm_changed).

