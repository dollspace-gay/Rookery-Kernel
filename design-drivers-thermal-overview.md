---
title: "Tier-2: drivers/thermal — thermal framework (thermal_zone + cooling_device + governors + per-driver thermal sensors)"
tags: ["tier-2", "drivers", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 wrapper for the thermal framework — every CPU temperature sensor, every per-package PMIC temperature, every NVMe / SSD temperature, every battery-pack thermal monitor, every laptop fan, every per-IC thermal trip-point. Components: `thermal_core.c` + `thermal_helpers.c` + `thermal_sysfs.c` + `thermal_trip.c` + `thermal_thermometer.c` + `thermal_hwmon.c`: framework — `thermal_zone_device` + `thermal_cooling_device` + `thermal_governor` + per-zone trip points + per-cooling-device action, `thermal_netlink.c` (NETLINK_THERMAL_GENL via genetlink for thermald), `gov_*.c` (governors: `bang_bang`, `step_wise`, `fair_share`, `power_allocator`, `user_space`), per-driver subdirs: `intel/{int340x_thermal, intel_pch_thermal, intel_powerclamp, intel_quark_dts, intel_soc_dts_iosf, intel_tcc_cooling, x86_pkg_temp_thermal}`, `amd_hsmp/`, per-SoC ARM (`armada_thermal.c`, `db8500_thermal.c`, etc. — out of v0). v0 maintenance set: thermal_core, intel/* (most), amd_hsmp, hwmon-thermal-zone bridge.

### compatibility contract — outline

- `/sys/class/thermal/thermal_zone<N>/{type, temp, mode, policy, available_policies, trip_point_<N>_temp, trip_point_<N>_type, trip_point_<N>_hyst, ...}` byte-identical (lm-sensors + thermald + sensors consume).
- `/sys/class/thermal/cooling_device<N>/{type, max_state, cur_state}` byte-identical.
- NETLINK_THERMAL_GENL (genetlink "thermal") wire format byte-identical (thermald consumes).
- Cmdline `thermal.act=`, `thermal.crt=`, `thermal.psv=`, `thermal.tzp=` parsed identically.

### tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `drivers/thermal/core.md` | `thermal_core.c` + `thermal_helpers.c` + `thermal_trip.c`: framework |
| `drivers/thermal/sysfs-netlink.md` | `thermal_sysfs.c` + `thermal_netlink.c` |
| `drivers/thermal/governors.md` | `gov_*.c` |
| `drivers/thermal/hwmon-bridge.md` | `thermal_hwmon.c`: hwmon ↔ thermal-zone bridge |
| `drivers/thermal/intel.md` | `intel/`: Intel-specific thermal sensors |
| `drivers/thermal/amd-hsmp.md` | `amd_hsmp/`: AMD HSMP |

### compatibility outline / ac / verification / hardening

- REQ-O1: `/sys/class/thermal/...` byte-identical.
- REQ-O2: NETLINK_THERMAL_GENL byte-identical (thermald consumes).
- REQ-O3: All governors behave identically.
- REQ-O4: TLA+ models (governor-decision-loop + concurrent zone-temp-update + cooling-device-state-change race-freedom; per-zone trip-point monotonicity).
- REQ-O5: AC: laptop CPU temp visible in `sensors`; CPU throttling triggers when temp > trip_point_passive.
- Hardening: row-1 features; thermal-zone mode write requires CAP_SYS_ADMIN; thermal_user_space governor disabled by default (defense against userspace bypassing thermal limits).

