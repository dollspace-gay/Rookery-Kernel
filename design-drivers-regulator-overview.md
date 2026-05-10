---
title: "Tier-2: drivers/regulator — voltage/current regulator framework (core + per-PMIC drivers + machine + userspace)"
tags: ["tier-2", "drivers", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 wrapper for the voltage/current regulator framework — every PMIC LDO, DC-DC converter, charge pump exposed via `regulator_get` / `regulator_enable` / `regulator_set_voltage`. Components: `core.c` (subsystem core), `helpers.c` (per-PMIC helpers), `fixed.c` + `fixed-helper.c` (fixed-voltage), `gpio-regulator.c` (GPIO-controlled), `userspace-consumer.c` (userspace `/sys/class/regulator/...`), `dummy.c` (no-PMIC fallback), `of_regulator.c` (DT bindings), `irq_helpers.c` (per-PMIC interrupt fault handling), per-PMIC drivers (~150 — most are ARM SoC PMICs; v0 maintenance set: `axp20x-regulator.c` (X-Powers), `da9*.c` (Dialog), `max77*.c` + `max8*.c` (Maxim), `mt6*.c` (MediaTek), `pca9450-regulator.c`, `palmas-regulator.c`, `qcom_*.c`, `tps6*.c` + `tps8*.c` (TI), `wm8*.c` (Wolfson)).

### compatibility contract — outline

- `regulator_*` consumer API source-compat.
- `/sys/class/regulator/regulator.<N>/{name, microvolts, microamps, opmode, type, state, min_microvolts, max_microvolts, num_users, requested_microamps, suspend_*, ...}` byte-identical.
- `/sys/class/regulator/.../consumers` byte-identical (per-consumer enable count).
- DT regulator bindings + machine constraints parsed identically.

### tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `drivers/regulator/core.md` | `core.c` + `helpers.c` + `irq_helpers.c`: framework |
| `drivers/regulator/fixed.md` | `fixed.c` + `fixed-helper.c` + `gpio-regulator.c` |
| `drivers/regulator/userspace.md` | `userspace-consumer.c` + `of_regulator.c` |
| `drivers/regulator/pmic-x86.md` | x86-relevant PMICs (intel-tps68470, intel-axp288, da9*) |

### compatibility outline / ac / verification / hardening

- REQ-O1: regulator_* consumer API source-compat.
- REQ-O2: `/sys/class/regulator/...` byte-identical.
- REQ-O3: TLA+ models (per-regulator enable-count refcount; voltage-set under multiple consumers respecting min/max constraint intersection).
- REQ-O4: AC: laptop boot brings up all PMIC regulators; runtime PM voltage scaling works.
- Hardening: regulator_*set_voltage requires consumer-context; voltage-set outside of constraints rejected; per-LSM hook on /sys/class/regulator writes.

