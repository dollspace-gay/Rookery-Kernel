# Tier-3: drivers/cpuidle/ — CPUIdle framework (governors menu/teo/ladder/haltpoll, C-state ladder, sysfs)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/00-overview.md
upstream-paths:
  - drivers/cpuidle/cpuidle.c
  - drivers/cpuidle/driver.c
  - drivers/cpuidle/governor.c
  - drivers/cpuidle/sysfs.c
  - drivers/cpuidle/poll_state.c
  - drivers/cpuidle/coupled.c
  - drivers/cpuidle/governors/menu.c
  - drivers/cpuidle/governors/teo.c
  - drivers/cpuidle/governors/ladder.c
  - drivers/cpuidle/governors/haltpoll.c
  - include/linux/cpuidle.h
-->

## Summary

The cpuidle subsystem is the kernel's framework for selecting + entering CPU idle states (C-states on x86, WFI/WFE plus PSCI states on ARM, SBI HSM states on RISC-V). It exposes three coupled object models — the **driver** (per-arch backend providing the per-C-state `.enter` callback that issues `MWAIT`/`WFI`/`PSCI_CPU_SUSPEND`), the **device** (per-CPU runtime control block holding `last_state_idx`, per-state usage counters, latency-req), and the **governor** (per-system selection policy: `menu`, `teo`, `ladder`, `haltpoll`). The core (`drivers/cpuidle/cpuidle.c`, ~830 lines) drives lifecycle (per-CPU device register / unregister, pause / resume, suspend-to-idle), the sysfs surface (`/sys/devices/system/cpu/cpuN/cpuidle/`), and the s2idle path which freezes the tick + enters the deepest enter_s2idle-capable state for whole-system suspend.

This Tier-3 covers `cpuidle.c` core + `driver.c` (per-driver registration / refcount / state-table validation) + `governor.c` (governor list + active-governor swap) + `sysfs.c` (per-state attributes group) + `poll_state.c` (synthetic POLL state-0) + `coupled.c` (ARM-only coupled-CPU rendezvous). Per-driver Tier-3 docs cover `intel_idle` (drivers/idle/intel_idle.c) and the per-governor algorithm details (covered in `governors.md`).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct cpuidle_driver` | per-arch driver (name, states[], state_count, cpumask, bctimer, governor) | `drivers::cpuidle::Driver` |
| `struct cpuidle_state` | per-C-state record (name, desc, exit_latency_ns, target_residency_ns, power_usage, flags, enter, enter_dead, enter_s2idle) | `drivers::cpuidle::State` |
| `struct cpuidle_device` | per-CPU runtime state (last_state_idx, last_residency_ns, poll_limit_ns, states_usage[], registered, enabled) | `drivers::cpuidle::Device` |
| `struct cpuidle_governor` | per-governor (rating, init, exit, enable, disable, select, reflect) | `drivers::cpuidle::Governor` |
| `struct cpuidle_state_usage` | per-CPU per-state usage counters (usage, time_ns, above, below, rejected, disable) | `drivers::cpuidle::StateUsage` |
| `cpuidle_register_driver(drv)` / `cpuidle_unregister_driver(drv)` | global driver install (only one active) | `Subsystem::register_driver` / `_unregister_driver` |
| `cpuidle_register_device(dev)` / `cpuidle_unregister_device(dev)` | per-CPU device install | `Subsystem::register_device` / `_unregister_device` |
| `cpuidle_register(drv, coupled_cpus)` | combined helper: driver + per-CPU devices | `Subsystem::register` |
| `cpuidle_register_governor(gov)` | append governor to `cpuidle_governors`; if highest rating, activate | `Subsystem::register_governor` |
| `cpuidle_switch_governor(gov)` | swap `cpuidle_curr_governor` + re-init per-device | `Subsystem::switch_governor` |
| `cpuidle_select(drv, dev, &stop_tick)` | invoke `curr_governor->select` to pick C-state index | `Governor::select` |
| `cpuidle_enter(drv, dev, idx)` | call `drv->states[idx].enter(dev, drv, idx)` with timing + tracepoints | `Device::enter` |
| `cpuidle_enter_state(dev, drv, idx)` | inner enter path: clock_event_program, ct_cpuidle_enter, mwait/wfi, ct_cpuidle_exit, residency accounting | `Device::enter_state` |
| `cpuidle_reflect(dev, idx)` | post-enter governor feedback (`curr_governor->reflect(dev, idx)`) | `Governor::reflect` |
| `cpuidle_enter_s2idle(drv, dev, latency_limit_ns)` | suspend-to-idle entry — find deepest enter_s2idle + freeze tick | `Device::enter_s2idle` |
| `cpuidle_use_deepest_state(latency_limit_ns)` | force per-CPU governor override to deepest state within latency | `Device::use_deepest_state` |
| `cpuidle_play_dead()` | cpu offline — pick state with `enter_dead` callback (no return) | `Device::play_dead` |
| `cpuidle_pause()` / `cpuidle_resume()` | global pause (for thermal / debug) | `Subsystem::pause` / `_resume` |
| `cpuidle_pause_and_lock()` / `_resume_and_unlock()` | locked variant for register/unregister | `Subsystem::pause_and_lock` / `_resume_and_unlock` |
| `cpuidle_get_cpu_driver(dev)` | per-CPU current driver lookup | `Device::get_cpu_driver` |
| `cpuidle_governor_latency_req(cpu)` | aggregate pm_qos `PM_QOS_CPU_DMA_LATENCY` for governor select | `Governor::latency_req` |
| `cpuidle_add_interface()` / `_remove_interface()` | global sysfs root creation | `Sysfs::add_interface` / `_remove_interface` |
| `cpuidle_add_device_sysfs(dev)` / `_remove_device_sysfs(dev)` | per-CPU + per-state sysfs (state0/state1/...) | `Sysfs::add_device_sysfs` / `_remove_device_sysfs` |

## Compatibility contract

REQ-1: Exactly one `cpuidle_driver` may be active globally (or per-CPU when `CPUIDLE_DRIVER_FLAGS_PER_CPU` is set + cpumask non-overlap); per-CPU registration paths managed via `__cpuidle_driver_init`.

REQ-2: Per-CPU lifecycle: `cpuidle_register_device` validates dev->cpu, links to driver, allocates per-state kobjects, sets `enabled = 1`. Reverse on unregister.

REQ-3: Per-driver state table: each `cpuidle_state` entry must declare `name`, `desc`, `enter` callback, and either `target_residency_ns` + `exit_latency_ns` (the residency-based selection inputs). Index 0 is reserved for POLL (synthetic, from `poll_state.c`) when CONFIG_CPU_IDLE_GOV_LADDER is not the active governor.

REQ-4: Governor selection: at registration, the governor with highest `.rating` wins and becomes `cpuidle_curr_governor`. Defaults: `teo` (rating 19) or `menu` (rating 20) typically; `ladder` (10) overrides to 25 when `!tick_nohz_enabled` (NO_HZ off); `haltpoll` (9) registered only when `kvm_para_available()`.

REQ-5: Governor swap via `/sys/devices/system/cpu/cpuidle/current_governor` write: serialize via `cpuidle_lock`, pause all CPUs, call `gov.disable` on every device, switch, call `gov.enable`, resume.

REQ-6: pm_qos integration: `cpuidle_governor_latency_req(cpu)` returns `min(PM_QOS_RESUME_LATENCY[cpu], PM_QOS_CPU_DMA_LATENCY)`; governor uses this as max acceptable exit_latency.

REQ-7: s2idle: `cpuidle_enter_s2idle(drv, dev, latency)` finds deepest state with `enter_s2idle` callback ≤ latency, freezes the tick (`tick_freeze`), invokes `state->enter_s2idle`, unfreezes on wake.

REQ-8: Per-state sysfs: `/sys/devices/system/cpu/cpuN/cpuidle/stateK/`: `name`, `desc`, `latency` (µs), `residency` (µs), `power`, `usage`, `time`, `disable`, `above`, `below`, `rejected`, `s2idle/usage`, `s2idle/time`.

REQ-9: Per-state `disable` write CAP_SYS_ADMIN: stores into `dev->states_usage[idx].disable` (bitmask); disabled states skipped by all governors.

REQ-10: `enter_dead`: per-state callback invoked from `cpuidle_play_dead` during CPU offline; must not return (sets CPU to deepest sleep + waits for re-online via wake).

REQ-11: Coupled C-states (ARM): when `CONFIG_ARCH_NEEDS_CPU_IDLE_COUPLED=y`, per-driver `coupled_cpus` mask requires all coupled CPUs to rendezvous before entering coupled state; `drivers/cpuidle/coupled.c` implements rendezvous protocol.

REQ-12: Tracepoints: `cpu_idle` + `cpu_idle_miss` emitted around state enter; governor predictions vs actual residency observable via tracing.

## Acceptance Criteria

- [ ] AC-1: `cat /sys/devices/system/cpu/cpu0/cpuidle/state*/name` lists per-arch C-states (POLL, C1, C1E, C6, C8, C10 on Intel; WFI, RET, OFF on ARM).
- [ ] AC-2: `cat /sys/devices/system/cpu/cpuidle/current_governor` returns active governor; write switches successfully.
- [ ] AC-3: Idle workload (no user processes): per-state `usage` + `time` counters increment for the deepest states.
- [ ] AC-4: pm_qos test: `dd if=/dev/zero of=/dev/null oflag=direct` with `PM_QOS_CPU_DMA_LATENCY=10us` → only state0/state1 selected by governor.
- [ ] AC-5: s2idle test: `echo freeze > /sys/power/state` enters deepest enter_s2idle state; wakeup restores tick.
- [ ] AC-6: CPU hotplug: offline CPU triggers `enter_dead` for deepest state; online reverses.
- [ ] AC-7: Disable test: `echo 1 > .../stateK/disable` excludes state from governor selection.
- [ ] AC-8: Governor swap cycle: menu → teo → ladder → menu under load; no oops + per-state counters continue accumulating.
- [ ] AC-9: kselftest `tools/testing/selftests/cpufreq/cpuidle/` cases pass.

## Architecture

`Device` lives in `drivers::cpuidle::Device`:

```
struct Device {
  registered: AtomicU32,         // 0 = not registered, 1 = registered
  enabled: AtomicU32,            // can enter states (0 during pause)
  cpu: u32,
  poll_time_limit: bool,         // poll_state exited because limit
  last_residency_ns: u64,        // last state's actual residency
  poll_limit_ns: u64,            // for haltpoll-driven dynamic poll
  forced_idle_latency_limit_ns: u64,
  next_hrtimer: u64,
  last_state_idx: i32,
  states_usage: [StateUsage; CPUIDLE_STATE_MAX] {
    disable: u64,
    usage: u64,
    time_ns: u64,
    above: u64,                  // governor too-shallow
    below: u64,                  // governor too-deep
    rejected: u64,               // state-specific rejection
    s2idle_usage: u64,
    s2idle_time: u64,
  },
  kobjs: [*mut StateKobj; CPUIDLE_STATE_MAX],
  kobj_driver: *mut DriverKobj,
  kobj_dev: *mut DeviceKobj,
  device_list: ListHead,
  #[cfg(CONFIG_ARCH_NEEDS_CPU_IDLE_COUPLED)]
  coupled_cpus: Cpumask,
  coupled: *mut CoupledData,
  safe_state_idx: i32,
}

struct Driver {
  name: [u8; CPUIDLE_NAME_LEN],
  owner: *mut Module,
  refcnt: AtomicU32,
  states: [State; CPUIDLE_STATE_MAX] {
    name: [u8; CPUIDLE_NAME_LEN],
    desc: [u8; CPUIDLE_DESC_LEN],
    exit_latency_ns: u64,
    target_residency_ns: u64,
    flags: u32,                  // POLLING / TIMER_STOP / RCU_IDLE / UNUSABLE / etc.
    power_usage: u32,
    enter: fn(&Device, &Driver, i32) -> i32,
    enter_dead: Option<fn(&Device, i32)>,
    enter_s2idle: Option<fn(&Device, &Driver, i32) -> i32>,
  },
  state_count: i32,
  safe_state_index: i32,
  cpumask: *mut Cpumask,
  governor: *const u8,           // optional governor preference per driver
}
```

Subsystem init `Subsystem::init`:
1. `cpuidle_init()` (core_initcall) — create `/sys/devices/system/cpu/cpuidle/` global sysfs.
2. Each governor (`menu`, `teo`, `ladder`, `haltpoll`) registers via `postcore_initcall`; `cpuidle_register_governor` picks highest-rating as `cpuidle_curr_governor`.
3. Per-arch driver registers via subsys_initcall (intel_idle: `subsys_initcall_sync`).
4. `cpuidle_register_driver` validates state-count, populates safe_state_idx, refcnts driver.
5. Per-CPU `cpuidle_register_device` allocates state kobjs + sysfs group.

Per-CPU enter path `cpuidle_enter_state(dev, drv, idx)`:
1. `clockevents_notify(CLOCK_EVT_NOTIFY_BROADCAST_ENTER)` if state stops local timer.
2. `tick_broadcast_enter()` for broadcast-tick wakeup if `CPUIDLE_FLAG_TIMER_STOP`.
3. `time_start = local_clock()`.
4. `ct_cpuidle_enter()` — context-tracking transition to idle.
5. `state->enter(dev, drv, idx)` — per-driver issue MWAIT / WFI / PSCI.
6. `ct_cpuidle_exit()`; `time_end = local_clock()`.
7. `dev->last_residency_ns = time_end - time_start`; `states_usage[idx].time_ns += last_residency_ns; states_usage[idx].usage++`.
8. `cpuidle_reflect(dev, idx)` — governor feedback.

Governor select path:
1. From `cpu_idle_loop()` (kernel/sched/idle.c): `cpuidle_select(drv, dev, &stop_tick)`.
2. `curr_governor->select(drv, dev, &stop_tick)` returns target idx.
3. Governors:
   - `menu`: 6-bucket per-magnitude correction factor + 8-interval repeatable-pattern detector.
   - `teo`: per-bin "hits" vs "intercepts" metric; predicts next-wakeup-by-timer-or-interrupt.
   - `ladder`: residency-history climber (promotion after 4 hits above promo_time, demotion after 1 miss).
   - `haltpoll`: KVM-guest dynamic poll window (grow on long sleeps, shrink on short).

Suspend-to-idle `cpuidle_enter_s2idle`:
1. `find_deepest_state(drv, dev, latency_limit_ns, 0, true)` — pick deepest with `enter_s2idle`.
2. `tick_freeze()` — freeze the tick subsystem.
3. `state->enter_s2idle(dev, drv, idx)` — driver issues MWAIT with deep-sleep hint.
4. WARN if IRQs enabled on return.
5. `tick_unfreeze()`.

Governor swap `cpuidle_switch_governor`:
1. `cpuidle_pause_and_lock` — set every dev->enabled = 0.
2. For each dev: `old_gov.disable(drv, dev)` (if hook present).
3. `cpuidle_curr_governor = new_gov`.
4. For each dev: `new_gov.enable(drv, dev)`.
5. `cpuidle_resume_and_unlock`.

## Hardening

(Inherits row-1 features from `drivers/00-overview.md` § Hardening.)

cpuidle-core specific reinforcement:

- **`cpuidle_lock` mutex serializes register / unregister / pause / governor-swap** — defense against concurrent state mutation.
- **`drv->refcnt` atomic with paired get/put** — defense against driver unregister-while-in-use.
- **per-CPU dev->enabled atomic flag** — defense against re-entry during pause.
- **state-table validation at register** — `state_count <= CPUIDLE_STATE_MAX`, every state has `enter` callback, `safe_state_idx` valid.
- **`enter_dead` only called from `cpuidle_play_dead`** — defense against accidental no-return enter from normal idle path.
- **s2idle requires non-coupled state** — defense against coupled-CPU rendezvous in single-CPU suspend.
- **per-state `disable` mask sticky** — defense against governor selecting a state that user/admin disabled.
- **Per-state kobject lifecycle paired** — defense against sysfs UAF on hotplug-offline.
- **`tick_freeze`/`tick_unfreeze` paired with WARN on return-with-IRQs-on** — defense against state-driver leaving IRQs enabled.
- **`clockevents_notify` BROADCAST_ENTER/EXIT paired** — defense against broadcast-tick leak when state stops local APIC.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `cpuidle_device`, `cpuidle_driver` copies, and per-CPU state-usage counters; sysfs read buffers strictly bounded by PAGE_SIZE.
- **PAX_KERNEXEC** — cpuidle core in W^X kernel text; `cpuidle_driver` state[].enter / enter_dead / enter_s2idle function pointers and `cpuidle_governor` vtable live in `__ro_after_init` after registration.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `cpuidle_enter_state`, governor `.select`, governor `.reflect`, and `cpuidle_enter_s2idle` paths.
- **PAX_REFCOUNT** — saturating `refcount_t`-style atomic on `cpuidle_driver.refcnt` and per-device registered counts; overflow trap defeats register/unregister race UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `cpuidle_device`, per-state usage counters, governor private (`menu_device`, `teo_cpu`, `ladder_device`), and coupled-CPU rendezvous structs; prevents leaked residency histograms and prediction state across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every sysfs store entry (`current_governor`, per-state `disable`, `current_driver`); reject user-pointer deref outside `kstrtouint_from_user` / `sysfs_streq` helpers.
- **PAX_RAP / kCFI** — `cpuidle_driver` state[].enter / enter_dead / enter_s2idle and `cpuidle_governor` (`init`, `exit`, `enable`, `disable`, `select`, `reflect`) marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms for cpuidle helpers and per-CPU device pointers behind CAP_SYSLOG; suppress `%p` in `cpu_idle` tracepoints.
- **GRKERNSEC_DMESG** — restrict cpuidle init banners, governor-rating tables, and per-CPU state-enable diagnostics to CAP_SYSLOG so attackers cannot fingerprint silicon C-state inventory via dmesg.
- **C-state table RO/kCFI** — `cpuidle_driver.states[]` published as `__ro_after_init`; refuse runtime mutation; per-state `enter` callbacks dispatched through kCFI-typed pointer to defeat function-pointer overwrite.
- **Governor swap CAP_SYS_ADMIN audit** — `current_governor` sysfs writes require CAP_SYS_ADMIN + are logged via audit subsystem with caller + new-governor name; defense against silent prediction-policy hijack.
- **Per-state `disable` mask CAP_SYS_ADMIN** — defense against unprivileged disable of shallow states to force deep-sleep wakeup-latency side channels.
- **Residency stats sanitize** — per-state `time_ns` / `usage` counters readable only by owner / CAP_SYS_ADMIN; defense against per-CPU workload fingerprinting via residency histograms.
- **s2idle latency-limit bound** — `cpuidle_enter_s2idle` rejects latency_limit_ns of zero or wider than the deepest state's exit_latency; defense against caller forcing the wrong state via crafted PM_QOS request.

Rationale: cpuidle is the interface between the scheduler's idle loop and the silicon's deepest power states — a tampered governor `.select` or `.reflect`, a sysfs-driven `disable` race, or an unbounded s2idle latency lets userspace either fingerprint silicon (per-state residency disclosure) or push the package into a state that violates the PM_QOS contract held by latency-sensitive userspace (audio, real-time). RAP/kCFI on driver state-tables and governor vtables, CAP_SYS_ADMIN on governor swap and per-state disable, audit on prediction-policy mutation, refcount overflow trapping on driver refs, and residency-stats access control turn cpuidle from "best-effort sleep" into a structural enforcement boundary that respects both the silicon's wake-latency spec and the pm_qos latency contract.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- intel_idle driver internals (covered in `intel-idle.md`)
- Per-governor algorithm details (covered in `governors.md`)
- ARM PSCI-based cpuidle drivers (covered in PSCI Tier-3 future)
- POWER cpuidle (covered in powernv Tier-3 future)
- RISC-V SBI HSM idle (covered in RISC-V Tier-3 future)
- Coupled-CPU ARM rendezvous algorithm (covered in `coupled.md` future Tier-3)
- 32-bit-only paths
- Implementation code
