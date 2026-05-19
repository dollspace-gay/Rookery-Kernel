# Tier-3: drivers/cpuidle/governors/{menu,teo,ladder,haltpoll}.c — cpuidle governors (per-governor heuristics, predicted-wake)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/cpuidle/00-overview.md
upstream-paths:
  - drivers/cpuidle/governors/menu.c
  - drivers/cpuidle/governors/teo.c
  - drivers/cpuidle/governors/ladder.c
  - drivers/cpuidle/governors/haltpoll.c
  - drivers/cpuidle/governors/gov.h
  - drivers/cpuidle/governor.c
-->

## Summary

The cpuidle governors implement four distinct algorithms for picking which C-state to enter on each idle entry. The governor framework (`drivers/cpuidle/governor.c`) maintains a global ordered list `cpuidle_governors`; the governor with the highest `.rating` becomes the active `cpuidle_curr_governor` at boot, and userspace can re-select via `/sys/devices/system/cpu/cpuidle/current_governor`. Per-governor heuristics use different inputs: `menu` and `teo` consume next-timer-event-time plus historical wake patterns to predict idle duration; `ladder` uses only the previous state's actual residency to climb/descend a state ladder; `haltpoll` is a KVM-guest-specific governor that adds a busy-poll window before issuing the actual halt instruction. Each governor exposes `.init`, `.exit`, `.enable`, `.disable`, `.select`, `.reflect` callbacks and stores per-CPU prediction state.

This Tier-3 covers `menu.c` (~370 lines: 6-bucket per-magnitude correction-factor + 8-interval repeatable-pattern), `teo.c` (~600 lines: per-bin hits-vs-intercepts metric, recent-intercepts gate), `ladder.c` (~190 lines: promotion-count / demotion-count residency-history climber), and `haltpoll.c` (~150 lines: KVM-guest dynamic-poll-window governor).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `cpuidle_governors` (LIST_HEAD) + `cpuidle_curr_governor` | global governor registry + active pointer | `drivers::cpuidle::governor::Registry` |
| `cpuidle_register_governor(gov)` / `cpuidle_switch_governor(gov)` | append + activate / swap active | `Registry::register` / `_switch` |
| `cpuidle_governor_latency_req(cpu)` | pm_qos aggregate read for governors | `Registry::latency_req` |
| `menu_governor` (struct cpuidle_governor, rating 20) | menu governor instance | `governors::menu::Governor` |
| `struct menu_device` | per-CPU: `correction_factor[BUCKETS]`, `intervals[INTERVALS]`, `interval_ptr`, `bucket`, `next_timer_ns` | `governors::menu::Device` |
| `menu_select(drv, dev, &stop_tick)` | menu select: bucket lookup + correction + repeatable-interval | `governors::menu::select` |
| `menu_reflect(dev, idx)` | request governor data update | `governors::menu::reflect` |
| `menu_update(drv, dev)` | post-idle: update correction factor + intervals from `last_residency_ns` | `governors::menu::update` |
| `which_bucket(duration_ns)` | classify duration into one of 6 magnitude bins | `governors::menu::which_bucket` |
| `teo_governor` (struct cpuidle_governor, rating 19) | teo governor instance | `governors::teo::Governor` |
| `struct teo_bin` | per-state bin (`intercepts`, `hits`, `recent`) | `governors::teo::Bin` |
| `struct teo_cpu` | per-CPU: `state_bins[CPUIDLE_STATE_MAX]`, `total`, `next_recent_idx`, `recent_idx[NR_RECENT]`, `tick_wakeup`, `time_span_ns`, `sleep_length_ns` | `governors::teo::CpuData` |
| `teo_select(drv, dev, &stop_tick)` | teo select: hits/intercepts sum across bins + shallow-state preference if intercept-dominant | `governors::teo::select` |
| `teo_update(drv, dev)` | bin update from `last_residency_ns` + `next_timer_ns` | `governors::teo::update` |
| `teo_find_shallower_state` | helper: walk shallower states by intercepts | `governors::teo::find_shallower_state` |
| `ladder_governor` (struct cpuidle_governor, rating 10 or 25) | ladder governor instance | `governors::ladder::Governor` |
| `struct ladder_device_state` | per-state per-CPU: `threshold.promotion_count / _time_ns / demotion_count / _time_ns` + `stats.promotion_count / _demotion_count` | `governors::ladder::DeviceState` |
| `struct ladder_device` | per-CPU array of `ladder_device_state` | `governors::ladder::Device` |
| `ladder_select_state(drv, dev, dummy)` | ladder select: compare `last_residency` against thresholds, promote/demote/stay | `governors::ladder::select_state` |
| `ladder_do_selection` | reset stats on state change | `governors::ladder::do_selection` |
| `ladder_enable_device(drv, dev)` | populate thresholds from `state->exit_latency_ns` | `governors::ladder::enable_device` |
| `haltpoll_governor` (struct cpuidle_governor, rating 9) | haltpoll governor instance (KVM-guest only) | `governors::haltpoll::Governor` |
| `haltpoll_select(drv, dev, &stop_tick)` | KVM-guest select: poll-or-halt decision based on `poll_limit_ns` | `governors::haltpoll::select` |
| `haltpoll_reflect(dev, idx)` | adjust `poll_limit_ns` (grow/shrink) | `governors::haltpoll::reflect` |
| `adjust_poll_limit(dev, block_ns)` | grow/shrink dynamic-poll window per heuristic | `governors::haltpoll::adjust_poll_limit` |

## Compatibility contract

REQ-1: Governor list ordering: `cpuidle_register_governor` appends; the highest-rating governor with successful `init` becomes `cpuidle_curr_governor`. Override via cmdline `cpuidle.governor=` or sysfs `current_governor` write.

REQ-2: menu — 6 buckets (`BUCKETS=6`), 8 recent intervals (`INTERVALS=8`), DECAY=8, RESOLUTION=1024. `correction_factor[bucket] * RESOLUTION / DECAY` scales `next_timer_ns` to a predicted-duration. `intervals[8]` rolling buffer for repeatable-pattern detector.

REQ-3: menu select: predict duration = `min(next_timer_ns * correction_factor / RESOLUTION, MAX_INTERESTING)`; pick deepest state with `target_residency_ns ≤ predicted_duration` AND `exit_latency_ns ≤ pm_qos_latency`.

REQ-4: menu repeatable-pattern: if standard deviation of `intervals[8]` is low enough (per `compute_intervals_average_and_stddev`), use the average of intervals instead of next_timer correction (handles fixed-rate hardware interrupts like mice).

REQ-5: teo — per-state bin with `hits` (sleep_length ≈ residency) and `intercepts` (sleep_length ≫ residency, i.e., non-timer wake). Each bin sums into a running total via `LATENCY_THRESHOLD_NS` weighted classification.

REQ-6: teo select: start with deepest enabled state; sum hits + intercepts across shallower bins; if intercepts > half-sum, walk shallower picking the state with max intercepts in its sub-range. If candidate is shallow enough that target_residency < TICK_NSEC, prevent tick stop (`*stop_tick = false`).

REQ-7: teo NR_RECENT = 9 recent-wake ring buffer: if recent wake-pattern dominates current selection, override to that state.

REQ-8: ladder — `PROMOTION_COUNT=4`, `DEMOTION_COUNT=1`. After 4 consecutive idle entries where `last_residency_ns > exit_latency_ns(next_state)`, promote to next state. After 1 entry where `last_residency_ns < exit_latency_ns(current_state)`, demote.

REQ-9: ladder rating override: when `!tick_nohz_enabled`, ladder rating bumped 10 → 25 so it wins over menu (which depends on NO_HZ next-timer prediction).

REQ-10: haltpoll: only registered if `kvm_para_available()` (running as KVM guest). Tunables `guest_halt_poll_ns` (default 200000), `guest_halt_poll_shrink` (default 2), `guest_halt_poll_grow` (default 2), `guest_halt_poll_grow_start` (default 50000), `guest_halt_poll_allow_shrink` (default true).

REQ-11: haltpoll select: if last state was poll and didn't exceed limit, poll again. If last was halt, poll first. `*stop_tick = false` during poll window. Adjust `dev->poll_limit_ns` based on observed block duration.

REQ-12: Per-governor enable/disable lifecycle paired with active-governor swap: on `cpuidle_switch_governor`, old `.disable` called per-CPU, new `.enable` called per-CPU.

## Acceptance Criteria

- [ ] AC-1: `cat /sys/devices/system/cpu/cpuidle/available_governors` lists all registered governors (`menu`, `teo`, `ladder`, plus `haltpoll` under KVM).
- [ ] AC-2: `cat current_governor` returns highest-rating registered governor; `echo teo > current_governor` swaps successfully.
- [ ] AC-3: menu test: synthetic 500us-interval timer workload → menu repeatable-pattern detector picks state with `target_residency ≈ 500us`.
- [ ] AC-4: teo test: per-bin metrics observable via tracing/bpftrace; intercept-dominant phase → shallower state.
- [ ] AC-5: ladder test (boot `nohz=off`): ladder rating wins (25 > 20); promotion/demotion observable via tracing under varied load.
- [ ] AC-6: haltpoll test in KVM guest: `cat /sys/.../cpuidle/state0/usage` (POLL) grows during low-latency workload; `poll_limit_ns` adapts via per-CPU sysfs.
- [ ] AC-7: pm_qos test: `PM_QOS_CPU_DMA_LATENCY=10us` request → all governors restricted to states with `exit_latency_ns ≤ 10000`.
- [ ] AC-8: Governor swap stress: rapid swap menu ↔ teo ↔ ladder under workload — no oops, counters continue.
- [ ] AC-9: Cmdline test: `cpuidle.governor=teo` overrides default.

## Architecture

`menu::Device` per-CPU:

```
struct Device {                 // menu_device
  needs_update: i32,
  tick_wakeup: i32,
  next_timer_ns: u64,
  bucket: u32,                  // 0..5 magnitude bucket
  correction_factor: [u32; 6],  // per-bucket [RESOLUTION * DECAY = 8192] init
  intervals: [u32; 8],          // recent residency us
  interval_ptr: i32,            // ring index
}
```

`menu_select` (paraphrased):
1. `next_timer_ns = tick_nohz_get_sleep_length(&delta_next_us)`.
2. `bucket = which_bucket(next_timer_ns)`.
3. `predicted_ns = next_timer_ns * correction_factor[bucket] / (RESOLUTION * DECAY)`.
4. If `intervals[]` low-variance: `predicted_ns = min(predicted_ns, avg_intervals)`.
5. Walk `drv->states[]`: pick deepest with `target_residency_ns ≤ predicted_ns AND exit_latency_ns ≤ latency_req`.
6. If predicted-stop-tick threshold reached: `*stop_tick = true`.

`menu_update`:
1. On reflect, read `dev->last_residency_ns`.
2. Compute `new_factor = ((correction_factor[bucket] * (DECAY - 1)) + RESOLUTION * last_residency_ns / next_timer_ns) / DECAY`.
3. Store back into `correction_factor[bucket]`.
4. Push `measured_us` into `intervals[interval_ptr]`; advance ptr.

`teo::CpuData` per-CPU:

```
struct CpuData {
  state_bins: [Bin; CPUIDLE_STATE_MAX] {
    intercepts: u32,
    hits: u32,
    recent: u32,
  },
  total: u32,
  next_recent_idx: u32,
  recent_idx: [i32; NR_RECENT],
  tick_wakeup: bool,
  time_span_ns: u64,
  sleep_length_ns: u64,
}
```

`teo_select`:
1. `sleep_length_ns = tick_nohz_get_sleep_length(&delta_tick)`.
2. Find deepest enabled state ≤ latency_req → `candidate_idx`.
3. Sum `hits_shallow = Σ hits[0..candidate_idx]`, `intercepts_shallow = Σ intercepts[0..candidate_idx]`.
4. Find `max_intercept_idx` among shallower bins.
5. If `intercepts_shallow > (hits_shallow + intercepts_shallow + Σ deeper) / 2`: candidate = `max_intercept_idx`.
6. If `state[candidate].target_residency_ns < TICK_NSEC`: `*stop_tick = false`.
7. If `sleep_length_ns < state[candidate].target_residency_ns`: `teo_find_shallower_state` walks back.

`ladder::DeviceState` per-CPU per-state:

```
struct DeviceState {
  threshold: { promotion_count: u32, demotion_count: u32, promotion_time_ns: u64, demotion_time_ns: u64 },
  stats:     { promotion_count: i32, demotion_count: i32 },
}
```

`ladder_select_state`:
1. `last_idx = dev->last_state_idx`.
2. `last_residency = dev->last_residency_ns - drv->states[last_idx].exit_latency_ns`.
3. Promote if: `last_idx < state_count - 1` AND `next state enabled` AND `last_residency > promotion_time_ns` AND `exit_latency of next ≤ latency_req` → bump `stats.promotion_count`; if ≥ `threshold.promotion_count` (=4), `ladder_do_selection(dev, ldev, last_idx, last_idx + 1)`.
4. Demote if `last_residency < demotion_time_ns` AND `last_idx > 0` → bump `stats.demotion_count`; if ≥ `threshold.demotion_count` (=1), demote.
5. Else stay.

`haltpoll_select`:
1. If `latency_req == 0`: pick state 0, `*stop_tick = false`.
2. If `dev->poll_limit_ns == 0`: pick state 1 (halt).
3. If `dev->last_state_idx == 0` (was polling): if `poll_time_limit`, halt (idx=1); else continue polling (idx=0, `*stop_tick = false`).
4. If last was halt: poll first (idx=0, `*stop_tick = false`).

`adjust_poll_limit` (called from haltpoll_reflect):
1. If `block_ns > poll_limit_ns AND ≤ guest_halt_poll_ns`: grow → `poll_limit_ns *= guest_halt_poll_grow` (capped at `guest_halt_poll_ns`).
2. If `block_ns > guest_halt_poll_ns AND allow_shrink`: shrink → `poll_limit_ns /= guest_halt_poll_shrink`.

## Hardening

(Inherits row-1 features from `drivers/cpuidle/00-overview.md` § Hardening.)

governor-specific reinforcement:

- **Per-governor per-CPU state allocated by governor `.enable`, freed by `.disable`** — paired with governor swap.
- **menu `correction_factor[]` init to `RESOLUTION * DECAY`** — defense against div-by-zero on first sample.
- **menu `which_bucket` returns 0..5** — defense against bucket OOB.
- **menu `intervals[]` ring index masked** — defense against ptr OOB.
- **teo `recent_idx[NR_RECENT]`** — bounded ring; defense against unbounded prediction history.
- **teo per-bin `intercepts + hits` capped by `total` decay** — defense against integer overflow on long-running idle.
- **ladder `promotion_count` / `demotion_count` reset on selection** — defense against stuck-in-state.
- **ladder threshold derived from `exit_latency_ns`** — defense against malformed cpuidle_state with 0 latency.
- **haltpoll only registers under `kvm_para_available()`** — defense against running KVM-guest governor on bare metal.
- **haltpoll tunables `module_param(uint, 0644)`** — write CAP_SYS_ADMIN-gated via filesystem perms.
- **Governor swap holds `cpuidle_lock`** — defense against in-flight `.select` racing teardown.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `menu_device`, `teo_cpu`, `ladder_device`; per-CPU prediction state DEFINE_PER_CPU not exposed via copy_*_user.
- **PAX_KERNEXEC** — governor cores in W^X kernel text; `menu_governor`, `teo_governor`, `ladder_governor`, `haltpoll_governor` cpuidle_governor vtables live in `__ro_after_init` after registration.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `menu_select`, `teo_select`, `ladder_select_state`, `haltpoll_select`, and per-governor `.reflect` paths.
- **PAX_REFCOUNT** — saturating `refcount_t` on per-governor active-CPU count during swap (governor-disable per-CPU sequence); overflow trap defeats swap-during-select races.
- **PAX_MEMORY_SANITIZE** — zero-on-free for per-CPU `menu_device`, `teo_cpu`, `ladder_device`, and haltpoll dynamic-poll caches on governor swap or cpu hotplug-offline; prevents leaked correction-factor / hits-intercepts history across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every governor-related sysfs store (`current_governor`, haltpoll `guest_halt_poll_ns` family); reject user-pointer deref outside `kstrtouint_from_user` / `sysfs_streq`.
- **PAX_RAP / kCFI** — `cpuidle_governor` (`init`, `exit`, `enable`, `disable`, `select`, `reflect`) for menu/teo/ladder/haltpoll marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms for governor helpers and per-CPU prediction-state pointers behind CAP_SYSLOG; suppress `%p` in governor-decision tracepoints.
- **GRKERNSEC_DMESG** — restrict governor-registration banners and per-governor tunable dumps to CAP_SYSLOG so attackers cannot fingerprint prediction-state layout via dmesg.
- **Prediction heuristics constant-time** — menu correction-factor math + teo bin-sum loop bounded by `BUCKETS=6` / `CPUIDLE_STATE_MAX`; refuse any data-dependent branch that would create a timing channel between CPU-affinity-isolated workloads sharing a package.
- **Halt-poll gate** — haltpoll governor registered iff `kvm_para_available()` returns true at init; `module_param` writes audit-logged; defense against haltpoll loading on bare-metal where the poll-then-halt sequence is wasted energy and creates a non-determinism side channel.
- **Idle-load avoidance** — governor `.select` paths use only per-CPU prediction state (no shared atomic counters in the hot path); defense against cross-CPU contention escalating into a covert channel between SMT siblings.
- **Tunable bounded floor / ceiling** — haltpoll `guest_halt_poll_ns`, `guest_halt_poll_grow_start`, ladder `PROMOTION_COUNT` / `DEMOTION_COUNT` validated to non-zero defaults; refuse sysfs/module-param values that would force pathological poll windows or zero-threshold ladder promotion.
- **Governor swap audit** — `cpuidle_switch_governor` logs caller, prior governor, new governor via audit subsystem; defense against silent prediction-policy hijack from a privileged-but-untrusted process.

Rationale: cpuidle governors translate scheduler-state + pm_qos requests into MWAIT/WFI hints — their predictions directly affect wake latency observable by realtime workloads and microarchitectural state observable by side-channel attackers. A tampered `.select` callback can pin the package in a state that violates pm_qos, a leaked per-CPU `correction_factor[]` reveals workload patterns to a SMT-sibling, and an unbounded `guest_halt_poll_ns` burns CPU + power in a hostile guest. RAP/kCFI on every governor vtable, constant-time prediction loops, KVM-detection gate on haltpoll, audit on governor swap, refcount overflow trapping on per-CPU enable/disable counts, and per-CPU-only state with no shared hot-path counters turn the governor layer from "best-effort heuristic" into a structural enforcement boundary that meets pm_qos contracts + cross-tenant isolation guarantees.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-governor sysfs tunables beyond the standard `cpuidle/current_governor` (covered as needed in per-governor expansion future Tier-3)
- `cpuidle-haltpoll` driver (the driver wrapper around the haltpoll governor in `drivers/cpuidle/cpuidle-haltpoll.c`, covered separately)
- ACPI _CST → cpuidle_state interplay (covered in `acpi-processor-idle.md` future Tier-3)
- Coupled CPU rendezvous (covered in `coupled.md` future Tier-3)
- POWER-specific governors (covered per-arch)
- 32-bit-only paths
- Implementation code
