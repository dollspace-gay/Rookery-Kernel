# Tier-3: drivers/cpufreq/intel_pstate.c — Intel P-state driver (HWP, IA32_HWP_REQUEST, EPP, active vs passive)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/cpufreq/00-overview.md
upstream-paths:
  - drivers/cpufreq/intel_pstate.c
  - arch/x86/include/asm/msr-index.h
  - Documentation/admin-guide/pm/intel_pstate.rst
-->

## Summary

`intel_pstate` is the native Intel P-state driver — the default cpufreq backend on every Intel client / server from Sandy Bridge (2011) onward. It replaces the generic `acpi-cpufreq` because Intel parts expose richer hardware: per-core P-state selection independent of voltage rail (so cores in the same package can run at different frequencies), Hardware-Controlled P-States (HWP, introduced Skylake), MSR-driven request lanes, Energy Performance Preference (EPP) bias hint, hybrid (P-core + E-core) capacity scaling, and turbo boost limits programmable via `MSR_IA32_MISC_ENABLE`.

The driver operates in two registration modes selected by cmdline `intel_pstate={active,passive}` or by HWP presence: **active** mode (HWP-only, default on HWP-capable parts) registers a `cpufreq_driver` with `setpolicy` (policy is "performance" or "powersave"; the actual freq selection is delegated to the HW via HWP_REQUEST MSR), and **passive** mode registers a classic `target` driver where governors (schedutil typically) request explicit frequencies and the driver translates to a non-HWP `IA32_PERF_CTL` MSR write or to HWP_REQUEST_MIN/MAX/DESIRED. This Tier-3 covers `intel_pstate.c` (~3950 lines: per-CPU sample, HWP init, EPP plumbing, sysfs, ACPI _PSS fallback, hybrid scaling).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct cpudata` | per-CPU sample state + HWP cache + EPP cache + ACPI _PSS perf-domain | `drivers::intel_pstate::CpuData` |
| `struct pstate_data` | per-CPU min/max/turbo pstate + scaling factor | `drivers::intel_pstate::PstateData` |
| `struct global_params` | `no_turbo`, `turbo_disabled`, `min_perf_pct`, `max_perf_pct` global tunables | `drivers::intel_pstate::GlobalParams` |
| `struct sample` | APERF/MPERF/TSC delta sample (used in non-HWP busy estimation) | `drivers::intel_pstate::Sample` |
| `intel_pstate_init()` | subsys init: detect HWP, register driver, expose sysfs | `Subsystem::init` |
| `intel_pstate_register_driver(drv)` | register either `intel_pstate` (active) or `intel_cpufreq` (passive) `cpufreq_driver` | `Subsystem::register_driver` |
| `intel_pstate` (struct cpufreq_driver, active mode) | `setpolicy`, no `target_index` — pure HWP control | `Driver::active` |
| `intel_cpufreq` (struct cpufreq_driver, passive mode) | `target_index` + `fast_switch` + `adjust_perf` | `Driver::passive` |
| `intel_pstate_hwp_set(policy)` | program HWP_REQUEST MSR with min/max/desired/EPP from policy | `CpuData::hwp_set` |
| `intel_pstate_hwp_enable(cpu)` | enable HWP via `MSR_PM_ENABLE` write (one-way; cannot be disabled until reset) | `CpuData::hwp_enable` |
| `intel_pstate_hwp_offline(cpu)` / `_online(cpu)` | per-cpu HWP cache stash on offline + restore on online | `CpuData::hwp_offline` / `_online` |
| `store_energy_performance_preference` / `show_energy_performance_preference` | EPP sysfs handlers (per-cpu `energy_performance_preference`) | `Sysfs::epp_*` |
| `intel_pstate_set_epp(cpu, epp)` | write EPP byte to HWP_REQUEST (bits 31:24) | `CpuData::set_epp` |
| `intel_pstate_update_perf_limits(policy, min_perf, max_perf)` | compute per-cpu HWP_MIN/HWP_MAX from policy->min/max + global pct | `Policy::update_perf_limits` |
| `intel_pstate_get_hwp_cap(cpu)` | read `MSR_HWP_CAPABILITIES` for highest/guaranteed/most-efficient/lowest perf | `CpuData::get_hwp_cap` |
| `intel_pstate_init_acpi_perf_limits(policy)` | parse ACPI _PSS / _PPC for fallback non-HWP scaling | `Policy::init_acpi_perf_limits` |
| `intel_pstate_get_cpu_pstates(cpu)` | populate min/max/turbo pstates from `MSR_PLATFORM_INFO` + `MSR_TURBO_RATIO_LIMIT` | `CpuData::get_cpu_pstates` |
| `intel_pstate_update_util(data, time, flags)` | scheduler util-update hook for non-HWP passive mode | `CpuData::update_util` |
| `intel_pstate_update_util_hwp_local(data, time)` | scheduler hook in HWP passive (programs HWP_REQUEST.desired) | `CpuData::update_util_hwp_local` |
| `store_no_turbo` / `show_no_turbo` | global no-turbo sysfs (writes `MSR_IA32_MISC_ENABLE` IDA bit) | `Sysfs::no_turbo_*` |
| `store_max_perf_pct` / `_min_perf_pct` | global percent caps applied across all policies | `Sysfs::max_perf_pct_*` / `_min_perf_pct_*` |
| `intel_pstate_request_control_from_smm()` | tell SMM to relinquish P-state control via `MSR_MISC_PWR_MGMT` | `Subsystem::request_control_from_smm` |

## Compatibility contract

REQ-1: HWP detection: `X86_FEATURE_HWP` + `X86_FEATURE_HWP_NOTIFY` CPUID checks at init; HWP enabled via one-way `MSR_PM_ENABLE.HWP_ENABLE` write (bit 0). Once enabled it cannot be disabled until next platform reset.

REQ-2: Active mode (HWP-only): cpufreq policy is `performance` or `powersave` (`setpolicy` driver, no per-freq target); on policy change, write `MSR_HWP_REQUEST` with min=lowest, max=highest, desired=0 (HW autonomous), EPP per `energy_performance_preference` sysfs.

REQ-3: Passive mode: cpufreq governor (typically `schedutil`) chooses frequency; driver maps freq to perf-ratio + writes `MSR_HWP_REQUEST.desired` (HWP path) or `MSR_IA32_PERF_CTL` (legacy non-HWP path) on every scheduler util-update.

REQ-4: EPP (Energy Performance Preference, HWP only): 8-bit hint in `MSR_HWP_REQUEST` bits 31:24; sysfs accepts `default` / `performance` / `balance_performance` / `balance_power` / `power` or raw 0-255; per-CPU value cached + reapplied on resume.

REQ-5: Per-CPU sysfs `energy_performance_available_preferences` enumerates the allowed string set; `energy_performance_preference` is read/write CAP_SYS_ADMIN.

REQ-6: Global sysfs `/sys/devices/system/cpu/intel_pstate/`: `no_turbo`, `max_perf_pct`, `min_perf_pct`, `turbo_pct`, `num_pstates`, `status` ("active"/"passive"/"off"), `hwp_dynamic_boost` (HWP passive mode only).

REQ-7: ACPI _PSS fallback: when `intel_pstate=no_hwp` or HWP not available, driver parses ACPI _PSS perf-state list and writes `MSR_IA32_PERF_CTL` (legacy P-state architecture, pre-Skylake).

REQ-8: Hybrid (P-core + E-core, Alder Lake+): per-CPU `cpudata->hybrid_scaling_factor` allows P-core max-perf 39 and E-core max-perf 27 to map to comparable frequency ranges; scheduler invariance reports per-core scale-freq.

REQ-9: No-turbo: `MSR_IA32_MISC_ENABLE.NO_TURBO` bit (38) toggled per `no_turbo` sysfs; OS-level cap independent of BIOS hide-turbo. On HWP, also clamps `MSR_HWP_REQUEST.max` to guaranteed perf.

REQ-10: `intel_pstate_request_control_from_smm` writes `MSR_MISC_PWR_MGMT.LOCK_TM_BIT` to tell SMM-resident P-state daemons (some OEM platforms) to yield control to OS.

REQ-11: Power-Control EE (Energy Efficient) optimization disabled on specific Intel models via `intel_pstate_cpu_ee_disable_ids` x86_cpu_id table (Skylake server quirk).

REQ-12: Suspend/resume: HWP MSR cache stashed per-cpu on `intel_pstate_suspend` / restored in `_resume`; EPP reapplied; turbo state restored.

## Acceptance Criteria

- [ ] AC-1: `cat /sys/devices/system/cpu/intel_pstate/status` reports `active` on default boot of HWP-capable Intel HW.
- [ ] AC-2: `cat /sys/devices/system/cpu/cpufreq/policy0/scaling_driver` reports `intel_pstate` (active) or `intel_cpufreq` (passive).
- [ ] AC-3: EPP cycle: `echo performance > /sys/.../cpufreq/policy0/energy_performance_preference` → verify `MSR_HWP_REQUEST` bits 31:24 == perf EPP value via `rdmsr 0x774`.
- [ ] AC-4: No-turbo cycle: `echo 1 > /sys/.../intel_pstate/no_turbo` caps observed freq under stress-ng to guaranteed perf.
- [ ] AC-5: Passive-mode swap: `echo passive > status` re-registers as `intel_cpufreq`; governor swap to `schedutil` succeeds.
- [ ] AC-6: Suspend/resume: S3 cycle preserves EPP + governor + min/max consistent with pre-suspend.
- [ ] AC-7: Hybrid test (Alder Lake+): per-core max-perf ratio reported correctly via `lscpu --extended`; scheduler util reflects hybrid capacity.
- [ ] AC-8: Cmdline test: `intel_pstate=disable` boots with `acpi-cpufreq` instead; `intel_pstate=no_hwp` boots active driver with legacy `IA32_PERF_CTL` path.
- [ ] AC-9: ACPI _PSS test: on a part with HWP disabled via cmdline, `cpupower frequency-info` lists ACPI-declared frequencies.

## Architecture

`CpuData` lives in `drivers::intel_pstate::CpuData`:

```
struct CpuData {
  cpu: u32,
  policy: u32,                  // POLICY_PERFORMANCE / _POWERSAVE
  update_util: UpdateUtilData,  // scheduler hook
  update_util_set: AtomicBool,
  iowait_boost: u64,
  last_update: u64,
  pstate: PstateData {
    current_pstate: i32,
    min_pstate: i32,            // from MSR_PLATFORM_INFO[47:40]
    max_pstate: i32,             // guaranteed perf (HWP_CAP) or MAX_NON_TURBO
    max_pstate_physical: i32,
    turbo_pstate: i32,           // from MSR_TURBO_RATIO_LIMIT (1-core turbo)
    perf_ctl_scaling: i32,
    scaling: i32,                // freq-per-pstate (typically 100000 kHz)
    min_freq: u32, max_freq: u32, turbo_freq: u32,
  },
  vid: VidData,                  // Atom-platform voltage data
  last_sample_time: u64,
  aperf_mperf_shift: i32,
  prev_aperf: u64,
  prev_mperf: u64,
  prev_tsc: u64,
  sample: Sample,
  acpi_perf_data: Option<&'static AcpiProcessorPerformance>,
  valid_pss_table: bool,
  epp_default: i32,
  epp_powersave: i32,
  epp_policy: i32,
  epp_saved: i32,
  epp_cached: i32,
  hwp_req_cached: u64,           // last MSR_HWP_REQUEST written
  hwp_cap_cached: u64,           // MSR_HWP_CAPABILITIES read
  last_io_update: u64,
  capacity_perf: u32,
  sched_flags: u32,
  hwp_boost_min: u32,
  suspended: AtomicBool,
  pd_registered: AtomicBool,
  hybrid_scaling_factor: i32,    // 1024 for P-core, 540 for E-core typ.
}
```

Subsystem init `intel_pstate_init`:
1. CPUID check `X86_FEATURE_HWP` / `_HWP_NOTIFY` / `_HWP_EPP`; if HWP and no_hwp not set, `hwp_active = 1`.
2. `intel_pstate_request_control_from_smm()` — write `MSR_MISC_PWR_MGMT` LOCK bit.
3. Allocate per-cpu `cpudata` array via `vzalloc(num_possible_cpus * sizeof(*))`.
4. `intel_pstate_sysfs_expose_params()` — create global `/sys/devices/system/cpu/intel_pstate/` group.
5. Pick default driver: `intel_pstate` (active) if HWP available + not `intel_pstate=passive`, else `intel_cpufreq` (passive).
6. `intel_pstate_register_driver(default_driver)` — register with cpufreq core.
7. Per-CPU `intel_pstate_init_cpu` on policy attach: read `MSR_PLATFORM_INFO` + `MSR_TURBO_RATIO_LIMIT` + (HWP) `MSR_HWP_CAPABILITIES`.

Active-mode policy install `intel_pstate_set_policy`:
1. Map `policy->policy` (PERFORMANCE / POWERSAVE) to HWP_REQUEST shape:
   - PERFORMANCE: min=guaranteed, max=highest, desired=0 (HW autonomous), EPP=performance.
   - POWERSAVE: min=lowest, max=highest, desired=0, EPP=balance_power.
2. `intel_pstate_hwp_set(policy)` writes `MSR_HWP_REQUEST` (0x774) on every CPU in `policy->cpus`.

Passive-mode util-update `intel_pstate_update_util_hwp_local` (HWP) or `intel_pstate_update_util` (legacy):
1. Called from `cpufreq_update_util` in scheduler context (rq->lock held).
2. Sample APERF/MPERF/TSC deltas via `rdmsr` to compute `core_avg_perf` (HWP path skips this — HW does it).
3. Compute target pstate via PID controller (legacy) or directly from util (HWP passive: `hwp_req.desired = util_to_perf(util, max)`).
4. Write `MSR_HWP_REQUEST.desired` (bits 23:16) or `MSR_IA32_PERF_CTL` ratio.

EPP set `intel_pstate_set_epp`:
1. Clamp `epp` to `[EPP_PERFORMANCE, EPP_POWERSAVE]` (0-255).
2. RMW `MSR_HWP_REQUEST` (read cached → mask off bits 31:24 → OR epp << 24 → write).
3. Cache new value in `cpudata->hwp_req_cached`.

Hybrid scaling `intel_pstate_hybrid_hwp_perf_ctl_parity`:
1. Per-CPU type (P-core / E-core) via `MSR_PLATFORM_INFO` CPUID family/model + `intel_hybrid_scaling_factor` x86_cpu_id table.
2. Scaling factor applied: P-core 1024, E-core 540 (Alder Lake) — so a P-core "perf 39" reports the same cpu-capacity as E-core "perf 21".

ACPI _PSS fallback `intel_pstate_init_acpi_perf_limits`:
1. `acpi_processor_register_performance(cpu)` walks ACPI _PSS.
2. Build freq_table from _PSS entries (control_register = MSR `IA32_PERF_CTL` write value).
3. Mark `valid_pss_table = true`; driver routes `target_index` through ACPI table on legacy path.

## Hardening

(Inherits row-1 features from `drivers/cpufreq/00-overview.md` § Hardening.)

intel_pstate specific reinforcement:

- **HWP one-way enable** — `MSR_PM_ENABLE.HWP_ENABLE` set exactly once at init; never cleared.
- **`MSR_HWP_REQUEST` RMW serialized via per-cpu cache** — defense against partial-MSR-write race losing EPP / min / max bytes.
- **EPP value clamped to byte range** — defense against sysfs store of out-of-range integer.
- **No-turbo gate via `MSR_IA32_MISC_ENABLE.NO_TURBO`** — toggle CAP_SYS_ADMIN; bit set/clear is atomic + per-cpu.
- **ACPI _PSS table cardinality bounded by `CPUFREQ_TABLE_SIZE_MAX`** — defense against malformed _PSS exposing thousands of entries.
- **Per-CPU `cpudata` allocation `vzalloc` (per-cpu, num_possible_cpus)** — fixed at init; no late realloc.
- **`intel_pstate_request_control_from_smm` precedes any HWP write** — defense against SMM concurrently writing the same MSR.
- **HWP cap read once at init + on hotplug** — defense against repeated MSR read failure causing zero-cap divide.
- **EE-disable quirk per x86_cpu_id** — defense against Skylake-X PowerControlEE bug that produces stuck high P-state.
- **Per-CPU HWP_REQUEST stash on offline + restore on online** — defense against state loss across hotplug.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `cpudata`, `acpi_processor_performance` copies, and EPP-string parser buffers; per-CPU `vzalloc`'d region not exported via sysfs/debugfs.
- **PAX_KERNEXEC** — intel_pstate driver core in W^X kernel text; `cpufreq_driver` vtable (active + passive variants) lives in `__ro_after_init`.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `intel_pstate_hwp_set`, `intel_pstate_update_util`, `intel_pstate_set_epp`, and sysfs store handlers.
- **PAX_REFCOUNT** — saturating `refcount_t` on per-CPU cpudata pinning during policy operations; overflow trap defeats hotplug + sysfs-store race UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `cpudata`, EPP caches, HWP_REQUEST cached MSR values, and ACPI _PSS copies; prevents leaked MSR shapes and per-CPU sample residuals across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every sysfs store (`no_turbo`, `max_perf_pct`, `min_perf_pct`, `energy_performance_preference`, `status`); reject user-pointer deref outside `kstrtouint_from_user`.
- **PAX_RAP / kCFI** — active + passive `cpufreq_driver` (`init`, `verify`, `setpolicy`, `target_index`, `fast_switch`, `adjust_perf`, `suspend`, `resume`) and util-update callbacks marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms for intel_pstate helpers and per-CPU cpudata pointers behind CAP_SYSLOG; suppress `%p` in HWP-debug tracepoints.
- **GRKERNSEC_DMESG** — restrict HWP-enable banners, EPP-default tables, and EE-disable quirk diagnostics to CAP_SYSLOG so attackers cannot fingerprint silicon stepping via dmesg.
- **HWP MSR writes CAP_SYS_RAWIO** — every `MSR_HWP_REQUEST` write is gated by CAP_SYS_RAWIO in addition to CAP_SYS_ADMIN on the calling sysfs path; defense against ring-0 MSR manipulation via cpufreq policy spoof.
- **EPP perf-bias side-channel mitigation** — EPP changes coalesced to a per-policy work item that batches updates across `policy->cpus`; defense against EPP-toggle covert channel between SMT siblings reading frequency-dependent timing.
- **No-turbo gate audit** — `no_turbo` sysfs writes logged via audit subsystem with caller capability; defense against silent thermal-envelope removal by privileged-but-untrusted processes.
- **HWP cap freeze** — `MSR_HWP_CAPABILITIES` cached at init + on hotplug-online only; refuse mid-run re-read paths that could be coerced to expose adjacent MSR data via speculative read.
- **ACPI _PSS table sanitization** — _PSS entries validated against the freq_table cardinality limit and per-entry `core_frequency` / `power` ranges; refuse driver attach on malformed firmware tables instead of trusting BIOS.

Rationale: intel_pstate writes MSRs that directly drive the silicon's voltage + frequency planes — a sysfs-driven HWP_REQUEST or EPP store on the wrong CPU, with the wrong byte clipping, or under SMM contention can park a core, cap turbo across a socket, or expose adjacent MSR bytes via partial-RMW races. RAP/kCFI on the active and passive driver vtables, CAP_SYS_RAWIO on every HWP MSR write, EPP coalescing to defeat side channels, audit on no-turbo flips, and refcount overflow trapping on per-CPU cpudata turn intel_pstate from "trust the BIOS + trust userspace" into a structural enforcement boundary that meets the silicon spec's MSR-write ordering + EPP-byte clamping requirements.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- AMD pstate driver (covered in `amd-pstate.md` future Tier-3)
- Atom-platform VID data (legacy, deferred)
- ACPI CPPC transport (covered in `cppc-cpufreq.md` future Tier-3)
- HWP-notify interrupt path (deferred to interrupt-routing Tier-3)
- Per-package RAPL coordination (covered in `intel-rapl.md` future Tier-3)
- 32-bit-only paths
- Implementation code
