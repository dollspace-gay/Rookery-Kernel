# Tier-3: net/bluetooth/hci_core.c — Bluetooth HCI core controller lifecycle, cmd-event flow, tx/rx workqueues

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/bluetooth/00-overview.md
upstream-paths:
  - net/bluetooth/hci_core.c (~4166 lines)
  - net/bluetooth/hci_sync.c (hci_dev_open_sync, hci_init{1..4}_sync)
  - net/bluetooth/hci_sysfs.c (hci_init_sysfs, /sys/class/bluetooth)
  - net/bluetooth/hci_event.c (hci_event_packet)
  - net/bluetooth/hci_conn.c (hci_conn_hash_flush)
  - include/net/bluetooth/hci.h
  - include/net/bluetooth/hci_core.h
  - include/net/bluetooth/hci_sync.h
-->

## Summary

The **HCI core** owns the `struct hci_dev` per-controller state and arbitrates every byte that enters or leaves a Bluetooth radio. A transport driver (btusb, hci_uart serdev, hci_vhci virtual, btsdio, virtio-bt, ...) allocates an `hci_dev` via `hci_alloc_dev_priv`, wires four function pointers (`hdev->open`, `hdev->close`, `hdev->send`, optional `hdev->setup`/`hdev->shutdown`/`hdev->flush`/`hdev->post_init`), then calls `hci_register_dev(hdev)`. Core picks an ID from `hci_index_ida`, names the controller `hci<N>`, creates the sysfs class device under `/sys/class/bluetooth/hci<N>/`, allocates two ordered workqueues (`hdev->workqueue` for cmd/rx/tx, `hdev->req_workqueue` for sync requests), creates the debugfs directory `/sys/kernel/debug/bluetooth/hci<N>/`, registers an rfkill node, links the device into `hci_dev_list`, emits the `HCI_DEV_REG` mgmt event, and schedules the `power_on` work. Per-frame TX: callers push skbs onto `hdev->cmd_q` (commands), per-connection ACL/SCO/ISO queues (data); the scheduler workers (`hci_cmd_work`, `hci_tx_work`) dequeue and call `hci_send_frame()` → `hdev->send(hdev, skb)`. Per-frame RX: the driver calls `hci_recv_frame(hdev, skb)`; core stamps the packet, queues onto `hdev->rx_q`, and schedules `hci_rx_work` which dispatches by `hci_skb_pkt_type(skb)` to `hci_event_packet` / `hci_acldata_packet` / `hci_scodata_packet` / `hci_isodata_packet`. Per-power-on: `hci_dev_open` → `hci_dev_do_open` → `hci_req_sync_lock(hdev)` → `hci_dev_open_sync(hdev)` which calls the driver `hdev->open`, then runs a four-stage init sequence `hci_init1_sync` (HCI_Reset, Read_Local_Version) → `hci_init2_sync` (Read_Local_Supported_Commands, Read_Local_Supported_Features, Read_BD_ADDR) → `hci_init3_sync` (LE/feature-specific commands) → `hci_init4_sync` (final config, write event mask) and finally sets `HCI_UP`. Per-cmd-complete: `hci_event_packet` dispatches `HCI_EV_CMD_COMPLETE`/`HCI_EV_CMD_STATUS` to `hci_req_cmd_complete` which matches against `hdev->sent_cmd` / `hdev->req_skb` and wakes `hdev->req_wait_q` for sync waiters; the cmd timer `hci_cmd_timeout` rearms or fires `hci_cmd_sync_cancel_sync` on stall. Per-scan/advertise/connect: requests come in via the BlueZ Management API socket (`mgmt.c`), translate to LE or BR/EDR controller commands, and dispatch via `hci_cmd_sync_queue` to per-controller serialized request workers. Critical for: every Bluetooth driver and every BlueZ userspace consumer (bluetoothd, hciconfig, hcitool, btmgmt, btmon, obexpushd, pulseaudio-bluetooth, pipewire-bluetooth, gnome-bluetooth).

This Tier-3 covers `hci_core.c` (~4166 lines) and the directly load-bearing helpers it dispatches into (`hci_sync.c`, `hci_sysfs.c`, `hci_event.c` for the event dispatch contract).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct hci_dev` | per-controller state (~250 fields) | `HciDev` |
| `struct hci_conn` | per-connection (ACL/SCO/LE/ESCO/ISO) | `HciConn` |
| `struct hci_cb` | per-callback (upper-protocol observer) | `HciCb` |
| `struct hci_driver` (`hdev->open/close/send/setup/...`) | per-transport vtable | `HciDriverOps` |
| `hci_alloc_dev_priv()` | per-driver alloc hdev + sizeof_priv | `Hci::alloc_dev_priv` |
| `hci_free_dev()` | per-driver free hdev (pre-register failure path) | `Hci::free_dev` |
| `hci_register_dev()` | per-driver register controller | `Hci::register_dev` |
| `hci_unregister_dev()` | per-driver unregister controller | `Hci::unregister_dev` |
| `hci_release_dev()` | per-final cleanup (refcount→0) | `Hci::release_dev` |
| `hci_dev_get(index)` | per-lookup by index + refcount | `Hci::dev_get` |
| `hci_dev_hold` / `hci_dev_put` | per-refcount | `Hci::dev_hold` / `dev_put` |
| `hci_dev_lock` / `hci_dev_unlock` | per-mutex (mgmt + state mutation) | `HciDev::lock` / `unlock` |
| `hci_dev_open(__u16 dev)` | per-controller power-on (ioctl path) | `Hci::dev_open` |
| `hci_dev_close(__u16 dev)` | per-controller power-off (ioctl path) | `Hci::dev_close` |
| `hci_dev_do_open()` | per-controller per-`req_sync_lock` open | `Hci::dev_do_open` |
| `hci_dev_do_close()` | per-controller per-`req_sync_lock` close | `Hci::dev_do_close` |
| `hci_dev_do_reset()` | per-controller reset (purge queues, call `hdev->flush`) | `Hci::dev_do_reset` |
| `hci_dev_open_sync()` | per-controller open + init1..init4 (hci_sync.c) | `HciSync::dev_open` |
| `hci_dev_close_sync()` | per-controller orderly close (hci_sync.c) | `HciSync::dev_close` |
| `hci_init_sync()` / `hci_init1_sync` ... `hci_init4_sync` | per-init steps INIT0..INIT4 | `HciSync::init_sync` |
| `hci_send_cmd(hdev, opcode, plen, param)` | per-cmd enqueue | `Hci::send_cmd` |
| `__hci_cmd_send()` | per-VS-cmd (vendor-specific, no event expected) | `Hci::cmd_send_vs` |
| `hci_send_frame()` | per-tx to driver (`hdev->send`) | `Hci::send_frame` |
| `hci_send_acl()` | per-ACL data send (`hci_chan` queueing) | `Hci::send_acl` |
| `hci_send_sco()` | per-SCO audio send | `Hci::send_sco` |
| `hci_send_iso()` | per-ISO (LE Audio) send | `Hci::send_iso` |
| `hci_recv_frame()` | per-pkt RX from driver | `Hci::recv_frame` |
| `hci_recv_diag()` | per-diag pkt RX | `Hci::recv_diag` |
| `hci_cmd_work()` / `hci_cmd_timeout()` / `hci_ncmd_timeout()` | per-cmd workqueue + timeouts | `Hci::cmd_work` |
| `hci_rx_work()` | per-rx-dispatcher | `Hci::rx_work` |
| `hci_tx_work()` / `hci_sched_{acl,sco,le,iso}` | per-tx-scheduler | `Hci::tx_work` |
| `hci_event_packet()` | per-event-code dispatch (hci_event.c) | `HciEvent::event_packet` |
| `hci_acldata_packet()` | per-ACL data RX dispatch | `Hci::acldata_packet` |
| `hci_scodata_packet()` | per-SCO data RX dispatch | `Hci::scodata_packet` |
| `hci_isodata_packet()` | per-ISO data RX dispatch | `Hci::isodata_packet` |
| `hci_req_cmd_complete()` | per-CMD_COMPLETE/CMD_STATUS match + wake | `Hci::req_cmd_complete` |
| `hci_register_cb()` / `hci_unregister_cb()` | per-upper-protocol observer | `Hci::register_cb` / `unregister_cb` |
| `hci_init_sysfs()` | per-/sys/class/bluetooth setup (hci_sysfs.c) | `HciSysfs::init` |
| `hci_register_suspend_notifier()` | per-PM notifier (S3/S4) | `Hci::register_suspend_notifier` |
| `hci_suspend_dev()` / `hci_resume_dev()` | per-PM suspend / resume | `Hci::suspend_dev` / `resume_dev` |
| `hci_reset_dev()` | per-hard-reset injected by driver | `Hci::reset_dev` |
| `hci_sent_cmd_data()` | per-last-sent-cmd payload accessor | `Hci::sent_cmd_data` |
| `hci_recv_event_data()` | per-last-received-event payload accessor | `Hci::recv_event_data` |
| `hci_set_hw_info()` / `hci_set_fw_info()` | per-vendor identification | `Hci::set_hw_info` / `set_fw_info` |
| `hci_power_on()` / `hci_power_off()` | per-power workqueue tasks | `Hci::power_on_work` / `power_off_work` |
| `hci_error_reset()` | per-error reset workqueue task | `Hci::error_reset_work` |
| `hci_rfkill_set_block()` | per-rfkill block callback | `Hci::rfkill_set_block` |
| `hci_inquiry()` | per-BR/EDR inquiry ioctl (`HCIINQUIRY`) | `Hci::inquiry` |
| `hci_get_dev_list()` / `hci_get_dev_info()` | per-`HCIGETDEVLIST` / `HCIGETDEVINFO` | `Hci::get_dev_list` / `get_dev_info` |
| `hci_dev_cmd()` | per-`HCIDEVUP/DOWN/RESET/RESTAT/SETSCAN/...` | `Hci::dev_cmd` |
| `HCI_ACLDATA_PKT` / `_SCODATA_PKT` / `_EVENT_PKT` / `_COMMAND_PKT` / `_ISODATA_PKT` / `_DIAG_PKT` / `_DRV_PKT` | wire pkt types | UAPI |

## Compatibility contract

REQ-1: `struct hci_dev` per-controller state (subset, ABI-stable for in-tree drivers + BlueZ Management API):
- name: char[8] ("hciN").
- id: u16 (allocated from `hci_index_ida`, 0..HCI_MAX_ID-1).
- bus: u8 (HCI_VIRTUAL=0, HCI_USB=1, HCI_PCCARD=2, HCI_UART=3, HCI_RS232=4, HCI_PCI=5, HCI_SDIO=6, HCI_SPI=7, HCI_I2C=8, HCI_SMD=9, HCI_VIRTIO=10).
- dev_type: u8 (HCI_PRIMARY=0 dual-mode classic+LE / HCI_AMP=1 alternate MAC/PHY).
- bdaddr: bdaddr_t (6 bytes, little-endian on wire).
- flags: unsigned long bitmask — HCI_UP, HCI_INIT, HCI_RUNNING, HCI_RESET, HCI_AUTO_OFF, HCI_CMD_PENDING, HCI_CMD_DRAIN_WORKQUEUE.
- features[8][8]: per-page extended classic feature bytes.
- le_features[8]: per-LE feature bytes (from HCI_OP_LE_READ_LOCAL_FEATURES).
- commands[64]: per-`Read_Local_Supported_Commands` bitmap (512 bits).
- cmd_q: `struct sk_buff_head` — pending HCI commands FIFO.
- rx_q: pending RX skbs.
- raw_q: raw HCI socket RX.
- workqueue, req_workqueue: two ordered workqueues with `WQ_HIGHPRI`.
- power_on, power_off (delayed), error_reset, cmd_work, rx_work, tx_work, cmd_timer (delayed), ncmd_timer (delayed): work items.
- req_wait_q: wait_queue_head_t for `hci_req_sync` waiters.
- req_skb: per-currently-pending-request skb (cloned from `cmd_q` head on send).
- sent_cmd: per-most-recent skb passed to `hci_send_frame` (cloned).
- recv_event: per-most-recent event skb (for `hci_recv_event_data` accessor).
- conn_hash: `struct hci_conn_hash` — per-handle hash bucket of `hci_conn`.
- adv_instances_lock, adv_instances: per-LE-advertising-instance list.
- adv_monitors_idr: per-MSFT/AOSP advertising-monitor identifiers.
- mgmt_pending: per-pending mgmt-cmd list.
- unregister_lock: mutex serialising unregister.
- srcu: SRCU instance for safe `hci_dev_get` under unregister.
- open, close, flush, setup, shutdown, post_init, prevent_wake, send, set_bdaddr, set_diag, get_data_path_id, get_codec_config_data, get_codec, wakeup, hw_info, fw_info: function pointers (`hci_driver`-like).
- debugfs: per-controller debugfs dir `/sys/kernel/debug/bluetooth/hciN/`.
- rfkill: per-rfkill instance.

REQ-2: `hci_register_dev(hdev) -> int`:
- Pre-cond: `hdev->open`, `hdev->close`, `hdev->send` non-null else `-EINVAL`.
- `id = ida_alloc_max(&hci_index_ida, HCI_MAX_ID - 1, GFP_KERNEL)`; return `id` on failure.
- `dev_set_name(&hdev->dev, "hci%u", id)`; on failure return error.
- `hdev->name = dev_name(&hdev->dev); hdev->id = id`.
- `hdev->workqueue = alloc_ordered_workqueue("%s", WQ_HIGHPRI, hdev->name)`; `-ENOMEM` on failure.
- `hdev->req_workqueue = alloc_ordered_workqueue("%s", WQ_HIGHPRI, hdev->name)` (separate ordered wq so a long-running sync request cannot block command/event processing).
- If `bt_debugfs` non-null: `hdev->debugfs = debugfs_create_dir(hdev->name, bt_debugfs)`.
- `device_add(&hdev->dev)` — wires the kobject under `/sys/class/bluetooth/`; on failure goto err_wqueue.
- `hci_leds_init(hdev)`.
- `hdev->rfkill = rfkill_alloc(hdev->name, &hdev->dev, RFKILL_TYPE_BLUETOOTH, &hci_rfkill_ops, hdev)`; if alloc succeeds attempt `rfkill_register`; on rfkill-register failure `rfkill_destroy` and zero the pointer (controller still functional, just no rfkill node).
- If rfkill blocked at register-time: set `HCI_RFKILLED`.
- Set `HCI_SETUP`, `HCI_AUTO_OFF`, `HCI_BREDR_ENABLED` (BR/EDR assumed until init clears it).
- `write_lock(&hci_dev_list_lock); list_add(&hdev->list, &hci_dev_list); write_unlock(...)`.
- If `HCI_QUIRK_RAW_DEVICE`: set `HCI_UNCONFIGURED`.
- If `hdev->wakeup` callback present: `hdev->conn_flags |= HCI_CONN_FLAG_REMOTE_WAKEUP`.
- `hci_sock_dev_event(hdev, HCI_DEV_REG)` — broadcast to raw HCI sockets + Management API listeners.
- `hci_dev_hold(hdev)` — pin reference for power_on work.
- `hci_register_suspend_notifier(hdev)`.
- `queue_work(hdev->req_workqueue, &hdev->power_on)` — schedule auto-power-on.
- `idr_init(&hdev->adv_monitors_idr); msft_register(hdev)`.
- Return `id` on success; on failure unwind workqueues + ida.

REQ-3: `hci_unregister_dev(hdev)`:
- `mutex_lock(&hdev->unregister_lock); hci_dev_set_flag(hdev, HCI_UNREGISTER); mutex_unlock(...)` — prevents new `hci_dev_get` consumers.
- `write_lock(&hci_dev_list_lock); list_del(&hdev->list); write_unlock(...)`.
- `synchronize_srcu(&hdev->srcu); cleanup_srcu_struct(&hdev->srcu)` — drain in-flight readers acquired via `hci_dev_get_srcu`.
- `disable_work_sync` on rx_work, cmd_work, tx_work, power_on, error_reset (per-Linux 7.x `disable_work_sync` rather than `cancel_work_sync` to prevent re-arming).
- `hci_cmd_sync_clear(hdev)` — flush sync request queue.
- `hci_unregister_suspend_notifier(hdev)`.
- `hci_dev_do_close(hdev)` — if up, take it down.
- If not in HCI_INIT/HCI_SETUP/HCI_CONFIG: `mgmt_index_removed(hdev)` to notify userspace BlueZ.
- `BUG_ON(!list_empty(&hdev->mgmt_pending))` — pending mgmt commands MUST have been dispatched / dropped.
- `hci_sock_dev_event(hdev, HCI_DEV_UNREG)`.
- `rfkill_unregister + rfkill_destroy` if present.
- `device_del(&hdev->dev)` — removes sysfs entry.
- `hci_dev_put(hdev)` — drops register-side ref; actual free runs in `hci_release_dev` when refcount hits zero.

REQ-4: `hci_release_dev(hdev)` (kobj release path):
- `debugfs_remove_recursive(hdev->debugfs)`.
- `kfree_const(hw_info); kfree_const(fw_info)`.
- `destroy_workqueue(workqueue); destroy_workqueue(req_workqueue)`.
- Under `hci_dev_lock`: clear bdaddr lists (reject/accept/LE accept/LE resolv), uuids, link keys, SMP LTKs, SMP IRKs, remote OOB data, adv instances, adv monitors, conn params, discovery filter, blocked keys, local codecs; `msft_release`.
- `ida_destroy(&hdev->unset_handle_ida); ida_free(&hci_index_ida, hdev->id)`.
- `kfree_skb(sent_cmd); kfree_skb(req_skb); kfree_skb(recv_event)`.
- `kfree(hdev)`.

REQ-5: `hci_dev_open(__u16 dev) -> int` (ioctl `HCIDEVUP` + autopower paths):
- `hdev = hci_dev_get(dev)`; `-ENODEV` if missing.
- If `HCI_UNCONFIGURED` and not `HCI_USER_CHANNEL`: `-EOPNOTSUPP` (unconfigured controllers usable only via user-channel HCI socket).
- If `HCI_AUTO_OFF` set + cleared atomically: `cancel_delayed_work(&hdev->power_off)`.
- `flush_workqueue(hdev->req_workqueue)` — ensure pending `power_on`/`error_reset` work has settled.
- If not user-channel and not mgmt-controlled: set `HCI_BONDABLE` (legacy tools rely on bondable being on).
- `err = hci_dev_do_open(hdev)`.
- `hci_dev_put(hdev); return err`.

REQ-6: `hci_dev_do_open(hdev) -> int`:
- `hci_req_sync_lock(hdev)` — serialises against `hci_dev_close`, `hci_dev_reset`, request workers.
- `ret = hci_dev_open_sync(hdev)` (hci_sync.c).
- `hci_req_sync_unlock(hdev); return ret`.

REQ-7: `hci_dev_open_sync(hdev)` (hci_sync.c, but called from hci_core.c) — sequence:
- `hci_sock_dev_event(hdev, HCI_DEV_OPEN)`.
- Driver callback `hdev->open(hdev)` — typically opens USB endpoint / opens UART / starts virtio queues.
- Set `HCI_RUNNING` (drivers may now submit on `hdev->send`).
- `atomic_set(&hdev->cmd_cnt, 1); set_bit(HCI_INIT, &hdev->flags)`.
- `hci_dev_setup_sync(hdev)`: if `hdev->setup` set, call it (e.g., btusb firmware-download + chip-init); apply `HCI_QUIRK_BROKEN_*` warnings; handle `HCI_QUIRK_USE_BDADDR_PROPERTY` / `HCI_QUIRK_INVALID_BDADDR` / `HCI_QUIRK_EXTERNAL_CONFIG` (may set `HCI_UNCONFIGURED`).
- If configured + not user-channel: `hci_init_sync(hdev)`:
  - `hci_init1_sync`: HCI_Reset (`HCI_OP_RESET`), Read_Local_Version_Information.
  - `hci_init2_sync`: Read_Local_Supported_Commands, Read_Local_Supported_Features, Read_BD_ADDR, Read_Buffer_Size (ACL/SCO pool, classic-only).
  - `hci_init3_sync`: LE-specific (Read_LE_Local_Supported_Features, Read_LE_Buffer_Size, LE_Read_Maximum_Data_Length, etc.) + extended classic commands per features bitmap.
  - `hci_init4_sync`: Write_LE_Host_Supported (if dual-mode), Read_Local_Codecs (BR/EDR), Read_Local_Codec_Capabilities, Write_Default_Erroneous_Data_Reporting, Write_Default_Link_Policy_Settings, Write_Page_Scan_Activity, write event mask, etc.
- Optional `hdev->post_init(hdev)`.
- Clear `HCI_INIT`. Set `HCI_UP`.
- If `HCI_SETUP` was set: `hci_setup_sync_complete(hdev)` — clears `HCI_SETUP` + emits `HCI_DEV_UP` via mgmt.

REQ-8: `hci_dev_close(__u16 dev) -> int`:
- `hdev = hci_dev_get(dev); -ENODEV` if missing.
- If `HCI_USER_CHANNEL`: `-EBUSY` (user channel must release via socket-close).
- `cancel_work_sync(&hdev->power_on)`; if `HCI_AUTO_OFF` clear: `cancel_delayed_work(&hdev->power_off)`.
- `err = hci_dev_do_close(hdev) = hci_req_sync_lock + hci_dev_close_sync + unlock`.
- `hci_dev_put; return err`.

REQ-9: `hci_dev_close_sync(hdev)` (hci_sync.c):
- Emit `HCI_DEV_CLOSE` via mgmt.
- `hci_clear_async_filter`, cancel discovery/inquiry/advertising.
- `hci_conn_hash_flush(hdev)` — disconnect & free every `hci_conn`.
- `hdev->shutdown(hdev)` if provided.
- Driver callback `hdev->flush(hdev)` if provided.
- `hdev->close(hdev)`.
- Purge `cmd_q`, `rx_q`, `raw_q`. `kfree_skb(sent_cmd)=NULL; kfree_skb(req_skb)=NULL; kfree_skb(recv_event)=NULL`.
- Clear `HCI_RUNNING`, `HCI_UP`, `HCI_INIT`.

REQ-10: `hci_send_cmd(hdev, opcode, plen, param) -> int`:
- `skb = hci_cmd_sync_alloc(hdev, opcode, plen, param, NULL)` — builds skb with `hci_command_hdr { opcode_le16, plen_u8 }` + payload; sets `hci_skb_pkt_type(skb)=HCI_COMMAND_PKT`, `hci_skb_opcode(skb)=opcode`.
- `bt_cb(skb)->hci.req_flags |= HCI_REQ_START` (single-command request marker).
- `skb_queue_tail(&hdev->cmd_q, skb)`.
- `queue_work(hdev->workqueue, &hdev->cmd_work)` — wake cmd_work.
- Return 0.

REQ-11: `__hci_cmd_send(hdev, opcode, plen, param) -> int` (vendor-specific OGF==0x3f only, no event expected):
- Validate `hci_opcode_ogf(opcode) == 0x3f` else `-EINVAL`.
- Allocate same way.
- Bypass `cmd_q`; `hci_send_frame(hdev, skb)` directly.
- Return 0.

REQ-12: `hci_send_frame(hdev, skb) -> int` (static, called from cmd_work, tx_work, conn paths):
- `__net_timestamp(skb)`.
- `hci_send_to_monitor(hdev, skb)` — copies to btmon `/dev/hci_monitor` listeners.
- If `atomic_read(&hdev->promisc) > 0`: `hci_send_to_sock(hdev, skb)` (raw HCI sockets in promisc mode).
- `skb_orphan(skb)` — drops any sk_buff destructor before handing to driver.
- If `!test_bit(HCI_RUNNING, &hdev->flags)`: `kfree_skb(skb); return -EINVAL`.
- If `hci_skb_pkt_type(skb) == HCI_DRV_PKT`: route via `hci_drv_process_cmd` (driver-management cmds; do not hit `hdev->send`).
- `err = hdev->send(hdev, skb)`.
- On error: log + `kfree_skb(skb)`.
- Return `err`.

REQ-13: `hci_cmd_work(work)` (cmd workqueue, runs serialised under ordered wq):
- If `atomic_read(&hdev->cmd_cnt) > 0`:
  - `skb = skb_dequeue(&hdev->cmd_q)`; if empty, return.
  - `err = hci_send_cmd_sync(hdev, skb)`.
  - On success: store `hdev->sent_cmd = skb_clone(skb)`; if pending sync request, also clone into `req_skb`; `atomic_dec(&hdev->cmd_cnt)`.
- Under `rcu_read_lock`: if `HCI_RESET` or `HCI_CMD_DRAIN_WORKQUEUE`: drain `cmd_q` rather than rearming.
- Else if `cmd_q` non-empty + `cmd_cnt > 0`: arm `hci_cmd_timeout` delayed work (HCI_CMD_TIMEOUT) to detect a wedged controller.
- `hci_ncmd_timeout` covers the no-command-credits stall (NCMD).

REQ-14: `hci_recv_frame(hdev, skb) -> int` (called by driver in IRQ/softirq/process context):
- Reject if `hdev` null or neither `HCI_UP` nor `HCI_INIT`: `kfree_skb; return -ENXIO`.
- `dev_pkt_type = hci_dev_classify_pkt_type(hdev, skb)`; if driver-attached `pkt_type` mismatches, override per-classifier (handles `HCI_QUIRK_*` packet-type ambiguities).
- Switch on `hci_skb_pkt_type(skb)`:
  - HCI_EVENT_PKT / HCI_SCODATA_PKT / HCI_ISODATA_PKT / HCI_DRV_PKT: accept.
  - HCI_ACLDATA_PKT: if any CIS/BIS/PA connection exists, look up the connection by ACL handle; if its type is CIS/BIS/PA, reclassify as `HCI_ISODATA_PKT` (drivers that don't distinguish ISO at the wire layer).
  - default: `kfree_skb; return -EINVAL`.
- `bt_cb(skb)->incoming = 1; __net_timestamp(skb)`.
- `skb_queue_tail(&hdev->rx_q, skb); queue_work(hdev->workqueue, &hdev->rx_work)`.

REQ-15: `hci_rx_work(work)`:
- For each skb in `rx_q`:
  - Under `kcov_remote_start_common(skb_get_kcov_handle(skb)) / kcov_remote_stop()` so fuzzers attribute coverage to the syscall thread that injected the data.
  - `hci_send_to_monitor(hdev, skb)`. If promisc>0: `hci_send_to_sock`.
  - If `HCI_USER_CHANNEL` + not `HCI_INIT`: drop (userspace owns the controller).
  - If `HCI_INIT`: drop ACL/SCO/ISO (only events processed during init).
  - Dispatch by pkt_type:
    - HCI_EVENT_PKT → `hci_event_packet(hdev, skb)`.
    - HCI_ACLDATA_PKT → `hci_acldata_packet(hdev, skb)` → conn lookup + L2CAP-recv hook.
    - HCI_SCODATA_PKT → `hci_scodata_packet(hdev, skb)` → SCO socket.
    - HCI_ISODATA_PKT → `hci_isodata_packet(hdev, skb)` → ISO socket / LE Audio.

REQ-16: `hci_req_cmd_complete(hdev, opcode, status, req_complete, req_complete_skb)`:
- If sent_cmd opcode mismatches event opcode:
  - If `HCI_INIT` + opcode == HCI_OP_RESET: `hci_resend_last(hdev)` (CSR-based controllers emit spontaneous reset-complete events).
  - Else return — leave the next CMD_COMPLETE to match.
- Else:
  - Clear `HCI_CMD_PENDING`.
  - If status==0 + request not complete (more cmds in the in-flight `hci_req`): return; subsequent cmd-complete continues the request.
  - If `req_skb` carries `HCI_REQ_SKB`: deliver `req_complete_skb` callback. Else if `HCI_REQ_COMPLETE`: deliver `req_complete`.
  - Drain trailing-cmds-belonging-to-same-request from `cmd_q` under `cmd_q.lock`.

REQ-17: `hci_send_acl(chan, skb, flags)`:
- ACL queues are per-`hci_chan` (per-L2CAP-cid logical channel), themselves attached to per-`hci_conn`.
- Sets PB/BC flags in ACL header (`hci_add_acl_hdr`).
- Per-MTU-fragmentation via `hci_queue_acl` (chains into `chan->data_q`).
- Wakes tx_work which interleaves ACL/SCO/LE/ISO per priority + buffer credits.

REQ-18: `hci_register_cb(cb)` / `hci_unregister_cb(cb)`:
- Upper protocols (L2CAP, SCO, ISO, mgmt) register `struct hci_cb { name, connect_cfm, disconn_cfm, security_cfm, key_change_cfm, role_switch_cfm, ... }`.
- Per-event callbacks invoked from `hci_event.c` handlers (e.g., `HCI_EV_CONN_COMPLETE` triggers `cb->connect_cfm`).
- `hci_cb_list` mutated under `mutex_lock(&hci_cb_list_lock)`; iterated under `rcu_read_lock` in event handlers.

REQ-19: sysfs hierarchy `/sys/class/bluetooth/`:
- `/sys/class/bluetooth/hci<N>/` per-controller (via `device_add(&hdev->dev)`):
  - `name`, `type`, `address`, `manufacturer`, `version`, `revision`, `idle_timeout`, `sniff_max_interval`, `sniff_min_interval`, `supervision_timeout`.
  - `power/`, `subsystem/`, `uevent`.
  - per-symlink to transport device (USB intf / UART tty / virtio devfs).
- `/sys/class/bluetooth/hci<N>/<conn>/` per `hci_conn`:
  - `address`, `type`, `link_mode`, `idle_timeout`, etc.
- rfkill node `/sys/class/rfkill/rfkill<K>/` linked to `hdev->dev`.

REQ-20: PM (suspend/resume):
- `hci_register_suspend_notifier`: registers `hdev->suspend_notifier` if not `HCI_QUIRK_NO_SUSPEND_NOTIFIER`.
- `hci_suspend_notifier(nb, action, data)`: on `PM_SUSPEND_PREPARE`/`PM_HIBERNATION_PREPARE`: `hci_suspend_dev(hdev)` → emit `HCI_DEV_SUSPEND` + queue suspend work that issues `Set_Event_Mask`, sets up wake-on-LE, disables scanning.
- `hci_resume_dev(hdev)`: emit `HCI_DEV_RESUME` + queue resume work + re-enable scanning.

REQ-21: rfkill integration:
- `hci_rfkill_set_block(data, blocked)`: if blocked, set `HCI_RFKILLED` + queue `power_off` work; if unblocked, clear `HCI_RFKILLED`.
- `rfkill_blocked` checked on every `hci_dev_open` indirectly via the management API.

REQ-22: monitor & raw-socket fanout:
- Every frame leaving `hci_send_frame` and entering `hci_recv_frame` is copied to:
  - `hci_send_to_monitor` (btmon `/proc/net/bluetooth/{hci_dev,...}` + `BT_HCI_CMSG_*`).
  - if `hdev->promisc > 0`: `hci_send_to_sock` (raw HCI sockets with `HCI_CHANNEL_RAW`).

REQ-23: namespace scope: Bluetooth devices live in `init_net`-only (no per-netns wiphy-equivalent). One global `hci_dev_list` protected by `hci_dev_list_lock` (rwlock).

REQ-24: ioctl surface (`hci_sock.c` calls into `hci_core.c`): `HCIDEVUP`, `HCIDEVDOWN`, `HCIDEVRESET`, `HCIDEVRESTAT`, `HCIGETDEVLIST`, `HCIGETDEVINFO`, `HCIINQUIRY`, `HCISETRAW`, `HCISETSCAN`, `HCISETAUTH`, `HCISETENCRYPT`, `HCISETPTYPE`, `HCISETLINKPOL`, `HCISETLINKMODE`, `HCISETACLMTU`, `HCISETSCOMTU`, `HCIBLOCKADDR`, `HCIUNBLOCKADDR`. Dispatch routed by `hci_dev_cmd`.

## Acceptance Criteria

- [ ] AC-1: `hci_register_dev` succeeds with valid `open/close/send`: `/sys/class/bluetooth/hciN/` exists; `/sys/kernel/debug/bluetooth/hciN/` exists; index allocated; HCI_SETUP+HCI_AUTO_OFF+HCI_BREDR_ENABLED set; HCI_DEV_REG event delivered to raw HCI socket subscribers.
- [ ] AC-2: `hci_register_dev` with `open==NULL || close==NULL || send==NULL` returns `-EINVAL`.
- [ ] AC-3: `hci_dev_open` from down: HCI_INIT then HCI_UP; INIT1..INIT4 commands observed on `hdev->send`; HCI_OP_RESET issued first.
- [ ] AC-4: `hci_send_cmd(HCI_OP_RESET, 0, NULL)`: skb appears on `cmd_q`; cmd_work runs; `hdev->send` invoked exactly once with COMMAND_PKT; HCI_EV_CMD_COMPLETE matches sent opcode; `hci_req_cmd_complete` wakes `req_wait_q`.
- [ ] AC-5: `hci_recv_frame` on `HCI_EVENT_PKT(HCI_EV_LE_META_EVENT)`: queued on rx_q; rx_work dispatches to `hci_event_packet` which routes LE-meta sub-event.
- [ ] AC-6: `hci_recv_frame` on malformed pkt_type: returns `-EINVAL`; skb freed.
- [ ] AC-7: `hci_recv_frame` during `HCI_INIT`: ACL/SCO/ISO dropped silently; EVENT still processed (init commands need cmd-complete events).
- [ ] AC-8: `hci_dev_open` with `HCI_UNCONFIGURED && !HCI_USER_CHANNEL`: returns `-EOPNOTSUPP`.
- [ ] AC-9: `hci_dev_close`: `HCI_UP` cleared; rx/cmd/tx queues purged; `hci_conn_hash_flush` invoked; driver `hdev->close` called exactly once.
- [ ] AC-10: `hci_unregister_dev`: `HCI_UNREGISTER` set under unregister_lock; `hci_dev_list` no longer contains hdev; mgmt_pending empty; HCI_DEV_UNREG broadcast; rfkill destroyed; sysfs device deleted; refcount drops to zero → `hci_release_dev` invoked.
- [ ] AC-11: `hci_dev_get(idx)` after `hci_unregister_dev` started (HCI_UNREGISTER set): returns NULL.
- [ ] AC-12: PM suspend with default notifier: HCI_DEV_SUSPEND emitted; on resume, HCI_DEV_RESUME emitted; scanning/advertising state restored.
- [ ] AC-13: rfkill-blocked controller: `hci_dev_open` succeeds only via user-channel; mgmt-bring-up returns `-ERFKILL`.
- [ ] AC-14: BlueZ `hciconfig hci0 up` (legacy ioctl `HCIDEVUP`) succeeds for HCI_PRIMARY controller post-register.
- [ ] AC-15: Raw HCI socket with `HCI_CHANNEL_MONITOR`: receives a copy of every TX + RX frame; ordering preserved.

## Architecture

`HciDev` (subset of `hci_dev`):

```
struct HciDev {
  list: ListLink,                                  // hci_dev_list (rwlock-protected)
  srcu: SrcuStruct,
  unregister_lock: Mutex,
  dev: Device,                                     // wraps kobject for sysfs
  name: [u8; 8],                                   // "hciN"
  id: u16,
  bus: u8,                                         // HCI_USB / HCI_UART / HCI_VIRTUAL / ...
  dev_type: u8,                                    // HCI_PRIMARY / HCI_AMP
  bdaddr: BdAddr,
  flags: AtomicUsize,                              // HCI_UP / HCI_INIT / HCI_RUNNING / HCI_AUTO_OFF / ...
  dev_flags: AtomicUsize,                          // HCI_SETUP / HCI_CONFIG / HCI_USER_CHANNEL / HCI_UNCONFIGURED / HCI_UNREGISTER / HCI_DEBUGFS_CREATED / ...
  features: [[u8; 8]; 8],
  le_features: [u8; 8],
  commands: [u8; 64],
  cmd_q: SkbQueue,
  rx_q: SkbQueue,
  raw_q: SkbQueue,
  cmd_cnt: AtomicI32,                              // command credits
  sent_cmd: Option<SkbBox>,                        // last skb passed to send
  req_skb:  Option<SkbBox>,                        // current sync-request anchor
  recv_event: Option<SkbBox>,                      // last RX event
  workqueue: WorkqueueOrdered,
  req_workqueue: WorkqueueOrdered,
  cmd_work: WorkStruct,
  rx_work:  WorkStruct,
  tx_work:  WorkStruct,
  power_on: WorkStruct,
  power_off: DelayedWork,
  error_reset: WorkStruct,
  cmd_timer:  DelayedWork,
  ncmd_timer: DelayedWork,
  req_wait_q: WaitQueueHead,
  conn_hash: HciConnHash,
  mgmt_pending: ListHead,
  adv_instances: ListHead,
  adv_instances_lock: Mutex,
  adv_monitors_idr: Idr,
  unset_handle_ida: Ida,
  rfkill: Option<RfkillBox>,
  rfkill_ops: RfkillOps,
  debugfs: Option<DentryBox>,
  suspend_notifier: NotifierBlock,
  open:       fn(&mut HciDev) -> i32,
  close:      fn(&mut HciDev) -> i32,
  flush:      Option<fn(&mut HciDev) -> i32>,
  setup:      Option<fn(&mut HciDev) -> i32>,
  shutdown:   Option<fn(&mut HciDev) -> i32>,
  post_init:  Option<fn(&mut HciDev) -> i32>,
  send:       fn(&mut HciDev, SkbBox) -> i32,
  set_bdaddr: Option<fn(&mut HciDev, &BdAddr) -> i32>,
  set_diag:   Option<fn(&mut HciDev, bool) -> i32>,
  wakeup:     Option<fn(&HciDev) -> bool>,
  hw_info: Option<Box<str>>,
  fw_info: Option<Box<str>>,
  ...
}
```

`Hci::register_dev(hdev) -> Result<u16, Errno>`:
1. If `hdev.open == 0 || hdev.close == 0 || hdev.send == 0` ⟹ `Err(EINVAL)`.
2. `id = ida_alloc_max(&HCI_INDEX_IDA, HCI_MAX_ID - 1, GFP_KERNEL)?`.
3. `dev_set_name(&hdev.dev, "hci{id}")?`.
4. `hdev.name = dev_name(&hdev.dev); hdev.id = id`.
5. `hdev.workqueue = alloc_ordered_workqueue("{name}", WQ_HIGHPRI)?`.
6. `hdev.req_workqueue = alloc_ordered_workqueue("{name}", WQ_HIGHPRI)?`.
7. If `bt_debugfs.is_some()`: `hdev.debugfs = debugfs_create_dir(name, bt_debugfs)`.
8. `device_add(&hdev.dev)?` on failure: tear down workqueues + ida.
9. `hci_leds_init(hdev)`.
10. `hdev.rfkill = rfkill_alloc(name, &hdev.dev, RFKILL_TYPE_BLUETOOTH, &HCI_RFKILL_OPS, hdev)`; if alloc-ok: try `rfkill_register`; on failure destroy + clear.
11. If rfkill & `rfkill_blocked`: set `HCI_RFKILLED`.
12. Set `HCI_SETUP | HCI_AUTO_OFF | HCI_BREDR_ENABLED`.
13. Under `HCI_DEV_LIST_LOCK.write()`: `list_add(&hdev.list, &HCI_DEV_LIST)`.
14. If `HCI_QUIRK_RAW_DEVICE`: set `HCI_UNCONFIGURED`.
15. If `hdev.wakeup.is_some()`: `hdev.conn_flags |= HCI_CONN_FLAG_REMOTE_WAKEUP`.
16. `hci_sock_dev_event(hdev, HCI_DEV_REG)`.
17. `hci_dev_hold(hdev)`.
18. `hci_register_suspend_notifier(hdev)` — warn-but-don't-fail.
19. `queue_work(hdev.req_workqueue, &hdev.power_on)`.
20. `idr_init(&hdev.adv_monitors_idr); msft_register(hdev)`.
21. `Ok(id)`.

`Hci::unregister_dev(hdev)`:
1. `hdev.unregister_lock.lock(); hdev.set_flag(HCI_UNREGISTER); .unlock()`.
2. `HCI_DEV_LIST_LOCK.write(); list_del(&hdev.list); .unlock()`.
3. `synchronize_srcu(&hdev.srcu); cleanup_srcu_struct(&hdev.srcu)`.
4. `disable_work_sync(&hdev.rx_work)` + cmd_work + tx_work + power_on + error_reset.
5. `hci_cmd_sync_clear(hdev)`.
6. `hci_unregister_suspend_notifier(hdev)`.
7. `hci_dev_do_close(hdev)`.
8. If not (`HCI_INIT | HCI_SETUP | HCI_CONFIG`): under `hci_dev_lock(hdev)` call `mgmt_index_removed(hdev)`.
9. Debug-assert `hdev.mgmt_pending` empty.
10. `hci_sock_dev_event(hdev, HCI_DEV_UNREG)`.
11. If rfkill set: `rfkill_unregister + rfkill_destroy`.
12. `device_del(&hdev.dev)`.
13. `hci_dev_put(hdev)`.

`Hci::dev_open(idx) -> Result<(), Errno>`:
1. `hdev = hci_dev_get(idx).ok_or(ENODEV)?`.
2. Defer-drop `hdev` ref via RAII guard.
3. If `hdev.test_flag(HCI_UNCONFIGURED) && !hdev.test_flag(HCI_USER_CHANNEL)` ⟹ `EOPNOTSUPP`.
4. If `test_and_clear(HCI_AUTO_OFF)`: `cancel_delayed_work(&hdev.power_off)`.
5. `flush_workqueue(hdev.req_workqueue)`.
6. If `!HCI_USER_CHANNEL && !HCI_MGMT`: set `HCI_BONDABLE`.
7. `Hci::dev_do_open(hdev)`.

`Hci::dev_do_open(hdev) -> Result<(), Errno>`:
1. `hci_req_sync_lock(hdev)`.
2. `ret = HciSync::dev_open(hdev)`.
3. `hci_req_sync_unlock(hdev)`.
4. `ret`.

`HciSync::dev_open(hdev) -> Result<(), Errno>`:
1. `hci_sock_dev_event(hdev, HCI_DEV_OPEN)`.
2. `hdev.open(hdev)?`.
3. `hdev.set_flag(HCI_RUNNING)`.
4. `atomic_set(&hdev.cmd_cnt, 1); hdev.set_flag(HCI_INIT)`.
5. `HciSync::dev_setup(hdev)?` (driver `setup` + quirk handling).
6. If `!hdev.test_flag(HCI_UNCONFIGURED) && !hdev.test_flag(HCI_USER_CHANNEL)`:
   - `HciSync::init1(hdev)?` /* HCI_Reset, Read_Local_Version_Information */
   - if `HCI_SETUP`: `hci_debugfs_create_basic(hdev)`.
   - `HciSync::init2(hdev)?` /* Read_Local_Supported_Commands, Read_Local_Supported_Features, Read_BD_ADDR, Read_Buffer_Size */
   - `HciSync::init3(hdev)?` /* LE_Read_Local_Features, LE_Read_Buffer_Size, ... */
   - `HciSync::init4(hdev)?` /* Write_LE_Host_Supported, Write_Default_Erroneous_Data_Reporting, Write_Default_Link_Policy_Settings, Read_Local_Codecs, Set_Event_Mask, ... */
   - If `hdev.post_init.is_some()`: `hdev.post_init(hdev)?`.
7. `hdev.clear_flag(HCI_INIT); hdev.set_flag(HCI_UP)`.
8. `hci_sock_dev_event(hdev, HCI_DEV_UP)`.
9. `Ok(())`.

`Hci::send_cmd(hdev, opcode, plen, param) -> Result<(), Errno>`:
1. `skb = HciCmdSync::alloc(hdev, opcode, plen, param, None).ok_or(ENOMEM)?`.
2. `bt_cb(&mut skb).hci.req_flags |= HCI_REQ_START`.
3. `skb_queue_tail(&hdev.cmd_q, skb)`.
4. `queue_work(&hdev.workqueue, &hdev.cmd_work)`.
5. `Ok(())`.

`Hci::send_frame(hdev, skb) -> Result<(), Errno>` (internal):
1. `__net_timestamp(&mut skb)`.
2. `hci_send_to_monitor(hdev, &skb)`.
3. if `atomic_read(&hdev.promisc) > 0`: `hci_send_to_sock(hdev, &skb)`.
4. `skb_orphan(&mut skb)`.
5. if `!hdev.test_flag(HCI_RUNNING)` ⟹ `Err(EINVAL)` + `kfree_skb`.
6. if pkt_type == HCI_DRV_PKT: `HciDrv::process_cmd(hdev, skb)`; `kfree_skb`; return.
7. `hdev.send(hdev, skb)`.

`Hci::recv_frame(hdev, skb) -> Result<(), Errno>`:
1. `if hdev.is_null() || (!HCI_UP && !HCI_INIT) ⟹ Err(ENXIO) + free`.
2. `t = hci_dev_classify_pkt_type(hdev, &skb)`.
3. if `hci_skb_pkt_type(&skb) != t`: set to `t`.
4. match pkt_type:
   - HCI_EVENT_PKT / HCI_SCODATA_PKT / HCI_ISODATA_PKT / HCI_DRV_PKT: accept.
   - HCI_ACLDATA_PKT: if any CIS/BIS/PA conn exists, lookup by ACL-handle; if conn-type ∈ {CIS,BIS,PA}: reclassify ISODATA.
   - default: `Err(EINVAL) + free`.
5. `bt_cb(&mut skb).incoming = 1; __net_timestamp(&mut skb)`.
6. `skb_queue_tail(&hdev.rx_q, skb); queue_work(&hdev.workqueue, &hdev.rx_work)`.
7. `Ok(())`.

`Hci::rx_work(hdev)`:
1. while let `Some(skb) = skb_dequeue(&hdev.rx_q)`:
   - kcov_remote_start(handle); defer kcov_remote_stop.
   - `hci_send_to_monitor(hdev, &skb)`; if promisc: `hci_send_to_sock`.
   - if `HCI_USER_CHANNEL && !HCI_INIT`: drop.
   - if `HCI_INIT && pkt_type ∈ {ACL,SCO,ISO}`: drop.
   - dispatch:
     - EVENT → `HciEvent::event_packet(hdev, skb)`.
     - ACL → `Hci::acldata_packet(hdev, skb)`.
     - SCO → `Hci::scodata_packet(hdev, skb)`.
     - ISO → `Hci::isodata_packet(hdev, skb)`.

`Hci::cmd_work(hdev)`:
1. if `cmd_cnt > 0`:
   - `skb = skb_dequeue(&hdev.cmd_q)?`.
   - `err = Hci::send_cmd_sync(hdev, skb)?`.
   - `hdev.sent_cmd = Some(skb_clone(&skb)?)`.
   - if a sync request is pending + `!test_and_set(HCI_CMD_PENDING)`: `hdev.req_skb = Some(skb_clone(&hdev.sent_cmd)?)`.
   - `atomic_dec(&hdev.cmd_cnt)`.
2. under rcu: if `HCI_RESET | HCI_CMD_DRAIN_WORKQUEUE`: drop remainder.
3. else if cmd_q non-empty + cmd_cnt > 0: `schedule_delayed_work(&hdev.cmd_timer, HCI_CMD_TIMEOUT)`.

`Hci::req_cmd_complete(hdev, opcode, status, req_complete, req_complete_skb)`:
1. if `hci_sent_cmd_data(hdev, opcode).is_none()`:
   - if `HCI_INIT && opcode == HCI_OP_RESET`: `Hci::resend_last(hdev)`.
   - return.
2. `hdev.clear_flag(HCI_CMD_PENDING)`.
3. if `status == 0 && !Hci::req_is_complete(hdev)`: return.
4. let skb = `hdev.req_skb.take()`.
5. if `bt_cb(&skb).hci.req_flags & HCI_REQ_SKB`: deliver via `req_complete_skb`.
6. else: deliver via `req_complete`.
7. Drain trailing same-request cmds from `cmd_q` under `cmd_q.lock` IRQ-safe.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `register_requires_open_close_send` | INVARIANT | `register_dev` returns Err(EINVAL) iff any of open/close/send is null. |
| `id_unique` | INVARIANT | `ida_alloc_max` from HCI_INDEX_IDA ⟹ no two live hdev share the same id. |
| `hci_up_implies_running` | INVARIANT | `HCI_UP` set ⟹ `HCI_RUNNING` set (driver send-callback usable). |
| `init_clears_user_acl_sco_iso` | INVARIANT | `HCI_INIT` set ⟹ rx_work drops ACL/SCO/ISO before dispatch. |
| `user_channel_blocks_normal_path` | INVARIANT | `HCI_USER_CHANNEL && !HCI_INIT` ⟹ rx_work drops every skb regardless of pkt_type. |
| `pkt_type_in_set` | INVARIANT | accepted pkt_type ∈ {EVENT, ACL, SCO, ISO, DRV}; everything else freed with EINVAL. |
| `recv_acl_reclassify_iso_when_handle_is_iso` | INVARIANT | ACL pkt whose conn-handle resolves to CIS/BIS/PA reclassified to ISODATA. |
| `cmd_q_fifo` | INVARIANT | `cmd_work` dequeues in enqueue order under ordered wq. |
| `sent_cmd_clone_lifetime` | INVARIANT | `hdev.sent_cmd` lives until next `hci_send_cmd_sync` overwrites it. |
| `cmd_timer_only_armed_when_cmd_in_flight` | INVARIANT | `cmd_timer` scheduled iff `cmd_q` non-empty + `cmd_cnt > 0` + !DRAIN. |
| `register_then_unregister_balances_ida` | INVARIANT | for each successful register, unregister + release_dev calls `ida_free(hci_index_ida, id)` exactly once. |
| `unregister_blocks_dev_get` | INVARIANT | `HCI_UNREGISTER` set ⟹ `hci_dev_get(idx)` returns None. |
| `monitor_fanout_for_every_send_and_recv` | INVARIANT | every skb routed through send_frame/recv_frame is copied to `hci_send_to_monitor`. |

### Layer 2: TLA+

`net/bluetooth/hci_core.tla`:
- State: per-hdev (flags, cmd_q, rx_q, cmd_cnt, sent_cmd-opcode, req_pending) + abstract driver (send-success/fail).
- Actions: Register, Open (Init1..Init4), SendCmd, DriverDeliverCmdComplete, RecvFrame, RxWork, TxWork, CmdTimeout, Close, Unregister, Suspend, Resume.
- Properties:
  - `safety_no_send_before_running` — `HCI_RUNNING` cleared ⟹ no `send_frame` succeeds.
  - `safety_cmd_complete_matches_sent` — CMD_COMPLETE/CMD_STATUS event opcode == hdev.sent_cmd opcode OR triggers `hci_resend_last` (HCI_INIT + HCI_OP_RESET only).
  - `safety_user_channel_isolates_kernel` — `HCI_USER_CHANNEL` set ⟹ kernel ignores ACL/SCO/ISO RX.
  - `safety_init_order_INIT1_to_INIT4` — `HCI_UP` set ⟹ INIT1..INIT4 ran in order with no skipped stage.
  - `liveness_cmd_eventually_completes` — every accepted cmd: either CMD_COMPLETE/CMD_STATUS arrives, or `cmd_timer` fires within HCI_CMD_TIMEOUT and the cmd is cancelled.
  - `liveness_register_eventually_powers_on` — `register_dev` ⟹ eventually `power_on` work runs OR rfkill blocks.
  - `liveness_unregister_eventually_drains` — `unregister_dev` ⟹ eventually `release_dev` runs (after all srcu readers + refs drop).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `register_dev` post: id ∈ [0, HCI_MAX_ID), hdev linked, sysfs+debugfs created, ref held, HCI_DEV_REG broadcast | `Hci::register_dev` |
| `unregister_dev` post: HCI_UNREGISTER set, hdev unlinked, mgmt_pending empty, HCI_DEV_UNREG broadcast | `Hci::unregister_dev` |
| `release_dev` post: workqueues destroyed, ida freed, all per-list cleared, hdev freed | `Hci::release_dev` |
| `send_cmd` post: skb appended to cmd_q tail, cmd_work queued | `Hci::send_cmd` |
| `send_frame` post: if HCI_RUNNING then hdev.send invoked exactly once with `skb_orphan`'d skb; if not HCI_RUNNING, skb freed and EINVAL returned | `Hci::send_frame` |
| `recv_frame` post: pkt classified, queued on rx_q, rx_work queued; malformed pkt_type returns EINVAL with skb freed | `Hci::recv_frame` |
| `rx_work` post: each pkt dispatched to per-pkt-type handler; ACL/SCO/ISO suppressed during HCI_INIT | `Hci::rx_work` |
| `cmd_work` post: at most one cmd dequeued per credit; cmd_timer scheduled iff cmd_q non-empty + cmd_cnt > 0 + !DRAIN | `Hci::cmd_work` |
| `req_cmd_complete` post: matched cmd ⟹ HCI_CMD_PENDING cleared; req_complete{,_skb} delivered exactly once | `Hci::req_cmd_complete` |
| `dev_open_sync` post: HCI_UP ⟺ INIT1..INIT4 succeeded ∨ HCI_UNCONFIGURED ∨ HCI_USER_CHANNEL | `HciSync::dev_open` |
| `dev_close_sync` post: HCI_UP cleared, queues purged, conn_hash flushed, driver close called | `HciSync::dev_close` |

### Layer 4: Verus/Creusot functional

`Per-controller HCI lifecycle (register → power_on → init1..4 → up → cmd/event/data flow → down → unregister → release)` semantic equivalence: per-Bluetooth Core Specification v6.0 HCI layer (Vol 4, Part E) + per-Linux 7.1.0-rc2 driver ABI (`include/net/bluetooth/hci_core.h`).

## Hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening — KASAN, KCSAN, SLUB_DEBUG, RANDSTRUCT, fortify-source.)

HCI-specific reinforcement:

- **Per-hdev refcount via `hci_dev_hold`/`hci_dev_put`** — defense against use-after-unregister UAF; release_dev only runs at refcount==0.
- **Per-`HCI_UNREGISTER` flag + SRCU + unregister_lock** — defense against `hci_dev_get` racing unregister; new lookups return None.
- **Per-`open/close/send` null-check pre-register** — defense against half-wired transport driver crashing on first send.
- **Per-ordered workqueue (`alloc_ordered_workqueue`) for cmd/rx/tx + separate req_workqueue** — defense against parallel cmd injection re-ordering CMD_COMPLETE matching; defense against power-on work blocking command dispatch.
- **Per-`pkt_type` validated + `hci_dev_classify_pkt_type` override** — defense against driver mis-classifying ISO-as-ACL on quirky controllers.
- **Per-`HCI_RUNNING` checked in `hci_send_frame`** — defense against TX after driver has torn down its endpoint/URB pool.
- **Per-`skb_orphan` before driver send** — defense against driver freeing skb back to a stale destructor.
- **Per-cmd_timer + ncmd_timer** — defense against wedged controller silently swallowing commands (forces cancel_sync after `HCI_CMD_TIMEOUT`).
- **Per-`HCI_CMD_DRAIN_WORKQUEUE` flag + RCU sync** — defense against late-queued cmd timers re-arming after reset.
- **Per-`hci_send_to_monitor` fanout for every TX + RX** — defense against tampered loggers (btmon sees everything authoritatively).
- **Per-`mgmt_pending` BUG_ON empty at unregister** — defense against mgmt-cmd leak across unregister.
- **Per-rfkill integration auto-shutdown** — defense against radio-blocked controller transmitting.
- **Per-`HCI_USER_CHANNEL` isolation in rx_work** — defense against kernel L2CAP/SCO/ISO accepting data while userspace owns the controller (BlueZ user-channel mode).
- **Per-CSR HCI_OP_RESET resend tolerance** — defense against bogus spontaneous CMD_COMPLETE breaking the request state-machine.
- **Per-PM suspend notifier with `HCI_QUIRK_NO_SUSPEND_NOTIFIER` opt-out** — defense against suspend-vs-cmd races on misbehaving controllers.
- **Per-`disable_work_sync` (not `cancel_work_sync`) at unregister** — defense against work re-arming during teardown.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `net/bluetooth/hci_event.c` per-event-code handler bodies (covered by an `hci-event.md` Tier-3 if expanded)
- `net/bluetooth/hci_sock.c` raw HCI socket family (covered separately)
- `net/bluetooth/hci_sync.c` full INIT1..INIT4 command bodies (header sequence here is normative; per-command bodies covered separately)
- `net/bluetooth/hci_conn.c` per-`hci_conn` state-machine details (covered separately)
- `net/bluetooth/hci_request.c` legacy `__hci_req_sync` helpers (covered separately)
- `net/bluetooth/mgmt.c` Management API command/event surface (covered separately)
- `net/bluetooth/{l2cap,sco,iso,rfcomm,bnep,hidp,smp,6lowpan,mesh}.c` upper-protocol layers
- `drivers/bluetooth/*` per-transport drivers (btusb, btintel, btmtk, btrtl, hci_uart, btsdio, hci_vhci, ...)
- BlueZ userspace daemons (bluetoothd, hciconfig, hcitool, btmgmt, btmon) — out-of-tree
- Bluetooth Core Specification — out-of-tree normative reference
- Implementation code
