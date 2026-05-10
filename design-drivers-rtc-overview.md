---
title: "Tier-2: drivers/rtc — Real Time Clock subsystem (rtc class + per-RTC drivers + alarm + IRQ)"
tags: ["tier-2", "drivers", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 wrapper for RTC — every chassis CMOS RTC, every I2C/SPI battery-backed RTC, every per-PMIC RTC. Components: **core** (`class.c` + `interface.c` + `lib.c` + `lib_test.c` + `nvmem.c` + `proc.c` + `sysfs.c` + `dev.c`: framework — `rtc_class_open` + `rtc_read_time` / `_set_time` / `_set_alarm` / `_irq_enable`; per-RTC NVRAM via NVMEM bridge; chardev `/dev/rtc<N>`), **per-driver drivers** (~140; v0 set: `rtc-cmos.c` (legacy x86 CMOS / mc146818), `rtc-acpi.c` (ACPI RTC), `rtc-efi.c` (EFI Runtime Services RTC), `rtc-ds1307.c` + `rtc-ds3232.c` (Maxim/Dallas I2C), `rtc-pcf8523.c` + `rtc-pcf8563.c` + `rtc-pcf85063.c` (NXP I2C), `rtc-rx8025.c` + `rtc-rx8581.c` (Epson I2C), `rtc-mc146818-lib.c`, `rtc-rv8803.c`, `rtc-m41t80.c`, plus PMIC-integrated RTCs from per-PMIC drivers).

### compatibility contract — outline

- `/dev/rtc<N>` chardev IOCTLs byte-identical: `RTC_AIE_ON`, `_AIE_OFF`, `_UIE_ON`, `_UIE_OFF`, `_PIE_ON`, `_PIE_OFF`, `_WIE_ON`, `_WIE_OFF`, `_ALM_SET`, `_ALM_READ`, `_RD_TIME`, `_SET_TIME`, `_IRQP_READ`, `_IRQP_SET`, `_EPOCH_READ`, `_EPOCH_SET`, `_WKALM_SET`, `_WKALM_RD`, `_VL_READ`, `_VL_CLR`, `_PARAM_GET`, `_PARAM_SET`. Wire format byte-identical (hwclock + rtcwake consume unchanged).
- `/sys/class/rtc/rtc<N>/{name, date, time, since_epoch, hctosys, max_user_freq, wakealarm, offset, ...}` byte-identical.
- `/proc/driver/rtc` (when `/dev/rtc0` corresponds to legacy CMOS RTC) byte-identical.
- DT + ACPI bindings parsed identically.

### tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `drivers/rtc/class.md` | `class.c` + `interface.c` + `lib.c`: framework |
| `drivers/rtc/dev-proc-sysfs.md` | `dev.c` + `proc.c` + `sysfs.c`: chardev + procfs + sysfs |
| `drivers/rtc/nvmem.md` | `nvmem.c`: per-RTC NVRAM via NVMEM bridge |
| `drivers/rtc/cmos.md` | `rtc-cmos.c` + `rtc-mc146818-lib.c`: legacy x86 CMOS RTC |
| `drivers/rtc/acpi-efi.md` | `rtc-acpi.c` + `rtc-efi.c`: ACPI + EFI Runtime Services RTC |
| `drivers/rtc/i2c-rtcs.md` | `rtc-ds*.c` + `rtc-pcf*.c` + `rtc-rx*.c` + `rtc-rv*.c`: I2C RTCs |

### compatibility outline / ac / verification / hardening

- REQ-O1: `/dev/rtc*` IOCTLs + `/sys/class/rtc/...` byte-identical.
- REQ-O2: Wakeup-from-suspend via RTC alarm works.
- REQ-O3: TLA+ models (per-RTC alarm-IRQ + timer-tick + UIE serialization).
- REQ-O4: AC: `hwclock --systohc` + `--hctosys` round-trip; `rtcwake -m mem -s 60` suspends + resumes after 60s.
- Hardening: row-1; `/dev/rtc*` set-time requires CAP_SYS_TIME; per-RTC LSM mediation.

