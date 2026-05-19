# Tier-3: drivers/cpufreq/ — CPUFreq core (policy / driver / governor model + sysfs + freq scaling)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/00-overview.md
upstream-paths:
  - drivers/cpufreq/cpufreq.c
  - drivers/cpufreq/cpufreq_governor.c
  - drivers/cpufreq/cpufreq_governor_attr_set.c
  - drivers/cpufreq/freq_table.c
  - drivers/cpufreq/cpufreq_stats.c
  - drivers/cpufreq/cpufreq_performance.c
  - drivers/cpufreq/cpufreq_powersave.c
  - drivers/cpufreq/cpufreq_userspace.c
  - drivers/cpufreq/cpufreq_ondemand.c
  - drivers/cpufreq/cpufreq_conservative.c
  - include/linux/cpufreq.h
-->

## Summary

The cpufreq subsystem is the kernel's authoritative arbiter of CPU clock frequency. It exports three coupled object models — the **driver** (per-arch hardware backend that programs the PLL / MSR / SCMI mailbox / firmware coproc), the **policy** (per-cpufreq-domain control block aggregating one or more CPUs sharing a clock + voltage rail), and the **governor** (per-policy frequency-selection policy: `performance`, `powersave`, `userspace`, `ondemand`, `conservative`, `schedutil`). The core (`drivers/cpufreq/cpufreq.c`, ~3070 lines) drives lifecycle (per-CPU online/offline, suspend/resume, freq transitions), the sysfs surface (`/sys/devices/system/cpu/cpufreqN/`), pm_qos integration (max-freq / min-freq constraints from thermal + idle subsystems), and the transition-notifier chain (clients hooked via `cpufreq_register_notifier`).

This Tier-3 covers `cpufreq.c` core + `cpufreq_governor.c` (sampling-rate framework for ondemand/conservative) + `freq_table.c` (per-driver frequency-table helpers) + `cpufreq_stats.c` (per-policy residency histogram) + the simple governors (`performance`, `powersave`, `userspace`). Per-driver drivers (`intel_pstate.c`, `amd_pstate.c`, `acpi-cpufreq.c`, `cppc_cpufreq.c`, `cpufreq-dt.c`, etc.) are covered in driver-specific Tier-3 docs. The `schedutil` governor lives in `kernel/sched/cpufreq_schedutil.c` and is covered in `schedutil.md`.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct cpufreq_driver` | per-arch driver vtable (`init`, `verify`, `target`/`target_index`, `get`, `setpolicy`, `fast_switch`, `adjust_perf`, `suspend`, `resume`) | `drivers::cpufreq::Driver` |
| `struct cpufreq_policy` | per-cpufreq-domain control block (cpus, related_cpus, cur, min, max, governor, freq_table, dvfs_possible_from_any_cpu) | `drivers::cpufreq::Policy` |
| `struct cpufreq_governor` | per-governor vtable (`init`, `exit`, `start`, `stop`, `limits`, `store_setspeed`, `show_setspeed`, `flags`) | `drivers::cpufreq::Governor` |
| `cpufreq_register_driver(drv)` / `cpufreq_unregister_driver(drv)` | global driver install/uninstall (only one driver active) | `Subsystem::register_driver` / `_unregister_driver` |
| `cpufreq_register_governor(gov)` / `cpufreq_unregister_governor(gov)` | governor add/remove (list-based, multiple available) | `Subsystem::register_governor` / `_unregister_governor` |
| `cpufreq_cpu_get(cpu)` / `cpufreq_cpu_put(policy)` | per-CPU policy ref-grab/drop | `Policy::get` / `_put` |
| `cpufreq_driver_target(policy, freq, relation)` | request a target frequency (CPUFREQ_RELATION_L/H/C) | `Policy::driver_target` |
| `__cpufreq_driver_target(policy, freq, relation)` | unlocked variant called from governor under policy->rwsem | `Policy::__driver_target` |
| `cpufreq_driver_fast_switch(policy, target_freq)` | scheduler-context lockless freq set | `Policy::fast_switch` |
| `cpufreq_driver_adjust_perf(cpu, min_perf, target_perf, capacity)` | HWP-style perf-domain hint (intel_pstate passive) | `Policy::adjust_perf` |
| `cpufreq_freq_transition_begin(policy, freqs)` / `_end` | transition-notifier paired bracketing | `Policy::transition_begin` / `_end` |
| `cpufreq_register_notifier(nb, list)` / `_unregister_notifier` | policy + transition notifier chains | `Subsystem::register_notifier` / `_unregister_notifier` |
| `cpufreq_get(cpu)` / `cpufreq_quick_get(cpu)` / `cpufreq_quick_get_max(cpu)` | external query helpers | `Policy::get_freq` / `_quick_get` / `_quick_get_max` |
| `cpufreq_update_policy(cpu)` / `cpufreq_update_limits(cpu)` | force re-eval after pm_qos / thermal pressure change | `Policy::update` / `_update_limits` |
| `store_scaling_governor` / `show_scaling_governor` (sysfs ops) | governor swap entry point | `Sysfs::scaling_governor_*` |
| `store_scaling_min_freq` / `_max_freq` / `_setspeed` (sysfs ops) | per-policy bounds tunables | `Sysfs::scaling_min_freq_*` / `_max_freq_*` / `_setspeed_*` |
| `cpufreq_frequency_table_verify(policy, table)` / `_get_index(policy, freq)` | table helpers | `FreqTable::verify` / `_get_index` |
| `cpufreq_table_index_unsorted(policy, target_freq, relation, idx)` | freq->index lookup with relation | `FreqTable::index_unsorted` |
| `cpufreq_stats_create_table(policy)` / `_record_transition(policy, freq)` | per-policy residency stats | `Stats::create_table` / `_record_transition` |
| `cpufreq_supports_freq_invariance()` / `arch_set_freq_scale(cpus, cur, max)` | scheduler invariance signal | `Subsystem::supports_freq_invariance` / `arch_set_freq_scale` |

## Compatibility contract

REQ-1: Exactly one `cpufreq_driver` may be registered globally at a time; `cpufreq_register_driver` returns -EEXIST otherwise. Multiple governors may be registered concurrently (linked list `cpufreq_governor_list`).

REQ-2: Per-CPU online: cpuhp callback `cpuhp_cpufreq_online` invokes `cpufreq_online(cpu)` which either creates a new policy (when no related-policy exists) or attaches the CPU to an existing policy (when `related_cpus` already contains it).

REQ-3: Per-policy lifecycle: `init` (driver populates min/max/freq_table/transition_latency) → `cpufreq_init_governor` (governor `.init` + `.start`) → ready. `exit` reverses: governor `.stop` + `.exit` → driver `.exit` → kobject release.

REQ-4: Frequency transitions are bracketed by `cpufreq_freq_transition_begin`/`_end` so that the SRCU transition-notifier chain is invoked with `CPUFREQ_PRECHANGE` / `CPUFREQ_POSTCHANGE` (clients: tsc rescaling on x86 non-invariant TSC, cpufreq_stats, devfreq-event correlations).

REQ-5: Fast-path: drivers with `fast_switch` callback support invocation from scheduler context (rq->lock held, no sleeping); schedutil uses this. Slow-path drivers schedule a kworker via governor.

REQ-6: pm_qos integration: per-policy `min_freq_req` + `max_freq_req` `freq_qos_request` nodes track aggregate min/max constraints from thermal, userland, idle subsystems; `cpufreq_update_limits` re-runs governor's `.limits` callback on aggregate change.

REQ-7: Sysfs surface per `/sys/devices/system/cpu/cpufreq/policyN/`: `scaling_min_freq`, `scaling_max_freq`, `scaling_cur_freq`, `scaling_governor`, `scaling_available_governors`, `scaling_driver`, `cpuinfo_min_freq`, `cpuinfo_max_freq`, `cpuinfo_cur_freq`, `cpuinfo_transition_latency`, `affected_cpus`, `related_cpus`, plus per-governor tunables under `scaling_governor` directory.

REQ-8: Frequency-invariance: when `cpufreq_supports_freq_invariance()` is true (set per-driver via `CPUFREQ_HAVE_FREQ_INVARIANCE` flag + `arch_set_freq_scale`), scheduler PELT calculations factor (cur/max) into utilization so a 50%-of-max task computes a util proportional to half of max-perf capacity.

REQ-9: Suspend/resume: `cpufreq_suspend()` calls per-policy `.suspend` then sets `cpufreq_suspended`; governors freeze; resume reverses + re-invokes `.limits` to re-pin frequency.

REQ-10: Boost: per-driver `boost` flag indicates turbo / boost frequencies above `cpuinfo_max_freq`; toggled via `/sys/devices/system/cpu/cpufreq/boost` (CAP_SYS_ADMIN).

REQ-11: Default governor selection: cmdline `cpufreq.default_governor=` or CONFIG_CPU_FREQ_DEFAULT_GOV_* picks initial governor; falls back to `performance` if requested governor not registered.

REQ-12: EAS readiness: `cpufreq_ready_for_eas(cpu_mask)` returns true iff every CPU in the mask runs schedutil governor — gate condition for Energy Aware Scheduling.

## Acceptance Criteria

- [ ] AC-1: `cpufreq-info -p` reports per-policy current frequency, governor, min/max consistent with `/proc/cpuinfo` MHz column on Intel + AMD + ARM reference HW.
- [ ] AC-2: Governor swap via `echo schedutil > /sys/.../scaling_governor` succeeds across all standard governors (`performance`, `powersave`, `userspace`, `ondemand`, `conservative`, `schedutil`).
- [ ] AC-3: pm_qos cap: `cpupower frequency-set -u 1.5GHz` clamps observed freq under stress-ng workload.
- [ ] AC-4: Suspend/resume: S3 cycle restores per-policy governor + min/max + cur consistent with pre-suspend state.
- [ ] AC-5: CPU hotplug: `echo 0 > /sys/devices/system/cpu/cpuN/online` followed by `echo 1` re-attaches CPU to its policy without leak.
- [ ] AC-6: Stats: `/sys/.../stats/time_in_state` + `/sys/.../stats/trans_table` populate consistent counters under workload variation.
- [ ] AC-7: Notifier chain test: a transition-notifier registered in a test module observes paired PRECHANGE/POSTCHANGE for every transition.
- [ ] AC-8: kselftest `tools/testing/selftests/cpufreq/` baseline passes.
- [ ] AC-9: Boost toggle: `echo 0 > /sys/.../cpufreq/boost` caps observed turbo freq; `echo 1` re-enables.

## Architecture

`Policy` lives in `drivers::cpufreq::Policy`:

```
struct Policy {
  cpu: u32,                      // owner CPU
  cpus: Cpumask,                 // online CPUs in this policy
  related_cpus: Cpumask,         // all possible CPUs in this policy
  real_cpus: Cpumask,            // CPUs that have ever been online
  shared_type: PolicyShared,     // NONE / HW / SW
  cur: AtomicU32,                // current freq (kHz)
  suspend_freq: u32,             // freq to restore on resume
  min: u32,                      // active min (post pm_qos)
  max: u32,                      // active max
  policy: u32,                   // POLICY_PERFORMANCE / _POWERSAVE / _UNKNOWN
  last_policy: u32,
  governor: KArc<Governor>,
  governor_data: AtomicPtr,      // per-governor private
  cpuinfo: CpuInfo {
    min_freq: u32, max_freq: u32, transition_latency: u32,
  },
  freq_table: Option<&'static [FreqTableEntry]>,
  driver_data: AtomicPtr,        // per-driver private
  rwsem: RwSemaphore,            // governor + limit serialization
  transition_lock: RawSpinlock,  // notifier-chain serialization
  transition_ongoing: AtomicBool,
  transition_task: AtomicPtr<TaskStruct>,
  min_freq_req: FreqQosRequest,
  max_freq_req: FreqQosRequest,
  constraints: FreqConstraints,  // pm_qos aggregate
  fast_switch_possible: bool,
  fast_switch_enabled: bool,
  strict_target: bool,
  efficiencies_available: bool,
  transition_delay_us: u32,
  dvfs_possible_from_any_cpu: bool,
  kobj: KObject,
  nb_min: NotifierBlock,
  nb_max: NotifierBlock,
  update: WorkStruct,            // deferred update kworker
  cdev: Option<&'static ThermalCoolingDevice>,
  stats: Option<KBox<Stats>>,
}
```

Subsystem init `Subsystem::init`:
1. `cpufreq_core_init()` creates global `cpufreq` kobject under `/sys/devices/system/cpu/`.
2. Per-arch boot driver registration: x86 `intel_pstate` (or `acpi-cpufreq`), ARM `cpufreq-dt`, etc. — exactly one wins.
3. `cpufreq_register_driver` registers `cpufreq_interface` subsys interface; cpuhp installs `cpuhp_cpufreq_online`.
4. Per-CPU online: `cpufreq_online(cpu)` walks `cpufreq_policy_list`, either attaches or builds a new policy.
5. Default governor activated; pm_qos constraints initialized with `[cpuinfo_min, cpuinfo_max]`.

Per-policy build `cpufreq_online`:
1. `driver.init(policy)` populates freq_table + cpuinfo + transition_latency + shared_type.
2. `cpufreq_table_validate_and_sort` validates + sorts freq_table.
3. `cpufreq_init_policy(policy)` picks default governor + invokes `cpufreq_set_policy` to install governor.
4. `cpufreq_stats_create_table(policy)` if CONFIG_CPU_FREQ_STAT.
5. sysfs kobject created; per-governor tunable group added under it.

Per-transition `__cpufreq_driver_target`:
1. Validate `target_freq` against `[policy->min, policy->max]`.
2. Resolve via `cpufreq_driver_resolve_freq` (freq_table relation-L/H/C).
3. `cpufreq_freq_transition_begin(policy, freqs)` — set `transition_ongoing`, invoke PRECHANGE notifier chain.
4. `driver.target_index(policy, idx)` or `driver.target(policy, target_freq)` — driver programs HW.
5. `cpufreq_freq_transition_end(policy, freqs, failed)` — invoke POSTCHANGE; update `policy->cur`; `cpufreq_stats_record_transition`.

Fast-path `cpufreq_driver_fast_switch`:
1. Caller (schedutil) holds rq->lock; no policy->rwsem.
2. `driver.fast_switch(policy, target_freq)` — driver writes MSR / mailbox without notifier chain.
3. Notifier chain SKIPPED in fast-path (transition_notifiers must accept this).

Governor lifecycle `cpufreq_set_policy`:
1. Hold policy->rwsem write.
2. If old governor: `gov.stop(policy)` + `gov.exit(policy)`.
3. New governor: `gov.init(policy)` + `gov.start(policy)`.
4. `gov.limits(policy)` to apply min/max.

pm_qos limit-change `cpufreq_notifier_min` / `_max`:
1. Notifier-block on `policy->constraints.min_freq` / `_max_freq` `freq_qos_request` chain.
2. Schedules `policy->update` workqueue work; work invokes `refresh_frequency_limits(policy)` → `cpufreq_set_policy(policy, NULL, policy->policy)` re-running governor.limits.

Suspend/resume `cpufreq_pm_notifier`:
1. PM_SUSPEND_PREPARE: per-policy `driver.suspend(policy)` → set `cpufreq_suspended` → governor stop.
2. PM_POST_SUSPEND: per-policy `driver.resume(policy)` → re-install governor + apply limits.

## Hardening

(Inherits row-1 features from `drivers/00-overview.md` § Hardening.)

cpufreq-core specific reinforcement:

- **policy->rwsem write-held across governor swap + limit change** — defense against concurrent `scaling_governor` write races yielding mismatched governor state.
- **transition_ongoing flag + transition_task pointer** — defense against re-entrant transition (notifier client calling cpufreq_driver_target within PRECHANGE).
- **SRCU notifier chain for transition** — RCU-grace-period bound; defense against notifier-unregister UAF while transition in flight.
- **freq_table validation at register** — defense against driver providing unsorted / overlapping freq_table corrupting `cpufreq_driver_resolve_freq`.
- **per-policy pm_qos default `[cpuinfo_min, cpuinfo_max]`** — defense against unbounded user store to `scaling_min_freq` / `_max_freq` driving HW out of spec.
- **boost flag CAP_SYS_ADMIN gated** — defense against unprivileged thermal-bypass via turbo enable.
- **CPU hotplug serialized via `cpus_read_lock`** — defense against concurrent online/offline + driver register racing.
- **Per-policy freq_qos_request init/destroy paired in `cpufreq_online`/`_offline`** — defense against qos-request leak across hotplug cycles.
- **Default-governor cmdline restricted to CPUFREQ_NAME_LEN** — defense against overrun via `cpufreq.default_governor=AAAA…`.
- **Stats trans_table allocation bounded by `nr_freqs * nr_freqs`** — defense against memory-bomb on freq-table register with extreme cardinality.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `cpufreq_policy`, `cpufreq_stats`, per-governor private (`menu_device`, `sugov_policy`, ondemand `cs_dbs_tuners`); sysfs read/write buffers strictly bounded by `CPUFREQ_NAME_LEN` and `PAGE_SIZE`.
- **PAX_KERNEXEC** — cpufreq core in W^X kernel text; `cpufreq_driver` and `cpufreq_governor` vtable pointers live in `__ro_after_init` after registration freeze.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `__cpufreq_driver_target`, `cpufreq_set_policy`, `cpufreq_update_policy`, and per-governor `.select`/`.limits` callbacks invoked from sysfs.
- **PAX_REFCOUNT** — saturating `refcount_t` on `cpufreq_policy` (`cpufreq_cpu_get`/`_put`) and per-governor data; overflow trap defeats hot-plug + governor-swap race UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `cpufreq_policy`, freq_table copies, stats trans_table, per-governor private; prevents leaked frequency-table addresses and per-CPU util history across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every sysfs store entry (`scaling_min_freq`, `scaling_max_freq`, `scaling_governor`, `scaling_setspeed`, `boost`); reject user-pointer deref outside `kstrtouint_from_user`-style helpers.
- **PAX_RAP / kCFI** — `cpufreq_driver` (`init`, `verify`, `target`, `target_index`, `get`, `setpolicy`, `fast_switch`, `adjust_perf`, `suspend`, `resume`), `cpufreq_governor` (`init`, `exit`, `start`, `stop`, `limits`), and transition-notifier-block callbacks marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms for cpufreq-core helpers and driver pointers behind CAP_SYSLOG; suppress `%p` in transition tracepoints.
- **GRKERNSEC_DMESG** — restrict cpufreq init banners and transition-failure diagnostics to CAP_SYSLOG so attackers cannot probe per-policy freq-table contents or governor selection via dmesg.
- **CAP_SYS_ADMIN strict on policy sysfs** — `scaling_min_freq`, `scaling_max_freq`, `scaling_setspeed`, `boost`, and `scaling_governor` writes require CAP_SYS_ADMIN in the cpu-controller cgroup's user namespace; defense against unprivileged downclock / overclock.
- **scaling-max-freq overwrite guard** — store-handler clamps the requested value to `[cpuinfo_min_freq, cpuinfo_max_freq]` before pm_qos publish; reject `INT_MAX` / negative inputs that would silently saturate.
- **Governor swap RAP** — governor vtable resolved through kCFI-typed pointer with each callback declared `__ro_after_init`; reject swap to a governor not on the registered list.
- **Transition-notifier-block install audit** — register/unregister logged via audit subsystem with caller capability + governor name; defense against silent notifier hijack from out-of-tree modules.

Rationale: cpufreq is on the boundary between unprivileged userspace (sysfs writers) and bare-metal hardware (PLL, voltage rails, package thermals). A relaxed sysfs CAP check, a tainted freq_table, or a missed transition-notifier RCU-grace lets userspace driver the silicon into thermal violation or stale-frequency UAFs. RAP/kCFI on driver and governor vtables, CAP_SYS_ADMIN on every tunable that touches HW limits, scaling-max-freq clamping, refcount overflow trapping on policy refs, and audit on notifier-block install turn cpufreq from "best-effort scaling" into a structural enforcement boundary that meets the thermal + voltage envelope guarantees the silicon spec demands.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- intel_pstate driver internals (covered in `intel-pstate.md`)
- schedutil governor (covered in `schedutil.md`)
- ondemand / conservative sampling governors (deferred to `ondemand.md` future Tier-3)
- ACPI _PPC / _PSS / _PCT parsing (covered in `acpi-cpufreq.md` future Tier-3)
- CPPC / SCMI / SCPI mailbox transports (covered in respective transport Tier-3 docs)
- AMD pstate driver (covered in `amd-pstate.md` future Tier-3)
- ARM-specific DT cpufreq drivers (covered per-platform)
- 32-bit-only paths
- Implementation code
