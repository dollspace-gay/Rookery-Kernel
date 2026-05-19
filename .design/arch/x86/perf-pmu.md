# Tier-3: arch/x86/events/core.c — x86 perf PMU core (vendor-agnostic skeleton + counter scheduling)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: arch/x86/00-overview.md
upstream-paths:
  - arch/x86/events/core.c (~3152 lines)
  - arch/x86/events/perf_event.h
  - arch/x86/events/intel/core.c (hsw_hw_config, Intel-specific init)
  - arch/x86/events/intel/ds.c (PEBS handlers)
  - arch/x86/events/intel/lbr.c (Last Branch Record)
  - arch/x86/events/amd/core.c (AMD vendor init)
  - arch/x86/include/asm/perf_event.h
  - arch/x86/include/asm/cpu_device_id.h (x86_match_cpu)
-->

## Summary

The x86 vendor-agnostic skeleton of the perf PMU — the bridge between architecture-independent `kernel/events/core.c` (`struct pmu` consumer) and vendor-specific `intel/core.c` / `amd/core.c` / `zhaoxin/core.c` (PMU vtable providers). Hosts per-CPU `struct cpu_hw_events` (which events are in which counter slot), per-counter MSR programming (`x86_pmu_enable_event`, `x86_perf_event_update`, `x86_perf_event_set_period`), the assignment / scheduling algorithm (`perf_sched_*`, `x86_schedule_events`) that maps N requested events to M physical counters subject to per-event constraints, and the global PMU registration (`pmu = { .pmu_enable = x86_pmu_enable, … }` → `perf_pmu_register`). Critical for: PEBS / LBR sampling correctness, AnyThread / counter-class constraint satisfaction, NMI watchdog cycles counter, perf-stat / perf-record on x86, BPF-backed hardware-event probing.

This Tier-3 covers `arch/x86/events/core.c` (~3152 lines). Vendor-specific (`intel/`, `amd/`) details are referenced as `Pmu::Vendor` impls but not exhaustively enumerated here.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct x86_pmu` (perf_event.h) | per-vendor PMU vtable + cap | `arch::x86::events::X86Pmu` |
| `x86_pmu` (global) | per-boot vendor selection | `arch::x86::events::X86_PMU` (static) |
| `struct cpu_hw_events` (perf_event.h) | per-CPU PMU state | `arch::x86::events::CpuHwEvents` |
| `static struct pmu pmu` | per-boot top-level struct pmu registered with core | `arch::x86::events::PMU_STRUCT` |
| `init_hw_perf_events()` (early_initcall) | per-boot vendor init dispatch | `arch::x86::events::init_hw_perf_events` |
| `intel_pmu_init()` | per-Intel vendor init | `arch::x86::events::intel::init` |
| `amd_pmu_init()` | per-AMD vendor init | `arch::x86::events::amd::init` |
| `zhaoxin_pmu_init()` | per-Zhaoxin vendor init | `arch::x86::events::zhaoxin::init` |
| `__x86_pmu_event_init(event)` | per-event init common path | `X86Pmu::event_init_common` |
| `x86_pmu_event_init(event)` (= pmu.event_init) | per-event init top-level | `X86Pmu::event_init` |
| `x86_pmu_hw_config(event)` | per-event hw_config validation | `X86Pmu::hw_config` |
| `hsw_hw_config(event)` (intel/core.c) | per-Intel-HSW+ hw_config (precise_ip, lbr_*) | `arch::x86::events::intel::hsw_hw_config` |
| `x86_setup_perfctr(event)` | per-event MSR address & event-code setup | `X86Pmu::setup_perfctr` |
| `x86_pmu_add(event, flags)` (= pmu.add) | per-event install into per-CPU PMU | `X86Pmu::add` |
| `x86_pmu_del(event, flags)` (= pmu.del) | per-event remove from per-CPU PMU | `X86Pmu::del` |
| `x86_pmu_start(event, flags)` (= pmu.start) | per-event begin counting | `X86Pmu::start` |
| `x86_pmu_stop(event, flags)` (= pmu.stop) | per-event stop counting | `X86Pmu::stop` |
| `x86_pmu_enable(pmu)` (= pmu.pmu_enable) | per-CPU PMU global enable | `X86Pmu::pmu_enable` |
| `x86_pmu_disable(pmu)` (= pmu.pmu_disable) | per-CPU PMU global disable | `X86Pmu::pmu_disable` |
| `x86_pmu_enable_event(event)` | per-event MSR write to enable | `X86Pmu::enable_event` |
| `x86_pmu_enable_all(added)` | per-CPU enable-all | `X86Pmu::enable_all` |
| `x86_pmu_disable_all()` | per-CPU disable-all | `X86Pmu::disable_all` |
| `x86_pmu_handle_irq(regs)` | per-PMI (Performance Monitoring Interrupt) NMI handler | `X86Pmu::handle_irq` |
| `x86_perf_event_update(event)` | per-read counter delta into event->count | `X86Pmu::event_update` |
| `x86_perf_event_set_period(event)` | per-overflow reload sample period | `X86Pmu::set_period` |
| `x86_schedule_events(cpuc, n, assign)` | per-context schedule N events → counter slots | `X86Pmu::schedule_events` |
| `collect_events(cpuc, leader, dogrp)` | per-group enumerate group events | `X86Pmu::collect_events` |
| `collect_event(cpuc, event, max, n)` | per-event add to scheduling array | `X86Pmu::collect_event` |
| `perf_sched_init/_next_event/_find_counter/_save_state/_restore_state` | per-schedule backtracking search | `arch::x86::events::PerfSched::*` |
| `perf_assign_events(constraints, n, ...)` | per-call backtracking assignment | `arch::x86::events::perf_assign_events` |
| `check_hw_exists(pmu, cntr_mask, fixed_cntr_mask)` | per-boot PMU presence probe | `X86Pmu::check_hw_exists` |
| `reserve_pmc_hardware()` / `release_pmc_hardware()` | per-pmc-reservation under perfctl_reserve | `X86Pmu::reserve_hw` / `_release_hw` |
| `x86_reserve_hardware()` / `x86_release_hardware()` | per-active_events ref-counted reserve | `X86Pmu::reserve` / `_release` |
| `x86_add_exclusive(what)` / `x86_del_exclusive(what)` | per-exclusive-feature (PT vs LBR) ref count | `X86Pmu::add_exclusive` / `_del_exclusive` |
| `x86_pmu_extra_regs(config, event)` | per-event extra MSR lookup (offcore, ldlat) | `X86Pmu::extra_regs` |
| `hw_perf_event_destroy(event)` | per-event destroy callback | `X86Pmu::event_destroy` |
| `hw_perf_lbr_event_destroy(event)` | per-LBR-event destroy callback | `X86Pmu::lbr_event_destroy` |
| `perf_event_print_debug()` | per-CPU dump on `echo p > /proc/sysrq-trigger`-style | `X86Pmu::print_debug` |
| `perf_events_lapic_init()` | per-CPU APIC LVTPC vector wire-up | `X86Pmu::lapic_init` |
| `perf_load_guest_lvtpc(u32)` / `perf_put_guest_lvtpc()` | per-vcpu KVM guest LVTPC swap | `X86Pmu::load_guest_lvtpc` / `_put_guest_lvtpc` |
| `x86_pmu_max_precise(pmu)` | per-vendor max `precise_ip` cap | `X86Pmu::max_precise` |
| `x86_match_cpu(match[])` (cpu_device_id.h) | per-model-quirk table match | `arch::x86::cpu::x86_match_cpu` |
| `precise_br_compat(event)` | per-precise-branch-mode compat check | `X86Pmu::precise_br_compat` |
| `is_x86_event(event)` | predicate: event belongs to x86 PMU | `X86Pmu::owns_event` |
| `static_call(x86_pmu_drain_pebs)` | per-vendor PEBS drain (Intel only) | `X86Pmu::drain_pebs` (static_call) |
| `static_call(x86_pmu_pebs_enable / _disable / _enable_all / _disable_all)` | per-vendor PEBS gating | `X86Pmu::pebs_*` (static_call) |
| `static_call(x86_pmu_pebs_aliases)` | per-vendor PEBS event aliasing | `X86Pmu::pebs_aliases` (static_call) |
| `struct event_constraint` (perf_event.h) | per-event idxmsk + weight | `arch::x86::events::EventConstraint` |
| `emptyconstraint` / `unconstrained` | per-boot sentinel constraints | `EMPTY_CONSTRAINT` / `UNCONSTRAINED` |
| `x86_pmu_format_group` / `x86_pmu_events_group` / `x86_pmu_attr_group` | sysfs `/sys/bus/event_source/devices/cpu/` groups | `arch::x86::events::SYSFS_*_GROUP` |

## Compatibility contract

REQ-1: `struct x86_pmu` is a vendor-supplied vtable with caps:
- `name` (string for /sys/bus/event_source/devices/cpu/type).
- `handle_irq` (PMI NMI handler).
- `disable_all` / `enable_all(added)`.
- `enable` / `disable` (per-event).
- `add` / `del` / `hw_config` / `schedule_events`.
- `eventsel` / `perfctr` (MSR base addresses).
- `cntr_mask64` / `fixed_cntr_mask64` (which counter indices are usable).
- `event_map(i)` (HW event-id → event-code translator).
- `max_events`, `num_counters`, `num_counters_fixed`, `cntval_bits`, `cntval_mask`.
- `apic` (true if PMI delivered via LAPIC), `pebs`, `pebs_active`, `pebs_broken`, `lbr_nr`, `lbr_*`.
- `extra_regs`, `flags`, `attrs`, `pmu_attrs`, `event_attrs`, `caps_attrs`.

REQ-2: `init_hw_perf_events` (early_initcall):
- Calls `intel_pmu_init` / `amd_pmu_init` / `zhaoxin_pmu_init` based on `boot_cpu_data.x86_vendor`.
- Vendor init fills `x86_pmu` struct fields.
- After: `pmu.event_init = x86_pmu_event_init; pmu.add = x86_pmu_add; …` populated from `x86_pmu`.
- `perf_pmu_register(&pmu, "cpu", PERF_TYPE_RAW)` registers with kernel/events/core.c.
- Per-CPU `cpu_hw_events` zeroed; cpuhp_setup_state hooks for prepare/online/starting/dying/dead CPUs.

REQ-3: `__x86_pmu_event_init(event)` validates one perf_event for x86 PMU:
- Reserve hardware (`x86_reserve_hardware`).
- Compute `event->hw.config` from `attr.config` via vendor `hw_config`.
- Compute `event->hw.event_base` / `_idx` / `extra_reg` etc.
- Return error if event unsatisfiable on this PMU.

REQ-4: `x86_pmu_event_init` (= pmu.event_init):
- /* Vendor-agnostic gate */
- if event->attr.type ∉ {PERF_TYPE_RAW, PERF_TYPE_HARDWARE, PERF_TYPE_HW_CACHE, PERF_TYPE_BREAKPOINT} ∧ not our PMU type: return -ENOENT.
- call `__x86_pmu_event_init`.
- if event->destroy == NULL: event->destroy = hw_perf_event_destroy (so refcount decrements on release).

REQ-5: `x86_pmu_hw_config(event)` validates per-event config:
- exclude_user / exclude_kernel / exclude_hv translate to USR/OS/INV bits in event->hw.config.
- precise_ip > x86_pmu_max_precise(pmu): -EOPNOTSUPP.
- sample_type & PERF_SAMPLE_BRANCH_STACK without LBR: -EOPNOTSUPP.
- attr.precise_ip in [0..max_precise]; PEBS reqs vendor-specific.
- if event->attr.config has reserved bits: -EINVAL.

REQ-6: `hsw_hw_config(event)` (Intel HSW+ vendor hook):
- Additionally validates: lbr_select for `attr.branch_sample_type`, intel_pebs setup, intel-specific event constraints.
- Wires `event->hw.flags` with PERF_X86_EVENT_PEBS_HSW / _LBR / _PEBS_LD_HSW as applicable.

REQ-7: `x86_pmu_add(event, flags)` (= pmu.add) installs into per-CPU PMU:
- cpuc = this_cpu_ptr(&cpu_hw_events).
- If !(flags & PERF_EF_START): event starts STOPPED; else may auto-START.
- Append event to `cpuc->event_list[cpuc->n_events]` after `collect_event`.
- Run `x86_schedule_events(cpuc, cpuc->n_events, cpuc->assign)`. If returns -EAGAIN (can't fit): refuse the add.
- For each event: cpuc->event_constraint[i] = vendor constraint.
- If flags & PERF_EF_START: x86_pmu_start(event, PERF_EF_RELOAD).
- Returns 0 on success.

REQ-8: `x86_pmu_del(event, flags)` (= pmu.del):
- x86_pmu_stop(event, PERF_EF_UPDATE).
- Remove from cpuc->event_list[]; shift down.
- Update cpuc->assign[] accordingly.
- Decrement nr_metric_event / pebs_users / lbr_users as applicable.

REQ-9: `x86_pmu_start(event, flags)`:
- if flags & PERF_EF_RELOAD: x86_perf_event_set_period(event).
- event->hw.state = 0.
- Write event->hw.config_base / event_base MSR slots via x86_pmu_enable_event.
- Update userpage (mmap mapping for rdpmc).

REQ-10: `x86_pmu_stop(event, flags)`:
- if event->hw.state & PERF_HES_STOPPED: return (already stopped).
- x86_pmu.disable(event). — vendor-specific MSR write to clear ENABLE bit.
- if flags & PERF_EF_UPDATE: x86_perf_event_update(event).
- event->hw.state |= PERF_HES_STOPPED | PERF_HES_UPTODATE.

REQ-11: `x86_perf_event_update(event)`:
- /* Read 64-bit counter via composite of low+high (or single 48-bit rdpmc) */
- prev_raw = event->hw.prev_count.
- new_raw = rdpmcl(event->hw.event_base_rdpmc) & x86_pmu.cntval_mask.
- delta = (new_raw - prev_raw) & x86_pmu.cntval_mask.
- atomic64_add(delta, &event->count).
- atomic64_sub(delta, &event->hw.period_left).
- event->hw.prev_count = new_raw.

REQ-12: `x86_perf_event_set_period(event)`:
- /* Reload counter for sample-period mode */
- left = atomic64_read(&event->hw.period_left).
- if left > sample_period_max: left = sample_period_max.
- if left < 2: left = 2 (overflow margin).
- counter_init_val = (-left) & x86_pmu.cntval_mask.
- wrmsrl(event->hw.event_base, counter_init_val).
- event->hw.prev_count = counter_init_val.
- perf_event_update_userpage(event).

REQ-13: `x86_schedule_events(cpuc, n, assign)`:
- /* N events with possibly-overlapping constraint masks → assign each a unique counter idx */
- Try fast path: each event's prior assign[] still satisfies its constraint → keep.
- Else: backtracking search via `perf_sched_init` + `perf_sched_find_counter` + `perf_sched_next_event` + `perf_sched_save_state` + `perf_sched_restore_state`.
- Returns 0 on success (all N events assigned), -EAGAIN if infeasible.
- Side-effect: writes assign[i] = chosen counter idx for event i.

REQ-14: `x86_pmu_handle_irq(regs)` (PMI NMI):
- Called from `perf_event_nmi_handler` (NMI dispatcher).
- For each active counter, check `status` MSR (Intel: IA32_PERF_GLOBAL_STATUS / AMD: derivable).
- For each set bit: x86_perf_event_update; if overflow → perf_event_overflow (cross-ref `kernel/perf-events.md` REQ-9).
- Drain PEBS records if any (static_call(x86_pmu_drain_pebs)).
- ACK overflow status (writeback to global_ovf_ctrl on Intel).

REQ-15: `x86_match_cpu(match[])` quirk gating:
- Iterates `struct x86_cpu_id match[]` entries `{vendor, family, model, stepping, feature, …}`.
- Returns first match (or NULL).
- Used to patch x86_pmu fields at boot for known-broken / known-special CPUs (e.g. Haswell BDC, Skylake PEBS errata).

REQ-16: PEBS (Precise Event-Based Sampling):
- Vendor sets x86_pmu.pebs = true; PEBS buffer allocated per-CPU in `intel/ds.c`.
- Per-event with `attr.precise_ip > 0` and PEBS-capable event → PEBS-mode counter.
- Overflow causes hardware to write a record (RIP, RFLAGS, R0-R15, …) to PEBS buffer instead of taking PMI immediately; threshold reached → PMI.
- Handler drains PEBS records (static_call(x86_pmu_drain_pebs)) → builds samples → calls perf_event_overflow per record.
- precise_ip semantics: 1=imprecise PMI, 2=PEBS skid-corrected, 3=PEBS with LBR-augmentation.

REQ-17: LBR (Last Branch Record):
- Intel: per-CPU MSR_LBR_TOS + MSR_LASTBRANCH_{0..n}_FROM/_TO ring.
- Saved on context-switch and at PMI/PEBS overflow.
- Exposed via `attr.branch_sample_type` (USER / KERNEL / ANY_CALL / ANY_RETURN / IND_CALL / COND / IND_JUMP / CALL / NO_FLAGS / NO_CYCLES / TYPE_SAVE / HW_INDEX).
- LBR users ref-counted via cpuc->lbr_users; LBR enabled when ref>0.

REQ-18: Counter constraints + scheduling:
- Each event has `struct event_constraint` = (idxmsk, code, cmask, weight, flags).
- idxmsk: bitmap of allowable counter indices.
- weight: popcount(idxmsk) — fewer choices first (most-constrained-first heuristic).
- Backtracker `perf_sched_*` tries to assign each event to a counter satisfying its constraint without collision.
- Fixed counters (Intel: INSTR_RETIRED.ANY, CPU_CLK_UNHALTED.THREAD, CPU_CLK_UNHALTED.REF_TSC, TOPDOWN.SLOTS) have dedicated fixed-PMC indices.

REQ-19: `x86_add_exclusive(what)` / `_del_exclusive`:
- Mutually-exclusive features: BTS, PT, LBR (some combinations conflict).
- Per-feature atomic ref count; first-on enables HW, last-off disables.

REQ-20: sysfs attribute groups:
- `/sys/bus/event_source/devices/cpu/format/*` — `x86_pmu_format_group` (event encoding hints).
- `/sys/bus/event_source/devices/cpu/events/*` — `x86_pmu_events_group` (predefined event names).
- `/sys/bus/event_source/devices/cpu/caps/*` — capability strings (pmu_name, max_precise, branches, …).

## Acceptance Criteria

- [ ] AC-1: `init_hw_perf_events` runs at early_initcall; vendor `init` selected by `boot_cpu_data.x86_vendor`.
- [ ] AC-2: After init: `/sys/bus/event_source/devices/cpu/type` present; `perf_pmu_register` succeeded with PERF_TYPE_RAW.
- [ ] AC-3: `perf_event_open(attr={.type=HARDWARE, .config=CYCLES})` succeeds on any x86 PMU.
- [ ] AC-4: N≤num_counters events: each gets a counter; N>num_counters: multiplexing engages (`time_running < time_enabled`).
- [ ] AC-5: `precise_ip=2` event on PEBS-capable model: samples have IP within ±0 of fault instruction (vs. PMI skid).
- [ ] AC-6: `attr.branch_sample_type=PERF_SAMPLE_BRANCH_USER`: each sample has nonempty `branch_stack` from LBR.
- [ ] AC-7: PMI fires within `sample_period` instructions; perf_event_overflow called.
- [ ] AC-8: Overflow on PEBS-mode event drains PEBS buffer before exit; one `PERF_RECORD_SAMPLE` per PEBS record.
- [ ] AC-9: Conflicting constraints (group of 5 events all requiring fixed-PMC-0): event_init returns -EINVAL or schedule returns -EAGAIN.
- [ ] AC-10: `x86_match_cpu` model quirk applies: x86_pmu fields differ from default per quirk on matched CPU.
- [ ] AC-11: `x86_pmu.pebs_broken` model: PEBS auto-disabled (precise_ip>0 rejected).
- [ ] AC-12: NMI watchdog: hardware-cycles event installed via `perf_event_create_kernel_counter`; periodic overflow drives `lockup_detector_event_create`.
- [ ] AC-13: BTS + PT exclusive: `x86_add_exclusive(X86_PMU_EXCLUSIVE_BTS)` then `_PT` returns -EBUSY.

## Architecture

```
struct X86Pmu {                                // = upstream struct x86_pmu
    name: &'static str,
    handle_irq: fn(*PtRegs) -> i32,            // PMI NMI handler
    disable_all: fn(),
    enable_all: fn(added: i32),
    enable: fn(*PerfEvent),
    disable: fn(*PerfEvent),
    add: fn(*PerfEvent, flags: i32) -> i32,
    del: fn(*PerfEvent, flags: i32),
    hw_config: fn(*PerfEvent) -> i32,
    schedule_events: fn(*CpuHwEvents, n: i32, assign: *i32) -> i32,
    eventsel: u32,                             // base MSR address
    perfctr: u32,                              // base counter MSR address
    cntr_mask64: u64,                          // allowed GP counter indices
    fixed_cntr_mask64: u64,                    // allowed fixed counter indices
    num_counters: i32, num_counters_fixed: i32,
    cntval_bits: i32, cntval_mask: u64,
    apic: bool,                                // LAPIC LVTPC vector for PMI
    pebs: bool, pebs_active: bool, pebs_broken: bool,
    lbr_nr: i32, lbr_tos: u32, lbr_from: u32, lbr_to: u32,
    extra_regs: *ExtraReg,
    flags: u64,
    attrs: *AttrGroup,
    pmu_attrs: *AttrGroup,
    event_attrs: *AttrGroup,
    caps_attrs: *AttrGroup,
    ...
}

struct CpuHwEvents {                           // = upstream struct cpu_hw_events
    events: [*PerfEvent; X86_PMC_IDX_MAX],     // counter idx → event
    active_mask: [u64; ...],
    enabled: i32,
    n_events: i32, n_added: i32, n_txn: i32,
    assign: [i32; X86_PMC_IDX_MAX],            // event-list idx → counter idx
    event_list: [*PerfEvent; X86_PMC_IDX_MAX],
    event_constraint: [*EventConstraint; X86_PMC_IDX_MAX],
    pebs_output: i32,                          // 0=none, 1=PT, 2=DS
    lbr_users: i32,
    lbr_stack: LbrStack,
    ...
}

struct EventConstraint {                       // = upstream struct event_constraint
    idxmsk: [u64; ...],                        // bitmap of allowed counter idxs
    code: u64, cmask: u64,
    weight: i32,                               // popcount(idxmsk) for ordering
    flags: i32,
}
```

`arch::x86::events::init_hw_perf_events` (early_initcall):
1. err = match boot_cpu_data.x86_vendor:
   - INTEL ⟹ intel_pmu_init().
   - AMD | HYGON ⟹ amd_pmu_init().
   - ZHAOXIN | CENTAUR ⟹ zhaoxin_pmu_init().
   - _ ⟹ -ENODEV.
2. if err: pr_cont("no PMU driver, software events only.\n"); return 0.
3. /* Vendor populated x86_pmu */
4. pmu_check_apic(). — warn if APIC absent but x86_pmu.apic=true.
5. /* Populate top-level struct pmu */
6. PMU_STRUCT.event_init = x86_pmu_event_init.
7. PMU_STRUCT.pmu_enable = x86_pmu_enable; PMU_STRUCT.pmu_disable = x86_pmu_disable.
8. PMU_STRUCT.add = x86_pmu_add; PMU_STRUCT.del = x86_pmu_del.
9. PMU_STRUCT.start = x86_pmu_start; PMU_STRUCT.stop = x86_pmu_stop.
10. PMU_STRUCT.read = x86_pmu_read.
11. PMU_STRUCT.attr_groups = (x86_pmu.attrs, x86_pmu.pmu_attrs, x86_pmu.event_attrs, x86_pmu.caps_attrs, x86_pmu.format_group, ...).
12. /* Wire static_calls */
13. static_call_update(x86_pmu_drain_pebs, x86_pmu.drain_pebs).
14. static_call_update(x86_pmu_pebs_enable, x86_pmu.pebs_enable). ... (all pebs_* slots).
15. /* Per-CPU init via cpuhp_setup_state */
16. cpuhp_setup_state(CPUHP_PERF_X86_PREPARE, x86_pmu_prepare_cpu, x86_pmu_dead_cpu).
17. cpuhp_setup_state(CPUHP_AP_PERF_X86_STARTING, x86_pmu_starting_cpu, x86_pmu_dying_cpu).
18. cpuhp_setup_state(CPUHP_AP_PERF_X86_ONLINE, x86_pmu_online_cpu, NULL).
19. perf_pmu_register(&PMU_STRUCT, "cpu", PERF_TYPE_RAW).
20. return 0.

`X86Pmu::event_init` (= pmu.event_init, top-level entry from kernel/events/core.c):
1. /* Filter by attr.type */
2. if event->attr.type ≠ PMU_STRUCT.type ∧ attr.type ∉ {HARDWARE, HW_CACHE, RAW}: return -ENOENT.
3. err = __x86_pmu_event_init(event).
4. if err: return err.
5. if !event->destroy: event->destroy = hw_perf_event_destroy.
6. return 0.

`X86Pmu::event_init_common` (= __x86_pmu_event_init):
1. if !x86_pmu_initialized(): return -ENODEV.
2. err = x86_reserve_hardware(). — refcount HW; if first user, reserve_pmc_hardware allocates MSRs.
3. if err: return err.
4. atomic_inc(&active_events).
5. event->destroy_filled = false. (later in event_init we'll set destroy.)
6. event->hw.idx = -1.
7. event->hw.last_cpu = -1.
8. event->hw.last_tag = ~0ULL.
9. /* Vendor hook */
10. err = x86_pmu.hw_config(event). — populates event->hw.config, validates precise_ip, branch_sample_type, etc.
11. if err: goto out_release_hw.
12. err = x86_setup_perfctr(event). — populates event->hw.event_base / .config_base / .extra_reg.
13. if err: goto out_release_hw.
14. return 0.
15. out_release_hw: x86_release_hardware(); return err.

`X86Pmu::hw_config` (= x86_pmu_hw_config, vendor-agnostic part):
1. hwc = &event->hw.
2. hwc.config_base = x86_pmu.eventsel. — base eventselect MSR.
3. hwc.event_base = x86_pmu.perfctr. — base counter MSR.
4. hwc.idx = -1.
5. /* exclude_* → INV/USR/OS bits */
6. if event->attr.exclude_user: hwc.config |= ~ARCH_PERFMON_EVENTSEL_USR.
7. if event->attr.exclude_kernel: hwc.config |= ~ARCH_PERFMON_EVENTSEL_OS.
8. if event->attr.exclude_hv: hwc.config |= ARCH_PERFMON_EVENTSEL_INV (vendor-specific).
9. /* precise_ip */
10. if event->attr.precise_ip > x86_pmu_max_precise(pmu): return -EOPNOTSUPP.
11. if event->attr.precise_ip ∧ !x86_pmu.pebs: return -EOPNOTSUPP.
12. /* branch_sample_type */
13. if event->attr.sample_type & PERF_SAMPLE_BRANCH_STACK ∧ x86_pmu.lbr_nr == 0: return -EOPNOTSUPP.
14. /* config validation */
15. if (event->attr.config & x86_pmu.event_map_config_mask) != event->attr.config_filtered: return -EINVAL.
16. return 0.

`X86Pmu::add` (= x86_pmu_add):
1. hwc = &event->hw.
2. cpuc = this_cpu_ptr(&cpu_hw_events).
3. n0 = cpuc->n_events.
4. err = collect_events(cpuc, event, false). — adds event (and group siblings if dogrp) to cpuc->event_list[].
5. if err < 0: return err.
6. n = err.
7. /* Try to schedule */
8. err = x86_schedule_events(cpuc, n, cpuc->assign).
9. if err: goto out_undo.
10. /* Commit */
11. memcpy(cpuc->event_list_filled, cpuc->event_list, ...).
12. /* Mark counter slot taken */
13. for i in [n0..n]: cpuc->assign[i] = chosen idx.
14. hwc.state = PERF_HES_UPTODATE | PERF_HES_STOPPED.
15. if flags & PERF_EF_START: x86_pmu_start(event, PERF_EF_RELOAD).
16. /* Update userpage so rdpmc users see correct counter idx */
17. perf_event_update_userpage(event).
18. return 0.
19. out_undo: /* Roll back collect_events */ cpuc->n_events = n0; return err.

`X86Pmu::start` (= x86_pmu_start):
1. hwc = &event->hw.
2. if WARN_ON_ONCE(!(hwc.state & PERF_HES_STOPPED)): return.
3. if WARN_ON_ONCE(hwc.idx == -1): return.
4. if flags & PERF_EF_RELOAD:
   - x86_perf_event_set_period(event). — reload counter for sample period.
5. hwc.state = 0.
6. x86_pmu.enable(event). — vendor MSR write to set ENABLE bit on event's eventselect.
7. cpuc->events[hwc.idx] = event.
8. __set_bit(hwc.idx, cpuc->active_mask).

`X86Pmu::stop` (= x86_pmu_stop):
1. hwc = &event->hw.
2. cpuc = this_cpu_ptr(&cpu_hw_events).
3. if hwc.state & PERF_HES_STOPPED: return. — already stopped.
4. x86_pmu.disable(event). — clear ENABLE bit.
5. __clear_bit(hwc.idx, cpuc->active_mask).
6. cpuc->events[hwc.idx] = NULL.
7. hwc.state |= PERF_HES_STOPPED.
8. if flags & PERF_EF_UPDATE:
   - x86_perf_event_update(event). — read counter, accumulate delta.
   - hwc.state |= PERF_HES_UPTODATE.

`X86Pmu::event_update` (= x86_perf_event_update):
1. /* Read 64-bit counter as (raw_now - raw_prev) modulo cntval_mask */
2. shift = 64 - x86_pmu.cntval_bits.
3. again:
4.   prev_raw_count = atomic64_read(&hwc.prev_count).
5.   rdpmcl(hwc.event_base_rdpmc, new_raw_count).
6. while atomic64_cmpxchg(&hwc.prev_count, prev_raw_count, new_raw_count) ≠ prev_raw_count: goto again.
7. delta = ((new_raw_count - prev_raw_count) << shift) >> shift. — sign-extend
8. local64_add(delta, &event->count).
9. local64_sub(delta, &hwc.period_left).
10. return new_raw_count.

`X86Pmu::set_period` (= x86_perf_event_set_period):
1. hwc = &event->hw.
2. period = hwc.sample_period.
3. left = atomic64_read(&hwc.period_left).
4. if unlikely(left ≤ 0): left += period; atomic64_set(&hwc.period_left, left); hwc.last_period = period; ret = 1. — overflow signal.
5. if unlikely(left < 2): left = 2.
6. if left > x86_pmu.max_period: left = x86_pmu.max_period.
7. atomic64_set(&hwc.prev_count, (u64)(-left)).
8. wrmsrl(hwc.event_base, (u64)(-left) & x86_pmu.cntval_mask).
9. perf_event_update_userpage(event).
10. return ret.

`X86Pmu::schedule_events` (= x86_schedule_events):
1. /* Fast path: prior assignments still satisfy constraints */
2. for i in [0..n]:
   - hwc = &cpuc->event_list[i]->hw.
   - c = cpuc->event_constraint[i].
   - if c.flags & PERF_X86_EVENT_DYNAMIC ∨ hwc.idx == -1: goto slow.
   - if !test_bit(hwc.idx, c.idxmsk): goto slow.
   - assign[i] = hwc.idx.
3. /* All fit fast — done */
4. return 0.
5. slow: /* Backtracking via perf_sched_* */
6. perf_sched_init(&sched, constraints, n, wmin, wmax, gpmax).
7. do:
8.   ok = perf_sched_find_counter(&sched). — finds counter for current event index.
9.   if !ok: return -EAGAIN.
10. while perf_sched_next_event(&sched).
11. for i in [0..n]: assign[i] = sched.state[i].counter.
12. return 0.

`X86Pmu::handle_irq` (= x86_pmu_handle_irq, called from NMI):
1. cpuc = this_cpu_ptr(&cpu_hw_events).
2. apic_write(APIC_LVTPC, APIC_DM_NMI). — re-arm LVTPC (Intel quirk).
3. status = x86_pmu.get_status(). — vendor read of overflow status MSR.
4. if !status: return 0. — not our IRQ.
5. handled = 1.
6. for bit in for_each_set_bit(status):
   - event = cpuc->events[bit].
   - if !event: x86_pmu.disable_event(bit); continue.
   - if !x86_perf_event_set_period(event): continue. — not yet at overflow.
   - perf_event_overflow(event, &data, regs). — cross-ref kernel/perf-events.md REQ-9.
7. /* PEBS drain */
8. static_call(x86_pmu_drain_pebs)(regs, &data).
9. /* ACK status */
10. x86_pmu.ack_status(status).
11. return handled.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cpuc_assign_unique` | INVARIANT | per-schedule_events: assign[i] all distinct ∧ each ∈ event_constraint[i].idxmsk. |
| `event_init_paired_with_destroy` | INVARIANT | per-event_init success: hw_perf_event_destroy enqueued so x86_release_hardware runs at free. |
| `pmu_enable_disable_balanced` | INVARIANT | per-CPU pmu_enable / pmu_disable strict ref-count balance across context-switch. |
| `pmi_status_acks_only_handled` | INVARIANT | per-handle_irq: only bits we processed get ACKed in status MSR. |
| `pebs_drain_before_period_set` | INVARIANT | per-PEBS-overflow handler: PEBS buffer drained before counter MSR reload. |
| `lbr_users_refcount_pairing` | INVARIANT | per-LBR-event add/del: cpuc.lbr_users incremented / decremented in lockstep. |
| `cntval_mask_applies` | INVARIANT | per-event_update: delta within cntval_mask after sign-extension. |
| `precise_ip_within_max` | INVARIANT | per-hw_config: precise_ip ≤ x86_pmu.max_precise. |
| `exclusive_feature_mutual` | INVARIANT | per-x86_add_exclusive: PT and BTS cannot both be active. |
| `match_cpu_first_match` | INVARIANT | per-x86_match_cpu: returns first matching entry; no late-bind override. |

### Layer 2: TLA+

`arch/x86/perf-pmu.tla`:
- Per-CPU model: cpuc.n_events, cpuc.assign[], cpuc.active_mask, pmu.enabled.
- Per-transitions: pmu_add, pmu_del, pmu_start, pmu_stop, pmu_enable, pmu_disable, pmi_arrive, pebs_overflow, context_switch.
- Properties:
  - `safety_assignment_unique` — ∀i≠j: assign[i] ≠ assign[j].
  - `safety_assignment_satisfies_constraint` — ∀i: assign[i] ∈ event_constraint[i].idxmsk.
  - `safety_overflow_handled` — every overflow eventually delivered to perf_event_overflow.
  - `safety_pmu_disable_stops_all_counters` — pmu_disable ⟹ active_mask = 0.
  - `liveness_per_pmi_eventually_ack` — every PMI eventually ACKs overflow status.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `X86Pmu::add` post: cpuc.n_events incremented exactly by count of group | `X86Pmu::add` |
| `X86Pmu::del` post: cpuc.n_events decremented; cpuc.events[hwc.idx] = NULL | `X86Pmu::del` |
| `X86Pmu::start` post: hwc.state = 0; cpuc.events[hwc.idx] = event | `X86Pmu::start` |
| `X86Pmu::stop` post: hwc.state \| PERF_HES_STOPPED; counter MSR disabled | `X86Pmu::stop` |
| `X86Pmu::event_update` post: event.count increased by delta; delta ≥ 0 (mod cntval_mask) | `X86Pmu::event_update` |
| `X86Pmu::set_period` post: counter MSR = -left; period_left in (0, max_period] | `X86Pmu::set_period` |
| `X86Pmu::schedule_events` post: ret=0 ⟹ assign[0..n] all valid | `X86Pmu::schedule_events` |
| `X86Pmu::handle_irq` post: handled bits cleared in status; perf_event_overflow called per overflow | `X86Pmu::handle_irq` |
| `init_hw_perf_events` post: PMU_STRUCT registered with PERF_TYPE_RAW; sysfs nodes present | `arch::x86::events::init_hw_perf_events` |

### Layer 4: Verus/Creusot functional

`Per-event lifecycle: event_init (hw_config → setup_perfctr) → pmu.add → pmu.start → (PMI overflow → handle_irq → event_update → set_period) → pmu.stop → pmu.del → destroy` semantic equivalence: per-Documentation/perf/intel-hybrid.rst, per-Intel SDM Vol 3 (Performance Monitoring), per-AMD APM Vol 2 (Performance Monitoring Counters).

`Per-PEBS sample: counter overflow → HW writes PEBS record → threshold → PMI → drain_pebs → perf_event_overflow per record` semantic equivalence: per-Intel SDM Vol 3 Ch 19 (Performance Monitoring) + Ch 20 (PEBS).

## Hardening

(Inherits row-1 features from `arch/x86/00-overview.md` § Hardening.)

x86 PMU reinforcement:

- **Per-event x86_reserve_hardware refcount strict** — defense against per-PMU leak / double-release.
- **Per-schedule_events backtracker bounded** — defense against per-pathological-group hang during assignment.
- **Per-pmu_enable / pmu_disable strict pairing** — defense against per-context-switch leak.
- **Per-PMI handler re-arms LVTPC apic_write** — defense against per-Intel-LVTPC-mask-after-NMI bug.
- **Per-PEBS drain before period set** — defense against per-overflow-record-lost.
- **Per-LBR users refcount** — defense against per-LBR-leak across event lifetime.
- **Per-x86_match_cpu quirk gating** — defense against per-broken-CPU silent-mis-count.
- **Per-pebs_broken auto-reject precise_ip>0** — defense against per-known-buggy-PEBS silent corruption.
- **Per-exclusive PT/BTS/LBR mutual lock** — defense against per-feature-conflict undefined-behavior.
- **Per-x86_pmu_max_precise rejection** — defense against per-precise_ip overflow into invalid range.
- **Per-attr.config reserved-bits validation** — defense against per-future-bit-poisoning.
- **Per-cpu_hw_events assigned-counter-unique invariant** — defense against per-double-mapped-counter silent overcount.
- **Per-userpage update on every period set** — defense against per-rdpmc-userspace stale-counter race.

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **GRKERNSEC_PERF_HARDEN** — `perf_event_open(2)` restricted to CAP_PERFMON (or CAP_SYS_ADMIN under legacy policy); `perf_event_paranoid` defaults to 3 (only privileged users can observe kernel events).
- **CAP_PERFMON gating** — required for hardware counter access; CAP_SYS_ADMIN required for kernel-tracepoint and raw PMU MSR exposure.
- **GRKERNSEC_HIDESYM** — kernel-pointer hiding in /proc/kallsyms; PMU MSR addresses, x86_pmu struct fields not exposed to unprivileged users.
- **GRKERNSEC_DMESG** — restrict syslog output to CAP_SYSLOG (including PMI overflow warnings and PEBS-broken model logs).
- **PAX_KERNEXEC** — W^X enforcement for kernel-text mappings; PMI NMI handler `.text` is RX/RO.
- **PAX_USERCOPY** — bounded copy_to_user on PEBS samples, LBR records, sysfs PMU attribute groups.
- **PAX_MEMORY_SANITIZE** — zero-on-free for per-CPU PEBS buffers, LBR ring slots on event destroy.
- **PAX_REFCOUNT** — saturating refcount on `x86_reserve_hardware` / `x86_release_hardware` accounting; `x86_add_exclusive` / `_del_exclusive` ref counts for BTS/PT/LBR mutual exclusion.
- **PAX_RANDKSTACK** — PMI NMI handler enters via paranoid_entry with per-syscall stack randomization preserved.
- **PAX_RAP / kCFI** — indirect-call signature enforcement on `x86_pmu` vtable (`handle_irq`, `add`, `del`, `start`, `stop`, `enable`, `disable`, `hw_config`, `schedule_events`), static_call slots (`x86_pmu_drain_pebs`, `_pebs_enable`, `_pebs_aliases`).
- **MSR-write capability gate (CAP_SYS_RAWIO)** — direct MSR access via `/dev/cpu/N/msr` gated; perf PMU MSR writes go through vetted x86_pmu dispatch.
- **perf_paranoid sysctl strict default** — value 3 by default: no hardware-event access without CAP_PERFMON, no kernel events without CAP_SYS_ADMIN, no tracepoint without CAP_SYS_ADMIN.
- **PEBS buffer in per-CPU read-only mapping** — defense against userspace tampering with PEBS sample stream.
- **LBR users refcount strict pairing** — defense against LBR leak across event lifetimes.
- **x86_pmu_max_precise rejection** — defense against precise_ip overflow into invalid range.
- **attr.config reserved-bits validation** — defense against future-bit poisoning.
- **NMI watchdog event privileged** — kernel-internal NMI watchdog uses `perf_event_create_kernel_counter` directly; no userspace exposure.

Per-doc rationale: perf PMU exposes Intel/AMD hardware performance counters which can leak microarchitectural state (cache lines, branch history, instruction retirement timing) and execute kernel-controlled MSR writes via vtable dispatch; grsec/PaX hardening here is centered on PERF_HARDEN + CAP_PERFMON gating, perf_paranoid=3 default, and kCFI on the vendor vtable so a hostile userspace cannot weaponize a side channel through `perf_event_open`.

## Open Questions

- (none at this Tier-3 level; vendor-specific PEBS / LBR drain detail in `intel/ds.c` + `intel/lbr.c` deferred to future siblings)

## Out of Scope

- Vendor PMU init detail (`arch/x86/events/intel/core.c` / `amd/core.c`) — covered by future Tier-3 siblings.
- Intel PT (Processor Trace) AUX-area handling (`arch/x86/events/intel/pt.c`) — separate Tier-3.
- BTS (Branch Trace Store) (`arch/x86/events/intel/bts.c`) — separate Tier-3.
- Uncore PMUs (`arch/x86/events/intel/uncore*.c`) — separate Tier-3.
- AMD IBS (Instruction-Based Sampling) (`arch/x86/events/amd/ibs.c`) — separate Tier-3.
- Hardware breakpoints arch glue (`arch/x86/kernel/hw_breakpoint.c`) — covered by perf overview Tier-2.
- perf-event core syscall + ring buffer — covered by `kernel/perf-events.md` and `kernel/perf/perf-event-core.md`.
- KVM guest PMU virtualization (`arch/x86/kvm/pmu.c`) — covered by KVM Tier-2.
- Implementation code.
