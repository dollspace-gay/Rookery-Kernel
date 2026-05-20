# Tier-3: drivers/devfreq/{devfreq,governor_*}.c — devfreq framework + bundled governors (OPP-backed device frequency scaling)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/devfreq/devfreq.c
  - drivers/devfreq/devfreq-event.c
  - drivers/devfreq/governor_simpleondemand.c
  - drivers/devfreq/governor_performance.c
  - drivers/devfreq/governor_powersave.c
  - drivers/devfreq/governor_userspace.c
  - drivers/devfreq/governor_passive.c
  - include/linux/devfreq.h
  - include/linux/devfreq-governor.h
  - include/linux/devfreq-event.h
-->

## Summary

devfreq is the generic frequency-scaling framework for non-CPU devices that nonetheless expose an OPP (Operating Performance Point — cross-ref `drivers/opp/opp-core.md`) table — memory-bus controllers (Exynos MIF/INT, Tegra30 ACTMON, Rockchip DMC, MediaTek CCI, Allwinner MBUS, NXP DDR, HiSilicon uncore), GPU mali devfreq nodes, ISP/VPU/NPU clock domains. It mirrors `cpufreq` (cross-ref `drivers/cpufreq/cpufreq.md`): per-device `struct devfreq` carries a governor, a polling interval, a profile (driver-supplied `target` + optional `get_dev_status`), QoS min/max constraints, transition notifiers, and statistics; governors plug in via `struct devfreq_governor` and supply `get_target_freq`.

This Tier-3 covers `devfreq.c` (~2300 lines: core, sysfs, governor registry, monitor work, QoS) and the five bundled governors: `simple_ondemand`, `performance`, `powersave`, `userspace`, `passive`.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct devfreq` | per-device control block — governor, freq, profile, QoS | `drivers::devfreq::Devfreq` |
| `struct devfreq_dev_profile` | driver-supplied: `target`, `get_dev_status`, `get_cur_freq`, `polling_ms`, `freq_table[]` | `drivers::devfreq::DevProfile` |
| `struct devfreq_governor` | governor ops: `get_target_freq`, `event_handler`, `attrs` flag | `drivers::devfreq::Governor` |
| `struct devfreq_dev_status` | per-tick: total/busy time + private | `drivers::devfreq::DevStatus` |
| `devfreq_add_device(dev, profile, gov_name, data)` | register a device into devfreq with named governor + private data | `Devfreq::add` |
| `devfreq_remove_device(devfreq)` | unregister + stop monitor + release governor | `Devfreq::remove` |
| `devfreq_add_governor(gov)` / `_remove_governor(gov)` | register/unregister a governor in `devfreq_governor_list` | `Governor::register` / `_unregister` |
| `devfreq_get_freq_range(devfreq, *min, *max)` | compute effective min/max from profile + QoS + scaling_{min,max}_freq | `Devfreq::freq_range` |
| `devfreq_get_freq_level(devfreq, freq)` | lookup freq → freq_table index | `Devfreq::freq_level` |
| `set_freq_table(devfreq)` | per-add-device: build freq_table from OPP table if profile didn't supply one | `Devfreq::set_freq_table` |
| `devfreq_update_status(devfreq, freq)` | bump per-level time-in-state stats + transition counter | `Devfreq::update_status` |
| `devfreq_set_target(devfreq, new_freq, flags)` | clamp + call profile.target(freq, flags) + emit transition notifiers | `Devfreq::set_target` |
| `devfreq_update_target(devfreq, freq)` | wrapper for governors: re-evaluate target given a freq hint | `Devfreq::update_target` |
| `update_devfreq(devfreq)` | core reevaluation: governor.get_target_freq → set_target | `Devfreq::update` |
| `devfreq_monitor(work)` / `_start/_stop/_suspend/_resume` | polling-timer workqueue entry point | `Devfreq::monitor_*` |
| `devfreq_notifier_call(nb, type, devp)` | OPP-table availability notifier | `Devfreq::on_opp_notify` |
| `qos_{min,max}_notifier_call(...)` | PM-QoS min/max freq change notifier | `Devfreq::on_qos_notify` |
| `devfreq_register_notifier(devfreq, nb, list)` | transition-notifier registration (PRE/POSTCHANGE) | `Devfreq::register_notifier` |
| `devfreq_simple_ondemand_func` | governor: percent-busy threshold with upthreshold/downdifferential | `Governor::SimpleOndemand` |
| `governor_performance` / `_powersave` | governor: pin to max/min | `Governor::Performance` / `_Powersave` |
| `governor_userspace` | governor: sysfs `set_freq` driven; no monitor | `Governor::Userspace` |
| `governor_passive` | governor: follow a parent devfreq/devfreq_event with rate map | `Governor::Passive` |

## Compatibility contract

REQ-1: Per-device sysfs surface at `/sys/class/devfreq/<dev>/`:
- `governor` (R/W, swap-governor)
- `available_governors` (R)
- `cur_freq` / `target_freq` (R)
- `min_freq` / `max_freq` (R/W, PM-QoS-backed)
- `available_frequencies` (R)
- `polling_interval` (R/W, governor-dependent)
- `trans_stat` (R) — per-level time + transition matrix
- per-governor attrs gated by `governor.attrs` bit (`DEVFREQ_GOV_ATTR_POLLING_INTERVAL`, `_TIMER`).

REQ-2: Governor swap (`echo simple_ondemand > governor`) must: stop current governor (`event_handler(DEVFREQ_GOV_STOP)`), validate new governor against `governor.immutable` (rejects swap of `passive` ↔ non-passive), then start new (`DEVFREQ_GOV_START`). Locking: `devfreq_list_lock` + per-`devfreq->lock` mutex.

REQ-3: freq_table built from OPP table on add when profile.freq_table is NULL; missing OPPs (disabled) excluded; `available_frequencies` enumerates enabled OPPs.

REQ-4: `set_freq_table` re-runs on OPP `OPP_EVENT_ADD/_REMOVE/_ENABLE/_DISABLE` notifier — frequency set may auto-shift if current freq becomes unavailable.

REQ-5: PM-QoS integration — per-`devfreq`, register min-freq + max-freq QoS requests; on change, `qos_notifier_call → update_devfreq`. CPU-freq-style cooling integration via `devfreq_cooling.c`.

REQ-6: Monitor work runs on `devfreq_wq` (system_freezable_wq) every `polling_ms` and calls `update_devfreq`. Per-governor `event_handler(DEVFREQ_GOV_SUSPEND/_RESUME)` invoked on `pm_suspend_prepare`.

REQ-7: Transition notifier chain emits `DEVFREQ_PRECHANGE` + `DEVFREQ_POSTCHANGE` around `profile.target` — consumers (interconnect, regulator coupling) hook here.

REQ-8: Governor `event_handler(DEVFREQ_GOV_UPDATE_INTERVAL)` propagates a polling-interval change without restart.

REQ-9: `passive` governor: read-only follower of a "parent" `devfreq` or `devfreq_event` — useful for paired controllers (DDR ↔ uncore on Exynos). Configured via `devfreq_passive_data` table.

REQ-10: `userspace` governor: user-space writes `set_freq` sysfs → governor `target_freq` set; monitor not enabled; bypasses busy-time logic.

REQ-11: `simple_ondemand` governor parameters: `upthreshold` (% busy to raise freq, default 90), `downdifferential` (% headroom below upthreshold, default 5).

REQ-12: devfreq-event subsystem (`devfreq-event.c`) — PMU-like counter sources (Exynos PPMU, Rockchip dfi, Allwinner H6 etc.) registered via `devfreq_event_add_edev`; governors consume via `devfreq_event_get_event`.

## Acceptance Criteria

- [ ] AC-1: `devfreq_add_device(&exynos_bus_dev, &profile, "simple_ondemand", data)` on Exynos5422 boots with `cur_freq` matching profile.initial_freq.
- [ ] AC-2: `echo performance > /sys/class/devfreq/<dev>/governor` swaps governor and `cur_freq == max_freq` within one monitor tick.
- [ ] AC-3: `echo 100000000 > /sys/class/devfreq/<dev>/min_freq` raises `cur_freq` to at least 100 MHz via PM-QoS.
- [ ] AC-4: Reading `/sys/class/devfreq/<dev>/trans_stat` shows non-zero `total_trans` after a workload that crosses thresholds.
- [ ] AC-5: Suspend/resume cycle preserves governor + last freq; `devfreq_monitor_suspend/_resume` invoked.
- [ ] AC-6: `passive` governor on a child devfreq follows parent transitions per `parent_freq_table`.
- [ ] AC-7: OPP table disable (`dev_pm_opp_disable`) of current freq triggers `update_devfreq` to a still-available freq.
- [ ] AC-8: `userspace` governor: `echo 800000000 > set_freq` sets `cur_freq` to 800 MHz exactly.
- [ ] AC-9: KUnit + selftest on `tools/testing/selftests/devfreq/` (where present) passes under KASAN.

## Architecture

`Devfreq` lives in `drivers::devfreq::Devfreq`:

```
struct Devfreq {
  node: ListNode,                  // devfreq_list
  lock: Mutex,                     // per-devfreq
  dev: Arc<Device>,                // class_device /sys/class/devfreq/<name>
  profile: Arc<DevProfile>,        // driver-supplied
  governor: Option<Arc<Governor>>,
  governor_name: KString,
  opp_table: Option<Arc<OppTable>>,
  freq_table: KBox<[u32]>,
  max_state: u32,
  previous_freq: u64,
  last_status: DevStatus,
  data: *mut (),                   // governor private
  governor_data: *mut (),
  scaling_min_freq: u64,
  scaling_max_freq: u64,
  stop_polling: AtomicBool,
  suspend_freq: u64,
  resume_freq: u64,
  suspend_count: AtomicU32,
  time_in_state: KBox<[u64]>,
  total_trans: u32,
  trans_table: KBox<[u32]>,
  qos_min_req: PmQosRequest,
  qos_max_req: PmQosRequest,
  qos_min_nb: NotifierBlock,
  qos_max_nb: NotifierBlock,
  nh: NotifierHead,                // transition notifiers
  work: DelayedWork,               // devfreq_monitor
  transition_notifier_list: RawNotifierHead,
}
```

Per-add `Devfreq::add`:
1. Validate `profile.initial_freq != 0`, `profile.target != NULL`.
2. Allocate `Devfreq`, init `lock`, attach to `dev->parent`.
3. `set_freq_table(devfreq)` — copy from profile or build from OPP table.
4. `find_devfreq_governor(gov_name)` (under `devfreq_list_lock`); if absent, `try_then_request_governor` loads the module.
5. `governor.event_handler(DEVFREQ_GOV_START)` — governor allocates `governor_data`, kicks monitor.
6. Register sysfs class device; init PM-QoS reqs; register OPP notifier.

Per-monitor tick `devfreq_monitor`:
1. Re-take `devfreq->lock`.
2. `update_devfreq(devfreq)`:
   a. `governor.get_target_freq(devfreq, &freq)` — governor may consult `profile.get_dev_status(devfreq, &status)`.
   b. `devfreq_get_freq_range(devfreq, &min, &max)` — clamp.
   c. `devfreq_set_target(devfreq, freq, flags)`:
      - `nh.call_chain(DEVFREQ_PRECHANGE, freq_info)`.
      - `profile.target(dev, &freq, flags)` — driver actually programs PLL/regulator.
      - `nh.call_chain(DEVFREQ_POSTCHANGE, freq_info)`.
      - `devfreq_update_status(devfreq, freq)`.
3. Re-queue `delayed_work` if `polling_ms > 0` and not stopped.

Per-governor `simple_ondemand_func`:
1. `profile.get_dev_status(dev, &status)`.
2. compute `busy = status.busy_time * 100 / status.total_time`.
3. If `busy >= upthreshold` → `*freq = DEVFREQ_MAX_FREQ`.
4. Else compute `target = status.current_frequency * busy / (upthreshold - downdifferential)`; clamp.

Per-governor swap (sysfs `governor` store):
1. `find_devfreq_governor(new_name)`.
2. Old governor: `event_handler(DEVFREQ_GOV_STOP)`.
3. New governor immutable? must match old immutable flag (passive ↔ passive only).
4. New governor: `event_handler(DEVFREQ_GOV_START)`.
5. `update_devfreq(devfreq)`.

Per-OPP notify `devfreq_notifier_call`:
- On `OPP_EVENT_ADD/_REMOVE/_ENABLE/_DISABLE` → `set_freq_table` rebuild + `update_devfreq`.

Per-PM-QoS notify `qos_{min,max}_notifier_call`:
- Update `scaling_{min,max}_freq` → `update_devfreq`.

## Hardening

(Inherits row-1 features from `drivers/00-overview.md` § Hardening.)

devfreq-specific reinforcement:

- **`governor.immutable` flag for passive** — swap to/from `passive` denied unless symmetric; defense against governor-confusion.
- **Per-`devfreq->lock` mutex around all sysfs writes** — defense against TOCTOU on freq/governor changes.
- **`profile.target` invoked only under `devfreq->lock`** — defense against re-entrant freq programming during transition notifier callback.
- **OPP-disable shifts current freq atomically** — defense against driver attempting to operate at a disabled OPP.
- **`devfreq_remove_device` waits for outstanding monitor work** — defense against use-after-free in delayed_work.
- **Governor module ref bumped while attached** — defense against module unload while devfreq holds a governor pointer.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `devfreq`, `devfreq_governor`, `devfreq_dev_profile`, and per-governor private data; sysfs read/write paths use `kstrtoX` bounded copies.
- **PAX_KERNEXEC** — `devfreq_class`, `governor`-list ops, sysfs attribute groups, and per-governor `event_handler` placed in `__ro_after_init` text.
- **PAX_RANDKSTACK** — randomize kernel-stack offset on `devfreq_monitor` workqueue entries, sysfs store handlers, and `update_devfreq` paths.
- **PAX_REFCOUNT** — saturating refcount on `devfreq` class_device + governor module reference; overflow trap defeats race UAFs around `add_device` / `remove_device`.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `devfreq` struct, governor private data, and `freq_table`/`trans_table`/`time_in_state` arrays so stale freq layouts cannot bleed across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on sysfs store callbacks (`governor`, `min_freq`, `max_freq`, `polling_interval`, `set_freq`); reject mis-sized writes.
- **PAX_RAP / kCFI** — `devfreq_dev_profile.target` / `.get_dev_status` / `.get_cur_freq`, governor `get_target_freq` / `event_handler`, and notifier chain dispatch marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms exposure of `devfreq` pointers in trans_stat seq files behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict `devfreq: ...` governor-load + transition-failure banners to CAP_SYSLOG so attackers cannot probe PLL topology.
- **Governor swap CAP_SYS_ADMIN** — sysfs `governor` store requires CAP_SYS_ADMIN in the device's user_ns; defense against unprivileged DoS via repeated governor thrash.
- **Freq-table validated** — every entry in `freq_table` cross-checked against `dev_pm_opp_find_freq_exact` at build time; refuse to attach if profile lies about supported freqs.
- **Throttling lock** — `devfreq->lock` held across `profile.target` so that a malicious governor cannot interleave two transitions on the same device.
- **`target_freq` SIZE_OVERFLOW** — `min_freq * busy / (upthreshold - downdifferential)` arithmetic uses `u64` saturating math; defense against integer overflow producing a `target_freq = 0` UAF.
- **PM-QoS bounded** — sysfs `min_freq`/`max_freq` writes validated against OPP table; refuse out-of-range; defense against PM-QoS injection forcing the bus controller above safe limits.

Rationale: a misbehaving devfreq path can over-volt or over-clock a memory bus / GPU, leading to silent data corruption or hardware damage. RAP/kCFI on `profile.target` and governor vtables, refcount-overflow trapping on governors, SMAP/PAN on every sysfs store, CAP_SYS_ADMIN on governor swap, OPP-table validation on every freq publish, and saturating arithmetic in the busy-percent ondemand math turn devfreq from "a polling loop driving PLLs" into a structurally hardened frequency-scaling boundary.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- devfreq-event PMU sources (covered in `devfreq-event.md` future Tier-3)
- Per-SoC devfreq drivers (`exynos-bus.c`, `rk3399_dmc.c`, `tegra30-devfreq.c`, …) — each gets its own future Tier-3 if upstream warrants
- devfreq_cooling thermal integration (covered in `devfreq-cooling.md` future Tier-3)
- 32-bit-only paths
- Implementation code
