---
title: "Tier-3: arch/x86/kvm/vmx/vmx.c (event injection subset) — VMCS event injection (VM_ENTRY_INTR_INFO_FIELD + IDT-vectoring + double-fault detection)"
tags: ["tier-3", "virt-kvm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

KVM's event-injection mechanism translates pending guest-events (IRQs from LAPIC, NMIs, exceptions, software-interrupts) into VMCS fields that CPU consumes at next VMRESUME. Per-VMRESUME, CPU reads `VM_ENTRY_INTR_INFO_FIELD` (32-bit: vector | type<<8 | error-code-valid<<11 | valid<<31) and dispatches as injected event before resuming guest at vmcs.guest.RIP. Critical interaction: if guest already executing event-handler when new event arrives, KVM uses IDT-vectoring info to detect: a) double-fault (per Intel SDM rules), b) reinject lost event after VMexit during inject. Per-vmexit IDT_VECTORING_INFO_FIELD captured to identify "I was about to inject X but vmexit happened mid-vectoring".

This Tier-3 covers event-injection code in `vmx.c` (~300-400 lines) + arch-agnostic in `x86.c`.

### Acceptance Criteria

- [ ] AC-1: Inject IRQ from LAPIC: vector V queued; next vmenter VM_ENTRY_INTR_INFO valid; guest sees IRQ V.
- [ ] AC-2: Inject NMI: vmx_inject_nmi composes type=NMI; guest sees NMI handler invoked.
- [ ] AC-3: Inject exception with error code: queue_exception(vector=14, has_error_code, error_code=0); guest sees #PF with error.
- [ ] AC-4: Double-fault: inject #PF; guest #PF handler triggers another #PF; CPU converts to #DF.
- [ ] AC-5: Triple-fault: inject #DF; guest handler triggers #PF; CPU triggers TRIPLE_FAULT vmexit.
- [ ] AC-6: vmexit-mid-inject: synthetic vmexit during injection; IDT_VECTORING_INFO captures; reinject on next vmenter.
- [ ] AC-7: Live migration: pending exception preserved; resumes at destination.
- [ ] AC-8: Multi-event: IRQ + NMI both pending; NMI takes priority; IRQ delivered after NMI handled.
- [ ] AC-9: Software-interrupt INT3: instruction-length correctly set; guest sees breakpoint at correct RIP.
- [ ] AC-10: kvm-unit-tests `event-injection` test passes.

### Architecture

`VcpuVmx::inject_irq(vcpu, reinjected)`:
1. vmx := to_vmx(vcpu).
2. irq := vcpu.arch.interrupt.
3. intr := irq.nr | INTR_INFO_VALID_MASK | INTR_TYPE_EXT_INTR.
4. if irq.soft: intr |= INTR_TYPE_SOFT_INTR | (irq.instruction_len << 24).
5. VMWRITE VM_ENTRY_INTR_INFO_FIELD = intr.
6. If reinjected: skip IRR-clear (will reinject same event).
7. Else: vmcs.guest_intr_status update.

`VcpuVmx::queue_exception(vcpu, vector, has_error_code, error_code)`:
1. intr_info := vector | (INTR_TYPE_HARD_EXCEPTION << 8) | INTR_INFO_VALID_MASK.
2. If has_error_code: intr_info |= INTR_INFO_DELIVER_CODE_MASK.
3. vmcs.VM_ENTRY_EXCEPTION_ERROR_CODE = error_code.
4. VMWRITE VM_ENTRY_INTR_INFO_FIELD = intr_info.

`Vcpu::inject_pending_event(vcpu)`:
1. Acquire kvm event lock.
2. Priority loop:
   - SMI pending: enter_smm(vcpu).
   - NMI pending && can_inject_nmi: vmx_inject_nmi(vcpu).
   - Exception pending: vmx_queue_exception(vcpu, ...).
   - LAPIC IRR has pending && can_inject_irq: vmx_inject_irq(vcpu, ...).
3. Release lock.

`VcpuVmx::complete_interrupts(vcpu, idt_vectoring_info)`:
1. valid := idt_vectoring_info & VECTORING_INFO_VALID_MASK.
2. If !valid: return.
3. type := (idt_vectoring_info >> 8) & 0x7.
4. vector := idt_vectoring_info & 0xFF.
5. Switch type:
   - EXT_INTR: vcpu.arch.interrupt.pending = true; vcpu.arch.interrupt.nr = vector.
   - NMI: vcpu.arch.nmi_pending++.
   - HW_EXCEPTION: vcpu.arch.exception.pending = true; vcpu.arch.exception.vector = vector.
6. kvm_make_request(KVM_REQ_EVENT, vcpu).

### Out of Scope

- VMX core (covered in `x86-vmx.md` Tier-3)
- LAPIC (covered in `x86-lapic.md` Tier-3)
- KVM core (covered in `kvm-core.md` Tier-3)
- SMM (covered in `x86-smm.md` Tier-3)
- Nested-VMX event-inject (covered in `x86-nested-vmx.md` Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `vmx_inject_irq(vcpu, reinjected)` | IRQ inject via VM_ENTRY_INTR_INFO | `VcpuVmx::inject_irq` |
| `vmx_inject_nmi(vcpu)` | NMI inject | `VcpuVmx::inject_nmi` |
| `vmx_queue_exception(vcpu, vector, has_error_code, error_code)` | exception queue | `VcpuVmx::queue_exception` |
| `kvm_inject_pending_event(vcpu)` (arch/x86/kvm/x86.c) | per-vmenter dispatch decision | `Vcpu::inject_pending_event` |
| `kvm_can_inject_irq(vcpu)` / `kvm_can_inject_nmi(vcpu)` / `kvm_can_inject_smi(vcpu)` | per-state inject-allowed check | `Vcpu::can_inject_*` |
| `vmx_handle_exit_irqoff(vcpu)` | per-vmexit immediate-handle | `VcpuVmx::handle_exit_irqoff` |
| `__vmx_complete_interrupts(vcpu, idt_vectoring_info, ...)` | per-vmexit IDT-vectoring rebind | `VcpuVmx::complete_interrupts` |
| `enter_smm(vcpu)` | SMI entry path | covered in `x86-smm.md` |
| `exit_intr_info` (per-vCPU snapshot) | vmexit interrupt-info | per-VcpuVmx |
| `idt_vectoring_info` (per-vCPU snapshot) | per-vmexit pre-injecting info | per-VcpuVmx |
| `VM_ENTRY_INTR_INFO_FIELD` (vmcs encoding 0x4016) | inject-info field | UAPI |
| `VM_EXIT_INTR_INFO_FIELD` (vmcs encoding 0x4404) | exit-info field | UAPI |
| `IDT_VECTORING_INFO_FIELD` (vmcs encoding 0x440A) | mid-inject vmexit info | UAPI |
| `INTR_TYPE_*` (EXT_INTR / NMI / SOFT_INTR / HARD_EXCEPTION / SOFT_EXCEPTION) | inject-type encoding | UAPI |
| `kvm_make_request(KVM_REQ_EVENT, vcpu)` | per-vCPU event-pending request | shared |
| `kvm_x86_inject_exception(vcpu, ...)` | per-vendor exception-inject hook | per-vendor |

### compatibility contract

REQ-1: `VM_ENTRY_INTR_INFO_FIELD` (32-bit):
- Bits[7:0]: vector.
- Bits[10:8]: interruption type:
  - 0 = External interrupt.
  - 1 = Reserved.
  - 2 = NMI.
  - 3 = Hardware exception.
  - 4 = Software interrupt (INTn).
  - 5 = Privileged software exception (INT1).
  - 6 = Software exception (INT3, INTO).
  - 7 = Other event (INIT, MTF, SIPI).
- Bit[11]: deliver error code (set if exception with error code).
- Bits[30:12]: reserved.
- Bit[31]: valid.

REQ-2: VM_ENTRY_EXCEPTION_ERROR_CODE field:
- 32-bit error code if VM_ENTRY_INTR_INFO_FIELD.bit-11 set.
- Used for #PF, #DF, #TS, #NP, #SS, #GP, #AC, #VC, etc.

REQ-3: Per-vmenter inject flow:
1. KVM checks pending events (LAPIC IRR, NMI, exception queue).
2. Compute next-event-priority (per Intel SDM Vol 3 Sec 6.9):
   - SMI > NMI > MCE > exception > IRQ > INIT > SIPI.
3. Compose VM_ENTRY_INTR_INFO with vector + type + error-code-valid + valid bits.
4. VMWRITE VM_ENTRY_INTR_INFO_FIELD = composed.
5. If error-code: VMWRITE VM_ENTRY_EXCEPTION_ERROR_CODE = code.
6. VMRESUME → CPU injects event before guest resumes.

REQ-4: Per-vmexit IDT-vectoring detection (`__vmx_complete_interrupts`):
1. Read IDT_VECTORING_INFO_FIELD from VMCS.
2. If valid bit set: VMexit happened mid-event-injection.
3. Per-type field:
   - External-interrupt: re-queue in LAPIC IRR.
   - NMI: re-pending NMI.
   - Hardware exception: re-queue in vcpu.arch.exception (will reinject on next vmenter).
4. Defense against losing event due to vmexit during inject.

REQ-5: Double-fault detection (per Intel SDM):
- If injecting one exception triggers another exception:
- Hardware automatically converts to #DF (vector 8).
- KVM detects via comparing currently-injecting-vector + new-fault-vector against double-fault table.
- If triple-fault would occur: convert to TRIPLE_FAULT vmexit.

REQ-6: Per-vCPU exception queue:
- vcpu.arch.exception (struct kvm_queued_exception): vector, has_error_code, error_code, pending, injected.
- Pending: not yet injected at next vmenter.
- Injected: currently being injected (set right before VMRESUME; cleared on success).

REQ-7: Per-vCPU NMI queue:
- vcpu.arch.nmi_pending (count of pending NMIs).
- vcpu.arch.nmi_injected (currently being injected).

REQ-8: Per-vCPU IRQ source:
- LAPIC IRR (Interrupt Request Register).
- vmx_can_inject_irq checks: if (vmcs.guest_intr_status & RFLAGS.IF) AND (no pending higher-priority).
- LAPIC dispatches highest-priority pending IRQ via inject_irq.

REQ-9: Per-vCPU pending-event-quirks:
- KVM_X86_DISABLE_QUIRK_FORCE_EFLAGS_RF: force RFLAGS.RF=1 on inject (legacy quirk).
- Various per-event quirks for buggy guest OS detection.

REQ-10: Software-interrupt instruction-length:
- For INTn / INT3: vmcs.VM_ENTRY_INSTRUCTION_LEN field needed.
- KVM populates per-vmexit-decoded instruction length.

REQ-11: vmexit during event-injection:
- vmcs.IDT_VECTORING_INFO_FIELD captures pending state.
- KVM re-injects same event on next vmenter.
- Edge case: instruction that caused vmexit may have partial-effect; may need to roll back.

REQ-12: Live migration of pending events:
- KVM_GET_VCPU_EVENTS / _SET_VCPU_EVENTS migrate per-vCPU exception/NMI/SMI state.
- Per-vCPU IRR migrated via KVM_GET_LAPIC.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `intr_info_field_format` | INVARIANT | per-VM_ENTRY_INTR_INFO bits valid; defense against malformed value causing CPU exception. |
| `vector_range_validated` | INVARIANT | vector ∈ [0, 255]; defense against OOB-vector. |
| `error_code_only_with_exceptions` | INVARIANT | DELIVER_CODE_MASK set iff type == HW_EXCEPTION + vector has error code. |
| `priority_strict` | INVARIANT | per-vmenter priority follows SMI > NMI > MCE > exception > IRQ. |
| `idt_vectoring_reinject_idempotent` | INVARIANT | re-inject same event after vmexit produces same effect. |

### Layer 2: TLA+

`virt/kvm/event_inject_priority.tla`:
- Per-vCPU pending events: {SMI, NMI, MCE, Exception, IRQ}.
- Per-vmenter dispatch: highest-priority chosen.
- Properties:
  - `safety_priority_invariant` — chosen event is highest-priority pending.
  - `safety_one_event_per_vmenter` — at most one event injected per vmenter.
  - `liveness_pending_eventually_delivered` — every pending event eventually delivered.

`virt/kvm/idt_vectoring_reinject.tla`:
- Per-vmexit captured event-state.
- Properties:
  - `safety_no_lost_event` — vmexit-mid-inject events captured + re-injected.
  - `safety_double_fault_detection` — second exception during injection converts to #DF per Intel SDM.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `VcpuVmx::inject_irq` post: VM_ENTRY_INTR_INFO has valid bit + correct vector + type | `VcpuVmx::inject_irq` |
| `VcpuVmx::queue_exception` post: VM_ENTRY_EXCEPTION_ERROR_CODE populated; intr_info has DELIVER_CODE | `VcpuVmx::queue_exception` |
| `Vcpu::inject_pending_event` post: at most one event injected; priority respected | `Vcpu::inject_pending_event` |
| `VcpuVmx::complete_interrupts` post: per-IDT-vectoring info captured into per-vCPU pending state | `VcpuVmx::complete_interrupts` |

### Layer 4: Verus/Creusot functional

`Per-vmenter event injection: guest sees event before resuming guest-code at vmcs.guest.RIP` semantic equivalence: per-event the guest-side handler is invoked at vector V with appropriate context (error code if applicable).

### hardening

(Inherits row-1 features from `virt/kvm/x86-vmx.md` § Hardening.)

event-injection-specific reinforcement:

- **VM_ENTRY_INTR_INFO bits validated** — defense against malformed inject causing CPU exception or wrong dispatch.
- **vector range [0, 255]** — defense against OOB.
- **Strict priority order per Intel SDM** — defense against per-event mis-dispatch.
- **IDT-vectoring rebind on vmexit-mid-inject** — defense against lost event.
- **Double-fault detection** — defense against guest-controlled-fault-cascade escaping VM context.
- **Triple-fault → vmexit reason** — defense against silent-reset of guest.
- **Per-event injection bookkeeping atomic** — defense against torn pending state.
- **VM_ENTRY_INSTRUCTION_LEN populated for software-int** — defense against guest mid-instruction state-corruption.
- **Per-exception error_code only with exception types** — defense against meaningless error-code in non-exception inject.
- **NMI count bounded** — defense against unbounded NMI-pending build-up.
- **Live-migrate pending events serialized** — defense against post-migrate event loss.

