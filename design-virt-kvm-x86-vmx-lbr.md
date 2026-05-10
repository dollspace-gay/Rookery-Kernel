---
title: "Tier-3: arch/x86/kvm/vmx/pmu_intel.c (LBR subset) — Intel LBR (Last-Branch-Record) virtualization"
tags: ["tier-3", "virt-kvm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Intel LBR (Last-Branch-Record) is a CPU feature that records the last 8/16/32 branch-from/to addresses + branch-info into per-CPU MSRs (MSR_LBR_*_FROM, _TO, _INFO). KVM virtualizes LBR for guests so guest perf can use call-graph sampling: per-vCPU `lbr_desc` snapshots LBR-MSR values; on vmenter/vmexit transitions, LBR-MSR-pass-through dynamically toggled (MSR-bitmap configured to allow direct guest RDMSR access while LBR-active). Per-vCPU LBR state preserved across vmexits via auto-load/auto-store MSR lists. Critical for: cloud-perf-tooling using `perf record -e cycles --call-graph=lbr`, hot-spot identification with branch-trace.

This Tier-3 covers LBR-related code in `arch/x86/kvm/vmx/pmu_intel.c` (~200-300 lines).

### Acceptance Criteria

- [ ] AC-1: KVM_FEATURE_LBR advertised: guest CPUID + MSR_IA32_PERF_CAPABILITIES LBR-format bits set.
- [ ] AC-2: Boot Linux guest with `perf record -e cycles --call-graph=lbr`: LBR active; call-graph captured.
- [ ] AC-3: Per-record fields: FROM/TO/INFO populated correctly; matches expected branch addresses.
- [ ] AC-4: MSR-bitmap pass-through: guest RDMSR(MSR_LBR_TOS) ≤ 100 cycles (no vmexit when active).
- [ ] AC-5: Live migration: in-flight LBR-active perf record preserved.
- [ ] AC-6: Multi-vCPU: 4-vCPU guest each with independent LBR; per-vCPU records distinct.
- [ ] AC-7: Disable test: guest WRMSR(MSR_IA32_DEBUGCTLMSR, 0) clears LBR; subsequent vmexit removes pass-through.
- [ ] AC-8: kvm-unit-tests `lbr` test passes.
- [ ] AC-9: Host stress: host running its own LBR-perf concurrently; per-host vs per-VM LBR isolated.

### Architecture

`LbrDesc` per-vCPU:

```
struct LbrDesc {
  records: X86PmuLbr,                          // host LBR caps snapshot
  event: Option<KArc<PerfEvent>>,
  msr_passthrough: bool,
}

struct X86PmuLbr {
  nr: u32,                                       // 8/16/32
  info_msr: u32,                                  // MSR_LBR_INFO_0 base
}
```

`IntelLbr::msr_setup(vcpu)`:
1. desc := vcpu_to_lbr_desc(vcpu).
2. desc.records := host LBR caps (from x86_pmu).
3. msr_passthrough := false (initial).

`IntelLbr::create_guest_event(vcpu)`:
1. attr.type = PERF_TYPE_RAW.
2. attr.config = LBR-only-sampling-config.
3. attr.exclude_kernel = 0 (guest can sample its own kernel).
4. event := perf_event_create_kernel_counter(...).
5. desc.event = event.

`VcpuVmx::enable_lbr(vmx)`:
1. desc := vcpu_to_lbr_desc(vcpu).
2. If !desc.event: IntelLbr::create_guest_event.
3. For each LBR MSR: vmx_set_intercept_for_msr(MSR_BITMAP, msr, READ | WRITE, false). (allow direct access)
4. desc.msr_passthrough = true.

`VcpuVmx::disable_lbr(vmx)`:
1. For each LBR MSR: vmx_set_intercept_for_msr(MSR_BITMAP, msr, READ | WRITE, true). (force vmexit)
2. desc.msr_passthrough = false.

`Vcpu::handle_msr` MSR_IA32_DEBUGCTLMSR branch:
1. If write && data & DEBUGCTLMSR_LBR: vmx_enable_lbr.
2. Else if write && !(data & DEBUGCTLMSR_LBR): vmx_disable_lbr.

### Out of Scope

- KVM PMU core (covered in `x86-pmu.md` Tier-3)
- VMX vendor (covered in `x86-vmx.md` Tier-3)
- Intel PEBS (covered in `x86-vmx-pebs.md` Tier-3)
- AMD IBS (covered in `x86-svm-ibs.md` future Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct lbr_desc` | per-vCPU LBR state | `kernel::kvm::x86::vmx::lbr::LbrDesc` |
| `struct x86_pmu_lbr` | host LBR caps (nr, info) | `X86PmuLbr` |
| `vcpu_to_lbr_desc(vcpu)` | per-vCPU lookup | `Vcpu::lbr_desc` |
| `vcpu_to_lbr_records(vcpu)` | per-vCPU records cap | `Vcpu::lbr_records` |
| `intel_pmu_lbr_is_compatible(vcpu)` | compat-check at vCPU init | `IntelLbr::is_compatible` |
| `intel_pmu_lbr_is_enabled(vcpu)` | per-vCPU enabled query | `IntelLbr::is_enabled` |
| `intel_pmu_is_valid_lbr_msr(vcpu, idx)` | per-MSR-idx validation | `IntelLbr::is_valid_lbr_msr` |
| `intel_pmu_lbr_msr_setup(vcpu)` | per-vCPU LBR-MSR setup | `IntelLbr::msr_setup` |
| `intel_pmu_create_guest_lbr_event(vcpu)` | create host-side perf-event for LBR | `IntelLbr::create_guest_event` |
| `intel_pmu_release_guest_lbr_event(vcpu)` | release | `IntelLbr::release_guest_event` |
| `intel_pmu_lbr_event(vcpu)` | per-vCPU LBR perf_event | `IntelLbr::event` |
| `vmx_disable_lbr(vmx)` | disable LBR pass-through | `VcpuVmx::disable_lbr` |
| `vmx_enable_lbr(vmx)` | enable LBR pass-through | `VcpuVmx::enable_lbr` |
| `vmx_lbr_msrs(vmx)` | per-vCPU MSR-bitmap config | `VcpuVmx::lbr_msrs` |

### compatibility contract

REQ-1: LBR-feature gating:
- Host LBR-supported: per-arch CPUID + host perf core.
- Guest sees LBR via per-vCPU CPUID + MSR_IA32_PERF_CAPABILITIES.

REQ-2: Per-vCPU `lbr_desc`:
- `records` (x86_pmu_lbr struct: nr, info_msr).
- `event` (host-side perf_event with LBR-enabled type).
- `msr_passthrough` (MSR-bitmap state; whether guest can directly RDMSR LBR MSRs).

REQ-3: LBR MSRs:
- MSR_LBR_TOS / MSR_LASTBRANCH_TOS (0x1c9): top-of-stack pointer.
- MSR_LBR_SELECT (0x1c8): select-mask.
- MSR_LBR_FROM_0..N (0x680..) / MSR_LBR_INFO_0..N (0xdc0..) / MSR_LBR_TO_0..N: per-record entries.
- N depends on CPU: typically 8 (Sandy Bridge), 16 (Haswell), 32 (Skylake+).

REQ-4: Per-vCPU LBR-MSR setup:
1. intel_pmu_lbr_is_compatible(vcpu) — verify guest CPUID matches host LBR cap.
2. records.nr := host LBR record count.
3. Allocate per-vCPU lbr_desc.
4. perf_event_create_kernel_counter for LBR-watchdog event.

REQ-5: MSR-bitmap pass-through:
- When guest enables LBR via MSR_IA32_DEBUGCTLMSR.LBR=1:
  - vmx_enable_lbr: allow direct RDMSR/WRMSR for LBR_FROM/_TO/_INFO/_TOS in MSR-bitmap.
- When guest disables: vmx_disable_lbr.
- Reduces vmexit cost for LBR-heavy perf record.

REQ-6: VM-entry MSR auto-load:
- Per-vCPU MSR_IA32_DEBUGCTLMSR added to VM-entry-MSR-load-list.
- Ensures guest sees own DEBUGCTL on vmenter.

REQ-7: VM-exit MSR auto-save:
- Per-vCPU LBR MSRs added to VM-exit-MSR-save-list (when active).
- KVM saves guest LBR state on vmexit; restores on vmenter.

REQ-8: Per-vCPU lbr_event lifecycle:
- intel_pmu_create_guest_lbr_event: host-side perf-event with LBR-only sampling-mode.
- Holds host-LBR-MSRs from being overwritten by other host workload.
- intel_pmu_release_guest_lbr_event: destroy on vCPU destroy / disable.

REQ-9: Per-vCPU LBR record format:
- 8/16/32 entries × (FROM, TO, INFO) = 24/48/96 MSRs.
- Per-entry FROM/TO contains last-branch addresses.
- INFO contains TSX-bits, mispredict, cycles.

REQ-10: Live migration:
- Per-vCPU LBR MSRs migrated.
- lbr_desc recreated at destination vmenter.

REQ-11: Per-VM gating via KVM_CAP_PMU_CAPABILITY:
- Userspace can disable LBR for security-sensitive VMs.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `lbr_msr_idx_bounded` | OOB | per-LBR-MSR access bounded by records.nr; defense against guest accessing OOB LBR MSR. |
| `passthrough_state_consistent` | INVARIANT | msr_passthrough flag matches MSR-bitmap state. |
| `event_lifecycle_paired` | UAF | per-vCPU event created/released paired with enable/disable. |
| `lbr_records_match_host` | INVARIANT | guest-visible records.nr ≤ host LBR records (intersection). |

### Layer 2: TLA+

`virt/kvm/lbr_passthrough.tla`:
- Per-vCPU LBR state ∈ {Disabled, Enabled(passthrough), Migrating}.
- Properties:
  - `safety_passthrough_iff_enabled` — MSR pass-through only in Enabled state.
  - `safety_no_record_leak` — host LBR records not exposed to guest beyond what guest LBR-mode allows.
  - `liveness_disable_eventually_intercepts` — Enabled → Disabled transition results in intercept-restored.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `IntelLbr::msr_setup` post: records populated; lbr_desc allocated | `IntelLbr::msr_setup` |
| `VcpuVmx::enable_lbr` post: MSR-bitmap allows direct LBR access; event created | `VcpuVmx::enable_lbr` |
| `VcpuVmx::disable_lbr` post: MSR-bitmap forces intercept; event released | `VcpuVmx::disable_lbr` |
| Per-vCPU lbr_desc.event nonexistent when msr_passthrough == false | invariants on enable/disable |

### Layer 4: Verus/Creusot functional

`Per-LBR-active vCPU: guest LBR_FROM/_TO/_INFO/_TOS reads return values reflecting last 8/16/32 guest branches` semantic equivalence: per-vCPU the LBR-MSR values seen by guest match what bare-metal LBR would record for the same guest-instructions.

### hardening

(Inherits row-1 features from `virt/kvm/x86-pmu.md` § Hardening.)

LBR-specific reinforcement:

- **Per-vCPU LBR records.nr ≤ host max** — defense against guest claiming more records than exist.
- **MSR pass-through tied to DEBUGCTLMSR.LBR** — defense against guest direct-access without enabling.
- **Per-vCPU lbr_event held across vmexits** — defense against host-LBR-state overwrite by other host workload.
- **MSR-bitmap atomic update under vcpu.lock** — defense against torn pass-through during enable/disable race.
- **Per-VM KVM_CAP_PMU_CAPABILITY gating** — defense against LBR enabled on confidential VMs.
- **DEBUGCTLMSR.LBR transitions checked at WRMSR** — defense against guest enabling LBR without proper setup.
- **Per-LBR-MSR validation at intercepted access** — defense against guest accessing reserved LBR MSR.
- **Per-event PERF_TYPE_RAW config validated** — defense against guest tricks via attr.config.
- **Live-migrate clears lbr_desc.event** — defense against post-migrate stale host-event.
- **Per-vCPU LBR data not leaked across vCPU migrate** — defense against cross-vCPU branch-record leak.

