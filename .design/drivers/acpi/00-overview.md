# Tier-3: drivers/acpi/ — ACPI subsystem overview (ACPICA, AML interpreter, namespace, enumeration)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/00-overview.md
upstream-paths:
  - drivers/acpi/Makefile
  - drivers/acpi/Kconfig
  - drivers/acpi/internal.h
  - drivers/acpi/bus.c
  - drivers/acpi/scan.c
  - drivers/acpi/osl.c
  - drivers/acpi/osi.c
  - drivers/acpi/tables.c
  - drivers/acpi/utils.c
  - drivers/acpi/glue.c
  - drivers/acpi/resource.c
  - drivers/acpi/device_pm.c
  - drivers/acpi/device_sysfs.c
  - drivers/acpi/ec.c
  - drivers/acpi/ec_sys.c
  - drivers/acpi/event.c
  - drivers/acpi/evged.c
  - drivers/acpi/sleep.c
  - drivers/acpi/wakeup.c
  - drivers/acpi/processor*.c
  - drivers/acpi/button.c
  - drivers/acpi/battery.c
  - drivers/acpi/ac.c
  - drivers/acpi/fan*.c
  - drivers/acpi/thermal.c
  - drivers/acpi/dock.c
  - drivers/acpi/container.c
  - drivers/acpi/apei/
  - drivers/acpi/nfit/
  - drivers/acpi/numa/
  - drivers/acpi/dptf/
  - drivers/acpi/x86/
  - drivers/acpi/arm64/
  - drivers/acpi/riscv/
  - drivers/acpi/acpica/
  - drivers/acpi/cppc_acpi.c
  - drivers/acpi/property.c
  - drivers/acpi/spcr.c
  - drivers/acpi/viot.c
  - drivers/acpi/prmt.c
  - drivers/acpi/debugfs.c
  - drivers/acpi/proc.c
  - include/acpi/
  - include/linux/acpi.h
-->

## Summary

`drivers/acpi/` wraps ACPICA (third-party BSD-licensed AML core in `drivers/acpi/acpica/`) and adds the Linux-native glue layer: the OS Services Layer (`osl.c`) that ACPICA calls back into for memory mapping / IRQs / locks / sleep / I/O, the bus type and namespace-walk based device enumeration (`bus.c` + `scan.c`), per-device PM (`device_pm.c`) and sysfs (`device_sysfs.c`), table loading (`tables.c`), object-handle helpers (`utils.c`), per-driver glue (`glue.c`), resource translation (`resource.c`), and the per-spec drivers built on top — Embedded Controller (`ec.c`), Generic Event Device (`evged.c`), Event/Wakeup, Processor (idle/perf/thermal/throttle), Button, Battery, AC, Fan, Thermal, Dock, Container, PRM (Platform Runtime Mechanism), CPPC, SPCR (Serial Port Console Redirection), VIoT (Virtual I/O Topology), APEI (Platform Error), NFIT (NVDIMM Firmware Interface Table), DPTF (Dynamic Platform & Thermal Framework), and the per-arch tail under `x86/`, `arm64/`, `riscv/`.

This Tier-3 fixes the contract for the whole `drivers/acpi/` namespace and the relationship between Rookery and ACPICA. Per-piece internals (bus + scan; OSL; EC; APEI; NFIT; processor) descend from this overview as separate Tier-3 docs.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct bus_type acpi_bus_type` | the ACPI bus | `drivers::acpi::Bus` (cross-ref `bus.md`) |
| `struct acpi_device` | per-namespace-node Linux device | `drivers::acpi::Device` |
| `struct acpi_driver` | per-HID/CID driver | `drivers::acpi::Driver` |
| `struct acpi_scan_handler` | per-bus-prefix namespace-walk handler | `drivers::acpi::ScanHandler` |
| `acpi_walk_namespace(type, start, depth, descend, ascend, ctx, retval)` | namespace tree walk | `Namespace::walk` |
| `acpi_get_handle(parent, pathname, &out)` / `acpi_get_name(handle, type, buf)` / `acpi_get_object_info(handle, &info)` | per-handle queries | `Namespace::get_*` |
| `acpi_evaluate_object(handle, pathname, in, out)` / `acpi_evaluate_integer(handle, pathname, in, &val)` | per-method eval | `Aml::evaluate` |
| `acpi_install_notify_handler(handle, type, fn, ctx)` / `acpi_remove_notify_handler(...)` | per-handle notify | `Notify::install` / `remove` |
| `acpi_bus_get_status(device)` / `acpi_bus_get_status_handle(handle, &sta)` | `_STA` evaluation | `Device::get_status` |
| `acpi_bus_get_device(handle, &device)` / `acpi_bus_scan(handle)` | per-handle device enumerate | `Bus::scan` |
| `acpi_load_tables()` / `acpi_load_table(table, &idx)` / `acpi_get_table(sig, instance, &out)` | per-table install / lookup | `Tables::load` / `get` |
| `acpi_initialize_subsystem()` / `acpi_initialize_objects()` / `acpi_initialize_tables()` / `acpi_enable_subsystem()` / `acpi_initialize_glue()` | ACPICA init phases | `Acpica::init_phase_N` |
| `acpi_install_address_space_handler(handle, space_id, handler, setup, ctx)` | per-AddressSpace AML hook | `OpRegion::install_handler` |
| `acpi_subsys_pm_ops` | ACPI-aware bus PM ops | `DevicePm::subsys_ops` (cross-ref `bus.md`) |
| `acpi_dev_resource_*` / `acpi_dev_get_resources(adev, &resource_list, preproc, ctx)` | per-device _CRS translation | `Resource::get` |
| `acpi_attach_data(handle, handler, data)` / `acpi_detach_data(handle, handler)` | per-handle opaque storage | `Namespace::attach` |
| `acpi_os_*` (cross-ref `osl.md`) | OS Services Layer (memory map, IRQ install, locks, sleep, I/O) | `Osl::*` |
| `acpi_disabled` / `acpi_noirq` / `acpi_pci_disabled` | per-mode disables | `Subsystem::flags` |
| `acpi_root` / `acpi_root_dir` | namespace root + proc dir | `Subsystem::root` |
| `acpi_ec_*` / `acpi_apei_*` / `acpi_nfit_*` / `cppc_*` / `acpi_processor_*` | per-driver entry points | per-driver Tier-3 docs |

## Compatibility contract

REQ-1: ACPICA (third-party BSD-licensed AML core under `drivers/acpi/acpica/`) is consumed as a shim — Rookery re-implements every `acpi_os_*` callback (the OS Services Layer) in Rust but **reuses ACPICA C code unchanged** as the AML interpreter, table parser, and resource translator. The OSL shim provides a stable C ABI exporting `acpi_os_*` so ACPICA does not need to be modified.

REQ-2: Per-arch ACPI gate via `acpi_disabled` flag: x86 default-on, arm64 / riscv default-off until DT or efi-stub sets `EFI_BOOT` + ACPI config tables. `acpi=off` cmdline disables the subsystem before init.

REQ-3: Per-spec subsystem-init sequence: tables phase (early) -> ACPICA `acpi_initialize_tables` -> `acpi_initialize_subsystem` -> `acpi_load_tables` (parse + load DSDT/SSDTs) -> `acpi_enable_subsystem` (switch HW to ACPI mode, install handlers, init GPEs) -> `acpi_initialize_objects` (run `_INI` per-namespace) -> per-bus + per-device scan.

REQ-4: AML interpreter (`acpica/`) runs in process context on the ACPI workqueue (`kacpid_wq`); concurrent AML evaluation is serialized per the AML mutex (`AcpiOsAcquireMutex`).

REQ-5: Device enumeration via `acpi_bus_scan` walks the namespace, calls `_STA` per-node (present + enabled + functional + UI), and creates an `acpi_device` for every node that exists; per-HID/CID match runs against `acpi_scan_handlers_list` (for built-in PNP IDs like `LNXSYSTM`, `LNXPWRBN`, `LNXSYBUS`) and `acpi_driver` registry (for hot-pluggable PnP ACPI drivers).

REQ-6: `_OSI` string filter: AML may probe the OS via `_OSI("string")`; the kernel responds true for a hard-coded allowlist (`Linux`, `Module Device`, `Processor Device`, `3.0 _SCP Extensions`, `Processor Aggregator Device`, `Extended Address Space Descriptor`); `_OSI("Windows")` returns true so vendor AML targeting Windows works; non-allowlisted strings return false.

REQ-7: Per-device PM: `acpi_subsys_pm_ops` is the generic PM-bus glue that translates per-device `_PSx` (PowerState set), `_PRx` (Power Resource), `_PS0/1/2/3/4` calls into Linux PM-runtime transitions.

REQ-8: ACPI Embedded Controller (`ec.c`) provides the kernel's interface to the EC firmware (battery, lid, thermal events); the EC GPE is registered with ACPICA via `acpi_install_gpe_handler`.

REQ-9: Generic Event Device (GED) `evged.c` for arm64 / virtualized platforms wires an ACPI namespace event source (a single GSI rather than the SCI) into the kernel.

REQ-10: APEI (`apei/`) consumes the BERT, HEST, EINJ, ERST tables and provides the kernel-side platform-error reporter + ghes (Generic Hardware Error Source) handler.

REQ-11: PRM (`prmt.c`) is the Platform Runtime Mechanism — UEFI-defined invocation of signed firmware handlers at OS runtime; per-handler GUID validated and signed.

REQ-12: NUMA / VIOT / IORT / RIMT / IO-APIC parsing lives under per-arch / per-spec subtree (`numa/`, `viot.c`, `arm64/`, `riscv/`); per-table register flow funneled through `acpi_table_parse(sig, fn)`.

## Acceptance Criteria

- [ ] AC-1: `dmesg | grep -i acpi` on UEFI x86 shows `ACPI: Early table checksum verification disabled` plus the per-table parse banner (DSDT, FACP, MADT, MCFG, SRAT, ...).
- [ ] AC-2: `ls /sys/firmware/acpi/tables/` enumerates every loaded table.
- [ ] AC-3: `ls /sys/bus/acpi/devices/` lists every namespace device with present+enabled status.
- [ ] AC-4: Lid + power button events on a laptop generate input events via `acpi_button`.
- [ ] AC-5: Battery + AC sysfs (`/sys/class/power_supply/BAT0`, `AC`) populated and updated by `acpi_battery`, `acpi_ac`.
- [ ] AC-6: Thermal zones registered under `/sys/class/thermal/thermal_zoneN/` by `acpi_thermal`.
- [ ] AC-7: Processor idle states (`/sys/devices/system/cpu/cpuN/cpuidle/stateN/`) populated by `acpi_processor_idle`.
- [ ] AC-8: CPPC counters under `/sys/devices/system/cpu/cpuN/acpi_cppc/` populated when `_CPC` is present.
- [ ] AC-9: APEI test: `apei-inject` injects a corrected memory error; `ras-mc-ctl --summary` reports it.
- [ ] AC-10: PRM test: invoking a vendor PRM handler via `/dev/acpi_prm` returns the documented status.
- [ ] AC-11: kselftest `tools/testing/selftests/acpi/` plus `fwts acpi` clean.

## Architecture

`Subsystem` lives in `drivers::acpi::Subsystem`:

```
struct Subsystem {
  root: Arc<Device>,                 // acpi_root
  root_dir: KBox<ProcDirEntry>,      // /proc/acpi/ (compat)
  bus_type: &'static BusType,        // acpi_bus_type
  acpica: KCell<AcpicaShim>,
  osl: KCell<Osl>,                   // covered by osl.md
  tables: RwLock<Tables>,
  scan_handlers: RwLock<Vec<&'static ScanHandler>>,
  drivers: RwLock<Vec<Arc<Driver>>>,
  ec: Mutex<Option<Arc<EmbeddedController>>>,
  apei: Mutex<Option<Arc<Apei>>>,
  flags: AtomicU32,                  // acpi_disabled / acpi_noirq / acpi_pci_disabled
}

struct Device {
  handle: AcpiHandle,                // ACPICA opaque
  pnp: AcpiPnpInfo,                  // HID, CID, UID, bus_id
  status: AcpiDeviceStatus,          // _STA bits
  power: AcpiDevicePower,            // power resources + power-state callbacks
  flags: AcpiDeviceFlags,
  refcount: Refcount,
  children: List<Arc<Device>>,
  parent: Option<Arc<Device>>,
  driver_data: KCell<Option<Arc<dyn Any>>>,
}
```

ACPICA shim layout:

```
+----------------------------------+
| Rookery drivers/acpi/* (Rust)    |  -- bus.c, scan.c, ec.c, ...
+----------------------------------+
| acpi_os_* shim (Rust→C ABI)      |  -- osl.c
+----------------------------------+
| ACPICA core (BSD-licensed C)     |  -- drivers/acpi/acpica/*
|   table parser, AML interpreter, |
|   namespace, resource translator |
+----------------------------------+
```

The OSL boundary is the firewall. ACPICA never calls into Rust directly; it only calls `acpi_os_*` exported functions. Rookery's Rust code may call into ACPICA via the published `acpi_*` C ABI (declared in `include/acpi/acpixf.h`).

Boot ordering:
1. Early: per-arch `acpi_table_init()` -> `acpi_initialize_tables()` parses the RSDP/RSDT/XSDT and per-table headers.
2. Per-table-signature parsers fire (`acpi_table_parse("MADT", madt_parse_fn)`) for tables consumed during early init (MADT for IRQ, MCFG for PCI ECAM, SRAT for NUMA, FACP for FADT pointer).
3. `acpi_bus_init()` (subsys_initcall): `acpi_initialize_subsystem` -> `acpi_load_tables` (DSDT + SSDTs) -> `acpi_enable_subsystem` (switch to ACPI mode, install fixed-event/GPE handlers) -> `acpi_initialize_objects` (run `_INI` namespace-wide).
4. `acpi_scan_init()` walks the namespace, instantiating `acpi_device` per node and calling per-HID `acpi_scan_handler` attach.
5. Per-driver subsys_initcalls register `acpi_driver` instances (`button`, `battery`, `ac`, `fan`, `thermal`, `processor`, `ec`).

AML evaluation path:
1. Caller `acpi_evaluate_integer(handle, "_STA", NULL, &val)` (in process context).
2. ACPICA acquires the AML mutex via `AcpiOsAcquireMutex`.
3. AML interpreter walks the opcode stream; per-OpRegion read/write calls into the registered address-space handler (PCI config, EC space, embedded controller, SystemMemory, SystemIO, ...).
4. Returns to caller with `acpi_status` translated to Linux `errno` by the helper layer.

Namespace walk:
1. `acpi_walk_namespace(ACPI_TYPE_DEVICE, ACPI_ROOT_OBJECT, depth, descend_cb, ascend_cb, ctx, &retval)`.
2. Per-node: ACPICA dereferences the handle, invokes the descend callback, recurses into children, invokes ascend on the way up.
3. Callback must not call back into ACPICA in a way that would deadlock the AML mutex.

Event dispatch:
- SCI (System Control Interrupt) is the firmware-defined GSI for ACPI events; handler is `acpi_irq` (installed by `acpi_os_install_interrupt_handler`). Cross-ref `osl.md`.
- Fixed events (power button, sleep button, RTC, GPE) dispatched per `acpi_event_*`.
- GPE (General Purpose Event) per-bit handler installed via `acpi_install_gpe_handler`.

`_OSI` filter:
- Allowlist hardcoded; per-string match returns ACPI_OSI_TRUE / ACPI_OSI_FALSE.
- `_OSI("Linux")` returns true (per upstream policy — most vendor AML expects this off, but kernel keeps backward compat).
- Userland override via `osi=` cmdline.

`/proc/acpi/` (compat) + `/sys/firmware/acpi/`:
- `/sys/firmware/acpi/tables/` — raw table bytes RO.
- `/sys/firmware/acpi/interrupts/sciN/` — per-fixed-event + per-GPE counters.
- `/sys/firmware/acpi/hotplug/{container,cpu,memory,pci_root}/` — per-bus-prefix hotplug enable.
- `/sys/firmware/acpi/platform_profile` — `_PPC` integration.

Per-driver subtree map:

| File / dir | Purpose | Tier-3 |
|---|---|---|
| `acpica/` | third-party BSD AML core, table parser, namespace, resource translator | imported, OSL shim documented in `osl.md` |
| `bus.c`, `scan.c` | ACPI bus type + namespace walk + device enumeration | `bus.md` |
| `osl.c` | OS Services Layer for ACPICA | `osl.md` |
| `tables.c` | early per-signature table parser registry | `tables.md` (future) |
| `ec.c`, `ec_sys.c` | Embedded Controller | `ec.md` (future) |
| `event.c`, `evged.c`, `wakeup.c` | event / GED / wakeup | `event.md` (future) |
| `device_pm.c`, `device_sysfs.c`, `glue.c` | per-device PM + sysfs + glue | `device-pm.md` (future) |
| `resource.c` | _CRS translation | `resource.md` (future) |
| `processor_*.c`, `cppc_acpi.c` | processor idle / perf / thermal / throttle / CPPC | `processor.md` (future) |
| `button.c`, `battery.c`, `ac.c`, `fan_*.c`, `thermal*.c`, `dock.c`, `container.c` | per-PnP-ID drivers | per-driver Tier-3 (future) |
| `apei/` | platform error (BERT, HEST, EINJ, ERST) | `apei.md` (future) |
| `nfit/` | NVDIMM Firmware Interface Table | `nfit.md` (future) |
| `numa/`, `viot.c`, `spcr.c` | per-spec topology | `acpi-topology.md` (future) |
| `dptf/` | Intel Dynamic Platform & Thermal Framework | `dptf.md` (future) |
| `prmt.c` | Platform Runtime Mechanism | `prm.md` (future) |
| `x86/`, `arm64/`, `riscv/` | per-arch glue | per-arch Tier-3 (future) |

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `namespace_walk_bounded` | OOB | `acpi_walk_namespace` bounded by maximum depth + maximum nodes per depth; per-callback iteration bounded. |
| `osi_filter_constant_time` | UNIQUENESS | `_OSI` string lookup against fixed allowlist; no attacker-controlled string influences cache eviction. |
| `device_status_bits_valid` | OOB | `_STA` return bits masked to defined positions; reserved bits ignored. |
| `aml_eval_no_uaf` | UAF | `acpi_evaluate_object` output buffer reference-counted; never freed under an active iterator. |

### Layer 2: TLA+

`models/acpi/init_phase.tla` (this doc): models the ACPICA init phase ordering (`initialize_tables` -> `initialize_subsystem` -> `load_tables` -> `enable_subsystem` -> `initialize_objects`) and proves that no AML evaluation observes a partially initialized namespace.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `acpi_bus_scan` post: every namespace node with `_STA & PRESENT` has a corresponding `acpi_device` registered | `Bus::scan` |
| `acpi_evaluate_object` post: output buffer length matches ACPICA-reported length | `Aml::evaluate` |
| `_OSI` filter post: response is identical for the same query string across calls (deterministic) | `Osi::query` |
| `acpi_install_notify_handler` post: handler list is RCU-safe under concurrent fire | `Notify::install` |

### Layer 4: Verus/Creusot functional

DSDT byte stream -> ACPICA parse -> namespace tree -> per-node `_STA` -> per-HID match -> `acpi_device` instantiated -> `acpi_driver` probe. Encoded as a refinement from the DSDT byte view into the device tree.

## Hardening

- **`acpi_disabled` cmdline kill-switch** — `acpi=off` cleanly skips initialization, no partial state.
- **Table checksum verification on load** — each table's per-spec checksum re-verified at load (early checksums can be disabled by firmware quirk, but post-load is always strict).
- **DSDT/SSDT signature check** — load refused for tables whose signature does not match the expected ASCII tag.
- **_OSI hard-coded allowlist** — attacker-controlled AML cannot force the kernel to claim arbitrary OS capabilities.
- **Namespace walk depth bound** — defense against malformed AML chaining cycles (ACPICA detects, but Rookery double-checks).
- **AML method timeout** — long-running AML methods (e.g. `_PSx`) bounded by a watchdog; refusal returns `-ETIMEDOUT`.
- **GPE handler audit** — per-GPE handler install logged via audit subsystem.
- **Lockdown checks on table override** — `acpi_load_table` from userspace gated by lockdown integrity check.
- **OSPM-claim string immutability** — `_OS` returns "Microsoft Windows NT" by default for vendor-compat; cmdline override audited.
- **Sysfs table content RO** — `/sys/firmware/acpi/tables/` is RO; defense against runtime table tampering.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `acpi_device`, `acpi_driver`, per-table buffer, AML output buffer; every `copy_to_user` from these caches is bounded by `acpi_buffer.length`.
- **PAX_KERNEXEC** — `drivers/acpi/*.c` core in `__ro_after_init` text; `acpi_bus_type`, `acpi_subsys_pm_ops`, every `acpi_scan_handler` vtable, and every `acpi_driver.ops` marked `__ro_after_init`.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `acpi_evaluate_object`, `acpi_walk_namespace`, `acpi_bus_scan`, GPE handler entry, and SCI handler entry.
- **PAX_REFCOUNT** — saturating `refcount_t` on `acpi_device.refcount` and on every Arc<Device> reference; overflow trap kills the offending task.
- **PAX_MEMORY_SANITIZE** — zero-on-free for AML output buffers, table override buffers, per-device power-state caches, and ACPICA scratch buffers.
- **PAX_UDEREF** — SMAP/PAN enforced on every ACPI ioctl / sysfs write entry; reject user-pointer deref outside canonical helpers.
- **PAX_RAP / kCFI** — AML opcode dispatch tables (`AcpiGbl_AmlOpInfo` and friends) marked RO, `acpi_scan_handler.attach/detach`, `acpi_driver.ops.{add,remove,notify}`, and address-space handlers marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms in ACPI fault paths behind CAP_SYSLOG; suppress `%p` of AML method handles in trace.
- **GRKERNSEC_DMESG** — suppress ACPI traces (per-table parse, per-namespace walk, per-device _STA, per-AML evaluation banner) under CAP_SYSLOG so attackers cannot harvest motherboard model + firmware revision + GPE topology via dmesg.
- **AML opcodes RO / kCFI** — the entire AML opcode-info / dispatch table is `__ro_after_init` so a write primitive cannot redirect an opcode to attacker code.
- **`_OSI` string filter** — hard-coded allowlist; attacker AML cannot probe OS identity outside the allowlist; cmdline `osi=` override audited and lockdown-gated.
- **Table load lockdown gate** — `acpi_load_table` from any source (`/sys/firmware/acpi/tables/...`, EFI SSDT override, kexec) requires `LOCKDOWN_ACPI_TABLES` permission.
- **Namespace walk bounds** — depth + breadth bounded; defense against malformed AML cycles consuming the AML stack.
- **AML method-timeout watchdog** — defense against firmware DoS via long-running `_PSx` / `_INI`.
- **GPE / fixed-event audit** — per-handler install logged; defense against runtime GPE hijack.

Rationale: ACPI is the kernel's entry point for trusted firmware code (DSDT/SSDT bytecode interpreted by ACPICA) and for the platform's event channel. A compromised AML opcode table redirects every firmware call, a missing `_OSI` allowlist lets vendor AML claim arbitrary OS capabilities, and an open table-load surface lets attackers inject signed-looking DSDT overrides. AML opcodes in RO with kCFI, lockdown gates on table load, dmesg suppression of ACPI scrape, refcount-overflow trapping on per-device Arc, and a hard-coded `_OSI` allowlist turn `drivers/acpi/` from "firmware is implicitly trusted across the AML boundary" into "the AML boundary is enforced by structurally-immutable dispatch tables and capability-gated table loads."

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- ACPICA core internals (third-party BSD code, imported unchanged)
- Per-piece internals (covered in `bus.md`, `osl.md` here; `ec.md`, `apei.md`, `nfit.md`, `processor.md`, `prm.md` future)
- Per-arch ACPI table parse (covered in arch-specific docs)
- Implementation code
