# Tier-2: net/bluetooth — Bluetooth stack (HCI + L2CAP + SCO + ISO + RFCOMM + BNEP + HIDP + 6LoWPAN + SMP + Mgmt + Cmd + LE + Mesh)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/bluetooth/
  - drivers/bluetooth/
  - include/net/bluetooth/
-->

## Summary

Tier-2 wrapper for the Bluetooth stack — every Bluetooth headset, mouse, keyboard, file-transfer, BLE peripheral, audio sink/source, mesh node, beacon, fitness tracker, smart-home device, GoPro/dashcam pairing on Linux. Two physical halves: **kernel-side** (`net/bluetooth/` — protocol stack) + **drivers** (`drivers/bluetooth/` — per-controller LLDs: btusb, btintel, btmtk, btrtl, btbcm, btsdio, hci_uart line discipline + serdev, hci_vhci virtual). Components in `net/bluetooth/`:

- **af_bluetooth** (`af_bluetooth.c`): AF_BLUETOOTH socket family root
- **HCI** (`hci_core.c` + `hci_event.c` + `hci_conn.c` + `hci_sock.c` + `hci_sync.c` + `hci_debugfs.c` + `hci_codec.c` + `hci_drv.c` + `hci_sysfs.c` + `hci_request.c` + `eir.c`): Host-Controller-Interface — controller mgmt, link mgmt, event dispatch, raw HCI socket (`/dev/bluetooth/...`)
- **L2CAP** (`l2cap_core.c` + `l2cap_sock.c`): Logical Link Control Adaptation Protocol — connection-oriented + connectionless channels; supports both BR/EDR (classic) + LE
- **SCO** (`sco.c`): Synchronous Connection-Oriented — voice channels for headsets
- **ISO** (`iso.c`): Isochronous Channels — LE Audio (BAP/CAP/HAP profiles for next-gen wireless audio)
- **RFCOMM** (`rfcomm/`): emulated serial-port over L2CAP — used by SPP, DUN, OBEX, AT-command modems
- **BNEP** (`bnep/`): Bluetooth Network Encapsulation Protocol — Ethernet-over-Bluetooth (legacy PAN profile)
- **HIDP** (`hidp/`): HID Profile — wireless keyboards/mice (cross-ref `drivers/hid/00-overview.md`)
- **6LoWPAN** (`6lowpan.c`): IPv6 over BLE
- **SMP** (`smp.c` + `smp.h`): Security Manager Protocol — pairing (Just-Works, Passkey, Numeric Comparison, OOB) + key distribution + LE Secure Connections + ECDH P-256
- **Management** (`mgmt.c` + `mgmt_util.c` + `mgmt_config.c`): Bluetooth Management API over HCI socket (consumed by bluetoothd)
- **Mesh / AOSP / MSFT extensions** (`aosp.c` + `msft.c`): vendor-specific HCI extensions
- **LEDs** (`leds.c`): Bluetooth-state LED triggers
- **selftest** (`selftest.c`): Bluetooth selftests
- **ECDH helper** (`ecdh_helper.c`): ECDH P-256 for SMP LE-SC

`drivers/bluetooth/` contents: btusb (the dominant USB Bluetooth controller driver — Intel, Qualcomm, Broadcom, Realtek, MediaTek dongles + chassis controllers), btintel + btintel_pcie (Intel-specific firmware/setup), btmtk (MediaTek), btrtl (Realtek), btbcm (Broadcom), btsdio (SDIO Bluetooth), hci_uart + hci_serdev (UART + serdev attachment for embedded BT controllers + Bluetooth-over-UART headers), hci_vhci (virtual controller for testing/CI), bcm203x.c, bfusb.c, bluecard_cs.c, bpa10x.c, bt3c_cs.c, btmrvl_*.c (Marvell), btqca.c (Qualcomm Atheros chassis), btsdio.c, dtl1_cs.c, hci_*.c.

## Compatibility contract — outline

- AF_BLUETOOTH socket protocols byte-identical: `BTPROTO_HCI`, `_L2CAP`, `_SCO`, `_RFCOMM`, `_BNEP`, `_CMTP`, `_HIDP`, `_AVDTP`, `_ISO`. Socket interface (sockaddr_l2 / sockaddr_sco / sockaddr_rfcomm / sockaddr_hci) byte-identical so bluez + bluetoothd + obex + cups-bluetooth + pulseaudio-bluetooth + pipewire consume unchanged.
- HCI socket (`socket(AF_BLUETOOTH, SOCK_RAW, BTPROTO_HCI)`): `HCIDEVUP`, `HCIDEVDOWN`, `HCIDEVRESET`, `HCIDEVRESTAT`, `HCIGETDEVLIST`, `HCIGETDEVINFO`, `HCIGETCONNLIST`, `HCIGETCONNINFO`, `HCISETRAW`, `HCISETSCAN`, `HCISETAUTH`, `HCISETENCRYPT`, `HCISETPTYPE`, `HCISETLINKPOL`, `HCISETLINKMODE`, `HCISETACLMTU`, `HCISETSCOMTU`, `HCIBLOCKADDR`, `HCIUNBLOCKADDR`, `HCIINQUIRY` IOCTLs byte-identical (hciconfig + hcitool consume).
- BR/EDR + LE controller modes (dual-mode + LE-only + BR/EDR-only) all supported.
- Mgmt API event-format + command-format byte-identical (bluetoothd consumes — defines a kernel-userspace ABI distinct from HCI raw).
- `/sys/class/bluetooth/hci<N>/` byte-identical.
- `events/{bluetooth,hci}/` tracepoints byte-identical.
- Firmware load via `/lib/firmware/{intel,brcm,mediatek,rtl_bt,...}` + `request_firmware` cross-ref `drivers/base/firmware-loader.md`.

## Tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `net/bluetooth/af-bluetooth.md` | `af_bluetooth.c`: AF_BLUETOOTH root |
| `net/bluetooth/hci-core.md` | `hci_core.c` + `hci_conn.c` + `hci_sync.c` + `hci_debugfs.c` + `hci_sysfs.c` + `hci_request.c`: HCI controller core |
| `net/bluetooth/hci-event.md` | `hci_event.c` + `eir.c`: HCI event dispatch + Extended Inquiry Response |
| `net/bluetooth/hci-sock.md` | `hci_sock.c`: HCI raw socket |
| `net/bluetooth/hci-codec-drv.md` | `hci_codec.c` + `hci_drv.c`: codec-cap exchange + driver glue |
| `net/bluetooth/l2cap.md` | `l2cap_core.c` + `l2cap_sock.c` |
| `net/bluetooth/sco.md` | `sco.c`: voice channels |
| `net/bluetooth/iso.md` | `iso.c`: LE Audio isochronous |
| `net/bluetooth/rfcomm.md` | `rfcomm/`: serial-emulation |
| `net/bluetooth/bnep.md` | `bnep/`: Ethernet-over-BT (legacy PAN) |
| `net/bluetooth/hidp.md` | `hidp/`: HID Profile (wireless KB/mouse) |
| `net/bluetooth/6lowpan.md` | `6lowpan.c`: IPv6 over BLE |
| `net/bluetooth/smp.md` | `smp.c` + `ecdh_helper.c`: SMP pairing + LE-SC + ECDH |
| `net/bluetooth/mgmt.md` | `mgmt.c` + `mgmt_util.c` + `mgmt_config.c`: Management API |
| `net/bluetooth/vendor-ext.md` | `aosp.c` + `msft.c`: vendor HCI extensions |
| `net/bluetooth/leds.md` | `leds.c`: state LED triggers |
| `drivers/bluetooth/btusb.md` | `drivers/bluetooth/btusb.c` + per-vendor sub: Intel/Realtek/Broadcom/MediaTek/Qualcomm USB |
| `drivers/bluetooth/btintel.md` | `btintel.c` + `btintel_pcie.c`: Intel-specific firmware setup |
| `drivers/bluetooth/btmtk.md` | `btmtk.c`: MediaTek |
| `drivers/bluetooth/btrtl.md` | `btrtl.c`: Realtek |
| `drivers/bluetooth/btbcm.md` | `btbcm.c`: Broadcom |
| `drivers/bluetooth/hci-uart-serdev.md` | `hci_uart.c` + `hci_serdev.c`: UART + serdev attach |
| `drivers/bluetooth/hci-vhci.md` | `hci_vhci.c`: virtual controller for testing |

## Compatibility outline

- REQ-O1: AF_BLUETOOTH socket protocols + sockaddr structs byte-identical (bluez + bluetoothd + pulseaudio-bluetooth + pipewire consume unchanged).
- REQ-O2: HCI socket IOCTLs byte-identical (hciconfig + hcitool consume).
- REQ-O3: Mgmt API event/command byte-identical.
- REQ-O4: Per-controller sysfs + tracepoints byte-identical.
- REQ-O5: BR/EDR + LE + LE Audio modes work on supported HW.
- REQ-O6: TLA+ models (HCI cmd-sync + concurrent event delivery; SMP pairing state machine; L2CAP credit-based flow control; ISO BIG/CIG state machine).
- REQ-O7: Hardening: HCI raw socket CAP_NET_RAW; Mgmt API CAP_NET_ADMIN; SMP LE-SC required default-on; pairing-by-just-works denied for HID profile.

## Acceptance Criteria

- [ ] AC-O1: `bluetoothctl` enumerates controllers, scans, pairs reference BLE peripheral, connects.
- [ ] AC-O2: PulseAudio/PipeWire BT headset (A2DP) playback works.
- [ ] AC-O3: Bluetooth keyboard pairs + types into focused window.
- [ ] AC-O4: LE Audio test on supported HW: BAP unicast call works.
- [ ] AC-O5: kselftest `tools/testing/selftests/bluetooth/` passes.

## Verification

| TLA+ Model | Owner |
|---|---|
| `models/bluetooth/hci_cmd_sync.tla` | `net/bluetooth/hci-core.md` (proves: HCI cmd-sync + concurrent event delivery; event for completed cmd matched correctly; cmd timeout cancels safely) |
| `models/bluetooth/smp_pairing.tla` | `net/bluetooth/smp.md` (proves: SMP pairing state machine — Pairing-Request → Pairing-Response → Public-Keys-Exchange → Authentication → DHKey-Check → LTK-distribution; downgrade attack rejected; ECDH key validated) |
| `models/bluetooth/l2cap_credit.tla` | `net/bluetooth/l2cap.md` (proves: L2CAP credit-based flow control under concurrent S-frame + I-frame send; credit-grant + credit-debit serialized correctly) |

## Hardening

| Feature | Default |
|---|---|
| **REFCOUNT** | per-hci_dev + per-hci_conn + per-l2cap_chan + per-sock refcounts use `Refcount` | § Mandatory |
| **CONSTIFY** | per-protocol `proto_ops`, per-driver `hci_uart_proto` `static const` | § Mandatory |
| **SIZE_OVERFLOW** | per-PDU length + per-MTU arithmetic checked | § Mandatory |
| **MEMORY_SANITIZE** | freed sock priv data + SMP key state cleared (carries pairing keys) | § Default-on configurable |

Bluetooth-specific reinforcement: HCI raw socket requires CAP_NET_RAW (defends against unprivileged BT-injection); Mgmt API requires CAP_NET_ADMIN; SMP LE-SC required for BLE pairings (legacy LE-Legacy pairing default-disabled); pairing-via-Just-Works denied for HID profile (defense against trojan keyboard); per-LSM hook `security_socket_create` mediates AF_BLUETOOTH; vendor-extension HCI commands rate-limited.

## Open Questions
(none at Tier-2)

## Out of Scope
- Implementation code
- 32-bit-only paths
- bluez userspace
