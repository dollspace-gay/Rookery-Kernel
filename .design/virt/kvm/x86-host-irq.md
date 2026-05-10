# Tier-3: arch/x86/kvm/vmx/posted_intr.c — VMX posted-interrupt processing (per-vCPU PI desc + IOMMU-direct posted-IRQ + cross-vCPU posted-IPI)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/x86-vmx.md
upstream-paths:
  - arch/x86/kvm/vmx/posted_intr.c
  - arch/x86/kvm/vmx/posted_intr.h
  - arch/x86/kvm/vmx/vmx.c (PI integration)
  - drivers/iommu/intel/irq_remapping.c (PI IRTE programming)
-->

## Summary

VMX posted-interrupt processing is the Intel-VMX hardware feature that delivers external interrupts (from devices via IOMMU IRTE.posted=1, OR from cross-vCPU sender) directly to a running guest without VM-exit. Per-vCPU 64-byte aligned `pi_desc` (Posted-Interrupt Descriptor) contains: PIR (256-bit posted-interrupt-request bitmap), control field (Outstanding-Notification ON / Suppress-Notification SN / Notification-Vector NV / Notification-Destination NDST). When source pre-pends interrupt: atomic-test-and-set PIR-bit + (if was-clear) signal NV-IPI to NDST. Target host-CPU running the vCPU receives NV-IPI; CPU processes via vmcs.posted-interrupt-notification-vector logic; injects IRR-bit on next instruction boundary. Drives major perf gains for SR-IOV passthrough + heavy IPI workloads.

This Tier-3 covers `arch/x86/kvm/vmx/posted_intr.c` (~319 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct pi_desc` | per-vCPU posted-IRQ descriptor | `kernel::kvm::x86::vmx::posted_intr::PiDesc` |
| `pi_desc_init(vcpu)` | per-vCPU pi_desc init | `PiDesc::init` |
| `vmx_vcpu_pi_load(vcpu, cpu)` | per-vCPU load to host CPU (update NDST) | `VcpuVmx::pi_load` |
| `vmx_vcpu_pi_put(vcpu)` | per-vCPU put from host CPU | `VcpuVmx::pi_put` |
| `pi_test_and_set_pir(vec, pi_desc)` | per-vec atomic-set PIR-bit | `PiDesc::test_and_set_pir` |
| `pi_test_pir(vec, pi_desc)` | per-vec PIR-bit query | `PiDesc::test_pir` |
| `pi_test_and_set_on(pi_desc)` | per-PI atomic-set ON | `PiDesc::test_and_set_on` |
| `pi_clear_on(pi_desc)` | per-PI clear ON | `PiDesc::clear_on` |
| `pi_test_on(pi_desc)` | per-PI ON query | `PiDesc::test_on` |
| `pi_set_sn(pi_desc)` / `pi_clear_sn(pi_desc)` | suppress-notification toggle | `PiDesc::set_sn` / `_clear_sn` |
| `pi_test_sn(pi_desc)` | suppress-notification query | `PiDesc::test_sn` |
| `vmx_pi_send_ipi(vcpu, vec)` | per-vCPU posted-IPI send | `VcpuVmx::pi_send_ipi` |
| `vmx_pi_update_irte(...)` | IOMMU IRTE update for posted-IRQ | `VcpuVmx::pi_update_irte` |
| `vmx_pi_post_block(...)` | per-vCPU post-block (deferred posted-IRQ delivery) | `VcpuVmx::pi_post_block` |
| `vmx_pi_pre_block(...)` | per-vCPU pre-block (configure SN+NV) | `VcpuVmx::pi_pre_block` |
| `pi_wakeup_handler(...)` | per-CPU wakeup-vector handler | `Vmx::pi_wakeup_handler` |
| `pi_test_and_clear_pir_to_irr(...)` | sync PIR → vAPIC IRR | `PiDesc::sync_pir_to_irr` |

## Compatibility contract

REQ-1: Per-vCPU `pi_desc` (64 bytes, cacheline-aligned):
- Bytes 0..31: PIR (256 bits = 1 per IRQ vector).
- Bytes 32..35: control field:
  - Bit[0]: ON (Outstanding Notification).
  - Bit[1]: SN (Suppress Notification).
  - Bits[7:0]: NV (Notification Vector; 16-255).
  - Bits[15:8]: NDST low byte (Notification Destination low byte).
  - Bits[31:16]: NDST high (cluster).
- Bytes 36..63: reserved.

REQ-2: Per-vCPU pi_desc allocation:
- 64-byte aligned via kzalloc + cacheline-align attribute.
- IO-coherent mapping for IOMMU-direct PIR-write.
- Per-vCPU separate.

REQ-3: Per-vCPU NV vector:
- Configurable; typically 0xF2 (POSTED_INTR_VECTOR) for Linux host.
- Different from Notification-Wakeup-Vector (used when vCPU blocked).

REQ-4: NDST (Notification Destination):
- 32-bit; encodes target host-CPU's APIC ID.
- Updated when vCPU migrates host-CPU (vmx_vcpu_pi_load).

REQ-5: Per-vCPU SN (Suppress Notification):
- Set when vCPU not running (e.g., halted/blocked).
- Source still sets PIR-bit but skips NV-IPI (source CPU doesn't waste IPI).
- Cleared when vCPU resumes; deferred posted-IRQs re-evaluated.

REQ-6: Per-source IPI delivery (`vmx_pi_send_ipi`):
1. pi_desc := target_vcpu.pi_desc.
2. was_set := pi_test_and_set_pir(vec, pi_desc).
3. If !was_set + !pi_test_sn(pi_desc):
   - was_on := pi_test_and_set_on(pi_desc).
   - If !was_on: apic_send_IPI(NDST, NV).
4. Else: skip IPI (already pending or suppressed).

REQ-7: Target host-CPU NV-IPI handler:
- CPU receives NV-IPI vector.
- HW processes: read pi_desc; clear ON; sync PIR → vAPIC IRR.
- If guest running: process IRQ on next instruction boundary.
- If not guest running (between vmenter/vmexit): vmexit reason POSTED_INTR_NV.

REQ-8: Per-vmexit POSTED_INTR_NV handler:
- Sync PIR → vAPIC IRR.
- Re-vmenter to deliver IRQ.

REQ-9: vCPU pre-block / post-block (halted vCPU):
- pre_block: SN := 1; NV := POSTED_INTR_WAKEUP_VECTOR; arm wakeup-handler.
- post_block: SN := 0; NV := POSTED_INTR_VECTOR.
- Wakeup-handler (per-CPU IRQ): per-blocked-vCPU walk; check pi_desc.PIR; if bits set: kvm_vcpu_kick.

REQ-10: IOMMU posted-IRQ via IRTE:
- For SR-IOV passthrough: IOMMU IRTE.posted=1; IRTE.pda = pi_desc_addr; IRTE.NV = posted_intr_nv.
- IOMMU writes PIR-bit + sends NV-IPI directly.
- Bypass host LAPIC.

REQ-11: Per-vCPU sync_pir_to_irr (per-vmenter):
- Walk PIR-bitmap (256 bits).
- For each set-bit: set vAPIC IRR-bit; clear PIR-bit.
- Atomic via cmpxchg.
- Used when ON observed but no NV-IPI delivered (race recovery).

REQ-12: Live migration of pending PI:
- Per-vCPU pi_desc state migrated.
- Pending PIR bits visible to destination vCPU on resume.

## Acceptance Criteria

- [ ] AC-1: Boot 16-vCPU guest with KVM-VMX + APICv enabled: per-vCPU pi_desc allocated.
- [ ] AC-2: Cross-vCPU IPI: vCPU0 sends IPI to vCPU1; vCPU1 receives via posted-IRQ; no source vmexit.
- [ ] AC-3: SR-IOV NIC + IOMMU-IR: per-VF MSI-X delivered via posted-IRQ; PIR populated; no vmexit.
- [ ] AC-4: Halted vCPU IPI: vCPU0 sends IPI to halted vCPU1; SN set so no IPI; wake handler kicks vCPU1.
- [ ] AC-5: NDST update on cross-CPU vCPU migrate: vCPU moves CPU; pi_desc.NDST updated; subsequent IPI to new CPU.
- [ ] AC-6: PIR overflow protection: 256-vector ID space full; further sets idempotent.
- [ ] AC-7: ON race recovery: source sets ON+PIR but vmexit before NV-IPI; sync_pir_to_irr at next vmenter recovers.
- [ ] AC-8: kvm-unit-tests `posted_intr` test passes.
- [ ] AC-9: Performance: posted-IRQ eliminates ≥ 90% of IPI vmexits on cross-vCPU IPI workload.
- [ ] AC-10: Live migration: in-flight PIR pending preserved; destination vCPU sees IRQ on first vmenter.

## Architecture

`PiDesc`:

```
#[repr(C, align(64))]
struct PiDesc {
  pir: [AtomicU64; 4],                          // 256 bits
  control: AtomicU64,                            // [ON | SN | NV | NDST]
  reserved: [u64; 3],
}
```

Per-vCPU additions (already in VcpuVmx):

```
struct VcpuVmx {
  ...
  pi_desc: KArc<PiDesc>,
  pi_desc_addr: u64,
  posted_intr_nv: u8,
  cpu: AtomicI32,
}
```

`VcpuVmx::pi_load(vcpu, cpu)`:
1. pi_desc := vcpu.arch.vmx.pi_desc.
2. ndst := apic_id_of_cpu(cpu).
3. cmpxchg pi_desc.control: { NDST = ndst; SN := 0; }.
4. vcpu.arch.vmx.cpu := cpu.

`VcpuVmx::pi_put(vcpu)`:
1. pi_desc := vcpu.arch.vmx.pi_desc.
2. cmpxchg pi_desc.control: { SN := 1 }.

`VcpuVmx::pi_send_ipi(vcpu, vec)`:
1. pi_desc := vcpu.arch.vmx.pi_desc.
2. was_pir_set := pi_test_and_set_pir(vec, pi_desc).
3. If was_pir_set: return (no extra IPI needed).
4. If pi_test_sn(pi_desc): return (suppressed).
5. was_on := pi_test_and_set_on(pi_desc).
6. If was_on: return.
7. apic_send_IPI(pi_desc.NDST, pi_desc.NV).

`VcpuVmx::pi_pre_block(vcpu)` (halted-vCPU prep):
1. pi_desc := vcpu.arch.vmx.pi_desc.
2. Save old NV; set NV := POSTED_INTR_WAKEUP_VECTOR.
3. Set SN := 1.
4. Add vcpu to per-CPU pi_blocked_vcpus list.

`VcpuVmx::pi_post_block(vcpu)`:
1. Restore NV := POSTED_INTR_VECTOR.
2. Clear SN.
3. Remove from per-CPU pi_blocked_vcpus list.
4. sync_pir_to_irr(vcpu) → check for any pending-during-block IRQs.

`Vmx::pi_wakeup_handler(cpu)` (per-CPU IRQ on POSTED_INTR_WAKEUP_VECTOR):
1. For each vcpu in per-CPU pi_blocked_vcpus:
   - if pi_test_pir_any(vcpu.pi_desc): kvm_vcpu_kick(vcpu).

`PiDesc::sync_pir_to_irr(vcpu)`:
1. apic := vcpu.arch.apic.
2. For each PIR word: cmpxchg out the bits; OR them into IRR.
3. clear ON.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pi_desc_aligned` | INVARIANT | pi_desc 64-byte aligned; defense against split-cacheline access. |
| `pir_atomic_setbit` | INVARIANT | per-PIR-bit set via atomic test-and-set. |
| `nv_vector_validated` | INVARIANT | NV ∈ [16, 255]; defense against reserved-vector causing CPU exception. |
| `ndst_apic_id_valid` | INVARIANT | NDST encodes valid host APIC ID. |
| `sn_consistent_with_blocked` | INVARIANT | SN==1 iff vCPU blocked or not running. |

### Layer 2: TLA+

`virt/kvm/posted_intr.tla` (already exists from earlier Tier-3 collection) covers per-IPI delivery semantics.

`virt/kvm/posted_intr_blocked.tla`:
- Per-vCPU state ∈ {Running(cpu), Blocked, Migrating}.
- Transitions per pi_load / pi_put / pi_pre_block / pi_post_block.
- Properties:
  - `safety_sn_iff_not_running` — SN set iff vCPU in Blocked or Migrating.
  - `safety_wakeup_eventually_kicked` — pi_wakeup_handler kicks vCPU iff PIR has bits.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `VcpuVmx::pi_load` post: pi_desc.NDST == apic_id(cpu); SN cleared | `VcpuVmx::pi_load` |
| `VcpuVmx::pi_put` post: pi_desc.SN set | `VcpuVmx::pi_put` |
| `VcpuVmx::pi_send_ipi` post: PIR-bit set; NV-IPI sent iff !suppressed && was-clear | `VcpuVmx::pi_send_ipi` |
| `PiDesc::sync_pir_to_irr` post: per-PIR-bit moved to IRR; PIR cleared | `PiDesc::sync_pir_to_irr` |
| Per-vCPU pi_desc allocation guaranteed aligned + IO-coherent | `PiDesc::init` |

### Layer 4: Verus/Creusot functional

`Per-IPI: source set PIR + NV-IPI → target eventually sees IRR-bit set + delivered` semantic equivalence: per-IPI the target vCPU sees IRQ at correct vector, regardless of running-on-CPU vs blocked.

## Hardening

(Inherits row-1 features from `virt/kvm/x86-vmx.md` § Hardening.)

posted-IRQ-specific reinforcement:

- **pi_desc 64-byte aligned + IO-coherent** — defense against IOMMU + CPU concurrent access tearing.
- **pir atomic test-and-set** — defense against torn PIR-bit-set.
- **NV vector validated [16, 255]** — defense against reserved-vector causing CPU exception.
- **NDST update under cmpxchg** — defense against cross-CPU NDST race during migrate.
- **SN coordinated with vCPU run-state** — defense against IPI-loss during blocked-state.
- **POSTED_INTR_WAKEUP_VECTOR distinct from POSTED_INTR_VECTOR** — defense against IPI-loop on blocked vCPU.
- **Per-CPU pi_blocked_vcpus list under per-CPU lock** — defense against concurrent block/wake races.
- **sync_pir_to_irr at vmenter unconditional** — defense against ON-set-but-no-IPI race.
- **IOMMU IRTE.PDA validated against pi_desc allocation** — defense against IOMMU writing to host kernel memory.
- **Live-migrate pi_desc serialized** — defense against post-migrate stale pending-state.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- VMX core (covered in `x86-vmx.md` Tier-3)
- LAPIC (covered in `x86-lapic.md` Tier-3)
- KVM core (covered in `kvm-core.md` Tier-3)
- AMD AVIC (covered in `x86-svm-avic.md` Tier-3; analog feature)
- IOMMU IR posted-IRQ programming (covered in `drivers/iommu/intel-irq-remap.md` Tier-3)
- KVM event injection (covered in `x86-vmx-eventinj.md` Tier-3)
- Implementation code
