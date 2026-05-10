---
title: "Tier-2: drivers/watchdog — watchdog timer framework + per-controller LLDs"
tags: ["tier-2", "drivers", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 wrapper for watchdog — chassis hang-detection + auto-reboot. Components: `watchdog_core.c` + `watchdog_dev.c` + `watchdog_pretimeout.c` + `watchdog_hrtimer_pretimeout.c`: framework — `watchdog_device` registration + `/dev/watchdog<N>` chardev + governor framework for pretimeout, per-controller LLDs (~80, v0 set: `iTCO_wdt.c` (Intel TCO chipsets), `iTCO_vendor_support.c`, `softdog.c` (software watchdog using hrtimer; backbone of CI/test setups), `i6300esb.c` (Intel 6300ESB), `i8xx_tco.c` (legacy Intel), `f71808e_wdt.c` + `f71862fg.c` + `f71808e.c` (Fintek SuperIO), `it87_wdt.c` + `it8712f_wdt.c` (ITE), `via_wdt.c` (VIA), `wdat_wdt.c` (ACPI WDAT), `nv_tco.c` (NVIDIA chipset), `sp5100_tco.c` (AMD SP5100), `w83627hf_wdt.c` + `w83977f_wdt.c` + `w83977f_wdt.c` (Winbond/Nuvoton), `mlx_wdt.c` (Mellanox switch chassis BMC), `dell_rbu.c` glue, `intel-mid_wdt.c`, `xen_wdt.c` (Xen guest), `hpwdt.c` (HP iLO chassis), `cpu5wdt.c`, `eurotechwdt.c`, `wdt_pci.c`, `mpc8xxx_wdt.c` (out of v0 PowerPC).

### compatibility contract — outline

- `/dev/watchdog<N>` chardev IOCTLs byte-identical: `WDIOC_GETSUPPORT`, `_GETSTATUS`, `_GETBOOTSTATUS`, `_GETTEMP`, `_SETOPTIONS`, `_KEEPALIVE`, `_SETTIMEOUT`, `_GETTIMEOUT`, `_SETPRETIMEOUT`, `_GETPRETIMEOUT`, `_GETTIMELEFT`. Wire format byte-identical (watchdog daemon + systemd watchdog consume unchanged).
- `/dev/watchdog0` symlink to first registered.
- Heartbeat write to `/dev/watchdog*` keeps watchdog alive; missing write triggers reset.
- `/sys/class/watchdog/watchdog<N>/{nowayout, state, status, identity, timeout, pretimeout, timeleft, options, fw_version, bootstatus, pretimeout_governor, pretimeout_available_governors}` byte-identical.
- Cmdline `watchdog.handle_boot_enabled=`, `watchdog.open_timeout=`, `watchdog.stop_on_reboot=`, per-driver `<drv>.timeout=`, `<drv>.nowayout=` parsed identically.
- Magic-close char `'V'` written to /dev/watchdog stops watchdog (when CONFIG_WATCHDOG_NOWAYOUT=N).

### tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `drivers/watchdog/core.md` | `watchdog_core.c` + `watchdog_dev.c`: framework + chardev |
| `drivers/watchdog/pretimeout.md` | `watchdog_pretimeout.c` + `watchdog_hrtimer_pretimeout.c`: pretimeout governors |
| `drivers/watchdog/softdog.md` | `softdog.c`: hrtimer-based software watchdog |
| `drivers/watchdog/intel.md` | `iTCO_wdt.c` + `i6300esb.c` + `i8xx_tco.c` + `intel-mid_wdt.c`: Intel chipsets |
| `drivers/watchdog/super-io.md` | `f71*.c` + `it*.c` + `via_wdt.c` + `w83*.c` + `nv_tco.c`: SuperIO chips |
| `drivers/watchdog/wdat-acpi.md` | `wdat_wdt.c`: ACPI WDAT generic watchdog |
| `drivers/watchdog/chassis-bmc.md` | `hpwdt.c` + `mlx_wdt.c` + `dell_rbu.c` glue |
| `drivers/watchdog/xen.md` | `xen_wdt.c`: Xen guest watchdog |

### compatibility outline / ac / verification / hardening

- REQ-O1: `/dev/watchdog<N>` IOCTLs byte-identical (watchdog daemon + systemd watchdog consume).
- REQ-O2: Per-class sysfs surface byte-identical.
- REQ-O3: Cmdline params parsed identically.
- REQ-O4: TLA+ models (heartbeat-vs-timeout race; pretimeout governor → restart pipeline).
- REQ-O5: AC: systemd watchdog with `WatchdogSec=30` resets on hung system; softdog test passes.
- Hardening: `/dev/watchdog*` open requires CAP_SYS_RAWIO; nowayout default-Y prevents accidental close stopping watchdog (defense against malicious userspace disabling watchdog before-attack).

