---
title: "Tier-3: arch/x86/kvm/lapic.c (x2APIC subset) — x2APIC mode virtualization (32-bit APIC IDs + MSR-based access + MMIO-disable)"
tags: ["tier-3", "virt-kvm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

x2APIC is the modernized LAPIC mode where APIC ID is 32-bit (vs 8-bit for legacy xAPIC), MMIO interface is disabled, and APIC access uses MSR-based RDMSR/WRMSR (MSR range 0x800..0x83F) instead of MMIO read/write to 0xFEE00000. Required for: VMs with > 255 vCPUs, IOMMU IR (Intel x2APIC requires IR active), guest using physical-mode APIC ID > 254. KVM virtualizes x2APIC by: emulating x2APIC MSR accesses; supporting 32-bit dest_id in IPIs; coordinating with IOMMU IR for MSI delivery; using VMX's "virtualize x2APIC mode" + "APIC-register virtualization" features to bypass vmexit on common APIC operations.

This Tier-3 covers x2APIC-related code in `arch/x86/kvm/lapic.c` (~300-400 lines) + VMX integration.

### Acceptance Criteria

- [ ] AC-1: Boot Linux guest with `-cpu host,x2apic=on,+x2apic`: guest CPUID 0x01:ECX[21] x2APIC = 1.
- [ ] AC-2: x2APIC enable: guest WRMSR(MSR_IA32_APICBASE, EXTD|EN) succeeds; subsequent reads return EXTD set.
- [ ] AC-3: 256-vCPU guest: APIC IDs 0..255; x2APIC required.
- [ ] AC-4: ICR write IPI: vCPU0 WRMSR(0x830, ICR_value); vCPU1 receives IRQ.
- [ ] AC-5: Cluster mode IPI: dest_id = (cluster_id << 16) | vCPU_bitmap; per-cluster broadcast works.
- [ ] AC-6: SELF_IPI: WRMSR(0x83F, vector); self-IRQ delivered immediately.
- [ ] AC-7: Live migration: x2APIC state preserved across migrate.
- [ ] AC-8: IR coupling: host-IOMMU-IR disabled → guest cannot enable x2APIC (returns -EINVAL on KVM_CAP).
- [ ] AC-9: kvm-unit-tests `x2apic` test passes.
- [ ] AC-10: Performance: x2APIC ICR write < 100 cycles when virtualize-x2apic-mode active (vs 2000+ cycles for vmexit).

### Architecture

`Lapic` per-vCPU (extends from x86-lapic Tier-3):

```
struct KvmLapic {
  ...
  apic_base: u64,                              // MSR_IA32_APICBASE
  regs: KBox<[u8; APIC_REG_SIZE]>,             // 4KiB register page
  x2apic_id: u32,                               // cached 32-bit ID
  pending_events: u64,
}
```

`Lapic::is_x2apic_mode(apic)`:
- Return apic.apic_base & MSR_IA32_APICBASE_EXTD != 0.

`Lapic::x2apic_msr_write(vcpu, msr_idx, data)`:
1. reg_offset := (msr_idx - 0x800) * 0x10.
2. Switch on reg_offset:
   - APIC_ID (0x20): read-only via x2APIC; -EINVAL.
   - APIC_LVR (0x30): read-only.
   - APIC_TASKPRI (0x80): apic.regs[reg_offset] = data; recompute interrupt priority.
   - APIC_EOI (0xB0): clear ISR top; trigger pending-irq update.
   - APIC_LDR (0xD0): read-only via x2APIC.
   - APIC_DFR (0xE0): not present in x2APIC; -EINVAL.
   - APIC_ICR (0x300): write triggers IPI:
     - vector := data & 0xFF.
     - delivery_mode := (data >> 8) & 0x7.
     - dest_id := (data >> 32).
     - kvm_apic_match_dest + kvm_irq_delivery_to_apic.
   - APIC_LVT* (0x320..0x370): per-LVT-entry write.
   - APIC_TMICT (0x380): timer initial-count.
   - APIC_TDCR (0x3E0): timer divide.
   - APIC_SELF_IPI (0x3F0; MSR 0x83F): vector := data & 0xFF; self-IRQ.

`VcpuVmx::set_msr_bitmap_x2apic(vmx, &mode)` flow:
1. For each MSR in 0x800..0x83F:
   - Determine intercept policy:
     - APIC_ID, APIC_LVR, etc. (R/O): READ-passthrough.
     - APIC_TASKPRI, APIC_EOI, APIC_TMICT (state-changing): write-intercept.
     - APIC_ICR, APIC_SELF_IPI: write-intercept (must emulate IPI).
2. Update vmcs.MSR-bitmap.

`Lapic::set_base(vcpu, value, host_initiated)` flow:
1. Old base := apic.apic_base.
2. Validate: if EXTD set: EN must also be set.
3. Validate: cannot directly EXTD → !EXTD without first !EN.
4. apic.apic_base = value.
5. If transitioned to x2APIC: x2apic_id = vcpu.vcpu_id; update LDR; vmx_set_msr_bitmap_x2apic.
6. If transitioned to xAPIC: similar reverse.

### Out of Scope

- xAPIC mode (covered in `x86-lapic.md` Tier-3)
- VMX vendor (covered in `x86-vmx.md` Tier-3)
- IOMMU IR (covered in `drivers/iommu/intel-irq-remap.md` Tier-3)
- Hyper-V SynIC (different IRQ controller; covered in `x86-hyperv.md` Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `kvm_apic_set_x2apic_id(apic, id)` | per-vCPU set 32-bit APIC ID | `Lapic::set_x2apic_id` |
| `kvm_x2apic_id(apic)` | per-vCPU get 32-bit ID | `Lapic::x2apic_id` |
| `kvm_apic_state_fixup(vcpu, ...)` | apic-state migration fixup | `Lapic::state_fixup` |
| `kvm_x2apic_msr_write(...)` | x2APIC-MSR write entry | `Lapic::x2apic_msr_write` |
| `kvm_x2apic_msr_read(...)` | x2APIC-MSR read entry | `Lapic::x2apic_msr_read` |
| `apic_x2apic_mode(apic)` | per-vCPU x2APIC-mode query | `Lapic::is_x2apic_mode` |
| `kvm_set_x2apic_id(vcpu, id)` | KVM_SET_LAPIC ioctl x2APIC update | `Lapic::set_x2apic_id_ioctl` |
| `kvm_apic_set_base(vcpu, value, host_initiated)` | MSR_IA32_APICBASE write handler | `Lapic::set_base` |
| `kvm_lapic_set_x2apic_register(...)` | x2APIC register-write helper | `Lapic::set_x2apic_register` |
| `kvm_lapic_get_x2apic_register(...)` | x2APIC register-read helper | `Lapic::get_x2apic_register` |
| `vmx_set_msr_bitmap_x2apic_*(...)` (vmx.c) | per-x2APIC-mode MSR-bitmap config | `VcpuVmx::set_msr_bitmap_x2apic` |
| `kvm_apic_match_dest(...)` extended for 32-bit | per-IPI dest-match | `Lapic::match_dest` |

### compatibility contract

REQ-1: APIC MSR space (x2APIC mode):
- 0x800..0x83F: 64 register addresses.
- Each MSR maps to legacy xAPIC register (e.g., MSR 0x802 = APIC ID register).
- Some xAPIC registers absent in x2APIC (DFR, ICR2 merged into ICR).

REQ-2: x2APIC mode entry:
- Guest sets MSR_IA32_APICBASE bit 10 (EXTD = x2APIC enable).
- Validate: bit 11 (xAPIC enable) also set.
- Cannot disable EXTD without disabling xAPIC entirely (no EXTD→xAPIC direct transition).

REQ-3: Per-vCPU x2APIC state:
- vcpu.arch.apic.regs (4KiB; same as xAPIC; x2APIC reads/writes use this).
- vcpu.arch.apic.x2apic_id (32-bit; cached for fast lookup).
- VMCS APIC-access-page disabled in x2APIC mode (VMX virtualize-x2apic-mode bit).

REQ-4: x2APIC MSR write semantics:
1. Validate MSR index in 0x800..0x83F.
2. Some registers read-only in x2APIC (e.g., LDR, APR).
3. ICR (Inter-Processor-Interrupt Command Register) write triggers IPI emit.
4. EOI write clears ISR top.

REQ-5: x2APIC ICR (Interrupt Command Register):
- 64-bit: combined dest_id (high 32) + vector + delivery-mode + dest-mode + level + trigger-mode + shorthand (low 32).
- Atomic write triggers IPI emission.

REQ-6: 32-bit dest_id IPI delivery:
- Physical mode: dest_id == apic_id of target vCPU.
- Logical mode: dest_id is bitmap; per-vCPU LDR (Logical Destination Register) bit-match.
- Cluster mode (x2APIC default): dest_id high 16 = cluster ID; low 16 = bitmap within cluster.

REQ-7: Per-vCPU x2APIC ID:
- Default = vcpu_id (guest-visible).
- Configurable via KVM_SET_LAPIC ioctl.
- Up to 2^32 unique IDs (vs 2^8 in xAPIC).

REQ-8: VMX virtualize-x2APIC-mode + APIC-register-virtualization:
- vmcs.secondary_exec_control bit 4 = virtualize-x2apic-mode.
- vmcs.secondary_exec_control bit 8 = APIC-register-virtualization.
- When set: guest x2APIC accesses to most registers handled by CPU directly without vmexit.
- Exits only for state-changing operations (ICR write, etc.).

REQ-9: Per-vCPU MSR-bitmap configuration:
- For x2APIC-active vCPU: MSR-bitmap masks x2APIC MSR range (0x800..0x83F).
- KVM enables shadowing: read-passthrough for harmless registers; intercept for ICR/EOI/SELF_IPI.

REQ-10: x2APIC + IOMMU IR coupling:
- x2APIC requires IR (Intel-spec); without IR active, IPI delivery undefined.
- KVM verifies host-IOMMU IR enabled before allowing guest x2APIC.

REQ-11: APIC ID >= 256 requires x2APIC:
- Cluster ID + per-cluster ID together encode > 8 bits.
- Required for VMs with > 255 vCPUs (cloud-hyperscale).

REQ-12: SELF_IPI MSR (0x83F):
- Direct self-targeted IPI (no ICR programming needed).
- Lower latency than ICR-based self-IPI.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `x2apic_msr_idx_bounded` | OOB | per-MSR idx in 0x800..0x83F; defense against OOB register access. |
| `apic_id_bits_validated` | INVARIANT | x2apic_id 32-bit; xAPIC mode caps at 8-bit. |
| `apic_base_transitions_valid` | INVARIANT | EXTD requires EN; transitions follow spec. |
| `ir_coupling_enforced` | INVARIANT | x2APIC enable gated on host-IOMMU-IR; defense against IPI mis-delivery. |
| `self_ipi_local_only` | INVARIANT | SELF_IPI vector ≤ 255; defense against vector-overflow. |

### Layer 2: TLA+

`virt/kvm/x2apic_mode_transition.tla`:
- States: xAPICOff, xAPICOn, x2APICOn.
- Transitions per MSR_IA32_APICBASE WRMSR.
- Properties:
  - `safety_no_direct_x2_to_x` — x2APICOn → xAPICOn requires xAPICOff intermediate.
  - `safety_extd_implies_en` — x2APICOn implies APICBASE.EN.

`virt/kvm/x2apic_ipi.tla`:
- Per-IPI dispatch with 32-bit dest_id.
- Properties:
  - `safety_dest_match_correct` — IPI delivered to matching dest in physical/logical/cluster mode.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Lapic::set_base` post: apic.apic_base updated; mode-transition side-effects applied | `Lapic::set_base` |
| `Lapic::x2apic_msr_write` post: per-register effect applied; ICR triggers IPI | `Lapic::x2apic_msr_write` |
| `VcpuVmx::set_msr_bitmap_x2apic` post: MSR-bitmap reflects x2APIC-mode pass-through policy | `VcpuVmx::set_msr_bitmap_x2apic` |
| Per-vCPU x2apic_id consistent with apic.regs APIC_ID register | invariants on x2apic_id update |

### Layer 4: Verus/Creusot functional

`x2APIC mode: guest WRMSR(0x830, ICR) → IPI delivered to dest_id (32-bit) target vCPU` semantic equivalence: per-IPI delivery follows Intel-SDM cluster/physical/logical-mode dispatch.

### hardening

(Inherits row-1 features from `virt/kvm/x86-lapic.md` § Hardening.)

x2APIC-specific reinforcement:

- **APICBASE.EXTD requires .EN** — defense against guest enabling x2APIC without basic APIC enabled.
- **No direct x2 → xAPIC transition** — defense against state-machine bypass.
- **x2APIC requires IR** — defense against guest x2APIC IPI mis-delivered to wrong CPU.
- **Per-MSR R/W intercept policy** — defense against guest direct-write to read-only x2APIC registers.
- **Per-VCPU MSR-bitmap update under apic.lock** — defense against torn pass-through state during mode transition.
- **APIC ID > 255 only allowed in x2APIC mode** — defense against xAPIC overflow.
- **Cluster ID validated against existing vCPUs** — defense against IPI to non-existent cluster.
- **Per-vmexit ICR-write atomic** — defense against partial-IPI on torn ICR write.
- **VMX virtualize-x2apic-mode validated** — defense against guest enabling x2APIC on host without VMX support.
- **Per-VM KVM_CAP_X2APIC_API gating** — defense against legacy QEMU expecting xAPIC.
- **Live-migrate APIC state version** — defense against post-migrate mode-mismatch.

