# Tier-3: drivers/cdx/ — AMD CDX bus (ARM APU ↔ RPU ↔ FPGA MMIO bus + MSI doorbell)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/cdx/cdx.c
  - drivers/cdx/cdx.h
  - drivers/cdx/cdx_msi.c
  - drivers/cdx/controller/
  - include/linux/cdx/cdx_bus.h
-->

## Summary

The CDX bus (`drivers/cdx/`) is AMD's Linux device-model representation of a hardware architecture used by AMD/Xilinx Versal Adaptive Compute Acceleration Platforms (Versal Net). A Versal SoC contains three execution domains: APUs (Cortex-A78 Application CPUs running Linux), RPUs (Cortex-R52 Realtime CPUs running firmware), and the FPGA programmable logic. The RPU acts as the gatekeeper for the FPGA — it loads the FPGA bitstream, enumerates the FPGA-defined devices (device personality, resource ranges, MSI doorbell addresses), and mediates discover / reset / rescan operations from the APU side. The CDX bus exposes these FPGA-loaded devices as `cdx_device` objects on `cdx_bus_type` so that APU-side Linux drivers can bind to FPGA peripherals as if they were normal MMIO devices. The companion `cdx_msi.c` builds an MSI domain on top of the GIC-ITS that uses a per-device doorbell-write protocol (RPU firmware programmed via the `dev_configure` callback) so FPGA devices can deliver MSI-like events back to the APU. Hot-add via a `rescan` sysfs attribute lets the FPGA bitstream be reloaded at runtime and the APU device tree updated.

This Tier-3 covers `cdx.c` (~971 lines: bus_type, controller register, device add, MMIO resource sysfs binding, reset, debugfs), `cdx_msi.c` (~193 lines: MSI domain init + doorbell write protocol), `cdx.h` (driver-private headers), and the public UAPI in `include/linux/cdx/cdx_bus.h`.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct cdx_device` | per-CDX-device Linux device object | `drivers::cdx::Device` |
| `struct cdx_driver` | per-driver descriptor with `match_id_table: cdx_device_id[]` | `drivers::cdx::Driver` |
| `struct cdx_controller` | per-CDX-controller (per-RPU-channel) | `drivers::cdx::Controller` |
| `struct cdx_ops` | controller callbacks: `bus_enable`, `bus_disable`, `scan`, `dev_configure` | `drivers::cdx::Ops` |
| `struct cdx_dev_params` | parameters passed to `cdx_device_add` (vendor/device/IRQ/etc.) | `drivers::cdx::DevParams` |
| `cdx_bus_type` | bus_type registered at module_init | `drivers::cdx::BUS_TYPE` |
| `cdx_bus_match(dev, drv)` / `cdx_probe` / `cdx_remove` / `cdx_shutdown` | bus_type ops | `drivers::cdx::Bus::*` |
| `__cdx_driver_register(drv, owner)` / `cdx_driver_unregister(drv)` | driver register UAPI | `drivers::cdx::register_driver` / `_unregister` |
| `cdx_device_add(dev_params)` / `cdx_bus_add` / `cdx_unregister_devices` | per-device lifecycle | `drivers::cdx::Device::add` / `Bus::add` / `_unregister_devices` |
| `cdx_register_controller(cdx)` / `cdx_unregister_controller(cdx)` | per-controller lifecycle (CONTROLLER namespace) | `drivers::cdx::register_controller` / `_unregister` |
| `cdx_dev_reset(dev)` | per-device reset via RPU `dev_configure(CDX_DEV_RESET_CONF)` | `drivers::cdx::Device::reset` |
| `cdx_set_master(dev)` / `cdx_clear_master(dev)` | bus-mastering enable/disable | `drivers::cdx::Device::set_master` / `_clear_master` |
| `cdx_enable_msi(dev)` / `cdx_disable_msi(dev)` | MSI enable via `dev_configure(CDX_DEV_MSI_ENABLE)` | `drivers::cdx::Msi::enable` / `_disable` |
| `cdx_msi_domain_init(dev)` | build MSI domain on GIC-ITS parent | `drivers::cdx::Msi::domain_init` |
| `cdx_msi_write_msg(irq_data, msg)` / `_irq_lock` / `_irq_unlock` / `cdx_msi_prepare` | MSI doorbell program | `drivers::cdx::Msi::write_msg` / `_lock` / `_unlock` / `_prepare` |
| `CDX_DEV_MSI_CONF` / `_BUS_MASTER_CONF` / `_RESET_CONF` / `_MSI_ENABLE` | dev_configure type discriminator | `drivers::cdx::DevConfigType::*` |
| `MAX_CDX_DEV_RESOURCES = 4` | per-device MMIO resource cap | `drivers::cdx::MAX_RESOURCES` |
| `MAX_CDX_CONTROLLERS = 16` | per-system controller cap | `drivers::cdx::MAX_CONTROLLERS` |
| `cdx_controller_ida` | controller ID allocator | `drivers::cdx::CONTROLLER_IDA` |

## Compatibility contract

REQ-1: `cdx_bus_type.name == "cdx"`; ABI surface `/sys/bus/cdx/` with per-device groups (vendor, device, modalias, driver_override, resource, reset, enable, …) + bus-wide `rescan`.

REQ-2: Per-device identification: 16-bit vendor + 16-bit device (+ 16-bit subvendor / subdevice = `CDX_ANY_ID` by default); modalias `cdx:v<vend>d<dev>sv<subv>sd<subdev>`.

REQ-3: Match algorithm: `cdx_bus_match` honors `driver_override` (writable sysfs); else walks `match_id_table` matching `(vendor, device)` with `CDX_ANY_ID` wildcards.

REQ-4: Per-device MMIO resources: up to `MAX_CDX_DEV_RESOURCES = 4` per device; exposed via sysfs binary attribute `resource<N>` for userspace mmap (with PFNMAP semantics).

REQ-5: Per-system controllers: up to `MAX_CDX_CONTROLLERS = 16` registered via `cdx_register_controller`; each controller corresponds to one RPU-mediated MMIO channel. Controller ID encoded into bus number via `CDX_CONTROLLER_ID_SHIFT = 4` + `CDX_BUS_NUM_MASK = 0xF`.

REQ-6: Hot-add via bus-attribute `rescan`: write `1` → `cdx_unregister_devices(bus)` then for each compatible OF node (`xlnx,versal-net-cdx`) call `cdx->ops->scan(cdx)` to repopulate.

REQ-7: Reset: `CDX_DEV_RESET_CONF` calls into `cdx_drv->reset_prepare`, issues `dev_configure(RESET)` to RPU, then calls `reset_done`. Reset is per-device; bus-level reset not supported.

REQ-8: Bus-mastering: per-device `cdx_set_master` / `cdx_clear_master` via `CDX_DEV_BUS_MASTER_CONF` — APU explicitly enables FPGA-device's DMA initiator role.

REQ-9: MSI: per-device MSI vectors allocated against parent GIC-ITS domain; per-vector MSI message (`{addr_hi, addr_lo, data}`) programmed via doorbell write protocol: APU calls `cdx_msi_write_msg` which stores pending message + on `irq_bus_sync_unlock` invokes `dev_configure(CDX_DEV_MSI_CONF)` to push to RPU which programs the FPGA device's MSI table.

REQ-10: `cdx_msi_prepare` looks up the FPGA device's `msi_dev_id` (requestor-ID) via OF `msi-map` / `msi-map-mask` → parent ITS device ID; stored in `info->scratchpad[0]`.

REQ-11: Controller register ns: `EXPORT_SYMBOL_NS_GPL(... , "CDX_BUS_CONTROLLER")` — only controller drivers (e.g., MCDI-over-shared-mem) can import the lifecycle symbols.

REQ-12: DMA configure via `cdx_dma_configure` → `of_dma_configure` + `iommu_device_use_default_domain` (FPGA-device DMA stream-IDs SMMU-mapped via msi-map equivalent).

## Acceptance Criteria

- [ ] AC-1: On Versal Net board with FPGA bitstream loaded: `ls /sys/bus/cdx/devices/` shows enumerated CDX devices (e.g., `cdx-0:00`, `cdx-0:01`).
- [ ] AC-2: `cat /sys/bus/cdx/devices/<dev>/modalias` returns `cdx:v<NNNN>d<NNNN>sv<NNNN>sd<NNNN>`.
- [ ] AC-3: Per-device driver bind: register a `cdx_driver` with matching `cdx_device_id` → driver `probe` called.
- [ ] AC-4: Per-device reset: `echo 1 > /sys/bus/cdx/devices/<dev>/reset` triggers `reset_prepare` + RPU reset + `reset_done`.
- [ ] AC-5: MSI: bound driver requests MSI vectors via `cdx_enable_msi` + `platform_msi_domain_alloc_irqs`; vectors fire on FPGA-device interrupt.
- [ ] AC-6: Hot-rescan: write `1` to `/sys/bus/cdx/rescan` → all devices unregistered + repopulated (FPGA bitstream may have changed).
- [ ] AC-7: Resource map: userspace mmap of `/sys/bus/cdx/devices/<dev>/resource0` gets FPGA MMIO region; valid + size matches.
- [ ] AC-8: VFIO-cdx passthrough: bind FPGA device to vfio-cdx; QEMU guest sees device.

## Architecture

`Controller` (per-RPU-mediated channel):

```
struct Controller {
  dev: &Device,            // owning platform device (xlnx,versal-net-cdx)
  priv: *mut (),           // controller-private state (MCDI buffer, …)
  msi_domain: &IrqDomain,  // built from cdx_msi_domain_init(parent_dev)
  id: u32,                 // 0..15 allocated via cdx_controller_ida
  controller_registered: AtomicBool,
  ops: &CdxOps,
}

struct Device {
  dev: KBox<DeviceCore>,
  cdx: &Controller,
  vendor: u16,
  device: u16,
  subvendor: u16,
  subdevice: u16,
  bus_num: u8,         // [controller_id:4 | bus:4]
  dev_num: u8,
  msi_dev_id: u32,
  num_msi: u32,
  res: [Resource; 4],
  num_res: u32,
  driver_override: Option<KString>,
  irqchip_lock: Mutex<()>,
  msi_write_pending: AtomicBool,
  debugfs_dir: Option<&Dentry>,
}
```

Controller register `register_controller(cdx)`:
1. `ida_alloc_range(cdx_controller_ida, 0, MAX_CDX_CONTROLLERS-1)`.
2. `cdx->msi_domain = cdx_msi_domain_init(cdx->dev)`.
3. Mark `controller_registered = true`.

Per-device add `Device::add(dev_params)`:
1. Allocate `cdx_device`, populate vendor/device/IRQ/resources from `dev_params`.
2. `dev->bus = &cdx_bus_type`; `dev_set_name("cdx-%d:%02x", controller_id, dev_num)`.
3. Allocate per-resource binary sysfs attributes (`resource0..N`) for userspace mmap.
4. `device_add` → `cdx_bus_match` + `cdx_probe` fires for matched drivers.

Scan `Bus::scan` (controller callback invoked from `rescan_store`):
1. RPU side reads FPGA descriptor table.
2. For each descriptor: build `cdx_dev_params` + call `cdx_device_add`.

MSI write `Msi::write_msg(irq_data, msg)`:
1. Store `msg` in `msi_desc->msg`; set `msi_write_pending = true`.
2. Do NOT issue MMIO from atomic context (controller callback may sleep — talks to RPU).
3. `irq_bus_sync_unlock` (preemptible) then fires:
   - Read pending message.
   - Build `cdx_device_config { type = MSI_CONF, msi = { addr, data, msi_index } }`.
   - Call `cdx->ops->dev_configure(cdx, bus_num, dev_num, &dev_config)` → RPU programs FPGA-device MSI table.

MSI prepare `Msi::prepare(msi_domain, dev, nvec, info)`:
1. `of_map_id(parent->of_node, msi_dev_id, "msi-map", "msi-map-mask", NULL, &dev_id)` — resolve into ITS device ID.
2. `info->scratchpad[0].ul = dev_id` (passed to GIC-ITS).
3. Forward to parent `msi_prepare`.

Hot-rescan `rescan_store`:
1. Parse `1` from sysfs write.
2. Hold `cdx_controller_lock` (single rescan at a time across system).
3. `cdx_unregister_devices(&cdx_bus_type)` — detach every device.
4. For each compatible OF node: `cdx->ops->scan(cdx)` — RPU re-enumerates + adds new devices.

Reset `Device::reset`:
1. `cdx_drv->reset_prepare(cdx_dev)` — driver quiesces.
2. `cdx->ops->dev_configure(cdx, bus_num, dev_num, &(struct cdx_device_config){.type = RESET_CONF})` — RPU resets the FPGA device.
3. `cdx_drv->reset_done(cdx_dev)` — driver re-init.

## Hardening

- **`MAX_CDX_CONTROLLERS = 16`** — fixed cap on controller IDA range.
- **`MAX_CDX_DEV_RESOURCES = 4`** — fixed per-device MMIO resource count.
- **`cdx_controller_lock` over rescan** — prevents concurrent rescan + device-add races.
- **MSI write deferred to sync-unlock context** — RPU mailbox call cannot block IRQ-context.
- **Per-device `irqchip_lock` mutex** — concurrent MSI program serialized.
- **`msi-map` parse via `of_map_id`** — refuses missing / malformed `msi-map` property.
- **Device release via `cdx_device_release`** — `kfree(cdx_dev)` only after `device_del` confirms all refs gone.
- **`cdx_bus_add` / `cdx_register_controller` namespaced** — only controller drivers can call.
- **`controller_registered` flag** — refuses `dev_configure` to a half-registered controller.
- **Per-resource sysfs binary attr 0440** — root-only mmap of FPGA MMIO.
- **DMA bypass refusal** — `cdx_dma_configure` rejects devices without IOMMU stream-ID mapping unless `driver_managed_dma`.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist slab caches for `cdx_device`, `cdx_controller`, `cdx_dev_params`, MSI message scratch, and resource bin-attribute buffers; sysfs `driver_override` + `rescan` writes bounded by `kstrtobool` / `driver_set_override`.
- **PAX_KERNEXEC** — CDX driver core in W^X kernel image; `cdx_bus_type`, `cdx_msi_irq_chip`, `cdx_msi_domain_info`, `cdx_msi_ops`, and `cdx_dev_groups` placed in `__ro_after_init`.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `cdx_bus_match`, `cdx_probe`, `cdx_msi_write_msg`, `rescan_store`, and `dev_configure` callbacks.
- **PAX_REFCOUNT** — saturating `refcount_t` on `cdx_device` via `get_device`/`put_device`; per-controller IDA-tracked id never reused while old controller still referenced.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `cdx_device`, MSI message metadata, per-device resource arrays, and `driver_override` strings; per-controller `priv` zeroed on unregister.
- **PAX_UDEREF** — SMAP/PAN enforced on sysfs entry (`driver_override_store`, `rescan_store`, `reset_store`) and on resource bin-attribute mmap; no raw user-pointer deref.
- **PAX_RAP / kCFI** — `cdx_bus_type` ops, `cdx_ops` (controller callbacks `bus_enable`, `bus_disable`, `scan`, `dev_configure`), `cdx_msi_ops` (`msi_prepare`, `set_desc`), and `cdx_msi_irq_chip` vtable marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms exposure of CDX symbols behind CAP_SYSLOG; suppress `%p` of `cdx_device`, `cdx_controller->priv`, and MSI doorbell addresses in tracepoints / dev_dbg.
- **GRKERNSEC_DMESG** — restrict controller register, rescan, reset, MSI-program-fail, and bus-master toggle banners to CAP_SYSLOG so attackers cannot enumerate FPGA-device layout via dmesg.
- **MSI doorbell PAX_USERCOPY** — `cdx_msi_config { addr, data, msi_index }` is kernel-internal; never copied to userspace; `irq_bus_sync_unlock` path bounded against per-device `num_msi`.
- **Hot-add CAP_SYS_ADMIN** — `rescan` bus-attribute, `driver_override` store, and `reset` device-attribute require CAP_SYS_ADMIN in the owning user-ns.
- **RPU firmware signature** — FPGA bitstream + RPU firmware loaded via `request_firmware` are signature-verified by the platform firmware (BootROM / PLM); host CDX driver refuses to register a controller whose attestation chain is broken.
- **MMIO range bounded** — per-device resource entries strictly bounded by `MAX_CDX_DEV_RESOURCES`; resource `start`/`end` validated against controller-declared MMIO window before sysfs bin-attr exposure.
- **Controller register namespace** — `cdx_register_controller`/`cdx_bus_add`/`cdx_device_add` exported in `CDX_BUS_CONTROLLER` symbol namespace; out-of-tree modules cannot smuggle a fake controller without explicit MODULE_IMPORT_NS.
- **Driver-override bounded** — string-copy capped via `driver_set_override`; world-write blocked by 0644 (root-only) sysfs perms.
- **Bus-mastering audit** — `cdx_set_master` / `cdx_clear_master` logged at audit boundary; FPGA-device DMA initiator state transitions auditable.

Rationale: the CDX bus is the APU's only window into FPGA-loaded devices on Versal Net SoCs, and those FPGA devices are full bus-mastering DMA initiators against the APU's memory. A compromised or malformed `dev_configure` call can reprogram an FPGA-device's MSI doorbell to inject spurious interrupts; an unrestricted `rescan` lets a non-root user reload arbitrary bitstreams (if the platform allows); an MMIO resource without bounds checking lets a userland `mmap` of `resource<N>` read arbitrary host memory if the FPGA window wasn't validated. RAP/kCFI on the controller-ops + MSI-ops vtables, refcount-trapping on `cdx_device`, MEMORY_SANITIZE on MSI metadata, mandatory PLM-signed firmware, CAP_SYS_ADMIN on rescan/reset/override, and namespace-gated controller symbols turn CDX from a "FPGA control-plane primitive" into a structurally constrained APU bus.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- AMD CDX MCDI controller (`drivers/cdx/controller/`) — separate Tier-3.
- PLM (Platform Loader & Manager) firmware on-chip protocol.
- VFIO-cdx passthrough internals (covered in `drivers/vfio/cdx/`).
- FPGA bitstream signing / attestation (covered in firmware loader Tier-3).
- Per-FPGA-device driver internals.
- Implementation code
