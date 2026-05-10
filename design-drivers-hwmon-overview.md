---
title: "Tier-2: drivers/hwmon — hardware monitoring (chip temperatures + voltages + fans + currents + powers)"
tags: ["tier-2", "drivers", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 wrapper for hwmon — every chip-level temperature/voltage/fan/current/power sensor exposed to userspace `lm-sensors`/`sensors` tool. Components: **core** (`hwmon.c` + `hwmon-vid.c`: framework — `hwmon_device_register_with_info` + per-channel sysfs), **per-chip drivers** (~250, v0 set: `coretemp.c` (Intel CPU per-core temp), `k10temp.c` + `k8temp.c` (AMD CPU temp), `nct6683.c` + `nct6775*.c` + `nct7361.c` (Nuvoton SuperIO), `it87.c` (ITE SuperIO), `f71805f.c` + `f71882fg.c` + `f75375s.c` (Fintek SuperIO), `w83627hf.c` + `w83627ehf.c` + `w83627ehf.c` + `w83792d.c` + `w83793.c` + `w83795.c` + `w83l786ng.c` + `w83l785ts.c` (Winbond/Nuvoton SuperIO), `pc87360.c` + `pc87427.c` (National PC8736x), `lm63.c` + `lm70.c` + `lm73.c` + `lm75.c` + `lm77.c` + `lm78.c` + `lm80.c` + `lm83.c` + `lm85.c` + `lm87.c` + `lm90.c` + `lm92.c` + `lm93.c` + `lm95234.c` + `lm95241.c` + `lm95245.c` (National LM-series), `tmp*.c` (TI temperature ICs), `adt7*.c` + `ad741*.c` + `adm1*.c` (Analog Devices), `max1*.c` + `max3*.c` + `max6*.c` (Maxim), `pwm-fan.c` (PWM-controlled fan generic), `dell-smm-hwmon.c` (Dell laptops), `acpi_power_meter.c` (ACPI), `applesmc.c` (Apple Mac), `asus-ec-sensors.c` + `asus-wmi-sensors.c`, `asb100.c`, `oxp-sensors.c` (handheld gaming PCs), `peci/`, `intel-m10-bmc-hwmon.c`. Plus subdirs: `pmbus/` (PMBus power-supply chips), `peci/` (Intel PECI thermal interface).

### compatibility contract — outline

- `/sys/class/hwmon/hwmon<N>/{name, in<N>_{input,label,min,max}, temp<N>_{input,label,crit,max}, fan<N>_{input,label,div,target,min,max}, pwm<N>{,_enable,_mode}, curr<N>_{input,label}, power<N>_{input,label,average,cap}, energy<N>_input, ...}` byte-identical (lm-sensors `sensors`, sosreport, gnome-system-monitor consume).
- Channel naming convention preserved (in0_input = +5V, etc. per chip-specific labelling table).
- DT bindings + ACPI table integration parsed identically.
- Cross-ref `drivers/thermal/hwmon-bridge.md` for thermal-zone bridging.

### tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `drivers/hwmon/core.md` | `hwmon.c` + `hwmon-vid.c`: framework |
| `drivers/hwmon/cpu-x86.md` | `coretemp.c` + `k10temp.c` + `k8temp.c`: x86 CPU temp |
| `drivers/hwmon/super-io.md` | `nct*.c` + `it87.c` + `f71*.c` + `w83*.c` + `pc87*.c`: SuperIO chips |
| `drivers/hwmon/i2c-temp.md` | `lm*.c` + `tmp*.c` + `adt7*.c` + `max*.c`: I2C temperature ICs |
| `drivers/hwmon/pmbus.md` | `pmbus/`: PMBus power-supply chips |
| `drivers/hwmon/peci.md` | `peci/`: Intel PECI |
| `drivers/hwmon/laptop.md` | `dell-smm-hwmon.c` + `applesmc.c` + `asus-*.c` + `oxp-sensors.c` |
| `drivers/hwmon/acpi.md` | `acpi_power_meter.c` |

### compatibility outline / ac / verification / hardening

- REQ-O1: `/sys/class/hwmon/...` byte-identical (`sensors` consumes unchanged).
- REQ-O2: All v0 chip drivers register channels matching upstream chip-spec labels.
- REQ-O3: TLA+ models (concurrent reading + chip-update lockout per-driver).
- REQ-O4: AC: `sensors-detect && sensors` shows correct CPU/board/PSU sensors.
- Hardening: row-1; pwm fan-speed write requires CAP_SYS_RAWIO + LSM mediation (defense against fan-stop attack causing thermal damage).

