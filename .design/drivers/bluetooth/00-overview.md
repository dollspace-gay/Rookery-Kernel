# Tier-3: drivers/bluetooth/ — HCI transports overview (USB, UART, SDIO, virtio)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/00-overview.md
upstream-paths:
  - drivers/bluetooth/
  - include/net/bluetooth/hci.h
  - include/net/bluetooth/hci_core.h
  - include/net/bluetooth/hci_sync.h
-->

## Summary

`drivers/bluetooth/` contains the per-controller HCI transport drivers — the layer that adapts the Bluetooth Host-Controller Interface (Bluetooth Core spec § 4) onto the physical bus the controller actually sits on (USB, UART, SDIO, SPI, virtio, PCI). The HCI core itself (cmd/event dispatch, sync helpers, conn mgmt, L2CAP, SCO, ISO, RFCOMM, SMP, MGMT) lives in `net/bluetooth/` and is covered by separate Tier-3s. Each transport driver instantiates `struct hci_dev`, registers `hci_uart_proto` or USB intf, and routes HCI packet types (`HCI_COMMAND_PKT`=0x01, `HCI_ACLDATA_PKT`=0x02, `HCI_SCODATA_PKT`=0x03, `HCI_EVENT_PKT`=0x04, `HCI_ISODATA_PKT`=0x05, `HCI_DRV_PKT`=0x07) up to / down from the core.

This Tier-3 overview catalogs the transport drivers (`btusb`, `btintel`/`btintel_pcie`, `btbcm`, `btmtk`/`btmtksdio`/`btmtkuart`, `btrtl`, `btqca`/`btqcomsmd`, `btsdio`/`btmrvl_sdio`, `hci_uart`/`hci_ldisc` family, `hci_vhci`, `virtio_bt`, `bfusb`, `bpa10x`, `bt3c_cs`/`bluecard_cs`/`dtl1_cs` PCMCIA legacy, `ath3k`, `bcm203x`), describes the firmware/vendor-quirk discipline shared across them, and fixes the hardening contract that per-controller Tier-3s (e.g. `btusb.md`) inherit.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct hci_dev` (`include/net/bluetooth/hci_core.h`) | per-controller HCI device | `bluetooth::HciDev` |
| `struct hci_driver` (`include/net/bluetooth/hci_drv.h`) | per-driver vtable (`open`/`close`/`flush`/`send`/`setup`/`shutdown`/`set_diag`) | `bluetooth::HciDriver` |
| `hci_alloc_dev(priv_size)` / `_alloc_dev_priv(priv_size)` | alloc | `HciDev::alloc` |
| `hci_register_dev(hdev)` / `_unregister_dev(hdev)` | register/unregister with core | `HciDev::register` / `_unregister` |
| `hci_recv_frame(hdev, skb)` | transport→core inbound frame | `HciDev::recv_frame` |
| `hci_send_frame(hdev, skb)` | core→transport outbound frame | `HciDev::send_frame` |
| `hci_uart_proto` (`hci_uart.h`) | per-UART-line-discipline vtable | `bluetooth::HciUartProto` |
| `hci_uart_register_proto(p)` / `_unregister_proto(p)` | per-protocol register (H4, BCSP, ATH3K, LL, Intel, ...) | `HciUartProto::register` |
| `btintel_*` (`drivers/bluetooth/btintel.c`) | Intel vendor lib (HCI cmd extensions, fw upload) | `bluetooth::vendor::intel` |
| `btbcm_*` (`drivers/bluetooth/btbcm.c`) | Broadcom vendor lib (patchram upload) | `bluetooth::vendor::bcm` |
| `btmtk_*` (`drivers/bluetooth/btmtk.c`) | MediaTek vendor lib | `bluetooth::vendor::mtk` |
| `btrtl_*` (`drivers/bluetooth/btrtl.c`) | Realtek vendor lib (rtl_fw_cfg) | `bluetooth::vendor::rtl` |
| `btqca_*` (`drivers/bluetooth/btqca.c`) | Qualcomm vendor lib (nvm + ramfw) | `bluetooth::vendor::qca` |
| `hci_recv_pkt_type` (`hci.h`) | HCI packet-type enum | `bluetooth::PacketType` |
| `btmgmt` / `mgmt_*` (`net/bluetooth/mgmt.c`) | mgmt socket (userspace control plane) | `bluetooth::Mgmt` |

## Compatibility contract

REQ-1: Per-controller `hci_dev` allocated by transport driver; `hci_dev.bus` set to `HCI_USB`/`HCI_UART`/`HCI_SDIO`/`HCI_SPI`/`HCI_VIRTIO`/`HCI_PCI`/`HCI_VHCI`. `hci_dev.dev_type` set to `HCI_PRIMARY` (BR/EDR + LE) or `HCI_AMP`.

REQ-2: Per-driver `hci_driver` vtable: `open(hdev)` (initial open, fw upload, controller-info read), `close(hdev)`, `flush(hdev)` (drop pending), `send(hdev, skb)` (route per packet type), `setup(hdev)` (per-vendor init cmds), `shutdown(hdev)` (driver-specific quiesce), optional `cmd_timeout(hdev)`/`prevent_wake(hdev)`.

REQ-3: Packet-type routing on send/recv: per-skb `hci_skb_pkt_type(skb)` set before `hci_send_frame`/`hci_recv_frame`; transport driver must validate type ∈ {COMMAND, ACL, SCO, EVENT, ISO, DRV} and route to the appropriate bus endpoint.

REQ-4: Firmware upload contract: per-vendor `setup` callback invokes `request_firmware(&fw, fw_name, &hdev->dev)` and pushes opcode-specific upload commands (Intel `INTEL_OP_DOWNLOAD`, BCM `BCM_VENDOR_PATCH_RAM`, Realtek `RTL_OP_DOWNLOAD`, MTK `MTK_OP_DOWNLOAD`). Firmware signature mandatory per vendor — refuse upload otherwise.

REQ-5: Vendor quirk mask: `hci_dev.quirks` bitmap (`HCI_QUIRK_*` in `hci.h`) carries per-controller workarounds — `RAW_DEVICE`, `RESET_ON_CLOSE`, `INVALID_BDADDR`, `STRICT_DUPLICATE_FILTER`, `FIXUP_INQUIRY_MODE`, `BROKEN_LE_RESOLV_LIST`, etc. Per-driver populates quirks based on detected hw revision before `hci_register_dev`.

REQ-6: Per-bus probe IDs (USB VID/PID, UART of_match, SDIO sdio_device_id) constrain which controllers each driver claims; allowlist enforced via per-driver id-table.

REQ-7: USB-specific ALT setting: SCO/ISO audio data uses USB isochronous endpoints; per-controller `hdev->voice_setting` selects 8-bit / 16-bit μ-law / a-law / CVSD / mSBC; transport driver validates supported ALT setting against USB descriptor before claiming intf.

REQ-8: Per-`hci_dev` rfkill node under `RFKILL_TYPE_BLUETOOTH`; userspace `rfkill block bluetooth` propagates to per-controller `hdev->flags & HCI_RFKILLED`.

REQ-9: HCI socket (AF_BLUETOOTH + BTPROTO_HCI): raw HCI access via `HCI_CHANNEL_RAW`/`_USER`/`_MONITOR`/`_CONTROL`/`_LOGGING`; per-channel CAP_NET_ADMIN gating (RAW + USER require it; CONTROL = btmgmt requires it).

REQ-10: Per-controller `hci_dev.power_state` machine: `OFF` → `INIT` → `RUNNING` → `SUSPEND` → `RESUME` → `OFF`. Setup runs on `INIT`; shutdown on `OFF`. Transitions serialized under `hci_dev.req_lock`.

## Acceptance Criteria

- [ ] AC-1: `lsusb` shows a Bluetooth controller (class 0xe0 / subclass 0x01); `btusb.ko` autoloads; `hci0` appears.
- [ ] AC-2: `hciconfig hci0 up` succeeds; controller-info read returns valid `bdaddr`, manufacturer-id, lmp_version.
- [ ] AC-3: `bluetoothctl scan on` discovers nearby devices; LE advertising report events flow.
- [ ] AC-4: SCO audio path: HFP call via `pulseaudio` routes audio through HCI SCO; no underrun under steady playback.
- [ ] AC-5: Firmware reload test (`rmmod btusb && modprobe btusb`) re-runs vendor setup; `dmesg | grep -i 'firmware'` shows clean upload.
- [ ] AC-6: rfkill test: `rfkill block bluetooth` → `hci0` flagged HCI_RFKILLED; `rfkill unblock` restores.
- [ ] AC-7: btmon trace (`btmon -t`) captures cmd/event/ACL flow without packet corruption.
- [ ] AC-8: `mgmt --enable-le` toggles LE mode; subsequent `hci_le_scan_start` succeeds.
- [ ] AC-9: PR-2 (firmware-signature mandatory) tested: tampered firmware blob refused with `dmesg`-logged signature failure.

## Architecture

`bluetooth::Subsystem` lives in `drivers::bluetooth::Subsystem`:

```
struct Subsystem {
  class: KBox<Class>,                  // /sys/class/bluetooth/
  hci_devices: Mutex<XArray<u32, Arc<HciDev>>>,
  hci_proto: Mutex<Vec<&'static HciUartProto>>,
  mgmt_index: AtomicU32,
  rfkill_class: NonNull<RfkillClass>,  // RFKILL_TYPE_BLUETOOTH
}

struct HciDev {
  refcount: Refcount,
  driver: &'static HciDriver,
  bus: u8,                             // HCI_USB / HCI_UART / ...
  dev_type: u8,                        // HCI_PRIMARY / HCI_AMP
  flags: AtomicU64,
  quirks: u64,
  manufacturer: u16,
  hw_version: u32,
  fw_version: u32,
  bdaddr: BdAddr,
  power_state: Mutex<PowerState>,
  req_lock: Mutex<()>,
  cmd_q: SkbQueue,
  rx_q: SkbQueue,
  raw_q: SkbQueue,
  acl_pkts: AtomicU16,
  sco_pkts: AtomicU16,
  iso_pkts: AtomicU16,
  conn_hash: Mutex<HashMap<u16, Arc<HciConn>>>,
  rfkill: NonNull<RfkillNode>,
  workqueue: Workqueue,
  driver_data: NonNull<u8>,
}
```

Transport-driver register `HciDev::register`:
1. Validate `driver` vtable shape (open, close, send required).
2. Assign per-subsystem index via IDA → `hci_dev.id`.
3. Create `/sys/class/bluetooth/hci<id>/` + sysfs attrs (`type`, `address`, `name`, `manufacturer`).
4. Register rfkill node under RFKILL_TYPE_BLUETOOTH.
5. Broadcast index-added via mgmt socket.
6. Auto-power-on if not flagged HCI_RFKILLED.

Per-bus dispatch:
- **USB** (`btusb.c`): per-intf probe matches against `btusb_table[]` (VID/PID + driver_info quirk-bits). Bulk IN/OUT for ACL+EVENT, ISOC IN/OUT for SCO/ISO, INTR IN for events on some controllers.
- **UART** (`hci_uart.c` + per-proto `hci_h4`/`hci_h5`/`hci_bcsp`/`hci_ll`/`hci_ath`/`hci_intel`/`hci_qca`/`hci_bcm4377`/`hci_aml`/`hci_nokia`): N_HCI tty line-discipline; per-proto framing on the wire.
- **SDIO** (`btsdio.c`, `btmtksdio.c`, `btmrvl_sdio.c`): SDIO function-2 cmd53 block read/write; per-vendor firmware download path.
- **SPI** (`hci_bcm4377.c` SPI variant): chip-select toggle + per-frame header.
- **virtio** (`virtio_bt.c`): VIRTIO_ID_BT; tx/rx virtqueues.

Firmware upload path (per-vendor `setup`):
1. Read controller version (`HCI_OP_READ_LOCAL_VERSION` or vendor-specific opcode).
2. Build firmware filename from manufacturer + hw_version + fw_id.
3. `request_firmware(&fw, fw_name, &hdev->dev)`.
4. Per-vendor signature check on fw blob — reject if mismatched.
5. Push fw via chunked vendor-specific HCI cmds; wait for per-chunk ack event.
6. Issue vendor-specific "fw apply" cmd; wait for `HCI_EV_CMD_COMPLETE` with status 0.
7. Re-read controller version to confirm new fw is live.

Power state transitions:
- `OFF` → `INIT`: `driver.open(hdev)` → `driver.setup(hdev)` → `HCI_OP_RESET` → controller-info reads.
- `INIT` → `RUNNING`: HCI core enables event reporting + scan parameters.
- `RUNNING` → `SUSPEND`: `mgmt_set_suspend_state` → controller put into suspend per `HCI_OP_LE_SUSPEND` (LE) or hci scan disable.
- `SUSPEND` → `RESUME`: reverse; restore filters/advertising/connections.
- `RUNNING` → `OFF`: `driver.shutdown(hdev)` → `driver.close(hdev)`.

## Hardening

- **Per-bus probe id-table** — drivers claim only allowlisted VID/PID / of_match / sdio_device_id.
- **Vendor-quirk allowlist** — `quirks` populated from vendor table at probe; refuse uncontrolled bit-set.
- **Firmware signature mandatory** — per-vendor signature check before fw upload; failed sig refuses to flash.
- **Power-state mutex** — `req_lock` serializes setup/shutdown across transports.
- **rfkill enforcement** — `dev_up` refuses while RFKILL_TYPE_BLUETOOTH globally blocked.
- **HCI socket channel CAP gate** — RAW/USER/CONTROL channels require CAP_NET_ADMIN.
- **Per-controller packet credit accounting** — `acl_pkts`/`sco_pkts`/`iso_pkts` saturate against controller-reported limit.
- **Vendor manfid PAX_USERCOPY whitelist** — per-controller manufacturer-data field disclosure bounded.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `hci_dev`, `hci_conn`, HCI skbs, vendor-fw staging buffers, and `hci_uart` framing buffers; per-controller manfid blob bounded.
- **PAX_KERNEXEC** — Bluetooth transport drivers in W^X kernel text; `hci_driver`, `hci_uart_proto` tables, and per-vendor setup vtables live in `__ro_after_init` text.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `hci_recv_frame`, `hci_send_frame`, `setup`, `shutdown`, and rx-workqueue entries.
- **PAX_REFCOUNT** — saturating `refcount_t` on `hci_dev`, `hci_conn`, mgmt session, and HCI sockets; overflow trap defeats teardown UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for HCI skbs, vendor-fw staging, controller-info caches, and per-conn keys/IRK so RF/audio data cannot bleed across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every HCI-socket + mgmt-socket + sysfs entry; reject user-pointer deref outside canonical helpers.
- **PAX_RAP / kCFI** — `hci_driver`, `hci_uart_proto`, mgmt-cmd dispatch, and event-handler tables marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms and per-`hci_dev` pointer disclosure behind CAP_SYSLOG; suppress `%p` in cmd/event/conn tracepoints.
- **GRKERNSEC_DMESG** — restrict scan-result, bdaddr, IRK, and firmware-upload banners to CAP_SYSLOG so attackers cannot probe device presence + identity via dmesg.
- **HCI CAP_NET_ADMIN strict** — every HCI socket channel (RAW, USER, CONTROL, LOGGING) requires CAP_NET_ADMIN; MONITOR gated by CAP_NET_RAW + audit-logged.
- **Firmware signature mandatory** — every per-vendor `setup` refuses unsigned/tampered fw blobs; CRC/signature failure aborts the upload.
- **Transport-specific quirk-mask** — `hci_dev.quirks` populated from vendor table at probe; runtime mutation refused; unknown quirk bits dropped.
- **manfid PAX_USERCOPY** — per-controller manfid, lmp_subversion, and hw_variant fields disclosed via mgmt socket only over a slab usercopy-whitelist window.
- **rfkill enforced** — `dev_up` refuses while RFKILL_TYPE_BLUETOOTH global block set; airplane-mode honored without exception.
- **Power-state mutex enforced** — `req_lock` taken before every state transition; concurrent open/close from genl-races trapped.

Rationale: Bluetooth attack surface comes both from radio peers (BlueBorne-class) and from per-vendor firmware blobs flashed into the controller MCU. Missing CAP_NET_ADMIN on `HCI_CHANNEL_USER`, a tampered firmware blob accepted by `setup`, a runtime-mutable quirk-mask, or a leaked IRK in dmesg is enough to take over the host. RAP/kCFI on `hci_driver`/`hci_uart_proto`, CAP_NET_ADMIN on every channel, signature-mandatory firmware, fixed quirk-mask, and IRK suppression in dmesg turn the Bluetooth transport layer from "trust the controller, trust the radio peers" into a structurally enforced boundary.

## Open Questions

(none at this Tier-3 level)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `hci_skb_pkt_type_no_oob` | OOB | per-skb packet type ∈ {COMMAND, ACL, SCO, EVENT, ISO, DRV}. |
| `usb_alt_setting_no_oob` | OOB | claimed USB ALT setting ≤ intf->num_altsetting. |
| `hci_conn_hash_no_uaf` | UAF | `Arc<HciConn>` outlives per-conn rx callback; teardown drains conn-hash before free. |
| `vendor_fw_chunk_bounded` | OOB | per-fw-upload chunk index < fw->size / chunk_size. |

### Layer 2: TLA+

`models/bluetooth/power_state.tla` (this overview declares it): proves OFF↔INIT↔RUNNING↔SUSPEND↔RESUME state machine deadlock-freedom across mgmt cmds, rfkill toggles, suspend/resume callbacks, and IRQ-recv concurrency.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `HciDev::register` post: id is unique within Subsystem; `/sys/class/bluetooth/hci<id>` exists | `HciDev::register` |
| HCI socket entry pre: channel.requires_cap() ⊂ current caps | every `hci_sock_*` handler |
| Per-vendor `setup` post: fw signature OK ∧ controller_version matches expected post-upload | per-vendor `setup` |

### Layer 4: Verus/Creusot functional

`hciconfig hci0 up` → mgmt cmd → power_state INIT → `driver.open(hdev)` → `driver.setup(hdev)` → fw upload → HCI_OP_RESET → controller info → power_state RUNNING → mgmt event index-added. Encoded as Verus invariant chained with per-driver Tier-3s.

## Out of Scope

- HCI core internals (covered by future `net-bluetooth-hci-core.md`)
- L2CAP/SCO/ISO/RFCOMM/SMP/MGMT (future Tier-3s under `net/bluetooth/`)
- Per-controller-specific firmware-upload protocols (per-driver Tier-3s, e.g. `btusb.md`)
- 32-bit-only paths
- Implementation code
