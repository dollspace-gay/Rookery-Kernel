# Tier-3: arch/x86/kvm/vmx/pmu_intel.c — KVM Intel PMU virtualization

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/x86-pmu.md
upstream-paths:
  - arch/x86/kvm/vmx/pmu_intel.c (~850 lines)
  - arch/x86/kvm/pmu.c
  - arch/x86/kvm/pmu.h
  - arch/x86/include/asm/perf_event.h
  - arch/x86/include/asm/msr-index.h (MSR_IA32_PERF*, MSR_CORE_PERF_*)
-->

## Summary

Intel Architectural-PMU (SDM-Vol3 §20) virtualization: per-vCPU N general-purpose perf-counters (IA32_PMC0..N-1) + per-counter event-select MSR (IA32_PERFEVTSEL0..N-1) + 3 fixed-function counters (IA32_FIXED_CTR0..2 + CTR_CTRL) + global control MSRs (IA32_PERF_GLOBAL_CTRL / GLOBAL_STATUS / GLOBAL_OVF_CTRL / GLOBAL_INUSE) + per-counter PEBS + LBR + Top-Down-Counters. KVM hosts each vCPU's counters by host-side perf_event objects, reprogramming on guest WRMSR. Per-VM-EXIT_LOAD_IA32_PERF_GLOBAL_CTRL auto-load. Critical for: profiling guest workloads (VTune / perf); guest-side `perf record`.

This Tier-3 covers `pmu_intel.c` (~850 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `intel_pmu_init()` | per-vCPU PMU init | `IntelPmu::init` |
| `intel_pmu_refresh()` | per-vCPU re-detect cap | `IntelPmu::refresh` |
| `intel_pmu_get_msr()` | RDMSR handler | `IntelPmu::get_msr` |
| `intel_pmu_set_msr()` | WRMSR handler | `IntelPmu::set_msr` |
| `reprogram_counter()` | per-counter perf_event reprogram | `Pmu::reprogram_counter` |
| `MSR_IA32_PERF_GLOBAL_CTRL` (0x38F) | per-vCPU global enable bitmap | UAPI |
| `MSR_IA32_PERF_GLOBAL_STATUS` (0x38E) | per-vCPU overflow-status RO | UAPI |
| `MSR_IA32_PERF_GLOBAL_OVF_CTRL` (0x390) | per-vCPU overflow-clear WO | UAPI |
| `MSR_IA32_FIXED_CTR_CTRL` (0x38D) | per-vCPU fixed-counter ctrl | UAPI |
| `MSR_IA32_FIXED_CTR0..2` (0x309..0x30B) | per-vCPU fixed counters | UAPI |
| `MSR_IA32_PERFCTR0..N` (0xC1..) | per-vCPU GP counters | UAPI |
| `MSR_IA32_EVNTSEL0..N` (0x186..) | per-vCPU event-select | UAPI |
| `intel_pmu_handle_lbr_msrs_access()` | per-vCPU LBR MSR | `IntelPmu::handle_lbr_msrs` |
| `vcpu->arch.pmu` | per-vCPU PMU state | `Vcpu::arch::pmu` |
| `struct kvm_pmu` | per-vCPU PMU shadow | `KvmPmu` |
| `struct kvm_pmc` | per-counter shadow | `KvmPmc` |

## Compatibility contract

REQ-1: CPUID 0x0A architectural-PMU leaf:
- EAX[7..0] PMU version (1, 2, 3, 4, 5).
- EAX[15..8] number-of-GP-counters.
- EAX[23..16] GP-counter bit-width.
- EAX[31..24] EBX-events-bitmap-length.
- EBX[7..0] events-not-available-bitmap.
- EDX[4..0] number-of-fixed-counters.
- EDX[12..5] fixed-counter bit-width.

REQ-2: Per-vCPU GP counters:
- IA32_PMC0..N-1 (typically N = 4 for v0; up to 8 modern).
- Per-counter: 48-bit width.
- Per-counter event-select (IA32_PERFEVTSEL0..N-1):
  - Bits[7:0] event-select.
  - Bits[15:8] umask.
  - Bit[16] USR.
  - Bit[17] OS.
  - Bit[18] EDGE.
  - Bit[19] PC (pin-control; not virtualized).
  - Bit[20] INT (APIC-interrupt-on-overflow).
  - Bit[21] ANY-thread (HT; not virtualized).
  - Bit[22] EN (enable).
  - Bit[23] INV (invert).
  - Bits[31:24] CMASK.

REQ-3: Per-vCPU fixed counters:
- 3 fixed: CTR0=INST_RETIRED, CTR1=CPU_CLK_UNHALTED.CORE, CTR2=CPU_CLK_UNHALTED.REF.
- Per-counter 48-bit.
- IA32_FIXED_CTR_CTRL: per-counter 4-bit control nibble [3:0]/[7:4]/[11:8]:
  - Bits[1:0] ENABLE-ring (0=disable, 1=OS, 2=USER, 3=BOTH).
  - Bit[2] ANY-thread.
  - Bit[3] PMI-on-overflow.

REQ-4: IA32_PERF_GLOBAL_CTRL (0x38F):
- Per-bit enable per-counter:
  - Bits[0..N-1] GP counter N enable.
  - Bits[32..34] fixed counter 0..2 enable.
- Per-WRMSR: triggers reprogram_counters on diff.

REQ-5: IA32_PERF_GLOBAL_STATUS (0x38E) RO:
- Per-bit overflow-set per-counter (same layout as GLOBAL_CTRL).
- Bits[55..58] events: Top-Down / LBR-Frozen / etc.
- Bit[62] Overflow-Buffer (PEBS).
- Bit[63] CONFIG-CHANGE.

REQ-6: IA32_PERF_GLOBAL_OVF_CTRL (0x390) WO:
- Per-WRMSR clears bits in GLOBAL_STATUS.

REQ-7: PMI (Performance Monitor Interrupt):
- Per-counter overflow + INT-bit set: deliver LVT_PERF_VECTOR via local APIC.
- Per-vCPU if EN+INT: kvm_pmu_deliver_pmi(vcpu).

REQ-8: VMCS auto-load IA32_PERF_GLOBAL_CTRL:
- VM_EXIT_LOAD_IA32_PERF_GLOBAL_CTRL: vmcs.host_ia32_perf_global_ctrl loaded on vmexit.
- VM_ENTRY_LOAD_IA32_PERF_GLOBAL_CTRL: vmcs.guest_ia32_perf_global_ctrl loaded on vmenter.

REQ-9: WRMSR(IA32_PERFCTRn, count):
- Update vcpu.pmu.gp_counters[n].counter.
- If counter armed: re-set host perf_event sample_period.

REQ-10: WRMSR(IA32_EVNTSELn, evtsel):
- Update vcpu.pmu.gp_counters[n].eventsel.
- If GLOBAL_CTRL bit n set: reprogram_counter to use new event.

REQ-11: WRMSR(IA32_PERF_GLOBAL_CTRL, mask):
- Per-bit toggled: reprogram_counters(pmu, mask_diff).
- Per-counter: arm/disarm host perf_event.

REQ-12: WRMSR(IA32_FIXED_CTR_CTRL):
- Per-fixed-counter nibble: arm/disarm host perf_event with appropriate event.

REQ-13: LBR (Last-Branch-Record):
- Per-vCPU LBR_SELECT + LBR_FROM_n + LBR_TO_n + LBR_INFO_n + LBR_TOS + LBR_DEPTH MSRs.
- Per-LBR-enable: VM_ENTRY_LOAD_DEBUG_CTL ∨ guest WRMSR(MSR_IA32_DEBUGCTLMSR.LBR=1).
- Per-vCPU LBR access via passthrough or intercept.

REQ-14: PEBS (Precise-Event-Based-Sampling):
- Per-vCPU MSR_IA32_PEBS_ENABLE bitmap.
- DS_BUFFER_BASE / INDEX / MAX / THRESHOLD MSRs.
- v0: PEBS NOT supported; intercepted return -EOPNOTSUPP.

REQ-15: Per-vCPU CPUID gating:
- CPUID 0x0A:EAX[7..0] = PMU version (≥ 1 advertises basic).
- Guest-PMU enabled iff host-PMU + per-VM not disabled-by-userspace (`-no-pmu`).

## Acceptance Criteria

- [ ] AC-1: Boot Linux guest: `dmesg | grep -i pmu` shows "Architectural PMU".
- [ ] AC-2: Guest `perf list`: shows architectural events.
- [ ] AC-3: Guest `perf stat -e cycles,instructions sleep 1`: counts non-zero.
- [ ] AC-4: Guest WRMSR(IA32_PERF_GLOBAL_CTRL, 0x07): host perf_event armed for counters 0..2.
- [ ] AC-5: Guest WRMSR(IA32_EVNTSEL0, INST_RETIRED.ANY | EN | USR): counter increments on user-mode insns.
- [ ] AC-6: Counter overflow + INT bit: PMI delivered via local-APIC.
- [ ] AC-7: WRMSR(IA32_PERF_GLOBAL_OVF_CTRL, mask): clears bits in GLOBAL_STATUS.
- [ ] AC-8: vmcs.entry-controls VM_ENTRY_LOAD_IA32_PERF_GLOBAL_CTRL set: counter resumes on vmenter.
- [ ] AC-9: Guest fixed-counter 0 (instructions retired) increments correctly.
- [ ] AC-10: Per-VM `-no-pmu` (KVM_PMU_CAP_DISABLE): RDMSR(IA32_PERFCTR0) → #GP.
- [ ] AC-11: Live migration: per-vCPU PMC counter values preserved.
- [ ] AC-12: kvm-unit-tests `pmu` test passes.

## Architecture

Per-vCPU PMU state:

```
struct KvmPmu {
  version: u8,                                   // CPUID 0x0A:EAX[7:0]
  nr_arch_gp_counters: u8,                       // CPUID 0x0A:EAX[15:8]
  nr_arch_fixed_counters: u8,                    // CPUID 0x0A:EDX[4:0]
  counter_bitmask: [u64; 2],                     // [0]=gp, [1]=fixed; 48-bit
  fixed_ctr_ctrl: u64,
  global_ctrl: u64,
  global_status: u64,                            // RO
  counter_bitmask_for_global: u64,               // valid bits in GLOBAL_CTRL
  reprogram_counter_pending: AtomicU64,          // bit per pending reprogram
  global_status_mask: u64,                       // valid bits
  available_event_types: u8,                     // CPUID 0x0A:EBX bitmap
  gp_counters: [KvmPmc; KVM_INTEL_PMC_MAX_GENERIC],
  fixed_counters: [KvmPmc; KVM_PMC_MAX_FIXED],
  ds_area: u64,                                  // PEBS DS area
  pebs_enable: u64,
  pebs_data_cfg: u64,
  lbr: KvmPmuLbr,
}

struct KvmPmc {
  type_: PmcType,                                // GP or FIXED
  idx: u8,
  counter: u64,                                  // 48-bit
  prev_counter: u64,
  eventsel: u64,                                 // for GP
  pebs: bool,
  current_config: u64,
  is_paused: bool,
  intr: bool,
  perf_event: Option<HostPerfEvent>,             // host-side perf_event object
  vcpu: &Vcpu,
}
```

`IntelPmu::init(vcpu)`:
1. pmu = vcpu.arch.pmu.
2. pmu.version = 0; cleared on cpuid_update.

`IntelPmu::refresh(vcpu)`:
1. cpuid_0x0a = guest_cpuid(0x0a).
2. pmu.version = cpuid_0x0a.eax & 0xff.
3. pmu.nr_arch_gp_counters = (cpuid_0x0a.eax >> 8) & 0xff.
4. pmu.nr_arch_fixed_counters = cpuid_0x0a.edx & 0x1f.
5. pmu.counter_bitmask[0] = (1 << ((cpuid_0x0a.eax >> 16) & 0xff)) - 1.  // GP
6. pmu.counter_bitmask[1] = (1 << ((cpuid_0x0a.edx >> 5) & 0xff)) - 1.   // fixed
7. pmu.global_ctrl_mask = ((1 << pmu.nr_arch_gp_counters) - 1) | (((1 << pmu.nr_arch_fixed_counters) - 1) << 32).
8. Allocate gp_counters[0..N], fixed_counters[0..M].

`IntelPmu::get_msr(vcpu, msr_info)`:
1. switch msr_info.index:
   - MSR_IA32_PERF_GLOBAL_CTRL: msr_info.data = pmu.global_ctrl.
   - MSR_IA32_PERF_GLOBAL_STATUS: msr_info.data = pmu.global_status.
   - MSR_IA32_FIXED_CTR_CTRL: msr_info.data = pmu.fixed_ctr_ctrl.
   - MSR_IA32_FIXED_CTR0..2: pmc = pmu.fixed_counters[msr - 0x309]; msr_info.data = pmc.counter.
   - MSR_IA32_PERFCTR0..N: pmc = pmu.gp_counters[msr - MSR_IA32_PERFCTR0]; msr_info.data = pmc.counter.
   - MSR_IA32_EVNTSEL0..N: pmc = pmu.gp_counters[msr - MSR_IA32_EVNTSEL0]; msr_info.data = pmc.eventsel.
   - LBR MSR range: dispatch to handle_lbr_msrs(true).

`IntelPmu::set_msr(vcpu, msr_info)`:
1. switch msr_info.index:
   - MSR_IA32_PERF_GLOBAL_CTRL:
     - data = msr_info.data & pmu.global_ctrl_mask.
     - diff = pmu.global_ctrl ^ data.
     - pmu.global_ctrl = data.
     - reprogram_counters(pmu, diff).
   - MSR_IA32_PERF_GLOBAL_OVF_CTRL: pmu.global_status &= ~msr_info.data.
   - MSR_IA32_FIXED_CTR_CTRL: pmu.fixed_ctr_ctrl = data; reprogram_fixed_counters(pmu, diff).
   - IA32_PERFCTRn: pmu.gp_counters[n].counter = data & counter_bitmask[0]; if armed: reprogram_counter.
   - IA32_EVNTSELn: pmu.gp_counters[n].eventsel = data; if GLOBAL_CTRL.n: reprogram_counter.
   - LBR MSR range: dispatch to handle_lbr_msrs(false).

`Pmu::reprogram_counter(pmc)`:
1. Stop existing pmc.perf_event if any.
2. Build perf_event_attr from pmc.eventsel + counter_bitmask:
   - .config = eventsel & 0xff_ff_ff (event + umask + cmask).
   - .exclude_user = !(eventsel & USR).
   - .exclude_kernel = !(eventsel & OS).
   - .sample_period = counter_bitmask - pmc.counter.
3. host pmc.perf_event = perf_event_create_kernel_counter(attr, host_cpu, vcpu_thread, &on_overflow_callback, pmc).
4. perf_event_enable(pmc.perf_event).

`Pmu::on_overflow(pmc)`:
1. pmu.global_status |= (1 << pmc.idx).
2. if pmc.intr: kvm_pmu_deliver_pmi(pmc.vcpu).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `nr_gp_counters_le_max` | INVARIANT | pmu.nr_arch_gp_counters ≤ KVM_INTEL_PMC_MAX_GENERIC. |
| `nr_fixed_counters_le_max` | INVARIANT | pmu.nr_arch_fixed_counters ≤ KVM_PMC_MAX_FIXED. |
| `global_ctrl_within_mask` | INVARIANT | per-WRMSR: pmu.global_ctrl & ~global_ctrl_mask == 0. |
| `counter_within_bitmask` | INVARIANT | per-pmc.counter & ~counter_bitmask[type] == 0. |
| `pebs_disabled_v0` | INVARIANT | v0: pebs_enable == 0 always. |
| `perf_event_lifetime_bounded` | INVARIANT | pmc.perf_event lifetime ≤ vCPU lifetime. |

### Layer 2: TLA+

`virt/kvm/pmu_intel.tla`:
- Per-vCPU PMU MSR sequence + per-counter reprogram + overflow.
- Properties:
  - `safety_global_ctrl_diff_triggers_reprogram` — per-WRMSR(GLOBAL_CTRL) + diff bit set ⟹ reprogram pmc.
  - `safety_overflow_sets_global_status` — per-pmc overflow ⟹ global_status bit set.
  - `safety_pmi_delivered_iff_intr` — per-overflow: PMI delivered iff EN + INT.
  - `liveness_overflow_eventually_pmi` — per-overflow + INT ⟹ PMI eventually delivered.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `IntelPmu::set_msr` post: pmu state updated; reprogram_counter called for diff bits | `IntelPmu::set_msr` |
| `IntelPmu::get_msr` post: returned data == per-MSR shadow | `IntelPmu::get_msr` |
| `Pmu::reprogram_counter` post: pmc.perf_event programmed with eventsel | `Pmu::reprogram_counter` |
| `Pmu::on_overflow` post: global_status bit set; PMI iff intr | `Pmu::on_overflow` |

### Layer 4: Verus/Creusot functional

`Per-WRMSR(EVNTSEL+GLOBAL_CTRL) → host perf_event armed → guest RDMSR(PMC) returns count consistent with event` semantic equivalence: per-architectural-event matches Intel SDM §20.

## Hardening

(Inherits row-1 features from `virt/kvm/x86-pmu.md` § Hardening.)

PMU-Intel-specific reinforcement:

- **Per-vCPU CPUID 0x0A bounded** — defense against guest faking unsupported PMU version.
- **Per-EVNTSEL EN-bit gates host perf_event** — defense against runaway perf_event with no enable.
- **Per-counter overflow → host perf_event sample_period reset** — defense against per-overflow runaway.
- **PEBS disabled in v0** — defense against complex PEBS attack-surface.
- **Per-vCPU LBR per-MSR validated** — defense against guest accessing LBR with no host-LBR support.
- **Per-vCPU PMU per-VM optional via `-no-pmu`** — defense against per-VM unwanted profiling.
- **Per-counter perf_event scoped to vCPU thread** — defense against host-perf-event leakage across vCPUs.
- **Per-PMI delivered via lapic with rate-limit** — defense against PMI-storm DoS.
- **Per-vCPU PERF_GLOBAL_CTRL bits ∈ global_ctrl_mask** — defense against guest setting non-existent counter.
- **Per-vCPU pmc.counter masked to bitmask** — defense against >48-bit counter values.
- **Per-vCPU LBR access intercepted unless host-LBR available** — defense against guest reading uninitialized MSR.
- **Per-vCPU live-migrate pmu state via KVM_GET_MSRS** — defense against post-migrate counter desync.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- KVM PMU core (covered in `virt/kvm/x86-pmu.md` Tier-3)
- AMD PMU (covered in `virt/kvm/x86-pmu-amd.md` Tier-3)
- KVM LBR detail (covered in `virt/kvm/x86-vmx-lbr.md` Tier-3)
- KVM PEBS detail (covered in `virt/kvm/x86-vmx-pebs.md` Tier-3)
- Host perf subsystem (kernel/events/; covered separately)
- Implementation code
