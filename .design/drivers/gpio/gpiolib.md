# Tier-3: drivers/gpio/gpiolib.c — GPIO descriptor library / gpio_chip framework

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/gpio/00-overview.md
upstream-paths:
  - drivers/gpio/gpiolib.c (~5528 lines)
  - drivers/gpio/gpiolib.h
  - drivers/gpio/gpiolib-cdev.c (chardev for /dev/gpiochipN)
  - include/linux/gpio/driver.h (struct gpio_chip + struct gpio_irq_chip)
  - include/linux/gpio/consumer.h (gpiod_* API + enum gpiod_flags)
  - include/linux/gpio/machine.h (struct gpiod_lookup_table)
  - include/uapi/linux/gpio.h (GPIO_V2_LINE_*, struct gpio_v2_line_request)
-->

## Summary

The GPIO core (gpiolib) provides three coupled abstractions:

- **Driver side** — `struct gpio_chip` (per-controller ops: request / free / get_direction / direction_input / direction_output / get / set / set_config / to_irq / dbg_show / can_sleep) wrapped by `struct gpio_device` (registered Linux device with id, descriptor array, SRCU, label, ngpio, base).
- **Consumer side** — `struct gpio_desc` (per-line opaque handle carrying flags, hwgpio offset, gpio_device backref, label, debounce_period_us, hog backref) returned by `gpiod_get*` family, freed by `gpiod_put*`.
- **Userspace side** — `/dev/gpiochipN` character device (gpiolib-cdev) speaking the GPIO uAPI V2 ioctls (`GPIO_V2_GET_LINEINFO_IOCTL`, `GPIO_V2_GET_LINE_IOCTL`, `GPIO_V2_LINE_SET_VALUES_IOCTL`, etc.) plus line-state notifications.

Lifecycle: a controller driver allocates `gc`, fills in `gc->ngpio`, `gc->parent`, `gc->label`, function pointers, and calls `gpiochip_add_data(gc, drvdata)` (which expands to `gpiochip_add_data_with_key`). This allocates a `gpio_device`, assigns a base (dynamic if `gc->base < 0`), inserts into the global `gpio_devices` SRCU-protected list, allocates `gdev->descs[ngpio]`, builds valid_mask, registers OF / ACPI fwnode bindings, installs the IRQ chip (if `gc->irq.chip` set), processes hog lines (`gpio-hog` from DTS), then registers the cdev and the sysfs device. Teardown is `gpiochip_remove(gc)`: refuses to remove a chip with any in-use descriptors, tears down hogs / irqchip / pin ranges / valid_mask / debugfs / cdev / sysfs / SRCU.

Consumers fetch lines via fwnode (`gpiod_get(dev, con_id, GPIOD_OUT_LOW)` walks DT `con_id-gpios` → `gpiochip_get_desc` → `gpiod_request_commit` → `gpiod_configure_flags`) or via machine `gpiod_lookup_table`. `enum gpiod_flags` selects initial direction + value + open-drain (GPIOD_IN, GPIOD_OUT_LOW, GPIOD_OUT_HIGH, GPIOD_OUT_LOW_OPEN_DRAIN, GPIOD_OUT_HIGH_OPEN_DRAIN, GPIOD_ASIS). Per-line descriptor flags (`GPIOD_FLAG_*` bitmap inside `desc->flags`) track REQUESTED, IS_OUT, ACTIVE_LOW, OPEN_DRAIN, OPEN_SOURCE, PULL_UP, PULL_DOWN, BIAS_DISABLE, EDGE_RISING, EDGE_FALLING, IS_HOGGED, USED_AS_IRQ, TRANSITORY.

IRQ routing: a GPIO line acts as an interrupt source when `gc->irq.chip` is registered; `gpiochip_to_irq(gc, offset)` (assigned to `gc->to_irq` by `gpiochip_add_irqchip`) does either `irq_create_mapping(domain, offset)` for flat domains or `irq_create_fwspec_mapping` for hierarchical ones. Consumers call `gpiod_to_irq(desc)`. Per-IRQ resource management uses `gpiochip_lock_as_irq` to prevent the same hwgpio being driven as output while wired to an IRQ.

This Tier-3 covers `drivers/gpio/gpiolib.c` (~5528 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct gpio_chip` | per-controller driver ops + ngpio + base | `GpioChip` |
| `struct gpio_device` | per-registered chip device + descs[] + SRCU | `GpioDevice` |
| `struct gpio_desc` | per-line opaque handle (flags + offset + gdev) | `GpioDesc` |
| `struct gpio_descs` | per-array result of gpiod_get_array | `GpioDescs` |
| `struct gpio_irq_chip` | per-chip irqchip embedded info | `GpioIrqChip` |
| `struct gpiod_lookup_table` | machine-side lookup | `GpiodLookupTable` |
| `gpiochip_add_data` / `_add_data_with_key` | per-register chip | `Gpio::chip_add_data` |
| `gpiochip_remove` | per-unregister | `Gpio::chip_remove` |
| `gpiochip_find_base_unlocked` | per-dynamic-base alloc | `Gpio::find_base` |
| `gpiochip_setup_dev` | per-cdev + sysfs install | `Gpio::setup_dev` |
| `gpiodev_release` | per-final put | `Gpio::release` |
| `gpio_to_desc` / `gpiochip_get_desc` | per-(gpio,offset) → desc | `Gpio::to_desc` |
| `desc_to_gpio` / `gpiod_hwgpio` | per-desc → number / hwgpio | `Gpio::desc_to_gpio` |
| `gpiod_request` / `gpiod_request_commit` | per-claim line | `Gpio::request` |
| `gpiod_free` / `gpiod_free_commit` | per-release line | `Gpio::free` |
| `gpiod_get` / `gpiod_get_index` / `gpiod_get_optional` / `gpiod_get_array` | per-consumer fwnode-resolve + request | `Gpio::get` |
| `gpiod_put` / `gpiod_put_array` | per-consumer release | `Gpio::put` |
| `gpiod_configure_flags` | per-apply gpiod_flags after request | `Gpio::configure_flags` |
| `gpiod_direction_input` / `_output` / `_output_raw` | per-set direction | `Gpio::direction_input` / `output` |
| `gpiod_get_value` / `_raw_value` / `_array_value` | per-read value (active-low aware) | `Gpio::get_value` |
| `gpiod_set_value` / `_raw_value` / `_array_value` | per-write value | `Gpio::set_value` |
| `gpiod_set_config` / `gpio_set_bias` / `gpio_set_debounce_timeout` | per-pinctrl config | `Gpio::set_config` |
| `gpiod_to_irq` | per-desc → irq | `Gpio::to_irq` |
| `gpiochip_to_irq` (gc->to_irq) | per-chip → irq | `Gpio::chip_to_irq` |
| `gpiochip_lock_as_irq` / `_unlock_as_irq` | per-line-as-IRQ resource lock | `Gpio::lock_as_irq` |
| `gpiochip_irqchip_add_domain` / `gpiochip_add_irqchip` | per-irqchip register | `Gpio::irqchip_add` |
| `gpiochip_add_hog` / `gpiochip_hog_lines` / `gpiod_hog` | per-hog DT-declared lines | `Gpio::hog` |
| `gpiochip_free_hogs` | per-release hogs | `Gpio::free_hogs` |
| `gpiod_add_lookup_table` / `_remove_lookup_table` | per-machine lookup register | `Gpio::lookup_register` |
| `gpiod_count` | per-DT-count function lines | `Gpio::count` |
| `gpiochip_line_is_valid` / `gpiochip_query_valid_mask` | per-valid_mask query | `Gpio::line_is_valid` |
| `gpiod_line_state_notify` | per-cdev state-change notifier | `Gpio::line_state_notify` |
| `gpio_device_find` / `_find_by_label` / `_find_by_fwnode` | per-iterate registered chips | `Gpio::device_find` |

## Compatibility contract

REQ-1 — `struct gpio_chip` driver-supplied fields (sufficient for register):
- `label`: free-form string.
- `parent`: backing `struct device` (NULL for early static chips).
- `owner`: THIS_MODULE (or inherited from `parent->driver->owner`).
- `ngpio`: count of lines (>0).
- `base`: starting Linux GPIO number; <0 ⟹ dynamic via `gpiochip_find_base_unlocked(ngpio)`.
- `names[ngpio]`: optional per-line names (also overridable by fwnode `gpio-line-names`).
- Direction ops: `direction_input(gc, off)`, `direction_output(gc, off, val)`, `get_direction(gc, off)` (returns 1 for input, 0 for output).
- I/O ops: `get(gc, off)`, `set(gc, off, val)` and optional vectorised `get_multiple` / `set_multiple`.
- Config: `set_config(gc, off, config)` consuming `pin_config_param` packing.
- Lifecycle: `request(gc, off)` / `free(gc, off)`.
- IRQ glue (optional): `to_irq(gc, off)` set by gpiolib if `gc->irq.chip` provided.
- `can_sleep`: true ⟹ I/O may sleep (I²C/SPI expanders); _cansleep variants required.
- `init_valid_mask(gc, mask, ngpio)`: per-controller hardware-validity mask.

REQ-2 — `struct gpio_device` (allocated by gpiolib, not the driver):
- `id`: ida-allocated index → `gpiochipN` name.
- `dev`: embedded `struct device` (class `gpio_class`, bus `gpio_bus_type`).
- `chrdev`: cdev of `/dev/gpiochipN` (gpio_devt is the registered chrdev region).
- `chip`: RCU-protected pointer to caller's `gpio_chip` (cleared on remove).
- `descs[ngpio]`: array of `gpio_desc` (one per hwgpio).
- `label`: kstrdup of `gc->label`.
- `ngpio`, `base`, `can_sleep`: snapshotted from `gc`.
- `srcu` + `desc_srcu`: per-device SRCU domains (chip-pointer / desc-array).
- `line_state_lock`: rwlock guarding desc flags during state-change notifications.
- `line_state_notifier` / `device_notifier`: notifier chains for cdev + uevent.
- `pin_ranges` (CONFIG_PINCTRL): pinctrl ranges registered via `gpiochip_add_pin_range*`.
- `owner`: module owning the chip (refcounted on `gpiod_request`).

REQ-3 — `struct gpio_desc->flags` (atomic bitmap, accessed via test_bit / set_bit / clear_bit):
- `GPIOD_FLAG_REQUESTED` — set by `gpiod_request_commit` (test_and_set guard).
- `GPIOD_FLAG_IS_OUT` — direction = output (0 ⟹ input).
- `GPIOD_FLAG_ACTIVE_LOW` — logical inversion of get/set.
- `GPIOD_FLAG_OPEN_DRAIN` — drive low only; release for high.
- `GPIOD_FLAG_OPEN_SOURCE` — drive high only; release for low.
- `GPIOD_FLAG_PULL_UP` / `_PULL_DOWN` / `_BIAS_DISABLE` — bias config.
- `GPIOD_FLAG_TRANSITORY` — line not preserved across reset/suspend.
- `GPIOD_FLAG_USED_AS_IRQ` — set by `gpiochip_lock_as_irq`; blocks output writes.
- `GPIOD_FLAG_IS_HOGGED` — set by `gpiod_hog`; freed at chip-remove.
- `GPIOD_FLAG_EDGE_RISING` / `_FALLING` — cdev edge config.
- `GPIOD_FLAG_SLEEP_MAY_LOSE_VALUE` — informational.

REQ-4 — `enum gpiod_flags` (consumer init flags passed to `gpiod_get*`):
- `GPIOD_ASIS` = 0 — keep current direction/value.
- `GPIOD_IN` = `GPIOD_FLAGS_BIT_DIR_SET` — force input.
- `GPIOD_OUT_LOW` = `GPIOD_FLAGS_BIT_DIR_SET | GPIOD_FLAGS_BIT_DIR_OUT` — output driven low.
- `GPIOD_OUT_HIGH` = `... | GPIOD_FLAGS_BIT_DIR_VAL` — output driven high.
- `GPIOD_OUT_LOW_OPEN_DRAIN` / `_HIGH_OPEN_DRAIN` — output with open-drain.

Bit-set semantics: bit `DIR_SET` = "set direction"; `DIR_OUT` = "output (else input)"; `DIR_VAL` = "high (else low)"; `OPEN_DRAIN` = "open-drain"; `OPEN_SOURCE` = "open-source"; `TRANSITORY` / `BIAS_*` extensions are or-ed in by `gpiod_configure_flags` from lookup-table `flags`.

REQ-5 — `gpiochip_add_data_with_key(gc, data, lock_key, request_key)` (1137-1368):
- Allocate `gdev` (kzalloc, GFP_KERNEL); fail ⟹ -ENOMEM.
- `gc->gpiodev = gdev`; `gpiochip_set_data(gc, data)` (stores into `gc->data`).
- `gdev->id = ida_alloc(&gpio_ida)`; init `gdev->srcu` and `gdev->desc_srcu`.
- `rcu_assign_pointer(gdev->chip, gc)`.
- `dev_set_name(&gdev->dev, "gpiochip%d", gdev->id)`; `device_initialize`.
- `gdev->dev.type = &gpio_dev_type`; `bus = &gpio_bus_type`; `parent = gc->parent`.
- `device_set_node(&gdev->dev, gpiochip_choose_fwnode(gc))`.
- `gpiochip_get_ngpios(gc, &gdev->dev)` resolves `gc->ngpio` (may read `ngpios` DT property if 0).
- `gdev->descs = kcalloc(gc->ngpio)`; `gdev->label = kstrdup_const(gc->label ?: "unknown")`.
- `gdev->can_sleep = gc->can_sleep`.
- Init notifier heads (`line_state_notifier`, `device_notifier`), `pin_ranges` list.
- Choose owner: `gc->parent->driver->owner` else `gc->owner` else `THIS_MODULE`.
- Under `gpio_devices_lock`: resolve `base` (`gpiochip_find_base_unlocked(gc->ngpio)` if <0); `gdev->base = base`; `gpiodev_add_to_list_unlocked(gdev)` ensures no integer-space overlap.
- `gpiochip_set_desc_names(gc)` if `gc->names`; `gpiochip_set_names(gc)` parses fwnode.
- `gpiochip_init_valid_mask(gc)` — allocates `gc->valid_mask` via driver `init_valid_mask` callback.
- For each desc: `desc->gdev = gdev`; initial IS_OUT bit derived from `gc->get_direction` or `gc->direction_input` presence.
- `of_gpiochip_add(gc)` registers OF binding; `gpiochip_add_pin_ranges`; `acpi_gpiochip_add`.
- `gpiochip_hog_lines(gc)` claims any DT `gpio-hog` children.
- `gpiochip_irqchip_init_valid_mask` / `_init_hw` / `gpiochip_add_irqchip(gc, lock_key, request_key)` if `gc->irq.chip` set.
- `gpiochip_setup_shared` (for pinctrl shared use).
- If `gpiolib_initialized`: `gpiochip_setup_dev(gc)` creates cdev + adds device. Otherwise deferred to `gpiolib_dev_init` core_initcall.
- Unwind labels (`err_*`) tear down in exact reverse order; on failure other than -EPROBE_DEFER, pr_err with "GPIOs base..base+ngpio-1 failed to register".

REQ-6 — `gpiochip_remove(gc)` (1376-1413):
- `gdev = gc->gpiodev`.
- `gpio_device_teardown_shared(gdev)`; `gpiochip_irqchip_remove(gc)`; `gpiochip_irqchip_free_valid_mask`.
- `gpiochip_free_hogs(gc)`; `acpi_gpiochip_remove`; `gpiochip_remove_pin_ranges`; `of_gpiochip_remove`; `gpiochip_free_valid_mask`.
- Under `gpio_devices_lock`: `list_del_rcu(&gdev->list)`; `synchronize_srcu(&gpio_devices_srcu)`.
- `gpiochip_free_remaining_irqs(gc)` warns if any line still locked as IRQ.
- `rcu_assign_pointer(gdev->chip, NULL)`; `synchronize_srcu(&gdev->srcu)`.
- `device_del(&gdev->dev)` if registered.
- `gpio_device_put(gdev)` drops the final refcount; `gpiodev_release` frees descs / label / ida / srcu.

REQ-7 — `gpiod_request(desc, label)` (2597-2615):
- `VALIDATE_DESC(desc)` (rejects NULL / ERR_PTR / invalid index; returns -EINVAL).
- `try_module_get(desc->gdev->owner)` ⟹ failure returns -EPROBE_DEFER.
- `gpiod_request_commit(desc, label)`:
  - `CLASS(gpio_chip_guard, guard)(desc)` SRCU-locks the chip pointer; NULL ⟹ -ENODEV.
  - `test_and_set_bit(GPIOD_FLAG_REQUESTED)` — already set ⟹ -EBUSY.
  - `gpiochip_line_is_valid(guard.gc, offset)` — invalid hardware mask ⟹ -EINVAL.
  - Optional `gc->request(gc, offset)` callback; >0 normalised to -EBADE; nonzero ⟹ failure.
  - `gpiod_get_direction(desc)` if `gc->get_direction` exists.
  - `desc_set_label(desc, label ?: "?")` stores label via `kasprintf` under RCU.
- On success: `gpio_device_get(desc->gdev)` (gdev refcount up).
- On failure: `module_put(owner)`; pr_debug "%s: status %d".

REQ-8 — `gpiod_free(desc)` / `gpiod_free_commit(desc)` (2617-2660):
- `might_sleep()` (debounce_period_us / state notify may sleep).
- Under `gpio_chip_guard`: if REQUESTED bit set:
  - `gc->free(gc, hwgpio)` if provided.
  - Clear flags: ACTIVE_LOW, REQUESTED, OPEN_DRAIN, OPEN_SOURCE, PULL_UP, PULL_DOWN, BIAS_DISABLE, EDGE_RISING, EDGE_FALLING, IS_HOGGED.
  - `desc->hog = NULL` under CONFIG_OF_DYNAMIC.
  - `desc_set_label(desc, NULL)`; `WRITE_ONCE(desc->flags, flags)`; clear debounce.
  - `gpiod_line_state_notify(desc, GPIO_V2_LINE_CHANGED_RELEASED)`.
- `module_put(owner)`; `gpio_device_put(gdev)`.

REQ-9 — `gpiod_get(dev, con_id, gpiod_flags)` consumer path:
- `gpiod_find_and_request(dev, NULL, con_id, 0, flags, label, optional=false)` (4756):
  - `gpiod_fwnode_lookup(dev_fwnode(dev), con_id, 0, &lflags, &dflags)` walks DT (`<con_id>-gpios`, `gpios`, `<con_id>-gpio`) or ACPI.
  - Falls back to `gpiod_find(dev, con_id, ...)` machine lookup_table on miss.
  - `gpiod_request(desc, label)`; `gpiod_configure_flags(desc, con_id, lflags, dflags)`.
- `gpiod_configure_flags` (4964) applies ACTIVE_LOW / OPEN_DRAIN / OPEN_SOURCE / PULL_* / TRANSITORY then `gpiod_direction_input`/`_output*` from `dflags`.

REQ-10 — `gpiod_direction_input(desc)` / `_output_raw(desc, value)`:
- `gpiochip_direction_input(gc, off)` ⟹ `gc->direction_input(gc, off)`; clear IS_OUT.
- `gpiochip_direction_output(gc, off, val)` ⟹ honors OPEN_DRAIN / OPEN_SOURCE: open-drain ⟹ either `direction_output(low)` or `direction_input` (release); open-source mirror; otherwise `gc->direction_output(gc, off, val)`; set IS_OUT.
- `gpiod_direction_output(desc, val)` applies ACTIVE_LOW inversion to `val` then forwards to `_raw`.

REQ-11 — `gpiod_get_value(desc)` / `gpiod_set_value(desc, val)`:
- `gpio_chip_get_value(gc, desc)` returns raw 0/1; `gpiod_get_value` XORs with ACTIVE_LOW bit.
- `gpiod_set_value` XORs with ACTIVE_LOW then dispatches:
  - OPEN_DRAIN ⟹ `gpio_set_open_drain_value_commit`.
  - OPEN_SOURCE ⟹ `gpio_set_open_source_value_commit`.
  - Else ⟹ `gpiod_set_raw_value_commit` ⟹ `gpiochip_set(gc, off, val)`.
- `_cansleep` variants assert `gc->can_sleep` permitted and may block; non-`_cansleep` variants WARN_ON if `gc->can_sleep` is true.

REQ-12 — IRQ mapping (`gpiod_to_irq(desc)` / `gpiochip_to_irq(gc, off)`):
- `gpiod_to_irq(desc)` (4127): `gc->to_irq(gc, hwgpio)` if non-NULL; else `irq_find_mapping(gc->irq.domain, hwgpio)`; else -ENXIO.
- `gpiochip_to_irq(gc, off)` (2016): wait for `gc->irq.initialized` (else -EPROBE_DEFER); reject if `!gpiochip_irqchip_irq_valid(gc, off)` ⟹ -ENXIO; hierarchical domain ⟹ `irq_create_fwspec_mapping(&spec)`; flat ⟹ `irq_create_mapping(domain, offset)`.
- `gpiochip_lock_as_irq(gc, off)` enforces:
  - Line must be input or open-drain/open-source (not driving), or REQUESTED unset, before being usable as IRQ.
  - Sets USED_AS_IRQ; pairs with `gpiochip_unlock_as_irq` clearing it.
- `gpiochip_disable_irq` / `_enable_irq` toggle the masking bit; `gpiochip_irq_reqres` / `_relres` route through `gpiochip_reqres_irq` / `_relres_irq` which take `gpiod_request` on demand so the descriptor is reserved while the IRQ is mapped.

REQ-13 — Hogs (`gpio-hog` DTS / `gpiod_hog`):
- `gpiochip_add_hog(gc, fwnode)` (936) creates a synthetic `gpio_desc` from a child node carrying `gpios` + (`input` / `output-low` / `output-high` / `line-name` / `active_low` / `bias-*`) properties. Per child:
  - `gpiochip_request_own_desc(gc, hwnum, name, lflags, dflags)` allocates a kernel-owned consumer.
  - On error: undo.
- `gpiochip_hog_lines(gc)` walks `gc->parent->of_node` children with `gpio-hog` property.
- `gpiod_hog(desc, name, lflags, dflags)` (5096):
  - `test_and_set_bit(IS_HOGGED)` — already hogged ⟹ return 0.
  - `gpiochip_request_own_desc(gc, hwnum, name, lflags, dflags)` requests + configures.
  - Failure clears IS_HOGGED and pr_err.
- `gpiochip_free_hogs(gc)` iterates descs with `GPIOD_FLAG_IS_HOGGED` and calls `gpiochip_free_own_desc`.

REQ-14 — `gpio_chip` IRQ subsystem (`gc->irq` block, `struct gpio_irq_chip`):
- `chip`: `struct irq_chip` template (may be IRQCHIP_IMMUTABLE).
- `domain`: assigned `struct irq_domain *` (gpiolib-created via simple or hierarchical helpers, or driver-supplied).
- `parent_domain`, `child_to_parent_hwirq`, `populate_parent_alloc_arg`: hierarchical-domain glue.
- `parent_handler`: chained-IRQ parent handler installed via `irq_set_chained_handler_and_data`.
- `parents[num_parents]` / `map[ngpio]`: per-line parent-irq mapping.
- `threaded`: request each line's interrupt as threaded.
- `default_type`, `lock_key`, `request_key`: lockdep classes.
- `irq_enable` / `irq_disable` / `irq_mask` / `irq_unmask` / `irq_ack` / `irq_set_type` / `irq_set_wake`: filled in by `gpiochip_set_irq_hooks` wrapping driver hooks with gpiolib bookkeeping (enables `gpiochip_enable_irq` / `_disable_irq`).
- `initialized`: set true at end of `gpiochip_add_irqchip`; consulted by `gpiochip_to_irq` for -EPROBE_DEFER.
- `valid_mask`: optional per-chip "this offset may act as IRQ".

REQ-15 — `gpiod_lookup_table` machine-side registration:
- `gpiod_add_lookup_table(table)` appends to `gpio_lookup_list` under `gpio_lookup_lock`.
- `gpiod_remove_lookup_table(table)` removes.
- Lookup entries (`struct gpiod_lookup`): `key` (chip label or dev_name), `chip_hwnum`, `con_id` (function), `idx`, `flags` (`GPIO_ACTIVE_LOW`, `GPIO_OPEN_DRAIN`, `GPIO_PULL_UP`, ...).
- `gpiod_find(dev, con_id, idx, flags*)` (4666) walks the list, matches by `dev_name(dev)` or `chip_label`, fills `flags*` and returns the matched `gpio_desc`.

REQ-16 — `/dev/gpiochipN` (gpiolib-cdev) summary (covered in cdev sibling doc):
- Backing file ops: `lineinfo_unwatch`, `line_create`, ioctls `GPIO_GET_CHIPINFO_IOCTL`, `GPIO_V2_GET_LINEINFO_IOCTL`, `_WATCH`, `_UNWATCH`, `GPIO_V2_GET_LINE_IOCTL`, `GPIO_V2_LINE_SET_CONFIG_IOCTL`, `_GET_VALUES_IOCTL`, `_SET_VALUES_IOCTL`.
- State-change notifications: `GPIO_V2_LINE_CHANGED_REQUESTED`, `_RELEASED`, `_CONFIG` emitted via `gpiod_line_state_notify`.
- Line events (edge): `GPIO_V2_LINE_EVENT_RISING_EDGE`, `_FALLING_EDGE`.
- Per-line flags echoed to/from descriptor flags (active_low, open_drain, open_source, bias, edge_*).

REQ-17 — Pinctrl ranges:
- `gpiochip_add_pin_range_with_pins(gc, pinctrl_name, gpio_offset, pin_offset, npins)` registers a gpiolib→pinctrl mapping so that `gpio_set_bias` / `gpio_set_config` can dispatch through pinctrl.
- `gpiochip_add_pingroup_range(gc, pctldev, gpio_offset, pin_group)` analog by pin-group name.
- `gpiochip_remove_pin_ranges(gc)` empties the list.
- Pinctrl is the substrate for bias / drive-strength / debounce that `gpio_set_bias`, `gpio_set_debounce_timeout`, `gpiod_set_config` apply.

REQ-18 — Locking model:
- `gpio_devices_lock` (mutex) — guards `gpio_devices` list mutation.
- `gpio_devices_srcu` — SRCU for readers of `gpio_devices`.
- `gpio_lookup_lock` — guards `gpio_lookup_list`.
- `gdev->srcu` — SRCU for `gdev->chip` pointer (cleared on remove; readers via `gpio_chip_guard`).
- `gdev->desc_srcu` — SRCU for `descs[]` label string.
- `gdev->line_state_lock` (rwlock) — read-mostly during get; write under set/configure.
- Atomic bitmap (`set_bit` / `clear_bit` / `test_and_set_bit`) — `desc->flags`.
- `gc->irq.lock_key` + `request_key` — lockdep classes for irqchip request paths.

## Acceptance Criteria

- [ ] AC-1: `gpiochip_add_data` registers chip; `/sys/bus/gpio/devices/gpiochipN` and `/dev/gpiochipN` appear; `gpio_device_find_by_label(gc->label)` returns the registered gdev.
- [ ] AC-2: `gpiochip_remove` of a chip with a requested descriptor warns and leaks the chip (gpiolib refuses to free); after free of all descs the `gdev` count drops to zero.
- [ ] AC-3: `gpiod_get(dev, con_id, GPIOD_OUT_HIGH)` on a DT-defined `<con_id>-gpios` returns a descriptor with REQUESTED + IS_OUT set and the line driven high; `gpiod_put` clears all per-line state.
- [ ] AC-4: ACTIVE_LOW: `gpiod_set_value(desc, 1)` writes 0 to the wire, `gpiod_get_value(desc)` reading 0 from wire returns 1.
- [ ] AC-5: OPEN_DRAIN: setting value=1 transitions the line to input (release); value=0 drives low. OPEN_SOURCE: setting value=0 transitions to input.
- [ ] AC-6: `gpiod_direction_input` followed by `gpiochip_lock_as_irq` succeeds and sets USED_AS_IRQ; subsequent `gpiod_direction_output` returns -EIO while USED_AS_IRQ is set.
- [ ] AC-7: `gpiod_to_irq(desc)` on a chip with `gc->irq.chip` returns a positive Linux IRQ; without an irqchip returns -ENXIO; before `gc->irq.initialized` returns -EPROBE_DEFER.
- [ ] AC-8: Hog declared in DT (`gpio-hog;`, `output-high;`) gets IS_HOGGED + IS_OUT and value=1 at probe time; `gpiochip_free_hogs` releases it at remove.
- [ ] AC-9: `gpiod_request` of an already-requested line returns -EBUSY without altering state.
- [ ] AC-10: `gpiod_get_optional(dev, con_id, ...)` with no matching DT/lookup returns NULL (not ERR_PTR(-ENOENT)).
- [ ] AC-11: cdev `GPIO_V2_LINE_SET_VALUES_IOCTL` honors the bitmask and updates only the requested lines; `GPIO_V2_LINE_CHANGED_CONFIG` notification is broadcast.
- [ ] AC-12: `gpiochip_find_base_unlocked(ngpio)` returns a non-overlapping base; static `base >= 0` triggers a dev_warn and a list-overlap check.
- [ ] AC-13: `gpiochip_add_pin_range*` followed by `gpio_set_bias(GPIO_PULL_UP)` delegates to the registered pinctrl device.
- [ ] AC-14: `gpiod_get_array(dev, con_id, GPIOD_OUT_LOW)` returns `gpio_descs` with NDESC entries; `gpiod_set_array_value(NDESC, descs, NULL, vals)` performs a single vectored chip-level write per chip on can_sleep=false chips.
- [ ] AC-15: `gpiolib_seq_show` debugfs entry lists each chip with base, ngpio, label, and per-line direction/value/label.

## Architecture

```
struct GpioChip<'drv> {
  label:           &'drv str,
  parent:          Option<*Device>,
  owner:           *Module,
  ngpio:           u16,
  base:            i32,                                        // <0 ⟹ dynamic
  names:           Option<&'drv [&'drv str]>,
  valid_mask:      Option<Box<[u64]>>,                         // ngpio bits
  request:         Option<fn(&Self, u16) -> i32>,
  free:            Option<fn(&Self, u16)>,
  get_direction:   Option<fn(&Self, u16) -> i32>,              // 1=in, 0=out
  direction_input: Option<fn(&Self, u16) -> i32>,
  direction_output:Option<fn(&Self, u16, bool) -> i32>,
  get:             Option<fn(&Self, u16) -> i32>,
  get_multiple:    Option<fn(&Self, &Mask, &mut Mask) -> i32>,
  set:             Option<fn(&Self, u16, bool) -> i32>,
  set_multiple:    Option<fn(&Self, &Mask, &Mask) -> i32>,
  set_config:      Option<fn(&Self, u16, PinConfigParam) -> i32>,
  to_irq:          Option<fn(&Self, u16) -> i32>,
  dbg_show:        Option<fn(&Self, &mut SeqFile)>,
  init_valid_mask: Option<fn(&Self, &mut Mask, u16) -> i32>,
  add_pin_ranges:  Option<fn(&Self) -> i32>,
  of_xlate:        Option<fn(&Self, &OfPhandleArgs) -> i32>,
  can_sleep:       bool,
  data:            *mut (),                                    // driver data
  gpiodev:         Option<NonNull<GpioDevice>>,                // gpiolib-owned
  irq:             GpioIrqChip,
}

struct GpioIrqChip {
  chip:                   Option<*const IrqChip>,
  domain:                 Option<*mut IrqDomain>,
  parent_domain:          Option<*mut IrqDomain>,
  child_to_parent_hwirq:  Option<fn(&GpioChip, u32, u32, *mut u32, *mut u32) -> i32>,
  populate_parent_alloc_arg: Option<fn(&GpioChip, *mut IrqFwspec, u32, u32) -> *mut ()>,
  parent_handler:         Option<irq_flow_handler_t>,
  parent_handler_data:    *mut (),
  parents:                Option<Box<[u32]>>,
  map:                    Option<Box<[u32]>>,
  threaded:               bool,
  default_type:           u32,
  init_hw:                Option<fn(&GpioChip) -> i32>,
  init_valid_mask:        Option<fn(&GpioChip, &mut Mask, u16)>,
  valid_mask:             Option<Box<[u64]>>,
  lock_key:               LockClassKey,
  request_key:            LockClassKey,
  initialized:            bool,
  irq_enable:             Option<fn(&IrqData)>,
  irq_disable:            Option<fn(&IrqData)>,
  irq_mask:               Option<fn(&IrqData)>,
  irq_unmask:             Option<fn(&IrqData)>,
}

struct GpioDevice {
  id:                  u32,                                   // ida_alloc
  dev:                 Device,
  chrdev:              Cdev,
  chip:                Rcu<*mut GpioChip>,                    // cleared on remove
  descs:               Box<[GpioDesc]>,                       // [ngpio]
  ngpio:               u16,
  base:                i32,
  can_sleep:           bool,
  label:               KStr,                                  // kstrdup_const
  owner:               *Module,
  line_state_lock:     RwLock,
  line_state_notifier: RawNotifierHead,
  device_notifier:     BlockingNotifierHead,
  pin_ranges:          ListHead,                              // gpio_pin_range
  srcu:                Srcu,                                  // chip pointer
  desc_srcu:           Srcu,                                  // labels
  list:                ListHead,                              // global list link
}

struct GpioDesc {
  gdev:                *GpioDevice,                           // back-pointer
  flags:               AtomicBitmap,                          // GPIOD_FLAG_*
  label:               Rcu<KStr>,                             // SRCU-protected
  name:                Option<&'static str>,
  hog:                 Option<*Device>,                       // CONFIG_OF_DYNAMIC
  debounce_period_us:  u32,                                   // CONFIG_GPIO_CDEV
}

struct GpiodLookupTable {
  list:    ListHead,
  dev_id:  &'static str,
  table:   &'static [GpiodLookup],
}
struct GpiodLookup {
  key:        &'static str,                                   // chip label or dev_id
  chip_hwnum: u16,
  con_id:     Option<&'static str>,                           // function
  idx:        u16,
  flags:      u32,                                            // GPIO_ACTIVE_LOW | ...
}
```

`Gpio::chip_add_data(gc, data, lock_key, request_key) -> i32`:
1. /* Allocate gpio_device + ida */
2. `gdev = kzalloc(GpioDevice)`; on ENOMEM return -ENOMEM.
3. `gc.gpiodev = Some(gdev)`; store driver data.
4. `gdev.id = ida_alloc(gpio_ida)`; init both SRCU domains.
5. `rcu_assign_pointer(gdev.chip, gc)`.
6. `device_initialize(&gdev.dev)` (type=gpio_dev_type, bus=gpio_bus_type, parent=gc.parent).
7. `gpiochip_get_ngpios(gc, &gdev.dev)` resolves ngpio (DT `ngpios`).
8. `gdev.descs = kcalloc(gc.ngpio)`; init each `desc.gdev`, IS_OUT bit per `gc.get_direction` / `gc.direction_input`.
9. /* Base + global list */
10. Under `gpio_devices_lock`:
    - if `gc.base < 0`: `base = find_base_unlocked(gc.ngpio)`; on err return.
    - else: dev_warn "Static allocation deprecated".
    - `gdev.base = base`; `gpiodev_add_to_list_unlocked(gdev)` — overlap returns -EBUSY.
11. `gpiochip_set_names(gc)`; `gpiochip_init_valid_mask(gc)`.
12. `of_gpiochip_add(gc)`; `gpiochip_add_pin_ranges(gc)`; `acpi_gpiochip_add(gc)`.
13. `gpiochip_hog_lines(gc)` — claim every `gpio-hog` child.
14. `gpiochip_irqchip_init_valid_mask(gc)`; `_init_hw(gc)`.
15. `gpiochip_add_irqchip(gc, lock_key, request_key)` — registers irq_domain (simple or hierarchical), sets `gc.to_irq = gpiochip_to_irq`, sets `gc.irq.initialized = true`.
16. `gpiochip_setup_shared(gc)` (pinctrl-shared); then `gpiochip_setup_dev(gc)` if `gpiolib_initialized`.
17. On any failure, unwind in reverse and pr_err unless -EPROBE_DEFER.

`Gpio::chip_remove(gc)`:
1. `gdev = gc.gpiodev`.
2. `gpio_device_teardown_shared(gdev)`.
3. `gpiochip_irqchip_remove(gc)`; `gpiochip_irqchip_free_valid_mask(gc)`.
4. `gpiochip_free_hogs(gc)`.
5. `acpi_gpiochip_remove(gc)`; `gpiochip_remove_pin_ranges(gc)`; `of_gpiochip_remove(gc)`.
6. `gpiochip_free_valid_mask(gc)`.
7. Under `gpio_devices_lock`: `list_del_rcu(&gdev.list)`; synchronize SRCU.
8. `gpiochip_free_remaining_irqs(gc)` (WARN if any).
9. `rcu_assign_pointer(gdev.chip, NULL)`; synchronize `gdev.srcu`.
10. `device_del(&gdev.dev)`; `gpio_device_put(gdev)` ⟹ `gpiodev_release` (kfree descs/label, ida_free, cleanup_srcu, kfree gdev).

`Gpio::get(dev, con_id, flags) -> GpioDesc | -errno`:
1. `desc = gpiod_find_and_request(dev, dev_fwnode(dev), con_id, idx=0, flags, label, optional=false)`.
2. Inside `find_and_request`:
   - Try `gpiod_fwnode_lookup` — walks DT for `<con_id>-gpios` etc., honors `gpio-cells`, fills lflags (lookup flags) + dflags (init flags) overlay.
   - On miss: `gpiod_find(dev, con_id, idx, &lflags)` walks `gpio_lookup_list`.
   - `gpiod_request(desc, label)` — REQUESTED guard + module_get + gpio_device_get.
   - `gpiod_configure_flags(desc, con_id, lflags, dflags)`:
     - ACTIVE_LOW / OPEN_DRAIN / OPEN_SOURCE / PULL_* / TRANSITORY → atomic set.
     - `gpio_set_config_with_argument_optional` for bias.
     - If `dflags & DIR_SET`: `gpiod_direction_output*` or `_input*`.
3. Return desc.

`Gpio::request(desc, label) -> i32`:
1. `VALIDATE_DESC(desc)` — -EINVAL on NULL/ERR_PTR or invalid.
2. `try_module_get(desc.gdev.owner)` — fail ⟹ -EPROBE_DEFER.
3. `gpiod_request_commit(desc, label)`:
   - `CLASS(gpio_chip_guard, guard)(desc)` SRCU-locks chip; NULL ⟹ -ENODEV.
   - `test_and_set_bit(REQUESTED)` ⟹ -EBUSY if already set.
   - `gpiochip_line_is_valid` check ⟹ -EINVAL.
   - `gc.request(gc, offset)` callback if any; >0 → -EBADE; nonzero unwind.
   - `gpiod_get_direction(desc)` to refresh IS_OUT.
   - `desc_set_label(desc, label ?: "?")` under RCU.
4. On error: `module_put(owner)`; `gpiod_dbg` log; return.
5. On success: `gpio_device_get(gdev)`.

`Gpio::free(desc)`:
1. `VALIDATE_DESC_VOID(desc)`.
2. `gpiod_free_commit(desc)`:
   - `might_sleep()`.
   - Under `gpio_chip_guard`: if REQUESTED:
     - `gc.free(gc, hwgpio)` if any.
     - Clear ACTIVE_LOW, REQUESTED, OPEN_DRAIN, OPEN_SOURCE, PULL_UP, PULL_DOWN, BIAS_DISABLE, EDGE_RISING, EDGE_FALLING, IS_HOGGED.
     - `desc.hog = None` (CONFIG_OF_DYNAMIC).
     - `desc_set_label(desc, NULL)`; `WRITE_ONCE(desc.flags, flags)`; debounce=0.
     - `gpiod_line_state_notify(desc, V2_LINE_CHANGED_RELEASED)`.
3. `module_put(gdev.owner)`; `gpio_device_put(gdev)`.

`Gpio::direction_output(desc, value) -> i32`:
1. `VALIDATE_DESC(desc)`.
2. Apply ACTIVE_LOW: `value = test_bit(ACTIVE_LOW) ? !value : value`.
3. Forward to `Gpio::direction_output_raw_commit(desc, value)`:
   - OPEN_DRAIN ⟹ if value: `gc.direction_input(gc, off)` (release line) else `gc.direction_output(gc, off, 0)`.
   - OPEN_SOURCE ⟹ if value: `gc.direction_output(gc, off, 1)` else `gc.direction_input(gc, off)`.
   - Else: `gc.direction_output(gc, off, value)`.
4. On success: `set_bit(IS_OUT, &desc.flags)`; emit `V2_LINE_CHANGED_CONFIG`.

`Gpio::get_value(desc) -> i32`:
1. `VALIDATE_DESC(desc)`.
2. `value = gpio_chip_get_value(gc, desc)`:
   - `gc.get(gc, off)` (or `gc.get_multiple` with single-bit mask).
3. Return `value ^ test_bit(ACTIVE_LOW)`.

`Gpio::set_value(desc, value)`:
1. `VALIDATE_DESC_VOID(desc)`.
2. `value ^= test_bit(ACTIVE_LOW)`.
3. Dispatch:
   - OPEN_DRAIN ⟹ `gpio_set_open_drain_value_commit`.
   - OPEN_SOURCE ⟹ `gpio_set_open_source_value_commit`.
   - Else ⟹ `gpiochip_set(gc, off, value)` ⟹ `gc.set(gc, off, value)`.

`Gpio::to_irq(desc) -> i32`:
1. `VALIDATE_DESC(desc)`.
2. If `gc.to_irq`: ret = `gc.to_irq(gc, hwgpio)`.
3. Else if `gc.irq.domain`: ret = `irq_find_mapping(gc.irq.domain, hwgpio)` (0 means not mapped).
4. Else: -ENXIO.
5. If ret > 0: `set_bit(USED_AS_IRQ, &desc.flags)`.

`Gpio::chip_to_irq(gc, off) -> i32` (default `gc.to_irq` set by `gpiochip_add_irqchip`):
1. If `!gc.irq.initialized`: return -EPROBE_DEFER.
2. If `!gpiochip_irqchip_irq_valid(gc, off)`: return -ENXIO.
3. If `irq_domain_is_hierarchy(gc.irq.domain)`:
   - Build `irq_fwspec { fwnode, count=2, [child_offset_to_irq(off), IRQ_TYPE_NONE] }`.
   - Return `irq_create_fwspec_mapping(&spec)`.
4. Else: return `irq_create_mapping(gc.irq.domain, off)`.

`Gpio::lock_as_irq(gc, off) -> i32`:
1. `desc = gpiochip_get_desc(gc, off)`.
2. If REQUESTED ∧ !IS_OUT ⟹ ok (input already requested).
3. Else if !REQUESTED ⟹ implicit request via `gpiochip_reqres_irq`.
4. Else if IS_OUT ⟹ return -EIO (refuse output→IRQ).
5. `set_bit(USED_AS_IRQ, &desc.flags)`.

`Gpio::hog(desc, name, lflags, dflags) -> i32`:
1. `CLASS(gpio_chip_guard, guard)(desc)`; NULL ⟹ -ENODEV.
2. `test_and_set_bit(IS_HOGGED)` ⟹ already hogged: return 0.
3. `gpiochip_request_own_desc(gc, hwnum, name, lflags, dflags)`:
   - `function_name_or_default(con_id)` resolves label.
   - `gpiod_request_commit(desc, label)`.
   - `gpiod_configure_flags(desc, name, lflags, dflags)`.
4. On error: clear IS_HOGGED; pr_err.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `request_balanced_with_free` | INVARIANT | per-desc: REQUESTED set ⟹ exactly one matching free; module + gdev refcounts balanced. |
| `request_set_bit_atomic` | INVARIANT | per-desc: test_and_set_bit(REQUESTED) is the only path that returns -EBUSY. |
| `valid_mask_respected` | INVARIANT | per-line ops: gpiochip_line_is_valid(gc, off) must return true before driver ops invoked. |
| `chip_remove_no_in_use` | INVARIANT | per-gpiochip_remove: no descriptor with REQUESTED bit observed when device_del runs (WARN). |
| `irq_lock_excludes_output` | INVARIANT | per-line: USED_AS_IRQ set ⟹ direction_output paths return -EIO. |
| `srcu_chip_pointer_valid` | INVARIANT | per-`gpio_chip_guard`: guard.gc non-NULL ⟹ gdev.chip pointer still rcu-valid for the read-side critical section. |
| `active_low_xor_correctness` | INVARIANT | per-get/set: ACTIVE_LOW bit XOR'd exactly once between hardware level and consumer level. |
| `hogged_lines_freed_at_remove` | INVARIANT | per-chip_remove: IS_HOGGED descriptors freed via gpiochip_free_hogs before list_del. |

### Layer 2: TLA+

`drivers/gpio/gpiolib.tla`:
- States per descriptor: FREE → REQUESTED → REQUESTED+OUT → REQUESTED+IRQ → FREE.
- Properties:
  - `safety_no_double_request` — REQUESTED ⟹ next request returns -EBUSY.
  - `safety_no_output_while_irq` — USED_AS_IRQ ⟹ direction_output forbidden.
  - `safety_chip_pointer_srcu` — readers see a non-NULL chip pointer or take the !ENODEV exit.
  - `safety_hog_idempotent` — gpiod_hog twice on the same desc is a no-op.
  - `liveness_request_eventually_observed` — gpiod_request notifies the cdev with `V2_LINE_CHANGED_REQUESTED`.
  - `liveness_remove_terminates` — gpiochip_remove with all descriptors freed completes in finite steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Gpio::chip_add_data` post: registered chip has gdev.id allocated, base assigned, descs[ngpio] initialized, listed in gpio_devices | `Gpio::chip_add_data` |
| `Gpio::chip_remove` post: gdev removed from list, chip pointer NULL, all hogs freed | `Gpio::chip_remove` |
| `Gpio::request` post: REQUESTED set, label installed, module + gdev refcounts up by one | `Gpio::request` |
| `Gpio::free` post: REQUESTED + per-line flags cleared, label = NULL, refcounts balanced, notification emitted | `Gpio::free` |
| `Gpio::direction_output` post: IS_OUT set, OPEN_DRAIN/OPEN_SOURCE asymmetry respected | `Gpio::direction_output` |
| `Gpio::get_value` post: returned value = hw_value XOR ACTIVE_LOW | `Gpio::get_value` |
| `Gpio::to_irq` post: returned irq > 0 ⟹ USED_AS_IRQ bit set | `Gpio::to_irq` |
| `Gpio::chip_to_irq` post: -EPROBE_DEFER ⟹ irq.initialized was false | `Gpio::chip_to_irq` |

### Layer 4: Verus/Creusot functional

`Per-driver gpiochip_add_data(gc, data) → gpiolib registers chip → consumers gpiod_get(dev, "foo", GPIOD_OUT_LOW) → gpiod_direction_output / gpiod_get_value with ACTIVE_LOW honored → gpiod_to_irq returns mapped IRQ → gpiochip_remove releases all hogs / irqchip / cdev` semantic equivalence: per-`Documentation/driver-api/gpio/`, `Documentation/userspace-api/gpio/chardev.rst`, `Documentation/devicetree/bindings/gpio/gpio.txt`, and the consumer.h `enum gpiod_flags` contract.

## Hardening

(Inherits row-1 features from `drivers/gpio/00-overview.md` § Hardening.)

gpiolib reinforcement:

- **Per-`REQUESTED` test_and_set strict guard** — defense against per-double-request races (two drivers fighting over the same line).
- **Per-`module_get(gdev->owner)` on request** — defense against per-chip-unload-while-line-held UAF.
- **Per-`gpiochip_remove` refuses in-use chips (WARN)** — defense against per-removal of a chip with active consumers.
- **Per-SRCU chip pointer** — defense against per-remove tearing down `gc` while a reader is mid-op.
- **Per-`USED_AS_IRQ` excludes direction_output** — defense against per-line concurrently driven and used as IRQ source.
- **Per-`valid_mask` enforced before any driver op** — defense against per-touching reserved hardware pins.
- **Per-hog list freed before chip teardown** — defense against per-hogged-line leaks at unbind.
- **Per-`gpio_devices_lock` + SRCU readers** — defense against per-iterating-while-mutating the global chip list.
- **Per-`gpio_chip_guard` (CLASS) auto-unlocks SRCU on early return** — defense against per-leaked SRCU read-side critical sections.
- **Per-`can_sleep` cross-check** — defense against per-sleepable-ops called from atomic context (and vice versa).
- **Per-ACTIVE_LOW XOR enforced at exactly one boundary** — defense against per-double-inversion bugs.
- **Per-`/dev/gpiochipN` line-state notifier rate-limited via blocking-notifier semantics** — defense against per-userspace flood of state events.
- **Per-cdev ioctl arg copy bounded by `struct gpio_v2_line_request`** — defense against per-uAPI overread.

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — bounded user-buffer copy on `/dev/gpiochipN` IOCTLs (`GPIO_V2_GET_LINE_IOCTL`, `GPIO_V2_LINE_SET_CONFIG_IOCTL`, `GPIO_V2_LINE_SET_VALUES_IOCTL`).
- **PAX_KERNEXEC** — W^X enforcement on `gpiochip` ops + irqchip handler dispatch.
- **PAX_RANDKSTACK** — kernel-stack randomization on gpio cdev IOCTL entry + IRQ-thread handler entry.
- **PAX_REFCOUNT** — saturating refcount on `gpio_device`, `gpio_desc`, line-event consumer state, and irqchip parent ref.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `gpio_device`, `gpio_desc` arrays, line-event ring buffer, hog list slabs.
- **PAX_UDEREF** — SMAP/SMEP enforcement on every gpio cdev IOCTL user-pointer access.
- **PAX_RAP / kCFI** — `gpio_chip` ops (`request` / `free` / `get_direction` / `direction_input` / `direction_output` / `get` / `set` / `to_irq`) and `irq_chip` ops (`irq_ack` / `irq_mask` / `irq_unmask` / `irq_set_type` / `irq_set_wake`) hardened against indirect-call hijack; per-driver tables `static const`.
- **GRKERNSEC_HIDESYM** — kernel-pointer hiding in `/sys/class/gpio/*` and `gpioinfo` debugfs.
- **GRKERNSEC_DMESG** — syslog restriction on gpiolib WARN (invalid-line, USED_AS_IRQ vs direction_output conflict, hog list errors).
- **PAX_CONSTIFY_PLUGIN** — every `gpio_chip` and `irq_chip` literal `static const`.
- **CAP_SYS_ADMIN strict** — `/dev/gpiochipN` IOCTLs gated; GR-RBAC denies opportunistic grant.
- **GRKERNSEC_KMOD** — denies opportunistic driver load on gpio hot-plug.
- **GRKERNSEC_SYSFS_RESTRICT** — `/sys/class/gpio/*` (legacy ABI) restricted; new code uses chardev with LSM gates.
- **PAX_SIZE_OVERFLOW** — line offset, line count, ring-buffer producer/consumer arithmetic checked; `GPIO_V2_LINES_MAX` enforced.
- **LSM `security_file_ioctl`** — every gpio cdev IOCTL gated per GR-RBAC subject; line-event sub-channel inherits the subject.
- **LSM `security_locked_down(LOCKDOWN_DEV_MEM)`** — denies legacy `/dev/gpiochip` raw register pokes under lockdown.

Per-doc rationale: GPIOs are bare-metal I/O — flipping a line can reset the SoC, toggle external write-protect pins on flash, drive bus-isolation gates, or strobe security peripherals (TPM reset, TRNG enable). PAX_RAP locks both the per-driver `gpio_chip` ops and the IRQ-chip vtable that every IRQ entry indirects through (highest-frequency indirect calls in the IRQ subsystem); CAP_SYS_ADMIN + LSM `security_file_ioctl` gate `/dev/gpiochipN`, PAX_USERCOPY + UDEREF clamp the IOCTL ingress, and PAX_MEMORY_SANITIZE wipes the line-event ring buffer (which can carry timing-channel data leaked by neighbor consumers).

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `drivers/gpio/gpiolib-cdev.c` userspace chardev implementation (covered in its own Tier-3 doc).
- `drivers/gpio/gpiolib-of.c` Device Tree parsing details (covered in its own Tier-3 doc).
- `drivers/gpio/gpiolib-acpi.c` ACPI binding (covered in its own Tier-3 doc).
- `drivers/gpio/gpiolib-sysfs.c` legacy sysfs interface (deprecated; covered separately if expanded).
- Individual gpio_chip driver implementations (e.g. `drivers/gpio/gpio-omap.c`, `gpio-pca953x.c`).
- `kernel/irq/*` IRQ-domain core (covered in `kernel/irq/` Tier-3).
- `drivers/pinctrl/core.c` pinctrl substrate (covered in `drivers/pinctrl/` Tier-3 when authored).
- Implementation code.
