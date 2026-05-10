---
title: "Tier-3: arch/x86/kvm/vmx/posted_intr.c — Intel VT-d Posted Interrupts (PI)"
tags: ["tier-3", "virt-kvm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Intel VT-d Posted Interrupts (Sky-Lake-EP+; SDM-Vol3 §29.6 / VT-d-Spec §5.2.3) deliver per-(device, vCPU) interrupts directly into running guest without VMexit by writing to the per-vCPU **Posted-Interrupt Descriptor** (PID) and signaling a per-vCPU **Posted-Interrupt Notification** (PIN) vector. KVM virtualizes per-vCPU PID alloc + per-IRQ remapping (IRTE.posted=1, PID-pointer, vector) + per-vCPU sleep/wake via `pi_wakeup_list`. Used by: VFIO assigned-device IRQ + per-vCPU IPI fast-path. Critical for: low-latency VFIO device-assignment + virtual-machine IPI bypass.

This Tier-3 covers `posted_intr.c` (319 lines).

### Acceptance Criteria

- [ ] AC-1: Boot KVM with VFIO-assigned NIC + posted-intr support: `dmesg` shows "VT-d: Posted Interrupts enabled".
- [ ] AC-2: VFIO IRQ delivered to running guest: no VMexit (verified via tracepoints).
- [ ] AC-3: VFIO IRQ delivered to halted guest: pi_wakeup_handler resumes vCPU.
- [ ] AC-4: Per-vCPU pi_pre_block: NV ← 0xF1; NDST ← host-CPU.
- [ ] AC-5: Per-vCPU pi_post_block: NV ← 0xF2; NDST ← guest-target.
- [ ] AC-6: Per-vCPU SN flicker: vCPU paused → SN=1; resumed → SN=0.
- [ ] AC-7: Per-IRQ IRTE update on vcpu-affinity change: IRTE.pda updated atomically.
- [ ] AC-8: Multi-vCPU: per-vCPU PID independent; cross-vCPU PI doesn't collide.
- [ ] AC-9: kvm-unit-tests `posted_intr` test passes.
- [ ] AC-10: Per-IOMMU without posting-cap: graceful fallback to remapped (non-posted) mode.

### Architecture

Per-vCPU Posted-Interrupt-Descriptor:

```
#[repr(C, align(64))]
struct PiDesc {
  pir: [u64; 4],                                 // 256-bit pending-vector bitmap
  control: u64,                                  // ON | SN | reserved | NV (8b) | reserved | NDST (32b)
  reserved: [u64; 3],                            // pad to 64 bytes
}
const POSTED_INTR_VECTOR: u8 = 0xF2;
const POSTED_INTR_WAKEUP_VECTOR: u8 = 0xF1;
const PI_ON: u64 = 1 << 0;
const PI_SN: u64 = 1 << 1;
```

Per-vCPU vmx-trampoline (vt) state:

```
struct VmxVcpu {
  ...
  pi_desc: Box<PiDesc>,                          // page-aligned + cache-line aligned
  pi_wakeup_list: ListLink,                      // queued on per-CPU wakeup-list
}
```

Per-host CPU wakeup list:

```
PerCpu<Mutex<LinkedList<&VmxVcpu>>> WAKEUP_LIST;
```

`Pi::pre_block(vcpu)`:
1. If !vmx_needs_pi_wakeup(vcpu): return.
2. host_cpu = smp_processor_id().
3. PiDesc: NV ← POSTED_INTR_WAKEUP_VECTOR; NDST ← per-CPU APIC ID; SN ← 0.
4. If PiDesc.PIR contains any pending: kvm_vcpu_kick(vcpu); return (don't sleep).
5. WAKEUP_LIST[host_cpu].lock(); list_add(&vt.pi_wakeup_list, &list); unlock.

`Pi::post_block(vcpu)`:
1. host_cpu = smp_processor_id().
2. WAKEUP_LIST[host_cpu].lock(); list_del(&vt.pi_wakeup_list); unlock.
3. PiDesc: NV ← POSTED_INTR_VECTOR; NDST ← per-vCPU host-target APIC.

`Pi::wakeup_handler()` (per-host ISR):
1. host_cpu = smp_processor_id().
2. WAKEUP_LIST[host_cpu].lock().
3. For each vt in list: if PiDesc.ON: kvm_vcpu_kick(vt.vcpu).
4. WAKEUP_LIST[host_cpu].unlock().

`Pi::update_irte(irqfd, kvm, host_irq, guest_irq, set)`:
1. If !set: clear IRTE.posted = 0; restore remapped-mode.
2. Else:
   - target_vcpu = kvm_irq_routing_lookup(kvm, guest_irq).
   - Build IRTE.posted = 1; IRTE.pda = target_vcpu.pi_desc.phys; IRTE.vector = guest_vector.
   - irq_set_vcpu_affinity(host_irq, &irq_data).

`Pi::sn_set(vcpu)` (when paused):
1. PiDesc.control |= PI_SN (atomic).

`Pi::sn_clear(vcpu)` (when resumed):
1. PiDesc.control &= !PI_SN (atomic).

### Out of Scope

- Intel VT-d IOMMU core (covered separately)
- VFIO (covered separately)
- KVM core (covered in `kvm-core.md` Tier-3)
- KVM lapic (covered in `x86-lapic.md` Tier-3)
- AMD AVIC (covered in `x86-svm-avic.md` Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct pi_desc` | per-vCPU Posted-Intr Descriptor (64-byte) | `PiDesc` |
| `pi_desc.pir[8]` | 256-bit pending-vector bitmap | `PiDesc::pir` |
| `pi_desc.control` | ON / SN / NDST / NV fields | `PiDesc::control` |
| `pi_desc_get_nv()` | per-vCPU notification-vector | `PiDesc::nv` |
| `pi_desc_set_nv()` | per-vCPU set NV | `PiDesc::set_nv` |
| `POSTED_INTR_VECTOR` (0xF2) | per-vCPU notification IRQ | UAPI |
| `POSTED_INTR_WAKEUP_VECTOR` (0xF1) | per-host wakeup IRQ for sleeping vCPUs | UAPI |
| `pi_pre_block(vcpu)` / `pi_post_block(vcpu)` | per-vCPU sleep transition | `Pi::pre_block` / `post_block` |
| `pi_wakeup_handler()` | per-host PI wakeup ISR | `Pi::wakeup_handler` |
| `pi_wakeup_list` | per-CPU sleeping-vCPU list | `Pi::wakeup_list` |
| `vmx_pi_update_irte()` | per-IRQ IRTE update | `Pi::update_irte` |
| `intel_irq_remapping_cap(IRQ_POSTING_CAP)` | per-IOMMU posting cap | `IommuOps::posting_cap` |
| `irq_set_vcpu_affinity()` | per-IRQ vcpu-affinity setter | `IrqRouting::set_vcpu_affinity` |

### compatibility contract

REQ-1: Per-vCPU PID structure (64 bytes, cache-line aligned):
- Bytes [0..32]: PIR (Posted-Interrupt Requests) — 256-bit bitmap of pending vectors.
- Bytes [32..40]: control field:
  - Bit[0] ON (Outstanding-Notification).
  - Bit[1] SN (Suppress-Notification).
  - Bits[7..0] of byte[33] NV (Notification-Vector).
  - Bytes[34..36] reserved.
  - Bytes[36..40] NDST (Notification-Destination — per-CPU APIC ID, x2APIC ext = 32 bits).
- Bytes [40..64] reserved.

REQ-2: VMX activation:
- vmcs.pin-based-controls bit [PROCESS_POSTED_INTR] = 1.
- vmcs.posted-interrupt-notification-vector = NV (typically POSTED_INTR_VECTOR 0xF2).
- vmcs.posted-interrupt-descriptor-address = guest-physical PA of pi_desc.

REQ-3: HW posting flow (per-IRQ from VT-d remapping):
- IOMMU IRTE.posted = 1 + PDA-H/L = PID phys-addr + vector = guest-vector.
- HW: atomically set PIR[vector]; test-and-set ON.
- If !ON-was-already-set: HW issues per-NDST CPU IPI with NV vector.
- If running vCPU NV = POSTED_INTR_VECTOR: HW handles ack-without-VMexit, transfers PIR → guest-VIRR, clears ON.
- If !running: per-CPU receives POSTED_INTR_WAKEUP_VECTOR; pi_wakeup_handler() invoked.

REQ-4: Per-vCPU sleep transition (pi_pre_block):
- vCPU about to halt (HLT/MWAIT/cpuidle):
  - PiDesc.NV = POSTED_INTR_WAKEUP_VECTOR.
  - PiDesc.NDST = current host-CPU APIC ID.
  - Test SN; if PIR-already-pending: don't sleep (return early).
  - list_add(&vt.pi_wakeup_list, &per_cpu(wakeup_list, host_cpu)).

REQ-5: Per-vCPU wake transition (pi_post_block):
- vCPU resumed:
  - list_del(&vt.pi_wakeup_list).
  - PiDesc.NV = POSTED_INTR_VECTOR (back to in-guest notification).
  - PiDesc.NDST = guest-target APIC ID (current host-CPU).

REQ-6: Per-host PI-wakeup ISR (pi_wakeup_handler):
- IRQ vector = POSTED_INTR_WAKEUP_VECTOR (0xF1).
- Walks per-CPU pi_wakeup_list; per-vt with ON-set: kvm_vcpu_kick(vcpu).
- Per-walk holds per-CPU spinlock.

REQ-7: Per-IRQ IRTE update (vmx_pi_update_irte):
- Per-VFIO/PCI device: when guest assigns IRQ to vCPU:
  - Lookup target vCPU from kvm_irq_routing.
  - intel_irq_remapping IRTE.posted = 1; IRTE.pda = target.pi_desc.phys.
  - IRTE.vector = guest-vector.
- Per-mask-change: re-update IRTE.

REQ-8: Per-vCPU PI-cap requirements:
- VMX: PROCESS_POSTED_INTR pin-based control supported.
- VT-d: IOMMU posting-cap.
- Per-IRQ requires posted-mode IRTE.

REQ-9: Per-vCPU SN (Suppress-Notification):
- Set when vCPU temporarily not running (between scheduled-out & scheduled-in).
- HW: !SN → IPI sent. SN → silent posting (no IPI).

REQ-10: Per-vCPU vmcs.posted-intr-desc-addr:
- Guest-phys cached at vmcs.posted-intr-desc-addr.
- Per-vCPU live-migrate: PID re-allocated on dest; vmcs updated.

REQ-11: ACPI MADT POSTED_INTR_VECTOR allocation:
- Per-host kernel reserves 0xF2 (POSTED_INTR_VECTOR) + 0xF1 (POSTED_INTR_WAKEUP_VECTOR).
- Per-IDT entries point to apic_eoi + pi_wakeup_handler respectively.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pid_64byte_aligned` | INVARIANT | per-vCPU PID phys-addr 64-byte aligned. |
| `nv_in_pre_block_is_wakeup` | INVARIANT | post-pre_block PiDesc.NV == POSTED_INTR_WAKEUP_VECTOR. |
| `nv_in_post_block_is_normal` | INVARIANT | post-post_block PiDesc.NV == POSTED_INTR_VECTOR. |
| `wakeup_list_per_cpu_disjoint` | INVARIANT | per-vt at most on one CPU's wakeup_list. |
| `pre_block_implies_in_wakeup_list_or_kicked` | INVARIANT | post-pre_block: vcpu in wakeup_list ∨ kicked-immediately. |

### Layer 2: TLA+

`virt/kvm/posted_intr.tla`:
- Per-vCPU PI lifecycle: running → pre_block → asleep → post_block → running.
- Per-IRQ posting + ON-bit signaling + IPI delivery.
- Properties:
  - `safety_no_lost_wakeup` — per-IRQ posted while sleeping ⟹ vCPU eventually kicked.
  - `safety_no_double_wakeup_list_add` — per-vt at most one entry in any wakeup_list.
  - `liveness_pi_eventually_delivered` — per-IRQ posted to running vCPU eventually visible in guest VIRR.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Pi::pre_block` post: PiDesc.NV == 0xF1; vcpu in wakeup_list ∨ kicked | `Pi::pre_block` |
| `Pi::post_block` post: PiDesc.NV == 0xF2; vcpu not in wakeup_list | `Pi::post_block` |
| `Pi::wakeup_handler` post: per-PiDesc.ON-set vcpu kicked | `Pi::wakeup_handler` |
| `Pi::update_irte` post: IRTE.posted bit consistent with set-flag | `Pi::update_irte` |

### Layer 4: Verus/Creusot functional

`Per-IRQ from VT-d → PIR-bit set + ON-bit set + IPI to running CPU → guest VIRR updated → vCPU sees vector` semantic equivalence: per-VT-d posting matches Intel SDM §29.6 + VT-d Spec §5.2.3.

### hardening

(Inherits row-1 features from `virt/kvm/kvm-core.md` § Hardening.)

PI-specific reinforcement:

- **Per-vCPU PID 64-byte aligned** — defense against per-cache-line crossing causing torn write.
- **Per-host wakeup_list per-CPU spinlock** — defense against concurrent add/del.
- **Per-pre_block PIR-pending check** — defense against vCPU sleeping with pending IPI.
- **NV ← WAKEUP_VECTOR before list-add** — defense against race where IRQ arrives mid-pre-block.
- **NV ← POSTED_INTR_VECTOR before list-del** — defense against sleeping IPI mis-routing.
- **Per-IRTE update atomic via IOMMU API** — defense against partial-update IRTE causing wrong-target.
- **Per-vCPU SN-bit on pause** — defense against IPI to non-running vCPU spuriously waking host.
- **POSTED_INTR_VECTOR / WAKEUP_VECTOR reserved by host** — defense against guest claiming these.
- **VT-d posting-cap negotiated at boot** — defense against runtime mismatch.
- **Per-vCPU PID page locked in host phys-addr** — defense against guest swap migrating PID page.
- **Per-vCPU PID lifetime ≥ vCPU lifetime** — defense against use-after-free from late HW posting.

