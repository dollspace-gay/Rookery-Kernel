# Tier-3: net/bluetooth/hci_core.c — Bluetooth Host Controller Interface (HCI) core

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/bluetooth/00-overview.md
upstream-paths:
  - net/bluetooth/hci_core.c (~4166 lines)
  - net/bluetooth/hci_event.c
  - net/bluetooth/hci_conn.c
  - net/bluetooth/hci_sync.c
  - include/net/bluetooth/hci.h
  - include/net/bluetooth/hci_core.h
-->

## Summary

Bluetooth HCI core provides per-controller (`hci_dev`) lifecycle + cmd/event flow + connection mgmt + L2CAP/SCO/ISO transport plumbing. Per-hci_dev binds to a transport-driver (USB, UART, virtio-bt). Per-cmd: kernel sends HCI command-packet via `hci_send_cmd`; per-cmd-status / cmd-complete event acks. Per-ACL/SCO/ISO: data packets per-connection. Per-event-handler dispatches to per-event-code handler in hci_event.c. HCI-sync command-flow gates pre-/post-power-on init sequences. Critical for: any Bluetooth driver, BlueZ userspace, audio (HFP/A2DP), peripherals.

This Tier-3 covers `hci_core.c` (~4166 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct hci_dev` | per-controller state | `HciDev` |
| `hci_register_dev()` | per-driver register controller | `Hci::register_dev` |
| `hci_unregister_dev()` | per-driver unregister | `Hci::unregister_dev` |
| `hci_dev_open()` | per-controller power-on | `Hci::dev_open` |
| `hci_dev_close()` | per-controller power-off | `Hci::dev_close` |
| `hci_send_cmd()` | per-cmd send | `Hci::send_cmd` |
| `hci_send_acl()` | per-ACL data send | `Hci::send_acl` |
| `hci_send_sco()` | per-SCO audio send | `Hci::send_sco` |
| `hci_recv_frame()` | per-pkt RX from driver | `Hci::recv_frame` |
| `hci_event_packet()` | per-event dispatch | `Hci::event_packet` |
| `hci_acldata_packet()` | per-ACL data dispatch | `Hci::acldata_packet` |
| `hci_scodata_packet()` | per-SCO data dispatch | `Hci::scodata_packet` |
| `hci_send_frame()` | per-driver tx callback | `HciDriver::send_frame` |
| `hci_init_sync()` | per-controller init | `Hci::init_sync` |
| `hci_dev_set_drvdata()` | per-driver private | `Hci::set_drvdata` |
| `hci_le_set_*` | per-BLE setters | `Hci::le_*` |
| `hci_register_cb()` | per-event-callback | `Hci::register_cb` |
| `HCI_ACLDATA_PKT` / `_SCODATA_PKT` / `_EVENT_PKT` / `_COMMAND_PKT` / `_ISODATA_PKT` | pkt types | UAPI |

## Compatibility contract

REQ-1: Per-hci_dev:
- type: HCI_PRIMARY (Classic + LE), HCI_AMP.
- bus: HCI_VIRTUAL / HCI_USB / HCI_PCCARD / HCI_UART / HCI_RS232 / HCI_USB_BCM / HCI_USB_INTEL / etc.
- name: "hci0".
- dev_addr: BD_ADDR (6 bytes).
- features[8]: per-Classic features.
- le_features[8]: per-LE features.
- supported commands[64]: per-HCI cmd-table.

REQ-2: hci_register_dev(hdev):
- Per-driver alloc'd hdev populated.
- Allocate hci_id (atomic).
- hci_init_sysfs(hdev): per-/sys/class/bluetooth/hci<id>.
- list_add_rcu(&hdev.list, &hci_dev_list).
- hci_dev_hold reference.

REQ-3: hci_unregister_dev:
- list_del_rcu(&hdev.list).
- Per-conn_hash drain.
- per-cb teardown.
- hci_dev_put.

REQ-4: hci_dev_open(dev_idx):
- hdev = hci_dev_get(dev_idx).
- err = hci_dev_open_sync(hdev): per-driver hdev.open + per-init-sequence (Reset, Read_Local_Supported_Features, etc.).
- HCI_UP flag set.

REQ-5: hci_dev_close:
- err = hci_dev_close_sync.
- per-conn cleanup.
- per-driver hdev.close.
- HCI_UP cleared.

REQ-6: hci_send_cmd(hdev, opcode, plen, param):
- skb = bt_skb_alloc(plen + sizeof(hci_command_hdr), GFP_KERNEL).
- hdr.opcode = htole16(opcode).
- hdr.plen = plen.
- memcpy data.
- skb_queue_tail(&hdev.cmd_q, skb).
- queue_work(hdev.workqueue, &hdev.cmd_work).

REQ-7: hci_send_frame (per-driver send-out):
- hdev.send(hdev, skb): per-driver-bound transport.
- USB: usb_submit_urb. UART: serial-tx. virtio: virtqueue_add_outbuf.

REQ-8: hci_recv_frame(hdev, skb):
- bt_cb(skb).pkt_type from skb.data[0].
- Per-pkt-type:
  - HCI_EVENT_PKT: hci_event_packet.
  - HCI_ACLDATA_PKT: hci_acldata_packet.
  - HCI_SCODATA_PKT: hci_scodata_packet.
  - HCI_ISODATA_PKT: hci_isodata_packet.

REQ-9: hci_event_packet:
- Per-event-code (1 byte) → per-handler.
- Examples: HCI_EV_INQUIRY_COMPLETE, HCI_EV_CONN_COMPLETE, HCI_EV_DISCONN_COMPLETE, HCI_EV_AUTH_COMPLETE, HCI_EV_LE_META_EVENT (BLE).
- Per-cmd-complete / cmd-status: wake hdev.req_wait_q.

REQ-10: Per-conn types:
- ACL (data): per-Classic Bluetooth.
- SCO (audio): per-headset audio (CVSD).
- LE (LE-ACL): per-Bluetooth-LE.
- ESCO (extended SCO): per-newer headset.
- ISO (isochronous): LE Audio.

REQ-11: Per-namespace:
- Bluetooth devices not net-namespace-aware (single global namespace).

REQ-12: Per-driver type:
- struct hci_driver: { open, close, send, setup, set_diag, get_data_path_id, get_codec_config_data, ... }.
- Per-bus (USB/UART/...) supplies appropriate driver-ops.

## Acceptance Criteria

- [ ] AC-1: hci_register_dev: hdev in hci_dev_list; sysfs entry created.
- [ ] AC-2: hci_dev_open: per-init sequence runs; HCI_UP set.
- [ ] AC-3: hci_send_cmd HCI_OP_RESET: HCI_EV_CMD_COMPLETE received.
- [ ] AC-4: ACL connection inquiry → HCI_EV_INQUIRY_COMPLETE.
- [ ] AC-5: HCI_OP_LE_SET_ADV_ENABLE (BLE): ADV active.
- [ ] AC-6: hci_send_acl: data packets reach peer per-handle.
- [ ] AC-7: hci_recv_frame on event-pkt: per-event handler invoked.
- [ ] AC-8: hci_unregister_dev: hdev removed; references drained.
- [ ] AC-9: BlueZ user-tools (`hciconfig hci0 up`): operational.
- [ ] AC-10: ISO (LE Audio): data flow works on LE-ISO controller.

## Architecture

Per-hci_dev state:

```
struct HciDev {
  list: ListLink,                                 // hci_dev_list
  id: u16,
  name: String,                                   // "hciN"
  type_: u8,                                      // HCI_PRIMARY / HCI_AMP
  bus: u8,                                        // HCI_USB / HCI_UART / ...
  flags: HciDevFlags,                             // HCI_UP / HCI_INIT / HCI_AUTO_OFF / ...
  dev_addr: BdAddr,
  features: [[u8; 8]; 8],
  le_features: [u8; 8],
  cmd_q: SkbQueue,
  raw_q: SkbQueue,
  rx_q: SkbQueue,
  tx_q: SkbQueue,
  conn_hash: HciConnHash,
  workqueue: WorkqueueStruct,
  cmd_work: WorkStruct,
  rx_work: WorkStruct,
  tx_work: WorkStruct,
  power_on: WorkStruct,
  req_wait_q: WaitQueueHead,
  open: fn(hdev) -> i32,
  close: fn(hdev) -> i32,
  send: fn(hdev, skb) -> i32,
  ...
}
```

Per-conn:

```
struct HciConn {
  list: ListLink,
  type_: u8,                                      // ACL / SCO / LE / ESCO / ISO
  handle: u16,                                    // 12-bit conn handle
  state: u8,                                      // BT_OPEN / BT_BOUND / BT_CONNECT / BT_CONNECTED / ...
  dst: BdAddr,                                    // peer BD_ADDR
  hdev: *HciDev,
  ...
}
```

`Hci::register_dev(hdev) -> Result<()>`:
1. hdev.id = ida_alloc(&hci_index_ida).
2. snprintf(hdev.name, "hci%d", hdev.id).
3. err = hci_init_sysfs(hdev)?
4. list_add_rcu(&hdev.list, &hci_dev_list).
5. hci_dev_hold(hdev).
6. queue_work(hdev.workqueue, &hdev.power_on).

`Hci::unregister_dev(hdev)`:
1. list_del_rcu(&hdev.list).
2. hci_conn_hash_flush(hdev).
3. cancel_work_sync(&hdev.cmd_work).
4. hci_remove_sysfs(hdev).
5. hci_dev_put(hdev).

`Hci::dev_open(dev_idx) -> Result<()>`:
1. hdev = hci_dev_get(dev_idx).
2. err = hci_dev_open_sync(hdev).
3. hdev.flags |= HCI_UP.

`Hci::dev_close(dev_idx) -> Result<()>`:
1. hdev = hci_dev_get(dev_idx).
2. err = hci_dev_close_sync(hdev).
3. hdev.flags &= !HCI_UP.

`Hci::send_cmd(hdev, opcode, plen, param) -> Result<()>`:
1. skb = bt_skb_alloc(sizeof(HciCommandHdr) + plen, GFP_KERNEL).
2. hdr = skb_put(skb, sizeof(HciCommandHdr)).
3. hdr.opcode = htole16(opcode).
4. hdr.plen = plen.
5. memcpy(skb_put(skb, plen), param, plen).
6. bt_cb(skb).pkt_type = HCI_COMMAND_PKT.
7. skb_queue_tail(&hdev.cmd_q, skb).
8. queue_work(hdev.workqueue, &hdev.cmd_work).

`Hci::recv_frame(hdev, skb) -> Result<()>`:
1. pkt_type = bt_cb(skb).pkt_type.
2. switch pkt_type:
   - HCI_EVENT_PKT: skb_queue_tail(&hdev.rx_q, skb); queue_work(rx_work).
   - HCI_ACLDATA_PKT: hci_acldata_packet(hdev, skb).
   - HCI_SCODATA_PKT: hci_scodata_packet(hdev, skb).
   - HCI_ISODATA_PKT: hci_isodata_packet(hdev, skb).

`Hci::event_packet(hdev, skb)` (rx_work):
1. evt = skb.data[0] (event-code).
2. switch evt:
   - HCI_EV_CMD_COMPLETE: extract opcode + status; wake req_wait_q.
   - HCI_EV_CMD_STATUS: similar.
   - HCI_EV_CONN_COMPLETE: hci_conn_create.
   - HCI_EV_DISCONN_COMPLETE: hci_conn_del.
   - HCI_EV_LE_META_EVENT: hci_le_meta_packet (sub-event).
3. /* Per-event-callback list */
4. list_for_each_entry(cb, &hci_cb_list): cb.event_packet(hdev, skb).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pkt_type_in_set` | INVARIANT | bt_cb(skb).pkt_type ∈ {EVENT, ACL, SCO, ISO, COMMAND, VENDOR}. |
| `hdev_id_unique` | INVARIANT | per-hdev.id allocated by ida (unique system-wide). |
| `hci_up_iff_open` | INVARIANT | HCI_UP set ⟺ hci_dev_open succeeded. |
| `cmd_q_in_order` | INVARIANT | cmd-skb dequeued in enqueue order. |
| `event_handler_dispatch_per_code` | INVARIANT | per-event-code: at most one primary handler. |

### Layer 2: TLA+

`net/bluetooth/hci.tla`:
- Per-hdev lifecycle + per-cmd req-wait + per-event dispatch.
- Properties:
  - `safety_no_send_when_down` — !HCI_UP ⟹ no hci_send_cmd successful.
  - `safety_cmd_complete_match_opcode` — per-CMD_COMPLETE event matches sent opcode.
  - `liveness_cmd_eventually_completed` — per-cmd sent ⟹ eventually CMD_COMPLETE ∨ timeout.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Hci::register_dev` post: hdev in list; sysfs entry; ref held | `Hci::register_dev` |
| `Hci::send_cmd` post: skb on cmd_q; cmd_work queued | `Hci::send_cmd` |
| `Hci::recv_frame` post: per-pkt-type handler invoked or queued | `Hci::recv_frame` |
| `Hci::event_packet` post: per-event-code handler invoked; cb_list walked | `Hci::event_packet` |

### Layer 4: Verus/Creusot functional

`Per-controller HCI cmd/event flow + per-conn ACL/SCO/LE/ISO data` semantic equivalence: per-Bluetooth Core Spec v5.x HCI layer.

## Hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

HCI-specific reinforcement:

- **Per-hdev refcount via hci_dev_hold/put** — defense against UAF on unregister.
- **Per-cmd_q workqueue serialized** — defense against per-cmd race.
- **Per-pkt-type dispatched validated** — defense against per-malformed-pkt crash.
- **Per-event-handler bounded** — defense against per-handler-loop DoS.
- **Per-driver send fn-ptr null-checked** — defense against per-driver-detach send.
- **Per-hci_dev_open init-seq fail backoff** — defense against per-driver init-loop livelock.
- **Per-conn_hash flush on unregister** — defense against per-conn lingering.
- **Per-le_features per-LE only** — defense against per-Classic-only driver claiming LE.
- **Per-cmd opcode validated against supported_commands bitmap** — defense against per-unsupported cmd.
- **Per-cmd_work cancel_work_sync on unregister** — defense against in-flight cmd UAF.
- **Per-bt_cb(skb) layout** — defense against per-skb-cb overflow.

## Grsecurity/PaX-style Reinforcement

Baseline grsec/PaX posture inherited workspace-wide:

- **PAX_USERCOPY** — strict bounds on every `copy_from_sockptr`/`put_user` against the HCI control-channel and HCI raw-channel paths.
- **PAX_KERNEXEC** — `.rodata` `hci_proto_ops` / `hci_dev_ops`; W^X for the HCI event/cmd dispatch tables.
- **PAX_RANDKSTACK** — per-syscall stack randomisation across `hci_sock_*` syscall entries.
- **PAX_REFCOUNT** — saturating `refcount_t` on `struct hci_dev`, `hci_conn`, and `hci_cmd_sync` entries.
- **PAX_MEMORY_SANITIZE** — zero-on-free for HCI skb buffers carrying ACL/SCO payloads and link-key material.
- **PAX_UDEREF** — enforced isolation between user mgmt-cmd buffers and the in-kernel `hci_request` queue.
- **PAX_RAP / kCFI** — forward-edge CFI on `hdev->open/close/send/setup/shutdown` and per-event handler dispatch.
- **GRKERNSEC_HIDESYM** — HCI symbols withheld from non-CAP_SYSLOG kallsym readers.
- **GRKERNSEC_DMESG** — HCI controller log/warn messages gated by CAP_SYSLOG.

HCI-core-specific reinforcement:

- **`struct hci_dev` PAX_REFCOUNT saturation** — defense against per-controller pin-leak from misuse of `hci_dev_hold`/`hci_dev_put`.
- **BlueZ HCI USB-descriptor strict validation** — defense against malicious USB-bluetooth controllers (BadBluetooth) presenting crafted endpoint descriptors.
- **CAP_NET_ADMIN gate on raw HCI sockets and `HCI_USER_CHANNEL`** — defense against unprivileged direct-controller access.
- **`supported_commands` bitmap enforced at opcode dispatch** — defense against forged HCI events triggering unimplemented handler paths.
- **GRKERNSEC_PROC_USER on `/sys/kernel/debug/bluetooth/*`** — controller-internal state restricted from unprivileged enumeration.

Rationale: hci_core sits between an untrusted radio transport and the BlueZ-stack subsystems above. The grsec layer ensures that neither a hostile controller nor an unprivileged AF_BLUETOOTH client can corrupt the request state-machine, exfiltrate link-key material via freed slabs, or hijack `hdev->send` indirect calls.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- net/bluetooth/{l2cap, rfcomm, hidp, smp}.c (covered separately)
- BlueZ userspace (out-of-tree)
- Per-driver (USB, UART, virtio-bt; covered separately)
- Bluetooth core spec (out-of-tree)
- Implementation code
