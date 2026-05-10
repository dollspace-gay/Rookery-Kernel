---
title: "Tier-2: drivers/input — Input subsystem (input core + evdev/joydev/mousedev/uinput + serio i8042 + force-feedback + per-class keyboards/mice/touchpads/joysticks/touchscreens/tablets/misc)"
tags: ["tier-2", "drivers", "input", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 overview for the Linux input subsystem — the layer between every device that produces "the user did something" events (keyboard keypress, mouse motion, touchpad gesture, joystick deflection, touchscreen tap, drawing-tablet pressure, gas-pedal absolute-axis, power-button press, lid-close, headphone-jack-detect, ambient-light-sensor threshold) and every userspace consumer (X11 evdev driver, Wayland compositors, libinput, SDL2 gamepad, Steam Input, GNOME/KDE input dialogs, console keymaps, accessibility tools).

Sits as the consumer of HID (cross-ref `drivers/hid/00-overview.md` — HID-class is one of the largest input producers) and as the producer for everything userspace. Six concentric layers:

1. **Input core** (`input.c` + `input-mt.c` + `input-poller.c` + `input-compat.c` + `input-leds.c` + `input-core-private.h` + `ff-core.c` + `ff-memless.c` + `matrix-keymap.c` + `sparse-keymap.c` + `touchscreen.c` + `touch-overlay.c` + `vivaldi-fmap.c` + `apm-power.c`): `input_dev` lifecycle, event-coalesce + sync, MT-protocol B (multi-touch slot tracking), keymap framework, force-feedback core (memless effect synthesizer), input-LEDs (per-input-device LEDs like CapsLock + NumLock + ScrollLock + Compose), input-poller (poll-based input devices), 32-bit compat layer
2. **Userspace chardev interfaces**:
   - **evdev** (`evdev.c`): `/dev/input/event<N>` — the modern unified chardev for every input device; consumed by libinput, X11 evdev, Wayland compositors
   - **joydev** (`joydev.c`): `/dev/input/js<N>` — legacy joystick chardev (still consumed by SDL1, older games)
   - **mousedev** (`mousedev.c`): `/dev/input/mouse<N>` + `/dev/input/mice` — legacy PS/2-mouse-like chardev (X11 fallback, console gpm)
   - **uinput** (`misc/uinput.c`): `/dev/uinput` — userspace virtual input-device creation (consumed by Steam Input, virtual gamepads, Wayland virtual-keyboards, accessibility tools)
3. **Bus drivers** (transports providing input devices):
   - **serio** (`serio/`): serial-input bus — `i8042` (legacy PS/2 Keyboard + Mouse via Intel 8042 KBC; the i8042 ACPI-PNP path is the dominant x86_64 KBC entry); also `serport.c` (UART-as-serio for serial-mice + serial-touchscreens), `userio.c` (userspace serio emulator), `serio_raw.c` (raw chardev for serio-bus passthrough), `parkbd.c` (parport keyboard), `pcips2.c` (PCI PS/2), `serio.c` (bus core), `libps2.c` (PS/2 protocol helper), `hyperv-keyboard.c` (Hyper-V synthetic keyboard)
   - **gameport** (`gameport/`): legacy ISA gameport bus (joysticks); mostly dead but kept for legacy ISA + PCI-ISA bridge support
4. **Per-class drivers**:
   - **keyboard/** (`keyboard/`): atkbd (the dominant AT/PS/2 keyboard driver), gpio_keys, gpio_keys_polled, snvs_pwrkey, soc_button_array, cros_ec_keyb, dlink_dir685_touchkeys, hilkbd, locomokbd, lkkbd, lpc32xx-keys, matrix_keypad, max7359_keypad, mcs_touchkey, mtk-pmic-keys, mt6779-keypad, opencores-kbd, qt1050, qt1070, qt2160, samsung-keypad, sh_keysc, snvs_pwrkey, spear-keyboard, st-keyscan, stowaway, sun4i-lradc-keys, sunkbd, tca6416-keypad, tegra-kbc, tm2-touchkey, twl4030_keypad, xtkbd, mpr121_touchkey, …
   - **mouse/** (`mouse/`): synaptics (the dominant Synaptics PS/2 touchpad driver), elantech, alps (ALPS touchpad), cypress_ps2, byd, focaltech, trackpoint, navpoint, sermouse, vsxxxaa, gpio_mouse, lifebook, sentelic, logips2pp, amimouse, atarimouse, hil_ptr, inport, logibm, maplemouse, pc110pad, pxa930_trkball, rpcmouse, sermouse, vsxxxaa, …
   - **joystick/** (`joystick/`): xpad (Xbox 360/One/Series controller — wired + wireless), iforce/, fsia6b, twidjoy, db9, gamecon, turbografx, magellan, spaceball, spaceorb, stinger, twidjoy, walkera0701, warrior, sidewinder, gf2k, joydump, magellan, adi, analog, a3d, cobra, grip, gravsignal, microsoftpad, …
   - **touchscreen/** (`touchscreen/`): ~150 drivers covering the entire industry — ads7846, ar1021_i2c, atmel_mxt_ts, auo-pixcir-ts, bu21013_ts, bu21029_ts, chipone_icn8318, chipone_icn8505, cy8ctma140, cy8ctmg110_ts, cyttsp4_*, cyttsp5, cyttsp_*, edt-ft5x06, eeti_ts, egalax_ts, ektf2127, elants_i2c, elo, exc3000, fts5sx, fujitsu_ts, gnav_ts, goodix-berlin, goodix, gunze, hampshire, himax_hx83112b, hycon-hy46xx, hyperv_input, ili210x, ilitek_ts_i2c, imagis, imx_sc_thermal, inexio, ipaq-micro-ts, iqs5xx, iqs7211, lpc32xx_ts, mainstone-wm97xx, max11801_ts, mc13783_ts, mcs5000_ts, melfas_mip4, migor_ts, mms114, msg2638, mtouch, novatek-nvt-ts, of_touchscreen, pixcir_i2c_ts, raspberrypi-ts, raydium_i2c_ts, resistive-adc-touch, rohm_bu21023, s6sy761, silead, sis_i2c, st1232, stmpe-ts, sun4i-ts, sur40, surface3_spi, tca8418_keypad, tps6507x-ts, ts4800-ts, tsc200x-core, tsc2004, tsc2005, tsc2007_*, ttsp, ucb1400_ts, usbtouchscreen, wacom_i2c, wacom_w8001, wdt87xx_i2c, wm831x-ts, wm97xx-core, …
   - **tablet/** (`tablet/`): drawing tablets (most now via HID — these are legacy non-HID): aiptek, gtco, hanwang, kbtab, pegasus_notetaker, wacom_serial4
   - **misc/** (`misc/`): grab-bag — adc-keys, ad714x, ad714x-i2c, ad714x-spi, adxl34x, apanel, atc260x-onkey, atlas_btns, atmel_captouch, axp20x-pek, bma150, cm109, cma3000_d0x, cma3000_d0x_i2c, da7280, da9052_onkey, da9055_onkey, da9063_onkey, drv260x (haptic motor driver), drv2665, drv2667, dw_imx_thermal, gp2ap002a00f, gpio-beeper, gpio_decoder, hisi_powerkey, hp_sdc_rtc, ideapad_slidebar, iqs269a, iqs626a, iqs7222, kxtj9, m68kspkr, mantis_pad, max8925_onkey, mc13783-pwrbutton, mma8450, msm-vibrator, palmas-pwrbutton, pcap_keys, pcf50633-input, pcf8574_keypad, pcspkr (PC speaker), powermate (Griffin PowerMate USB knob), pwm-beeper, pwm-vibra, regulator-haptic, retu-pwrbutton, rk805-pwrkey, rotary_encoder, sc27xx-vibra, sgi_btns, soc_button_array, sparcspkr, stpmic1_onkey, tps65218-pwrbutton, twl4030-pwrbutton, twl4030-vibra, twl6040-vibra, uinput, vl53l0x, wistron_btns, wm831x-on, xen-kbdfront, yealink, …
   - **rmi4/** (`rmi4/`): Synaptics RMI4 protocol (HID + I2C + SMBus + SPI transports) — touchpads + touchscreens
5. **Helpers**: matrix-keymap (matrix-keypad drivers), sparse-keymap (lookup keymap by scancode), touchscreen.c (common touchscreen helpers), touch-overlay.c (touch-overlay-on-fbcon), vivaldi-fmap.c (ChromeOS Vivaldi function-row keymap)
6. **tests** (`tests/`): KUnit tests

Heavily cross-referenced from `drivers/hid/00-overview.md` (HID is the dominant input producer; usbhid + i2c-hid emit input events via this subsystem), `drivers/usb/00-overview.md` (USB transport for usbhid; xpad direct-USB), `drivers/i2c/` (I2C transport for many touchscreens), `drivers/spi/` (SPI transport for some touchscreens), `arch/x86/idt.md` (PS/2 KBC i8042 IRQ delivery via IDT), `kernel/cgroup/` (per-cgroup input-device delegation via /dev/input/), `drivers/iio/` (sensor-hub gyroscope/accelerometer for auto-rotate; cross-ref HID Sensor Hub).

### Out of Scope

- Per-Tier-3 (Phase D) — per-driver docs arrive incrementally
- Console kbd / VT layer (`drivers/tty/vt/keyboard.c` + `vt_ioctl.c`) — owned by future TTY Tier-2 wrapper; cross-referenced here as input consumer
- ARM-only SoC input drivers (most SoC-specific keypads, touchscreens, vibrators) — keep present but compile-gated off for v0
- 32-bit-only paths
- Implementation code

### components

(See Summary for the per-subdir breakdown.)

### scope

This Tier-2 governs `/home/doll/linux-src/drivers/input/` (~30 top-level files + `serio/`, `gameport/`, `keyboard/`, `mouse/`, `joystick/`, `touchscreen/`, `tablet/`, `misc/`, `rmi4/`, `tests/` subdirs — ~600+ files total), public headers `include/linux/input*.h` + `include/linux/serio.h`, UAPI `include/uapi/linux/input.h` + `include/uapi/linux/input-event-codes.h` + `include/uapi/linux/uinput.h` + `include/uapi/linux/joystick.h`.

### compatibility contract — outline

### `/dev/input/event<N>` chardev

The modern unified input device. `EVIOC*` IOCTLs byte-identical:
- `EVIOCGVERSION`, `_GID`, `_GNAME`, `_GPHYS`, `_GUNIQ`, `_GPROP`, `_GMTSLOTS`, `_GKEY`, `_GLED`, `_GSND`, `_GSW`, `_GBIT`, `_GABS`, `_SABS`, `_GREP`, `_SREP`, `_GKEYCODE`, `_GKEYCODE_V2`, `_SKEYCODE`, `_SKEYCODE_V2`, `_SFF`, `_RMFF`, `_GEFFECTS`, `_GRAB`, `_REVOKE`, `_SCLOCKID`, `_SMASK`, `_GMASK`, `EVIOCSKEYCODE_V2`

Wire format byte-identical including 32-bit timeval/timespec compat (`input_event` struct's `time` field): on 64-bit timeval kernel, evdev exposes 32-bit-compat events for legacy-built apps via `EVIOCSCLOCKID` + 32-bit timeval struct variant. Behavior identical so libevdev + libinput + X11 evdev + Wayland compositors consume unchanged.

### `/dev/input/js<N>` chardev

Legacy joystick API: `JSIOCGVERSION`, `_GAXES`, `_GBUTTONS`, `_GNAME`, `_SBTNMAP`, `_GBTNMAP`, `_SAXMAP`, `_GAXMAP`, `_SCORR`, `_GCORR`. Read-event format `struct js_event`. Wire format byte-identical so SDL1 + jstest-gtk + older Linux ports of games consume unchanged.

### `/dev/input/mouse<N>` + `/dev/input/mice` chardev

Legacy PS/2-mouse-emulation chardev (intermixed-mouse or per-mouse). 3-byte (or 4-byte with wheel) packets in IMPS/2 protocol. Consumed by X11 mouse driver fallback + console-mouse `gpm`. Wire format byte-identical.

### `/dev/uinput` chardev

`UI_DEV_CREATE`, `_DEV_DESTROY`, `_DEV_SETUP`, `_ABS_SETUP`, `_SET_*BIT` (event/key/rel/abs/msc/sw/led/snd/ff), `_BEGIN_FF_UPLOAD`, `_END_FF_UPLOAD`, `_BEGIN_FF_ERASE`, `_END_FF_ERASE`, `_GET_VERSION`, `_GET_SYSNAME` IOCTLs byte-identical so virtual-input creation (Steam Input, virtual keyboards, Wayland virtual-keyboard-protocol implementations) work unchanged.

### `/sys/class/input/{event,js,mouse}<N>/` + `/sys/class/input/input<N>/`

Per-input-device sysfs:
- `name`, `phys`, `uniq`, `properties`, `modalias`, `id/{bustype,vendor,product,version}`, `capabilities/{ev,key,rel,abs,msc,led,snd,ff,sw}`
- `device/{name,phys,uniq,properties,driver,subsystem}`

Layout + content byte-identical so `udevadm info /dev/input/event0` + `evtest` + `libinput list-devices` consume unchanged.

### Modalias format

`input:b<bus>v<vid>p<pid>e<ver>-{ev,key,rel,abs,msc,led,snd,ff,sw}<bitmap>` byte-identical for depmod/modprobe.

### Multi-touch protocol

MT-protocol-B (slot-based, the modern protocol used by every Win8+ touchscreen + every modern touchpad) with `ABS_MT_SLOT` + `ABS_MT_TRACKING_ID` + `ABS_MT_POSITION_X/Y` + `ABS_MT_TOUCH_MAJOR/MINOR` + `ABS_MT_PRESSURE` + `ABS_MT_DISTANCE` + `ABS_MT_ORIENTATION` + `ABS_MT_TOOL_TYPE` + `ABS_MT_TOOL_X/Y` + `ABS_MT_BLOB_ID`. Behavior identical so libinput multi-touch gestures work.

### Force-feedback

`EVIOCSFF` uploads an effect (`struct ff_effect` — types FF_RUMBLE / FF_PERIODIC / FF_CONSTANT / FF_SPRING / FF_FRICTION / FF_DAMPER / FF_INERTIA / FF_RAMP); `EV_FF` event with `code=effect_id` plays. Per-driver implementation; behavior identical so SDL gamepad rumble, Wine gamepad FF, racing-wheel FF work unchanged.

### Repeat rate / delay

`EVIOCSREP` sets per-device repeat rate + delay. Console kbd uses `KDSETKEYCODE` from `drivers/tty/vt/`; evdev uses EVIOCSREP. Default 250ms delay + 33ms (30Hz) repeat preserved.

### Console keymap

Console virtual-terminal kbd layer (`drivers/tty/vt/keyboard.c` + `vt_ioctl.c` — outside this Tier-2's drivers/input scope but cross-referenced) consumes input-events from /dev/input/event<N>-equivalent kernel-internal channel + applies VT keymap (`loadkeys` userspace). Behavior identical.

### `/proc/bus/input/devices` + `/proc/bus/input/handlers`

Legacy procfs:
- `/proc/bus/input/devices` enumerates every input device with bus / vendor / product / version / name / phys / sysfs / uniq / handlers / EV / KEY / REL / ABS / MSC / LED / SND / FF / SW bitmaps
- `/proc/bus/input/handlers` enumerates registered handlers (kbd, mousedev, evdev, joydev, …)

Format byte-identical.

### LED triggers

Per-input-device LEDs (CapsLock / NumLock / ScrollLock) registered with LED-trigger framework so they can be re-routed to chassis LEDs. Wire identical via `/sys/class/leds/.../trigger`.

### Input cgroup delegation

Per-cgroup `/dev/input/event*` access controlled via device cgroup-v1 / cgroup-v2 device controller. Cross-ref `kernel/cgroup/devices.md`.

### `/sys/devices/.../i8042/` topology

Legacy KBC at `/sys/devices/platform/i8042/serio<N>/` — KBD + AUX (mouse) channels. Sysfs `/sys/devices/platform/i8042/serio0/{description,driver,modalias,subsystem}`. Behavior identical.

### tier-3 docs governed by this tier-2

(Phase D will add these incrementally.)

| Tier-3 doc | Scope |
|---|---|
| `drivers/input/input-core.md` | `input.c` + `input-mt.c` + `input-compat.c` + `input-poller.c` + `input-leds.c`: input_dev lifecycle + event coalescing + MT-protocol-B + 32-bit compat |
| `drivers/input/input-ff.md` | `ff-core.c` + `ff-memless.c`: force-feedback core + memless effect synthesizer |
| `drivers/input/input-keymap.md` | `matrix-keymap.c` + `sparse-keymap.c` + `vivaldi-fmap.c`: keymap helpers |
| `drivers/input/input-touchscreen-helpers.md` | `touchscreen.c` + `touch-overlay.c`: touchscreen common helpers |
| `drivers/input/evdev.md` | `evdev.c`: `/dev/input/event<N>` chardev |
| `drivers/input/joydev.md` | `joydev.c`: `/dev/input/js<N>` chardev (legacy) |
| `drivers/input/mousedev.md` | `mousedev.c`: `/dev/input/mouse<N>` + `/dev/input/mice` (legacy) |
| `drivers/input/uinput.md` | `misc/uinput.c`: `/dev/uinput` userspace virtual-input creation |
| `drivers/input/serio-bus.md` | `serio/serio.c` + `serio_raw.c` + `userio.c`: serio bus core + raw passthrough + userspace serio emulator |
| `drivers/input/serio-i8042.md` | `serio/i8042*.c` + `serio/i8042-acpipnpio.h` + `serio/libps2.c`: i8042 KBC + ACPI-PNP enumeration + PS/2 protocol helper |
| `drivers/input/serio-serport.md` | `serio/serport.c`: UART-as-serio (serial mice + touchscreens) |
| `drivers/input/serio-hyperv.md` | `serio/hyperv-keyboard.c`: Hyper-V synthetic keyboard |
| `drivers/input/gameport.md` | `gameport/`: legacy ISA gameport bus |
| `drivers/input/keyboard-atkbd.md` | `keyboard/atkbd.c`: AT/PS/2 keyboard (the dominant) |
| `drivers/input/keyboard-misc.md` | `keyboard/{gpio_keys, gpio_keys_polled, soc_button_array, cros_ec_keyb, matrix_keypad, ...}` |
| `drivers/input/mouse-synaptics.md` | `mouse/synaptics.c`: Synaptics PS/2 touchpad |
| `drivers/input/mouse-alps-elantech.md` | `mouse/{alps, elantech}.c`: ALPS + Elantech touchpads |
| `drivers/input/mouse-misc.md` | `mouse/{trackpoint, sentelic, byd, focaltech, sermouse, ...}.c` |
| `drivers/input/joystick-xpad.md` | `joystick/xpad.c`: Xbox controller direct-USB |
| `drivers/input/touchscreen-elants.md` | `touchscreen/elants_i2c.c`: ELAN touchscreens |
| `drivers/input/touchscreen-goodix.md` | `touchscreen/{goodix, goodix-berlin}.c`: Goodix touchscreens |
| `drivers/input/touchscreen-atmel.md` | `touchscreen/atmel_mxt_ts.c`: Atmel maxTouch |
| `drivers/input/touchscreen-edt.md` | `touchscreen/edt-ft5x06.c`: EDT FT5x06 |
| `drivers/input/touchscreen-misc.md` | `touchscreen/{ads7846, cyttsp*, raspberrypi-ts, surface3_spi, ucb1400_ts, ...}.c` |
| `drivers/input/tablet.md` | `tablet/`: legacy non-HID drawing tablets |
| `drivers/input/rmi4.md` | `rmi4/`: Synaptics RMI4 protocol (HID + I2C + SMBus + SPI) |
| `drivers/input/misc-haptic.md` | `misc/{drv260x, drv2665, drv2667, da7280, palmas-haptic, regulator-haptic, ...}.c` |
| `drivers/input/misc-pwrkeys.md` | `misc/{snvs_pwrkey, axp20x-pek, da9052/55_onkey, mc13783-pwrbutton, palmas-pwrbutton, ...}.c` |
| `drivers/input/misc-rotary.md` | `misc/{rotary_encoder, gpio_decoder, powermate}.c` |
| `drivers/input/misc-pcspkr.md` | `misc/{pcspkr, gpio-beeper, pwm-beeper, sparcspkr, m68kspkr}.c`: PC speaker + beepers |

### compatibility outline (top-level)

- REQ-O1: `/dev/input/event<N>` evdev IOCTLs + event wire format byte-identical (libevdev + libinput + X11 evdev + Wayland compositors consume unchanged).
- REQ-O2: `/dev/input/js<N>` joydev IOCTLs + event wire format byte-identical (SDL1 / jstest consume unchanged).
- REQ-O3: `/dev/input/mouse<N>` + `/dev/input/mice` mousedev IMPS/2 protocol byte-identical.
- REQ-O4: `/dev/uinput` IOCTLs byte-identical (Steam Input + virtual-keyboard implementations consume unchanged).
- REQ-O5: `/sys/class/input/...` sysfs surface byte-identical.
- REQ-O6: `/proc/bus/input/{devices,handlers}` legacy procfs format byte-identical.
- REQ-O7: Modalias format `input:b<bus>v<vid>p<pid>e<ver>-{ev,key,rel,abs,msc,led,snd,ff,sw}<bitmap>` byte-identical (depmod/modprobe load right driver).
- REQ-O8: MT-protocol-B (slot-based multi-touch) per-event semantics byte-identical (libinput multi-touch gestures work).
- REQ-O9: Force-feedback effect types + upload/erase/play wire format byte-identical (SDL gamepad rumble, racing-wheel FF work).
- REQ-O10: i8042 KBC + ACPI-PNP enumeration + PS/2 keyboard + PS/2 mouse work on every BIOS that exposes them.
- REQ-O11: Console keymap pipeline (drivers/tty/vt/ consumer) sees input-events identically.
- REQ-O12: TLA+ models declared at this Tier-2 (input_dev event coalesce + sync, evdev per-fd mask + read race, MT-slot-tracking-ID lifecycle, force-feedback effect-upload + concurrent-play, uinput create/destroy + concurrent UI_BEGIN_FF_UPLOAD).
- REQ-O13: Verus/Creusot Layer-4 functional contracts on input_event time field 64/32-bit consistency, MT-slot tracking-ID monotonicity, FF-effect-id allocation.
- REQ-O14: Hardening: row-1 features applied per `00-security-principles.md`, with input-specific reinforcement (uinput CAP_SYS_ADMIN, evdev EVIOCGRAB mediation for keyloggers, console keymap LSM mediation, BadUSB-keyboard authorization integration with drivers/usb).

### acceptance criteria (top-level)

- [ ] AC-O1: `evtest /dev/input/event0` shows correct events for keyboard / mouse / touchpad / touchscreen / gamepad on a reference machine. (covers REQ-O1)
- [ ] AC-O2: `jstest /dev/input/js0` works for SDL1-based games on a USB joystick. (covers REQ-O2)
- [ ] AC-O3: `xinput list` + `libinput list-devices` enumerate every input device with byte-identical descriptor strings vs upstream baseline. (covers REQ-O5, REQ-O7)
- [ ] AC-O4: GNOME Wayland session: keyboard + touchpad + touchscreen multi-touch gestures work. (covers REQ-O1, REQ-O8)
- [ ] AC-O5: Steam Input virtual gamepad: Steam creates virtual `/dev/input/event<N>` via uinput; SDL2-game sees the virtual gamepad. (covers REQ-O4)
- [ ] AC-O6: Xbox One controller (xpad): rumble + button + analog-stick + dpad events all work; SDL2 GameController API maps correctly. (covers REQ-O1, REQ-O9)
- [ ] AC-O7: Racing-wheel force-feedback: SDL FF spring effect uploaded + plays correctly on a Logitech G29. (covers REQ-O9)
- [ ] AC-O8: PS/2 keyboard via i8042 ACPI-PNP works on a reference legacy laptop without USB-keyboard. (covers REQ-O10)
- [ ] AC-O9: Console kbd `loadkeys us` + `showkey -a` work on text-mode VT. (covers REQ-O11)
- [ ] AC-O10: kselftest `tools/testing/selftests/drivers/input/` passes. (covers REQ-O12)
- [ ] AC-O11: BadUSB-keyboard defense: rogue USB device claiming HID-keyboard rejected by `authorized_default=0` policy from drivers/usb. (covers REQ-O14)
- [ ] AC-O12: PowerMate USB knob: rotation events delivered via evdev with correct REL_DIAL semantics. (covers REQ-O1)

### verification (top-level)

### Layer 2: TLA+ models — mandatory list

| Model | Owned by |
|---|---|
| `models/input/input_event_sync.tla` | `drivers/input/input-core.md` (proves: input_dev event coalescing + EV_SYN delivery — concurrent input_event from IRQ context + workqueue context produces correct synced batches; SYN_REPORT delivery never lost; SYN_DROPPED triggered if buffer overflow) |
| `models/input/evdev_per_fd.tla` | `drivers/input/evdev.md` (proves: evdev per-fd event mask + EVIOCGRAB exclusivity + concurrent reads + close-during-read; per-fd queue refcount maintained; EVIOCREVOKE atomically prevents further reads) |
| `models/input/mt_slot_tracking.tla` | `drivers/input/input-core.md` (proves: MT-protocol-B slot tracking-ID lifecycle — per-slot tracking-ID monotonic within session; slot reuse after release; concurrent slot updates from multi-touch never produce two slots with same active tracking-ID) |
| `models/input/ff_upload_play.tla` | `drivers/input/input-ff.md` (proves: FF effect upload + concurrent play + erase — per-device effect table refcount; play-while-erasing never references freed effect; per-driver upload callback completes before play allowed) |
| `models/input/uinput_lifetime.tla` | `drivers/input/uinput.md` (proves: uinput device CREATE → DESTROY lifecycle + concurrent UI_BEGIN_FF_UPLOAD from kernel-side requestor + UI_END_FF_UPLOAD from userspace; close during in-flight FF-upload completes the upload with -EINTR) |

### Layer 4: Functional contracts (Verus / Creusot)

| Component | Contract topic |
|---|---|
| `drivers/input/input-core.md` | `input_event` time field invariant: timestamp monotonic per-device; 32-bit-compat path produces equivalent timeval values; SYN_REPORT delivers complete event group |
| `drivers/input/evdev.md` | evdev per-fd ringbuffer invariant: read returns events in submission order; SYN_DROPPED inserted on buffer-full; EVIOCGRAB excludes other readers |
| `drivers/input/uinput.md` | uinput UI_DEV_CREATE post: input_dev registered, refcount=1; UI_DEV_DESTROY: input_dev unregistered, all open fds revoked, no in-flight FF-upload leaked |

### hardening (top-level)

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this Tier-2

| Feature | Default inherited from |
|---|---|
| **REFCOUNT** | per-input_dev + per-evdev_client + per-uinput_device + per-serio + per-ff_effect refcounts use `Refcount` (saturating) | § Mandatory |
| **CONSTIFY** | per-driver `input_dev` ops, per-bus `serio_driver`, per-class `input_handler` `static const` | § Mandatory |
| **SIZE_OVERFLOW** | per-event ringbuffer + MT-slot-count + FF-effect-table arithmetic uses checked operators | § Mandatory |
| **RAP / FORTIFY_RW** | per-handler input_handler vtables + per-bus serio_driver vtables read-only-after-init | § Mandatory |
| **MEMORY_SANITIZE** | freed input_dev priv data + freed FF-effect state cleared (carries per-effect parameters; per-keyboard scancode-to-keycode tables) | § Default-on configurable off |
| **STRICT_KERNEL_RWX** | input has no JIT — N/A | § Mandatory |

### Row-2 (LSM-stackable) features for input

LSM hooks called: file-LSM hooks on `/dev/input/*` open + ioctl + read/write, `/dev/uinput` open + ioctl, `/sys/class/input/.../*` writes. `security_input_*` family (proposed for Rookery — upstream coverage minimal beyond file-LSM).

GR-RBAC adds:
- Per-role disallow `/dev/input/event*` open in non-compositor processes (defends against background keyloggers).
- Per-role disallow `EVIOCGRAB` (defends against process exclusively grabbing keyboard input from compositor).
- Per-role disallow `/dev/uinput` open (defends against userspace creating virtual keyboard injecting events into other processes' focused windows).
- Per-role disallow `KDSKBMODE`/`KDSKBSENT`/console-keymap loads (defends against console keylogger via VT-layer).
- Per-role audit of every uinput CREATE + every EVIOCGRAB (which uid grabbed which device).

### Input-specific reinforcement

- **uinput CREATE requires CAP_SYS_ADMIN**: virtual-input creation can synthesize keystrokes/mouse-clicks into other processes' focused window; gate it.
- **EVIOCGRAB requires CAP_SYS_NICE in non-init-userns**: defends against grabbing keyboard from compositor in container scenarios (in init-userns the compositor is generally root anyway).
- **BadUSB-keyboard defense via drivers/usb authorization**: `authorized_default=0` for HID-keyboard class on hot-plug USB devices (defense against rogue USB device that enumerates as keyboard + types a malicious shell command); chassis-internal keyboards auto-authorized via ACPI _PLD location info. Cross-ref `drivers/usb/00-overview.md` § Hardening.
- **Console keymap LSM mediation**: `loadkeys` userspace tool requires CAP_SYS_TTY_CONFIG + LSM hook; defends against console keylogger via custom keymap.
- **i8042 power-on DOS protection**: per-channel command-rate limiting (defense against runaway PS/2 device sending continuous IRQs).
- **Userio (userspace serio) CAP_SYS_ADMIN**: `userio_register` requires CAP_SYS_ADMIN — userio creates a kernel-side serio device backed by userspace data, useful for testing but a privilege channel.
- **EVIOCREVOKE for compositor handover**: when one compositor session ends + another starts (TTY switch), prior compositor's evdev fds get EVIOCREVOKE'd by the kernel so they can't read input from the new session's user.
- **Per-cgroup input device delegation**: device cgroup controller mediates `/dev/input/event*` access per-cgroup.
- **Auto-rotate sensor LSM mediation**: HID Sensor Hub IIO accelerometer/gyroscope readable channels mediated (cross-ref `drivers/hid/00-overview.md` Sensor Hub section + `drivers/iio/`).

