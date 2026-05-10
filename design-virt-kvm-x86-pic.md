---
title: "Tier-3: arch/x86/kvm/i8259.c — KVM in-kernel 8259 PIC (Programmable Interrupt Controller) emulation"
tags: ["tier-3", "virt-kvm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Per-VM dual cascaded Intel-8259A PIC emulation (master @ I/O 0x20-0x21 + slave @ I/O 0xA0-0xA1 cascaded through master IRQ2). 16 IRQ lines (IRQ0..IRQ15). Per-PIC: IRR (Interrupt-Request-Register), IMR (Interrupt-Mask-Register), ISR (In-Service-Register), priority + ICW1-4 init sequence + OCW1-3 operational commands + ELCR (Edge/Level Control Register at I/O 0x4D0/0x4D1). Coexists with IOAPIC: legacy software (BIOS, DOS) reaches 8259; modern OS uses IOAPIC. Per-VM PIC instance allocated alongside IOAPIC on KVM_CREATE_IRQCHIP. Critical for: legacy/early-boot/BIOS guest software.

This Tier-3 covers `i8259.c` (~655 lines).

### Acceptance Criteria

- [ ] AC-1: KVM_CREATE_IRQCHIP: per-VM PIC pair allocated; ports registered.
- [ ] AC-2: Guest writes 0x11 to 0x20: ICW1 init sequence started; expects ICW2-4.
- [ ] AC-3: Guest writes ICW2-master 0x30: master base-vector=0x30.
- [ ] AC-4: Guest writes ICW3-master 0x04: cascade-bit-2 set.
- [ ] AC-5: kvm_pic_set_irq(irq=0, level=1): IRR-bit 0 set; INT raised.
- [ ] AC-6: INTA cycle: vector = master.icw2 + 0 = 0x30; ISR-bit 0 set.
- [ ] AC-7: OCW2 EOI to 0x20 (0x20 specific): ISR-bit 0 cleared.
- [ ] AC-8: Slave IRQ8: cascade through master IRQ2; INTA returns slave.icw2 + 0.
- [ ] AC-9: ELCR write 0x4D0 = 0x10: IRQ4 set to level-trigger.
- [ ] AC-10: Per-IMR mask: kvm_pic_set_irq blocks INT raise.
- [ ] AC-11: KVM_GET_IRQCHIP(.chip_id=0/1): returns per-PIC state.
- [ ] AC-12: kvm-unit-tests `pic` test passes.

### Architecture

Per-PIC instance:

```
struct KvmKpicState {
  last_irr: u8,                                  // edge detection
  irr: u8,                                       // pending
  imr: u8,                                       // mask
  isr: u8,                                       // in-service
  priority_add: u8,                              // rotate-priority
  irq_base: u8,                                  // ICW2
  read_reg_select: u8,                           // OCW3
  poll: u8,
  special_mask: u8,                              // OCW3
  init_state: u8,                                // ICW1..4 sequence
  auto_eoi: u8,                                  // ICW4.AEOI
  rotate_on_auto_eoi: u8,
  special_fully_nested_mode: u8,                 // ICW4.SFNM
  init4: u8,                                     // ICW1.IC4 = ICW4-needed
  elcr: u8,                                      // edge/level per-IRQ
  elcr_mask: u8,                                 // valid ELCR bits
  isr_ack: u8,                                   // legacy IRQ8 ack tracking
  pics_state: &mut KvmPic,
}
```

Per-VM PIC pair:

```
struct KvmPic {
  pics: [KvmKpicState; 2],                       // [0]=master [1]=slave
  pic_lock: SpinLock,
  output: u32,                                   // INT-line state (0 / 1)
  pending_acks: u32,
  kvm: &Kvm,
  irqchip_in_kernel: bool,
  dev_master: KvmIoDevice,                       // ports 0x20..0x21
  dev_slave: KvmIoDevice,                        // ports 0xA0..0xA1
  dev_elcr: KvmIoDevice,                         // ports 0x4D0..0x4D1
  wakeup_needed: bool,
}
```

`Pic::init(kvm)`:
1. Allocate KvmPic; pic_init each.
2. kvm_io_bus_register_dev(KVM_PIO_BUS, 0x20, 2, &dev_master).
3. kvm_io_bus_register_dev(KVM_PIO_BUS, 0xA0, 2, &dev_slave).
4. kvm_io_bus_register_dev(KVM_PIO_BUS, 0x4D0, 2, &dev_elcr).
5. kvm.arch.vpic = pic.

`Pic::set_irq(e, kvm, irq_source_id, irq, level, line_status)`:
1. pic_lock.lock().
2. ret = pic_set_irq1(pic.pics[irq >> 3], irq & 7, level).
3. pic_update_irq(pic).
4. pic_lock.unlock().
5. Return ret.

`Pic::set_irq1(s, irq, level)`:
1. mask = 1 << irq.
2. if s.elcr & mask (level-trigger):
   - if level: ret = !(s.irr & mask); s.irr |= mask; s.last_irr |= mask.
   - else: s.irr &= !mask; s.last_irr &= !mask.
3. else (edge):
   - if level && !(s.last_irr & mask): ret = !(s.irr & mask); s.irr |= mask.
   - if !level: s.last_irr &= !mask.
4. Return (s.imr & mask) ? -1 : ret.

`Pic::read_irq(kvm) -> u8`:
1. pic_lock.lock().
2. master = &pic.pics[0]; irq = pic_get_irq(master).
3. if irq == 2 (cascade-to-slave): slave_irq = pic_get_irq(&pic.pics[1]); intack(slave); intack(master); ret = slave.irq_base + slave_irq.
4. else: intack(master); ret = master.irq_base + irq.
5. pic_lock.unlock().
6. Return ret.

`Pic::intack(s, irq)`:
1. s.isr |= 1 << irq.
2. if s.auto_eoi: pic_clear_isr(s, irq).
3. else: s.irr &= !(1 << irq) [unless ELCR level].

`Pic::ioport_write(s, addr, val)`:
1. switch (addr & 1):
   - 0 (CMD):
     - if val & 0x10: ICW1 init: reset state; init_state = 1.
     - elif (val >> 3) & 1: OCW3: special_mask + read_reg_select.
     - else: OCW2: EOI / rotate / specific-EOI logic.
   - 1 (DATA):
     - switch s.init_state:
       - 0: OCW1 IMR write.
       - 1: ICW2 base vector.
       - 2: ICW3 cascade.
       - 3: ICW4.
       - 4: complete; init_state = 0.

`Pic::elcr_write(s, addr, val)`:
1. s.elcr = val & s.elcr_mask.

`Pic::update_irq(pic)`:
1. m = &pic.pics[0]; sl = &pic.pics[1].
2. if pic_get_irq(sl) >= 0: m.irr |= (1 << 2).
3. if pic_get_irq(m) >= 0: pic.output = 1; kvm_apic_set_irq(BSP, &lapic_irq{vector=ext_INTA, ext=true}).
4. else: pic.output = 0.

### Out of Scope

- IOAPIC (covered in `x86-ioapic.md` Tier-3)
- LAPIC (covered in `x86-lapic.md` Tier-3)
- 8254 PIT (covered in `x86-pit.md` future Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct kvm_pic` | per-VM PIC pair (master+slave) | `KvmPic` |
| `struct kvm_kpic_state` | per-PIC state (one of two) | `KvmKpicState` |
| `kvm_pic_init()` | per-VM PIC alloc | `Pic::init` |
| `kvm_pic_destroy()` | per-VM PIC free | `Pic::destroy` |
| `kvm_pic_set_irq()` | per-IRQ assert | `Pic::set_irq` |
| `kvm_pic_read_irq()` | per-vCPU INTA-cycle vector read | `Pic::read_irq` |
| `kvm_pic_clear_isr_ack()` | per-EOI ISR clear | `Pic::clear_isr_ack` |
| `kvm_pic_update_irq()` | per-INT-line raise/lower | `Pic::update_irq` |
| `pic_lock()` / `pic_unlock()` | per-VM pic_lock | `Pic::lock` / `unlock` |
| `pic_ioport_read()` / `_write()` | per-I/O dispatch | `Pic::ioport_read` / `ioport_write` |
| `elcr_ioport_read()` / `_write()` | per-ELCR dispatch | `Pic::elcr_read` / `elcr_write` |
| `picdev_master_ops` / `picdev_slave_ops` / `picdev_elcr_ops` | per-I/O bus device | `Pic::Master` / `Slave` / `Elcr` |
| `KVM_IRQCHIP_PIC_MASTER` / `KVM_IRQCHIP_PIC_SLAVE` | per-VM type | UAPI |

### compatibility contract

REQ-1: I/O port layout:
- Master PIC: 0x20 (CMD), 0x21 (DATA).
- Slave PIC: 0xA0 (CMD), 0xA1 (DATA).
- ELCR: 0x4D0 (master), 0x4D1 (slave).

REQ-2: Per-PIC state:
- IRR (Interrupt-Request-Register): pending IRQs.
- IMR (Interrupt-Mask-Register): masked-out IRQs.
- ISR (In-Service-Register): currently-servicing IRQs.
- ICW1: init mode + ICW4-needed flag + LTM (level-trigger-mode-default).
- ICW2: per-PIC base vector (master typically 0x08; modern Linux 0x30).
- ICW3: master cascade-bitmap (bit-2 for slave); slave cascade-IRQ-num (2).
- ICW4: 8086 mode + AEOI + buffered + nested.
- OCW1: write to data-port = IMR.
- OCW2: EOI / specific-EOI / rotate-priority / etc.
- OCW3: special-mask-mode + read-IRR/ISR.

REQ-3: ELCR (Edge/Level-Control-Register):
- 1 byte each for master + slave.
- Per-bit: 0 = edge-triggered, 1 = level-triggered.
- Per-IRQ override (ICW1.LTM).

REQ-4: Per-VM dual-PIC cascading:
- Master IRQ2 = slave INT-line.
- Slave INT raised → master IRR-bit-2 set → master delivers via INTA.
- INTA cycle: master responds vector for IRQ0/1/3..7; slave responds for IRQ2 (8..15 with slave base = 0x70 typically).

REQ-5: kvm_pic_set_irq(pic, irq, level):
- Per-irq < 8: target master.
- Per-irq ≥ 8: target slave.
- Per-edge-trig: rising-edge sets IRR.
- Per-level-trig (per-ELCR): raised → IRR = level; lowered → IRR cleared.
- Per-IMR-masked: no INT-line raise.

REQ-6: kvm_pic_read_irq(kvm) (INTA):
- Pick highest-priority unmasked-IRR-set IRQ.
- Set ISR-bit; clear IRR-bit (unless ELCR-level).
- Return vector = ICW2-base + irq.
- If slave: cascade through master IRQ2; slave INTA returns vector.

REQ-7: kvm_pic_clear_isr_ack:
- OCW2 specific-EOI: clear specified ISR-bit.
- OCW2 non-specific-EOI: clear highest-priority ISR-bit.
- AEOI mode: ISR auto-clears at INTA-cycle-2.

REQ-8: Per-INT line update:
- After IRR/IMR/ISR change: pic_update_irq(s):
  - if (s.irr & ~s.imr): raise s.int_output.
  - else: lower s.int_output.
- Master.int_output → kvm_apic_set_irq via lapic.

REQ-9: Per-MMIO/IO bus:
- KVM_PIO_BUS device: ports 0x20, 0x21, 0xA0, 0xA1, 0x4D0, 0x4D1.
- Per-port read/write dispatched via kvm_io_bus.

REQ-10: Per-VM userspace ABI:
- KVM_GET_IRQCHIP(.chip_id=0/1): per-PIC state.
- KVM_SET_IRQCHIP: restore per-PIC.

REQ-11: Per-VM SPLIT-IRQ-CHIP:
- KVM_CREATE_IRQCHIP: PIC + IOAPIC + LAPIC in-kernel.
- KVM_CAP_SPLIT_IRQCHIP: only LAPIC in-kernel; PIC + IOAPIC userspace (e.g. QEMU).

REQ-12: Per-PIC reset:
- ICW1 (bit-4 set): triggers init sequence; reset IMR/IRR/ISR.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `irq_lt_16` | INVARIANT | per-set_irq irq < 16. |
| `pic_lock_held_during_state` | INVARIANT | per-state-mutation pic_lock held. |
| `cascade_through_irq2` | INVARIANT | slave INT raised ⟹ master.irr bit 2 set. |
| `init_state_in_range` | INVARIANT | per-pic init_state ∈ [0, 4]. |
| `eoi_clears_isr_in_service` | INVARIANT | per-EOI: cleared ISR bit was set. |

### Layer 2: TLA+

`virt/kvm/pic.tla`:
- Per-PIC IRQ assert + INTA + EOI sequence.
- Properties:
  - `safety_isr_set_only_after_inta` — per-ISR-bit set ⟹ INTA happened.
  - `safety_irr_clears_on_inta_unless_level` — per-INTA: edge-trigger IRR cleared.
  - `safety_cascade_path_consistent` — per-slave-INT ⟹ master IRR.bit2 + INTA → slave-base + slave_irq.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Pic::set_irq1` post: per-edge rising sets IRR; per-level updates IRR=level | `Pic::set_irq1` |
| `Pic::read_irq` post: ISR-bit set; per-non-level IRR cleared; vector returned | `Pic::read_irq` |
| `Pic::ioport_write` post: per-CMD/DATA dispatched per-ICW/OCW state machine | `Pic::ioport_write` |
| `Pic::update_irq` post: master.output reflects pic_get_irq(master) | `Pic::update_irq` |

### Layer 4: Verus/Creusot functional

`Per-IRQ assert + INTA → vector = ICW2-base + irq + cascade through slave-via-master-IRQ2` semantic equivalence: per-PIC matches Intel 8259A datasheet.

### hardening

(Inherits row-1 features from `virt/kvm/kvm-core.md` § Hardening.)

PIC-specific reinforcement:

- **Per-VM pic_lock serializes state mutation** — defense against concurrent INT-update corrupting IRR/ISR/IMR.
- **Per-init_state machine bounded** — defense against malformed ICW sequence stuck in unknown state.
- **Per-IRQ < 16 enforced** — defense against guest claiming non-existent IRQ.
- **Per-ELCR per-IRQ-bit-mask** — defense against guest setting reserved ELCR bits.
- **Per-AEOI auto-clear ISR atomically** — defense against per-EOI race losing IRQ.
- **Per-slave cascade through master IRQ2** — defense against bypassing master in INTA cycle.
- **Per-VM PIC restricted to KVM_CREATE_IRQCHIP** — defense against split-irqchip VMs accidentally getting in-kernel PIC.
- **Per-IOPort range only registered when irqchip in-kernel** — defense against userspace probing host-PIC.
- **Per-IOPort read returns last-written-CMD** — defense against guest leaking host-state via uninitialized read.
- **Per-PIC state migrated via KVM_GET_IRQCHIP** — defense against post-migrate guest seeing wrong PIC state.
- **Per-output → BSP only** — defense against multi-vCPU PIC INT going to wrong CPU.

