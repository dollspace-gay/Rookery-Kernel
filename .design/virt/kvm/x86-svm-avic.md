# Tier-3: arch/x86/kvm/svm/avic.c — AMD AVIC (Advanced Virtual Interrupt Controller) per-vCPU vAPIC + IPI bypass + posted-IRQ

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/x86-svm.md
upstream-paths:
  - arch/x86/kvm/svm/avic.c
  - arch/x86/kvm/svm/svm.h (AVIC fields)
  - arch/x86/kvm/svm/svm.c (AVIC integration)
-->

## Summary

AMD AVIC (Advanced Virtual Interrupt Controller) is the AMD-SVM hardware feature that bypasses VM-exit on guest IPI delivery — analog to Intel's APICv. Per-VM physical-id table (4KiB; one entry per LAPIC) + per-VM logical-id table + per-vCPU AVIC backing-page (mapped IO-coherent so HW can write directly to guest's vAPIC). Per-IPI from guest CPU-A to running CPU-B: AVIC HW writes IRR-bit in CPU-B's backing page + signals doorbell; CPU-B sees IRR change at next vmenter or via doorbell-IPI. Avoids vmexit-storm on IPI-heavy workloads (LAPIC IPI, Linux scheduler-IPI, NMI). Per-VM IOMMU integration coordinates posted-MSI from passthrough devices.

This Tier-3 covers `arch/x86/kvm/svm/avic.c` (~1319 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `avic_init_vcpu(vcpu)` | per-vCPU AVIC setup | `Avic::init_vcpu` |
| `avic_init_vmcb(svm)` | per-VMCB AVIC setup | `Avic::init_vmcb` |
| `avic_vm_init(kvm)` | per-VM AVIC setup | `Avic::vm_init` |
| `avic_vm_destroy(kvm)` | per-VM teardown | `Avic::vm_destroy` |
| `avic_set_running(vcpu, is_running)` | per-vCPU run-state to AVIC | `Avic::set_running` |
| `avic_vcpu_load(vcpu, cpu)` | per-vCPU load to host CPU | `Avic::vcpu_load` |
| `avic_vcpu_put(vcpu)` | per-vCPU put from host CPU | `Avic::vcpu_put` |
| `avic_vcpu_blocking(vcpu)` | per-vCPU blocking | `Avic::vcpu_blocking` |
| `avic_vcpu_unblocking(vcpu)` | per-vCPU unblocking | `Avic::vcpu_unblocking` |
| `avic_incomplete_ipi_interception(vcpu)` | AVIC_INCOMPLETE_IPI vmexit | `Avic::handle_incomplete_ipi` |
| `avic_unaccel_trap_write(vcpu, pa)` | unaccelerated guest vAPIC-page-write | `Avic::unaccel_trap_write` |
| `avic_handle_apic_id_update(vcpu)` | per-vCPU APIC-id change | `Avic::handle_apic_id_update` |
| `avic_handle_dfr_update(vcpu)` | per-vCPU DFR update | `Avic::handle_dfr_update` |
| `avic_handle_ldr_update(vcpu)` | per-vCPU LDR update | `Avic::handle_ldr_update` |
| `avic_pi_update_irte(...)` | posted-IRQ IRTE update | `Avic::pi_update_irte` |
| `avic_get_phys_id_table_entry(...)` | per-vCPU phys-id table accessor | `Avic::get_phys_id_table_entry` |
| `avic_doorbell(...)` | per-vCPU doorbell IPI | `Avic::doorbell` |
| `avic_inhibit_set(kvm, reason)` / `_clear` | per-VM AVIC-inhibit | `Avic::inhibit_set` / `_clear` |

## Compatibility contract

REQ-1: Per-VM AVIC tables (4KiB each):
- `avic_logical_id_table`: per-LAPIC logical-id encoding (cluster + bitmap-within-cluster).
- `avic_physical_id_table`: per-LAPIC physical-id → AVIC backing-page-pa.

REQ-2: Per-vCPU AVIC backing page (4KiB):
- HW-mapped vAPIC page; CPU writes IRR/ISR/etc. directly.
- IO-coherent for IOMMU-direct posted-IRQ.
- Per-vCPU separate page.

REQ-3: VMCB AVIC fields:
- `avic_backing_page_pa` (vCPU's backing-page phys-addr).
- `avic_logical_id_table_pa` (per-VM table).
- `avic_physical_id_table_pa` (per-VM table).
- `avic_vapic_bar` (vAPIC base; typically xfee0_0000).
- secondary_exec_control bit ENABLE_AVIC.

REQ-4: Per-vCPU running flag:
- avic_set_running(vcpu, true) when entering guest.
- avic_set_running(vcpu, false) when vCPU blocked / not on a CPU.
- AVIC HW writes per-vCPU phys-id-table-entry.is_running = true; HW IPI direct-delivery enabled.

REQ-5: IPI delivery cases:
- Source guest writes ICR (Inter-Processor-Interrupt-Command-Register) to vAPIC backing page.
- HW reads dest_id, looks up phys-id-table:
  - Target vCPU running (is_running=1): HW writes IRR-bit in target vCPU's backing-page + signals doorbell IPI to host CPU.
  - Target vCPU not running (is_running=0): AVIC_INCOMPLETE_IPI vmexit on source.
- Source vCPU continues without vmexit if all targets running.

REQ-6: AVIC_INCOMPLETE_IPI vmexit handler:
- Source vCPU exits with reason AVIC_INCOMPLETE_IPI.
- KVM dispatches: per-target vCPU kvm_vcpu_kick.
- Wakes blocked target; AVIC-running re-enabled at next vmenter.

REQ-7: AVIC_UNACCELERATED_ACCESS vmexit:
- Some vAPIC writes not handled by AVIC HW (e.g., x2APIC mode regs, ICR2 in xAPIC).
- VMexit; KVM emulates write.

REQ-8: AVIC inhibit reasons:
- Various conditions force per-VM AVIC disable:
  - APICV_INHIBIT_REASON_DISABLED (unconditional).
  - APICV_INHIBIT_REASON_BLOCKIRQ (NMI / SMM / etc.).
  - APICV_INHIBIT_REASON_X2APIC (x2APIC requires different path; per-AMD older revs).
  - APICV_INHIBIT_REASON_NESTED.
  - APICV_INHIBIT_REASON_LOGICAL_ID_ALIASED (multiple LAPICs with same LDR).

REQ-9: avic_logical_id_table layout:
- 256 entries × 4 bytes.
- Per-entry: bit-mapped LAPICs in this logical-id-cluster.
- Cluster mode: entry index = cluster-id; bits = LAPICs in cluster.
- Flat mode: legacy; entry-0 holds bitmap of all LAPICs.

REQ-10: avic_physical_id_table layout:
- 256 entries × 8 bytes.
- Per-entry: backing-page-pa | is_running flag.

REQ-11: Posted-IRQ via IOMMU:
- For SR-IOV passthrough VFs: IOMMU IRTE.posted=1 + PDA = vCPU backing-page.
- IOMMU writes directly to backing-page IRR; signals doorbell.
- Bypasses host LAPIC; reduced overhead.

REQ-12: Per-vCPU APIC ID update:
- Guest writes APIC_ID register; AVIC traps → avic_handle_apic_id_update.
- KVM updates phys-id-table entry mapping.

## Acceptance Criteria

- [ ] AC-1: KVM-AMD with AVIC: 16-vCPU guest IPI-stress; per-IPI vmexit count near zero.
- [ ] AC-2: AVIC_INCOMPLETE_IPI: IPI to halted vCPU; vmexit handled; halted vCPU woken.
- [ ] AC-3: AVIC_UNACCELERATED_ACCESS: x2APIC-mode access traps; KVM emulates; correct result.
- [ ] AC-4: Posted-IRQ + SR-IOV: passthrough VF; per-VF MSI delivered via IOMMU posted; backing-page IRR set.
- [ ] AC-5: AVIC inhibit on x2APIC: enabling x2APIC inhibits AVIC; subsequent IPIs trap normally.
- [ ] AC-6: AVIC inhibit on NMI: NMI-inject inhibits AVIC briefly; clears post-injection.
- [ ] AC-7: Live migration: AVIC state preserved; tables re-mapped at destination.
- [ ] AC-8: Multi-VM AVIC: 4 VMs × 4 vCPUs all using AVIC; no cross-VM IRR leak.
- [ ] AC-9: kvm-unit-tests `avic` test passes.
- [ ] AC-10: Performance: AVIC reduces IPI-heavy workload runtime by ≥ 30% vs no-AVIC baseline.

## Architecture

`KvmAvic` per-VM (in kvm.arch):

```
struct KvmAvic {
  avic_logical_id_table: KArc<Page>,            // 4KiB
  avic_logical_id_table_pa: u64,
  avic_physical_id_table: KArc<Page>,            // 4KiB
  avic_physical_id_table_pa: u64,
  ldr_mode: u32,                                  // FLAT or CLUSTER
  apicv_inhibit_reasons: AtomicU64,               // bitmap
}
```

`VcpuAvic` per-vCPU (in vcpu_svm):

```
struct VcpuAvic {
  avic_backing_page: KArc<Page>,                  // 4KiB
  avic_backing_page_pa: u64,
  avic_logical_id: u32,
  avic_physical_id: u32,
  is_running: AtomicBool,
}
```

`Avic::init_vcpu(vcpu)`:
1. Allocate avic_backing_page; map IO-coherent.
2. backing_page_pa := page_to_pfn(backing_page) << PAGE_SHIFT.
3. avic_physical_id := vcpu.vcpu_id.
4. avic_logical_id := vcpu.vcpu_id.
5. Phys-id-table-entry[avic_physical_id] := backing_page_pa | is_running_bit.
6. Logical-id-table per-cluster bit set.

`Avic::init_vmcb(svm)`:
1. svm.vmcb01.control.avic_backing_page_pa = vcpu_avic.avic_backing_page_pa.
2. svm.vmcb01.control.avic_logical_id_table_pa = kvm_avic.avic_logical_id_table_pa.
3. svm.vmcb01.control.avic_physical_id_table_pa = kvm_avic.avic_physical_id_table_pa.
4. svm.vmcb01.control.secondary_exec_control |= AVIC_ENABLE.

`Avic::set_running(vcpu, is_running)`:
1. avic_phys_entry := per-vCPU phys-id-table entry.
2. Atomic-set or clear AVIC_PHYSICAL_ID_ENTRY_IS_RUNNING_MASK.
3. Atomic-update entry.

`Avic::handle_incomplete_ipi(vcpu)` flow:
1. Read VMCB.exit_info_1: type (INCOMPLETE_REASON_*) + dest_apicid + vector.
2. Switch type:
   - INVALID_TARGET: kvm_apic_match_dest fails; emulate via standard LAPIC.
   - GUEST_INVALID: target vCPU not running; kvm_vcpu_kick(target).
   - LOGICAL_DEST_NOT_FOUND: per-logical-id-table lookup failed.
3. Continue source vCPU.

`Avic::pi_update_irte(kvm, host_irq, guest_irq, set)` (IOMMU posted-IRQ):
1. If set: IRTE.posted = 1; IRTE.pda = vcpu_avic.avic_backing_page_pa; IRTE.notification-vector.
2. Else: IRTE.posted = 0.
3. iommu_modify_irte.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `phys_id_table_idx_bounded` | OOB | per-physical_id < 256; defense against OOB table access. |
| `backing_page_aligned` | INVARIANT | per-vCPU backing-page 4KiB-aligned + IO-coherent mapped. |
| `is_running_atomic` | INVARIANT | per-vCPU is_running flag transitions atomic. |
| `inhibit_reasons_validated` | INVARIANT | apicv_inhibit_reasons ⊆ valid bitmask. |
| `ipi_target_validated` | INVARIANT | INCOMPLETE_IPI dest_apicid validated against existing vCPU. |

### Layer 2: TLA+

`virt/kvm/avic_ipi.tla` (already exists from earlier Tier-3 collection in x86-svm.md) covers AVIC IPI delivery.

`virt/kvm/avic_inhibit.tla`:
- Per-VM AVIC state ∈ {Active, Inhibited(reasons)}.
- Properties:
  - `safety_inhibit_disables_avic` — Inhibited implies vmcb.control.AVIC_ENABLE clear.
  - `safety_inhibit_clear_re_enables` — clearing all reasons re-enables AVIC.
  - `safety_no_running_with_avic_disabled` — disabled AVIC implies all vCPU is_running cleared.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Avic::init_vcpu` post: backing-page allocated + mapped; phys-id-table populated | `Avic::init_vcpu` |
| `Avic::set_running` post: phys-id-table entry is_running matches | `Avic::set_running` |
| `Avic::handle_incomplete_ipi` post: target vCPU kicked OR emulated via LAPIC | `Avic::handle_incomplete_ipi` |
| `Avic::pi_update_irte` post: IRTE posted-bit + PDA atomically updated | `Avic::pi_update_irte` |

### Layer 4: Verus/Creusot functional

`Per-IPI: source vCPU writes ICR; if all targets running, no vmexit; target IRR-bit set + delivered via doorbell` semantic equivalence: per-IPI the target vAPIC sees the IPI vector + delivery-mode equivalent to native LAPIC IPI.

## Hardening

(Inherits row-1 features from `virt/kvm/x86-svm.md` § Hardening.)

AVIC-specific reinforcement:

- **Per-vCPU backing-page IO-coherent** — defense against device-DMA-write-during-CPU-read tearing.
- **Per-vCPU is_running atomic transitions** — defense against torn state during cross-CPU migration.
- **Per-VM phys-id-table protected by KVM lock** — defense against guest direct-write corrupting AVIC state.
- **Per-VM AVIC inhibit reasons bitmap** — defense against partial-inhibit causing inconsistent AVIC state.
- **AVIC-incomplete-IPI dest validation** — defense against guest IPI to non-existent vCPU.
- **Posted-IRQ IRTE PDA validated** — defense against PDA pointing to host kernel memory.
- **Per-vCPU LDR/DFR update under apic.lock** — defense against logical-id-table inconsistency.
- **Live-migrate clears AVIC state** — defense against post-migrate stale tables.
- **AVIC inhibit on x2APIC + nested** — defense against feature-mismatch causing IPI mis-delivery.
- **Per-VM AVIC enable bit gated on host AVIC support** — defense against running on non-AVIC host.
- **Per-vCPU APIC ID validated** — defense against guest setting APIC_ID > 255 in xAPIC mode.
- **Doorbell IPI vector validated** — defense against attacker-controlled vector causing #UD.

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — AVIC backing page / logical-ID / physical-ID tables allocated kernel-side; no user-pointer copy into VMCB.control.avic_* fields.
- **PAX_KERNEXEC** — `avic_init_vcpu`, `avic_vcpu_load`, `avic_ga_log_notifier`, `avic_handle_ldr_update` resolve through RX-only kernel text; SVM ops vtable RO.
- **PAX_RANDKSTACK** — AVIC unaccelerated-IPI / NPF and KVM_RUN paths inherit RANDKSTACK.
- **PAX_REFCOUNT** — per-VM AVIC physical-id-table and per-vCPU AVIC backing page refcount saturating.
- **PAX_MEMORY_SANITIZE** — AVIC backing page (per-vCPU APIC register file), physical-ID table, logical-ID table zeroed on alloc and zeroed on free so stale APIC-ID mapping cannot leak across VM destroy.
- **PAX_UDEREF** — no user pointer in AVIC table-write path; all writes through kernel `kvm_write_guest`.
- **PAX_RAP / kCFI** — `kvm_x86_ops.refresh_apicv_exec_ctrl`, `apicv_post_state_restore`, `set_apic_access_page_addr`, AVIC NPF / unaccel-write handlers RAP-signed.
- **GRKERNSEC_HIDESYM** — AVIC backing page kaddr, physical/logical-ID-table HPAs redacted unless CAP_SYSLOG + gr-rbac.
- **GRKERNSEC_DMESG** — AVIC enable/inhibit transitions, unaccelerated-IPI rate warnings, GA-log overflow rate-limited and gated by `dmesg_restrict`.
- **KVM ioctl CAP_SYS_ADMIN strict** — AVIC enable / KVM_DEV_VFIO posted-IRQ setup gated to CAP_SYS_ADMIN + gr-rbac VM-create + device-passthrough roles.
- **VMCB AVIC backing page validated** — `vmcb01.control.avic_backing_page` validated against kernel-allocated GPA before VMRUN.
- **Per-vCPU APIC ID validated** — xAPIC APIC_ID ≤ 255 enforced; out-of-range writes rejected so AVIC physical-ID-table index cannot OOB.
- **AVIC inhibit on x2APIC + nested** — feature-mismatch detected; AVIC disabled rather than running with inconsistent state.
- **Posted-IRQ IRTE PDA validated** — IRTE Posting-Descriptor-Address validated against kernel-allocated PD so guest cannot redirect IOMMU writes to host kernel memory.
- **Doorbell IPI vector validated** — doorbell vector ∈ allowed range; attacker-controlled value cannot cause #UD on host.
- **Nested-virt strict** — AVIC inhibited when L2 active; nested AVIC not supported (L0 cannot directly post to L2).

Per-doc rationale: AVIC bypasses VMVMCB-mediated APIC writes by mapping a per-vCPU APIC register page directly; the grsec reinforcement here validates the backing-page HPA + physical-ID-table indices to prevent OOB into host memory, inhibits AVIC when feature-mismatched (x2APIC, nested) rather than allowing inconsistent operation, and SANITIZEs AVIC tables on free to prevent successor-VM cross-tenant APIC-ID exposure.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- SVM core (covered in `x86-svm.md` Tier-3)
- LAPIC (covered in `x86-lapic.md` Tier-3)
- KVM core (covered in `kvm-core.md` Tier-3)
- Intel APICv (covered in `x86-vmx.md` Tier-3; analog feature)
- IOMMU IR (covered in `drivers/iommu/intel-irq-remap.md` Tier-3)
- Implementation code
