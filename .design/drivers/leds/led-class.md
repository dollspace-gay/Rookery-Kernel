# Tier-3: drivers/leds/led-class.c — LED class core (sysfs + lifecycle)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/leds/00-overview.md
upstream-paths:
  - drivers/leds/led-class.c (~716 lines)
  - drivers/leds/led-core.c
  - drivers/leds/led-triggers.c
  - drivers/leds/leds.h
  - include/linux/leds.h
-->

## Summary

The **LED class core** owns the `/sys/class/leds/<name>/` lifecycle for every status LED on the system — keyboard caps/num lock, laptop charge/power/disk indicators, server chassis fault LEDs, NIC port-link LEDs, GPIO-driven LEDs, PWM-driven LEDs, I2C/SPI LED controller chips, and smart-bulb gateways. Per-device descriptor: `struct led_classdev` (name, brightness, max_brightness, flags, `brightness_set`/`brightness_set_blocking`/`brightness_get` callbacks, blink ops `blink_set`/`blink_brightness_set`, `set_brightness_work` deferred-update work_struct, `delayed_set_value`, `work_flags` (LED_BLINK_SW / LED_BLINK_ONESHOT / LED_BLINK_INVERT / ...), `led_access` mutex, embedded `dev`, optional `trigger` + `trigger_lock` rw_semaphore, `flash_resume`, `brightness_hw_changed`, `brightness_hw_changed_kn` sysfs_dirent, `wq` workqueue, `groups` attribute_group, `node` list_head). Per-class: `leds_class` registered at module init; `dev_groups = led_groups` provides default attributes (`brightness`, `max_brightness`, conditional `trigger` bin_attr if `CONFIG_LEDS_TRIGGERS`, conditional `brightness_hw_changed` if `CONFIG_LEDS_BRIGHTNESS_HW_CHANGED`). Per-workqueue: `leds_wq = alloc_ordered_workqueue("leds", 0)` shared across all LED classdevs for deferred brightness changes (when caller is atomic context but driver's `brightness_set` may sleep). Per-trigger framework hook: `led_trigger_set_default(led_cdev)` wires `linux,default-trigger` DT/fwnode property at registration; `led_trigger_set(led_cdev, NULL)` at unregister to detach. Per-lookup-table: `led_add_lookup` / `led_remove_lookup` + `led_get(dev, con_id)` for non-DT platforms. Per-devres: `devm_led_classdev_register_ext` / `devm_led_get` / `devm_of_led_get` for automatic cleanup on parent unbind. Critical for: every desktop status LED, every laptop indicator, every server fault diagnostic LED, every embedded indicator.

This Tier-3 covers `drivers/leds/led-class.c` (~716 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct led_classdev` | per-LED descriptor | `LedClassdev` |
| `struct led_init_data` | per-register helper | `LedInitData` |
| `struct led_lookup_data` | per-lookup-table entry | `LedLookupData` |
| `leds_class` | per-sysfs class | `LEDS_CLASS` |
| `leds_wq` | per-class workqueue | `LEDS_WQ` |
| `leds_lookup_lock` / `leds_lookup_list` | per-lookup-table state | `LEDS_LOOKUP_*` |
| `led_classdev_register_ext()` | per-register | `LedClass::register_ext` |
| `led_classdev_unregister()` | per-unregister | `LedClass::unregister` |
| `devm_led_classdev_register_ext()` / `devm_led_classdev_unregister()` | per-devres | `LedClass::devm_register_ext` / `devm_unregister` |
| `led_classdev_suspend()` / `led_classdev_resume()` | per-PM | `LedClass::suspend` / `resume` |
| `led_classdev_next_name()` | per-name-collision | `LedClass::next_name` |
| `led_classdev_notify_brightness_hw_changed()` | per-HW-side notify | `LedClass::notify_brightness_hw_changed` |
| `led_get()` / `led_put()` | per-consumer lookup | `LedClass::get` / `put` |
| `devm_led_get()` / `devm_of_led_get()` / `devm_of_led_get_optional()` | per-devres consumer | `LedClass::devm_get` / `devm_of_get` / `devm_of_get_optional` |
| `of_led_get()` | per-DT consumer | `LedClass::of_get` |
| `led_module_get()` | per-module-pin | `LedClass::module_get` |
| `led_add_lookup()` / `led_remove_lookup()` | per-lookup-table | `LedClass::add_lookup` / `remove_lookup` |
| `brightness_show` / `brightness_store` | per-sysfs RW | `LedClass::brightness_show` / `brightness_store` |
| `max_brightness_show` | per-sysfs RO | `LedClass::max_brightness_show` |
| `brightness_hw_changed_show` | per-sysfs RO | `LedClass::brightness_hw_changed_show` |
| `led_suspend()` / `led_resume()` (PM op) | per-class PM | `LedClass::pm_suspend` / `pm_resume` |
| `leds_init()` / `leds_exit()` | per-module-init | `LedClass::init` / `exit` |

External hooks (drivers/leds/led-core.c + drivers/leds/led-triggers.c):

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `led_init_core()` | per-INIT_WORK on `set_brightness_work` | `LedCore::init_core` |
| `set_brightness_delayed()` | per-deferred-update worker | `LedCore::set_brightness_delayed` |
| `led_set_brightness()` | per-public-setter | `LedCore::set_brightness` |
| `led_set_brightness_nopm()` / `led_set_brightness_nosleep()` | per-context-variant | `LedCore::set_brightness_*` |
| `led_update_brightness()` | per-readback | `LedCore::update_brightness` |
| `led_blink_set()` / `led_blink_set_oneshot()` | per-blink-API | `LedCore::blink_set` / `_oneshot` |
| `led_stop_software_blink()` | per-blink-cancel | `LedCore::stop_software_blink` |
| `led_trigger_set()` | per-attach-trigger | `LedTrigger::set` |
| `led_trigger_set_default()` | per-default-from-fwnode | `LedTrigger::set_default` |
| `led_trigger_remove()` | per-detach-on-manual-brightness | `LedTrigger::remove` |
| `led_trigger_read()` / `led_trigger_write()` | per-trigger bin_attr | `LedTrigger::read` / `write` |
| `led_sysfs_is_disabled()` | per-sysfs-gate | `LedClass::sysfs_is_disabled` |
| `led_compose_name()` | per-name from init_data | `LedCore::compose_name` |

## Compatibility contract

REQ-1: struct led_classdev (ABI-visible subset):
- name: *const u8 — final node name under /sys/class/leds/.
- brightness: u32 — current logical brightness (cache; reconciled by led_update_brightness).
- max_brightness: u32 — driver-declared maximum (LED_FULL=255 default).
- color: u32 — LED_COLOR_ID_WHITE / _RED / _GREEN / _BLUE / _AMBER / _VIOLET / _YELLOW / _IR / _MULTI / _RGB / _PURPLE / _ORANGE / _PINK / _CYAN / _LIME.
- flags: u32 — LED_CORE_SUSPENDRESUME (0x4) | LED_SYSFS_DISABLE (0x100) | LED_DEV_CAP_FLASH | LED_BRIGHT_HW_CHANGED | LED_RETAIN_AT_SHUTDOWN | LED_REJECT_NAME_CONFLICT | LED_UNREGISTERING | LED_SUSPENDED.
- work_flags: u32 — LED_BLINK_SW | LED_BLINK_ONESHOT | LED_BLINK_INVERT | LED_BLINK_BRIGHTNESS_CHANGE | LED_BLINK_DISABLE | LED_SET_BRIGHTNESS_OFF | LED_SET_BRIGHTNESS | LED_SET_BLINK.
- delayed_set_value: u32 — staged brightness for set_brightness_work.
- delayed_delay_on / delayed_delay_off: u64 — blink timings.
- blink_delay_on / blink_delay_off: u64 — current SW blink timings.
- blink_timer: timer_list — SW blink.
- brightness_set: Option<fn(&LedClassdev, u32)> — atomic-safe setter (no sleep).
- brightness_set_blocking: Option<fn(&LedClassdev, u32) -> i32> — may sleep (I2C/SPI).
- brightness_get: Option<fn(&LedClassdev) -> u32>.
- blink_set: Option<fn(&LedClassdev, &mut u64, &mut u64) -> i32> — driver HW blink.
- pattern_set / pattern_clear: Option<fn(...)> — HW pattern engine.
- flash_resume: Option<fn(&LedClassdev)>.
- led_access: Mutex — protects sysfs read/write paths.
- node: ListHead — entry in global `leds_list`.
- trigger: Option<*LedTrigger> — currently-attached trigger.
- trigger_lock: RwSemaphore.
- trigger_data: *mut u8 — per-trigger state pointer.
- default_trigger: *const u8 — fwnode "linux,default-trigger" string.
- brightness_hw_changed: i32 — last HW-reported brightness (-1 if none).
- brightness_hw_changed_kn: *KernfsNode — sysfs_notify target.
- dev: *Device.
- groups: *mut *const AttributeGroup — extra driver-provided attribute groups.
- wq: *Workqueue — shared `leds_wq`.
- set_brightness_work: WorkStruct — INIT_WORKed by led_init_core.

REQ-2: led_classdev_register_ext(parent, led_cdev, init_data):
- Validate init_data:
  - if init_data.devname_mandatory ∧ !init_data.devicename → -EINVAL ("Mandatory device name is missing").
  - led_compose_name(parent, init_data, composed_name) → final name.
  - if init_data.fwnode:
    - fwnode_property_read_string(fwnode, "linux,default-trigger", &led_cdev.default_trigger).
    - if fwnode_property_present("retain-state-shutdown"): led_cdev.flags |= LED_RETAIN_AT_SHUTDOWN.
    - fwnode_property_read_u32(fwnode, "max-brightness", &led_cdev.max_brightness).
    - if fwnode_property_present("color"): fwnode_property_read_u32(fwnode, "color", &led_cdev.color).
- proposed_name = composed_name (or led_cdev.name if no init_data).
- led_classdev_next_name(proposed_name, final_name, LED_MAX_NAME_SIZE) — append "_N" suffix until class_find_device_by_name miss.
  - on collision ∧ LED_REJECT_NAME_CONFLICT → -EEXIST.
  - on rename: dev_warn("Led %s renamed to %s due to name collision").
- if led_cdev.color ≥ LED_COLOR_ID_MAX: dev_warn("LED %s color identifier out of range").
- mutex_init(&led_cdev.led_access); mutex_lock(&led_cdev.led_access).
- led_cdev.dev = device_create_with_groups(&leds_class, parent, MKDEV(0,0), led_cdev, led_cdev.groups, "%s", final_name).
- if IS_ERR(led_cdev.dev): mutex_unlock; return PTR_ERR(led_cdev.dev).
- if init_data ∧ init_data.fwnode: device_set_node(led_cdev.dev, init_data.fwnode).
- if LED_BRIGHT_HW_CHANGED: led_add_brightness_hw_changed(led_cdev) — create brightness_hw_changed sysfs file + sysfs_get_dirent for notify.
- led_cdev.work_flags = 0.
- if CONFIG_LEDS_TRIGGERS: init_rwsem(&led_cdev.trigger_lock).
- if CONFIG_LEDS_BRIGHTNESS_HW_CHANGED: led_cdev.brightness_hw_changed = -1.
- if !led_cdev.max_brightness: led_cdev.max_brightness = LED_FULL (=255).
- led_update_brightness(led_cdev) — readback HW state via brightness_get.
- led_cdev.wq = leds_wq.
- led_init_core(led_cdev) — INIT_WORK(&led_cdev.set_brightness_work, set_brightness_delayed).
- down_write(&leds_list_lock); list_add_tail(&led_cdev.node, &leds_list); up_write.
- if CONFIG_LEDS_TRIGGERS: led_trigger_set_default(led_cdev) — match default_trigger string against registered triggers.
- mutex_unlock(&led_cdev.led_access).
- return 0.

REQ-3: led_classdev_unregister(led_cdev):
- if IS_ERR_OR_NULL(led_cdev.dev): return (idempotent on never-registered).
- if CONFIG_LEDS_TRIGGERS:
  - down_write(&led_cdev.trigger_lock); if led_cdev.trigger: led_trigger_set(led_cdev, NULL); up_write.
- led_cdev.flags |= LED_UNREGISTERING.
- led_stop_software_blink(led_cdev) — del_timer_sync(&blink_timer); clear LED_BLINK_*.
- if !(led_cdev.flags & LED_RETAIN_AT_SHUTDOWN): led_set_brightness(led_cdev, LED_OFF).
- flush_work(&led_cdev.set_brightness_work) — drain pending deferred update.
- if LED_BRIGHT_HW_CHANGED: led_remove_brightness_hw_changed.
- device_unregister(led_cdev.dev).
- down_write(&leds_list_lock); list_del(&led_cdev.node); up_write.
- mutex_destroy(&led_cdev.led_access).

REQ-4: brightness_show / brightness_store:
- brightness_show: mutex_lock(led_access); led_update_brightness(led_cdev); read led_cdev.brightness; mutex_unlock. sysfs_emit("%u\n", brightness).
- brightness_store:
  - mutex_lock(led_access).
  - if led_sysfs_is_disabled(led_cdev): -EBUSY.
  - kstrtoul(buf, 10, &state) — parse decimal u32.
  - if state == LED_OFF (=0): led_trigger_remove(led_cdev) — manual "off" detaches any active trigger.
  - led_set_brightness(led_cdev, state).
  - mutex_unlock; return size.

REQ-5: max_brightness_show:
- mutex_lock(led_access); read max_brightness; mutex_unlock. sysfs_emit("%u\n", max_brightness).

REQ-6: brightness_hw_changed_show (CONFIG_LEDS_BRIGHTNESS_HW_CHANGED):
- if led_cdev.brightness_hw_changed == -1: -ENODATA (never been HW-notified).
- sysfs_emit("%u\n", brightness_hw_changed).

REQ-7: led_classdev_notify_brightness_hw_changed(led_cdev, brightness):
- WARN_ON if !brightness_hw_changed_kn → return.
- led_cdev.brightness_hw_changed = brightness.
- sysfs_notify_dirent(brightness_hw_changed_kn) — wakes poll(2) on the file.

REQ-8: led_classdev_suspend / led_classdev_resume:
- suspend: led_cdev.flags |= LED_SUSPENDED; led_set_brightness_nopm(led_cdev, 0); flush_work(&set_brightness_work).
- resume: led_set_brightness_nopm(led_cdev, led_cdev.brightness); if flash_resume: flash_resume(led_cdev); flags &= ~LED_SUSPENDED.

REQ-9: led_suspend / led_resume (PM ops on the class device):
- LED_CORE_SUSPENDRESUME gates whether class core auto-handles PM.
- Else: driver owns PM.

REQ-10: leds_wq:
- alloc_ordered_workqueue("leds", 0) — single-threaded, deterministic ordering across all LEDs.
- led_cdev.wq = leds_wq at register.

REQ-11: led_init_core(led_cdev):
- INIT_WORK(&led_cdev.set_brightness_work, set_brightness_delayed) — workqueue handler for deferred brightness change.

REQ-12: set_brightness_delayed worker (drivers/leds/led-core.c):
- Inspect led_cdev.work_flags:
  - LED_BLINK_DISABLE: led_stop_software_blink + clear flag.
  - LED_SET_BRIGHTNESS_OFF: set_brightness_delayed_set_brightness(led_cdev, LED_OFF).
  - LED_SET_BRIGHTNESS: set_brightness_delayed_set_brightness(led_cdev, delayed_set_value).
  - LED_SET_BLINK: led_blink_set(led_cdev, delayed_delay_on, delayed_delay_off).
- set_brightness_delayed_set_brightness: try __led_set_brightness (atomic); fall back to __led_set_brightness_blocking on -EAGAIN/-ENOTSUPP.

REQ-13: led_set_brightness (drivers/leds/led-core.c):
- If brightness > max_brightness: clamp.
- If LED_SUSPENDED ∨ LED_UNREGISTERING: store delayed_set_value but skip dispatch.
- If brightness_set is set (atomic-safe) AND not in_interrupt-blocked-by-driver: synchronous led_set_brightness_nosleep.
- Else (need-sleep): set delayed_set_value, set LED_SET_BRIGHTNESS in work_flags, queue_work(led_cdev.wq, &set_brightness_work).

REQ-14: trigger framework hook (CONFIG_LEDS_TRIGGERS):
- led_groups[] includes led_trigger_group (bin_attr_trigger).
- bin_attr_trigger.read = led_trigger_read; write = led_trigger_write.
- led_trigger_write(buf, count) parses trigger name ∈ {"none", "default-on", "timer", "oneshot", "heartbeat", "disk-activity", "mtd", "netdev", "pattern", "transient", ...}; calls led_trigger_set(led_cdev, trig).
- led_trigger_set:
  - down_write(&led_cdev.trigger_lock).
  - If led_cdev.trigger != NULL: trigger.deactivate(led_cdev); led_cdev.trigger = NULL.
  - If new trig != NULL: led_cdev.trigger = trig; trig.activate(led_cdev).
  - up_write.
- led_trigger_set_default: scan registered triggers for default_trigger string; attach.
- led_trigger_remove: led_trigger_set(led_cdev, NULL) — invoked from brightness_store(LED_OFF).

REQ-15: Consumer API (led_get / devm_led_get / devm_of_led_get):
- led_get(dev, con_id):
  - of_led_get(dev.of_node, -1, con_id): look up "led-names" property → index → of_parse_phandle(dev.of_node, "leds", index) → class_find_device_by_fwnode → led_module_get.
  - if -ENOENT and lookup table: scan leds_lookup_list for (dev_id, con_id) match → class_find_device_by_name(provider) → led_module_get.
  - ERR_PTR(-ENOENT) on miss.
- led_module_get(led_dev):
  - dev_get_drvdata(led_dev) → led_cdev.
  - try_module_get(led_cdev.dev.parent.driver.owner); on fail put_device + -ENODEV.
- led_put(led_cdev): module_put + put_device.
- devm_led_get / devm_of_led_get: devres wrapper releasing via led_put on parent unbind.
- devm_of_led_get_optional: if -ENOENT return NULL (LED is optional).

REQ-16: led_add_lookup / led_remove_lookup:
- mutex_lock(&leds_lookup_lock); list_add_tail / list_del(&led_lookup.list, &leds_lookup_list); mutex_unlock.

REQ-17: leds_init / leds_exit:
- init: leds_wq = alloc_ordered_workqueue("leds", 0); class_register(&leds_class).
- exit: class_unregister(&leds_class).
- subsys_initcall(leds_init); module_exit(leds_exit).

REQ-18: Per-name collision policy:
- Default: append "_N" suffix (N starting at 1) until free; warn.
- LED_REJECT_NAME_CONFLICT flag: fail with -EEXIST instead.

## Acceptance Criteria

- [ ] AC-1: /sys/class/leds/<name>/brightness RW (mode 0644); reads sysfs_emit("%u\n", brightness).
- [ ] AC-2: /sys/class/leds/<name>/max_brightness RO (mode 0444).
- [ ] AC-3: Writing "0" to brightness: detaches trigger first (led_trigger_remove) then sets brightness.
- [ ] AC-4: Writing decimal to brightness: clamps to max_brightness; sets via led_set_brightness.
- [ ] AC-5: brightness_store while led_sysfs_is_disabled: -EBUSY.
- [ ] AC-6: led_classdev_register_ext with init_data.devname_mandatory ∧ !devicename: -EINVAL.
- [ ] AC-7: Name collision with LED_REJECT_NAME_CONFLICT: -EEXIST.
- [ ] AC-8: Name collision without flag: append "_N" suffix + dev_warn.
- [ ] AC-9: led_classdev_unregister: trigger detached + blink stopped + (unless LED_RETAIN_AT_SHUTDOWN) brightness=0 + flush_work.
- [ ] AC-10: led_classdev_suspend: brightness 0 via nopm; flush_work; LED_SUSPENDED set.
- [ ] AC-11: led_classdev_resume: brightness restored via nopm; LED_SUSPENDED cleared.
- [ ] AC-12: brightness_hw_changed_show: returns -ENODATA when never notified.
- [ ] AC-13: led_classdev_notify_brightness_hw_changed: sysfs_notify_dirent wakes poll().
- [ ] AC-14: led_get / devm_led_get: pins driver module via try_module_get; led_put / devres release calls module_put.
- [ ] AC-15: CONFIG_LEDS_TRIGGERS: /sys/class/leds/<name>/trigger bin_attr (mode 0644) shows available triggers; writing attaches.
- [ ] AC-16: fwnode "linux,default-trigger" property: attached at register via led_trigger_set_default.

## Architecture

```
struct LedClassdev {
  name: *const u8,
  brightness: u32,
  max_brightness: u32,
  color: u32,
  flags: u32,                              // LED_CORE_SUSPENDRESUME | LED_SYSFS_DISABLE | ...
  work_flags: u32,                         // LED_BLINK_SW | LED_SET_BRIGHTNESS | ...
  delayed_set_value: u32,
  delayed_delay_on: u64,
  delayed_delay_off: u64,
  blink_delay_on: u64, blink_delay_off: u64,
  blink_timer: TimerList,
  brightness_set: Option<fn(&LedClassdev, u32)>,
  brightness_set_blocking: Option<fn(&LedClassdev, u32) -> i32>,
  brightness_get: Option<fn(&LedClassdev) -> u32>,
  blink_set: Option<fn(&LedClassdev, &mut u64, &mut u64) -> i32>,
  pattern_set: Option<fn(...)>, pattern_clear: Option<fn(...)>,
  flash_resume: Option<fn(&LedClassdev)>,
  led_access: Mutex,
  node: ListHead,
  trigger: Option<*LedTrigger>,
  trigger_lock: RwSemaphore,
  trigger_data: *mut u8,
  default_trigger: *const u8,
  brightness_hw_changed: i32,
  brightness_hw_changed_kn: *KernfsNode,
  dev: *Device,
  groups: *mut *const AttributeGroup,
  wq: *Workqueue,                          // = &LEDS_WQ
  set_brightness_work: WorkStruct,
}
```

`LedClass::register_ext(parent, led_cdev, init_data) -> Result<(), Errno>`:
1. /* Validate + compose name */
2. let mut composed_name = [0u8; LED_MAX_NAME_SIZE].
3. let mut final_name = [0u8; LED_MAX_NAME_SIZE].
4. let proposed_name = if let Some(d) = init_data:
   - if d.devname_mandatory ∧ d.devicename.is_none(): return Err(EINVAL).
   - LedCore::compose_name(parent, d, &mut composed_name)?.
   - if let Some(fw) = d.fwnode:
     - fwnode_property_read_string(fw, "linux,default-trigger", &mut led_cdev.default_trigger).ok().
     - if fwnode_property_present(fw, "retain-state-shutdown"): led_cdev.flags |= LED_RETAIN_AT_SHUTDOWN.
     - fwnode_property_read_u32(fw, "max-brightness", &mut led_cdev.max_brightness).ok().
     - if fwnode_property_present(fw, "color"): fwnode_property_read_u32(fw, "color", &mut led_cdev.color).ok().
   - &composed_name.
   - else: led_cdev.name.
5. /* Resolve collisions */
6. let ret = LedClass::next_name(proposed_name, &mut final_name).
7. if ret < 0: return Err(-ret).
8. if ret > 0 ∧ (led_cdev.flags & LED_REJECT_NAME_CONFLICT): return Err(EEXIST).
9. if ret > 0: dev_warn(parent, "Led %s renamed to %s due to name collision").
10. if led_cdev.color ≥ LED_COLOR_ID_MAX: dev_warn(parent, "LED %s color identifier out of range").
11. /* Sysfs device */
12. mutex_init(&led_cdev.led_access).
13. let _g = led_cdev.led_access.lock().
14. led_cdev.dev = device_create_with_groups(&LEDS_CLASS, parent, MKDEV(0,0), led_cdev as *mut u8, led_cdev.groups, "%s", &final_name)?.
15. if let Some(fw) = init_data.and_then(|d| d.fwnode): device_set_node(led_cdev.dev, fw).
16. /* HW-changed sysfs */
17. if led_cdev.flags & LED_BRIGHT_HW_CHANGED: LedClass::add_brightness_hw_changed(led_cdev)?.
18. /* Init */
19. led_cdev.work_flags = 0.
20. if CONFIG_LEDS_TRIGGERS: init_rwsem(&led_cdev.trigger_lock).
21. if CONFIG_LEDS_BRIGHTNESS_HW_CHANGED: led_cdev.brightness_hw_changed = -1.
22. if led_cdev.max_brightness == 0: led_cdev.max_brightness = LED_FULL.
23. LedCore::update_brightness(led_cdev).
24. led_cdev.wq = &LEDS_WQ.
25. LedCore::init_core(led_cdev). // INIT_WORK(set_brightness_work, set_brightness_delayed).
26. /* Global list */
27. let _l = LEDS_LIST_LOCK.write().
28. LEDS_LIST.push_back(&led_cdev.node).
29. drop(_l).
30. /* Default trigger */
31. if CONFIG_LEDS_TRIGGERS: LedTrigger::set_default(led_cdev).
32. drop(_g).
33. dev_dbg(parent, "Registered led device: %s", led_cdev.name).
34. Ok(()).

`LedClass::unregister(led_cdev)`:
1. if IS_ERR_OR_NULL(led_cdev.dev): return.
2. /* Detach trigger */
3. if CONFIG_LEDS_TRIGGERS:
   - let _t = led_cdev.trigger_lock.write().
   - if led_cdev.trigger.is_some(): LedTrigger::set(led_cdev, None).
   - drop(_t).
4. led_cdev.flags |= LED_UNREGISTERING.
5. /* Stop SW blink */
6. LedCore::stop_software_blink(led_cdev). // del_timer_sync.
7. /* Default-off */
8. if !(led_cdev.flags & LED_RETAIN_AT_SHUTDOWN): LedCore::set_brightness(led_cdev, LED_OFF).
9. /* Drain deferred */
10. flush_work(&led_cdev.set_brightness_work).
11. /* HW-changed sysfs */
12. if led_cdev.flags & LED_BRIGHT_HW_CHANGED: LedClass::remove_brightness_hw_changed(led_cdev).
13. /* Sysfs device */
14. device_unregister(led_cdev.dev).
15. /* Global list */
16. let _l = LEDS_LIST_LOCK.write().
17. LEDS_LIST.remove(&led_cdev.node).
18. drop(_l).
19. mutex_destroy(&led_cdev.led_access).

`LedClass::brightness_store(dev, attr, buf, size) -> Result<usize, Errno>`:
1. led_cdev = dev_get_drvdata(dev).
2. let _g = led_cdev.led_access.lock().
3. if led_sysfs_is_disabled(led_cdev): return Err(EBUSY).
4. let state = kstrtoul(buf, 10)?.
5. if state == LED_OFF (=0): LedTrigger::remove(led_cdev). // detach trigger first.
6. LedCore::set_brightness(led_cdev, state).
7. Ok(size).

`LedClass::next_name(init_name, name, len) -> Result<u32, Errno>`:
1. let mut i = 0u32.
2. let mut ret = 0i32.
3. strscpy(name, init_name, len).
4. while ret < len ∧ class_find_device_by_name(&LEDS_CLASS, name).is_some():
   - put_device(dev).
   - i += 1.
   - ret = snprintf(name, len, "%s_%u", init_name, i).
5. if ret ≥ len: return Err(ENOMEM).
6. Ok(i).

`LedClass::module_get(led_dev) -> Result<*LedClassdev, Errno>`:
1. if led_dev.is_null(): return Err(EPROBE_DEFER).
2. led_cdev = dev_get_drvdata(led_dev).
3. if !try_module_get(led_cdev.dev.parent.driver.owner):
   - put_device(led_cdev.dev).
   - return Err(ENODEV).
4. Ok(led_cdev).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `name_unique_per_class` | INVARIANT | per-register: no two led_classdev share the same /sys/class/leds/<name> (next_name appends until unique). |
| `led_access_held_for_brightness_sysfs` | INVARIANT | per-brightness_show / _store: led_access mutex held during read / write. |
| `unregister_drains_deferred_work` | INVARIANT | per-unregister: flush_work returns before device_unregister returns. |
| `unregister_detaches_trigger` | INVARIANT | per-unregister: led_cdev.trigger == NULL after return. |
| `suspend_then_resume_restores_brightness` | INVARIANT | per-PM: suspend(b)→resume(): brightness == b on resume. |
| `module_get_balanced_by_put` | INVARIANT | per-led_get: try_module_get balanced by led_put's module_put. |
| `lookup_list_held_under_mutex` | INVARIANT | per-add_lookup / remove_lookup: leds_lookup_lock held. |
| `brightness_clamped_to_max` | INVARIANT | per-set_brightness: caller never observes brightness > max_brightness. |
| `unregister_idempotent_on_null_dev` | INVARIANT | per-unregister with IS_ERR_OR_NULL(dev): early return. |

### Layer 2: TLA+

`drivers/leds/led-class.tla`:
- Per-register + per-unregister + per-brightness-set + per-trigger-attach + per-suspend/resume.
- Properties:
  - `safety_unique_name_per_class` — per-/sys/class/leds: no two devices with identical name at any state.
  - `safety_no_brightness_after_unregister` — per-LedClassdev: state ∈ {Unregistered, Registered}; brightness_set forbidden in Unregistered.
  - `safety_trigger_detached_before_unregister_returns` — per-unregister: trigger == None on return.
  - `safety_deferred_work_drained` — per-unregister: set_brightness_work flushed before device_unregister.
  - `safety_module_refcount_balanced` — per-led_get + led_put: module ref non-negative; reaches 0 only after all puts.
  - `liveness_register_completes` — per-register: success or error; no hang.
  - `liveness_set_brightness_eventually_applied` — per-set_brightness in non-suspended state: brightness reaches HW (possibly via deferred worker).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `LedClass::register_ext` post: success ⟹ led_cdev.dev != NULL ∧ name in LEDS_LIST ∧ work_flags == 0 ∧ wq == &LEDS_WQ | `LedClass::register_ext` |
| `LedClass::unregister` post: dev unregistered ∧ trigger None ∧ blink stopped ∧ work flushed ∧ removed from LEDS_LIST | `LedClass::unregister` |
| `LedClass::brightness_store` post: state ≤ max_brightness ⟹ led_cdev.brightness eventually == state (via set_brightness_work if blocking) | `LedClass::brightness_store` |
| `LedClass::suspend` post: brightness == 0 (HW) ∧ work flushed ∧ LED_SUSPENDED set | `LedClass::suspend` |
| `LedClass::resume` post: brightness == pre-suspend value (HW) ∧ LED_SUSPENDED cleared | `LedClass::resume` |
| `LedClass::module_get` post: success ⟹ try_module_get'd; failure ⟹ put_device called + no leak | `LedClass::module_get` |

### Layer 4: Verus/Creusot functional

`Per-register → device_create_with_groups → sysfs brightness/max_brightness/trigger appear → led_init_core INIT_WORK → led_trigger_set_default → leds_list insertion → consumer (sysfs writer or trigger) sees node` semantic equivalence: per-Documentation/leds/leds-class.rst.

`Per-brightness_store("N") → led_trigger_remove (if N==0) → led_set_brightness → either nosleep direct or queue_work → set_brightness_delayed → brightness reaches HW` semantic equivalence: per-user-space-writes to /sys/class/leds/<name>/brightness in upstream traces.

## Hardening

(Inherits row-1 features from `drivers/leds/00-overview.md` § Hardening.)

LED-class reinforcement:

- **Per-led_access mutex around every sysfs RW** — defense against per-concurrent-store data race.
- **Per-trigger detach before unregister** — defense against per-trigger-callback-into-freed-classdev UAF.
- **Per-flush_work before device_unregister** — defense against per-deferred-work-into-freed-classdev UAF.
- **Per-LED_REJECT_NAME_CONFLICT for safety-critical LEDs** — defense against per-second-driver hijacking by name.
- **Per-led_classdev_next_name suffix + length check** — defense against per-truncation-into-collision.
- **Per-led_set_brightness clamp to max_brightness** — defense against per-overdrive damaging LED.
- **Per-LED_OFF detaches trigger** — defense against per-user-thinks-LED-is-off-but-trigger-re-enables confusion.
- **Per-try_module_get on led_get** — defense against per-driver-module-unload-while-consumer-holds.
- **Per-led_sysfs_is_disabled gate** — defense against per-sysfs-write-during-driver-bringup.
- **Per-LED_UNREGISTERING flag stops new dispatches** — defense against per-race in flight at unregister.
- **Per-LED_RETAIN_AT_SHUTDOWN honored** — defense against per-power-LED-flicker on reboot.
- **Per-leds_wq ordered single-thread** — defense against per-out-of-order brightness applies (later "off" passing earlier "on").
- **Per-brightness_hw_changed_kn refcount via sysfs_get_dirent** — defense against per-sysfs_notify on freed dirent.

## Grsecurity/PaX-style Reinforcement

Rationale: LED-class sysfs is one of the most user-facing kernel attribute surfaces — `brightness`, `trigger`, and `brightness_hw_changed` are commonly chmod-permissive to allow desktop status indicators. That makes the parse/dispatch path and trigger-attach allowlist a classic vector for unprivileged code to feed arbitrary strings through `kstrtoul` and `led_trigger_set` into per-driver callbacks. The reinforcement below restates baseline PaX/grsec coverage applied to `drivers/leds/led-class.c` plus LED-class-specific reinforcement.

Baseline (cross-ref `drivers/leds/00-overview.md` § Hardening):
- **PAX_USERCOPY**: per-`brightness_store` / `trigger_write` source buffers are sysfs-attribute PAGE-allocated and copy via `kstrtoul` / `sysfs_emit` — both whitelisted by USERCOPY against the slab cache they live in.
- **PAX_KERNEXEC**: `leds_class.dev_groups`, `led_groups`, and per-driver `groups` placed in `__ro_after_init`; `brightness_set` / `blink_set` fn pointers fixed at register time.
- **PAX_RANDKSTACK**: every sysfs entry (`brightness_store`, `trigger_write`, `brightness_hw_changed_show`) re-randomises stack offset before mutex acquisition.
- **PAX_REFCOUNT**: per-`led_classdev.dev` kobject refcount saturating; per-`try_module_get` count saturating; defense against `led_get`/`led_put` imbalance underflow.
- **PAX_MEMORY_SANITIZE**: `device_release` on `led_cdev.dev` zero-fills slab; per-`trigger_data` pointer (driver-private trigger state) zeroed on `led_trigger_set(NULL)` so a stale trigger pointer cannot survive into a later attach.
- **PAX_UDEREF**: sysfs `store` callbacks read user pages through `sysfs_kf_write` which already enforces UDEREF; no raw user deref in `led-class.c`.
- **PAX_RAP / kCFI**: `brightness_set`, `brightness_set_blocking`, `brightness_get`, `blink_set`, `pattern_set`, `flash_resume`, `trigger->activate`/`deactivate` indirect calls are kCFI-tagged with signature checks.
- **GRKERNSEC_HIDESYM**: `dev_dbg` / `dev_warn` from led-class emit only kobject name, never `&led_cdev`; `%pK` for any remaining ptr render.
- **GRKERNSEC_DMESG**: rename / collision warnings ratelimited per-class; `dmesg` access requires CAP_SYSLOG.

LED-class-specific reinforcement:
- **CAP_SYS_ADMIN for brightness write** — grsec policy elevates `/sys/class/leds/<name>/brightness` from default `0644` to `0640` (root+leds-group only); refuses unprivileged write via `led_sysfs_is_disabled` extension. Defends against unprivileged keyboard-LED covert channels and indicator spoofing.
- **Trigger-name allowlist** — `led_trigger_write` compares incoming name against a compile-time allowlist (`none`, `default-on`, `timer`, `oneshot`, `heartbeat`, `disk-activity`, `netdev`, `pattern`, `transient`); refuses arbitrary strings even if a malicious module registered them. Defends against rogue-trigger-registration → privileged callback hijack.
- **`led_classdev_register_ext` rejects fwnode-supplied `max-brightness` > 65535** — clamp + warn; defends against fwnode-driven u32 wrap into clamp logic.
- **`LED_REJECT_NAME_CONFLICT` default-on for safety-critical color classes** — grsec policy auto-applies the flag for `LED_COLOR_ID_RED` / `_AMBER` so a non-fault-status LED cannot squat the "fault" name. Defends against operator misreading status.
- **`led_get` consumer audited** — `led_module_get` rejects when callee module is not in the same userns or lacks CAP_SYS_ADMIN at registration; defends against unprivileged module pinning the LED driver to block unbind.
- **`brightness_hw_changed_kn` sysfs_get_dirent + drop on unregister** — already in row-2 hardening; extended with grsec audit log on sysfs_notify after `LED_UNREGISTERING` (caught race window).
- **`led_trigger_set` write-lock barrier before `activate`** — guarantees `trigger_data` cannot be observed half-initialised by a racing `brightness_store`.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `drivers/leds/led-core.c` set_brightness machinery + SW blink timer (covered in `drivers/leds/core.md` companion)
- `drivers/leds/led-triggers.c` trigger framework internals (covered in `drivers/leds/triggers.md`)
- `drivers/leds/led-class-flash.c` flash/torch class (covered in `drivers/leds/flash-multicolor.md`)
- `drivers/leds/led-class-multicolor.c` RGB/multicolor class (covered in `drivers/leds/flash-multicolor.md`)
- Per-vendor LED controller drivers (covered in `drivers/leds/per-controller.md`)
- Implementation code
