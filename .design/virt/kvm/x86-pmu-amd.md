# Tier-3: arch/x86/kvm/svm/pmu.c — AMD PMU vendor backend (perfmon-v1 + perfmon-v2 + AMD-specific PMC MSRs)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/x86-pmu.md
upstream-paths:
  - arch/x86/kvm/svm/pmu.c
  - arch/x86/kvm/svm/svm.h (PMU fields)
  - arch/x86/include/uapi/asm/msr-index.h (MSR_K7_*, MSR_F15H_*, MSR_AMD64_PERF_*)
-->

## Summary

AMD PMU vendor-half is the per-vendor backend for KVM's arch-agnostic PMU virtualization (covered in `x86-pmu.md`). AMD has two PMU revisions: legacy perfmon-v1 (Family 0x10+; per-counter MSR_K7_EVNTSELn/_PERFCTRn at 0xC001_0000+) and perfmon-v2 (Family 0x19+; adds MSR_AMD64_PERF_CNTR_GLOBAL_CTL/_STATUS, harmonizing with Intel arch-perfmon). svm/pmu.c provides amd_pmu_ops vtable that hooks into pmu.c arch-agnostic core. AMD has no fixed counters (all GP); no PEBS (uses IBS instead); has perfmon-v2 introduced with Zen3.

This Tier-3 covers `arch/x86/kvm/svm/pmu.c` (~286 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `amd_pmu_ops` | per-vendor ops vtable | `kernel::kvm::x86::svm::pmu::AMD_PMU_OPS` |
| `amd_pmu_init(vcpu)` | per-vCPU init | `AmdPmu::init` |
| `amd_pmu_set_msr(vcpu, msr_info)` | guest WRMSR | `AmdPmu::set_msr` |
| `amd_pmu_get_msr(vcpu, msr_info)` | guest RDMSR | `AmdPmu::get_msr` |
| `amd_pmu_refresh(vcpu)` | per-CPUID-change refresh | `AmdPmu::refresh` |
| `amd_pmu_reset(vcpu)` | per-vCPU reset | `AmdPmu::reset` |
| `amd_pmu_handle_event(vcpu)` | per-event service | `AmdPmu::handle_event` |
| `amd_pmu_msr_idx_to_pmc(vcpu, idx, pmc_idx)` | MSR-idx → PMC | `AmdPmu::msr_idx_to_pmc` |
| `amd_is_valid_msr(vcpu, idx)` | per-MSR validity | `AmdPmu::is_valid_msr` |
| `amd_is_valid_rdpmc_ecx(vcpu, idx)` | per-rdpmc validity | `AmdPmu::is_valid_rdpmc_ecx` |
| `amd_pmu_save_pmu_context(vcpu)` | per-vmexit save | `AmdPmu::save_pmu_context` |
| `amd_pmu_restore_pmu_context(vcpu)` | per-vmenter restore | `AmdPmu::restore_pmu_context` |
| `MSR_K7_EVNTSEL0..3` (0xC001_0000..) | legacy event-select MSRs | UAPI |
| `MSR_K7_PERFCTR0..3` (0xC001_0004..) | legacy counter MSRs | UAPI |
| `MSR_F15H_PERF_CTL0..5` (0xC001_0200..) | Family 15h+ event-select | UAPI |
| `MSR_F15H_PERF_CTR0..5` (0xC001_0201..) | Family 15h+ counter | UAPI |
| `MSR_AMD64_PERF_CNTR_GLOBAL_CTL` (0xC000_0301) | perfmon-v2 global ctrl | UAPI |
| `MSR_AMD64_PERF_CNTR_GLOBAL_STATUS` (0xC000_0300) | perfmon-v2 global status | UAPI |
| `MSR_AMD64_PERF_CNTR_GLOBAL_STATUS_CLR` (0xC000_0302) | perfmon-v2 status clear | UAPI |
| `MSR_AMD64_PERF_CNTR_GLOBAL_STATUS_SET` (0xC000_0303) | perfmon-v2 status set | UAPI |

## Compatibility contract

REQ-1: Per-vendor PMU caps:
- AMD perfmon-v1: 4 GP counters; per-counter EVNTSELn/PERFCTRn pair.
- AMD perfmon-v2: 6 GP counters (Zen3+); adds GLOBAL_CTL/_STATUS.
- Family 15h+ extended: 6 counters via PERF_CTL0..5 / PERF_CTR0..5.

REQ-2: Per-vCPU PMU init (amd_pmu_init):
1. Read AMD CPUID 0x80000022:EBX (perfmon-v2 enumeration):
   - Bits[3:0]: NumPerfCtrCore (typically 6).
2. Allocate per-PMC kvm_pmc structures (no fixed counters).
3. Set per-vCPU global_ctrl, global_status MSR shadows.
4. Bind ops vtable.

REQ-3: Per-MSR write (amd_pmu_set_msr):
1. Switch on msr_idx:
   - MSR_K7_EVNTSEL0..3 / MSR_F15H_PERF_CTL0..5: pmc.eventsel = data; reprogram_counter.
   - MSR_K7_PERFCTR0..3 / MSR_F15H_PERF_CTR0..5: pmc.counter = data; perf_event update.
   - MSR_AMD64_PERF_CNTR_GLOBAL_CTL: pmu.global_ctrl = data; per-PMC enable/disable.
   - MSR_AMD64_PERF_CNTR_GLOBAL_STATUS_CLR: pmu.global_status &= ~data.
   - MSR_AMD64_PERF_CNTR_GLOBAL_STATUS_SET: pmu.global_status |= data.

REQ-4: Per-MSR read (amd_pmu_get_msr):
1. Switch on msr_idx similarly.
2. Return MSR shadow value.

REQ-5: Per-vmexit save_pmu_context:
1. Read host PMC MSRs that may have been modified by guest direct-access.
2. Save into per-vCPU shadows.

REQ-6: Per-vmenter restore_pmu_context:
1. Write guest PMC MSR shadows back to host MSRs.
2. Resume guest with PMU state.

REQ-7: PMC index ↔ MSR mapping:
- Family 15h+: MSR pair (CTL0+CTR0) → PMC[0]; (CTL1+CTR1) → PMC[1]; etc.
- Per-AMD spec: 4 pairs in legacy + 6 pairs on Family 15h+ + 6 on perfmon-v2.

REQ-8: AMD-specific event quirks:
- Some events have CPU-erratum requiring specific eventsel format.
- KVM filters per-erratum events via deny-list.

REQ-9: KVM_FEATURE_PERFMON CPUID:
- Advertised in CPUID 0x40000001:EAX.
- Independent of AMD CPUID 0x80000022.

REQ-10: Live migration:
- All PMC MSRs included in migration list.
- per-vCPU global_ctrl/_status migrated.

REQ-11: AMD-specific vmexit reasons (for PMU):
- VMEXIT_RDPMC: emulate per-PMC read.
- Legacy AMD doesn't have per-counter pass-through (no Intel-style virtualize_x2apic equivalent).

REQ-12: SVM AVIC + PMU:
- AVIC enabled VMs may have PMU intercept differences.
- PMU MSRs may pass through with AVIC active.

## Acceptance Criteria

- [ ] AC-1: Boot Linux guest under qemu+kvm with kvm-amd: guest sees AMD PMU via CPUID 0x80000022.
- [ ] AC-2: `perf stat -e cycles ls` inside guest: counter advances; non-zero count.
- [ ] AC-3: Per-counter PMI: enable counter with sample_period; PMI fires; guest perf-handler runs.
- [ ] AC-4: AMD perfmon-v2: 6 counters available; global_ctrl masks per-counter.
- [ ] AC-5: Live migration: AMD guest perf state preserved across migrate.
- [ ] AC-6: Multi-vCPU AMD guest: 4-vCPU `perf stat`; per-vCPU counters distinct.
- [ ] AC-7: kvm-unit-tests `pmu` test passes on AMD.
- [ ] AC-8: F15h+ extended counters accessed via MSR_F15H_PERF_CTL/CTR work correctly.
- [ ] AC-9: AMD-specific erratum-events filtered (denied via -EINVAL on guest WRMSR).
- [ ] AC-10: Spec-violation defense: WRMSR(MSR_K7_EVNTSELn) with reserved bit set raises #GP.

## Architecture

`AmdPmu` extends `kvm_pmu` (already in pmu.c base):

```
struct AmdPmu {
  base: KvmPmu,                                // from x86-pmu.md
  perfmon_v2: bool,                             // perfmon-v2 detected
  num_counters: u32,                            // typically 4 (legacy) or 6 (v2)
  global_ctrl_mask: u64,                        // per-counter bitmask
  global_status_mask: u64,
  counter_msr_base: u32,                        // MSR_K7_PERFCTR0 or MSR_F15H_PERF_CTR0
  evntsel_msr_base: u32,                        // MSR_K7_EVNTSEL0 or MSR_F15H_PERF_CTL0
}
```

`AmdPmu::init(vcpu)`:
1. pmu := &vcpu.arch.pmu.
2. Read AMD CPUID 0x80000022:EBX[3:0] → num_counters.
3. If pmu_v2_supported(vcpu): pmu.perfmon_v2 = true.
4. Set counter_msr_base based on Family15h+ vs legacy.
5. Set global_ctrl_mask = (1 << num_counters) - 1.
6. Initialize per-PMC structures.

`AmdPmu::set_msr(vcpu, msr_info)`:
1. msr_idx := msr_info.index.
2. data := msr_info.data.
3. Switch:
   - MSR_AMD64_PERF_CNTR_GLOBAL_CTL:
     - If !pmu.perfmon_v2: return -EINVAL.
     - If data & ~pmu.global_ctrl_mask: return -EINVAL.
     - diff := pmu.global_ctrl ^ data.
     - pmu.global_ctrl = data.
     - For each bit in diff: per-PMC reprogram.
   - MSR_AMD64_PERF_CNTR_GLOBAL_STATUS_CLR:
     - pmu.global_status &= ~data.
   - MSR_AMD64_PERF_CNTR_GLOBAL_STATUS_SET:
     - If host_initiated: pmu.global_status |= data.
     - Else: -EINVAL (read-only from guest perspective; only set by overflow).
   - MSR_K7_EVNTSEL0..3 / MSR_F15H_PERF_CTL0..5:
     - pmc_idx := (msr_idx - evntsel_msr_base) / 2 OR 1; depending on version.
     - pmu.gp_counters[pmc_idx].eventsel = data.
     - reprogram_counter(&pmu.gp_counters[pmc_idx]).
   - MSR_K7_PERFCTR0..3 / MSR_F15H_PERF_CTR0..5:
     - pmu.gp_counters[pmc_idx].counter = data.
     - perf_event_period_set.

`AmdPmu::msr_idx_to_pmc(vcpu, idx, pmc_idx)`:
1. If idx in [counter_msr_base, counter_msr_base + 2 * num_counters - 1]:
   - pmc_idx := (idx - counter_msr_base) / 2 (or 1, version-dependent).
2. Return pmu.gp_counters[pmc_idx].

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pmc_idx_bounded` | OOB | per-PMC idx < num_counters. |
| `global_ctrl_mask_match` | INVARIANT | data & ~global_ctrl_mask == 0 enforced at WRMSR. |
| `perfmon_v2_gating` | INVARIANT | global_ctrl/_status MSRs gated on perfmon_v2 detection. |
| `eventsel_validation` | INVARIANT | per-EVNTSEL reserved bits == 0 enforced. |
| `counter_msr_pair_match` | INVARIANT | per-PMC EVNTSEL + PERFCTR pair address-aligned correctly. |

### Layer 2: TLA+

(See `x86-pmu.md` Tier-3 for general PMU TLA+; AMD-specific quirks captured in invariants.)

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `AmdPmu::init` post: num_counters detected from CPUID; gp_counters allocated | `AmdPmu::init` |
| `AmdPmu::set_msr` post: per-MSR shadow updated; per-counter reprogrammed | `AmdPmu::set_msr` |
| `AmdPmu::msr_idx_to_pmc` post: returned pmc_idx < num_counters OR -EINVAL | `AmdPmu::msr_idx_to_pmc` |
| Per-vCPU global_ctrl bits ⊆ global_ctrl_mask | invariants on WRMSR |

### Layer 4: Verus/Creusot functional

`AMD guest perf record produces correct per-counter samples` semantic equivalence: per-event the host PMC counter increments visible in guest match what bare-metal AMD perfmon would emit.

## Hardening

(Inherits row-1 features from `virt/kvm/x86-pmu.md` § Hardening.)

AMD-PMU-specific reinforcement:

- **Per-perfmon version detection** — defense against guest using v2 features on v1 CPU.
- **Per-MSR reserved-bits enforced** — defense against guest WRMSR causing host CPU exception.
- **Per-counter eventsel deny-list** — defense against erratum-events causing host PMC malfunction.
- **Family15h+ vs legacy MSR base** — defense against MSR-overlap confusion.
- **GLOBAL_CTL per-counter bitmask** — defense against guest enabling more counters than exist.
- **GLOBAL_STATUS_SET host-only** — defense against guest spoofing overflow.
- **Per-vmexit save_pmu_context** — defense against guest direct-access leaking PMU state to host.
- **Live-migrate AMD PMC MSRs** — defense against post-migrate state inconsistency.
- **Per-CPUID-change refresh** — defense against stale per-counter caps after live-migrate.
- **AVIC interaction validated** — defense against AVIC-enabled VM leaking PMU state across vmexits.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- KVM PMU core (covered in `x86-pmu.md` Tier-3)
- Intel PMU (covered in `x86-vmx-pebs.md` + `x86-vmx-lbr.md` + `x86-pmu.md` Tier-3s)
- AMD IBS (Instruction-Based Sampling; AMD analog of PEBS; covered in `x86-svm-ibs.md` future Tier-3)
- SVM core (covered in `x86-svm.md` Tier-3)
- Implementation code
