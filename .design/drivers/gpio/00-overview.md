# Tier-2: drivers/gpio — GPIO subsystem (gpiolib + gpiochip + gpio-cdev + per-controller drivers + ACPI/OF binding)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/gpio/
  - include/linux/gpio.h
  - include/linux/gpio/
  - include/uapi/linux/gpio.h
-->

## Summary

Tier-2 wrapper for the GPIO (General Purpose I/O) subsystem — every reset-line, every chip-enable, every status LED, every interrupt-line, every reset-button, every PCIe slot wake#, every PMC notify-pin, every BMC sideband, every laptop dock-detect line. Components: **gpiolib core** (`gpiolib.c` + `gpiolib.h` + `gpiolib-of.c` + `gpiolib-acpi.c` + `gpiolib-acpi-{core,quirks}.c` + `gpiolib-cdev.c` + `gpiolib-sysfs.c` + `gpiolib-swnode.c`: `gpio_chip` + `gpio_desc` + `gpiod_*` consumer API + ACPI/DT/swnode pin lookup + cdev v2 API + legacy sysfs API), **per-controller drivers** (~150 controllers; v0 maintenance set: gpio-amd-fch + gpio-amd8111 + gpio-amdpt (AMD chipsets), gpio-ich (Intel ICH), gpio-i801 (Intel x86 SMBus-attached), gpio-it87 (ITE Super-IO), gpio-f7188x (Fintek Super-IO), gpio-sch (Intel SCH/Tunnel-Creek), gpio-vx855 (VIA), gpio-winbond, gpio-w83627hf (Winbond Super-IO); plus generic: gpio-aggregator (combine multiple gpio_chips into virtual one), gpio-mockup + gpio-sim (test mocks), gpio-mmio + gpio-generic-platform, gpio-virtio (virtio-GPIO for guest VMs)).

## Compatibility contract — outline

- **gpio-cdev v2** (`/dev/gpiochip<N>` chardev) IOCTLs byte-identical: `GPIO_GET_CHIPINFO_IOCTL`, `GPIO_GET_LINEINFO_IOCTL`, `GPIO_GET_LINE_IOCTL`, `GPIO_GET_LINEEVENT_IOCTL`, `GPIO_V2_GET_LINEINFO_IOCTL`, `GPIO_V2_GET_LINE_IOCTL`, `GPIO_V2_LINE_SET_VALUES_IOCTL`, `GPIO_V2_LINE_GET_VALUES_IOCTL`, `GPIO_V2_LINE_SET_CONFIG_IOCTL`, `GPIO_V2_LINE_REQUEST_IOCTL`, `GPIO_V2_GET_LINEINFO_WATCH_IOCTL`, `GPIO_V2_GET_LINEINFO_UNWATCH_IOCTL`. Wire format byte-identical so libgpiod (gpioget/gpioset/gpiomon/gpioinfo/gpiodetect) and Python gpiod consume unchanged.
- **gpio-cdev v1** (legacy line-handle + line-event) preserved for backward compat.
- **legacy /sys/class/gpio/** sysfs API preserved (CONFIG_GPIO_SYSFS=y default-N upstream since 4.8 — Rookery default-N).
- ACPI _DSD `gpios` property + `_CRS` GpioIo / GpioInt resources + DT `gpios = <&...>` parsing identical.
- Per-controller `/sys/class/gpio/gpiochip<N>/{base,label,ngpio}` byte-identical.

## Tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `drivers/gpio/gpiolib-core.md` | `gpiolib.c` + `gpiolib.h` + `gpiolib-swnode.c`: gpio_chip + gpio_desc + gpiod_* consumer API |
| `drivers/gpio/gpiolib-cdev.md` | `gpiolib-cdev.c`: `/dev/gpiochip<N>` chardev v1 + v2 |
| `drivers/gpio/gpiolib-sysfs.md` | `gpiolib-sysfs.c`: legacy `/sys/class/gpio/` (default-off) |
| `drivers/gpio/gpiolib-acpi.md` | `gpiolib-acpi*.c`: ACPI _DSD + _CRS GpioIo/GpioInt parsing |
| `drivers/gpio/gpiolib-of.md` | `gpiolib-of.c`: DT `gpios` property parsing |
| `drivers/gpio/controllers-x86.md` | `gpio-{amd-fch, amd8111, amdpt, ich, i801, it87, f7188x, sch, vx855, winbond, w83627hf}.c`: x86 chipset + Super-IO GPIO controllers |
| `drivers/gpio/controllers-generic.md` | `gpio-{aggregator, mmio, generic-platform, virtio, mockup, sim, 74x164, pca953x, pcf857x, mcp23s08}.c`: generic + virtual + i2c/spi-attached GPIO expanders |

## Compatibility outline

- REQ-O1: `/dev/gpiochip<N>` cdev v1 + v2 IOCTLs byte-identical (libgpiod consumes unchanged).
- REQ-O2: ACPI / DT pin lookup produces identical line numbering.
- REQ-O3: Per-class sysfs surface byte-identical.
- REQ-O4: gpiod_* consumer API source-compat for in-tree drivers.
- REQ-O5: TLA+ models (per-line request/release race, edge-event delivery to userspace via cdev).
- REQ-O6: Hardening: `/dev/gpiochip*` CAP_SYS_RAWIO + LSM mediation.

## Acceptance Criteria

- [ ] AC-O1: `gpiodetect` lists every gpiochip on a reference x86 system matching upstream baseline.
- [ ] AC-O2: `gpioset gpiochip0 5=1` toggles a GPIO line; `gpioget gpiochip0 5` reads back.
- [ ] AC-O3: `gpiomon -r gpiochip0 5` captures rising edges from a button.
- [ ] AC-O4: kselftest `tools/testing/selftests/gpio/` passes.

## Verification

| TLA+ Model | Owner |
|---|---|
| `models/gpio/line_request.tla` | `drivers/gpio/gpiolib-core.md` (proves: gpio_desc per-line request/release with concurrent claimers; per-line owner + ref tracking; release-while-held propagates correctly) |
| `models/gpio/edge_event.tla` | `drivers/gpio/gpiolib-cdev.md` (proves: edge-event delivery from IRQ context to userspace via per-line ringbuf; concurrent rising/falling edges + cdev poll/read never lose event or duplicate) |

## Hardening

| Feature | Default |
|---|---|
| **REFCOUNT** | per-gpio_chip + per-gpio_desc refcounts use `Refcount` | § Mandatory |
| **CONSTIFY** | per-controller `gpio_chip` ops `static const` | § Mandatory |
| **SIZE_OVERFLOW** | per-chip ngpio + line-event ringbuf arithmetic checked | § Mandatory |
| **MEMORY_SANITIZE** | freed gpio_desc state cleared | § Default-on configurable |

GPIO-specific reinforcement: `/dev/gpiochip*` open requires CAP_SYS_RAWIO (defense against userspace toggling reset/power lines on chassis devices); per-line ACL via `GPIO_V2_LINE_REQUEST_IOCTL` flags + LSM mediation; `gpio-aggregator` virtual-chip creation requires CAP_SYS_ADMIN.

## Open Questions
(none at Tier-2)

## Out of Scope
- ARM-only SoC GPIO controllers (most of `gpio-{at91, bcm-kona, davinci, exynos, ...}.c` compile-gated off for v0)
- Legacy sysfs API (default-N; deprecation candidate)
- Implementation code
- 32-bit-only paths
