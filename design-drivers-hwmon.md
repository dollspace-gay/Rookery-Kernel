---
title: "Tier-3: drivers/hwmon/hwmon.c — hwmon class + per-channel sysfs generation"
tags: ["tier-3", "drivers", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

The **hwmon core** is the framework that publishes every chip-level temperature/voltage/fan/PWM/current/power/energy/humidity/intrusion sensor under `/sys/class/hwmon/hwmon<N>/`, with the per-channel attribute names that `lm-sensors`/`sensors` and downstream tools depend on. Every modern driver registers via `hwmon_device_register_with_info(parent, name, drvdata, chip, extra_groups)` passing a `struct hwmon_chip_info` whose `info[]` is a NULL-terminated array of `struct hwmon_channel_info`, each one of type `hwmon_chip | hwmon_temp | hwmon_in | hwmon_curr | hwmon_power | hwmon_energy | hwmon_energy64 | hwmon_humidity | hwmon_fan | hwmon_pwm | hwmon_intrusion` with a `config[]` bitmask-per-channel (e.g., `HWMON_T_INPUT | HWMON_T_MAX | HWMON_T_CRIT`). The core mechanically walks `info[]`, calls `chip->ops->is_visible(drvdata, type, attr, channel)` (or shortcut `ops->visible`) per (type, attr, channel), and synthesizes one `struct device_attribute` per visible (type, attr, channel) via a template table (`hwmon_<type>_attr_templates`). Reads dispatch through `hwmon_attr_show[_string]` → `chip->ops->read[_string]`; writes through `hwmon_attr_store` → `chip->ops->write` — all serialized on `hwdev->lock`. Optionally bridges hwmon-temp channels into the thermal subsystem (`hwmon_thermal_register_sensors` if `HWMON_C_REGISTER_TZ` and `CONFIG_THERMAL_OF`) so `sensors` and `thermald` see the same data. Critical for: `sensors` working end-to-end, server BMC monitoring, fan-speed control, laptop-thermal feedback.

This Tier-3 covers `drivers/hwmon/hwmon.c` (~1343 lines).

### Acceptance Criteria

- [ ] AC-1: `hwmon_device_register_with_info(dev, "k10temp", drvdata, &k10temp_chip, NULL)` creates `/sys/class/hwmon/hwmon<N>/` with `name == "k10temp"` and one `temp<I+1>_input` per visible (hwmon_temp, HWMON_T_INPUT, channel).
- [ ] AC-2: `hwmon_device_register_with_info` returns `-EINVAL` when `chip == NULL ∨ ops == NULL ∨ (visible == 0 ∧ is_visible == NULL) ∨ info == NULL`.
- [ ] AC-3: `hwmon_device_register_with_info` returns `-EINVAL` when computed `nattrs == 0`.
- [ ] AC-4: Driver setting `HWMON_T_MAX` but providing no `read` op: synthesis returns `-EINVAL` (read-or-write-without-callback rejected).
- [ ] AC-5: `cat /sys/class/hwmon/hwmon0/temp1_input`: dispatches to `chip->ops->read(dev, hwmon_temp, hwmon_temp_input, 0, &val)` under `hwdev->lock` and emits `"%lld\n"`.
- [ ] AC-6: `cat /sys/class/hwmon/hwmon0/temp1_label`: dispatches to `chip->ops->read_string(dev, hwmon_temp, hwmon_temp_label, 0, &s)` and emits `"%s\n"`.
- [ ] AC-7: `echo 80000 > /sys/class/hwmon/hwmon0/temp1_max`: dispatches to `chip->ops->write(..., hwmon_temp_max, 0, 80000)`; emits trace event; returns `count`.
- [ ] AC-8: `hwmon_in` channels are 0-based (`in0_input`); `hwmon_temp`/`hwmon_fan`/etc are 1-based (`temp1_input`).
- [ ] AC-9: Channel-info config bit not in template table is silently dropped (no error, no attribute).
- [ ] AC-10: `chip->info[0]->type == hwmon_chip ∧ config[0] & HWMON_C_REGISTER_TZ ∧ CONFIG_THERMAL_OF ∧ of_node present`: `temp<I+1>_input` channels with `is_visible(...) > 0` get registered with `devm_thermal_of_zone_register`; if the OF binding has no zone, log `dev_info("temp<N>_input not attached to any thermal zone")` and continue without failure.
- [ ] AC-11: SMBus host without `I2C_FUNC_SMBUS_PEC`: `hwmon_pec_register` returns 0 silently; no `pec` attribute.
- [ ] AC-12: `hwmon_notify_event(dev, hwmon_temp, hwmon_temp_crit_alarm, 0)`: triggers `sysfs_notify` on `temp1_crit_alarm` and a `KOBJ_CHANGE` uevent with `NAME=temp1_crit_alarm`; additionally pokes the bound thermal zone for channel 0 if any.
- [ ] AC-13: `hwmon_device_unregister` on a `hwmon<N>` device frees the id slot in `hwmon_ida`.
- [ ] AC-14: `devm_hwmon_device_register_with_info(dev, NULL, drvdata, chip, NULL)`: `name` auto-sanitized from `dev_name(dev)` (every non-`hwmon_is_bad_char` char replaced with `_`).
- [ ] AC-15: `hwmon_device_register(dev)`: emits the deprecation warning and proceeds.

### Architecture

```
struct HwmonDevice {
  name: Option<&'static str>,
  label: Option<KString>,
  dev: Device,                             // class = hwmon_class, parent = caller's dev
  chip: Option<&'static HwmonChipInfo>,
  lock: Mutex,
  tzdata: List<HwmonThermalData>,
  group: AttributeGroup,                   // synthesized attrs
  groups: Vec<&'static AttributeGroup>,    // NULL-terminated, [&group, extras...]
}

struct HwmonChipInfo {
  ops: &'static HwmonOps,
  info: &'static [&'static HwmonChannelInfo],   // NULL-terminated
}

struct HwmonChannelInfo {
  type: HwmonSensorType,                   // chip | temp | in | curr | power | energy | energy64 | humidity | fan | pwm | intrusion
  config: &'static [u32],                  // 0-terminated, one bitmask per channel
}

struct HwmonOps {
  visible: u32,                            // 0 = use is_visible
  is_visible: Option<fn(&dyn Any, HwmonSensorType, u32, i32) -> u16>,
  read: Option<fn(&Device, HwmonSensorType, u32, i32, *mut i64) -> Result<()>>,
  read_string: Option<fn(&Device, HwmonSensorType, u32, i32, *mut *const str) -> Result<()>>,
  write: Option<fn(&Device, HwmonSensorType, u32, i32, i64) -> Result<()>>,
}

struct HwmonDeviceAttribute {
  dev_attr: DeviceAttribute,
  ops: &'static HwmonOps,
  type: HwmonSensorType,
  attr: u32,                               // index within type's template table
  index: i32,                              // channel index
  name: [u8; MAX_SYSFS_ATTR_NAME_LENGTH],  // 32 bytes
}

struct HwmonThermalData {
  node: ListNode,                          // hwdev.tzdata
  dev: *mut Device,                        // hwmon device
  index: i32,                              // channel index
  tzd: *mut ThermalZoneDevice,
}
```

`Hwmon::register_inner(dev, name, drvdata, chip, groups) -> Result<&Device>`:
1. If `name.contains_any("-* \t\n")`: `dev_warn("hwmon: ... is not a valid name attribute, please fix")`.
2. `id = hwmon_ida.alloc()?`.
3. Allocate `hwdev`.
4. If `chip.is_some()`:
   a. `nattrs = Σ Hwmon::num_channel_attrs(chip.info[i])`.
   b. If `nattrs == 0`: return `-EINVAL`.
   c. Allocate `attrs[nattrs + 1]` (NULL-terminated).
   d. For each `chip.info[i]`: `aindex += Hwmon::gen_channel_attrs(drvdata, &attrs[aindex], chip.ops, info[i])?`.
   e. `hwdev.group.attrs = attrs`.
   f. Allocate `hwdev.groups[ngroups + 2]`; first slot = `&hwdev.group`; append `groups`; NULL-terminate.
   g. `hdev.groups = hwdev.groups`.
5. Else: `hdev.groups = groups`.
6. If `dev.property_present("label")`: `hwdev.label = kstrdup(dev.property_read_string("label")?)`.
7. `hwdev.name = name; hdev.class = &hwmon_class; hdev.parent = dev`.
8. Walk `tdev` parents to find first `of_node`.
9. `hwdev.chip = chip; mutex_init(&hwdev.lock); hdev.set_drvdata(drvdata); hdev.set_name("hwmon{}", id)`.
10. `device_register(hdev)?`.
11. `INIT_LIST_HEAD(&hwdev.tzdata)`.
12. If `hdev.of_node ∧ chip.is_some() ∧ chip.ops.read.is_some() ∧ chip.info[0].type == HwmonSensorType::Chip`:
    a. `config = chip.info[0].config[0]`.
    b. If `config & HWMON_C_REGISTER_TZ`: `Hwmon::thermal_register_sensors(hdev)?` (on error: `device_unregister + ida_free + return`).
    c. If `config & HWMON_C_PEC` (CONFIG_I2C): `Hwmon::pec_register(hdev)?` (same cleanup).
13. Return `Ok(hdev)`.

`Hwmon::register_with_info(dev, name, drvdata, chip, extra_groups) -> Result<&Device>`:
1. Validate `dev.is_some() ∧ name.is_some() ∧ chip.is_some()`.
2. Validate `chip.ops.is_some() ∧ (chip.ops.visible != 0 ∨ chip.ops.is_visible.is_some()) ∧ chip.info != NULL`.
3. `Hwmon::register_inner(dev, name, drvdata, Some(chip), extra_groups)`.

`Hwmon::create_attrs(drvdata, chip) -> Result<Vec<&'static Attribute>>`:
1. `nattrs = chip.info.iter().map(|i| Hwmon::num_channel_attrs(i)).sum()`.
2. If `nattrs == 0`: return `-EINVAL`.
3. Allocate `attrs[nattrs + 1]`.
4. For each `info ∈ chip.info`: `aindex += Hwmon::gen_channel_attrs(drvdata, &mut attrs[aindex..], chip.ops, info)?`.
5. Return `attrs`.

`Hwmon::gen_channel_attrs(drvdata, attrs, ops, info) -> Result<usize>`:
1. If `info.type ≥ TEMPLATE_COUNT`: return `-EINVAL`.
2. `templates = __templates[info.type]; size = __templates_size[info.type]`.
3. `aindex = 0`.
4. For `(i, config) in info.config.iter().take_while(|&&c| c != 0).enumerate()`:
   - `mask = *config`.
   - While `mask != 0`:
     - `attr = mask.trailing_zeros()`; `mask &= !(1 << attr)`.
     - If `attr ≥ size ∨ templates[attr].is_null()`: continue.
     - `match Hwmon::gen_attr(drvdata, info.type, attr, i, templates[attr], ops) { Ok(a) => { attrs[aindex] = a; aindex += 1 }, Err(ENOENT) => continue, Err(e) => return Err(e) }`.
5. Return `Ok(aindex)`.

`Hwmon::gen_attr(drvdata, type, attr, index, template, ops) -> Result<&'static Attribute>`:
1. `is_string = Hwmon::is_string_attr(type, attr)`.
2. `mode = Hwmon::is_visible(ops, drvdata, type, attr, index)`.
3. If `mode == 0`: return `Err(ENOENT)`.
4. If `(mode & 0o444) ∧ ((is_string ∧ ops.read_string.is_none()) ∨ (!is_string ∧ ops.read.is_none()))`: return `Err(EINVAL)`.
5. If `(mode & 0o222) ∧ ops.write.is_none()`: return `Err(EINVAL)`.
6. Allocate `hattr: HwmonDeviceAttribute`.
7. If `type == HwmonSensorType::Chip`: `name = template`.
8. Else: `scnprintf!(hattr.name, template, index + Hwmon::attr_base(type))`; `name = &hattr.name`.
9. `hattr.type = type; hattr.attr = attr; hattr.index = index; hattr.ops = ops`.
10. `hattr.dev_attr.show = if is_string { Hwmon::attr_show_string } else { Hwmon::attr_show }`.
11. `hattr.dev_attr.store = Hwmon::attr_store`.
12. `sysfs_attr_init(&hattr.dev_attr.attr); hattr.dev_attr.attr.name = name; hattr.dev_attr.attr.mode = mode`.
13. Return `Ok(&hattr.dev_attr.attr)`.

`Hwmon::attr_show(dev, devattr, buf) -> ssize_t`:
1. /* hwdev.lock guarded */.
2. `val64: i64`.
3. If `hattr.type == HwmonSensorType::Energy64`: `hattr.ops.read(dev, hattr.type, hattr.attr, hattr.index, &mut val64 as *mut i64 as *mut _)?`.
4. Else: `let mut val: i64 = 0; hattr.ops.read(dev, hattr.type, hattr.attr, hattr.index, &mut val)?; val64 = val`.
5. `trace_hwmon_attr_show(hattr.index + Hwmon::attr_base(hattr.type), &hattr.name, val64)`.
6. `sysfs_emit!(buf, "{}\n", val64)`.

`Hwmon::attr_show_string(dev, devattr, buf) -> ssize_t`:
1. /* hwdev.lock guarded */.
2. `let mut s: *const c_char = ptr::null();`
3. `hattr.ops.read_string(dev, hattr.type, hattr.attr, hattr.index, &mut s)?`.
4. `trace_hwmon_attr_show_string`.
5. `sysfs_emit!(buf, "{}\n", CStr::from_ptr(s).to_str())`.

`Hwmon::attr_store(dev, devattr, buf, count) -> ssize_t`:
1. `val: i64 = kstrtol(buf, 10)?`.
2. /* hwdev.lock guarded */.
3. `hattr.ops.write(dev, hattr.type, hattr.attr, hattr.index, val)?`.
4. `trace_hwmon_attr_store`.
5. Return `count`.

`Hwmon::notify_event(dev, type, attr, channel) -> Result<()>`:
1. Validate `type < TEMPLATE_COUNT ∧ attr < TEMPLATE_SIZE[type]`.
2. `template = __templates[type][attr]`.
3. `base = Hwmon::attr_base(type)`.
4. `scnprintf!(sattr, template, base + channel)`.
5. `scnprintf!(event, "NAME={}", sattr)`.
6. `sysfs_notify(&dev.kobj, NULL, sattr)`.
7. `kobject_uevent_env(&dev.kobj, KOBJ_CHANGE, &[event])`.
8. If `type == HwmonSensorType::Temp ∧ CONFIG_THERMAL_OF`: `Hwmon::thermal_notify(dev, channel)`.

`Hwmon::thermal_register_sensors(dev) -> Result<()>`:
1. If `!CONFIG_THERMAL_OF`: return Ok(()).
2. For `(i, info)` in `chip.info` enumerated from 1 (skipping the leading `hwmon_chip` entry):
   - If `info.type != HwmonSensorType::Temp`: continue.
   - For `(j, config)` in `info.config` enumerated:
     - If `!(config & HWMON_T_INPUT) ∨ Hwmon::is_visible(chip.ops, drvdata, HwmonSensorType::Temp, HWMON_T_INPUT, j) == 0`: continue.
     - `Hwmon::thermal_add_sensor(dev, j)?`.

`Hwmon::thermal_add_sensor(dev, index) -> Result<()>`:
1. Allocate `tdata: HwmonThermalData` via `devm_kzalloc`.
2. `tdata.dev = dev; tdata.index = index`.
3. `tzd = devm_thermal_of_zone_register(dev, index, tdata, &hwmon_thermal_ops)`.
4. If `tzd == Err(ENODEV)`: `dev_info("temp{}_input not attached to any thermal zone", index + 1)`; `devm_kfree(dev, tdata)`; return Ok(()).
5. Else if `Err(e)`: return `Err(e)`.
6. `devm_add_action(dev, Hwmon::thermal_remove_sensor, &tdata.node)?`.
7. `tdata.tzd = tzd`; `list_add(&tdata.node, &hwdev.tzdata)`.

`Hwmon::thermal_get_temp(tz, *temp) -> Result<()>`:
1. `tdata = thermal_zone_device_priv(tz)`.
2. `hwdev = container_of(tdata.dev)`.
3. /* hwdev.lock guarded */.
4. `t = hwdev.chip.ops.read(tdata.dev, HwmonSensorType::Temp, HWMON_TEMP_INPUT, tdata.index)?`.
5. `*temp = t`.

`Hwmon::thermal_set_trips(tz, low, high) -> Result<()>`:
1. `tdata`, `hwdev`, `chip`, `info` resolved.
2. If `chip.ops.write.is_none()`: return Ok(()).
3. Find `info[i].type == HwmonSensorType::Temp`.
4. /* hwdev.lock guarded */.
5. If `info[i].config[tdata.index] & HWMON_T_MIN`: `chip.ops.write(..., HWMON_TEMP_MIN, tdata.index, low)?` (tolerate `EOPNOTSUPP`).
6. If `info[i].config[tdata.index] & HWMON_T_MAX`: `chip.ops.write(..., HWMON_TEMP_MAX, tdata.index, high)?` (tolerate `EOPNOTSUPP`).

`Hwmon::pec_register(hdev) -> Result<()>`:
1. `client = i2c_verify_client(hdev.parent)`.
2. If `client.is_none()`: return `-EINVAL`.
3. If `!client.adapter.check_functionality(I2C_FUNC_SMBUS_PEC)`: return Ok(()).
4. `device_create_file(&client.dev, &dev_attr_pec)?`.
5. `devm_add_action_or_reset(hdev, Hwmon::remove_pec, &client.dev)`.

`Hwmon::unregister(dev)`:
1. `id = sscanf(dev_name(dev), "hwmon{}")?` — on parse error: `dev_dbg("bad class ID!")` and return.
2. `device_unregister(dev)`.
3. `hwmon_ida.free(id)`.

`Hwmon::sanitize_name(name) -> Result<KString>`:
1. Validate `name.is_some()`.
2. Duplicate.
3. For each byte: if `hwmon_is_bad_char(b)`: replace with `_`.
4. Return.

### Out of Scope

- `drivers/hwmon/hwmon-vid.c` VID-table lookup helpers — covered alongside core in Tier-3.
- Per-chip drivers (`coretemp.c`, `k10temp.c`, `nct67xx*.c`, `lm75.c`, `tmp*.c`, etc.) — covered in `drivers/hwmon/cpu-x86.md` / `drivers/hwmon/super-io.md` / `drivers/hwmon/i2c-temp.md` / `drivers/hwmon/laptop.md` Tier-3.
- `drivers/hwmon/pmbus/` — covered in `drivers/hwmon/pmbus.md` Tier-3.
- `drivers/hwmon/peci/` — covered in `drivers/hwmon/peci.md` Tier-3.
- `drivers/hwmon/acpi_power_meter.c` — covered in `drivers/hwmon/acpi.md` Tier-3.
- `drivers/thermal/thermal_hwmon.c` (the *reverse* bridge: thermal-zone → hwmon attribute) — covered in `drivers/thermal/hwmon-bridge.md` Tier-3.
- `drivers/thermal/of-thermal.c` (`devm_thermal_of_zone_register`) — covered in `drivers/thermal/` Tier-3 alongside core.
- Implementation code.

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct hwmon_device` | per-class device | `HwmonDevice` |
| `struct hwmon_chip_info` | per-driver `ops` + `info[]` | `HwmonChipInfo` |
| `struct hwmon_channel_info` | per-channel type + config bitmask | `HwmonChannelInfo` |
| `struct hwmon_ops` | per-driver visible/read/read_string/write | `HwmonOps` |
| `struct hwmon_device_attribute` | per-attribute container (`type`, `attr`, `index`, `name`) | `HwmonDeviceAttribute` |
| `struct hwmon_thermal_data` | per-temp-channel thermal-zone bridge | `HwmonThermalData` |
| `hwmon_device_register_with_info()` | per-modern register | `Hwmon::register_with_info` |
| `hwmon_device_register_with_groups()` | per-legacy register (attribute groups) | `Hwmon::register_with_groups` |
| `hwmon_device_register_for_thermal()` | per-thermal-subsys register | `Hwmon::register_for_thermal` |
| `hwmon_device_register()` | per-deprecated register (warn) | `Hwmon::register_deprecated` |
| `hwmon_device_unregister()` | per-unregister | `Hwmon::unregister` |
| `devm_hwmon_device_register_with_info()` / `_with_groups()` | per-devres-managed register | `Hwmon::devm_register*` |
| `__hwmon_device_register()` | per-internal common register | `Hwmon::register_inner` |
| `__hwmon_create_attrs()` | per-chip attribute synthesis | `Hwmon::create_attrs` |
| `hwmon_genattrs()` | per-channel-info attribute walk | `Hwmon::gen_channel_attrs` |
| `hwmon_genattr()` | per-(type, attr, index) attribute synthesis | `Hwmon::gen_attr` |
| `hwmon_is_visible()` | per-attr visibility (`ops.visible` shortcut or `ops.is_visible`) | `Hwmon::is_visible` |
| `is_string_attr()` | per-attr string-vs-number discriminator | `Hwmon::is_string_attr` |
| `hwmon_attr_show()` / `_show_string()` / `_store()` | per-attr accessors | `Hwmon::attr_show*` / `attr_store` |
| `hwmon_notify_event()` | per-event sysfs_notify + uevent | `Hwmon::notify_event` |
| `hwmon_lock()` / `_unlock()` | per-driver-exported serialization | `Hwmon::lock` / `unlock` |
| `hwmon_num_channel_attrs()` | per-channel attr count via `hweight32` | `Hwmon::num_channel_attrs` |
| `hwmon_attr_base()` | per-type 0-based vs 1-based numbering | `Hwmon::attr_base` |
| `hwmon_free_attrs()` | per-cleanup | `Hwmon::free_attrs` |
| `hwmon_dev_release()` | per-class release | `Hwmon::dev_release` |
| `hwmon_dev_attr_is_visible()` | per-`name`/`label` visibility | `Hwmon::dev_attr_is_visible` |
| `name_show()` / `label_show()` | per-`name`/`label` show | `Hwmon::name_show` / `label_show` |
| `pec_show()` / `pec_store()` | per-SMBus-PEC sysfs (CONFIG_I2C) | `Hwmon::pec_show` / `pec_store` |
| `hwmon_pec_register()` / `hwmon_remove_pec()` | per-PEC attribute lifecycle | `Hwmon::pec_register` / `remove_pec` |
| `hwmon_match_device()` | per-i2c-child lookup | `Hwmon::match_device` |
| `hwmon_thermal_get_temp()` | per-thermal `get_temp` bridge | `Hwmon::thermal_get_temp` |
| `hwmon_thermal_set_trips()` | per-thermal `set_trips` bridge | `Hwmon::thermal_set_trips` |
| `hwmon_thermal_add_sensor()` / `_remove_sensor()` | per-channel TZ bind | `Hwmon::thermal_add_sensor` / `_remove` |
| `hwmon_thermal_register_sensors()` | per-chip TZ-register sweep | `Hwmon::thermal_register_sensors` |
| `hwmon_thermal_notify()` | per-event TZ update | `Hwmon::thermal_notify` |
| `hwmon_sanitize_name()` / `devm_hwmon_sanitize_name()` | per-name char-sanitization | `Hwmon::sanitize_name` / `devm_sanitize_name` |
| `hwmon_pci_quirks()` | per-MSI MS-7031 ATI SB port-open quirk | `Hwmon::pci_quirks` |
| `hwmon_ida` | per-class IDA | shared |
| `hwmon_class` | per-class device | shared |

### compatibility contract

REQ-1: struct hwmon_device fields:
- `name`: per-chip name (validated via `hwmon_is_bad_char`; warned if it contains `-`, `*`, space, tab, newline).
- `label`: per-`label` device-property override.
- `dev`: embedded `struct device` (class = `hwmon_class`, parent = caller's `dev`, `of_node` inherited).
- `chip`: per-driver `hwmon_chip_info` snapshot pointer.
- `lock`: per-device mutex (serializes every `chip->ops->{read, read_string, write}` call).
- `tzdata`: per-thermal-zone list (`hwmon_thermal_data` entries).
- `group`: per-synthesized `attribute_group` (`attrs` = pointer-array of generated `device_attribute`).
- `groups`: NULL-terminated combined list (`&hwdev->group` first, then `extra_groups`).

REQ-2: struct hwmon_chip_info fields:
- `ops`: pointer to `struct hwmon_ops` (`is_visible` or `visible` mandatory; `read`/`read_string`/`write` optional per attribute mode).
- `info`: NULL-terminated array of `const struct hwmon_channel_info *`.

REQ-3: struct hwmon_channel_info fields:
- `type`: `enum hwmon_sensor_types ∈ {hwmon_chip, hwmon_temp, hwmon_in, hwmon_curr, hwmon_power, hwmon_energy, hwmon_energy64, hwmon_humidity, hwmon_fan, hwmon_pwm, hwmon_intrusion}`.
- `config[]`: 0-terminated array of u32 attribute-bitmasks, one entry per channel; `__ffs(config[ch])` selects each per-channel attribute index into the template table.

REQ-4: struct hwmon_ops fields:
- `visible: u32` (shortcut: same mode for all attributes).
- `is_visible(drvdata, type, attr, channel) -> umode_t` (per-attribute mode; 0 = absent).
- `read(dev, type, attr, channel, *val) -> int`.
- `read_string(dev, type, attr, channel, *str) -> int`.
- `write(dev, type, attr, channel, val) -> int`.

REQ-5: struct hwmon_device_attribute fields:
- `dev_attr`: embedded `struct device_attribute`.
- `ops`: per-`hwmon_ops` snapshot.
- `type`: per-channel-info type.
- `attr`: per-attribute index within the type.
- `index`: per-channel index.
- `name`: per-attribute synthesized name (`MAX_SYSFS_ATTR_NAME_LENGTH = 32`).

REQ-6: __hwmon_device_register(dev, name, drvdata, chip, groups) — common path:
- Warn (`dev_warn`) if `name` contains `-`, `*`, space, tab, or newline.
- `id = ida_alloc(&hwmon_ida)`; on failure return ERR_PTR.
- Allocate `hwdev`.
- /* attribute synthesis */
- If `chip != NULL`:
  - `ngroups = 2 + |groups|` (terminating NULL + `&hwdev->group` + extra).
  - Allocate `hwdev->groups` (pointer array).
  - `attrs = __hwmon_create_attrs(drvdata, chip)` (synthesize one `device_attribute` per visible (type, attr, channel)).
  - `hwdev->group.attrs = attrs`.
  - `hwdev->groups[0] = &hwdev->group`.
  - Append `extra_groups` after.
  - `hdev->groups = hwdev->groups`.
- Else: `hdev->groups = groups` (legacy `register_with_groups` path).
- If parent `dev` has `"label"` device-property: `device_property_read_string(dev, "label", &label)`, `hwdev->label = kstrdup(label)`.
- `hwdev->name = name`.
- `hdev->class = &hwmon_class; hdev->parent = dev`.
- Walk parents to find `of_node` (`tdev` chain).
- `hwdev->chip = chip`; `mutex_init(&hwdev->lock)`.
- `dev_set_drvdata(hdev, drvdata)`.
- `dev_set_name(hdev, "hwmon%d", id)`.
- `device_register(hdev)`; on failure `put_device + ida_free`.
- `INIT_LIST_HEAD(&hwdev->tzdata)`.
- If `hdev->of_node ∧ chip ∧ chip->ops->read ∧ chip->info[0]->type == hwmon_chip`:
  - `config = chip->info[0]->config[0]`.
  - If `HWMON_C_REGISTER_TZ`: `hwmon_thermal_register_sensors(hdev)`; on failure `device_unregister + ida_free`.
  - If `HWMON_C_PEC` (CONFIG_I2C): `hwmon_pec_register(hdev)`; on failure `device_unregister + ida_free`.
- Return `hdev`.

REQ-7: hwmon_device_register_with_info(dev, name, drvdata, chip, extra_groups):
- Validate `dev ∧ name ∧ chip != NULL`.
- Validate `chip->ops ∧ (chip->ops->visible ∨ chip->ops->is_visible) ∧ chip->info != NULL`.
- Tail-call `__hwmon_device_register`.

REQ-8: hwmon_device_register_with_groups(dev, name, drvdata, groups):
- Validate `name != NULL`.
- Tail-call `__hwmon_device_register(..., chip=NULL, groups)`.

REQ-9: hwmon_device_register_for_thermal(dev, name, drvdata):
- Validate `name ∧ dev != NULL`.
- Tail-call `__hwmon_device_register(..., chip=NULL, groups=NULL)`.
- Restricted symbol — exported `EXPORT_SYMBOL_NS_GPL(..., "HWMON_THERMAL")`.

REQ-10: hwmon_device_register(dev):
- Deprecated path. `dev_warn("hwmon_device_register() is deprecated. Please convert the driver to use hwmon_device_register_with_info().")`.
- Tail-call `__hwmon_device_register(dev, NULL, NULL, NULL, NULL)`.

REQ-11: hwmon_device_unregister(dev):
- `sscanf(dev_name(dev), "hwmon%d", &id)` must succeed.
- `device_unregister(dev)`; `ida_free(&hwmon_ida, id)`.
- Else: `dev_dbg(dev->parent, "hwmon_device_unregister() failed: bad class ID!")`.

REQ-12: __hwmon_create_attrs(drvdata, chip):
- `nattrs = Σ hwmon_num_channel_attrs(chip->info[i])` over all `i`.
- If `nattrs == 0`: return `ERR_PTR(-EINVAL)`.
- Allocate `attrs` (size `nattrs + 1`, NULL-terminated).
- For each `info[i]`: `hwmon_genattrs(drvdata, &attrs[aindex], chip->ops, info[i])`; advance by returned count.
- On failure: `hwmon_free_attrs(attrs)`; return error.

REQ-13: hwmon_num_channel_attrs(info):
- `Σ hweight32(info->config[i])` over all `i` until `config[i] == 0`.

REQ-14: hwmon_genattrs(drvdata, attrs, ops, info):
- Validate `info->type < ARRAY_SIZE(__templates)`.
- `templates = __templates[info->type]`; `template_size = __templates_size[info->type]`.
- For each channel `i` until `info->config[i] == 0`:
  - For each set bit `attr` in `info->config[i]` (via `__ffs` + clear):
    - If `attr ≥ template_size ∨ templates[attr] == NULL`: skip (invisible).
    - `a = hwmon_genattr(drvdata, info->type, attr, i, templates[attr], ops)`.
    - If `a == ERR_PTR(-ENOENT)`: skip; else if `IS_ERR(a)`: return error.
    - Else `attrs[aindex++] = a`.
- Return `aindex`.

REQ-15: hwmon_genattr(drvdata, type, attr, index, template, ops):
- `is_string = is_string_attr(type, attr)`.
- `mode = hwmon_is_visible(ops, drvdata, type, attr, index)`.
- If `mode == 0`: return `ERR_PTR(-ENOENT)` (callee may skip).
- If `(mode & 0444) ∧ ((is_string ∧ !ops->read_string) ∨ (!is_string ∧ !ops->read))`: return `ERR_PTR(-EINVAL)`.
- If `(mode & 0222) ∧ !ops->write`: return `ERR_PTR(-EINVAL)`.
- Allocate `hattr`.
- If `type == hwmon_chip`: `name = template` (chip attrs are literal names like `update_interval`, not `%d`-templated).
- Else: `scnprintf(hattr->name, template, index + hwmon_attr_base(type))` (e.g., `temp1_input`, `in0_label`).
- `hattr->type, attr, index, ops` filled.
- `dattr->show = is_string ? hwmon_attr_show_string : hwmon_attr_show`.
- `dattr->store = hwmon_attr_store`.
- `sysfs_attr_init`; `a->name = name; a->mode = mode`.
- Return `&dattr->attr`.

REQ-16: hwmon_attr_base(type):
- Returns `0` for `hwmon_in` and `hwmon_intrusion`; returns `1` otherwise.
- → `in0_input` ... `in8_input` for voltages, `intrusion0_alarm` for intrusion; `temp1_input` ... `temp7_input` for temperatures, `fan1_input` for fans, etc.

REQ-17: hwmon_is_visible(ops, drvdata, type, attr, channel):
- If `ops->visible != 0`: return `ops->visible` (driver-wide constant mode shortcut).
- Else: return `ops->is_visible(drvdata, type, attr, channel)` (per-attribute computed mode).

REQ-18: is_string_attr(type, attr):
- True iff `(type, attr) ∈ {(hwmon_temp, hwmon_temp_label), (hwmon_in, hwmon_in_label), (hwmon_curr, hwmon_curr_label), (hwmon_power, hwmon_power_label), (hwmon_energy, hwmon_energy_label), (hwmon_energy64, hwmon_energy_label), (hwmon_humidity, hwmon_humidity_label), (hwmon_fan, hwmon_fan_label)}`.

REQ-19: hwmon_attr_show(dev, devattr, buf):
- /* hwdev->lock guarded */.
- `ret = hattr->ops->read(dev, hattr->type, hattr->attr, hattr->index, val_target)`.
- For `hwmon_energy64`: read into `s64 val64` directly; else into `long val` then widen.
- On err: return err.
- `trace_hwmon_attr_show(hattr->index + hwmon_attr_base(type), hattr->name, val64)`.
- `sysfs_emit(buf, "%lld\n", val64)`.

REQ-20: hwmon_attr_show_string(dev, devattr, buf):
- /* hwdev->lock guarded */.
- `ret = hattr->ops->read_string(dev, hattr->type, hattr->attr, hattr->index, &s)`.
- On err: return err.
- `trace_hwmon_attr_show_string`.
- `sysfs_emit(buf, "%s\n", s)`.

REQ-21: hwmon_attr_store(dev, devattr, buf, count):
- `kstrtol(buf, 10, &val)` → on err return err.
- /* hwdev->lock guarded */.
- `ret = hattr->ops->write(dev, hattr->type, hattr->attr, hattr->index, val)`.
- `trace_hwmon_attr_store`.
- Return `count`.

REQ-22: hwmon_notify_event(dev, type, attr, channel):
- Validate `type < ARRAY_SIZE(__templates)` and `attr < __templates_size[type]`.
- `template = __templates[type][attr]`.
- `base = hwmon_attr_base(type)`.
- `scnprintf(sattr, template, base + channel)` → e.g., `"temp1_crit_alarm"`.
- `scnprintf(event, "NAME=%s", sattr)`.
- `sysfs_notify(&dev->kobj, NULL, sattr)`.
- `kobject_uevent_env(&dev->kobj, KOBJ_CHANGE, {event, NULL})`.
- If `type == hwmon_temp ∧ CONFIG_THERMAL_OF`: `hwmon_thermal_notify(dev, channel)` (kick bound thermal-zone update).

REQ-23: hwmon_thermal_register_sensors(dev) — gated by `CONFIG_THERMAL_OF`:
- For each `info[i]` starting at `i = 1` (skip `hwmon_chip`):
  - If `info[i]->type != hwmon_temp`: continue.
  - For each channel `j` until `config[j] == 0`:
    - If `!(config[j] & HWMON_T_INPUT) ∨ !hwmon_is_visible(chip->ops, drvdata, hwmon_temp, hwmon_temp_input, j)`: continue.
    - `hwmon_thermal_add_sensor(dev, j)`.

REQ-24: hwmon_thermal_add_sensor(dev, index):
- Allocate `hwmon_thermal_data` (devres-managed).
- `tdata->dev = dev; tdata->index = index`.
- `tzd = devm_thermal_of_zone_register(dev, index, tdata, &hwmon_thermal_ops)`.
- If `IS_ERR(tzd) ∧ PTR_ERR(tzd) == -ENODEV`: `dev_info("temp<N+1>_input not attached to any thermal zone")`; free tdata; return 0.
- Else if IS_ERR: return error.
- `devm_add_action(dev, hwmon_thermal_remove_sensor, &tdata->node)`.
- `tdata->tzd = tzd`; `list_add(&tdata->node, &hwdev->tzdata)`.

REQ-25: hwmon_thermal_get_temp(tz, *temp):
- `tdata = thermal_zone_device_priv(tz)`; `hwdev = container_of(tdata->dev)`.
- /* hwdev->lock guarded */.
- `ret = hwdev->chip->ops->read(tdata->dev, hwmon_temp, hwmon_temp_input, tdata->index, &t)`.
- On err: return err.
- `*temp = t`.

REQ-26: hwmon_thermal_set_trips(tz, low, high):
- `tdata`, `hwdev`, `chip`, `info` resolved.
- If `!chip->ops->write`: return 0 (silently honored).
- Find `info[i]` of type `hwmon_temp`.
- /* hwdev->lock guarded */.
- If `config[index] & HWMON_T_MIN`: `chip->ops->write(..., hwmon_temp_min, index, low)` (tolerate `-EOPNOTSUPP`).
- If `config[index] & HWMON_T_MAX`: `chip->ops->write(..., hwmon_temp_max, index, high)` (tolerate `-EOPNOTSUPP`).

REQ-27: hwmon_thermal_notify(dev, index):
- If `!CONFIG_THERMAL_OF`: return.
- For each `tzdata ∈ hwdev->tzdata`: if `tzdata->index == index`: `thermal_zone_device_update(tzdata->tzd, THERMAL_EVENT_UNSPECIFIED)`.

REQ-28: hwmon_pec_register(hdev) — gated by `IS_REACHABLE(CONFIG_I2C)`:
- `client = i2c_verify_client(hdev->parent)`; if NULL return `-EINVAL`.
- If `!i2c_check_functionality(client->adapter, I2C_FUNC_SMBUS_PEC)`: return 0 (silently skipped).
- `device_create_file(&client->dev, &dev_attr_pec)`.
- `devm_add_action_or_reset(hdev, hwmon_remove_pec, &client->dev)`.

REQ-29: pec_show(dev, ...) / pec_store(dev, ..., buf, count):
- `pec_show`: `sysfs_emit("%d\n", !!(client->flags & I2C_CLIENT_PEC))`.
- `pec_store`:
  - `kstrtobool(buf, &val)`.
  - `hdev = device_find_child(dev, NULL, hwmon_match_device)`; if not found `-ENODEV`.
  - Under `hwdev->lock`: if `chip->ops->write`: `chip->ops->write(hdev, hwmon_chip, hwmon_chip_pec, 0, val)` (tolerate `-EOPNOTSUPP`).
  - Update `client->flags` PEC bit.
  - `put_device(hdev)`.
  - Return `count`.

REQ-30: hwmon_sanitize_name(name) / devm_hwmon_sanitize_name(dev, name):
- Validate `name != NULL` (and `dev != NULL` for devm variant).
- Duplicate (kstrdup / devm_kstrdup).
- Replace every char where `hwmon_is_bad_char` is true with `_`.
- Returned string is caller-owned (or devres-owned for devm variant).

REQ-31: devm-wrapped registers:
- `devm_hwmon_device_register_with_groups(dev, name, drvdata, groups)` — `devres_alloc(devm_hwmon_release)`, call register, attach via `devres_add`.
- `devm_hwmon_device_register_with_info(dev, name, drvdata, chip, extra_groups)` — if `name == NULL`: sanitize `dev_name(dev)` via `devm_hwmon_sanitize_name`.
- Cleanup: `devm_hwmon_release` calls `hwmon_device_unregister(*hwdev)`.

REQ-32: hwmon_class:
- `.name = "hwmon"`.
- `.dev_groups = hwmon_dev_attr_groups` — single group of `{name, label}` with `hwmon_dev_attr_is_visible` (hide `name` if NULL, hide `label` if NULL).
- `.dev_release = hwmon_dev_release` (free `group.attrs`, `groups`, `label`, `hwdev`).

REQ-33: hwmon_pci_quirks() — `__init` x86+PCI only:
- Open `0x295-0x296` SMBus base port on MSI MS-7031 ATI south-bridge: probe `PCI_VENDOR_ID_ATI, 0x436c`; check subsys-vendor `0x1462` + subsys-dev `0x0031`; if base==0 and bit2 of config-0x48 is clear: write base `0x295` to config-0x64, set bit2 of 0x48. Pre-flight before `class_register`.

REQ-34: hwmon_init():
- `subsys_initcall`.
- `hwmon_pci_quirks`.
- `class_register(&hwmon_class)`; on err `pr_err`.
- `hwmon_exit`: `class_unregister`.

REQ-35: Locking discipline:
- `hwmon_ida` — kernel-internal IDA serialization.
- `hwdev->lock` — per-device mutex serializing every `chip->ops->{read, read_string, write}` invocation.
- `hwmon_lock(dev)` / `hwmon_unlock(dev)` — exported for drivers that need to extend the critical section across multiple ops calls.
- No global hwmon mutex (unlike thermal-core); the class-device model + per-device mutex suffices.

REQ-36: Sysfs surface (per-channel naming):
- `/sys/class/hwmon/hwmon<N>/name` — read-only chip name.
- `/sys/class/hwmon/hwmon<N>/label` — read-only `"label"` device-property override (if present).
- `/sys/class/hwmon/hwmon<N>/temp<I+1>_input | _min | _max | _crit | _crit_hyst | _emergency | _alarm | _fault | _offset | _label | _enable | _type | _lcrit | _lcrit_hyst | _lcrit_alarm | _min_hyst | _min_alarm | _max_hyst | _max_alarm | _crit_alarm | _emergency_hyst | _emergency_alarm | _lowest | _highest | _reset_history | _rated_min | _rated_max | _beep`.
- `/sys/class/hwmon/hwmon<N>/in<I>_input | _min | _max | _lcrit | _crit | _average | _lowest | _highest | _reset_history | _label | _enable | _alarm | _min_alarm | _max_alarm | _lcrit_alarm | _crit_alarm | _rated_min | _rated_max | _beep | _fault`.
- `/sys/class/hwmon/hwmon<N>/curr<I+1>_*` — same set as `in` (with appropriate substitutions).
- `/sys/class/hwmon/hwmon<N>/power<I+1>_input | _average | _average_interval | _average_interval_max | _average_interval_min | _average_highest | _average_lowest | _average_max | _average_min | _input_highest | _input_lowest | _reset_history | _accuracy | _cap | _cap_hyst | _cap_max | _cap_min | _min | _max | _lcrit | _crit | _label | _enable | _alarm | _cap_alarm | _min_alarm | _max_alarm | _lcrit_alarm | _crit_alarm | _rated_min | _rated_max`.
- `/sys/class/hwmon/hwmon<N>/energy<I+1>_input | _label | _enable` (`hwmon_energy` 32-bit and `hwmon_energy64` 64-bit share the same templates).
- `/sys/class/hwmon/hwmon<N>/humidity<I+1>_*` (`_input`, `_min`, `_min_hyst`, `_max`, `_max_hyst`, `_alarm`, `_fault`, ...).
- `/sys/class/hwmon/hwmon<N>/fan<I+1>_input | _min | _max | _div | _pulses | _target | _alarm | _min_alarm | _max_alarm | _fault | _label | _enable | _beep`.
- `/sys/class/hwmon/hwmon<N>/pwm<I+1> | pwm<I+1>_enable | _mode | _freq | _auto_channels_temp`.
- `/sys/class/hwmon/hwmon<N>/intrusion<I>_alarm | _beep`.
- `/sys/class/hwmon/hwmon<N>/temp_reset_history | in_reset_history | curr_reset_history | power_reset_history | update_interval | alarms | samples | curr_samples | in_samples | power_samples | temp_samples | beep_enable` — chip-level attributes (no `%d` index — `hwmon_chip` type, literal name).
- `/sys/class/hwmon/hwmon<N>/../pec` — SMBus-PEC attribute attached to *parent* i2c-client device (not the hwmon device itself).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `nattrs_zero_rejected` | INVARIANT | per-`__hwmon_create_attrs`: rejects `nattrs == 0` early — no zero-sized allocation. |
| `attrs_array_null_terminated` | INVARIANT | per-`__hwmon_create_attrs`: `attrs[aindex] == NULL` past last valid entry. |
| `attr_mode_callbacks_consistent` | INVARIANT | per-`hwmon_genattr`: (`mode & 0444 ⟹ read[_string] != NULL`) ∧ (`mode & 0222 ⟹ write != NULL`). |
| `read_locked_under_hwdev_lock` | INVARIANT | per-`hwmon_attr_show*`: `hwdev->lock` held for entire `chip->ops->read[_string]` window. |
| `write_locked_under_hwdev_lock` | INVARIANT | per-`hwmon_attr_store`: `hwdev->lock` held for entire `chip->ops->write` window. |
| `id_freed_on_register_failure` | INVARIANT | per-`__hwmon_device_register`: every error path eventually `ida_free`s id. |
| `groups_array_null_terminated` | INVARIANT | per-`__hwmon_device_register`: `hwdev->groups[last] == NULL`. |
| `release_frees_attrs_groups_label` | INVARIANT | per-`hwmon_dev_release`: free order `attrs → groups → label → hwdev`; idempotent on NULL `group.attrs`. |
| `template_index_bounds_checked` | INVARIANT | per-`hwmon_genattrs`: `attr < template_size`; per-`hwmon_notify_event`: `type < ARRAY_SIZE(__templates) ∧ attr < __templates_size[type]`. |
| `thermal_zone_register_tolerates_enodev` | INVARIANT | per-`hwmon_thermal_add_sensor`: `Err(ENODEV)` is non-fatal (logs and continues). |
| `pec_skipped_without_func` | INVARIANT | per-`hwmon_pec_register`: returns 0 without creating `pec` attribute when adapter lacks `I2C_FUNC_SMBUS_PEC`. |
| `name_bad_char_only_warns` | INVARIANT | per-`__hwmon_device_register`: bad-char in `name` warns but does not abort registration. |

### Layer 2: TLA+

`drivers/hwmon/hwmon.tla`:
- Per-register-with-info + per-attribute-synthesis + per-read/write + per-notify-event + per-thermal-bridge + per-unregister.
- Properties:
  - `safety_attr_count_matches_visible_set` — per-register: `|hwdev.group.attrs| == |{(type, attr, ch) : is_visible(...) > 0 ∧ template exists}|`.
  - `safety_attr_name_template_invariant` — per-attr: stored `name` matches `templates[type][attr]` with `index + attr_base(type)` substituted.
  - `safety_attr_base_zero_iff_in_or_intrusion` — `attr_base(type) == 0 ⟺ type ∈ {in, intrusion}`.
  - `safety_no_op_call_concurrently_same_device` — per-`chip->ops->{read, read_string, write}`: at most one in flight per `hwdev`.
  - `safety_tzdata_indexed_uniquely` — per-`hwdev.tzdata`: every entry has a unique `(dev, index)` and corresponds to a valid hwmon-temp channel with `HWMON_T_INPUT` set and visible.
  - `safety_chip_pec_only_with_chip_info_zero` — `hwmon_pec_register` only invoked when `chip->info[0]->type == hwmon_chip ∧ config[0] & HWMON_C_PEC`.
  - `safety_unregister_releases_id` — `hwmon_device_unregister` returns ⟹ id freed.
  - `liveness_register_publishes` — `register_with_info` returns Ok ⟹ `/sys/class/hwmon/hwmon<N>` visible and every visible (type, attr, ch) attribute file exists.
  - `liveness_notify_eventually_delivers` — `hwmon_notify_event` returns Ok ⟹ `sysfs_notify` fired and `KOBJ_CHANGE` uevent dispatched.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Hwmon::register_with_info` post: returns Ok ⟹ `/sys/class/hwmon/hwmon<N>` visible; `name` attribute (if `name.is_some()`) emits the registered string | `Hwmon::register_with_info` |
| `Hwmon::register_inner` post: every visible (type, attr, channel) with template generates exactly one attribute; invisible / no-template skipped | `Hwmon::register_inner` |
| `Hwmon::gen_attr` post: returns Ok ⟹ produced `device_attribute` has the right `show`/`store` (string vs scalar) and the synthesized `name` matches the template | `Hwmon::gen_attr` |
| `Hwmon::attr_show` post: bytes written = result of `sysfs_emit("%lld\n", val)`; `chip->ops->read` invoked under `hwdev.lock` | `Hwmon::attr_show` |
| `Hwmon::attr_store` post: returns `count` on success; `chip->ops->write` invoked under `hwdev.lock`; otherwise propagates error from `kstrtol` or `write` | `Hwmon::attr_store` |
| `Hwmon::notify_event` post: `sysfs_notify` and `kobject_uevent_env` both invoked; `Hwmon::thermal_notify` invoked iff `type == Temp ∧ CONFIG_THERMAL_OF` | `Hwmon::notify_event` |
| `Hwmon::thermal_register_sensors` post: per-channel `hwmon_thermal_add_sensor` called only for `HWMON_T_INPUT`-set ∧ visible channels | `Hwmon::thermal_register_sensors` |
| `Hwmon::thermal_get_temp` post: `*temp` matches `chip->ops->read(hwmon_temp, hwmon_temp_input, index)` value | `Hwmon::thermal_get_temp` |
| `Hwmon::unregister` post: device unregistered; id freed; if class-id parse failed: nothing freed | `Hwmon::unregister` |
| `Hwmon::sanitize_name` post: returned bytes ∀ b. `!hwmon_is_bad_char(b)` | `Hwmon::sanitize_name` |

### Layer 4: Verus/Creusot functional

`Per-driver `hwmon_chip_info` → walk `info[]` → call `is_visible` per (type, attr, ch) → instantiate template → resulting sysfs surface byte-identical to upstream` — semantic equivalence to `Documentation/hwmon/sysfs-interface.rst`. `Per-`HWMON_C_REGISTER_TZ` → walk `temp*_input` visible channels → `devm_thermal_of_zone_register` → bridged thermal-zone forwards `get_temp`/`set_trips` to `chip->ops->read`/`write`` — semantic equivalence to `Documentation/hwmon/hwmon-kernel-api.rst § Thermal subsystem integration`. `Per-`hwmon_notify_event` → `sysfs_notify` + `KOBJ_CHANGE` uevent with `NAME=<sysfs-attr>` → optional thermal-zone update for temp` — semantic equivalence to lm-sensors / udev expectations.

### hardening

(Inherits row-1 features from `drivers/hwmon/00-overview.md` § Hardening.)

Hwmon-core reinforcement:

- **Per-device mutex serializing every `ops->{read, read_string, write}`** — defense against per-driver-race when sensors use multi-register I2C transactions.
- **Per-attribute callback-presence check at synthesis time** — defense against per-NULL-deref at sysfs-read time (`mode & 0444 ⟹ read[_string] != NULL`; `mode & 0222 ⟹ write != NULL`).
- **Per-`chip == NULL` modern API rejection** — defense against accidental use of deprecated `hwmon_device_register()`; deprecated path emits `dev_warn`.
- **Per-`name` sanitization (`hwmon_is_bad_char`)** — defense against per-sysfs-injection of `-`, `*`, space, tab, newline in attribute paths used by `sensors`.
- **Per-PCI quirk gated to MSI MS-7031** — defense against blanket port-open enabling on unrelated boards.
- **Per-PEC SMBus opt-in** — defense against per-unsupported-adapter false attribute creation; gated on `I2C_FUNC_SMBUS_PEC`.
- **Per-thermal-zone bridge requires `HWMON_C_REGISTER_TZ` + `of_node`** — defense against per-spurious thermal-zone creation on bare-bones drivers.
- **Per-`hwmon_thermal_add_sensor` `-ENODEV` tolerated** — defense against per-OF-mapping-absence aborting whole driver-probe.
- **Per-`is_visible` shortcut (`ops.visible`) restricted to constant mode** — defense against per-driver inconsistency between attributes that should differ.
- **Per-`hwdev->groups` NULL-terminated + `hwmon_dev_release` chain-free** — defense against per-UAF / per-leak on register-failure paths.
- **Per-IDA-allocated stable id** — defense against per-collision in `/sys/class/hwmon/hwmonN` enumeration.
- **Per-`hwmon_lock`/`unlock` exported for drivers extending critical section** — defense against per-driver re-implementing locking poorly (e.g., interleaved register pairs).
- **Per-`device_property_present("label")` honored** — defense against per-firmware-rename invalidating `sensors.conf` labels.

