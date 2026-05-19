# Tier-3: drivers/soc/* — SoC bus, soc-info, vendor SoC service drivers

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/00-overview.md
upstream-paths:
  - drivers/base/soc.c
  - include/linux/sys_soc.h
  - drivers/soc/Kconfig
  - drivers/soc/Makefile
  - drivers/soc/amlogic
  - drivers/soc/apple
  - drivers/soc/aspeed
  - drivers/soc/atmel
  - drivers/soc/bcm
  - drivers/soc/fsl
  - drivers/soc/hisilicon
  - drivers/soc/imx
  - drivers/soc/mediatek
  - drivers/soc/microchip
  - drivers/soc/qcom
  - drivers/soc/renesas
  - drivers/soc/rockchip
  - drivers/soc/samsung
  - drivers/soc/sophgo
  - drivers/soc/sunxi
  - drivers/soc/tegra
  - drivers/soc/ti
  - drivers/soc/xilinx
-->

## Summary

`drivers/soc/` aggregates vendor-supplied service drivers that present per-SoC infrastructure too platform-specific for any single subsystem to own: SoC-info publishing (machine / family / revision / soc-id / serial through `/sys/bus/soc/devices/socN/`), power-management ICs and PMU companions, system-controller bridges (SCMI / SCPI fallbacks), bus arbitration / NoC fabric programming, IPC mailboxes (Apple RTKit, TI WkupM3, Qualcomm RPMh), DMA-fabric controllers, reset / clock gating helpers that don't fit the generic `drivers/reset`, `drivers/clk` shape, and per-SoC `dtpm` (Dynamic Thermal Power Management) actors. The `drivers/base/soc.c` core registers the `soc` bus type, exposes the canonical `struct soc_device_attribute` sysfs surface, and provides `soc_device_match` — a glob-matched runtime lookup used by per-driver SoC quirk tables.

This Tier-3 covers the shared core (`drivers/base/soc.c`, ~280 lines: bus + ida + sysfs + match) plus the architectural shape of the per-vendor subdirectories. Per-vendor Tier-3 docs (one per family) descend from here.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct soc_device_attribute` | per-SoC identity (machine, family, revision, soc_id, serial_number, custom_attr_group, data) | `drivers::soc::SocDeviceAttribute` |
| `struct soc_device` | runtime per-SoC instance (private to core) | `drivers::soc::SocDevice` |
| `struct bus_type soc_bus_type` | "soc" bus root | `drivers::soc::SocBus` |
| `soc_device_register(attr)` / `soc_device_unregister(dev)` | per-SoC publish/unpublish | `SocDevice::register` / `_unregister` |
| `soc_device_to_device(soc_dev)` | extract `struct device *` for parent assignment | `SocDevice::to_device` |
| `soc_device_match(matches)` | glob-match over machine/family/revision/soc_id; return first match (with `.data`) | `SocDevice::match` |
| `soc_attr_read_machine(attr)` | DT-derived machine name fallback (`of_machine_read_model`) | `SocDeviceAttribute::read_machine` |
| `early_soc_dev_attr` | early-registration latch for callers ahead of `core_initcall` | `SocBus::early_attr` |
| `soc_ida` | per-instance numeric id allocator (`socN`) | `SocBus::ida` |
| per-vendor `*_soc_init()` | vendor entrypoint (e.g. `imx8m_soc_init`, `tegra_init_revision_from_chipid`, `qcom_socinfo_probe`, `k3_socinfo_probe`) | `drivers::soc::<vendor>::Init` |

## Compatibility contract

REQ-1: `soc` bus registered at `core_initcall` priority; any caller invoking `soc_device_register` before that latches one early attribute in `early_soc_dev_attr` (only one such caller is supported).

REQ-2: per-SoC sysfs node at `/sys/bus/soc/devices/socN/` with attributes `machine`, `family`, `revision`, `serial_number`, `soc_id`; each attribute visible only if the corresponding `soc_device_attribute` field is non-NULL.

REQ-3: per-SoC custom attribute group (`soc_dev_attr->custom_attr_group`) optionally exposes vendor-specific surface (e.g. fuse-derived part number, lot id, secure-boot fuses, ECID); attached as second `attribute_group` on the device.

REQ-4: per-SoC `data` field is opaque caller-defined and used by `soc_device_match` consumers to carry quirk callbacks or quirk-flag bitmaps; consumers cast at known type.

REQ-5: `soc_device_match` is glob-matched (`glob_match`) per field — all non-NULL fields in the match entry must match the registered SoC; first match wins; returns NULL if none.

REQ-6: per-vendor probe paths conventionally read SoC identity from: DT `compatible` + `socN` reserved nodes (Allwinner / Renesas / Rockchip / iMX), nvmem fuses (TI K3, Tegra, Apple), system-controller mailbox (Qualcomm SMEM, NXP SCMI), or ROM tables (Aspeed, Microchip).

REQ-7: per-vendor service drivers register additional auxiliary buses or platform devices that consume the soc-device match result; e.g. `dtpm` per-SoC actors on Rockchip / Mediatek, RTKit on Apple, KNAV QMSS on TI Keystone.

REQ-8: per-SoC numeric id allocated from `soc_ida` and reclaimed on unregister; device name `socN`.

REQ-9: `dev_uevent` carries `OF_MODALIAS` / `OF_COMPATIBLE_*` if DT-described; consumers can match via `MODULE_DEVICE_TABLE(of, ...)` on the parent platform-device level rather than the synthetic soc device.

REQ-10: Per-vendor sub-driver `probe` must not panic on missing optional fields (e.g. when `revision` cannot be read from fuses); `soc_attr_read_machine` falls back to DT `model` property.

## Acceptance Criteria

- [ ] AC-1: `ls /sys/bus/soc/devices/` on a Tegra X1 / RK3399 / iMX8MP / Apple M1 board lists exactly one `socN` per SoC die.
- [ ] AC-2: `cat /sys/bus/soc/devices/soc0/{family,machine,revision,soc_id}` matches the documented per-vendor format on the listed boards.
- [ ] AC-3: vendor quirk table (e.g. `tegra_pcie_compatible` `soc_device_match` table) matches correctly on the expected board and not on others.
- [ ] AC-4: `rmmod` of a soc-info driver (where modular, e.g. `soc-tegra-fuse` test variant) unregisters the soc device cleanly; re-insmod re-registers with same numeric id (post-IDA reclaim).
- [ ] AC-5: Apple SoC: `drivers/soc/apple/rtkit` mailbox round-trip with co-processor firmware succeeds; crashlog parse exposes panic origin.
- [ ] AC-6: Qualcomm SoC: `qcom_socinfo` reports the expected `chip_id`, `chip_name`, `pmic_*` fields on a Snapdragon reference platform.
- [ ] AC-7: TI K3 / Keystone: `k3-socinfo` publishes JTAG-id + secure/non-secure variant; consumers (`ti_sci`, sysc reset) bind successfully.
- [ ] AC-8: Microchip / Atmel: SAMA5/SAMA7 family `soc-atmel` reports correct revision per CIDR/EXID register.
- [ ] AC-9: Custom attribute group: at least one vendor (Rockchip / Mediatek / Samsung) exposes a vendor-specific attribute and consumers see it under `/sys/bus/soc/devices/socN/`.

## Architecture

`SocBus` lives in `drivers::soc`:

```
struct SocBus {
  bus_type: BusType,               // "soc", core_initcall registered
  ida: Mutex<IdAllocator>,         // soc_ida
  early_attr: Mutex<Option<&'static SocDeviceAttribute>>,
}

struct SocDeviceAttribute {
  machine: Option<&'static str>,
  family: Option<&'static str>,
  revision: Option<&'static str>,
  serial_number: Option<&'static str>,
  soc_id: Option<&'static str>,
  custom_attr_group: Option<&'static AttributeGroup>,
  data: Option<NonNull<u8>>,       // opaque, consumer-typed
}

struct SocDevice {
  dev: Device,                     // bus = SocBus, name = "socN"
  attr: &'static SocDeviceAttribute,
  soc_dev_num: i32,
}
```

Registration (`soc_device_register`):
1. `soc_attr_read_machine(attr)` — if `attr->machine` NULL and a DT `model` is available, fill it via `of_machine_read_model` (kstrdup, owned by attr from this point).
2. If `soc_bus_registered == false`: latch the single attr in `early_soc_dev_attr` and return NULL — vendor caller is expected to retry-or-trust-core after core_initcall.
3. Else: allocate `soc_device`, allocate a 3-slot `attribute_group` array (`[&soc_attr_group, attr->custom_attr_group, NULL]`), allocate IDA slot, set bus + groups + release callback, `device_register`.

Unregistration (`soc_device_unregister`):
1. `device_unregister(&soc_dev->dev)`; release callback frees the attribute-group array + IDA slot.
2. Clear `early_soc_dev_attr` if it points to this attr.

soc_device_match (consumer side):
1. For each entry in the user-supplied zero-terminated `matches[]` array:
   - `bus_for_each_dev(soc_bus_type, NULL, matches, soc_device_match_one)` — walk all registered SocDevices.
   - Per device: glob-match every non-NULL match field; all-match → return the matching entry pointer (so consumers can read `entry->data`).
   - If no live SocDevice yet and `early_soc_dev_attr` set, try matching against that latch.

Vendor subdir conventions:
- `drivers/soc/<vendor>/Kconfig` + `Makefile` gates per-board build via `SOC_<VENDOR>_<SOC>` / `ARCH_<vendor>` / `MACH_<vendor>` symbols.
- A single `<vendor>-soc.c` (or `soc-<vendor>.c`) probes the soc-info platform device DT-described as `<vendor>,<chip>-soc-info`, reads identity from SoC-specific mechanism (fuses / scu / smem / mbox), and calls `soc_device_register`.
- Companion drivers in the same subdir provide IP-block helpers: PMU power-domain (e.g. `drivers/soc/dove/pmu.c`, `imx93-src.c`), reset (`renesas/rcar-rst.c`), mailbox / IPC (`apple/mailbox.c`, `ti/wkup_m3_ipc.c`), QoS / NoC (`tegra/fuse`, `qcom/icc-bwmon`), and security-coprocessor bridges (`fsl/qe`, `qcom/qcom_aoss`).
- Apple RTKit (`drivers/soc/apple/rtkit*.c`) is the most complex example: mailbox + work-queue + crashlog + buffer-allocator for Apple Silicon co-processors (SMC, NVMe controller, M1 RTKit endpoints).
- TI Keystone QMSS (`drivers/soc/ti/knav_qmss_queue.c`) — DMA queue manager, ~3K lines.
- Samsung Exynos `chipid` + ACP / PMU bridge.
- Tegra fuse + powergate + PMC bus drivers (cross-ref `drivers/soc/tegra/`).
- Mediatek `mtk-pmic-wrap` + `mtk-svs` (sense-and-veto).

## Hardening

(Inherits row-1 features from `drivers/00-overview.md`.)

drivers/soc specific:

- **per-vendor probe must validate fuse + SCU + SMEM reads** — every per-vendor identity source is hardware-readback and may return zero / all-ones on broken silicon; per-driver probe rejects implausible identity rather than registering `soc-id=ffff`.
- **glob-match input audited** — `soc_device_match` arrays are static const; bad glob patterns (e.g. `*` in `serial_number`) caught at audit time.
- **early-attr latch is single-shot** — concurrent early registration returns `-EBUSY` rather than overwriting; defeats race between two pre-`core_initcall` callers.
- **IDA reclaim under release** — `ida_free` happens in `soc_release`, guaranteeing slot is reclaimed only after the device's reference count drops; safe under racing match traversal.
- **custom_attr_group treated as RO surface by default** — vendor-specific writeable attributes require explicit CAP_SYS_ADMIN guards in their `store` handlers; defeats per-fuse re-write attempts via sysfs from unprivileged users.
- **per-vendor mbox / SMEM region clamped** — every per-SoC IPC bridge bounds incoming firmware-supplied region lengths against a per-driver max; defeats co-processor-supplied length-confusion exploits.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `soc_device`, vendor `socinfo` private data, and per-driver mbox/IPC payload bounce buffers.
- **PAX_KERNEXEC** — soc-bus + per-vendor probe text W^X; `soc_attr` arrays, `soc_attribute_mode`, and `soc_info_show` in `__ro_after_init` text.
- **PAX_RANDKSTACK** — randomize kstack across `soc_device_register`, `soc_device_match`, and per-vendor `*_socinfo_probe` entries.
- **PAX_REFCOUNT** — saturating `refcount_t` on the underlying `struct device` of every SocDevice and on every per-vendor IPC mailbox endpoint (Apple RTKit, TI WkupM3, Qualcomm RPMh).
- **PAX_MEMORY_SANITIZE** — zero-on-free for `soc_attr_groups` arrays, `soc_device` slabs, and per-vendor fuse-readback scratch buffers (lot-id, ECID); secure attributes never bleed across reuse.
- **PAX_UDEREF** — SMAP/PAN across every vendor-specific sysfs `store` callback that touches firmware registers; user pointer never deref'd outside `kstrtox` helpers.
- **PAX_RAP / kCFI** — `bus_type` ops, `soc_release`, `soc_info_show`, `attribute_group::is_visible`, and per-vendor mailbox-ops vtables marked `__ro_after_init` with kCFI dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms + per-SoC fuse-derived serial / ECID / JTAG-id disclosure behind CAP_SYSLOG; `/sys/bus/soc/devices/socN/serial_number` is informational, not for unprivileged readers when policy demands.
- **GRKERNSEC_DMESG** — restrict per-vendor probe banners (chip-revision, fuse-status, secure-boot-state) to CAP_SYSLOG; attackers cannot fingerprint the exact SoC silicon rev via dmesg.
- **soc_device PAX_REFCOUNT pairing** — underlying `device::kobj.kref` is `refcount_t`-overflow-trapped; defeats `device_register` + concurrent `_unregister` race UAFs.
- **soc-attr sysfs RO enforcement** — `soc_attr` group is mode `0444`; any vendor `custom_attr_group` exposing a writeable attribute is required (by audit policy) to gate on CAP_SYS_ADMIN.
- **Vendor-specific MFD gated** — per-SoC `mfd` companion drivers (Samsung S2MPS, Qualcomm SPMI-PMIC, Mediatek MT6359) require explicit CAP_SYS_RAWIO for the underlying register-poke debug paths.
- **DT-property bounds** — every `of_property_read_*` in soc subdirs uses `read_u32` / `read_string` rather than raw `of_get_property` with caller-side length math; defeats DT-overlay-supplied length overflow.
- **Fuse readback validated** — implausible (all-zero / all-ones) chip-id / revision rejected at probe; defeats lock-step-mistake silicon being misidentified as a richer-feature variant.
- **Early-attr single-shot guard** — `early_soc_dev_attr` set exactly once before `core_initcall`; second pre-bus registration returns `-EBUSY` rather than silently clobbering.

Rationale: SoC service drivers are the kernel's authoritative source of "what platform am I on" for downstream quirk dispatch. A spoofed `soc_id` or stale `revision` propagates into per-IP quirk tables (PCIe RC, USB PHY, CPU PM) and can re-enable known-broken-silicon workarounds (or disable needed ones) — both of which trade off against security. RAP/kCFI on bus + match, CAP_SYS_ADMIN on writeable custom attributes, fuse-readback validation, and refcount-overflow trapping on per-SoC kobjects keep the SoC bus from becoming an attacker-influenced trust signal.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-vendor SoC families (future Tier-3 docs: `apple-rtkit.md`, `qcom-socinfo.md`, `tegra-fuse.md`, `imx-soc.md`, `rockchip-grf.md`, `ti-k3-socinfo.md`, `mediatek-pmic-wrap.md`)
- Architectural SoC-PM frameworks (covered in `drivers/base/power/` + `kernel/power/` docs)
- SCMI / SCPI standard transports (covered in `drivers/firmware/arm_scmi.md` future Tier-3)
- DT bindings (covered in `Documentation/devicetree/bindings/soc/`)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `ida_slot_no_collision` | UNIQUENESS | `soc_ida` slot allocated under register, freed under release; never re-issued while live |
| `early_attr_single_shot` | UNIQUENESS | `early_soc_dev_attr` set at most once before bus register |
| `attr_groups_no_oob` | OOB | per-device `soc_attr_groups` is a 3-slot NULL-terminated array; never indexed > 2 |
| `release_balanced` | LEAK | every `soc_device_register` success path matched by exactly one `soc_device_unregister` or device-tree teardown |

### Layer 2: TLA+

`models/soc/soc_bus.tla`: proves early-registration latch + concurrent core_initcall + per-vendor register cannot interleave to double-register the same numeric id.

### Layer 3: Verus invariants

- `soc_device_match` post: returned pointer is either NULL or a member of the user-supplied `matches[]` array (never internally-allocated).
- `soc_device_register` post: on success, exactly one new bus device with name `socN` for the freshly-allocated `N`.

### Layer 4: Functional

Per-vendor boot tests on QEMU virt + reference HW (Tegra TX1, RK3399, iMX8MP, Apple M1, SAMA5D2, BCM2711, Snapdragon 845) verify sysfs surface + match-table consumers.
