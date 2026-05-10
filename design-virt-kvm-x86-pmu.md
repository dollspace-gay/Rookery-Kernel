---
title: "Tier-3: arch/x86/kvm/pmu.c — KVM architectural PMU virtualization (per-vCPU virtual PMCs + Intel/AMD vendor backends)"
tags: ["tier-3", "virt-kvm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Virtual PMU lets a guest use Intel architectural perfmon (or AMD perfmon-v2) inside a VM as if it had bare-metal access — guest reads MSR_IA32_PERF_GLOBAL_CTRL, programs PMC counters, takes per-counter overflow PMI. KVM's pmu.c is the arch-agnostic core (per-vCPU `kvm_pmu` structure + per-PMC virtualization + MSR-bitmap pass-through optimization); pmu_intel.c is the Intel vendor backend (architectural perfmon CPUID enumeration + Intel-specific MSRs); svm/pmu.c is AMD perfmon-v1 + perfmon-v2 backend. Each guest PMC is implemented via a host `perf_event` (gives KVM scheduler-aware multiplexing across guest+host counter usage).

This Tier-3 covers `pmu.c` (~1407 lines) + `pmu.h` (~270 lines) + `vmx/pmu_intel.c` (~850 lines) + `svm/pmu.c` (~286 lines).

### Acceptance Criteria

- [ ] AC-1: Boot Linux guest under qemu+kvm with `-cpu host`: `lscpu | grep -i pmu` shows perfmon enabled.
- [ ] AC-2: `perf stat -e instructions,cycles ls` inside guest: counters report nonzero per-instruction count.
- [ ] AC-3: Per-counter PMI: enable counter with sample_period=1000; after 1000 instructions PMI fires; guest perf-handler runs.
- [ ] AC-4: Live migration: guest perf_event state preserved (counter values, eventsel) across migration.
- [ ] AC-5: Multi-vCPU guest: 4-vCPU `perf stat` system-wide shows per-vCPU counters; no cross-vCPU contamination.
- [ ] AC-6: AMD host: same tests on Zen+ host with kvm-amd loaded; AMD perfmon-v2 features enumerated correctly.
- [ ] AC-7: kvm-unit-tests `pmu` test passes (Intel) + `pmu_lbr` (last-branch-record) where supported.
- [ ] AC-8: Guest RDPMC fast-path: passthrough enabled; guest RDPMC < 100 cycles (no vmexit).
- [ ] AC-9: PMU disable via CPUID: guest with CPUID.0x0A.EAX = 0 (no perfmon) cannot trigger any PMU-MSR accesses; vmexit-emulator returns #GP.

### Architecture

`Pmu` per-vCPU:

```
struct Pmu {
  vcpu: KArc<Vcpu>,
  version: u8,                                // arch-perfmon version
  nr_arch_gp_counters: u8,
  nr_arch_fixed_counters: u8,
  counter_bitmask: [u64; 2],                  // [GP, FIXED] counter masks
  available_event_types: u64,                 // arch-event bitmap (arch-perfmon v3)
  fixed_counters: [Pmc; INTEL_PMC_MAX_FIXED],
  gp_counters: [Pmc; INTEL_PMC_MAX_GENERIC],
  global_ctrl: u64,
  global_status: u64,
  global_ovf_ctrl: u64,
  fixed_ctr_ctrl: u64,
  event_count: u32,                            // rate-limit counter
  pmc_in_use: u64,                             // bitmap
  global_ctrl_mask: u64,
  ...
}

struct Pmc {
  idx: u8,
  counter: AtomicU64,
  eventsel: u64,
  perf_event: Option<KArc<PerfEvent>>,
  vcpu: KWeak<Vcpu>,
  current_config: u64,
  is_paused: bool,
  ...
}
```

`Pmu::init(vcpu)`:
1. Allocate per-vCPU pmu struct.
2. Set version/counter-counts from CPUID.0x0A (Intel) or CPUID.0x80000022 (AMD).
3. Init per-PMC structures with idx + back-references.
4. Vendor-specific init via `pmu_ops.init` (Intel: pmu_intel_init; AMD: pmu_amd_init).

`Pmu::set_msr(vcpu, msr_info)`:
1. Decode msr_info.index → resolve to which PMU MSR.
2. If global_ctrl: validate write; update shadow; per-PMC reprogram if enabled-bit changed.
3. If eventsel: per-PMC update + reprogram_counter.
4. If perfctr (counter MSR): write counter-value; perf_event_period_set.
5. Vendor-specific dispatch via `pmu_ops.set_msr`.

`Pmc::reprogram(pmc)`:
1. If perf_event exists: release_perf_event(pmc).
2. Decode pmc.eventsel → perf_event_attr (type=raw, config=lower-event-bits, sample_period=...).
3. perf_event_create_kernel_counter(...) on host CPU.
4. pmc.perf_event = result.
5. Set overflow callback to `kvm_perf_overflow`.

`__kvm_perf_overflow(perf_event)`:
1. pmc := perf_event.context (back-reference).
2. vcpu := pmc.vcpu (KWeak upgrade).
3. pmu.global_status |= (1 << pmc.idx).
4. kvm_make_request(KVM_REQ_PMI, vcpu) (deferred PMI inject).
5. kvm_vcpu_kick(vcpu).

`Pmu::deliver_pmi(vcpu)`:
1. Inject LVT_PMI via lapic.
2. Guest takes PMI on next vmenter.

`Pmu::handle_event(vcpu)` (ratelimited service hook):
1. event_count += 1.
2. If event_count > rate_limit: skip (overload protection).
3. For each pmc in pmc_in_use: pmc.counter = perf_event_read_value(pmc.perf_event).

### Out of Scope

- Intel PEBS specifics (covered in `virt/kvm/x86-pmu-pebs.md` future Tier-3)
- Intel LBR (Last-Branch-Record) (covered in `virt/kvm/x86-pmu-lbr.md` future Tier-3)
- AMD IBS (Instruction-Based Sampling) (covered in `virt/kvm/x86-pmu-ibs.md` future Tier-3)
- Per-event arch-perfmon enumeration tables (data tables; trivial)
- KVM core (covered in `kvm-core.md` Tier-3)
- VMX vendor (covered in `x86-vmx.md` Tier-3)
- SVM vendor (covered in `x86-svm.md` Tier-3)
- LAPIC PMI delivery (covered in `x86-lapic.md` Tier-3)
- Perf_event subsystem core (covered in `kernel/perf/perf-event-core.md` Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct kvm_pmu` | per-vCPU PMU state | `kernel::kvm::x86::pmu::Pmu` |
| `struct kvm_pmc` | per-PMC virtual counter | `Pmc` |
| `kvm_pmu_init(vcpu)` / `_destroy(vcpu)` | per-vCPU PMU lifecycle | `Pmu::init` / `_destroy` |
| `kvm_pmu_handle_event(vcpu)` | per-vCPU PMU service hook (ratelimited) | `Pmu::handle_event` |
| `kvm_pmu_get_msr(vcpu, msr_info)` | guest RDMSR(PMU MSR) handler | `Pmu::get_msr` |
| `kvm_pmu_set_msr(vcpu, msr_info)` | guest WRMSR(PMU MSR) handler | `Pmu::set_msr` |
| `kvm_pmu_refresh(vcpu)` | per-vCPU CPUID-change reflect | `Pmu::refresh` |
| `kvm_pmu_reset(vcpu)` | per-vCPU reset (boot / INIT) | `Pmu::reset` |
| `kvm_pmu_cleanup(vcpu)` | per-vCPU cleanup (release perf_events) | `Pmu::cleanup` |
| `kvm_pmu_deliver_pmi(vcpu)` | guest PMI inject | `Pmu::deliver_pmi` |
| `kvm_pmu_check_rdpmc_early(vcpu, idx)` | guest RDPMC validation | `Pmu::check_rdpmc_early` |
| `kvm_pmu_rdpmc(vcpu, idx, &val)` | RDPMC emulation | `Pmu::rdpmc` |
| `intel_pmu_msr_idx_to_pmc(...)` (vmx/pmu_intel.c) | Intel vendor MSR-idx-to-PMC | `IntelPmu::msr_idx_to_pmc` |
| `amd_pmu_get_msr(...)` (svm/pmu.c) | AMD vendor MSR get | `AmdPmu::get_msr` |
| `pmc_idx_to_pmc(pmu, idx)` | per-PMC index → struct kvm_pmc | `Pmu::idx_to_pmc` |
| `reprogram_counter(pmc)` | rebuild perf_event for PMC | `Pmc::reprogram` |
| `pmc_pause_counter(pmc)` / `_resume_counter(pmc)` | per-PMC perf_event pause/resume | `Pmc::pause` / `_resume` |

### compatibility contract

REQ-1: Per-vCPU `kvm_pmu`:
- `nr_arch_gp_counters` (typically 4 or 8 per Intel arch perfmon).
- `nr_arch_fixed_counters` (typically 3 fixed: instructions-retired, unhalted-cycles, unhalted-ref-cycles).
- `gp_counters[INTEL_PMC_MAX_GENERIC]` (per-PMC kvm_pmc).
- `fixed_counters[INTEL_PMC_MAX_FIXED]`.
- `global_ctrl` / `global_status` / `global_ovf_ctrl` (per-vCPU shadow of MSR_IA32_PERF_GLOBAL_*).
- `fixed_ctr_ctrl` (MSR_CORE_PERF_FIXED_CTR_CTRL shadow).
- `event_count` (per-vCPU rate-limit counter).
- `pmc_in_use` (bitmap of PMCs currently programmed).

REQ-2: Per-PMC `kvm_pmc`:
- `idx` (0..nr_gp + nr_fixed).
- `counter` (current 64-bit counter value).
- `eventsel` (event-select MSR shadow).
- `perf_event` (host perf_event backing this PMC).
- `vcpu` (back-reference).
- `current_config` (last-applied config).

REQ-3: Per-PMC perf_event backing:
- KVM allocates host-side perf_event when guest programs PMC eventsel.
- perf_event configured with: type=raw OR type=hardware, config=eventsel-bits, sample_period=initial-counter-value.
- Host perf_core schedules counter on/off as guest enters/exits.
- On overflow: perf_event callback delivers PMI to guest via `kvm_pmu_deliver_pmi`.

REQ-4: Per-vCPU MSR pass-through optimization:
- For PMC-MSRs guest reads frequently (e.g., RDPMC index 0), KVM can pass-through directly without vmexit if perf_event running.
- Pass-through controlled per-vCPU via msr_bitmap; vendor handles direct register access.

REQ-5: PMI delivery:
- Per-overflow: perf_event callback calls `__kvm_perf_overflow(perf_event)`.
- KVM marks per-vCPU `pmu.global_status |= 1 << pmc.idx`.
- KVM injects LVT_PMI as #PMI on next vmenter.
- Guest reads global_status to identify overflowed counter.

REQ-6: Guest RDPMC emulation:
- `kvm_pmu_rdpmc(vcpu, idx, val)`:
  - Validate idx (in-bound; PMC enabled in global_ctrl + counter not paused).
  - Resolve idx → kvm_pmc via vendor msr_idx_to_pmc.
  - val := pmc.counter (latest from perf_event_read).

REQ-7: Per-vCPU PMC count from CPUID:
- Intel CPUID.0x0A.EAX[15:8] (number of GP counters per LP).
- Intel CPUID.0x0A.EAX[23:16] (GP counter bit-width).
- AMD CPUID.0x80000022.EBX (perfmon-v2 enumeration).
- `kvm_pmu_refresh(vcpu)` re-reads CPUID and resizes per-vCPU pmu state.

REQ-8: Vendor-specific MSRs:
- Intel: MSR_IA32_PERFCTRn (n=0..7) for GP counters; MSR_CORE_PERF_FIXED_CTRn for fixed; MSR_IA32_PERF_GLOBAL_CTRL/_STATUS/_OVF_CTRL for global; PEBS-specific MSRs for branch-record sampling (covered in Intel-PMU-PEBS Tier-3 future).
- AMD: MSR_K7_EVNTSELn / MSR_K7_PERFCTRn (legacy); MSR_F15H_PERF_CTLn / MSR_F15H_PERF_CTRn (Family 15h+); MSR_AMD_DBG_EXTN_CFG (for AMD-specific debug ext).

REQ-9: per-vCPU pmc_in_use bitmap:
- Updated on perf_event create/destroy.
- Used to skip per-vmenter scan of all PMCs.

REQ-10: Reschedule on CPUID-change:
- Per-vCPU PMU re-enumerated when guest CPUID changes (rare; live-migration restore).
- `kvm_pmu_refresh(vcpu)` releases all perf_events + reinit.

REQ-11: AMD perfmon-v2 hooks (svm/pmu.c):
- AMD perfmon-v2 adds CPUID.0x80000022 enumeration + SVM-specific PMC MSRs.
- AMD does not have fixed counters; all GP.

REQ-12: PEBS (Precise Event-Based Sampling) - Intel:
- PEBS records additional CPU state on counter overflow.
- KVM passes PEBS-buffer through to guest if host supports.
- Out-of-scope at this Tier-3 level (covered in pmu-pebs future Tier-3).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pmc_idx_no_oob` | OOB | per-PMC index < (nr_gp + nr_fixed); enforced at idx_to_pmc lookup. |
| `perf_event_no_uaf` | UAF | per-PMC.perf_event as KArc; release on reprogram + cleanup. |
| `global_status_bitmap_bounded` | INVARIANT | global_status bits ≤ nr_gp + nr_fixed. |
| `event_count_rate_limited` | INVARIANT | event_count ≤ rate_limit per-tick; defense against guest-induced perf-event-storm. |

### Layer 2: TLA+

`virt/kvm/pmu_pmc_lifecycle.tla`:
- States: NotProgrammed, Programmed(eventsel), Active(eventsel, perf_event), Overflowed, Paused.
- Transitions:
  - NotProgrammed → Programmed via WRMSR(eventsel).
  - Programmed → Active via WRMSR(global_ctrl bit set).
  - Active → Overflowed via perf_event_overflow callback.
  - Overflowed → Active via guest WRMSR(global_status bit clear).
  - Active → Paused via WRMSR(global_ctrl bit clear).
- Properties:
  - `safety_perf_event_only_in_active` — perf_event running iff state == Active.
  - `safety_pmi_only_in_overflow` — PMI delivered iff Overflowed transition.
  - `liveness_overflow_eventually_pmi` — Overflowed → Active requires guest acknowledgment via global_status clear.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Pmu::init` post: per-vCPU pmu struct allocated; per-PMC initialized; vendor ops installed | `Pmu::init` |
| `Pmc::reprogram` post: old perf_event released; new perf_event created with current eventsel | `Pmc::reprogram` |
| `Pmu::deliver_pmi` post: LVT_PMI inject queued via lapic.set_irq | `Pmu::deliver_pmi` |
| Per-PMC.perf_event present iff PMC.idx bit set in pmu.pmc_in_use | invariant on operations |
| Per-vCPU global_ctrl bit cleared iff per-PMC.is_paused | invariant on `Pmu::set_msr` |

### Layer 4: Verus/Creusot functional

`Guest WRMSR(eventsel) + WRMSR(global_ctrl bit set) → guest RDPMC produces correct count` round-trip equivalence: per-PMC counter value visible to guest via RDPMC equals host perf_event count for matching event during guest execution time.

### hardening

(Inherits row-1 features from `virt/kvm/kvm-core.md` § Hardening.)

PMU-specific reinforcement:

- **Per-PMC.eventsel mask validated** at WRMSR-set — defense against guest selecting privileged-only events (e.g., uncore events, host-only events).
- **Per-vCPU PMU access gated by CPUID.0x0A.EAX != 0** — defense against unsupported guest accessing PMU MSRs.
- **Per-PMC perf_event released on vcpu cleanup** — defense against host-perf-event leak post-VM-destroy.
- **event_count rate-limit per-vCPU** — defense against guest-induced perf-event-storm filling host perf_event arena.
- **Per-PMC overflow callback dispatched via perf_event subsystem** — defense against in-handler nested overflow.
- **Per-vCPU global_ctrl_mask** filters supported counters — defense against guest enabling counter-bits beyond host capability.
- **Per-PMC perf_event configured for guest-only mode** (exclude_host=1) — defense against PMC counting host-mode events leaking host workload info to guest.
- **PMI deliver gated by guest LVT_PMI unmask** — defense against guest receiving PMI when LVT_PMI masked (causing spurious-IRQ).
- **AMD/Intel vendor ops in const vtable** — defense against ops table corruption.
- **Per-PMC.is_paused guard prevents post-pause RDPMC access** — defense against stale-counter-read.
- **Per-vCPU PMU refresh on CPUID change drops all perf_events** — defense against stale-CPU-config perf_event after migration.
- **Per-VM `disable_pmu` per-VM-arg honored** — defense against guests using PMU when VM-create requested no-PMU.

