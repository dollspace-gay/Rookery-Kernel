# Tier-2: drivers/clk — common clock framework (CCF + per-SoC clk providers + clk-mux/gate/divider)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/clk/
  - include/linux/clk.h
  - include/linux/clk-provider.h
-->

## Summary

Tier-2 wrapper for the Common Clock Framework (CCF) — every per-SoC PLL + clock-divider + clock-gate + clock-mux. Components: `clk.c` (core: clk_get / clk_prepare / clk_enable / clk_set_rate / clk_get_parent), `clk-divider.c` + `clk-gate.c` + `clk-mux.c` + `clk-fixed-rate.c` + `clk-fixed-factor.c` + `clk-composite.c` + `clk-multiplier.c` + `clk-fractional-divider.c` (basic clock-element implementations), `clkdev.c` (legacy clk-lookup), per-SoC subdirs (mostly ARM — `at91/`, `bcm/`, `imx/`, `mediatek/`, `meson/`, `mvebu/`, `qcom/`, `renesas/`, `rockchip/`, `samsung/`, `sunxi/`, `tegra/`, `ti/`, `versatile/` — compile-gated off for x86_64 v0). x86 v0 set: `clk-fixed-rate.c`, `clk-pmc-atom.c` (Intel Atom PMC), generic CCF for clkdev consumers.

## Compatibility contract — outline

- `clk_*` consumer API source-compat for in-tree drivers.
- `/sys/kernel/debug/clk/` clk-tree introspection byte-identical (clk_summary/clk_dump consumed by `clk-dump` userspace tool).
- DT `clocks = <&...>` + `clock-names = "..."` + ACPI clock binding parsed identically.

## Tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `drivers/clk/clk-core.md` | `clk.c` + `clk-provider.h` consumer-API + lifecycle |
| `drivers/clk/basic-elements.md` | clk-{divider, gate, mux, fixed-rate, fixed-factor, composite, multiplier, fractional-divider}.c |
| `drivers/clk/clkdev.md` | `clkdev.c`: legacy clk-lookup |
| `drivers/clk/x86-pmc.md` | `clk-pmc-atom.c`: Intel Atom PMC clocks |

## Compatibility outline / AC / Verification / Hardening

- REQ-O1: clk_* consumer API source-compat.
- REQ-O2: clk-tree debugfs byte-identical.
- REQ-O3: TLA+ models (clk-rate propagation through cascaded dividers/multiplexers; concurrent clk_set_rate from multiple consumers serialized via prepare-mutex).
- REQ-O4: AC: every in-tree driver consuming clk_* compiles + works; clk-summary debugfs shows correct tree.
- Hardening: row-1 features; clk_set_rate write-paths require parent-driver context; debugfs root-only.

## Out of Scope
ARM-only SoC clk providers; implementation code; 32-bit-only paths.
