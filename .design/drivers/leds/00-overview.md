# Tier-2: drivers/leds — LED class subsystem (core + per-controller drivers + triggers + flash + RGB)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/leds/
  - include/linux/leds.h
-->

## Summary

Tier-2 wrapper for the LED class subsystem — every status LED (CapsLock/NumLock keyboard LEDs, laptop power/charge/disk indicators, RGB-keyboard backlights, server chassis LEDs, NIC port-link LEDs, smart bulbs via I2C). Components: `led-class.c` + `led-core.c` + `led-class-flash.c` + `led-class-multicolor.c`: framework — `led_classdev` lifecycle + brightness control + flash/torch ops + multicolor (RGB / RGBW), `led-triggers.c` (trigger framework — `none`, `default-on`, `timer`, `oneshot`, `disk-activity`, `mtd`, `nand-disk`, `panic`, `audio-mic-mute`, `audio-vol-down`, `audio-vol-up`, `bluetooth`, `cpu`, `gpio`, `heartbeat`, `kbd-*`, `morse`, `mtd`, `netdev`, `pattern`, `transient`, `tty`, `usbport`), per-driver subdirs: `blink/`, `flash/`, `rgb/`, `simple/`, `trigger/`. Per-chip drivers (~120 — most ARM SoC; v0 maintenance set: leds-class.c, leds-pca955x, leds-pca963x, leds-pca9532, leds-pwm, leds-gpio, leds-spi-byte, leds-ipipe (Intel), leds-mlxreg, leds-cht-wcove, leds-clevo-mail, leds-asus-wmi, leds-dell-netbooks).

## Compatibility contract — outline

- `/sys/class/leds/<name>/{brightness, max_brightness, trigger, ...}` byte-identical.
- Per-trigger sysfs (e.g., `delay_on`, `delay_off`, `pattern`, `repeat`, `interval`) byte-identical.
- DT `leds` binding + ACPI integration parsed identically.

## Tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `drivers/leds/core.md` | `led-core.c` + `led-class.c`: framework |
| `drivers/leds/flash-multicolor.md` | `led-class-flash.c` + `led-class-multicolor.c` |
| `drivers/leds/triggers.md` | `led-triggers.c` + `trigger/`: per-trigger drivers |
| `drivers/leds/per-controller.md` | per-vendor LED IC drivers |

## Compatibility outline / AC / Verification / Hardening

- REQ-O1: `/sys/class/leds/...` byte-identical.
- REQ-O2: All triggers behave identically.
- REQ-O3: TLA+ models (concurrent brightness-set + trigger-fire serialization).
- REQ-O4: AC: keyboard CapsLock LED reflects state; disk-activity trigger blinks on IO; pattern trigger plays.
- Hardening: row-1 features; per-LED LSM mediation; flash/torch ops require CAP_SYS_RAWIO (defense against retina-flash injection from camera flash LED).

## Out of Scope
Implementation code; ARM-only SoC LED drivers; 32-bit-only paths.
