# Tier-3: drivers/i3c/master.c + device.c — I3C master controller framework

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/i3c/00-overview.md
upstream-paths:
  - drivers/i3c/master.c
  - drivers/i3c/device.c
  - drivers/i3c/internals.h
  - include/linux/i3c/master.h
  - include/linux/i3c/device.h
  - include/linux/i3c/ccc.h
-->

## Summary

I3C ("eye-three-see", MIPI I3C basic spec + I3C v1.1.1) is the successor to I2C: same two-wire SDA/SCL topology, backward-compatible with I2C devices on the same bus, but adds (a) **Dynamic Address Assignment** (DAA) — devices arrive at startup with a 48-bit Provisional ID + BCR/DCR registers and the master assigns 7-bit dynamic addresses via ENTDAA broadcast CCC, (b) **In-Band Interrupts** (IBI) — devices can request attention without a side-band IRQ line, (c) **HDR modes** (HDR-DDR, HDR-TSP, HDR-TSL) for higher throughput, (d) **Hot-Join** for live-attach devices, (e) **Common Command Codes** (CCC) — standardized broadcast / directed control messages.

The Linux subsystem is *single-master-per-bus* today (no multi-master arbitration). `drivers/i3c/master.c` is the framework spine: it manages the I3C `bus_type`, per-bus address-slot bitmap, DAA orchestration via per-controller `do_daa` op, IBI dispatch, hot-join enable, and CCC send. Per-controller drivers (Cadence, Synopsys DesignWare, Aspeed AST2600, Silvaco SVC, MIPI HCI, Renesas, ADI) implement `i3c_master_controller_ops` and call `i3c_master_register` to bring the bus up.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct i3c_bus` | per-bus: dynamic+static address slot bitmap, mode, scl_rate, lock | `drivers::i3c::I3cBus` |
| `struct i3c_master_controller` | per-controller: bus + ops + i2c-adapter shim + IBI workqueue + hot-join workqueue | `drivers::i3c::I3cMasterController` |
| `struct i3c_master_controller_ops` | bus_init / bus_cleanup / do_daa / send_ccc_cmd / supports_ccc_cmd / priv_xfers / attach_i3c_dev / reattach_i3c_dev / detach_i3c_dev / attach_i2c_dev / detach_i2c_dev / request_ibi / free_ibi / enable_ibi / disable_ibi / enable_hotjoin / disable_hotjoin / set_speed / set_dev_nack_retry | `drivers::i3c::ControllerOps` |
| `struct i3c_dev_desc` | per-I3C-device descriptor (PID, BCR, DCR, dynamic_addr, info, master backpointer, ibi_slot pool) | `drivers::i3c::I3cDevDesc` |
| `struct i3c_device` | per-I3C-device "client" exposed to driver-core; wraps `i3c_dev_desc` | `drivers::i3c::I3cDevice` |
| `i3c_master_register(m, parent, ops, secondary)` | per-controller register: alloc bus_idr id, init slot bitmap, call ops->bus_init, run initial DAA, register children | `I3cMasterController::register` |
| `i3c_master_unregister(m)` | per-controller unregister: ops->bus_cleanup, detach children, free bus | `I3cMasterController::unregister` |
| `i3c_master_do_daa(m)` / `_do_daa_ext(m, rstdaa)` | per-bus DAA: take maintenance lock, scan, assign dynamic addrs, register new i3c_dev | `I3cMasterController::do_daa` |
| `i3c_master_set_info(m, info)` | controller declares its own bus-master device info | `I3cMasterController::set_info` |
| `i3c_master_get_free_addr(m, start)` | per-bus free-address allocator from slot bitmap | `I3cBus::get_free_addr` |
| `i3c_master_entdaa_locked(m)` / `_disec_locked(m, addr, evts)` / `_enec_locked(m, addr, evts)` / `_defslvs_locked(m)` | per-controller CCC builders | `I3cMasterController::ccc_*` |
| `i3c_master_queue_ibi(dev, slot)` / `i3c_master_handle_ibi(work)` | IBI receive path: per-device pool slot + workqueue dispatch to client handler | `I3cMasterController::{queue,handle}_ibi` |
| `i3c_master_enable_hotjoin(m)` / `_disable_hotjoin(m)` | per-bus hot-join enable/disable | `I3cMasterController::hotjoin_*` |
| `i3c_device_do_xfers(dev, xfers, n)` | per-device SDR/HDR private transfer | `I3cDevice::do_xfers` |
| `i3c_device_request_ibi(dev, &req)` / `_enable_ibi(dev)` / `_disable_ibi(dev)` / `_free_ibi(dev)` | per-device IBI request/enable/disable | `I3cDevice::ibi_*` |
| `i3c_bus_maintenance_lock(bus)` / `_unlock(bus)` | per-bus exclusive lock for DAA / re-address / mastership / topology change | `I3cBus::maintenance_lock` |
| `i3c_bus_normaluse_lock(bus)` / `_unlock(bus)` | per-bus shared lock for normal transfers | `I3cBus::normaluse_lock` |

## Compatibility contract

REQ-1: Per-bus address space = 128 dynamic + reserved I2C statics; `i3c_bus.addrslots` bitmap encodes per-address `enum i3c_addr_slot_status` (FREE / RSVD / I2C_DEV / I3C_DEV). DAA never assigns an addr already RSVD/I2C/I3C.

REQ-2: DAA via `ENTDAA` CCC: master broadcasts ENTDAA, each unassigned device clocks out its 48-bit Provisional ID + BCR + DCR using open-drain SCL arbitration; lowest-PID device wins each round; master assigns dynamic addr via SETDASA-equivalent; repeat until no more devices respond.

REQ-3: Per-device `i3c_dev_desc` holds PID (compared against `i3c_device_id.{manuf_id,part_id,extra_info}` for driver matching), BCR (Bus Characteristics Register: device role, max data rate, IBI capability, offline capability), DCR (Device Characteristics Register: device class identifier).

REQ-4: CCC commands: `i3c_master_send_ccc_cmd(m, cmd)` dispatches through `ops->send_ccc_cmd` after `ops->supports_ccc_cmd` check; spec-defined CCC IDs include ENEC/DISEC (enable/disable events), RSTDAA (reset DAA), ENTDAA, DEFSLVS (define slave list, used in mastership handoff), GETMRL/GETMWL (get max read/write length), GETHDRCAP (HDR mode caps).

REQ-5: IBI flow: device drives SDA low during start condition; master arbitrates; ack/nack; if ack, master reads MDB (Mandatory Data Byte) + optional payload; per-device `ibi_handler` invoked on workqueue with the slot. Per-device `ibi_slot` pool sized at request time.

REQ-6: HDR modes: `GETHDRCAP` enumerates per-device HDR capability bitmap; per-controller `priv_xfers` op accepts `i3c_priv_xfer.mode` (SDR / HDR_DDR / HDR_TSP / HDR_TSL); refuse modes unsupported by controller or device.

REQ-7: Hot-join: device drives SDA after bus idle; `i3c_master_enable_hotjoin` arms the controller; per-controller IRQ -> framework `i3c_master_queue_hotjoin` -> hot-join workqueue runs DAA to enumerate the newcomer.

REQ-8: Maintenance vs normal-use lock: DAA, re-address, mastership handoff, topology change MUST hold `bus->lock` write (maintenance); transfers + CCC during normal operation hold read (normaluse). Lock ordering: maintenance > normaluse.

REQ-9: I2C backward compat: I3C bus exposes a `struct i2c_adapter` so legacy I2C devices on the same bus enumerate via i2c-core; per-I2C-device attach goes through `ops->attach_i2c_dev`.

REQ-10: Per-bus speed: `bus->scl_rate.i3c` (typ 12.5 MHz push-pull) + `.i2c` (open-drain mode for legacy I2C devs); `ops->set_speed` programs the controller.

## Acceptance Criteria

- [ ] AC-1: I3C controller (e.g. Cadence on FPGA dev board) probes; `i3c_master_register` succeeds; `/sys/bus/i3c/devices/` enumerates the master.
- [ ] AC-2: DAA assigns dynamic addresses to all responding devices in PID order; per-device sysfs (pid, bcr, dcr, dynamic_address, hdrcap) populated.
- [ ] AC-3: Per-device driver bound by PID match in `i3c_device_id` table; probe receives populated `i3c_device_info`.
- [ ] AC-4: SDR private transfer through `i3c_device_do_xfers` round-trips correctly against a known I3C slave (e.g. MIPI I3C sensor).
- [ ] AC-5: IBI from an enabled device delivers MDB + payload to the per-device handler.
- [ ] AC-6: Hot-join: physically attach a new I3C device with hot-join enabled; DAA reruns; new device enumerates without bus reset.
- [ ] AC-7: HDR-DDR mode transfer on a controller that supports it (Cadence does); throughput observably ~2x SDR.
- [ ] AC-8: I2C backward compat: an I2C-only device on the same bus accessible via `i2c_master_send` through the I2C adapter shim.

## Architecture

`I3cMasterController::register(m, parent, ops, secondary)`:

1. Validate `ops` non-NULL + mandatory hooks present (`bus_init`, `priv_xfers`, `attach_i3c_dev`, `attach_i2c_dev`).
2. `i3c_bus_init(&m->bus, np)`: alloc `bus_idr` id, init slot bitmap (mark spec-reserved addresses RSVD), parse DT `assigned-address`/`reg` for static I2C devices.
3. `m->ops = ops`; `m->this_dev` populated from `i3c_master_set_info` (controller advertises its own device info on the bus).
4. `device_register(&m->dev)` under `i3c_bus_type`; `/sys/bus/i3c/devices/iN-bus/` appears.
5. Take maintenance lock: `ops->bus_init(m)` -> controller-specific init (clock, FIFOs, IRQ); `i3c_master_do_daa(m)` to enumerate.
6. For each I2C device declared in DT: `ops->attach_i2c_dev(m, i2c_dev)` -> register as i2c-core client via `i2c_new_client_device`.
7. Release maintenance lock.

DAA `i3c_master_do_daa_ext(m, rstdaa)`:

1. `i3c_bus_maintenance_lock(&m->bus)`.
2. If `rstdaa`: send RSTDAA CCC (broadcast, all devices forget assigned addrs).
3. `ops->do_daa(m)` -> controller runs ENTDAA loop; controller calls back into framework via `i3c_master_add_i3c_dev_locked(m, pid, bcr, dcr, dyn_addr)` for each enumerated device.
4. Per-device: allocate `i3c_dev_desc`, populate PID/BCR/DCR/dyn_addr, mark slot I3C_DEV in bitmap, link into `m->boardinfo->devs` or alloc fresh, set `info`.
5. After ENTDAA loop, `i3c_master_register_new_i3c_devs(m)` walks the freshly-registered list and creates per-device `i3c_device` + `device_register(&i3c_dev->dev)`; driver-core matches against registered `i3c_driver.id_table`.
6. Release maintenance lock.

CCC dispatch (`i3c_master_send_ccc_cmd`):

```
i3c_master_send_ccc_cmd(m, cmd):
  if !ops->supports_ccc_cmd(m, cmd): return -ENOTSUPP
  return ops->send_ccc_cmd(m, cmd)
```

Per-CCC helper functions (`i3c_master_entdaa_locked`, `_disec_locked`, `_enec_locked`, `_defslvs_locked`, `_getmrl_locked`, etc.) build the `i3c_ccc_cmd` then call `i3c_master_send_ccc_cmd`. Required-locked variants demand bus maintenance lock held.

IBI receive (`i3c_master_queue_ibi` from controller IRQ context):

1. Controller IRQ identifies which device IBI'd (matching to `i3c_dev_desc` via dynamic_addr).
2. `i3c_master_queue_ibi(dev, slot)`: increment `slot->ref`, `queue_work(m->wq, &slot->work)`.
3. `i3c_master_handle_ibi(work)` (workqueue context): invoke `dev->ibi->handler(dev, slot)`, then `i3c_master_get_free_ibi_slot(dev)` to return slot to pool.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `daa_no_addr_collision` | UNIQUENESS | `i3c_master_get_free_addr` never returns an addr marked RSVD/I2C_DEV/I3C_DEV |
| `ibi_slot_no_uaf` | UAF | per-device `ibi_slot` returned to pool exactly once between `queue_ibi` and handler completion |
| `bus_lock_order` | LOCK-ORDER | maintenance lock always acquired before any normal-use op; normal-use op never upgrades to maintenance |
| `ccc_cmd_supported` | INVARIANT | `send_ccc_cmd` invoked only after `supports_ccc_cmd` returns true |

### Layer 2: TLA+

`models/i3c/daa.tla` (parent-declared): proves ENTDAA loop terminates under N devices and arbitration ties (lowest PID wins each round); proves DAA preserves slot-bitmap invariant; proves rescind under hot-join during DAA does not cause double-assignment.

### Layer 3: Verus invariants

| Invariant | Component |
|---|---|
| Post `i3c_master_register`: bus initialized AND children registered with valid dynamic addrs AND maintenance lock released | `I3cMasterController::register` |
| Post `i3c_master_do_daa`: every enumerated device has unique dynamic_addr AND slot bitmap reflects assignment | `I3cMasterController::do_daa` |
| Post `i3c_device_request_ibi`: per-device `ibi_slot` pool allocated of size `req.max_payload_len`; `ibi_handler` set | `I3cDevice::request_ibi` |
| Post `i3c_master_handle_ibi`: slot returned to pool AND ref decremented | `I3cMasterController::handle_ibi` |

### Layer 4: Verus functional

End-to-end: controller probe -> `i3c_master_register` -> DAA -> per-device driver probe -> `i3c_device_request_ibi` + `_enable_ibi` -> device IBIs -> handler invoked -> `_disable_ibi` -> `_free_ibi` -> controller unregister with bus quiescent.

## Hardening

DAA timing is bus-physics-sensitive: during ENTDAA each unassigned device drives its PID on open-drain SDA; weak pull-ups + arbitration determine the winner. A malicious or buggy device can stall DAA by holding SDA low; per-controller `do_daa` MUST timebox the ENTDAA loop (typically 1 second wall-clock per device + per-bus overall) and abort with `-ETIMEDOUT` to defend against bus-stall DoS. Capture as Verus precondition on `ops->do_daa`.

IBI MDB (Mandatory Data Byte) is the first byte of an IBI payload — it indicates IBI class (hot-join, mastership-req, slave-interrupt) and per-class semantics. Framework should validate MDB against the per-device declared IBI class (set at `i3c_device_request_ibi`) and reject MDBs claiming a class the device never registered; defense vs. device-spoofing IBI escalation.

HDR modes are bus-rate-pushing; certain controllers + cables can fail HDR transfers silently corrupting data. Per-controller HDR support is advertised via `GETHDRCAP`; the framework should enforce that `i3c_priv_xfer.mode == HDR_*` only when both device AND controller declare support, AND that HDR-mode transfers go through a per-vendor allowlist to defeat firmware-pretend-supports-HDR-but-actually-broken devices.

Per-device `i3c_device_info` (PID, BCR, DCR) is propagated to drivers via `i3c_device_get_info` which returns a *copy*; the underlying `i3c_dev_desc.info` must not be modified post-DAA except under maintenance lock — drivers reading via the copy are safe but raw `i3c_dev_desc` dereference paths require the lock.

`i3c_master_register` ultimately calls `device_register` under maintenance lock held; driver-core probe runs without the lock; per-driver probe MUST take normaluse lock for any bus operation. Encode lock-mode in Verus.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist `i3c_master_controller`, `i3c_bus`, `i3c_dev_desc`, `i3c_device`, `i3c_ibi_slot`, and `i3c_ccc_cmd` slabs; bound copy_to_user on per-device sysfs (`pid`, `bcr`, `dcr`, `dynamic_address`, `hdrcap`).
- **PAX_KERNEXEC** — `i3c_master_controller_ops` vtable (`bus_init`, `bus_cleanup`, `do_daa`, `send_ccc_cmd`, `supports_ccc_cmd`, `priv_xfers`, `attach_i3c_dev`, `reattach_i3c_dev`, `detach_i3c_dev`, `attach_i2c_dev`, `detach_i2c_dev`, `i2c_xfers`, `request_ibi`, `free_ibi`, `enable_ibi`, `disable_ibi`, `enable_hotjoin`, `disable_hotjoin`, `set_speed`, `set_dev_nack_retry`) in `__ro_after_init`; `i3c_bus_type` ops RO.
- **PAX_RANDKSTACK** — randomize stack across `i3c_master_register`, `i3c_master_do_daa`, `i3c_device_do_xfers`, `i3c_master_handle_ibi`, hot-join workqueue worker.
- **PAX_REFCOUNT** — saturating refcount on `i3c_dev_desc` (must outlive in-flight IBI slots + in-flight transfers), `i3c_ibi_slot.ref`, and per-`i3c_master_controller` reference; overflow trap defeats hot-join/rescind race UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free on `i3c_dev_desc`, `i3c_ibi_slot` payloads (which may carry sensor-data secrets), CCC command buffers, and address-slot bitmap on bus teardown.
- **PAX_UDEREF** — SMAP/PAN on i3cdev userspace passthrough (when `CONFIG_I3CDEV` is enabled and `/dev/i3c-X/` is opened) and on sysfs writers.
- **PAX_RAP / kCFI** — `i3c_master_controller_ops`, `i3c_bus_type` ops, per-`i3c_driver.{probe,remove,suspend,resume}`, IBI handler dispatch (`dev->ibi->handler`), and CCC-cmd dispatch (`ops->send_ccc_cmd`) all kCFI-typed; vtables `__ro_after_init`.
- **GRKERNSEC_HIDESYM** — gate `/sys/bus/i3c/devices/*/pid`, `/bcr`, `/dcr`, `/dynamic_address` disclosure behind CAP_SYSLOG when bus carries security-relevant peripherals (TPMs, secure elements).
- **GRKERNSEC_DMESG** — restrict DAA enumeration banners + per-device PID disclosure + IBI payload dumps to CAP_SYSLOG.
- **DAA bus-arbitration timing bounded** — `ops->do_daa` MUST time-box ENTDAA per-device + per-bus overall; framework rejects controllers without timeout enforcement; defense against bus-stall DoS.
- **IBI MDB validation** — per-device `ibi->handler` invoked only when MDB matches the IBI class declared at `i3c_device_request_ibi`; reject spoofed-class MDBs; rate-limit per-device IBI to defeat IBI-flood DoS.
- **HDR modes vendor allowlist** — `i3c_priv_xfer.mode == HDR_DDR / HDR_TSP / HDR_TSL` permitted only when both controller `GETHDRCAP` AND device `GETHDRCAP` advertise support AND device PID is on the per-platform HDR-stable allowlist.
- **Dev-info PAX_USERCOPY** — `i3c_device_get_info` copy_to client driver bounded by `sizeof(i3c_device_info)`; refuse partial / oversize reads.
- **Master ops kCFI** — all entries in `i3c_master_controller_ops` typed against framework-declared signatures; refuse `i3c_master_register` if typesig drifts (defense vs. shim-controller subverting CCC dispatch).
- **CAP_SYS_ADMIN strict** — sysfs `i3c-X/hotjoin`, `do_daa`, `rstdaa`, and master-unbind paths require CAP_SYS_ADMIN in init user namespace.
- **Hot-join framework gate** — `i3c_master_enable_hotjoin` rejected unless device-tree declares `i3c-hotjoin = "enabled"` or admin explicitly opts in via sysfs; defense against unsolicited device attach on physically accessible buses.

Rationale: I3C buses appear in modern laptops + servers (battery-gauge, fan controllers, thermal sensors, TPM, ambient light) and increasingly in security-sensitive contexts (laptop lid sensor, secure element on the I3C side-channel). A malicious peripheral attached at hot-join, or a buggy device stalling DAA, is a realistic attack surface on a serviceable device. DAA time-boxing, IBI MDB validation, HDR vendor-allowlisting, kCFI on `i3c_master_controller_ops`, and hot-join gating turn an unconstrained bus enumeration into a policed boundary.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-controller drivers (Cadence, DesignWare, Aspeed AST2600, Silvaco SVC, MIPI HCI, Renesas, ADI) — covered in future per-driver Tier-3s
- Secondary master / mastership handoff — covered in future `i3c-secondary.md`
- i3cdev character-device userspace passthrough — covered separately when implemented
- I2C backward-compat shim internals — covered under `drivers/i2c/` Tier-3
- 32-bit-only paths
- Implementation code
