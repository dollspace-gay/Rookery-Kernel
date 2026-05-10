---
title: "Tier-2: drivers/spi — SPI bus subsystem (core + per-controller LLDs + spidev + memops + slave)"
tags: ["tier-2", "drivers", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 wrapper for the SPI (Serial Peripheral Interface) bus subsystem — used by every SPI flash chip (M25Pxx / W25Q / SST / Macronix) for boot ROM and BMC/IPMI firmware, every SPI-NAND, every SPI-attached EEPROM, every SPI touchscreen, every SPI sensor (Bosch BMI / BMP / BME-series IMU), every SPI ADC/DAC, every SPI radio (CC-series, NRF24, Si446x), every SPI display panel, every SPI-attached secure-element / TPM. Components: **SPI core** (`spi.c` + `internals.h` — `spi_master` / `spi_controller` / `spi_device` / `spi_driver` / `spi_message` / `spi_transfer` lifecycle, ACPI / OF / fwnode enumeration, async transfer support), **spidev** (`spidev.c` — `/dev/spidev<bus>.<cs>` userspace chardev), **memops** (`spi-mem.c` + per-controller mem-ops integration — used by SPI-NOR/NAND), **per-controller LLDs** (`spi-{8250-pci, designware, intel, intel-pci, intel-platform, atmel, mxs, sun4i, sun6i, ...}.c` — ~150 controller drivers; v0 maintenance set: spi-amd, spi-amd-pci, spi-cadence, spi-designware-pci, spi-intel, spi-intel-pci, spi-intel-platform, spi-pxa2xx, spi-sc18is602 (USB-SPI bridge), spi-loopback-test for testing).

### Acceptance Criteria

- [ ] AC-O1: `spi_pipe` test on `/dev/spidev0.0` round-trips loopback bytes.
- [ ] AC-O2: SPI-NOR enumerates as `/dev/mtd<N>` via spi-mem path on a reference board.
- [ ] AC-O3: kselftest `tools/testing/selftests/drivers/spi/` passes.

### Out of Scope

- ARM-only SoC controllers (most of `spi-{atmel, ...}.c` compile-gated off for v0)
- Implementation code
- 32-bit-only paths

### compatibility contract — outline

- `/dev/spidev<bus>.<cs>` chardev IOCTLs byte-identical: `SPI_IOC_RD_MODE`, `_WR_MODE`, `_RD_MODE32`, `_WR_MODE32`, `_RD_LSB_FIRST`, `_WR_LSB_FIRST`, `_RD_BITS_PER_WORD`, `_WR_BITS_PER_WORD`, `_RD_MAX_SPEED_HZ`, `_WR_MAX_SPEED_HZ`, `_MESSAGE(N)` (with `struct spi_ioc_transfer[N]`).
- Per-controller `/sys/class/spi_master/spi<N>/` + per-device `/sys/bus/spi/devices/spi<bus>.<cs>/{modalias,driver,subsystem,statistics/}` byte-identical.
- `MODULE_DEVICE_TABLE(spi, …)` modalias `spi:<name>` byte-identical (ACPI: `acpi:<HID>`; OF: `of:NaTalcCacc`).
- ACPI + DT + fwnode + swnode + spi_board_info (legacy) enumeration paths preserved.
- spi-mem operations identical for SPI-NOR (`drivers/mtd/spi-nor/`) consumer.

### tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `drivers/spi/core.md` | `spi.c` + `internals.h`: bus core + master/device/driver lifecycle + ACPI/OF enumeration |
| `drivers/spi/spidev.md` | `spidev.c`: `/dev/spidev<bus>.<cs>` chardev |
| `drivers/spi/spi-mem.md` | `spi-mem.c`: SPI memory operations (consumed by SPI-NOR/NAND) |
| `drivers/spi/controllers-x86.md` | `spi-{amd,amd-pci,intel,intel-pci,intel-platform,pxa2xx,designware-pci}.c`: x86-relevant controllers |
| `drivers/spi/controllers-misc.md` | `spi-{cadence,sc18is602,loopback-test}.c`: misc + USB-SPI bridges + test |

### compatibility outline

- REQ-O1: `/dev/spidev*` IOCTLs byte-identical (libgpiod / spi-tools / fwts consume unchanged).
- REQ-O2: Per-class sysfs surface byte-identical.
- REQ-O3: Modalias formats byte-identical.
- REQ-O4: spi-mem op semantics identical.
- REQ-O5: Master/device/driver registration source-compat.
- REQ-O6: TLA+ models (per-controller transfer queue, async-transfer completion ordering, chip-select toggle race).
- REQ-O7: Hardening: `/dev/spidev*` CAP_SYS_RAWIO + LSM mediation; SPI-flash write paths require additional cap.

### verification

| TLA+ Model | Owner |
|---|---|
| `models/spi/transfer_queue.tla` | `drivers/spi/core.md` (proves: per-controller transfer-queue + async completion ordering; concurrent submit + completion-callback never produce out-of-order callbacks for in-order transfers) |
| `models/spi/cs_toggle.tla` | `drivers/spi/core.md` (proves: chip-select toggle between transfers in a message respects per-message `cs_change` flag; back-to-back transfers to same CS never glitch CS line) |

### hardening

| Feature | Default |
|---|---|
| **REFCOUNT** | per-controller + per-device refcounts use `Refcount` | § Mandatory |
| **CONSTIFY** | per-controller `spi_controller_mem_ops`, per-driver `spi_driver` `static const` | § Mandatory |
| **SIZE_OVERFLOW** | per-transfer len arithmetic checked | § Mandatory |
| **MEMORY_SANITIZE** | freed transfer buffers cleared (carry SPI-flash secrets, TPM session data) | § Default-on configurable |

SPI-specific reinforcement: `/dev/spidev*` open requires CAP_SYS_RAWIO (defends against userspace flashing SPI-NOR boot ROM); SPI-attached TPM access mediated by TPM-LSM hooks (cross-ref `security/keys/`); per-controller transfer-rate cap default-on.

