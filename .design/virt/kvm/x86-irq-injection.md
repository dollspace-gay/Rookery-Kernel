# Tier-3: arch/x86/kvm/x86.c (IRQ-injection subset) — KVM x86 per-vCPU IRQ injection state machine

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/x86-lapic.md
upstream-paths:
  - arch/x86/kvm/x86.c (inject_pending_event ~10742, kvm_cpu_has_injectable_intr ~10570, kvm_cpu_get_interrupt, et al.)
  - arch/x86/include/asm/kvm_host.h (kvm_queued_exception, vcpu->arch state)
-->

## Summary

Per-vCPU IRQ-injection state-machine resolves competing event sources (exception / NMI / SMI / external IRQ) into the right vmcs/vmcb event-injection field at vmenter. Per-priority order: exception (highest, except double-fault) → SMM → NMI → external IRQ. Per-vCPU `arch.exception` (pending/injected), `arch.nmi_pending`, `arch.smi_pending`, `arch.interrupt.pending` tracked. Per-vmenter `inject_pending_event` writes vmcs.entry-interrupt-info-field. Per-injection-blocked: schedule interrupt-window-vmexit (vmcs.cpu-based-controls bit), wait for IF/STI/MOV-SS-blocking lift. Critical for: correct guest interrupt priorities + vmenter-time event injection.

This Tier-3 covers IRQ-injection subset of `x86.c` (~250 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `vcpu->arch.exception` | per-vCPU exception state | `VcpuArch::exception` |
| `vcpu->arch.exception_vmexit` | per-nested L2 mirror | `VcpuArch::exception_vmexit` |
| `vcpu->arch.nmi_pending` | per-vCPU u32 NMI count | `VcpuArch::nmi_pending` |
| `vcpu->arch.nmi_injected` | per-vCPU bool current NMI in-flight | `VcpuArch::nmi_injected` |
| `vcpu->arch.smi_pending` | per-vCPU SMI count | `VcpuArch::smi_pending` |
| `vcpu->arch.interrupt` | per-vCPU pending external IRQ | `VcpuArch::interrupt` |
| `inject_pending_event()` | per-vmenter dispatcher | `Inj::inject_pending_event` |
| `kvm_cpu_has_injectable_intr()` | per-vCPU injectable check | `Inj::has_injectable_intr` |
| `kvm_cpu_get_interrupt()` | per-vCPU pop highest-pri intr | `Inj::cpu_get_interrupt` |
| `kvm_inject_pending_timer_irqs()` | per-tsc-deadline handler | `Inj::inject_pending_timer_irqs` |
| `kvm_arch_vcpu_runnable()` | per-vCPU runnable check | `Inj::vcpu_runnable` |
| `kvm_check_request()` | per-bit request handler | `Inj::check_request` |
| `kvm_x86_ops.set_irq` | per-arch inject IRQ | `KvmX86Ops::set_irq` |
| `kvm_x86_ops.set_nmi` | per-arch inject NMI | `KvmX86Ops::set_nmi` |
| `kvm_x86_ops.queue_exception` | per-arch queue #PF/#GP/etc | `KvmX86Ops::queue_exception` |
| `kvm_x86_ops.enable_irq_window` | per-arch enable IRQ-window | `KvmX86Ops::enable_irq_window` |
| `kvm_x86_ops.enable_nmi_window` | per-arch enable NMI-window | `KvmX86Ops::enable_nmi_window` |

## Compatibility contract

REQ-1: Priority order at vmenter:
- exception.injected > nmi.injected > exception.pending > smi.pending > nmi.pending > apic.has_interrupt > extint (PIC).

REQ-2: Per-exception state (kvm_queued_exception):
- pending: bool — needs delivery.
- injected: bool — currently in flight (vmcs has it).
- has_error_code: bool.
- nr: u8 vector (0..31).
- error_code: u32.
- nested_apf: bool (paravirt #PF).
- has_payload: bool / payload: u64 (CR2 / DR6 etc).

REQ-3: kvm_queue_exception(vcpu, vector):
- If !exception.pending ∧ !exception.injected:
  - exception.pending = true.
  - exception.nr = vector.
  - Per-vector specifics (e.g. #PF): exception.error_code, exception.payload.

REQ-4: NMI injection:
- vcpu.nmi_pending counter (max ~2 simultaneous in real HW; KVM caps at 1).
- nmi_injected = true while in flight (HW IRET clears).
- Per-vmenter: if !nmi_injected ∧ nmi_pending > 0: inject NMI.

REQ-5: SMI injection:
- vcpu.smi_pending count.
- Per-vmenter: if smi_pending > 0 ∧ !nested_run_pending ∧ allowed: enter_smm(vcpu).

REQ-6: External IRQ:
- vcpu.interrupt.pending: legacy PIC-routed IRQ (extint).
- LAPIC IRQ: kvm_cpu_get_interrupt reads ISR/IRR + TPR.
- Per-vmenter: if has_injectable_intr ∧ allowed: vector = cpu_get_interrupt.

REQ-7: Injection-blocked conditions:
- IF=0 (RFLAGS.IF clear).
- MOV-SS pending (1-instr blocking).
- STI pending (1-instr blocking).
- nested-run pending.
- Per-blocked: enable_irq_window (sets vmcs.cpu-based-controls.irq_window_exiting).

REQ-8: NMI-blocked conditions:
- IRET-block (nmi_blocked from vmcs.guest_intr_state).
- Per-blocked: enable_nmi_window.

REQ-9: inject_pending_event flow:
1. If exception.injected: defer (vmexit handler will re-inject).
2. If nmi_injected: defer.
3. If exception.pending: queue_exception; exception.injected = true; pending = false.
4. ElIf smi_pending ∧ allowed: enter_smm.
5. ElIf nmi_pending > 0 ∧ allowed: queue_nmi; nmi_injected = true.
6. ElIf has_injectable_intr ∧ allowed: vector = cpu_get_interrupt; queue_irq.
7. If pending events but blocked: enable_irq_window or enable_nmi_window.

REQ-10: Per-VMX vmcs.entry-interrupt-info (32 bits):
- Bits[7:0] vector.
- Bits[10:8] interrupt-type (0=ext, 2=NMI, 3=hw-exception, 6=sw-exception, 7=privileged-sw-exception).
- Bit[11] error-code-valid.
- Bit[12..30] reserved.
- Bit[31] valid.
- Per-error-code: vmcs.entry-error-code.

REQ-11: Per-SVM vmcb.control.event_inj (32 bits):
- Same general layout.
- Per-error-code: vmcb.control.event_inj_err.

REQ-12: Per-double-fault detection:
- If exception.pending while exception.injected of certain types: raise #DF.
- Per-#DF while #DF in flight: triple-fault → KVM_EXIT_SHUTDOWN.

## Acceptance Criteria

- [ ] AC-1: Guest #PF: kvm_queue_exception(vcpu, PF_VECTOR, err_code, cr2). Subsequent vmenter: vmcs.entry-interrupt-info has vector=14.
- [ ] AC-2: kvm_inject_nmi: nmi_pending++; vmenter injects NMI when allowed.
- [ ] AC-3: External-IRQ via APIC: kvm_cpu_get_interrupt returns highest-priority unmasked vector.
- [ ] AC-4: IF=0: enable_irq_window invoked; vmcs.cpu-based-controls.irq_window_exiting=1.
- [ ] AC-5: MOV-SS-blocking: irq-window deferred per-block lift.
- [ ] AC-6: NMI while nmi_blocked: enable_nmi_window invoked.
- [ ] AC-7: SMI: enter_smm invoked; vcpu.arch.hflags |= HF_SMM_MASK.
- [ ] AC-8: Double-fault: subsequent inject during exception.injected of contributory class causes #DF.
- [ ] AC-9: Triple-fault: KVM_EXIT_SHUTDOWN reason.
- [ ] AC-10: Pending IRQ at vmenter + nested-run-pending: deferred.

## Architecture

Per-vCPU exception state:

```
struct KvmQueuedException {
  pending: bool,
  injected: bool,
  has_error_code: bool,
  nr: u8,
  error_code: u32,
  has_payload: bool,
  payload: u64,
  nested_apf: bool,
}

struct VcpuArch {
  ...
  exception: KvmQueuedException,
  exception_vmexit: KvmQueuedException,           // for nested L2 mirror
  nmi_pending: u32,
  nmi_injected: bool,
  smi_pending: u8,
  interrupt: KvmQueuedInterrupt,                  // for ExtINT
  hflags: u32,                                    // HF_SMM_MASK, etc.
  nested_run_pending: bool,
}

struct KvmQueuedInterrupt {
  injected: bool,
  soft: bool,
  nr: u8,
}
```

`Inj::inject_pending_event(vcpu) -> i32`:
1. If vcpu.arch.exception.injected:
   - return 0  // arch handler re-injects via vmexit handler.
2. If vcpu.arch.nmi_injected:
   - return 0.
3. exception_pending = vcpu.arch.exception.pending.
4. If exception_pending ∧ kvm_x86_ops.exception_allowed(vcpu):
   - kvm_x86_ops.queue_exception(vcpu).
   - vcpu.arch.exception.injected = true.
   - vcpu.arch.exception.pending = false.
5. ElIf vcpu.arch.smi_pending > 0 ∧ kvm_x86_ops.smi_allowed(vcpu):
   - enter_smm(vcpu).
   - --vcpu.arch.smi_pending.
6. ElIf vcpu.arch.nmi_pending > 0 ∧ kvm_x86_ops.nmi_allowed(vcpu):
   - kvm_x86_ops.queue_nmi(vcpu).
   - vcpu.arch.nmi_injected = true.
   - --vcpu.arch.nmi_pending.
7. ElIf Inj::has_injectable_intr(vcpu) ∧ kvm_x86_ops.interrupt_allowed(vcpu):
   - irq_nr = Inj::cpu_get_interrupt(vcpu).
   - kvm_x86_ops.set_irq(vcpu, irq_nr).
8. /* Pending but blocked → enable windows */
9. If vcpu.arch.exception.pending: kvm_x86_ops.queue_exception_vmcs.
10. If vcpu.arch.nmi_pending > 0 ∧ !kvm_x86_ops.nmi_allowed(vcpu): kvm_x86_ops.enable_nmi_window(vcpu).
11. If kvm_cpu_has_injectable_intr(vcpu) ∧ !kvm_x86_ops.interrupt_allowed(vcpu): kvm_x86_ops.enable_irq_window(vcpu).
12. Return 0.

`Inj::has_injectable_intr(vcpu) -> bool`:
1. If vcpu.arch.interrupt.pending: return true.    // ExtINT
2. If kvm_apic_has_pending_init_or_sipi(vcpu): return true.
3. If kvm_apic_has_interrupt(vcpu): return true.
4. Return false.

`Inj::cpu_get_interrupt(vcpu) -> i32`:
1. If kvm_check_extint_irq(vcpu): return kvm_pic_read_irq(kvm).
2. Else: return kvm_apic_acknowledge_interrupt(vcpu).

`Inj::queue_exception(vcpu, nr, has_error, error_code, has_payload, payload, reinject)`:
1. If !reinject ∧ vcpu.arch.exception.injected:
   - /* double-fault classify */
   - vector_check_double_fault: if combined (exception.nr, nr) is contributory + contributory or page-fault + ...:
     - vcpu.arch.exception.nr = DF_VECTOR; clear original.
   - Return.
2. vcpu.arch.exception.pending = true.
3. vcpu.arch.exception.nr = nr.
4. vcpu.arch.exception.has_error_code = has_error.
5. vcpu.arch.exception.error_code = error_code.
6. vcpu.arch.exception.has_payload = has_payload.
7. vcpu.arch.exception.payload = payload.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `priority_order` | INVARIANT | exception.injected > nmi.injected > exception.pending > smi.pending > nmi.pending > extint > apic. |
| `at_most_one_injected_per_class` | INVARIANT | per-class: at most one injected at a time. |
| `nmi_pending_le_max` | INVARIANT | vcpu.arch.nmi_pending ≤ KVM_MAX_NMI_PENDING. |
| `exception_clear_on_inject` | INVARIANT | post-inject: exception.injected=true; exception.pending=false. |
| `df_when_two_contributory` | INVARIANT | two-contributory exceptions ⟹ #DF queued. |

### Layer 2: TLA+

`virt/kvm/irq_inject.tla`:
- Per-vCPU exception/NMI/SMI/IRQ queuing + per-vmenter dispatch.
- Properties:
  - `safety_no_lost_exception` — per-pending exception eventually injected unless cleared.
  - `safety_window_enabled_when_blocked` — per-pending + blocked ⟹ irq_window or nmi_window enabled.
  - `liveness_pending_eventually_injected` — per-pending + unblocked ⟹ injected at next vmenter.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Inj::inject_pending_event` post: per-priority highest pending injected | `Inj::inject_pending_event` |
| `Inj::queue_exception` post: exception.pending=true OR #DF | `Inj::queue_exception` |
| `Inj::has_injectable_intr` post: returns true ⟺ at least one IRQ source non-empty | `Inj::has_injectable_intr` |
| `Inj::cpu_get_interrupt` post: returns vector from PIC or LAPIC | `Inj::cpu_get_interrupt` |

### Layer 4: Verus/Creusot functional

`Per-event source competing → priority-ordered injection at vmenter; per-blocked-state delays via window-vmexit; per-#DF on contributory pair` semantic equivalence: per-Intel SDM §6.15 + AMD APM exception-prio.

## Hardening

(Inherits row-1 features from `virt/kvm/kvm-core.md` § Hardening.)

IRQ-injection-specific reinforcement:

- **Per-priority order strict** — defense against per-class injection out-of-order.
- **Per-injected state tracked atomic** — defense against per-vmenter race.
- **Per-NMI capped at MAX_NMI_PENDING (1)** — defense against per-NMI flood.
- **Per-SMI gated by allowed-state** — defense against per-SMI-during-nested mismatch.
- **Per-double-fault detection** — defense against per-spurious injection cascade.
- **Per-triple-fault → KVM_EXIT_SHUTDOWN** — defense against per-VM hung infinite-#DF loop.
- **Per-exception_vmexit nested mirror** — defense against L1 missing L2's exception state.
- **Per-window-enable when blocked** — defense against per-IF=0 leaving IRQ pending forever.
- **Per-nested_run_pending defers all** — defense against per-injecting-during-vmenter-prep race.
- **Per-vector range bounded [0, 255]** — defense against per-config OOB.
- **Per-arch queue_exception_vmcs flushes vmcs/vmcb** — defense against per-shadow stale.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- LAPIC core (covered in `x86-lapic.md` Tier-3)
- VMX event-inj details (covered in `x86-vmx-eventinj.md` Tier-3)
- IOAPIC / PIC (covered in respective Tier-3)
- SMM (covered in `x86-smm.md` Tier-3)
- Implementation code
