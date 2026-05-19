# Tier-3: drivers/power/supply/{power_supply_core,power_supply_sysfs}.c — power_supply class

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/power/00-overview.md
upstream-paths:
  - drivers/power/supply/power_supply_core.c
  - drivers/power/supply/power_supply_sysfs.c
  - drivers/power/supply/power_supply_leds.c
  - drivers/power/supply/power_supply_hwmon.c
  - drivers/power/supply/power_supply.h
  - drivers/power/supply/samsung-sdi-battery.c
  - include/linux/power_supply.h
-->

## Summary

The `power_supply` class is the kernel-side abstraction for batteries, chargers, mains/USB power sources, fuel-gauges, and wireless inductive sinks. Per-device drivers under `drivers/power/supply/` (88PM, AB8500, BQ27xxx, BQ25xxx, AXP20x, CW2015, MAX17xxx, RT9, smb / smb47x, cros_charge-control, …) register a `struct power_supply_desc` plus a callback table; the core publishes a uniform `/sys/class/power_supply/<name>/` surface with O(80) typed properties (STATUS, CHARGE_NOW, VOLTAGE_NOW, CURRENT_NOW, CAPACITY, TEMP, MODEL_NAME, …), routes uevents to udev for graphical battery indicators, dispatches `external_power_changed` notifications to upstream supplies (charger sees its battery transition online), provides an in-kernel API for ACPI / EC / charge-control consumers, and integrates with thermal-zones, LEDs, and hwmon.

This Tier-3 covers `power_supply_core.c` (~1800 lines: class + register + property accessors + battery-info parser + notifier) and `power_supply_sysfs.c` (~670 lines: per-property typed `device_attribute` matrix + uevent assembly).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct power_supply` | runtime per-device descriptor + state | `drivers::power::supply::PowerSupply` |
| `struct power_supply_desc` | per-driver const descriptor (name, type, properties[], get/set, ext-changed, charge-behaviour, …) | `drivers::power::supply::Desc` |
| `struct power_supply_config` | per-instance probe-time config (of_node, supplied_to, num_supplicants, drv_data) | `drivers::power::supply::Config` |
| `struct power_supply_ext` / `power_supply_register_extension` / `_unregister_extension` | per-device extension (extra properties, e.g. charge-thresholds from ACPI) | `drivers::power::supply::Extension` |
| `struct power_supply_battery_info` | DT-parsed battery characterization (energy/charge full-design, OCV table, R_int, technology, …) | `drivers::power::supply::BatteryInfo` |
| `power_supply_register(parent, desc, cfg)` / `devm_power_supply_register` / `power_supply_unregister(psy)` | per-driver lifecycle | `PowerSupply::register` / `_unregister` |
| `power_supply_changed(psy)` | wake notifier worker, emit uevent + LED + thermal update | `PowerSupply::changed` |
| `power_supply_get_property(psy, psp, *val)` / `_set_property` / `_property_is_writeable` / `_get_property_direct` | per-property accessor | `PowerSupply::get_property` / `_set_property` / `_property_is_writeable` |
| `power_supply_uevent(dev, env)` | per-uevent env builder (POWER_SUPPLY_<PROP>=...) | `PowerSupply::uevent` |
| `power_supply_get_by_name(name)` / `_put(psy)` / `_get_by_reference` / `devm_power_supply_get_by_reference` | lookup helpers | `PowerSupply::get_by_name` / `_put` / `_get_by_reference` |
| `power_supply_am_i_supplied(psy)` / `power_supply_is_system_supplied()` / `power_supply_get_property_from_supplier` | supplier-graph query | `PowerSupply::am_i_supplied` / `_is_system_supplied` |
| `power_supply_get_battery_info(psy, *info)` / `_put_battery_info` / `_battery_info_properties` / `_battery_info_get_prop` | DT battery-info parser | `BatteryInfo::get` / `_put` / `_properties` / `_get_prop` |
| `power_supply_reg_notifier(nb)` / `_unreg_notifier(nb)` | global blocking notifier (PSY_EVENT_PROP_CHANGED) | `Notifier::register` / `_unregister` |
| `power_supply_external_power_changed(psy)` | propagate to suppliers (battery → charger) | `PowerSupply::external_power_changed` |
| `power_supply_update_leds(psy)` / `power_supply_add_hwmon_sysfs` | LED + hwmon companions | `Companions::update_leds` / `_add_hwmon_sysfs` |
| `power_supply_attr_groups` | sysfs surface descriptor (typed dev_attrs) | `Sysfs::attr_groups` |

## Compatibility contract

REQ-1: per-driver `power_supply_desc` carries `enum power_supply_property properties[]`, `enum power_supply_type type`, `enum power_supply_usb_type usb_types[]`, `get_property`, `set_property` (optional), `property_is_writeable` (optional), `external_power_changed` (optional), `name`, `no_thermal` flag.

REQ-2: per-property typed `union power_supply_propval` (intval / strval); enumerated properties (STATUS, HEALTH, TECHNOLOGY, …) render via per-enum text-table; integer properties render decimal (μA, μV, μW, μAh, μWh, °0.1°C as per UAPI in `Documentation/ABI/testing/sysfs-class-power`).

REQ-3: per-device sysfs at `/sys/class/power_supply/<name>/` with one attribute per property in `desc->properties[]`; `is_visible` callback hides attributes whose `get_property` returns `-EINVAL` at registration; CHARGE_BEHAVIOUR / CHARGE_TYPES enumerate the per-driver-supplied bitmask.

REQ-4: per-write property requires `desc->property_is_writeable(psy, psp) != 0`, and `desc->set_property` is invoked under the per-`psy` mutex; CAP-required writes (e.g. CHARGE_CONTROL_START_THRESHOLD on a thinkpad/cros laptop) handled by the underlying driver (CAP_SYS_ADMIN) — but the class surface is mode `0644` by default for properties advertised writeable.

REQ-5: per-`power_supply_changed`: bumps `psy->changed` under `psy->changed_lock`; schedules `changed_work`; defers `pm_relax` until all transitions handled; emits `KOBJ_CHANGE` uevent only when `changed == true` at the start of the work execution.

REQ-6: uevent env carries `POWER_SUPPLY_NAME=<name>`, `POWER_SUPPLY_TYPE=<type-text>`, then per-property `POWER_SUPPLY_<PROP>=<val>` for every successfully-read property; total env size clamped (`UEVENT_BUFFER_SIZE`).

REQ-7: supplier graph: per-device `supplied_to[]` (per-supplier names) and `supplied_from[]` (per-supplied-from); `power_supply_for_each_psy` walks the class; `__power_supply_is_supplied_by` symmetric check; on `changed`, every supplied-from supplier sees `external_power_changed` to recompute charging state.

REQ-8: per-`psy` extensions via `power_supply_register_extension(psy, ext, dev, drv_data)`: list-anchored extra `power_supply_ext` with own properties + get/set; used by `cros_charge-control`, `acpi-cpc`, etc. to attach platform-specific properties (charge-threshold) without modifying the base driver.

REQ-9: battery-info: `power_supply_get_battery_info(psy, *info)` parses `simple-battery` DT subnode (energy-full-design-microwatt-hours, charge-full-design-microamp-hours, voltage-min-design-microvolt, ocv-capacity-table-N, resistance-table, technology) and exposes derived properties via `power_supply_battery_info_get_prop`.

REQ-10: per-`psy` hwmon companion (if `desc->no_thermal == false` and `TEMP` / `TEMP_AMBIENT` properties present): `power_supply_add_hwmon_sysfs(psy)` registers a hwmon device exposing temperatures + currents + voltages with the standard hwmon attribute layout.

REQ-11: per-`psy` thermal-zone (if `desc->no_thermal == false` and `POWER_SUPPLY_PROP_TEMP` in properties): `power_supply_register_thermal_zone` registers a `thermal_zone_device` reflecting current temperature for thermal-governor consumption.

REQ-12: deferred registration: if `dev_uevent` fails (notably during boot before udev), `psy->deferred_register_work` retries after `POWER_SUPPLY_DEFERRED_REGISTER_TIME` (10 ms) until success.

## Acceptance Criteria

- [ ] AC-1: `ls /sys/class/power_supply/` on a laptop shows `BAT0`, `AC`, optionally `ucsi-source-psy-USBC*` etc.; `cat /sys/class/power_supply/BAT0/{status,capacity,voltage_now,current_now}` returns plausible values.
- [ ] AC-2: `udevadm monitor --kernel` captures `KOBJ_CHANGE` uevents on charge/discharge transitions; `POWER_SUPPLY_*` env vars present.
- [ ] AC-3: charge-control: `echo 80 > /sys/class/power_supply/BAT0/charge_control_end_threshold` is accepted by drivers that advertise the property writeable; value clamped to range and persisted across read-back.
- [ ] AC-4: supplier graph: pulling mains while battery charging → battery status transitions Charging → Discharging; charger driver sees `external_power_changed`.
- [ ] AC-5: battery-info DT parse: a `simple-battery` node containing OCV table + technology populates `BatteryInfo`; consumer driver (e.g. `cw2015_battery`) uses OCV → capacity translation.
- [ ] AC-6: hwmon companion: `/sys/class/hwmon/hwmonN/temp1_input` reflects battery temperature for drivers exposing TEMP; same value via class sysfs.
- [ ] AC-7: extension registration: cros_charge-control adds custom thresholds; sysfs surface gains new entries without restarting the underlying psy.
- [ ] AC-8: deferred-register-work: artificially fail `dev_uevent`; psy register retries until uevent succeeds (visible via tracepoint).
- [ ] AC-9: KASAN/UBSAN clean across `cat` + `udevadm`-monitor loop while toggling mains 1000×.

## Architecture

`PowerSupply` lives in `drivers::power::supply::PowerSupply`:

```
struct PowerSupply {
  desc: &'static Desc,
  dev: Device,                       // class = power_supply_class
  changed_work: WorkStruct,
  deferred_register_work: DelayedWork,
  supplied_to: Option<KBox<[KString]>>,
  num_supplicants: u32,
  supplied_from: Option<KBox<[KString]>>,
  num_supplies: u32,
  of_node: Option<DeviceNode>,
  drv_data: NonNull<u8>,             // per-driver private
  changed_lock: SpinLock,
  changed: bool,
  initialized: bool,
  removing: bool,
  update_groups: bool,
  battery_info: Option<KBox<BatteryInfo>>,
  thermal_zone: Option<Arc<ThermalZoneDevice>>,
  hwmon_dev: Option<Arc<HwmonDevice>>,
  led: Option<Arc<LedTrigger>>,
  ws: WakeupSource,
  extensions_sem: RwSemaphore,
  extensions: LinkedList<Extension>,
}

struct Desc {
  name: &'static str,
  type_: PsyType,
  usb_types: &'static [PsyUsbType],
  properties: &'static [PsyProperty],
  get_property: fn(&PowerSupply, PsyProperty, &mut PropVal) -> i32,
  set_property: Option<fn(&PowerSupply, PsyProperty, &PropVal) -> i32>,
  property_is_writeable: Option<fn(&PowerSupply, PsyProperty) -> i32>,
  external_power_changed: Option<fn(&PowerSupply)>,
  no_thermal: bool,
  charge_behaviours: u32,
  // ...
}
```

Registration (`power_supply_register`):
1. Allocate + init `power_supply`; assign `desc`, `cfg`, name.
2. `device_initialize`; assign class + parent + drvdata.
3. Compute attribute-group visibility — call `desc->get_property` for every advertised property; ones returning `-EINVAL` set `attrs_failed` bit so `is_visible` hides them.
4. `device_add` — triggers `power_supply_uevent`; if it fails, schedule `deferred_register_work` for retry.
5. Initialize wake-source + changed-work + LED trigger + (optional) thermal-zone + hwmon companion + battery-info parse if DT subnode present.
6. Walk class once → notify suppliers of new supplied (`power_supply_external_power_changed`).

Property read (`power_supply_get_property`):
1. Verify `psy->initialized && !psy->removing`.
2. Walk per-`psy` extensions (under `extensions_sem` read); if any extension owns the property, dispatch to its `get_property`.
3. Else dispatch to `desc->get_property`.
4. Return `-ENODATA` if property advertised but produces no meaningful value at this moment.

Sysfs write (`power_supply_store_property` → `_set_property`):
1. Parse text per property type (int / enum-text).
2. Acquire `psy->extensions_sem` write; dispatch to owning extension or `desc->set_property`.
3. On success → `power_supply_changed(psy)`.

Changed work (`power_supply_changed_work`):
1. Under `changed_lock`: if `update_groups`, `sysfs_update_groups` (extensions added attributes).
2. If `changed`, drop lock, walk all `psy` in class, for each that is `supplied_by(psy)`, invoke its `external_power_changed`.
3. `power_supply_update_leds(psy)`; `blocking_notifier_call_chain(...)`; `kobject_uevent(KOBJ_CHANGE)`.
4. Re-take lock; if not `changed` again, `pm_relax(&psy->dev)`.

Uevent (`power_supply_uevent`):
1. Add `POWER_SUPPLY_NAME` + `POWER_SUPPLY_TYPE`.
2. For each property in `desc->properties[]`: get value, render via `power_supply_show_property` text-table or decimal; add as `POWER_SUPPLY_<NAME>=<val>`.

## Hardening

(Inherits from `drivers/power/00-overview.md`.)

power_supply-core specific:

- **per-property -EINVAL at register hides the attribute** — drivers cannot accidentally expose a property whose backend isn't actually wired; defeats sysfs read returning random kernel memory or a wedged register.
- **`removing` flag + initialized flag** — accessors short-circuit returning `-ENODEV` between `_unregister` start and final put; defeats UAF on concurrent `cat /sys/.../voltage_now`.
- **`changed_lock` spinlock around `changed` + `update_groups`** — coalesces bursts; defeats kobject_uevent storm under noisy chargers.
- **deferred-register retry bounded** — `_DEFERRED_REGISTER_TIME * MAX_TRIES` cap; never retries forever, returns `-EAGAIN` to driver.
- **`pm_stay_awake` during changed-work, `pm_relax` on completion** — ensures system doesn't suspend mid-transition (would otherwise drop udev events).
- **extensions list under rw_semaphore** — concurrent get/set safe against extension add/remove.
- **battery-info OCV table sorted check** — `power_supply_batinfo_ocv2cap` asserts monotonicity; misordered DT entries rejected at parse.
- **`samsung-sdi-battery`-style static battery DBs are RO** — built-in tables are `const` and live in `__ro_after_init`.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `power_supply`, `power_supply_battery_info`, `power_supply_ext`, `supplied_to[]`/`supplied_from[]` string arrays, and `power_supply_attrs` is `__ro_after_init`.
- **PAX_KERNEXEC** — `power_supply_class`, `power_supply_dev_type`, `power_supply_attrs[]`, and per-property text tables (`POWER_SUPPLY_TYPE_TEXT`, `_STATUS_TEXT`, `_HEALTH_TEXT`, …) in W^X kernel image, `__ro_after_init`.
- **PAX_RANDKSTACK** — randomize kstack across `power_supply_changed`, sysfs show/store callbacks, `power_supply_uevent`, and per-extension dispatch.
- **PAX_REFCOUNT** — saturating `refcount_t` on the underlying `device::kobj.kref` and on per-extension references; overflow trap defeats unregister-races on hot-unplugged chargers.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `power_supply`, `power_supply_battery_info` (carries vendor-specific OCV tables that may be vendor IP), and supplier-name string arrays.
- **PAX_UDEREF** — SMAP/PAN enforced on every sysfs `store` callback; user pointer never deref'd outside `kstrtoint`/`sysfs_match_string` helpers.
- **PAX_RAP / kCFI** — `power_supply_desc::get_property` / `set_property` / `property_is_writeable` / `external_power_changed`, plus per-extension `power_supply_ext` ops, marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms + per-driver fuel-gauge model coefficient disclosure behind CAP_SYSLOG; suppress `%p` in any debug tracepoints.
- **GRKERNSEC_DMESG** — restrict charger fault, OVP/OCP/thermal-trip banners to CAP_SYSLOG; battery health profile inferable from dmesg becomes a fingerprinting channel.
- **sysfs write CAP_SYS_ADMIN** — charge-control attributes (CHARGE_CONTROL_LIMIT, START_THRESHOLD, END_THRESHOLD, CHARGE_BEHAVIOUR, INHIBIT_*) gated CAP_SYS_ADMIN at the class layer; per-driver `set_property` MUST re-check capability for back-stop.
- **Charge-control range clamp** — per-driver `set_property` clamps start/end thresholds to `[0, 100]` and start < end; refuse out-of-range writes with `-ERANGE` rather than masking.
- **`set_property` kCFI** — per-driver write callbacks (potentially toggling charger IC registers) are kCFI-typed; cannot be diverted to arbitrary kernel functions via vtable confusion.
- **Extension list immutable to user** — `power_supply_register_extension` is in-kernel only; userspace cannot inject extensions.
- **Uevent env clamped** — per-property render bounded; total env clamped to `UEVENT_BUFFER_SIZE` defeats per-property pathological-length DoS on udev.
- **Battery-info DT parse validates lengths** — OCV / R_int / temperature tables clamped per `simple-battery` binding; over-long arrays rejected at parse rather than overrunning fixed-size members.

Rationale: power_supply-core is read by every desktop battery indicator and every userspace power-policy daemon, and write-driven by laptop charge-threshold tools and Chrome OS power policy. A misvalidated sysfs write can wedge a charger IC into a thermally unsafe state (over-current, over-voltage, deep-discharge); a leaked battery cycle history is a privacy signal. RAP/kCFI on `set_property`, CAP_SYS_ADMIN on charge-control, range clamping with `-ERANGE` semantics, refcount-overflow trapping, and battery-info parse hardening turn the class from "USB-ID for batteries" into a structurally bounded privileged surface.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-device fuel-gauge / charger drivers (covered in their own Tier-3 docs as needed: `bq27xxx.md`, `max17042.md`, `axp20x.md`, `cw2015.md`, `cros_charge-control.md`, …)
- Power-supply LED triggers (`power_supply_leds.c` — future Tier-3 if needed)
- Power-supply hwmon companion (`power_supply_hwmon.c` — future Tier-3)
- DT bindings for `simple-battery` (covered in `Documentation/devicetree/bindings/power/supply/`)
- UPS / APC / NUT (userspace)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `supplied_arrays_no_oob` | OOB | `supplied_to[]` and `supplied_from[]` indexed strictly < `num_supplicants` / `num_supplies` |
| `extension_list_locked` | RACE | every `get/set_property` enters with `extensions_sem` read held; every register/unregister with write |
| `changed_work_idempotent` | RACE | reentrant `power_supply_changed` collapses to single `KOBJ_CHANGE` per work cycle |
| `removing_blocks_access` | UAF | `removing == true` short-circuits all accessors before psy memory freed |

### Layer 2: TLA+

`models/power/changed_notif.tla`: proves the changed/initialized/removing/update_groups state machine cannot livelock and reaches `pm_relax` for every wakeup raised by `power_supply_changed`.

### Layer 3: Verus invariants

- `power_supply_register` post: on success, the new psy is visible to `_get_by_name(name)` and emits exactly one `KOBJ_ADD` uevent.
- `power_supply_unregister` post: removes from class, drains `changed_work`, runs `_unregister_extension` for all extensions, and frees only after last ref drop.

### Layer 4: Functional

Sysfs-soak: 1000 mains-toggle cycles + `udevadm monitor` capture; charge-control write-stress with KASAN + UBSAN; battery-info DT parse fuzz with malformed `simple-battery` nodes.
