# Tier-3: drivers/nfc/ + net/nfc/ — NFC subsystem overview (transport drivers + NCI/HCI/digital cores + netlink user API)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/00-overview.md
upstream-paths:
  - drivers/nfc/
  - net/nfc/
  - include/net/nfc/nfc.h
  - include/net/nfc/nci_core.h
  - include/net/nfc/hci.h
  - include/uapi/linux/nfc.h
-->

## Summary

Near Field Communication (ISO/IEC 14443 / 18092 / FeliCa / ISO/IEC 15693) subsystem. Split into two halves: `net/nfc/` provides the protocol cores (NCI per the NFC-Forum Controller Interface spec, HCI per ETSI TS 102 622, digital protocol implementation for "soft" controllers, LLCP for peer-to-peer LLC, SE — Secure Element bridge) and the genl/socket user API; `drivers/nfc/` provides the per-controller transport drivers (USB, UART, SPI, I2C, MEI) that wrap a vendor radio + microcontroller and expose either an NCI command pipe (most modern parts) or an HCI pipe (older ST/NXP) to the core.

This Tier-3 overview catalogs the relationships, the supported transport drivers, the user-facing genl + raw-socket interfaces, and the cross-cutting hardening posture. Per-controller drivers (`pn533`, `nfcmrvl`, `s3fwrn5`, `st-nci`, `st21nfca`, `nxp-nci`, `pn544`, `microread`, `port100`, `trf7970a`, `st95hf`, `mei_phy`, `nfcsim`, `virtual_ncidev`, `fdp`) inherit this contract; the NCI core is documented in `nci.md`.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct nfc_dev` (`include/net/nfc/nfc.h`) | per-controller device | `drivers::nfc::Device` |
| `struct nfc_ops` | per-driver vtable (`dev_up`/`dev_down`/`start_poll`/`activate_target`/`im_transceive`/`tm_send`/`fw_download`/`se_io`) | `drivers::nfc::Ops` |
| `nfc_allocate_device(ops, supported_protocols, tx_headroom, tx_tailroom)` | alloc skeleton | `Device::alloc` |
| `nfc_register_device(dev)` / `_unregister_device(dev)` | register/unregister | `Device::register` / `_unregister` |
| `struct nci_dev` (`include/net/nfc/nci_core.h`) | NCI-mode child of `nfc_dev` | `drivers::nfc::nci::Dev` |
| `nci_allocate_device(ops, supported_protocols, tx_headroom, tx_tailroom)` | NCI alloc | `nci::Dev::alloc` |
| `nci_register_device(ndev)` / `_unregister_device(ndev)` | NCI register/unregister | `nci::Dev::register` / `_unregister` |
| `struct nfc_hci_dev` (`include/net/nfc/hci.h`) | HCI-mode child of `nfc_dev` | `drivers::nfc::hci::Dev` |
| `nfc_hci_allocate_device(...)` / `_register_device(...)` | HCI alloc/register | `hci::Dev::*` |
| `nfc_genl_*` (`net/nfc/netlink.c`) | NFC_GENL_NAME generic-netlink family | `subsystem::genl::*` |
| `nfc_llcp_register_device(...)` (`net/nfc/llcp/`) | LLCP socket plumbing | `subsystem::llcp::Device` |
| `nfc_add_se(dev, type)` / `_remove_se(dev, idx)` | Secure-Element enumeration | `Device::add_se` / `_remove_se` |

## Compatibility contract

REQ-1: One controller — one `nfc_dev`. Drivers allocate via `nfc_allocate_device` (raw mode), `nci_allocate_device` (NCI mode), or `nfc_hci_allocate_device` (HCI mode); only one is permitted per controller.

REQ-2: Per-controller protocol set declared at alloc: `NFC_PROTO_JEWEL`, `_MIFARE`, `_FELICA`, `_ISO14443`, `_ISO14443_B`, `_NFC_DEP`, `_ISO15693`. Driver must reject `start_poll(im_protocols)` outside its declared set.

REQ-3: Per-transport tx_headroom + tx_tailroom: NCI adds 3-byte header + 4-byte CRC for UART, USB requires no headroom but vendor LLC may; HCI adds gate/pipe wrappers. Driver declares at alloc; core enforces on `skb` push.

REQ-4: Genl user API (`net/nfc/netlink.c`): `NFC_CMD_DEV_UP`, `_DEV_DOWN`, `_START_POLL`, `_STOP_POLL`, `_DEP_LINK_UP`, `_GET_TARGET`, `_FW_DOWNLOAD`, `_SE_IO`, `_GET_SE`, `_ENABLE_SE`, `_DISABLE_SE`. Each command requires CAP_NET_ADMIN (cross-ref `net/nfc/netlink.c`).

REQ-5: Raw socket family `AF_NFC` exposed via `net/nfc/af_nfc.c`; socket types `SOCK_RAW` (target i/o), `SOCK_DGRAM` (LLCP datagram), `SOCK_STREAM` (LLCP stream). Per-socket protocol = `NFC_SOCKPROTO_RAW` / `_LLCP`.

REQ-6: Firmware download path: `dev_up(NULL)` → `fw_download(fw_name)` → driver pulls firmware via `request_firmware()`; per-driver signature/checksum check is mandatory before flashing radio MCU.

REQ-7: Polling state machine: `IDLE` → `POLL_ACTIVE` (im_protocols) / `LISTEN_ACTIVE` (tm_protocols) → `TARGET_PRESENT` → activate → `DATA_EXCHANGE` → deactivate → `IDLE`. State held in `nfc_dev.dev_lock` mutex.

REQ-8: Secure Element bridge: per-`nfc_dev`-attached SE enumerated as type `NFC_SE_UICC` / `_EMBEDDED` / `_DISABLED`; SE_IO carries APDU via `se_io(se_idx, apdu, apdu_len, cb)` with per-SE state machine.

REQ-9: RF parameter constraints: poll period bounded by `NFC_RF_DISC_PERIOD_MIN`/`_MAX`; per-protocol bit-rate vector validated against controller capability blob received in `CORE_INIT_RSP` (NCI) or `ADMIN_GATE` reply (HCI).

REQ-10: Per-`nfc_dev` rfkill integration: `dev_up`/`_down` honors RFKILL_TYPE_NFC global block; userspace `rfkill` toggles propagate via `rfkill_set_states`.

## Acceptance Criteria

- [ ] AC-1: `lsmod | grep nfc` shows core (`nfc.ko`) + at least one transport on supported HW; `dmesg | grep nfc` shows `nfc_register_device` for each enumerated controller.
- [ ] AC-2: `neard` daemon enumerates controllers via genl; `mdbus2 org.neard /` returns one path per controller.
- [ ] AC-3: ISO/IEC 14443-A poll test (e.g. NTAG215) succeeds: `nfctool -p 0` discovers target; `nfctool -t 0 -r <APDU>` returns response.
- [ ] AC-4: P2P LLCP test between two NCI-mode controllers exchanges SNEP message via `neardal`.
- [ ] AC-5: Firmware download test (`nfctool -d --fw <name>`) succeeds for at least one supported part (`s3fwrn5`, `nfcmrvl`).
- [ ] AC-6: SE enumeration test on a UICC-equipped controller (`st-nci`) returns the UICC index via genl.
- [ ] AC-7: CAP_NET_ADMIN missing → genl returns `-EPERM` on every command verb (unit-tested via `nfc-test`).
- [ ] AC-8: kselftest `tools/testing/selftests/nfc/` passes (currently a smoke-test against `nfcsim`/`virtual_ncidev`).

## Architecture

`Subsystem` lives in `drivers::nfc::Subsystem`:

```
struct Subsystem {
  class: KBox<Class>,                  // /sys/class/nfc/
  genl: GenericNetlinkFamily,          // NFC_GENL_NAME, CAP_NET_ADMIN gated
  socket_protocols: [SocketProto; 2],  // RAW + LLCP
  devices: Mutex<XArray<u32, Arc<Device>>>,
  llcp: KBox<llcp::Subsystem>,
  rfkill_class: NonNull<RfkillClass>,  // RFKILL_TYPE_NFC
}

struct Device {
  refcount: Refcount,
  ops: &'static Ops,
  supported_protocols: u32,
  tx_headroom: u16,
  tx_tailroom: u16,
  state: Mutex<DeviceState>,           // IDLE / POLL / LISTEN / ACTIVE
  targets: Vec<Target>,
  lock: Mutex<()>,                     // dev_lock
  rfkill: NonNull<RfkillNode>,
  se_list: Mutex<Vec<SecureElement>>,
  llcp_local: Option<Arc<llcp::Local>>,
}
```

Driver registration `Device::register`:
1. Validate `ops` vtable shape (no NULLs in required slots: `dev_up`, `dev_down`, `start_poll`).
2. Assign per-subsystem device-index via `IDA`.
3. Create `/sys/class/nfc/nfcN` + sysfs attributes (`protocols`, `targets`, `se*`).
4. Register rfkill node under RFKILL_TYPE_NFC.
5. Broadcast `NFC_EVENT_DEVICE_ADDED` via genl multicast group `nfc_mcgrp_events`.

Polling start `Device::start_poll(im_protocols, tm_protocols)`:
1. Validate `(im_protocols | tm_protocols)` ⊆ `supported_protocols`; reject otherwise.
2. Transition state IDLE → POLL_ACTIVE under `lock`.
3. Call `ops.start_poll(dev, im_protocols, tm_protocols)` → driver issues vendor-specific (or NCI `RF_DISCOVER_CMD`) discovery request.
4. On target found, driver calls `nfc_targets_found(dev, targets, n_targets)` → state → TARGET_PRESENT.
5. Genl multicast `NFC_EVENT_TARGETS_FOUND` to listeners.

Transport drivers (per `drivers/nfc/`):

| Transport | Driver dir/file | Protocol | Notes |
|---|---|---|---|
| `fdp` | `drivers/nfc/fdp/` | NCI | Intel Field Detect Platform (USB) |
| `mei_phy` | `drivers/nfc/mei_phy.c` | HCI | Intel MEI (CSME) bridge |
| `microread` | `drivers/nfc/microread/` | HCI | Inside Secure MicroRead (i2c, MEI) |
| `nfcmrvl` | `drivers/nfc/nfcmrvl/` | NCI | Marvell 88W8987 (USB, UART, I2C, SPI) |
| `nfcsim` | `drivers/nfc/nfcsim.c` | NFC sim | Loopback for CI |
| `nxp-nci` | `drivers/nfc/nxp-nci/` | NCI | NXP PN7150 (I2C) |
| `pn533` | `drivers/nfc/pn533/` | proprietary | NXP PN533 (USB, UART) — pre-NCI |
| `pn544` | `drivers/nfc/pn544/` | HCI | NXP PN544 (I2C, MEI) |
| `port100` | `drivers/nfc/port100.c` | proprietary | Sony RC-S380 (USB) |
| `s3fwrn5` | `drivers/nfc/s3fwrn5/` | NCI | Samsung S3FWRN5 (I2C, UART) |
| `st-nci` | `drivers/nfc/st-nci/` | NCI | ST ST21NFCC, ST54J (I2C, SPI) |
| `st21nfca` | `drivers/nfc/st21nfca/` | HCI | ST ST21NFCA (I2C) — pre-NCI |
| `st95hf` | `drivers/nfc/st95hf/` | digital | ST ST95HF (SPI) — soft NCI via digital core |
| `trf7970a` | `drivers/nfc/trf7970a.c` | digital | TI TRF7970A (SPI) — soft NCI |
| `virtual_ncidev` | `drivers/nfc/virtual_ncidev.c` | NCI | `/dev/virtual_nci` for userspace controllers |

## Hardening

- **CAP_NET_ADMIN on every genl command** — refuse unprivileged userspace from initiating poll, opening SE, or triggering FW download.
- **Per-controller `supported_protocols` mask** — driver-declared at alloc; core rejects `start_poll` for masks the radio cannot actually drive.
- **Per-transport allowlist** — module autoload restricted to controllers present at boot or hot-plugged with a USB/I2C ID match; refuse arbitrary `nci_register_device` from outside the controller-driver allowlist.
- **RF parameter validation** — poll-period, listen-period, bit-rate, max-data-rate, max-data-length validated against capability blob in `CORE_INIT_RSP` / `ADMIN_GATE`-reply.
- **Firmware blob signing** — `request_firmware` consumes blobs from `/lib/firmware/`; per-driver header signature and CRC checked before flashing radio MCU.
- **rfkill integration** — userspace `rfkill block all` blocks every NFC controller; per-controller airplane-mode honored across `dev_up`/`_down`.
- **State machine under mutex** — `nfc_dev.dev_lock` serializes user genl, driver IRQ callbacks, and async target callbacks.
- **SE APDU length cap** — per-`se_io` APDU bounded by `NFC_MAX_SE_APDU_LEN`.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `nfc_dev`, `nci_dev`, `nfc_hci_dev`, `nfc_target`, `secure_element`, and genl reply buffers; per-target manfid blobs strictly bounded.
- **PAX_KERNEXEC** — NFC core and per-transport drivers in W^X kernel text; `nfc_ops`, `nci_ops`, and genl-policy tables live in `__ro_after_init` text.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `nfc_genl_*`, `start_poll`, target-activate, `im_transceive`, and skb-recv entries.
- **PAX_REFCOUNT** — saturating `refcount_t` on `nfc_dev`, `nci_dev`, `nfc_target`, LLCP socket, and SE references; overflow trap defeats poll/activate/teardown UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for target buffers, APDU caches, NCI/HCI skbs, firmware-download staging pages, and SE state so RF-captured data cannot bleed across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every genl + raw-socket entry into the NFC subsystem; reject user-pointer deref outside canonical helpers.
- **PAX_RAP / kCFI** — `nfc_ops`, `nci_ops`, `nci_driver_ops`, HCI gate-callbacks, and genl-policy dispatch vtables marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms and per-target/per-SE pointer disclosure behind CAP_SYSLOG; suppress `%p` in poll/activate tracepoints.
- **GRKERNSEC_DMESG** — restrict target-found, SE-attach, and firmware-download banners to CAP_SYSLOG so attackers cannot probe tag presence via dmesg.
- **CAP_NET_ADMIN strict** — every genl command verb gated by CAP_NET_ADMIN in the owning user namespace; refuse from non-init user-ns unless explicit allowlist.
- **Transport allowlist** — `nfc_register_device` accepts only drivers from the platform allowlist; refuse arbitrary controller registration from non-allowlisted modules.
- **RF parameter validation** — poll-period, bit-rate, max-data-length validated against `CORE_INIT_RSP` capability blob; refuse out-of-spec parameters.
- **Firmware signature mandatory** — refuse `fw_download` blobs without a valid per-vendor signature; CRC failure aborts flash.
- **SE APDU bound** — every `se_io` APDU capped at `NFC_MAX_SE_APDU_LEN`; refuse larger transfers to defeat radio-side overflow.
- **rfkill enforced** — `dev_up` refuses while RFKILL_TYPE_NFC global block set; per-controller airplane-mode honored without exception.

Rationale: NFC's radio is a near-field antenna driving an opaque MCU on the controller die; a missing CAP_NET_ADMIN check on `NFC_CMD_FW_DOWNLOAD`, an unbounded SE APDU, or a misvalidated RF parameter is enough for a co-resident user to flash the radio MCU or to drive the RF field outside spec. RAP/kCFI on `nfc_ops`/`nci_ops`, refcount-overflow trapping, rfkill enforcement, firmware-signature mandatory, and SE APDU bounds turn the NFC stack from "user-space talks to a black box" into a structurally policed boundary.

## Open Questions

(none at this Tier-3 level)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `nfc_dev_proto_no_oob` | OOB | per-driver protocol mask bounded; never exceeds `NFC_PROTO_MAX`. |
| `nfc_genl_policy_no_oob` | OOB | per-command nl_policy access index-bounded by `NFC_ATTR_MAX`. |
| `target_list_no_uaf` | UAF | `Arc<Target>` outlives every async im_transceive callback; deactivate drains before free. |
| `se_io_buf_no_overflow` | OOB | per-SE APDU length capped at `NFC_MAX_SE_APDU_LEN`. |

### Layer 2: TLA+

`models/nfc/poll_state.tla` (this overview declares it): proves IDLE↔POLL↔TARGET_PRESENT↔ACTIVE↔IDLE state-machine deadlock-freedom across genl, driver IRQ, and async target callbacks under `dev_lock`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Device::start_poll` post: state ∈ {POLL_ACTIVE, LISTEN_ACTIVE} ∧ `im_protocols` ⊆ `supported_protocols` | `Device::start_poll` |
| `Device::add_se` post: every SE has unique idx within device | `Device::add_se` |
| Genl entry pre: caller has CAP_NET_ADMIN in target user-ns | every `nfc_genl_*` handler |

### Layer 4: Verus/Creusot functional

Userspace `NFC_CMD_START_POLL` → genl handler → `Device::start_poll` → driver `start_poll` → controller RF discover → IRQ → `nfc_targets_found` → genl multicast `NFC_EVENT_TARGETS_FOUND`. Encoded as Verus invariant chained with per-driver Tier-3s.

## Out of Scope

- NCI core internals (covered in `nci.md`)
- HCI core internals (future `hci.md` Tier-3)
- LLCP protocol (future `llcp.md` Tier-3)
- Digital protocol soft-NCI (future `digital.md` Tier-3)
- Per-controller specifics beyond the transport-list overview
- 32-bit-only paths
- Implementation code
