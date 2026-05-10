# Tier-3: arch/x86/kvm/ioapic.c — KVM in-kernel IOAPIC emulation

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/kvm-core.md
upstream-paths:
  - arch/x86/kvm/ioapic.c (~779 lines)
  - arch/x86/kvm/ioapic.h
  - arch/x86/kvm/irq.c (delivery dispatch)
-->

## Summary

Per-VM in-kernel emulated IOAPIC (Intel-82093 IOAPIC SDM-Vol3 §11.4 / Intel-MP-Spec). Up to 24 redirection-table entries (RTEs) — each maps an external IRQ source to a destination APIC + delivery-mode + vector. Per-VM IOAPIC instance allocated when userspace requests in-kernel IRQ chip (`KVM_CREATE_IRQCHIP`). MMIO at 0xFEC00000 (default) accessed via per-VM mmio-bus dispatcher. Per-RTE EOI-handled. Per-EOI broadcast across all vCPUs to clear remote-IRR. Critical for: legacy ISA IRQ + per-PCI INTx + GSI fan-in path.

This Tier-3 covers `ioapic.c` (~779 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct kvm_ioapic` | per-VM IOAPIC state | `KvmIoapic` |
| `kvm_ioapic_init()` | per-VM IOAPIC alloc | `Ioapic::init` |
| `kvm_ioapic_destroy()` | per-VM IOAPIC free | `Ioapic::destroy` |
| `ioapic_mmio_read()` / `_write()` | per-MMIO dispatch | `Ioapic::mmio_read` / `mmio_write` |
| `kvm_ioapic_set_irq()` | per-IRQ assert | `Ioapic::set_irq` |
| `kvm_ioapic_eoi_inject_work()` | per-EOI handler | `Ioapic::eoi_inject` |
| `kvm_ioapic_update_eoi()` | per-EOI clear remote-IRR | `Ioapic::update_eoi` |
| `kvm_ioapic_send_eoi()` | per-vCPU broadcast EOI | `Ioapic::send_eoi` |
| `KVM_IOAPIC_NUM_PINS` (24) | per-IOAPIC pin count | shared |
| `IOAPIC_REG_VERSION` (0x01) | per-MMIO reg | UAPI |
| `IOAPIC_REG_APIC_ID` (0x00) | per-MMIO reg | UAPI |
| `IOAPIC_REG_ARB` (0x02) | per-MMIO reg | UAPI |
| `IOAPIC_REG_REDIR_TBL` (0x10..0x3F) | per-RTE | UAPI |
| `KVM_IRQCHIP_IOAPIC` | per-VM type indicator | UAPI |

## Compatibility contract

REQ-1: Per-VM IOAPIC at default MMIO base 0xFEC00000:
- 0xFEC00000: IOREGSEL (selects register).
- 0xFEC00010: IOWIN (read/write selected register).
- 0xFEC00040: EOI register (Sandy-Bridge+).

REQ-2: Per-RTE 64-bit format:
- Bits[7:0] vector (0x10..0xFE).
- Bits[10:8] delivery_mode (000=Fixed, 001=Lowest-Priority, 010=SMI, 100=NMI, 101=INIT, 111=ExtINT).
- Bit[11] dest_mode (0=Physical, 1=Logical).
- Bit[12] delivery_status (RO; set when sent, clear on EOI).
- Bit[13] polarity (0=Active-High, 1=Active-Low).
- Bit[14] remote_IRR (RO; set on send for level-triggered; cleared on EOI).
- Bit[15] trigger_mode (0=Edge, 1=Level).
- Bit[16] mask (1=masked).
- Bits[55:32] reserved.
- Bits[63:56] destination (per-mode: physical APIC ID or logical mask).

REQ-3: Per-VM 24 RTEs (KVM_IOAPIC_NUM_PINS):
- Per-RTE indexed via IOREGSEL = 0x10 + 2*N (low 32 bits) / 0x11 + 2*N (high 32 bits).

REQ-4: kvm_ioapic_set_irq(ioapic, irq, level, line_status):
- Per-RTE indexed by irq (0..23).
- Edge-trigger: per-rising-edge issues delivery; auto-clears.
- Level-trigger: per-asserted (level=1) issues delivery if !remote_IRR; per-deasserted no-op.
- Per-mask bit set: drop delivery.
- Returns 1 if delivered; 0 if dropped.

REQ-5: Delivery to LAPICs:
- Per-(dest_mode, destination): broadcast to matching vCPU LAPICs.
- Per-Physical mode: APIC ID match.
- Per-Logical mode: bit-mask vs LAPIC LDR.
- Per-LowestPriority: lowest-TPR vCPU receives.

REQ-6: Per-EOI flow:
- Per-LAPIC EOI (write IA32_X2APIC_EOI): if vector matches IOAPIC delivery → kvm_ioapic_send_eoi(vector).
- Per-EOI: broadcast across all vCPUs; per-RTE with vector + remote_IRR set: clear remote_IRR.
- If level still asserted + RTE.mask=0: re-deliver.

REQ-7: Per-mmio bus:
- IOAPIC registered on per-VM kvm_io_bus (KVM_MMIO_BUS).
- Per-MMIO read/write dispatched via ioapic_mmio_read/write.

REQ-8: Per-VM userspace ABI:
- KVM_GET_IRQCHIP / KVM_SET_IRQCHIP: per-VM IOAPIC + PIC + LAPIC state migration.
- KVM_IRQ_LINE: userspace asserts per-pin level.

REQ-9: Per-irqfd integration:
- Per-irqfd → kvm_set_irq → kvm_ioapic_set_irq if target = IOAPIC GSI.

REQ-10: Per-NMI:
- Per-RTE delivery_mode=NMI: kvm_apic_nmi.
- Per-EXTINT (ExtINT): from PIC; routes to BSP.

## Acceptance Criteria

- [ ] AC-1: KVM_CREATE_IRQCHIP: per-VM IOAPIC allocated; MMIO bus has entry.
- [ ] AC-2: Guest reads IOAPIC IOREGSEL=0x01: returns version (0x11 + 24<<16).
- [ ] AC-3: Guest writes RTE for IRQ 0 with vector=0x30, dest=BSP, level: kvm_ioapic_set_irq routes correctly.
- [ ] AC-4: kvm_ioapic_set_irq(irq=0, level=1, line=1) for level-RTE: delivery; remote_IRR=1.
- [ ] AC-5: kvm_ioapic_send_eoi(vector=0x30): clears remote_IRR for matching RTE.
- [ ] AC-6: Per-irqfd backed by GSI 16: triggering eventfd asserts IOAPIC IRQ 16.
- [ ] AC-7: KVM_GET_IRQCHIP returns per-VM IOAPIC state including all 24 RTEs.
- [ ] AC-8: KVM_SET_IRQCHIP restores per-VM IOAPIC.
- [ ] AC-9: Logical-mode delivery: matching vCPUs in LDR-mask receive.
- [ ] AC-10: LowestPriority: lowest-TPR vCPU among destinations receives.
- [ ] AC-11: Per-RTE.mask=1: delivery suppressed.
- [ ] AC-12: kvm-unit-tests `ioapic` test passes.

## Architecture

Per-VM IOAPIC state:

```
struct KvmIoapic {
  base_address: u64,                             // 0xFEC00000 default
  ioregsel: u32,                                 // current selected register
  id: u32,                                       // APIC ID (typically 0)
  arb_id: u32,
  redirtbl: [IoapicRedirEntry; KVM_IOAPIC_NUM_PINS], // 24 RTEs
  irr: u32,                                      // pending pins (bit per IRQ)
  irr_delivered: u32,                            // delivered flags
  pending_eoi: u32,                              // EOI pending
  rtc_status: RtcStatus,                          // for RTC IRQ8 handling
  eoi_inject: AsyncCallback,                      // workqueue for level-triggered re-delivery
  rtc_dest_eoi_count: u32,
  kvm: &Kvm,
  ioapic_lock: SpinLock,
  mmio_dev: KvmIoDevice,
}
```

Per-RTE:

```
#[repr(C)]
union IoapicRedirEntry {
  raw: u64,
  fields: {
    vector: u8,                                  // [7:0]
    delivery_mode: u8,                           // [10:8]
    dest_mode: u8,                               // [11]
    delivery_status: u8,                         // [12] RO
    polarity: u8,                                // [13]
    remote_irr: u8,                              // [14] RO
    trig_mode: u8,                               // [15] 0=Edge 1=Level
    mask: u8,                                    // [16]
    reserved: u64,                               // [55:17]
    dest_id: u8,                                 // [63:56]
  },
}
const KVM_IOAPIC_NUM_PINS: usize = 24;
```

`Ioapic::init(kvm)`:
1. Allocate KvmIoapic; zero RTEs (mask=1 default).
2. ioapic.base_address = IOAPIC_DEFAULT_BASE_ADDR (0xFEC00000).
3. Register on KVM_MMIO_BUS via kvm_io_bus_register_dev.
4. kvm.arch.vioapic = ioapic.

`Ioapic::mmio_read(ioapic, addr, len, val)`:
1. offset = addr - ioapic.base_address.
2. switch offset:
   - 0x00 (IOREGSEL): *val = ioapic.ioregsel.
   - 0x10 (IOWIN): *val = read_indirect_reg(ioapic, ioapic.ioregsel).
   - 0x40 (EOI): *val = 0 (RO; or last-EOI vector).
3. Return Ok.

`Ioapic::mmio_write(ioapic, addr, len, val)`:
1. offset = addr - ioapic.base_address.
2. switch offset:
   - 0x00 (IOREGSEL): ioapic.ioregsel = val & 0xff.
   - 0x10 (IOWIN): write_indirect_reg(ioapic, ioapic.ioregsel, val).
   - 0x40 (EOI): kvm_ioapic_eoi_broadcast(ioapic, val).
3. Return Ok.

`Ioapic::write_indirect_reg(ioapic, idx, val)`:
1. switch idx:
   - 0x00 (APIC_ID): ioapic.id = val.
   - 0x01 (VERSION): RO; ignore.
   - 0x02 (ARB): ioapic.arb_id = val.
   - 0x10..0x3F (RTE): rte_idx = (idx - 0x10) >> 1; lo_high = (idx - 0x10) & 1.
     - rte = ioapic.redirtbl[rte_idx].
     - if lo_high == 0: rte.raw = (rte.raw & 0xffff_ffff_0000_0000) | val.
     - else: rte.raw = (rte.raw & 0xffff_ffff) | (val << 32).
     - ioapic.redirtbl[rte_idx] = rte.
     - If level-trigger ∧ irr & (1 << rte_idx) ∧ !rte.mask: re-deliver.

`Ioapic::set_irq(ioapic, irq, level, line_status)`:
1. ioapic_lock.lock().
2. rte = ioapic.redirtbl[irq].
3. if rte.trig_mode == LEVEL:
   - Update ioapic.irr per level.
   - If level=1 ∧ !rte.mask ∧ !rte.remote_irr: deliver; rte.remote_irr=1.
4. else (EDGE):
   - If level transitioning 0→1 ∧ !rte.mask: deliver.
5. ioapic_lock.unlock().

`Ioapic::deliver(ioapic, irq)`:
1. rte = ioapic.redirtbl[irq].
2. Build kvm_lapic_irq from rte fields.
3. kvm_irq_delivery_to_apic(kvm, NULL, &irq, NULL).

`Ioapic::eoi_broadcast(ioapic, vector)`:
1. For irq in 0..KVM_IOAPIC_NUM_PINS:
   - rte = ioapic.redirtbl[irq].
   - if rte.vector == vector ∧ rte.trig_mode == LEVEL ∧ rte.remote_irr:
     - rte.remote_irr = 0.
     - if (ioapic.irr & (1 << irq)) ∧ !rte.mask: re-deliver via async eoi_inject.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pin_count_eq_24` | INVARIANT | KVM_IOAPIC_NUM_PINS == 24. |
| `rte_idx_lt_pin_count` | INVARIANT | per-write rte_idx < KVM_IOAPIC_NUM_PINS. |
| `remote_irr_implies_level_trig` | INVARIANT | rte.remote_irr ⟹ rte.trig_mode == LEVEL. |
| `mmio_offset_in_range` | INVARIANT | per-MMIO access offset ∈ {0, 0x10, 0x40}. |
| `delivery_status_eoi_balanced` | INVARIANT | per-deliver remote_irr=1 ⟹ EOI eventually clears. |

### Layer 2: TLA+

`virt/kvm/ioapic.tla`:
- Per-IRQ assert + RTE → LAPIC delivery + EOI broadcast.
- Properties:
  - `safety_no_double_delivery_level` — per-level rte.remote_irr=1 ⟹ no re-deliver until EOI.
  - `safety_eoi_clears_remote_irr` — per-EOI matching vector ⟹ remote_irr=0.
  - `liveness_pending_irq_eventually_delivered` — per-asserted level rte.mask=0 ∧ EOI ⟹ eventually re-deliver.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Ioapic::set_irq` post: edge transition triggers delivery; level idempotent | `Ioapic::set_irq` |
| `Ioapic::write_indirect_reg` post: per-RTE field updated; re-deliver if was-pending | `Ioapic::write_indirect_reg` |
| `Ioapic::eoi_broadcast` post: per-matching-vector RTE remote_irr=0 | `Ioapic::eoi_broadcast` |
| `Ioapic::mmio_read` post: returned data per-spec | `Ioapic::mmio_read` |

### Layer 4: Verus/Creusot functional

`Per-pin set_irq + per-RTE config → LAPIC receives ESR/IRR-bit set + EOI clears remote_IRR` semantic equivalence: per-RTE matches Intel 82093 IOAPIC datasheet.

## Hardening

(Inherits row-1 features from `virt/kvm/kvm-core.md` § Hardening.)

IOAPIC-specific reinforcement:

- **Per-RTE writes serialized via ioapic_lock** — defense against concurrent RTE-write corruption.
- **Per-RTE reserved-bits not propagated to LAPIC** — defense against guest setting unspecified bits.
- **Per-MMIO offset validated** — defense against guest probing non-mapped offsets.
- **Per-VM IOAPIC restricted to KVM_CREATE_IRQCHIP'd VMs** — defense against non-irqchip VMs accidentally registering.
- **Per-RTE.mask suppresses delivery** — defense against guest spurious-IRQ floods.
- **Per-EOI broadcast scoped per-VM** — defense against cross-VM EOI leakage.
- **Per-async eoi_inject queued** — defense against per-EOI re-deliver causing IRR-recursion.
- **Per-RTE.delivery_mode validated** — defense against guest claiming unsupported mode.
- **Per-RTC IRQ8 special tracking** — defense against per-tickless guest seeing wrong RTC IRQ.
- **Per-VM ioapic memory page locked** — defense against guest MMIO probe of host-physical mem.
- **Per-VM EOI vector ↔ RTE matched only for level** — defense against edge-RTE getting EOI-clear.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- LAPIC core (covered in `x86-lapic.md` Tier-3)
- 8259 PIC (covered in `x86-pic.md` future Tier-3)
- 8254 PIT (covered in `x86-pit.md` future Tier-3)
- VFIO/IRTE direct posting (covered in `x86-posted-intr.md` Tier-3)
- Implementation code
