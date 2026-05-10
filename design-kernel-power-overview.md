---
title: "Tier-2: kernel/power — power management (suspend + hibernate + autosleep + wakeup_reason + console + energy_model + qos)"
tags: ["tier-2", "kernel", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 wrapper for the system-wide power management subsystem — sleep states (S0ix / S1 / S3 / S4 / S5), hibernate-to-disk (image creation + resume), autosleep (Android-style suspend-when-idle), per-device PM ordering (cross-ref `drivers/base/pm-system.md`), energy-model (per-CPU / per-task energy estimation for scheduler), pm_qos (latency / device-state QoS limits), wakeup-source enumeration. Components: `main.c` (subsystem init + sysfs registration), `suspend.c` (S1/S3 suspend-to-RAM), `hibernate.c` (S4 suspend-to-disk: image-create + resume from swap or dedicated partition), `swap.c` (hibernate-image swap layout), `snapshot.c` (hibernate snapshot generation), `swsusp.c` (sw-suspend image format), `wakelock.c` (Android-style wakelocks), `wakeup_reason.c` (per-wakeup-event source reporting), `console.c` (console suspend/resume), `process.c` (freeze/thaw of userspace processes), `energy_model.c` (per-CPU/per-domain energy table for EAS scheduler), `qos.c` (system-wide pm-qos), `autosleep.c` (autosleep workqueue), `poweroff.c` (poweroff syscall handler), `power.h` (private), `user.c` (`/dev/snapshot` chardev for userspace hibernate-image manipulation), `suspend_test.c` (CONFIG_PM_TEST_SUSPEND).

### Out of Scope

- Implementation code; 32-bit-only paths; ARM-only PM-runtime SoC drivers

### compatibility contract — outline

- `/sys/power/{state, mem_sleep, disk, image_size, reserved_size, sync_on_suspend, autosleep, wake_lock, wake_unlock, wakeup_count, ...}` byte-identical (systemd + elogind consume).
- `/sys/power/state` write triggers transition: `mem`/`standby`/`disk`/`freeze` byte-identical.
- `/sys/power/disk` write selects hibernate-mode: `platform`/`shutdown`/`reboot`/`suspend`/`test_resume` byte-identical.
- `/sys/power/mem_sleep` selects S3 vs S0ix mode (s2idle): `[s2idle] deep` byte-identical.
- `/sys/power/wakeup_count` for race-free suspend-arm.
- `/dev/snapshot` chardev IOCTLs for userspace hibernate-image manipulation: `SNAPSHOT_FREEZE`, `_UNFREEZE`, `_ATOMIC_RESTORE`, `_GET_IMAGE_SIZE`, `_PLATFORM_SUPPORT`, `_POWER_OFF`, `_AVAIL_SWAP_SIZE`, `_ALLOC_SWAP_PAGE`, `_FREE_SWAP_PAGES`, `_PREF_IMAGE_SIZE`, `_S2RAM` byte-identical.
- `/proc/sys/kernel/{wake_lock_max,nmi_watchdog,...}` byte-identical.
- ACPI _S0ix / _S3 / _S4 / _S5 + GPE wake routing handled identically.
- `events/power/` tracepoints byte-identical.

### tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `kernel/power/main.md` | `main.c`: subsystem init + sysfs |
| `kernel/power/suspend.md` | `suspend.c` + `process.c`: S1/S3 + freezer |
| `kernel/power/hibernate.md` | `hibernate.c` + `snapshot.c` + `swsusp.c` + `swap.c` + `user.c`: S4 hibernate + `/dev/snapshot` |
| `kernel/power/wakelock.md` | `wakelock.c` + `autosleep.c`: Android-style wakelocks + autosleep |
| `kernel/power/wakeup-reason.md` | `wakeup_reason.c`: per-wakeup-event reporting |
| `kernel/power/console.md` | `console.c`: console suspend/resume |
| `kernel/power/energy-model.md` | `energy_model.c`: per-CPU/per-domain energy table |
| `kernel/power/qos.md` | `qos.c`: system pm-qos |
| `kernel/power/poweroff.md` | `poweroff.c`: poweroff handler |
| `kernel/power/test.md` | `suspend_test.c`: CONFIG_PM_TEST_SUSPEND |

### compatibility outline / ac / verification / hardening

- REQ-O1: `/sys/power/*` UAPI byte-identical (systemd + elogind consume).
- REQ-O2: `/dev/snapshot` IOCTLs byte-identical (uswsusp consumes).
- REQ-O3: ACPI S0ix / S3 / S4 / S5 transitions work on reference HW.
- REQ-O4: events/power/ tracepoints byte-identical.
- REQ-O5: TLA+ models (suspend-arm vs wake-event race via wakeup_count; freezer cooperative-stop convergence; hibernate snapshot atomicity; energy-model per-task EAS computation).
- REQ-O6: AC: `systemctl suspend` / `systemctl hibernate` work; resume preserves all per-device state via dpm_list ordering; wakeup_reason reports correct wake source.
- Hardening: `/sys/power/state` write requires CAP_SYS_ADMIN; hibernate-image encryption (CONFIG_HIBERNATE_VERIFICATION) signature-verified before resume (defense against tampering with on-disk swap image to inject post-resume kernel state); per-LSM hook on /sys/power/state.

