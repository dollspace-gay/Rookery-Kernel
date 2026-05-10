---
title: "Tier-3: drivers/i2c/i2c-core-base.c — I2C subsystem core (adapter / client / driver lifecycle, transfer, enumeration)"
tags: ["tier-3", "drivers", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

The **I2C core base** is the framework that registers the `i2c` bus_type at boot, brokers per-adapter (HW controller) ↔ per-client (slave chip) ↔ per-driver (in-kernel module) match/probe/remove via the standard Linux driver-model, runs the actual on-wire transactions via `i2c_transfer` (which delegates to `adapter.algo.master_xfer`), and exposes board / DT / ACPI / fwnode enumeration paths so that every i2c slave declared in firmware gets a `struct i2c_client` instance bound to its driver. Lifecycle: at boot, `i2c_init` (postcore_initcall) calls `bus_register(&i2c_bus_type)`, sets `is_registered = true`, and registers the dummy_driver. Per-adapter: `i2c_add_adapter` allocates an idr slot, calls `i2c_register_adapter` (which `rt_mutex_init`s `bus_lock` + `mux_lock`, runs `i2c_setup_host_notify_irq_domain`, `device_add`s the adapter device, then enumerates pre-declared OF/ACPI/board_info slaves, and finally calls every i2c_driver's `__process_new_adapter`). Per-client: `i2c_new_client_device(adap, &board_info)` allocates `struct i2c_client`, validates address (7-bit or 10-bit via `I2C_CLIENT_TEN`), checks address-busy against parent mux + child devices, registers with driver core (which triggers match → probe). Per-driver: `i2c_register_driver` calls `driver_register` then walks every registered adapter calling `__process_new_driver` → `i2c_do_add_adapter` → `i2c_detect` if `driver->detect` set. Per-transfer: `i2c_transfer(adap, msgs, num)` takes `bus_lock` (rt_mutex, segment- or root-mux scope), calls `__i2c_transfer` → `adapter.algo.master_xfer(adap, msgs, num)` with retry loop on `-EAGAIN` (up to `adap->retries`, bounded by `adap->timeout` jiffies). Critical for: every i2c-hid touchpad/touchscreen, every i2c sensor (hwmon/iio), every codec, every camera, every PMIC, every EEPROM, every RTC, every DDC monitor — the contract for ABI-compat must be byte-exact.

This Tier-3 covers `drivers/i2c/i2c-core-base.c` (~2687 lines).

### Acceptance Criteria

- [ ] AC-1: `i2c_add_adapter(adap)` with valid `adap->algo->master_xfer` succeeds; `/sys/bus/i2c/devices/i2c-<N>/name` byte-matches Linux.
- [ ] AC-2: `i2c_add_adapter` without `algo` returns -EINVAL and the adapter is removed from idr.
- [ ] AC-3: `i2c_del_adapter` releases all clients (real + dummy + userspace) and waits on `dev_released` before idr-remove.
- [ ] AC-4: `i2c_new_client_device(adap, &info)` with 7-bit addr 0x05 (reserved range) returns -EINVAL.
- [ ] AC-5: `i2c_new_client_device` race-test: two concurrent calls with same addr — exactly one wins, other gets -EBUSY.
- [ ] AC-6: `i2c_transfer(adap, msgs, num)` with `algo->master_xfer = NULL` returns -EOPNOTSUPP without taking bus_lock changes.
- [ ] AC-7: `i2c_transfer` -EAGAIN return retries up to `adap->retries`, bounded by `adap->timeout`.
- [ ] AC-8: `i2c_transfer` from atomic context with `algo->master_xfer_atomic` set: uses atomic path, never sleeps.
- [ ] AC-9: `i2c_register_driver` with `address_list` + `detect`: per registered adapter, detect runs and any positive result instantiates client linked into `driver->clients`.
- [ ] AC-10: `i2c_del_driver` releases every `detect()`-instantiated client (in `driver->clients`).
- [ ] AC-11: `i2c_device_probe` IRQ resolution: OF irq → ACPI irq → 0; first hit wins. `client->irq` matches.
- [ ] AC-12: `MODALIAS` uevent: per-OF: `of:Nfoo...`; per-ACPI: `acpi:HID`; per-id_table: `i2c:<name>` byte-identical.
- [ ] AC-13: Userspace `echo "<name> <addr>" > new_device` instantiates a client; `echo <addr> > delete_device` removes it.
- [ ] AC-14: `i2c_check_for_quirks` rejects msg whose len exceeds `q->max_write_len` with -EOPNOTSUPP and dev_err_ratelimited.
- [ ] AC-15: Mux-aware nested bus_lock: child-segment lock + parent-mux lock acquired without lockdep splat (per `i2c_adapter_depth` subclass).
- [ ] AC-16: `i2c_get_adapter(nr)` returns adapter only after `try_module_get` succeeds; `i2c_put_adapter` is reverse.

### Architecture

```
struct Adapter {
  owner: *Module,
  class: u32,
  algo: *Algorithm,
  algo_data: *void,
  lock_ops: *LockOps,            // default = i2c_adapter_lock_ops
  bus_lock: RtMutex,             // per-segment
  mux_lock: RtMutex,             // per-mux parent-child
  timeout: u32,                  // jiffies; default HZ
  retries: i32,
  dev: Device,
  nr: i32,                       // idr-allocated (or static via OF alias)
  name: [u8; I2C_NAME_SIZE],
  userspace_clients: ListHead,
  userspace_clients_lock: Mutex,
  bus_recovery_info: Option<*BusRecoveryInfo>,
  quirks: Option<*AdapterQuirks>,
  dev_released: Completion,
  host_notify_domain: Option<*IrqDomain>,
  addrs_in_instantiation: BitMap<I2C_ADDR_7BITS_COUNT>,
  debugfs: Option<*Dentry>,
  locked_flags: u32,
}

struct Client {
  flags: u16,                    // I2C_CLIENT_TEN | _WAKE | _HOST_NOTIFY | _PEC | _SLAVE
  addr: u16,                     // 7-bit or 10-bit
  name: [u8; I2C_NAME_SIZE],
  adapter: *Adapter,
  dev: Device,
  init_irq: i32,
  irq: i32,
  detected: ListHead,            // linkage in driver->clients
  debugfs: Option<*Dentry>,
  devres_group_id: Option<*void>,
}

struct Driver {
  class: u32,
  probe: fn(*Client) -> i32,
  remove: fn(*Client),
  shutdown: fn(*Client),
  command: fn(*Client, u32, *void),
  driver: DeviceDriver,
  id_table: *I2cDeviceId,
  detect: Option<fn(*Client, *BoardInfo) -> i32>,
  address_list: Option<*u16>,
  clients: ListHead,             // per-driver list of detect()-instantiated
  flags: u32,
}
```

`Adapter::add(adap) -> Result<()>`:
1. `id = of_alias_get_id(dev->of_node, "i2c")`.
2. If `id >= 0`: `adap->nr = id; Adapter::add_numbered_inner(adap)`.
3. Else: `mutex_lock(&core_lock); id = idr_alloc(&i2c_adapter_idr, adap, __i2c_first_dynamic_bus_num, 0, GFP_KERNEL); mutex_unlock`.
4. If `id < 0`: WARN; return Err.
5. `adap->nr = id`.
6. `Adapter::register_inner(adap)`.

`Adapter::register_inner(adap) -> Result<()>`:
1. If `!is_registered`: return Err(-EAGAIN).
2. Sanity: `adap->name[0] != 0` ∧ `adap->algo != NULL`.
3. `adap->lock_ops ||= &i2c_adapter_lock_ops`.
4. `rt_mutex_init(&adap->bus_lock)`; `rt_mutex_init(&adap->mux_lock)`.
5. `mutex_init(&adap->userspace_clients_lock)`; `INIT_LIST_HEAD(&adap->userspace_clients)`.
6. `adap->timeout ||= HZ`.
7. `i2c_setup_host_notify_irq_domain(adap)`.
8. `dev_set_name(&adap->dev, "i2c-%d", adap->nr)`; `dev.bus = &i2c_bus_type; dev.type = &i2c_adapter_type`.
9. `device_initialize`; `device_enable_async_suspend`; `pm_runtime_no_callbacks`; `pm_suspend_ignore_children(true)`; `pm_runtime_enable`.
10. `device_add(&adap->dev)`.
11. `adap->debugfs = debugfs_create_dir(...)`.
12. `i2c_setup_smbus_alert(adap)`.
13. `i2c_init_recovery(adap)` (handle -EPROBE_DEFER).
14. `of_i2c_register_devices(adap)`; `i2c_acpi_install_space_handler`; `i2c_acpi_register_devices`.
15. If `adap->nr < __i2c_first_dynamic_bus_num`: `i2c_scan_static_board_info`.
16. `mutex_lock(&core_lock); bus_for_each_drv(&i2c_bus_type, NULL, adap, __process_new_adapter); mutex_unlock`.

`Adapter::del(adap)`:
1. `mutex_lock(&core_lock); found = idr_find(&i2c_adapter_idr, adap->nr); mutex_unlock`.
2. If `found != adap`: pr_debug; return.
3. `i2c_acpi_remove_space_handler(adap)`.
4. `mutex_lock(&core_lock); bus_for_each_drv(__process_removed_adapter); mutex_unlock`.
5. `mutex_lock_nested(&adap->userspace_clients_lock, i2c_adapter_depth(adap))`; per-`c in userspace_clients`: `i2c_unregister_device(c)`.
6. `device_for_each_child(&adap->dev, NULL, __unregister_client)`.
7. `device_for_each_child(&adap->dev, NULL, __unregister_dummy)`.
8. `pm_runtime_disable; i2c_host_notify_irq_teardown; debugfs_remove_recursive`.
9. `init_completion(&adap->dev_released); device_unregister; wait_for_completion`.
10. `mutex_lock(&core_lock); idr_remove(&i2c_adapter_idr, adap->nr); mutex_unlock`.
11. `memset(&adap->dev, 0, sizeof(adap->dev))`.

`Client::new(adap, info) -> Result<*Client>`:
1. Allocate; on fail return Err(-ENOMEM).
2. Populate `adapter`, `dev.platform_data`, `flags`, `addr`, `init_irq`, `name`.
3. `i2c_check_addr_validity(addr, flags)?`.
4. `i2c_lock_addr(adap, addr, flags)?` (test_and_set_bit).
5. `i2c_check_addr_busy(adap, i2c_encode_flags_to_addr(client))?`.
6. `dev.parent = &adap->dev; dev.bus = &i2c_bus_type; dev.type = &i2c_client_type`.
7. `device_enable_async_suspend(&dev)`.
8. `device_set_node(&dev, fwnode_handle_get(info->fwnode))`.
9. If `info->swnode`: `device_add_software_node`.
10. `i2c_dev_set_name(adap, client, info)`.
11. `device_register(&dev)` — triggers match/probe.
12. `i2c_unlock_addr(adap, addr, flags)`.

`Adapter::transfer(adap, msgs, num) -> i32`:
1. `ret = __i2c_lock_bus_helper(adap)`.
2. If ret: return ret.
3. `ret = Adapter::transfer_unlocked(adap, msgs, num)`.
4. `i2c_unlock_bus(adap, I2C_LOCK_SEGMENT)`.
5. return ret.

`Adapter::transfer_unlocked(adap, msgs, num) -> i32`:
1. If `!adap->algo->master_xfer`: return -EOPNOTSUPP.
2. If `!msgs || num < 1`: WARN; return -EINVAL.
3. `__i2c_check_suspended(adap)?`.
4. If `adap->quirks && i2c_check_for_quirks(adap, msgs, num)`: return -EOPNOTSUPP.
5. If trace static-branch hot: per-msg `trace_i2c_{read,write}`.
6. `orig_jiffies = jiffies`.
7. Loop `try ∈ 0..=adap->retries`:
   - If `i2c_in_atomic_xfer_mode() && algo->master_xfer_atomic`: `ret = algo->master_xfer_atomic(adap, msgs, num)`.
   - Else: `ret = algo->master_xfer(adap, msgs, num)`.
   - If `ret != -EAGAIN`: break.
   - If `time_after(jiffies, orig_jiffies + adap->timeout)`: break.
8. If trace hot: per-reply tracepoint + `trace_i2c_result`.
9. return ret.

`Driver::register(owner, driver) -> Result<()>`:
1. If `!is_registered`: return Err(-EAGAIN).
2. `driver->driver.owner = owner; .bus = &i2c_bus_type`.
3. `INIT_LIST_HEAD(&driver->clients)`.
4. `driver_register(&driver->driver)?` (driver-core match → probe runs for already-present devices).
5. `i2c_for_each_dev(driver, __process_new_driver)` — per-adapter `i2c_do_add_adapter` (runs detect if address_list).

### Out of Scope

- `i2c-core-acpi.c` ACPI enumeration internals (covered in `drivers/i2c/core.md` once split, or treated as helper called from base).
- `i2c-core-of.c` / `i2c-core-of-prober.c` DT enumeration internals (likewise).
- `i2c-core-slave.c` slave-mode protocol callbacks (separate Tier-3).
- `i2c-core-smbus.c` SMBus emulation arithmetic (separate Tier-3).
- `i2c-dev.c` `/dev/i2c-<N>` chardev (separate Tier-3 `drivers/i2c/dev.md`).
- `i2c-mux.c` + `muxes/` (separate Tier-3 `drivers/i2c/mux.md`).
- Per-controller LLDs in `busses/` (separate Tier-3s).
- Implementation code.

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct i2c_adapter` | per-controller (HW bus) | `i2c::Adapter` |
| `struct i2c_client` | per-slave (chip on bus) | `i2c::Client` |
| `struct i2c_driver` | per-driver (in-kernel module) | `i2c::Driver` |
| `struct i2c_msg` | per-transfer message | `i2c::Msg` |
| `struct i2c_board_info` | per-board pre-declared chip | `i2c::BoardInfo` |
| `struct i2c_algorithm` | per-adapter master_xfer ops | `i2c::Algorithm` |
| `struct i2c_lock_operations` | per-adapter bus-lock ops | `i2c::LockOps` |
| `struct i2c_adapter_quirks` | per-adapter HW limit constraints | `i2c::AdapterQuirks` |
| `i2c_bus_type` | per-bus driver-model bus_type | `i2c::BUS_TYPE` |
| `i2c_adapter_type` / `i2c_client_type` | per-device_type discriminators | `i2c::ADAPTER_TYPE` / `CLIENT_TYPE` |
| `i2c_init()` (postcore_initcall) | per-subsystem init | `Subsystem::init` |
| `i2c_exit()` | per-subsystem teardown | `Subsystem::exit` |
| `i2c_device_match(dev, drv)` | per-match (OF → ACPI → id_table) | `Subsystem::device_match` |
| `i2c_device_uevent(dev, env)` | per-modalias uevent | `Subsystem::device_uevent` |
| `i2c_device_probe(dev)` | per-driver-core probe trampoline | `Subsystem::device_probe` |
| `i2c_device_remove(dev)` / `i2c_device_shutdown(dev)` | per-remove / shutdown | `Subsystem::device_remove` / `_shutdown` |
| `i2c_add_adapter(adap)` | per-dynamic-bus-id add | `Adapter::add` |
| `i2c_add_numbered_adapter(adap)` | per-static-bus-id add | `Adapter::add_numbered` |
| `i2c_register_adapter(adap)` | per-internal register | `Adapter::register_inner` |
| `i2c_del_adapter(adap)` | per-adapter unregister | `Adapter::del` |
| `devm_i2c_add_adapter(dev, adap)` | per-devm variant | `Adapter::devm_add` |
| `i2c_new_client_device(adap, info)` | per-client instantiate | `Client::new` |
| `i2c_unregister_device(client)` | per-client unregister | `Client::unregister` |
| `i2c_new_dummy_device(adap, addr)` | per-dummy-driver bind | `Client::new_dummy` |
| `i2c_new_ancillary_device(...)` | per-secondary-addr alias | `Client::new_ancillary` |
| `i2c_new_scanned_device(...)` | per-probe-then-instantiate | `Client::new_scanned` |
| `i2c_find_device_by_fwnode(fwnode)` | per-fwnode lookup | `Subsystem::find_device_by_fwnode` |
| `i2c_register_driver(owner, drv)` | per-driver register | `Driver::register` |
| `i2c_del_driver(drv)` | per-driver unregister | `Driver::del` |
| `i2c_for_each_dev(data, fn)` | per-bus device iterator | `Subsystem::for_each_dev` |
| `i2c_clients_command(adap, cmd, arg)` | per-adapter broadcast cmd | `Adapter::clients_command` |
| `__i2c_transfer(adap, msgs, num)` | per-unlocked transfer | `Adapter::transfer_unlocked` |
| `i2c_transfer(adap, msgs, num)` | per-locked transfer | `Adapter::transfer` |
| `i2c_transfer_buffer_flags(client, buf, count, flags)` | per-single-msg shortcut | `Client::transfer_buffer_flags` |
| `i2c_check_for_quirks(adap, msgs, num)` | per-adapter-quirk validation | `Adapter::check_quirks` |
| `i2c_check_addr_validity(addr, flags)` | per-addr 7-bit/10-bit validation | `Adapter::check_addr_validity` |
| `i2c_check_addr_busy(adap, addr)` | per-addr busy (incl. mux parent/child) | `Adapter::check_addr_busy` |
| `i2c_lock_addr` / `_unlock_addr` | per-addr instantiation serialization | `Adapter::lock_addr` / `unlock_addr` |
| `i2c_adapter_lock_bus` / `_trylock_bus` / `_unlock_bus` | per-adapter rt_mutex lock ops | `Adapter::lock_bus` / `trylock_bus` / `unlock_bus` |
| `__i2c_lock_bus_helper(adap)` | per-locked-flags-aware lock | `Adapter::lock_bus_helper` |
| `i2c_adapter_depth(adap)` | per-mux-tree depth for nested rt_mutex | `Adapter::depth` |
| `i2c_get_adapter(nr)` / `i2c_put_adapter(adap)` | per-idr lookup + refcount | `Adapter::get_by_nr` / `put` |
| `i2c_for_each_dev` / `i2c_verify_client` / `i2c_verify_adapter` | per-type discriminant | `Subsystem::verify_*` |
| `i2c_match_id` / `i2c_get_match_data` | per-id_table match | `Driver::match_id` / `get_match_data` |
| `i2c_detect(adap, drv)` / `i2c_detect_address` / `i2c_default_probe` | per-driver address scan | `Driver::detect` / `Driver::detect_address` |
| `i2c_get_device_id(client, id)` | per-SMBus device-id query | `Client::get_device_id` |
| `i2c_get_dma_safe_msg_buf` / `i2c_put_dma_safe_msg_buf` | per-DMA bounce-buffer | `Msg::dma_safe_buf` / `put_dma_safe_buf` |
| `i2c_recover_bus(adap)` | per-stuck-bus recovery | `Adapter::recover_bus` |
| `i2c_generic_scl_recovery(adap)` | per-SCL-toggle recovery | `Adapter::generic_scl_recovery` |
| `i2c_handle_smbus_host_notify(adap, addr)` | per-SMBus-host-notify IRQ | `Adapter::handle_smbus_host_notify` |
| `i2c_freq_mode_string(hz)` | per-bus-frequency label | `Adapter::freq_mode_string` |
| `i2c_parse_fw_timings(dev, t, ...)` | per-firmware-timings parse | `Adapter::parse_fw_timings` |

Cross-subsystem helpers (defined in sibling cores, called by base):
- `of_i2c_register_devices(adap)` — DT enumeration (i2c-core-of.c).
- `i2c_acpi_register_devices(adap)` / `i2c_acpi_install_space_handler(adap)` / `i2c_acpi_get_irq(client)` — ACPI enumeration + OpRegion (i2c-core-acpi.c).
- `i2c_setup_smbus_alert(adap)` — SMBus alert wiring (i2c-core-smbus.c / i2c-smbus.c).
- `i2c_smbus_xfer(adap, addr, flags, read_write, command, size, data)` — SMBus emulation (i2c-core-smbus.c).
- `i2c_scan_static_board_info(adap)` — legacy `i2c_register_board_info` table walk (i2c-boardinfo.c).

### compatibility contract

REQ-1: `struct i2c_adapter` shape (driver-visible):
- `owner: *Module` (driver-module ref).
- `class: u32` (I2C_CLASS_HWMON | _DEPRECATED | _SPD).
- `algo: *I2cAlgorithm` (master_xfer / master_xfer_atomic / smbus_xfer / functionality).
- `algo_data: *void`.
- `lock_ops: *I2cLockOperations` (default = `i2c_adapter_lock_ops`).
- `bus_lock: RtMutex` (per-segment lock).
- `mux_lock: RtMutex` (per-mux parent-child serialization).
- `timeout: u32 jiffies` (default HZ).
- `retries: i32` (auto-retry count on -EAGAIN).
- `dev: Device` (driver-model device).
- `nr: i32` (idr-allocated bus number; appears as `i2c-<nr>` in sysfs).
- `name[I2C_NAME_SIZE]: u8` (mandatory; checked at register).
- `userspace_clients: ListHead` (created via `/sys/bus/i2c/devices/i2c-<N>/new_device`).
- `userspace_clients_lock: Mutex`.
- `bus_recovery_info: *I2cBusRecoveryInfo` (optional stuck-bus recovery).
- `quirks: *I2cAdapterQuirks` (optional msg-list constraints).
- `dev_released: Completion` (used by `i2c_del_adapter` to drain refs).
- `host_notify_domain: *IrqDomain` (SMBus Host-Notify).
- `addrs_in_instantiation: DECLARE_BITMAP(I2C_ADDR_7BITS_COUNT)` (race-free addr lock during `i2c_new_client_device`).
- `debugfs: *Dentry` (per-adapter debugfs root).
- `locked_flags: u32` (segment / root-mux locking state).

REQ-2: `struct i2c_client` shape:
- `flags: u16`: I2C_CLIENT_TEN | _WAKE | _HOST_NOTIFY | _SCCB | _PEC | _SLAVE.
- `addr: u16` (7-bit canonical or 10-bit if TEN flag).
- `name[I2C_NAME_SIZE]: u8`.
- `adapter: *I2cAdapter`.
- `dev: Device`.
- `init_irq: i32` / `irq: i32`.
- `detected: ListHead` (linkage in driver->clients).
- `debugfs: *Dentry`.
- `devres_group_id: *void`.
- `slave_cb: SlaveCb` (CONFIG_I2C_SLAVE).

REQ-3: `struct i2c_driver` shape:
- `class: u32` (per-class detect).
- `probe(client: *I2cClient) -> i32`.
- `remove(client: *I2cClient) -> void`.
- `shutdown(client: *I2cClient) -> void`.
- `command(client, cmd, arg) -> void`.
- `driver: DeviceDriver` (driver-model embedding).
- `id_table: *I2cDeviceId` (NULL-terminated).
- `detect(client: *I2cClient, info: *I2cBoardInfo) -> i32`.
- `address_list: *u16` (I2C_CLIENT_END-terminated).
- `clients: ListHead` (per-driver list of `detect()`-instantiated clients).
- `flags: u32` (I2C_DRV_ACPI_WAIVE_D0_PROBE).

REQ-4: `i2c_bus_type` (per-bus driver-model registration):
- `name = "i2c"`.
- `dev_groups = i2c_dev_groups` (name + modalias attributes).
- `match = i2c_device_match` (OF → ACPI → id_table, in that order).
- `uevent = i2c_device_uevent` (modalias).
- `probe = i2c_device_probe`.
- `remove = i2c_device_remove`.
- `shutdown = i2c_device_shutdown`.

REQ-5: `i2c_init()` boot init (postcore_initcall):
- `of_alias_get_highest_id("i2c")` → bump `__i2c_first_dynamic_bus_num` past static IDs.
- `bus_register(&i2c_bus_type)`.
- `is_registered = true` (gates `i2c_add_adapter` and `i2c_register_driver`).
- Create `i2c_debugfs_root = debugfs_create_dir("i2c", NULL)`.
- `i2c_add_driver(&dummy_driver)`.
- If `CONFIG_OF_DYNAMIC`: register `of_reconfig_notifier`.
- If `CONFIG_ACPI`: register `acpi_reconfig_notifier`.

REQ-6: `i2c_add_adapter(adap)`:
- Try `of_alias_get_id(dev->of_node, "i2c")` first → if ≥ 0, use as static `adap->nr` and call `__i2c_add_numbered_adapter`.
- Else: `mutex_lock(&core_lock); id = idr_alloc(&i2c_adapter_idr, adap, __i2c_first_dynamic_bus_num, 0, GFP_KERNEL); mutex_unlock`.
- `adap->nr = id`.
- `i2c_register_adapter(adap)`.

REQ-7: `i2c_register_adapter(adap)`:
- If `!is_registered`: return -EAGAIN.
- If `adap->name[0] == 0`: WARN; return -EINVAL.
- If `!adap->algo`: pr_err; return -EINVAL.
- If `!adap->lock_ops`: `adap->lock_ops = &i2c_adapter_lock_ops`.
- `adap->locked_flags = 0`.
- `rt_mutex_init(&adap->bus_lock)`.
- `rt_mutex_init(&adap->mux_lock)`.
- `mutex_init(&adap->userspace_clients_lock)`.
- `INIT_LIST_HEAD(&adap->userspace_clients)`.
- If `adap->timeout == 0`: `adap->timeout = HZ`.
- `i2c_setup_host_notify_irq_domain(adap)` (creates per-adapter `host_notify_domain`).
- `dev_set_name(&adap->dev, "i2c-%d", adap->nr)`.
- `adap->dev.bus = &i2c_bus_type; adap->dev.type = &i2c_adapter_type`.
- `device_initialize` → `device_enable_async_suspend` → `pm_runtime_no_callbacks` → `pm_suspend_ignore_children(true)` → `pm_runtime_enable`.
- `device_add(&adap->dev)`.
- `adap->debugfs = debugfs_create_dir(dev_name(&adap->dev), i2c_debugfs_root)`.
- `i2c_setup_smbus_alert(adap)`.
- `i2c_init_recovery(adap)` (allows -EPROBE_DEFER for GPIO recovery deferral).
- Enumerate pre-declared slaves: `of_i2c_register_devices(adap)`, `i2c_acpi_install_space_handler(adap)`, `i2c_acpi_register_devices(adap)`.
- If `adap->nr < __i2c_first_dynamic_bus_num`: `i2c_scan_static_board_info(adap)`.
- `bus_for_each_drv(&i2c_bus_type, NULL, adap, __process_new_adapter)` under `core_lock` (notify every i2c_driver of new adapter; runs `driver->detect` if `address_list`).

REQ-8: `i2c_del_adapter(adap)`:
- `idr_find(&i2c_adapter_idr, adap->nr) == adap` else return.
- `i2c_acpi_remove_space_handler(adap)`.
- `bus_for_each_drv(__process_removed_adapter)` under `core_lock` (releases `detect()`-instantiated clients).
- Walk `adap->userspace_clients` under `userspace_clients_lock` (nested by `i2c_adapter_depth`); `i2c_unregister_device` each.
- Two-pass `device_for_each_child(__unregister_client)` then `(__unregister_dummy)` (dummies last so real drivers can release them first).
- `pm_runtime_disable`.
- `i2c_host_notify_irq_teardown`.
- `debugfs_remove_recursive(adap->debugfs)`.
- `init_completion(&adap->dev_released); device_unregister(&adap->dev); wait_for_completion(&adap->dev_released)` (drain refs).
- `idr_remove(&i2c_adapter_idr, adap->nr)` under `core_lock`.
- `memset(&adap->dev, 0, sizeof(adap->dev))` (so re-add works).

REQ-9: `i2c_new_client_device(adap, info)`:
- Allocate `client` (kzalloc; ERR_PTR(-ENOMEM) on failure).
- `client->adapter = adap`.
- `client->dev.platform_data = info->platform_data`.
- `client->flags = info->flags; client->addr = info->addr`.
- `client->init_irq = info->irq` (or derive from `info->resources` via `i2c_dev_irq_from_resources`).
- `strscpy(client->name, info->type, I2C_NAME_SIZE)`.
- `i2c_check_addr_validity(client->addr, client->flags)` — 7-bit: [0x08, 0x77]; 10-bit: [0, 0x3ff].
- `i2c_lock_addr(adap, client->addr, client->flags)` — `test_and_set_bit(addr, adap->addrs_in_instantiation)` race lock (7-bit only).
- `i2c_check_addr_busy(adap, i2c_encode_flags_to_addr(client))` — walk mux parent + device_for_each_child.
- `client->dev.parent = &adap->dev; .bus = &i2c_bus_type; .type = &i2c_client_type`.
- `device_enable_async_suspend(&client->dev)`.
- `device_set_node(&client->dev, fwnode_handle_get(info->fwnode))`.
- If `info->swnode`: `device_add_software_node`.
- `i2c_dev_set_name`: per-`info->dev_name` → "i2c-<dev_name>"; else per-ACPI_COMPANION → "i2c-<acpi_dev_name>"; else "<adap-nr>-<addr-hex>".
- `device_register(&client->dev)` — triggers driver-core match → probe.
- `i2c_unlock_addr` (clear instantiation bit).

REQ-10: `i2c_unregister_device(client)`:
- If `is_of_node(fwnode)`: `of_node_clear_flag(OF_POPULATED)`.
- If `is_acpi_device_node(fwnode)`: `acpi_device_clear_enumerated`.
- `fwnode_handle_put(fwnode)` (skip if software-node, freed by `device_remove_software_node`).
- `device_remove_software_node` + `device_unregister`.

REQ-11: `i2c_register_driver(owner, driver)`:
- If `!is_registered`: return -EAGAIN.
- `driver->driver.owner = owner; .bus = &i2c_bus_type`.
- `INIT_LIST_HEAD(&driver->clients)`.
- `driver_register(&driver->driver)` (driver core matches against all unbound devices).
- `i2c_for_each_dev(driver, __process_new_driver)` — per-existing-adapter `i2c_do_add_adapter` (runs `driver->detect` against `address_list`).

REQ-12: `i2c_del_driver(driver)`:
- `i2c_for_each_dev(driver, __process_removed_driver)` — per-adapter `i2c_do_del_adapter` (releases `detect()`-instantiated clients).
- `driver_unregister(&driver->driver)`.

REQ-13: `i2c_device_match(dev, drv)` (bus.match) ordering:
- 1. `i2c_of_match_device(drv->of_match_table, client)` (DT).
- 2. `acpi_driver_match_device(dev, drv)` (ACPI).
- 3. `i2c_match_id(driver->id_table, client)` (legacy id_table).
- First hit wins. No-match → 0.

REQ-14: `i2c_device_probe(dev)`:
- IRQ resolution: `client->irq = client->init_irq`; if 0, derive per:
  - `I2C_CLIENT_HOST_NOTIFY`: `i2c_smbus_host_notify_to_irq`; `pm_runtime_get_sync(&adapter->dev)` to keep adapter active.
  - OF: `fwnode_irq_get_byname(fwnode, "irq")` then `fwnode_irq_get(fwnode, 0)`.
  - ACPI: `i2c_acpi_get_irq` (also detects wake-capable).
- If no `id_table` ∧ no ACPI match ∧ no OF match: return -ENODEV.
- If `I2C_CLIENT_WAKE`: query "wakeup" irq; `device_init_wakeup(true)`; `dev_pm_set_dedicated_wake_irq` or `dev_pm_set_wake_irq`.
- `of_clk_set_defaults`.
- `dev_pm_domain_attach(&client->dev, PD_FLAG_DETACH_POWER_OFF | (do_power_on ? PD_FLAG_ATTACH_POWER_ON : 0))` (do_power_on = !`i2c_acpi_waive_d0_probe`).
- `client->devres_group_id = devres_open_group`.
- `client->debugfs = debugfs_create_dir(dev_name, adapter->debugfs)`.
- `driver->probe(client)` — must be set, else -EINVAL.
- Note: devres group is intentionally left open so resources acquired post-probe (e.g. firmware loading) are still released on `i2c_device_remove`.

REQ-15: `i2c_device_remove(dev)`:
- `driver->remove(client)` if set.
- `debugfs_remove_recursive(client->debugfs)`.
- `devres_release_group(&client->dev, client->devres_group_id)`.
- `dev_pm_clear_wake_irq; device_init_wakeup(false)`.
- `client->irq = 0`.
- If `I2C_CLIENT_HOST_NOTIFY`: `pm_runtime_put(&adapter->dev)`.

REQ-16: `i2c_device_shutdown(dev)`:
- If `driver->shutdown`: call it.
- Else if `client->irq > 0`: `disable_irq(client->irq)` (defensive).

REQ-17: Address validation (`i2c_check_addr_validity`):
- 7-bit: must be in [I2C_ADDR_DEVICE_ID + 1 … I2C_ADDR_7BITS_MAX (0x77)] for general devices; addresses 0x00 (gen-call), 0x01 (CBUS), 0x02 (reserved-different-bus), 0x03–0x07 (reserved), 0x78–0x7B (10-bit slave), 0x7C–0x7F (DEVICE_ID + reserved) excluded.
- 10-bit: I2C_CLIENT_TEN flag; must be ≤ 0x3ff.
- Strict variant (`i2c_check_7bit_addr_validity_strict`) used by detect path.

REQ-18: `__i2c_transfer(adap, msgs, num)` (caller holds bus_lock):
- If `!adap->algo->master_xfer`: -EOPNOTSUPP.
- If `!msgs || num < 1`: WARN; -EINVAL.
- `__i2c_check_suspended(adap)` — bail if adapter suspended (PM).
- If `adap->quirks`: `i2c_check_for_quirks(adap, msgs, num)`.
- If `i2c_trace_msg_key` static-branch enabled: emit `trace_i2c_read` / `trace_i2c_write` per msg.
- `orig_jiffies = jiffies`.
- Loop `try ∈ [0, adap->retries]`:
  - If `i2c_in_atomic_xfer_mode()` ∧ `algo->master_xfer_atomic`: `master_xfer_atomic(adap, msgs, num)`.
  - Else: `algo->master_xfer(adap, msgs, num)`.
  - If `ret != -EAGAIN`: break.
  - If `time_after(jiffies, orig_jiffies + adap->timeout)`: break.
- If trace enabled: per-reply tracepoint + `trace_i2c_result(num, ret)`.
- Return ret.

REQ-19: `i2c_transfer(adap, msgs, num)`:
- `__i2c_lock_bus_helper(adap)` — `rt_mutex_lock_nested(&bus_lock, i2c_adapter_depth(adap))` honoring `lock_ops`.
- `__i2c_transfer(adap, msgs, num)`.
- `i2c_unlock_bus(adap, I2C_LOCK_SEGMENT)`.

REQ-20: `i2c_check_for_quirks(adap, msgs, num)`:
- `q = adap->quirks`.
- `max_num = q->max_num_msgs`.
- If `I2C_AQ_COMB` ∧ `num == 2`: per-`I2C_AQ_COMB_WRITE_FIRST`, `_COMB_READ_SECOND`, `_COMB_SAME_ADDR`, `q->max_comb_1st_msg_len`, `q->max_comb_2nd_msg_len`.
- If `num > max_num`: i2c_quirk_error("too many messages").
- Per-msg: if RD: `q->max_read_len`; else: `q->max_write_len`. Per-`I2C_AQ_NO_ZERO_LEN_READ` / `_WRITE`.
- Returns -EOPNOTSUPP on any violation.

REQ-21: Bus-lock model:
- Per-adapter `bus_lock: rt_mutex` (prio-inherit; PI-aware for RT).
- Per-`i2c_adapter_lock_bus`: `rt_mutex_lock_nested(&bus_lock, i2c_adapter_depth(adap))`.
- Per-`i2c_adapter_depth(adap)`: walks `i2c_parent_is_i2c_adapter` chain; lockdep needs subclass = depth for nested mux locks.
- Per-`mux_lock`: separate, serializes mux-parent ↔ mux-child operations.

REQ-22: Per-adapter idr:
- `i2c_adapter_idr` IDR (`core_lock` mutex guards).
- `i2c_get_adapter(nr)` → `idr_find` + `try_module_get(adapter->owner)` + `get_device(&adap->dev)`. Returns NULL on either failure.
- `i2c_put_adapter(adap)` → `module_put` then `put_device` (device-put last to avoid UAF).

REQ-23: 10-bit addressing:
- `I2C_CLIENT_TEN` flag on client.
- `i2c_encode_flags_to_addr` encodes TEN bit into `I2C_ADDR_OFFSET_TEN_BIT = 0xa000` OR-mask (sysfs name only).
- 10-bit address space: 0…0x3ff. Functionality reported via `I2C_FUNC_10BIT_ADDR` (algorithm-set).

REQ-24: Slave address (`I2C_CLIENT_SLAVE`):
- `I2C_ADDR_OFFSET_SLAVE = 0x1000` (sysfs disambiguation).
- Routed via `i2c-core-slave.c` (separate Tier-3).

REQ-25: Per-driver auto-detect (`i2c_detect`):
- `driver->detect` ∧ `driver->address_list` (terminated by `I2C_CLIENT_END = 0xfffe`).
- Skip if `adapter->class == I2C_CLASS_DEPRECATED` (warn).
- Require `adapter->class & driver->class`.
- Allocate temp_client; per addr: `i2c_check_7bit_addr_validity_strict`, `i2c_check_addr_busy`, `i2c_default_probe` (SMBus quick-write [0x30..0x37 / 0x50..0x5f use byte-read]; x86 + 0x73 hwmon uses byte-data), then `driver->detect(temp_client, &info)`. On success: `i2c_new_client_device(adap, &info)` and link into `driver->clients`.

REQ-26: Per-board enumeration:
- `i2c_register_board_info(busnum, info[], n)` (legacy boardinfo.c) populates a global list.
- `i2c_scan_static_board_info(adap)` consumes entries for static bus numbers at adapter-register time.
- Per-info: `i2c_new_client_device` per entry.

REQ-27: OF / ACPI enumeration entry-points:
- OF: `of_i2c_register_devices(adap)` — walks child OF nodes, builds `i2c_board_info`, calls `i2c_new_client_device`. Reentrant on `of_reconfig` (dynamic DT overlays).
- ACPI: `i2c_acpi_install_space_handler(adap)` (OpRegion for ACPI methods that read/write i2c), `i2c_acpi_register_devices(adap)` (walks ACPI namespace for slave nodes). Reentrant on `acpi_reconfig` notifier.

REQ-28: Per-stuck-bus recovery:
- `adap->bus_recovery_info` (optional): `recover_bus`, `prepare_recovery`, `unprepare_recovery`, `scl_gpiod`, `sda_gpiod`, `set_scl`, `set_sda`, `get_scl`, `get_sda`.
- `i2c_generic_scl_recovery`: bit-bang up to 9 clocks while reading SDA; abort once SDA goes high.
- `i2c_init_recovery(adap)`: if pinctrl-based: `i2c_gpio_init_pinctrl_recovery`; else if generic GPIO: `i2c_gpio_init_generic_recovery`. Returns -EPROBE_DEFER if GPIO not yet ready.

REQ-29: Per-adapter PM:
- `adap->dev` has `pm_runtime_no_callbacks` (controller PM is per-driver, not framework).
- `pm_suspend_ignore_children(true)` so client suspend doesn't trigger adapter suspend prematurely.
- Per-client `device_enable_async_suspend(true)`.
- Host-Notify probe acquires `pm_runtime_get_sync(&adapter->dev)` for active-keep.

REQ-30: Userspace `new_device` / `delete_device` sysfs:
- `/sys/bus/i2c/devices/i2c-<N>/new_device`: write "<chip-name> <addr-hex>" → `i2c_new_client_device` with `i2c_board_info{ .type = name, .addr = addr }`. Appends to `adap->userspace_clients`.
- `/sys/bus/i2c/devices/i2c-<N>/delete_device`: write "<addr-hex>" → `i2c_unregister_device` and remove from `userspace_clients`. Uses `DEVICE_ATTR_IGNORE_LOCKDEP` because of nested-mux locking.

REQ-31: SMBus Host-Notify:
- Per-adapter `irq_domain` (`linear` style, `I2C_ADDR_7BITS_COUNT` slots).
- `i2c_handle_smbus_host_notify(adap, addr)` → `generic_handle_irq` on the per-addr virq.
- `i2c_smbus_host_notify_to_irq(client)` resolves at probe.

REQ-32: DMA-safe message buffers:
- `i2c_get_dma_safe_msg_buf(msg, threshold)`: if `msg->flags & I2C_M_DMA_SAFE` return `msg->buf` (caller-stamped). Else for `len ≥ threshold` and `len > 0`: kmalloc-bounce (kzalloc for RD, kmemdup for WR).
- `i2c_put_dma_safe_msg_buf(buf, msg, xferred)`: on `xferred ∧ msg->flags & I2C_M_RD`: `memcpy(msg->buf, buf, msg->len)`. Then kfree.
- Threshold typically `adapter->dma_threshold` controller-specific.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `register_only_if_subsystem_up` | INVARIANT | per-Adapter::register / Driver::register: `is_registered == true` else Err(-EAGAIN). |
| `register_requires_algo` | INVARIANT | per-Adapter::register: `algo != NULL` else Err. |
| `register_initializes_bus_lock` | INVARIANT | per-Adapter::register: `bus_lock` initialized before any client probe. |
| `addr_busy_held_during_instantiation` | INVARIANT | per-Client::new: `addrs_in_instantiation[addr]` set across registration. |
| `idr_consistent_with_device` | INVARIANT | per-i2c_get_adapter: `idr_find(nr) == adap` ⟹ `module_get + device_get` paired. |
| `transfer_needs_master_xfer` | INVARIANT | per-Adapter::transfer: `algo->master_xfer != NULL` else -EOPNOTSUPP without state-mutation. |
| `transfer_retries_bounded` | INVARIANT | per-Adapter::transfer: `try ≤ retries` ∧ `(jiffies - orig_jiffies) ≤ timeout`. |
| `del_drains_refs_before_idr_remove` | INVARIANT | per-Adapter::del: `wait_for_completion(dev_released)` before `idr_remove`. |
| `unlock_addr_paired_with_lock_addr` | INVARIANT | per-Client::new: every path releases the addrs_in_instantiation bit. |

### Layer 2: TLA+

`drivers/i2c/core-base.tla`:
- Per-bus-init → per-adapter-add → per-driver-register → per-client-instantiate → per-transfer → per-adapter-del.
- Properties:
  - `safety_no_register_before_subsystem_init` — Adapter::add / Driver::register: only after `is_registered = true`.
  - `safety_addr_uniqueness_under_race` — concurrent Client::new with same (adap, addr): exactly one succeeds, other -EBUSY.
  - `safety_bus_lock_excludes_concurrent_xfer` — Adapter::transfer: serialized per-segment via bus_lock.
  - `safety_driver_clients_balanced` — `driver->clients` populated by detect, drained by `i2c_del_driver` / `i2c_del_adapter`.
  - `safety_adapter_del_drains_clients` — Adapter::del: every child client `i2c_unregister_device`'d before `idr_remove`.
  - `safety_idr_consistent` — `i2c_adapter_idr` matches set of live adapters.
  - `liveness_transfer_completes_or_times_out` — Adapter::transfer: returns within `adap->timeout`.
  - `liveness_adapter_del_completes` — `wait_for_completion(dev_released)` eventually returns (refcount → 0).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Adapter::register` post: `adap->dev` registered ∧ enumerated ∧ in `i2c_adapter_idr` | `Adapter::register_inner` |
| `Adapter::del` post: idr-remove ∧ all children unregistered ∧ `adap->dev` zeroed | `Adapter::del` |
| `Client::new` post: device_register'd ∨ ERR_PTR ∧ `addrs_in_instantiation[addr]` cleared on both paths | `Client::new` |
| `Adapter::transfer` post: returned ret reflects last try ∧ bus_lock dropped | `Adapter::transfer` |
| `Adapter::transfer_unlocked` post: tries ≤ retries + 1 ∧ elapsed ≤ timeout | `Adapter::transfer_unlocked` |
| `Driver::register` post: `driver_register` invoked ∧ per-adapter detect run | `Driver::register` |
| `i2c_device_match` post: returns 1 iff OF ∨ ACPI ∨ id_table matches | `Subsystem::device_match` |
| `i2c_check_addr_validity` post: rejects reserved 7-bit ranges; rejects > 0x3ff for 10-bit | `Adapter::check_addr_validity` |
| `i2c_check_for_quirks` post: ret 0 ∨ -EOPNOTSUPP; never mutates msgs | `Adapter::check_quirks` |

### Layer 4: Verus/Creusot functional

`Per-i2c_init → bus_register; per-i2c_add_adapter → idr_alloc + i2c_register_adapter (rt_mutex_init bus_lock, device_add, OF/ACPI/boardinfo enum, bus_for_each_drv); per-i2c_new_client_device → addr-validate + addr-lock + device_register (triggers match+probe); per-i2c_register_driver → driver_register + per-adapter detect; per-i2c_transfer → __i2c_lock_bus_helper + __i2c_transfer (algo->master_xfer or _atomic, retry on -EAGAIN bounded by retries and timeout); per-i2c_del_adapter → drain detect-clients + userspace-clients + driver-children + dummies + dev_released wait + idr_remove` semantic equivalence: per-Documentation/i2c/ + per-`include/linux/i2c.h` API contract.

### hardening

(Inherits row-1 features from `drivers/i2c/00-overview.md` § Hardening.)

I2C core-base reinforcement:

- **Per-is_registered gate** — defense against per-pre-bus-init driver/adapter register UAF.
- **Per-name + per-algo sanity at register** — defense against per-malformed-adapter NULL deref in transfer.
- **Per-rt_mutex bus_lock with nested-subclass = depth** — defense against per-mux-tree lockdep splat + per-priority-inversion under RT.
- **Per-addrs_in_instantiation bitmap** — defense against per-concurrent-new_device duplicate-instantiate race.
- **Per-7-bit-reserved-address rejection (0x00–0x07, 0x78–0x7f)** — defense against per-SMBus-reserved-collision.
- **Per-`i2c_check_addr_busy` walks mux parent + children** — defense against per-mux-leaf addr-collision.
- **Per-`__i2c_check_suspended` in transfer** — defense against per-transfer-during-suspend UAF or wedged HW.
- **Per-`i2c_check_for_quirks` enforces adapter HW limits** — defense against per-msg-too-long HW lockup / silent corruption.
- **Per-retry bound + per-timeout jiffies bound** — defense against per-unbounded-EAGAIN livelock.
- **Per-`module_get` before `get_device` in `i2c_get_adapter`** — defense against per-adapter-module-unload UAF.
- **Per-`dev_released` completion drain** — defense against per-adapter-deregister-while-in-use UAF.
- **Per-dev_err_ratelimited on quirk-violation** — defense against per-log-flood.
- **Per-detect class-match required (`adapter->class & driver->class`)** — defense against per-cross-class auto-instantiate (HWMON probing a non-HWMON adapter).
- **Per-`I2C_CLASS_DEPRECATED` warn-at-detect** — defense against per-silent-legacy-instantiate.
- **Per-DMA-safe bounce-buffer path** — defense against per-stack-buffer-DMA UAF.

