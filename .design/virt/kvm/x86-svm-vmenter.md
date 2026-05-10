# Tier-3: arch/x86/kvm/svm/vmenter.S — SVM vmenter assembly stub (host-state save + guest GPR-load + VMRUN/VMSAVE/VMLOAD + return-handler)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/x86-svm.md
upstream-paths:
  - arch/x86/kvm/svm/vmenter.S
  - arch/x86/kvm/svm/svm.h (regs save area)
  - arch/x86/kvm/svm/svm_ops.h (low-level SVM ops)
-->

## Summary

vmenter.S (SVM) is the per-vCPU-run inner-loop assembly stub for AMD-SVM: saves all host GPRs, loads guest GPRs from `vcpu.regs[]`, executes VMSAVE-host + VMRUN + VMSAVE-guest + VMLOAD-host (per AMD APM sequence), then returns to handle_exit dispatch. AMD SVM differs from Intel VMX: VMSAVE/VMLOAD save/restore extra state (FS/GS/etc. base, TR, LDTR, sysenter/syscall MSRs) automatically per-vCPU around VMRUN. Implements Spectre-v2 retpoline + per-VM IBPB (Indirect-Branch-Predictor Barrier) flush. Slightly simpler than Intel-VMX vmenter (no VMfail-via-CF; AMD SVM uses VMRUN exit-code).

This Tier-3 covers `arch/x86/kvm/svm/vmenter.S` (~404 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `__svm_vcpu_run` | inner asm vmenter | `VcpuSvm::__svm_vcpu_run` |
| `__svm_sev_es_vcpu_run` | SEV-ES variant (encrypted state) | `VcpuSvm::__svm_sev_es_vcpu_run` |
| `svm_vmrun` | per-vCPU VMRUN wrapper | `VcpuSvm::vmrun` |
| `svm_spec_ctrl_restore_host` | per-vmexit IBRS restore | `VcpuSvm::spec_ctrl_restore_host` |
| `regs[VCPU_REGS_RAX]..[VCPU_REGS_R15]` (svm.h) | per-vCPU GPR save offsets | shared |
| `IBPB_FLUSH` (per-vmexit) | Spectre-v2 mitigation | shared |
| `VMSAVE` / `VMLOAD` instructions | per-vCPU extra-state save/restore | UAPI |
| `noinstr` annotations | per-section no-instrumentation | shared |

## Compatibility contract

REQ-1: `__svm_vcpu_run(vcpu)` ABI:
- RDI: `vcpu` pointer.
- Or per-newer: `(vcpu, sd)` where sd is per-CPU svm_cpu_data.

REQ-2: Pre-vmenter sequence:
1. push host GPRs to stack.
2. Load `vcpu.arch.svm.vmcb_pa` for VMRUN argument.
3. Save host extra-state via VMSAVE host_save_area (host TR/FS/GS bases/sysenter MSRs).
4. Load guest CR2 from vcpu.cr2.
5. Load guest GPRs from regs[].
6. Spectre-v2 mitigation: restore guest's IBRS via WRMSR.

REQ-3: VMRUN:
- Single instruction; takes RAX = vmcb_pa.
- CPU loads guest state from vmcb.save area + executes guest until vmexit.
- On vmexit: returns to instruction after VMRUN.
- vmcb.control.exit_code set by CPU.

REQ-4: Post-vmexit sequence:
1. Save guest GPRs to regs[].
2. Save guest CR2 to vcpu.cr2.
3. Restore host extra-state via VMLOAD host_save_area.
4. Restore host IBRS via SPEC_CTRL MSR.
5. IBPB barrier (per-VM; defense against branch-predictor leak across VMs).
6. Pop host GPRs from stack.
7. Return to C-level caller; vmcb.control.exit_code in EAX (or via vmcb pointer).

REQ-5: VMSAVE / VMLOAD instructions:
- VMSAVE addr: saves additional state (FS/GS bases, TR, LDTR, sysenter MSRs, kernel_gs_base) to memory at addr.
- VMLOAD addr: loads from memory at addr.
- Per-spec, these instructions only valid in SVM mode (after STGI).
- Used by KVM to save host state pre-VMRUN + restore post-VMRUN.

REQ-6: SEV-ES variant (`__svm_sev_es_vcpu_run`):
- Guest GPRs not in regs[]; encrypted in VMSA page.
- Pre-vmenter: skip GPR-load (CPU does it from VMSA via SEV-ES decryption).
- Post-vmexit: skip GPR-save.
- Only host-side state managed.

REQ-7: Spectre-v2 retpoline:
- Indirect-jump-via-call+ret pattern.
- Wrapped via __ALTERNATIVE_RETPOLINE macros.

REQ-8: IBPB (Indirect-Branch-Predictor Barrier):
- Issue WRMSR(MSR_IA32_PRED_CMD, IBPB) on vCPU-switch boundaries.
- Defense against guest-trained branch-predictor leaking host secrets.
- Optional based on host CPU + KVM module param.

REQ-9: GIF (Global Interrupt Flag) hw-managed:
- AMD SVM uses GIF to gate IRQ/NMI/exception delivery.
- STGI (Set-GIF) before VMRUN; CLGI (Clear-GIF) post-vmexit (in C-level).
- vmenter.S code runs with CLGI=1 (no IRQs).

REQ-10: noinstr / no-tracing sections:
- vmenter.S code annotated noinstr.
- Critical: instrumentation could leak speculation between vmexit and IBPB.

REQ-11: Per-CPU svm_cpu_data ref:
- Used in newer kernels for per-CPU host_save_pa pass-through.
- Older versions use per-vCPU.

REQ-12: Per-vCPU IBRS state stored in vmcb.control.virt_spec_ctrl:
- AMD-specific; differs from Intel SPEC_CTRL MSR shadowing.

## Acceptance Criteria

- [ ] AC-1: VMRUN first-entry: CPU loads guest from vmcb; guest executes; vmexit observed.
- [ ] AC-2: GPR save/restore: write known GPR pattern to vcpu.regs; VMRUN; guest reads correct values; vmexit; vcpu.regs reflect guest changes.
- [ ] AC-3: CR2 round-trip: guest #PF sets CR2; vmexit; KVM reads vcpu.cr2 = expected.
- [ ] AC-4: Spectre-v2: IBRS state saved guest + restored host across vmexit.
- [ ] AC-5: IBPB: vCPU-switch fires IBPB; counter shows IBPB_COUNT increment.
- [ ] AC-6: VMSAVE/VMLOAD: host TR/FS/GS preserved across VMRUN.
- [ ] AC-7: SEV-ES variant: guest GPRs in encrypted VMSA; KVM doesn't access; vmexit retains encrypted state.
- [ ] AC-8: noinstr verified: ftrace + kcov instrumentation absent in compiled vmenter.S.
- [ ] AC-9: Stress: 1M vmexit/sec; per-cycle overhead < 200 cycles for vmenter+exit.
- [ ] AC-10: kvm-unit-tests `svm_vmexit` test passes.

## Architecture

```asm
; Conceptual SVM vmenter.S structure (in real Linux, GAS syntax):

__svm_vcpu_run:
    ; ABI: RDI=vcpu
    push rbp
    mov  rbp, rdi             ; preserve vcpu in rbp
    push rbx
    push r12
    push r13
    push r14
    push r15
    
    ; Load vmcb_pa
    mov  rax, [rbp + VCPU_ARCH_SVM_VMCB_PA]
    
    ; VMSAVE host extra-state to per-CPU host_save area
    mov  rcx, [host_save_pa]
    vmsave rcx
    
    ; Restore guest IBRS if needed
    test ..., VMX_RUN_SAVE_SPEC_CTRL_FLAG
    jz   .skip_ibrs_restore_guest
    mov  ecx, MSR_IA32_SPEC_CTRL
    mov  rax, [rbp + VCPU_ARCH_SPEC_CTRL]
    xor  rdx, rdx
    wrmsr
.skip_ibrs_restore_guest:
    
    ; Load guest CR2
    mov  rax, [rbp + VCPU_ARCH_CR2]
    mov  cr2, rax
    
    ; Load guest GPRs
    mov  rsi, [rbp + VCPU_ARCH_REGS]
    mov  rax, [rsi + VCPU_REGS_RAX*8]
    mov  rbx, [rsi + VCPU_REGS_RBX*8]
    ; ...etc rcx-r15...
    
    ; VMRUN with vmcb_pa in RAX
    mov  rax, vmcb_pa
    vmrun rax
    
    ; vmexit returns here:
    
    ; Save guest GPRs to regs[]
    mov  [rsi + VCPU_REGS_RAX*8], rax
    mov  [rsi + VCPU_REGS_RBX*8], rbx
    ; ...etc...
    
    ; Save guest CR2
    mov  rax, cr2
    mov  [rbp + VCPU_ARCH_CR2], rax
    
    ; VMLOAD host state from per-CPU host_save area
    mov  rcx, [host_save_pa]
    vmload rcx
    
    ; Restore host IBRS
    test ..., VMX_RUN_SAVE_SPEC_CTRL_FLAG
    jz   .skip_ibrs_restore_host
    rdmsr ; SPEC_CTRL → save guest IBRS
    mov  [rbp + VCPU_ARCH_SPEC_CTRL], eax
    mov  ecx, MSR_IA32_SPEC_CTRL
    mov  rax, host_ibrs_default
    xor  rdx, rdx
    wrmsr
.skip_ibrs_restore_host:
    
    ; IBPB if needed (per-VM-switch)
    cmp  byte [vmx_l1d_flush_always], 0  ; or IBPB analog
    jz   .skip_ibpb
    mov  ecx, MSR_IA32_PRED_CMD
    mov  rax, IBPB_VAL
    xor  rdx, rdx
    wrmsr
.skip_ibpb:
    
    ; Pop host GPRs
    pop  r15
    pop  r14
    pop  r13
    pop  r12
    pop  rbx
    pop  rbp
    
    ret
```

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `host_gpr_save_restore_balanced` | INVARIANT | per-VMRUN all pushed host GPRs popped on vmexit. |
| `cr2_round_trip` | INVARIANT | guest CR2 saved on vmexit; host CR2 not corrupted. |
| `vmcb_pa_aligned` | INVARIANT | per-VMRUN vmcb_pa aligned to 4KiB; defense against #UD on misaligned. |
| `ibrs_save_restore_paired` | INVARIANT | per-IBRS-save flag: guest IBRS saved + host IBRS restored. |
| `vmsave_vmload_paired` | INVARIANT | every VMSAVE paired with VMLOAD (host-side). |
| `regs_array_within_bounds` | INVARIANT | per-VCPU_REGS_* offset < 16 * 8 bytes. |

### Layer 2: TLA+

`virt/kvm/svm_vmenter_lifecycle.tla`:
- Per-vCPU vmenter state ∈ {PreVmenter, GuestRunning, PostVmexit}.
- Properties:
  - `safety_state_save_complete` — GuestRunning implies all guest state set in vmcb.
  - `safety_state_restore_complete` — PostVmexit implies all guest GPRs in regs[].
  - `liveness_guest_running_eventually_exits` — every GuestRunning eventually PostVmexit (assuming no vCPU hang).

`virt/kvm/svm_spectre_isolation.tla`:
- Per-VMRUN IBRS + IBPB state save/restore boundary.
- Properties:
  - `safety_no_speculation_leak` — host IBRS restored before any host indirect-branch.
  - `safety_ibpb_at_vm_switch` — IBPB issued at vCPU-switch if config requires.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `__svm_vcpu_run` post: VMRUN executed; guest GPRs loaded into HW context | `__svm_vcpu_run` |
| Post-vmexit: regs[] populated with guest GPRs; vcpu.cr2 saved | `__svm_vcpu_run` |
| VMSAVE host_save before VMRUN; VMLOAD host_save after | `__svm_vcpu_run` |
| Per-vmenter SPEC_CTRL transitions: guest-IBRS in vmenter; host-IBRS post-vmexit | `svm_spec_ctrl_restore_host` |

### Layer 4: Verus/Creusot functional

`Per-vmexit cycle: guest GPR state in regs[] matches what guest wrote pre-vmexit; host GPR + host extra-state restored to pre-vmenter values` semantic equivalence: per-cycle full state round-trip preserved.

## Hardening

(Inherits row-1 features from `virt/kvm/x86-svm.md` § Hardening.)

SVM-vmenter-specific reinforcement:

- **VMSAVE/VMLOAD discipline** — defense against host extra-state corruption across VMRUN.
- **Spectre-v2 retpoline** — defense against indirect-branch speculation leak.
- **IBRS save/restore boundary** — defense against guest-trained branch-predictor leak.
- **IBPB at vCPU-switch** — defense against per-VM branch-predictor reuse.
- **noinstr annotation** — defense against ftrace/kcov instrumentation introducing speculation gadgets.
- **vmcb_pa aligned** — defense against VMRUN #UD.
- **CR2 explicit save/restore** — defense against CR2-leak across vmexit.
- **SEV-ES skip GPR-access** — defense against KVM accessing encrypted guest state.
- **Per-vmenter VMSAVE host save area** — defense against per-CPU stale host-state on resume.
- **GIF cleared on vmexit** — defense against IRQ delivery during vmenter cleanup.
- **Stack-protector active** — defense against vmenter stack overrun.
- **Per-vCPU vmcb_pa validated** — defense against attacker-controlled vmcb pointer.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- SVM core (covered in `x86-svm.md` Tier-3)
- KVM core (covered in `kvm-core.md` Tier-3)
- VMX vmenter (covered in `x86-vmx-vmenter.md` Tier-3; analog)
- SEV core + SEV-ES VMSA (covered in `x86-sev.md` Tier-3)
- IBPB / SPEC_CTRL host policy (covered in `arch/x86/cpu/bugs.md` future Tier-3)
- Implementation code
