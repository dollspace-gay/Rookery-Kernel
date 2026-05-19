# Tier-3: net/nfc/nci/ + drivers/nfc/nfcmrvl/main.c — NCI core (NFC Forum Controller Interface)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/nfc/00-overview.md
upstream-paths:
  - net/nfc/nci/core.c
  - net/nfc/nci/data.c
  - net/nfc/nci/ntf.c
  - net/nfc/nci/rsp.c
  - net/nfc/nci/hci.c
  - net/nfc/nci/uart.c
  - net/nfc/nci/spi.c
  - net/nfc/nci/lib.c
  - drivers/nfc/nfcmrvl/main.c
  - include/net/nfc/nci.h
  - include/net/nfc/nci_core.h
-->

## Summary

NCI (NFC Controller Interface, NFC-Forum spec NCI 2.0) implementation. Provides the control + data pipe used by every modern NCI-mode NFC controller — `s3fwrn5`, `nfcmrvl`, `nxp-nci`, `st-nci`, `fdp`, plus the digital-soft path used by `trf7970a`/`st95hf`. Per controller, the driver supplies `struct nci_ops` (`open`/`close`/`send`/`setup`/`fw_download`/`post_setup`) and the core drives the protocol: command/response/notification dispatch on a 3-byte-header packet pipe, request/reply synchronization, RF discovery state machine, data connection (logical NCI conn) management, and the embedded HCI sub-protocol used by SE bridges.

This Tier-3 covers `net/nfc/nci/core.c` (~1650 lines: cmd issue, response/notification dispatch, RF discover, conn create/close, send/recv frame), `data.c` (~305 lines: data-credit + segmentation/reassembly), `ntf.c` (~1040 lines: per-notification handlers), `rsp.c` (~430 lines: per-response handlers), `hci.c` (~795 lines: NCI-embedded HCI for SE), `uart.c` + `spi.c` (transport line-disciplines), and `drivers/nfc/nfcmrvl/main.c` (~257 lines: representative NCI-mode driver wrapping a Marvell 88W8987).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct nci_dev` (`include/net/nfc/nci_core.h`) | per-controller NCI device | `nci::Dev` |
| `struct nci_ops` | per-driver vtable (`open`/`close`/`send`/`setup`/`post_setup`/`fw_download`/`prop_response`/`prop_ntf`) | `nci::Ops` |
| `nci_allocate_device(ops, supported_protocols, tx_headroom, tx_tailroom)` | alloc | `nci::Dev::alloc` |
| `nci_free_device(ndev)` | free | `nci::Dev::drop` |
| `nci_register_device(ndev)` / `_unregister_device(ndev)` | register/unregister with nfc-core | `nci::Dev::register` / `_unregister` |
| `nci_recv_frame(ndev, skb)` | driver→core inbound frame | `nci::Dev::recv_frame` |
| `nci_send_frame(ndev, skb)` | core→driver outbound frame | `nci::Dev::send_frame` |
| `nci_request(ndev, req_fn, opt, timeout)` / `__nci_request` (`net/nfc/nci/core.c`) | issue cmd + wait for rsp under lock | `nci::Dev::request` |
| `nci_core_reset(ndev)` / `_init(ndev)` | core reset + init handshake | `nci::Dev::core_reset` / `_init` |
| `nci_set_config(ndev, id, len, val)` | `CORE_SET_CONFIG_CMD` | `nci::Dev::set_config` |
| `nci_core_conn_create(ndev, dest_type, num_dest_params, dest_params, ...)` / `_close(ndev, conn_id)` | logical NCI data connection mgmt | `nci::Dev::conn_create` / `_close` |
| `nci_nfcee_discover(ndev, action)` / `_mode_set(ndev, nfcee_id, mode)` | SE discovery + mode | `nci::Dev::nfcee_discover` / `_mode_set` |
| `nci_prop_cmd(ndev, oid, len, payload)` / `nci_core_cmd(ndev, opcode, len, payload)` | proprietary + generic cmd | `nci::Dev::prop_cmd` / `_core_cmd` |
| `nci_data_exchange(ndev, conn_id, skb, cb, cb_context)` (`data.c`) | per-conn data exchange | `nci::Dev::data_exchange` |
| `nci_uart_register(nu)` (`uart.c`) | UART line discipline N_NCI | `nci::Uart::register` |
| `nci_spi_send(nspi, skb)` / `_read(nspi)` (`spi.c`) | SPI transport helper | `nci::Spi::send` / `_read` |
| `nfcmrvl_nci_register_dev` / `_unregister_dev` / `_recv_frame` (`drivers/nfc/nfcmrvl/main.c`) | Marvell phy-independent register | `drivers::nfc::nfcmrvl::register_dev` |

## Compatibility contract

REQ-1: NCI packet header is 3 bytes: `MT|PBF|GID` (1B) + `OID` (1B) + `payload_len` (1B). Packets are either Control (MT=1 cmd / 2 rsp / 3 ntf) or Data (MT=0, conn_id in low nibble of byte0). Core rejects malformed headers.

REQ-2: Per-command request/response synchronization: `__nci_request(ndev, req_fn, opt, timeout)` acquires `req_lock` mutex, issues `req_fn` (which calls `nci_send_cmd`), waits on `req_complete`, returns response status. `req_lock` defends against concurrent userspace genl commands stomping each other.

REQ-3: Opcode allowlist — only GID ∈ {`NCI_GID_CORE`(0x0), `_RF_MGMT`(0x1), `_NFCEE_MGMT`(0x2), `_PROPRIETARY`(0xf)} accepted; per-GID OID ranges checked.

REQ-4: Per-controller init handshake on `dev_up`: `CORE_RESET_CMD` → wait `CORE_RESET_NTF` → `CORE_INIT_CMD` → wait `CORE_INIT_RSP` (carries capability blob: max_logical_conns, max_routing_table_size, max_control_packet_payload_size, manfid). Driver `setup` + `post_setup` callbacks invoked after init.

REQ-5: RF discovery: `RF_DISCOVER_CMD` with per-protocol descriptors, `RF_DISCOVER_RSP` status, `RF_DISCOVER_NTF` per-target. `RF_DISCOVER_SELECT_CMD` activates a specific target, `RF_INTF_ACTIVATED_NTF` reports activation parameters.

REQ-6: Data connection management: per-conn id (1..7) created via `CORE_CONN_CREATE_CMD`, destroyed via `CORE_CONN_CLOSE_CMD`. Per-conn credit counter tracked in core; `CORE_CONN_CREDITS_NTF` replenishes; send blocks if credits == 0.

REQ-7: Segmentation/reassembly: control + data packets larger than `max_control_packet_payload_size` / `max_data_packet_payload_size` split with PBF (Packet Boundary Flag) bit set on non-final fragments.

REQ-8: Embedded HCI: `nci_hci.c` provides ETSI HCI gate/pipe protocol over NCI logical-conn-1, used to bridge to SE (UICC, embedded SE).

REQ-9: Transport-driver contract: `nci_ops.send(ndev, skb)` must consume the skb (free on error) and must NOT block; transport drivers do the actual MMIO/USB-submit/UART-tx asynchronously.

REQ-10: Per-cmd timeouts: `NCI_RESET_TIMEOUT`=5s, `NCI_INIT_TIMEOUT`=5s, `NCI_RF_DISC_TIMEOUT`=5s, `NCI_RF_DEACTIVATE_TIMEOUT`=30s, `NCI_CMD_TIMEOUT`=5s, `NCI_DATA_TIMEOUT`=3s.

REQ-11: Retransmit timer: per-conn segmentation uses driver-side or core-side retransmit on credit-stall; bounded to a configurable retry count under CAP_SYS_ADMIN.

## Acceptance Criteria

- [ ] AC-1: `s3fwrn5` UART driver loads, NCI `CORE_INIT` succeeds, `lsmod | grep nci` confirms core present.
- [ ] AC-2: `nfctool -p 0 -s` issues `RF_DISCOVER_CMD`, controller returns `RF_DISCOVER_RSP` STATUS_OK, `RF_DISCOVER_NTF` lists at least one target when a tag is in field.
- [ ] AC-3: `nci_data_exchange` round-trip on conn-id=0 (RF data) returns response APDU within `NCI_DATA_TIMEOUT`.
- [ ] AC-4: `virtual_ncidev` self-test: userspace controller plumbing replies to `CORE_RESET_CMD`/`CORE_INIT_CMD`, NCI state machine reaches OPENED.
- [ ] AC-5: HCI-over-NCI SE access (`nci_hci_send_event`, `nci_hci_send_cmd`) round-trip succeeds on `st-nci` with UICC present.
- [ ] AC-6: Malformed inbound frame (bad GID, len > MTU, truncated header) → frame dropped, no kernel oops, ratelimited dmesg.
- [ ] AC-7: kselftest `tools/testing/selftests/nfc/nci_test.sh` smoke-test passes against `virtual_ncidev`.

## Architecture

`nci::Dev` lives in `drivers::nfc::nci::Dev`:

```
struct Dev {
  nfc_dev: Arc<nfc::Device>,           // parent
  ops: &'static nci::Ops,
  state: AtomicU32,                    // NCI_INIT / NCI_OPEN / NCI_OPENED / NCI_CLOSED
  flags: AtomicU32,                    // NCI_UP / NCI_DATA_EXCHANGE / NCI_DATA_EXCHANGE_TO
  cmd_pending: Mutex<()>,              // req_lock
  req_complete: Completion<i32>,
  req_status: AtomicI32,
  cmd_timer: Timer,                    // NCI_CMD_TIMEOUT
  data_timer: Timer,                   // NCI_DATA_TIMEOUT
  cmd_q: SkbQueue,                     // outbound cmd queue
  rx_q: SkbQueue,                      // inbound (de-fragmented)
  tx_q: SkbQueue,                      // outbound data
  rx_wq: Workqueue,                    // run rx dispatch off softirq
  cmd_wq: Workqueue,
  tx_wq: Workqueue,
  conn_info: Mutex<Vec<ConnInfo>>,     // per-conn-id state (credits, dest_type)
  max_data_pkt_len: AtomicU16,
  max_ctrl_pkt_len: AtomicU16,
  hci_dev: Option<KBox<HciDev>>,       // embedded HCI for SE
  prop_ops: &'static [PropOp],         // proprietary opcode handlers
  prop_ntf_ops: &'static [PropOp],
  driver_data: NonNull<u8>,
}
```

Command issue path `nci::Dev::request(req_fn, opt, timeout)`:
1. Acquire `cmd_pending` mutex.
2. Init `req_complete`.
3. Call `req_fn(ndev, opt)` → builds skb + `nci_send_cmd(ndev, opcode, plen, payload)`.
4. `nci_send_cmd` allocates skb, prepends 3-byte header (`MT_CTRL`|`GID`, `OID`, `PLEN`), pushes onto `cmd_q`, schedules `cmd_wq`.
5. `cmd_wq` worker pops cmd, calls `ops.send(ndev, skb)`, arms `cmd_timer`.
6. Wait `wait_for_completion_interruptible_timeout(req_complete, timeout)`.
7. Return `req_status` (NCI status code 0=OK or POSIX errno).

Response/notification dispatch (`net/nfc/nci/core.c::nci_rx_work`):
1. Worker drains `rx_q` (populated by `nci_recv_frame` from driver).
2. Per-skb: parse header — MT, GID, OID, PBF, payload_len.
3. Validate GID ∈ allowlist; reject otherwise (drop + ratelimit-log).
4. If PBF set: append to per-conn reassembly buffer + continue.
5. Dispatch:
   - MT_CTRL+MT_RSP → `nci_rsp_packet(ndev, skb)` → opcode-table lookup in `rsp.c` → handler → complete `req_complete`.
   - MT_CTRL+MT_NTF → `nci_ntf_packet(ndev, skb)` → opcode-table lookup in `ntf.c` → handler → may update target list / NFCEE list / conn credits.
   - MT_DATA → `nci_rx_data_packet(ndev, skb)` → per-conn callback or `nfc_tm_data_received`.

Data exchange `nci::Dev::data_exchange(conn_id, skb, cb)`:
1. Validate `conn_id` ∈ per-dev table; refuse otherwise.
2. Stash `cb` in per-conn state.
3. Segment skb per `max_data_pkt_len`; per-segment header has PBF set on non-final.
4. Push each segment onto `tx_q` only if credit > 0; otherwise wait on `CORE_CONN_CREDITS_NTF`.
5. `tx_wq` worker: pop, decrement credit, `ops.send(ndev, skb)`.
6. Arm `data_timer` on first segment; cancel on per-conn data-recv notification.
7. On response: drain reassembly buffer, invoke `cb(context, skb, status)`.

RF discover path `nci_start_poll`:
1. Build `RF_DISCOVER_CMD` payload — per-protocol discover-config entries (mode + frequency).
2. `nci_request(nci_rf_discover_req, ...)` blocks 5s.
3. Per-target `RF_DISCOVER_NTF` accumulates targets into `ndev->targets[]`.
4. Final `RF_DISCOVER_NTF` (`Notification_Type=Last`) → `nfc_targets_found(nfc_dev, targets, n)` → upstream genl multicast.

Transport line-disciplines:
- **UART** (`uart.c`): registers N_NCI tty line-discipline; per-byte framed parse — header[0..2], then payload — with HDLC-style framing optional per-driver. Recv assembles skb, calls `nci_recv_frame`.
- **SPI** (`spi.c`): half-duplex per-message framing — driver pulls IRQ, reads header byte, reads payload, builds skb. CRC validation optional per `NCI_SPI_CRC_ENABLED` flag.

`drivers/nfc/nfcmrvl/main.c` (representative NCI-mode driver):
- `nfcmrvl_nci_register_dev(phy, dev, ops, fw_dnld_ops, pdata)`:
  1. Alloc `nfcmrvl_private` + `nci_allocate_device(&nfcmrvl_nci_ops, protocols, tx_headroom, tx_tailroom)`.
  2. Init firmware-download state machine (`nfcmrvl_fw_dnld_init`).
  3. `nci_register_device(priv->ndev)`.
- `nfcmrvl_nci_ops` provides: `open`, `close`, `send`, `setup`, `fw_download`.
- `nfcmrvl_nci_recv_frame`: forwards driver-recv'd skb to `nci_recv_frame` (or to fw-dnld state machine if mid-download).
- `nfcmrvl_nci_send`: forwards to phy-specific `ops->send` (USB/UART/I2C/SPI).

## Hardening

- **Opcode allowlist** — GID restricted to {CORE, RF_MGMT, NFCEE_MGMT, PROPRIETARY}; per-GID OID range checked before dispatch.
- **PBF reassembly cap** — per-conn reassembly buffer bounded by `max_ctrl_pkt_len * NCI_MAX_FRAGMENTS`; overflow drops + ratelimits.
- **Per-cmd timeout** — every `nci_request` armed with cmd_timer; firmware lockup recovered to userspace as `-ETIMEDOUT`.
- **Credit accounting** — per-conn credit counter saturating; never goes negative; `CORE_CONN_CREDITS_NTF` granting > `max_data_credits` rejected.
- **Cmd serialization** — `cmd_pending` mutex ensures only one outstanding `nci_request`; defeats genl-race-induced response cross-talk.
- **Proprietary opcode opt-in** — `prop_ops` table per-driver; unknown OID under PROPRIETARY GID dropped instead of dispatched.
- **Frame header validation** — header[2] (payload_len) must match remaining skb length; truncated frames dropped.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `nci_dev`, `conn_info`, NCI-skb head/tail, reassembly buffers, and `prop_op` arrays; per-message payload strictly bounded by `max_ctrl_pkt_len` / `max_data_pkt_len`.
- **PAX_KERNEXEC** — NCI core in W^X kernel text; `nci_ops`, `rsp.c` / `ntf.c` opcode-dispatch tables, and per-driver `prop_ops` live in `__ro_after_init` text.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `nci_request`, `nci_rx_work`, `nci_data_exchange`, and `nci_recv_frame` entries.
- **PAX_REFCOUNT** — saturating `refcount_t` on `nci_dev`, per-conn-info, and proprietary-op handlers; overflow trap defeats conn-create/close race UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for NCI skb head/tail, reassembly buffers, per-conn state, and firmware-download staging; defeats RF-traffic bleed across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every genl + `nci_data_exchange` user-data path; reject user-pointer deref outside canonical helpers.
- **PAX_RAP / kCFI** — `nci_ops`, `nci_driver_ops`, response/notification dispatch tables, and `prop_ops` indirect dispatch marked `__ro_after_init` with kCFI-typed signatures.
- **GRKERNSEC_HIDESYM** — gate kallsyms and per-`nci_dev` pointer disclosure behind CAP_SYSLOG; suppress `%p` in cmd/rsp/ntf tracepoints.
- **GRKERNSEC_DMESG** — restrict `RF_DISCOVER_NTF`, `RF_INTF_ACTIVATED_NTF`, target manfid, and SE banners to CAP_SYSLOG so attackers cannot probe tag presence via dmesg.
- **NCI message PAX_USERCOPY whitelist** — slab caches for NCI skbs marked usercopy-whitelist only over the precise payload window the genl path needs; refuse out-of-window copies.
- **Opcode allowlist hardened** — GID restricted to {CORE, RF_MGMT, NFCEE_MGMT, PROPRIETARY}; per-GID OID range checked; unknown opcodes dropped + ratelimited.
- **Retransmit/timer CAP_SYS_ADMIN** — per-conn retry count + retransmit timer adjustable only by CAP_SYS_ADMIN; defeats unprivileged DoS via repeated retransmits.
- **Target-buffer SIZE_OVERFLOW** — `RF_DISCOVER_NTF` per-target manfid + parameters length-checked against header[2] under SIZE_OVERFLOW; refuse mismatched lengths.
- **Reassembly bound** — per-conn PBF reassembly capped at `max_ctrl_pkt_len * NCI_MAX_FRAGMENTS`; overflow drops the in-progress reassembly without trusting attacker length fields.
- **Credit-overflow trap** — per-conn credit grant from `CORE_CONN_CREDITS_NTF` saturated and bounded against `max_data_credits`; refuse over-grant.

Rationale: an NCI controller is a black-box MCU on the other side of a 3-byte-header pipe; a missing length check on `RF_DISCOVER_NTF`, an unbounded reassembly buffer, a stomp-on-credit, or an out-of-allowlist GID is enough to corrupt kernel memory from the RF field — i.e. from any tag waved near the antenna. RAP/kCFI on `nci_ops`/`prop_ops`, opcode allowlist, reassembly bound, credit saturation, refcount-overflow trapping, and CAP_SYS_ADMIN on retransmit parameters turn the NCI core from "we trust the controller MCU" into a defense-in-depth boundary against a hostile peer that need not even authenticate to the host.

## Open Questions

(none at this Tier-3 level)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `nci_header_no_oob` | OOB | per-skb header[0..2] access bounded; payload_len ≤ skb_data_len. |
| `conn_id_no_oob` | OOB | `conn_id` ∈ 0..NCI_MAX_CONN_ID before per-conn table access. |
| `pbf_reassembly_no_overflow` | OOB | per-conn reassembly buffer bounded by `max_ctrl_pkt_len * NCI_MAX_FRAGMENTS`. |
| `cmd_complete_no_uaf` | UAF | `req_complete` outlives async response dispatch; cmd_timer cancels before complete-drop. |

### Layer 2: TLA+

`models/nfc/nci_req_rsp.tla` (this doc declares it): proves cmd→rsp serialization under `cmd_pending` mutex, including timer-fire concurrent with response-arrival, and including reassembled-segmented responses; encodes credit accounting as a counting semaphore.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `nci::Dev::request` post: returns with `cmd_pending` released, regardless of completion vs timeout | `nci::Dev::request` |
| Per-conn invariant: credit counter ≥ 0 always; never overflows `max_data_credits` | `data.c` credit account |
| Opcode dispatch pre: GID ∈ allowlist ∧ OID ∈ per-GID table | `nci_rsp_packet` / `nci_ntf_packet` |

### Layer 4: Verus/Creusot functional

Userspace `NFC_CMD_START_POLL` → genl handler → `nci_start_poll` → `nci_request(nci_rf_discover_req)` → controller MCU dispatches RF discover → tag returned in `RF_DISCOVER_NTF` → reassemble → `nfc_targets_found` → genl multicast. Encoded as Verus invariant chained with `00-overview.md`'s poll-state machine.

## Out of Scope

- HCI-over-NCI internals (covered by future `nci-hci.md`)
- LLCP-over-NCI peer-to-peer (covered by future `llcp.md`)
- Per-driver firmware-download details (per-driver Tier-3)
- 32-bit-only paths
- Implementation code
