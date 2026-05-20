# Tier-3: drivers/amba/ — ARM AMBA PrimeCell bus (PID/CID autoprobing + Tegra AHB)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/amba/bus.c
  - drivers/amba/tegra-ahb.c
  - include/linux/amba/bus.h
-->

## Summary

The `drivers/amba/` subsystem implements the Linux device-model glue for the ARM Advanced Microcontroller Bus Architecture (AMBA), originally created for ARM PrimeCell peripherals (UART PL011, SMC PL022, MMCI PL18x, KMI PL050, DMA PL08x/PL080, GIC) and now also used for ARM CoreSight components (ETM, ETF, ETB, TPIU, replicators, funnels). The defining feature of AMBA peripherals is a self-describing register tail: the last 32 bytes of each peripheral's MMIO region encode a Peripheral ID (PID, 4 bytes at `size-0x20..size-0x10`) and a Component ID (CID, 4 bytes at `size-0x10..size`) — when CID matches the magic constant `0xb105f00d` (legacy "PrimeCell") or `0xb105900d` (CoreSight), the bus driver knows the peripheral conforms to AMBA conventions and the PID identifies the part. The bus driver auto-reads PID + CID on first `bus->match`, populates `amba_device.periphid` + `.cid` + per-CoreSight `.uci`, and uses them to match against `amba_id` tables in registered `amba_driver`s. The companion `tegra-ahb.c` is the NVIDIA Tegra-specific AHB arbiter / prefetch / SMMU-init shim that sits in front of an AMBA bus on Tegra20/30/124 SoCs.

This Tier-3 covers `bus.c` (~706 lines: bus_type, match, probe, device alloc, driver register, runtime PM glue, proxy stub driver) and `tegra-ahb.c` (~289 lines: AHB GIZMO/PREFETCH/ARBITRATION register layout + SMMU-init handshake).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct amba_device` | per-peripheral device (extends `struct device`) | `drivers::amba::Device` |
| `struct amba_driver` | per-driver descriptor with `id_table: amba_id[]` | `drivers::amba::Driver` |
| `struct amba_id` | (periphid, mask, data) match record | `drivers::amba::IdEntry` |
| `struct amba_cs_uci_id` | CoreSight unique-component-id (devarch/devtype) | `drivers::amba::CsUciId` |
| `amba_bustype` | bus_type registered at postcore_initcall | `drivers::amba::BUS_TYPE` |
| `amba_match(dev, drv)` | bus_type.match: lazy read periphid + lookup table | `drivers::amba::Bus::match` |
| `amba_read_periphid(dev)` | ioremap tail + read 4 PID bytes + 4 CID bytes | `drivers::amba::read_periphid` |
| `amba_lookup(table, dev)` | walk amba_id table; CoreSight UCI sub-match | `drivers::amba::lookup` |
| `amba_probe(dev)` / `amba_remove(dev)` | bus_type.probe / remove: enable pclk + PM + call driver | `drivers::amba::Bus::probe` / `_remove` |
| `__amba_driver_register(drv, owner)` / `amba_driver_unregister(drv)` | driver register UAPI | `drivers::amba::register_driver` / `_unregister_driver` |
| `amba_device_alloc(name, base, size)` / `_add(dev, parent)` / `_register(dev, parent)` / `_unregister(dev)` | per-device lifecycle | `drivers::amba::Device::alloc` / `_add` / `_register` / `_unregister` |
| `amba_request_regions` / `amba_release_regions` | request_mem_region wrapper | `drivers::amba::Device::request_regions` |
| `amba_dma_configure(dev)` / `_dma_cleanup(dev)` | bus dma_configure: OF/ACPI DMA + IOMMU attach | `drivers::amba::Bus::dma_configure` |
| `amba_pm_runtime_suspend` / `_resume` | runtime PM: gate pclk | `drivers::amba::Bus::pm_runtime_*` |
| `amba_proxy_drv` | stub driver to bootstrap amba_match() | `drivers::amba::ProxyDrv` |
| `tegra_ahb_enable_smmu` (`include <soc/tegra/ahb.h>`) | program AHB_ARBITRATION_XBAR_CTRL_SMMU_INIT_DONE | `drivers::amba::tegra_ahb::enable_smmu` |
| `AMBA_CID == 0xb105f00d` / `CORESIGHT_CID == 0xb105900d` | magic CID values | `drivers::amba::AMBA_CID` / `CORESIGHT_CID` |

## Compatibility contract

REQ-1: `amba_device` layout includes `dev: struct device`, `res: struct resource`, `pclk: struct clk *`, `periphid: u32`, `periphid_lock: mutex`, `cid: u32`, `uci: amba_cs_uci_id`, `irq[AMBA_NR_IRQS=9]`, `driver_override: const char *`; userland ABI surfaces via sysfs (`id`, `resource`, `driver_override` attributes).

REQ-2: PID format: four 8-bit fields read from `[size-0x20, size-0x20+4, +8, +12]` each masked to low 8 bits + shifted into a `u32`; PID convention (per ARM AMBA spec): bits[31:24] revision, [23:20] config, [19:16] revision2, [15:12] designer (low nibble), [11:8] designer (high nibble), [7:0] part_number (low), with extension across the 4 words.

REQ-3: CID format: four 8-bit fields at `[size-0x10, +4, +8, +12]` packed into a `u32`; valid values restricted to `AMBA_CID=0xb105f00d` (PrimeCell, class 0xF) or `CORESIGHT_CID=0xb105900d` (class 0x9).

REQ-4: For CoreSight (CID class 0x9): additional `devarch` (offset 0xFBC) and `devtype` (offset 0xFCC, low byte) read for UCI sub-matching.

REQ-5: Modalias `amba:d<periphid>` emitted via uevent (`add_uevent_var(env, "MODALIAS=amba:d%08X", periphid)`); used by modprobe + udev for module auto-load.

REQ-6: `amba_match` reads PID lazily (in match handler under `periphid_lock`) when driver registration occurs and PID is not pre-set; if read fails, returns `-EPROBE_DEFER` to retry later.

REQ-7: `driver_override` (writable sysfs attribute) forces match-by-name (skip id_table) — used for VFIO passthrough; clear by writing empty string.

REQ-8: AMBA-stub-driver registered at `late_initcall_sync` to guarantee `amba_match` runs even if all real drivers are modules.

REQ-9: Per-bus DMA configuration via `of_dma_configure` (OF) or `acpi_dma_configure` (ACPI); IOMMU `iommu_device_use_default_domain` attached when `driver_managed_dma == false`.

REQ-10: Runtime PM gates `apb_pclk` clock at suspend/resume; IRQ-safe variant uses `clk_disable`/`enable` (skips prepare cycle).

REQ-11: Tegra AHB: SMMU initialization sequenced via `AHB_ARBITRATION_XBAR_CTRL_SMMU_INIT_DONE` bit; SMMU driver waits for AHB driver to set it before completing init.

REQ-12: `AMBA_NR_IRQS == 9` IRQ slots per peripheral; decoded from devicetree `interrupts` property.

## Acceptance Criteria

- [ ] AC-1: On ARM PrimeCell platform (Versatile / Versatile-Express / RealView): `ls /sys/bus/amba/devices/` shows enumerated peripherals.
- [ ] AC-2: Per-device `cat /sys/bus/amba/devices/<dev>/id` returns 8-hex periphid matching ARM spec for the part.
- [ ] AC-3: Per-driver match: `ls /sys/bus/amba/drivers/uart-pl011/` shows binding to a PL011 device.
- [ ] AC-4: CoreSight devices match via UCI: `cat /sys/bus/amba/devices/<csdev>/id` valid; `/sys/bus/coresight/` enumerates them as well.
- [ ] AC-5: `echo vfio-amba > /sys/bus/amba/devices/<dev>/driver_override` then `echo <dev> > /sys/bus/amba/devices/<dev>/driver/unbind`; `echo <dev> > /sys/bus/amba/drivers_probe` rebinds to vfio.
- [ ] AC-6: Module unload race: `modprobe -r uart-pl011` releases device cleanly; `lsmod` empty.
- [ ] AC-7: Runtime PM: idle device → `cat /sys/bus/amba/devices/<dev>/power/runtime_status` shows `suspended`; pclk gated.
- [ ] AC-8: Tegra: `dmesg | grep tegra-ahb` shows AHB init; SMMU init proceeds + completes.

## Architecture

`Bus::match(dev, drv)`:
1. `pcdev = to_amba_device(dev)`, `pcdrv = to_amba_driver(drv)`.
2. Hold `pcdev.periphid_lock`.
3. If `pcdev.periphid == 0`: call `read_periphid(pcdev)`; on err map to `-EPROBE_DEFER`; on success `dev_set_uevent_suppress(false)` + `kobject_uevent(KOBJ_ADD)`.
4. If `pcdev.driver_override` set: `strcmp` against `drv.name`.
5. Else: `lookup(pcdrv.id_table, pcdev)` → walk entries with `(periphid & mask) == id`; for CoreSight CID, sub-match via `cs_uci_id_match`.

`read_periphid(dev)`:
1. `dev_pm_domain_attach(dev, PD_FLAG_ATTACH_POWER_ON)` to power up.
2. `clk_prepare_enable(apb_pclk)` (via `amba_get_enable_pclk`).
3. Optional reset deassert: `of_reset_control_array_get_optional_shared` + `reset_control_deassert`.
4. `ioremap(res.start, size)` → `tmp`.
5. Loop 4 iterations: `pid |= (readl(tmp + size - 0x20 + 4*i) & 0xff) << (i*8)` (build PID from 4 bytes).
6. Loop 4 iterations: similar for CID.
7. If `cid == CORESIGHT_CID`: read `devarch = readl(csbase + UCI_REG_DEVARCH_OFFSET)`, `devtype = readl(csbase + UCI_REG_DEVTYPE_OFFSET) & 0xff` where `csbase = tmp + size - 4096`.
8. If `cid == AMBA_CID || cid == CORESIGHT_CID`: store periphid + cid; else `-ENODEV`.
9. `iounmap` + disable clk + detach PM.

`Bus::probe(dev)`:
1. `of_amba_device_decode_irq(pcdev)` — populate `irq[]` from OF.
2. `of_clk_set_defaults(of_node, false)`.
3. `dev_pm_domain_attach(PD_FLAG_ATTACH_POWER_ON | PD_FLAG_DETACH_POWER_OFF)`.
4. `amba_get_enable_pclk`.
5. `pm_runtime_get_noresume` + `pm_runtime_set_active` + `pm_runtime_enable`.
6. `pcdrv.probe(pcdev, id)` — driver's per-device init runs with bus clock on.
7. On err: undo runtime PM + disable pclk.

`Bus::dma_configure(dev)`:
1. OF: `of_dma_configure(dev, of_node, true)` — set DMA mask + parse `dma-ranges`.
2. ACPI: `acpi_get_dma_attr` + `acpi_dma_configure`.
3. If driver doesn't claim `driver_managed_dma`: `iommu_device_use_default_domain(dev)` — attach to default IOMMU domain.

Tegra AHB (`tegra-ahb.c`):
1. Platform driver bound to `nvidia,tegra20-ahb` / `tegra30-ahb` / `tegra124-ahb` compat strings.
2. ioremap AHB control region (legacy DT files used `base+0x4` instead of `base+0x0` — driver detects + adjusts via `INCORRECT_BASE_ADDR_LOW_BYTE`).
3. Save/restore set of GIZMO + ARBITRATION + PREFETCH + USB-master-priority registers across suspend.
4. `tegra_ahb_enable_smmu` sets `AHB_ARBITRATION_XBAR_CTRL_SMMU_INIT_DONE` after Tegra-SMMU driver finishes init — gates AHB master DMA until IOMMU is live.

## Hardening

- **Periphid read under per-device `periphid_lock`** — concurrent first-match from two driver registrations safe.
- **`-EPROBE_DEFER` on PID read failure** — never bubble a hard error from `match` (would deregister the driver from the bus).
- **`amba_proxy_drv` stub** — guarantees `match` runs even when all real drivers are modules + late-loaded.
- **`driver_override` set via `driver_set_override`** — bounded length, frees old override, exclusive of id_table match.
- **Reset/PD/clk error paths unwind in reverse order** — `read_periphid` failure paths cannot leak clk/PM refs.
- **uevent_suppress until PID readable** — avoids emitting "AMBA_ID=00000000" modalias.
- **`AMBA_NR_IRQS == 9` bound** — IRQ array indexed within this constant; OF parser caps at 9.
- **`iommu_device_use_default_domain` failure → `arch_teardown_dma_ops`** — rollback to prevent unattached-IOMMU DMA.
- **Tegra AHB MMIO offset detection** — legacy DT misalignment (low-byte 0x4) detected + corrected; no MMIO into wrong region.
- **Per-device dma_parms initialized in `amba_device_initialize`** — ensures coherent dma_mask + dma_parms before any DMA op.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist slab caches for `amba_device`, `amba_driver_override` strings, and `tegra_ahb_context` save-state buffers; sysfs writes (`driver_override`, etc.) bounded by `driver_set_override`'s length check.
- **PAX_KERNEXEC** — `drivers/amba/bus.c` text in W^X kernel image; `amba_bustype`, `amba_dev_attrs`, `amba_proxy_drv`, and `amba_pm` ops table placed in `__ro_after_init`.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `amba_match`, `amba_probe`, `amba_read_periphid`, and `tegra_ahb_probe` so PID-read MMIO timing cannot leak stack layout.
- **PAX_REFCOUNT** — saturating `refcount_t` on `amba_device` via `get_device`/`put_device`; per-driver `kref` saturates so module-unload race cannot underflow.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `amba_device` (in `amba_device_release`), per-driver match scratch, and Tegra AHB context save buffers so stale periphid / pclk / driver_override pointers cannot bleed across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on sysfs entry (`driver_override_store`); per-sysfs `count` bounded by `driver_set_override`.
- **PAX_RAP / kCFI** — `amba_bustype` ops (`match`, `uevent`, `probe`, `remove`, `shutdown`, `dma_configure`, `dma_cleanup`), `amba_pm` ops, `amba_dev_groups`, and per-driver `id_table` pointers marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms exposure of AMBA + per-PrimeCell driver symbols behind CAP_SYSLOG; suppress `%p` of `amba_device` + ioremap-base in `dev_dbg` / `dev_err`.
- **GRKERNSEC_DMESG** — restrict periphid-read failures, PROBE_DEFER spam, and Tegra AHB / SMMU init banners to CAP_SYSLOG so attackers cannot enumerate present PrimeCells via dmesg.
- **PID/CID matching kCFI** — `amba_lookup` + `amba_cs_uci_id_match` invocation paths typed and CFI-checked; defeats vtable-style indirect-call hijack.
- **`periphid` range bound** — per-PID-mask match strictly `(periphid & mask) == id`; refuse table entries with `mask == 0` (would universal-match); refuse periphid == 0 (sentinel for unread).
- **`amba_device` PAX_REFCOUNT** — `dev` embedded `kref`/`refcount_t` saturated; `amba_device_release` only runs on true zero, never on overflow.
- **`driver_override` bounded** — string copy capped + zeroed-on-replace; world-write blocked via 0644 (root-only) sysfs perms enforced by `DEVICE_ATTR_RW`.
- **Reset-control + PM-domain unwind audit** — every error path in `read_periphid` matches a put/detach; no leaked references on EPROBE_DEFER.
- **Tegra AHB SMMU handshake** — refuse to clear `SMMU_INIT_DONE` once set; defense against hot-attack that would briefly drop master DMA isolation.

Rationale: AMBA is the routing fabric for bus-mastering PrimeCell peripherals on every ARM SoC — a peripheral on this bus can DMA anywhere the system memory allows. A relaxed `amba_match` that hard-errored (instead of `-EPROBE_DEFER`-ed) would silently lose drivers; a `driver_override` writable by non-root would let any user re-bind devices to VFIO; an unsanitized PID lookup with `mask == 0` would universally bind. RAP/kCFI on the bus-ops vtable, refcount-trapping on `amba_device`, MEMORY_SANITIZE on override strings, and CAP-gated sysfs writes turn AMBA from "early-boot autoprobe primitive" into a structurally constrained bus interface.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-PrimeCell drivers (PL011 UART, PL022 SPI, PL18x MMCI, PL08x DMA, PL050 KMI) — each gets its own Tier-3 under appropriate subsystem.
- CoreSight subsystem internals (covered in `drivers/hwtracing/coresight/`).
- Tegra SMMU driver (covered in `drivers/iommu/tegra-smmu.c` future Tier-3).
- ACPI AMBA enumeration internals.
- 32-bit-only DT-specific paths beyond what `bus.c` defines.
- Implementation code
