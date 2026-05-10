# Tier-3: arch/x86/kvm/lapic.c (TPR/CR8 subset) — Task Priority Register virtualization (CR8 shadow + TPR_THRESHOLD vmexit)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/x86-lapic.md
upstream-paths:
  - arch/x86/kvm/lapic.c (TPR/CR8 sections; see grep "kvm_lapic_set_tpr|kvm_lapic_get_cr8|tpr_threshold")
  - arch/x86/kvm/x86.c (kvm_set_cr8 + kvm_get_cr8)
  - arch/x86/kvm/vmx/vmx.c (vmcs.TPR_THRESHOLD + USE_TPR_SHADOW)
-->

## Summary

x86 TPR (Task Priority Register) is the LAPIC-internal priority threshold determining which interrupt priorities the CPU accepts. CR8 is a control register alias for TPR (writes to CR8 update TPR; reads from CR8 read TPR). Long-mode-32-bit OS uses TPR via APIC.TASKPRI MMIO; long-mode-64-bit uses CR8 directly. KVM virtualizes via VMX's USE_TPR_SHADOW + TPR_THRESHOLD: per-vCPU virtual_apic_page contains TPR; CPU updates without vmexit unless threshold exceeded; vmexit reason TPR_BELOW_THRESHOLD when guest lowers TPR below pending IRQ priority. AMD SVM uses VMCB.V_TPR field similarly. Critical for: optimizing IRQ-delivery on busy guest (avoiding per-MMIO-write vmexit).

This Tier-3 covers TPR/CR8 virtualization in `arch/x86/kvm/lapic.c` (~150-200 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `kvm_lapic_set_tpr(vcpu, cr8)` | per-vCPU TPR set | `Lapic::set_tpr` |
| `kvm_lapic_get_cr8(vcpu)` | per-vCPU CR8 get | `Lapic::get_cr8` |
| `kvm_set_cr8(vcpu, cr8)` (x86.c) | guest CR8 write | `Vcpu::set_cr8` |
| `kvm_get_cr8(vcpu)` (x86.c) | guest CR8 read | `Vcpu::get_cr8` |
| `vmcs.TPR_THRESHOLD` | per-vCPU pending-IRQ priority threshold | UAPI |
| `vmcs.exec_control bit USE_TPR_SHADOW` | enable TPR-shadow | UAPI |
| `vmcs.virtual_apic_page` (covered in apicv) | per-vCPU vAPIC page | shared |
| `vmcb.control.v_tpr` (SVM) | per-vCPU TPR shadow | UAPI |
| `vmcb.control.v_intr_prio` (SVM) | pending-IRQ priority | UAPI |
| `update_cr8_intercept(vcpu)` | per-vCPU recompute TPR-threshold | `Vcpu::update_cr8_intercept` |
| `EXIT_REASON_TPR_BELOW_THRESHOLD` | vmexit reason | UAPI |
| `handle_tpr_below_threshold(vcpu)` | per-vmexit handler | `VcpuVmx::handle_tpr_below_threshold` |

## Compatibility contract

REQ-1: TPR (Task Priority Register):
- LAPIC.TASKPRI MMIO at 0xFEE00080.
- CR8 alias.
- 8-bit: bits[7:4] = priority class (0-15); bits[3:0] = sub-class.
- Higher value = higher priority threshold (IRQ < TPR not delivered).

REQ-2: Per-vCPU TPR shadow:
- vcpu.arch.apic.regs[APIC_TASKPRI] (in vAPIC page).
- HW-mapped IO-coherent under APICv.

REQ-3: VMX TPR_THRESHOLD:
- vmcs.TPR_THRESHOLD := pending-IRQ-priority - 1.
- CPU comparing guest TPR < TPR_THRESHOLD: vmexit reason TPR_BELOW_THRESHOLD.
- Allows KVM to inject pending IRQ once guest TPR drops.

REQ-4: VMX USE_TPR_SHADOW:
- vmcs.exec_control bit 21.
- When set: CPU reads/writes vmcs.virtual_apic_page TPR field directly.
- Without TPR_SHADOW: every TPR access vmexits.

REQ-5: AMD SVM V_TPR + V_INTR:
- vmcb.control.v_tpr: per-vCPU TPR shadow.
- vmcb.control.v_intr_prio: pending-IRQ priority.
- vmcb.control.v_irq + .v_intr_vector for direct injection.
- Per-VMRUN: HW compares + auto-injects pending IRQ once TPR drops.

REQ-6: Per-vCPU CR8 set:
1. `kvm_set_cr8(vcpu, cr8)`:
   - Validate cr8 < 16 (only low 4 bits valid).
   - lapic_in_kernel? kvm_lapic_set_tpr(vcpu, cr8) : vcpu.arch.cr8 = cr8.

REQ-7: Per-vCPU CR8 get:
1. `kvm_get_cr8(vcpu)`:
   - lapic_in_kernel? kvm_lapic_get_cr8(vcpu) : vcpu.arch.cr8.

REQ-8: Per-vCPU update_cr8_intercept:
1. tpr := vcpu.arch.apic.regs[APIC_TASKPRI] >> 4 (priority class).
2. max_irr := find highest-priority pending IRQ class.
3. threshold := max_irr > tpr ? tpr : max_irr.
4. VMWRITE TPR_THRESHOLD = threshold.

REQ-9: TPR_BELOW_THRESHOLD vmexit handler:
1. KVM re-evaluates pending IRQ vs TPR.
2. If TPR-below-pending-IRQ: inject IRQ.
3. Else: kvm_vcpu_kick (re-evaluate priority).
4. skip_emulated_instruction.

REQ-10: 32-bit guest CR8 access:
- 32-bit OS uses MMIO writes to LAPIC.TASKPRI register.
- Without TPR_SHADOW: every access vmexits as APIC_ACCESS.

REQ-11: 64-bit guest CR8 access:
- 64-bit OS uses MOV CR8 instruction.
- Per-write: vmexit reason CR_ACCESS (CR8) OR direct via TPR_SHADOW.

REQ-12: Per-vCPU TPR-related stat:
- TPR-vmexit count, IRQ-injection count.
- Visible via debugfs.

## Acceptance Criteria

- [ ] AC-1: 64-bit Linux guest: kernel writes CR8 via MOV CR8; reads via MOV from CR8.
- [ ] AC-2: USE_TPR_SHADOW active: per-CR8 write doesn't vmexit (shadow updates).
- [ ] AC-3: TPR_BELOW_THRESHOLD vmexit: guest lowers TPR below pending IRQ; vmexit fires; KVM injects IRQ.
- [ ] AC-4: 32-bit guest LAPIC.TASKPRI MMIO: TPR_SHADOW handles direct write.
- [ ] AC-5: AMD SVM V_TPR: VMCB.v_tpr updated; CPU auto-injects pending IRQ on guest CLI/STI.
- [ ] AC-6: Per-IRQ priority: vCPU TPR=0x80; IRQ at priority 0x40 not delivered until TPR drops.
- [ ] AC-7: Live migration: TPR/CR8 state preserved.
- [ ] AC-8: Multi-vCPU: per-vCPU TPR distinct.
- [ ] AC-9: kvm-unit-tests `tpr` test passes.
- [ ] AC-10: Performance: TPR_SHADOW reduces vmexit count by ~95% on TPR-heavy workload.

## Architecture

`Lapic::set_tpr(vcpu, cr8)`:
1. apic := vcpu.arch.apic.
2. tpr := cr8 << 4 (extend 4-bit cr8 to 8-bit TPR).
3. Write apic.regs[APIC_TASKPRI] = tpr.
4. apic_update_ppr(apic) — recompute Processor Priority Register.
5. update_cr8_intercept(vcpu) — refresh TPR_THRESHOLD.

`Lapic::get_cr8(vcpu)`:
1. tpr := apic.regs[APIC_TASKPRI].
2. Return tpr >> 4.

`Vcpu::set_cr8(vcpu, cr8)`:
1. If cr8 >= 16: return -EINVAL.
2. If lapic_in_kernel: kvm_lapic_set_tpr(vcpu, cr8).
3. Else: vcpu.arch.cr8 = cr8.

`Vcpu::update_cr8_intercept(vcpu)`:
1. apic := vcpu.arch.apic.
2. max_irr := __apic_search_irr(apic).
3. tpr_class := apic.regs[APIC_TASKPRI] >> 4.
4. threshold := (max_irr >> 4) > tpr_class ? tpr_class : 0.
5. Vendor: vmx_update_cr8_intercept OR svm_update_cr8_intercept.

`VcpuVmx::handle_tpr_below_threshold(vcpu)`:
1. kvm_make_request(KVM_REQ_EVENT, vcpu).
2. Return 1 (continue).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cr8_value_bounded` | INVARIANT | per-CR8 value < 16; defense against per-OOB into TPR upper bits. |
| `tpr_class_priority_consistent` | INVARIANT | per-vCPU TPR-class matches max-IRR-priority comparison. |
| `tpr_shadow_atomic` | INVARIANT | virtual_apic_page TPR-write atomic vs HW. |
| `update_cr8_intercept_after_irr_change` | INVARIANT | every per-IRR change refreshes TPR_THRESHOLD. |

### Layer 2: TLA+

`virt/kvm/tpr_shadow.tla`:
- Per-vCPU TPR-state ∈ {Idle, BelowThreshold, AboveThreshold}.
- Properties:
  - `safety_no_inject_when_above` — IRQ not delivered when TPR > IRR-priority.
  - `safety_inject_when_below_or_equal` — IRQ delivered when TPR ≤ IRR-priority.
  - `liveness_pending_eventually_delivered` — assuming TPR eventually drops, pending IRQ delivered.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Lapic::set_tpr` post: APIC_TASKPRI updated; PPR recomputed; TPR_THRESHOLD refreshed | `Lapic::set_tpr` |
| `Vcpu::set_cr8` post: per-vCPU CR8 / TPR consistent | `Vcpu::set_cr8` |
| `Vcpu::update_cr8_intercept` post: TPR_THRESHOLD reflects per-vCPU IRR + TPR | `Vcpu::update_cr8_intercept` |
| `VcpuVmx::handle_tpr_below_threshold` post: KVM_REQ_EVENT pending | `VcpuVmx::handle_tpr_below_threshold` |

### Layer 4: Verus/Creusot functional

`Per-vCPU TPR-write: subsequent IRQ-eligibility matches IA-32-architecture rules` semantic equivalence: per-IRQ delivery iff IRR-priority ≥ TPR-class.

## Hardening

(Inherits row-1 features from `virt/kvm/x86-lapic.md` § Hardening.)

TPR-shadow-specific reinforcement:

- **Per-vCPU TPR_SHADOW gated on hw-feature-detect** — defense against running on non-TPR-SHADOW CPU.
- **Per-vCPU TPR-write atomic** — defense against torn-update during cross-CPU access.
- **TPR_THRESHOLD recomputed on every IRR change** — defense against stale-threshold causing missed IRQ.
- **CR8 value validated < 16** — defense against guest writing OOB CR8 bits.
- **PPR recomputation per APIC_TASKPRI write** — defense against stale Processor-Priority.
- **Per-vCPU virtual_apic_page IO-coherent** — defense against device-DMA-write tearing.
- **AMD V_TPR atomic-update** — defense against torn-update during VMRUN.
- **TPR_BELOW_THRESHOLD handled idempotently** — defense against repeated vmexit.
- **lapic_in_kernel switch** — defense against userspace LAPIC race with KVM.
- **Per-vmexit IRQ-priority re-evaluated** — defense against stale stale per-vCPU IRR vs TPR.
- **Live-migrate TPR shadow + virtual_apic_page** — defense against post-migrate inconsistent state.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- LAPIC core (covered in `x86-lapic.md` Tier-3)
- APICv (covered in `x86-vmx-apicv.md` Tier-3)
- AVIC (covered in `x86-svm-avic.md` Tier-3)
- KVM core (covered in `kvm-core.md` Tier-3)
- VMX core (covered in `x86-vmx.md` Tier-3)
- SVM core (covered in `x86-svm.md` Tier-3)
- Implementation code
