---
title: "Tier-3: drivers/regulator/core.c — Voltage/current regulator framework"
tags: ["tier-3", "drivers", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

The regulator framework mediates between **regulator producer drivers** (PMICs, LDOs, switching converters) and **regulator consumer drivers** (any device requiring power). Three coupled abstractions:

- **Driver side** — `struct regulator_desc` is a static template (`name`, `id`, `n_voltages`, `ops`, `type`, `owner`, `fixed_uV`, `linear_min_sel`, `vsel_reg`, `vsel_mask`, `enable_reg`, `enable_mask`, ...). The driver fills it plus `struct regulator_config` (parent `struct device`, regmap, init_data, of_node, optional ena_gpiod) and calls `regulator_register(dev, desc, cfg)`. This allocates a `struct regulator_dev` (kobject under `regulator_class`, owner module-ref'd, mutex `ww_mutex`, machine constraints, supply list, consumer list, debugfs, coupling), parses `init_data` (DT or platform), sets machine constraints (`min_uV` / `max_uV` / `min_uA` / `max_uA` / `valid_ops_mask` / `valid_modes_mask` / `always_on` / `boot_on` / `apply_uV` / `system_load`), resolves the supply chain (its own "supply" regulator if any), and exposes sysfs (`/sys/class/regulator/regulator.N/`).
- **Consumer side** — `struct regulator` is a handle from `regulator_get(dev, "id")` (or `_get_exclusive`, `_get_optional`, or bulk `regulator_bulk_get`). Each call to `regulator_enable(reg)` is refcounted against the producer (`rdev->use_count`); the supply is enabled the first time, disabled the last time. `regulator_set_voltage(reg, min_uV, max_uV)` records the consumer voltage range, intersects with machine constraints and all sibling consumers, and reprograms the hardware via `rdev->desc->ops->set_voltage_sel` or `set_voltage`.
- **Userspace side** — read-only sysfs (`microvolts`, `microamps`, `state`, `status`, `opmode`, `bypass`, `min_microvolts`, `max_microvolts`, `requested_microamps`, `num_users`, `type`, plus `under_voltage` / `over_current` / `over_temp` / `fail` event attributes) and the optional `regulator_summary` debugfs node.

`struct regulator_ops` is the producer's vtable: `enable`, `disable`, `is_enabled`, `set_voltage`, `set_voltage_sel`, `get_voltage`, `get_voltage_sel`, `list_voltage`, `map_voltage`, `set_current_limit`, `get_current_limit`, `set_mode`, `get_mode`, `set_load`, `set_bypass`, `get_bypass`, `set_active_discharge`, `set_suspend_*`, `get_error_flags`. Each op may be NULL — `regulator_ops_is_valid` cross-checks `valid_ops_mask` (per `REGULATOR_CHANGE_*`).

This Tier-3 covers `drivers/regulator/core.c` (~6890 lines).

### Acceptance Criteria

- [ ] AC-1: `regulator_register(dev, desc, cfg)` with valid args returns a non-ERR rdev; `/sys/class/regulator/regulator.N` appears with attributes per `regulator_attr_is_visible`.
- [ ] AC-2: `regulator_register` with `desc->ops == NULL` or `desc->type` outside {VOLTAGE,CURRENT} returns ERR_PTR(-EINVAL).
- [ ] AC-3: `regulator_get(dev, "vdd")` resolves a matching `regulator-name` DT child or `consumer_supply` map; returns a `struct regulator` with `rdev->open_count` incremented; subsequent `regulator_put` decrements and balances `module_put`.
- [ ] AC-4: `regulator_get_optional(dev, "vbus")` with no matching producer returns ERR_PTR(-ENODEV) (and is converted to NULL by the consumer wrapper); `regulator_get` returns -EPROBE_DEFER until producer registers.
- [ ] AC-5: `regulator_get_exclusive` on a producer with `open_count > 0` returns -EBUSY.
- [ ] AC-6: Refcount: `regulator_enable` N times then `regulator_disable` N times brings `regulator->enable_count` and `rdev->use_count` back to 0 and triggers a single hardware-disable on the last call (subject to `always_on`).
- [ ] AC-7: `regulator_set_voltage(reg, 1_800_000, 1_900_000)` clamps to `constraints->min_uV / max_uV`; if clamp empties the range, returns -EINVAL.
- [ ] AC-8: `regulator_set_voltage` honors the union of all consumer ranges via `regulator_check_consumers`.
- [ ] AC-9: `regulator_set_current_limit` returns -EPERM when `valid_ops_mask & REGULATOR_CHANGE_CURRENT == 0`.
- [ ] AC-10: Supply chain: enabling a regulator whose `rdev->supply` is set triggers `regulator_enable(rdev->supply)` before `desc->ops->enable(rdev)`.
- [ ] AC-11: `regulator_bulk_get` of N supplies followed by `regulator_bulk_enable` enables all in parallel via async; failure of any unwinds.
- [ ] AC-12: `regulator_bulk_disable` failure mid-array re-enables already-disabled entries to keep the system consistent.
- [ ] AC-13: `regulator_unregister` while `open_count > 0` WARNs but proceeds; consumers retain stale handles only because they prevented unregister via `module_get` in normal flow (driver bug path).
- [ ] AC-14: `regulator_register_notifier` followed by an `_notifier_call_chain(REGULATOR_EVENT_VOLTAGE_CHANGE, &new_uV)` delivers to the consumer's callback.
- [ ] AC-15: `regulator_set_voltage` on a coupled set adjusts all coupled rdevs to maintain `coupling_desc->max_spread` invariant.
- [ ] AC-16: `regulator_disable_deferred(reg, ms)` queues a delayed work; immediate `regulator_enable` cancels the deferred disable.

### Architecture

```
struct RegulatorDev {
  desc:             *const RegulatorDesc,
  owner:            *Module,
  regmap:           Option<*Regmap>,
  dev:              Device,                              // class regulator_class
  bdev:             Device,                              // bus regulator_bus (lazy)
  mutex:            WwMutex<RegulatorWwClass>,
  constraints:      Option<Box<RegulationConstraints>>,
  supply:           Option<NonNull<Regulator>>,          // upstream consumer handle
  supply_name:      Option<KStr>,
  consumer_list:    ListHead,                            // struct regulator
  notifier:         BlockingNotifierHead,
  use_count:        u32,                                 // # enabled consumers
  open_count:       u32,                                 // # outstanding handles
  exclusive:        u32,                                 // 0 or 1
  ena_pin:          Option<*RegulatorEnableGpio>,
  ena_gpio_state:   u32,
  ena_pin_inverted: u32,
  enable_time_us:   u32,
  last_off:         KtimeT,
  is_switch:        bool,
  reg_data:         *mut (),
  cached_err:       u32,                                 // REGULATOR_ERROR_*
  err_lock:         Spinlock,
  disable_work:     DelayedWork,
  coupling_desc:    CouplingDesc,
  supply_fwd_nb:    NotifierBlock,
  constraints_pending: bool,
  list:             ListHead,                            // global rdev list link
}

struct RegulatorDesc {
  name:                    &'static str,
  supply_name:             Option<&'static str>,
  regulators_node:         Option<&'static str>,
  of_match:                Option<&'static str>,
  of_parse_cb:             Option<fn(*Device, &RegulatorDesc, &mut RegulatorConfig) -> i32>,
  id:                      i32,
  continuous_voltage_range: bool,
  n_voltages:              u32,
  n_current_limits:        u32,
  ops:                     *const RegulatorOps,
  type:                    RegulatorType,                // VOLTAGE | CURRENT
  owner:                   *Module,
  min_uV:                  i32,
  uV_step:                 u32,
  linear_min_sel:          u32,
  fixed_uV:                i32,
  ramp_delay:              u32,
  enable_time:             u32,
  off_on_delay:            u32,
  poll_enabled_time:       u32,
  vsel_range_reg:          u32,    vsel_range_mask:  u32,
  vsel_reg:                u32,    vsel_mask:        u32,
  enable_reg:              u32,    enable_mask:      u32,
  enable_val:              u32,    disable_val:      u32,
  enable_is_inverted:      bool,
  bypass_reg:              u32,    bypass_mask:      u32,
  bypass_val_on:           u32,    bypass_val_off:   u32,
  csel_reg:                u32,    csel_mask:        u32,
  apply_reg:               u32,    apply_bit:        u32,
  active_discharge_reg:    u32,    active_discharge_mask: u32,
  active_discharge_on:     u32,    active_discharge_off:  u32,
  pull_down_reg:           u32,    pull_down_mask:        u32,
  pull_down_val_on:        u32,
  n_linear_ranges:         u32,
  linear_ranges:           *const LinearRange,
  volt_table:              *const u32,
  init_cb:                 Option<fn(*RegulatorDev, *RegulatorConfig) -> i32>,
}

struct RegulatorOps {
  list_voltage:        Option<fn(*RegulatorDev, u32) -> i32>,
  set_voltage:         Option<fn(*RegulatorDev, i32, i32, *mut u32) -> i32>,
  set_voltage_sel:     Option<fn(*RegulatorDev, u32) -> i32>,
  map_voltage:         Option<fn(*RegulatorDev, i32, i32) -> i32>,
  set_voltage_time:    Option<fn(*RegulatorDev, i32, i32) -> i32>,
  set_voltage_time_sel:Option<fn(*RegulatorDev, u32, u32) -> i32>,
  get_voltage:         Option<fn(*RegulatorDev) -> i32>,
  get_voltage_sel:     Option<fn(*RegulatorDev) -> i32>,
  set_current_limit:   Option<fn(*RegulatorDev, i32, i32) -> i32>,
  get_current_limit:   Option<fn(*RegulatorDev) -> i32>,
  enable:              Option<fn(*RegulatorDev) -> i32>,
  disable:             Option<fn(*RegulatorDev) -> i32>,
  is_enabled:          Option<fn(*RegulatorDev) -> i32>,
  set_mode:            Option<fn(*RegulatorDev, u32) -> i32>,
  get_mode:            Option<fn(*RegulatorDev) -> u32>,
  set_load:            Option<fn(*RegulatorDev, i32) -> i32>,
  get_status:          Option<fn(*RegulatorDev) -> i32>,
  set_bypass:          Option<fn(*RegulatorDev, bool) -> i32>,
  get_bypass:          Option<fn(*RegulatorDev, *mut bool) -> i32>,
  set_active_discharge:Option<fn(*RegulatorDev, bool) -> i32>,
  set_pull_down:       Option<fn(*RegulatorDev) -> i32>,
  set_over_voltage_protection: Option<fn(*RegulatorDev, i32, i32, bool) -> i32>,
  set_under_voltage_protection: Option<fn(*RegulatorDev, i32, i32, bool) -> i32>,
  set_over_current_protection: Option<fn(*RegulatorDev, i32, i32, bool) -> i32>,
  set_thermal_protection: Option<fn(*RegulatorDev, i32, i32, bool) -> i32>,
  set_soft_start:      Option<fn(*RegulatorDev) -> i32>,
  set_ramp_delay:      Option<fn(*RegulatorDev, i32) -> i32>,
  set_suspend_voltage: Option<fn(*RegulatorDev, i32) -> i32>,
  set_suspend_enable:  Option<fn(*RegulatorDev) -> i32>,
  set_suspend_disable: Option<fn(*RegulatorDev) -> i32>,
  set_suspend_mode:    Option<fn(*RegulatorDev, u32) -> i32>,
  resume:              Option<fn(*RegulatorDev) -> i32>,
  get_error_flags:     Option<fn(*RegulatorDev, *mut u32) -> i32>,
}

struct Regulator {                                          // consumer handle
  rdev:           *RegulatorDev,
  list:           ListHead,                                // rdev->consumer_list link
  enable_count:   u32,
  always_on:      bool,                                    // overrides constraints
  voltage:        [RegulatorVoltage; PM_SUSPEND_MAX + 1],
  uA_load:        i32,
  enable_time_us: u32,
  dev:            Option<*Device>,
  supply_name:    Option<KStr>,                            // user-provided id
  device_link:    Option<*DeviceLink>,
  debugfs:        Option<*Dentry>,
}
struct RegulatorVoltage { min_uV: i32, max_uV: i32 }

struct RegulationConstraints {
  name:                 Option<&'static str>,
  min_uV:               i32, max_uV: i32,
  uV_offset:            i32,
  min_uA:               i32, max_uA: i32,
  ilim_uA:              i32,
  system_load:          i32,
  valid_modes_mask:     u32,
  valid_ops_mask:       u32,
  input_uV:             i32,
  state_mem:            RegulatorState,
  state_disk:           RegulatorState,
  state_standby:        RegulatorState,
  initial_state:        SuspendStateT,
  initial_mode:         u32,
  ramp_delay:           u32,
  settling_time_up_us:  u32,
  settling_time_down_us:u32,
  enable_time:          u32,
  over_current_protection: bool,
  over_voltage_detection:  bool,
  under_voltage_detection: bool,
  over_temp_detection:     bool,
  active_discharge:        bool,
  always_on:               bool,
  boot_on:                 bool,
  apply_uV:                bool,
  ramp_disable:            bool,
  soft_start:              bool,
  pull_down:               bool,
}
```

`Regulator::register(dev, desc, cfg) -> Result<*RegulatorDev, errno>`:
1. Validate cfg / desc / desc->name / desc->ops / desc->type ∈ {VOLTAGE, CURRENT}.
2. Validate ops shape (no both get_voltage & get_voltage_sel; etc.).
3. Allocate `rdev`; initialize embedded `device`, ww_mutex, lists, notifier, delayed_work.
4. `init_data = regulator_of_get_init_data(dev, desc, cfg, &of_node)` — DT may override.
5. Resolve regmap (cfg → dev → parent).
6. Allocate `rdev->constraints` (kmemdup of init_data->constraints or zalloc).
7. Optional `desc->init_cb(rdev, cfg)`.
8. If `cfg->ena_gpiod`: `regulator_ena_gpio_request(rdev, cfg)` (joining `regulator_ena_gpio_list` for shared gating).
9. `set_machine_constraints(rdev, false)` — clamp + apply initial voltage/current/mode; on EPROBE_DEFER try `regulator_resolve_supply` once, else mark constraints_pending.
10. `regulator_init_coupling(rdev)`.
11. For each consumer_supply: `set_consumer_device_supply(rdev, dev_name, supply)`.
12. `rdev->is_switch = !ops->get_voltage && !ops->list_voltage && !desc->fixed_uV`.
13. `device_add(&rdev->dev)`.
14. Best-effort `regulator_resolve_supply`; if still unresolved & supply_name present, register `rdev->bdev` on `regulator_bus`.
15. `rdev_init_debugfs(rdev)`.
16. Under `regulator_list_mutex`: `regulator_resolve_coupling(rdev)`.
17. Return rdev.

`Regulator::unregister(rdev)`:
1. If `rdev == NULL` return.
2. If `rdev->supply`: unregister supply_fwd_nb; drain `use_count` by repeated `Regulator::disable(rdev->supply)`; `Regulator::put(rdev->supply)`.
3. `flush_work(&rdev->disable_work.work)`.
4. Under `regulator_list_mutex`: WARN on open_count; remove_coupling; unset supplies; list_del; ena_gpio_free; unregister bdev (if registered); unregister dev → `regulator_dev_release` frees constraints / coupling / debugfs / rdev.

`Regulator::get(dev, id, get_type) -> Result<*Regulator, errno>`:
1. Sanity: `_regulator_get_common_check(dev, id, get_type)`.
2. `rdev = regulator_dev_lookup(dev, id)` — per-device map → DT → name → supply alias.
3. `_regulator_get_common(rdev, dev, id, get_type)`:
   - NULL rdev: OPTIONAL_GET ⟹ -ENODEV; NORMAL_GET ⟹ -EPROBE_DEFER / -ENODEV.
   - `try_module_get(rdev->owner)` ⟹ -EPROBE_DEFER.
   - EXCLUSIVE: refuse if open_count > 0.
   - `create_regulator(rdev, dev, id)`:
     - kzalloc Regulator; copy id (kstrdup_const) into supply_name.
     - Set rdev backref; init voltage[].
     - Create device_link + sysfs symlink + debugfs.
     - Under `regulator_lock`: add to consumer_list; ++open_count.
4. Return regulator.

`Regulator::put(regulator)`:
1. Under `regulator_list_mutex`: `_regulator_put(regulator)`:
   - WARN if `enable_count != 0`.
   - `destroy_regulator(regulator)`: remove debugfs / device_link / sysfs link; list_del; --open_count; exclusive=0; kfree supply_name; kfree regulator.
   - `module_put(rdev->owner)`; `put_device(&rdev->dev)`.

`Regulator::enable(regulator) -> i32`:
1. `regulator_lock_dependent(rdev, &ww_ctx)`.
2. `_regulator_enable(regulator)`:
   - If `use_count == 0 && supply`: recurse into `_regulator_enable(supply)`.
   - If coupled: `regulator_balance_voltage(rdev, PM_SUSPEND_ON)`.
   - `_regulator_handle_consumer_enable(regulator)`: ++enable_count; intersect consumer voltage.
   - If `use_count == 0`:
     - `is_enabled = _regulator_is_enabled(rdev)`.
     - If `is_enabled <= 0` AND `valid_ops_mask & REGULATOR_CHANGE_STATUS`:
       - `_regulator_do_enable(rdev)` (GPIO or `ops->enable(rdev)`; honor delays).
       - `_notifier_call_chain(rdev, REGULATOR_EVENT_ENABLE, NULL)`.
     - On is_enabled > 0: skip (already enabled, e.g. boot_on).
   - If `enable_count == 1`: ++use_count.
3. On error: unwind consumer_enable; if `use_count == 0` and supply was enabled, recurse-disable supply.
4. `regulator_unlock_dependent(rdev, &ww_ctx)`.

`Regulator::disable(regulator) -> i32`:
1. `regulator_lock_dependent`.
2. `_regulator_disable(regulator)`:
   - WARN if `enable_count == 0` ⟹ -EIO.
   - If `enable_count == 1`:
     - If `use_count == 1 && !constraints->always_on`:
       - If `valid_ops_mask & REGULATOR_CHANGE_STATUS`:
         - `_notifier_call_chain(PRE_DISABLE)` — STOP_MASK ⟹ -EINVAL.
         - `_regulator_do_disable(rdev)` (GPIO or `ops->disable`); on failure `_notifier_call_chain(ABORT_DISABLE)`.
         - On success `_notifier_call_chain(DISABLE)`.
       - `use_count = 0`.
     - Else: --use_count.
   - `_regulator_handle_consumer_disable`: --enable_count.
   - If coupled: `regulator_balance_voltage`.
   - If `use_count == 0 && supply`: recurse-disable supply.
3. `regulator_unlock_dependent`.

`Regulator::set_voltage(regulator, min_uV, max_uV) -> i32`:
1. `regulator_lock_dependent`.
2. `regulator_set_voltage_unlocked(regulator, min_uV, max_uV, PM_SUSPEND_ON)`:
   - Capture old voltage[PM_SUSPEND_ON].
   - `regulator_check_voltage(rdev, &min, &max)` — -EPERM if !CHANGE_VOLTAGE; clamps to constraints.
   - Update consumer voltage range.
   - `regulator_check_consumers(rdev, &min, &max, state)` intersect siblings.
   - Coupled ⟹ `regulator_balance_voltage(rdev, state)`. Else ⟹ `_regulator_do_set_voltage(rdev, min, max)`:
     - Determine new selector via `regulator_map_voltage` (linear_range / linear / volt_table).
     - `_regulator_call_set_voltage_sel` or `_call_set_voltage` (depending on ops).
     - Compute transition delay (`set_voltage_time` or `set_voltage_time_sel` or `ramp_delay`).
     - usleep_range / udelay for delay.
     - `_notifier_call_chain(REGULATOR_EVENT_VOLTAGE_CHANGE, &new_uV)`.
3. `regulator_unlock_dependent`.

`Regulator::set_current_limit(regulator, min_uA, max_uA) -> i32`:
1. `regulator_lock(rdev)`.
2. `regulator_check_current_limit(rdev, &min, &max)` — -EPERM if !CHANGE_CURRENT; clamp.
3. `desc->ops->set_current_limit(rdev, min, max)`.
4. `regulator_unlock`.

`Regulator::bulk_get(dev, num, consumers[])`:
1. For i in 0..num: `consumers[i].consumer = Regulator::get(dev, consumers[i].supply, NORMAL_GET)`.
2. If `consumers[i].init_load_uA > 0`: `regulator_set_load`.
3. On any error: `regulator_put` already-acquired; return error.

`Regulator::bulk_enable(num, consumers[])`:
1. ASYNC_DOMAIN_EXCLUSIVE(domain).
2. For i: `async_schedule_domain(regulator_bulk_enable_async, &consumers[i], &domain)`.
3. `async_synchronize_full_domain(&domain)`.
4. If any `consumers[i].ret != 0`: disable already-enabled entries and return.

`Regulator::resolve_supply(rdev) -> i32`:
1. Locate supply rdev (DT `<supply_name>-supply` or `regulator_dev_lookup`).
2. `rdev->supply = _regulator_get(rdev->dev.parent, rdev->supply_name, NORMAL_GET)`.
3. `register_regulator_event_forwarding(rdev)` so supply events propagate.
4. If `rdev->use_count > 0`: `regulator_enable(rdev->supply)` to reflect state.

### Out of Scope

- `drivers/regulator/of_regulator.c` Device Tree parsing details (covered separately if expanded).
- Individual regulator driver implementations (e.g. `drivers/regulator/axp20x-regulator.c`, `tps65xxx-*`, `pwm-regulator.c`).
- `drivers/regulator/userspace-consumer.c` userspace consumer test stub (covered separately if expanded).
- `drivers/regulator/dummy.c` dummy regulator (covered separately if expanded).
- `drivers/regulator/devres.c` devm helpers (covered separately if expanded).
- `kernel/power/suspend.c` system suspend orchestration (covered in `kernel/power/` Tier-3 when authored).
- Coupling-implementations beyond the generic coupler (`drivers/regulator/{vqmmc,fan53880,...}-coupler.c`).
- Implementation code.

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct regulator_dev` | per-registered regulator (producer) | `RegulatorDev` |
| `struct regulator_desc` | per-driver static template | `RegulatorDesc` |
| `struct regulator_ops` | per-driver vtable | `RegulatorOps` |
| `struct regulator` | per-consumer handle | `Regulator` |
| `struct regulator_config` | per-register-call config | `RegulatorConfig` |
| `struct regulation_constraints` | per-machine constraints | `RegulationConstraints` |
| `struct regulator_init_data` | per-init constraints + consumer supplies | `RegulatorInitData` |
| `struct regulator_consumer_supply` | per-supply binding (dev_name + id) | `RegulatorConsumerSupply` |
| `struct regulator_voltage` | per-consumer per-state voltage range | `RegulatorVoltage` |
| `struct regulator_supply_alias` | per-device alias mapping | `RegulatorSupplyAlias` |
| `struct regulator_bulk_data` | per-bulk consumer descriptor | `RegulatorBulkData` |
| `struct regulator_map` | per-consumer→producer device-side map | `RegulatorMap` |
| `struct regulator_enable_gpio` | per-ena-gpio shared state | `RegulatorEnableGpio` |
| `regulator_register` | per-driver register | `Regulator::register` |
| `regulator_unregister` | per-driver unregister | `Regulator::unregister` |
| `regulator_get` / `_get_exclusive` / `_get_optional` | per-consumer acquire | `Regulator::get` / `get_exclusive` / `get_optional` |
| `regulator_put` | per-consumer release | `Regulator::put` |
| `regulator_enable` / `_disable` / `_force_disable` | per-refcounted enable/disable | `Regulator::enable` / `disable` |
| `regulator_disable_deferred` | per-deferred-ms disable | `Regulator::disable_deferred` |
| `regulator_is_enabled` | per-state query | `Regulator::is_enabled` |
| `regulator_set_voltage` / `_set_voltage_rdev` | per-set range | `Regulator::set_voltage` |
| `regulator_get_voltage` / `_get_voltage_rdev` | per-read voltage | `Regulator::get_voltage` |
| `regulator_set_current_limit` / `_get_current_limit` | per-current cap | `Regulator::set_current_limit` |
| `regulator_set_mode` / `_get_mode` / `_set_load` | per-operating mode | `Regulator::set_mode` |
| `regulator_set_bypass` / `_allow_bypass` | per-bypass mode | `Regulator::allow_bypass` |
| `regulator_check_voltage` / `_check_consumers` / `_check_current_limit` | per-constraint check | `Regulator::check_voltage` |
| `set_machine_constraints` | per-init constraints | `Regulator::set_constraints` |
| `regulator_resolve_supply` | per-supply chain resolve | `Regulator::resolve_supply` |
| `regulator_register_notifier` / `_unregister_notifier` / `_notifier_call_chain` | per-event chain | `Regulator::notifier` |
| `regulator_bulk_get` / `_enable` / `_disable` / `_force_disable` / `_free` | per-bulk helpers | `Regulator::bulk_*` |
| `regulator_bulk_register_supply_alias` / `_unregister_supply_alias` | per-bulk alias | `Regulator::bulk_supply_alias` |
| `regulator_has_full_constraints` | per-platform-asserts-all-known | `Regulator::has_full_constraints` |
| `regulator_get_error_flags` | per-error mask read | `Regulator::get_error_flags` |
| `regulator_suspend_enable` / `_suspend_disable` / `_set_suspend_voltage` | per-suspend state | `Regulator::suspend` |
| `regulator_coupler_register` / `_init_coupling` / `_balance_voltage` | per-coupling | `Regulator::coupling` |
| `_regulator_do_enable` / `_do_disable` | per-hw enable/disable transitions | `Regulator::do_enable` / `do_disable` |
| `_regulator_do_set_voltage` | per-hw voltage transition + delay | `Regulator::do_set_voltage` |

### compatibility contract

REQ-1 — `struct regulator_dev` (producer side):
- `desc`: const pointer to driver-provided `regulator_desc`.
- `owner`: producer module (refcounted on consumer `get`).
- `regmap`: regmap for register I/O (resolved from `cfg->regmap` or `dev_get_regmap(parent)`).
- `dev`: embedded `struct device` (class `regulator_class`).
- `bdev`: bus-side `struct device` registered under `regulator_bus` (only if supply unresolved).
- `mutex`: `struct ww_mutex` (wait/wound) for the per-rdev lock; used cooperatively across coupled/chained regulators.
- `constraints`: machine-side `struct regulation_constraints` (kmemdup'd from init_data).
- `supply`: parent `struct regulator *` (this rdev's input rail).
- `supply_name`: free-form input-rail name (from `init_data->supply_regulator` or `desc->supply_name`).
- `consumer_list`: head of per-consumer `struct regulator` entries.
- `notifier`: blocking notifier head for `REGULATOR_EVENT_*`.
- `use_count`: number of currently-enabled consumers.
- `open_count`: number of `regulator` handles outstanding.
- `exclusive`: 1 if a `get_exclusive` caller is currently holding the only handle.
- `ena_pin` / `ena_gpio_state` / `ena_pin_inverted`: enable-GPIO state (shared via `regulator_ena_gpio_list`).
- `last_off`: ktime of last hardware-off (for `desc->off_on_delay`).
- `is_switch`: derived: true ⟹ no list_voltage / get_voltage / fixed_uV (pure on/off switch).
- `coupling_desc`: array of coupled rdevs + a coupler implementation.
- `cached_err`: most recent error flags (REGULATOR_ERROR_*).
- `disable_work`: delayed work for `regulator_disable_deferred`.
- `constraints_pending`: flag — supply not yet resolved at register; deferred constraint apply.

REQ-2 — `struct regulator_desc` driver template:
- `name`: regulator name (mandatory).
- `supply_name`: input-rail label.
- `regulators_node`: DT subnode key (commonly `"regulators"`).
- `of_match` / `of_match_full_name` / `of_parse_cb`: DT matching glue.
- `id`: driver-assigned index.
- `continuous_voltage_range`: 1 ⟹ no selector table.
- `n_voltages`: number of voltage selectors.
- `n_current_limits`: number of current-limit selectors.
- `ops`: pointer to `struct regulator_ops` (mandatory).
- `type`: `REGULATOR_VOLTAGE` or `REGULATOR_CURRENT` (mandatory).
- `owner`: THIS_MODULE.
- `min_uV`, `uV_step`, `linear_min_sel`, `fixed_uV`: linear-mapping fields.
- `volt_table`: explicit lookup table (alternative to linear).
- `vsel_range_reg` / `vsel_range_mask` / `vsel_reg` / `vsel_mask`: regmap register layout.
- `enable_reg` / `enable_mask` / `enable_val` / `disable_val`: enable control register.
- `enable_time`, `ramp_delay`, `off_on_delay`: timing knobs.
- `bypass_reg` / `bypass_mask`: bypass control register.
- `csel_reg` / `csel_mask`: current-limit register.
- `apply_reg` / `apply_bit`: apply latch.
- `pull_down_reg` / `pull_down_mask` / `pull_down_val_on`: pull-down on disable.
- `active_discharge_reg` / `active_discharge_mask` / `active_discharge_on` / `active_discharge_off`: active discharge.
- `init_cb`: optional post-init callback invoked under register.
- `n_linear_ranges` / `linear_ranges`: piecewise linear mapping.

REQ-3 — `struct regulator_ops` (vtable; producer fills the subset its hardware can do):
- `list_voltage(rdev, selector) -> uV` — selector → micro-volts.
- `set_voltage(rdev, min_uV, max_uV, *selector)` — direct uV API.
- `set_voltage_sel(rdev, selector)` — selector API.
- `map_voltage(rdev, min_uV, max_uV) -> selector` — uV → selector mapping (default `regulator_map_voltage_linear`).
- `set_voltage_time(rdev, old_uV, new_uV) -> us` — transition delay.
- `set_voltage_time_sel(rdev, old_sel, new_sel) -> us` — selector-form delay.
- `get_voltage(rdev) -> uV` / `get_voltage_sel(rdev) -> selector` (mutually exclusive).
- `set_current_limit(rdev, min_uA, max_uA)` / `get_current_limit(rdev) -> uA`.
- `enable(rdev)` / `disable(rdev)` / `is_enabled(rdev) -> int` (-1 unknown).
- `set_mode(rdev, mode)` / `get_mode(rdev) -> mode`.
- `set_load(rdev, load_uA)` — drives DRMS (dynamic regulator mode switching).
- `get_status(rdev) -> REGULATOR_STATUS_*`.
- `set_bypass(rdev, enable)` / `get_bypass(rdev) -> bool`.
- `set_active_discharge(rdev, enable)`.
- `set_pull_down(rdev)`.
- `set_over_voltage_protection(rdev, lim, severity, enable)` / `set_under_voltage_protection` / `set_over_current_protection` / `set_thermal_protection`.
- `set_soft_start(rdev)`.
- `set_ramp_delay(rdev, ramp_delay)`.
- `set_suspend_voltage(rdev, uV)` / `set_suspend_enable(rdev)` / `set_suspend_disable(rdev)` / `set_suspend_mode(rdev, mode)`.
- `resume(rdev)`.
- `get_error_flags(rdev, *flags) -> int` — produces REGULATOR_ERROR_* bitmap.

REQ-4 — `struct regulation_constraints` (machine constraints):
- `name`: human-readable label.
- `min_uV`, `max_uV`, `min_uA`, `max_uA`, `ilim_uA`: hard ranges.
- `system_load`: always-on baseline load (uA) carved off DRMS.
- `valid_modes_mask`: bitwise OR of `REGULATOR_MODE_{FAST,NORMAL,IDLE,STANDBY}`.
- `valid_ops_mask`: bitwise OR of `REGULATOR_CHANGE_{VOLTAGE,CURRENT,MODE,STATUS,DRMS,BYPASS}`.
- `input_uV`: input voltage hint.
- `state_mem` / `state_disk` / `state_standby`: per-`PM_SUSPEND_*` `regulator_state` (enabled/disabled/mode/uV).
- `initial_state`: `PM_SUSPEND_ON | _MEM | _DISK`.
- `initial_mode`: REGULATOR_MODE_*.
- `ramp_delay`, `enable_time`, `over_current_protection`, `active_discharge`: derived knobs.
- Boolean flags: `always_on`, `boot_on`, `apply_uV`, `ramp_disable`, `soft_start`, `pull_down`, `over_current_detection`, `over_voltage_detection`, `under_voltage_detection`, `over_temp_detection`.

REQ-5 — `regulator_register(dev, desc, cfg) -> *rdev` (5984-6262):
- Reject if `cfg == NULL`, `desc == NULL`, `desc->name == NULL`, `desc->ops == NULL` ⟹ -EINVAL.
- Reject if `desc->type != REGULATOR_VOLTAGE && != REGULATOR_CURRENT` ⟹ -EINVAL.
- WARN_ON if both `get_voltage` and `get_voltage_sel` are set, or both `set_voltage` and `set_voltage_sel`.
- Reject if `get_voltage_sel` or `set_voltage_sel` is set without `list_voltage` ⟹ -EINVAL.
- Allocate `rdev` (kzalloc_obj); `device_initialize(&rdev->dev)`; `rdev->dev.class = &regulator_class`; `spin_lock_init(&rdev->err_lock)`.
- Duplicate `cfg` into a kmemdup'd `config` so DT can mutate it.
- `regulator_of_get_init_data(dev, desc, config, &of_node)` — DT path returns init_data (or -EPROBE_DEFER for unresolved phandles).
- `ww_mutex_init(&rdev->mutex, &regulator_ww_class)`; `rdev->reg_data = config->driver_data`; `rdev->owner = desc->owner`.
- Resolve `rdev->regmap` from `config->regmap || dev_get_regmap(dev) || dev_get_regmap(dev->parent)`.
- `INIT_LIST_HEAD(&rdev->consumer_list / list)`; `BLOCKING_INIT_NOTIFIER_HEAD(&rdev->notifier)`; `INIT_DELAYED_WORK(&rdev->disable_work, regulator_disable_work)`.
- `rdev->supply_name = init_data->supply_regulator ?: desc->supply_name`.
- `dev_set_name(&rdev->dev, "regulator.%lu", atomic_inc_return(&regulator_no))`.
- Allocate `rdev->constraints` (kmemdup of `init_data->constraints` or zalloc).
- Optional `desc->init_cb(rdev, config)`.
- If `config->ena_gpiod`: `regulator_ena_gpio_request(rdev, config)` (shared GPIO via `regulator_ena_gpio_list`).
- `set_machine_constraints(rdev, false)`:
  - On -EPROBE_DEFER: try `regulator_resolve_supply(rdev)` then retry.
  - On other error: goto wash.
  - On still-EPROBE_DEFER: `rdev->constraints_pending = true`.
- `regulator_init_coupling(rdev)`.
- For each `init_data->consumer_supplies[i]`: `set_consumer_device_supply(rdev, dev_name, supply)`.
- `rdev->is_switch = !desc->ops->get_voltage && !desc->ops->list_voltage && !desc->fixed_uV`.
- `device_add(&rdev->dev)` — sysfs entry appears.
- If supply unresolved: `regulator_resolve_supply(rdev)` best-effort.
- If `rdev->supply_name && !rdev->supply`: register bdev under `regulator_bus` so a future producer probe re-triggers resolution.
- `rdev_init_debugfs(rdev)`.
- Under `regulator_list_mutex`: `regulator_resolve_coupling(rdev)`.
- Return rdev. On any failure: unwind through `del_cdev_and_bdev → unset_supplies → wash → clean → rinse` (each label progressively undoes the prior step, including putting any dangling `config->ena_gpiod`).

REQ-6 — `regulator_unregister(rdev)` (6271-6300):
- Return if rdev == NULL.
- If `rdev->supply`: `regulator_unregister_notifier(rdev->supply, &rdev->supply_fwd_nb)`; drain `use_count` by repeated `regulator_disable(rdev->supply)`; `regulator_put(rdev->supply)`.
- `flush_work(&rdev->disable_work.work)`.
- Under `regulator_list_mutex`:
  - WARN_ON if `rdev->open_count > 0`.
  - `regulator_remove_coupling(rdev)`.
  - `unset_regulator_supplies(rdev)` (consumer map list).
  - `list_del(&rdev->list)`.
  - `regulator_ena_gpio_free(rdev)`.
  - If `rdev->bdev.bus == &regulator_bus`: `device_unregister(&rdev->bdev)`.
  - `device_unregister(&rdev->dev)` ⟹ `regulator_dev_release` (kfree constraints / coupling_desc / debugfs / kfree rdev).

REQ-7 — `regulator_get(dev, id)` family (2561-2619):
- `_regulator_get(dev, id, NORMAL_GET | EXCLUSIVE_GET | OPTIONAL_GET)`:
  - `_regulator_get_common_check(dev, id, get_type)` — sanity (dev valid; id non-NULL for NORMAL).
  - `regulator_dev_lookup(dev, id)` (2128) — checks per-device `regulator_map_list`, then DT (`regulator_dt_lookup`), then name (`regulator_lookup_by_name`), then supply alias.
  - `_regulator_get_common(rdev, dev, id, get_type)` (2407):
    - If NULL rdev:
      - OPTIONAL_GET ⟹ return ERR_PTR(-ENODEV) (caller converts to NULL on -ENODEV).
      - NORMAL_GET ⟹ -EPROBE_DEFER if pending; -ENODEV otherwise.
    - `try_module_get(rdev->owner)` ⟹ -EPROBE_DEFER on failure.
    - For EXCLUSIVE_GET: refuse if `rdev->open_count > 0` ⟹ -EBUSY; set `rdev->exclusive = 1`.
    - `create_regulator(rdev, dev, id)` allocates `struct regulator`:
      - `regulator->rdev = rdev`; copy supply_name (kstrdup_const); init voltage[PM_SUSPEND_MAX+1].
      - `regulator->dev = dev`; create device_link (CONSUMER↔SUPPLIER) optionally.
      - `link_and_create_debugfs(...)`; install sysfs symlink at `/sys/class/regulator/regulator.N/<id>`.
      - `list_add(&regulator->list, &rdev->consumer_list)` under `regulator_lock(rdev)`.
      - `rdev->open_count++`.
    - For EXCLUSIVE_GET: if hardware was enabled at register and ops permit, `regulator->enable_count = 1` and `rdev->use_count = 1`.
- `regulator_get` / `_get_exclusive` / `_get_optional` differ only by `enum regulator_get_type` argument.

REQ-8 — `regulator_put(regulator)` (2675-2681):
- Under `regulator_list_mutex`: `_regulator_put(regulator)`:
  - WARN_ON if `regulator->enable_count` != 0.
  - `destroy_regulator(regulator)`:
    - `debugfs_remove_recursive`.
    - `device_link_remove` if linked; `sysfs_remove_link`.
    - Under `regulator_lock`: `list_del(&regulator->list)`; `rdev->open_count--`; `rdev->exclusive = 0`.
    - `kfree_const(supply_name)`; `kfree(regulator)`.
  - `module_put(rdev->owner)`; `put_device(&rdev->dev)`.

REQ-9 — `regulator_enable(regulator)` (3194-3206):
- Under `regulator_lock_dependent(rdev, &ww_ctx)` (wait/wound across coupled tree):
  - `_regulator_enable(regulator)`:
    - If `rdev->use_count == 0 && rdev->supply`: recursively enable the supply.
    - If coupled (`n_coupled > 1`): `regulator_balance_voltage(rdev, PM_SUSPEND_ON)` for the coupled group.
    - `_regulator_handle_consumer_enable(regulator)` — `regulator->enable_count++`; update voltage range; intersect with siblings.
    - If `rdev->use_count == 0`:
      - `_regulator_is_enabled(rdev)` — if 0 / -EINVAL and `valid_ops_mask & REGULATOR_CHANGE_STATUS`:
        - `_regulator_do_enable(rdev)` ⟹ either GPIO toggle (`regulator_ena_gpio_ctrl(rdev, true)`) or `rdev->desc->ops->enable(rdev)`.
        - Honor `desc->off_on_delay` and `desc->enable_time`.
        - `_notifier_call_chain(rdev, REGULATOR_EVENT_ENABLE, NULL)`.
    - If `regulator->enable_count == 1`: `rdev->use_count++`.
- On any error: unwind via `err_consumer_disable` / `err_disable_supply`.
- `regulator_unlock_dependent(rdev, &ww_ctx)`.

REQ-10 — `regulator_disable(regulator)` (3306-3318):
- Mirrors enable under `regulator_lock_dependent`:
  - `_regulator_disable(regulator)`:
    - WARN + -EIO on unbalanced (enable_count == 0).
    - If `regulator->enable_count == 1`:
      - If `rdev->use_count == 1 && !constraints->always_on`:
        - If `valid_ops_mask & REGULATOR_CHANGE_STATUS`:
          - `_notifier_call_chain(rdev, REGULATOR_EVENT_PRE_DISABLE, NULL)` — NOTIFY_STOP_MASK ⟹ -EINVAL.
          - `_regulator_do_disable(rdev)` ⟹ GPIO toggle or `desc->ops->disable(rdev)`.
          - On failure: `_notifier_call_chain(rdev, REGULATOR_EVENT_ABORT_DISABLE)` then return error.
          - On success: `_notifier_call_chain(rdev, REGULATOR_EVENT_DISABLE)`.
        - `rdev->use_count = 0`.
      - Else: `rdev->use_count--`.
    - `_regulator_handle_consumer_disable(regulator)` — `enable_count--`.
    - If coupled: `regulator_balance_voltage(rdev, PM_SUSPEND_ON)`.
    - If `use_count == 0 && rdev->supply`: recursively disable supply.

REQ-11 — `regulator_set_voltage(regulator, min_uV, max_uV)` (4500-4514):
- `regulator_lock_dependent(rdev, &ww_ctx)`.
- `regulator_set_voltage_unlocked(regulator, min_uV, max_uV, PM_SUSPEND_ON)` (4038):
  - Capture old `voltage->min_uV` / `max_uV` per-state.
  - `regulator_check_voltage(rdev, &min, &max)` — clamps to constraints; -EPERM if `!REGULATOR_CHANGE_VOLTAGE`.
  - Update per-consumer `voltage->min_uV` / `max_uV` for the given state.
  - `regulator_check_consumers(rdev, &min, &max, state)` — intersects all consumers; -EINVAL on empty.
  - If coupled: `regulator_balance_voltage(rdev, state)`.
  - Else: `_regulator_do_set_voltage(rdev, min, max)`:
    - If hardware is in supply chain: enforce input >= output + headroom (resolved via supply).
    - `rdev->desc->ops->set_voltage` (or `set_voltage_sel(rdev, regulator_map_voltage(min, max))`).
    - Honor `desc->ramp_delay` + `set_voltage_time` / `set_voltage_time_sel` for transition delay.
    - `_notifier_call_chain(rdev, REGULATOR_EVENT_VOLTAGE_CHANGE, &new_uV)`.

REQ-12 — `regulator_set_current_limit(regulator, min_uA, max_uA)` (4858):
- `regulator_lock(rdev)`.
- `regulator_check_current_limit(rdev, &min, &max)` clamps to `constraints->min_uA / max_uA`; -EPERM if `!REGULATOR_CHANGE_CURRENT`.
- `desc->ops->set_current_limit(rdev, min, max)`.
- `regulator_unlock`.

REQ-13 — Supply chain (`rdev->supply`):
- `regulator_resolve_supply(rdev)` (2171):
  - Look up supply rdev by `rdev->supply_name` via DT (`<supply_name>-supply` property on the producer's of_node) or by `regulator_dev_lookup(dev, supply_name)`.
  - Acquire it as a consumer: `_regulator_get(rdev->dev.parent, rdev->supply_name, NORMAL_GET)` stored in `rdev->supply`.
  - Register forwarding notifier `rdev->supply_fwd_nb` so events on the supply propagate to this rdev's consumers.
  - If `rdev->use_count > 0` after resolution: `regulator_enable(rdev->supply)` to reflect the active state.
- `_regulator_enable` walks the chain upward; `_regulator_disable` walks it downward.
- A producer rdev that exposes the same supply name back to itself (cycle) is rejected via `set_supply` check.

REQ-14 — Constraints application (`set_machine_constraints`, 1446-1697):
- Copy `init_data->constraints` into `rdev->constraints` (kmemdup).
- `machine_constraints_voltage(rdev, constraints)`: clamp `min_uV` / `max_uV` to selector grid; if `apply_uV` set, drive initial voltage to `(min + max) / 2` via `_regulator_do_set_voltage`.
- `machine_constraints_current(rdev, constraints)`: clamp `min_uA` / `max_uA`; apply `set_current_limit`.
- If `desc->ops->set_input_current_limit` and `constraints->ilim_uA`: apply.
- If `constraints->initial_mode` and `desc->ops->set_mode`: apply (must be in `valid_modes_mask`).
- If `constraints->ramp_delay` and `desc->ops->set_ramp_delay`: apply.
- If `constraints->soft_start` / `pull_down` / `over_current_protection` / etc.: invoke the corresponding op.
- For each suspend state: `__suspend_set_state(rdev, state)` programs `set_suspend_voltage` / `set_suspend_enable` / `set_suspend_disable` / `set_suspend_mode`.
- If `constraints->always_on || boot_on`: `_regulator_do_enable(rdev)` (and propagate to supply).
- `print_constraints(rdev)` emits a one-line dev_info summary.
- Per-error: returns -EPROBE_DEFER if supply unresolved at constraint time.

REQ-15 — `regulator_bulk_*` (5311-5521):
- `regulator_bulk_get(dev, num, consumers[])`:
  - For each consumer: `consumers[i].consumer = _regulator_get(dev, consumers[i].supply, NORMAL_GET)`.
  - If `init_load_uA > 0`: `regulator_set_load(consumers[i].consumer, init_load_uA)`.
  - On any error: `regulator_put` all already-acquired entries.
- `regulator_bulk_enable(num, consumers[])`:
  - Schedule `regulator_bulk_enable_async` for each (parallel enable via async_domain).
  - `async_synchronize_full_domain` waits.
  - Any failure ⟹ unwind: disable any successfully-enabled entries; first error is returned.
- `regulator_bulk_disable(num, consumers[])`:
  - Disable in reverse order; on failure, re-enable already-disabled entries (best-effort rollback).
- `regulator_bulk_force_disable(num, consumers[])`:
  - `regulator_force_disable` each; first error captured; never re-enables (used for emergency shutdown).
- `regulator_bulk_free(num, consumers[])`:
  - `regulator_put` each non-NULL consumer.

REQ-16 — sysfs hierarchy (`/sys/class/regulator/regulator.N/`):
- Always-present: `name`, `type`, `num_users`, `microvolts`, `microamps`, `state`, `status`, `opmode`, `min_microvolts`, `max_microvolts`, `min_microamps`, `max_microamps`, `requested_microamps`.
- Conditional via `regulator_attr_is_visible`:
  - `bypass` (if `desc->ops->get_bypass`).
  - `power_budget_milliwatt`, `power_requested_milliwatt` (if power-budget tracked).
  - `suspend_{mem,disk,standby}_microvolts` / `_mode` / `_state` (if suspend state present).
  - Error attributes (read-only one-shot): `under_voltage`, `over_current`, `regulation_out`, `fail`, `over_temp`, `under_voltage_warn`, `over_current_warn`, `over_voltage_warn`, `over_temp_warn`.
- `regulator_class.dev_groups = regulator_dev_groups` (single attr group filtered by `regulator_attr_is_visible`).
- `regulator_dev_release` is the class release function.

REQ-17 — Locking model:
- `regulator_list_mutex` — guards `regulator_map_list`, `regulator_ena_gpio_list`, `regulator_supply_alias_list`, `regulator_coupler_list`, and per-rdev `consumer_list` mutations during put.
- `regulator_nesting_mutex` — guards nested-lock counters for ww_mutex chains.
- `regulator_ww_class` — ww_mutex class for `rdev->mutex`. `regulator_lock_dependent` walks the coupling + supply graph and locks in deadlock-free order (wait/wound).
- `rdev->mutex` (ww_mutex) — guards `rdev` state during ops.
- `rdev->err_lock` (spinlock) — guards `cached_err`.
- `regulator_lock_two` / `_unlock_two` / `_lock_recursive` — coupling + supply-chain locking primitives.

REQ-18 — Coupling (`struct regulator_coupler`):
- Coupled regulators must move voltages together within constraints (e.g. DDR rail + VDD-LOGIC).
- `regulator_coupler_register(coupler)` adds to `regulator_coupler_list`.
- `regulator_init_coupling(rdev)` parses `regulator-coupled-with` DT phandles; attaches `generic_coupler` or a custom one.
- `regulator_balance_voltage(rdev, state)` walks coupled set and calls `coupler->balance_voltage(coupler, rdev, state)` which computes per-rdev target voltage subject to `coupling_desc->max_spread`.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `enable_disable_balanced` | INVARIANT | per-consumer: regulator->enable_count never negative; matched 1:1 by lifetime end. |
| `use_count_matches_enabled` | INVARIANT | per-rdev: use_count == 0 ⟹ hardware off (after _do_disable) and supply disabled. |
| `voltage_within_constraints` | INVARIANT | per-set_voltage: post-condition min_uV ≥ constraints->min_uV ∧ max_uV ≤ constraints->max_uV. |
| `consumer_intersection_nonempty` | INVARIANT | per-set_voltage: intersection of all consumer ranges nonempty (else -EINVAL). |
| `module_get_balanced_with_put` | INVARIANT | per-get/put: module reference taken on get returned on put. |
| `exclusive_implies_open_count_one` | INVARIANT | per-rdev: exclusive ⟹ open_count == 1. |
| `coupling_max_spread_respected` | INVARIANT | per-balance_voltage: coupled rdevs differ by ≤ coupling_desc->max_spread. |
| `always_on_blocks_hw_disable` | INVARIANT | per-disable: constraints->always_on ⟹ _do_disable not invoked. |

### Layer 2: TLA+

`drivers/regulator/core.tla`:
- States per consumer: GOT → ENABLED(N) → DISABLED → PUT.
- States per rdev: REGISTERED → CONSTRAINED → use_count++/-- → UNREGISTERED.
- Properties:
  - `safety_no_unbalanced_disable` — disable_count ≤ enable_count at all times.
  - `safety_supply_enabled_before_consumer` — `_regulator_do_enable(rdev)` happens-after `_regulator_enable(rdev->supply)`.
  - `safety_no_voltage_out_of_constraints` — every `_do_set_voltage` call lies within `constraints->min_uV / max_uV`.
  - `safety_exclusive_excludes` — EXCLUSIVE_GET on rdev with `open_count > 0` returns -EBUSY.
  - `safety_constraints_pending_eventually_applied` — `constraints_pending` becomes false after supply resolves.
  - `liveness_bulk_get_completes` — bulk_get with N consumers either succeeds for all or unwinds for all.
  - `liveness_disable_deferred_fires_or_cancelled` — deferred work either fires after `ms` or is cancelled by a subsequent enable.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Regulator::register` post: rdev->dev added under regulator_class; constraints kmemdup'd; consumer_list empty | `Regulator::register` |
| `Regulator::unregister` post: rdev removed from list; open_count == 0 enforced; ena_gpio freed | `Regulator::unregister` |
| `Regulator::get` post: open_count incremented; module + device refs taken; sysfs symlink installed | `Regulator::get` |
| `Regulator::put` post: open_count decremented; refs balanced; sysfs symlink removed | `Regulator::put` |
| `Regulator::enable` post: enable_count + use_count incremented; supply enabled first; notifier ENABLE fired | `Regulator::enable` |
| `Regulator::disable` post: enable_count + use_count decremented; supply disabled last; notifier DISABLE fired | `Regulator::disable` |
| `Regulator::set_voltage` post: voltage[state] updated; hardware reprogrammed; transition delay observed | `Regulator::set_voltage` |
| `Regulator::set_current_limit` post: ops->set_current_limit called; -EPERM if !CHANGE_CURRENT | `Regulator::set_current_limit` |
| `Regulator::bulk_enable` post: all succeeded ⟹ all enabled; any failed ⟹ all that succeeded are disabled | `Regulator::bulk_enable` |
| `Regulator::resolve_supply` post: rdev->supply non-NULL ⟹ supply_fwd_nb registered | `Regulator::resolve_supply` |

### Layer 4: Verus/Creusot functional

`Per-producer regulator_register(dev, desc, cfg) → constraints applied (always_on/boot_on/initial_mode) → consumer regulator_get(dev, "vdd") → regulator_set_voltage(min, max) clamped by constraints + sibling ranges → regulator_enable supply-chain-up + hardware enable + notifier → regulator_disable refcount-balanced + hardware disable when last user → regulator_unregister flushes work + tears down sysfs/coupling/supplies` semantic equivalence: per-`Documentation/power/regulator/{overview,consumer,machine,regulator}.rst`, `Documentation/devicetree/bindings/regulator/regulator.yaml`, and the `regulator_ops` contract in `include/linux/regulator/driver.h`.

### hardening

(Inherits row-1 features from `drivers/regulator/00-overview.md` § Hardening.)

regulator core reinforcement:

- **Per-`valid_ops_mask` cross-check before every op** — defense against per-driver missing-op vs. unwanted-op race.
- **Per-`constraints->min_uV` / `max_uV` clamp in `regulator_check_voltage`** — defense against per-out-of-spec voltage damaging the device tree.
- **Per-consumer-intersection enforced in `regulator_check_consumers`** — defense against per-sibling-conflict (one consumer forcing voltage out of another's required range).
- **Per-`always_on` blocks `_do_disable`** — defense against per-disable-of-critical-rail (e.g. SoC core supply).
- **Per-`use_count` strict refcount** — defense against per-supply-disabled-while-consumer-active.
- **Per-`exclusive` slot reservation** — defense against per-shared-use of a regulator that requires sole control.
- **Per-`ww_mutex` deadlock-free supply+coupling locking** — defense against per-AB-BA deadlock in coupled chains.
- **Per-EPROBE_DEFER `constraints_pending` until supply resolves** — defense against per-applying-constraints-on-stale-input.
- **Per-`supply_fwd_nb` propagates events** — defense against per-consumer-blind-to-supply-failure (OVP, UVP, OCP).
- **Per-`ena_gpio` shared via `regulator_ena_gpio_list`** — defense against per-double-toggle of a shared enable line by multiple rdevs.
- **Per-`regulator_disable_deferred` cancelled on subsequent enable** — defense against per-spurious-disable racing with re-acquire.
- **Per-`open_count` WARN at unregister** — defense against per-driver-removal with live consumers.
- **Per-error-flag latched in `cached_err` + sysfs error attrs read-once** — defense against per-event-loss between userspace polls.

