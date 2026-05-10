---
title: "Tier-2: drivers/clocksource — clocksource + clockevent device drivers (HPET + TSC + KVM-clock + acpi_pm + per-SoC timers)"
tags: ["tier-2", "drivers", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 wrapper for clocksource + clockevent device drivers — providers of `kernel/time/clocksource.md` and `kernel/time/clockevents.md` registries. v0 set: `acpi_pm.c` (ACPI Power Management Timer), `hpet.c` (`/dev/hpet` legacy chardev), per-arch x86 TSC + KVM-clock + LAPIC-timer live in `arch/x86/kernel/{tsc,kvmclock}.c` and `arch/x86/kernel/apic/`; from `drivers/clocksource/` perspective most files are ARM SoC timers (compile-gated off): `arm_arch_timer.c`, `arm_global_timer.c`, `bcm2835_timer.c`, `bcm_kona_timer.c`, `cadence_ttc_timer.c`, `clksrc-dbx500-prcmu.c`, `clps711x-timer.c`, `dw_apb_timer*.c`, `em_sti.c`, `exynos_mct.c`, `gxp-timer.c`, `h8300_timer*.c`, `i8253.c` (legacy x86 PIT — keep), `imx-tpm.c`, `ingenic-ost.c`, `ingenic-sysost.c`, `ingenic-timer.c`, `jcore-pit.c`, `mips-gic-timer.c`, `mmio.c` (generic MMIO clocksource), `numachip.c`, `pxa_timer.c`, `qcom-timer.c`, `renesas-ostm.c`, `rockchip_timer.c`, `samsung_pwm_timer.c`, `scx200_hrt.c`, `sh_cmt.c`, `sh_mtu2.c`, `sh_tmu.c`, `sp804.c`, `sun4i_timer.c`, `sun5i.c`, `tegra20_timer.c`, `time-armada-370-xp.c`, `time-efm32.c`, `time-orion.c`, `time-pistachio.c`, `timer-atcpit100.c`, `timer-atmel-pit.c`, `timer-atmel-st.c`, `timer-atmel-tcb.c`, `timer-cadence-ttc.c`, `timer-clint.c` (RISC-V), `timer-davinci.c`, `timer-fsl-ftm.c`, `timer-fttmr010.c`, `timer-gx6605s.c`, `timer-imx-gpt.c`, `timer-imx-sysctr.c`, `timer-imx-tpm.c`, `timer-integrator-ap.c`, `timer-keystone.c`, `timer-lpc32xx.c`, `timer-meson6.c`, `timer-microchip-pit64b.c`, `timer-mp-csky.c`, `timer-nps.c`, `timer-npcm7xx.c`, `timer-of.c`, `timer-orion.c`, `timer-owl.c`, `timer-prima2.c`, `timer-pxa.c`, `timer-qcom.c`, `timer-rda.c`, `timer-riscv.c`, `timer-sp804.c`, `timer-stm32.c`, `timer-sun4i.c`, `timer-sun5i.c`, `timer-sunplus.c`, `timer-tegra.c`, `timer-tegra186.c`, `timer-ti-32k.c`, `timer-ti-dm.c`, `timer-versatile.c`, `timer-vf-pit.c`, `timer-vt8500.c`, `timer-zevio.c`, `vf_pit_timer.c`. v0 maintenance: acpi_pm + hpet + i8253 + mmio (generic) + numachip; rest compile-gated off.

### compatibility contract — outline

- `/dev/hpet` chardev (HPET_INFO, HPET_IRQFREQ, HPET_EPI, HPET_DPI, HPET_IE_ON/OFF, HPET_ALARM_ON/OFF) byte-identical.
- `/sys/devices/system/clocksource/clocksource0/{available_clocksource, current_clocksource, unbind_clocksource}` byte-identical (cross-ref `kernel/time/00-overview.md`).
- Per-arch entry x86 TSC clocksource registration handled in `arch/x86/kernel/tsc.c` (separate Tier-3 there).

### tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `drivers/clocksource/acpi-pm.md` | `acpi_pm.c` |
| `drivers/clocksource/hpet.md` | `hpet.c` |
| `drivers/clocksource/i8253.md` | `i8253.c`: legacy x86 PIT |
| `drivers/clocksource/mmio-generic.md` | `mmio.c` |

### compatibility outline / ac / verification / hardening

- REQ-O1: `/dev/hpet` IOCTLs byte-identical.
- REQ-O2: Per-arch clocksource enumeration unchanged.
- REQ-O3: TLA+ models (per-clocksource read seqcount under wraparound; per-clockevent oneshot vs periodic mode switch).
- REQ-O4: AC: `cat /sys/devices/system/clocksource/clocksource0/available_clocksource` shows TSC+HPET+ACPI-PMT on x86 reference; `cat current_clocksource` = "tsc".
- Hardening: row-1; `/dev/hpet` open requires CAP_SYS_TIME.

