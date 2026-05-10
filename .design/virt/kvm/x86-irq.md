# Tier-3: arch/x86/kvm/irq.c — KVM in-kernel IRQ chip dispatch (PIC + IOAPIC + PIT + IRQ-routing)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/kvm-core.md
upstream-paths:
  - arch/x86/kvm/irq.c
  - arch/x86/kvm/i8259.c
  - arch/x86/kvm/ioapic.c
  - arch/x86/kvm/i8254.c
  - arch/x86/kvm/irq.h
  - arch/x86/kvm/ioapic.h
  - include/uapi/linux/kvm.h (KVM_IRQ_LINE / KVM_IRQ_LINE_STATUS / KVM_SET_GSI_ROUTING / KVM_CREATE_IRQCHIP / KVM_SET_IRQCHIP / KVM_GET_IRQCHIP)
-->

## Summary

KVM in-kernel IRQ chips emulate the Intel-platform legacy IRQ hardware: i8259 PIC (Programmable Interrupt Controller, master+slave with 16 IRQ lines), IOAPIC (24-pin Advanced PIC for PCI-era platforms), i8254 PIT (Programmable Interval Timer for legacy timer IRQ0). All three are emulated entirely in-kernel under `KVM_CREATE_IRQCHIP` so userspace QEMU does not need to handle every IRQ delivery (which would require KVM_EXIT_IO per-IRQ assertion). irq.c is the per-vCPU dispatcher that arbitrates between PIC vs LAPIC vs SMI/NMI/INIT for the next event to inject. KVM_SET_GSI_ROUTING configures per-GSI (Global System Interrupt) → IRQ chip-pin mapping.

This Tier-3 covers `irq.c` (~634) + `i8259.c` (~655) + `ioapic.c` (~779) + `i8254.c` (~816).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `kvm_arch_vcpu_runnable(vcpu)` | per-vCPU runnable check (used by sched-out) | `Vcpu::runnable` |
| `kvm_cpu_get_interrupt(vcpu)` | get next IRQ to inject (LAPIC + ExtINT) | `Irq::cpu_get_interrupt` |
| `kvm_cpu_has_interrupt(vcpu)` | check if any IRQ pending | `Irq::cpu_has_interrupt` |
| `kvm_cpu_has_extint(vcpu)` | check ExtINT-mode (PIC) interrupt | `Irq::cpu_has_extint` |
| `kvm_cpu_get_extint(vcpu)` | get next PIC-routed IRQ | `Irq::cpu_get_extint` |
| `kvm_pic_init(kvm)` (i8259.c) | per-VM PIC alloc + register MMIO | `KvmPic::init` |
| `kvm_pic_destroy(kvm)` | per-VM PIC teardown | `KvmPic::destroy` |
| `kvm_pic_set_irq(pic, irq, level)` | per-IRQ assert/deassert | `KvmPic::set_irq` |
| `kvm_pic_read_irq(kvm)` | per-vCPU INTA cycle | `KvmPic::read_irq` |
| `pic_lock(...)` / `_unlock(...)` | per-PIC spinlock | `KvmPic::lock` |
| `kvm_ioapic_init(kvm)` (ioapic.c) | per-VM IOAPIC alloc | `KvmIoapic::init` |
| `kvm_ioapic_destroy(kvm)` | per-VM IOAPIC teardown | `KvmIoapic::destroy` |
| `kvm_ioapic_set_irq(ioapic, irq, level)` | per-pin assert/deassert | `KvmIoapic::set_irq` |
| `kvm_ioapic_send_eoi(...)` | per-pin EOI handling | `KvmIoapic::send_eoi` |
| `kvm_ioapic_update_eoi(...)` | per-pin EOI update | `KvmIoapic::update_eoi` |
| `ioapic_service(ioapic, irq, line_status)` | per-pin → LAPIC IRQ delivery | `KvmIoapic::service` |
| `kvm_create_pit(kvm, flags)` (i8254.c) | per-VM PIT alloc | `KvmPit::create` |
| `kvm_free_pit(kvm)` | per-VM PIT free | `KvmPit::destroy` |
| `pit_set_gate(...)` / `pit_get_gate(...)` | per-channel gate ctrl | `KvmPit::set_gate` |
| `pit_load_count(...)` | per-channel reload-count program | `KvmPit::load_count` |
| `pit_timer_fn(hrtimer)` | per-channel hrtimer callback | `KvmPit::timer_fn` |
| `kvm_set_irq_routing(...)` | KVM_SET_GSI_ROUTING ioctl | `Kvm::set_irq_routing` |
| `kvm_set_irq(kvm, irq_source_id, gsi, level, line_status)` | dispatch GSI to PIC/IOAPIC | `Kvm::set_irq` |
| `kvm_irq_delivery_to_apic(...)` | per-GSI → LAPIC delivery (MSI path) | `Kvm::irq_delivery_to_apic` |

## Compatibility contract

REQ-1: KVM_CREATE_IRQCHIP ioctl creates: PIC (master + slave), IOAPIC, in-kernel LAPIC for each vCPU. After creation, all interrupt routing is in-kernel until VM destroy.

REQ-2: i8259 PIC emulation (i8259.c):
- Master + slave PIC; total 16 IRQ lines (IRQ 0-7 master, IRQ 8-15 slave routed via master pin 2).
- Per-PIC state: imr (interrupt-mask reg), isr (in-service reg), irr (interrupt-request reg), priority_add, etc.
- Programmed via I/O ports 0x20/0x21 (master) + 0xA0/0xA1 (slave); KVM emulates port-IO via in-kernel MMIO/IO bus dispatch.
- ICW1-ICW4 init sequence + OCW1-OCW3 operation commands.
- Per-IRQ cascade through priority + masking → INTA cycle yields vector.

REQ-3: IOAPIC emulation (ioapic.c):
- 24-pin (or more on some platforms; KVM emulates 24).
- Per-pin redirection-table entry (RTE): destination, vector, delivery-mode, polarity, trigger-mode, mask.
- IOAPIC programmed via MMIO 0xFEC00000+; KVM emulates 4-byte IOREGSEL + IOWIN registers.
- Per-pin level-triggered: EOI broadcast resets remote-IRR.
- Edge-triggered: pulse-trigger.

REQ-4: i8254 PIT emulation (i8254.c):
- 3 channels: ch0 (system timer at IRQ0), ch1 (DMA refresh; obsolete), ch2 (PC speaker).
- Per-channel: count, mode, gate, output.
- Modes 0-5 (interrupt-on-terminal, hardware-retriggerable, rate-generator, square-wave, software-strobe, hardware-strobe).
- Per-channel hrtimer drives expiration → IRQ0 assertion.

REQ-5: Per-vCPU IRQ dispatch (`kvm_cpu_get_interrupt`):
1. Check LAPIC pending (`kvm_apic_has_interrupt`); if yes: return LAPIC vector.
2. Else check ExtINT-mode (LAPIC LVT_LINT0 in ExtINT delivery): `kvm_cpu_get_extint`.
3. ExtINT path: `kvm_pic_read_irq(kvm)` → INTA cycle → returns PIC-routed vector.
4. Else: no IRQ.

REQ-6: KVM_SET_GSI_ROUTING:
- Per-GSI route: type (irqchip / msi / hv_sint / xen_evtchn) + parameters.
- irqchip: per-(chip, pin) — chip ∈ {PIC_MASTER, PIC_SLAVE, IOAPIC}; pin ∈ [0..15] for PIC, [0..23] for IOAPIC.
- msi: per-(addr_lo, addr_hi, data) — direct-message IRQ to LAPIC bypassing PIC/IOAPIC.

REQ-7: Per-IRQ source-id tracking:
- Multiple sources (userspace, kernel-emulated devices, irqfd) may assert same GSI.
- `kvm_irq_source_id` per-source registered.
- Per-GSI level-tracked across all sources via shared bitmap + reference-count.
- Defense against spurious deassert by one source clearing IRQ asserted by another.

REQ-8: Per-pin RTE programming:
- IOAPIC RTE bits 0..7 = vector; 8..10 = delivery-mode; 11 = dest-mode; 12 = delivery-status; 13 = polarity; 14 = remote-IRR; 15 = trigger-mode; 16 = mask; 56..63 = destination.

REQ-9: KVM_GET_IRQCHIP / KVM_SET_IRQCHIP for snapshot/restore (live migration):
- Per-PIC complete state (imr, isr, irr, etc.).
- Per-IOAPIC state (24 RTEs + irr + ioregsel).

REQ-10: Per-PIT channel `pit_lock` + `pit_state` mutex serialize state changes vs hrtimer fire.

REQ-11: PIT gate signal:
- For modes 0/2/3: gate must be 1 for counting.
- For modes 1/5: rising-edge of gate triggers count-load + count-down.

REQ-12: irqfd integration:
- userspace eventfd writes → `eventfd_signal` → KVM-irqfd workqueue → `kvm_set_irq` per pre-registered GSI.
- Bypasses ioctl per-IRQ; high-throughput vhost-net + virtio.

## Acceptance Criteria

- [ ] AC-1: KVM_CREATE_IRQCHIP succeeds; per-VM PIC + IOAPIC + LAPIC instantiated.
- [ ] AC-2: Boot Linux guest under qemu+kvm: BIOS programs PIC + IOAPIC + PIT; guest IRQ0 (timer) drives jiffies advancement.
- [ ] AC-3: Guest IO-APIC programming: Linux guest's i8259-mode + IOAPIC-mode bringup both work.
- [ ] AC-4: PIT timer at 100Hz: guest perceived clock advancement matches host wall clock to ±0.1%.
- [ ] AC-5: Per-IRQ source-id ref-count: 2 userspace asserts on same GSI; 1 deassert keeps GSI active; 2 deasserts clear.
- [ ] AC-6: Live migration: KVM_GET_IRQCHIP+ KVM_SET_IRQCHIP roundtrip preserves PIC + IOAPIC state.
- [ ] AC-7: irqfd: vhost-net pre-registered GSI-eventfd; per-IRQ delivery via eventfd signal; guest sees IRQ.
- [ ] AC-8: kvm-unit-tests `pit` + `ioapic` + `pic` tests pass.
- [ ] AC-9: x86 KVM `kvmclock_irq_pin_test` + `apic_test` pass.

## Architecture

`KvmIrq` per-VM struct:

```
struct KvmIrq {
  pic: Option<KArc<KvmPic>>,
  ioapic: Option<KArc<KvmIoapic>>,
  pit: Option<KArc<KvmPit>>,
  irq_routing: KArc<RcuPointer<IrqRoutingTable>>,
  irq_ack_notifier_list: HlistHead,
  ...
}

struct KvmPic {
  master: PicState,
  slave: PicState,
  lock: SpinLock<()>,
  output: bool,                              // is master PIC asserting?
  io_dev: KvmIoDevice,                       // MMIO/IO bus registration
  notify: Mutex<KvmIrqAckNotifier>,
}

struct PicState {
  imr: u8,                                    // interrupt-mask reg
  irr: u8,                                    // interrupt-request reg
  isr: u8,                                    // in-service reg
  priority_add: u8,
  irq_base: u8,                               // ICW2 vector base
  read_reg_select: u8,
  poll: bool,
  special_mask: u8,
  init_state: u8,
  auto_eoi: u8,
  ...
}

struct KvmIoapic {
  base: u64,                                   // 0xFEC00000
  redirtbl: [Rte; IOAPIC_NUM_PINS],            // 24 entries
  ioregsel: u32,
  ioapic_id: u8,
  irr: u32,                                    // remote-IRR bitmap
  irr_delivered: u32,
  io_dev: KvmIoDevice,
  lock: SpinLock<()>,
  rtc_status: RtcStatus,                       // for RTC GSI
  ack_notifier: KvmIrqAckNotifier,
}

struct KvmPit {
  channels: [PitChannel; 3],
  io_dev: KvmIoDevice,                         // 0x40 IO-port
  speaker_dev: KvmIoDevice,                    // 0x61 IO-port (ch2)
  lock: Mutex<PitState>,
  worker_thread: KThread,
}

struct PitChannel {
  count: u32,
  latched_count: u16,
  count_latched: bool,
  mode: u8,
  bcd: bool,
  gate: bool,
  count_load_time: ktime_t,
  hrtimer: HrTimer,
}
```

`Irq::cpu_get_interrupt` flow:
1. If `kvm_apic_accept_pic_intr(vcpu)`: pic-int considered.
2. lapic-pending := `kvm_apic_has_interrupt(vcpu)`; if pending: return lapic vector.
3. extint-pending := `kvm_cpu_has_extint(vcpu)`; if pending: return `kvm_cpu_get_extint(vcpu)` → `kvm_pic_read_irq(kvm)`.
4. Else: -1.

`KvmPic::set_irq(pic, irq, level)` flow:
1. Acquire pic.lock.
2. If level == 1: pic.irr |= (1 << irq).
3. If level == 0: pic.irr &= ~(1 << irq).
4. pic_update_irq(pic) — recalculate output:
   - For each non-masked, non-in-service irr-bit: highest-priority requested.
   - If found: pic.output = 1; mark INTR pending in master.
5. If pic.output: kvm_make_request(KVM_REQ_EVENT, vcpu); kick all vCPUs.
6. Release pic.lock.

`KvmIoapic::set_irq(ioapic, irq, level)` flow:
1. Acquire ioapic.lock.
2. rte := ioapic.redirtbl[irq].
3. If rte.mask: drop.
4. If level == 1:
   - ioapic.irr |= (1 << irq) (level-triggered tracks remote-IRR).
   - ioapic_service(ioapic, irq, line_status):
     - Compose MSI message from rte (dest, vector, delivery-mode).
     - Dispatch via `kvm_irq_delivery_to_apic(rte.dest, rte.vector, ...)`.
5. If level == 0 + edge-triggered: trigger pulse-fire.
6. Release ioapic.lock.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pic_irr_no_oob` | OOB | per-IRQ index ∈ [0..15]; defense against irr/imr/isr out-of-bounds. |
| `ioapic_pin_no_oob` | OOB | per-pin index ∈ [0..23]; defense against rte[] OOB. |
| `pit_channel_no_oob` | OOB | per-PIT channel ∈ [0..2]. |
| `irq_source_refcount_no_underflow` | INVARIANT | per-(GSI, source) ref-count ≥ 0 monotonic; defense against double-deassert wrap. |
| `pic_isr_no_orphan` | INVARIANT | every isr-bit-set has corresponding prior INTA cycle; defense against stuck-isr causing IRQ-storm. |

### Layer 2: TLA+

`virt/kvm/lapic_injection.tla` (already exists from earlier Tier-3 collection) covers per-vCPU LAPIC injection.

`virt/kvm/pic_state.tla`:
- Per-IRQ state ∈ {Quiescent, Requested, InService, Acked}.
- Transitions:
  - Quiescent → Requested via set_irq(level=1) when not masked.
  - Requested → InService via INTA cycle (read_irq).
  - InService → Acked via EOI.
  - Acked → Quiescent via deassert.
- Properties:
  - `safety_at_most_one_in_service_per_pic` — i8259 spec: at most one IRQ in-service per PIC.
  - `safety_eoi_clears_isr` — every EOI clears one ISR bit.
  - `liveness_requested_eventually_serviced` — assuming guest unmasks, every Requested eventually InService.

`virt/kvm/ioapic_remote_irr.tla`:
- Per-pin level-triggered state ∈ {Idle, Asserting, Delivered, EoiPending}.
- Properties:
  - `safety_remote_irr_set_at_delivery` — remote-IRR bit set on delivery; cleared on EOI.
  - `liveness_eoi_clears_remote_irr`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `KvmPic::set_irq` post: irr-bit reflects level; output recalculated | `KvmPic::set_irq` |
| `KvmIoapic::set_irq` post: irr-bit reflects level; ioapic_service dispatched if asserted | `KvmIoapic::set_irq` |
| `KvmPit::load_count` post: per-channel hrtimer armed at correct period | `KvmPit::load_count` |
| `Irq::cpu_get_interrupt` post: returned vector matches highest-priority pending IRQ source | `Irq::cpu_get_interrupt` |
| `kvm_set_irq_routing` post: rcu-pointer updated; old table SRCU-freed after grace period | `Kvm::set_irq_routing` |

### Layer 4: Verus/Creusot functional

`Guest INTA cycle returns vector matching highest-priority unmasked + asserted IRQ on PIC at that moment`: per-INTA the returned vector V matches PIC.irq_base + index of highest-priority irr-bit AND not-masked AND not-in-service-of-equal-priority.

## Hardening

(Inherits row-1 features from `virt/kvm/kvm-core.md` § Hardening.)

KVM-IRQ-specific reinforcement:

- **Per-PIC.lock spinlock** — defense against concurrent irr/imr/isr update races.
- **Per-IOAPIC.lock spinlock** — same.
- **Per-PIT lock + worker thread serialization** — defense against PIT state racing with hrtimer fire.
- **Per-(GSI, source) ref-count** — defense against single-source-deassert clearing IRQ asserted by another source.
- **Per-irqfd workqueue dispatch atomic** — defense against eventfd signal-loss.
- **KVM_SET_GSI_ROUTING via RCU** — readers (irq-delivery path) see consistent table; updaters allocate new table + RCU-swap.
- **Per-PIC ICW1-ICW4 state machine validated** — defense against guest using PIC in undefined state.
- **Per-IOAPIC RTE write 32-bit-half-word at-a-time** — defense against partial-update reading half-formed RTE.
- **Per-PIT mode validation** at load_count — defense against undefined mode causing hrtimer-armed-with-zero-period.
- **kvm_irq_delivery_to_apic with dest-mode validation** — defense against malformed rte.dest causing OOB-cpu-walk.
- **rtc_status special tracking** — RTC GSI level-tracked specially for HPET-coexistence; defense against missed RTC-IRQ during host clocksource switch.
- **irq_ack_notifier list under SRCU** — defense against ack-notifier UAF during dispatch.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- LAPIC (covered in `virt/kvm/x86-lapic.md` Tier-3)
- KVM eventfd / irqfd (covered in `virt/kvm/eventfd.md` Tier-3)
- KVM core (covered in `kvm-core.md` Tier-3)
- VMX (covered in `x86-vmx.md` Tier-3)
- SVM (covered in `x86-svm.md` Tier-3)
- HPET emulation (out-of-tree; userspace QEMU emulates)
- Implementation code
