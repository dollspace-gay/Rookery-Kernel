# Tier-3: arch/x86/kvm/irq.c — KVM x86 IRQ delivery dispatch (LAPIC + IOAPIC + PIC + MSI fan-in)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/irqchip.md
upstream-paths:
  - arch/x86/kvm/irq.c (~634 lines)
  - arch/x86/kvm/irq.h
-->

## Summary

Per-VM x86 IRQ-delivery glue: routes per-GSI assertions and per-MSI message-cycles to the right per-vCPU LAPIC. `kvm_set_msi` translates {address_lo, address_hi, data} → kvm_lapic_irq → per-LAPIC accept (apicv-fast or normal). `kvm_arch_set_irq_inatomic` is the atomic-context fast path called from posted-IRQ flows. Per-x86 set_routing_entry parses userspace KVM_IRQ_ROUTING_* into `e.set` callback (kvm_set_msi / kvm_set_pic_irq / kvm_set_ioapic_irq / kvm_xen_set_evtchn / kvm_hv_set_sint). Per-VM EOI broadcast hookup. Critical for: PCI MSI/MSI-X, IOAPIC pin-asserted IRQs, paravirt-SVM-AVIC + VMX-APICv handling.

This Tier-3 covers `arch/x86/kvm/irq.c` (~634 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `kvm_set_msi()` | per-MSI route dispatch | `IrqArch::set_msi` |
| `kvm_set_pic_irq()` | per-IOAPIC-routed-PIC pin | `IrqArch::set_pic_irq` |
| `kvm_set_ioapic_irq()` | per-IOAPIC pin | `IrqArch::set_ioapic_irq` |
| `kvm_arch_set_irq_inatomic()` | atomic-context fast path | `IrqArch::set_irq_inatomic` |
| `kvm_set_routing_entry()` | per-arch routing-entry parser | `IrqArch::set_routing_entry` |
| `kvm_msi_to_lapic_irq()` | per-MSI → kvm_lapic_irq decode | `IrqArch::msi_to_lapic_irq` |
| `kvm_irq_delivery_to_apic()` | per-LAPIC delivery (slow-path) | `IrqArch::delivery_to_apic` |
| `kvm_irq_delivery_to_apic_fast()` | per-LAPIC delivery (fast-path) | `IrqArch::delivery_to_apic_fast` |
| `kvm_arch_post_irq_routing_update()` | post-routing-update hook | `IrqArch::post_routing_update` |
| `kvm_register_irq_ack_notifier()` | per-IRQ ack-notifier register | `IrqArch::register_ack_notifier` |
| `kvm_notify_acked_irq()` | per-EOI broadcast | `IrqArch::notify_acked_irq` |
| `kvm_arch_irq_bypass_*()` | per-VFIO bypass hooks | `IrqArch::bypass_*` |
| `KVM_MSI_VALID_DEVID` | per-MSI flag (interrupt-remapping) | UAPI |

## Compatibility contract

REQ-1: Per-MSI message decode (kvm_msi_to_lapic_irq):
- address_lo bits[19:12]: dest_id (8 bits; Logical or Physical).
- address_lo bits[11:11]: dest_mode (0=Phys, 1=Logical).
- address_lo bits[3:2]: redirection-hint (1=Lowest-Priority).
- address_lo bits[2]: trigger-mode (0=Edge).
- address_hi bits[31:0]: dest_id high-bits (x2APIC).
- data bits[7:0]: vector.
- data bits[10:8]: delivery-mode (Fixed / LowestPrio / SMI / NMI / INIT / SIPI / extINT).
- data bits[14]: assert-edge.
- data bits[15]: trigger-mode.

REQ-2: kvm_set_msi (per-routing-entry handler):
- e.msi.{address_lo, address_hi, data} → kvm_msi_to_lapic_irq.
- kvm_irq_delivery_to_apic(kvm, NULL, &irq, NULL).
- Returns: 0 = no recipient; > 0 = #-of-recipients-injected; -1 = error.

REQ-3: kvm_set_pic_irq (per-routing-entry):
- e.irqchip.pin (0..15) → kvm_pic_set_irq.
- Routes through master/slave PIC pair.

REQ-4: kvm_set_ioapic_irq (per-routing-entry):
- e.irqchip.pin (0..23) → kvm_ioapic_set_irq.
- Per-RTE delivery as per `x86-ioapic.md`.

REQ-5: kvm_arch_set_irq_inatomic (fast path):
- Caller from posted-IRQ-VFIO context (cannot sleep).
- For MSI: try delivery_to_apic_fast; if not delivered: -EAGAIN (kick to threaded ctx).

REQ-6: kvm_set_routing_entry (per-arch parser):
- ue.type == KVM_IRQ_ROUTING_IRQCHIP:
  - Sub-type IRQCHIP_PIC_MASTER / IRQCHIP_PIC_SLAVE / IRQCHIP_IOAPIC.
  - e.set = appropriate handler.
- ue.type == KVM_IRQ_ROUTING_MSI:
  - e.set = kvm_set_msi.
  - Validate MSI-flags (KVM_MSI_VALID_DEVID).
- ue.type == KVM_IRQ_ROUTING_HV_SINT: e.set = kvm_hv_set_sint.
- ue.type == KVM_IRQ_ROUTING_XEN_EVTCHN: e.set = kvm_xen_set_evtchn.

REQ-7: Per-irq-ack-notifier:
- Per-IOAPIC level-triggered RTE may register a notifier.
- Per-EOI broadcast: walk notifiers; invoke for matching GSI.
- E.g. PIT registers ACK notifier on IRQ0 to track missed-tick reinject.

REQ-8: Per-VFIO bypass:
- kvm_arch_irq_bypass_add_producer: VFIO-IRQ producer registers.
- kvm_arch_irq_bypass_del_producer: unregister.
- Per-bypass updates IRTE.posted=1 + PID-pointer (see `x86-posted-intr.md`).

REQ-9: kvm_arch_post_irq_routing_update:
- Per-routing-table swap: re-update IOAPIC RTE-cache + VFIO-PI IRTEs.

REQ-10: Per-LAPIC-mode dependent dispatch:
- xAPIC: 8-bit destination.
- x2APIC: 32-bit destination.
- LowestPriority: lowest-TPR vCPU among destinations.

## Acceptance Criteria

- [ ] AC-1: Userspace KVM_SET_GSI_ROUTING with MSI entry: kvm_set_routing_entry installs e.set = kvm_set_msi.
- [ ] AC-2: kvm_set_msi(addr=0xFEE00000, data=0x4030): vector 0x30, dest BSP, edge-trig.
- [ ] AC-3: kvm_irq_delivery_to_apic_fast: delivers without slow-path lock.
- [ ] AC-4: PIC IRQ assert via kvm_set_pic_irq: master PIC IRR-bit set.
- [ ] AC-5: IOAPIC pin assert via kvm_set_ioapic_irq: RTE delivery.
- [ ] AC-6: x2APIC msi: 32-bit dest_id used.
- [ ] AC-7: LowestPriority delivery: lowest-TPR vCPU receives.
- [ ] AC-8: kvm_arch_set_irq_inatomic with non-deliverable IRQ: returns -EAGAIN.
- [ ] AC-9: irq-ack notifier: PIT IRQ0 ack triggers reinject decrement.
- [ ] AC-10: VFIO bypass add: posted-IRQ IRTE updated.

## Architecture

Per-MSI decode helper:

```
struct KvmLapicIrq {
  vector: u8,
  delivery_mode: u8,                              // APIC_DM_*
  dest_mode: u8,                                  // 0=Phys / 1=Logical
  level: u8,
  trig_mode: u8,
  shorthand: u8,
  dest_id: u32,
  msi_redir_hint: bool,
}

fn IrqArch::msi_to_lapic_irq(kvm, e, irq):
  irq.dest_id = (e.msi.address_lo & 0xFF000) >> 12;
  if x2apic_format(e):
    irq.dest_id |= e.msi.address_hi << 8;
  irq.vector = e.msi.data & 0xFF;
  irq.delivery_mode = (e.msi.data >> 8) & 0x7;
  irq.dest_mode = (e.msi.address_lo >> 2) & 1;
  irq.trig_mode = (e.msi.data >> 15) & 1;
  irq.level = (e.msi.data >> 14) & 1;
  irq.shorthand = APIC_DEST_NOSHORT;
  irq.msi_redir_hint = (e.msi.address_lo >> 3) & 1;
```

`IrqArch::set_msi(e, kvm, irq_source_id, level, line_status) -> i32`:
1. If !level: return -1.
2. kvm_msi_to_lapic_irq(kvm, e, &irq).
3. Return kvm_irq_delivery_to_apic(kvm, NULL, &irq, NULL).

`IrqArch::set_pic_irq(e, kvm, irq_source_id, level, line_status) -> i32`:
1. pin = e.irqchip.pin.
2. Return kvm_pic_set_irq(e, kvm, irq_source_id, pin, level, line_status).

`IrqArch::set_ioapic_irq(e, kvm, irq_source_id, level, line_status) -> i32`:
1. pin = e.irqchip.pin.
2. Return kvm_ioapic_set_irq(kvm.arch.vioapic, pin, level, line_status).

`IrqArch::set_irq_inatomic(e, kvm, irq_source_id, level, line_status) -> i32`:
1. If level == 0: return -1.
2. switch e.type:
   - KVM_IRQ_ROUTING_MSI:
     - kvm_msi_to_lapic_irq(kvm, e, &irq).
     - if kvm_irq_delivery_to_apic_fast(kvm, NULL, &irq, &r): return r.
   - Else: not supported in atomic ctx.
3. Return -EWOULDBLOCK.

`IrqArch::set_routing_entry(kvm, e, ue) -> Result<()>`:
1. switch ue.type:
   - KVM_IRQ_ROUTING_IRQCHIP:
     - if pic_in_kernel ∧ (irqchip == PIC_MASTER ∨ PIC_SLAVE): e.set = set_pic_irq; e.data.irqchip.{irqchip, pin}.
     - elif ioapic_in_kernel ∧ irqchip == IOAPIC: e.set = set_ioapic_irq; e.data.irqchip.{irqchip=IOAPIC, pin}.
     - else: return Err(EINVAL).
   - KVM_IRQ_ROUTING_MSI:
     - if (ue.flags & KVM_MSI_VALID_DEVID) without IRQ-remapping cap: return Err.
     - e.set = set_msi.
     - e.data.msi.{address_lo, address_hi, data, devid}.
   - KVM_IRQ_ROUTING_HV_SINT: e.set = kvm_hv_set_sint; e.data.hv_sint.{vcpu, sint}.
   - KVM_IRQ_ROUTING_XEN_EVTCHN: e.set = kvm_xen_set_evtchn; e.data.xen.{type, port_or_pirq, vcpu_idx}.
   - else: return Err(EINVAL).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pic_pin_lt_16` | INVARIANT | per-set_pic_irq: e.irqchip.pin < 16. |
| `ioapic_pin_lt_pins` | INVARIANT | per-set_ioapic_irq: e.irqchip.pin < KVM_IOAPIC_NUM_PINS. |
| `msi_vector_in_range` | INVARIANT | per-set_msi: irq.vector ∈ [0x10, 0xFE]. |
| `inatomic_only_msi` | INVARIANT | per-set_irq_inatomic: only MSI handled in atomic ctx. |
| `routing_entry_set_non_null` | INVARIANT | post-set_routing_entry: e.set != null. |

### Layer 2: TLA+

`virt/kvm/irq_x86.tla`:
- Per-routing-entry parse + per-set dispatch + per-LAPIC delivery.
- Properties:
  - `safety_no_dispatch_to_unsupported_chip` — pic_in_kernel/ioapic_in_kernel checked at parse.
  - `safety_msi_decode_consistent` — per-MSI decode produces valid kvm_lapic_irq fields.
  - `liveness_routed_irq_eventually_delivered` — per-asserted GSI ⟹ LAPIC-IRR or vmexit.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `IrqArch::set_msi` post: kvm_irq_delivery_to_apic invoked with decoded irq | `IrqArch::set_msi` |
| `IrqArch::set_routing_entry` post: per-type handler installed; data populated | `IrqArch::set_routing_entry` |
| `IrqArch::set_irq_inatomic` post: returns delivery-count ∨ -EWOULDBLOCK | `IrqArch::set_irq_inatomic` |
| `IrqArch::msi_to_lapic_irq` post: per-bit decode matches Intel SDM §10.11 | `IrqArch::msi_to_lapic_irq` |

### Layer 4: Verus/Creusot functional

`Per-userspace KVM_SET_GSI_ROUTING(MSI) → e.set=set_msi → kvm_set_irq → set_msi → decode → delivery_to_apic → vCPU LAPIC IRR set` semantic equivalence: per-MSI matches PCI-Spec MSI-X message format + Intel SDM APIC.

## Hardening

(Inherits row-1 features from `virt/kvm/irqchip.md` § Hardening.)

x86-irq-arch-specific reinforcement:

- **Per-MSI x2APIC dest decoded only when x2apic_msi_format** — defense against per-32-bit-dest in xAPIC mode.
- **Per-pic-pin/ioapic-pin range-checked** — defense against per-pin OOB.
- **Per-set_irq_inatomic only handles MSI** — defense against per-PIC/IOAPIC non-atomic ops in atomic ctx.
- **Per-routing-entry handler non-null after parse** — defense against per-stale fn-ptr crash.
- **Per-VFIO bypass requires post-routing-update propagation** — defense against per-IRTE stale after reroute.
- **Per-MSI-DEVID requires IRQ-remap cap** — defense against per-incomplete MSI for IOMMU.
- **Per-xen evtchn requires CAP_KVM_XEN_HVM** — defense against per-non-Xen VM accidentally enabling Xen routing.
- **Per-irq-ack notifier scope per-VM** — defense against cross-VM EOI leakage.
- **Per-set_pic_irq guards on pic_in_kernel** — defense against per-userspace-PIC routing leak.
- **Per-set_ioapic_irq guards on ioapic_in_kernel** — defense against per-split-irqchip mis-route.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- KVM cross-arch IRQ routing (covered in `irqchip.md` Tier-3)
- LAPIC core (covered in `x86-lapic.md` Tier-3)
- IOAPIC (covered in `x86-ioapic.md` Tier-3)
- PIC (covered in `x86-pic.md` Tier-3)
- Hyper-V SynIC (covered in `x86-hyperv.md` Tier-3)
- Xen evtchn (covered in `x86-xen.md` Tier-3)
- Posted-IRQ (covered in `x86-posted-intr.md` Tier-3)
- Implementation code
