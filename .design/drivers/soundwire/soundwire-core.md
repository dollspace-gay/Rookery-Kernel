# Tier-3: drivers/soundwire/{bus,bus_type,mipi_disco}.c — MIPI SoundWire bus core (enumeration + stream mgmt + DisCo)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/soundwire/soundwire-core.md
upstream-paths:
  - drivers/soundwire/bus.c
  - drivers/soundwire/bus_type.c
  - drivers/soundwire/mipi_disco.c
  - drivers/soundwire/bus.h
  - drivers/soundwire/stream.c
  - drivers/soundwire/master.c
  - drivers/soundwire/slave.c
  - drivers/soundwire/irq.c
  - include/linux/soundwire/sdw.h
  - include/linux/soundwire/sdw_type.h
  - include/linux/soundwire/sdw_registers.h
-->

## Summary

SoundWire — a MIPI Alliance two-wire audio + control bus (clock + data) defined by the SoundWire spec; one or more Manager (Master) controllers each manage up to 11 Peripheral (Slave) devices over a serial link. The kernel framework provides bus enumeration (parking + attached + alert state machine per peripheral), device discovery via ACPI / DT with MIPI DisCo properties, control-channel messaging (ping / read / write of per-device registers across a payload-shaped 6-cycle frame), stream management (per-stream multi-port multi-master bandwidth allocator over PCM data ports DP1..DPn), and an interrupt domain for per-slave wakeups.

This Tier-3 covers `bus.c` (~2100 lines: master add/delete, dev_num assignment, read/write/update_status, clock-stop, IRQ delivery), `bus_type.c` (~250 lines: `sdw_bus_type`, modalias = `sdw:m<mfg>p<part>v<ver>c<class>`, probe/remove/shutdown), and `mipi_disco.c` (~530 lines: ACPI/DT DisCo property parser for master + slave + per-DP capabilities).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct sdw_bus` | per-link Manager state: ops, slaves list, m_rt_list (stream runtimes), params, prop, IRQ domain | `drivers::soundwire::Bus` |
| `struct sdw_master_device` / `sdw_master_ops` | Manager device + vtable: read_prop, xfer_msg, set_bus_conf, send_intr, clock_stop, pre_bank_switch, post_bank_switch | `drivers::soundwire::Master` |
| `struct sdw_slave` / `sdw_slave_ops` | Peripheral state + vtable: read_prop, interrupt_callback, update_status, port_prep, bus_config, port_intr | `drivers::soundwire::Slave` |
| `struct sdw_master_prop` / `sdw_slave_prop` / `sdw_dpn_prop` | DisCo-derived properties | `drivers::soundwire::Props` |
| `sdw_bus_master_add(bus, parent, fwnode)` / `_master_delete(bus)` | per-link bring-up / teardown | `Bus::master_add` / `master_delete` |
| `sdw_get_id(bus)` / `sdw_bus_ida` | per-link unique id | `Bus::get_id` |
| `sdw_handle_slave_status(bus, status[])` | per-link status walk: assign dev_num to ATTACHED, mark ABSENT for missing | `Bus::handle_slave_status` |
| `sdw_assign_device_number(slave)` | bitmap-based 1..11 dev_num alloc, programs DEV_NUMBER register | `Bus::assign_dev_num` |
| `sdw_read(slave, addr)` / `_write(slave, addr, val)` / `_nread(...)` / `_nwrite(...)` | per-slave register I/O via Manager xfer_msg | `Slave::read/write` |
| `sdw_update_slave_status(slave, status)` | drive per-slave state machine NOT_PRESENT → ATTACHED_OK → ALERT | `Slave::update_status` |
| `sdw_clock_stop(bus)` / `sdw_clock_stop_exit(bus)` | enter / exit Clock-Stop mode0/mode1 | `Bus::clock_stop` |
| `sdw_compute_params(bus)` | per-stream bandwidth allocation across m_rt_list | `Stream::compute_params` |
| `sdw_alloc_stream(stream_name)` / `_release_stream(stream)` | per-stream runtime alloc | `Stream::alloc` / `release` |
| `sdw_stream_add_master(...)` / `_add_slave(...)` / `_remove_master(...)` / `_remove_slave(...)` | stream membership ops | `Stream::add_master` / etc. |
| `sdw_prepare_stream(stream)` / `_enable_stream(stream)` / `_disable_stream(stream)` / `_deprepare_stream(...)` | stream lifecycle | `Stream::{prepare,enable,disable,deprepare}` |
| `sdw_master_read_prop(bus)` / `sdw_slave_read_prop(slave)` | MIPI DisCo property parser | `Disco::read_*` |
| `sdw_bus_type` | bus_type with match by `(mfg_id, part_id, sdw_version, class_id)` | `drivers::soundwire::BusType` |
| `sdw_irq_create(bus, fwnode)` / `_delete(...)` / `_create_mapping(slave)` | IRQ-domain per-link, per-slave mapping | `Bus::irq_*` |

## Compatibility contract

REQ-1: SoundWire device numbers 0–15: 0 ENUM, 14 MASTER, 15 BROADCAST, 12+13 GROUP12/GROUP13 reserved; per-slave dev_num assigned in [1..11].

REQ-2: Per-bus required ops: `read_prop` (DisCo parse), `xfer_msg` (control-channel I/O), `compute_params` (bandwidth allocator). Refuse `sdw_bus_master_add` if either is missing.

REQ-3: Per-slave matched via `(mfg_id, part_id, sdw_version, class_id)`; modalias `sdw:m%04Xp%04Xv%02Xc%02X`; class_id 0 wildcard matches any class.

REQ-4: Bus locking: per-bus `bus_lock` (slave list / dev_num bitmap mutations) + `msg_lock` (control-channel message serialization); each has unique lockdep key per bus to permit multi-bus stream compose.

REQ-5: Frame shape: 2..16 columns × 6..256 rows; per-frame contains control header + payload slots; allocator chooses (rows, cols) that fit declared bandwidth.

REQ-6: Clock-Stop mode0 (slave stays attached, can wake on bus event) + mode1 (slave fully powered down, requires re-enumeration); per-slave declares supported modes via `clk_stop_modes` bitmask.

REQ-7: Stream bandwidth allocator: per-`sdw_master_runtime` traverses attached `sdw_slave_runtime`s on each Manager; sums port bandwidth; selects frame-shape; calls `pre_bank_switch` → write next-bank registers → `post_bank_switch` → switch banks atomically.

REQ-8: DisCo properties parsed: `mipi-sdw-sw-interface-revision`, `mipi-sdw-master-count`, per-link `mipi-sdw-link-N-subproperties` (max-clock-frequency, supported-frequencies, clock-stop modes, default-frame-row-size, default-frame-col-size, dynamic-frame-shape-supported); per-slave properties; per-data-port DPn properties.

REQ-9: ACPI / DT slave discovery: `sdw_acpi_find_slaves(bus)` or `sdw_of_find_slaves(bus)`; per-slave reg property carries `unique_id` + `mfg_id` + `part_id`.

REQ-10: IRQ domain via `sdw_irq_create` providing per-slave `linear` IRQ mapping when `slave->prop.use_domain_irq` is set; per-slave IRQ wired to slave-side interrupt-callback.

REQ-11: Bus shutdown sequence: deprepare + disable + remove all streams → put all slaves to clock-stop mode1 → `master_delete` → free `bus_ida`.

REQ-12: Per-slave probe order: `sdw_bus_probe` checks fwnode present, runs `dev_pm_domain_attach`, allocates per-bus slave_ida index, calls `drv->probe(slave, id)`, then `drv->ops->read_prop`, optional IRQ mapping, sysfs init, and `update_status` notification.

## Acceptance Criteria

- [ ] AC-1: On Intel reference HW (ICL/TGL/MTL) with audio codec on SoundWire, `lsmod | grep snd_soc_sof_sdw` loaded; `/sys/bus/soundwire/devices/` lists peripherals.
- [ ] AC-2: ALSA PCM playback through SoundWire codec produces audio (verified via `aplay -D hw:sdw0`).
- [ ] AC-3: dev_num exhaustion: 11 slaves attach successfully; 12th attempt receives `-ENODEV` from `sdw_assign_device_number`.
- [ ] AC-4: Clock-stop cycle: `pm_runtime_put` on idle bus enters mode0; new audio request exits mode0 within bounded latency.
- [ ] AC-5: Slave dynamic remove: physically detached slave is marked `ABSENT` by status walk; sysfs node removed.
- [ ] AC-6: Bandwidth allocator: enabling >1 stream with conflicting BW fails `sdw_compute_params` with `-EBUSY`, never overshoots `max_dr_freq`.
- [ ] AC-7: Multi-master stream (Intel ACE2.x): stream spanning two Manager links composes correctly under per-bus lockdep keys without deadlock.
- [ ] AC-8: DisCo property validation: malformed `mipi-sdw-master-count` or missing `mipi-sdw-link-N-subproperties` returns error from `sdw_master_read_prop`, refuses `sdw_bus_master_add`.

## Architecture

`Bus` lives in `drivers::soundwire::Bus`:

```
struct Bus {
  dev: Arc<KernelDevice>,
  md: Arc<MasterDevice>,
  link_id: u32,
  id: u32,                        // ida-assigned
  controller_id: i32,
  ops: &'static dyn MasterOps,
  port_ops: &'static dyn MasterPortOps,
  compute_params: fn(&Bus) -> Result<()>,
  slaves: Mutex<ListHead<Slave>>,
  m_rt_list: Mutex<ListHead<MasterRuntime>>,
  prop: MasterProp,
  assigned: BitBox<[u64; 1]>,       // 16-bit dev_num bitmap
  slave_ida: Ida,
  bus_lock_key: LockClassKey,
  bus_lock: Mutex<()>,
  msg_lock_key: LockClassKey,
  msg_lock: Mutex<()>,
  multi_link: bool,
  params: BusParams { max_dr_freq, curr_dr_freq, curr_bank, next_bank, ... },
  irq_chip: IrqChip,
  domain: Option<IrqDomain>,
  clk_stop_timeout: u32,
}
```

Bring-up `Bus::master_add(bus, parent, fwnode)`:
1. `sdw_get_id(bus)` from `sdw_bus_ida`.
2. `sdw_master_device_add(bus, parent, fwnode)`.
3. Validate `bus->ops` + `bus->compute_params` non-null.
4. Register unique lockdep keys + init `msg_lock` / `bus_lock`.
5. Init `slaves` + `m_rt_list`.
6. `bus->ops->read_prop(bus)` — DisCo parse via `sdw_master_read_prop`.
7. Initialize `bus->assigned` bitmap, masking ENUM/BROADCAST/GROUP/MASTER device numbers (0, 14, 15, 12, 13).
8. `sdw_irq_create(bus, fwnode)`.
9. ACPI / DT slave discovery: `sdw_acpi_find_slaves(bus)` or `sdw_of_find_slaves(bus)`.
10. Init `bus->params`: `max_dr_freq = prop.max_clk_freq * SDW_DOUBLE_RATE_FACTOR`; `curr_bank = SDW_BANK0`.

Slave enumeration `Bus::handle_slave_status(status[16])`:
1. For each dev_num 0..15:
   - `NOT_PRESENT` → mark slave as `SDW_SLAVE_UNATTACHED` if previously present.
   - `ATTACHED_OK` → if dev_num 0, run `sdw_assign_device_number(slave)`:
     a. Find free bit in `bus->assigned` after masking reserved.
     b. Program slave's `SCP_DEVNUMBER` register with chosen number.
     c. Mark slave `SDW_SLAVE_REPORT_ATTACHED`.
   - `ALERT` → invoke `slave_ops->interrupt_callback`.

Bus_type match (`bus_type.c`):
1. `is_sdw_slave(dev)` filter.
2. Walk `drv->id_table` matching `mfg_id == id->mfg_id && part_id == id->part_id && (id->sdw_version==0 || ==slave.sdw_version) && (id->class_id==0 || ==slave.class_id)`.

Probe path (`sdw_bus_probe`):
1. Require `dev->fwnode` + ACPI-or-OF presence.
2. `sdw_get_device_id(slave, drv)`; return `-ENODEV` if no match.
3. `dev_pm_domain_attach(dev, 0)`.
4. `ida_alloc_max(&bus->slave_ida, SDW_FW_MAX_DEVICES-1)` → `slave->index`.
5. `drv->probe(slave, id)`.
6. Lock `slave->sdw_dev_lock`; `drv->ops->read_prop(slave)`.
7. If `slave->prop.use_domain_irq`: `sdw_irq_create_mapping(slave)`.
8. `sdw_slave_sysfs_dpn_init(slave)`.
9. `slave->bus->clk_stop_timeout = max(..., slave->prop.clk_stop_timeout || 300ms)`.
10. `slave->probed = true`; if bus already started, propagate current status via `update_status`.

Stream mgmt (`stream.c`):
- `sdw_alloc_stream(name)` → `sdw_stream_runtime` with `m_rt_list` + `state = SDW_STREAM_ALLOCATED`.
- `sdw_stream_add_master/_add_slave` add runtime entries linking ports + channels.
- `prepare → enable → disable → deprepare` state machine with per-state `pre_bank_switch` + `post_bank_switch` hooks for atomic frame-shape change.

DisCo parser (`mipi_disco.c`):
- `mipi_fwnode_property_read_bool` reads u8 via fwnode then casts to bool — defends against absent-prop ambiguity.
- `sdw_master_read_prop`: reads `mipi-sdw-sw-interface-revision`, locates `mipi-sdw-link-<id>-subproperties` child node, reads clock-stop modes + max-clock-frequency + supported-frequencies + dynamic-frame-shape support.

## Hardening

(Inherits from `drivers/soundwire/00-overview.md` § Hardening.)

soundwire-core specific reinforcement:

- **Per-bus dev_num bitmap masks reserved numbers** — refuses to assign 0/12/13/14/15 to peripherals.
- **dev_num count strictly 1..11** — defense against attacker-controlled slave claiming reserved numbers.
- **DisCo property bounds checked** — refuses `max-clock-frequency = 0`, missing required arrays, malformed link-N-subproperties child.
- **Per-bus unique lockdep keys** — multi-bus stream compose deadlock-free; `lockdep_register_key` per `msg_lock`/`bus_lock`.
- **Per-slave probe requires fwnode + (ACPI or DT)** — refuse probe without firmware description.
- **Clock-stop timeout floor 300ms** — refuses to accept shorter than spec recommendation.
- **IRQ domain per-bus** — slave wakeup interrupts isolated; refuses ungated cross-bus dispatch.
- **`sdw_compute_params` returns -EBUSY on overcommit** — never overshoots max double-rate frequency.
- **slave_ida bounded by `SDW_FW_MAX_DEVICES-1`** — refuse new slave index alloc beyond limit.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist `sdw_bus`, `sdw_slave`, `sdw_master_device`, `sdw_stream_runtime`, `sdw_master_runtime`, `sdw_slave_runtime` slabs; userland sysfs buffers strictly bounded.
- **PAX_KERNEXEC** — `sdw_master_ops`, `sdw_slave_ops`, `sdw_bus_type` callbacks (`match`, `probe`, `remove`, `shutdown`, `uevent`), and DisCo parsers live in `__ro_after_init` text.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `sdw_bus_master_add`, `sdw_handle_slave_status`, `sdw_assign_device_number`, stream-prepare/enable, and IRQ dispatch entries.
- **PAX_REFCOUNT** — saturating `refcount_t` on `sdw_slave`, `sdw_stream_runtime`, `sdw_master_runtime`, IRQ-domain mappings; overflow trap defeats hotplug + stream-compose races.
- **PAX_MEMORY_SANITIZE** — zero-on-free for stream runtimes, master/slave runtimes, DisCo prop tables, IRQ-mapping caches; defense against bw-leak across hot-detach + reattach.
- **PAX_UDEREF** — SMAP/PAN enforced on every sysfs / debugfs entry into SoundWire core; reject user-pointer deref outside canonical helpers.
- **PAX_RAP / kCFI** — `sdw_master_ops` (`read_prop`, `xfer_msg`, `set_bus_conf`, `send_intr`, `clock_stop`, `pre_bank_switch`, `post_bank_switch`, `get_device_num`, `put_device_num`), `sdw_slave_ops`, and `sdw_bus_type` callbacks marked `__ro_after_init` with kCFI typed dispatch.
- **GRKERNSEC_HIDESYM** — suppress `%p` of `sdw_bus`, `sdw_slave`, IRQ-domain pointers in tracepoints; gate kallsyms behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict DisCo-parse-fail, dev_num-exhaustion, stream-bw-fail banners to CAP_SYSLOG; suppress mfg/part-id disclosure.
- **CAP_SYS_ADMIN on bus IRQ pcap / debugfs** — debugfs `soundwire/` and any IRQ packet-capture surfaces require CAP_SYS_ADMIN in the owning user namespace.
- **MIPI DisCo prop validation** — every parsed property strictly bounds-checked (range, count, presence); refuse `sdw_bus_master_add` if mandatory props absent or out-of-range.
- **Device-id PAX_REFCOUNT** — `sdw_device_id` and per-slave dev_num lifetimes refcount-protected; refuse re-issue of dev_num while in-flight.
- **Master-state-machine kCFI** — `sdw_master_runtime` state transitions (ALLOCATED → CONFIGURED → PREPARED → ENABLED → DISABLED → DEPREPARED → RELEASED) enforced via typed indirect dispatch; reject out-of-order callbacks.
- **Slave ALERT rate-limit** — per-slave ALERT IRQ rate-limited; defense against malicious slave generating IRQ-storm to soft-lockup host.
- **DPn-port allowlist** — per-data-port direction + sample-format validated against DisCo-declared capabilities; refuse stream-add that would activate undeclared ports.

Rationale: SoundWire peripherals often include security-sensitive audio paths (always-on voice triggers, secure boot signing keys, ambient-light sensors) and the bus is wholly under DisCo/firmware-table control. A loose dev_num allocator, missing DisCo bounds checks, refcount underflow on `sdw_slave`, or unrestricted debugfs lets a malicious DSDT, a hot-attached rogue peripheral, or an unprivileged userland process pivot from bus control into voice-data exfiltration, IRQ-storm DoS, or kernel memory corruption. RAP/kCFI on the per-master/slave vtables, CAP_SYS_ADMIN on debugfs, mandatory DisCo bounds, dev_num bitmap masking, and refcount-overflow trapping turn SoundWire from "trust the firmware" into a structural enforcement boundary.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Intel SoundWire Manager (`intel.c`, `intel_ace2x.c`, `cadence_master.c` — separate Tier-3s)
- Qualcomm SoundWire (`qcom.c` future Tier-3)
- AMD SoundWire (`amd_manager.c` future Tier-3)
- Stream bandwidth-allocator algorithm internals (covered in `soundwire-stream.md` future Tier-3)
- Per-slave sysfs DPn details (covered in `soundwire-sysfs.md` future Tier-3)
- Implementation code
