# Tier-3: drivers/i2c/i2c-smbus.c — SMBus alert / Host-Notify / SPD

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/i2c/00-overview.md
upstream-paths:
  - drivers/i2c/i2c-smbus.c (~493 lines)
  - include/linux/i2c-smbus.h
  - include/linux/i2c.h (i2c_smbus_xfer / _read_byte* / _write_byte* / PEC — defined in i2c-core-smbus.c)
  - Documentation/i2c/smbus-protocol.rst
-->

## Summary

`drivers/i2c/i2c-smbus.c` is **not** the SMBus protocol-level transaction helper file — those live in `drivers/i2c/i2c-core-smbus.c` (`i2c_smbus_xfer`, `i2c_smbus_read_byte`, `i2c_smbus_write_byte`, `i2c_smbus_read_word_data`, `i2c_smbus_write_word_data`, `i2c_smbus_read_block_data`, `i2c_smbus_write_block_data`, `i2c_smbus_process_call`, `i2c_smbus_xfer_emulated`, plus PEC append/validate via `i2c_smbus_pec` / `i2c_smbus_msg_pec` using the CRC-8 polynomial 0x07 lookup table). What `i2c-smbus.c` provides on top of those primitives is the **higher-level event infrastructure** specified by SMBus 2.0/3.0: (1) the **SMBus alert** mechanism — an open-drain SMBALERT# line that a slave can pull low to request the host's attention, after which the host issues an ARA (Alert Response Address, 0x0C) read to find out which device alerted, plus the framework to register an IRQ or threaded GPIO handler, schedule a sleeping worker, walk every child of the adapter looking for the alerting device, and call its driver's `.alert` callback; (2) the **SMBus Host Notify protocol** (under `CONFIG_I2C_SLAVE`) — a slave-mode pseudo-device at address 0x08 that receives 3-byte messages (host-notify reserved address + 2 data bytes) from peripherals advertising themselves as bus masters, dispatching them via `i2c_handle_smbus_host_notify`; and (3) the **SPD instantiation helper** (under `CONFIG_DMI`) — boot-time auto-probing of DDR/LPDDR EEPROMs at addresses 0x50..0x57 driven by SMBIOS Type 17 memory-device tables, picking the correct EEPROM driver (`spd` / `ee1004` / `spd5118`) for the detected DRAM type and honouring a write-disabled flag for DDR5/LPDDR5 write-protect policy. Critical for: thermal-fault interrupts on every server SMBus, PMBus/UCD9xxx voltage-monitor alerts, battery-controller wake-ups on laptops, push-button mailbox messages from EC firmware, and auto-detection of DIMM SPD modules feeding lm-sensors / DDR debug tools.

This Tier-3 covers `drivers/i2c/i2c-smbus.c` (~493 lines). Pure SMBus-protocol byte/word/block/PEC helpers in `i2c-core-smbus.c` are covered in `drivers/i2c/core-base.md`.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct i2c_smbus_alert` | per-adapter alert state (work + ARA client) | `I2cSmbusAlert` |
| `struct alert_data` | per-event payload (addr, type, data) | `AlertData` |
| `struct i2c_smbus_alert_setup` | per-platform-data setup (irq) | `SmbusAlertSetup` |
| `smbus_do_alert()` | per-child iter: match-addr → call driver.alert | `Smbus::do_alert_one` |
| `smbus_do_alert_force()` | per-child iter: call driver.alert unconditionally (fallback) | `Smbus::do_alert_force` |
| `smbus_alert()` | per-IRQ-context: read ARA, dispatch | `Smbus::alert_thread` |
| `smbalert_work()` | per-workqueue context: same body as smbus_alert | `Smbus::alert_work` |
| `smbalert_probe()` | per-i2c_client probe: parse IRQ source, request_threaded_irq | `Smbus::alert_probe` |
| `smbalert_remove()` | per-remove: cancel work | `Smbus::alert_remove` |
| `smbalert_driver` | per-i2c_driver registered via module_i2c_driver | `Smbus::driver` |
| `smbalert_ids` | per-i2c_device_id `"smbus_alert"` | constant |
| `i2c_handle_smbus_alert()` | per-bus-driver-irq: schedule_work | `Smbus::handle_alert` |
| `struct i2c_slave_host_notify_status` | per-host-notify slave parse state | `HostNotifyStatus` |
| `i2c_slave_host_notify_cb()` | per-slave-event FSM | `Smbus::host_notify_cb` |
| `i2c_new_slave_host_notify_device()` | per-create 0x08 slave + register | `Smbus::new_slave_host_notify` |
| `i2c_free_slave_host_notify_device()` | per-tear down | `Smbus::free_slave_host_notify` |
| `i2c_register_spd_write_disable()` | per-write-disabled SPD instantiation | `Smbus::register_spd_wdis` |
| `i2c_register_spd_write_enable()` | per-write-enabled SPD instantiation | `Smbus::register_spd_wen` |
| `i2c_register_spd()` (static) | per-DMI-walk per-DRAM-type pick + probe-instantiate | `Smbus::register_spd` |
| `I2C_PROTOCOL_SMBUS_ALERT` | per-alert protocol enum value | shared |
| `I2C_PROTOCOL_SMBUS_HOST_NOTIFY` | per-host-notify protocol enum value | shared |
| `SMBUS_HOST_NOTIFY_LEN` (3) | per-host-notify message length | constant |

(For the byte/word/block/quick/PEC helpers — `i2c_smbus_xfer`, `i2c_smbus_read_byte`, `i2c_smbus_write_byte`, `i2c_smbus_read_byte_data`, `i2c_smbus_write_byte_data`, `i2c_smbus_read_word_data`, `i2c_smbus_write_word_data`, `i2c_smbus_read_block_data`, `i2c_smbus_write_block_data`, `i2c_smbus_process_call`, `i2c_smbus_write_quick`, `i2c_smbus_pec`, `i2c_smbus_msg_pec` — see `drivers/i2c/core-base.md`. These live in `drivers/i2c/i2c-core-smbus.c`, not in this file.)

## Compatibility contract

REQ-1: struct i2c_smbus_alert (per-adapter alert framework):
- alert: per-`struct work_struct` queued from IRQ context to a sleeping worker.
- ara: per-`*i2c_client` representing the ARA (Alert Response Address) pseudo-device at 0x0C on the adapter.

REQ-2: struct alert_data (per-dispatch payload):
- addr: per-7-bit address of the device that responded to the ARA read.
- type: per-`enum i2c_alert_protocol` (I2C_PROTOCOL_SMBUS_ALERT or I2C_PROTOCOL_SMBUS_HOST_NOTIFY).
- data: per-1-bit "flag" embedded in the ARA response (LSB) or per-host-notify-data.

REQ-3: smbus_do_alert(dev, addrp):
- /* per-child callback used by device_for_each_child during ARA dispatch */
- client = i2c_verify_client(dev); if !client: return 0.
- data = addrp.
- if client.addr != data.addr: return 0.       /* not the alerting device */
- if client.flags & I2C_CLIENT_TEN: return 0.    /* 10-bit clients excluded (SMBus 2.0 reserved) */
- device_lock(dev).
- if client.dev.driver:
  - driver = to_i2c_driver(client.dev.driver).
  - if driver.alert: driver.alert(client, data.type, data.data); ret = -EBUSY (signals "stop iter").
  - else: dev_warn "no driver alert()!"; ret = -EOPNOTSUPP.
- else: dev_dbg "alert with no driver"; ret = -ENODEV.
- device_unlock(dev).
- return ret.

REQ-4: smbus_do_alert_force(dev, addrp) — fallback when REQ-3 loop stalls:
- client = i2c_verify_client(dev); if !client ∨ I2C_CLIENT_TEN: return 0.
- device_lock(dev).
- if client.dev.driver:
  - driver = to_i2c_driver(client.dev.driver).
  - if driver.alert: driver.alert(client, data.type, data.data).
- device_unlock(dev).
- return 0.   /* never -EBUSY: keep iterating all children */

REQ-5: smbus_alert(irq, d) — IRQ thread / work body:
- alert = d; ara = alert.ara.
- prev_addr = I2C_CLIENT_END (sentinel, not a valid address).
- loop:
  - status = i2c_smbus_read_byte(ara).        /* ARA read: 7-bit addr + flag */
  - if status < 0: break.                     /* no more pending devices */
  - data.data = status & 1.
  - data.addr = status >> 1.
  - data.type = I2C_PROTOCOL_SMBUS_ALERT.
  - dev_dbg "SMBALERT# from dev 0x<addr>, flag <data>".
  - status = device_for_each_child(&ara.adapter.dev, &data, smbus_do_alert).
  - /* If the same addr replied twice and no driver handled it, fall through to "force" */
  - if data.addr == prev_addr ∧ status != -EBUSY:
    - device_for_each_child(&ara.adapter.dev, &data, smbus_do_alert_force).
    - break.
  - prev_addr = data.addr.
- return IRQ_HANDLED.

REQ-6: smbalert_work(work):
- alert = container_of(work, i2c_smbus_alert, alert).
- smbus_alert(0, alert).
- /* used by bus drivers that detect SMBALERT# in their own ISR and call i2c_handle_smbus_alert from non-sleepable context */.

REQ-7: smbalert_probe(ara) — per-i2c_client.probe:
- setup = dev_get_platdata(&ara.dev).
- alert = devm_kzalloc(&ara.dev, sizeof(*alert), GFP_KERNEL); on null → -ENOMEM.
- irqflags = IRQF_SHARED | IRQF_ONESHOT.
- if setup: irq = setup.irq.
- else:
  - irq = fwnode_irq_get_byname(dev_fwnode(adapter.dev.parent), "smbus_alert").
  - if irq <= 0:
    - gpiod = devm_gpiod_get(adapter.dev.parent, "smbalert", GPIOD_IN).
    - on IS_ERR(gpiod): return PTR_ERR(gpiod).
    - irq = gpiod_to_irq(gpiod); if irq <= 0: return irq.
    - irqflags |= IRQF_TRIGGER_FALLING.       /* GPIO path: edge-triggered on falling edge of open-drain SMBALERT# */
- INIT_WORK(&alert.alert, smbalert_work).
- alert.ara = ara.
- if irq > 0: devm_request_threaded_irq(&ara.dev, irq, NULL, smbus_alert, irqflags, "smbus_alert", alert).
- i2c_set_clientdata(ara, alert).
- dev_info "supports SMBALERT#".
- return 0.

REQ-8: smbalert_remove(ara):
- alert = i2c_get_clientdata(ara).
- cancel_work_sync(&alert.alert).
- /* IRQ and memory are devm-managed; freed automatically */.

REQ-9: i2c_handle_smbus_alert(ara) — exported entry point for bus drivers' own IRQ paths:
- alert = i2c_get_clientdata(ara).
- return schedule_work(&alert.alert).

REQ-10: smbalert_driver — i2c_driver:
- name: "smbus_alert".
- probe: smbalert_probe.
- remove: smbalert_remove.
- id_table: smbalert_ids = { "smbus_alert" }.
- Registered via module_i2c_driver(smbalert_driver).

REQ-11: SMBus Host Notify (CONFIG_I2C_SLAVE) — protocol:
- Reserved address 0x08, master-mode message from a peripheral wishing to notify the host.
- Message format: 1 address byte + 2 data bytes (LSB, MSB) — SMBUS_HOST_NOTIFY_LEN == 3.
- Reception modelled as a slave-mode receive on address 0x08.

REQ-12: struct i2c_slave_host_notify_status:
- index: per-byte counter (0..255).
- addr: per-captured address (first byte received).

REQ-13: i2c_slave_host_notify_cb(client, event, val) — slave-mode FSM:
- status = client.dev.platform_data.
- on I2C_SLAVE_WRITE_RECEIVED:
  - if status.index == 0: status.addr = *val.
  - if status.index < U8_MAX: status.index++.
- on I2C_SLAVE_STOP:
  - if status.index == SMBUS_HOST_NOTIFY_LEN: i2c_handle_smbus_host_notify(client.adapter, status.addr).
  - fallthrough.
- on I2C_SLAVE_WRITE_REQUESTED: status.index = 0.
- on I2C_SLAVE_READ_REQUESTED / _READ_PROCESSED: *val = 0xff.   /* host-notify is master→host only, never solicited */
- return 0.

REQ-14: i2c_new_slave_host_notify_device(adapter):
- board_info = I2C_BOARD_INFO("smbus_host_notify", 0x08) | I2C_CLIENT_SLAVE.
- status = kzalloc(i2c_slave_host_notify_status); on null → ERR_PTR(-ENOMEM).
- board_info.platform_data = status.
- client = i2c_new_client_device(adapter, &board_info); on IS_ERR: kfree(status); return client.
- ret = i2c_slave_register(client, i2c_slave_host_notify_cb); on failure: i2c_unregister_device + kfree(status); return ERR_PTR(ret).
- return client.

REQ-15: i2c_free_slave_host_notify_device(client):
- if IS_ERR_OR_NULL(client): return.
- i2c_slave_unregister(client).
- kfree(client.dev.platform_data).
- i2c_unregister_device(client).

REQ-16: SPD instantiation (CONFIG_DMI) — i2c_register_spd(adap, write_disabled):
- Walk dmi_memdev_handle(0..) until 0xffff:
  - skip slots with mem_size == 0 (empty) or mem_type <= 0x02 (Invalid/Other/Unknown).
  - first non-skipped: common_mem_type = mem_type.
  - subsequent: if mem_type != common_mem_type: dev_warn "Different memory types mixed"; return.
  - dimm_count++.
- if dimm_count == 0: return.
- slot_count = min(slot_count, 8).
- Select EEPROM driver by common_mem_type:
  - 0x12..0x18, 0x1B..0x1D (DDR/DDR2/DDR3/LPDDR/LPDDR2/LPDDR3): name = "spd".
  - 0x1A, 0x1E (DDR4/LPDDR4): name = "ee1004".
  - 0x22, 0x23 (DDR5/LPDDR5): name = "spd5118"; instantiate = !write_disabled.   /* write-protect honoured */
  - default: dev_info "Memory type 0x<MM> not supported yet"; return.
- For n in 0..slot_count while dimm_count > 0:
  - info.type = name; addr_list = { 0x50 + n, I2C_CLIENT_END }.
  - if !instantiate: continue.
  - i2c_new_scanned_device(adap, &info, addr_list, NULL); on success: dimm_count--; dev_info "Successfully instantiated SPD at 0x<AA>".

REQ-17: i2c_register_spd_write_disable(adap) / _write_enable(adap):
- Thin wrappers calling i2c_register_spd(adap, true) / (adap, false). Both EXPORT_SYMBOL_GPL.

REQ-18: Module:
- MODULE_LICENSE("GPL"), MODULE_AUTHOR("Jean Delvare"), MODULE_DESCRIPTION("SMBus protocol extensions support").

## Acceptance Criteria

- [ ] AC-1: A slave with `.alert` callback receives `(client, I2C_PROTOCOL_SMBUS_ALERT, flag)` after pulling SMBALERT# and the host issuing the ARA read.
- [ ] AC-2: smbus_do_alert returns -EBUSY after a successful driver dispatch — terminates `device_for_each_child` iteration on the matching address.
- [ ] AC-3: A repeating same-address ARA reply with no handling driver triggers smbus_do_alert_force fallback exactly once, then loop terminates.
- [ ] AC-4: 10-bit clients (I2C_CLIENT_TEN) are skipped during ARA dispatch.
- [ ] AC-5: smbalert_probe with platform-data setup: uses setup.irq.
- [ ] AC-6: smbalert_probe with adapter fwnode "smbus_alert": uses that IRQ.
- [ ] AC-7: smbalert_probe with neither: gets `smbalert` GPIO, uses gpiod_to_irq with IRQF_TRIGGER_FALLING.
- [ ] AC-8: smbalert_probe sets devm-managed threaded IRQ with IRQF_SHARED | IRQF_ONESHOT.
- [ ] AC-9: i2c_handle_smbus_alert schedules work; the worker performs the same ARA-read + dispatch loop as the threaded IRQ.
- [ ] AC-10: smbalert_remove cancels pending work synchronously.
- [ ] AC-11: Host-notify slave at 0x08: 3-byte write produces `i2c_handle_smbus_host_notify(adapter, addr)`; <3 bytes ignored.
- [ ] AC-12: Host-notify slave: read events return 0xff (per protocol — never solicited).
- [ ] AC-13: i2c_new_slave_host_notify_device: on i2c_slave_register failure, status freed and client unregistered.
- [ ] AC-14: i2c_register_spd: mixed DRAM types → dev_warn and bail without instantiation.
- [ ] AC-15: i2c_register_spd: DDR5 + write_disabled=true → no instantiation; DDR5 + write_disabled=false → spd5118 instantiated.
- [ ] AC-16: i2c_register_spd: slot_count > 8 → capped at 8.

## Architecture

```
struct I2cSmbusAlert {
  alert: WorkStruct,
  ara: *I2cClient,           // ARA pseudo-device at addr 0x0C
}

struct AlertData {
  addr: u16,
  type: I2cAlertProtocol,    // SMBUS_ALERT | SMBUS_HOST_NOTIFY
  data: u32,
}

struct SmbusAlertSetup {
  irq: i32,                  // platform-data: explicit IRQ
}

struct I2cSlaveHostNotifyStatus {
  index: u8,
  addr: u8,
}
```

`Smbus::alert_probe(ara) -> i32`:
1. setup = platdata(ara).
2. alert = devm_kzalloc(ara.dev, I2cSmbusAlert, GFP_KERNEL); on null → -ENOMEM.
3. irqflags = IRQF_SHARED | IRQF_ONESHOT.
4. if setup: irq = setup.irq.
5. else:
   - irq = fwnode_irq_get_byname(adapter.dev.parent fwnode, "smbus_alert").
   - if irq <= 0:
     - gpiod = devm_gpiod_get(adapter.dev.parent, "smbalert", GPIOD_IN).
     - if Err: return Err.
     - irq = gpiod_to_irq(gpiod). if irq <= 0: return irq.
     - irqflags |= IRQF_TRIGGER_FALLING.
6. INIT_WORK(alert.alert, alert_work).
7. alert.ara = ara.
8. if irq > 0: devm_request_threaded_irq(ara.dev, irq, NULL, alert_thread, irqflags, "smbus_alert", alert).
9. set_clientdata(ara, alert).
10. dev_info "supports SMBALERT#".
11. return 0.

`Smbus::alert_thread(irq, alert) -> IrqReturn`:
1. ara = alert.ara; prev_addr = I2C_CLIENT_END.
2. loop:
   - status = i2c_smbus_read_byte(ara).
   - if status < 0: break.
   - data = AlertData { addr: status >> 1, data: status & 1, type: SMBUS_ALERT }.
   - status = device_for_each_child(&ara.adapter.dev, &data, Smbus::do_alert_one).
   - if data.addr == prev_addr ∧ status != -EBUSY:
     - device_for_each_child(&ara.adapter.dev, &data, Smbus::do_alert_force).
     - break.
   - prev_addr = data.addr.
3. return IRQ_HANDLED.

`Smbus::do_alert_one(dev, addrp) -> i32`:
1. client = i2c_verify_client(dev); if None: return 0.
2. data = addrp.
3. if client.addr != data.addr: return 0.
4. if client.flags & I2C_CLIENT_TEN: return 0.
5. device_lock(dev).
6. ret = match client.dev.driver:
   - Some(d) ∧ d.alert: d.alert(client, data.type, data.data); ret = -EBUSY.
   - Some(d): dev_warn "no driver alert()"; ret = -EOPNOTSUPP.
   - None: dev_dbg "alert with no driver"; ret = -ENODEV.
7. device_unlock(dev).
8. return ret.

`Smbus::do_alert_force(dev, addrp) -> i32`:
1. client = i2c_verify_client(dev); if None ∨ I2C_CLIENT_TEN: return 0.
2. device_lock(dev).
3. if client.dev.driver:
   - d = to_i2c_driver(client.dev.driver). if d.alert: d.alert(client, data.type, data.data).
4. device_unlock(dev).
5. return 0.

`Smbus::alert_work(work)`:
1. alert = container_of(work, I2cSmbusAlert, alert).
2. alert_thread(0, alert).

`Smbus::handle_alert(ara) -> i32`:
1. alert = get_clientdata(ara).
2. return schedule_work(alert.alert).

`Smbus::host_notify_cb(client, event, val) -> i32`:
1. status = client.dev.platform_data.
2. match event:
   - I2C_SLAVE_WRITE_RECEIVED:
     - if status.index == 0: status.addr = *val.
     - if status.index < U8_MAX: status.index += 1.
   - I2C_SLAVE_STOP:
     - if status.index == SMBUS_HOST_NOTIFY_LEN: i2c_handle_smbus_host_notify(client.adapter, status.addr).
     - fallthrough to I2C_SLAVE_WRITE_REQUESTED.
   - I2C_SLAVE_WRITE_REQUESTED: status.index = 0.
   - I2C_SLAVE_READ_REQUESTED | _READ_PROCESSED: *val = 0xff.
3. return 0.

`Smbus::new_slave_host_notify(adapter) -> Result<*I2cClient>`:
1. info = I2C_BOARD_INFO("smbus_host_notify", 0x08) | I2C_CLIENT_SLAVE.
2. status = kzalloc(I2cSlaveHostNotifyStatus). on null → Err(-ENOMEM).
3. info.platform_data = status.
4. client = i2c_new_client_device(adapter, &info). if Err: kfree(status); return client.
5. ret = i2c_slave_register(client, host_notify_cb). on err: i2c_unregister_device(client); kfree(status); return Err(ret).
6. return Ok(client).

`Smbus::free_slave_host_notify(client)`:
1. if IS_ERR_OR_NULL(client): return.
2. i2c_slave_unregister(client).
3. kfree(client.dev.platform_data).
4. i2c_unregister_device(client).

`Smbus::register_spd(adap, write_disabled)`:
1. slot_count = 0; dimm_count = 0; common_mem_type = 0; instantiate = true.
2. loop handle = dmi_memdev_handle(slot_count); while handle != 0xffff:
   - slot_count += 1.
   - mem_size = dmi_memdev_size(handle). if 0: continue.
   - mem_type = dmi_memdev_type(handle). if mem_type <= 2: continue.
   - if common_mem_type == 0: common_mem_type = mem_type.
   - else if mem_type != common_mem_type: dev_warn "Different memory types mixed"; return.
   - dimm_count += 1.
3. if dimm_count == 0: return.
4. slot_count = min(slot_count, 8).
5. name = match common_mem_type:
   - 0x12 | 0x13 | 0x18 | 0x1B | 0x1C | 0x1D → "spd".
   - 0x1A | 0x1E → "ee1004".
   - 0x22 | 0x23 → "spd5118"; instantiate = !write_disabled.
   - _ → dev_info "Memory type 0x<MM> not supported yet"; return.
6. for n in 0..slot_count while dimm_count > 0:
   - info = I2cBoardInfo { type: name, ... }.
   - addr_list = [0x50 + n, I2C_CLIENT_END].
   - if !instantiate: continue.
   - if i2c_new_scanned_device(adap, &info, addr_list, NULL).is_ok(): dimm_count -= 1; dev_info "Successfully instantiated SPD at 0x<AA>".

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `alert_iter_terminates_on_minus1` | INVARIANT | per-alert_thread: ARA read returning <0 exits the loop. |
| `alert_iter_terminates_on_repeat` | INVARIANT | per-alert_thread: repeated same address with no handler → fallback dispatch + break. |
| `do_alert_locks_device` | INVARIANT | per-do_alert_one/_force: device_lock held when dereferencing client.dev.driver. |
| `do_alert_skip_ten_bit` | INVARIANT | per-do_alert_*: I2C_CLIENT_TEN clients are skipped. |
| `alert_probe_devm_paths` | INVARIANT | per-alert_probe: every alloc / IRQ request is devm-managed; explicit free unnecessary. |
| `host_notify_index_bound` | INVARIANT | per-host_notify_cb: status.index never exceeds U8_MAX. |
| `host_notify_dispatch_iff_len` | INVARIANT | per-host_notify_cb on STOP: i2c_handle_smbus_host_notify called iff index == 3. |
| `new_slave_host_notify_no_leak` | INVARIANT | per-new_slave_host_notify: failure paths free status and unregister client. |
| `register_spd_max_8_slots` | INVARIANT | per-register_spd: slot_count capped at 8 before EEPROM-probe loop. |
| `register_spd_mixed_type_aborts` | INVARIANT | per-register_spd: mixed memory types → return without instantiation. |
| `spd5118_write_disabled_honored` | INVARIANT | per-register_spd: DDR5/LPDDR5 + write_disabled → instantiate == false. |
| `handle_alert_schedules_work` | INVARIANT | per-handle_alert: schedule_work returns and work executes alert_thread body. |

### Layer 2: TLA+

`drivers/i2c/i2c-smbus.tla`:
- Per-alert-IRQ + per-ARA-read + per-child-dispatch + per-driver-alert lifecycle.
- Per-host-notify slave FSM (REQUESTED → RECEIVED* → STOP).
- Properties:
  - `safety_alert_loop_terminates` — per-alert_thread: bounded by ARA-replies-distinct + at-most-one-force.
  - `safety_alert_force_at_most_once` — per-alert_thread: smbus_do_alert_force runs at most once per IRQ.
  - `safety_host_notify_dispatch_exactly_iff_three_bytes` — per-host_notify_cb STOP: dispatch iff index == 3.
  - `safety_alert_skip_10bit` — per-do_alert_*: 10-bit clients never dispatched.
  - `safety_register_spd_homogeneous` — per-register_spd: at completion, instantiated EEPROM type matches single common_mem_type.
  - `liveness_alert_eventually_dispatched` — per-IRQ assertion: smbalert_work scheduling → alert_thread body runs within finite steps.
  - `liveness_alert_clears_smbalert_line` — per-alert_thread: SMBALERT# is deasserted when the offending device's ARA response has been answered (driver-level invariant).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Smbus::alert_probe` post: alert allocated, work initialised, IRQ requested when irq > 0, clientdata set | `Smbus::alert_probe` |
| `Smbus::alert_thread` post: ret == IRQ_HANDLED; loop bounded; force fallback at most once | `Smbus::alert_thread` |
| `Smbus::do_alert_one` post: -EBUSY iff dispatched; -EOPNOTSUPP iff driver lacks .alert; -ENODEV iff no driver; 0 iff non-match / 10-bit | `Smbus::do_alert_one` |
| `Smbus::handle_alert` post: schedule_work returned | `Smbus::handle_alert` |
| `Smbus::host_notify_cb` post: per-REQ-13 state transitions | `Smbus::host_notify_cb` |
| `Smbus::new_slave_host_notify` post: on Ok, slave registered with host_notify_cb at addr 0x08; on Err, all partial state freed | `Smbus::new_slave_host_notify` |
| `Smbus::free_slave_host_notify` post: slave unregistered, platform_data freed, client unregistered | `Smbus::free_slave_host_notify` |
| `Smbus::register_spd` post: at most slot_count <= 8 EEPROM probes; only common_mem_type-matching driver chosen | `Smbus::register_spd` |

### Layer 4: Verus/Creusot functional

`Per-SMBALERT# assertion → ARA-read → device_for_each_child(do_alert) → driver.alert → ARA-deassert → loop until ARA returns <0 (or repeat-address fallback) → IRQ_HANDLED` and `Per-host-notify slave-write of 3 bytes → i2c_handle_smbus_host_notify(adapter, addr)` and `Per-DMI-walk → homogeneous DRAM type → 0x50+n EEPROM instantiation (write-disabled gates DDR5)` semantic equivalence vs Documentation/i2c/smbus-protocol.rst and SMBus 2.0/3.0 specifications + DMTF DSP0134 SMBIOS Memory Device table.

## Hardening

(Inherits row-1 features from `drivers/i2c/00-overview.md` § Hardening.)

i2c-smbus reinforcement:

- **Per-alert force-fallback at-most-once** — defense against per-runaway-IRQ caused by a misbehaving slave that latches SMBALERT# without responding to ARA.
- **Per-10-bit-client skipped in alert dispatch** — defense against per-spec-violation (SMBus 2.0 reserves 10-bit for future use).
- **Per-device_lock around driver.alert** — defense against per-UAF when driver unbinds mid-dispatch.
- **Per-cancel_work_sync on remove** — defense against per-stale-work after smbalert client unbinds.
- **Per-devm-managed IRQ + memory** — defense against per-resource leak on probe-failure or rebind storms.
- **Per-IRQF_ONESHOT** — defense against per-re-entrant alert handler racing with the worker.
- **Per-IRQF_SHARED** — required for shared SMBALERT# wired to multiple controllers on the same line.
- **Per-host-notify length-strict (exactly 3 bytes)** — defense against per-malicious-master injecting short/long writes to spoof notifications.
- **Per-host-notify read returns 0xff** — defense against per-spec-violation (host-notify is master→host only; read is meaningless).
- **Per-SPD homogeneous-type check** — defense against per-mis-instantiation of the wrong EEPROM driver on mixed-DIMM systems.
- **Per-SPD slot_count capped at 8** — defense against per-out-of-bounds I2C address probe (0x50..0x57 only).
- **Per-DDR5 write_disabled honoured** — defense against per-accidental SPD5118 NVM corruption on write-protected platforms.
- **Per-IS_ERR_OR_NULL guard on free_slave_host_notify** — defense against per-double-free.
- **Per-i2c_verify_client guard in alert iter** — defense against per-non-i2c-child accidentally on the adapter's device list.

## Open Questions

- The `device_lock` taken inside `smbus_do_alert` overlaps with the alerting slave's own driver-bind lock; if the alerting device is currently being unbound, the ARA dispatch can serialise behind unbind. Worth noting in the Rust trait that `.alert` callbacks must not call back into the driver core registration paths. Out of scope for parity, queue for ergonomics post-parity.
- SMBus 3.0 32-bit register accesses (PEC-protected) are emulated through `i2c_smbus_xfer_emulated` in `i2c-core-smbus.c`; this file is unaware of the wider width. No action needed at this Tier-3.
- DMI memory-type table values 0x24+ (LPDDR5X, future DDR generations) currently fall into the `default: dev_info "not supported yet"` arm. Track upstream for additions.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `i2c_smbus_alert`, `alert_data`, and PEC scratch buffers; emulated 32-bit SMBus 3.0 accesses bounded against `I2C_SMBUS_BLOCK_MAX`.
- **PAX_KERNEXEC** — SMBus core in W^X kernel text; ARA host-notify dispatch table and `i2c_driver.alert` callbacks not patchable at runtime.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `smbus_do_alert`, host-notify IRQ paths, and `i2c_smbus_xfer_emulated` so PEC retry windows cannot be stack-groomed.
- **PAX_REFCOUNT** — saturating `refcount_t` on `i2c_smbus_alert_setup` host-notify devices and per-client alert references; overflow trap defeats unbind-vs-alert UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for SPD payload buffers and host-notify alert records, so prior DIMM-serial / vendor-NV data never bleeds across reuses.
- **PAX_UDEREF** — SMAP/PAN enforced on any userland-driven SMBus path (`I2C_SMBUS` ioctl) feeding into emulated transfers.
- **PAX_RAP / kCFI** — `i2c_driver.alert`, ARA dispatch callbacks, and `smbus_host_notify`/`smbalert_irq` handlers marked `__ro_after_init`-style and kCFI-typed; SMBALERT vtable confusion cannot redirect to attacker text.
- **GRKERNSEC_HIDESYM** — gate kallsyms and per-alert client pointer leaks behind CAP_SYSLOG; suppress `%p` in ARA dispatch traces.
- **GRKERNSEC_DMESG** — restrict SPD parse warnings, PEC mismatch banners, and DDR write-disable diagnostics to CAP_SYSLOG so attackers cannot fingerprint DIMM topology via dmesg.
- **SMBus-alert CAP_SYS_RAWIO** — userland injection of synthetic alerts (test paths) requires CAP_SYS_RAWIO; ARA polling refuses to dispatch to `i2c_client` whose `dev->driver` is mid-unbind.
- **PEC enforced** — `I2C_SMBUS_PEC` honoured unconditionally on adapters advertising it; silent PEC-off downgrade refused.
- **SPD write protection** — DDR5 SPD5118 `write_disabled` strict gate; SPD NV-write paths require CAP_SYS_RAWIO and platform opt-in.
- **`i2c_verify_client` in alert iter** — every host-notify walk verifies the child is an actual `i2c_client` (not a foreign device on the parent's child list) to defeat type-confusion across the adapter device tree.
- **Host-notify IRQ bounds** — ARA addresses (0x0c) hard-coded; bogus address-claim attempts via `I2C_SLAVE_FORCE` rejected for ARA reservation.

Rationale: SMBus alert dispatch hands an interrupt directly to a driver-supplied `.alert` callback while holding the alerting device's `device_lock`, which makes refcount-overflow and unbind-vs-alert races a privileged code-execution primitive. SPD parsing additionally trusts DIMM-supplied bytes for DDR5 NV-write enable, so without strict CAP_SYS_RAWIO gating, RAP/kCFI on `.alert`, PEC enforcement, and SANITIZE on SPD buffers, a malicious DIMM or a forced address-claim turns into kernel-text redirect or persistent DIMM corruption.

## Out of Scope

- `i2c_smbus_xfer` / `_read_byte` / `_write_byte` / `_read_byte_data` / `_write_byte_data` / `_read_word_data` / `_write_word_data` / `_read_block_data` / `_write_block_data` / `_process_call` / `_write_quick` / `i2c_smbus_pec` / `i2c_smbus_msg_pec` — these live in `drivers/i2c/i2c-core-smbus.c` and are covered in `drivers/i2c/core-base.md`.
- I2C adapter / client / driver lifecycle — `drivers/i2c/core-base.md`.
- I2C mux core — `drivers/i2c/i2c-mux.md`.
- I2C slave-mode core (`i2c-core-slave.c`) — `drivers/i2c/core-base.md`.
- /dev/i2c-N chardev — `drivers/i2c/dev.md`.
- Per-controller SMBus bus drivers (i801, piix4, etc.) — `drivers/i2c/busses-x86.md`.
- DDR SPD EEPROM driver internals (`drivers/misc/eeprom/{ee1004,spd5118}.c`) — covered elsewhere.
- Implementation code.
