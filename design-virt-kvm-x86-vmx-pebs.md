---
title: "Tier-3: arch/x86/kvm/vmx/pmu_intel.c (PEBS subset) — Intel PEBS (Precise-Event-Based-Sampling) virtualization"
tags: ["tier-3", "virt-kvm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Intel PEBS (Precise-Event-Based-Sampling) is a CPU feature that on counter overflow records detailed CPU state (RIP, GPRs, latency, branch records) into a per-CPU buffer at MSR_IA32_DS_AREA, allowing accurate per-instruction sampling without IPI-skid. KVM virtualizes PEBS for guests so guest perf can use precise events: per-vCPU MSR_IA32_DS_AREA + MSR_PEBS_DATA_CFG + MSR_IA32_PEBS_ENABLE shadowed; per-counter PEBS-record format dispatched through host PEBS facility with guest-visible buffer remapping. Critical for: cloud-perf-tooling (production-perf-debugging in VMs), hot-spot identification with per-instruction granularity.

This Tier-3 covers PEBS-related code in `arch/x86/kvm/vmx/pmu_intel.c` (~150-200 lines) + `vmx.c` PEBS vmexit handler.

### Acceptance Criteria

- [ ] AC-1: KVM_FEATURE_PEBS advertised: guest CPUID 0xA + MSR_IA32_PERF_CAPABILITIES PEBS bit set.
- [ ] AC-2: Boot Linux guest with `perf record -e cycles:p` (precise event): PEBS path active; samples collected.
- [ ] AC-3: Per-instruction precision: sampled RIP within 1 instruction of overflow (vs ~10 instructions for non-PEBS).
- [ ] AC-4: DS-area accessible: guest writes MSR_IA32_DS_AREA = guest-VA; subsequent overflows write records there.
- [ ] AC-5: PMI delivery: PEBS_index reaches threshold; guest LVT_PMI fires; guest perf-handler runs.
- [ ] AC-6: Live migration: in-flight perf record session preserved.
- [ ] AC-7: Multi-vCPU: 4-vCPU guest each running perf-record; per-vCPU records distinct.
- [ ] AC-8: kvm-unit-tests `pebs` test passes (where supported).
- [ ] AC-9: Spec-violation defense: WRMSR(MSR_IA32_PEBS_ENABLE) with reserved bit set raises #GP.

### Architecture

`Pmu` per-vCPU additions for PEBS:

```
struct Pmu {
  ...
  pebs_enable: u64,                            // per-counter PEBS-enable bitmap
  pebs_enable_rsvd: u64,                       // per-vCPU reserved bits
  pebs_data_cfg: u64,
  pebs_data_cfg_rsvd: u64,
  ds_area: u64,                                // guest VA of DS-Save-Area
  pebs_buffer_cache: GfnToHvaCache,            // for fast guest-DS-area access
  ...
}
```

`Pmu::pebs_enable(vcpu, msr_data)`:
1. If data & pmu.pebs_enable_rsvd: return -EINVAL.
2. diff := pmu.pebs_enable ^ data.
3. pmu.pebs_enable = data.
4. For each bit in diff: per-PMC reprogram with new PEBS state.

`Pmu::pebs_handle(vcpu, perf_event)` (per-overflow):
1. pmc := perf_event_to_pmc(perf_event).
2. Read host-side PEBS record.
3. Translate guest DS-area pointer + index to host VA via pebs_buffer_cache.
4. memcpy_to_guest(host-record-bytes → guest-VA-of-DS-area + index).
5. Increment guest PEBS_index.
6. If PEBS_index >= PEBS_interrupt_threshold:
   - kvm_pmu_deliver_pmi(vcpu).

`Vcpu::handle_msr` MSR_PEBS_DATA_CFG branch:
1. Validate: data & pmu.pebs_data_cfg_rsvd == 0.
2. pmu.pebs_data_cfg = data.
3. Reprogram per-counter to emit per-config PEBS record fields.

### Out of Scope

- KVM PMU core (covered in `x86-pmu.md` Tier-3)
- VMX vendor (covered in `x86-vmx.md` Tier-3)
- Intel LBR (Last-Branch-Record) (covered in `x86-vmx-lbr.md` future Tier-3)
- AMD IBS (Instruction-Based-Sampling; AMD analog of PEBS; covered in `x86-svm-ibs.md` future Tier-3)
- KVM core (covered in `kvm-core.md` Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `intel_pmu_set_msr(...)` PEBS branches | guest WRMSR(PEBS_*) handler | covered in `Pmu::set_msr` |
| `intel_pmu_get_msr(...)` PEBS branches | guest RDMSR(PEBS_*) handler | `Pmu::get_msr` |
| `MSR_IA32_PEBS_ENABLE` (0x3F1) | per-counter PEBS enable bitmap | UAPI |
| `MSR_IA32_DS_AREA` (0x600) | guest DS area pointer | UAPI |
| `MSR_PEBS_DATA_CFG` (0x3F2) | per-counter PEBS payload config | UAPI |
| `MSR_IA32_PERF_CAPABILITIES` (0x345) | PEBS feature enumeration | UAPI |
| `intel_pmu_lbr_enable(vcpu)` | LBR (Last-Branch-Record) enable; complementary | covered in `Pmu::lbr_enable` |
| `intel_pmu_pebs_enable(vcpu)` | per-vCPU PEBS enable | `Pmu::pebs_enable` |
| `intel_pmu_pebs_disable(vcpu)` | per-vCPU PEBS disable | `Pmu::pebs_disable` |
| `kvm_pmu_pebs_handle(vcpu)` | per-overflow PEBS-record dispatch | `Pmu::pebs_handle` |
| `setup_pebs_buffer(vcpu, addr, size)` | per-vCPU PEBS-buffer mapping | `Pmu::setup_pebs_buffer` |
| `vmx_pmu_event_overflow(perf_event)` | per-overflow callback (PEBS path) | `VcpuVmx::pmu_event_overflow` |

### compatibility contract

REQ-1: PEBS-feature gating:
- Host PEBS-supported: CPUID 0xA + MSR_IA32_PERF_CAPABILITIES.
- Guest sees PEBS via per-vCPU CPUID 0xA + MSR_IA32_PERF_CAPABILITIES (when KVM_FEATURE_PEBS allowed).

REQ-2: Per-vCPU PEBS state in `kvm_pmu`:
- `pebs_enable` (MSR shadow; per-counter enable bitmap).
- `ds_area` (MSR_IA32_DS_AREA shadow; guest VA of DS-Save-Area).
- `pebs_data_cfg` (MSR_PEBS_DATA_CFG shadow; per-counter payload config).
- `pebs_enable_rsvd` (per-vCPU reserved-bits in pebs_enable based on host capabilities).

REQ-3: DS-Save-Area layout:
- 4-record area: BTS_buffer_base, BTS_index, BTS_absolute_max, BTS_interrupt_threshold (legacy BTS).
- Then: PEBS_buffer_base, PEBS_index, PEBS_absolute_max, PEBS_interrupt_threshold.
- Pointer field MSR_IA32_DS_AREA must point to valid mapped region.

REQ-4: Per-counter PEBS-record format:
- Per-counter generates 60-byte (or larger for adaptive PEBS) record on overflow.
- Record fields: RIP, GPRs (RAX..R15), RFLAGS, last-branch-records (when LBR-PEBS), latency.

REQ-5: Per-vCPU pebs_enable WRMSR:
1. Validate: data & pebs_enable_rsvd == 0.
2. Update vcpu.arch.pmu.pebs_enable.
3. For changed bits: per-PMC reprogram with PEBS-enabled.

REQ-6: MSR_IA32_DS_AREA WRMSR:
1. Validate: addr aligned + within guest mappable range.
2. Update vcpu.arch.pmu.ds_area.
3. Reprogram per-counter to use new DS area.

REQ-7: PEBS overflow handling:
1. Host PMC overflow → host PEBS engine writes record to host DS area.
2. perf_event callback fires.
3. KVM intercepts: `vmx_pmu_event_overflow`.
4. KVM relocates record from host DS area to guest DS area (guest-VA).
5. Increment guest's PEBS_index.
6. If PEBS_index reaches PEBS_interrupt_threshold: inject PMI to guest.

REQ-8: Adaptive-PEBS (newer Intel CPUs):
- Per-counter pebs_data_cfg specifies extra fields.
- Variable-size record per config.

REQ-9: Live migration:
- Per-vCPU PEBS MSRs migrated.
- DS area state checkpointed.

REQ-10: Nested-VMX PEBS:
- L2 may use PEBS via L1 KVM.
- Cascading DS-area; out-of-scope at this Tier-3.

REQ-11: KVM_CAP_PMU_CAPABILITY for per-VM PEBS gating:
- Userspace can disable PEBS for security-sensitive VMs.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pebs_record_size_validated` | INVARIANT | per-config record size ≤ buffer-end - PEBS_index; defense against record-write OOB. |
| `ds_area_aligned` | INVARIANT | guest DS-area VA aligned per CPU spec; defense against straddling-page write. |
| `pebs_enable_reserved_bits` | INVARIANT | data & pebs_enable_rsvd == 0 enforced. |
| `pebs_index_advances` | INVARIANT | PEBS_index monotonically advances; clamped at absolute_max. |

### Layer 2: TLA+

`virt/kvm/pebs_overflow.tla`:
- Per-PMC overflow → host record write → guest relocation → potential PMI.
- Properties:
  - `safety_record_in_guest_ds_area` — every relocated record lies within guest DS-area bounds.
  - `safety_pmi_at_threshold` — PMI delivered iff PEBS_index reaches PEBS_interrupt_threshold.
  - `liveness_relocated_eventually` — every host-overflow eventually relocated to guest.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Pmu::pebs_enable` post: pmu.pebs_enable matches data; per-PMC reprogrammed | `Pmu::pebs_enable` |
| `Pmu::pebs_handle` post: guest DS-area updated; PEBS_index incremented; PMI queued if threshold | `Pmu::pebs_handle` |
| Per-vCPU pebs_buffer_cache validated against memslot | invariants on relocation |
| PEBS_index ≤ PEBS_absolute_max at all times | invariants on increment |

### Layer 4: Verus/Creusot functional

`Per-counter overflow → record formatted per pebs_data_cfg → record copied to guest DS-area → PMI when threshold reached` semantic equivalence: per-overflow guest sees same record sequence as bare-metal PEBS would produce.

### hardening

(Inherits row-1 features from `virt/kvm/x86-pmu.md` § Hardening.)

PEBS-specific reinforcement:

- **Per-vCPU PEBS-enable reserved-bits** — defense against guest enabling unsupported PEBS features causing host CPU exception.
- **DS-area pointer validated against memslot** — defense against guest pointing to host kernel memory.
- **PEBS_index bound checked against absolute_max** — defense against PEBS-index-overflow corrupting adjacent memory.
- **Record relocation atomic** — defense against guest reading half-written record.
- **Per-counter PEBS gating via pebs_enable bitmap** — defense against record-leak across unrelated counters.
- **Live-migrate preserves PEBS state** — defense against post-migrate inconsistent counter ↔ DS-area binding.
- **Nested-VMX cascade gating** — defense against L2 PEBS escaping into L0 host.
- **KVM_CAP_PMU_CAPABILITY disable** — defense against PEBS-on-untrusted-VM revealing host PMU state.
- **MSR_IA32_PERF_CAPABILITIES advertised mask filtered** — defense against guest detecting PEBS feature beyond what KVM supports.
- **Per-VM PEBS-buffer accounting** — defense against VM with malicious PEBS-buffer-config consuming host memory.

