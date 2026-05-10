---
title: "Tier-2: drivers/mmc — MMC/SD/SDIO subsystem (core + host + per-controller LLDs + block + sdio function drivers)"
tags: ["tier-2", "drivers", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 wrapper for MMC/SD/SDIO — every laptop SD card slot, every embedded eMMC chip, every SDIO Wi-Fi/Bluetooth combo card, every microSD adapter. Components: **core** (`core/`: `core.c`, `bus.c`, `host.c`, `mmc.c`, `sd.c`, `sdio.c`, `sdio_io.c`, `sdio_cis.c`, `mmc_ops.c`, `sd_ops.c`, `sdio_ops.c`, `block.c`, `block.h`, `crypto.c`, `debugfs.c`, `mmc_test.c`, `pwrseq.c`, `pwrseq_*.c`, `quirks.h`, `regulator.c`, `slot-gpio.c` — bus core + per-card type [MMC, SD, SDIO] + ops + block-layer integration), **host** (`host/`: per-controller LLDs ~80, v0 set: `sdhci.c` (Standard SDHCI core) + `sdhci-pci-*.c` (Intel PCIe SD), `sdhci-acpi.c` (ACPI-enumerated SDHCI), `sdhci-msm.c`, `sdhci-omap.c` (out of v0), `sdhci-pltfm.c`, `sdhci-amd.c`).

### compatibility contract — outline

- `/dev/mmcblk<N>{,p<P>}` block device + `/dev/mmcblk<N>boot<0,1>` + `/dev/mmcblk<N>rpmb` (replay-protected memory block — secure storage) byte-identical.
- `/sys/class/mmc_host/mmc<N>/{max_freq, current_freq, ios/, ...}` byte-identical.
- `/sys/block/mmcblk<N>/device/{cid, csd, scr, fwrev, hwrev, manfid, oemid, name, type, ...}` byte-identical (mmc-utils consumes).
- `MMC_IOC_CMD` / `MMC_IOC_MULTI_CMD` IOCTLs byte-identical (mmc-utils consumes for fw-update, RPMB ops).

### tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `drivers/mmc/core.md` | `core/core.c` + `bus.c` + `host.c` + `mmc.c` + `sd.c` + `sdio.c`: card type + ops |
| `drivers/mmc/block.md` | `core/block.c`: block-layer integration |
| `drivers/mmc/sdio-func.md` | `core/sdio_io.c` + `sdio_cis.c` + `sdio_ops.c`: SDIO function driver framework |
| `drivers/mmc/crypto.md` | `core/crypto.c`: Inline Crypto Engine |
| `drivers/mmc/host-sdhci.md` | `host/sdhci*.c`: SDHCI standard host + ACPI/PCI/platform variants |

### compatibility outline / ac / verification / hardening

- REQ-O1: Block device naming + ioctls byte-identical (mmc-utils consumes).
- REQ-O2: SDIO function driver framework source-compat (Wi-Fi SDIO drivers like brcmfmac SDIO consume).
- REQ-O3: TLA+ models (per-card command + data state machine; SDIO interrupt + concurrent function-call race; tuning state machine for HS200/HS400).
- REQ-O4: AC: SD card boot + read+write; mmc-utils `mmc info /dev/mmcblk0` shows correct CID/CSD; eMMC RPMB write+read works on supported HW.
- Hardening: row-1; RPMB ops require CAP_SYS_RAWIO; ICE keys keyring-shielded.

