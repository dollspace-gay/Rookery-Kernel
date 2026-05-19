# Tier-3: kernel/sched/cpufreq_schedutil.c — schedutil governor (scheduler-integrated, util-driven, fast-path)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/cpufreq/00-overview.md
upstream-paths:
  - kernel/sched/cpufreq_schedutil.c
  - kernel/sched/cpufreq.c
  - kernel/sched/sched.h
  - include/linux/sched/cpufreq.h
  - Documentation/admin-guide/pm/cpufreq.rst
-->

## Summary

`schedutil` is the cpufreq governor that consumes the scheduler's PELT (Per-Entity Load Tracking) utilization signals to drive frequency selection — instead of polling the CPU separately like `ondemand` or `conservative`, schedutil's `.select` callback runs directly from `cpufreq_update_util` invocations that the scheduler emits on every task enqueue / dequeue / migration / tick. This produces sub-millisecond response (the scheduler's own clock is the trigger) and is the mandatory governor for Energy Aware Scheduling (EAS): EAS needs an authoritative util-to-freq mapping that schedutil provides via `map_util_freq`. schedutil lives in `kernel/sched/cpufreq_schedutil.c` (~940 lines) rather than `drivers/cpufreq/` because it is intimately coupled to scheduler internals (sched.h, PELT, dl_se, rt_avg).

This Tier-3 covers `cpufreq_schedutil.c` core (sugov_policy / sugov_cpu, get_next_freq, sugov_update_single_*, sugov_update_shared, kthread / irq_work plumbing for slow-path drivers) plus the scheduler-side `kernel/sched/cpufreq.c` glue (`cpufreq_add_update_util_hook`, `cpufreq_this_cpu_can_update`).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct sugov_tunables` | per-policy tunables (`rate_limit_us`) | `kernel::sched::cpufreq_schedutil::Tunables` |
| `struct sugov_policy` | per-cpufreq-policy schedutil state (next_freq, last_freq_update_time, irq_work, kthread, work_lock) | `kernel::sched::cpufreq_schedutil::Policy` |
| `struct sugov_cpu` | per-CPU schedutil state (iowait_boost, util, bw_min, update_util) | `kernel::sched::cpufreq_schedutil::CpuState` |
| `schedutil_gov` (struct cpufreq_governor) | governor vtable (`init`, `exit`, `start`, `stop`, `limits`; flags = DYNAMIC_SWITCHING) | `Governor::schedutil` |
| `sugov_init(policy)` / `sugov_exit(policy)` | per-policy alloc/free of sugov_policy + tunables | `Policy::init` / `_exit` |
| `sugov_start(policy)` / `sugov_stop(policy)` | install/uninstall per-CPU `cpufreq_add_update_util_hook` callbacks | `Policy::start` / `_stop` |
| `sugov_limits(policy)` | on min/max change, set `limits_changed` + smp_wmb so next util-update re-evals | `Policy::limits` |
| `sugov_update_single_freq(hook, time, flags)` | per-CPU util-update for slow-path drivers (single-CPU policy, target_index) | `Policy::update_single_freq` |
| `sugov_update_single_perf(hook, time, flags)` | per-CPU util-update for fast-path drivers supporting `adjust_perf` (HWP passive) | `Policy::update_single_perf` |
| `sugov_update_shared(hook, time, flags)` | shared-policy util-update (multiple CPUs in one policy) | `Policy::update_shared` |
| `get_next_freq(sg_policy, util, max)` | compute target freq from util via `map_util_freq` + cached_raw_freq | `Policy::get_next_freq` |
| `sugov_get_util(sg_cpu, boost)` | read PELT util (`cpu_util_cfs`, `cpu_util_rt`, `cpu_util_dl`, `cpu_bw_dl`, irq scaling) | `CpuState::get_util` |
| `sugov_iowait_apply(sg_cpu, time, max_cap)` | apply iowait boost (decay halve every `rate_limit_us`) | `CpuState::iowait_apply` |
| `sugov_should_update_freq(sg_policy, time)` | rate-limit gate (>= rate_limit_us since last update) + limits_changed override | `Policy::should_update_freq` |
| `sugov_update_next_freq(sg_policy, time, next_freq)` | commit next_freq if changed | `Policy::update_next_freq` |
| `sugov_fast_switch(sg_policy, time, next_freq)` | scheduler-context lockless freq write via `cpufreq_driver_fast_switch` | `Policy::fast_switch` |
| `sugov_deferred_update(sg_policy)` | schedule `irq_work` → kthread for slow-path driver | `Policy::deferred_update` |
| `sugov_work(work)` | kthread worker: takes `work_lock`, calls `__cpufreq_driver_target` | `Policy::work` |
| `sugov_irq_work(irq_work)` | irq_work callback → wakes kthread | `Policy::irq_work_handler` |
| `sugov_is_governor(policy)` | EAS gate: returns true iff `policy->governor == &schedutil_gov` | `Governor::is_schedutil` |
| `cpufreq_add_update_util_hook(cpu, data, func)` / `_remove` | install per-CPU scheduler util-update callback | `kernel::sched::cpufreq::add_update_util_hook` / `_remove` |
| `map_util_freq(util, freq_ref, max)` | compute `freq_ref * util / max * 1.25` (25% margin) | `kernel::sched::cpufreq::map_util_freq` |

## Compatibility contract

REQ-1: schedutil is registered as a `cpufreq_governor` with flag `CPUFREQ_GOV_DYNAMIC_SWITCHING` (dynamic, called from scheduler util-update path, not via periodic timer).

REQ-2: Per-policy lifecycle: `init` allocates `sugov_policy` + `sugov_tunables` (rate_limit_us defaulted from `policy->transition_delay_us`); `start` chooses per-CPU update-callback variant (`single_freq` / `single_perf` / `shared`) + installs hooks via `cpufreq_add_update_util_hook`; `stop` removes hooks + `synchronize_rcu` + cancels kthread; `exit` frees.

REQ-3: Slow-path (non-fast-switch drivers): per-policy realtime FIFO kthread (`sched_set_fifo_low`) services freq changes via `kthread_work`. `irq_work` wakes the kthread from scheduler context.

REQ-4: Fast-path (`policy->fast_switch_enabled == true`): freq write happens inline in scheduler `cpufreq_update_util` context without kthread; driver's `fast_switch` must be sleep-free + IRQ-safe.

REQ-5: PELT util composition: `util = max(cfs_util + cfs_bw_min, rt_util) + dl_util + irq_scale_factor`. RT class util capped at `max - dl - irq` (RT does not run at below-min frequency).

REQ-6: iowait boost: per-CPU `iowait_boost` starts at `IOWAIT_BOOST_MIN` (SCHED_CAPACITY_SCALE/8), doubles on each iowait wakeup up to `max_cap`; halves on each non-iowait update interval.

REQ-7: Frequency-invariance: `get_capacity_ref_freq` returns `arch_scale_freq_ref(cpu)` if available, else `cpuinfo.max_freq` if invariant, else `policy->cur + 25%` (non-invariant fallback).

REQ-8: Rate limit: `rate_limit_us` tunable (default = `policy->transition_delay_us`) gates how often a new freq write may issue; `sugov_should_update_freq` enforces. Bypassed when `limits_changed` flag set (pm_qos change).

REQ-9: EAS coupling: `cpufreq_ready_for_eas` (in `drivers/cpufreq/cpufreq.c`) iterates `cpu_mask` and requires every CPU's policy to use schedutil; this is the EAS feasibility precondition.

REQ-10: PREEMPT_RT: fast-path is safe under PREEMPT_RT (drivers with `fast_switch` must be designed to run with rq->lock held); slow-path kthread is `sched_set_fifo_low` so it preempts SCHED_OTHER but not RT tasks.

REQ-11: SMP single-CPU policy: when `policy_is_shared(policy) == false`, `update_single_freq` (or `_single_perf`) skips the per-CPU loop in `sugov_get_util` (only one CPU contributes).

REQ-12: ignore_dl_rate_limit: when a DL task arrives with bandwidth pressure, `need_freq_update` is set to bypass rate_limit and force an immediate freq raise (handled in `sugov_should_update_freq`'s second branch).

## Acceptance Criteria

- [ ] AC-1: `echo schedutil > /sys/.../scaling_governor` succeeds on every CPU; `cat scaling_governor` returns `schedutil`.
- [ ] AC-2: `cat /sys/.../schedutil/rate_limit_us` returns transition_delay_us-derived default; write-back changes are observed.
- [ ] AC-3: Stress workload (single CPU pinned 100% busy-loop) drives `scaling_cur_freq` to max; release → ramp-down to min within ~rate_limit_us * 8.
- [ ] AC-4: EAS test: with CONFIG_ENERGY_MODEL + asymmetric CPU capacities + schedutil on all policies, `sched_energy_present` static key is on; otherwise off.
- [ ] AC-5: iowait test: dd bs=1M oflag=sync → observed freq above pure-CPU-util prediction due to iowait_boost.
- [ ] AC-6: DL/RT test: a SCHED_DEADLINE task with `cpu_bw_dl` triggers freq raise above CFS-only target.
- [ ] AC-7: pm_qos test: thermal-driven max_freq drop triggers `sugov_limits` → next util-update reflects clamped freq.
- [ ] AC-8: PREEMPT_RT test: schedutil active under PREEMPT_RT does not lockdep-splat on rq->lock + work_lock interactions.
- [ ] AC-9: kselftest `tools/testing/selftests/sched/` schedutil cases pass.

## Architecture

`Policy` (sugov_policy) lives in `kernel::sched::cpufreq_schedutil::Policy`:

```
struct Policy {
  policy: *mut CpufreqPolicy,
  tunables: *mut Tunables {
    attr_set: GovAttrSet,
    rate_limit_us: u32,
  },
  tunables_hook: ListHead,
  update_lock: RawSpinlock,
  last_freq_update_time: u64,    // monotonic ns
  freq_update_delay_ns: i64,     // rate_limit_us * 1000
  next_freq: u32,                // last computed freq
  cached_raw_freq: u32,          // for freq table inverse lookup
  // Slow-path fields (only when !fast_switch_enabled):
  irq_work: IrqWork,
  work: KthreadWork,
  work_lock: Mutex,
  worker: KthreadWorker,
  thread: *mut TaskStruct,
  work_in_progress: bool,
  limits_changed: bool,
  need_freq_update: bool,
}

struct CpuState {                // sugov_cpu, per-CPU
  update_util: UpdateUtilData,   // installed via cpufreq_add_update_util_hook
  sg_policy: *mut Policy,
  cpu: u32,
  iowait_boost_pending: bool,
  iowait_boost: u32,
  last_update: u64,
  util: u64,
  bw_min: u64,
  #[cfg(CONFIG_NO_HZ_COMMON)]
  saved_idle_calls: u64,         // single-CPU iowait gating
}
```

Lifecycle `sugov_init`:
1. Validate policy not yet attached to schedutil.
2. `kobject_init`-style `kobject_add` against governor sysfs parent (cross-ref cpufreq core).
3. Allocate `sugov_policy` (kzalloc) + `sugov_tunables` (one per policy if `policy->governor_data` already set, else first-policy creates).
4. `sg_policy->tunables = tunables; tunables->rate_limit_us = policy->transition_delay_us / 1000`.
5. Set `policy->governor_data = sg_policy`.

Lifecycle `sugov_start`:
1. Initialize fields (`freq_update_delay_ns = rate_limit_us * NSEC_PER_USEC`).
2. Pick update callback based on `policy_is_shared` + `fast_switch_enabled` + `cpufreq_driver_has_adjust_perf`.
3. For each CPU in `policy->cpus`: `memset(sg_cpu); sg_cpu->cpu = cpu; cpufreq_add_update_util_hook(cpu, &sg_cpu->update_util, uu)`.
4. For slow-path: create FIFO_LOW kthread bound to `policy->cpus` first CPU.

Per-CPU util-update `sugov_update_single_freq` (representative):
1. Called from `cpufreq_update_util` (scheduler hook) with rq->lock held + IRQs off.
2. `sugov_get_util(sg_cpu, boost)` — read PELT util (`cpu_util_cfs`, `cpu_util_rt`, `cpu_util_dl`, etc.).
3. `sugov_iowait_apply(sg_cpu, time, max_cap)` — bump util by iowait_boost.
4. `sugov_should_update_freq(sg_policy, time)` — rate-limit check; return if too soon.
5. `next_freq = get_next_freq(sg_policy, util, max)`.
6. `sugov_update_next_freq(sg_policy, time, next_freq)` — commit if changed.
7. If `fast_switch_enabled`: `sugov_fast_switch(sg_policy, time, next_freq)` writes inline; else `sugov_deferred_update(sg_policy)` queues irq_work.

Frequency computation `get_next_freq`:
1. `freq_ref = get_capacity_ref_freq(policy)` (arch_scale_freq_ref or max_freq).
2. `freq = map_util_freq(util, freq_ref, max)` = `(freq_ref * util / max) * 5/4` (25% headroom).
3. If `freq == sg_policy->cached_raw_freq && next_freq != 0`: return cached next_freq.
4. Else: `cpufreq_driver_resolve_freq(policy, freq)` rounds to nearest table entry via relation-L/H/C; cache.

Shared-policy update `sugov_update_shared`:
1. Lock `sg_policy->update_lock`.
2. Loop `for_each_cpu(j, policy->cpus)`: `sugov_get_util(sg_j); util_max = max(util_max, util_j); bw_min_max = max(bw_min_max, bw_min_j)`.
3. Run rate-limit + get_next_freq + commit + slow-or-fast-path with `update_lock` held.
4. Unlock.

Slow-path kthread `sugov_work`:
1. Lock `work_lock`.
2. `__cpufreq_driver_target(policy, next_freq, CPUFREQ_RELATION_L)`.
3. `work_in_progress = false`.
4. Unlock; `irq_work_sync` semantics ensure no concurrent re-queue.

Limits change `sugov_limits`:
1. If not fast_switch: lock `work_lock`, `cpufreq_policy_apply_limits(policy)`, unlock.
2. `smp_wmb()`; `WRITE_ONCE(limits_changed, true)`.
3. Pairs with `smp_mb()` + `WRITE_ONCE(limits_changed, false)` in `sugov_should_update_freq`.

## Hardening

(Inherits row-1 features from `drivers/cpufreq/00-overview.md` § Hardening.)

schedutil specific reinforcement:

- **Per-policy `update_lock` raw_spinlock** — held across shared-policy util-update; IRQ-safe + non-sleeping.
- **smp_wmb/smp_mb pairing on `limits_changed`** — defense against missed limit refresh after pm_qos update.
- **`cpufreq_this_cpu_can_update(policy)` gate** — defense against cross-CPU update where target driver cannot service.
- **rate_limit_us bounded > 0** — defense against `rate_limit_us = 0` causing every util-update to issue a freq write + DoS.
- **iowait_boost halved per interval** — defense against pinned iowait task holding turbo permanently.
- **PELT util-RT capped at `max - dl - irq`** — defense against RT spinner forcing freq below DL bandwidth request.
- **kthread sched_set_fifo_low** — defense against schedutil kthread starving SCHED_RR / SCHED_FIFO user tasks.
- **`cpufreq_add_update_util_hook` per-CPU once** — install/remove paired with synchronize_rcu in `sugov_stop`.
- **fast_switch path NMI-safe** — driver fast_switch may run from irq_work or rq->lock context.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `sugov_policy`, `sugov_tunables`; per-CPU `sugov_cpu` is per-cpu memory not exposed via copy_*_user paths.
- **PAX_KERNEXEC** — schedutil core in W^X kernel text; `schedutil_gov` cpufreq_governor vtable and per-CPU update-util function pointers (`sugov_update_single_freq` / `_single_perf` / `_shared`) live in `__ro_after_init`.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `sugov_update_single_freq`, `sugov_update_shared`, `get_next_freq`, and the slow-path `sugov_work` kthread entry.
- **PAX_REFCOUNT** — saturating `refcount_t` on `sugov_tunables` shared across policies in non-per-policy-tunable mode; overflow trap defeats governor-swap + tunable-rcu race UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `sugov_policy`, `sugov_tunables`, per-CPU `sugov_cpu` regions on hotplug-offline + governor-exit; prevents leaked PELT residuals and iowait_boost history across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on sysfs store entries (`rate_limit_us`); reject user-pointer deref outside `kstrtouint_from_user`.
- **PAX_RAP / kCFI** — `schedutil_gov` (`init`, `exit`, `start`, `stop`, `limits`), `sugov_update_*` callbacks (registered via `cpufreq_add_update_util_hook`), irq_work and kthread_work function pointers marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms for schedutil helpers and per-CPU sugov_cpu pointers behind CAP_SYSLOG; suppress `%p` in PELT/util-update tracepoints.
- **GRKERNSEC_DMESG** — restrict schedutil init banners and EAS-readiness diagnostics to CAP_SYSLOG so attackers cannot derive scheduler invariance state via dmesg.
- **rcu_dereference of policy under rq->lock** — `sg_policy->policy` accesses validated through `rcu_dereference_sched()` within scheduler-context util-update; defense against UAF if governor is being torn down concurrently.
- **RT-class util capping** — RT-task contribution capped at `max - dl - irq` so an unprivileged RT spinner cannot force the package into a frequency that violates DL bandwidth or thermal envelope.
- **Fast-path PREEMPT_RT safety** — the fast-path inline freq write is marked `notrace` + executes with rq->lock held + IRQs disabled; mis-use from sleeping context flagged by lockdep + might_sleep assertions in debug builds.
- **rate_limit_us minimum floor** — sysfs store clamps to `transition_delay_us / 1000` floor; defense against `rate_limit_us = 0` driving a MSR write per scheduler tick.
- **Tunables ref audit** — shared-tunables refcount transitions logged via audit when crossing the policy-add / policy-remove boundary; defense against silent tunable hijack across governor reattach.

Rationale: schedutil sits inside the scheduler's hottest hot path — every task enqueue / dequeue / tick may invoke `cpufreq_update_util`, which under fast-switch drivers writes MSRs inline with rq->lock held. A relaxed RCU policy on `sg_policy->policy`, a tunable store that bypasses the rate_limit floor, an RT-class util signal that escapes the cap, or a fast-path write that takes a sleeping lock collapses into either DoS (MSR-write storm), thermal violation (RT-driven turbo pin), or UAF (policy torn down mid-callback). RAP/kCFI on the governor vtable and per-CPU update-util pointers, RCU-protected policy deref under rq->lock, RT-util cap, PREEMPT_RT fast-path discipline, and audit on shared-tunable refcount transitions turn schedutil from "fast and trusted" into a structurally bounded scheduler component that meets the rq->lock + PELT invariants the rest of the scheduler relies on.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Detailed PELT internals (covered in scheduler Tier-3 docs)
- EAS energy-model integration (covered in `kernel/sched/cpufreq.md` + EAS Tier-3)
- ondemand / conservative sampling governors (covered in `ondemand.md` future Tier-3)
- Per-driver `fast_switch` impl details (covered per-driver, e.g., `intel-pstate.md`)
- Schedutil-EAS for asymmetric SMP topologies (covered in topology Tier-3)
- 32-bit-only paths
- Implementation code
