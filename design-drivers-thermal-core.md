---
title: "Tier-3: drivers/thermal/thermal_core.c — thermal-zone + cooling-device framework"
tags: ["tier-3", "drivers", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

The **thermal core** is the framework registering every temperature sensor (`struct thermal_zone_device`) and every actuator that can dissipate heat (`struct thermal_cooling_device` — CPU-freq throttler, fan, idle-injector), and binding them through a per-zone **governor** (`struct thermal_governor`) that decides how aggressively to throttle when a trip-point is crossed. Per-zone trip-points carry a `type` (`THERMAL_TRIP_ACTIVE` = engage fan, `THERMAL_TRIP_PASSIVE` = throttle CPU, `THERMAL_TRIP_HOT` = userspace notification, `THERMAL_TRIP_CRITICAL` = emergency poweroff) and a `temperature` + `hysteresis`. Per-zone polling runs out of a workqueue (`thermal_wq`, `WQ_POWER_EFFICIENT`) that calls `__thermal_zone_device_update` → `ops.get_temp` → trip-classification (`thermal_zone_handle_trips`) → governor → optional `ops.set_trips` interrupt-window programming. Per-`THERMAL_TRIP_CRITICAL` a board can request `HWPROT_ACT_SHUTDOWN` or `HWPROT_ACT_REBOOT`; default is `__hw_protection_trigger` after `CONFIG_THERMAL_EMERGENCY_POWEROFF_DELAY_MS`. Critical for: laptop never melting, server racks shedding load before silicon damage, container-OOM/thermal-OOM coordination.

This Tier-3 covers `drivers/thermal/thermal_core.c` (~1921 lines).

### Acceptance Criteria

- [ ] AC-1: `thermal_zone_device_register_with_trips` creates `/sys/class/thermal/thermal_zone%d/` with `type`, `temp`, `mode`, `policy`, `available_policies`, `trip_point_<N>_temp`, `trip_point_<N>_type`, `trip_point_<N>_hyst`.
- [ ] AC-2: `thermal_cooling_device_register` creates `/sys/class/thermal/cooling_device%d/` with `type`, `max_state`, `cur_state`.
- [ ] AC-3: Crossing a `THERMAL_TRIP_PASSIVE` trip upward: `tz->passive++`; subsequent polls use `passive_delay_jiffies`; governor `trip_crossed` invoked with `upward=true`.
- [ ] AC-4: Crossing a `THERMAL_TRIP_CRITICAL` trip upward: `tz->ops.critical(tz)` called (default `__hw_protection_trigger`).
- [ ] AC-5: Crossing a `THERMAL_TRIP_HOT` trip upward: `tz->ops.hot(tz)` called if non-NULL; governor `trip_crossed` NOT invoked.
- [ ] AC-6: `ops.get_temp` returning `-EAGAIN`: re-poll at `THERMAL_RECHECK_DELAY` without back-off.
- [ ] AC-7: `ops.get_temp` repeatedly failing past `THERMAL_MAX_RECHECK_DELAY`: zone forced to `THERMAL_DEVICE_DISABLED`; `dev_crit` if any critical trip remained valid.
- [ ] AC-8: Echo `step_wise` to `/sys/class/thermal/thermal_zone0/policy`: `thermal_set_governor` succeeds; netlink `THERMAL_GENL_EVENT_TZ_GOV_CHANGE` emitted.
- [ ] AC-9: `thermal_cooling_device_register` with NULL `ops.set_cur_state`: returns `-EINVAL`.
- [ ] AC-10: Register cdev after zones already exist: cdev binds to every zone whose `ops.should_bind` returns true (with `cool_spec.upper == THERMAL_NO_LIMIT` mapped to `max_state`).
- [ ] AC-11: `thermal_zone_device_unregister`: `poll_queue` cancelled synchronously; `removal` completion awaited; netlink `THERMAL_GENL_EVENT_TZ_DELETE` emitted.
- [ ] AC-12: `thermal_zone_set_trip_temp` with `temp == THERMAL_TEMP_INVALID` when current temperature ≥ old threshold: synthetic falling-edge `thermal_trip_crossed(upward=false)`; trip moved to `trips_invalid`.
- [ ] AC-13: `thermal_pm_prepare`: in-flight polls cancelled, `thermal_pm_suspended = true`, workqueue flushed; resume re-runs `thermal_zone_device_init` + governor `THERMAL_TZ_RESUME` + update.
- [ ] AC-14: `thermal_cooling_device_update` after `max_state` shrinks: instances with `upper > new max_state` clamped; `target` clamped if defined.
- [ ] AC-15: Concurrent `thermal_zone_device_register_with_trips` and `thermal_register_governor` matching by `tzp->governor_name`: zone ends up bound under exactly one governor regardless of ordering.

### Architecture

```
struct ThermalZoneDevice {
  id: u32,                              // /sys/class/thermal/thermal_zone%d
  type: BoundedString<THERMAL_NAME_LEN>,
  temperature: AtomicI32,               // millidegrees-C
  last_temperature: i32,
  mode: ThermalDeviceMode,              // ENABLED | DISABLED
  state: BitFlags<TzStateFlag>,         // READY | INIT | SUSPENDED | RESUMING | EXIT
  governor: Option<Arc<ThermalGovernor>>,
  ops: ThermalZoneDeviceOps,            // get_temp + optional set_trips/change_mode/should_bind/critical/hot/get_crit_temp
  tzp: Option<Box<ThermalZoneParams>>,  // governor_name + no_hwmon + ...
  trips: Vec<ThermalTripDesc>,          // num_trips
  trips_high: SortedList,               // ↑ threshold = temperature
  trips_reached: SortedList,            // ↑ threshold = temperature − hyst
  trips_invalid: List,                  // threshold = INT_MAX
  passive: i32,                         // count of crossed passive trips
  polling_delay_jiffies: u64,
  passive_delay_jiffies: u64,
  recheck_delay_jiffies: u64,
  prev_low_trip: i32,
  prev_high_trip: i32,
  poll_queue: DelayedWork,
  node: ListNode,                       // thermal_tz_list
  ida: Ida,                             // instance ids
  lock: Mutex,
  removal: Completion,
  resume: Completion,
  devdata: *mut (),
}

struct ThermalCoolingDevice {
  id: u32,                              // /sys/class/thermal/cooling_device%d
  type: KStringConst,
  max_state: u64,
  ops: ThermalCoolingDeviceOps,         // get_max_state + get_cur_state + set_cur_state (all mandatory)
  thermal_instances: List,              // bound (trip, instance)
  np: Option<*const DeviceNode>,
  devdata: *mut (),
  node: ListNode,                       // thermal_cdev_list
  lock: Mutex,
  updated: bool,
}

struct ThermalGovernor {
  name: BoundedString<THERMAL_NAME_LEN>,
  bind_to_tz: Option<fn(&ThermalZoneDevice) -> Result<()>>,
  unbind_from_tz: Option<fn(&ThermalZoneDevice)>,
  update_tz: Option<fn(&ThermalZoneDevice, ThermalNotifyEvent)>,
  trip_crossed: Option<fn(&ThermalZoneDevice, &ThermalTrip, upward: bool)>,
  manage: Option<fn(&ThermalZoneDevice)>,
  governor_list: ListNode,
}

struct ThermalTrip {
  temperature: i32,                     // millidegrees-C or THERMAL_TEMP_INVALID
  hysteresis: i32,
  type: ThermalTripType,                // ACTIVE | PASSIVE | HOT | CRITICAL
  priv: *mut (),
}

struct ThermalTripDesc {
  trip: ThermalTrip,
  threshold: i32,                       // working threshold (re-derived per bucket)
  list_node: ListNode,                  // member of trips_high | trips_reached | trips_invalid
  thermal_instances: List,
}

struct ThermalInstance {
  id: u32,
  cdev: Arc<ThermalCoolingDevice>,
  trip: *const ThermalTrip,
  upper: u64,
  lower: u64,
  target: u64,                          // THERMAL_NO_TARGET = !0
  weight: u32,
  upper_no_limit: bool,
  initialized: bool,
  name: [u8; ...],                      // "cdevN"
  attr: DeviceAttribute,                // cdevN_trip_point
  attr_name: [u8; ...],
  weight_attr: DeviceAttribute,         // cdevN_weight
  weight_attr_name: [u8; ...],
  trip_node: ListNode,                  // ThermalTripDesc::thermal_instances
  cdev_node: ListNode,                  // ThermalCoolingDevice::thermal_instances
}
```

`ThermalCore::register_zone_with_trips(type, trips, num_trips, devdata, ops, tzp, passive_delay_ms, polling_delay_ms) -> Result<Arc<ThermalZoneDevice>>`:
1. Validate `type` non-empty + < `THERMAL_NAME_LEN`.
2. Validate `num_trips ≥ 0` and (`num_trips > 0` ⟹ `trips.is_some()`).
3. Validate `ops.get_temp.is_some()`.
4. Validate `polling_delay_ms == 0 ∨ passive_delay_ms ≤ polling_delay_ms`.
5. Verify `thermal_class` initialized.
6. Allocate `tz` with trailing trip slots; allocate `tzp` clone if provided.
7. Initialize lists, ida, mutex, completions, copy ops (defaulting `critical` to `ThermalCore::zone_critical`), copy trip table (each in `trips_invalid`).
8. `tz.id = thermal_tz_ida.alloc()`.
9. Convert delays to jiffies; `recheck_delay_jiffies = THERMAL_RECHECK_DELAY`; `state = INIT`.
10. `device.set_name("thermal_zone{}", id)`.
11. `ThermalCore::zone_init(&mut tz)` (poll-queue init; promote valid trips to `trips_high`).
12. `ThermalCore::init_governor(&mut tz)` under `thermal_governor_lock`.
13. `thermal_zone_create_device_groups`; `device_register`.
14. If `!tzp.no_hwmon`: `thermal_add_hwmon_sysfs` (drivers/thermal/thermal_hwmon.c).
15. `thermal_thresholds_init`.
16. `ThermalCore::zone_init_complete`: add to `thermal_tz_list`, sweep `thermal_cdev_list` calling `__thermal_zone_cdev_bind`, clear `INIT`, set `SUSPENDED` if `pm_suspended`, kick `update`.
17. Notify netlink (`tz_create`), debug (`tz_add`).
18. Return `Ok(tz)`.

`ThermalCore::update_zone_locked(tz, event)`:
1. /* tz->lock held */.
2. If `state != READY ∨ mode != ENABLED`: return.
3. `temp = __thermal_zone_get_temp(tz)?`.
4. On `Err(e)`: `ThermalCore::recheck_zone(tz, e)`; return.
5. If `temp ≤ THERMAL_TEMP_INVALID`: goto `monitor` (skip everything else).
6. `recheck_delay_jiffies = THERMAL_RECHECK_DELAY`.
7. `last_temperature = temperature; temperature = temp`.
8. `trace_thermal_temperature`; `thermal_genl_sampling_temp(id, temp)`; `notify_event = event`.
9. `low = -INT_MAX; high = INT_MAX`.
10. `ThermalCore::handle_trips(tz, governor, &low, &high)`.
11. `thermal_thresholds_handle(tz, &low, &high)`.
12. `thermal_zone_set_trips(tz, low, high)`.
13. If `governor.manage.is_some()`: `governor.manage(tz)`.
14. `thermal_debug_update_trip_stats`.
15. monitor: `ThermalCore::monitor_zone(tz)`.

`ThermalCore::handle_trips(tz, gov, low, high)`:
1. `way_down_list = List::new()`.
2. For `td` in `trips_reached` reversed: if `td.threshold > tz.temperature`: `trip_crossed(tz, td, gov, upward=false)`; `list_move(&td.node, &way_down_list)`.
3. For `td` in `trips_high`: if `td.threshold ≤ tz.temperature`: `trip_crossed(tz, td, gov, upward=true)`; `move_to_trips_reached`.
4. Drain `way_down_list` into `trips_high` via `move_to_trips_high`.
5. If `!trips_reached.empty()`: `*low = trips_reached.last().threshold - 1`.
6. If `!trips_high.empty()`: `*high = trips_high.first().threshold`.

`ThermalCore::trip_crossed(tz, td, gov, upward)`:
1. `trip = &td.trip`.
2. If upward:
   - If `trip.type == PASSIVE`: `tz.passive += 1`.
   - Else if `trip.type ∈ {CRITICAL, HOT}`: `handle_critical_trips(tz, trip)`.
   - `thermal_notify_tz_trip_up(tz, trip)`; `thermal_debug_tz_trip_up`.
3. Else (falling):
   - If `trip.type == PASSIVE`: `tz.passive -= 1`; `debug_assert!(tz.passive ≥ 0)`.
   - `thermal_notify_tz_trip_down`; `thermal_debug_tz_trip_down`.
4. `thermal_governor_trip_crossed(gov, tz, trip, upward)` — short-circuits for HOT/CRITICAL.

`ThermalCore::handle_critical_trips(tz, trip)`:
1. `trace_thermal_zone_trip`.
2. Match `trip.type`:
   - `CRITICAL`: `tz.ops.critical(tz)`.
   - else if `tz.ops.hot.is_some()`: `tz.ops.hot(tz)`.

`ThermalCore::zone_halt(tz, action)`:
1. `dev_emerg("Temperature too high: critical temperature reached")`.
2. `__hw_protection_trigger("Temperature too high", CONFIG_THERMAL_EMERGENCY_POWEROFF_DELAY_MS, action)`.

`ThermalCore::recheck_zone(tz, error)`:
1. If `error == -EAGAIN`: `set_polling(tz, THERMAL_RECHECK_DELAY)`; return.
2. If `recheck_delay_jiffies == THERMAL_RECHECK_DELAY`: `dev_info` once.
3. `set_polling(tz, recheck_delay_jiffies)`.
4. `recheck_delay_jiffies += max(recheck_delay_jiffies >> 1, 1)`.
5. If `recheck_delay_jiffies > THERMAL_MAX_RECHECK_DELAY`:
   - `disable_broken_zone(tz)`.
   - `recheck_delay_jiffies = THERMAL_RECHECK_DELAY`.

`ThermalCore::disable_broken_zone(tz)`:
1. `dev_err("Unable to get temperature, disabling!")`.
2. `__thermal_zone_device_set_mode(tz, DISABLED)`; netlink `tz_disable`.
3. For each trip with `type == CRITICAL ∧ temperature > THERMAL_TEMP_INVALID`: `dev_crit("Disabled thermal zone with critical trip point")`; break.

`ThermalCore::register_cdev(np, type, devdata, ops) -> Result<Arc<ThermalCoolingDevice>>`:
1. Validate `ops.{get_max_state, get_cur_state, set_cur_state}.are_present()`.
2. Validate `thermal_class` initialized.
3. Allocate `cdev`, allocate id from `thermal_cdev_ida`.
4. `kstrdup_const(type ?? "")`.
5. Initialize `lock`, `thermal_instances`, copy `np`, `ops`, `devdata`.
6. `cdev.max_state = ops.get_max_state(cdev)?`.
7. `current_state = ops.get_cur_state(cdev).unwrap_or(u64::MAX)` (debug only).
8. `thermal_cooling_device_setup_sysfs`.
9. `device.set_name("cooling_device{}", id)`; `device_register`.
10. If `current_state ≤ max_state`: `thermal_debug_cdev_add(cdev, current_state)`.
11. Under `thermal_list_lock`: `list_add(&cdev.node, &thermal_cdev_list)`; for each `tz ∈ thermal_tz_list`: `ThermalCore::zone_cdev_bind(tz, cdev)`.
12. Return `Ok(cdev)`.

`ThermalCore::bind_cdev_to_trip(tz, td, cdev, cool_spec) -> Result<()>`:
1. Default `cool_spec.lower = 0` if `THERMAL_NO_LIMIT`.
2. Default `cool_spec.upper = cdev.max_state` (and set `upper_no_limit = true`) if `THERMAL_NO_LIMIT`.
3. Validate `lower ≤ upper ≤ cdev.max_state`.
4. Allocate `instance` with `target = THERMAL_NO_TARGET`.
5. `instance.id = tz.ida.alloc()`.
6. Create sysfs link, `cdev<id>_trip_point` (RO), `cdev<id>_weight` (RW).
7. `ThermalCore::instance_add(instance, cdev, td)` — reject `(cdev, td)` duplicate; insert into both lists under `cdev.lock` for cdev-side.
8. `thermal_governor_update_tz(tz, THERMAL_TZ_BIND_CDEV)`.

`ThermalCore::unregister_zone(tz)`:
1. Idempotent on NULL.
2. `thermal_debug_tz_remove`.
3. `ThermalCore::zone_exit(tz)`: under `thermal_list_lock` — if `tz.node.is_unlinked()` return false; set `state |= EXIT`; `list_del_init(&tz.node)`; iter `thermal_cdev_list` calling `__thermal_zone_cdev_unbind`.
4. `cancel_delayed_work_sync(&tz.poll_queue)`.
5. `thermal_thresholds_exit`; `thermal_remove_hwmon_sysfs`.
6. `device_del + put_device`.
7. `thermal_notify_tz_delete`.
8. `tz.removal.wait()`.
9. `thermal_tz_ida.free(id)`; free `tzp`; free zone.

`ThermalCore::set_governor(tz, new_gov) -> Result<()>`:
1. If `tz.governor.unbind_from_tz`: call.
2. If `new_gov.bind_to_tz`: call; on err: `bind_previous_governor`; return err.
3. `tz.governor = new_gov`.

`ThermalCore::pm_prepare()`:
1. Under `thermal_list_lock`: `thermal_pm_suspended = true`.
2. For each `tz`: `pm_prepare_zone(tz)` — if `RESUMING` flag: drop lock, await `tz.resume` completion; set `SUSPENDED`; `cancel_delayed_work(&poll_queue)`.
3. After: `flush_workqueue(thermal_wq)`.

`ThermalCore::pm_complete()`:
1. Under `thermal_list_lock`: `thermal_pm_suspended = false`.
2. For each `tz`: `pm_complete_zone(tz)` — `reinit_completion(&tz.resume)`; set `RESUMING`; reinit `poll_queue` work-fn to `zone_resume_work`; queue with delay = 0.

`ThermalCore::zone_resume_work(tz)`:
1. Under `tz.lock`: if `EXIT`: return.
2. Clear `SUSPENDED | RESUMING`.
3. `thermal_debug_tz_resume`.
4. `zone_init(tz)`.
5. `governor.update_tz(tz, THERMAL_TZ_RESUME)`.
6. `update_zone_locked(tz, THERMAL_TZ_RESUME)`.
7. `tz.resume.complete()`.

### Out of Scope

- `drivers/thermal/thermal_sysfs.c` sysfs surface — covered in `drivers/thermal/sysfs-netlink.md` Tier-3.
- `drivers/thermal/thermal_netlink.c` NETLINK_THERMAL_GENL — covered in `drivers/thermal/sysfs-netlink.md` Tier-3.
- `drivers/thermal/thermal_helpers.c` per-zone-temp helpers — covered alongside core in Tier-3.
- `drivers/thermal/thermal_trip.c` per-trip-iteration helpers — covered alongside core in Tier-3.
- `drivers/thermal/thermal_hwmon.c` hwmon ↔ thermal-zone bridge — covered in `drivers/thermal/hwmon-bridge.md` Tier-3.
- `drivers/thermal/thermal_thresholds.c` userspace trip thresholds — covered separately.
- `drivers/thermal/gov_*.c` governors (`step_wise`, `fair_share`, `user_space`, `bang_bang`, `power_allocator`) — covered in `drivers/thermal/governors.md` Tier-3.
- `drivers/thermal/thermal_debugfs.c` — covered separately if expanded.
- Per-driver sensors (`intel/`, `amd_hsmp/`) — covered in `drivers/thermal/intel.md` / `drivers/thermal/amd-hsmp.md` Tier-3.
- Implementation code.

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct thermal_zone_device` | per-sensor zone | `ThermalZoneDevice` |
| `struct thermal_cooling_device` | per-actuator | `ThermalCoolingDevice` |
| `struct thermal_governor` | per-policy plug-in | `ThermalGovernor` |
| `struct thermal_trip` / `thermal_trip_desc` | per-trip-point | `ThermalTrip` / `ThermalTripDesc` |
| `struct thermal_instance` | per-(trip,cdev) binding | `ThermalInstance` |
| `struct cooling_spec` | per-bind upper/lower/weight | `CoolingSpec` |
| `thermal_register_governor()` / `_unregister_governor()` | per-governor lifecycle | `ThermalCore::register_governor` / `unregister_governor` |
| `thermal_set_governor()` | per-zone governor switch | `ThermalCore::set_governor` |
| `thermal_zone_device_set_policy()` | per-`policy` sysfs write | `ThermalCore::set_zone_policy` |
| `thermal_build_list_of_policies()` | per-`available_policies` sysfs | `ThermalCore::list_policies` |
| `thermal_zone_device_register_with_trips()` | per-zone register | `ThermalCore::register_zone_with_trips` |
| `thermal_tripless_zone_device_register()` | per-tripless zone register | `ThermalCore::register_tripless_zone` |
| `thermal_zone_device_unregister()` | per-zone unregister | `ThermalCore::unregister_zone` |
| `thermal_cooling_device_register()` / `_of_..._register()` / `devm_..._register()` | per-cdev register | `ThermalCore::register_cdev*` |
| `thermal_cooling_device_unregister()` | per-cdev unregister | `ThermalCore::unregister_cdev` |
| `thermal_cooling_device_update()` | per-`max_state` change refresh | `ThermalCore::update_cdev` |
| `thermal_zone_device_update()` / `__thermal_zone_device_update()` | per-poll-cycle entry | `ThermalCore::update_zone` / `update_zone_locked` |
| `thermal_zone_device_enable()` / `_disable()` / `_set_mode()` | per-mode switch | `ThermalCore::set_mode` |
| `thermal_zone_device_check()` | per-workqueue tick | `ThermalCore::zone_check_work` |
| `thermal_zone_device_set_polling()` | per-poll re-arm | `ThermalCore::set_polling` |
| `monitor_thermal_zone()` | per-poll-or-passive decision | `ThermalCore::monitor_zone` |
| `thermal_zone_recheck()` | per-get_temp-fail retry | `ThermalCore::recheck_zone` |
| `thermal_zone_broken_disable()` | per-broken-sensor disable | `ThermalCore::disable_broken_zone` |
| `thermal_zone_handle_trips()` | per-update trip-crossing | `ThermalCore::handle_trips` |
| `thermal_trip_crossed()` | per-trip crossing action | `ThermalCore::trip_crossed` |
| `handle_critical_trips()` | per-critical/hot dispatch | `ThermalCore::handle_critical_trips` |
| `thermal_zone_device_critical()` / `_shutdown()` / `_reboot()` | per-critical action | `ThermalCore::zone_critical*` |
| `thermal_zone_device_halt()` | per-halt trigger | `ThermalCore::zone_halt` |
| `thermal_zone_set_trip_temp()` / `_hyst()` | per-trip mutation | `ThermalCore::set_trip_temp` / `set_trip_hyst` |
| `move_to_trips_high()` / `_reached()` / `_invalid()` | per-trip-list bucket move | `ThermalCore::move_trip_high` / `_reached` / `_invalid` |
| `thermal_bind_cdev_to_trip()` / `_unbind_cdev_from_trip()` | per-bind/unbind | `ThermalCore::bind_cdev_to_trip` / `unbind_cdev_from_trip` |
| `__thermal_zone_cdev_bind()` / `_unbind()` | per-zone bind sweep | `ThermalCore::zone_cdev_bind` / `_unbind` |
| `thermal_instance_add()` / `_delete()` | per-instance list-ops | `ThermalCore::instance_add` / `_delete` |
| `thermal_zone_get_by_id()` / `_zone_by_name()` | per-lookup | `ThermalCore::zone_by_id` / `_by_name` |
| `thermal_zone_get_crit_temp()` | per-`temp_crit` query | `ThermalCore::zone_crit_temp` |
| `thermal_governor_update_tz()` | per-event governor notify | `ThermalCore::notify_governor` |
| `thermal_pm_prepare()` / `_complete()` | per-suspend/resume | `ThermalCore::pm_prepare` / `_complete` |
| `thermal_zone_device_resume()` | per-resume work | `ThermalCore::zone_resume_work` |
| `for_each_thermal_zone()` / `_governor()` / `_cooling_device()` | per-iterator | `ThermalCore::for_each_*` |
| `thermal_wq` | per-workqueue (`WQ_POWER_EFFICIENT`) | shared |
| `thermal_tz_list` / `_cdev_list` / `_governor_list` | per-global list | shared |
| `thermal_list_lock` / `_governor_lock` | per-global mutex | shared |

### compatibility contract

REQ-1: struct thermal_zone_device fields (subset):
- id: per-zone IDA-allocated id (`thermal_zone%d` device name).
- type: per-zone name (`THERMAL_NAME_LENGTH`-bounded).
- temperature / last_temperature: per-latest reading + previous.
- mode: `THERMAL_DEVICE_{ENABLED,DISABLED}`.
- state: per-bitset `TZ_STATE_READY | TZ_STATE_FLAG_INIT | _SUSPENDED | _RESUMING | _EXIT`.
- governor: per-active governor.
- ops: per-driver callbacks (`get_temp`, `set_trips`, `change_mode`, `should_bind`, `get_crit_temp`, `critical`, `hot`).
- tzp: per-zone params (`governor_name`, `no_hwmon`, ...).
- num_trips + trailing flexible array of `thermal_trip_desc`.
- trips_high / trips_reached / trips_invalid: per-bucket sorted lists.
- passive: per-passive-trip mitigation count (>0 ⟹ use `passive_delay_jiffies`).
- polling_delay_jiffies / passive_delay_jiffies / recheck_delay_jiffies.
- poll_queue: per-delayed_work.
- node: per-list-entry in `thermal_tz_list`.
- ida: per-cdev-instance ID allocator.
- lock: per-zone mutex.
- removal / resume: per-completion.
- devdata: per-driver private pointer.

REQ-2: struct thermal_cooling_device fields:
- id: per-cdev IDA-allocated id (`cooling_device%d` device name).
- type: per-cdev name (kstrdup_const'd).
- max_state: per-driver `ops.get_max_state` snapshot (refreshable via `thermal_cooling_device_update`).
- ops: `get_max_state`, `get_cur_state`, `set_cur_state` (all mandatory at register).
- thermal_instances: per-bound-trip list.
- node: per-list-entry in `thermal_cdev_list`.
- lock: per-cdev mutex.
- np: per-OF-node (for `thermal_of_cooling_device_register`).
- devdata: per-driver private pointer.
- updated: per-stats flag.

REQ-3: struct thermal_governor fields:
- name: per-governor identifier (`THERMAL_NAME_LENGTH`).
- bind_to_tz / unbind_from_tz: per-zone life-cycle.
- update_tz: per-event callback (`THERMAL_TZ_BIND_CDEV`, `_UNBIND_CDEV`, `THERMAL_TZ_RESUME`, ...).
- trip_crossed: per-trip-crossing callback (skipped for `THERMAL_TRIP_HOT` / `_CRITICAL`).
- manage: per-zone-update governor decision entrypoint.
- governor_list: per-list-entry in `thermal_governor_list`.

REQ-4: struct thermal_trip fields:
- type: `THERMAL_TRIP_ACTIVE | _PASSIVE | _HOT | _CRITICAL`.
- temperature: per-trip in millidegrees-Celsius (or `THERMAL_TEMP_INVALID`).
- hysteresis: per-trip release-window.
- priv: per-driver opaque.

REQ-5: thermal_zone_device_register_with_trips(type, trips, num_trips, devdata, ops, tzp, passive_delay, polling_delay):
- Validate `type` non-empty and < `THERMAL_NAME_LENGTH`.
- Validate `num_trips ≥ 0` and (if `num_trips > 0`) `trips != NULL`.
- Validate `ops != NULL` and `ops.get_temp != NULL`.
- Validate `polling_delay == 0 ∨ passive_delay ≤ polling_delay`.
- Validate class registered.
- Allocate zone with trailing flexible-array of trip descriptors.
- Allocate id from `thermal_tz_ida`.
- Initialize lists (`node`, `trips_high`, `trips_reached`, `trips_invalid`), `ida` (per-instance), `lock`, completions.
- Copy `ops`; if `ops.critical` NULL: default to `thermal_zone_device_critical`.
- Copy trips into descriptors; initialize each descriptor as invalid (`move_to_trips_invalid`).
- Convert delays to jiffies; `recheck_delay_jiffies = THERMAL_RECHECK_DELAY`.
- State: `TZ_STATE_FLAG_INIT`.
- `dev_set_name(..., "thermal_zone%d", id)`.
- `thermal_zone_device_init` (initialize poll_queue, move valid trips to `trips_high`).
- `thermal_zone_init_governor` (look up by `tzp->governor_name` or `def_governor`).
- `thermal_zone_create_device_groups` (sysfs).
- `device_register`.
- If `!tzp || !tzp->no_hwmon`: `thermal_add_hwmon_sysfs` (bridge to hwmon).
- `thermal_thresholds_init`.
- `thermal_zone_init_complete`: add to `thermal_tz_list`, bind matching cdevs, clear `TZ_STATE_FLAG_INIT`, kick `__thermal_zone_device_update`.
- `thermal_notify_tz_create` (netlink genl).
- `thermal_debug_tz_add`.
- Return zone or `ERR_PTR`.

REQ-6: thermal_zone_device_unregister(tz):
- Idempotent on NULL.
- `thermal_debug_tz_remove`.
- `thermal_zone_exit`: under `thermal_list_lock`, mark `TZ_STATE_FLAG_EXIT`, `list_del_init(&tz->node)`, unbind all cdevs.
- `cancel_delayed_work_sync(&tz->poll_queue)`.
- `thermal_thresholds_exit`, `thermal_remove_hwmon_sysfs`.
- `device_del` + `put_device`.
- `thermal_notify_tz_delete` (netlink).
- `wait_for_completion(&tz->removal)`.
- `ida_free`, free `tzp`, free zone.

REQ-7: thermal_cooling_device_register(type, devdata, ops) / `_of_..._register(np, ...)` / `devm_..._register(dev, np, ...)`:
- Validate `ops != NULL` and `get_max_state`, `get_cur_state`, `set_cur_state` all present.
- Validate class registered.
- Allocate cdev, allocate id from `thermal_cdev_ida`.
- `kstrdup_const(type)`.
- Initialize `lock`, `thermal_instances`, copy `np`, `ops`, `devdata`.
- Sample initial `max_state` via `ops.get_max_state` (failure aborts).
- Sample initial `cur_state` via `ops.get_cur_state` (failure tolerated → `current_state = ULONG_MAX` for debug only).
- `thermal_cooling_device_setup_sysfs`.
- `dev_set_name(..., "cooling_device%d", id)`.
- `device_register`.
- `thermal_cooling_device_init_complete`: under `thermal_list_lock`, add to `thermal_cdev_list`, bind to every zone whose `ops.should_bind` matches.
- Return cdev or `ERR_PTR`.

REQ-8: thermal_cooling_device_unregister(cdev):
- Idempotent on NULL.
- `thermal_debug_cdev_remove`.
- `thermal_cooling_device_exit`: under `thermal_list_lock`, verify still present, `list_del`, unbind from every zone (per-zone iteration of `__thermal_zone_cdev_unbind`).
- `device_unregister` (triggers `thermal_release` → free).

REQ-9: __thermal_zone_device_update(tz, event):
- /* Holds tz->lock (caller guard `thermal_zone`) */.
- If `state != TZ_STATE_READY ∨ mode != THERMAL_DEVICE_ENABLED`: return.
- `ret = __thermal_zone_get_temp(tz, &temp)`.
- If `ret`: `thermal_zone_recheck(tz, ret)` → return.
- Else if `temp ≤ THERMAL_TEMP_INVALID`: jump to `monitor` (continue polling, no-op).
- `recheck_delay_jiffies = THERMAL_RECHECK_DELAY` (reset back-off).
- `last_temperature = temperature; temperature = temp`.
- `trace_thermal_temperature`.
- `thermal_genl_sampling_temp(tz->id, temp)` (netlink).
- `notify_event = event`.
- `thermal_zone_handle_trips(tz, governor, &low, &high)` — update bucket lists, derive interrupt window.
- `thermal_thresholds_handle(tz, &low, &high)` — userspace thresholds.
- `thermal_zone_set_trips(tz, low, high)` — program hardware interrupt window if `ops.set_trips`.
- `governor->manage(tz)` if defined.
- `thermal_debug_update_trip_stats`.
- monitor: `monitor_thermal_zone(tz)` — re-arm poll if needed.

REQ-10: thermal_zone_handle_trips(tz, governor, *low, *high):
- /* Walk trips_reached in reverse: any whose threshold > current temp has been released */
- list_for_each_entry_safe_reverse(td, next, &tz->trips_reached): if `td->threshold > temperature`: `thermal_trip_crossed(tz, td, governor, upward=false)`; stash on `way_down_list`.
- /* Walk trips_high forward: any whose threshold ≤ current temp has been crossed */
- list_for_each_entry_safe(td, next, &tz->trips_high): if `td->threshold ≤ temperature`: `thermal_trip_crossed(tz, td, governor, upward=true)`; `move_to_trips_reached`.
- /* Move way-down trips back to trips_high */
- per-td on `way_down_list`: `move_to_trips_high`.
- Set `*low` = highest reached threshold − 1.
- Set `*high` = lowest pending high threshold.

REQ-11: thermal_trip_crossed(tz, td, governor, upward):
- /* On rising crossing */
- If upward:
  - `THERMAL_TRIP_PASSIVE`: `tz->passive++`.
  - `THERMAL_TRIP_CRITICAL ∨ _HOT`: `handle_critical_trips(tz, &td->trip)`.
  - `thermal_notify_tz_trip_up`, `thermal_debug_tz_trip_up`.
- /* On falling crossing */
- Else:
  - `THERMAL_TRIP_PASSIVE`: `tz->passive--`; `WARN_ON(tz->passive < 0)`.
  - `thermal_notify_tz_trip_down`, `thermal_debug_tz_trip_down`.
- `thermal_governor_trip_crossed(governor, tz, &td->trip, upward)` (skipped for HOT and CRITICAL).

REQ-12: handle_critical_trips(tz, trip):
- `trace_thermal_zone_trip`.
- `THERMAL_TRIP_CRITICAL`: `tz->ops.critical(tz)` (default `thermal_zone_device_critical` → `thermal_zone_device_halt(tz, HWPROT_ACT_DEFAULT)` → `__hw_protection_trigger("Temperature too high", CONFIG_THERMAL_EMERGENCY_POWEROFF_DELAY_MS, action)`).
- Else if `tz->ops.hot`: `tz->ops.hot(tz)`.

REQ-13: thermal_zone_device_critical() / `_shutdown()` / `_reboot()`:
- Pure dispatchers to `thermal_zone_device_halt(tz, HWPROT_ACT_{DEFAULT,SHUTDOWN,REBOOT})`.

REQ-14: thermal_zone_set_trip_temp(tz, trip, temp):
- `WRITE_ONCE(trip->temperature, temp)`.
- `thermal_notify_tz_trip_change`.
- /* State-machine over current bucket */
- If `old_temp == THERMAL_TEMP_INVALID`: `move_to_trips_high(tz, td)`.
- Else if `temp == THERMAL_TEMP_INVALID`:
  - If `tz->temperature ≥ td->threshold`: `thermal_trip_crossed(tz, td, gov, upward=false)`.
  - `move_to_trips_invalid(tz, td)`.
- Else:
  - If `tz->temperature ≥ td->threshold`: `move_to_trips_reached(tz, td)`.
  - Else: `move_to_trips_high(tz, td)`.

REQ-15: thermal_zone_set_trip_hyst(tz, trip, hyst):
- `WRITE_ONCE(trip->hysteresis, hyst)`.
- `thermal_notify_tz_trip_change`.
- If `tz->temperature ≥ td->threshold`: `move_to_trips_reached(tz, td)` (threshold = temperature − hysteresis).

REQ-16: thermal_zone_device_set_mode(tz, mode):
- /* Under `tz->lock` */.
- If `mode == tz->mode`: no-op.
- Call `ops.change_mode(tz, mode)` (if defined).
- Set `tz->mode = mode`.
- `__thermal_zone_device_update(tz, THERMAL_EVENT_UNSPECIFIED)`.
- Send netlink notification (`thermal_notify_tz_enable` / `_disable`).

REQ-17: thermal_zone_recheck(tz, error):
- If `error == -EAGAIN`: re-arm at `THERMAL_RECHECK_DELAY` (do not back off).
- Else:
  - First failure: `dev_info` once.
  - Re-arm at current back-off; double back-off (`recheck_delay_jiffies += max(recheck_delay_jiffies >> 1, 1)`).
  - If back-off > `THERMAL_MAX_RECHECK_DELAY`: `thermal_zone_broken_disable(tz)` (force `THERMAL_DEVICE_DISABLED`; `dev_crit` if any `THERMAL_TRIP_CRITICAL` trip remained valid); restore `recheck_delay_jiffies = THERMAL_RECHECK_DELAY`.

REQ-18: monitor_thermal_zone(tz):
- If `passive > 0 ∧ passive_delay_jiffies`: re-arm at `passive_delay_jiffies` (fast).
- Else if `polling_delay_jiffies`: re-arm at `polling_delay_jiffies` (slow).
- Else: no re-arm (interrupt-driven).
- `thermal_zone_device_set_polling` rounds delays > HZ via `round_jiffies_relative`.

REQ-19: thermal_register_governor(governor) / `_unregister_governor(governor)`:
- /* Under `thermal_governor_lock` */
- Reject duplicate name; append to `thermal_governor_list`.
- If matches `DEFAULT_THERMAL_GOVERNOR` and `!def_governor`: set `def_governor`.
- Under `thermal_list_lock`: for each zone with `governor == NULL` whose `tzp->governor_name` matches: `thermal_set_governor(pos, governor)`.
- Unregister mirrors: clear `def_governor` if it was this one (TODO: upstream does not, but switches all bound zones via `thermal_set_governor(pos, NULL)`).

REQ-20: thermal_set_governor(tz, new_gov):
- Call previous governor's `unbind_from_tz` if present.
- Call new governor's `bind_to_tz` if present; on failure attempt `bind_previous_governor` (re-bind old or set `governor = NULL`).
- Update `tz->governor`.

REQ-21: thermal_zone_device_set_policy(tz, policy):
- Look up governor by `strim(policy)`.
- `thermal_set_governor(tz, gov)`.
- `thermal_notify_tz_gov_change` (netlink).

REQ-22: thermal_bind_cdev_to_trip(tz, td, cdev, cool_spec):
- /* lower / upper defaults */
- If `lower == THERMAL_NO_LIMIT`: `lower = 0`.
- If `upper == THERMAL_NO_LIMIT`: `upper = cdev->max_state`; mark `upper_no_limit = true`.
- Validate `lower ≤ upper ≤ max_state`.
- Allocate `thermal_instance`.
- `instance->target = THERMAL_NO_TARGET`.
- `ida_alloc(&tz->ida)` for instance id.
- Create symbolic link `cdev<id>` in `/sys/class/thermal/thermal_zone<N>/`.
- Create `cdev<id>_trip_point` device file (RO).
- Create `cdev<id>_weight` device file (RW).
- `thermal_instance_add`: append to `td->thermal_instances` (zone-side) and (under `cdev->lock`) `cdev->thermal_instances` (cdev-side); reject duplicate.
- `thermal_governor_update_tz(tz, THERMAL_TZ_BIND_CDEV)`.

REQ-23: thermal_unbind_cdev_from_trip(tz, td, cdev):
- Find matching instance on `td->thermal_instances`.
- `thermal_instance_delete`: remove from both lists.
- `thermal_governor_update_tz(tz, THERMAL_TZ_UNBIND_CDEV)`.
- Remove weight + trip device files, symbolic link, `ida_free`, free instance.

REQ-24: thermal_cooling_device_update(cdev):
- Idempotent on NULL / IS_ERR.
- Under `thermal_list_lock`: verify present.
- Under `cdev->lock`: refresh `max_state` via `ops.get_max_state`.
- `thermal_cooling_device_stats_reinit`.
- For each `thermal_instance`: if `upper > new max_state`: clamp `upper`, then clamp `lower` and `target`. Per-`upper_no_limit` instances stretch up to new max.
- Refresh `cur_state` via `ops.get_cur_state` and `thermal_cooling_device_stats_update`.

REQ-25: thermal_pm_prepare() / `_complete()`:
- Suspend: under `thermal_list_lock` set `thermal_pm_suspended = true`; for each zone `thermal_zone_pm_prepare` — wait for any in-flight `RESUMING` to settle, set `TZ_STATE_FLAG_SUSPENDED`, `cancel_delayed_work(&poll_queue)`. After: `flush_workqueue(thermal_wq)`.
- Resume: under `thermal_list_lock` set `thermal_pm_suspended = false`; for each zone `thermal_zone_pm_complete` — `reinit_completion(&resume)`, set `TZ_STATE_FLAG_RESUMING`, `INIT_DELAYED_WORK(&poll_queue, thermal_zone_device_resume)`, queue immediately. Resume work clears `_SUSPENDED|_RESUMING`, re-runs `thermal_zone_device_init`, notifies governor `THERMAL_TZ_RESUME`, runs update.

REQ-26: Per-trip-list buckets:
- `trips_high`: trips whose threshold = `trip->temperature`, current temperature is below.
- `trips_reached`: trips whose threshold = `trip->temperature − trip->hysteresis`, current temperature is at-or-above.
- `trips_invalid`: trips whose temperature is `THERMAL_TEMP_INVALID`.
- All buckets are sorted ascending by `threshold` (`move_trip_to_sorted_list`).

REQ-27: Class device naming:
- Zones: `/sys/class/thermal/thermal_zone%d` (id from `thermal_tz_ida`).
- Cdevs: `/sys/class/thermal/cooling_device%d` (id from `thermal_cdev_ida`).
- Class release callback `thermal_release` discriminates by name prefix.

REQ-28: thermal_wq:
- Workqueue created once in `thermal_init` with `WQ_POWER_EFFICIENT, 0` (unbound, power-aware).
- Per-zone `poll_queue` is a `delayed_work` on this wq.
- `cancel_delayed_work_sync` required on unregister.

REQ-29: Locking discipline:
- `thermal_governor_lock`: governor list + governor-table walks.
- `thermal_list_lock`: zone + cdev lists; held when iterating to bind/unbind.
- `tz->lock`: per-zone state (acquired via `guard(thermal_zone)(tz)`); needed for `__thermal_zone_device_update`, `set_mode`, bucket mutation, resume work.
- `cdev->lock`: per-cdev state (acquired via `guard(cooling_dev)(cdev)`); needed for instance-list manipulation, `set_cur_state` updates.
- Acquisition order: governor_lock → list_lock → tz->lock → cdev->lock (deepest).

REQ-30: Critical-trip default action:
- `thermal_zone_device_halt(tz, action)` logs `dev_emerg`, calls `__hw_protection_trigger(msg, CONFIG_THERMAL_EMERGENCY_POWEROFF_DELAY_MS, action)`.
- Driver may override via `ops.critical` (and may install `thermal_zone_device_critical_shutdown` or `_reboot` instead).
- `thermal_zone_broken_disable` warns at `dev_crit` if a critical trip was active on a disabled zone.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `zone_register_no_leak_on_failure` | INVARIANT | per-register-with-trips: every error path drops `tz`, `tzp`, IDA, sysfs. |
| `cdev_register_no_leak_on_failure` | INVARIANT | per-`__thermal_cooling_device_register`: failure path drops `cdev`, `type`, IDA. |
| `delayed_work_cancelled_before_free` | INVARIANT | per-`thermal_zone_device_unregister`: `cancel_delayed_work_sync` strictly precedes `kfree(tz)`. |
| `removal_completion_awaited` | INVARIANT | per-`thermal_zone_device_unregister`: `wait_for_completion(&removal)` strictly before `kfree`. |
| `governor_lock_then_zone_lock` | INVARIANT | locking order: `thermal_governor_lock → thermal_list_lock → tz.lock → cdev.lock`. |
| `passive_counter_nonnegative` | INVARIANT | per-`thermal_trip_crossed`: `tz.passive ≥ 0` after every transition. |
| `trip_bucket_sorted` | INVARIANT | per-`move_trip_to_sorted_list`: each bucket sorted ascending by `threshold`. |
| `critical_trip_disabled_zone_warned` | INVARIANT | per-`thermal_zone_broken_disable`: `dev_crit` if any `THERMAL_TRIP_CRITICAL` remained valid. |
| `bind_lower_le_upper_le_max_state` | INVARIANT | per-`thermal_bind_cdev_to_trip`: rejects spec where `lower > upper ∨ upper > max_state`. |
| `cdev_ops_complete` | INVARIANT | per-`__thermal_cooling_device_register`: rejects ops missing any of `get_max_state`, `get_cur_state`, `set_cur_state`. |

### Layer 2: TLA+

`drivers/thermal/thermal-core.tla`:
- Per-zone-register + per-cdev-register + per-poll-cycle + per-trip-crossing + per-governor-switch + per-suspend/resume.
- Properties:
  - `safety_no_double_bind` — per-instance: `(trip, cdev)` pair appears at most once on `td.thermal_instances`.
  - `safety_trip_buckets_partition_trips` — each trip-desc is in exactly one of `trips_high | trips_reached | trips_invalid`.
  - `safety_passive_counter_matches_reached_passive_trips` — `tz.passive == |{td ∈ trips_reached : td.trip.type == PASSIVE}|`.
  - `safety_critical_action_within_emergency_delay` — per-CRITICAL crossing: `__hw_protection_trigger` invoked within `CONFIG_THERMAL_EMERGENCY_POWEROFF_DELAY_MS`.
  - `safety_governor_uniquely_bound` — per-zone: at most one governor at a time; `bind_to_tz` is called before `unbind_from_tz` of the previous on success path.
  - `safety_suspend_excludes_polling` — per-`SUSPENDED` ⟹ `poll_queue` not pending.
  - `liveness_every_poll_eventually_runs` — when `ENABLED ∧ READY ∧ !SUSPENDED`: `__thermal_zone_device_update` runs within ≤ `polling_delay`.
  - `liveness_register_eventually_publishes` — `register_with_trips` returns Ok ⟹ `/sys/class/thermal/thermal_zone%d` visible.
  - `liveness_unregister_releases_id` — `unregister` returns ⟹ `thermal_tz_ida` slot freed.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `ThermalCore::register_zone_with_trips` post: returns Ok ⟹ `/sys/class/thermal/thermal_zone%d` directory exists and zone on `thermal_tz_list` | `ThermalCore::register_zone_with_trips` |
| `ThermalCore::unregister_zone` post: zone not on `thermal_tz_list`; `id` returned to ida; `poll_queue` quiescent | `ThermalCore::unregister_zone` |
| `ThermalCore::register_cdev` post: cdev on `thermal_cdev_list`; bound to every zone whose `should_bind` matched | `ThermalCore::register_cdev` |
| `ThermalCore::handle_trips` post: `low < temperature ≤ high` (or `low = -INT_MAX` / `high = INT_MAX` at edges) | `ThermalCore::handle_trips` |
| `ThermalCore::trip_crossed` post: governor `trip_crossed` invoked iff `trip.type ∉ {HOT, CRITICAL}` | `ThermalCore::trip_crossed` |
| `ThermalCore::handle_critical_trips` post: `ops.critical` invoked on `CRITICAL`; `ops.hot` (if present) on `HOT` | `ThermalCore::handle_critical_trips` |
| `ThermalCore::set_governor` post: `tz.governor == new_gov` on success ∨ `tz.governor` restored on bind-failure | `ThermalCore::set_governor` |
| `ThermalCore::bind_cdev_to_trip` post: instance present on both `td.thermal_instances` and `cdev.thermal_instances` (or both absent on failure) | `ThermalCore::bind_cdev_to_trip` |
| `ThermalCore::update_cdev` post: every instance's `upper ≤ cdev.max_state ∧ lower ≤ upper ∧ target ≤ upper (if not THERMAL_NO_TARGET)` | `ThermalCore::update_cdev` |
| `ThermalCore::pm_prepare` post: every zone has `state & SUSPENDED ≠ 0` and `poll_queue` not pending | `ThermalCore::pm_prepare` |

### Layer 4: Verus/Creusot functional

`Per-zone register → bind matching cdevs → first poll-cycle reads temperature → trip classification → governor.manage → re-poll` — semantic equivalence to `Documentation/driver-api/thermal/sysfs-api.rst` lifecycle. `Per-cdev register → bind to matching zones → governor weight/upper/lower honored when setting cur_state → cdev_state stats updated` — semantic equivalence per `Documentation/driver-api/thermal/cpu-cooling-api.rst` and `power_allocator.rst`. `Per-critical trip crossing → tz.ops.critical → __hw_protection_trigger → emergency-poweroff delay → shutdown/reboot/halt` — semantic equivalence per `Documentation/driver-api/thermal/sysfs-api.rst § critical trip points`.

### hardening

(Inherits row-1 features from `drivers/thermal/00-overview.md` § Hardening.)

Thermal-core reinforcement:

- **Per-critical-trip mandatory action** — defense against per-thermal-runaway / silicon-damage; `THERMAL_TRIP_CRITICAL` cannot be a no-op (default `thermal_zone_device_critical` always installed if driver leaves `ops.critical` NULL).
- **Per-broken-sensor disable with exponential back-off** — defense against per-sensor-storm log-flood; `thermal_zone_broken_disable` after `THERMAL_MAX_RECHECK_DELAY`; `dev_crit` if a critical trip was still active.
- **Per-`ops.get_temp` `-EAGAIN` differentiated from hard fault** — defense against per-transient-i2c-error false back-off.
- **Per-zone polling on `WQ_POWER_EFFICIENT`** — defense against per-deep-idle wake-up; rounded delays via `round_jiffies_relative` consolidate wake-ups.
- **Per-`thermal_pm_prepare` flushes workqueue** — defense against per-suspend race; `cancel_delayed_work` then `flush_workqueue(thermal_wq)` so no in-flight poll runs against frozen devices.
- **Per-`cancel_delayed_work_sync` before `kfree(tz)`** — defense against per-UAF on unregister.
- **Per-`bind_to_tz` failure restores previous governor** — defense against per-broken-governor leaving zone unmanaged.
- **Per-cdev ops triplet mandatory at register** — defense against per-missing-`set_cur_state` causing NULL-deref.
- **Per-(trip,cdev) duplicate-bind rejected** — defense against per-double-counting in governor logic.
- **Per-`upper_no_limit` instance auto-stretch on `update_cdev`** — defense against per-stale-upper after firmware updates max cooling state.
- **Per-`thermal_list_lock` held during cdev↔zone bind iteration** — defense against per-mid-iteration register/unregister race.
- **Per-zone IDA + per-cdev IDA disjoint pools** — defense against per-id-collision in sysfs.
- **Per-`policy` sysfs write needs CAP_SYS_ADMIN (parent overview)** — defense against per-unprivileged-disable of throttling.

