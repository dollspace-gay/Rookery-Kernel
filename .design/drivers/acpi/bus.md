# Tier-3: drivers/acpi/{bus,scan}.c — ACPI bus type, namespace scan, hotplug, _STA/_HID enumeration

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/acpi/00-overview.md
upstream-paths:
  - drivers/acpi/bus.c
  - drivers/acpi/scan.c
  - drivers/acpi/internal.h
  - drivers/acpi/glue.c
  - drivers/acpi/device_pm.c
  - drivers/acpi/device_sysfs.c
  - include/linux/acpi.h
  - include/acpi/acpi_bus.h
-->

## Summary

`bus.c` defines `acpi_bus_type` (the Linux bus type that owns every `acpi_device`), the `_STA` evaluator, the `_OSC` / `_DSM` evaluators (capability negotiation with firmware), the `acpi_subsys_pm_ops` glue, and `acpi_bus_init` (the subsys_initcall that brings ACPI up post-ACPICA). `scan.c` does the heavy lifting: walks the ACPI namespace, instantiates per-node `acpi_device`, dispatches per-HID/CID to `acpi_scan_handler` or `acpi_driver`, runs `_INI` per node, builds the per-device dependency graph (_DEP), and supports namespace-rescan after hotplug.

This Tier-3 fixes the bus + scan contract: how the namespace walk is structured, how `_STA` and `_HID`/`_CID`/`_CLS`/`_UID` are read and used to find a handler, how `_DEP` dependencies are honoured, how hotplug (`_Ex0`/`_Lxx`/`Notify`) inserts/removes devices, how `_OSC`/`_DSM` capability negotiation runs, and how the ACPI bus type plugs into the Linux driver-core.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct bus_type acpi_bus_type` | the ACPI bus | `drivers::acpi::Bus` |
| `acpi_bus_match(dev, drv)` / `acpi_bus_probe_disabled(dev, drv)` / `acpi_bus_uevent(dev, env)` | bus-ops callbacks | `Bus::{match,probe_disabled,uevent}` |
| `struct acpi_device` (cross-ref `00-overview.md`) | per-node device | `drivers::acpi::Device` |
| `struct acpi_driver { name, ids, ops { add, remove, notify } }` | per-HID/CID driver | `drivers::acpi::Driver` |
| `struct acpi_scan_handler { ids, attach, detach, hotplug }` | per-HID/CID built-in scan-handler | `drivers::acpi::ScanHandler` |
| `acpi_bus_init()` | subsys_initcall bring-up | `Subsystem::bus_init` |
| `acpi_bus_get_status(dev)` / `acpi_bus_get_status_handle(handle, &sta)` | `_STA` eval | `Device::get_status` |
| `acpi_bus_scan(handle)` / `acpi_bus_trim(dev)` / `acpi_bus_rescan(...)` | scan / trim / rescan | `Bus::scan` / `trim` / `rescan` |
| `acpi_add_single_object(&device, handle, type, sta)` | per-node device alloc | `Bus::add_object` |
| `acpi_init_device_object(device, handle, type, sta)` | initialize the device object | `Bus::init_object` |
| `acpi_scan_attach_handler(device, list_type, handler)` | dispatch to scan_handler | `Bus::attach_handler` |
| `acpi_bus_match` ID-table walk against `acpi_driver.ids[].id` | per-driver HID/CID match | `Bus::match` |
| `acpi_dev_get_resources(adev, &list, preproc, ctx)` | _CRS translation (cross-ref `resource.md`) | `Resource::get` |
| `acpi_dev_pm_attach(dev, power_on)` | per-device PM attach | `DevicePm::attach` |
| `acpi_bus_get_ejd(handle, &ejd)` | _EJD parent lookup | `Hotplug::get_ejd` |
| `acpi_bus_notify(handle, type, data)` | global notify-handler | `Notify::bus_handler` |
| `acpi_eval_osc(handle, guid, rev, cap, &out)` / `acpi_run_osc(handle, &context)` | _OSC capability negotiation | `Bus::osc` |
| `acpi_evaluate_dsm(handle, guid, rev, fn, in)` / `acpi_check_dsm(handle, guid, rev, fn_mask)` | _DSM method call + presence | `Aml::dsm` |
| `acpi_initialize_hp_context(adev, &hp, notify, uevent)` | hotplug context | `Hotplug::init_context` |
| `acpi_dev_for_each_child(adev, fn, ctx)` / `acpi_dev_for_each_consumer(...)` | per-child / per-consumer iter | `Device::for_each_*` |
| `acpi_walk_dep_device_list(handle)` | resolve _DEP | `Hotplug::resolve_dep` |
| `acpi_install_notify_handler(handle, type, fn, ctx)` | per-handle notify | `Notify::install` |

## Compatibility contract

REQ-1: `acpi_bus_type` is the kernel's `bus_type` for every ACPI namespace device; `bus->match(dev, drv)` runs the HID/CID/UID/CLS matching, `bus->uevent` exports `ACPI_HID=` / `ACPI_CID=` to userspace, `bus->pm` is `acpi_subsys_pm_ops`.

REQ-2: `_STA` is the single source of truth for device presence: bits PRESENT (0), ENABLED (1), UI (2), FUNCTIONING (3), BATTERY (4). A device with `!PRESENT && ENABLED` is logged as `FW_BUG` and the ENABLED bit cleared.

REQ-3: `_STA` absent is treated as "present + enabled + UI + functioning" per spec.

REQ-4: Battery-bearing devices require their `_DEP` predecessors to be initialized before `_STA` is evaluated (`device->dep_unmet`); battery `_STA` returns 0 while predecessors are unmet.

REQ-5: `acpi_bus_scan(handle)` descends the namespace via `acpi_walk_namespace(ACPI_TYPE_ANY, ...)` and instantiates `acpi_device` per node of type DEVICE/PROCESSOR/THERMAL/POWER. Per-node `_HID`/`_CID`/`_UID`/`_CLS` is read via `acpi_get_object_info`.

REQ-6: Per-HID/CID match runs against `acpi_scan_handlers_list` (built-in: `LNXSYSTM`, `LNXPWRBN`, `LNXSLPBN`, `LNXVIDEO`, `LNXTHERM`, `LNXCPU`, `LNXSYBUS`, `PNP0Cxx`, `PNP0A0x`, ...) first; if no scan-handler claims the node, the `acpi_bus_type` driver match runs.

REQ-7: `_INI` is invoked per-node during `acpi_initialize_objects` (ACPICA-managed phase); device drivers must not assume `_INI` has run before `acpi_driver.ops.add`.

REQ-8: `_OSC` per-handle negotiates per-capability bits with firmware (PCI features, USB, CPU PM, ...); per-GUID + per-rev call must match a known-supported set or firmware may reject. `_DSM` is the device-specific method form: per-GUID + per-function-index.

REQ-9: Hotplug: `Notify(device, 0x80)` insert / `Notify(device, 0x03)` eject; the notify handler queues work on `kacpi_hotplug_wq` which then runs `acpi_bus_scan(handle)` or `acpi_bus_trim(device)`. Per-bus-prefix hotplug gating (`/sys/firmware/acpi/hotplug/<prefix>/enabled`) lets userland refuse hotplug.

REQ-10: `_DEP` dependencies are honoured at scan-time: when `_DEP` references a node that has not yet been scanned, the dependent device is deferred to a second pass.

REQ-11: `_EJD` (parent for eject) tracked per-device; eject of a parent recursively trims every descendant.

REQ-12: `acpi_subsys_pm_ops` (the per-device PM operations) translates Linux PM transitions (`runtime_suspend` / `runtime_resume` / `prepare` / `complete` / `suspend` / `resume`) into ACPI power-state moves via `_PSx` calls (cross-ref `device-pm.md`).

## Acceptance Criteria

- [ ] AC-1: `ls /sys/bus/acpi/devices/` lists every namespace device with a `_HID`/`_CID`/`_UID` directory.
- [ ] AC-2: `cat /sys/bus/acpi/devices/<dev>/status` returns the `_STA` integer.
- [ ] AC-3: `cat /sys/bus/acpi/devices/<dev>/hid` returns the canonical `_HID`.
- [ ] AC-4: Lid + power button hotplug events (`Notify(0x80)`) generate input events and update sysfs.
- [ ] AC-5: PCI root bridge `_OSC` negotiation logs the per-bit capability mask accepted by firmware.
- [ ] AC-6: USB / NVMe `_DSM` invocations succeed when the device's `_DSM` is present.
- [ ] AC-7: PCI hotplug eject test: `echo 1 > /sys/bus/pci/slots/<slot>/eject` triggers `_EJ0`, the ACPI hotplug wq removes the device, and the kernel cleanly unbinds drivers.
- [ ] AC-8: Hotplug gating test: `echo 0 > /sys/firmware/acpi/hotplug/pci_root/enabled` refuses subsequent insert/eject.
- [ ] AC-9: Dep-met test: a battery node with `_DEP` referencing an EC node initialises only after the EC node is up.
- [ ] AC-10: kselftest `tools/testing/selftests/acpi/` (where present) clean.

## Architecture

`Bus` lives in `drivers::acpi::Bus`:

```
struct Bus {
  bus_type: &'static BusType,        // acpi_bus_type
  scan_handlers: RwLock<Vec<&'static ScanHandler>>,
  drivers: RwLock<Vec<Arc<Driver>>>,
  scan_lock: Mutex<()>,              // serialise scan + trim + hotplug
  hotplug_wq: Arc<Workqueue>,
  notify_handlers: HashMap<AcpiHandle, Arc<NotifyHandler>>,
  bus_id_list: Mutex<Vec<AcpiDeviceBusId>>,
  dep_list: Mutex<Vec<DepEntry>>,
  scan_system_dev_list: Mutex<Vec<Arc<Device>>>,
}

struct ScanHandler {
  ids: &'static [AcpiDeviceId],      // {hid="PNP0Cxx", driver_data=...}
  attach: fn(&Device, &AcpiDeviceId) -> i32,
  detach: fn(&Device),
  bind: Option<fn(&Device)>,
  unbind: Option<fn(&Device)>,
  hotplug: AcpiHotplugProfile,
}

struct Driver {
  name: &'static str,
  ids: &'static [AcpiDeviceId],
  ops: DriverOps,
  refcount: Refcount,
}

struct DriverOps {
  add: fn(&Device) -> i32,
  remove: fn(&Device),
  notify: Option<fn(&Device, u32)>,
}
```

Boot ordering:
1. ACPICA init phase (cross-ref `00-overview.md`) completes.
2. `subsys_initcall(acpi_bus_init)`:
   - Register `acpi_bus_type` with the driver-core.
   - Register `acpi_subsys_pm_ops` as the per-bus PM ops.
   - Install bus-wide notify handler via `acpi_install_notify_handler(ACPI_ROOT_OBJECT, ACPI_SYSTEM_NOTIFY|ACPI_DEVICE_NOTIFY, acpi_bus_notify, NULL)`.
   - Create the kacpi_hotplug workqueue.
3. `acpi_scan_init()`:
   - Register built-in `acpi_scan_handler`s (PNP root bridges, container, power button, sleep button, video, thermal, processor, ...).
   - Call `acpi_bus_scan(ACPI_ROOT_OBJECT)` to walk the namespace.
   - Per-device `acpi_add_single_object` allocates Arc<Device>, populates `pnp.hid/cid/uid/bus_id`, evaluates `_STA`, runs `_DEP` resolution.
   - For each device: dispatch to scan_handler (built-in) or expose to driver registry.
4. Per-driver `module_init` registers `acpi_driver` instances. `acpi_bus_register_driver` triggers a probe pass against existing devices via `bus_for_each_dev`.

Namespace scan per node:
1. `acpi_walk_namespace` descend callback: receive `handle, level, ctx, &retval`.
2. `acpi_get_object_info(handle, &info)` -> read `Type`, `Name`, `HID`, `CID`, `UID`, `CLS`.
3. Skip if type is not DEVICE/PROCESSOR/THERMAL/POWER.
4. `acpi_bus_get_status_handle(handle, &sta)` (per REQ-4 dep_unmet check).
5. `acpi_add_single_object(&device, handle, type, sta)` allocates the device and inserts into the parent.
6. Resolve `_DEP` via `acpi_walk_dep_device_list(handle)`: if a dep is unmet, mark `device->dep_unmet++` and remember the dep.
7. Try each scan_handler.ids against pnp.hid/cid; first match wins, `handler.attach(device, &id)` called.
8. If no scan_handler matches: device remains discoverable to `acpi_driver` via the bus type.
9. `acpi_dev_pm_attach(device, true)` to power-on per `_PRx`.

Hotplug path:
1. AML notify `Notify(handle, 0x80)` -> ACPICA invokes the installed notify handler.
2. `acpi_bus_notify` queues work on `kacpi_hotplug_wq` with the handle + event.
3. Worker: take `scan_lock`, evaluate `_STA`, and either:
   - `_STA & PRESENT` -> `acpi_bus_scan(handle)` (insert).
   - `_STA & !PRESENT` -> `acpi_bus_trim(device)` (remove): recursively unbind drivers + drop Arc<Device>.
4. Release `scan_lock`.

`_OSC` negotiation (PCI root bridge example):
1. Caller builds the capability DWORDs: `OSC_CAPABILITIES_MASK_ERROR_RECOVERY`, `OSC_QUERY`, etc.
2. `acpi_eval_osc(handle, &PCI_HOSTBRIDGE_OSC_UUID, rev=1, &cap, &out)`.
3. ACPICA invokes the `_OSC` AML method.
4. Per-out-DWORD status bit: `OSC_REQUEST_ERROR`, `OSC_INVALID_UUID_ERROR`, `OSC_INVALID_REVISION_ERROR`, `OSC_CAPABILITIES_MASK_ERROR`.
5. Per-feature bits in the response indicate which features firmware grants to OSPM.

`_DSM` per-method call (NVMe shared-MSI quirk example):
1. `acpi_evaluate_dsm(adev->handle, &nvme_dsm_guid, rev=1, fn=index, in)` -> ACPICA invokes the `_DSM` method.
2. Returned package translated to `acpi_object` tree; caller extracts per-element values.
3. Caller must `kfree(output)` after consume.

Per-bus-prefix hotplug profile:
- `acpi_hotplug_profile_register(profile)` registers a per-prefix profile.
- Profile fields: `name`, `enabled` (sysfs gate), `scan_dependent_skip`.
- Sysfs at `/sys/firmware/acpi/hotplug/<name>/enabled` toggled by userland to refuse hotplug.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `namespace_walk_no_oob` | OOB | per-level recursion bounded by ACPICA's max nesting depth; per-callback iter bounded. |
| `device_alloc_no_uaf` | UAF | `Arc<Device>` owned by Bus + parent's children list; trim drops both before freeing the slab. |
| `notify_handler_rcu` | UAF | per-handle notify handler list is RCU-safe under fire-concurrent-with-detach. |
| `sta_bits_validated` | OOB | `_STA` bits masked to defined positions; reserved bits ignored. |
| `dep_graph_acyclic` | LIVENESS | `_DEP` resolution refuses cycles; cyclic dependency logged as `FW_BUG` and broken. |

### Layer 2: TLA+

`models/acpi/hotplug.tla` (this doc): models the kacpi_hotplug_wq + scan_lock + per-device refcount and proves that concurrent insert/eject of overlapping subtrees serializes correctly without UAF.

`models/acpi/scan_dep_order.tla` (this doc): models the `_DEP` resolution + two-pass scan and proves that every dependent's `_INI` runs after every predecessor's `_INI`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `acpi_bus_scan` post: every namespace node with `_STA & PRESENT` has a corresponding `acpi_device` registered | `Bus::scan` |
| `acpi_bus_trim` post: no descendant `acpi_device` remains; every driver unbound | `Bus::trim` |
| `_OSC` post: firmware-granted capability mask is a subset of the requested mask | `Bus::osc` |
| `acpi_bus_get_status` post: `device->status.enabled => device->status.present` | `Device::get_status` |
| `hotplug_profile.enabled = false` post: subsequent `Notify(0x80)` is refused with audit log | `Hotplug::gate` |

### Layer 4: Verus/Creusot functional

Firmware Notify(0x80) -> SCI handler -> kacpi_hotplug_wq -> scan_lock -> `_STA` -> `acpi_bus_scan` -> new `acpi_device` -> `acpi_driver.ops.add` -> uevent ADD. Encoded as a refinement from the AML notify event into the per-userspace uevent.

## Hardening

(Inherits row-1 features from `drivers/acpi/00-overview.md` § Hardening.)

bus/scan specific reinforcement:

- **scan_lock serialization** — defense against parallel scan + hotplug + rescan racing on the same handle subtree.
- **dep_unmet counter validated** — defense against `_DEP` referencing a non-existent node.
- **`_STA` invariant enforced** — `enabled && !present` triggers `FW_BUG` log and demotes enabled bit; defense against firmware bugs presenting impossible status.
- **Hotplug profile sysfs gate** — `/sys/firmware/acpi/hotplug/<prefix>/enabled = 0` refuses every subsequent hotplug for that prefix.
- **Per-eject capability check** — eject (`_EJ0`) initiated from sysfs requires CAP_SYS_ADMIN.
- **Notify handler rate-limited** — per-handle notify dispatch bounded; defense against firmware notify-flood DoS.
- **_DEP cycle detection** — cycles broken by detecting them at the second-pass and logging `FW_BUG`.
- **Bus uevent sanitization** — `ACPI_HID` / `ACPI_CID` values UTF-8-validated before export.
- **Per-namespace-walk depth bound** — defense against pathological DSDT tree depth.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `acpi_device`, `acpi_driver`, `acpi_bus_id`, dep_entry, hotplug_context; every uevent copy is `add_uevent_var` with strict size bound.
- **PAX_KERNEXEC** — `bus.c`, `scan.c`, `glue.c` core in `__ro_after_init` text; `acpi_bus_type`, `acpi_subsys_pm_ops`, scan_handler vtables, driver_ops vtables marked `__ro_after_init`.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `acpi_bus_scan`, `acpi_bus_trim`, `acpi_bus_notify`, scan-handler attach/detach, and hotplug worker entry.
- **PAX_REFCOUNT** — saturating `refcount_t` on `acpi_device.refcount`, on `acpi_driver` Arc, and on per-handle notify-handler Arc; overflow trap kills the offender. Namespace walk uses `refcount_inc_not_zero` to defeat detach races (PAX_REFCOUNT namespace walk).
- **PAX_MEMORY_SANITIZE** — zero-on-free for per-device sysfs-attribute buffers, `_DSM` output packages, `_OSC` request buffers, and hotplug context.
- **PAX_UDEREF** — SMAP/PAN enforced on every ACPI sysfs write (`enabled`, `eject`, `status`) entry; reject user-pointer deref outside canonical helpers.
- **PAX_RAP / kCFI** — `acpi_scan_handler.{attach,detach,bind,unbind}`, `acpi_driver.ops.{add,remove,notify}`, `bus_type.{match,probe,remove,uevent,pm}`, and per-OpRegion address-space handler vtables marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms in scan/hotplug error paths behind CAP_SYSLOG; suppress `%p` of `acpi_handle` and `acpi_device` pointers in trace.
- **GRKERNSEC_DMESG** — restrict per-device scan banner, `_STA` log, hotplug add/remove log, `_OSC` capability-mask log to CAP_SYSLOG.
- **Hotplug eject CAP_SYS_ADMIN** — every eject path (`/sys/bus/pci/slots/<slot>/eject`, `/sys/bus/acpi/devices/<dev>/eject`) requires CAP_SYS_ADMIN in the owning user namespace.
- **`_DSM` method allowlist** — per-device `_DSM` invocation gated by an allowlist of vetted GUIDs (NVMe, PCI runtime PM, USB, GPU, ...); attacker AML cannot probe arbitrary `_DSM` from userspace.
- **Namespace walk PAX_REFCOUNT** — every iterator increments the per-device refcount before yielding the reference; iterator drop decrements; defense against attach-during-walk UAF.
- **`_OSC` capability-mask audit** — firmware-granted capability bits logged via audit; defense against firmware silently downgrading OSPM.
- **Per-handle notify install audit** — `acpi_install_notify_handler` logs subject + handle path; defense against runtime notify hijack.
- **Bus uevent path filter** — uevent strings sanitized to ASCII-printable; defense against control-char injection into udev.

Rationale: the ACPI bus + scan layer is the kernel's authoritative converter from firmware namespace bytes into Linux `device` references. A refcount race on attach/detach, a missing CAP_SYS_ADMIN on eject, an open `_DSM` surface, or a relaxed `_OSC` capability check will let firmware-AML decide which devices Linux exposes — at runtime. CAP_SYS_ADMIN on eject, `_DSM` allowlists, kCFI on the per-driver vtables, refcount-overflow trapping on iterators, and audited `_OSC` outcomes turn the bus from "namespace walk produces whatever firmware says exists" into "namespace walk is a structurally enforced contract between the kernel and audited firmware methods only."

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- ACPICA AML interpreter (third-party BSD, imported)
- OSL shim (covered in `osl.md`)
- Per-driver internals (button, battery, ec, processor, thermal — future per-driver Tier-3)
- `_CRS` resource translation (covered in `resource.md` future Tier-3)
- Per-device PM details (covered in `device-pm.md` future Tier-3)
- Implementation code
