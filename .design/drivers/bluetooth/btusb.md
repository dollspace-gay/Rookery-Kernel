# Tier-3: drivers/bluetooth/btusb.c — USB HCI transport driver (vendor quirks + fw upload + ALT-setting selection)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/bluetooth/00-overview.md
upstream-paths:
  - drivers/bluetooth/btusb.c
  - drivers/bluetooth/btintel.c
  - drivers/bluetooth/btintel.h
  - drivers/bluetooth/btbcm.c
  - drivers/bluetooth/btmtk.c
  - drivers/bluetooth/btrtl.c
  - drivers/bluetooth/btqca.c
-->

## Summary

`btusb` is the canonical USB HCI transport driver — by line count (~4700 lines) and by VID/PID-table breadth, the largest single Bluetooth driver. It claims USB Bluetooth controllers (Class 0xe0 Subclass 0x01) plus a long quirk-listed allowlist of vendor-specific re-ids, sets up bulk + interrupt + isochronous endpoints, routes per-HCI-packet-type traffic to the right endpoint pair, drives vendor-specific firmware upload via the per-vendor helper libs (`btintel`, `btbcm`, `btmtk`, `btrtl`, `btqca`), and selects USB ALT-setting per `hdev->voice_setting` for SCO/ISO audio.

This Tier-3 covers `drivers/bluetooth/btusb.c` (~4669 lines: probe, intf claim, ep setup, packet routing, vendor-quirk-dispatch, fw upload integration) plus the read-only contract with the per-vendor helper libs.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct btusb_data` (`drivers/bluetooth/btusb.c`) | per-intf private data | `drivers::bluetooth::btusb::Data` |
| `btusb_table[]` | USB VID/PID + driver_info quirk-bits | `drivers::bluetooth::btusb::ID_TABLE` |
| `btusb_probe(intf, id)` / `_disconnect(intf)` | USB probe/disconnect | `Driver::probe` / `_disconnect` |
| `btusb_open(hdev)` / `_close(hdev)` / `_flush(hdev)` | `hci_driver` callbacks | `Driver::open` / `_close` / `_flush` |
| `btusb_send_frame(hdev, skb)` | route per `hci_skb_pkt_type(skb)` to bulk / intr / isoc | `Driver::send_frame` |
| `btusb_recv_intr(urb)` / `_bulk(urb)` / `_isoc(urb)` | completion handlers | `Driver::recv_*` |
| `btusb_setup_*` (`_intel`, `_bcm`, `_rtl`, `_mtk`, `_qca`, `_csr`) | per-vendor setup dispatch | `Driver::setup_*` |
| `btusb_set_isoc_alt(intf, alt)` | claim/select ISOC ALT-setting | `Driver::set_isoc_alt` |
| `btintel_setup(hdev)` / `btintel_bootloader_setup(hdev)` / `btintel_check_version(hdev, &ver)` / `btintel_download_firmware(hdev, ver, fw)` (`btintel.c`) | Intel-specific helpers | `vendor::intel::*` |
| `btbcm_patchram(hdev, fw)` / `btbcm_setup_patchram(hdev)` (`btbcm.c`) | Broadcom patchram | `vendor::bcm::*` |
| `btmtk_setup_firmware(hdev, fwname)` (`btmtk.c`) | MediaTek fw setup | `vendor::mtk::*` |
| `btrtl_setup_realtek(hdev)` / `_initialize(hdev, fw)` (`btrtl.c`) | Realtek setup | `vendor::rtl::*` |
| `qca_setup(hdev)` / `qca_uart_setup(hdev, ...)` (`btqca.c`) | Qualcomm setup | `vendor::qca::*` |
| BTUSB_* driver_info bits (`btusb.c`) | per-controller quirk mask | `bitflags BtUsbQuirk` |

## Compatibility contract

REQ-1: USB probe matches against `btusb_table[]` — generic Class 0xe0 Subclass 0x01 catches most parts; per-VID/PID rows add quirk bits (`BTUSB_INTEL_COMBINED`, `BTUSB_BCM_PATCHRAM`, `BTUSB_REALTEK`, `BTUSB_MEDIATEK`, `BTUSB_QCA_ROME`, `BTUSB_QCA_WCN6855`, `BTUSB_AMP`, `BTUSB_IFNUM_2`, `BTUSB_BCM_APPLE`, `BTUSB_BARROT`, `BTUSB_BROKEN_ISOC`, `BTUSB_WRONG_SCO_MTU`, `BTUSB_WIDEBAND_SPEECH`, ...).

REQ-2: Per-intf claim discipline — driver claims intf 0 of every BT-class device; controllers with `BTUSB_IFNUM_2` quirk also claim intf 1 (isoc). Refuse claim if intf descriptor lists endpoints inconsistent with HCI shape (1× bulk IN + 1× bulk OUT + 1× intr IN minimum; isoc pair optional).

REQ-3: ALT-setting selection for SCO/ISO — per-controller iso intf has multiple ALT settings each exposing a different `wMaxPacketSize` (typically 9, 17, 25, 33, 49, 63 bytes for 1× to 6× HV1/2/3/HV1 ESCO timeslot). Driver chooses ALT based on `hdev->voice_setting` (CVSD vs mSBC vs μ-law) and active SCO conn count.

REQ-4: Packet routing on `send_frame`:
- `HCI_COMMAND_PKT` → USB control transfer (vendor-specific request 0x00 to intf).
- `HCI_ACLDATA_PKT` → bulk OUT urb.
- `HCI_SCODATA_PKT` → isoc OUT urb (requires ISOC intf claimed).
- `HCI_EVENT_PKT` → received via intr IN (not sent).
- `HCI_ISODATA_PKT` → isoc OUT urb on iso intf.
- `HCI_DRV_PKT` → driver-internal, never seen on wire.

REQ-5: Vendor `setup` dispatch — based on `driver_info`:
- `BTUSB_INTEL_COMBINED` → `btintel_setup` → version-check → firmware download from `/lib/firmware/intel/ibt-<hw>-<fw>.sfi` + patches.
- `BTUSB_BCM_PATCHRAM` → `btbcm_setup_patchram` → patchram blob from `/lib/firmware/brcm/BCM*.hcd`.
- `BTUSB_REALTEK` → `btrtl_setup_realtek` → fw blob from `/lib/firmware/rtl_bt/`.
- `BTUSB_MEDIATEK` → `btmtk_setup_firmware`.
- `BTUSB_QCA_ROME` / `BTUSB_QCA_WCN6855` → `qca_setup` → NVM + RAM-FW from `/lib/firmware/qca/`.

REQ-6: Firmware signature mandatory — per-vendor lib verifies signature/CRC before pushing the upload commands. Tampered blob refused.

REQ-7: Per-urb credit accounting — bulk IN: max 8 concurrent urbs; intr IN: 1 urb; isoc IN/OUT: per-intf max packet-count from descriptor. Overruns refused.

REQ-8: USB autosuspend integration — `enable_autosuspend` module parameter (CONFIG_BT_HCIBTUSB_AUTOSUSPEND default); per-controller `usb_set_intfdata(intf, btusb_data)` allows USB-core to suspend the intf when no HCI activity.

REQ-9: Vendor quirk allowlist — `driver_info` bits restricted to BTUSB_* set defined in `btusb.c`; refuse unknown bit pattern. Refcounted via `try_module_get` on per-vendor lib.

REQ-10: Disconnect path — quiesce all urbs (`usb_kill_urb` on each), drop per-conn state, `hci_unregister_dev(hdev)`, free intf data.

REQ-11: Wakeup discipline — per-controller `intf->needs_remote_wakeup = 1` when LE-advertising or page-scan active; refuse remote-wake source otherwise.

## Acceptance Criteria

- [ ] AC-1: `lsusb -d <BT VID/PID>` shows class 0xe0 sub 0x01; `dmesg | grep btusb` shows clean probe + intf claim.
- [ ] AC-2: `hciconfig hci0 up` succeeds; controller-info read returns valid `bdaddr`, manufacturer, lmp_version.
- [ ] AC-3: ACL throughput test on LE link reaches ≥ 1 Mbps; bulk OUT/IN urbs complete without ETIMEDOUT.
- [ ] AC-4: SCO audio (HFP call via `pulseaudio`) routes through ALT-1 (CVSD) or ALT-6 (mSBC); no underrun in steady playback for ≥ 60s.
- [ ] AC-5: Per-vendor fw upload completes; `dmesg` shows "firmware download successful" or vendor-equivalent banner.
- [ ] AC-6: Suspend/resume cycle (`systemctl suspend` → wakeup): hci0 survives, mgmt index stays present, controller-info matches pre-suspend.
- [ ] AC-7: Tampered fw blob test: replace `/lib/firmware/intel/ibt-*.sfi` with a corrupt blob → `btintel_download_firmware` returns -EBADMSG, hci0 not registered.
- [ ] AC-8: USB device-removal (`echo 1 > /sys/bus/usb/devices/.../authorized` flip) cleans up; no kernel oops, all urbs killed.
- [ ] AC-9: btmon trace captures cmd/event/ACL flow without packet corruption across at least one full LE scan + connect + GATT-read cycle.

## Architecture

`btusb::Data` lives in `drivers::bluetooth::btusb::Data`:

```
struct Data {
  hdev: Arc<HciDev>,
  udev: NonNull<UsbDevice>,
  intf: NonNull<UsbInterface>,
  isoc: Option<NonNull<UsbInterface>>,
  isoc_anchor: UrbAnchor,              // isoc IN urbs
  bulk_anchor: UrbAnchor,
  intr_anchor: UrbAnchor,
  isoc_alt: AtomicI8,
  intr_ep: NonNull<UsbEndpointDescriptor>,
  bulk_tx_ep: NonNull<UsbEndpointDescriptor>,
  bulk_rx_ep: NonNull<UsbEndpointDescriptor>,
  isoc_tx_ep: Option<NonNull<UsbEndpointDescriptor>>,
  isoc_rx_ep: Option<NonNull<UsbEndpointDescriptor>>,
  cmd_timeout_cnt: AtomicU32,
  ctrl_anchor: UrbAnchor,              // cmd-pkt control transfers
  diag: Option<NonNull<UsbInterface>>, // Intel diag intf
  oob_wake_irq: Option<u32>,           // out-of-band wake IRQ
  diag_anchor: UrbAnchor,
  driver_info: u64,                    // BTUSB_* quirk mask
  poll_sync: bool,
  workqueue: Workqueue,
}
```

Probe path `Driver::probe(intf, id)`:
1. Validate intf altsetting 0 endpoints: at least bulk IN, bulk OUT, intr IN.
2. Allocate `btusb_data` + `hdev = hci_alloc_dev_priv(0)`.
3. `hdev->bus = HCI_USB`; `hdev->dev_type = HCI_PRIMARY` (unless BTUSB_AMP).
4. Populate `quirks` from `id->driver_info` (per-allowlisted BTUSB_* bit set).
5. Bind per-vendor setup based on driver_info:
   - `BTUSB_INTEL_COMBINED` → `hdev->setup = btusb_setup_intel_combined`.
   - `BTUSB_BCM_PATCHRAM` → `hdev->setup = btusb_setup_bcm`.
   - `BTUSB_REALTEK` → `hdev->setup = btusb_setup_realtek`.
   - `BTUSB_MEDIATEK` → `hdev->setup = btusb_mtk_setup`.
   - `BTUSB_QCA_ROME` / `_WCN6855` → `hdev->setup = btusb_setup_qca`.
6. `hci_register_dev(hdev)`.

Open path `Driver::open(hdev)`:
1. `usb_autopm_get_interface(intf)`.
2. Submit `n` bulk IN urbs (typically 8) under `bulk_anchor`.
3. Submit 1 intr IN urb under `intr_anchor`.
4. Mark hdev->flags |= HCI_RUNNING.
5. `setup(hdev)` runs (per-vendor lib).

Send frame routing `Driver::send_frame(hdev, skb)`:
```
match hci_skb_pkt_type(skb) {
  HCI_COMMAND_PKT  => submit control urb (vendor request 0x00),
  HCI_ACLDATA_PKT  => submit bulk OUT urb,
  HCI_SCODATA_PKT  => submit isoc OUT urb (ALT must be claimed),
  HCI_ISODATA_PKT  => submit isoc OUT urb on iso intf,
  _ => -EILSEQ
}
```

ISOC ALT-selection `Driver::set_isoc_alt(intf, alt)`:
1. `usb_set_interface(udev, intf->cur_altsetting->desc.bInterfaceNumber, alt)`.
2. Per-ALT iso endpoint `wMaxPacketSize` recalculated.
3. Per-alt iso IN/OUT endpoint pointers refreshed.

Per-vendor fw upload (Intel example):
1. `btintel_check_version(hdev, &ver)` reads HW + FW version via vendor-cmd `INTEL_OP_VERSION`.
2. Build fw filename `intel/ibt-<hw_variant>-<rom_version>.sfi`.
3. `request_firmware(&fw, name, &intf->dev)`.
4. Per-fw header signature check — Intel SFI carries CSE-signed header; mismatch → -EBADMSG.
5. Push fw in 252-byte chunks via `INTEL_OP_DOWNLOAD`; per-chunk wait for `HCI_EV_CMD_COMPLETE`.
6. Issue `INTEL_OP_RESET`; wait for boot complete event.
7. Re-read version to confirm new fw active.

Disconnect path `Driver::disconnect(intf)`:
1. `hci_unregister_dev(hdev)` → core ulinks index.
2. Kill all urbs (`usb_kill_anchored_urbs(&bulk_anchor)`, `&intr_anchor`, `&isoc_anchor`, `&ctrl_anchor`, `&diag_anchor`).
3. Release iso intf if claimed.
4. `hci_free_dev(hdev)`.

## Hardening

- **VID/PID allowlist** — `btusb_table[]` is exhaustive; no wildcard claim outside class 0xe0/sub 0x01.
- **Quirk allowlist** — `driver_info` restricted to BTUSB_* set; unknown bits dropped.
- **Endpoint descriptor validation** — bulk IN + bulk OUT + intr IN required; refuse otherwise.
- **ALT-setting validation** — selected ALT must exist in intf altsettings; refuse otherwise.
- **Per-fw signature mandatory** — per-vendor lib refuses tampered blobs.
- **urb credit accounting** — bulk/intr/isoc urb counts bounded; overruns refused.
- **Disconnect drains all urbs** — `usb_kill_anchored_urbs` on every anchor before free.
- **autosuspend disabled-by-default for SCO active** — refuse runtime-suspend while SCO/ISO active.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `btusb_data`, fw staging buffers, HCI skbs, and urb-transfer-buffer caches; vendor manfid + fw-version fields bounded.
- **PAX_KERNEXEC** — btusb driver in W^X kernel text; per-vendor setup vtables, `btusb_table[]`, and packet-routing dispatch live in `__ro_after_init` text.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `btusb_probe`, `send_frame`, recv-urb completions, and per-vendor setup entries.
- **PAX_REFCOUNT** — saturating `refcount_t` on per-intf `btusb_data`, urb anchors, and per-vendor fw staging; overflow trap defeats probe/disconnect race UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for fw staging blobs, urb transfer buffers, HCI cmd/event skbs, and ISO audio buffers so RF/audio data cannot bleed across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every HCI-socket entry that ultimately reaches btusb; reject user-pointer deref outside canonical helpers.
- **PAX_RAP / kCFI** — `hci_driver` vtable (`open`/`close`/`flush`/`send`/`setup`/`shutdown`), urb-completion callbacks, and per-vendor setup-dispatch marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms and per-`btusb_data`/`udev` pointer disclosure behind CAP_SYSLOG; suppress `%p` in vendor-setup tracepoints.
- **GRKERNSEC_DMESG** — restrict bdaddr, fw-version, vendor-version, and intf-claim banners to CAP_SYSLOG so attackers cannot probe device identity via dmesg.
- **USB intf claim CAP_SYS_RAWIO** — `usb_driver_claim_interface` for the auxiliary iso intf gated by CAP_SYS_RAWIO via the bus-driver allowlist; refuse claim from non-allowlisted modules.
- **ALT setting validation** — selected ALT must exist in intf altsetting array and report nonzero iso endpoint; refuse out-of-range ALT.
- **Firmware blob signature mandatory** — per-vendor lib refuses unsigned/tampered fw; signature failure aborts upload + leaves hdev unregistered.
- **Vendor-quirk allowlist** — `id->driver_info` bits restricted to defined BTUSB_* set; unknown bits dropped + ratelimited.
- **urb credit bound** — concurrent bulk/intr/isoc urb counts capped at per-controller limit; over-submit returns -ENOMEM.
- **Wake-source gating** — `intf->needs_remote_wakeup` enabled only while LE-advertising or page-scan active; refused otherwise.

Rationale: btusb is the front door for every USB Bluetooth device — including hostile devices delivered via supply-chain or BadUSB-class attacks. A missing endpoint validation, an unsigned fw blob, an attacker-set ALT setting, a runtime-mutable quirk-mask, or a leaked bdaddr/IRK in dmesg compounds into kernel takeover from a $5 dongle. RAP/kCFI on `hci_driver` + per-vendor setup, signature-mandatory fw, ALT validation, endpoint validation, quirk allowlist, refcount-overflow trapping, and wake-source gating turn btusb from "claim any Bluetooth-class intf and trust the controller" into a structurally policed boundary.

## Open Questions

(none at this Tier-3 level)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `btusb_pkt_route_no_oob` | OOB | per-skb pkt-type ∈ {COMMAND, ACL, SCO, EVENT, ISO, DRV} ∧ corresponding endpoint claimed. |
| `isoc_alt_in_range` | OOB | selected ALT < intf->num_altsetting. |
| `urb_anchor_no_uaf` | UAF | urb completion never references freed `btusb_data`; disconnect drains anchors before free. |
| `fw_chunk_index_bounded` | OOB | per-fw-upload chunk index < fw->size / chunk_size. |

### Layer 2: TLA+

`models/bluetooth/btusb_urb_lifecycle.tla` (this doc declares it): proves urb submit↔complete↔kill ordering under autosuspend, disconnect, and concurrent send_frame from HCI-core workqueue.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Driver::probe` post: hdev->bus == HCI_USB ∧ hdev->quirks ⊆ allowlisted BTUSB_* bits | `Driver::probe` |
| `Driver::send_frame` post: skb consumed (freed or anchored); return value reflects route success | `Driver::send_frame` |
| Per-vendor `setup` post: fw signature OK ∧ controller-info matches expected post-upload | `btusb_setup_*` |

### Layer 4: Verus/Creusot functional

USB probe → `btusb_probe` → quirk + endpoint validation → `hci_alloc_dev` → `hci_register_dev` → mgmt index-added → `hciconfig up` → `Driver::open` → urbs submitted → `setup` → fw upload (signature-checked) → controller running → LE scan → bulk IN urb → HCI event → mgmt scan-result. Encoded as Verus invariant chained with `00-overview.md`'s power-state machine.

## Out of Scope

- Per-vendor lib internals (`btintel`, `btbcm`, `btmtk`, `btrtl`, `btqca`) — future per-vendor Tier-3s
- UART transports (covered by `hci_uart` and `hci_qca`/`hci_intel`/etc. future Tier-3s)
- SDIO/PCI/virtio transports (future Tier-3s)
- HCI core packet handling (future `net-bluetooth-hci-core.md`)
- 32-bit-only paths
- Implementation code
