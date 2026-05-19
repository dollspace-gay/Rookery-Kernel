# Tier-3: arch/x86/kernel/tsc.c — x86 Time Stamp Counter

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: arch/x86/00-overview.md
upstream-paths:
  - arch/x86/kernel/tsc.c (~1609 lines)
  - arch/x86/kernel/tsc_sync.c (check_tsc_sync_target / tsc_store_and_check_tsc_adjust)
  - arch/x86/include/asm/tsc.h
  - arch/x86/include/asm/msr.h (MSR_IA32_TSC, MSR_IA32_TSC_ADJUST)
  - arch/x86/entry/vdso/vclock_gettime.c (__vdso_clock_gettime VCLOCK_TSC path)
  - include/linux/clocksource.h (struct clocksource, CLOCK_SOURCE_*)
-->

## Summary

The x86 Time Stamp Counter (TSC) is a per-CPU 64-bit cycle counter read via `RDTSC` / `RDTSCP`. On modern Intel/AMD it is **invariant** (constant frequency across P-states, never stops in C-states) which makes it both the primary clocksource (`clocksource_tsc`, rating 300) and the engine behind `sched_clock()` and the vDSO clock-gettime fast-path. Per-`tsc_init()` calibrates frequency in priority order: **CPUID 0x15** (TSC/Crystal ratio, `native_calibrate_tsc`) -> **CPUID 0x16** (`cpu_khz_from_cpuid`) -> **HPET / ACPI PM-timer / PIT** (`pit_hpet_ptimer_calibrate_cpu` / `quick_pit_calibrate`). Per-`cyc2ns` per-CPU multiplier/shift converts TSC cycles -> nanoseconds via `mul_u64_u32_shr` with a sequence-counter-latch for lock-free reads. Per-AP `check_tsc_sync_target()` runs against the BSP after bringup and on failure marks TSC unstable; per-`tsc_store_and_check_tsc_adjust()` reads/writes the per-CPU `MSR_IA32_TSC_ADJUST` so all CPUs agree on TSC origin. Per-`clocksource_tsc_early` (rating 299) is registered immediately and replaced by `clocksource_tsc` after `tsc_refine_calibration_work` confirms accuracy against HPET / PM-timer within 1 %. Per-vDSO: `__vdso_clock_gettime` reads `rdtsc` directly when `vdso_clock_mode == VDSO_CLOCKMODE_TSC`, avoiding syscall entry entirely. Critical for: high-resolution timekeeping, ftrace, perf, scheduler tick decisions, and userspace nanosecond-grained `clock_gettime(CLOCK_MONOTONIC)`.

This Tier-3 covers `arch/x86/kernel/tsc.c` (~1609 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `tsc_init()` | per-late TSC init + clocksource register | `Tsc::init` |
| `tsc_early_init()` | per-early TSC init for sched_clock | `Tsc::early_init` |
| `determine_cpu_tsc_frequencies()` | per-cpu_khz + tsc_khz detection | `Tsc::determine_frequencies` |
| `native_calibrate_tsc()` | per-CPUID-0x15 calibration | `Tsc::calibrate_native` |
| `cpu_khz_from_cpuid()` | per-CPUID-0x16 base-MHz read | `Tsc::cpu_khz_from_cpuid` |
| `pit_hpet_ptimer_calibrate_cpu()` | per-late HPET/PM/PIT calibration | `Tsc::calibrate_pit_hpet_ptimer` |
| `pit_calibrate_tsc()` | per-PIT slow calibration | `Tsc::calibrate_pit` |
| `quick_pit_calibrate()` | per-PIT-MSB-edge fast calibration | `Tsc::quick_calibrate_pit` |
| `native_calibrate_cpu_early()` | per-early cpu_khz before ACPI | `Tsc::calibrate_cpu_early` |
| `native_calibrate_cpu()` | per-late cpu_khz | `Tsc::calibrate_cpu` |
| `recalibrate_cpu_khz()` | per-recalib on demand | `Tsc::recalibrate_cpu_khz` |
| `read_tsc()` | per-clocksource read = rdtsc_ordered | `Tsc::read_tsc` |
| `native_sched_clock()` | per-sched_clock via __cycles_2_ns | `Tsc::native_sched_clock` |
| `sched_clock_noinstr()` | per-noinstr sched_clock entry | `Tsc::sched_clock_noinstr` |
| `__cycles_2_ns()` / `cycles_2_ns()` | per-CPU cyc -> ns conversion | `Tsc::cycles_to_ns` |
| `__set_cyc2ns_scale()` / `set_cyc2ns_scale()` | per-CPU multiplier setup | `Tsc::set_cyc2ns_scale` |
| `cyc2ns_init_boot_cpu()` / `cyc2ns_init_secondary_cpus()` | per-CPU cyc2ns init | `Tsc::cyc2ns_init_*` |
| `__cyc2ns_read()` / `cyc2ns_read_begin/_end()` | per-seqcount-latch lock-free read | `Tsc::cyc2ns_read_*` |
| `clocksource_tsc_early` (rating 299) | per-early clocksource | static `CLOCKSOURCE_TSC_EARLY` |
| `clocksource_tsc` (rating 300) | per-late clocksource | static `CLOCKSOURCE_TSC` |
| `tsc_cs_enable()` | per-vDSO clock-mode set | `Tsc::cs_enable` |
| `tsc_cs_mark_unstable()` | per-watchdog mark | `Tsc::cs_mark_unstable` |
| `tsc_cs_tick_stable()` | per-sched_clock tick stabilize | `Tsc::cs_tick_stable` |
| `tsc_resume()` | per-suspend resume verify | `Tsc::resume` |
| `mark_tsc_unstable()` | per-extern unstable mark | `Tsc::mark_unstable` |
| `check_tsc_unstable()` | per-query unstable flag | `Tsc::check_unstable` |
| `unsynchronized_tsc()` | per-multi-CPU sync heuristic | `Tsc::unsynchronized` |
| `check_system_tsc_reliable()` | per-CONSTANT/NONSTOP/ADJUST/<=4pkg gate | `Tsc::check_system_reliable` |
| `tsc_disable_clocksource_watchdog()` | per-watchdog-off | `Tsc::disable_watchdog` |
| `tsc_refine_calibration_work()` | per-delayed_work refine vs HPET/PM | `Tsc::refine_calibration_work` |
| `init_tsc_clocksource()` | per-device_initcall final register | `Tsc::init_clocksource` |
| `tsc_store_and_check_tsc_adjust()` (tsc_sync.c) | per-CPU MSR_IA32_TSC_ADJUST sanitize | `Tsc::store_check_adjust` |
| `tsc_verify_tsc_adjust()` (tsc_sync.c) | per-resume MSR re-verify | `Tsc::verify_adjust` |
| `check_tsc_sync_target()` (tsc_sync.c) | per-AP TSC sync vs BSP | `Tsc::check_sync_target` |
| `detect_art()` | per-ART (Always-Running-Timer) detect | `Tsc::detect_art` |
| `time_cpufreq_notifier()` | per-cpufreq drift handler | `Tsc::cpufreq_notifier` |
| `tsc_save_sched_clock_state()` / `tsc_restore_sched_clock_state()` | per-suspend save/restore | `Tsc::save_restore_sched_clock` |
| `cpu_khz` / `tsc_khz` | per-globals exported | `Tsc::CPU_KHZ` / `TSC_KHZ` |
| `__use_tsc` static key | per-fast-path gate | `Tsc::USE_TSC` |
| `tsc_unstable` | per-flag | `Tsc::UNSTABLE` |
| `tsc_clocksource_reliable` | per-flag | `Tsc::RELIABLE` |
| `tsc_watchdog` (TSC_WATCHDOG_DEFAULT/ON/OFF) | per-config tristate | `Tsc::WATCHDOG` |
| `notsc_setup()` / `tsc_setup()` | per-cmdline parsers | `Tsc::parse_notsc` / `parse_tsc` |

## Compatibility contract

REQ-1: struct cyc2ns_data (per-CPU, double-buffered under seqcount_latch):
- cyc2ns_mul: u32 — multiplier (khz, NSEC_PER_MSEC).
- cyc2ns_shift: u32 — shift, clamped to ≤ 31 (perf_event_mmap_page ABI).
- cyc2ns_offset: u64 — additive offset preserving continuity across re-scale.

REQ-2: tsc_early_init():
- if !X86_FEATURE_TSC: return.
- if is_early_uv_system(): return.
- snp_secure_tsc_init().
- if !determine_cpu_tsc_frequencies(early=true): return.
- tsc_enable_sched_clock().

REQ-3: tsc_init():
- if !X86_FEATURE_TSC: setup_clear_cpu_cap(X86_FEATURE_TSC_DEADLINE_TIMER); return.
- /* Swap early calibrator for late one if still pointing at early stub */
- if x86_platform.calibrate_cpu == native_calibrate_cpu_early:
  - x86_platform.calibrate_cpu = native_calibrate_cpu.
- if tsc_khz == 0:
  - if !determine_cpu_tsc_frequencies(early=false):
    - mark_tsc_unstable("could not calculate TSC khz").
    - setup_clear_cpu_cap(X86_FEATURE_TSC_DEADLINE_TIMER).
    - return.
  - tsc_enable_sched_clock().
- cyc2ns_init_secondary_cpus().
- if !no_sched_irq_time: enable_sched_clock_irqtime().
- lpj_fine = get_loops_per_jiffy().
- check_system_tsc_reliable().
- if unsynchronized_tsc(): mark_tsc_unstable("TSCs unsynchronized"); return.
- if tsc_clocksource_reliable ∨ tsc_watchdog == TSC_WATCHDOG_OFF: tsc_disable_clocksource_watchdog().
- clocksource_register_khz(&clocksource_tsc_early, tsc_khz).
- detect_art().

REQ-4: determine_cpu_tsc_frequencies(early):
- WARN_ON(cpu_khz ∨ tsc_khz).
- if early:
  - cpu_khz = x86_platform.calibrate_cpu().
  - tsc_khz = tsc_early_khz ? tsc_early_khz : x86_platform.calibrate_tsc().
- else:
  - WARN_ON(x86_platform.calibrate_cpu != native_calibrate_cpu).
  - cpu_khz = pit_hpet_ptimer_calibrate_cpu().
- /* Trust non-zero tsc_khz; sanity-check cpu_khz within 10% */
- if tsc_khz == 0: tsc_khz = cpu_khz.
- else if abs(cpu_khz - tsc_khz) * 10 > tsc_khz: cpu_khz = tsc_khz.
- if tsc_khz == 0: return false.
- pr_info("Detected ... MHz processor / TSC").
- return true.

REQ-5: native_calibrate_tsc() — CPUID 0x15 path:
- if vendor != INTEL: return 0.
- if cpuid_level < CPUID_LEAF_TSC (0x15): return 0.
- cpuid(0x15, &eax_denominator, &ebx_numerator, &ecx_hz, &edx).
- if ebx_numerator == 0 ∨ eax_denominator == 0: return 0.
- crystal_khz = ecx_hz / 1000.
- /* Denverton hardcode 25 MHz */
- if crystal_khz == 0 ∧ vfm == INTEL_ATOM_GOLDMONT_D: crystal_khz = 25000.
- if crystal_khz != 0: setup_force_cpu_cap(X86_FEATURE_TSC_KNOWN_FREQ).
- /* Skylake/Kabylake derive crystal from CPUID 0x16 base MHz */
- if crystal_khz == 0 ∧ cpuid_level >= CPUID_LEAF_FREQ (0x16):
  - cpuid(0x16, &eax_base_mhz, ...).
  - crystal_khz = eax_base_mhz * 1000 * eax_denominator / ebx_numerator.
- if crystal_khz == 0: return 0.
- if vfm == INTEL_ATOM_GOLDMONT: setup_force_cpu_cap(X86_FEATURE_TSC_RELIABLE).
- /* Reuse for LAPIC timer */
- lapic_timer_period = crystal_khz * 1000 / HZ.
- return crystal_khz * ebx_numerator / eax_denominator.

REQ-6: cpu_khz_from_cpuid() — CPUID 0x16 path:
- if vendor != INTEL ∨ cpuid_level < CPUID_LEAF_FREQ: return 0.
- cpuid(0x16, &eax_base_mhz, &ebx_max_mhz, &ecx_bus_mhz, &edx).
- return eax_base_mhz * 1000.

REQ-7: pit_hpet_ptimer_calibrate_cpu():
- Run PIT, HPET, ACPI PM-timer calibrators in parallel; cross-check.
- Returns tsc_khz consistent across two of three within tolerance.

REQ-8: quick_pit_calibrate() — fast PIT MSB edge detection:
- Loop pit_expect_msb / pit_verify_msb.
- Return tsc_khz computed from ~50 ms PIT window.

REQ-9: __cycles_2_ns(cyc):
- /* Lock-free read via seqcount-latch */
- __cyc2ns_read(&data).
- ns = data.cyc2ns_offset + mul_u64_u32_shr(cyc, data.cyc2ns_mul, data.cyc2ns_shift).
- return ns.

REQ-10: __set_cyc2ns_scale(khz, cpu, tsc_now):
- ns_now = cycles_2_ns(tsc_now).
- clocks_calc_mult_shift(&mul, &shift, khz, NSEC_PER_MSEC, 0).
- /* shift ≤ 31 invariant */
- if shift == 32: shift = 31; mul >>= 1.
- offset = ns_now - mul_u64_u32_shr(tsc_now, mul, shift).
- /* Seqcount-latch update: data[0] then latch then data[1] */
- write_seqcount_latch_begin(&c2n.seq).
- c2n.data[0] = data.
- write_seqcount_latch(&c2n.seq).
- c2n.data[1] = data.
- write_seqcount_latch_end(&c2n.seq).

REQ-11: native_sched_clock() — noinstr:
- if static_branch_likely(&__use_tsc): return __cycles_2_ns(rdtsc()).
- /* Fallback: jiffies-based */
- return (jiffies_64 - INITIAL_JIFFIES) * (1_000_000_000 / HZ).

REQ-12: read_tsc(cs) — clocksource .read callback:
- return rdtsc_ordered().

REQ-13: clocksource_tsc_early (rating 299):
- .read = read_tsc.
- .mask = CLOCKSOURCE_MASK(64).
- .flags = CLOCK_SOURCE_IS_CONTINUOUS | CLOCK_SOURCE_MUST_VERIFY.
- .vdso_clock_mode = VDSO_CLOCKMODE_TSC.

REQ-14: clocksource_tsc (rating 300):
- .read = read_tsc.
- .mask = CLOCKSOURCE_MASK(64).
- .flags = CLOCK_SOURCE_IS_CONTINUOUS | CLOCK_SOURCE_CAN_INLINE_READ | CLOCK_SOURCE_MUST_VERIFY | CLOCK_SOURCE_HAS_COUPLED_CLOCK_EVENT.
- .vdso_clock_mode = VDSO_CLOCKMODE_TSC.
- .mark_unstable = tsc_cs_mark_unstable.
- .tick_stable = tsc_cs_tick_stable.
- .resume = tsc_resume.

REQ-15: tsc_refine_calibration_work() — DELAYED_WORK:
- if tsc_unstable: goto unreg.
- First call: hpet = is_hpet_enabled(); tsc_start = tsc_read_refs(); reschedule HZ.
- Later call:
  - tsc_stop = tsc_read_refs().
  - if ref_start == ref_stop: bail.
  - delta = (tsc_stop - tsc_start) * 1_000_000.
  - freq = hpet ? calc_hpet_ref(delta, ref_start, ref_stop) : calc_pmtimer_ref(...).
  - /* X86_FEATURE_TSC_KNOWN_FREQ: warn-only on >500 ppm divergence */
  - if !known_freq ∧ abs(tsc_khz - freq) > tsc_khz/100: goto out.
  - tsc_khz = freq.
  - clocksource_tsc.flags |= CLOCK_SOURCE_CALIBRATED.
  - lapic_update_tsc_freq().
  - for_each_possible_cpu(cpu): set_cyc2ns_scale(tsc_khz, cpu, tsc_stop).
- out:
  - if X86_FEATURE_ART: have_art = true; clocksource_tsc.base = &art_base_clk.
  - /* Carry VALID_FOR_HRES from tsc-early */
  - clocksource_register_khz(&clocksource_tsc, tsc_khz).
- unreg:
  - clocksource_unregister(&clocksource_tsc_early).

REQ-16: check_system_tsc_reliable():
- Geode-LX RTSC_SUSP path -> tsc_clocksource_reliable = 1.
- if X86_FEATURE_TSC_RELIABLE: tsc_clocksource_reliable = 1.
- /* Watchdog auto-disable when ALL of: CONSTANT_TSC ∧ NONSTOP_TSC ∧ TSC_ADJUST ∧ packages <= 4 */
- if CONSTANT_TSC ∧ NONSTOP_TSC ∧ TSC_ADJUST ∧ topology_max_packages() <= 4:
  - tsc_disable_clocksource_watchdog().

REQ-17: unsynchronized_tsc():
- if !X86_FEATURE_TSC ∨ tsc_unstable: return 1.
- if SMP ∧ apic_is_clustered_box(): return 1.
- if X86_FEATURE_CONSTANT_TSC: return 0.
- if tsc_clocksource_reliable: return 0.
- if vendor != INTEL ∧ topology_max_packages() > 1: return 1.
- return 0.

REQ-18: mark_tsc_unstable(reason):
- if tsc_unstable: return.
- tsc_unstable = 1.
- if using_native_sched_clock(): clear_sched_clock_stable().
- pr_info("Marking TSC unstable due to %s", reason).
- clocksource_mark_unstable(&clocksource_tsc_early).
- clocksource_mark_unstable(&clocksource_tsc).

REQ-19: tsc_cs_mark_unstable(cs) — watchdog callback:
- if tsc_unstable: return.
- tsc_unstable = 1.
- if using_native_sched_clock(): clear_sched_clock_stable().
- pr_info("Marking TSC unstable due to clocksource watchdog").

REQ-20: detect_art():
- if cpuid_level < CPUID_LEAF_TSC: return.
- if HYPERVISOR ∨ !NONSTOP_TSC ∨ !TSC_ADJUST ∨ tsc_async_resets: return.
- cpuid(CPUID_LEAF_TSC, &art_base_clk.denominator, &art_base_clk.numerator, &art_base_clk.freq_khz, &unused).
- art_base_clk.freq_khz /= KHZ.
- if art_base_clk.denominator < ART_MIN_DENOMINATOR: return.
- rdmsrq(MSR_IA32_TSC_ADJUST, art_base_clk.offset).
- setup_force_cpu_cap(X86_FEATURE_ART).

REQ-21: Per-AP TSC sync (tsc_sync.c — referenced):
- check_tsc_sync_target() runs on AP from start_secondary BEFORE set_cpu_online.
- BSP latches its TSC; AP latches its TSC; |delta| > threshold -> mark_tsc_unstable.
- tsc_store_and_check_tsc_adjust(bool bootcpu):
  - rdmsrq(MSR_IA32_TSC_ADJUST, cur).
  - if bootcpu: ref_tsc_adjust = cur.
  - else: bootval = per_cpu(tsc_adjust, cpu); if diff > threshold ∧ X86_FEATURE_TSC_ADJUST: wrmsr to align; pr_warn.

REQ-22: Per-vDSO fast-path:
- clocksource.vdso_clock_mode == VDSO_CLOCKMODE_TSC.
- tsc_cs_enable() -> vclocks_set_used(VDSO_CLOCKMODE_TSC).
- __vdso_clock_gettime in userspace reads rdtsc + per-CPU cyc2ns from vvar page (no syscall).

REQ-23: tsc_setup() boot-param parser ("tsc=..."):
- "reliable" -> tsc_clocksource_reliable = 1.
- "noirqtime" -> no_sched_irq_time = 1.
- "unstable" -> mark_tsc_unstable("boot parameter").
- "nowatchdog" -> tsc_watchdog = TSC_WATCHDOG_OFF.
- "recalibrate" -> tsc_force_recalibrate = 1.
- "watchdog" -> tsc_watchdog = TSC_WATCHDOG_ON.

REQ-24: notsc_setup() — "notsc":
- mark_tsc_unstable("boot parameter notsc") on X86_TSC, else clear X86_FEATURE_TSC in cpu/common.c.

REQ-25: tsc_save_sched_clock_state() / tsc_restore_sched_clock_state():
- Suspend: capture per-CPU cyc2ns snapshot.
- Resume: re-apply offset such that monotonicity is preserved.

REQ-26: time_cpufreq_notifier(nb, val, data):
- /* Pre-CONSTANT_TSC: when CPU frequency changes the TSC frequency changes too */
- Mark TSC unstable on transition unless X86_FEATURE_CONSTANT_TSC.

## Acceptance Criteria

- [ ] AC-1: tsc_init bails cleanly if !X86_FEATURE_TSC (no clocksource registered).
- [ ] AC-2: native_calibrate_tsc returns 0 on non-Intel or cpuid_level < 0x15.
- [ ] AC-3: native_calibrate_tsc sets X86_FEATURE_TSC_KNOWN_FREQ when crystal_khz > 0.
- [ ] AC-4: determine_cpu_tsc_frequencies clamps cpu_khz to tsc_khz when |cpu_khz - tsc_khz|*10 > tsc_khz.
- [ ] AC-5: cyc2ns_shift never exceeds 31 (perf_event_mmap_page ABI).
- [ ] AC-6: native_sched_clock falls back to jiffies when __use_tsc static-key disabled.
- [ ] AC-7: mark_tsc_unstable is idempotent (early return when tsc_unstable already set).
- [ ] AC-8: unsynchronized_tsc returns 1 on clustered APIC box.
- [ ] AC-9: check_system_tsc_reliable disables watchdog only when CONSTANT_TSC ∧ NONSTOP_TSC ∧ TSC_ADJUST ∧ packages <= 4.
- [ ] AC-10: tsc_refine_calibration_work refuses delta < 1% to update tsc_khz unless TSC_KNOWN_FREQ allows warn-only.
- [ ] AC-11: clocksource_tsc rating (300) > clocksource_tsc_early rating (299), so tsc replaces tsc-early when registered.
- [ ] AC-12: tsc_cs_enable sets vdso_clock_mode = VDSO_CLOCKMODE_TSC.
- [ ] AC-13: detect_art bails inside hypervisor or without NONSTOP_TSC + TSC_ADJUST.
- [ ] AC-14: tsc=unstable boot param immediately marks TSC unstable.
- [ ] AC-15: Per-AP check_tsc_sync_target marks TSC unstable on sync failure.

## Architecture

```
struct Cyc2NsData {                     // per-CPU, double-buffered
  cyc2ns_mul:    u32,                   // multiplier
  cyc2ns_shift:  u32,                   // shift, <= 31
  cyc2ns_offset: u64,                   // additive offset
}

struct Cyc2Ns {                         // per-CPU container
  seq: SeqcountLatch,
  data: [Cyc2NsData; 2],                // latch index 0/1
}

struct Clocksource {
  name: &'static str,                   // "tsc-early" | "tsc"
  rating: u32,                          // 299 | 300
  read: fn(&Clocksource) -> u64,        // rdtsc_ordered
  mask: u64,                            // CLOCKSOURCE_MASK(64) = !0
  flags: u32,                           // CLOCK_SOURCE_*
  id: u32,                              // CSID_X86_TSC_EARLY | CSID_X86_TSC
  vdso_clock_mode: u32,                 // VDSO_CLOCKMODE_TSC
  enable: fn,
  resume: fn,
  mark_unstable: fn,
  tick_stable: fn,
}

struct TscGlobals {
  cpu_khz: u32,
  tsc_khz: u32,
  tsc_unstable: bool,
  tsc_clocksource_reliable: bool,
  tsc_watchdog: TscWatchdog,            // DEFAULT | ON | OFF
  tsc_force_recalibrate: bool,
  use_tsc: StaticKey,                   // __use_tsc
  art_base_clk: SystemCounterval,
  have_art: bool,
}
```

`Tsc::early_init()`:
1. if !cpu_feature(X86_FEATURE_TSC): return.
2. if is_early_uv_system(): return.
3. snp_secure_tsc_init().
4. if !determine_frequencies(true): return.
5. enable_sched_clock().

`Tsc::init()`:
1. if !cpu_feature(X86_FEATURE_TSC):
   - setup_clear_cpu_cap(X86_FEATURE_TSC_DEADLINE_TIMER); return.
2. if x86_platform.calibrate_cpu == native_calibrate_cpu_early:
   - x86_platform.calibrate_cpu = native_calibrate_cpu.
3. if tsc_khz == 0:
   - if !determine_frequencies(false):
     - mark_unstable("could not calculate TSC khz").
     - setup_clear_cpu_cap(X86_FEATURE_TSC_DEADLINE_TIMER); return.
   - enable_sched_clock().
4. cyc2ns_init_secondary_cpus().
5. if !no_sched_irq_time: enable_sched_clock_irqtime().
6. lpj_fine = get_loops_per_jiffy().
7. check_system_reliable().
8. if unsynchronized(): mark_unstable("TSCs unsynchronized"); return.
9. if reliable ∨ watchdog == OFF: disable_watchdog().
10. clocksource_register_khz(&CLOCKSOURCE_TSC_EARLY, tsc_khz).
11. detect_art().

`Tsc::calibrate_native() -> u32`:
1. if vendor != INTEL ∨ cpuid_level < 0x15: return 0.
2. cpuid(0x15, &den, &num, &hz, &_).
3. if num == 0 ∨ den == 0: return 0.
4. crystal_khz = hz / 1000.
5. if crystal_khz == 0 ∧ vfm == INTEL_ATOM_GOLDMONT_D: crystal_khz = 25000.
6. if crystal_khz != 0: setup_force_cpu_cap(X86_FEATURE_TSC_KNOWN_FREQ).
7. if crystal_khz == 0 ∧ cpuid_level >= 0x16:
   - cpuid(0x16, &base, &_, &_, &_); crystal_khz = base * 1000 * den / num.
8. if crystal_khz == 0: return 0.
9. if vfm == INTEL_ATOM_GOLDMONT: setup_force_cpu_cap(X86_FEATURE_TSC_RELIABLE).
10. lapic_timer_period = crystal_khz * 1000 / HZ.
11. return crystal_khz * num / den.

`Tsc::set_cyc2ns_scale(khz, cpu, tsc_now)`:
1. local_irq_save(flags).
2. sched_clock_idle_sleep_event().
3. if khz != 0: __set_cyc2ns_scale(khz, cpu, tsc_now).
4. sched_clock_idle_wakeup_event().
5. local_irq_restore(flags).

`Tsc::__set_cyc2ns_scale(khz, cpu, tsc_now)`:
1. ns_now = cycles_to_ns(tsc_now).
2. clocks_calc_mult_shift(&mul, &shift, khz, NSEC_PER_MSEC, 0).
3. if shift == 32: shift = 31; mul >>= 1.            /* perf ABI invariant */
4. offset = ns_now - mul_u64_u32_shr(tsc_now, mul, shift).
5. c2n = per_cpu_ptr(&CYC2NS, cpu).
6. write_seqcount_latch_begin(&c2n.seq).
7. c2n.data[0] = Cyc2NsData{ mul, shift, offset }.
8. write_seqcount_latch(&c2n.seq).
9. c2n.data[1] = Cyc2NsData{ mul, shift, offset }.
10. write_seqcount_latch_end(&c2n.seq).

`Tsc::native_sched_clock() -> u64` (noinstr):
1. if USE_TSC.is_enabled():
   - return cycles_to_ns_inner(rdtsc()).
2. return (jiffies_64 - INITIAL_JIFFIES) * (1_000_000_000 / HZ).

`Tsc::read_tsc(cs) -> u64`:
1. return rdtsc_ordered().

`Tsc::refine_calibration_work(work)` — delayed_work, HZ:
1. if tsc_unstable: goto unreg.
2. if tsc_start == ULLONG_MAX:
   - hpet = is_hpet_enabled().
   - tsc_start = tsc_read_refs(&ref_start, hpet).
   - schedule_delayed_work(&tsc_irqwork, HZ); return.
3. tsc_stop = tsc_read_refs(&ref_stop, hpet).
4. if ref_start == ref_stop: goto out.                /* no HPET/PM ticked */
5. if tsc_stop == ULLONG_MAX: goto restart.
6. delta = (tsc_stop - tsc_start) * 1_000_000.
7. freq = if hpet { calc_hpet_ref(delta, ref_start, ref_stop) } else { calc_pmtimer_ref(...) }.
8. if X86_FEATURE_TSC_KNOWN_FREQ:
   - if |tsc_khz - freq| > tsc_khz >> 11: pr_warn("TSC freq calibrated by CPUID/MSR differs ...").
   - pr_info("TSC freq recalibrated by [HPET|PM_TIMER]"); return.
9. if |tsc_khz - freq| > tsc_khz/100: goto out.
10. tsc_khz = freq; pr_info("Refined TSC clocksource calibration").
11. clocksource_tsc.flags |= CLOCK_SOURCE_CALIBRATED.
12. lapic_update_tsc_freq().
13. for_each_possible_cpu(cpu): set_cyc2ns_scale(tsc_khz, cpu, tsc_stop).
14. out:
    - if tsc_unstable: goto unreg.
    - if X86_FEATURE_ART: have_art = true; clocksource_tsc.base = &art_base_clk.
    - if clocksource_tsc_early.flags & CLOCK_SOURCE_VALID_FOR_HRES:
      - clocksource_tsc.flags |= CLOCK_SOURCE_VALID_FOR_HRES.
    - clocksource_register_khz(&clocksource_tsc, tsc_khz).
15. unreg:
    - clocksource_unregister(&clocksource_tsc_early).

`Tsc::mark_unstable(reason)`:
1. if tsc_unstable: return.
2. tsc_unstable = true.
3. if using_native_sched_clock(): clear_sched_clock_stable().
4. pr_info("Marking TSC unstable due to %s", reason).
5. clocksource_mark_unstable(&CLOCKSOURCE_TSC_EARLY).
6. clocksource_mark_unstable(&CLOCKSOURCE_TSC).

`Tsc::unsynchronized() -> bool`:
1. if !X86_FEATURE_TSC ∨ tsc_unstable: return true.
2. if SMP ∧ apic_is_clustered_box(): return true.
3. if X86_FEATURE_CONSTANT_TSC: return false.
4. if tsc_clocksource_reliable: return false.
5. if vendor != INTEL ∧ topology_max_packages() > 1: return true.
6. return false.

`Tsc::check_sync_target()` — AP side, from start_secondary:
1. ref_tsc, ref_jiffies = read_atomic(BSP_LATCHED).
2. cur_tsc = rdtsc().
3. delta = (cur_tsc - ref_tsc).abs().
4. if delta > tsc_sync_threshold: mark_unstable("check_tsc_sync_source failed").
5. store_check_adjust(false).                         /* bootcpu = false */

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cyc2ns_shift_le_31` | INVARIANT | per-__set_cyc2ns_scale: shift adjusted to ≤ 31. |
| `cyc2ns_offset_continuity` | INVARIANT | per-__set_cyc2ns_scale: ns_now after scale = ns_now before scale. |
| `seqcount_latch_double_buffer` | INVARIANT | per-Cyc2Ns: data[0] and data[1] both written under latch. |
| `mark_unstable_idempotent` | INVARIANT | per-mark_tsc_unstable: tsc_unstable already set ⟹ early return. |
| `clocksource_rating_strict` | INVARIANT | per-clocksources: tsc-early.rating (299) < tsc.rating (300). |
| `calibrate_native_intel_gate` | INVARIANT | per-native_calibrate_tsc: non-Intel ⟹ 0. |
| `refine_1pct_threshold` | INVARIANT | per-refine_calibration_work: !KNOWN_FREQ ∧ |tsc_khz - freq| > tsc_khz/100 ⟹ no update. |
| `watchdog_disable_conditions` | INVARIANT | per-check_system_tsc_reliable: requires CONSTANT_TSC ∧ NONSTOP_TSC ∧ TSC_ADJUST ∧ pkgs <= 4. |
| `sched_clock_use_tsc_gate` | INVARIANT | per-native_sched_clock: USE_TSC unset ⟹ jiffies fallback. |
| `detect_art_gate` | INVARIANT | per-detect_art: HYPERVISOR ∨ !NONSTOP_TSC ∨ !TSC_ADJUST ⟹ no ART. |

### Layer 2: TLA+

`arch/x86/tsc.tla`:
- States: UNINIT, EARLY_CALIBRATED, EARLY_REGISTERED, LATE_CALIBRATING, REFINED, STABLE, UNSTABLE.
- Properties:
  - `safety_no_double_register` — per-clocksource_tsc: registered at most once.
  - `safety_refine_replaces_early` — per-refine_calibration_work: clocksource_tsc_early unregistered after clocksource_tsc registered.
  - `safety_unstable_terminal` — per-mark_tsc_unstable: UNSTABLE ⟹ no transition back to STABLE.
  - `safety_cyc2ns_offset_preserved` — per-__set_cyc2ns_scale: ns(tsc_now) before == ns(tsc_now) after.
  - `safety_seq_latch_read_consistent` — per-__cyc2ns_read: reader sees a self-consistent (mul, shift, offset) triple.
  - `liveness_refine_terminates` — per-refine_calibration_work: eventually either registers or marks unstable.
  - `liveness_ap_sync_decides` — per-check_tsc_sync_target: AP decides PASS or marks unstable.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Tsc::init` post: cpu_khz > 0 ∧ tsc_khz > 0 ∨ tsc_unstable | `Tsc::init` |
| `Tsc::determine_frequencies` post: tsc_khz != 0 ⟹ ret true | `Tsc::determine_frequencies` |
| `Tsc::calibrate_native` post: ret > 0 ⟹ derived from CPUID 0x15 num/den | `Tsc::calibrate_native` |
| `Tsc::set_cyc2ns_scale` post: cyc2ns_shift ≤ 31 | `Tsc::set_cyc2ns_scale` |
| `Tsc::native_sched_clock` post: monotonic w.r.t. rdtsc on same CPU | `Tsc::native_sched_clock` |
| `Tsc::read_tsc` post: rdtsc_ordered serializes prior loads/stores | `Tsc::read_tsc` |
| `Tsc::refine_calibration_work` post: clocksource_tsc registered ∨ tsc_unstable | `Tsc::refine_calibration_work` |
| `Tsc::mark_unstable` post: tsc_unstable = true ∧ both clocksources marked | `Tsc::mark_unstable` |
| `Tsc::check_sync_target` post: |Δ| ≤ threshold ∨ tsc_unstable | `Tsc::check_sync_target` |
| `Tsc::cs_enable` post: vdso_clock_mode VCLOCK_TSC active | `Tsc::cs_enable` |

### Layer 4: Verus/Creusot functional

`Per-tsc_init pipeline: x86_platform.calibrate_cpu / calibrate_tsc -> determine_cpu_tsc_frequencies -> tsc_enable_sched_clock (cyc2ns_init_boot_cpu, static_branch_enable __use_tsc) -> cyc2ns_init_secondary_cpus -> clocksource_register_khz(tsc_early) -> tsc_refine_calibration_work (HPET / PM-timer cross-check) -> clocksource_register_khz(tsc) -> clocksource_unregister(tsc_early)` semantic equivalence: per-Documentation/x86/timekeeping.rst, per-Intel SDM Vol.3B §18.17 (Time-Stamp Counter), per-AMD64 APM Vol.2 §15.34.

Per-vDSO: `__vdso_clock_gettime(CLOCK_MONOTONIC, &ts)` reads rdtsc and per-CPU cyc2ns from vvar; latency target < 10 ns on modern x86. Path verified against `Documentation/arch/x86/vdso.rst` and `arch/x86/entry/vdso/vclock_gettime.c`.

## Hardening

(Inherits row-1 features from `arch/x86/00-overview.md` § Hardening.)

TSC reinforcement:

- **Per-CPU cyc2ns under seqcount-latch (double-buffer)** — defense against per-reader-tear during multiplier rescale.
- **Per-cyc2ns_shift clamp to ≤ 31** — defense against per-perf-event ABI break (mmap_page documented as 32-bit shift).
- **Per-clocksource watchdog (CLOCK_SOURCE_MUST_VERIFY)** — defense against per-undetected-TSC-drift (HPET / PM cross-check).
- **Per-AP check_tsc_sync_target before set_cpu_online** — defense against per-clock-skew-leaking-to-userspace.
- **Per-MSR_IA32_TSC_ADJUST sanitized via tsc_store_and_check_tsc_adjust** — defense against per-firmware-poisoned TSC origin.
- **Per-mark_tsc_unstable idempotent + propagates to both clocksources** — defense against per-half-marked state.
- **Per-tsc=unstable / notsc cmdline honored unconditionally** — defense against per-broken-CPUID lying about CONSTANT_TSC.
- **Per-hypervisor gate in detect_art** — defense against per-untrusted ART claims in a guest.
- **Per-SNP secure TSC init (snp_secure_tsc_init)** — defense against per-host TSC-manipulation in confidential VMs.
- **Per-cpufreq notifier marks TSC unstable on transition unless CONSTANT_TSC** — defense against per-P-state TSC-rescale racing sched_clock.
- **Per-tsc_refine_calibration_work 1 % threshold** — defense against per-spurious-recalibration on glitchy HPET.
- **Per-vDSO VDSO_CLOCKMODE_TSC enabled only after watchdog confirms stability** — defense against per-userspace-time-warp.
- **Per-suspend/resume tsc_verify_tsc_adjust + cyc2ns offset replay** — defense against per-firmware reset of TSC_ADJUST across S3.

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — `/sys/devices/system/clocksource/clocksource0/*` and `/proc/timer_list` reads through bounded `seq_printf` paths; no raw kernel-TSC bytes exposed.
- **PAX_KERNEXEC** — `tsc_clocksource.read` and `tsc_cs.read` fptr cells RO post-init; `vdso_clock` array constified before user-mode entry.
- **PAX_RANDKSTACK** — `clock_gettime` syscall fallback (TSC unavailable) inherits per-syscall RANDKSTACK; vDSO fast-path runs in user mode and is unaffected.
- **PAX_REFCOUNT** — `clocksource` refcount saturating on the `tsc-early` → `tsc` swap so concurrent watchdog runs cannot wrap.
- **PAX_MEMORY_SANITIZE** — `cyc2ns` double-buffer slots zeroed-on-replace; old multiplier never lingers in slab for an attacker to read via Spectre v1 gadget.
- **PAX_UDEREF** — N/A directly; downstream `__vdso_clock_gettime` user path enforced via vDSO Tier-3.
- **PAX_RAP / kCFI** — `clocksource.read`, `clocksource.enable`, `clocksource.disable` function pointers signed; `cpufreq_notifier_block` chain signed.
- **PAX_TIMING_SIDECHANNEL** — `RDTSC` not user-trappable on the BSP unless `CR4.TSD` set per-task via `TIF_NOTSC`; signal-handler / sigreturn cannot bypass.
- **GRKERNSEC_HIDESYM** — `vdso_clock`, `tsc_cs`, `__cyc2ns` addresses excluded from `/proc/kallsyms` for non-CAP_SYSLOG; `dump_kernel_offset` does not log TSC-deadline pointers.
- **GRKERNSEC_DMESG** — TSC calibration / `mark_tsc_unstable` prints gated by `dmesg_restrict`.
- **GRKERNSEC_TPE** — `tsc=unstable`, `notsc`, `clocksource=` cmdline accepted only at boot under lockdown.
- **GRKERNSEC_RDTSC** — optional opt-in to disable user-mode RDTSC on a per-cgroup basis (CR4.TSD propagated via `__switch_to_xtra`); blocks Spectre-style cache-line timing.
- **GRKERNSEC_SEV_TDX** — `snp_secure_tsc_init` mandatory in confidential VMs; do not trust host TSC.

Per-doc rationale: TSC is the primary side-channel oracle (Spectre, Meltdown, Foreshadow timing); RDTSC disablement per task, secure-TSC in confidential VMs, and RAP on the clocksource vtable directly raise the bar for cache-timing attacks while the watchdog + sanitize-on-replace prevent forged time leaking userland.

## Open Questions

- For confidential VMs (SEV-SNP / TDX) should Rookery require `snp_secure_tsc_init` even when guest CPUID claims CONSTANT_TSC + NONSTOP_TSC, or trust the host? Upstream trusts; Rookery may want stricter default.
- Should we expose `tsc=watchdog` as the default (rather than `TSC_WATCHDOG_DEFAULT` which auto-disables on small-package systems)? Trade-off: latency vs. detection of microcode-induced drift.
- Tolerance for `tsc_refine_calibration_work` 1 % threshold — keep upstream verbatim or tighten to 0.5 % once HPET stability is empirically verified on QEMU and target boards?
- vDSO clock-mode gating: register `CLOCKSOURCE_TSC` with VDSO_CLOCKMODE_TSC immediately (upstream) or defer one watchdog cycle to reduce blast radius of late-marking-unstable?

## Out of Scope

- arch/x86/kernel/tsc_sync.c — full TSC sync protocol (cited here; covered in dedicated Tier-3 if expanded)
- arch/x86/kernel/tsc_msr.c — MSR-based calibration fallback for low-end Atom (covered separately if expanded)
- kernel/time/clocksource.c — generic clocksource framework + watchdog (covered in `core/clocksource.md` Tier-3)
- kernel/time/sched_clock.c — generic sched_clock helpers (covered separately if expanded)
- arch/x86/entry/vdso/vclock_gettime.c — vDSO clock-gettime fast-path (covered in `vdso.md` companion)
- kernel/time/timekeeping.c — timekeeper core (covered in `core/timekeeping.md` Tier-3)
- ACPI PM-timer + HPET drivers (cross-referenced; covered separately if expanded)
- Implementation code
