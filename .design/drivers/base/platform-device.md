# Tier-3: drivers/base/platform.c — platform bus (platform_device + platform_driver)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/base/00-overview.md
upstream-paths:
  - drivers/base/platform.c
  - drivers/base/platform-msi.c
  - include/linux/platform_device.h
-->

## Summary

`platform.c` (~1540 LOC) implements the kernel's "platform bus" — a pseudo-bus that hosts every non-discoverable device the kernel must know about: on-SoC peripherals (UART, I2C, SPI, USB-host, eMMC, DMA-controller, IOMMU, watchdog, GPIO, pinctrl), board-fixed devices (NOR flash, fixed LED), and DT / ACPI populated nodes that have no real bus parent. Unlike PCI / USB / DT-of-platform-companion buses where devices announce themselves, platform-bus devices are *enumerated* by firmware (DT, ACPI, board file) and *registered* by code that constructs `struct platform_device` and calls `platform_device_register`. Drivers register via `platform_driver_register` and the bus `match` callback pairs them up by DT compatible, ACPI HID, `platform_device_id` table, or driver name.

This Tier-3 covers: `struct platform_device` (the per-instance class device with `resource[]` and `id`), `struct platform_driver` (probe/remove/shutdown + `id_table` / `of_match_table` / `acpi_match_table`), the bus_type vtable (`platform_bus_type`: match, uevent, probe, remove, shutdown, dma_configure, pm), resource accessors (`platform_get_resource`, `platform_get_irq`, `platform_get_irq_byname`, `devm_platform_ioremap_resource`), DT / ACPI populate (`of_platform_populate` / `acpi_create_platform_device` populate the bus), and the auto-allocated dev-id IDA.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct platform_device` | per-instance: name, id, dev, num_resources, resource[], id_entry, mfd_cell, archdata | `drivers::base::platform::Device` |
| `struct platform_driver` | probe / remove / shutdown / suspend / resume / id_table / driver / prevent_deferred_probe / driver_managed_dma | `drivers::base::platform::Driver` |
| `platform_bus_type` | `bus_type`: match, uevent, probe, remove, shutdown, dma_configure, dma_cleanup, pm | `Bus` (singleton) |
| `platform_bus` (struct device) | root device parented onto sysfs `/sys/devices/platform` | `Bus::root_dev` |
| `platform_device_alloc(name, id)` | alloc `platform_object` (pdev + flex-array name) | `Device::alloc` |
| `platform_device_add_resources(pdev, res, num)` | duplicate + attach resource array | `Device::add_resources` |
| `platform_device_add_data(pdev, data, size)` | attach platform_data blob | `Device::add_data` |
| `platform_device_add(pdev)` | publish to bus: assign IDA-id if auto, set bus, set parent, `device_add` | `Device::add` |
| `platform_device_register(pdev)` / `_register_full(info)` / `_unregister(pdev)` | full register helpers | `Device::register` / `_register_full` / `_unregister` |
| `__platform_driver_register(drv, owner)` / `platform_driver_unregister(drv)` | register driver | `Driver::register` / `_unregister` |
| `__platform_driver_probe(drv, probe)` (init-only) | one-shot init-time probe; sets `.probe = platform_probe_fail` after probe | `Driver::__probe_init` |
| `platform_get_resource(pdev, type, num)` / `_get_mem_or_io` / `_get_resource_byname` | resource lookup by index / type | `Device::get_resource` |
| `platform_get_irq(pdev, num)` / `_optional` / `_byname` / `_byname_optional` | IRQ resource lookup (OF/ACPI-aware) | `Device::get_irq` |
| `platform_irq_count(pdev)` | count of IRQ resources | `Device::irq_count` |
| `devm_platform_ioremap_resource(pdev, idx)` / `_byname` / `_get_and_ioremap_resource` | devres-managed ioremap | `Device::devm_ioremap_resource` |
| `devm_platform_get_irqs_affinity(...)` | devres-managed multi-IRQ + affinity descriptor alloc | `Device::devm_get_irqs_affinity` |
| `platform_match(dev, drv)` | bus match: driver_override → of_match → acpi_match → id_table → name | `Bus::match` |
| `platform_uevent(dev, env)` | emit MODALIAS (OF / ACPI / platform-prefix) | `Bus::uevent` |
| `platform_probe(dev)` / `platform_remove(dev)` / `platform_shutdown(dev)` | bus-level probe/remove/shutdown wrappers | `Bus::probe` / `_remove` / `_shutdown` |
| `platform_dma_configure(dev)` / `_dma_cleanup` | bind to IOMMU default domain (cross-ref `iommu-core.md`) | `Bus::dma_configure` |
| `platform_dev_pm_ops` | `dev_pm_ops` with USE_PLATFORM_PM_SLEEP_OPS + runtime suspend | `Bus::pm_ops` |
| `platform_device_release(dev)` | release: drop of_node ref, free platform_data + mfd_cell + resource | `Device::release` |
| `platform_devid_ida` | global IDA for `PLATFORM_DEVID_AUTO` | `Bus::devid_ida` |

## Compatibility contract

REQ-1: `platform_device` is a `struct device` with `dev.bus = &platform_bus_type`; instance identity is `(name, id)` with `id == PLATFORM_DEVID_NONE` meaning "name is unique" and `id == PLATFORM_DEVID_AUTO` meaning "allocate from IDA".

REQ-2: `platform_device_alloc` allocates a single `platform_object` whose flex-array stores the name, sets `pdev.name = pa->name`, `device_initialize(&pdev.dev)`, attaches `platform_device_release`, sets default 32-bit DMA mask via `setup_pdev_dma_masks`.

REQ-3: `platform_device_add_resources` does a `kmemdup_array` of the caller's resource[] so the platform_device owns its copy; same for `platform_device_add_data`.

REQ-4: `platform_match` order: `driver_override` (forced bind via `driver_override` sysfs) → `of_driver_match_device` → `acpi_driver_match_device` → `platform_match_id(id_table, pdev)` → strict `strcmp(pdev->name, drv->name)`.

REQ-5: `platform_probe` order: `of_clk_set_defaults` (set DT default clk parents) → `dev_pm_domain_attach(dev, PD_FLAG_ATTACH_POWER_ON | PD_FLAG_DETACH_POWER_OFF)` (cross-ref `pmdomain/genpd.md`) → `drv->probe(pdev)`; on `-EPROBE_DEFER` if `prevent_deferred_probe`, demote to `-ENXIO`.

REQ-6: `platform_dma_configure` walks the firmware node (OF or ACPI) and configures DMA-mask + IOMMU default domain attach via `iommu_device_use_default_domain` (cross-ref `iommu-core.md`).

REQ-7: `platform_get_resource(pdev, type, num)` linear-scans `pdev->resource[]` returning the `num`-th resource matching `type`; returns NULL when absent (caller turns into -ENXIO).

REQ-8: `platform_get_irq(pdev, num)` is OF-aware (calls `of_irq_get`), ACPI-aware (resolves `_CRS` IRQs), then falls back to scanning `resource[]` for `IORESOURCE_IRQ`; returns `-EPROBE_DEFER` if the IRQ domain isn't ready yet.

REQ-9: MODALIAS uevent format: OF-style `MODALIAS=of:N<node-name>Cc<compatible>` if OF, ACPI-style `MODALIAS=acpi:<HID>:<CID>:...` if ACPI, else `MODALIAS=platform:<pdev-name>`.

REQ-10: `platform_device_register_full(pdevinfo)` is the canonical builder API used by board-files / MFD-cell expansion: allocates pdev, copies name + id + resources + data + dma_mask + properties + fwnode, then `platform_device_add`.

REQ-11: `__platform_driver_probe` is an `__init_or_module` helper for drivers whose probe lives in `__init` and is freed after boot: registers with `.probe = drv->probe`, walks the bus to bind once, then replaces `.probe = platform_probe_fail` so later hot-plug events bail.

REQ-12: Per-bus `dev_pm_ops` integrates with PM-domains, runtime-PM, and S3/S0ix suspend via `USE_PLATFORM_PM_SLEEP_OPS` macro (calls `drv->suspend` / `drv->resume` then PM-domain hooks).

## Acceptance Criteria

- [ ] AC-1: Boot on any DT or ACPI system populates `/sys/bus/platform/devices/` with the expected per-SoC peripherals (UART, GPIO, pinctrl, eMMC, ...) each carrying `modalias` and `driver_override` files.
- [ ] AC-2: `platform_device_alloc("test", PLATFORM_DEVID_AUTO)` + `platform_device_add_resources` + `platform_device_add` succeeds; `id` is filled from IDA; `/sys/devices/platform/test.0` appears.
- [ ] AC-3: `platform_get_irq` returns `-EPROBE_DEFER` when the parent IRQ controller isn't initialized yet; consumer correctly defers and re-tries.
- [ ] AC-4: `platform_driver_register` followed by `platform_device_register` (matching name) binds; `/sys/bus/platform/drivers/<drv>/<dev>` symlink appears.
- [ ] AC-5: `driver_override` write of "no_driver" prevents auto-binding even when `match` would succeed.
- [ ] AC-6: `dev_pm_domain_attach` integration: probe of a DT node with `power-domains` phandle attaches the genpd before `drv->probe` runs.

## Architecture

`Bus` lives in `drivers::base::platform`:

```
struct Device {
  name: KStr,
  id: i32,                            // PLATFORM_DEVID_NONE / _AUTO / explicit
  id_auto: bool,
  dev: KDevice,
  platform_dma_mask: u64,
  dma_parms: DeviceDmaParams,
  num_resources: u32,
  resource: KBox<[Resource]>,
  id_entry: Option<&'static PlatformDeviceId>,
  mfd_cell: Option<KBox<MfdCell>>,
  archdata: PdevArchdata,
}

struct Driver {
  probe: Option<fn(&Device) -> Result<()>>,
  remove: Option<fn(&Device)>,
  shutdown: Option<fn(&Device)>,
  suspend: Option<fn(&Device, PmMessage) -> Result<()>>,
  resume: Option<fn(&Device) -> Result<()>>,
  driver: KDriver,
  id_table: Option<&'static [PlatformDeviceId]>,
  prevent_deferred_probe: bool,
  driver_managed_dma: bool,
}

struct Bus {
  bus_type: BusType,                  // platform_bus_type
  root_dev: KDevice,                  // platform_bus (parented at /sys/devices/platform)
  devid_ida: Ida,                     // platform_devid_ida
}
```

Add `Device::add`:
1. If `id == PLATFORM_DEVID_AUTO`: `ida_alloc(&devid_ida) → id` + set `id_auto = true`.
2. Build `dev_set_name(&pdev->dev, "%s.%d", pdev->name, pdev->id)` (or just `pdev->name` if `PLATFORM_DEVID_NONE`).
3. `pdev->dev.bus = &platform_bus_type`; `pdev->dev.parent = &platform_bus` unless caller already set.
4. Walk `resource[]`: `insert_resource_conflict(parent_region, &resource[i])` for `IORESOURCE_MEM`/`_IO`; reject overlap.
5. `device_add(&pdev->dev)` → triggers bus `match` over all registered drivers.

Bus probe `Bus::probe(dev)`:
1. `drv = to_platform_driver(_dev->driver)`.
2. If `drv->probe == platform_probe_fail`: -ENXIO (init-only probe driver re-bind blocked).
3. `of_clk_set_defaults(_dev->of_node, false)` — pre-program DT clk parents.
4. `dev_pm_domain_attach(_dev, PD_FLAG_ATTACH_POWER_ON | PD_FLAG_DETACH_POWER_OFF)` — attach + power-on PM-domain.
5. `drv->probe(pdev)`.
6. If `-EPROBE_DEFER` and `drv->prevent_deferred_probe`: demote to `-ENXIO`.

Resource lookup `Device::get_irq(pdev, num)`:
1. If `dev->of_node` set: `of_irq_get(dev->of_node, num)` — walks DT `interrupts` / `interrupts-extended`.
2. Else if `dev->fwnode` is ACPI: `acpi_irq_get(...)` — walks `_CRS`.
3. Else: linear scan `resource[]` for `IORESOURCE_IRQ`.
4. -EPROBE_DEFER if IRQ domain not yet registered.

DT populate (`drivers/of/platform.c` calls into here): `of_platform_populate(root, matches, lookup, parent)` walks DT, builds `platform_device` per matching node via `of_platform_device_create_pdata`, calls `platform_device_register`.

ACPI populate (`drivers/acpi/scan.c`): `acpi_create_platform_device(adev, properties)` constructs `platform_device` from `_CRS` resources, calls `platform_device_register_full`.

## Hardening

- `platform_object` flex-array name embeds the device name in the same allocation; preventing dangling-name UAFs on `platform_device_alloc`.
- Resource array `kmemdup_array` makes pdev own its resources independently of the caller's lifetime.
- IDA-allocated `id` is released in `platform_device_release` when refcount reaches zero.
- `dev_pm_domain_attach` failure aborts probe before `drv->probe` so partial-attach UAFs are impossible.
- `platform_match` always validates `driver_override` first so userspace bind requests are honoured.
- `prevent_deferred_probe` lets drivers bail on EPROBE_DEFER (e.g. when no realistic deferred-resource will appear later).

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `platform_object` (pdev + name), `resource[]` duplicate buffers, platform_data blobs, and sysfs-printed strings; uevent buffers strictly length-bounded.
- **PAX_KERNEXEC** — `platform.c` text W^X; `platform_bus_type`, `platform_dev_pm_ops`, `platform_dev_groups` live in `__ro_after_init` text.
- **PAX_RANDKSTACK** — randomize kernel stack offset across `platform_probe`, `platform_match`, `platform_get_irq`, `platform_device_add`, `platform_dma_configure`.
- **PAX_REFCOUNT** — saturating `refcount_t` on `platform_device.dev.kobj.kref` (inherited from `struct device`); overflow trap defeats refcount-leak UAFs on hot-unplug.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `platform_object`, `resource[]` dup buffers, `platform_data`, `mfd_cell`, and `dma_parms` so stale resource ranges cannot bleed across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every sysfs / uevent entry into platform-bus; reject user-pointer deref outside canonical helpers.
- **PAX_RAP / kCFI** — `platform_bus_type` vtable (`match`, `uevent`, `probe`, `remove`, `shutdown`, `dma_configure`, `dma_cleanup`, `pm`) and `platform_driver` `probe`/`remove`/`shutdown` indirect dispatch kCFI-typed in `__ro_after_init` cells.
- **GRKERNSEC_HIDESYM** — gate kallsyms exposure of bus-internal symbols (`platform_match`, `platform_probe`, `platform_dma_configure`) behind CAP_SYSLOG; suppress `%p` in platform tracepoints.
- **GRKERNSEC_DMESG** — restrict platform probe / match / EPROBE_DEFER / dma_configure banners to CAP_SYSLOG so attackers cannot map board topology via dmesg.
- **platform_device PAX_REFCOUNT** — saturating refcount on `pdev->dev`; refuse double-free; refuse `platform_device_unregister` after release callback already ran.
- **MODALIAS bounded** — `platform_uevent` emits MODALIAS with strict caps on OF/ACPI string lengths; truncate + log rather than overflow uevent buffer.
- **DT/ACPI populate kCFI** — `of_platform_populate` and `acpi_create_platform_device` indirect calls into `platform_device_register_full` kCFI-typed; refuse mis-typed populate paths injected via corrupted firmware tables.
- **IORESOURCE_* range check** — `platform_device_add` validates each `IORESOURCE_MEM` / `IORESOURCE_IO` / `IORESOURCE_IRQ` against `insert_resource_conflict`; refuse overlapping ranges against kernel image / `.text` / `.rodata` regions.
- **driver_override gating** — `driver_override` sysfs writes require CAP_SYS_ADMIN in the device's user namespace; refuse pathological override-strings.
- **dma_configure audit** — `platform_dma_configure` attach to IOMMU default domain logged and rate-limited; refuse if `iommu_device_use_default_domain` rejects.

Rationale: the platform bus is the entry point for every non-discoverable on-SoC peripheral, including secure-boot-relevant blocks (TPM, secure-EL3 mailbox, watchdog, secure SPI controller). A corrupted `platform_device.resource[]`, a forged `driver_override` write, a missed range-check on `IORESOURCE_MEM`, or a kCFI bypass on the probe callback can hijack control of an IOMMU-protected device or expose `.text` to DMA. Refcount saturation, MODALIAS length caps, resource overlap rejection, and CAP_SYS_ADMIN on `driver_override` turn the platform bus from a convenience layer into a structural device-attach boundary.

## Open Questions

(none at this Tier-3 level)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `resource_no_overlap` | INVARIANT | `platform_device_add` walks `resource[]` and refuses any overlap with the kernel image or another platform device's reservations. |
| `pdev_no_uaf` | UAF | `platform_object` lifetime tied to `dev.kobj.kref`; release frees name + resource + data only at refcount zero. |
| `irq_get_no_oob` | OOB | `platform_get_irq(num)` returns NULL / -ENXIO for `num >= irq_count`; never indexes past `resource[]`. |
| `modalias_no_overflow` | OOB | `platform_uevent` writes MODALIAS with snprintf into the `kobj_uevent_env` buffer; refuses truncation. |

### Layer 2: TLA+

`models/platform_bus/probe_attach.tla`: proves that `dev_pm_domain_attach` and `drv->probe` execute in order; failure of `dev_pm_domain_attach` blocks `drv->probe`; failure of `drv->probe` triggers domain detach via the `goto out` epilogue.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `platform_device_add` post: `pdev` visible in `/sys/bus/platform/devices/` iff `device_add` returned 0 | `Device::add` |
| `platform_match` post: returns 1 iff one of (driver_override, of_match, acpi_match, id_table, name) matched in order | `Bus::match` |
| `platform_get_irq` post: returned IRQ number is valid for `dev` (registered with an IRQ-domain) OR a negative errno | `Device::get_irq` |

### Layer 4: Verus/Creusot functional

`platform_device_alloc → add_resources → add → driver_register → match → probe → remove → unregister → release` invariant: every alloc has a matching release; every probed driver has a matching remove; no resource leak.

## Out of Scope

- `of_platform_populate` / DT-of-platform glue (covered in future `of-platform.md`)
- `acpi_create_platform_device` / ACPI scan (covered in `acpi/scan.md`)
- `platform-msi.c` MSI domain glue (future companion doc)
- MFD cell expansion (covered in `drivers/mfd/*.md` future)
- 32-bit-only paths
- Implementation code
