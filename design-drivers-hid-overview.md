---
title: "Tier-2: drivers/hid — HID subsystem (HID core + USB HID + I2C HID + Bluetooth HID + Intel ISH + AMD SFH + per-vendor + hidraw + hid-bpf)"
tags: ["tier-2", "drivers", "hid", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 overview for HID — the Human Interface Device subsystem that turns every keyboard, mouse, touchpad, touchscreen, joystick, gamepad, steering wheel, drawing tablet, force-feedback device, biometric reader, smartcard reader, sensor hub, FIDO/U2F key, software-defined HID, vendor-specific gaming peripheral and mass-market USB-pass-through device into kernel input events. Five-layer architectural sandwich:

1. **HID core** (`hid-core.c` + `hid-input.c` + `hid-debug.c`): generic Report Descriptor parser (the heart — interprets vendor-supplied descriptors per HID-1.11 spec), input-event mapper, force-feedback core, hidraw chardev, hid_driver registration framework, hid_bus type registration
2. **Transport buses**:
   - **USB HID** (`usbhid/`): USB-bus transport — bulk of keyboards, mice, gamepads. Quirk-list for buggy USB HID devices
   - **I2C HID** (`i2c-hid/`): I2C-bus transport — bulk of laptop touchpads + touchscreens (HID-over-I2C; ACPI-described)
   - **Bluetooth HID** (`net/bluetooth/hidp/`): Bluetooth transport — wireless keyboards/mice/gamepads (HID-over-L2CAP)
   - **Intel ISH** (`intel-ish-hid/`): Intel Integrated Sensor Hub — sensor hub on Intel SoCs (gyro/accel/ALS/proximity for laptop tablet-mode + auto-rotate)
   - **AMD SFH** (`amd-sfh-hid/`): AMD Sensor Fusion Hub — equivalent on AMD APUs
   - **Surface HID** (`surface-hid/`): Microsoft Surface Aggregator HID
   - **PS/2 HID** (consumed by `drivers/input/serio/` — separate Tier-2 future): PS/2-bus keyboards/mice via i8042
   - **MS-HID** (Microsoft proprietary keyboard/mouse via `hid-microsoft.c`)
3. **Per-vendor / per-device drivers** (~150 files): per-device quirk drivers that override default HID handling for specific devices (Apple keyboards' Fn-key, Logitech proprietary protocols, Razer LED control, Wacom tablet pressure curves, gamepads with non-spec layouts, etc.)
4. **HID-BPF** (`bpf/`): runtime BPF-based HID Report Descriptor mangling + event filtering (CONFIG_HID_BPF) — used by tools like `hid-recorder` + community-maintained `udev-hid-bpf` rule library to fix broken device descriptors without writing C drivers
5. **hidraw** (chardev `/dev/hidraw<N>`): raw HID report passthrough to userspace; consumed by libhidapi, libfido2, OpenRGB, Solaar (Logitech), libratbag (gaming mice), Razer-related tools
6. **uhid** (chardev `/dev/uhid`): userspace HID emulation — userspace process registers as a virtual HID device (used for Bluetooth HID profile-daemon side, virtual gamepad emulation via Steam Input, qmk_firmware-style remappers)

Heavily cross-referenced from `drivers/usb/00-overview.md` (USB transport for usbhid), `drivers/i2c/` (I2C transport for i2c-hid), `net/bluetooth/00-overview.md` (Bluetooth HID Profile), `drivers/input/00-overview.md` (HID maps to input-events), `kernel/bpf/00-overview.md` (hid-bpf programs are BPF_PROG_TYPE_* attached to HID descriptor + event hooks).

### Out of Scope

- Per-Tier-3 (Phase D) — per-vendor docs arrive incrementally
- ARM-only HID transports beyond i2c-hid (e.g., SoC-specific HID subdevices)
- Most legacy / single-device quirk drivers stay as upstream-FFI in v0
- `drivers/input/` core (evdev / joydev / mousedev / serio for PS/2) — separate Tier-2 wrapper future
- Bluetooth HID profile internals — covered by net/bluetooth Tier-2 wrapper future
- 32-bit-only paths
- Implementation code

### components

### Core (`drivers/hid/`)

- **hid-core.c** + **hid-debug.c**: Report Descriptor parser, hid_driver bus, generic device probe + report dispatch
- **hid-input.c**: HID-input usage-table mapping (HID-usage → Linux input-event)
- **hid-generic.c**: catchall driver matched when no per-vendor driver claims the device
- **hidraw.c**: `/dev/hidraw<N>` chardev passthrough
- **uhid.c**: `/dev/uhid` userspace HID emulation
- **hid-quirks.c**: per-device quirks table

### Per-bus transports

- **usbhid/** (`usbhid/`): USB bus transport — `hid-core.c`, `hid-pidff.c` (PID Force Feedback), `usbkbd.c` + `usbmouse.c` (legacy minimal-USB-HID drivers — kept for embedded use)
- **i2c-hid/** (`i2c-hid/`): I2C bus transport — `i2c-hid-core.c`, `i2c-hid-acpi.c` (ACPI-enumerated I2C-HID), `i2c-hid-of.c` (DT-enumerated; mostly ARM)
- **intel-ish-hid/** (`intel-ish-hid/`): Intel Integrated Sensor Hub
- **amd-sfh-hid/** (`amd-sfh-hid/`): AMD Sensor Fusion Hub
- **surface-hid/** (`surface-hid/`): MS Surface Aggregator HID
- **bpf/** (`bpf/`): HID-BPF runtime

(Bluetooth HID lives in `net/bluetooth/hidp/` — owned by the Bluetooth subsystem cross-reference)

### Per-vendor / per-device drivers (~140 files)

Selection of the most active maintained drivers (the rest are quirks-only one-shots):

| Driver | Coverage |
|---|---|
| `hid-apple.c` | Apple keyboards (Fn-key swap, brightness keys) |
| `hid-asus.c` | ASUS laptops + ROG keyboards |
| `hid-cherry.c` | Cherry mechanical keyboards |
| `hid-logitech-*` | Logitech Unifying receiver protocol + per-device driver (`hid-logitech-dj.c`, `hid-logitech-hidpp.c`) |
| `hid-microsoft.c` | Microsoft keyboards/mice + Xbox controllers |
| `hid-multitouch.c` | Multi-touch touchscreens (Win8+ MT-protocol) |
| `hid-roccat-*` | Roccat gaming mice |
| `hid-razer.c` (out-of-tree historically; kernel side via `hid-generic`) | (community openrgb covers most) |
| `hid-sony.c` | PlayStation 3/4/5 controllers |
| `hid-steam.c` | Steam Controller / Steam Deck buttons |
| `hid-wacom.c` (replaced by `wacom_*`) | Wacom drawing tablets — pressure + tilt + barrel |
| `wacom_sys.c` + `wacom_wac.c` | Wacom tablet driver |
| `hid-elan.c` | ELAN touchpads + touchscreens |
| `hid-uclogic-*.c` | UC-Logic / XP-Pen / Huion drawing tablets |
| `hid-corsair.c`, `hid-corsair-void.c` | Corsair keyboards + headsets |
| `hid-cougar.c` | Cougar gaming peripherals |
| `hid-glorious.c` | Glorious mice |
| `hid-jabra.c` | Jabra headsets (HID-call-control) |
| `hid-magicmouse.c` | Apple Magic Mouse + Magic Trackpad |
| `hid-multitouch.c` | All Win8+ touchscreens / touchpads |
| `hid-nti.c` | NTI KVMs |
| `hid-saitek.c`, `hid-thrustmaster.c` | Flight sticks / wheels |
| `hid-ntrig.c` | N-trig touchscreens (legacy) |
| `hid-prodikeys.c` | M-Audio MIDI controllers |
| `hid-redragon.c`, `hid-glorious.c` | Cheap gaming peripherals |
| `hid-cp2112.c` | Silicon Labs CP2112 USB-HID-to-SMBus bridge |
| `hid-mcp2221.c` | Microchip MCP2221 USB-HID-to-I2C/UART bridge |
| `hid-led.c` | Generic LED-via-HID dongles |
| `hid-led-rolling.c`, `hid-led-flash.c` | Specialized LED driver dongles |
| `hid-picolcd_*.c` | PicoLCD picoLCD device |
| `hid-pl.c`, `hid-zydacron.c`, `hid-zpff.c` | Force-feedback gamepads |
| `hid-google-*.c` | ChromeOS-specific HID quirks (Hammer, Whiskers, Stadia controller) |
| `hid-microsoft.c` | MS Surface keyboards |
| `hid-acrux.c`, `hid-axff.c`, `hid-bigbenff.c`, `hid-betopff.c`, `hid-dr.c`, `hid-emsff.c`, `hid-gembird.c`, `hid-gfrm.c`, `hid-gt683r.c`, `hid-holtekff.c`, `hid-keytouch.c`, `hid-kye.c`, `hid-lcpower.c`, `hid-lenovo.c`, `hid-letsketch.c`, `hid-magicmouse.c`, `hid-mf.c`, `hid-monterey.c`, `hid-multitouch.c`, `hid-nintendo.c`, `hid-nti.c`, `hid-petalynx.c`, `hid-pl.c`, `hid-plantronics.c`, `hid-playstation.c`, `hid-primax.c`, `hid-prodikeys.c`, `hid-pxrc.c`, `hid-quicksilver.c`, `hid-rmi.c` (Synaptics RMI4), `hid-roccat-*.c`, `hid-saitek.c`, `hid-samsung.c`, `hid-semitek.c`, `hid-sensor-*.c` (Sensor Hub generic), `hid-sigmamicro.c`, `hid-speedlink.c`, `hid-steam.c`, `hid-steelseries.c`, `hid-sunplus.c`, `hid-thrustmaster.c`, `hid-tivo.c`, `hid-topre.c`, `hid-topseed.c`, `hid-twinhan.c`, `hid-uclogic-*.c`, `hid-udraw-ps3.c`, `hid-viewsonic.c`, `hid-vivaldi*.c`, `hid-vrc2.c`, `hid-waltop.c`, `hid-wiimote-*.c` (Nintendo Wiimote), `hid-winwing.c`, `hid-xinmo.c`, `hid-zpff.c`, `hid-zydacron.c`

Compile-gated-off legacy / out-of-v0 candidates: drivers for SoC sensors not relevant to x86_64 (most ARM-only HID), some ancient PS/2-via-USB legacy.

### Sensor Hub framework (`hid-sensor-*.c`)

`hid-sensor-hub.c` + per-sensor wrappers (`hid-sensor-{accel-3d, gyro-3d, magn-3d, prox, als, press, temperature, custom-sensor, rotation, incl-3d, motion-detector, ...}`): generic HID Sensor Hub layer that exposes Intel ISH / AMD SFH / generic-USB-sensor-hub channels as IIO devices (`/sys/bus/iio/devices/iio:device<N>`). Used by iio-sensor-proxy + GNOME auto-rotate.

### scope

This Tier-2 governs:
- `/home/doll/linux-src/drivers/hid/` (~192 files + `usbhid/`, `i2c-hid/`, `intel-ish-hid/`, `amd-sfh-hid/`, `surface-hid/`, `bpf/` subdirs)
- `/home/doll/linux-src/net/bluetooth/hidp/` (Bluetooth HID Profile — physically under net/bluetooth but conceptually HID consumer)
- Public headers `include/linux/hid*.h` + `include/linux/hidraw.h`
- UAPI `include/uapi/linux/hidraw.h` + `include/uapi/linux/uhid.h`

`drivers/input/` (input-event subsystem proper — `evdev`, `joydev`, `mousedev`, `serio` for PS/2, force-feedback core consumed by HID) is a separate Tier-2 wrapper future. HID is a producer of input events; the input layer is the general-purpose consumer/dispatcher.

### compatibility contract — outline

### `/dev/hidraw<N>` chardev

`HIDIOCGRDESCSIZE`, `HIDIOCGRDESC`, `HIDIOCGRAWINFO`, `HIDIOCGRAWNAME`, `HIDIOCGRAWPHYS`, `HIDIOCGRAWUNIQ`, `HIDIOCSFEATURE`, `HIDIOCGFEATURE`, `HIDIOCSINPUT`, `HIDIOCGINPUT`, `HIDIOCSOUTPUT`, `HIDIOCGOUTPUT` IOCTLs byte-identical so libhidapi + libfido2 + OpenRGB + Solaar consume unchanged.

### `/dev/uhid` chardev

`UHID_*` IOCTLs + UHID protocol byte-identical so userspace HID emulators (Bluetooth-side bluetoothd HIDP impl, Steam-Input virtual controller, qmk_firmware-style remappers) consume unchanged.

### sysfs surface

Per-HID-device `/sys/bus/hid/devices/<bus>:<vid>:<pid>.<inst>/`:
- `country` (HID country code), `report_descriptor` (binary), `modalias`
- Per-class symlinks `/sys/class/{input, hidraw}/...`

Per-driver `/sys/bus/hid/drivers/<name>/`. Layout + content byte-identical so `udevadm` + `lsusb -v` + `lshid` consume unchanged.

### Modalias format

`hid:b<bus>v<vid>p<pid>` byte-identical for depmod/modprobe.

### HID Report Descriptor parsing

Per-HID-1.11 spec; per-driver Report Descriptor returned via HIDIOCGRDESC must match what the device sent (or the driver-applied fixup if any). Drivers may apply a `report_fixup()` callback to patch broken descriptors before parsing — fixup table identical to upstream.

### Force-feedback (FF)

`evdev` exposes FF via `EVIOCSFF` + `EVIOCRMFF`; per-driver FF effect implementation maps to vendor protocols. Behavior identical so SDL gamepad force-feedback works on Sony PS / MS Xbox / Logitech / Thrustmaster / Steam Deck.

### Sensor Hub IIO surface

`/sys/bus/iio/devices/iio:device<N>/{in_accel_*, in_anglvel_*, in_magn_*, in_intensity_*, in_proximity_*, in_pressure_*, in_temp_*}` byte-identical so `iio-sensor-proxy` consumes (auto-rotate works).

### Tracepoints

`events/hid/`: `hid_report_*`, `hid_input_event` byte-identical names + format.

### HID-BPF

CONFIG_HID_BPF=y. BPF prog types `BPF_PROG_TYPE_HID_FIX_REPORT` + `BPF_PROG_TYPE_HID_DEVICE_EVENT` attached via libbpf + bpf_link API. Consumed by udev-hid-bpf rule library. UAPI byte-identical.

### tier-3 docs governed by this tier-2

(Phase D will add these incrementally.)

| Tier-3 doc | Scope |
|---|---|
| `drivers/hid/hid-core.md` | `hid-core.c` + `hid-debug.c`: Report Descriptor parser + hid_driver bus + per-device probe + report dispatch |
| `drivers/hid/hid-input.md` | `hid-input.c`: HID-usage → Linux input-event mapping |
| `drivers/hid/hid-generic.md` | `hid-generic.c`: catchall driver |
| `drivers/hid/hidraw.md` | `hidraw.c`: `/dev/hidraw<N>` chardev passthrough |
| `drivers/hid/uhid.md` | `uhid.c`: `/dev/uhid` userspace emulation |
| `drivers/hid/hid-quirks.md` | `hid-quirks.c`: per-device quirks table |
| `drivers/hid/usbhid.md` | `usbhid/`: USB transport for HID + PID FF |
| `drivers/hid/i2c-hid.md` | `i2c-hid/`: I2C transport (laptop touchpad + touchscreen) |
| `drivers/hid/intel-ish-hid.md` | `intel-ish-hid/`: Intel Integrated Sensor Hub |
| `drivers/hid/amd-sfh-hid.md` | `amd-sfh-hid/`: AMD Sensor Fusion Hub |
| `drivers/hid/surface-hid.md` | `surface-hid/`: MS Surface Aggregator |
| `drivers/hid/hid-bpf.md` | `bpf/`: HID-BPF runtime |
| `drivers/hid/sensor-hub.md` | `hid-sensor-*.c`: HID Sensor Hub IIO bridge |
| `drivers/hid/wacom.md` | `wacom_sys.c` + `wacom_wac.c`: Wacom tablets |
| `drivers/hid/logitech-dj.md` | `hid-logitech-dj.c` + `hid-logitech-hidpp.c`: Unifying + HID++ |
| `drivers/hid/multitouch.md` | `hid-multitouch.c`: Win8+ multi-touch |
| `drivers/hid/playstation.md` | `hid-playstation.c` + `hid-sony.c`: PS3/PS4/PS5 controllers |
| `drivers/hid/microsoft.md` | `hid-microsoft.c`: MS keyboards + Xbox controllers |
| `drivers/hid/apple.md` | `hid-apple.c` + `hid-magicmouse.c` + `hid-appletb-*` + `hid-appleir.c`: Apple peripherals |
| `drivers/hid/nintendo.md` | `hid-nintendo.c` + `hid-wiimote-*.c`: Switch + Wiimote |
| `drivers/hid/steam.md` | `hid-steam.c`: Steam Controller / Steam Deck |
| `drivers/hid/uclogic.md` | `hid-uclogic-*.c`: UC-Logic / XP-Pen / Huion drawing tablets |
| `drivers/hid/lenovo.md` | `hid-lenovo.c`: ThinkPad keyboards |
| `drivers/hid/asus.md` | `hid-asus.c`: ASUS laptops + ROG |
| `drivers/hid/rmi.md` | `hid-rmi.c`: Synaptics RMI4 over HID |
| `drivers/hid/cp2112-mcp2221.md` | `hid-cp2112.c` + `hid-mcp2221.c`: USB-HID-to-SMBus/I2C bridges |
| `drivers/hid/google-quirks.md` | `hid-google-*.c`: ChromeOS quirks |
| `drivers/hid/bluetooth-hidp.md` | `net/bluetooth/hidp/`: Bluetooth HID Profile |

### compatibility outline (top-level)

- REQ-O1: `/dev/hidraw<N>` IOCTLs byte-identical (libhidapi + libfido2 + OpenRGB + Solaar consume unchanged).
- REQ-O2: `/dev/uhid` IOCTLs + protocol byte-identical (Bluetooth HIDP daemon, Steam Input, virtual gamepads work unchanged).
- REQ-O3: HID Report Descriptor parser produces identical input-event mapping vs upstream baseline (per-device input behavior preserved).
- REQ-O4: Per-vendor / per-device drivers produce identical input events + force-feedback as upstream baseline (SDL gamepad mapping table works).
- REQ-O5: Modalias `hid:b<bus>v<vid>p<pid>` format byte-identical (depmod/modprobe load right driver).
- REQ-O6: HID Sensor Hub bridges to IIO with byte-identical sysfs surface (iio-sensor-proxy works).
- REQ-O7: HID-BPF UAPI byte-identical (udev-hid-bpf rule library works).
- REQ-O8: USB HID + I2C HID + Bluetooth HID + Intel ISH + AMD SFH + MS Surface aggregator HID transports all enumerate + work on respective HW.
- REQ-O9: Force-feedback effects (rumble, constant, ramp, spring, friction, damper, periodic, custom) implemented identically per-vendor.
- REQ-O10: TLA+ models declared at this Tier-2 (Report Descriptor parse termination + bounds, hidraw read/write race-freedom, uhid request-response state machine, HID-BPF prog attach/detach race-freedom).
- REQ-O11: Verus/Creusot Layer-4 functional contracts on Report Descriptor parser bounds + per-report buffer arithmetic + uhid message validation.
- REQ-O12: Hardening: row-1 features applied per `00-security-principles.md`, with HID-specific reinforcement (hidraw write CAP_SYS_RAWIO, uhid create CAP_SYS_ADMIN, BadUSB-class HID authorization integration).

### acceptance criteria (top-level)

- [ ] AC-O1: USB keyboard + mouse + game controller enumerate; `evtest /dev/input/event<N>` shows correct events for each. (covers REQ-O3, REQ-O8)
- [ ] AC-O2: Touchpad multi-touch test on reference laptop: GNOME multi-touch gestures (3-finger swipe, pinch-zoom) work. (covers REQ-O3, REQ-O8)
- [ ] AC-O3: SDL2 gamepad test: PS5 DualSense, Xbox Series X, Steam Deck controller all map correctly via SDL_GameController API. (covers REQ-O4, REQ-O9)
- [ ] AC-O4: FIDO2 test: hardware FIDO key via libfido2 + Chrome U2F flow works. (covers REQ-O1)
- [ ] AC-O5: Wacom tablet test: pressure + tilt + barrel-rotation events delivered with correct ranges. (covers REQ-O3, REQ-O4)
- [ ] AC-O6: Auto-rotate test on laptop with HID Sensor Hub: rotating physical device rotates display via iio-sensor-proxy. (covers REQ-O6, REQ-O8)
- [ ] AC-O7: Bluetooth HID test: pair Bluetooth keyboard via bluetoothctl; key-press registers in OS. (covers REQ-O8)
- [ ] AC-O8: HID-BPF test: load `udev-hid-bpf` rule fixing a known broken-descriptor device; behavior corrected at probe time. (covers REQ-O7)
- [ ] AC-O9: uhid test: virtual gamepad emulator creates `/dev/input/event<N>` that other apps consume. (covers REQ-O2)
- [ ] AC-O10: kselftest `tools/testing/selftests/hid/` passes. (covers REQ-O10, REQ-O11)
- [ ] AC-O11: BadUSB defense test: rogue USB device claiming HID + keyboard simultaneously rejected by `authorized_default=0` policy from drivers/usb. (covers REQ-O12)
- [ ] AC-O12: Logitech Unifying receiver: 6 connected devices each enumerate as separate hidraw/input-event endpoints. (covers REQ-O4, REQ-O8)

### verification (top-level)

### Layer 2: TLA+ models — mandatory list

| Model | Owned by |
|---|---|
| `models/hid/report_descriptor_parse.tla` | `drivers/hid/hid-core.md` (proves: HID Report Descriptor parser termination — for any input descriptor of length N, parser visits at most O(N) descriptor items; collection-stack depth bounded; Push/Pop semantics correct under nested collections) |
| `models/hid/hidraw_rw.tla` | `drivers/hid/hidraw.md` (proves: hidraw chardev concurrent read + write + device-disconnect — read on disconnected device returns -ENODEV cleanly; write while disconnect-in-flight either succeeds or fails with -ENODEV, never UAF) |
| `models/hid/uhid_proto.tla` | `drivers/hid/uhid.md` (proves: uhid request-response state machine — userspace CREATE → DESTROY lifecycle; concurrent INPUT report from userspace + GET_REPORT request from kernel client never deadlock) |
| `models/hid/hidbpf_attach.tla` | `drivers/hid/hid-bpf.md` (proves: HID-BPF prog attach/detach + concurrent device probe/disconnect; prog refcount integrity; detach during in-flight Report invocation safely waits for completion) |
| `models/hid/usbhid_urb.tla` | `drivers/hid/usbhid.md` (proves: usbhid URB submission + completion + LED-output coalescing under concurrent input and output reports; no leaked URB on device disconnect) |

### Layer 4: Functional contracts (Verus / Creusot)

| Component | Contract topic |
|---|---|
| `drivers/hid/hid-core.md` | Report Descriptor parser bounds: every item parse advances ≥ 1 byte AND ≤ remaining bytes; descriptor item-size validated against descriptor-end before deref |
| `drivers/hid/hidraw.md` | hidraw read invariant: returned bytes ≤ user-supplied buffer size; per-device input queue refcount maintained across open/close |
| `drivers/hid/uhid.md` | uhid event validation: every userspace-supplied uhid_event has type in valid range + per-type payload size matches union member; no oversized read past union |

### hardening (top-level)

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this Tier-2

| Feature | Default inherited from |
|---|---|
| **REFCOUNT** | per-hid_device + per-hidraw_list + per-uhid_device + per-bus-driver refcounts use `Refcount` (saturating) | § Mandatory |
| **CONSTIFY** | per-driver `hid_driver`, per-bus `hid_ll_driver`, per-vendor quirk tables `static const` | § Mandatory |
| **SIZE_OVERFLOW** | report-buffer + report-descriptor item-count + per-collection nested-depth arithmetic uses checked operators | § Mandatory |
| **RAP / FORTIFY_RW** | per-driver hid_driver + per-bus hid_ll_driver vtables + quirks table read-only-after-init | § Mandatory |
| **MEMORY_SANITIZE** | freed hid_device priv data cleared (carries FIDO key challenges, smartcard PIN buffers via CCID-over-HID) | § Default-on configurable off |
| **STRICT_KERNEL_RWX** | HID has no JIT; HID-BPF programs JIT'd via core BPF JIT (W^X enforced there) | § Mandatory |

### Row-2 (LSM-stackable) features for HID

LSM hooks called: file-LSM hooks on `/dev/hidraw*` open + ioctl + read/write, `/dev/uhid` open + ioctl + read/write. `security_hidraw_*` family (proposed for Rookery — upstream coverage minimal).

GR-RBAC adds:
- Per-role disallow `/dev/hidraw*` open (denies raw HID write — defends against firmware-flash via OpenRGB-style tools).
- Per-role disallow `/dev/uhid` open (denies userspace HID injection — defends against malicious virtual keyboard).
- Per-role audit of every uhid CREATE (which uid created which virtual device).
- Per-role disallow HID-BPF prog attach (defends against userspace-supplied HID logic).

### HID-specific reinforcement

- **BadUSB defense via drivers/usb authorization**: `authorized_default=0` for HID class on hot-plug USB devices (defense against rogue USB device claiming HID + injecting keystrokes); chassis-internal HID auto-authorized via ACPI _PLD location info. Cross-ref `drivers/usb/00-overview.md` § Hardening.
- **hidraw write CAP_SYS_RAWIO**: writes to `/dev/hidraw*` require CAP_SYS_RAWIO — these writes can program LEDs, send HID feature reports that re-flash device firmware. Read access (input reports) requires only normal device permissions.
- **uhid CREATE requires CAP_SYS_ADMIN**: uhid registers a virtual HID device that injects events into the kernel input subsystem; requires CAP_SYS_ADMIN to prevent unprivileged process from forging keyboard input.
- **Report Descriptor sanitization**: descriptor-fixup applied before parsing so malformed descriptors from buggy/malicious devices don't crash parser. Layer-2 `report_descriptor_parse.tla` is the proof of bounded-time termination.
- **Per-vendor LED-write CAP_SYS_RAWIO**: per-vendor drivers exposing programmable LEDs (Razer, Corsair, etc.) gate LED writes on CAP_SYS_RAWIO via sysfs LED class.
- **HID-BPF prog attach CAP_BPF + CAP_NET_ADMIN**: HID-BPF programs can transform HID reports + spoof input events; require both CAPs to prevent untrusted users from attaching.
- **Sensor Hub LSM mediation**: HID Sensor Hub IIO exposure mediated by file-LSM hooks on `/sys/bus/iio/devices/iio:device<N>/in_*` reads (some sensors leak proximity / orientation that could enable keylogging via accelerometer).
- **Bluetooth HID security**: cross-ref `net/bluetooth/00-overview.md` for Bluetooth-side LE-Secure-Connection requirement; Rookery enforces Just-Works pairing rejection for HID profile by default (Bluetooth Low Energy keyboard requires SSP+OOB or numeric-comparison pairing).

