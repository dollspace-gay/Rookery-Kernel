---
title: "Tier-3: arch/x86/kvm/vmx/vmenter.S — VMX vmenter assembly stub (host-state save + guest GPR-load + VMRESUME/VMLAUNCH + return-handler)"
tags: ["tier-3", "virt-kvm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

vmenter.S is the per-vCPU-run inner-loop assembly stub: it saves all host GPRs to the per-CPU vmcs_host_state save area, loads guest GPRs from `vcpu.regs[]`, then executes VMRESUME (or VMLAUNCH on first-entry) to enter guest. On vmexit, the matching code path saves guest GPRs back to `vcpu.regs[]`, restores host GPRs, and returns to the C-level handle_exit dispatch. Implements Spectre-v2 retpoline + L1TF L1-data-cache flush mitigations. Tiny but absolutely critical (every vmexit goes through it).

This Tier-3 covers `arch/x86/kvm/vmx/vmenter.S` (~387 lines).

### Acceptance Criteria

- [ ] AC-1: VMRESUME first-entry: CPU loads guest state; guest executes; vmexit observed.
- [ ] AC-2: GPR save/restore: write known GPR pattern to vcpu.regs; vmenter; guest reads correct values; vmexit; vcpu.regs reflect guest changes.
- [ ] AC-3: CR2 round-trip: guest #PF sets CR2; vmexit; KVM reads vcpu.cr2 = expected.
- [ ] AC-4: Spectre-v2: IBRS state saved guest + restored host across vmexit.
- [ ] AC-5: L1TF: guest-untrusted scenario flushes L1D on vmexit; trace shows L1D_FLUSH counter increment.
- [ ] AC-6: noinstr verified: ftrace + kcov instrumentation does not appear in compiled vmenter.S.
- [ ] AC-7: VMfail detection: corrupted VMCS triggers vmenter.S CF-set; KVM logs failure.
- [ ] AC-8: Stress: 1M vmexit/sec; per-cycle overhead < 200 cycles for vmenter+exit minimal path.
- [ ] AC-9: kvm-unit-tests `vmexit` test passes.
- [ ] AC-10: Cross-CPU vCPU migrate: VMCS cache cleared on prior CPU; new CPU vmenter succeeds.

### Architecture

```asm
; Conceptual vmenter.S structure (in real Linux, GAS syntax):

__vmx_vcpu_run:
    ; ABI: RDI=vcpu, RSI=regs, RDX=run_flags
    push rbp
    mov  rbp, rdi             ; preserve vcpu in rbp
    
    ; Save host GPRs to stack
    push rbx
    push r12
    push r13
    push r14
    push r15
    
    ; Update HOST_RSP in VMCS
    mov  rcx, HOST_RSP_FIELD_ENC
    vmwrite rcx, rsp
    
    ; Restore guest IBRS if needed
    mov  rax, [rbp + VCPU_ARCH_SPEC_CTRL]
    test rdx, VMX_RUN_SAVE_SPEC_CTRL_FLAG
    jz   .skip_ibrs_restore_guest
    wrmsr ; SPEC_CTRL = guest's IBRS
.skip_ibrs_restore_guest:
    
    ; Load guest CR2
    mov  rax, [rbp + VCPU_ARCH_CR2]
    mov  cr2, rax
    
    ; Load guest GPRs from regs[]
    mov  rax, [rsi + VCPU_REGS_RAX*8]
    mov  rbx, [rsi + VCPU_REGS_RBX*8]
    mov  rcx, [rsi + VCPU_REGS_RCX*8]
    ; ...rdx, rsi, rdi, rbp, r8-r15...
    
    ; Choose VMRESUME or VMLAUNCH
    test rdx, VMX_RUN_VMRESUME_FLAG
    jz   .vmlaunch
    vmresume
    jmp  .vmfail
.vmlaunch:
    vmlaunch
.vmfail:
    ; CF set means VMfail; jump to C handler
    pop  r15
    pop  r14
    ; ...
    pop  rbx
    pop  rbp
    mov  eax, -EFAULT
    ret

; vmexit return target:
__vmx_vmexit:
    ; CPU loaded host RSP+RIP; we resume here
    ; Save guest GPRs to regs[]
    mov  [rsi + VCPU_REGS_RAX*8], rax
    mov  [rsi + VCPU_REGS_RBX*8], rbx
    ; ...rcx-r15...
    
    ; Save guest CR2
    mov  rax, cr2
    mov  [rbp + VCPU_ARCH_CR2], rax
    
    ; Restore host IBRS
    test rdx, VMX_RUN_SAVE_SPEC_CTRL_FLAG
    jz   .skip_ibrs_restore_host
    rdmsr ; SPEC_CTRL → save guest's IBRS
    mov  [rbp + VCPU_ARCH_SPEC_CTRL], eax
    ; Restore host
    mov  ecx, MSR_IA32_SPEC_CTRL
    mov  rax, host_ibrs_default
    wrmsr
.skip_ibrs_restore_host:
    
    ; L1TF flush if needed
    cmp  byte [vmx_l1d_flush_always], 0
    jz   .skip_l1d_flush
    call vmx_l1d_flush
.skip_l1d_flush:
    
    ; Pop host GPRs
    pop  r15
    pop  r14
    pop  r13
    pop  r12
    pop  rbx
    pop  rbp
    
    xor  eax, eax  ; success
    ret
```

### Out of Scope

- VMX core (covered in `x86-vmx.md` Tier-3)
- KVM core (covered in `kvm-core.md` Tier-3)
- Spec-ctrl userspace policy (covered separately)
- L1TF mitigation policy (covered in `arch/x86/cpu/bugs.md` future Tier-3)
- AMD SVM vmenter (covered in `x86-svm-vmenter.md` future Tier-3; analog)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `__vmx_vcpu_run` | inner asm vmenter | `VcpuVmx::__vmx_vcpu_run` |
| `vmx_vmenter` | VMRESUME/VMLAUNCH wrapper | `VcpuVmx::vmx_vmenter` |
| `vmx_vmexit` | post-vmexit return | `VcpuVmx::vmx_vmexit` |
| `vmx_spec_ctrl_restore_host` | per-vmexit IBRS restore | `VcpuVmx::spec_ctrl_restore_host` |
| `vmx_update_host_rsp` | per-vmenter RSP refresh | `VcpuVmx::update_host_rsp` |
| `noinstr` annotations | per-section no-instrumentation | shared |
| `regs[VCPU_REGS_RAX]..[VCPU_REGS_R15]` (vmx.h) | per-vCPU GPR save offsets | shared |
| `IBRS_LOCK` / `_UNLOCK` macros (per-vCPU IBRS) | Spectre-v2 mitigation | shared |
| `VMX_RUN_VMRESUME_FLAG` | flag for VMRESUME-vs-VMLAUNCH choice | `RunFlags::VMRESUME` |
| `VMX_RUN_SAVE_SPEC_CTRL_FLAG` | flag for IBRS save | `RunFlags::SAVE_SPEC_CTRL` |

### compatibility contract

REQ-1: `__vmx_vcpu_run(vcpu, regs, run_flags)` ABI:
- RDI: `vcpu` pointer (per-vCPU struct kvm_vcpu).
- RSI: `regs[]` pointer (16-entry GPR save array).
- RDX: `run_flags` (VMX_RUN_*_FLAG bits).

REQ-2: Pre-vmenter sequence:
1. push host GPRs to stack.
2. Load `vcpu` into RBP for backref during vmexit return.
3. Test VMX_RUN_VMRESUME_FLAG → choose VMRESUME or VMLAUNCH.
4. Load guest CR2 from vcpu.cr2 (CR2 not auto-saved/restored by VMX).
5. Load guest GPRs from regs[]:
   - RAX = regs[VCPU_REGS_RAX].
   - RBX = regs[VCPU_REGS_RBX].
   - RCX = regs[VCPU_REGS_RCX].
   - ...etc for all 16 GPRs except RSP/RBP (special handling).
6. Update vmcs.HOST_RSP = current RSP via vmwrite.
7. Spectre-v2 mitigation: if SPEC_CTRL save needed, restore guest's IBRS bit.

REQ-3: VMRESUME / VMLAUNCH:
- VMLAUNCH on first-entry to this VMCS.
- VMRESUME on subsequent entries.
- CPU loads guest state from vmcs.guest.* fields + executes guest until vmexit.

REQ-4: Post-vmexit sequence:
1. Save guest GPRs to regs[]:
   - regs[VCPU_REGS_RAX] = RAX (etc.)
2. Save guest CR2 to vcpu.cr2.
3. Restore host IBRS via SPEC_CTRL MSR.
4. L1TF mitigation: flush L1D cache if guest is untrusted.
5. Pop host GPRs from stack.
6. Return to C-level caller with vmexit-reason in EAX.

REQ-5: Spectre-v2 retpoline:
- Indirect-jump-via-call+ret pattern to avoid Spectre-v2 BTI.
- Wrapped via __ALTERNATIVE_RETPOLINE macros.

REQ-6: L1TF mitigation:
- Per-vmexit conditionally flush L1 data cache.
- KVM_CAP_L1TF_FLUSH_L1D advertised based on host CPU + module param.
- Strategy: always-flush, conditional-flush (only for non-trusted guests), or never-flush.

REQ-7: noinstr / no-tracing sections:
- vmenter.S code annotated noinstr to prevent ftrace/kcov from inserting instrumentation.
- Critical: instrumentation between vmexit and IBRS restore could leak speculation.

REQ-8: Per-vmenter VMCS setup (already done in C):
- vmcs.guest.* fields populated.
- vmcs.HOST_RSP / .HOST_RIP set.
- vmcs.HOST_CR0 / _CR3 / _CR4 set.
- vmcs.HOST_SEL_* segment selectors.
- vmcs.HOST_FS_BASE / _GS_BASE.

REQ-9: Per-vmexit consistency-check:
- VMEXIT-failure (VMfail) detected via JMP-on-CF-set after VMRESUME.
- Causes: VMCS not loaded, invalid VMCS, stale VMCS-cache.
- KVM C-level handler debug-print + fail.

REQ-10: Per-vmenter pf_intercepted state:
- Some vmexits (e.g., #PF on host page-table-walk) need immediate pre-IBRS-restore handling.
- vmenter.S defers to C via VMX_HANDLE_EXIT_IRQOFF if needed.

REQ-11: Frame-pointer + stack-protector:
- vmenter.S uses minimal stack to reduce code-gen complexity.
- Host return-address protected by call/ret pairing.

REQ-12: VMX-on / VMX-off boundaries:
- vmenter.S assumes VMX enabled on this CPU.
- VMX-disable elsewhere (KVM module unload).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `host_gpr_save_restore_balanced` | INVARIANT | per-vmenter all pushed host GPRs popped on vmexit return. |
| `cr2_round_trip` | INVARIANT | guest CR2 saved on vmexit; host CR2 not corrupted. |
| `ibrs_save_restore_paired` | INVARIANT | per-IBRS-save flag: guest IBRS saved + host IBRS restored on vmexit. |
| `regs_array_within_bounds` | INVARIANT | per-VCPU_REGS_* offset < 16 * 8 bytes; defense against regs[] OOB. |
| `vmcs_loaded_pre_vmenter` | INVARIANT | vmcs vmptrld'ed on this CPU before VMRESUME; defense against VMfail. |

### Layer 2: TLA+

`virt/kvm/vmenter_lifecycle.tla`:
- Per-vCPU vmenter state ∈ {PreVmenter, GuestRunning, PostVmexit}.
- Properties:
  - `safety_state_save_complete` — GuestRunning implies all guest state set in vmcs.
  - `safety_state_restore_complete` — PostVmexit implies all guest GPRs in regs[].
  - `liveness_guest_running_eventually_exits` — every GuestRunning eventually PostVmexit.

`virt/kvm/spectre_v2_isolation.tla`:
- Per-vmenter IBRS state save/restore boundary.
- Properties:
  - `safety_no_speculation_leak` — host IBRS restored before any host indirect-branch.
  - `safety_l1d_flushed_before_host_resume` — L1TF mitigation enforced if needed.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `__vmx_vcpu_run` post: VMRESUME executed (or VMfail returned); guest GPRs loaded | `__vmx_vcpu_run` |
| `__vmx_vmexit` post: regs[] populated with guest GPRs; vcpu.cr2 saved | `__vmx_vmexit` |
| Per-vmenter SPEC_CTRL transitions: guest-IBRS in vmenter; host-IBRS post-vmexit | `vmx_spec_ctrl_restore_host` |
| Per-vmexit L1D-flush conditional on host config | invariants on flush logic |

### Layer 4: Verus/Creusot functional

`Per-vmexit cycle: guest GPR state visible to KVM matches what guest wrote pre-vmexit; host GPR state restored to pre-vmenter values` semantic equivalence: per-cycle GPR round-trip preserved exactly except for guest-side modifications.

### hardening

(Inherits row-1 features from `virt/kvm/x86-vmx.md` § Hardening.)

vmenter-specific reinforcement:

- **Spectre-v2 retpoline** — defense against indirect-branch speculation leaking.
- **IBRS save/restore boundary** — defense against guest-trained branch-predictor leaking host secrets.
- **L1TF L1D flush before host resume** — defense against L1-data-cache leak across guest/host boundary.
- **noinstr annotation** — defense against ftrace/kcov instrumentation introducing speculation gadgets.
- **Per-vmenter HOST_RSP refresh** — defense against stale RSP after host-stack-relocation.
- **VMfail detection via CF** — defense against silent failure progressing.
- **CR2 explicit save/restore** — defense against CR2-leak (not auto-handled by VMX).
- **Frame-pointer minimal usage** — defense against complex stack-frame interaction with VMX.
- **VM-entry GPR-load order** — defense against stale GPR (e.g., RAX used for syscall enter).
- **Per-CPU vmcs_host_state** — defense against cross-CPU state confusion.
- **Per-vmenter VMRESUME-vs-VMLAUNCH flag** — defense against incorrect instruction selection.
- **Stack-protector active** — defense against vmenter stack overrun.

