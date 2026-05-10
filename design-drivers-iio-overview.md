---
title: "Tier-2: drivers/iio — Industrial I/O subsystem (sensors: ADC, accel, gyro, magn, prox, ALS, press, temp, hum, color, gas, IMU, dac, light)"
tags: ["tier-2", "drivers", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 wrapper for IIO — uniform sensor framework underneath every accelerometer (auto-rotate / fitness tracker), gyroscope, magnetometer (compass), proximity (auto-dim), ambient-light-sensor, pressure (altimeter), humidity, temperature (thermometer), gas (CO2/VOC/eCO2 air quality), color (RGB), heart-rate, IMU (6-axis/9-axis combined). Components: `industrialio-core.c` + `industrialio-buffer-cb.c` + `industrialio-buffer-dma.c` + `industrialio-buffer-dmaengine.c` + `industrialio-event.c` + `industrialio-trigger.c` + `industrialio-triggered-buffer.c` + `industrialio-sw-device.c` + `industrialio-sw-trigger.c` + `industrialio-configfs.c` + `industrialio-gts-helper.c`: framework — `iio_dev` lifecycle + per-channel readings + buffered sampling via DMA + per-trigger sample-trigger framework. **Sub-class subdirs**: `accel/`, `adc/`, `addac/`, `afe/`, `amplifiers/`, `buffer/`, `cdc/` (capacitive), `chemical/`, `common/{cros_ec_sensors, hid-sensor-hub, ms_sensors, scmi_sensors, ssp_sensors, st_sensors}`, `dac/`, `dummy/`, `filter/`, `frequency/`, `gyro/`, `health/`, `humidity/`, `imu/`, `inkern.c` (in-kernel client API), `industrialio-core.c`, `industrialio-event.c`, `industrialio-sw-device.c`, `industrialio-sw-trigger.c`, `industrialio-trigger.c`, `light/`, `magnetometer/`, `multiplexer/`, `orientation/`, `position/`, `potentiometer/`, `potentiostat/`, `pressure/`, `proximity/`, `resolver/`, `temperature/`, `test/`, `trigger/`. Shared sensor-hub backend: `common/hid-sensor-hub` consumed from `drivers/hid/00-overview.md` Sensor Hub.

### compatibility contract — outline

- `/sys/bus/iio/devices/iio:device<N>/{in_<type>_<index>_{raw,offset,scale,sampling_frequency,calibbias,calibscale,calibheight}, out_<type>_<index>_{raw,...}, name, sampling_frequency_available, scan_elements/, buffer/, trigger/}` byte-identical (iio-utils + iio-sensor-proxy + sensors consume unchanged).
- `/dev/iio:device<N>` chardev byte-identical for buffered access.
- `events/iio/` tracepoints byte-identical.

### tier-3 docs governed (selection — full list deferred)

| Tier-3 | Scope |
|---|---|
| `drivers/iio/iio-core.md` | `industrialio-core.c` + `industrialio-event.c` |
| `drivers/iio/iio-buffer.md` | `industrialio-buffer-*.c` + `industrialio-triggered-buffer.c` |
| `drivers/iio/iio-trigger.md` | `industrialio-trigger.c` + `industrialio-sw-trigger.c` |
| `drivers/iio/iio-configfs.md` | `industrialio-configfs.c` |
| `drivers/iio/inkern.md` | `inkern.c` |
| `drivers/iio/sensor-hid.md` | `common/hid-sensor-hub`: HID Sensor Hub IIO bridge (cross-ref drivers/hid) |
| `drivers/iio/cros-ec.md` | `common/cros_ec_sensors`: ChromeOS EC sensor bridge |
| `drivers/iio/accel-gyro-magn.md` | `accel/`, `gyro/`, `magnetometer/`, `imu/` per-chip drivers |
| `drivers/iio/light-prox.md` | `light/`, `proximity/` |
| `drivers/iio/temp-pressure.md` | `temperature/`, `pressure/`, `humidity/` |
| `drivers/iio/adc-dac.md` | `adc/`, `dac/`, `addac/` |
| `drivers/iio/chemical-health.md` | `chemical/`, `health/` |

### compatibility outline / ac / verification / hardening

- REQ-O1: `/sys/bus/iio/devices/...` byte-identical.
- REQ-O2: `/dev/iio:device<N>` chardev byte-identical.
- REQ-O3: All in-tree consumed sensor drivers compile + register channels.
- REQ-O4: TLA+ models (per-buffer DMA producer + userspace consumer ringbuf; per-trigger fire-once-per-event semantics).
- REQ-O5: AC: iio-sensor-proxy reports orientation correctly on laptop with HID sensor hub.
- Hardening: per-device LSM mediation; `/dev/iio:device*` open requires CAP_SYS_RAWIO + LSM (some sensors leak fingerprintable user state).

