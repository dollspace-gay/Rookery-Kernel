# Tier-3: arch/x86/kvm/svm/svm.c (ASID subset) — AMD ASID (Address Space Identifier) per-VM TLB tagging + per-CPU asid_generation

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/x86-svm.md
upstream-paths:
  - arch/x86/kvm/svm/svm.c (ASID-related sections; see grep "asid|ASID|new_asid|asid_generation")
  - arch/x86/kvm/svm/svm.h (struct svm_cpu_data, asid_generation field)
-->

## Summary

ASID (Address Space Identifier) is AMD-SVM's per-VM TLB-tagging mechanism (analog to Intel-VMX VPID): each VMCB carries a 32-bit ASID; CPU tags TLB entries by ASID; cross-VM context switch preserves TLB. AMD's allocation differs from Intel: instead of host-wide bitmap, KVM uses per-CPU asid_generation + next_asid fields — each VM-load on a CPU advances next_asid; when next_asid > max_asid, asid_generation increments (forcing TLB-FLUSH on next vmenter for stale-generation VMs). Reuses ASIDs across VMs but invalidates via generation-check. Reserves low ASIDs for SEV (per-VM SEV ASIDs need stable per-VM HKID-binding).

This Tier-3 covers ASID-related code in `arch/x86/kvm/svm/svm.c` (~150-200 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct svm_cpu_data` | per-CPU SVM state (asid mgmt) | `kernel::kvm::x86::svm::SvmCpuData` |
| `svm_cpu_data.next_asid` | per-CPU next-asid cursor | `SvmCpuData::next_asid` |
| `svm_cpu_data.max_asid` | per-CPU max ASID (CPUID) | `SvmCpuData::max_asid` |
| `svm_cpu_data.min_asid` | per-CPU min ASID (after SEV reserved) | `SvmCpuData::min_asid` |
| `svm_cpu_data.asid_generation` | per-CPU generation counter | `SvmCpuData::asid_generation` |
| `new_asid(svm, sd)` | per-vCPU assign ASID | `Svm::new_asid` |
| `init_vmcb(svm, init_event)` | per-VM vmcb-init incl. asid | `Svm::init_vmcb` |
| `svm_flush_tlb_user(vcpu)` | per-vCPU TLB flush | `VcpuSvm::flush_tlb_user` |
| `svm_flush_tlb_guest(vcpu)` | guest-only TLB flush | `VcpuSvm::flush_tlb_guest` |
| `svm_flush_tlb_current(vcpu)` | current-vCPU flush | `VcpuSvm::flush_tlb_current` |
| `svm_flush_tlb_all(vcpu)` | full TLB flush | `VcpuSvm::flush_tlb_all` |
| `pre_svm_run(svm)` | per-vmenter ASID-generation check | `VcpuSvm::pre_svm_run` |
| `vmcb.tlb_control` | per-VMCB TLB-flush control | `Vmcb::tlb_control` |
| `TLB_CONTROL_DO_NOTHING` / `_FLUSH_ASID` / `_FLUSH_ALL_ASIDS` / `_FLUSH_ASID_LOCAL` | TLB-control values | `TlbControl::*` |
| `max_sev_asid` | per-host SEV ASID upper bound | `Svm::MAX_SEV_ASID` |
| `min_sev_asid` | per-host SEV ASID lower bound | `Svm::MIN_SEV_ASID` |

## Compatibility contract

REQ-1: ASID feature detection:
- AMD CPUID 0x80000008:EBX[31:16] = max_asid + 1.
- Reserved range 1..max_sev_asid for SEV (per-VM stable assignment).
- Available range max_sev_asid+1..max_asid for non-SEV (rotated).

REQ-2: Per-CPU `svm_cpu_data`:
- `tss_desc` (per-CPU TSS for SVM hardware tasks).
- `save_area` (per-CPU host-save area for VMRUN).
- `next_asid` (next ASID to assign).
- `min_asid` (lower bound; max_sev_asid+1).
- `max_asid` (upper bound; from CPUID).
- `asid_generation` (per-CPU; bumped on overflow).
- `current_vmcb` (per-CPU last-loaded vmcb; for cache flush).

REQ-3: Per-vCPU `vcpu_svm` ASID fields:
- `asid` (current ASID).
- `current_vmcb.asid_generation` (per-VMCB generation snapshot).
- vmcb01.control.asid (programmed ASID for VMRUN).

REQ-4: Per-vCPU ASID assign (`new_asid`):
1. If sd.next_asid > sd.max_asid:
   - sd.asid_generation++.
   - sd.next_asid = sd.min_asid.
   - vmcb01.control.tlb_control = FLUSH_ALL_ASIDS (force flush all on next vmenter; old ASIDs may overlap).
2. svm.asid := sd.next_asid.
3. svm.current_vmcb.asid_generation := sd.asid_generation.
4. sd.next_asid++.
5. vmcb01.control.asid = svm.asid.

REQ-5: Per-vmenter ASID-generation check (`pre_svm_run`):
1. If svm.current_vmcb.asid_generation != sd.asid_generation:
   - This VM's ASID is stale (overlapped); reassign via new_asid.

REQ-6: TLB-control field semantics:
- DO_NOTHING (0): no flush; preserve TLB.
- FLUSH_ASID (1): flush this vmcb's ASID-tagged entries.
- FLUSH_ALL_ASIDS (3): flush all ASID-tagged entries.
- FLUSH_ASID_LOCAL (7): flush this vmcb's ASID-tagged entries on this CPU only.

REQ-7: Per-vCPU CR3 change:
- KVM's `svm_load_mmu_pgd(vcpu, root_hpa)` sets vmcb01.save.cr3 + vmcb01.tlb_control = FLUSH_ASID.
- Eliminates full-TLB-flush on per-process context-switch within guest.

REQ-8: Cross-CPU vCPU migrate:
- vCPU.cpu == new_cpu? per-CPU asid_generation may differ; new_asid called.
- This naturally re-assigns ASID + flushes stale TLB.

REQ-9: SEV ASID stability:
- Per-SEV-VM, ASID assigned at TDH.MNG.CREATE and not rotated.
- Encryption key bound to ASID in HW; rotating would break SEV.

REQ-10: Live migration:
- ASID is per-host transient state; not migrated.
- Destination host assigns new ASID at first vmenter.

REQ-11: Per-VMCB tlb_control validated:
- Only valid values; defense against L1 setting reserved value.

## Acceptance Criteria

- [ ] AC-1: Boot 16-vCPU guest under qemu+kvm with kvm-amd: per-vCPU vmcb01.control.asid populated.
- [ ] AC-2: Cross-VM context-switch: TLB-miss-rate substantially reduced vs no-ASID-tagging baseline.
- [ ] AC-3: Guest CR3-change: only FLUSH_ASID issued; not full TLB-flush.
- [ ] AC-4: ASID exhaustion: spawn many VMs until next_asid > max_asid; verify generation increments + FLUSH_ALL_ASIDS.
- [ ] AC-5: Cross-CPU migrate: vCPU moves CPU; new_asid reassigns + FLUSH_ASID; no functional regression.
- [ ] AC-6: SEV-VM: SEV ASID assigned at create + stable across vCPU lifetime.
- [ ] AC-7: Nested-SVM: L1 with vmcb01.asid; L2 with vmcb02.asid distinct.
- [ ] AC-8: Live migration: post-migrate ASID reassigned at first vmenter; no TLB-leak from source.
- [ ] AC-9: kvm-unit-tests `asid` test passes (where supported).
- [ ] AC-10: Performance: ASID-tagging gives ≥ 50% TLB-hit-rate improvement on 4-VM 4-vCPU stress.

## Architecture

`SvmCpuData` per-CPU:

```
struct SvmCpuData {
  tss_desc: KArc<TssDesc>,
  save_area: KArc<HostSave>,
  next_asid: AtomicU32,
  min_asid: u32,
  max_asid: u32,
  asid_generation: AtomicU32,
  current_vmcb: AtomicPtr<Vmcb>,
}
```

Per-vCPU additions (already in VcpuSvm; see x86-svm.md):

```
struct VcpuSvm {
  ...
  asid: u32,
  current_vmcb_asid_gen: u32,                  // per-VMCB generation snapshot
}
```

`Svm::new_asid(svm, sd)`:
1. If sd.next_asid > sd.max_asid:
   - sd.asid_generation = sd.asid_generation.fetch_add(1) + 1.
   - sd.next_asid = sd.min_asid.
   - svm.vmcb01.control.tlb_control = FLUSH_ALL_ASIDS.
2. svm.asid := sd.next_asid.
3. svm.current_vmcb_asid_gen := sd.asid_generation.
4. sd.next_asid := sd.next_asid + 1.
5. svm.vmcb01.control.asid := svm.asid.

`VcpuSvm::pre_svm_run(svm)` (per-vmenter):
1. sd := this_cpu_ptr(SVM_CPU_DATA).
2. If svm.current_vmcb_asid_gen != sd.asid_generation OR svm.cpu != smp_processor_id():
   - Svm::new_asid(svm, sd).
3. svm.cpu := smp_processor_id().

`VcpuSvm::flush_tlb_current(vcpu)`:
1. svm := vcpu.arch.svm.
2. If guest_mode (running L2): vmcb02.control.tlb_control = FLUSH_ASID.
3. Else: vmcb01.control.tlb_control = FLUSH_ASID.

`VcpuSvm::flush_tlb_all(vcpu)`:
1. svm.vmcb01.control.tlb_control = FLUSH_ALL_ASIDS.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `asid_within_range` | INVARIANT | per-vCPU asid ∈ [min_asid, max_asid]. |
| `asid_generation_monotonic` | INVARIANT | asid_generation per-CPU monotonically increases. |
| `asid_unique_per_generation` | INVARIANT | within one generation, per-CPU per-asid assigned to at most one vCPU. |
| `tlb_control_validated` | INVARIANT | tlb_control ∈ {DO_NOTHING, FLUSH_ASID, FLUSH_ALL_ASIDS, FLUSH_ASID_LOCAL}. |
| `sev_asid_reserved` | INVARIANT | non-SEV ASID assignment stays in [min_asid, max_asid]; SEV ASIDs untouched. |

### Layer 2: TLA+

`virt/kvm/svm_asid_generation.tla`:
- Per-CPU state: (asid_generation, next_asid).
- Per-vCPU state: (asid, asid_gen_snapshot).
- Properties:
  - `safety_no_stale_asid` — per-vmenter, vCPU.asid_gen_snapshot == sd.asid_generation OR ASID reassigned.
  - `safety_overflow_flush_all` — when next_asid > max_asid, FLUSH_ALL_ASIDS issued before next vmenter.
  - `liveness_eventual_assignment` — every vCPU eventually has valid ASID at vmenter.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Svm::new_asid` post: svm.asid valid; sd.next_asid advanced; generation matched | `Svm::new_asid` |
| `VcpuSvm::pre_svm_run` post: svm.current_vmcb_asid_gen == sd.asid_generation | `VcpuSvm::pre_svm_run` |
| Per-CPU sd.next_asid in [min_asid, max_asid+1] post-new_asid | invariants on assignment |
| Per-VMCB asid populated before VMRUN | invariants on init_vmcb + pre_svm_run |

### Layer 4: Verus/Creusot functional

`Per-VM ASID assignment + per-vmenter generation-check = no TLB-leak across VMs` semantic equivalence: per-VM TLB entries visible only to that VM's vCPUs running with matching ASID generation.

## Hardening

(Inherits row-1 features from `virt/kvm/x86-svm.md` § Hardening.)

ASID-specific reinforcement:

- **Per-CPU asid_generation overflow → FLUSH_ALL_ASIDS** — defense against post-overflow ASID-aliasing.
- **Per-vmenter generation-check** — defense against stale ASID across overflow.
- **SEV ASID range reserved** — defense against rotating SEV-VM ASID breaking encryption.
- **Cross-CPU migrate triggers new_asid** — defense against stale per-CPU TLB on different CPU.
- **tlb_control field validated** — defense against guest manipulation in nested-SVM.
- **Per-CPU sd.next_asid atomic** — defense against concurrent assignment race.
- **vmcb01.control.asid populated before VMRUN** — defense against VMRUN with asid=0 causing #GP.
- **Per-VM SEV ASID stable post-create** — defense against accidental rotation breaking SEV.
- **Nested-SVM L2 ASID distinct** — defense against L1/L2 TLB-aliasing.
- **Live-migrate clears pre-migrate ASID** — defense against post-migrate stale-TLB.
- **min_sev_asid/max_sev_asid range from CPUID** — defense against incorrect SEV-ASID-allocation.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- SVM core (covered in `x86-svm.md` Tier-3)
- Nested-SVM (covered in `x86-nested-svm.md` Tier-3)
- SEV (covered in `x86-sev.md` Tier-3)
- Intel VPID (covered in `x86-vmx-vpid.md` Tier-3; analog feature)
- Implementation code
