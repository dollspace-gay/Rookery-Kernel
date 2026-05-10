---
title: "Tier-3: arch/x86/kvm/vmx/vmx.c (vmexit-handler subset) — VMX exit-reason dispatch table + per-reason handler categories"
tags: ["tier-3", "virt-kvm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Per VMX vmexit, CPU writes vmcs.EXIT_REASON (16-bit basic + 16-bit qualifier flags); KVM's `vmx_handle_exit` reads this + dispatches via `kvm_vmx_exit_handlers[]` table to per-reason handler. Per-handler category: trivial (e.g., HLT, PAUSE), instruction-emulation (CR-access, MSR_R/W, IO_INSTRUCTION, CPUID), TDP/shadow page-fault (EPT_VIOLATION, EPT_MISCONFIG, EXCEPTION_NMI), interrupt-window (NMI_WINDOW, INTERRUPT_WINDOW), nested-VMX (VMRESUME, VMLAUNCH, INVEPT, INVVPID), virt-interrupt (APIC_ACCESS, APIC_WRITE, EOI_INDUCED), userspace-exit (HLT-userspace, IO-to-qemu). ~60 distinct EXIT_REASON_* values.

This Tier-3 covers vmexit-handler dispatch in `vmx.c` (~600-800 lines).

### Acceptance Criteria

- [ ] AC-1: Boot Linux guest: per-vmexit dispatched to correct handler; counter visible via debugfs.
- [ ] AC-2: CPUID emulation: guest CPUID → handle_cpuid → KVM responds with per-vCPU CPUID-table values.
- [ ] AC-3: HLT in-guest: KVM_CAP_X86_DISABLE_EXITS HLT → VM doesn't vmexit on HLT.
- [ ] AC-4: EPT_VIOLATION: guest #PF on un-mapped GPA → vmexit → TDP MMU installs SPTE → resume.
- [ ] AC-5: APIC_ACCESS: xAPIC MMIO write → vmexit → KVM emulates → resume.
- [ ] AC-6: VMCALL hypercall: KVM_HC_KICK_CPU works; vmexit dispatched + handled.
- [ ] AC-7: Per-reason stat: /sys/kernel/debug/kvm/<vm>/vcpu0/<reason>_count visible.
- [ ] AC-8: Nested-VMX VMRESUME: L1 issues VMRESUME → L0 dispatches → L2 starts.
- [ ] AC-9: VM-entry failure: malformed VMCS triggers VM_ENTRY_FAILURE; KVM logs + returns error.
- [ ] AC-10: Performance: per-EXIT_REASON dispatch < 50 cycles; hot-path fastpath < 200 cycles.

### Architecture

`VcpuVmx::EXIT_HANDLERS` (const dispatch table):

```
const EXIT_HANDLERS: [VmxExitHandler; NR_VMX_EXIT_REASONS] = [
  // EXIT_REASON_EXCEPTION_NMI = 0
  VcpuVmx::handle_exception_nmi,
  // EXIT_REASON_EXTERNAL_INTERRUPT = 1
  VcpuVmx::handle_external_interrupt,
  // EXIT_REASON_TRIPLE_FAULT = 2
  VcpuVmx::handle_triple_fault,
  // ... 50+ entries ...
  // EXIT_REASON_EPT_VIOLATION = 48
  VcpuVmx::handle_ept_violation,
  // ...
];
```

`VcpuVmx::handle_exit(vcpu, fastpath)`:
1. exit_reason := vmcs_read32(VM_EXIT_REASON).
2. exit_qualification := vmcs_readl(EXIT_QUALIFICATION).
3. Update per-vCPU `exit_reason` + `exit_qualification` shadows.
4. If fastpath != FASTPATH_NONE: handle inline; return.
5. If exit_reason.failed_vmentry: trace + log; return -EINVAL.
6. handler := kvm_vmx_exit_handlers[exit_reason.basic].
7. If !handler: handle_unexpected_vmexit; return -EOPNOTSUPP.
8. ret := handler(vcpu).
9. Per-stat: kvm_stat_vmexit_inc(vcpu, exit_reason.basic).
10. Return ret.

`VcpuVmx::handle_io(vcpu)`:
1. Read EXIT_QUALIFICATION: port + size + direction + string.
2. If string-IO: emulator dispatch.
3. Else:
   - vcpu.arch.pio.port = port.
   - vcpu.arch.pio.size = size.
   - vcpu.arch.pio.direction = dir.
   - Try in-kernel device (LAPIC, ioapic, pic, pit, virtio-mmio).
   - If handled: skip_emulated_instruction; return 1.
   - Else: vcpu.run_state.exit_reason = KVM_EXIT_IO; return 0 (userspace).

`VcpuVmx::handle_ept_violation(vcpu)`:
1. exit_qual := vmcs_readl(EXIT_QUALIFICATION).
2. error_code := compute from exit_qual (read/write/exec, present).
3. gpa := vmcs_read64(GUEST_PHYSICAL_ADDRESS).
4. ret := kvm_mmu_page_fault(vcpu, gpa, error_code, NULL, 0).
5. Return ret.

`VcpuVmx::handle_cpuid(vcpu)`:
1. eax := vcpu.regs[VCPU_REGS_RAX].
2. ecx := vcpu.regs[VCPU_REGS_RCX].
3. kvm_emulate_cpuid(vcpu): look up per-vCPU CPUID table; populate eax/ebx/ecx/edx.
4. skip_emulated_instruction.
5. Return 1.

### Out of Scope

- VMX core (covered in `x86-vmx.md` Tier-3)
- Per-handler details (CR / MSR / IO / etc. — partially covered; future per-handler Tier-3s)
- Nested-VMX (covered in `x86-nested-vmx.md` Tier-3)
- TDP MMU (covered in `x86-mmu-tdp.md` Tier-3)
- LAPIC (covered in `x86-lapic.md` Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `kvm_vmx_exit_handlers[]` | per-reason handler table | `VcpuVmx::EXIT_HANDLERS` |
| `vmx_handle_exit(vcpu, fastpath)` | per-vmexit dispatcher | `VcpuVmx::handle_exit` |
| `vmx_handle_exit_irqoff(vcpu)` | per-vmexit immediate-handler | `VcpuVmx::handle_exit_irqoff` |
| `handle_exception_nmi(vcpu)` (EXCEPTION_NMI) | per-exception | `VcpuVmx::handle_exception_nmi` |
| `handle_external_interrupt(vcpu)` | per-EXT_INTR | `VcpuVmx::handle_external_interrupt` |
| `handle_triple_fault(vcpu)` | per-triple-fault | `VcpuVmx::handle_triple_fault` |
| `handle_nmi_window(vcpu)` | per-NMI-window | `VcpuVmx::handle_nmi_window` |
| `handle_interrupt_window(vcpu)` | per-IRQ-window | `VcpuVmx::handle_interrupt_window` |
| `handle_io(vcpu)` | per-IO instruction | `VcpuVmx::handle_io` |
| `handle_cr(vcpu)` | per-CR access | `VcpuVmx::handle_cr` |
| `handle_dr(vcpu)` | per-DR access | `VcpuVmx::handle_dr` |
| `handle_cpuid(vcpu)` | per-CPUID | `VcpuVmx::handle_cpuid` |
| `handle_invlpg(vcpu)` | per-INVLPG | `VcpuVmx::handle_invlpg` |
| `handle_invlpgb(vcpu)` (newer) | per-INVLPGB | `VcpuVmx::handle_invlpgb` |
| `handle_rdmsr(vcpu)` / `handle_wrmsr(vcpu)` | per-RDMSR/WRMSR | `VcpuVmx::handle_rdmsr` / `_wrmsr` |
| `handle_ept_violation(vcpu)` | per-EPT-fault | `VcpuVmx::handle_ept_violation` |
| `handle_ept_misconfig(vcpu)` | per-EPT-misconfig (MMIO trap) | `VcpuVmx::handle_ept_misconfig` |
| `handle_apic_access(vcpu)` | per-APIC-access (xAPIC) | `VcpuVmx::handle_apic_access` |
| `handle_apic_write(vcpu)` | per-APIC-write (APICv) | `VcpuVmx::handle_apic_write` |
| `handle_eoi_induced(vcpu)` | per-EOI-induced | `VcpuVmx::handle_eoi_induced` |
| `handle_xsaves(vcpu)` / `handle_xrstors(vcpu)` | per-XSAVES/XRSTORS | `VcpuVmx::handle_xsaves` / `_xrstors` |
| `handle_invpcid(vcpu)` | per-INVPCID | `VcpuVmx::handle_invpcid` |
| `handle_invept(vcpu)` (nested) | per-nested INVEPT | `VcpuVmx::handle_invept` |
| `handle_invvpid(vcpu)` (nested) | per-nested INVVPID | `VcpuVmx::handle_invvpid` |
| `handle_vmcall(vcpu)` | per-VMCALL (guest hypercall) | `VcpuVmx::handle_vmcall` |
| `handle_vmclear/_vmlaunch/_vmptrld/_vmptrst/_vmread/_vmresume/_vmwrite/_vmoff/_vmon` (nested) | per-nested-instr | covered in nested.c |
| `handle_xsetbv(vcpu)` | per-XSETBV | `Vcpu::handle_xsetbv` |
| `handle_ldt(vcpu)` / `handle_gdtr_idtr(vcpu)` | per-descriptor-table | `VcpuVmx::handle_*` |

### compatibility contract

REQ-1: vmcs.EXIT_REASON encoding (32-bit):
- Bits[15:0]: basic reason number.
- Bits[27:16]: reserved.
- Bit[28]: enclave-mode (SGX).
- Bit[29]: pending-MTF VM-exit.
- Bit[30]: VM-entry-failure (post-vmenter inconsistency-check fail).
- Bit[31]: VM-exit-from-root (rare).

REQ-2: Basic exit-reason values (numeric examples):
- 0 EXCEPTION_NMI.
- 1 EXTERNAL_INTERRUPT.
- 2 TRIPLE_FAULT.
- 7 INTERRUPT_WINDOW.
- 8 NMI_WINDOW.
- 10 CPUID.
- 12 HLT.
- 13 INVD.
- 14 INVLPG.
- 15 RDPMC.
- 16 RDTSC.
- 18 VMCALL.
- 19-26 VMCLEAR / VMLAUNCH / VMPTRLD / VMPTRST / VMREAD / VMRESUME / VMWRITE / VMOFF / VMON (nested).
- 28 CR_ACCESS.
- 29 DR_ACCESS.
- 30 IO_INSTRUCTION.
- 31 RDMSR.
- 32 WRMSR.
- 36 MWAIT.
- 37 MONITOR_TRAP_FLAG.
- 39 MONITOR.
- 40 PAUSE.
- 41 MACHINE_CHECK.
- 43 TPR_BELOW_THRESHOLD.
- 44 APIC_ACCESS.
- 45 EOI_INDUCED.
- 48 EPT_VIOLATION.
- 49 EPT_MISCONFIG.
- 50 INVEPT (nested).
- 51 RDTSCP.
- 53 INVVPID (nested).
- 54 WBINVD.
- 55 XSETBV.
- 56 APIC_WRITE.
- 58 INVPCID.
- 59 VMFUNC.
- 60 ENCLS (SGX).
- 64 TPAUSE.

REQ-3: vmx_handle_exit flow:
1. Read vmcs.EXIT_REASON.
2. Read auxiliary fields: EXIT_QUALIFICATION, EXIT_INTR_INFO, IDT_VECTORING_INFO, GUEST_LINEAR_ADDR.
3. Determine fastpath:
   - EXIT_REASON_MSR_WRITE + matching IPI/TSC-deadline: fastpath emulate inline.
   - Else: fall-through to handler.
4. handler := kvm_vmx_exit_handlers[exit_reason.basic].
5. ret := handler(vcpu).
6. Switch ret:
   - >= 0: continue VM execution OR userspace return.
   - < 0: error to userspace.

REQ-4: Per-handler categories:

**Category A: Hardware-exit + emulate (CR / DR / IO / MSR / CPUID):**
- Per-handler decodes instruction state from VMCS fields.
- Calls into KVM emulator subsystem.
- Updates RIP via skip_emulated_instruction.

**Category B: TDP/shadow page-fault:**
- EPT_VIOLATION: pass GPA to TDP MMU; install SPTE; resume.
- EPT_MISCONFIG: typically MMIO-SPTE; emulate via in-kernel device.
- Both may return KVM_EXIT_MMIO to userspace.

**Category C: IRQ/NMI window:**
- INTERRUPT_WINDOW: vCPU is now ready to receive IRQ; KVM injects pending IRQ.
- NMI_WINDOW: similar for NMI.

**Category D: Nested-VMX instructions:**
- VMRESUME / VMLAUNCH: nested VMRESUME → enter L2 mode.
- VMREAD / VMWRITE: per-vmcs12 access.
- VMCALL / VMOFF / etc.: nested-VMX-specific.

**Category E: Virtual-interrupt:**
- APIC_ACCESS: xAPIC MMIO access trap.
- APIC_WRITE: APICv write-trap.
- EOI_INDUCED: per-vector EOI trap.

**Category F: Userspace-return:**
- HLT (KVM_HLT_IN_GUEST option).
- IO_INSTRUCTION (port-IO not handled by in-kernel devices).
- KVM_EXIT_HYPERCALL (specific KVM_HC codes).

REQ-5: skip_emulated_instruction:
- Reads vmcs.VM_EXIT_INSTRUCTION_LEN.
- Advances vmcs.GUEST_RIP by that length.
- Used after software-emulation of intercepted instruction.

REQ-6: VM-entry-failure (bit 30):
- vmcs basic reason indicates which inconsistency caused vmenter to fail.
- KVM logs + returns error.

REQ-7: SGX enclave-exit (bit 28):
- VMexit happened from SGX enclave.
- KVM handles via SGX-specific path (covered in x86-vmx-sgx.md Tier-3).

REQ-8: vmexit fastpath optimization:
- Some EXIT_REASONs handled in vmx_handle_exit_irqoff before re-enabling IRQ.
- Reduces vmexit-to-vmenter latency for hot-path events.

REQ-9: Per-handler return values:
- Positive: continue (no userspace return needed).
- Zero: continue (success).
- Negative: error path.
- Special: 0 with vcpu.run_state.exit_reason set → userspace return.

REQ-10: Per-handler stat counters:
- Per-EXIT_REASON KVM stat (visible via /sys/kernel/debug/kvm).
- Used for performance analysis.

REQ-11: Live migration:
- Pending-vmexit state migrated via KVM_GET_VCPU_EVENTS + KVM_SET_VCPU_EVENTS.

REQ-12: Nested-vmexit cascade:
- When L2 vmexits, L0 may decide to reflect to L1 (nested_vmx_check_intercept).
- Per-reason check determines reflect-to-L1 vs handle-in-L0.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `exit_reason_idx_bounded` | OOB | exit_reason.basic < NR_VMX_EXIT_REASONS; defense against table OOB. |
| `handler_pointer_valid` | INVARIANT | per-table entry non-NULL or handle_unexpected_vmexit fallback. |
| `skip_emulated_instruction_no_partial` | INVARIANT | RIP advance atomic with VM_EXIT_INSTRUCTION_LEN read. |
| `nested_intercept_check_consistent` | INVARIANT | nested-reflect decision based on vmcs12 fields. |
| `fastpath_only_for_safe_reasons` | INVARIANT | fastpath limited to MSR_WRITE + IPI/TSC-deadline. |

### Layer 2: TLA+

`virt/kvm/vmx_exit_dispatch.tla`:
- Per-vmexit state ∈ {Pending, Dispatching, Returning, UserspaceExit}.
- Properties:
  - `safety_per_reason_handler` — every exit_reason dispatched to correct handler.
  - `safety_skip_only_after_emulate` — skip_emulated_instruction only after successful emulation.
  - `liveness_pending_eventually_handled` — every Pending eventually Returning or UserspaceExit.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `VcpuVmx::handle_exit` post: per-handler called; stat incremented; ret propagated | `VcpuVmx::handle_exit` |
| `VcpuVmx::handle_io` post: bio dispatched in-kernel OR userspace exit set | `VcpuVmx::handle_io` |
| `VcpuVmx::handle_ept_violation` post: SPTE installed OR userspace return | `VcpuVmx::handle_ept_violation` |
| `VcpuVmx::handle_cpuid` post: per-vCPU regs populated; RIP advanced | `VcpuVmx::handle_cpuid` |

### Layer 4: Verus/Creusot functional

`Per-vmexit: handler invoked + appropriate state changes applied → guest resumes (or userspace exit)` semantic equivalence: per-vmexit the resulting effect matches Intel-SDM-described semantics.

### hardening

(Inherits row-1 features from `virt/kvm/x86-vmx.md` § Hardening.)

vmexit-handler-specific reinforcement:

- **EXIT_HANDLERS table size matches NR_VMX_EXIT_REASONS** — defense against OOB on unknown reason.
- **handle_unexpected_vmexit fallback** — defense against missing-handler crash.
- **VM-entry-failure differentiated** — defense against silent-failure path.
- **Per-fastpath whitelist** — defense against incorrect fastpath emulation for state-changing reasons.
- **skip_emulated_instruction sourced from vmcs** — defense against incorrect RIP advance.
- **Per-handler RFLAGS coordination** — defense against post-emulation RF/TF bits inconsistent.
- **Per-handler MSR/CR access validation** — defense against emulation accepting invalid state.
- **Nested intercept-check before reflect** — defense against L0-handler sequencing wrong vs L1.
- **Per-stat counter rate-limit** — defense against stat-overflow from pathological vmexit pattern.
- **Per-CPU vmexit-handler state isolated** — defense against cross-CPU-vCPU state leak.

