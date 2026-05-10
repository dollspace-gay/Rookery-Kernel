---
title: "Tier-2: drivers/nvmem — Non-Volatile Memory framework (EEPROM + OTP + EFUSE + per-controller drivers)"
tags: ["tier-2", "drivers", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 wrapper for NVMEM — uniform abstraction over per-IC NVRAM/EEPROM/OTP/eFUSE storage (board MAC addresses, SoC fuse bits, per-chip calibration, vendor cell data). Components: `core.c` (framework — `nvmem_register` + per-cell binding via DT/swnode), `layouts/` (per-product cell-layout descriptors), per-driver: `bcm-ocotp.c`, `imx-ocotp.c` (out of v0), `lpc18xx_eeprom.c`, `mtk-efuse.c`, `mxs-ocotp.c`, `nintendo-otp.c`, `qcom-spmi-sdam.c`, `qfprom.c`, `rave-sp-eeprom.c`, `rmem.c`, `rockchip-efuse.c`, `rockchip-otp.c`, `sc27xx-efuse.c`, `sec-qfprom.c`, `snvs_lpgpr.c`, `sprd-efuse.c`, `stm32-romem.c`, `sunplus-ocotp.c`, `sunxi_sid.c`, `u-boot-env.c`, `uniphier-efuse.c`, `vf610-ocotp.c`, `zynqmp_nvmem.c`. v0 set: rmem (DT-described reserved-memory), u-boot-env (U-Boot environment via MTD); rest mostly ARM SoC (compile-gated off).

### compatibility contract — outline

- `/sys/bus/nvmem/devices/<name>/` chardev for raw read/write (when `root_only=0`).
- Per-cell consumer API: `nvmem_cell_get_*` source-compat for in-tree drivers (Ethernet drivers reading MAC from EEPROM, etc.).

### tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `drivers/nvmem/core.md` | `core.c` + `layouts/` |
| `drivers/nvmem/u-boot-env.md` | `u-boot-env.c`: U-Boot environment via MTD |
| `drivers/nvmem/rmem.md` | `rmem.c`: DT reserved-memory NVMEM |

### compatibility outline / ac / verification / hardening

- REQ-O1: nvmem consumer API source-compat.
- REQ-O2: Per-cell DT/swnode binding parsed identically.
- REQ-O3: TLA+ models (per-NVMEM read/write serialization).
- REQ-O4: AC: in-tree NIC reads MAC from EEPROM-backed nvmem cell.
- Hardening: row-1; raw chardev write requires CAP_SYS_RAWIO + LSM mediation (defends against re-flashing OTP/eFUSE which is one-time-write).

