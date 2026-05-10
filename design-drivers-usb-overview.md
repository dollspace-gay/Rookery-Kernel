---
title: "Tier-2: drivers/usb — USB stack (core + xhci/ehci/ohci/uhci hosts + gadget + storage + serial + hid + net + audio + typec)"
tags: ["tier-2", "drivers", "usb", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 overview for the USB subsystem — the second-most-pervasive bus in modern Linux (every keyboard, mouse, headset, webcam, USB storage, dongle, smartcard, touchpad, FIDO key, programmer dongle, USB-Ethernet adapter, USB-serial adapter, JTAG probe, oscilloscope, software-defined radio, MIDI controller, scanner, printer, fingerprint reader, biometric sensor — all USB). Mirror-image dual side: **host stack** (the kernel manages USB devices plugged into the system) AND **gadget stack** (the kernel makes the system look like a USB device to a peer host).

Eight-layer architectural sandwich:

1. **Host Controller Drivers (HCDs)**: `xhci-*` (USB 3 / 4), `ehci-*` (USB 2 high-speed), `ohci-*` + `uhci-*` (USB 1 full-speed), per-platform variants (`*-pci.c` + `*-platform.c` + `*-omap.c` etc.). Plus `dwc2/` + `dwc3/` (Synopsys DesignWare — used by many SoCs in dual-role mode), `chipidea/` (Chipidea IP), `musb/` (Mentor MUSB), `mtu3/` (MediaTek), `cdns3/` (Cadence USBSS), `renesas_usbhs/`, `isp1760/`, `fotg210/`
2. **USB core**: enumeration, configuration selection, interface-driver matching, urb (USB Request Block) submission, hub driver, address allocation, bandwidth allocation, suspend/resume, OTG state machine, message handling
3. **Class drivers**: HID (`drivers/hid/usbhid/`), storage (`drivers/usb/storage/` + uas), serial (`drivers/usb/serial/` — pl2303/ftdi_sio/ch341/cp210x/etc.), audio (`sound/usb/`), video (`drivers/media/usb/uvc/`), printer (`usblp.c`), CDC-ACM (modems), CDC-ECM/CDC-NCM (USB-Ethernet), MTP, mass-storage, smartcard (CCID), fingerprint, biometric. Most class drivers are NOT under `drivers/usb/` directly but consume the core API
4. **Gadget controllers (UDCs)**: `dwc2/`, `dwc3/`, `bdc/`, `cdns3/`, `pxa27x_udc.c`, `mv_udc_core.c`, `mtu3/`, `tegra-xudc.c`, `udc/{net2280, dummy_hcd, snps_udc_*, fusb300_udc, fotg210-udc, gr_udc, lpc32xx_udc, m66592-udc, omap_udc, pch_udc, r8a66597-udc, atmel_usba_udc, max3420_udc}`. UDC core (`drivers/usb/gadget/udc/core.c`) presents a uniform API to gadget functions
5. **Gadget functions** (`drivers/usb/gadget/function/`): `f_acm` (modem), `f_ecm` / `f_ncm` (Ethernet), `f_eem`, `f_rndis`, `f_mass_storage`, `f_fs` (FunctionFS — userspace function), `f_hid`, `f_midi`, `f_phonet`, `f_printer`, `f_serial`, `f_obex`, `f_uac1` / `f_uac2` (audio), `f_uvc` (video), `f_tcm` (LIO target), `f_loopback`, `f_sourcesink`, `f_subset`. Compose into composite gadgets via configfs at `/sys/kernel/config/usb_gadget/`
6. **Gadget legacy** (`drivers/usb/gadget/legacy/`): pre-configfs single-function drivers (g_ether, g_mass_storage, g_serial, g_zero, g_audio, g_hid, g_midi, g_multi, g_ncm, g_printer, g_acm_ms, g_dbgp, g_webcam) — kept for backward compat
7. **typec** (`drivers/usb/typec/`): USB Type-C connector + Power Delivery + Alt-Mode (DisplayPort, Thunderbolt, USB4) — separate sub-tree
8. **dual-role / OTG / role-switch** (`drivers/usb/roles/` + `drivers/usb/common/`): cross-cuts host + gadget; manages the USB role of dual-role ports
9. **monitor / debug** (`drivers/usb/mon/`): usbmon — pcap-style packet capture for USB
10. **usbip** (`drivers/usb/usbip/`): USB-over-IP — share USB devices over network (kernel-side host + stub-driver)
11. **misc / class / image / serial / storage / phy / atm / image / early / chipidea / c67x00**: rest of subdirs

Heavily cross-referenced from `drivers/pci/00-overview.md` (xhci-pci/ehci-pci binding), `drivers/base/00-overview.md` (every USB device is an LDM device), `drivers/iommu/00-overview.md` (xhci DMA via IOMMU), `kernel/dma/00-overview.md` (urb_dma_buffer), `drivers/hid/00-overview.md` (usbhid is the largest class consumer — separate Tier-2 future), `sound/00-overview.md` (snd-usb-audio), `drivers/media/00-overview.md` (uvcvideo + usbtv), `net/00-overview.md` (CDC-NCM/CDC-ECM/RNDIS netdev), `drivers/typec/` (cross-ref), `kernel/cgroup/00-overview.md` (per-cgroup USB-device delegate via /sys/bus/usb/).

### Out of Scope

- Per-Tier-3 (Phase D) — per-driver docs arrive incrementally
- ARM-only SoC USB host glue beyond dwc2/dwc3/chipidea (compile-gated off for v0)
- USB ATM modems (`atm/`) — out of v0 (legacy DSL)
- USB hamradio + obsolete drivers (legousbtower, etc.) — keep present but compile-gated off
- HID-over-USB (lives in `drivers/hid/usbhid/` — separate Tier-2 future for HID)
- USB-audio (lives in `sound/usb/` — separate Tier-2 future for sound)
- USB-video / V4L2 USB drivers (`drivers/media/usb/` — separate Tier-2 future for media)
- 32-bit-only paths
- Implementation code

### components

### Core (`drivers/usb/core/`)

- `usb.c` + `hub.c` + `port.c` + `driver.c` + `urb.c` + `message.c` + `config.c` + `endpoint.c` + `devices.c` + `devio.c` + `file.c` + `generic.c` + `hcd.c` + `hcd-pci.c` + `hub.h` + `notify.c` + `port.c` + `quirks.c` + `sysfs.c` + `usb-acpi.c` + `usb.h`: enumeration, config/interface mgmt, urb sub-completion, hub driver (the largest single file in core), device naming, sysfs surface, ACPI integration

### Host controllers (`drivers/usb/host/`)

- xhci: `xhci.c` + `xhci-mem.c` + `xhci-ring.c` + `xhci-hub.c` + `xhci-dbg.c` + `xhci-trace.c` + `xhci-debugfs.c` + `xhci-ext-caps.c` + per-platform glue (`xhci-pci.c`, `xhci-plat.c`, `xhci-mvebu.c`, `xhci-tegra.c`, `xhci-rcar.c`, `xhci-renesas.c`, etc.)
- ehci: `ehci-hcd.c` + `ehci-hub.c` + `ehci-mem.c` + `ehci-q.c` + `ehci-sched.c` + `ehci-sysfs.c` + `ehci-timer.c` + `ehci-dbg.c` + per-platform glue (`ehci-pci.c`, `ehci-platform.c`, `ehci-omap.c`, `ehci-orion.c`, `ehci-fsl.c`, etc.)
- ohci: `ohci-hcd.c` + `ohci-hub.c` + `ohci-mem.c` + `ohci-q.c` + per-platform glue (`ohci-pci.c`, `ohci-platform.c`, etc.)
- uhci: `uhci-hcd.c` + `uhci-debug.c` + `uhci-hub.c` + `uhci-pci.c` + `uhci-q.c`
- USB-2 + USB-3 dual-role: `dwc2/`, `dwc3/`, `chipidea/`, `musb/`, `mtu3/`, `cdns3/`, `renesas_usbhs/`
- legacy: `isp1760/`, `fotg210/`, `r8a66597-hcd.c`, `oxu210hp-hcd.c`, `octicom-hcd.c`

### Gadget (`drivers/usb/gadget/`)

- `udc/core.c`: UDC API — uniform gadget controller abstraction
- `udc/<vendor>.c`: per-UDC drivers
- `function/<f_*>.c`: per-function drivers (composable)
- `legacy/<g_*>.c`: pre-configfs single-function gadgets
- `composite.c`: composite-gadget infrastructure
- `configfs.c`: configfs interface for runtime composition
- `epautoconf.c`: endpoint auto-allocation across UDCs

### Class subdirs (under `drivers/usb/`)

- `storage/`: usb-storage + uas (USB Attached SCSI) — exposes USB mass-storage as SCSI/blockdev
- `serial/`: USB-serial (pl2303, ftdi_sio, ch341, cp210x, mct_u232, kobil_sct, opticon, qcaux, quatech_serial, sierra, spcp8x5, …) — many vendor drivers, all expose `/dev/ttyUSB<N>`
- `class/`: standard USB class drivers — `cdc-acm.c` (modem), `usbtmc.c` (test/measurement), `usblp.c` (printer), `cdc-wdm.c` (USB modem ctrl-channel), `usb-midi-v2.c`
- `image/`: scanner driver (`microtek.c`)
- `atm/`: USB ATM modems (legacy DSL — out of v0 candidate)
- `mon/`: usbmon packet capture
- `usbip/`: USB-over-IP
- `misc/`: misc drivers — `iowarrior.c`, `usbsevseg.c`, `lvstest.c`, `usb251xb.c` (hub config), `chaoskey.c` (HW RNG), `legousbtower.c`, `appledisplay.c`, `usbtest.c` (host-side test), `ehset.c` (host-side electrical-test fixture), `usb-emi62.c`, `cypress_cy7c63.c`, `cytherm.c`, `idmouse.c`, `iowarrior.c`, `ldusb.c`, `qcom_eud.c`, `trancevibrator.c`, `usb3503.c` (hub PD), `uss720.c` (parport adapter), `yurex.c`, `ftdi-elan.c`, `usb-emi26.c`, `apple-mfi-fastcharge.c`, `onboard_usb_dev.c`, `usbcam.c` (legacy), `appleir.c`
- `typec/`: USB-C connector + PD + Alt-Mode + UCSI (UCSI = UCSI ACPI mailbox)
- `roles/`: dual-role/role-switch core
- `phy/`: USB PHY drivers (mostly ARM SoC — keep some for x86 chipset)
- `early/`: USB-HCI early-boot pre-printk console support
- `mtu3/`: MediaTek MUSB++
- `cdns3/`, `chipidea/`, `dwc2/`, `dwc3/`, `musb/`: dual-role IP cores
- `c67x00/`: Cypress C67x00
- `fotg210/`: Faraday FOTG210
- `isp1760/`: Philips ISP1760
- `renesas_usbhs/`: Renesas USB HS
- `common/`: shared utility code (LED triggers, etc.)
- `usb-skeleton.c`: example driver

### scope

This Tier-2 governs `/home/doll/linux-src/drivers/usb/` (~30 subdirs + thousands of files), public headers `include/linux/usb*.h` + `include/linux/usb/{ch9, gadget, hcd, otg, otg-fsm, role, typec, ulpi, ehci_pdriver, ohci_pdriver, ...}.h`, UAPI `include/uapi/linux/usbdevice_fs.h` + `include/uapi/linux/usb/{audio,cdc,functionfs,gadgetfs,midi,raw_gadget,tmc,video}.h`. typec sub-tree is large + half-independent — Tier-3s under this Tier-2 cover typec proper but a future Tier-2 split is possible.

`drivers/hid/usbhid/` (HID-over-USB consumer of USB core) is in scope for cross-ref but lives under HID Tier-2 future. `sound/usb/` likewise lives under sound Tier-2. `drivers/media/usb/` lives under media Tier-2. CDC-NCM/CDC-ECM/RNDIS netdevs live under net Tier-2 wrapper but consume USB core.

### compatibility contract — outline

### `/dev/bus/usb/<bus>/<dev>` chardev (usbfs)

`/dev/bus/usb/<BBB>/<DDD>` per-device chardev with `usbdevice_fs.h` IOCTLs:
- `USBDEVFS_CLAIMINTERFACE` / `_RELEASEINTERFACE` / `_SETINTERFACE` / `_SETCONFIGURATION` / `_GETDRIVER` / `_DISCONNECT` / `_CONNECT` / `_DISCONNECT_CLAIM` / `_CONNECTINFO`
- `USBDEVFS_CONTROL` / `_BULK` / `_RESETEP` / `_REAPURB` / `_REAPURBNDELAY` / `_DISCSIGNAL` / `_CLAIM_PORT` / `_RELEASE_PORT` / `_GET_CAPABILITIES` / `_DISCONNECT_CLAIM` / `_ALLOC_STREAMS` / `_FREE_STREAMS` / `_DROP_PRIVILEGES` / `_GET_SPEED`
- `USBDEVFS_SUBMITURB` / `_SUBMITURB32` / `_REAPURB32` / `_REAPURBNDELAY32`
- `USBDEVFS_HUB_PORTINFO`

Wire-format byte-identical so libusb consumes unchanged.

### sysfs surface

Per-device `/sys/bus/usb/devices/<bus>-<port>.<port2>...:<config>.<intf>/`:
- Per-device: `idVendor`, `idProduct`, `manufacturer`, `product`, `serial`, `version`, `bcdDevice`, `bDeviceClass`, `bDeviceSubClass`, `bDeviceProtocol`, `bMaxPacketSize0`, `bNumConfigurations`, `bcdUSB`, `bConfigurationValue`, `bMaxPower`, `urbnum`, `speed`, `tx_lanes`, `rx_lanes`, `removable`, `ltm_capable`, `usb2_lpm_l1_timeout`, `usb2_lpm_besl`, `usb2_hardware_lpm`, `usb3_hardware_lpm_u1`, `usb3_hardware_lpm_u2`, `power/{control,autosuspend,autosuspend_delay_ms,wakeup,wakeup_count,active_duration,connected_duration,...}`, `authorized`, `authorized_default`, `interface_authorized_default`, `descriptors`, `bos_descriptors`, `configuration`, `iManufacturer`, `iProduct`, `iSerial`, `iConfiguration`
- Per-interface: `bInterfaceNumber`, `bInterfaceClass`, `bInterfaceSubClass`, `bInterfaceProtocol`, `bAlternateSetting`, `bNumEndpoints`, `interface`, `modalias`, `supports_autosuspend`, `authorized`
- Per-port (under hub): `connect_type`, `disable`, `quirks`, `over_current_count`, `state`, `usb3_lpm_permit`, `location`

Layout + content byte-identical so udevadm + lsusb consume unchanged.

### `MODULE_DEVICE_TABLE(usb, ...)` + modalias

Per-driver `usb_device_id` table generates `usb:vXXXXpXXXXdlXXXXdhXXXXdcXXdscXXdpXXicXXiscXXipXXinXX` modalias. Format frozen.

### `/sys/kernel/debug/usb/`

usbmon `usbmon0`, `usbmon1`, ... (per-bus packet capture in pcap-readable format), debug counters per-HCD. Layout byte-identical so wireshark + tcpdump can decode.

### Configfs gadget tree

`/sys/kernel/config/usb_gadget/<name>/{idVendor,idProduct,bcdDevice,bcdUSB,bMaxPacketSize0,bDeviceClass,bDeviceSubClass,bDeviceProtocol,UDC,strings/0x409/{manufacturer,product,serialnumber},configs/c.1/{bmAttributes,MaxPower,strings/0x409/configuration,functions/<f_*>.<inst>},functions/<f_*>.<inst>/{...}}`. Layout byte-identical so libcomposite / Android-init / ConfigFS-Composer consume unchanged.

### USBIP wire format

`drivers/usb/usbip/` — kernel-side host (`vhci_hcd`) + stub (`usbip-host`) + userspace `usbip` tool. URB-over-TCP packet format byte-identical so cross-host USB sharing works.

### usbmon pcap format

usbmon binary + text format byte-identical so existing packet-capture analysis tools consume.

### USB Type-C UCSI

`/sys/class/typec/portN/`, `/sys/class/typec/portN-partner/`, `/sys/class/typec/portN-cable/`, alt-modes via `port<N>.<altmode-id>`. Layout byte-identical so typecd / dr-mode-tools consume.

### LED triggers

`drivers/usb/common/usb-conn-gpio.c` + LED-trigger glue exposes per-port LED triggers. Wire identical.

### `usbcore=` cmdline + module params

`usbcore.autosuspend`, `usbcore.usbfs_snoop`, `usbcore.blinkenlights`, `usbcore.old_scheme_first`, `usbcore.use_both_schemes`, `usbcore.initial_descriptor_timeout`, etc. parsed identically.

### tier-3 docs governed by this tier-2

(Phase D will add these incrementally; per-driver Tier-3 docs.)

| Tier-3 doc | Scope |
|---|---|
| `drivers/usb/core-enumerate.md` | `core/usb.c` + `core/hub.c` + `core/port.c` + `core/config.c`: device enumeration + hub driver + config selection |
| `drivers/usb/core-driver.md` | `core/driver.c`: usb_driver registration + interface-driver match |
| `drivers/usb/core-urb.md` | `core/urb.c` + `core/message.c`: URB submission + completion + sync messages |
| `drivers/usb/core-devio.md` | `core/devio.c` + `core/file.c`: usbfs (`/dev/bus/usb/...`) IOCTLs |
| `drivers/usb/core-hcd.md` | `core/hcd.c` + `core/hcd-pci.c`: HCD framework (per-HCD bus / port / urb dispatch) |
| `drivers/usb/core-sysfs.md` | `core/sysfs.c` + `core/devices.c`: per-device + per-interface sysfs surface |
| `drivers/usb/core-quirks.md` | `core/quirks.c`: per-device quirks table |
| `drivers/usb/core-acpi.md` | `core/usb-acpi.c`: ACPI _PLD + UPC integration |
| `drivers/usb/host-xhci.md` | `host/xhci-*`: xHCI host (USB 3 / 4) |
| `drivers/usb/host-ehci.md` | `host/ehci-*`: eHCI host (USB 2) |
| `drivers/usb/host-ohci.md` | `host/ohci-*`: oHCI host (USB 1.1) |
| `drivers/usb/host-uhci.md` | `host/uhci-*`: uHCI host (Intel USB 1.1) |
| `drivers/usb/host-dwc3.md` | `dwc3/`: Synopsys DesignWare USB3 dual-role |
| `drivers/usb/host-dwc2.md` | `dwc2/`: Synopsys DesignWare USB2 dual-role |
| `drivers/usb/storage-ums.md` | `storage/`: usb-storage (BBB transport) |
| `drivers/usb/storage-uas.md` | `storage/uas.c`: UAS — USB Attached SCSI |
| `drivers/usb/serial-core.md` | `serial/usb-serial.c` + `serial/console.c` + `serial/generic.c`: USB-serial framework |
| `drivers/usb/serial-vendors.md` | `serial/{ftdi_sio, pl2303, ch341, cp210x, ...}`: per-vendor USB-serial drivers |
| `drivers/usb/class-cdc-acm.md` | `class/cdc-acm.c`: CDC-ACM modem class |
| `drivers/usb/class-usbtmc.md` | `class/usbtmc.c`: USB Test+Measurement |
| `drivers/usb/class-usblp.md` | `class/usblp.c`: USB printer class |
| `drivers/usb/class-cdc-wdm.md` | `class/cdc-wdm.c`: CDC modem control channel |
| `drivers/usb/mon-usbmon.md` | `mon/`: usbmon packet capture |
| `drivers/usb/usbip-host.md` | `usbip/{stub_*}`: USBIP host side (export local USB to network) |
| `drivers/usb/usbip-vhci.md` | `usbip/{vhci_*}`: USBIP virtual host (import remote USB) |
| `drivers/usb/gadget-udc-core.md` | `gadget/udc/core.c`: UDC API (uniform gadget-controller abstraction) |
| `drivers/usb/gadget-composite.md` | `gadget/composite.c`: composite gadget infrastructure |
| `drivers/usb/gadget-configfs.md` | `gadget/configfs.c`: runtime composition via configfs |
| `drivers/usb/gadget-functions.md` | `gadget/function/`: per-function drivers (f_acm, f_ecm, f_ncm, f_mass_storage, f_fs, f_hid, f_uvc, f_uac1/2, f_tcm, ...) |
| `drivers/usb/gadget-legacy.md` | `gadget/legacy/`: pre-configfs single-function gadgets |
| `drivers/usb/typec-core.md` | `typec/`: USB Type-C connector + PD + Alt-Mode |
| `drivers/usb/typec-ucsi.md` | `typec/ucsi/`: UCSI ACPI mailbox |
| `drivers/usb/roles.md` | `roles/`: dual-role / role-switch core |
| `drivers/usb/early.md` | `early/`: USB pre-printk early console |

### compatibility outline (top-level)

- REQ-O1: usbfs `/dev/bus/usb/<bus>/<dev>` IOCTLs (USBDEVFS_*) byte-identical (libusb consumes unchanged).
- REQ-O2: Per-device + per-interface + per-port sysfs surface byte-identical (lsusb / udevadm / sosreport consume unchanged).
- REQ-O3: `MODULE_DEVICE_TABLE(usb, ...)` modalias format byte-identical.
- REQ-O4: usbmon packet-capture format byte-identical (wireshark / tcpdump-usbmon decode).
- REQ-O5: USBIP URB-over-TCP wire format byte-identical (cross-host USB sharing works).
- REQ-O6: Configfs gadget tree (`/sys/kernel/config/usb_gadget/`) layout byte-identical (libcomposite / Android consume unchanged).
- REQ-O7: usb-storage + uas SCSI binding identical (USB drives appear as `/dev/sd<X>`).
- REQ-O8: USB-serial naming (`/dev/ttyUSB<N>`) + chardev IOCTLs identical.
- REQ-O9: CDC-NCM / CDC-ECM / RNDIS netdev binding works as netdev consumers.
- REQ-O10: Type-C + PD + Alt-Mode sysfs surface byte-identical.
- REQ-O11: HCD support: xhci (USB 2 + 3 + 4), ehci, ohci, uhci, dwc2, dwc3 — all enumerate devices on respective HW.
- REQ-O12: TLA+ models declared at this Tier-2 (USB hub state machine, xhci ring management, gadget-composite UDC bind, urb-cancel race-freedom, usbfs claim/release).
- REQ-O13: Verus/Creusot Layer-4 functional contracts on URB lifetime + endpoint-buffer arithmetic + descriptor parsing bounds.
- REQ-O14: Hardening: row-1 features applied per `00-security-principles.md`, with USB-specific reinforcement (usbguard-style per-device authorization, devio CAP_SYS_RAWIO + LSM mediation, BadUSB defense via authorized=0 default-on for hot-plug class).

### acceptance criteria (top-level)

- [ ] AC-O1: `lsusb -v` enumerates every USB device matching upstream baseline byte-identical descriptor dump. (covers REQ-O1, REQ-O2)
- [ ] AC-O2: USB keyboard + mouse + USB-storage + USB-Ethernet adapter all enumerate + work on a reference machine. (covers REQ-O7, REQ-O9, REQ-O11)
- [ ] AC-O3: USB-3 NVMe-USB enclosure throughput within 5% upstream baseline (UAS). (covers REQ-O7, REQ-O11)
- [ ] AC-O4: USB-serial test: pl2303 / ftdi_sio adapter enumerates, `/dev/ttyUSB0` appears, echo + cat through it works. (covers REQ-O8)
- [ ] AC-O5: usbmon test: `tcpdump -i usbmon0 -w usb.pcap` captures a USB transaction; wireshark decodes. (covers REQ-O4)
- [ ] AC-O6: USBIP test: `usbip bind -b <bus>` on host-A; `usbip attach -r <ip> -b <bus>` on host-B; remote device appears on host-B + works. (covers REQ-O5)
- [ ] AC-O7: Gadget composite test: configfs builds `g_ether` + `g_mass_storage` composite; host PC sees Ethernet + storage. (covers REQ-O6)
- [ ] AC-O8: Type-C alt-mode DisplayPort: USB-C dock with DP alt-mode enumerates port-partner correctly + DP video appears. (covers REQ-O10)
- [ ] AC-O9: Suspend-resume test: USB devices restore connectivity post-resume; runtime PM auto-suspend works for idle keyboards. (covers REQ-O11)
- [ ] AC-O10: kselftest `tools/testing/selftests/drivers/usb/` passes. (covers REQ-O12)
- [ ] AC-O11: USB hub stress test (loop hot-plug 1000x): no urb leak, no devnode leak, no memory leak (KASAN clean). (covers REQ-O11, REQ-O12)
- [ ] AC-O12: BadUSB test: malicious composite device claiming HID + storage + serial — `authorized_default=0` policy denies binding until user explicitly authorizes. (covers REQ-O14)

### verification (top-level)

### Layer 2: TLA+ models — mandatory list

| Model | Owned by |
|---|---|
| `models/usb/hub_state.tla` | `drivers/usb/core-enumerate.md` (proves: hub port state machine — DISCONNECTED → POWERED_OFF → POWERED_ON → CONNECTED → ENABLED → SUSPENDED; concurrent disconnect + enumerate + suspend never produce ghost device or stuck-disabled port) |
| `models/usb/xhci_ring.tla` | `drivers/usb/host-xhci.md` (proves: xHCI command-ring + event-ring + transfer-ring producer-consumer ordering with xHC posting events to ERST while CPU writes commands; cycle-bit + DMS handling correct under concurrent enqueue/dequeue) |
| `models/usb/urb_lifetime.tla` | `drivers/usb/core-urb.md` (proves: urb refcount + submit + complete + cancel sequence — concurrent usb_unlink_urb + urb completion never produces double-complete or use-after-free) |
| `models/usb/usbfs_claim.tla` | `drivers/usb/core-devio.md` (proves: USBDEVFS_CLAIMINTERFACE + _RELEASEINTERFACE + _DISCONNECT + concurrent kernel-driver auto-bind never produces double-bind or interface stuck-claimed) |
| `models/usb/gadget_udc_bind.tla` | `drivers/usb/gadget-udc-core.md` (proves: gadget-bind-to-UDC sequence with concurrent UDC remove + composite reconfigure; bind / unbind never produces partial state or function dangling pointer) |

### Layer 4: Functional contracts (Verus / Creusot)

| Component | Contract topic |
|---|---|
| `drivers/usb/core-enumerate.md` | `usb_new_device` post-condition: device is registered with valid descriptors, refcount=1, sysfs entries created or fully rolled back |
| `drivers/usb/core-urb.md` | `usb_submit_urb` invariant: returned 0 means urb has been queued and complete handler will be called exactly once; non-zero means complete handler will NOT be called |
| `drivers/usb/host-xhci.md` | xhci ring advance: `enqueue_pointer` advances by exactly the number of TRBs added; cycle bit toggled at ring wrap; ERST `event_ring_dequeue_pointer` updated only after consuming events |

### hardening (top-level)

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this Tier-2

| Feature | Default inherited from |
|---|---|
| **REFCOUNT** | per-usb_device + per-usb_interface + per-urb + per-hcd refcounts use `Refcount` (saturating) | § Mandatory |
| **CONSTIFY** | per-driver `usb_driver`, per-HCD `hc_driver`, per-class `usb_class_driver`, per-gadget-function `usb_function_instance_ops` `static const` | § Mandatory |
| **SIZE_OVERFLOW** | URB transfer_buffer_length + endpoint wMaxPacketSize + descriptor wTotalLength arithmetic uses checked operators | § Mandatory |
| **RAP / FORTIFY_RW** | per-driver usb_driver + per-HCD hc_driver vtables read-only-after-init | § Mandatory |
| **MEMORY_SANITIZE** | freed urb buffers + per-device data cleared (carries CCID smartcard PIN buffers, FIDO key challenge data, etc.) | § Default-on configurable off |
| **STRICT_KERNEL_RWX** | USB has no JIT — N/A | § Mandatory |

### Row-2 (LSM-stackable) features for USB

LSM hooks called: `security_usb_*` family (proposed for Rookery — upstream has minimal coverage). File-LSM hooks on `/dev/bus/usb/*` open + ioctl, `/sys/bus/usb/devices/.../authorized` writes, `/sys/kernel/config/usb_gadget/` writes.

GR-RBAC adds:
- Per-role disallow `/dev/bus/usb/*` open (denies userspace USB drivers like libusb-using apps).
- Per-role disallow USBDEVFS_DISCONNECT / _CONNECT (denies driver-unbind from userspace).
- Per-role disallow `/sys/bus/usb/devices/.../authorized` write (defends against bypass of authorization policy).
- Per-role disallow `/sys/kernel/config/usb_gadget/` writes (only kernel-admin can configure gadget).
- Per-role disallow USBIP host-export (only kernel-admin can share USB devices over network).
- Per-role audit of every USB device authorization + every gadget composition change.

### USB-specific reinforcement

- **`authorized_default=0` on hot-plug class default-on** in Rookery for HID + storage + Ethernet (the BadUSB classes); user must explicitly authorize via `/sys/bus/usb/devices/.../authorized` write. Built-in USB devices (chassis-internal, e.g., laptop touchpad) auto-authorized via ACPI _PLD location info.
- **Per-port `connect_type` enforcement**: `/sys/bus/usb/devices/<bus>-<port>/connect_type` = "hardwired" auto-authorize; "hotplug" requires user authorization.
- **devio `USBDEVFS_DISCONNECT` requires CAP_SYS_RAWIO** (defense against userspace forcing kernel-driver unbind to attack via libusb).
- **Gadget-composite authorization**: `UDC` writeable file in configfs requires CAP_SYS_ADMIN + LSM mediation (gadget-bind exposes the system as a USB device — sensitive).
- **USBIP wire format input validation**: every URB packet from network validated for descriptor consistency before injecting into local USB stack (defense against cross-host USB-fuzz).
- **HCD descriptor parsing bounds**: every USB descriptor walk in `core/config.c` bounds-checked against wTotalLength + bLength fields (defense against malicious device with malformed descriptors causing OOB read).
- **xHCI doorbell + ring access fenced**: every doorbell write preceded by mb() to prevent reordering past TRB writes (already true upstream; Rookery makes explicit + asserts).
- **usbmon snoop access**: requires CAP_NET_ADMIN + CAP_SYS_ADMIN (usbmon captures every USB packet including HID keystrokes — heavily privileged).
- **Authorization revocation propagates**: writing `0` to authorized triggers immediate URB-cancel + driver-unbind so a previously-authorized rogue device can be locked out without unplug.

