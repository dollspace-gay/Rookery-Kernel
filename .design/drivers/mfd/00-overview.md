# Tier-2: drivers/mfd — Multi-Function Device framework (PMIC + system-controllers + per-IC core drivers)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/mfd/
  - include/linux/mfd/
-->

## Summary

Tier-2 wrapper for MFD — Multi-Function Device framework for ICs that bundle multiple functions (PMIC = LDOs + regulators + RTC + battery-charger + ADC + GPIO + thermal + audio-codec on one chip). The MFD core (`mfd-core.c`) instantiates child platform_devices for each function so each function can bind a separate per-class driver (regulator-driver for the LDO, rtc-driver for the RTC, etc.). Components: `mfd-core.c` (framework — `mfd_add_devices` + `mfd_cell` registration), per-IC drivers (~250, mostly per-PMIC; v0 set: `intel_pmc_core.c` (Intel PMC), `intel_soc_pmic_*.c` (Intel SoC PMICs: Crystal Cove, Whiskey Cove, BXT, CHT), `cs5535-mfd.c` (AMD CS5535 SuperIO), `lpc_ich.c` (Intel ICH/PCH LPC), `lpc_sch.c` (Intel SCH), `da9*.c` (Dialog), `max7*.c` + `max8*.c` (Maxim), `mt6*.c` (MediaTek), `palmas.c` (TI), `pcf50633-core.c`, `pm8606.c`, `qcom_*.c` (Qualcomm), `rk808.c` + `rk8xx-spi.c` (Rockchip — out of v0 ARM), `tps6*.c` + `tps8*.c` (TI), `wm831x-core.c` + `wm8350-core.c` + `wm8400-core.c` + `wm8994-core.c` + `wm8997-core.c` + `wm8998-core.c` + `wm5102-core.c` + `wm5110-core.c` + `wm9081-core.c` + `wm9090-core.c` + `wm9712-core.c` + `wm9713-core.c` (Wolfson)).

## Compatibility contract — outline

- Parent IC binding via i2c_driver / spi_driver / pci_driver matches identically; child function platform_devices appear under parent in `/sys/devices/...`.
- Per-MFD-cell DT/ACPI binding parsed identically.

## Tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `drivers/mfd/core.md` | `mfd-core.c` |
| `drivers/mfd/intel-pmc.md` | `intel_pmc_core.c` + `intel_soc_pmic_*.c` |
| `drivers/mfd/intel-lpc.md` | `lpc_ich.c` + `lpc_sch.c` |
| `drivers/mfd/per-vendor-pmic.md` | per-vendor PMICs (Maxim, TI, Dialog, MediaTek, Qualcomm, Wolfson) |

## Compatibility outline / AC / Verification / Hardening

- REQ-O1: Parent + child binding source-compat.
- REQ-O2: Per-PMIC cell registration produces identical sysfs hierarchy.
- REQ-O3: TLA+ models (per-IC concurrent child registration + parent unbind safety).
- REQ-O4: AC: laptop reference w/ Intel SoC PMIC enumerates regulator + GPIO + thermal child devices correctly.
- Hardening: row-1; PMIC voltage-write paths cross-ref `drivers/regulator/`; per-LSM hook on PMIC reg-write.

## Out of Scope
Implementation code; ARM SoC PMICs; 32-bit-only paths.
